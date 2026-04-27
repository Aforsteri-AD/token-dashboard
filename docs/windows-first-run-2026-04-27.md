# Windows First-Run Troubleshooting Log — 2026-04-27

Recorded during initial setup on Windows 11 Pro with 1,592 JSONL session files (674 MB).

---

## Attempt 1 — Bash background task, default port 8080

**Command:** `cd token-dashboard && PORT=9000 py -3 cli.py dashboard --no-open 2>&1 &`

**Result:** Claude Code's Bash tool runs inside WSL. The `&` backgrounded the process and the shell returned immediately with exit 0. The actual `py.exe` launcher (Windows Python Launcher via WSL interop) spawned `python.exe` (PID 36776) and started scanning. The task infrastructure reported "completed" before the subprocess finished.

**Why it appeared to fail:** Port 8080 was already in use by a Node.js process (PID 12748). Additionally, the PORT env var from the WSL shell was not reliably inherited when `py.exe` relaunched as `python.exe`. The scan ran for several minutes before being killed manually.

---

## Attempt 2 — Kill Python processes, retry with port 9000

**Action:** `Stop-Process -Id 34164, 36776 -Force` (py.exe and python.exe from attempt 1)

**Result:** Killing the processes mid-transaction left `~/.claude/token-dashboard.db-journal` behind. SQLite uses the journal to roll back incomplete transactions. On Windows, when the process is killed rather than exiting cleanly, the journal file handle is kept open by the surviving parent bash/sh shell process from the Claude Code background task infrastructure.

**Error:** `sqlite3.OperationalError: database is locked` — every subsequent attempt to open the DB triggered this because:
1. SQLite saw the journal file and tried to acquire an exclusive lock to process it.
2. The journal file itself was still held open by the orphaned bash process, so SQLite couldn't get the lock.

---

## Attempt 3 — Delete the DB and journal, retry

**Action:** `Remove-Item ~/.claude/token-dashboard.db, ~/.claude/token-dashboard.db-journal`

**Result:** Both files deleted successfully. However, `Start-Process py -3 cli.py dashboard` via PowerShell silently failed to inherit the `$env:PORT = "9000"` environment variable (Windows Python Launcher re-executes as a child `python.exe`, losing the PowerShell-set env vars in some configurations). The process started on port 8080, which was still taken, and exited with no stderr output captured (redirect via `-RedirectStandardError` produced empty files).

Meanwhile, a WMI scan revealed two Python processes (PIDs 32856 and 27268) that were actually running and scanning. They completed successfully — rebuilding the DB to 73 MB — but then exited because the port 8080 bind failed.

