# 5. Conditionals with `if`

## What & why

`if` is how a script makes decisions. Batch's `if` is limited and quirky: string comparison is the default, quoting matters enormously, and there are special forms for files, variables and exit codes.

## String comparison

```bat
@echo off
set "answer=yes"

if "%answer%"=="yes" (
    echo Confirmed.
) else (
    echo Cancelled.
)
```

**Always quote both sides.** If `answer` is empty, `if %answer%==yes` becomes `if ==yes` → `The syntax of the command is incorrect.` Quoting makes it `if ""=="yes"` → cleanly false.

`==` is the only string operator with that spelling. There is no `!=`; negate with `if not`:

```bat
if not "%answer%"=="yes" echo Not yes.
```

### Case-insensitive

```bat
if /i "%answer%"=="YES" echo Matches regardless of case.
```

## Numeric comparison

```bat
set /a count=5
if %count% GEQ 3 echo At least three.
```

| Operator | Meaning |
| --- | --- |
| `EQU` | equal |
| `NEQ` | not equal |
| `LSS` | less than |
| `LEQ` | less than or equal |
| `GTR` | greater than |
| `GEQ` | greater than or equal |

!!! warning
    These do a **numeric** comparison only when *both* sides look like numbers; otherwise they fall back to string comparison. `if "10" GTR "9"` is true numerically, but `if "10" GTR "9 "` (trailing space) compares as strings and is false. Keep both sides clean and unquoted for numeric intent: `if %a% LSS %b%`.

## `if exist` — files and folders

```bat
if exist "C:\logs\app.log" echo The log exists.
if not exist "%~dp0config.ini" (
    echo Missing config next to script.
    exit /b 1
)

rem A trailing \ tests for a directory specifically:
if exist "C:\logs\" echo C:\logs is a folder.
```

`if exist "folder\*"` is another way to test a non-empty folder.

## `if defined` — variables

```bat
if defined USERPROFILE echo Profile: %USERPROFILE%
if not defined MYVAR echo MYVAR is not set.
```

Use `if defined` instead of `if "%var%"==""` — it's cleaner and it works with delayed expansion (`if defined arr[%i%]`).

## `if errorlevel` — exit codes

```bat
ping -n 1 example.com >nul
if errorlevel 1 (
    echo Ping failed.
) else (
    echo Ping succeeded.
)
```

`if errorlevel N` means "**exit code is N or greater**." So `if errorlevel 1` catches any failure. To test for exactly one value, use `%errorlevel%`:

```bat
if %errorlevel% equ 0 echo Exactly zero.
```

Check them in order high-to-low if you branch on several:

```bat
if errorlevel 2 ( echo code ^>= 2 ) else if errorlevel 1 ( echo code == 1 ) else ( echo code == 0 )
```

More on this in [Intermediate Lesson 4](../intermediate/04-error-handling-exit-codes.md).

## Blocks, `else`, and the parenthesis trap

The `else` **must be on the same line as the closing `)`**:

```bat
if "%x%"=="1" (
    echo one
) else (
    echo not one
)
```

This is a syntax error:

```bat
if "%x%"=="1" (
    echo one
)
else (            REM  <-- "else" was unexpected at this time
    echo not one
)
```

Inside `( ... )` blocks, `%var%` is expanded **once, when the block is parsed** — not as it runs. This breaks counters and loop bodies. The fix is delayed expansion, [Intermediate Lesson 1](../intermediate/01-delayed-expansion.md). For now, keep `if` blocks simple.

## Chaining conditions

There is no `if A and B`. Nest, or use flags:

```bat
if "%user%"=="admin" if "%mode%"=="write" echo Allowed.

rem "or" via a helper flag:
set "ok="
if "%ext%"==".jpg" set "ok=1"
if "%ext%"==".png" set "ok=1"
if defined ok echo It's an image.
```

## Common mistakes

- **Unquoted comparison** with a possibly-empty value → syntax error.
- **`else` on its own line** → `else was unexpected at this time`.
- **`if errorlevel 0`** is *always true* (every code is ≥ 0). Use `if %errorlevel% equ 0` or `if not errorlevel 1`.
- **`==` with surrounding spaces**: `if "%x%" == "y"` — the spaces become part of neither side here, that's actually fine, but `if %x%==%y%` without quotes and with spaces around values is not. Prefer quoted, no spaces around `==`.
- **`if /i` position**: the `/i` goes right after `if`, before the first operand.
- **Testing a directory with `if exist name`** — works, but add a trailing `\` to be sure it's a folder not a file.

## Exercises

1. Prompt for a password; if it equals `hunter2`, print `Access granted`, else `Access denied` and `exit /b 1`.
2. Accept a number as `%1`. Print whether it is negative, zero, or positive using `LSS`/`EQU`/`GTR`.
3. Check whether `%~dp0notes.txt` exists; if not, create it with `echo.>"%~dp0notes.txt"` and print `Created`.
4. Accept `%1`; if it's empty, print usage and `exit /b 1`; if `/i` equal to `--help`, print help; otherwise echo it back.
5. Run `findstr xyz nul` (which fails), then branch on `if errorlevel 1`.
6. FizzBuzz for `%1`: use `set /a` and `if` with `%%` to print `Fizz`, `Buzz`, `FizzBuzz`, or the number.

## Recap

- Quote both sides: `if "%a%"=="%b%"`. Negate with `if not`. Case-insensitive with `if /i`.
- Numeric: `EQU NEQ LSS LEQ GTR GEQ` — keep operands clean numbers.
- `if exist "path"` (trailing `\` for folders), `if defined VAR`, `if errorlevel N` (means ≥ N).
- `else` sits on the `)` line. `%var%` inside `( )` is expanded at parse time — a real gotcha.

Next: [Loops with for →](06-loops-with-for.md)
