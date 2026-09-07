# 2. echo, Comments & Script Structure

## What & why

`echo` is how a script talks to you, and it has more edge cases than any command this simple deserves. This lesson covers printing text cleanly, the two comment styles, and the handful of commands that give a script a readable shape.

## `echo` basics

```bat
@echo off
echo Simple text.
echo Numbers 1 2 3 and symbols # $ %% are fine.
```

Note `%%` — a literal percent sign in a script is written `%%`. A single `%` starts variable expansion.

### Print a blank line

```bat
echo.
```

No space before the dot. `echo ` (with a trailing space and nothing else) prints `ECHO is off.` instead — a classic surprise. `echo.`, `echo:`, `echo(` all print an empty line; `echo(` is the most robust.

### `echo on` / `echo off` / bare `echo`

```bat
echo            &rem prints whether echo is on or off
echo on         &rem turn command-echoing back on (for debugging)
echo off        &rem turn it off again
```

### Characters that need escaping

The shell metacharacters `& | < > ^ ( )` must be escaped with a caret `^` when you want them printed literally:

```bat
echo Tom ^& Jerry
echo Use a pipe like this: ^|
echo 3 ^> 2 is true
```

Inside double quotes on an `echo` line the quotes are printed too, so quoting is *not* the way to escape here — use `^`.

## Comments

```bat
rem This is the official comment command.
:: This is a label that can never be jumped to, used as a comment.
```

`::` is a trick: it's a broken label, and `cmd` skips it. It's faster to type and visually cleaner, but has two hazards:

- **Inside a `for` loop or a parenthesised block, `::` can cause errors.** Use `rem` there.
- `::` on the same line as code doesn't work as a trailing comment. Use `&rem`:

```bat
copy a.txt b.txt   &rem trailing comment, note the single &
```

## Giving a script structure

```bat
@echo off
setlocal
title Deploy Tool
color 0A

cls
echo ============================
echo   Deploy Tool  v1.0
echo ============================
echo.

echo Step 1: checking prerequisites...
echo Step 2: copying files...
echo.

echo Done.
endlocal
pause
```

| Command | Purpose |
| --- | --- |
| `title Deploy Tool` | Sets the window title bar. |
| `color 0A` | Background 0 (black), text A (bright green). `color` with no args resets. |
| `cls` | Clears the screen. |
| `setlocal` / `endlocal` | Isolates variable changes to this script — [Intermediate Lesson 5](../intermediate/05-setlocal-endlocal-scope.md). Get in the habit now. |

## Multi-line output

Every `echo` is its own line. For a block, either repeat `echo`, or use one redirect:

```bat
(
  echo Line 1
  echo Line 2
  echo Line 3
) 

rem Or build a here-doc-ish file:
(
  echo [settings]
  echo verbose=1
) > config.ini
```

## Common mistakes

- **`echo.` with a space** (`echo .`) prints a literal dot. No space.
- **`echo %PATH%` truncating** — it doesn't truncate, but if `PATH` contains `)` inside a parenthesised block it breaks parsing. Quote or restructure.
- **`::` inside `for ( … )`** producing `The system cannot find the drive specified.` Use `rem`.
- **Unescaped `&` in an echo** — `echo Ben & Jerry` runs `Jerry` as a command.
- **Trailing spaces after a value** you `echo` into a file — they're included. Watch line ends.

## Exercises

1. Print this exact block using `echo`, including the blank line:
   ```
   Report
   
   Status: OK
   ```
2. Print the literal string `A & B | C > D` on one line.
3. Write a script that sets the window title to `Backup`, clears the screen, prints a 3-line banner, then `pause`s.
4. Write `notes.bat` with a `rem` header comment describing what it does, three `::` comments between sections, and one `&rem` trailing comment. Run it with `@echo off` removed and confirm no comment lines execute.
5. Make `echo` print `100%` correctly (hint: `%%`).

## Recap

- `echo text` prints; `echo.` (no space) or `echo(` prints a blank line.
- Literal `%` is `%%`; escape `& | < > ^ ( )` with `^`.
- `rem` is the safe comment; `::` is cleaner but avoid it inside `for`/blocks.
- `title`, `color`, `cls`, `setlocal`/`endlocal` give a script shape.

Next: [Variables & set →](03-variables-and-set.md)
