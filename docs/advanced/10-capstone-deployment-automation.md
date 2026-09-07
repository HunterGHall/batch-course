# 10. Capstone: Deployment Automation

## The goal

Build `deploy.bat` — a production-grade script that deploys a web app to a Windows server:

```bat
deploy.bat --env staging --ref v1.4.2
deploy.bat --env production --ref v1.4.2 --no-migrate
deploy.bat --env staging --rollback
```

It pulls a build, stops the service, swaps in the new release (keeping the last N for rollback), runs migrations, restarts, health-checks, and rolls back automatically if the health check fails. Everything is logged; the exit code says exactly what happened.

This pulls together every advanced lesson: recursion-free tree work, real argument parsing, the registry (or `sc`), PowerShell interop, `findstr`, performance awareness, packaging, and debugging hooks.

## Requirements

### CLI ([Lesson 2](02-robust-argument-parsing.md))

| Option | Meaning |
| --- | --- |
| `--env NAME` | required: `staging` or `production` |
| `--ref REF` | git ref / build tag to deploy (required unless `--rollback`) |
| `--rollback` | redeploy the previous release, skip build |
| `--no-migrate` | skip the DB migration step |
| `--keep N` | releases to retain for rollback (default 5) |
| `--dry-run` | print the plan, change nothing |
| `--yes` | skip the production confirmation prompt (for CI) |
| `--debug` | verbose trace to stderr |
| `-h` / `--help` | usage |
| `--version` | print version |

### Behaviour

1. **Validate**: env is one of the two; `--ref` present unless `--rollback`; `--keep` is a positive int; required tools present (`git`, `powershell`, `sc`); config file for the env exists.
2. **Config** ([Lesson 8](08-packaging-and-distribution.md)): per-env settings from `deploy.<env>.ini` next to the script — `service_name`, `deploy_root`, `health_url`, `repo_url`, `migrate_cmd`. Precedence: CLI > env var > ini > default.
3. **Confirm**: if `--env production` and not `--yes`, require the operator to type `production` ([Intermediate Lesson 8](../intermediate/08-user-interaction-and-menus.md)).
4. **Acquire build** (unless `--rollback`): `git` clone/fetch into `deploy_root\releases\<timestamp>_<ref>` ([Intermediate Lesson 7](../intermediate/07-date-time-and-math.md) for the timestamp).
5. **Stop service**: `sc stop <service>` and wait until `STATE : ... STOPPED` (poll `sc query`, parse with `findstr` — [Lesson 5](05-text-processing-findstr.md)).
6. **Swap**: repoint the `deploy_root\current` junction (`mklink /J`) to the new release. Save the previous target for rollback.
7. **Migrate** (unless `--no-migrate`): run `migrate_cmd` in `current`; on failure → rollback.
8. **Start service**: `sc start <service>`, wait for `RUNNING`.
9. **Health check**: hit `health_url` with PowerShell `Invoke-WebRequest`, expect HTTP 200 and a body containing `"ok"`. Retry a few times with backoff.
10. **On any failure after the swap**: automatically roll back (repoint `current`, restart service, health-check again), then exit non-zero.
11. **Retention**: keep the newest `--keep` releases; delete older ones (never the one `current` points at).
12. **Logging**: append to `deploy_root\deploy.log` — every step, timestamps, the operator (`%username%`), the outcome. Also tee to console unless `--quiet`.

### Exit codes

| Code | Meaning |
| --- | --- |
| 0 | deployed (or rolled back cleanly on request) and healthy |
| 1 | bad arguments / usage |
| 2 | validation failed (env, missing tool, missing config) |
| 3 | operator aborted at the confirmation prompt |
| 4 | build/fetch failed |
| 5 | service stop/start failed |
| 6 | migration failed → rolled back OK |
| 7 | health check failed → rolled back OK |
| 8 | **failure AND rollback also failed** — server may be down, page someone |

## Skeleton

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion
set "VERSION=1.0.0"
set "HERE=%~dp0" & set "HERE=%HERE:~0,-1%"

rem ---------------- args ----------------
set "ENV=" & set "REF=" & set "ROLLBACK=" & set "MIGRATE=1"
set "KEEP=5" & set "DRYRUN=" & set "ASSUMEYES=" & set "DEBUG=" & set "QUIET="

