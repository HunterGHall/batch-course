# 4. WMIC, `where` & Calling PowerShell

## What & why

Batch can't natively query system information — running processes with full paths, disk free space, OS version, service details, installed hotfixes. Historically you used `wmic`. Microsoft is **removing `wmic`** (deprecated since 2016, disabled by default on Windows 11 24H2). The modern answer is: call PowerShell for the hard parts. This lesson covers `where`, the remains of `wmic`, and — most importantly — how to embed PowerShell one-liners cleanly.

## `where` — find executables and files

```bat
where git                       &rem prints full path(s) of git on PATH, errorlevel 1 if none
where /q node && echo have node  &rem /q = quiet, just set errorlevel
where /r C:\proj *.dll           &rem recursive file search
where python:*.exe                &rem in the "python" PATH entry
```

Standard "is this tool installed" guard:

```bat
where /q docker || ( echo docker is required 1>&2 & exit /b 1 )
```

## `wmic` — still there on many machines, but don't rely on it

```bat
wmic os get Caption,Version,BuildNumber /value
wmic logicaldisk where "DeviceID='C:'" get FreeSpace,Size /value
wmic process where "name='chrome.exe'" get ProcessId,ExecutablePath
wmic path win32_localtime get * /value
```

`/value` (or `/format:list`) gives `Name=Value` lines that `for /F "delims== tokens=1,2"` parses cleanly:

```bat
for /F "tokens=2 delims==" %%v in ('wmic os get Version /value ^| findstr "="') do set "osver=%%v"
```

!!! danger "`wmic` is going away"
    On Windows 11 24H2 and Server 2025, `wmic` is not installed by default. Any script that depends on it will fail on new machines. Treat `wmic` as legacy: fine for a quick interactive check, **not** for scripts you ship. Port to PowerShell.

## Calling PowerShell from batch

The pattern:

```bat
for /F "usebackq delims=" %%R in (`powershell -NoProfile -ExecutionPolicy Bypass -Command "EXPRESSION"`) do set "RESULT=%%R"
```

