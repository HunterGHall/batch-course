# 4. Error Handling & Exit Codes

## What & why

A script that keeps going after a step fails will happily deploy broken code or delete the wrong thing. Batch's error model is `errorlevel` — a single integer left behind by the last command — plus the `&&` / `||` operators. It has enough quirks to deserve its own lesson.

## `errorlevel` vs `%errorlevel%`

These are **not the same thing**:

| Form | What it is |
| --- | --- |
| `if errorlevel N` | a syntax: true when the exit code is **≥ N** |
| `%errorlevel%` | an environment variable holding the last exit code |

```bat
somecmd
if errorlevel 1 echo failed (code is 1 or higher)
if %errorlevel% equ 3 echo failed with exactly 3
if %errorlevel% neq 0 echo failed somehow
```

### `if errorlevel 0` is always true

Every exit code is ≥ 0. To test "succeeded," use one of:

```bat
if %errorlevel% equ 0 echo ok
if not errorlevel 1 echo ok
```

### Test high-to-low when branching

```bat
somecmd
if errorlevel 3 ( echo severe ) else if errorlevel 1 ( echo minor ) else ( echo ok )
```

## The `%errorlevel%` staleness trap

`%errorlevel%` is expanded at **parse time** ([Intermediate Lesson 1](01-delayed-expansion.md)). Inside a block it's frozen:

```bat
(
    failing_command
    echo %errorlevel%          &rem prints the value from BEFORE the block
    if %errorlevel% neq 0 echo caught     &rem also stale!
)
```

Fixes:

```bat
rem A) delayed expansion:
setlocal EnableDelayedExpansion
(
    failing_command
    if !errorlevel! neq 0 echo caught
)

rem B) the bareword syntax isn't stale — it re-reads each time:
(
    failing_command
    if errorlevel 1 echo caught
)

rem C) check right after, outside the block
```

`if errorlevel N` (the keyword form) is **not** stale — prefer it inside blocks.

## Explicitly setting an exit code

```bat
exit /b 0        &rem success, return from script/subroutine
exit /b 1        &rem generic failure
exit /b %errorlevel%    &rem propagate whatever just happened
```

`exit /b` returns from the current script or `call`ed subroutine. Plain `exit` (no `/b`) terminates the **entire `cmd.exe`** — closes the window if it was launched for the script. In a subroutine, `exit /b N` also sets `errorlevel` to `N` for the caller.

### Resetting errorlevel

```bat
cmd /c exit 0            &rem forces errorlevel to 0
ver >nul                  &rem also a reliable "success" that clears it
(call )                    &rem sets errorlevel to 0 (obscure but common)
(call)                     &rem sets errorlevel to 1
```

`set "errorlevel=0"` **does not work** the way you want — it creates a normal variable that *shadows* the real one until you `set "errorlevel="` again. Don't.

## `&&` and `||`

```bat
build.exe && echo BUILD OK
build.exe || (echo BUILD FAILED & exit /b 1)

step1 && step2 && step3 || echo "a step failed"
```

- `A && B` — run `B` only if `A` exited `0`.
- `A || B` — run `B` only if `A` exited non-zero.
- They chain left to right; `&&` and `||` bind tighter than `&`.

!!! warning "Not every command sets errorlevel"
    Many internal commands (`echo`, `set` in `.bat`, `rem`, `cd` to a valid dir, `for`, `if`) **leave errorlevel unchanged** on success — and some don't set it on failure either. `&&`/`||` after them can act on a *stale* code. Reliable setters: external `.exe`s, `findstr`/`find`, `xcopy`/`robocopy`, `ping`, `reg`, `sc`, `net`. When in doubt, test explicitly.

## Common per-command notes

| Command | Failure signal |
| --- | --- |
| `robocopy` | code **≥ 8** is an error (0–7 are success variants) — see [Beginner Lesson 7](../beginner/07-files-and-folders.md) |
| `find` / `findstr` | `1` = no match found, `2` = bad syntax/file |
| `ping` | `1` on no reply (but also check output — `ping` of an unresolvable name may still be `0` on old builds) |
| `reg query` | `1` if the key/value doesn't exist |
| `sc query` | `1060` if the service doesn't exist |
| `del` of a missing file | `0` (!) — check `if exist` first |
| `md` of an existing dir | `1` |

## A robust step pattern

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion

call :step "Restore packages"  nuget restore MyApp.sln
call :step "Build"             msbuild MyApp.sln /p:Configuration=Release
call :step "Run tests"         dotnet test

echo All steps passed.
exit /b 0

:step
    set "label=%~1"
    shift
    set "cmd=%1"
    :collect
    shift
    if not "%~1"=="" ( set "cmd=!cmd! %1" & goto collect )
    echo === !label! ===
    !cmd!
    if errorlevel 1 (
        echo.
        echo FAILED at step: !label!  ^(exit code !errorlevel!^) 1>&2
        exit /b 1
    )
    goto :eof
```

(Argument re-assembly like this is fragile with quotes — [Advanced Lesson 2](../advanced/02-robust-argument-parsing.md) does it properly. For real pipelines, many people just write one `|| exit /b 1` per line.)

## Simple and reliable

For most scripts this is enough:

```bat
@echo off
setlocal

step_one    || exit /b 1
step_two    || exit /b 1
step_three  || exit /b 1

echo Done.
```

## Common mistakes

- **`if errorlevel 0`** — always true.
- **`%errorlevel%` inside `( )`** — stale; use `if errorlevel N` or `!errorlevel!`.
- **`set errorlevel=0`** — creates a shadow variable and hides the real exit code from then on.
- **Trusting `&&`/`||` after `echo`/`set`/`cd`** — they may not update errorlevel.
- **`del missing.txt` then `|| ...`** — `del` returns `0` for a missing file.
- **`robocopy` with `if errorlevel 1`** — flags a successful copy as failure.
- **Not propagating** — a subroutine fails, but the script `exit /b 0`s anyway. End with `exit /b %errorlevel%` or check.
- **`exit` vs `exit /b`** — `exit` closes the whole window, taking your logs with it.

## Exercises

1. Run `findstr foo nul` (fails), then show three ways to detect the failure: `if errorlevel 1`, `if %errorlevel% neq 0`, and `||`.
2. Write a block that runs a failing command inside `( )` and correctly reports the code both with `if errorlevel` and with `!errorlevel!` (delayed). Show the stale `%errorlevel%` version failing.
3. Chain three commands with `&&` so that a failure in step 2 skips step 3 and the script exits non-zero.
4. Call `robocopy` to mirror a folder and classify the result: nothing-to-do / copied / real-error, using the numeric ranges.
5. Write `:require_cmd NAME` that checks `where NAME >nul 2>nul` and `exit /b 1` with a message if the tool isn't installed. Use it to require `git` and `curl`.
6. Demonstrate the `set "errorlevel=5"` shadow bug and then un-break it.

## Recap

- `if errorlevel N` = "code ≥ N" (so `if errorlevel 0` is useless); `%errorlevel%` = the actual number.
- Inside `( )` blocks `%errorlevel%` is stale — use `if errorlevel N` or `!errorlevel!`.
- `exit /b N` returns with code N; plain `exit` kills the shell.
- `A && B` runs B on success; `A || B` runs B on failure. Don't trust them after `echo`/`set`/`cd`.
- Know the per-command conventions: `robocopy` ≥ 8, `find`/`findstr` = 1 for no-match, `del` of a missing file = 0.
- Simplest reliable pattern: one `|| exit /b 1` per step.

Next: [setlocal, endlocal & scope →](05-setlocal-endlocal-scope.md)