:args
if "%~1"=="" goto args_done
if /i "%~1"=="-h"          goto usage
if /i "%~1"=="--help"      goto usage
if /i "%~1"=="--version"   ( echo %~nx0 %VERSION% & exit /b 0 )
if /i "%~1"=="--env"       ( set "ENV=%~2" & shift & shift & goto args )
if /i "%~1"=="--ref"       ( set "REF=%~2" & shift & shift & goto args )
if /i "%~1"=="--keep"      ( set "KEEP=%~2" & shift & shift & goto args )
if /i "%~1"=="--rollback"  ( set "ROLLBACK=1" & shift & goto args )
if /i "%~1"=="--no-migrate"( set "MIGRATE=" & shift & goto args )
if /i "%~1"=="--dry-run"   ( set "DRYRUN=1" & shift & goto args )
if /i "%~1"=="--yes"       ( set "ASSUMEYES=1" & shift & goto args )
if /i "%~1"=="--debug"     ( set "DEBUG=1" & shift & goto args )
if /i "%~1"=="--quiet"     ( set "QUIET=1" & shift & goto args )
echo ERROR: unknown option "%~1" 1>&2 & exit /b 1
:args_done

rem ---------------- validate ----------------
if /i not "%ENV%"=="staging" if /i not "%ENV%"=="production" (
    echo ERROR: --env must be staging or production 1>&2 & exit /b 2
)
if not defined ROLLBACK if not defined REF (
    echo ERROR: --ref required unless --rollback 1>&2 & exit /b 1
)
set /a _k=KEEP 2>nul
if not "%_k%"=="%KEEP%" ( echo ERROR: --keep must be an integer 1>&2 & exit /b 1 )

call :need git        || exit /b 2
call :need powershell || exit /b 2
call :need sc         || exit /b 2

set "CFG=%HERE%\deploy.%ENV%.ini"
if not exist "%CFG%" ( echo ERROR: missing config %CFG% 1>&2 & exit /b 2 )
for /F "usebackq tokens=1,* delims== " %%k in ("%CFG%") do set "cfg.%%k=%%l"

set "SVC=%cfg.service_name%"
set "ROOT=%cfg.deploy_root%"
set "HEALTH=%cfg.health_url%"
set "LOG=%ROOT%\deploy.log"

rem ---------------- confirm ----------------
if /i "%ENV%"=="production" if not defined ASSUMEYES (
    set "typed="
    set /p "typed=Type 'production' to deploy %REF% to PRODUCTION: "
    if /i not "!typed!"=="production" ( echo Aborted. & exit /b 3 )
)

call :log "=== deploy start: env=%ENV% ref=%REF% rollback=%ROLLBACK% by %username% ==="

if defined DRYRUN ( call :plan & exit /b 0 )

rem ---------------- pipeline ----------------
if not defined ROLLBACK ( call :acquire || exit /b 4 )
call :resolve_release            || exit /b 4
call :stop_service               || exit /b 5
call :swap_current               || ( call :emergency_rollback & exit /b 5 )
if defined MIGRATE ( call :migrate || ( call :rollback & exit /b 6 ) )
call :start_service              || ( call :rollback & exit /b 5 )
call :health_check               || ( call :rollback & exit /b 7 )
call :retention

call :log "=== deploy OK: %ENV% now on %NEW_RELEASE% ==="
call :say "Deployed %REF% to %ENV%. Healthy."
exit /b 0

rem ================= subroutines =================
:need
    where /q "%~1" && exit /b 0
    echo ERROR: required tool not found: %~1 1>&2
    exit /b 1

:dbg
    if defined DEBUG echo [%time%] [DEBUG] %~1 1>&2
    goto :eof

:log
    >>"%LOG%" echo [%date% %time%] %~1
    goto :eof

:say
    if not defined QUIET echo %~1
    call :log "%~1"
    goto :eof

:plan
    call :say "[dry-run] would deploy ref '%REF%' to '%ENV%' (service %SVC%, root %ROOT%)"
    call :say "[dry-run] would keep newest %KEEP% releases, health-check %HEALTH%"
    goto :eof

