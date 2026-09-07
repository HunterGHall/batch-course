# 8. User Interaction & Menus

## What & why

Interactive scripts — installers, admin tools, "pick an environment" prompts — need to read choices reliably and re-prompt on bad input. Batch gives you `set /p` (free text) and `choice` (single keypress with validation). Building a solid menu loop is a rite of passage.

## `set /p` — a line of text

```bat
set "name="
set /p "name=Your name: "
if not defined name (
    echo Name is required.
    exit /b 1
)
echo Hello, %name%.
```

Remember: on empty Enter, `set /p` keeps the variable's previous value — so **clear it first** (`set "name="`) and then check `if defined`.

### `set /p` from a file (read one line)

```bat
set /p "firstline=" < data.txt
echo First line is: %firstline%
```

### `set /p` is not safe for secrets

The typed text is visible on screen. There's no masked input in pure batch. For passwords, call PowerShell:

```bat
for /F "usebackq delims=" %%p in (`powershell -NoProfile -Command "$p=Read-Host -AsSecureString; [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($p))"`) do set "pw=%%p"
```

(Even then the value ends up in a plain variable — avoid handling secrets in batch at all where you can.)

## `choice` — one validated keypress

```bat
choice /C YN /M "Proceed"
if errorlevel 2 goto :no
if errorlevel 1 goto :yes
```

`choice` sets `errorlevel` to the **1-based position** of the chosen letter in `/C`. Test **high to low**.

| Switch | Meaning |
| --- | --- |
| `/C ABC` | valid keys (default `YN`) |
| `/M "text"` | prompt message |
| `/N` | don't show the `[A,B,C]?` list |
| `/CS` | case-sensitive |
| `/D A` | default key if `/T` expires |
| `/T 10` | timeout in seconds (needs `/D`) |

```bat
choice /C YNC /N /M "Deploy to [Y]es / [N]o / [C]ancel? "
if errorlevel 3 ( echo Cancelled & exit /b 1 )
if errorlevel 2 ( echo Skipped   & goto next )
if errorlevel 1 ( echo Deploying & call :deploy )
```

!!! note
    `choice` is a real executable (`choice.exe`), present on Windows 7+. On ancient systems you'd fake it with `set /p` + validation.

Timeout example (unattended-friendly):

```bat
choice /C YN /D N /T 30 /M "Auto-continue in 30s"
if errorlevel 2 exit /b 1
```

## A robust menu loop

```bat
@echo off
setlocal EnableExtensions

:menu
cls
echo ============================
echo   Maintenance Menu
echo ============================
echo   1. Clear temp files
echo   2. Restart service
echo   3. Show disk usage
echo   Q. Quit
echo.

choice /C 123Q /N /M "Choose: "
set "sel=%errorlevel%"

if %sel%==4 goto :quit
if %sel%==1 ( call :clear_temp   & pause & goto :menu )
if %sel%==2 ( call :restart_svc  & pause & goto :menu )
if %sel%==3 ( call :disk_usage   & pause & goto :menu )
goto :menu

:quit
echo Bye.
exit /b 0

:clear_temp
    echo Clearing %TEMP% ...
    rem del /q "%TEMP%\*" 2>nul
    goto :eof

:restart_svc
    echo (would restart the service)
    goto :eof

:disk_usage
    wmic logicaldisk get caption,freespace,size 2>nul
    goto :eof
```

Why `choice` over `set /p` here: it rejects invalid keys automatically, needs no Enter, and can't be empty. The `set "sel=%errorlevel%"` snapshot avoids `%errorlevel%` shifting under you.

### The `set /p` menu variant (when you need multi-char options)

```bat
:menu
set "sel="
set /p "sel=Choose an action (add / remove / list / quit): "
if /i "%sel%"=="quit" goto :quit
if /i "%sel%"=="add"    ( call :add    & goto :menu )
if /i "%sel%"=="remove" ( call :remove & goto :menu )
if /i "%sel%"=="list"   ( call :list   & goto :menu )
echo Unknown option: "%sel%"
goto :menu
```

## Confirmations for dangerous actions

```bat
:confirm_delete
    choice /C YN /N /M "Really delete %count% files? This cannot be undone. [y/n] "
    if errorlevel 2 ( echo Aborted. & exit /b 1 )
    rem proceed...
```

For extra friction (type the word):

```bat
set "typed="
set /p "typed=Type DELETE to confirm: "
if /i not "%typed%"=="DELETE" ( echo Aborted. & exit /b 1 )
```

## Pausing and "press any key"

```bat
pause                          &rem "Press any key to continue . . ."
pause >nul                     &rem same, but silent
timeout /t 5                   &rem wait 5s, or any key
timeout /t 5 /nobreak          &rem wait 5s, ignore keypresses
timeout /t 5 /nobreak >nul     &rem ...silently
```

`timeout` fails if stdin is redirected (e.g. in some CI) — use `ping -n 6 127.0.0.1 >nul` as a portable "sleep 5."

## Common mistakes

- **Not clearing the variable before `set /p`** — stale value on empty Enter.
- **Testing `choice` errorlevels low-to-high** — `if errorlevel 1` catches everything. Go high-to-low, or snapshot `%errorlevel%` and use `==`.
- **`choice /C YN` then `if %errorlevel%==0`** — `choice` never returns 0 (except on Ctrl+C / error). Valid results start at 1.
- **`set /p` for passwords** — visible on screen.
- **`timeout` in redirected/no-console context** — "ERROR: Input redirection is not supported." Use the `ping` trick.
- **Menu loop with no `cls`/redraw** — output piles up; user loses the menu.
- **`goto :menu` from inside a `for` loop** — you can't resume the loop; structure the menu outside loops.
- **Forgetting `/N`** on `choice` when you've drawn your own menu — you get a duplicate `[1,2,3]?`.

## Exercises

1. Prompt for a username, rejecting empty input, and loop until something is entered.
2. Build a Y/N confirmation subroutine `:ask "question"` that returns `0` for yes, `1` for no via `exit /b`.
3. Build the maintenance menu above for real: option 3 shows disk usage; others just `echo` a placeholder. Use `choice`.
4. Add a 15-second auto-quit to the menu using `choice /D Q /T 15`.
5. Make a "type the environment name to deploy" prompt that only proceeds if the user types `production` exactly (case-insensitive).
6. Replace a `timeout /t 3` with a redirection-safe 3-second wait and verify it works when the script's stdin is redirected from `nul`.

## Recap

- `set /p "x=prompt: "` reads a line; clear `x` first and check `if defined x` (empty Enter keeps the old value).
- `choice /C keys /N /M "..."` reads one validated keypress; `errorlevel` = 1-based position; test high-to-low or snapshot and use `==`.
- Menu = a `:label` + draw + `choice` + dispatch + `goto :label`, kept outside any `for` loop, with `cls` each pass.
- No masked input in batch — shell out to PowerShell for secrets, or don't.
- `timeout /t N /nobreak >nul` to sleep; fall back to `ping -n N+1 127.0.0.1 >nul` when stdin is redirected.

Next: [Scheduling & running scripts →](09-scheduling-and-running.md)
