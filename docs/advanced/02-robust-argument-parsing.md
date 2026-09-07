# 2. Robust Argument Parsing

## What & why

The `%1 %2 %3` model breaks down as soon as you want optional flags, `--name value` pairs, `/opt:value` syntax, repeated options, or positional args mixed with flags. This lesson builds a parser you can drop into any script, and covers the quoting minefield underneath it.

## The target CLI

```
mytool [options] <input> [<input> ...]

  -v, --verbose            enable verbose output
  -q, --quiet              suppress non-error output
  -o, --output DIR         output directory (default: .)
  --format {json|csv|xml}  output format (default: json)
  --retries N              retry count (default: 3)
  --force                  overwrite existing files
  -h, --help               show help
```

## The parse loop

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion

rem ---- defaults ----
set "VERBOSE="
set "QUIET="
set "OUTDIR=."
set "FORMAT=json"
set "RETRIES=3"
set "FORCE="
set "INPUTS="
set "NINPUTS=0"

:argloop
if "%~1"=="" goto argsdone
set "arg=%~1"

if "!arg!"=="-h"          goto help
if "!arg!"=="--help"      goto help
if "!arg!"=="-v"          ( set "VERBOSE=1" & shift & goto argloop )
if "!arg!"=="--verbose"   ( set "VERBOSE=1" & shift & goto argloop )
if "!arg!"=="-q"          ( set "QUIET=1"   & shift & goto argloop )
if "!arg!"=="--quiet"     ( set "QUIET=1"   & shift & goto argloop )
if "!arg!"=="--force"     ( set "FORCE=1"   & shift & goto argloop )

if "!arg!"=="-o"          ( set "OUTDIR=%~2"  & shift & shift & goto argloop )
if "!arg!"=="--output"    ( set "OUTDIR=%~2"  & shift & shift & goto argloop )
if "!arg!"=="--format"    ( set "FORMAT=%~2"  & shift & shift & goto argloop )
if "!arg!"=="--retries"   ( set "RETRIES=%~2" & shift & shift & goto argloop )

rem ---- --key=value form ----
if "!arg:~0,2!"=="--" (
    for /F "tokens=1* delims==" %%k in ("!arg!") do (
        set "k=%%k" & set "v=%%l"
    )
    if defined v (
        if /i "!k!"=="--output"  ( set "OUTDIR=!v!"  & shift & goto argloop )
        if /i "!k!"=="--format"  ( set "FORMAT=!v!"  & shift & goto argloop )
        if /i "!k!"=="--retries" ( set "RETRIES=!v!" & shift & goto argloop )
    )
    echo ERROR: unknown option "!arg!" 1>&2
    exit /b 1
)

rem ---- lone dash-prefixed = unknown ----
if "!arg:~0,1!"=="-" (
    echo ERROR: unknown option "!arg!" 1>&2
    exit /b 1
)

rem ---- otherwise: a positional input ----
set /a NINPUTS+=1
set "INPUT[!NINPUTS!]=!arg!"
shift
goto argloop

:argsdone
if %NINPUTS%==0 ( echo ERROR: at least one input required 1>&2 & exit /b 1 )

rem ---- validate enums / numbers ----
if /i not "%FORMAT%"=="json" if /i not "%FORMAT%"=="csv" if /i not "%FORMAT%"=="xml" (
    echo ERROR: --format must be json^|csv^|xml 1>&2 & exit /b 1
)
set /a _r=RETRIES 2>nul
if not "%_r%"=="%RETRIES%" ( echo ERROR: --retries must be a number 1>&2 & exit /b 1 )

rem ---- use them ----
if defined VERBOSE echo verbose=on outdir="%OUTDIR%" format=%FORMAT% retries=%RETRIES% force=%FORCE%
for /L %%i in (1,1,%NINPUTS%) do echo input %%i: "!INPUT[%%i]!"
exit /b 0

