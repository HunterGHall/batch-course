# 2. String Manipulation

## What & why

Batch has no string library, but variable expansion has two built-in features — **substring** and **substitution** — that cover most needs. They only work on `%var%` / `!var!` expansion, not on literals or `%1` directly (copy an arg into a variable first).

## Substring: `%var:~start,length%`

```bat
set "s=Hello, World"

echo %s:~0,5%      &rem Hello        (from index 0, take 5)
echo %s:~7%        &rem World        (from index 7 to the end)
echo %s:~-5%       &rem World        (last 5 characters)
echo %s:~0,-1%     &rem Hello, Worl  (everything except the last char)
echo %s:~-5,3%     &rem Wor          (3 chars, starting 5 from the end)
echo %s:~3,-2%     &rem lo, Wor      (from index 3, stop 2 before the end)
```

Rules:

- Index is **0-based**.
- Negative start counts from the end.
- Negative length means "stop that many chars before the end."
- Out-of-range gives an empty string, not an error.

### Practical uses

```bat
rem First character:
set "first=%s:~0,1%"

rem Zero-pad a number to 2 digits:
set "n=7"
set "nn=0%n%"
set "nn=%nn:~-2%"           &rem "07"; works for 7 -> 07 and 12 -> 12

rem Strip a known prefix:
set "path=C:\proj\src\main.c"
set "rel=%path:~8%"          &rem "src\main.c"  (if you know the prefix length)
```

## Substitution: `%var:find=replace%`

```bat
set "s=one two two three"

echo %s:two=X%              &rem one X X three   (replaces ALL occurrences)
echo %s: =_%               &rem one_two_two_three
echo %s:two=%              &rem one   three      (delete by replacing with nothing)
```

### Anchored substitution

- `%var:*text=replace%` — the `*` matches everything from the start up to and including `text`:

```bat
set "url=https://example.com/path"
echo %url:*//=%            &rem example.com/path   (drop protocol)

set "line=KEY=some value"
echo %line:*==%            &rem "some value"       (everything after the first =)
```

There is no built-in "replace only at the end" — reverse the string, or use `findstr`/`for /F`.

## Length of a string

No direct function. Loop:

```bat
setlocal EnableDelayedExpansion
set "s=Hello, World"
set "len=0"
:len_loop
if not "!s:~%len%,1!"=="" (
    set /a len+=1
    goto len_loop
)
echo Length: %len%
```

Or a faster binary-search version — see exercises.

## Case conversion

No built-in. Common approaches:

```bat
rem Brute force (ASCII letters only):
set "s=Hello"
for %%A in ("a=A" "b=B" "c=C" "d=D" "e=E" "f=F" "g=G" "h=H" "i=I" "j=J" "k=K" "l=L" "m=M" "n=N" "o=O" "p=P" "q=Q" "r=R" "s=S" "t=T" "u=U" "v=V" "w=W" "x=X" "y=Y" "z=Z") do (
    call set "s=%%s:%%~A%%"
)
echo %s%          &rem HELLO
```

Or just shell out to PowerShell for one line ([Advanced Lesson 4](../advanced/04-wmic-where-powershell.md)):

```bat
for /F "delims=" %%U in ('powershell -NoProfile -Command "'Hello'.ToUpper()"') do set "upper=%%U"
```

## Trimming whitespace

```bat
rem Trim leading spaces:
set "s=    hello"
for /F "tokens=* delims= " %%A in ("%s%") do set "s=%%A"
echo [%s%]

rem Trim trailing spaces (loop off one at a time):
:rtrim
if "%s:~-1%"==" " set "s=%s:~0,-1%" & goto rtrim
```

## Split on a delimiter

```bat
set "csv=alice,30,london"
for /F "tokens=1,2,3 delims=," %%a in ("%csv%") do (
    echo name=%%a age=%%b city=%%c
)
```

For an unknown number of fields, use `tokens=1*` repeatedly or a `for` over a delimiter-swapped string:

```bat
set "list=a;b;c;d"
set "list=%list:;= %"
for %%x in (%list%) do echo item: %%x
```

(Careful: this breaks if items contain spaces.)

## Contains / starts-with / ends-with

```bat
rem contains:
if not "%s%"=="%s:World=%" echo contains "World"

rem starts with "http":
if /i "%s:~0,4%"=="http" echo starts with http

rem ends with ".txt":
if /i "%s:~-4%"==".txt" echo is a text file
```

## Common mistakes

- **Substring/substitution on `%1` directly** — `%1:~0,3%` doesn't work. `set "a=%~1"` first.
- **`%var:~0,-0%`** — `-0` is `0`, so length 0 → empty. Use a positive length or omit it.
- **Substitution is case-insensitive for the *match*** — `%s:hello=hi%` replaces `Hello` too. (Replacement text keeps your casing.)
- **A `=` or `:` or `*` inside your search text** — these are special in the `:find=replace` syntax and can't be searched literally without tricks.
- **`!` in the string with delayed expansion on** — see [Lesson 1](01-delayed-expansion.md).
- **Assuming there's a trim built-in** — there isn't.
- **`for %%x in (%list%)`** splitting on spaces *and* commas *and* tabs — all are `for` delimiters in that mode.

## Exercises

1. Given `set "file=2026-09-07_report.txt"`, extract the date `2026-09-07` and the label `report` using substrings and substitution.
2. Zero-pad the numbers 1, 9, 10, 100 to width 3 (`001`, `009`, `010`, `100`).
3. Write `:strlen str outvar`. Test on the empty string, `"a"`, and a 50-char string.
4. Parse `KEY=VALUE WITH SPACES` into `key` and `value`, keeping the spaces in the value (use `%line:*==%`).
5. Convert a path like `C:/Users/me/file.txt` to `C:\Users\me\file.txt` with one substitution.
6. Write `:trim str outvar` that removes leading and trailing spaces. Verify with `[` `]` markers.
7. Given `a,b,,d` (note the empty field), split on `,` and print each field with its index. Notice what `for /F` does with the empty field.

## Recap

- Substring: `%var:~start,len%` — 0-based, negatives count from the end, out-of-range = empty.
- Substitute: `%var:find=replace%` (all occurrences, case-insensitive match); delete with `find=`; `%var:*x=%` drops everything through the first `x`.
- These work on variable expansion only — copy `%1` into a variable first.
- No built-in length, case, or trim — loop, use a `for` table, or call PowerShell.

Next: [for /F deep dive →](03-for-f-deep-dive.md)
