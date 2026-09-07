# 5. Text Processing with `findstr`

## What & why

`findstr` is batch's grep: filter lines, match patterns, search files recursively. It's genuinely useful and genuinely weird — its regex dialect is tiny and non-standard, its quoting rules are broken in documented ways, and a few switches do the opposite of what you'd guess. Learn its real behaviour and you can do a lot without leaving batch.

## `find` vs `findstr`

| | `find` | `findstr` |
| --- | --- | --- |
| Literal substring | yes | yes |
| Regex | no | yes (limited) |
| Case-insensitive | `/i` | `/i` |
| Multiple patterns | no | yes (`/c:` repeated, or space-separated, or `/g:file`) |
| Recursive | no | `/s` |
| Count | `/c` | `/c` via `... | find /c /v ""` trick, or per-file |
| Line numbers | `/n` | `/n` |

Use `find` for dead-simple literal matches and counting; `findstr` for anything with a pattern or recursion.

## `findstr` basics

```bat
findstr "error" app.log                 &rem lines containing "error" (as a regex! see below)
findstr /i "error" app.log              &rem case-insensitive
findstr /v "DEBUG" app.log              &rem lines NOT containing DEBUG
findstr /n "error" app.log              &rem prefix matches with line numbers
findstr /c:"exact phrase" app.log       &rem literal string, spaces and all
findstr /s /i /m "TODO" *.cs            &rem recurse, list only file NAMES with a match
```

### `/m` — filenames only

```bat
for /F "delims=" %%f in ('findstr /s /m /c:"NotImplementedException" "*.cs"') do echo needs work: %%f
```

### Multiple patterns

```bat
findstr "warn error fatal" app.log             &rem OChR of the three (space = alternation of terms)
findstr /c:"disk full" /c:"out of memory" x.log  &rem OR of two literal phrases
findstr /g:patterns.txt app.log                  &rem patterns, one per line, from a file
```

!!! warning "Space-separated terms are an OR of *regexes*, not literals"
    `findstr "a.b c"` matches lines containing `a<any>b` **or** `c`. If you want the literal string `a.b c`, use `/c:"a.b c"` and `/l` (literal): `findstr /l /c:"a.b c"`.

## The `findstr` regex dialect

It is **not** PCRE, POSIX, or `.NET`. The entire supported set:

| Pattern | Meaning |
| --- | --- |
| `.` | any character |
| `*` | zero or more of the **preceding** character/class |
| `^` | start of line |
| `$` | end of line |
| `[abc]` | character class |
| `[^abc]` | negated class |
| `[a-z]` | range |
| `\<` | start of word |
| `\>` | end of word |
| `\x` | escape a metacharacter (`\.` `\*` `\[` ...) |

That's it. **No `+`, no `?`, no `{n,m}`, no `|` alternation** (use space-separated terms or multiple `/c:`), no groups, no backreferences, no `\d`/`\w`/`\s` (use `[0-9]`, `[A-Za-z0-9_]`, `[ ]`).

```bat
rem A line that is only digits:
findstr /r "^[0-9][0-9]*$" data.txt

rem Lines starting with a date like 2026-09-07:
findstr /r "^[12][0-9][0-9][0-9]-[01][0-9]-[0-3][0-9]" log.txt

rem The whole word "test" (not "testing"):
findstr /r "\<test\>" file.txt

rem IPv4-ish (crude):
findstr /r "[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*" access.log
```

`/r` forces regex mode; `/l` forces literal. Without either, `findstr` guesses (and treats `/c:` as literal, bare args as regex).

## Known bugs — memorize these

`findstr` has officially documented broken behaviour:

1. **`/f:file` line length**: lines over 8191 bytes cause it to break or hang.
2. **Backslash before non-metacharacter**: `\d` isn't "digit," but `\.` is a literal dot. `\q` is just `q`. Fine — but `\\` inside `[...]` is inconsistent.
3. **Quoting**: `findstr "a b"` — if you actually want to search for the literal two-word phrase you must use `/c:`. Bare quotes = OR of terms.
4. **`/i` with `/r` and character ranges**: `[A-Z]` with `/i` can match unexpectedly due to ASCII ordering (`[A-z]` catches `[ \ ] ^ _ ``).
5. **Piped input encoding**: `findstr` matching Unicode from a pipe is unreliable; it's an OEM-codepage tool.
6. **Leading `/` in a search string**: `findstr /c:"/foo"` is fine, but `findstr "/foo"` tries to parse `/foo` as a switch.
7. **Period as literal in a bare (non-`/r`) search still acts as regex `.`** unless you use `/l` or `/c:` (and even `/c:` is regex unless `/l`). When in doubt: `/l /c:"..."`.

## Practical recipes

### Count matching lines

```bat
findstr /r /c:"ERROR" app.log | find /c /v ""
```

`find /c /v ""` counts lines that do NOT contain the empty string — i.e. all of them.

### Filter a pipeline

```bat
tasklist /v /fo csv | findstr /i /c:"chrome.exe" | find /c /v ""
```

### Extract a field after a match

```bat
for /F "tokens=2 delims=:" %%v in ('ipconfig ^| findstr /r /c:"IPv4 Address"') do (
    for /F "tokens=* delims= " %%w in ("%%v") do echo IP is %%w
)
```

### "grep -A" (line after match) — not built in

`findstr` has no context option. Number the lines, find the match's number, then pull `N+1`:

```bat
for /F "tokens=1 delims=:" %%n in ('findstr /n /c:"MARKER" file.txt') do set /a want=%%n+1
for /F "tokens=1* delims=:" %%a in ('findstr /n "^" file.txt') do if "%%a"=="%want%" echo %%b
```

(This is where you start wishing you were in PowerShell — [Lesson 4](04-wmic-where-powershell.md).)

### Search-and-replace in a file

`findstr` can't replace. Options:

```bat
rem Small files, simple replace, pure batch (line by line, delayed expansion):
setlocal EnableDelayedExpansion
(for /F "usebackq delims=" %%L in ("in.txt") do (
    set "line=%%L"
    set "line=!line:OLD=NEW!"
    echo(!line!
)) > out.txt
```

Caveats: drops blank lines (use the `findstr /n "^"` prefix trick), mangles `!` and sometimes `^`, and is slow. For anything real:

```bat
powershell -NoProfile -Command "(Get-Content 'in.txt') -replace 'OLD','NEW' | Set-Content 'out.txt'"
```

## Common mistakes

- **`findstr "a.b"`** matching `axb` because `.` is regex even without `/r`. Use `/l /c:"a.b"`.
- **Expecting `|` alternation** — not supported. Space-separate terms or repeat `/c:`.
- **Expecting `+`, `?`, `{2,3}`, `\d`** — none exist.
- **`findstr /c:"has spaces"` without it** — bare quotes are an OR of terms, not a phrase.
- **Matching on huge lines** — 8KB limit, then it breaks.
- **Unicode/UTF-16 files** — garbled matches; convert to ANSI/UTF-8 first.
- **`[A-z]`** thinking it means "letters" — it also includes 6 punctuation chars.
- **Using `findstr` to edit** — it only filters; replace needs the `for /F` loop or PowerShell.
- **`/s` from the wrong directory** — it recurses from the *current* dir; `pushd` first or give a path.

## Exercises

1. From a log file, print only lines that start with a timestamp `HH:MM:SS` (regex).
2. Count how many lines contain `ERROR` **or** `FATAL` (two `/c:` patterns) and print the count.
3. List (filenames only) every `.md` file under `docs\` that contains the word `TODO` as a whole word.
4. Extract every unique IPv4 address from an access log (`findstr /r` to filter, `for /F` to isolate, a pseudo-set to dedupe).
5. Replace `http://` with `https://` throughout a file — once with the pure-batch loop (note what it does to blank lines), once with the PowerShell one-liner. Compare outputs with `fc`.
6. Reproduce the `[A-z]` bug: search a file containing `_`, `[`, `\` with `findstr /r "[A-z]"` and observe the punctuation matches.
7. Implement a crude "grep -B1" (line *before* each match) using line numbering.

## Recap

- `find` = literal + counting; `findstr` = regex, recursion (`/s`), filenames-only (`/m`), multiple patterns (`/c:` ×N or `/g:`).
- `findstr` regex is tiny: `.  *  ^  $  [ ]  [^ ]  [a-z]  \<  \>  \x` — **no `+ ? {} | \d \w \s` or groups**.
- Bare quoted terms = OR of regexes; use `/l /c:"..."` for a literal phrase.
- Known bugs: 8KB line limit, `.` is regex even without `/r`, `[A-z]` over-matches, Unicode from pipes is unreliable.
- `findstr` can't replace — loop with `!line:OLD=NEW!` (lossy) or shell out to PowerShell `-replace`.

Next: [Performance & pitfalls →](06-performance-and-pitfalls.md)
