---
name: run-token-dashboard
description: Use when the user asks to start, run, open, or launch the local Claude Code token usage dashboard. Also use when needing to restart the dashboard server.
---

# Run Token Dashboard

## Overview
Starts the token dashboard server at http://127.0.0.1:9000 on Windows. The server scans JSONL transcripts silently before accepting connections — always poll before declaring success, and only open the browser once.

**Project:** `C:\Users\ArnaudDeveugle\Claude\token-dashboard`

## Steps

### 1. Check If Already Running
If the dashboard is already up, open the browser once (if not already open from this session) and stop.

```powershell
try {
    $null = Invoke-WebRequest "http://127.0.0.1:9000/api/overview" -UseBasicParsing -TimeoutSec 2
    Start-Process "http://127.0.0.1:9000/"
    Write-Host "Dashboard already running at http://127.0.0.1:9000"
    exit
} catch {}
```

### 2. Kill Any Orphaned Processes
Orphaned python processes hold the SQLite write lock and silently block startup.

```powershell
Get-Process python* -ErrorAction SilentlyContinue | ForEach-Object {
    $wmi = Get-WmiObject Win32_Process -Filter "ProcessId = $($_.Id)" -ErrorAction SilentlyContinue
    if ($wmi.CommandLine -like "*cli.py*") { Stop-Process -Id $_.Id -Force }
}
```

### 3. Tell the User to Wait
Say: **"Starting the token dashboard, please wait…"**

### 4. Start the Server
`python` not `python3` — `python3` is not on the Windows PATH here. Use `--no-open` so the agent controls when the browser opens.

```powershell
$proj = "C:\Users\ArnaudDeveugle\Claude\token-dashboard"
$dbExists = Test-Path "$env:USERPROFILE\.claude\token-dashboard\token-dashboard.db"
$cmdArgs = @("cli.py", "dashboard", "--no-open")
if ($dbExists) { $cmdArgs += "--no-scan" }  # fast start when cache exists; server rescans every 30s anyway
Start-Process python -ArgumentList $cmdArgs -WorkingDirectory $proj -WindowStyle Minimized
```

If `$dbExists` is false, additionally tell the user: **"No cache found — first-time scan in progress (may take 1–2 minutes)…"**

### 5. Poll Until Ready (max 2 minutes)
The scan window is silent — do not tell the user to look at any window. Just poll.

```powershell
$ready = $false
for ($i = 0; $i -lt 40; $i++) {
    Start-Sleep 3
    try {
        $null = Invoke-WebRequest "http://127.0.0.1:9000/api/overview" -UseBasicParsing -TimeoutSec 2
        $ready = $true; break
    } catch {}
    if ($i -eq 9)  { Write-Host "Still scanning… (30s)" }
    if ($i -eq 19) { Write-Host "Still scanning… (60s)" }
}
```

### 6. Open Browser and Report Result
```powershell
if ($ready) {
    Start-Process "http://127.0.0.1:9000/"
    # Tell user: "Dashboard is ready — opening http://127.0.0.1:9000"
} else {
    # Tell user: "Server did not start within 2 minutes."
    # Diagnose: check for error output (see Self-Improvement section)
}
```

### 7. Background Health Check
After opening, wait 30s and confirm the server is still alive:

```powershell
Start-Sleep 30
try {
    $null = Invoke-WebRequest "http://127.0.0.1:9000/api/overview" -UseBasicParsing -TimeoutSec 3
    # Healthy — nothing to report
} catch {
    # Tell user: "Dashboard stopped responding. Run the skill again to restart it."
}
```

## Common Issues

| Problem | Fix |
|---------|-----|
| `database is locked` on startup | Step 2 missed orphan — kill all `python*` processes, retry |
| Empty/silent terminal window | Expected — scan runs silently; Step 5 poll handles this |
| `python3 not found` | Use `python` — Step 4 already does this |
| Browser opens but shows no data | DB empty; restart without `--no-scan` to force a fresh scan |
| Port 9000 already in use (non-dashboard) | Set `$env:PORT = "9001"` before Step 4, poll port 9001 |
| Server exits immediately | Add `-RedirectStandardError "$env:TEMP\td_err.txt"` to Start-Process, check that file |

## Self-Improvement
When you encounter a new error not in the table above:
1. Fix the problem
2. **Edit both copies of this skill:**
   - `C:\Users\ArnaudDeveugle\.claude\skills\run-token-dashboard\SKILL.md` (active)
   - `C:\Users\ArnaudDeveugle\Claude\token-dashboard\skills\run-token-dashboard\SKILL.md` (repo)
3. Add the fix to the Common Issues table
4. Update the Steps if the fix requires a process change
