# 3. Variables & `set`

## What & why

Batch has one data type: the string. Every variable is text. `set` creates and reads variables, `set /a` does arithmetic, and `set /p` reads user input. Getting the quoting right here saves you hours later.

## Creating variables

```bat
@echo off
setlocal

set name=Ada
set greeting=Hello there

echo %name%
echo %greeting%
```

### Quote the whole assignment

```bat
set "name=Ada"
set "path_to_logs=C:\Program Files\App\logs"
```

Quoting `"name=value"` means any trailing spaces or stray special characters after `value` are **not** included. Without quotes, `set name=Ada ` stores `"Ada "` with the trailing space. Make `set "x=y"` your default style.

!!! danger "Never name a variable `path`"
    `set path=C:\stuff` overwrites the system `PATH` for the rest of your script and breaks every command. Also avoid `set date=`, `set time=`, `set random=`, `set cd=` — these shadow dynamic built-ins.

## Reading variables: `%var%`

```bat
set "first=Grace"
echo %first%          &rem -> Grace
echo %first%%first%    &rem -> GraceGrace
echo Value is [%first%]
```

If a variable doesn't exist, `%missing%` expands to the literal text `%missing%` (not empty). This trips up comparisons — [Lesson 5](05-conditionals-with-if.md).

## Input: `set /p`

```bat
set /p "server=Enter server name: "
set /p "confirm=Delete everything? (y/n): "
echo You said %confirm% about %server%
```

`set /p` reads one line. If the user just presses Enter, the variable keeps its **previous** value (or stays undefined). Always initialise or check:

```bat
set "confirm=n"
set /p "confirm=Proceed? (y/n) [n]: "
```

## Arithmetic: `set /a`

```bat
set /a total = 3 + 4 * 2       &rem 11 (normal precedence)
set /a half = 10 / 3           &rem 3  (integer division, truncates)
set /a rem = 10 %% 3           &rem 1  (%% in a script, % at the prompt)
set /a next = count + 1
set /a count += 1              &rem compound assignment works
set /a hex = 0xFF              &rem 255
set /a oct = 010               &rem 8   (leading zero = octal! see mistakes)
```

Inside `set /a` you don't need `%` around variable names:

```bat
set "a=5"
set "b=6"
set /a "product = a * b"
echo %product%
```

Operators: `+ - * / %` and also `& | ^ ~ << >> ( )` and assignment forms `+= -= *= /= %= &= |= ^= <<= >>=`.

`set /a` only does **32-bit signed integers**. No floats. Overflows wrap around.

## Listing and clearing

```bat
set                 &rem list every variable
set p                &rem list every variable whose name starts with "p"
set "temp="          &rem delete the variable temp (note: no space before ")
```

## Environment vs script variables

There's no syntactic difference. `set "x=1"` creates `x`; if the script doesn't `setlocal`, that `x` leaks into the parent shell after the script ends. `setlocal` at the top prevents leaks — see [Intermediate Lesson 5](../intermediate/05-setlocal-endlocal-scope.md).

Useful built-ins: `%USERNAME%`, `%COMPUTERNAME%`, `%CD%` (current dir), `%DATE%`, `%TIME%`, `%RANDOM%` (0–32767), `%TEMP%`, `%APPDATA%`, `%USERPROFILE%`, `%WINDIR%`, `%PROCESSOR_ARCHITECTURE%`.

## Common mistakes

- **Spaces around `=`**: `set x = 5` creates variable `"x "` with value `" 5"`. Write `set "x=5"`.
- **Naming a variable `path`, `date`, `time`, `cd`** — shadows a built-in.
- **Expecting floats**: `set /a x=7/2` gives `3`, not `3.5`.
- **Leading zero = octal**: `set /a n=08` → `Invalid number. Numeric constants are either decimal (17), hexadecimal (0x11), or octal (021).` Strip leading zeros first (common with `%time%` and `%date%` fields).
- **`%var%` for a missing variable** stays as literal `%var%`, so `if %x%==1` becomes `if %x%==1` and errors on the space. Quote both sides: `if "%x%"=="1"`.
- **`set /p` on empty Enter** keeps the old value silently.

## Exercises

1. Prompt for a width and a height, then print the area and perimeter of the rectangle using `set /a`.
2. Prompt for a number of total seconds; print `M minutes and S seconds` using `/` and `%%`.
3. Prompt for a Celsius temperature and print Fahrenheit. (You'll notice the integer-only limitation — note where it loses precision.)
4. Write a script that rolls two dice: print two `%RANDOM% %% 6 + 1` values and their sum.
5. Demonstrate the trailing-space bug: `set greeting=Hi ` (trailing space) then `echo [%greeting%]`. Fix it with quotes.
6. Try `set /a n=09` and read the error. Then make it work.

## Recap

- Everything is a string. `set "name=value"` is the safe assignment form.
- `%var%` expands a variable; a missing one stays literal.
- `set /p "x=prompt: "` reads a line; empty input keeps the old value.
- `set /a` does 32-bit integer math; no `%` needed around names inside it; leading zeros mean octal.
- Never use `path`, `date`, `time`, `cd` as variable names.

Next: [Command-line arguments →](04-command-line-arguments.md)
