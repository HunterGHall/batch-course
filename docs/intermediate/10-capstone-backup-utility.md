# 10. Capstone: Backup Utility

## The goal

Build `backup.bat`, a production-shaped backup script:

```bat
backup.bat --source "C:\work\project" --dest "D:\backups\project" --keep 7
```

It mirrors `--source` into a timestamped folder under `--dest`, keeps the newest `--keep` backups and deletes older ones, logs everything, and returns a meaningful exit code. It must survive being run from a scheduled task.

## Requirements

1. **Named arguments**, any order: `--source PATH` (required), `--dest PATH` (required), `--keep N` (default 7), `--dry-run`, `--quiet`.
2. Validate: source must exist; dest parent must exist or be creatable; `--keep` must be a positive integer.
3. Each run creates `--dest\backup_YYYY-MM-DD_HH-MM-SS\` and mirrors the source into it with `robocopy`.
4. **Retention:** after a successful backup, if there are more than `--keep` backup folders, delete the oldest ones.
5. **Logging:** append to `--dest\backup.log` — start time, parameters, robocopy summary line, files copied, result, duration.
6. **Exit codes:** `0` success, `1` bad arguments, `2` source/dest problem, `3` backup (robocopy) failed, `4` retention cleanup failed.
7. Works from a scheduled task: absolute paths only, no interactive prompts unless a `--yes` is *absent* AND a console is present... (keep it simple: `--dry-run` is the safety, no prompts at all).
8. `--dry-run` prints the plan (timestamp folder name, robocopy command, which old folders would be pruned) and changes nothing.

## Design notes

- Get the timestamp locale-independently ([Intermediate Lesson 7](07-date-time-and-math.md)) — PowerShell `Get-Date -Format` is cleanest.
- `robocopy /MIR` into a *fresh empty folder* each time = a full copy. (A smarter version would hardlink unchanged files against the previous backup; that's an extension.)
- Classify `robocopy` by `errorlevel >= 8` ([Beginner Lesson 7](../beginner/07-files-and-folders.md)).
- List backup folders newest-first with `dir /b /a:d /o:-d`; skip the first `--keep`, delete the rest.
- Duration: capture start and end tick counts. Simplest portable "seconds" source is again PowerShell, or accept coarse timing.

## Reference implementation

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion

rem ---------- defaults ----------
set "SRC="
set "DST="
set "KEEP=7"
set "DRYRUN="
set "QUIET="

rem ---------- parse named args ----------
:parse
if "%~1"=="" goto parsed
if /i "%~1"=="--source"  ( set "SRC=%~2"  & shift & shift & goto parse )
if /i "%~1"=="--dest"    ( set "DST=%~2"  & shift & shift & goto parse )
if /i "%~1"=="--keep"    ( set "KEEP=%~2" & shift & shift & goto parse )
if /i "%~1"=="--dry-run" ( set "DRYRUN=1" & shift & goto parse )
if /i "%~1"=="--quiet"   ( set "QUIET=1"  & shift & goto parse )
echo ERROR: unknown option "%~1" 1>&2
goto usage
:parsed

rem ---------- validate ----------
if not defined SRC goto usage
if not defined DST goto usage

set /a _k=KEEP 2>nul
if not "%_k%"=="%KEEP%" ( echo ERROR: --keep must be an integer 1>&2 & exit /b 1 )
if %KEEP% lss 1 ( echo ERROR: --keep must be ^>= 1 1>&2 & exit /b 1 )

if not exist "%SRC%\" ( echo ERROR: source not found: "%SRC%" 1>&2 & exit /b 2 )
if not exist "%DST%\" (
    if defined DRYRUN ( echo [dry-run] would create dest "%DST%" ) else (
        md "%DST%" 2>nul || ( echo ERROR: cannot create dest: "%DST%" 1>&2 & exit /b 2 )
    )
)

rem ---------- timestamp ----------
for /F "usebackq delims=" %%t in (`powershell -NoProfile -Command "Get-Date -Format yyyy-MM-dd_HH-mm-ss"`) do set "TS=%%t"
if not defined TS ( echo ERROR: could not get timestamp 1>&2 & exit /b 2 )

set "TARGET=%DST%\backup_%TS%"
set "LOG=%DST%\backup.log"

call :log "=== backup start ==="
call :log "source=%SRC%  dest=%DST%  keep=%KEEP%  dryrun=%DRYRUN%"

rem ---------- do the backup ----------
if defined DRYRUN (
    call :say "[dry-run] would mirror:"
    call :say "  robocopy \"%SRC%\" \"%TARGET%\" /MIR /R:2 /W:5 /NFL /NDL /NP"
) else (
    md "%TARGET%" 2>nul
    robocopy "%SRC%" "%TARGET%" /MIR /R:2 /W:5 /NFL /NDL /NP /NJH /BYTES >>"%LOG%" 2>&1
    set "RC=!errorlevel!"
    if !RC! geq 8 (
        call :log "robocopy FAILED with code !RC!"
        call :say "Backup failed (robocopy !RC!). See %LOG%."
        exit /b 3
    )
    call :log "robocopy ok (code !RC!)"
)

rem ---------- retention ----------
set /a idx=0, pruned=0, prunefail=0
for /F "delims=" %%D in ('dir /b /a:d /o:-d "%DST%\backup_*" 2^>nul') do (
    set /a idx+=1
    if !idx! gtr %KEEP% (
        if defined DRYRUN (
            call :say "[dry-run] would prune old backup: %%D"
        ) else (
            rd /s /q "%DST%\%%D"
            if exist "%DST%\%%D\" (
                call :log "prune FAILED: %%D"
                set /a prunefail+=1
            ) else (
                call :log "pruned: %%D"
                set /a pruned+=1
            )
        )
    )
)

call :log "pruned=!pruned! prunefail=!prunefail!"
call :say "Backup complete: %TARGET%   (pruned !pruned! old, !prunefail! failed)"
call :log "=== backup end ==="

if !prunefail! gtr 0 exit /b 4
exit /b 0

rem ===================================================================
:usage
echo Usage: %~nx0 --source PATH --dest PATH [--keep N] [--dry-run] [--quiet]
exit /b 1

:log
    >>"%LOG%" echo [%date% %time%] %~1
    goto :eof

:say
    if not defined QUIET echo %~1
    goto :eof
```

