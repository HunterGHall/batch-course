# 4. Command-Line Arguments

## What & why

Arguments make a script reusable: `backup.bat C:\data D:\backup` instead of editing the script every time. Batch exposes arguments as the numbered variables `%1` through `%9`, plus `%0` for the script itself and `%*` for everything.

## The basics

```bat
@echo off
echo Script:     %0
echo First arg:  %1
echo Second arg: %2
echo All args:   %*
```

Run it:

```bat
args.bat hello world "two words"
```

Output:

```
Script:     args.bat
First arg:  hello
Second arg: world
All args:   hello world "two words"
```

Note `%3` here is `two words` (quotes preserved in `%3`, stripped when you use `%~3` — see below). Arguments are split on spaces unless quoted.

## Missing arguments

An unset `%1` expands to nothing. Guard it:

```bat
if "%~1"=="" (
    echo Usage: %~nx0 ^<source^> ^<dest^>
    exit /b 1
)
```

## The `~` modifiers

`%~1` strips surrounding quotes from argument 1. You can also decompose a path argument:

| Syntax | Meaning | Example (`%1` = `"C:\dir\file.txt"`) |
| --- | --- | --- |
| `%~1` | remove quotes | `C:\dir\file.txt` |
| `%~f1` | full path | `C:\dir\file.txt` |
| `%~d1` | drive | `C:` |
| `%~p1` | path (no drive, no file) | `\dir\` |
| `%~n1` | file name only | `file` |
| `%~x1` | extension | `.txt` |
| `%~nx1` | name + extension | `file.txt` |
| `%~dp1` | drive + path | `C:\dir\` |
| `%~s1` | short (8.3) path | `C:\dir\FILE.TXT` |
| `%~a1` | file attributes | `--a------` |
| `%~t1` | timestamp | `2026-01-15 09:30` |
| `%~z1` | size in bytes | `1024` |

These work on any argument (`%~nx2`) and — crucially — on `%0`:

```bat
echo This script is: %~nx0
echo Its folder is:  %~dp0
```

`%~dp0` (the script's own directory, with trailing `\`) is the single most useful expression in batch. Use it to reference files next to your script regardless of the current directory:

```bat
call "%~dp0helper.bat"
set "config=%~dp0settings.ini"
```

## More than 9 arguments: `shift`

`%10` does **not** mean argument 10 — it's `%1` followed by `0`. To reach further, use `shift`, which discards `%1` and slides everything down:

```bat
@echo off
:loop
if "%~1"=="" goto done
echo Processing: %~1
shift
goto loop
:done
echo All arguments processed.
```

!!! warning
    `shift` does **not** affect `%*`. And `shift /1` starts shifting from `%1` leaving `%0` alone (rarely needed).

## Quoting rules

- Wrap arguments containing spaces in `"double quotes"`.
- Inside the script, use `%~1` to get the value without quotes, then re-quote where needed: `copy "%~1" "%~2"`.
- `%*` keeps original quoting; it's for pass-through: `othertool.exe %*`.

## Common mistakes

- **`if %1==x`** with no argument → `if ==x` → syntax error. Always `if "%~1"=="x"`.
- **`%10`** to mean the tenth argument. It's `%1` + `0`.
- **Assuming `%*` reflects `shift`** — it doesn't.
- **Forgetting `%~dp0` ends with `\`** — `"%~dp0\file"` gives a double backslash (usually harmless, but ugly and sometimes breaks). Write `"%~dp0file"`.
- **Paths with `&` or `^`** passed unquoted — they break. Quote everything.
- **`%~t1` / `%~z1` format depends on locale** and on the file existing.

## Exercises

1. Write `greet.bat NAME` that prints `Hello, NAME!` and prints a usage line + `exit /b 1` if no name is given.
2. Write `fileinfo.bat PATH` that prints the drive, folder, base name, extension, and size of the file passed in.
3. Write `sum.bat` that accepts any number of integer arguments and prints their total (use `shift` and `set /a`).
4. Write `here.bat` that prints its own full path and its own folder, then `cd`s to its own folder and prints the new `%CD%`.
5. Call `greet.bat` from a second script using `call "%~dp0greet.bat" World`.
6. Pass an argument with spaces (`"C:\Program Files"`) and print it correctly with and without quotes.

## Recap

- `%0` is the script, `%1`–`%9` are positional args, `%*` is all of them.
- `%~1` strips quotes; `%~dp1`, `%~nx1`, `%~x1` etc. decompose a path.
- `%~dp0` = the script's own directory (with trailing `\`) — use it for sibling files.
- Beyond 9 args, loop with `shift` (which doesn't touch `%*`).
- Always compare arguments as `"%~1"=="..."`.

Next: [Conditionals with if →](05-conditionals-with-if.md)
