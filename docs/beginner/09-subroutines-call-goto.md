# 9. Subroutines with `call` & `goto`

## What & why

Batch has no functions in the normal sense, but you can fake them with **labels** and `call :label`. This gives you reusable, parameterised blocks with their own arguments and a way to return a value. `goto` handles plain jumps and loops.

## Labels and `goto`

A label is a line starting with `:`. `goto label` jumps there.

```bat
@echo off
goto start

:helper
echo (this is skipped on the way down)

:start
echo Running.
if "%1"=="" goto usage
echo Arg: %1
goto :eof

:usage
echo Usage: %~nx0 ^<name^>
exit /b 1
```

`goto :eof` is a built-in label meaning "end of file" — it ends the current script or subroutine cleanly. (The colon is required: `goto :eof`, not `goto eof`.)

## `call :label` — a subroutine

```bat
@echo off
setlocal

call :greet World
call :greet "Ada Lovelace"
call :add 3 4

echo Back in main.
exit /b 0

:greet
    echo Hello, %~1!
    goto :eof

:add
    set /a _sum = %1 + %2
    echo Sum is %_sum%
    goto :eof
```

Key points:

- `call :greet World` jumps to `:greet`, runs until `goto :eof` (or `exit /b`), then returns to the line after the `call`.
- Inside the subroutine, `%1 %2 ...` are the subroutine's **own** arguments, not the script's. `%0` is the label.
- `%~1` strips quotes, same modifiers as always.
- **Put `exit /b 0` before your first label** so main execution doesn't "fall through" into the subroutines.

## Returning a value

Batch subroutines can't `return x`. Two patterns:

### 1. Set a variable the caller reads

```bat
call :square 5 result
echo 5 squared is %result%
exit /b 0

:square
    set /a "%~2=%~1 * %~1"
    goto :eof
```

Here the caller passes the *name* of the output variable (`result`) as `%2`, and the subroutine assigns to it. This works because a `call`ed subroutine shares the caller's variable scope (no automatic `setlocal`).

### 2. Return via exit code (for small integers / status)

```bat
call :is_even 7
if errorlevel 1 (echo odd) else (echo even)
exit /b 0

:is_even
    set /a "r = %~1 %% 2"
    exit /b %r%
```

`exit /b N` sets `errorlevel` to `N` and returns from the subroutine. `exit /b` (no number) returns without changing `errorlevel`. `exit` **without** `/b` closes the entire Command Prompt — rarely what you want in a script.

## `call` for other things

`call` also re-runs the parser on a line, which lets you do a double expansion:

```bat
set "var1=Hello"
set "name=var1"
call echo %%%name%%%        &rem prints: Hello
```

And `call` another batch file (returns control after it finishes, unlike a bare invocation which... also returns, but `call` is explicit and required if you want `%errorlevel%` and to come back reliably):

```bat
call "%~dp0build.bat" release
if errorlevel 1 exit /b 1
call "%~dp0deploy.bat"
```

Without `call`, invoking `build.bat` from inside `run.bat` transfers control and **never comes back** to `run.bat`.

## Recursion

A subroutine can `call` itself. Use `setlocal`/`endlocal` inside to keep each level's variables separate, and guard the depth:

```bat
call :countdown 5
exit /b 0

:countdown
    if %~1 lss 0 goto :eof
    echo %~1
    set /a "next=%~1 - 1"
    call :countdown %next%
    goto :eof
```

More in [Advanced Lesson 1](../advanced/01-recursion-and-advanced-for.md).

## A clean script skeleton

```bat
@echo off
setlocal EnableExtensions

if "%~1"=="" goto :usage

call :main %*
exit /b %errorlevel%

:main
    echo Doing work with: %*
    call :log "started"
    rem ... real work ...
    call :log "finished"
    goto :eof

:log
    echo [%date% %time%] %~1
    goto :eof

:usage
    echo Usage: %~nx0 ^<args^>
    exit /b 1
```

## Common mistakes

- **Falling through into subroutines** — forgot `exit /b` before the first label. Every run "also" executes every subroutine top-to-bottom.
- **`goto eof`** instead of `goto :eof` — `eof` is a label you probably don't have → "cannot find the batch label."
- **Calling a subroutine without `call`** — `:greet` alone doesn't call it; `goto :greet` jumps and never returns.
- **Invoking another `.bat` without `call`** — control doesn't return.
- **`exit` vs `exit /b`** — `exit` kills the whole shell/window.
- **Expecting subroutine `%1` to be the script's `%1`** — it's the subroutine's own arg list.
- **Label names with spaces or after `:` a space** — `: mylabel` may not match `goto :mylabel`. No space after the colon.
- **`goto` into a `for` loop body** — doesn't resume the loop.

## Exercises

1. Write a script with a `:usage` label and `goto :usage` when `%1` is empty; otherwise `call :run %*`.
2. Write `:max a b` that prints the larger of two numbers. Call it three times with different pairs.
3. Write `:square n outvar` that sets the named variable to `n*n`. Verify `%result%` after the call.
4. Write `:is_prime n` that returns `0` via `exit /b` if prime, `1` if not. Test it on 2, 9, 13, 15, 17.
5. Write a recursive `:factorial n outvar`. Compute 5! (= 120) and 7! (= 5040). Note where it overflows 32-bit.
6. Demonstrate the fall-through bug: put a subroutine at the bottom with an `echo`, omit `exit /b` before it, run, and observe it runs unbidden. Fix it.

## Recap

- Labels are `:name`; `goto :name` jumps; `goto :eof` ends the script/subroutine.
- `call :name args` runs a subroutine and returns to the next line; inside, `%1..%9` are the subroutine's args.
- Put `exit /b` before the first label to avoid fall-through.
- Return values by assigning to a caller-named variable, or via `exit /b N` + `errorlevel`.
- `call other.bat` returns control; bare `other.bat` does not.
- `exit /b` returns from the script; `exit` closes the shell.

Next: [Capstone: file organizer →](10-capstone-file-organizer.md)