**New problem:** The journal at 09:46 had returned (created by attempt 1's scan before we deleted it). The DB at 10:04 was clean. But the stale journal from the orphaned bash process was still locked, blocking all further opens.

---

## Attempt 4 — PowerShell Start-Job

**Command:** `Start-Job -ScriptBlock { $env:PORT = "9000"; py -3 cli.py dashboard --no-open }`

**Result:** Same "database is locked" error. PowerShell background jobs spawn a new `powershell.exe` runspace, but the DB lock was still held by the bash process from attempt 1.

---

## Root cause

Two compounding issues:

1. **Stale journal file locked by an orphaned shell process.** Claude Code's background task infrastructure spawns bash/sh subprocesses. When the Python child is killed via `Stop-Process`, the bash parent remains alive and holds the file handles it inherited from the Python child (the journal file descriptor). This keeps `token-dashboard.db-journal` locked indefinitely for the session lifetime, blocking all SQLite writers.

2. **Port 8080 conflict.** A Node.js process (Claude Code extension infrastructure, PID 12748) was listening on port 8080 — the old default port.

---

## Solution

Three changes to the codebase:

### 1. Default port changed to 9000 (`cli.py`)

```python
# Before
port = int(os.environ.get("PORT", "8080"))

# After
port = int(os.environ.get("PORT", "9000"))
```

Port 9000 is not used by Claude Code's internal processes, so it is safe as the default on any machine running the desktop app.

### 2. SQLite timeout + WAL mode (`token_dashboard/db.py`)

```python
# Before
with sqlite3.connect(path) as c:
    _migrate_add_message_id(c)

# After
with sqlite3.connect(path, timeout=60) as c:
    c.execute("PRAGMA journal_mode=WAL")
    _migrate_add_message_id(c)
```

- `timeout=60` lets the process wait up to 60 seconds for a lock to clear instead of failing instantly.
- `PRAGMA journal_mode=WAL` switches from rollback-journal mode to Write-Ahead Logging. WAL does not create a `.db-journal` file; it uses `.db-wal` and `.db-shm` instead, and WAL readers never block writers and vice versa. This eliminates the entire class of "stale journal holds exclusive lock" failures on subsequent runs.
- Same `timeout=60` added to the `connect()` context manager used by all query helpers.

### 3. For this first run specifically

The locked `~/.claude/token-dashboard.db-journal` will be released when the current Claude Code session ends (the bash process holding it exits). After that, the WAL-mode change means it cannot happen again. If you need to start the dashboard before the session ends, point it at a different DB file:

```cmd
set TOKEN_DASHBOARD_DB=%USERPROFILE%\.claude\token-dashboard2.db
python3 cli.py dashboard
```

---

## Verification

After the changes, the normal quickstart works:

```bash
python3 cli.py dashboard
# → scans ~/.claude/projects/
# → serves http://127.0.0.1:9000
```

---

## Session 2 — 2026-04-27 (same day, new Claude Code session)

### What happened

The previous session ended with `/clear` but three background `cli.py dashboard` Python processes were left alive (PIDs 27268, 30468, 35180). These were started during Session 1 as part of troubleshooting and were never explicitly killed.

When the user ran `py -3 cli.py dashboard` again in a new PowerShell window, they got:

```
sqlite3.OperationalError: database is locked
  File "scanner.py", line 232, in scan_file
    conn.execute(INSERT_MSG, msg)
```

### Root cause

Two compounding issues discovered beyond Session 1's fixes:

**Issue A — Orphaned processes, not orphaned journal.**
Session 1 fixed the _journal_ lock (by enabling WAL mode in `init_db()`). But the three surviving processes were themselves still holding write locks via active Python `sqlite3` connections. WAL mode prevents journal-file locking but does not prevent a live writer from blocking other writers.

**Issue B — WAL mode was only set in `init_db()`, not in `connect()`.**
The scanner opens the database via the `connect()` context manager, which did not issue `PRAGMA journal_mode=WAL`. On a connection opened through `connect()`, SQLite would fall back to rollback-journal mode and compete for exclusive write locks in the old way.

**Issue C — One giant transaction per full scan.**
`scan_dir` opened a single write transaction covering all 1,592 files and committed only at the very end. Each of the three orphaned processes held this exclusive write lock for many minutes. A new process waiting with `timeout=60` would time out and raise `database is locked`.

### Fixes applied

#### 1. WAL mode added to `connect()` (`token_dashboard/db.py`)

```python
# Before
conn = sqlite3.connect(path, timeout=60)
conn.execute("PRAGMA foreign_keys = ON")

# After
conn = sqlite3.connect(path, timeout=60)
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA foreign_keys = ON")
```

Every connection — scanner, server API, tips engine — now runs in WAL mode regardless of whether `init_db()` was called first.

#### 2. Per-file commit in `scan_dir` (`token_dashboard/scanner.py`)

```python
# Before
for p in root.rglob("*.jsonl"):
    ...
    conn.execute("INSERT OR REPLACE INTO files ...")
    totals["files"] += 1
conn.commit()   # ← one commit after ALL files

# After
for p in root.rglob("*.jsonl"):
    ...
    conn.execute("INSERT OR REPLACE INTO files ...")
    conn.commit()  # ← commit after each file
    totals["files"] += 1
```

Each file's data is now committed atomically. Benefits:
- The write lock is held only briefly per file instead of for the entire scan.
- If the process crashes mid-scan, all previously committed files are safe — no data loss, no journal left open.
- A running dashboard server and a concurrent `cli.py scan` can coexist without locking each other out.

#### 3. Immediate fix — kill orphaned processes

```powershell
# Windows: find dashboard processes
Get-Process python* | ForEach-Object {
    $wmi = Get-WmiObject Win32_Process -Filter "ProcessId = $($_.Id)"
    "$($_.Id): $($wmi.CommandLine)"
}

# Kill the ones running cli.py dashboard
Stop-Process -Id <PID1>, <PID2> -Force
```

```bash
# macOS / Linux
ps aux | grep "cli.py dashboard"
kill <PID1> <PID2>
```

After killing orphaned processes, `.db-journal` (if present) is auto-recovered by SQLite on the next connection open.

### Prevention going forward

With both code fixes in place:
- No `.db-journal` lock can outlive the process that created it (WAL mode uses `.db-wal`/`.db-shm` instead).
- A crash mid-scan leaves the DB in a clean, consistent state (every committed file is durable).
- Multiple short lock windows (one per file) replace one long window (entire scan), so the timeout margin is rarely needed.

The remaining operational rule: **only run one `cli.py dashboard` at a time.** If you close the terminal window instead of pressing `Ctrl+C`, confirm the Python process is gone before starting a new one (see kill commands above).