:help
echo mytool [options] ^<input^> [^<input^> ...]
echo   -v/--verbose  -q/--quiet  -o/--output DIR  --format json^|csv^|xml
echo   --retries N   --force     -h/--help
exit /b 0
```

## Key techniques

### `shift` per consumed token

A boolean flag consumes 1 (`shift`), a `--key value` pair consumes 2 (`shift & shift`), and `--key=value` consumes 1.

### `%~1` everywhere

`"%~1"` strips one layer of quotes so `"C:\Program Files"` compares and stores correctly. Re-quote when you *use* the value: `md "%OUTDIR%"`.

### `--key=value` splitting

`for /F "tokens=1* delims==" %%k in ("--retries=5")` gives `%%k=--retries`, `%%l=5`. Guard against a value that itself contains `=` (only the first `=` splits, which is what you want for `--filter=a=b`).

### Number/enum validation after parsing, not during

Collect everything first, validate once. `set /a _r=RETRIES` + compare is the integer check ([Intermediate Lesson 10](../intermediate/10-capstone-backup-utility.md)).

### `--` end-of-options marker (optional, nice)

```bat
if "!arg!"=="--" (
    shift
    :restpositional
    if "%~1"=="" goto argsdone
    set /a NINPUTS+=1 & set "INPUT[!NINPUTS!]=%~1" & shift & goto restpositional
)
```

## The quoting minefield

| You type | `%1` is | `%~1` is |
| --- | --- | --- |
| `foo` | `foo` | `foo` |
| `"foo bar"` | `"foo bar"` | `foo bar` |
| `"C:\a\"` | `"C:\a\"` → trailing `\"` can escape the quote! | often `C:\a"` — messy |
| `a=b` | `a=b` | `a=b` |
| `a&b` (unquoted) | parser runs `b` as a command! | — |

Rules of thumb:

- Tell users to quote paths, and **never** end a quoted path with `\` (`"C:\dir\"` → use `"C:\dir"`).
- Inside the script, `set "X=%~1"` (quoted assignment, stripped value), use as `"%X%"`.
- `%*` preserves the original quoting for pass-through: `realtool.exe %*`.
- Args with `!` while delayed expansion is on get mangled — parse with delayed expansion *off*, or copy `%1` via `%%~x` in a `for` first.

### Delayed-expansion-safe argument capture

```bat
setlocal DisableDelayedExpansion
set "raw=%~1"
setlocal EnableDelayedExpansion
rem now work with !raw!, knowing the ! in it is preserved as data
```

## A reusable mini-parser (positional + `--flags` only)

For simpler tools, this is enough:

```bat
set "FLAGS= "
set "POS="
:p
if "%~1"=="" goto pd
set "a=%~1"
if "!a:~0,2!"=="--" ( set "FLAGS=!FLAGS!!a:~2! " ) else ( set "POS=!POS! "%~1"" )
shift & goto p
:pd
rem test a flag:  if "!FLAGS: verbose =!" neq "!FLAGS!" echo verbose set
```

## Common mistakes

- **Not `shift`ing the value** of a `--key value` pair — the value gets re-parsed as the next option.
- **Comparing `%1` unquoted** — empty or space-containing args → syntax error.
- **Trailing `\` in a quoted path arg** — `"C:\dir\"` escapes the closing quote; everything after shifts.
- **Validating inside the loop** — leads to order-dependent bugs; validate after.
- **`--format=` empty value** — `if defined v` guard, or default it.
- **Case sensitivity** — use `/i` on option compares if you want `--FORMAT` to work (or don't, and document it).
- **`!` in an argument** with delayed expansion on — parse args with it off.
- **Forgetting `%*` doesn't reflect `shift`** — build your own pass-through list if you've consumed some.

## Exercises

1. Implement the full `mytool` parser above and test: `-vo out --format csv a.txt b.txt`, `--retries=5 x`, `--output "C:\My Out" y`, `--bogus`, no inputs.
2. Add support for combined short flags: `-vq` = `-v -q`.
3. Add a `--` end-of-options marker so `mytool -- --weird-filename.txt` treats it as input.
4. Add `--config FILE` that loads defaults from an INI, with command-line args overriding.
5. Make every `--key value` also accept `--key=value` via a single helper subroutine instead of repeating `for /F` inline.
6. Write a torture test: a filename with a space, an `&`, a `!`, and a `%`. Get it through the parser intact and print it back byte-identical.

## Recap

- Parse loop: check `%~1` against each option, `shift` once per consumed token (twice for `--key value`), fall through to "positional input."
- Support `--key value` and `--key=value` (split on first `=` with `for /F "tokens=1* delims=="`).
- Store positionals in a pseudo-array; validate enums/numbers *after* the loop.
- `%~1` to compare/store, `"%VAR%"` to use, `%*` for pass-through.
- Never let a quoted path end in `\`; parse args with delayed expansion off if `!` may appear.

Next: [Working with the registry →](03-working-with-the-registry.md)
