# 8. Redirection & Pipes

## What & why

Redirection sends command output to a file instead of the screen; pipes send it to another command. Batch scripts use these constantly — to build files, suppress noise, capture results, and log. The stream numbers (`1` for stdout, `2` for stderr) are where it gets fiddly.

## The three streams

| Number | Name | Default |
| --- | --- | --- |
| `0` | stdin | keyboard |
| `1` | stdout | screen |
| `2` | stderr | screen |

## Writing to a file

```bat
echo Starting > run.log            &rem > overwrites (truncates first)
echo Step 1 >> run.log             &rem >> appends
dir /b *.txt >> run.log
```

`>` always starts fresh; `>>` adds to the end (and creates the file if missing).

## The null device

`nul` is a magic name that discards everything written to it:

```bat
ping -n 1 example.com >nul                &rem hide normal output
copy a.txt b.txt >nul                     &rem hide "1 file(s) copied."
where git >nul 2>nul && echo git present  &rem hide both streams, act on success
```

## Redirecting stderr

```bat
somecommand 2> errors.log                 &rem stderr to a file, stdout still on screen
somecommand 2>nul                          &rem discard errors only
somecommand > out.log 2> err.log           &rem split the streams
somecommand > out.log 2>&1                  &rem BOTH streams into out.log
somecommand 1>&2                            &rem send stdout to stderr
```

`2>&1` means "make stream 2 go wherever stream 1 currently goes." **Order matters:** `> out.log 2>&1` works (redirect 1 to the file, then point 2 at the same place). `2>&1 > out.log` does *not* combine them (2 gets pointed at the screen, *then* 1 moves to the file).

### Writing errors from your own script

```bat
echo ERROR: config not found 1>&2
exit /b 1
```

Sending your error messages to stderr lets callers separate them from real output.

## Reading from a file: `<`

```bat
rem Feed a file to a command's stdin:
sort < unsorted.txt > sorted.txt

rem Answer an interactive prompt:
del *.tmp < nul
```

`< nul` feeds an immediate end-of-input, which makes some prompting commands take their default or bail — occasionally useful, but prefer a real `/q` switch.

## Pipes

```bat
dir /b /s *.log | find /c /v ""          &rem count lines (= file count)
tasklist | findstr /i "chrome"           &rem filter process list
type big.txt | more                       &rem page through output
echo %PATH% | find /i "python"            &rem is python on PATH?
```

The left command's **stdout** becomes the right command's **stdin**. stderr is not piped unless you add `2>&1` on the left.

!!! warning "Each side of a pipe runs in its own `cmd.exe`"
    `something | something` spawns two subshells. Consequences:

    - Variables `set` inside a piped command **do not survive**: `echo x | set /p "v="` — `v` is set in the subshell and lost.
    - Delayed-expansion state and `setlocal` don't carry across.
    - It's slower than it looks (two process launches).

    Workarounds are in [Intermediate Lesson 3](../intermediate/03-for-f-deep-dive.md) (`for /F` to capture output) and [Advanced Lesson 6](../advanced/06-performance-and-pitfalls.md).

## Capturing command output into a variable

You can't do `set v=$(cmd)` like bash. Use `for /F`:

```bat
for /F "delims=" %%v in ('hostname') do set "host=%%v"
echo Running on %host%

for /F "tokens=2 delims=:" %%v in ('ipconfig ^| findstr /i "IPv4"') do set "ip=%%v"
echo IP:%ip%
```

Note the `^|` — inside the `for /F ('...')` quotes, pipes and redirects must be **caret-escaped**.

## Combining and grouping

```bat
(echo Line 1 & echo Line 2 & echo Line 3) > file.txt

(
    echo [config]
    echo debug=1
    echo path=%CD%
) > app.ini

command1 & command2                       &rem run both, unconditionally
command1 && command2                      &rem run command2 only if command1 succeeded
command1 || command2                      &rem run command2 only if command1 failed
```

## Special characters and escaping

To write a literal `>`, `<`, `|`, `&` to a file, escape with `^`:

```bat
echo Redirect with ^> and pipe with ^| >> notes.txt
```

Inside `for /F '...'` command blocks, the same applies to `|`, `>`, `<`.

## Common mistakes

- **`2>&1` before the `>`** — doesn't merge the streams. Put `2>&1` last.
- **Space in `> >file`** or between `2` and `>` (`2 > file`) — the `2` gets treated as an argument. Write `2>file` or `2> file` (space after `>` is fine, not before).
- **Expecting `set` inside a pipe to stick** — it runs in a subshell.
- **Piping when you meant to capture** — use `for /F` to get output into a variable.
- **`>` a file that's open/locked** — "The process cannot access the file." Close it, or write elsewhere.
- **Trailing space before `>>`**: `echo hi >> log` writes `hi ` (with the space). Use `echo hi>>log` or accept the space.
- **`>nul` not hiding errors** — that's stdout only; add `2>nul`.

## Exercises

1. Run a command that fails (e.g. `dir Z:\nope`) and send only its error text to `errors.log`, keeping stdout on screen. Then send both to one file.
2. Build `report.txt` in one grouped `( ... ) > report.txt` block: a title line, a blank line, and the output of `date /t` and `time /t`.
3. Count how many running processes contain "svchost" using `tasklist | findstr /i` piped into `find /c /v ""`.
4. Capture the output of `hostname` into a variable with `for /F` and print `This is <name>`.
5. Capture your IPv4 address into a variable (`ipconfig ^| findstr /i IPv4`, split on `:`), trimming the leading space.
6. Append a timestamped line to `run.log` every time the script runs: `echo %date% %time% run>>run.log`.

## Recap

- `>` overwrites, `>>` appends, `nul` discards.
- Streams: `1`=stdout, `2`=stderr. `2>&1` merges — and must come **after** the `>`.
- `<` feeds stdin from a file. `|` pipes stdout to the next command's stdin.
- Each pipe side is a separate `cmd.exe` — `set` inside a pipe is lost.
- Capture output into a variable with `for /F "..." %%v in ('command') do set "x=%%v"` (escape `|`/`>` as `^|`/`^>` inside).
- Group with `( ... )`; chain with `&`, `&&`, `||`.

Next: [Subroutines with call & goto →](09-subroutines-call-goto.md)