## Walkthrough of the tricky parts

| Bit | Why |
| --- | --- |
| `set /a _k=KEEP` then compare to `KEEP` | integer validation: if `--keep abc`, `_k` becomes `0` ≠ `abc` |
| `powershell ... Get-Date -Format` | locale-proof timestamp; the whole retention scheme depends on the folder names sorting chronologically, which `yyyy-MM-dd_HH-mm-ss` guarantees |
| `dir /b /a:d /o:-d "backup_*"` | directories only, newest first — so "index > KEEP" = "too old" |
| `2^>nul` inside `for /F ('...')` | caret-escaped redirect; hides the error when no backups exist yet |
| `!errorlevel!` after robocopy | inside... actually it's not in a block here, but delayed is safe; `%errorlevel%` would also work at that point |
| `>>"%LOG%" echo ...` (redirect first) | avoids a trailing space before `>>` sneaking into the log line |
| separate `exit /b` codes | a scheduled task's "Last Result" tells you *which* stage broke |

## Test plan

1. **Happy path:** small source, empty dest, `--keep 3`. Run 4 times (put a `timeout /t 2` or change a file between runs so timestamps differ). Confirm exactly 3 folders remain and the oldest was pruned. Check `backup.log`.
2. **Dry run:** `--dry-run` on a dest that already has 5 backups — confirm it names the folder it *would* create and the 2 it *would* prune, and touches nothing.
3. **Bad args:** missing `--source`; `--keep 0`; `--keep xyz`; unknown `--frobnicate`. Each should give the right message and exit code (`1`).
4. **Bad source:** `--source C:\does\not\exist` → exit `2`.
5. **Backup failure:** point `--dest` at a read-only location (or a path on a full disk) → exit `3`, logged.
6. **Retention failure:** open a file inside the oldest backup folder so `rd /s /q` can't remove it → exit `4`, `prunefail` logged.
7. **Scheduled:** register it with `schtasks` to run every few minutes ([Intermediate Lesson 9](09-scheduling-and-running.md)). Confirm it works with cwd = `System32` (it should — every path is absolute) and that `backup.log` fills up.

## Extensions (do at least two)

1. **Incremental via hardlinks:** `robocopy ... /MIR` won't help; use `/CREATE` or drive it so unchanged files are hardlinked from the previous backup (saves huge space). Or shell out to `robocopy` with `/DCOPY:DAT` + a compare step.
2. **Compress old backups:** after pruning, zip backups older than 2 but within `--keep` using `tar -a -c -f` (bundled `bsdtar` on modern Windows) or PowerShell `Compress-Archive`.
3. **Config file:** default `--source`/`--dest`/`--keep` from `backup.ini` next to the script; command-line overrides it.
4. **Email/Teams notification** on failure via a PowerShell one-liner.
5. **Lock file** so two overlapping scheduled runs don't collide.
6. **`--verify`:** after the copy, `robocopy /L` the source against the new backup and fail if there are differences.

## What you should be able to do now

- Explain each exit code and which failure triggers it.
- Add a new named option without breaking the parser.
- State exactly why the timestamp format matters for retention.
- Run it from a scheduled task with confidence.

If so — on to [Advanced](../advanced/index.md).