:acquire
    for /F "usebackq delims=" %%t in (`powershell -NoProfile -Command "Get-Date -Format yyyy-MM-dd_HH-mm-ss"`) do set "TS=%%t"
    set "NEW_RELEASE=%ROOT%\releases\%TS%_%REF%"
    call :dbg "cloning %cfg.repo_url% @ %REF% -> %NEW_RELEASE%"
    git clone --depth 1 --branch "%REF%" "%cfg.repo_url%" "%NEW_RELEASE%" >>"%LOG%" 2>&1
    if errorlevel 1 ( call :log "acquire FAILED" & exit /b 1 )
    goto :eof

:resolve_release
    rem for --rollback: NEW_RELEASE = the release before whatever current points at
    if not defined ROLLBACK goto :eof
    call :read_current_target PREV_TARGET
    set "NEW_RELEASE="
    for /F "delims=" %%d in ('dir /b /a:d /o:-d "%ROOT%\releases" 2^>nul') do (
        if defined SEEN_CURRENT if not defined NEW_RELEASE set "NEW_RELEASE=%ROOT%\releases\%%d"
        if "%ROOT%\releases\%%d"=="!PREV_TARGET!" set "SEEN_CURRENT=1"
    )
    if not defined NEW_RELEASE ( call :log "no previous release to roll back to" & exit /b 1 )
    goto :eof

:stop_service
    call :say "Stopping %SVC% ..."
    sc stop "%SVC%" >>"%LOG%" 2>&1
    call :wait_state "%SVC%" STOPPED 30 || ( call :log "stop timed out" & exit /b 1 )
    goto :eof

:start_service
    call :say "Starting %SVC% ..."
    sc start "%SVC%" >>"%LOG%" 2>&1
    call :wait_state "%SVC%" RUNNING 30 || ( call :log "start timed out" & exit /b 1 )
    goto :eof

:wait_state
    rem %1 service, %2 desired state word, %3 max seconds
    set /a _left=%~3
    :ws_loop
    for /F "tokens=3" %%s in ('sc query "%~1" ^| findstr /r /c:"STATE"') do set "_st=%%s"
    call :dbg "%~1 state=!_st! want=%~2 left=!_left!"
    if /i "!_st!"=="%~2" goto :eof
    set /a _left-=1
    if !_left! leq 0 exit /b 1
    >nul ping -n 2 127.0.0.1
    goto ws_loop

:read_current_target
    rem returns the junction target of %ROOT%\current in the named var
    for /F "tokens=2 delims=[]" %%p in ('dir "%ROOT%" ^| findstr /i /c:"<JUNCTION>    current"') do set "%~1=%%p"
    goto :eof

:swap_current
    call :read_current_target OLD_TARGET
    call :dbg "old current -> !OLD_TARGET!"
    if exist "%ROOT%\current" rd "%ROOT%\current"
    mklink /J "%ROOT%\current" "%NEW_RELEASE%" >>"%LOG%" 2>&1
    if errorlevel 1 exit /b 1
    call :log "current -> %NEW_RELEASE% (was !OLD_TARGET!)"
    goto :eof

:rollback
    call :say "ROLLING BACK ..."
    if not defined OLD_TARGET ( call :log "rollback: no OLD_TARGET known" & exit /b 1 )
    if exist "%ROOT%\current" rd "%ROOT%\current"
    mklink /J "%ROOT%\current" "!OLD_TARGET!" >>"%LOG%" 2>&1 || ( call :log "rollback link FAILED" & exit /b 8 )
    sc start "%SVC%" >>"%LOG%" 2>&1
    call :wait_state "%SVC%" RUNNING 30 || ( call :log "rollback start FAILED" & exit /b 8 )
    call :health_check || ( call :log "rollback health FAILED" & exit /b 8 )
    call :log "rollback OK -> !OLD_TARGET!"
    goto :eof

:emergency_rollback
    call :rollback
    goto :eof

:migrate
    call :say "Running migrations ..."
    pushd "%ROOT%\current"
    call %cfg.migrate_cmd% >>"%LOG%" 2>&1
    set "_rc=!errorlevel!"
    popd
    if !_rc! neq 0 ( call :log "migrate FAILED (!_rc!)" & exit /b 1 )
    goto :eof

