# 1. Delayed Expansion

## What & why

This is the single most important lesson in the course. Once you understand *when* `cmd.exe` substitutes variable values, half of batch's "haunted" behaviour becomes obvious.

## The two-phase model

`cmd.exe` processes a **whole command** (a line, or a parenthesised block) in two steps:

1. **Parse:** read the entire command, and replace every `%var%` with its current value **right now**.
2. **Execute:** run the resulting text, top to bottom.

If a value *changes during execution*, any `%var%` written in that same command is already gone — it was replaced with the old value during the parse step.

## The classic bug

```bat
@echo off
set /a count=0
for %%f in (*.txt) do (
    set /a count+=1
    echo Now at %count%
)
echo Final: %count%
```

Say there are 3 `.txt` files. Output:

```
Now at 0
Now at 0
Now at 0
Final: 3
```

The whole `for ( ... )` block is one command. At parse time, `echo Now at %count%` became `echo Now at 0` — three times, baked in. `set /a count+=1` *does* run and *does* update the variable (that's why `Final: 3` is right — that `echo` is a separate command, parsed later).

## The fix: delayed expansion

Turn it on, then use `!var!` instead of `%var%` for anything that changes mid-command:

```bat
@echo off
setlocal EnableDelayedExpansion
set /a count=0
for %%f in (*.txt) do (
    set /a count+=1
    echo Now at !count!
)
echo Final: !count!
```

Output:

```
Now at 1
Now at 2
Now at 3
Final: 3
```

`!count!` is expanded at **execution** time, once per iteration, with the live value.

## When to use `%` vs `!`

| Situation | Use |
| --- | --- |
| Value set **before** the current command/block | `%var%` (fine) |
| Value changes **inside** a `for` or `if ( )` block and read in the same block | `!var!` |
| Loop variable | `%%i` (never `!`) |
| Script arguments `%1`, `%~dp0` | `%` (they never change) |
| Inside `set /a` | bare name (`set /a x=y+1`) |

You can mix them: `%staticthing%` and `!counter!` on the same line is normal and correct.

## Turning it on

```bat
setlocal EnableDelayedExpansion
```

Or launch `cmd /V:ON`. It is **off by default** because `!` is a legal character in text, and enabling it changes how `!` is interpreted everywhere.

## The reverse gotcha: `!` when you don't want it

With delayed expansion ON, a literal `!` in a string gets eaten:

```bat
setlocal EnableDelayedExpansion
echo Hello World!!!          &rem prints "Hello World" — the !s vanish
set "x=a!b"                   &rem x becomes "ab"
```

Workarounds: escape as `^^!` inside quotes with delayed expansion, or (better) only `setlocal EnableDelayedExpansion` around the section that needs it and `endlocal` right after.

## Toggling around a section

```bat
setlocal DisableDelayedExpansion
set "password=p@ss!word"      &rem literal ! is safe here
endlocal & set "password=%password%"   &rem carry it out

setlocal EnableDelayedExpansion
for /L %%i in (1,1,3) do echo attempt !i-ish stuff
endlocal
```

## Reading a `for /F` token that has changed

```bat
setlocal EnableDelayedExpansion
set "last="
for /F "delims=" %%L in (log.txt) do (
    set "last=%%L"
    echo current: !last!
)
echo last line was: !last!
```

## Delayed expansion and `call`

`call` re-parses its line, giving you a *poor-man's* delayed expansion even without `EnableDelayedExpansion`:

```bat
for %%f in (*.txt) do (
    set /a count+=1
    call echo Now at %%count%%
)
```

`%%count%%` survives the block parse as `%count%`, then `call` expands it live. It's slower (a re-parse per iteration) and harder to read — prefer real delayed expansion. But you'll see it in older scripts.

## Common mistakes

- **Reading `%var%` set earlier in the same block** — always `0`/empty/stale.
- **Forgetting to `setlocal EnableDelayedExpansion`** — `!var!` just prints literally as `!var!`.
- **Strings with `!` while delayed expansion is on** — characters silently disappear.
- **Using `!!` around the loop variable** — the loop var is always `%%i`.
- **`endlocal` killing your result** — variables set inside are gone after `endlocal`; carry them out with `endlocal & set "x=%x%"` ([Lesson 5](05-setlocal-endlocal-scope.md)).
- **Assuming `set /a` needs `!`** — inside `set /a`, plain names are already read live.

## Exercises

1. Reproduce the counter bug with a `for /L 1..5` loop printing `%count%`. Then fix it with `!count!`.
2. Build a running total: loop over `10 20 30 40`, add each to `sum`, print the live subtotal each iteration and the final total.
3. In a `for /F` over the lines of a file, find the longest line and print it after the loop (needs `!` for the comparison and the saved line).
4. With delayed expansion on, try `echo Warning!` and `set "greeting=Hi there!"` then `echo %greeting%`. Explain what happened, then fix it two ways.
5. Rewrite exercise 2 using the `call echo %%sum%%` trick instead of delayed expansion. Compare readability.
6. Write a loop that builds a comma-separated list: from `a b c d` produce `a,b,c,d` with no trailing comma.

## Recap

- `cmd` parses a whole command/block first (substituting `%var%` **then**), and only afterwards executes it.
- `%var%` = value at parse time; `!var!` = value at execution time (needs `setlocal EnableDelayedExpansion`).
- Use `!var!` for anything that changes inside a `for`/`if` block; `%var%` is fine for everything static.
- Delayed expansion makes literal `!` disappear from strings — scope it tightly.
- `call echo %%var%%` is the fallback when you can't enable delayed expansion.

Next: [String manipulation →](02-string-manipulation.md)
