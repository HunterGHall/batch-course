# 8. Packaging & Distribution

## What & why

Once a script works on your machine, getting it to run reliably on *other* people's machines is its own problem: paths, dependencies, configuration, trust prompts, and updates. This lesson is about shipping batch scripts like software.

## Make it location-independent

Never assume the current directory. Anchor everything to the script:

```bat
@echo off
setlocal EnableExtensions
set "HERE=%~dp0"
set "HERE=%HERE:~0,-1%"          &rem strip trailing backslash for clean joins

set "CONFIG=%HERE%\config.ini"
set "TOOLS=%HERE%\tools"
set "LOGDIR=%HERE%\logs"
```

- `%~dp0` = script drive+path, always ends with `\`.
- `%~f0` = full path to the script itself.
- `%~nx0` = the script's own filename (for usage messages: `echo Usage: %~nx0 ...`).

If the script *must* run from its own directory (a tool that shells out to relative paths):

```bat
pushd "%~dp0" || ( echo Cannot access script directory. & exit /b 1 )
rem ... work ...
popd
```

## Configuration: don't hardcode, don't over-engineer

Ship a `config.ini` next to the script; read it with `for /F` ([Intermediate Lesson 3](../intermediate/03-for-f-deep-dive.md)):

```ini
; config.ini
source   = C:\work\project
dest     = D:\backups
keep     = 7
verbose  = 0
```

```bat
rem load config, skipping comments and blanks, trimming spaces around =
for /F "usebackq tokens=1,* delims== " %%k in ("%CONFIG%") do (
    set "cfg.%%k=%%l"
)
rem cfg.source, cfg.dest, cfg.keep now set
```

Order of precedence, high to low: **command-line arg > environment variable > config file > built-in default**. Implement that explicitly.

## Dependencies

Batch itself has none, but your script may call `git`, `7z`, `curl`, `robocopy`, PowerShell. At startup, check what you need and fail clearly:

```bat
call :need robocopy || goto :missingdeps
call :need powershell || goto :missingdeps
where /q git || ( echo [warn] git not found - some features disabled )
goto :depsok

:need
    where /q "%~1" && exit /b 0
    echo [error] required tool not found on PATH: %~1 1>&2
    exit /b 1

:missingdeps
    echo.
    echo Install the missing tools and try again.
    exit /b 1
