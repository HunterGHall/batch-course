# 6. Performance & Pitfalls

## What & why

Batch is an interpreter with no compilation, no real data types, and a habit of spawning subprocesses. Most scripts are I/O-bound and it doesn't matter. But when a script processes 100,000 lines or runs 10,000 iterations, naive batch can take *minutes*. This lesson covers where the time goes, how to speed it up, and — the actually important skill — recognising when to stop writing batch.

## Where the time goes

| Operation | Relative cost | Note |
| --- | --- | --- |
| Internal command (`set`, `echo`, `if`) | cheap | microseconds |
| `for /L` iteration | cheap-ish | ~1 µs of overhead each, plus the body |
| Variable expansion | cheap | but delayed expansion re-scans the line |
| Redirecting `>>` **inside a loop** | **expensive** | opens + closes the file every iteration |
| A pipe `|` | **expensive** | spawns **two** `cmd.exe` processes |
| Calling an external `.exe` | expensive | process creation ~1–5 ms |
| `powershell -Command` | **very expensive** | ~150–250 ms **each** |
| `call :sub` | moderate | re-parses; ~10× a `goto` |

## Rule 1: don't redirect inside a loop — redirect the loop

Slow (opens `out.txt` 100,000 times):

```bat
for /L %%i in (1,1,100000) do echo line %%i >> out.txt
```

Fast (opens it once):

```bat
> out.txt (
    for /L %%i in (1,1,100000) do echo line %%i
)
```

This one change can turn a 90-second script into a 2-second one.

## Rule 2: avoid pipes in loops

Each `|` is two new `cmd.exe` processes. `for /F "... in ('cmd | findstr ...')"` inside a loop that runs 5,000 times = 10,000 process launches. Restructure:

- Run the expensive command **once**, capture to a temp file, loop over the file.
- Combine filters: `dir /b /s | findstr x | findstr y` → `dir /b /s | findstr /r "x.*y"` (one pipe stage less), or `findstr` with `/g:`.

```bat
rem Instead of piping per iteration:
dir /b /s "%root%\*.log" > "%TEMP%\all.txt"
for /F "usebackq delims=" %%f in ("%TEMP%\all.txt") do call :process "%%f"
del "%TEMP%\all.txt"
```

## Rule 3: minimise `powershell` calls

One `powershell -Command` per loop iteration is the #1 batch performance killer. If you need PowerShell in a loop, **move the loop into PowerShell** and call it once:

```bat
rem BAD: 1000 × 200ms = 3+ minutes
for /F %%f in (list.txt) do (
    for /F %%h in ('powershell -NoProfile -Command "(Get-FileHash '%%f').Hash"') do echo %%f %%h
)

rem GOOD: one call
powershell -NoProfile -Command "Get-Content list.txt | ForEach-Object { $h=(Get-FileHash $_).Hash; \"$_ $h\" }" > hashes.txt
```

## Rule 4: prefer `for /F` over `set /p` loops for reading files

`set /p line=<file` in a loop reads only the first line each time; the pattern people use to "read line N" re-opens and re-scans. `for /F` reads the whole file in one pass.

## Rule 5: delayed expansion has a cost

With `EnableDelayedExpansion`, `cmd` re-scans each executed line for `!`. On a hot loop with long lines this adds up. If a loop body doesn't need `!`, you can `endlocal` the delayed scope around it — but usually the readability win is worth the small cost. Don't over-optimise this.

## Rule 6: `findstr`/`find` are fast; batch string loops are slow

Counting lines: `find /c /v "" < file` (one process, instant) beats a `for /F` loop with `set /a n+=1` (a line at a time in the interpreter) by orders of magnitude on big files.

## Measuring

Batch has no `time` builtin that gives elapsed ms. Options:

```bat
rem Coarse (seconds, locale issues):
echo start: %time%
call :work
echo end:   %time%

rem Better: PowerShell stopwatch around the whole thing
for /F %%m in ('powershell -NoProfile -Command "(Measure-Command { cmd /c work.bat }).TotalSeconds"') do echo %%m s
```

Or wrap sections and diff `%time%` converted to centiseconds (handle the leading space and midnight rollover).

## Non-performance pitfalls that bite hard

### Encoding

- `cmd` is an **OEM code page** (437/850) world by default. UTF-8 files with non-ASCII, or UTF-16 files, come through wrong in `for /F`, `type`, `findstr`.
- `chcp 65001` switches to UTF-8 but can break other things and isn't a full fix.
- BOMs at the start of a `.bat` file break the first command.

### Line endings

