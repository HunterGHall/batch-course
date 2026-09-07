# 7. Date, Time & Math

## What & why

Timestamped log files, "delete backups older than 30 days," "run only on weekdays" — all need date/time handling, and batch's is genuinely bad: `%date%` and `%time%` are **locale-dependent strings**. This lesson shows the portable ways to get a timestamp and the real limits of `set /a`.

## The problem with `%date%` and `%time%`

```bat
echo %date%      &rem  "Mon 09/07/2026"  or  "07.09.2026"  or  "2026-09-07" ...
echo %time%      &rem  "14:05:09.42"      or  " 9:05:09,42"  (note leading space!)
```

The format depends on the user's Region settings. A script that does `%date:~6,4%` to grab the year works on *your* machine and breaks on a colleague's. **Never parse `%date%`/`%time%` by fixed offsets in a script you'll share.**

## The portable timestamp: `wmic` / `PowerShell`

### `wmic` (works on most Windows, but deprecated — see [Advanced Lesson 4](../advanced/04-wmic-where-powershell.md))

```bat
for /F "usebackq skip=1 tokens=1-6" %%a in (`wmic path Win32_LocalTime get Day^,Hour^,Minute^,Month^,Second^,Year /format:table`) do (
    if not "%%f"=="" (
        set /a Y=%%f, M=1%%d-100, D=1%%a-100, hh=1%%b-100, mm=1%%c-100, ss=1%%e-100
    )
)
```

Simpler and very common:

```bat
for /F "usebackq tokens=1-6 delims=/-:. " %%a in (`wmic os get localdatetime ^| findstr ^[0-9]`) do (
    set "stamp=%%a"
)
rem stamp = 20260907140509.123456+000 ; slice the first 14: YYYYMMDDHHMMSS
set "stamp=%stamp:~0,14%"
echo %stamp%
```

### PowerShell (recommended on modern Windows)

```bat
for /F "usebackq delims=" %%t in (`powershell -NoProfile -Command "Get-Date -Format yyyy-MM-dd_HH-mm-ss"`) do set "TS=%%t"
echo Log file: app_%TS%.log
```

One clean line, locale-proof, any format you want. The cost is ~150 ms of PowerShell startup.

## A locale-independent trick using `%date%` tokens by name

If you can't spawn PowerShell, get the *order* of fields from the registry once, or use this common (imperfect) approach — parse by delimiter, not offset, and hope the field order matches:

```bat
for /F "tokens=1-4 delims=/-. " %%a in ("%date%") do (
    rem This still assumes an order. Better: use the WMIC/PS method above.
)
```

Honestly: **use the `wmic os get localdatetime` or PowerShell method.** Don't hand-roll date parsing.

## The leading-space in `%time%`

Before 10:00, `%time%` is `" 9:05:09.42"` — a leading space. This breaks `set /a` and filenames:

```bat
set "t=%time: =0%"        &rem replace space with 0 -> "09:05:09.42"
```

## `set /a` — the math you actually get

- **32-bit signed integers only.** Range −2,147,483,648 to 2,147,483,647. Overflow wraps silently.
- **No floating point.** `7/2` = `3`. Scale up: work in cents, or multiply by 100/1000 and format the result yourself.
- **Integer division truncates toward zero.**

```bat
set /a a = 17
set /a b = 5
set /a q = a / b            &rem 3
set /a r = a %% b           &rem 2
set /a pow = 2 * 2 * 2      &rem no ** operator; multiply
set /a neg = -a
set /a bit = (a ^& 1)       &rem bitwise AND: 1 if odd
set /a sh = 1 << 10         &rem 1024
```

### Number bases

```bat
set /a hex = 0x1F           &rem 31
set /a oct = 0755           &rem 493  -- leading zero = OCTAL
set /a bad = 08             &rem ERROR: 8 is not a valid octal digit
```

This bites when slicing time/date fields: `09` and `08` look decimal but `set /a` reads them as octal. Force decimal by prefixing `1` and subtracting `100`:

```bat
set "mm=08"
set /a minutes = 1%mm% - 100     &rem 8, safely
```

### Operators and compound assignment

`+ - * / %` `& | ^ ~ << >>` and `= *= /= %= += -= &= ^= |= <<= >>=`. Group with `( )`. Separate multiple assignments with commas:

```bat
set /a "x=1, y=2, sum=x+y"
```

### Faking decimals

```bat
rem Percentage with 1 decimal place: part=27, whole=40  -> 67.5%
set /a "scaled = part * 1000 / whole"     &rem 675
set /a "int = scaled / 10, frac = scaled %% 10"
echo %int%.%frac%%%
```

## Date arithmetic (days between dates)

Convert each date to a day number (e.g. via the "serial date" / Julian-day formula), subtract, done. It's fiddly in `set /a`. For anything real, use PowerShell:

```bat
for /F %%d in (`powershell -NoProfile -Command "((Get-Date) - (Get-Date '2026-01-01')).Days"`) do set "age=%%d"
echo %age% days since Jan 1
```

"Delete files older than N days" doesn't need date math at all — `forfiles`:

```bat
forfiles /p "C:\backups" /s /m *.zip /d -30 /c "cmd /c del @path"
```

`/d -30` = last modified more than 30 days ago.

## Common mistakes

- **Parsing `%date%` by fixed character offsets** — breaks on other locales.
- **`%time%` before 10 AM** having a leading space — breaks filenames and `set /a`.
- **`set /a` on `08` / `09`** — octal error. Use the `1%x%-100` trick.
- **Expecting floats** from `set /a`.
- **Silent 32-bit overflow** — e.g. bytes of a large drive, or `factorial(13)`.
- **`%%` at the prompt vs in a script** — modulo is `%` interactively, `%%` in a `.bat`.
- **`forfiles` date format in `/d`** — `-30` (days ago) or an absolute `MM/dd/yyyy` in *your* locale.
- **`wmic` missing** on Windows 11 24H2+ (it's being removed) — prefer PowerShell.

## Exercises

1. Produce a filename-safe timestamp `YYYY-MM-DD_HH-MM-SS` using the `wmic os get localdatetime` method. Then do it with PowerShell and compare.
2. Fix a `%time%`-derived value that has a leading space so `set /a` accepts the hour.
3. Compute `13!` and observe the 32-bit overflow. At what n does it break?
4. Given `part` and `whole`, print a percentage with one decimal place using integer scaling.
5. Use `forfiles` to list (don't delete) every `.log` under a folder older than 7 days.
6. Extract just the hour from `%time%` safely (handle both `09:...` and `14:...`), and print `Good morning`/`Good afternoon`/`Good evening`.
7. Write `:days_since YYYY-MM-DD outvar` using PowerShell; use it to compute your age in days.

## Recap

- `%date%`/`%time%` are locale-dependent strings — don't parse by offset in shared scripts.
- Portable timestamp: `wmic os get localdatetime` (deprecated) or `powershell Get-Date -Format ...` (preferred).
- `%time%` has a leading space before 10:00 — replace it: `%time: =0%`.
- `set /a` is 32-bit signed integer only: no floats, silent overflow, leading-zero = octal (use `1%x%-100`).
- Fake decimals by scaling; do real date math in PowerShell; use `forfiles /d -N` for age-based file cleanup.

Next: [User interaction & menus →](08-user-interaction-and-menus.md)