:depsok
```

If you bundle a tool (e.g. a portable `7za.exe` in `.\tools\`), prepend that folder to `PATH` for the script's lifetime:

```bat
set "PATH=%HERE%\tools;%PATH%"
```

(`setlocal` ensures this doesn't leak.)

## Single-file distribution

- **Just the `.bat`** — simplest. Email it, drop it in a shared folder, commit it.
- **`.bat` + `config.ini` + `tools\`** in a zip — the normal case.
- **Self-extracting**: `iexpress.exe` (built into Windows) can bundle files + a batch script into one `.exe`. Niche, and the `.exe` triggers more security scrutiny.
- **Hybrid** ([Lesson 7](07-hybrid-scripts.md)) to fold PowerShell logic into the one file.
- **Embed a payload in the script**: append a base64 blob after `:eof` / a marker, decode it at runtime with `certutil -decode`. Clever, fragile, and antivirus hates it. Avoid unless you have a real reason.

## Versioning

Put a version in the script and expose it:

```bat
set "VERSION=1.4.2"
if /i "%~1"=="--version" ( echo %~nx0 %VERSION% & exit /b 0 )
```

Keep a `CHANGELOG.md` alongside. Tag releases in your VCS. If the script self-updates (below), the version check is how it knows whether to.

## Self-update (optional)

```bat
:update
for /F "usebackq delims=" %%v in (`powershell -NoProfile -Command "(Invoke-WebRequest 'https://example.com/mytool/VERSION').Content.Trim()"`) do set "LATEST=%%v"
if "%LATEST%"=="%VERSION%" ( echo Up to date (%VERSION%). & exit /b 0 )
echo Updating %VERSION% -> %LATEST%
powershell -NoProfile -Command "Invoke-WebRequest 'https://example.com/mytool/mytool.bat' -OutFile '%~f0.new'"
rem swap on next run, since you can't overwrite a running .bat cleanly mid-execution:
move /y "%~f0.new" "%~f0" >nul
echo Updated. Re-run the command.
exit /b 0
```

Caveats: you generally **can't overwrite a `.bat` while it's executing** (the interpreter has it open and reads it line-by-line — replacing it mid-run causes chaos). Download to `.new`, then `move` it into place at the very end or on the next launch. Verify a hash/signature before trusting a downloaded update.

## Trust: SmartScreen, antivirus, execution policy

- A `.bat` downloaded from the internet gets the **Mark of the Web** (Zone.Identifier alternate data stream). Windows may warn. Users can unblock via file Properties → Unblock, or `powershell -Command "Unblock-File .\mytool.bat"`.
- **Code signing**: `.bat` files **cannot be Authenticode-signed** (no embedded signature format). Only `.ps1`, `.exe`, `.msi`, `.dll`, etc. can. If signing matters, ship the logic as a signed `.ps1` and a thin `.bat` launcher, or an `.exe`.
- Antivirus heuristics flag: base64 blobs, `certutil -decode`, `powershell -enc`, `reg add` to Run keys, obfuscated variable soup. Keep scripts readable and boring.
- Don't `powershell -ExecutionPolicy Bypass` casually in docs you give users — explain why. For `-File`/`-Command` it's normal and doesn't need admin.

## Documentation to ship with it

- A `--help` that's actually complete (options, exit codes, examples).
- A `README.md`: what it does, requirements, install, config, examples, exit-code table.
- Exit codes documented — callers and schedulers depend on them.
- The `CHANGELOG.md`.

## Cross-Windows compatibility

- Test on the **oldest** Windows you support. `choice`, `where`, `forfiles`, `robocopy`, `timeout` are all present on Win7+; `wmic` is *gone* on Win11 24H2+ ([Lesson 4](04-wmic-where-powershell.md)).
- `powershell` (5.1) is on everything Win7 SP1+; `pwsh` (7+) is not installed by default anywhere — don't assume it.
- Windows PowerShell default execution policy differs (client vs server). Use `-ExecutionPolicy Bypass` on your own invocations.
- ARM64 Windows runs x64 batch/PowerShell fine, but bundled native `.exe`s need the right arch.

## Common mistakes

- **Relative paths** — breaks the moment someone runs it from elsewhere or schedules it.
- **Hardcoded `C:\Users\you\...`** — use `%HERE%`, `%USERPROFILE%`, `%APPDATA%`, `%ProgramData%`.
- **No dependency check** — cryptic failure 40 lines in instead of a clear "install X" at the top.
- **Trying to code-sign a `.bat`** — not possible; sign a `.ps1`/`.exe` instead.
- **Self-update overwriting the running script** — download to `.new`, swap later.
- **Base64/`certutil` payload tricks** — antivirus false positives, unmaintainable.
- **Assuming `wmic`/`pwsh`** — the former is being removed, the latter isn't installed.
- **`setx PATH ...`** to "install" — `setx` truncates at 1024 chars and can corrupt `PATH`. Be very careful; prefer per-script `set "PATH=..."`.
- **No exit-code documentation** — schedulers and CI can't react to failures they can't interpret.

## Exercises

1. Refactor a script that uses relative paths and one hardcoded `C:\Users\...` path to be fully location- and user-independent.
2. Add a dependency-check phase that verifies `robocopy` and `powershell`, warns about optional `git`, and lists everything missing before exiting.
3. Implement config precedence: `--keep` arg beats `MYTOOL_KEEP` env var beats `config.ini` `keep=` beats default `7`. Prove all four paths with tests.
4. Bundle a portable tool in `.\tools\` and prepend it to `PATH` for the script only; confirm `PATH` is unchanged in the parent shell afterward.
5. Add `--version` and a `--check-update` that compares against a local `LATEST` file (simulate the remote), downloading to `.new` without touching the running script.
6. Write the `README.md` and exit-code table for the intermediate backup capstone as if you were shipping it to a team.
7. Download your own script via `Invoke-WebRequest`, observe the Mark of the Web, and unblock it with `Unblock-File`.

## Recap

- Anchor every path to `%~dp0` / `%~f0`; `pushd "%~dp0"` if the script needs its own cwd.
- Config in a sibling `.ini`, read with `for /F`; precedence: arg > env > file > default.
- Check dependencies up front with `where /q` and fail with a clear message.
- Ship: bare `.bat`, or `.bat`+`config`+`tools\` zip; avoid `certutil` base64 payloads.
- `.bat` can't be code-signed — put signed logic in a `.ps1`/`.exe` with a thin `.bat` launcher.
- Self-update downloads to `.new` and swaps later (can't overwrite a running `.bat`).
- Test on your oldest supported Windows; `wmic` is disappearing, `pwsh` isn't preinstalled.
- Document `--help`, a `README`, and the exit codes.

Next: [Debugging techniques →](09-debugging-techniques.md)