- Batch needs **CRLF**. A `.bat` saved with LF-only endings can fail in subtle ways (labels not found, `goto :eof` misbehaving), especially around `goto` and `call`.

### The 8191-character line limit

- Any single command line over ~8191 chars is truncated. Building a huge `PATH` or a long argument list hits this.

### `%` and `!` in data

- A filename or config value containing `%` or (with delayed expansion) `!` gets partially eaten. `for %%f` loop variables are safe; `%var%`/`!var!` round-trips are not.

### Locale

- `%date%`, `%time%`, number formatting, `sort` order, `findstr` ranges — all locale-sensitive. A script tested on `en-US` breaks on `de-DE` (comma decimal separator in `%time%`!).

### Concurrent runs

- Two copies of the same scheduled script writing the same log/temp file corrupt each other. Use a lock file or unique temp names (`%RANDOM%`, PID via `wmic`/PowerShell).

### `cmd` extension state

- If `EnableExtensions` is off (rare, but possible via registry or `cmd /E:OFF`), `for /F`, `call :label`, `%~dp0` all break. `setlocal EnableExtensions` at the top.

## When to stop writing batch

Switch to PowerShell (or Python, or a real program) when you hit **any** of:

- **JSON / XML / CSV parsing** beyond trivial splitting.
- **Arithmetic** with decimals, or numbers over 2^31.
- **Any loop calling `powershell`** — just write the whole thing in PowerShell.
- **Real text processing** — multiline patterns, replace-with-regex, context lines.
- **HTTP** beyond a single `Invoke-WebRequest`/`curl` call.
- **Data structures** — you're maintaining `arr[i]` pseudo-arrays and a `.count` and it's getting hairy.
- **Error handling that matters** — `try/catch`, structured logging, partial rollback.
- **The script is over ~200 lines** and still growing.
- **Unicode** input/output.
- You've written `setlocal EnableDelayedExpansion` and a `call set` double-expansion trick in the same script to work around the language.

Batch's remaining sweet spot: **short glue scripts, launchers, CI entry points, "double-click to run" tools, and anything that must work on a bare Windows box with nothing installed.** Use it there; reach for PowerShell everywhere else. It's fine — and common — to have a 5-line `.bat` whose whole job is `powershell -NoProfile -File "%~dp0real-logic.ps1" %*`.

## Common mistakes

- **`>> file` inside a loop** — redirect the whole loop instead.
- **Pipes / `powershell` calls per iteration** — capture once, loop over the result; or move the loop into PowerShell.
- **Counting with a `for /F` + `set /a` loop** — use `find /c /v ""`.
- **Ignoring encoding** until a customer with `é` in a path files a bug.
- **LF line endings** in a `.bat`.
- **No concurrency guard** on a scheduled script.
- **Pushing batch past its limits** because rewriting feels like giving up — the rewrite is usually shorter.

## Exercises

1. Write 200,000 lines to a file two ways — `>>` per iteration vs redirecting the whole `for /L`. Time both (`%time%` before/after). Report the ratio.
2. Take a script that pipes `dir /b /s | findstr` inside a 1,000-iteration loop and refactor it to one `dir` + temp file. Measure.
3. Convert a "hash every file in a list" script from per-file `powershell` to a single PowerShell invocation. Measure.
4. Count lines in a 500k-line file with `find /c /v ""` and with a `for /F` counter loop. Time both.
5. Create a file whose name contains `%` and one containing `!`; run them through a `for` loop that echoes `%%f` (works) and through a `set "x=%%f"` + `echo %x%` round-trip (breaks). Document the failure.
6. Take the intermediate backup capstone and add a lock file so two concurrent runs can't both proceed.
7. Pick a batch script you've written that's over 150 lines. Rewrite it in PowerShell. Compare line count, readability, and how the error handling turned out.

## Recap

- Cost order: internal cmds ≪ `for` iteration ≪ `call` ≪ external exe ≪ pipe (2 procs) ≪ `powershell -Command` (~200 ms).
- Redirect the **whole loop**, not each iteration. Avoid pipes and `powershell` inside loops — capture once, or move the loop into PowerShell.
- `find /c /v ""` beats a counter loop for line counts.
- Non-perf pitfalls: OEM codepage/Unicode, CRLF requirement, 8191-char line limit, `%`/`!` in data, locale-sensitive date/time/sort, concurrent-run collisions.
- Stop writing batch when you need real parsing, decimals/bignums, per-iteration PowerShell, structured errors, Unicode, or you're past ~200 lines. A `.bat` that just launches a `.ps1` is a legitimate design.

Next: [Hybrid scripts →](07-hybrid-scripts.md)
