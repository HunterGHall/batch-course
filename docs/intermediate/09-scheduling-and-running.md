# 9. Scheduling & Running Scripts

## What & why

A backup or cleanup script only earns its keep if it runs on a schedule without you. This lesson covers `schtasks` (create/query/delete scheduled tasks from the command line), how to launch things with `start`, elevation with `runas`, and the environment surprises that make "works when I run it, fails at 2 AM" happen.

## `schtasks` — scheduled tasks from batch

### Create

```bat
schtasks /Create /TN "MyBackup" /TR "\"%~dp0backup.bat\" --quiet" /SC DAILY /ST 02:00 /RL HIGHEST /F
```

| Switch | Meaning |
| --- | --- |
| `/TN` | task name (use a folder: `/TN "\MyOrg\MyBackup"`) |
| `/TR` | the command to run — **quote the script path**, and escape inner quotes as `\"` |
| `/SC` | schedule: `MINUTE HOURLY DAILY WEEKLY MONTHLY ONCE ONLOGON ONIDLE ONSTART` |
| `/ST` | start time `HH:MM` (24h) |
| `/MO` | modifier: `/SC MINUTE /MO 15` = every 15 min; `/SC WEEKLY /MO 1 /D MON,WED,FRI` |
| `/RL HIGHEST` | run elevated (needs admin to create) |
| `/RU` `/RP` | run-as user / password (`/RU SYSTEM` for the system account, no password) |
| `/F` | force: overwrite an existing task with the same name |
| `/IT` | interactive — only runs when that user is logged on |

### Query

```bat
schtasks /Query /TN "MyBackup" /V /FO LIST
schtasks /Query /FO CSV /NH | findstr /i "backup"
```

### Run now (test it)

```bat
schtasks /Run /TN "MyBackup"
rem then check the last result:
schtasks /Query /TN "MyBackup" /V /FO LIST | findstr /i "Last Result"
```

"Last Result" `0` = success; `267009` = currently running; `267011` = never run; other codes map to your script's `exit /b`.

### Delete

```bat
schtasks /Delete /TN "MyBackup" /F
```

### Self-installing script pattern

```bat
@echo off
if /i "%~1"=="--install" (
    schtasks /Create /TN "\MyOrg\Backup" /TR "\"%~f0\" --run" /SC DAILY /ST 02:00 /RL HIGHEST /F
    exit /b %errorlevel%
)
if /i "%~1"=="--uninstall" (
    schtasks /Delete /TN "\MyOrg\Backup" /F
    exit /b %errorlevel%
)
if /i "%~1"=="--run" goto :run
echo Usage: %~nx0 [--install ^| --uninstall ^| --run]
exit /b 1

:run
rem ... the actual work ...
```

## The "works for me, fails scheduled" checklist

A scheduled task runs in a **very different context** than your interactive session:

1. **Current directory is `C:\Windows\System32`**, not the script's folder. → Use `%~dp0` for every path, or `pushd "%~dp0"` at the top.
2. **No mapped network drives.** `Z:\` doesn't exist for the SYSTEM account or a non-logged-on user. → Use full UNC paths (`\\server\share\...`), or `net use` / `pushd \\server\share` inside the script.
3. **A leaner `PATH` and different env vars.** Don't assume `git`, `python`, `node` are on `PATH` — call them by full path or set `PATH` explicitly.
4. **No console / no interactivity.** `pause`, `timeout` (without `/nobreak`... actually even then it can error), `set /p`, `choice` may hang or fail. Gate them behind an `--interactive` flag.
5. **`%TEMP%` / `%APPDATA%` point elsewhere** (SYSTEM profile). Write logs to an explicit path.
6. **Different user = different permissions.** SYSTEM can't reach a user's OneDrive; a service account may lack "log on as batch job."
7. **UAC / elevation.** A task with `/RL HIGHEST` runs elevated; your interactive test may not have been (or vice versa).

### Defensive header

```bat
@echo off
setlocal EnableExtensions
pushd "%~dp0" || (echo cannot cd to script dir & exit /b 1)
set "LOG=%~dp0logs\run_%RANDOM%.log"
if not exist "%~dp0logs" md "%~dp0logs"
call :main >> "%LOG%" 2>&1
set "rc=%errorlevel%"
popd
exit /b %rc%
```

## `start` — launch and (optionally) don't wait

```bat
start "" "notepad.exe"                    &rem launch, don't wait (note the empty "" title!)
start "" /wait "installer.exe" /silent     &rem wait for it to finish
start "Build" /min cmd /c "build.bat"       &rem minimized, titled window
start "" /b longtask.exe                    &rem no new window (background in same console)
start "" "https://example.com"              &rem open a URL / document with its default app
```

!!! warning "The empty `\"\"` first argument"
    `start` treats the **first quoted argument as the window title**. `start "C:\My App\app.exe"` tries to run a window titled `C:\My App\app.exe` and fails. Always: `start "" "C:\My App\app.exe"`.

