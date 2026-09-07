# 1. Setup & Your First Script

## What & why

A batch script is a plain text file of Command Prompt commands. When you run it, `cmd.exe` executes the lines top to bottom, exactly as if you typed them. That's the whole model. This lesson gets you running scripts and explains the one line you'll put at the top of every script you write.

## Open a Command Prompt

Press <kbd>Win</kbd>, type `cmd`, press <kbd>Enter</kbd>. You get a window with a prompt like:

```
C:\Users\you>
```

Check your Windows version:

```bat
ver
```

## Your first script

Open your editor and create a file called `hello.bat`:

```bat
@echo off
echo Hello, world!

set /p name=What's your name?
echo Nice to meet you, %name%.
```

Save it, then in the Command Prompt `cd` to where you saved it and run it by name:

```bat
cd C:\Users\you\Desktop
hello.bat
```

You can also run it by just `hello` (the `.bat` extension is optional when typing).

!!! warning "Don't double-click it"
    Double-clicking runs the script in a window that **closes the instant it finishes** — you won't see output or errors. Always run scripts from an already-open Command Prompt while learning. (Add `pause` as the last line if you must double-click.)

### What each piece does

| Line | Meaning |
| --- | --- |
| `@echo off` | Stop printing each command before it runs. The `@` hides *this* line too. |
| `echo Hello, world!` | Print text. |
| `set /p name=...` | Print a prompt, wait for input, store it in the variable `name`. |
| `%name%` | Insert the value of `name`. Percent signs around a name mean "expand this variable." |

## `@echo off` — always

Without it, running the script prints every command *and then* its output:

```
C:\>hello.bat
C:\>echo Hello, world!
Hello, world!
```

Noisy. `@echo off` at the top of every script fixes that. The `@` prefix suppresses the echo of that single line; after `echo off` takes effect you don't need `@` anymore.

## `.bat` vs `.cmd`

Practically identical. Two differences worth knowing:

- `.cmd` is the newer extension (Windows NT and later). Some people use it to signal "this is a modern script, not DOS."
- With `.cmd`, some internal commands (`append`, `dpath`, `set`, `path`, ...) set `errorlevel` to `0` on success. With `.bat` they may leave `errorlevel` unchanged. This bites you rarely — see [Intermediate Lesson 4](../intermediate/04-error-handling-exit-codes.md).

Use whichever your team uses. This course writes `.bat`.

## Comments and stopping

```bat
@echo off
rem This is a comment.
echo Working...
pause
```

`pause` prints `Press any key to continue . . .` and waits. Handy while developing.

## Common mistakes

- **Double-clicking** and seeing a window flash and vanish. Run from an open prompt.
- **Saving as `hello.bat.txt`.** Notepad does this. Turn on file extensions in Explorer (View → Show → File name extensions) and check.
- **Saving as UTF-8 with BOM.** The BOM becomes an invisible first "command" and you get `'∩╗┐@echo' is not recognized`. Save as ANSI / "UTF-8" (without BOM).
- **A space in `set /p name =`** — that makes the variable name `"name "` with a trailing space. No spaces around `=`.
- **Running from the wrong folder.** `cd` into the script's folder first, or type its full path.

## Exercises

1. Write `whoami-info.bat` that prints your username (`echo %username%`), your computer name (`%computername%`), and the current folder (`%cd%`).
2. Write `greet.bat` that asks for a first name and last name on two separate prompts, then prints `Hello, FIRST LAST!`.
3. Add `pause` to the end of `greet.bat`, save, and **double-click** it in Explorer. Confirm you can now read the output. Remove the `pause` and double-click again to see the difference.
4. Break it: delete the `@echo off` line and run the script. Read the extra output. Put it back.

## Recap

- A batch script is commands in a text file, run top to bottom by `cmd.exe`.
- Run scripts from an **open** Command Prompt, not by double-clicking.
- Start every script with `@echo off`.
- `echo` prints, `set /p` reads input, `%name%` expands a variable, `rem` comments, `pause` waits.
- Watch out for `.txt` extensions and BOM encoding from Notepad.

Next: [echo, comments & script structure →](02-echo-comments-structure.md)
