# 3. Working with the Registry

## What & why

The registry is where Windows and installed software keep configuration: file associations, startup programs, environment variables, uninstall entries, policy. `reg.exe` reads and writes it from batch. This is powerful and dangerous — a bad `reg add` to `HKLM` can break the machine — so the emphasis here is on *reading* safely and *writing* carefully.

## `reg query` — reading

```bat
rem One value:
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" /v Hidden

rem A whole key (its values):
reg query "HKCU\Environment"

rem Recurse:
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" /s

rem Search for a value name or data:
reg query "HKLM\SOFTWARE" /f "Python" /s /k    &rem /k = match key names
```

### Parsing a single value with `for /F`

`reg query` output for a value looks like:

```
HKEY_CURRENT_USER\Software\...\Advanced
    Hidden    REG_DWORD    0x1
```

Extract the data (3rd token onward, tab/space delimited):

```bat
set "val="
for /F "tokens=2,*" %%a in ('reg query "HKCU\...\Advanced" /v Hidden 2^>nul ^| findstr /i "Hidden"') do set "val=%%b"
rem %%a = REG_DWORD, %%b = 0x1
echo Hidden = %val%
```

More robust (skip to the line with the value name, take the last token):

```bat
for /F "tokens=1,2*" %%a in ('reg query "HKCU\Console" /v FontSize 2^>nul ^| findstr /ri "REG_"') do (
    set "name=%%a"
    set "type=%%b"
    set "data=%%c"
)
```

!!! warning "Value data can contain spaces"
    `REG_SZ` data like `C:\Program Files\App` has spaces. Use `tokens=1,2*` so the third capture grabs "the rest," and don't split on it further. Value *names* with spaces exist too and make `findstr` matching ambiguous — match on the `REG_` type line instead when you can.

### Check existence

```bat
reg query "HKLM\SOFTWARE\MyApp" >nul 2>&1 && echo installed || echo not installed
reg query "HKCU\Environment" /v MY_VAR >nul 2>&1 && echo MY_VAR is set
```

`reg query` returns `1` when the key or value doesn't exist.

## `reg add` — writing

```bat
reg add "HKCU\Software\MyApp" /v "Version" /t REG_SZ /d "1.2.3" /f
reg add "HKCU\Software\MyApp" /v "Enabled" /t REG_DWORD /d 1 /f
reg add "HKCU\Software\MyApp" /v "Path" /t REG_EXPAND_SZ /d "%%USERPROFILE%%\MyApp" /f
```

| Switch | Meaning |
| --- | --- |
| `/v NAME` | value name (omit + `/ve` for the "(Default)" value) |
| `/t TYPE` | `REG_SZ`, `REG_DWORD`, `REG_EXPAND_SZ`, `REG_MULTI_SZ`, `REG_BINARY`, `REG_QWORD` |
| `/d DATA` | the data |
| `/f` | force — no "overwrite?" prompt (needed in scripts) |
| `/s SEP` | separator char for `REG_MULTI_SZ` (default `\0`) |

`REG_EXPAND_SZ` with a literal `%USERPROFILE%`: in a batch script you must write `%%USERPROFILE%%` so one `%` survives to the registry.

## `reg delete` — removing

```bat
reg delete "HKCU\Software\MyApp" /v "OldSetting" /f      &rem one value
reg delete "HKCU\Software\MyApp" /f                       &rem the whole key + subkeys
reg delete "HKCU\Software\MyApp" /va /f                    &rem all values, keep subkeys
```

!!! danger
    `reg delete "HKLM\SOFTWARE\..." /f` has no undo and no Recycle Bin. Export first (below), double-check the path isn't built from an empty variable, and prefer `HKCU` over `HKLM` whenever the setting is per-user.

## Backup & restore: `reg export` / `reg import`

```bat
reg export "HKCU\Software\MyApp" "%~dp0myapp-backup.reg" /y
rem ... make changes ...
rem to roll back:
reg import "%~dp0myapp-backup.reg"
```

