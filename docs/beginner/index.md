# Beginner

Start here if you've never written a `.bat` file. By the end you'll be writing real scripts with arguments, conditionals, loops and subroutines, and you'll have built a script that sorts a messy folder into subfolders by file type.

## Lessons

| # | Lesson | You'll learn |
| --- | --- | --- |
| 1 | [Setup & your first script](01-setup-and-first-script.md) | Command Prompt, `.bat` vs `.cmd`, `@echo off`, running scripts |
| 2 | [echo, comments & script structure](02-echo-comments-structure.md) | `echo`, `echo.`, `rem`, `::`, `pause`, `cls`, `title` |
| 3 | [Variables & `set`](03-variables-and-set.md) | `set`, `set /p`, `set /a`, `%var%` expansion, quoting |
| 4 | [Command-line arguments](04-command-line-arguments.md) | `%1`–`%9`, `%*`, `shift`, `%~dp0` and other modifiers |
| 5 | [Conditionals with `if`](05-conditionals-with-if.md) | `if`/`else`, `errorlevel`, `exist`, `defined`, comparisons |
| 6 | [Loops with `for`](06-loops-with-for.md) | plain `for`, `/L`, `/D`, `/R`, `/F`, `%%i` vs `%i` |
| 7 | [Files & folders](07-files-and-folders.md) | `copy`, `move`, `del`, `md`, `rd`, `ren`, `dir`, `xcopy`/`robocopy` |
| 8 | [Redirection & pipes](08-redirection-and-pipes.md) | `>`, `>>`, `2>`, `2>&1`, `<`, `|`, `nul` |
| 9 | [Subroutines with `call` & `goto`](09-subroutines-call-goto.md) | labels, `call :label`, `exit /b`, returning values |
| 10 | [Capstone: file organizer](10-capstone-file-organizer.md) | put it all together in a real script |

## How to work through it

- One lesson per sitting. Type every example. Do the exercises.
- Run scripts from an **open Command Prompt window** so you can read errors.
- Don't move on until the "Recap" all makes sense.
- Stuck? Re-read "Common mistakes" — your bug is probably there.

## When you're done

You should be able to explain every line of the capstone and add a feature to it unaided. Then go to [Intermediate](../intermediate/index.md).