`start /wait` is how you run installers sequentially and check each result:

```bat
start "" /wait msiexec /i app.msi /qn
if errorlevel 1 ( echo install failed & exit /b 1 )
```

## Elevation

Batch can't elevate itself directly. Options:

### Detect whether you're elevated

```bat
net session >nul 2>&1
if errorlevel 1 (
    echo This script needs to run as Administrator.
    exit /b 1
)
```

(`fltmc` or `openfiles` also work as admin probes.)

### Self-elevate via PowerShell

```bat
net session >nul 2>&1 || (
    powershell -NoProfile -Command "Start-Process -Verb RunAs -FilePath '%~f0' -ArgumentList '%*'"
    exit /b
)
echo Now running elevated.
```

This pops a UAC prompt and relaunches the script elevated in a new window.

### `runas` (different user, prompts for password, no `/savecred` for admin)

```bat
runas /user:DOMAIN\admin "cmd /c \"%~f0\" %*"
```

`runas` can't pass a password on the command line (by design) and can't be used non-interactively for elevation — that's what `schtasks /RU` or a service is for.

## `cmd /c` vs `cmd /k`

```bat
cmd /c script.bat        &rem run it, then the new cmd exits
cmd /k script.bat        &rem run it, then LEAVE the window open (great for debugging — see Advanced 9)
```

## Common mistakes

- **Relative paths in a scheduled script** — cwd is `System32`. Use `%~dp0` / `pushd "%~dp0"`.
- **Relying on mapped drives** in a task — use UNC.
- **`start "C:\path with space\app.exe"`** — the path becomes the title. Add `start "" ...`.
- **`schtasks /TR` without quoting/escaping** — path with spaces breaks; inner quotes need `\"`.
- **`pause`/`set /p` in a scheduled run** — hangs the task until it times out.
- **Assuming `git`/`python` on `PATH`** in the task's environment.
- **Not capturing output** — a scheduled failure with no log is un-debuggable. Redirect `>> log 2>&1`.
- **`/RL HIGHEST` but the run-as account isn't admin** — task runs un-elevated silently.
- **Testing elevated, deploying un-elevated** (or vice versa) — mismatched permissions.

## Exercises

1. Make `hello.bat` write `%date% %time% ran as %username% in %cd%` to a fixed log path. Schedule it every 5 minutes with `schtasks /SC MINUTE /MO 5`. Inspect the log — note the `%cd%`.
2. Add `--install` / `--uninstall` / `--run` handling to a script so it can register and remove its own scheduled task.
3. Write an "am I elevated?" check and have the script self-elevate via the PowerShell `RunAs` trick if not.
4. Use `start "" /wait` to run two commands in sequence and stop if the first fails.
5. Query your new task's "Last Result" and translate `0` / non-zero into a friendly message.
6. Break it on purpose: write a script that uses a relative path `.\data\in.txt`, schedule it, and confirm it fails; then fix it with `%~dp0`.

## Recap

- `schtasks /Create /TN ... /TR "\"%~f0\" args" /SC DAILY /ST 02:00 /RL HIGHEST /F` — quote the path, escape inner quotes.
- `/Query /V /FO LIST`, `/Run`, `/Delete /F` round out the lifecycle; "Last Result" is your script's exit code.
- Scheduled context ≠ interactive: cwd is `System32`, no mapped drives, lean `PATH`, no console, different profile. Use `%~dp0`, UNC paths, full tool paths, and log everything.
- `start "" ...` (empty title!), `/wait` to block, `/b` for no window.
- Batch can't self-elevate; detect with `net session`, relaunch via PowerShell `Start-Process -Verb RunAs`.

Next: [Capstone: backup utility →](10-capstone-backup-utility.md)