Always `reg export` the key before a script modifies it in place. `.reg` files are text — you can inspect them.

## Common real tasks

### Add a startup program (per-user, no admin)

```bat
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "MyTool" /t REG_SZ /d "\"%~f0\" --run" /f
rem remove:
reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "MyTool" /f
```

### Read a persistent environment variable (not the volatile session one)

```bat
for /F "tokens=2*" %%a in ('reg query "HKCU\Environment" /v PATH 2^>nul') do set "userpath=%%b"
```

(Editing `PATH` via `reg` then broadcasting `WM_SETTINGCHANGE` is fiddly — `setx` is the batch-friendly way to *write* a persistent var, though it truncates at 1024 chars and doesn't affect the current session.)

### Enumerate subkeys

```bat
for /F "tokens=*" %%k in ('reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"') do (
    for /F "tokens=2*" %%n in ('reg query "%%k" /v DisplayName 2^>nul ^| findstr /i "DisplayName"') do echo %%o
)
```

(Lists installed program names. `2>nul` because not every subkey has `DisplayName`.)

## 32-bit vs 64-bit views

On 64-bit Windows, 32-bit and 64-bit processes see different `HKLM\SOFTWARE` (the 32-bit view is redirected to `...\WOW6432Node`). Force a view:

```bat
reg query "HKLM\SOFTWARE\MyApp" /reg:64
reg query "HKLM\SOFTWARE\MyApp" /reg:32
```

A batch script launched from a 32-bit parent may be querying the wrong hive without `/reg:64`.

## Common mistakes

- **`reg add` without `/f`** — prompts "overwrite? (Yes/No)" and hangs an unattended script.
- **Splitting value data on spaces** — use `tokens=1,2*` and treat the rest as one string.
- **`%VAR%` in `/d` for `REG_EXPAND_SZ`** — batch expands it now; you wanted `%%VAR%%`.
- **Writing to `HKLM` without admin** — "Access is denied"; check elevation ([Intermediate Lesson 9](../intermediate/09-scheduling-and-running.md)).
- **`reg delete` with a path from an unvalidated variable** — could target something huge. Guard it.
- **No `reg export` before modifying** — no way back.
- **Wrong bitness view** — 32-bit script silently reads `WOW6432Node`. Add `/reg:64`.
- **Assuming `reg query` errorlevel** — `1` for missing is reliable; parse failures may still be `0` with junk output.
- **`REG_MULTI_SZ` editing** — the `\0` separators make this genuinely painful in batch; consider PowerShell.

## Exercises

1. Read the current user's `TEMP` value from `HKCU\Environment` and print it, handling the "not set" case.
2. Detect whether a program (say "7-Zip") is installed by searching the `Uninstall` keys; print its `DisplayName` and `DisplayVersion` if found.
3. Write `:reg_get KEY VALUE outvar` that returns the data of a single value (any type, spaces intact) or empty if missing.
4. Add your script to `HKCU\...\Run` as a startup item, then remove it. Verify with `reg query` each time.
5. Export a test key, add three values, delete one, then restore from the export and confirm you're back to the original state (`reg export` again and `fc` the two `.reg` files).
6. Query `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion` `/v ProductName` with and without `/reg:64` from the same script; note any difference on a 64-bit machine.

## Recap

- `reg query KEY /v NAME` reads; parse with `for /F "tokens=1,2*"` and treat token 3 as the whole (possibly spaced) data. Missing key/value → errorlevel `1`.
- `reg add ... /t TYPE /d DATA /f` writes (always `/f` in scripts); use `%%VAR%%` for `REG_EXPAND_SZ` literals.
- `reg delete ... /f` is irreversible — `reg export` first, guard the path, prefer `HKCU`.
- `reg export` / `reg import` are your backup/restore.
- Add `/reg:64` (or `/reg:32`) to avoid the WOW6432Node redirection surprise.
- For `REG_MULTI_SZ` and `PATH` broadcasting, consider PowerShell instead.

Next: [WMIC, where & calling PowerShell →](04-wmic-where-powershell.md)