:health_check
    set /a _try=0
    :hc
    set /a _try+=1
    for /F "usebackq delims=" %%c in (`powershell -NoProfile -Command ^
      "try { $r=Invoke-WebRequest -UseBasicParsing -TimeoutSec 10 '%HEALTH%'; if ($r.StatusCode -eq 200 -and $r.Content -match 'ok') {0} else {1} } catch {1}"`) do set "_hc=%%c"
    call :dbg "health try !_try! -> !_hc!"
    if "!_hc!"=="0" ( call :say "Health check passed." & goto :eof )
    if !_try! geq 5 ( call :log "health check FAILED after !_try! tries" & exit /b 1 )
    >nul ping -n 6 127.0.0.1
    goto hc

:retention
    set /a _i=0
    for /F "delims=" %%d in ('dir /b /a:d /o:-d "%ROOT%\releases" 2^>nul') do (
        set /a _i+=1
        if !_i! gtr %KEEP% (
            if /i not "%ROOT%\releases\%%d"=="%NEW_RELEASE%" (
                rd /s /q "%ROOT%\releases\%%d" && call :log "pruned release %%d"
            )
        )
    )
    goto :eof

:usage
echo Usage: %~nx0 --env staging^|production --ref REF [--rollback] [--no-migrate]
echo             [--keep N] [--dry-run] [--yes] [--debug] [--quiet]
exit /b 1
```

> This is deliberately near the ceiling of what batch should do. Read [Lesson 6](06-performance-and-pitfalls.md) again: a real team would likely write this in PowerShell. The exercise is to feel *where* batch fights you — the `sc query` parsing, the junction target extraction, the nested `call ... || ( call :rollback & exit /b N )` error plumbing — and to be able to justify the rewrite.

## Test plan

Set up a fake environment: a dummy Windows service (`sc create TestSvc binPath= "cmd /c ping -t localhost"`), a local git repo as `repo_url`, a tiny static server for `health_url` (`powershell -Command "..."` or a real one), and a `deploy_root` on a scratch disk.

1. **Dry run** — `--env staging --ref main --dry-run`: plan printed, nothing touched.
2. **First deploy** — `--env staging --ref v1`: clone, stop, swap, migrate, start, health OK, `current` junction points at the new release, log complete, exit `0`.
3. **Second deploy** — `--ref v2`: same, and `OLD_TARGET` recorded.
4. **Rollback command** — `--env staging --rollback`: `current` back to v1, service healthy, exit `0`.
5. **Migration failure** — point `migrate_cmd` at a command that exits 1: script rolls back automatically, exit `6`, log shows the rollback succeeded.
6. **Health failure** — make `health_url` return 500: auto-rollback, exit `7`.
7. **Double failure** — health fails *and* break the rollback (make `mklink` fail by locking `current`): exit `8`, log screams.
8. **Bad args** — missing `--ref`, bad `--env`, `--keep abc`: exit `1`/`2`.
9. **Production confirmation** — `--env production --ref v1` and type `nope`: exit `3`. Then `--yes` bypasses it.
10. **Retention** — deploy 7 times with `--keep 3`: exactly 3 release folders remain, and never the live one.
11. **Scheduled/CI context** — run via `schtasks` or a fresh `cmd` with a minimal `PATH`: confirm `call :need` catches missing tools instead of failing cryptically midway.

## Extensions

1. **Slack/Teams/email notification** on start, success, and failure (PowerShell webhook one-liner).
2. **Lock file** so two deploys can't overlap; stale-lock detection.
3. **Blue/green** instead of a single `current` junction — two slots, flip a load-balancer setting.
4. **`--ref` as a commit SHA** with shallow-fetch instead of branch clone.
5. **Structured JSON log** line per run (build it carefully, or — honestly — this is the point where you `powershell -File deploy-log.ps1`).
6. **Pre-flight**: disk space check on `deploy_root`, service exists, `current` junction is valid, last deploy wasn't < 60s ago.
7. **Rewrite the whole thing in PowerShell** and write a one-paragraph retro comparing the two.

## What you should be able to do now

- Trace any exit code back to the exact failing step and say whether the server is up.
- Explain why each `call ... || ( :rollback & exit /b )` is structured the way it is.
- Point at the three or four places where batch is clearly the wrong tool and say what you'd use instead.
- Add a new pipeline step (e.g. cache warm-up) between start and health check without breaking rollback.

That last skill — knowing the tool's edge and when you've reached it — is the whole point of the advanced level. You now know batch about as well as it's worth knowing.
