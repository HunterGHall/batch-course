# Intermediate

You can write basic scripts but you keep hitting bugs that make no sense — a variable that won't update inside a loop, a comparison that's always false, a script that "works" but leaves junk in your environment. This level explains *why*, and gives you the patterns that make batch scripts reliable.

## Lessons

| # | Lesson | You'll learn |
| --- | --- | --- |
| 1 | [Delayed expansion](01-delayed-expansion.md) | `setlocal enabledelayedexpansion`, `!var!`, the loop-variable bug |
| 2 | [String manipulation](02-string-manipulation.md) | substrings `%v:~n,m%`, substitution `%v:a=b%`, length, case |
| 3 | [`for /F` deep dive](03-for-f-deep-dive.md) | `tokens`, `delims`, `skip`, `eol`, `usebackq`; parsing command output |
| 4 | [Error handling & exit codes](04-error-handling-exit-codes.md) | `errorlevel` vs `%errorlevel%`, `&&`, `||`, `exit /b n` |
| 5 | [`setlocal`, `endlocal` & scope](05-setlocal-endlocal-scope.md) | environment isolation, returning a value past `endlocal` |
| 6 | [Arrays & pseudo data structures](06-arrays-and-data-structures.md) | `set "a[0]=..."`, counts, iteration, associative lookups |
| 7 | [Date, time & math](07-date-time-and-math.md) | `%date%`/`%time%` locale traps, `set /a` operators & bases |
| 8 | [User interaction & menus](08-user-interaction-and-menus.md) | `choice`, `set /p`, a robust menu loop |
| 9 | [Scheduling & running scripts](09-scheduling-and-running.md) | `schtasks`, `start`, elevation, `runas`, working directory |
| 10 | [Capstone: backup utility](10-capstone-backup-utility.md) | logging, rotation, error handling, exit codes |

## How to work through it

- Every lesson here fixes a bug you have already hit or are about to. Type the "broken" examples too — seeing the failure is the point.
- Keep [Lesson 1 (delayed expansion)](01-delayed-expansion.md) close; it explains half of all batch weirdness.

## When you're done

Go to [Advanced](../advanced/index.md).