| Flag | Why |
| --- | --- |
| `-NoProfile` | skip the user's `$PROFILE` — faster, predictable |
| `-ExecutionPolicy Bypass` | run despite a restrictive machine policy (for `-Command` it often isn't needed, but harmless) |
| `-Command "..."` | the code; use `-File script.ps1` for anything long |
| `-NonInteractive` | never prompt (good for scheduled tasks) |

### Examples

```bat
rem OS version:
for /F "usebackq delims=" %%v in (`powershell -NoProfile -Command "[Environment]::OSVersion.Version.ToString()"`) do set "OSVER=%%v"

rem Free space on C: in GB:
for /F "usebackq delims=" %%g in (`powershell -NoProfile -Command "[math]::Round((Get-PSDrive C).Free/1GB,1)"`) do set "FREEGB=%%g"

rem Is a process running:
powershell -NoProfile -Command "if (Get-Process chrome -ErrorAction SilentlyContinue) { exit 0 } else { exit 1 }"
if errorlevel 1 (echo chrome not running) else (echo chrome running)

rem Timestamp (see Intermediate Lesson 7):
for /F "usebackq delims=" %%t in (`powershell -NoProfile -Command "Get-Date -Format o"`) do set "NOW=%%t"

rem Download a file:
powershell -NoProfile -Command "Invoke-WebRequest -Uri 'https://example.com/x.zip' -OutFile 'x.zip'"

rem Unzip:
powershell -NoProfile -Command "Expand-Archive -Path 'x.zip' -DestinationPath '.\out' -Force"

rem Hash a file:
for /F "usebackq tokens=1" %%h in (`powershell -NoProfile -Command "(Get-FileHash 'setup.exe' -Algorithm SHA256).Hash"`) do set "SHA=%%h"
```

## The quoting nightmare (and how to survive it)

Batch and PowerShell both use `"` and both have their own escapes. Rules that keep you sane:

1. **Wrap the whole `-Command` in double quotes** (batch's), and use **single quotes inside** for PowerShell strings:
   ```bat
   powershell -NoProfile -Command "Get-ChildItem 'C:\Program Files' | Measure-Object"
   ```
2. If you *need* a double quote inside PowerShell, escape it for `cmd` as `\"` ... actually `cmd` doesn't use `\`; you pass `""` or use PowerShell's backtick. This is where it gets ugly — **move to `-File`**:
   ```bat
   powershell -NoProfile -File "%~dp0helper.ps1" -Name "%NAME%" -Count %COUNT%
   ```
3. Pass batch variables in by **string-building the command** before calling, or via `-File ... -Args`, not by hoping expansion works inside nested quotes:
   ```bat
   set "PSCMD=(Get-Item '%TARGET%').LastWriteTime.ToString('yyyy-MM-dd')"
   for /F "usebackq delims=" %%d in (`powershell -NoProfile -Command "%PSCMD%"`) do set "MTIME=%%d"
   ```
4. `%` inside the PowerShell code: in a `.bat` file, `%` is fine inside the backtick block *unless* it looks like `%name%` (batch will try to expand it). PowerShell's `%` alias for `ForEach-Object` often trips this — write `ForEach-Object` in full, or `| %%` ... no: use `ForEach-Object`.
5. Newlines: use `;` to separate statements in a one-liner. For real multi-line logic, `-File`.

### When to just write a `.ps1`

If your PowerShell is more than ~100 characters or has any `"` inside, stop fighting: put it in `helper.ps1` next to the script and call `powershell -NoProfile -File "%~dp0helper.ps1" args`. You get proper syntax, quoting, `-ErrorAction`, and you can test it standalone.

## `powershell` vs `pwsh`

- `powershell` = Windows PowerShell 5.1, ships with Windows, always present. Slower start (~150–250 ms).
- `pwsh` = PowerShell 7+, cross-platform, **not** installed by default. Faster, better language.

For portability in a batch script that must run anywhere, use `powershell`. If you control the machines and installed `pwsh`, prefer it. Probe:

```bat
where /q pwsh && set "PS=pwsh" || set "PS=powershell"
%PS% -NoProfile -Command "..."
```

## Performance note

Each `powershell -Command` call is a fresh process — 150 ms+ on Windows PowerShell. Calling it 500 times in a loop is a 2-minute script. If you're doing that, **write the loop in PowerShell** and call it once. Batch's job then is just to be the entry point. See [Lesson 6](06-performance-and-pitfalls.md).

## Common mistakes

- **Shipping a script that uses `wmic`** — dead on new Windows. Port to PowerShell.
- **Double quotes inside `-Command "..."`** — `cmd` ends the string early. Use single quotes inside, or `-File`.
- **`%var%` inside the PowerShell block** getting eaten by batch — build the command into a variable first, or use `-File -Args`.
- **PowerShell's `%` alias** in a `.bat` — collides with batch `%`. Spell out `ForEach-Object`.
- **Forgetting `-NoProfile`** — slow, and a user's profile can change behaviour or emit text that corrupts your `for /F` capture.
- **Not handling PowerShell errors** — a failed `Invoke-WebRequest` may still exit `0` unless you add `-ErrorAction Stop` / check `$?` / `exit $LASTEXITCODE`.
- **`pwsh` assumed present** — probe with `where /q pwsh`.
- **Calling PowerShell in a tight loop** — 150 ms × N. Invert: one PowerShell call does the loop.

## Exercises

1. Replace a `wmic logicaldisk` free-space check with a PowerShell one-liner; print free GB on every fixed drive.
2. Write `:have TOOL` using `where /q`; use it to require `git`, `curl`, and `7z`, listing all that are missing before exiting.
3. Get the machine's OS caption and build number via PowerShell into two variables.
4. Download a small file with `Invoke-WebRequest`, verify its SHA-256 against an expected value (compute with `Get-FileHash`), and `exit /b 1` on mismatch.
5. Move a 3-line PowerShell snippet from an ugly `-Command` string into `helper.ps1` called with `-File` and parameters. Note how much clearer it is.
6. Benchmark: call `powershell -NoProfile -Command "1"` 50 times in a `for /L` loop and time it. Then do the equivalent work in a single PowerShell invocation. Compare.

## Recap

- `where` / `where /q` = locate executables, the standard "is X installed" guard.
- `wmic` is deprecated and **absent on new Windows** — don't put it in shipped scripts.
- Call PowerShell for system info: `for /F "usebackq delims=" %%R in ('powershell -NoProfile -Command "EXPR"') do set "R=%%R"`.
- Quoting: batch `"` outside, PowerShell `'` inside; escalate to `-File script.ps1` the moment you need `"` inside or the logic grows.
- Use `powershell` (5.1, always present) for portability; probe for `pwsh`.
- One PowerShell process ≈ 150 ms — never call it in a hot loop; invert the control flow.

Next: [Text processing with findstr →](05-text-processing-findstr.md)
