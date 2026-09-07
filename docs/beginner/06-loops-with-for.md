# 6. Loops with `for`

## What & why

`for` is the workhorse of batch. It has five modes selected by a switch (`for`, `for /L`, `for /D`, `for /R`, `for /F`), and one syntax rule that catches everyone: the loop variable is written `%%i` in a script but `%i` at the prompt.

## The `%%i` rule

```bat
@echo off
for %%f in (a.txt b.txt c.txt) do echo Found %%f
```

At the interactive prompt you'd type `for %f in (...) do echo %f`. In a **script**, double the percent: `%%f`. If you ever see `%f was unexpected at this time`, you forgot to double it.

Loop variables are single letters, case-sensitive: `%%f` and `%%F` are different.

## Mode 1: plain `for` — iterate a list / files

```bat
rem Literal list:
for %%c in (red green blue) do echo Color: %%c

rem File wildcards (this is a glob, not a literal string):
for %%f in (*.txt) do echo Text file: %%f

for %%f in ("C:\logs\*.log") do (
    echo %%~nxf is %%~zf bytes
)
```

The same `~` modifiers from [Lesson 4](04-command-line-arguments.md) work: `%%~nxf`, `%%~dpf`, `%%~zf`, `%%~tf`, etc.

## Mode 2: `for /L` — counting

```bat
rem for /L %%i in (start, step, end)
for /L %%i in (1,1,5) do echo %%i          &rem 1 2 3 4 5
for /L %%i in (10,-2,0) do echo %%i        &rem 10 8 6 4 2 0
for /L %%i in (0,1,3) do echo tick
```

The end value **is** included (unlike many languages).

## Mode 3: `for /D` — directories

```bat
rem Immediate subdirectories of the current folder:
for /D %%d in (*) do echo Subfolder: %%d

for /D %%d in ("C:\Projects\*") do echo %%~nxd
```

## Mode 4: `for /R` — recurse a tree

```bat
rem Every .log under C:\logs and below:
for /R "C:\logs" %%f in (*.log) do echo %%f

rem Every directory under the current one:
for /R %%d in (.) do echo DIR: %%d
```

`for /R` walks the whole subtree. Combine with `if` to filter. More in [Advanced Lesson 1](../advanced/01-recursion-and-advanced-for.md).

## Mode 5: `for /F` — parse text

The most powerful and the most complex. It reads lines from a file, a string, or the output of a command, and splits each line into tokens.

```bat
rem Lines of a file:
for /F "usebackq delims=" %%L in ("names.txt") do echo LINE: %%L

rem Output of a command:
for /F "tokens=*" %%L in ('whoami') do echo I am %%L

rem Split on a delimiter, grab specific columns:
for /F "tokens=1,3 delims=," %%a in ("alice,30,london") do echo %%a lives in %%b
```

`for /F` gets its own lesson: [Intermediate Lesson 3](../intermediate/03-for-f-deep-dive.md). For now, know it exists and that `tokens=`, `delims=`, `skip=`, `usebackq` are its options.

## `break` and `continue`?

There is **no `break`** and **no `continue`** in batch `for`.

- To skip an iteration: wrap the body in `if` so nothing runs.
- To stop early: there's no clean way. Set a flag and `if defined done` around the body, or restructure using `goto` out of the loop (you cannot `goto` a label inside the same `for` and keep looping, but you can `goto` *out*).

```bat
set "found="
for %%f in (*.txt) do (
    if not defined found (
        findstr /m "TODO" "%%f" >nul && set "found=%%f"
    )
)
if defined found echo First match: %found%
```

## The counter bug (preview)

This does **not** work as written:

```bat
@echo off
set /a count=0
for %%f in (*.txt) do (
    set /a count+=1
    echo %count%        REM  <-- always prints 0
)
echo Total: %count%      REM  <-- this one is correct
```

`%count%` inside the block is frozen at parse time. Fix: `setlocal enabledelayedexpansion` and `!count!`. Full explanation in [Intermediate Lesson 1](../intermediate/01-delayed-expansion.md).

## Common mistakes

- **`%i` instead of `%%i`** in a script → `%i was unexpected at this time`.
- **Expecting `break`/`continue`** — they don't exist.
- **Reading `%var%` set inside the loop** — it's stale; you need delayed expansion.
- **`for %%f in (*.txt)`** when there are no matches — the loop body runs **once** with `%%f` = the literal `*.txt`. Guard with `if exist *.txt` or check `%%~ff`.
- **`for /F` on a file with no `usebackq`** and a quoted path — without `usebackq`, `"names.txt"` is treated as a literal string to parse, not a filename. Use `usebackq` for quoted filenames.
- **Spaces in the `(start,step,end)` of `/L`** are tolerated but be consistent: `(1,1,5)`.

## Exercises

1. Print the numbers 1 to 20, and separately the even numbers 2 to 20, using `for /L`.
2. List every `.md` file in the current folder with its size in bytes (`%%~zf`).
3. Print every immediate subfolder of `%USERPROFILE%` using `for /D`.
4. Recursively count how many `.txt` files are under the current folder. (You'll need the counter workaround or a `>>` to a temp file, or just `dir /s /b *.txt | find /c /v ""` — try both.)
5. Given the string `2026-09-07`, use `for /F "tokens=1-3 delims=-"` to print `Day 07, Month 09, Year 2026`.
6. Loop over `red green blue`; skip `green` with an `if`.

## Recap

- Script loop variable is `%%i`; prompt is `%i`. Single letter, case-sensitive.
- `for` (list/glob), `for /L` (counting, end inclusive), `for /D` (subdirs), `for /R` (recursive), `for /F` (parse text).
- `~` modifiers work on the loop variable: `%%~nxf`, `%%~zf`, ...
- No `break`, no `continue` — use flags and `if`.
- Variables set inside a `( )` block read stale without delayed expansion.

Next: [Files & folders →](07-files-and-folders.md)
