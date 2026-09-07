# 3. `for /F` Deep Dive

## What & why

`for /F` is how batch reads and parses text: files, strings, and — most importantly — the output of other commands. It's how you capture command results into variables, which the language otherwise can't do. It has five options and several sharp edges.

## The three sources

```bat
rem 1. A file (or files):
for /F %%L in (data.txt) do ...

rem 2. A literal string (in quotes):
for /F %%L in ("one two three") do ...

rem 3. A command's output (in single quotes):
for /F %%L in ('dir /b *.txt') do ...
```

## The options string

```bat
for /F "tokens=1,3 delims=,; skip=1 eol=# usebackq" %%a in (...) do ...
```

| Option | Meaning |
| --- | --- |
| `delims=xyz` | characters that separate tokens (default: space + tab) |
| `tokens=n,m` | which token(s) to hand to the loop variable(s) |
| `skip=n` | ignore the first `n` lines |
| `eol=c` | lines *starting* with `c` are treated as comments and skipped (default: `;`) |
| `usebackq` | change the quoting rules (see below) |

### `tokens=`

Each requested token goes into a **successive loop variable**, starting from the one you named:

```bat
rem name is %%a, so token 1->%%a, token 3->%%b
for /F "tokens=1,3 delims=," %%a in ("alice,30,london") do echo %%a in %%b

rem ranges:
for /F "tokens=1-3" %%a in ("x y z w") do echo %%a %%b %%c    &rem x y z

rem "the rest": * grabs everything after the last numbered token
for /F "tokens=1,2*" %%a in ("verb noun the rest of the line") do (
    echo verb=%%a noun=%%b rest=%%c
)
```

`tokens=*` = the whole line with **leading delimiters stripped**. `tokens=1,*` = first token, then the remainder including its internal spacing.

### `delims=`

```bat
rem Split on comma only:
for /F "tokens=1-3 delims=," %%a in ("a, b ,c") do echo [%%a][%%b][%%c]
rem -> [a][ b ][c]   (spaces are NOT delimiters here — only comma is)

rem Split on comma OR semicolon OR space:
for /F "tokens=1-3 delims=,; " %%a in ("a;b c") do echo %%a %%b %%c
```

**`delims=` must usually be last in the options string** (a trailing space in `delims= ` would otherwise swallow the next option). To split on nothing (whole line, no tokenising), use `delims=` as the final option:

```bat
for /F "usebackq delims=" %%L in ("file with spaces.txt") do echo LINE: %%L
```

### `skip=` and `eol=`

```bat
rem Skip a header row:
for /F "skip=1 tokens=1,2 delims=," %%a in (people.csv) do echo %%a=%%b

rem Ignore lines starting with #:
for /F "eol=# tokens=1,2 delims== " %%k in (config.ini) do set "cfg_%%k=%%l"
```

`eol` is a *single* character and only matches at the very start of a line. There's no multi-char or mid-line comment support.

### `usebackq`

Changes quoting so you can use quoted filenames and still run commands:

| Without `usebackq` | With `usebackq` |
| --- | --- |
| `("literal string")` | `('literal string')` ... no: string is `"..."`, still |
| `('command')` | `` (`command`) `` — backticks run a command |
| `(filename)` unquoted only | `("quoted filename")` allowed |

Practical rule: **use `usebackq` whenever your filename needs quotes**, and then run commands with backticks:

```bat
for /F "usebackq delims=" %%L in ("C:\My Files\list.txt") do echo %%L
for /F "usebackq tokens=2" %%a in (`hostname`) do set "host=%%a"
```

## Capturing command output into a variable

The idiom you'll use constantly:

```bat
for /F "delims=" %%v in ('git rev-parse --short HEAD') do set "commit=%%v"
echo Building commit %commit%
```

Multi-line output — last line wins unless you accumulate:

```bat
set "n=0"
for /F %%v in ('tasklist /fi "imagename eq notepad.exe" /nh') do set /a n+=1
```

### Escaping inside `('...')`

The command runs through `cmd`, so `|`, `>`, `<`, `&` inside must be **caret-escaped**:

```bat
for /F "tokens=2 delims=:" %%a in ('ipconfig ^| findstr /i "IPv4"') do set "ip=%%a"
for /F "delims=" %%f in ('dir /b /s *.log ^| findstr /v "archive"') do echo %%f
```

And `%` inside the command must be `%%` (script) as usual.

## Reading a file safely

```bat
setlocal EnableDelayedExpansion
set "count=0"
for /F "usebackq tokens=* delims=" %%L in ("%~dp0input.txt") do (
    set /a count+=1
    set "line[!count!]=%%L"
)
echo Read !count! lines.
```

!!! warning "Empty lines are skipped"
    `for /F` **silently ignores blank lines**. If line numbers matter, pipe through `findstr /n "^"` to prefix every line with `N:` (which also protects lines that would otherwise be skipped), then strip the prefix:
    ```bat
    for /F "usebackq delims=" %%L in (`findstr /n "^" "%~dp0input.txt"`) do (
        set "raw=%%L"
        set "raw=!raw:*:=!"      &rem drop the "N:" prefix
        echo !raw!
    )
    ```

## Common mistakes

- **`'single quotes'` vs `"double"` vs `(bare)`** — single = run a command, double = literal string, bare = filename. Mixing them up is the #1 error.
- **Quoted filename without `usebackq`** — treated as a literal string to parse, not a file.
- **`delims=` not last** — options after it get eaten.
- **Blank lines vanishing** — use the `findstr /n "^"` trick.
- **Lines starting with `;`** disappearing because that's the default `eol`. Set `eol=` to something impossible, or accept it.
- **Unescaped `|` / `>` inside `('...')`** — `The process tried to write to a nonexistent pipe` or worse.
- **`tokens=1,2,3` giving you `%%a %%b %%c`** but you only declared `%%a` — that's fine, it auto-allocates `%%b`, `%%c`; just don't collide with an outer loop's letters.
- **Long lines** — `for /F` truncates lines at ~8192 bytes.
- **Unicode / UTF-16 files** — `for /F` expects ANSI/OEM; UTF-16 comes through as garbage. Convert first (`type` won't help; use PowerShell).

## Exercises

1. Read `config.ini` (lines like `key = value`, `#` comments, blank lines) into variables named `cfg_<key>`. Print them all afterward.
2. Capture `whoami /groups` and count how many groups you're in.
3. Parse `ipconfig` output to extract your IPv4 address and default gateway into two variables.
4. Given a CSV with a header, `skip=1` it and print `row N: <col1> / <col3>` for each data row, with correct row numbers (mind the blank-line problem).
5. Read a file preserving blank lines and exact content using the `findstr /n "^"` technique; write it back out to a copy and `fc` the two files.
6. Use `for /F` with `usebackq` and a backtick command to store `git branch --show-current` in a variable; handle the "not a git repo" case with `2^>nul` and a fallback.

## Recap

- Sources: `(file)`, `("string")`, `('command')` — and with `usebackq`: `("file")`, `('string')`, `` (`command`) ``.
- Options: `tokens=` (which fields, `*` = the rest), `delims=` (separators, put it **last**), `skip=` (header lines), `eol=` (comment char), `usebackq` (quoting mode).
- `for /F ... in ('cmd') do set "x=%%v"` is how you capture command output.
- Escape `| > < &` as `^| ^> ^< ^&` inside `('...')`.
- Blank lines are dropped — use `findstr /n "^"` when they matter.

Next: [Error handling & exit codes →](04-error-handling-exit-codes.md)
