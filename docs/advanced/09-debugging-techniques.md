# 9. Debugging Techniques

## What & why

Batch has no debugger, no stack traces, and error messages that range from unhelpful (`The syntax of the command is incorrect.`) to actively misleading (`) was unexpected at this time.` pointing nowhere near the real problem). This lesson is a toolkit: how to see what's happening, decode the classic errors, and isolate the failing line.

## Rule 0: run it in a window that stays open

```bat
cmd /k script.bat arg1 arg2
```

`cmd /k` runs the script then **keeps the window open** so you can read the final error and inspect variables (`set`, `echo %errorlevel%`). Or, from an already-open prompt, just run it directly. Never debug by double-clicking.

## `echo on` — the poor man's tracer

Comment out `@echo off` (or add `echo on` at the point of interest). Now every command prints *with its variables already expanded* right before it runs:

```bat
@rem @echo off
echo on
set "target=%~1"
robocopy "%src%" "%target%" /MIR
```

You'll see exactly what `robocopy "%src%" "%target%" /MIR` expanded to — often the bug is right there (`%src%` empty, a stray quote, a trailing space).

Scoped tracing:

```bat
@echo off
rem ... quiet part ...
echo on
call :suspect_subroutine
@echo off
rem ... quiet again ...
```

## Breakpoints and inspection

```bat
echo [DEBUG] here; target=[%target%] count=[%count%] errorlevel=[%errorlevel%]
pause
```

Bracketing values (`[%x%]`) reveals empty strings and trailing spaces that bare `%x%` hides. `pause` freezes so you can look.

For loop bodies where `%var%` is stale, echo the delayed form:

```bat
setlocal EnableDelayedExpansion
for %%f in (*.txt) do (
    echo [DEBUG] file=[%%f] running_total=[!total!]
    set /a total+=%%~zf
)
```

## A reusable debug switch

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion
if /i "%~1"=="--debug" ( set "DEBUG=1" & shift )

call :dbg "starting with args: %*"
rem ...
call :dbg "target resolved to: %target%"
rem ...
exit /b 0

:dbg
    if defined DEBUG echo [%time%] [DEBUG] %~1 1>&2
    goto :eof
```

Send debug output to **stderr** (`1>&2`) so it doesn't pollute a `for /F` that's capturing the script's stdout.

## Logging everything

Wrap the real work and tee it:

```bat
@echo off
setlocal
set "LOG=%~dp0debug_%RANDOM%.log"
call :main %* > "%LOG%" 2>&1
set "rc=%errorlevel%"
type "%LOG%"
echo (exit %rc%, log: %LOG%)
exit /b %rc%

:main
    rem ... everything, with liberal echo ...
    goto :eof
```

Now every run leaves a full transcript including stderr.

## Decoding the classic errors

### `) was unexpected at this time.`

A parenthesised block broke. Usual causes:

- An unquoted value inside `if`/`for` that was empty or contained `)`: `if %x%==1 (` with `x` unset → `if ==1 (`. Fix: `if "%x%"=="1" (`.
- `else` on its own line instead of on the `)` line.
- A `)` inside a quoted string or path that isn't quoted in the command.
- A `::` comment inside a `( )` block — use `rem`.
- Missing `(` earlier, or a `for`/`if` header that didn't parse.

Find it: turn on `echo on`, or bisect — comment out half the block.

### `The syntax of the command is incorrect.`

- A `set /a` with a bad token, or `set "x=y"` with an unbalanced quote.
- A redirection with a stray space: `1 >file`.
- A `for` options string that's malformed.
- An empty `%var%` landing in a spot that needs *something*.

### `<something> was unexpected at this time.`

- `%i was unexpected` → you wrote `%i` instead of `%%i` in a script `for`.
- `0<something>` → a redirection parse issue.

### `The system cannot find the batch label specified - :eof` (or your label)

- `goto eof` instead of `goto :eof`.
- The `.bat` file has **LF-only line endings** — `cmd` can't locate labels reliably. Convert to CRLF.
- The label is inside a `( )` block or after an `exit /b` with no path to it.
- A typo, or the label has a trailing space / space after the `:`.

### `'x' is not recognized as an internal or external command`

- Typo, or the tool isn't on `PATH` in this context (scheduled task! — [Intermediate Lesson 9](../intermediate/09-scheduling-and-running.md)).
- A UTF-8 BOM at the top of the file turned `@echo` into `∩╗┐@echo`.
- A variable that should have held a command path is empty: `%TOOL% --version` with `%TOOL%` unset.

### The script "just exits" with no message

- An `exit` (no `/b`) somewhere killed the shell.
- A syntax error in a block that `cmd` swallowed.
- `goto` to a missing label after `@echo off` (the error scrolls past / window closes).
- Run under `cmd /k` and add `echo REACHED n` breadcrumbs.

## Bisecting

When you can't tell which line fails:

```bat
echo REACHED 1
call :phase_one
echo REACHED 2
call :phase_two
echo REACHED 3
```

The last `REACHED` you see brackets the failure. Then subdivide that section.

## `set` to dump state

At any breakpoint:

```bat
set                     &rem every variable
set cfg.                 &rem every variable starting cfg.
echo cd=[%cd%] el=[%errorlevel%]
```

## Common mistakes (in debugging itself)

- **Debugging by double-click** — the window closes on the error. `cmd /k`.
- **`echo %x%` without brackets** — you can't see that it's `"value "` with a trailing space, or empty.
- **Debug output on stdout** while something upstream captures it with `for /F` — send it to stderr.
- **Trusting the line number / token in the error** — batch often points at the wrong place; the real cause is earlier.
- **Not checking line endings** for "label not found" — LF is a silent killer.
- **Leaving `echo on`** in the shipped script.
- **Adding `pause` everywhere** then shipping it into a scheduled task that hangs.

## Exercises

1. Take a script with `if %x%==1 (` and an unset `x`. Reproduce `) was unexpected at this time.`, then fix it, using `echo on` to confirm what the line expanded to.
2. Save a working `.bat` with LF-only endings (your editor can do this). Run it; observe the label failure. Convert to CRLF; confirm it works.
3. Add a `--debug` flag and a `:dbg` subroutine (stderr output) to a script. Verify that piping the script's stdout to `find /c /v ""` still works with `--debug` on.
4. Wrap a script so every run tees full stdout+stderr to `debug_<rand>.log` and still shows on screen.
5. Put a UTF-8 BOM at the top of a `.bat` (save as "UTF-8 with BOM"). Observe the `'∩╗┐@echo' is not recognized` error. Re-save without BOM.
6. Given a 120-line script that "just exits," add `echo REACHED n` breadcrumbs to bisect to the offending line in under 4 runs.
7. Reproduce `The system cannot find the batch label specified` three different ways (`goto eof`, LF endings, label inside a block) and fix each.

## Recap

- Run under `cmd /k` (or from an open prompt) so the window and the error survive.
- `echo on` / commenting out `@echo off` traces every command **with variables expanded** — usually shows the bug directly.
- `echo [DEBUG] x=[%x%]` (with brackets) + `pause` = breakpoints; use `!x!` in loop bodies; send debug to **stderr**.
- Tee everything to a timestamped log for post-mortems.
- `) was unexpected` = broken block (usually an unquoted empty `if`/`for` value, or `else` on its own line, or `::` in a block).
- `label not found` = `goto :eof` missing the colon, **LF line endings**, or label in a block.
- `not recognized` = typo, not on `PATH` (scheduled context!), empty command variable, or a **BOM**.
- Bisect with `echo REACHED n`; dump state with `set`.

Next: [Capstone: deployment automation →](10-capstone-deployment-automation.md)
