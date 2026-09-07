# The Batch Scripting Course

A complete, self-paced course on **Windows batch scripting** (`.bat` / `.cmd` files run by `cmd.exe`). Work through it top to bottom, or jump to the level that matches your experience.

## How this course is organized

| Folder | For you if... | Outcome |
| --- | --- | --- |
| [Beginner](beginner/index.md) | You have never written a `.bat` file | Write real scripts: variables, arguments, `if`, `for`, file operations, subroutines |
| [Intermediate](intermediate/index.md) | You can write basic scripts but hit weird bugs | Delayed expansion, string surgery, `for /F`, error handling, menus, scheduling |
| [Advanced](advanced/index.md) | You maintain batch scripts and want to do it well | Recursion, registry, WMIC/PowerShell interop, `findstr` regex, hybrid scripts, packaging, debugging |

Each lesson is one file and follows the same shape:

1. **What & why** – the idea in plain language
2. **Examples** – runnable code you should type out yourself
3. **Common mistakes** – the traps everyone hits
4. **Exercises** – do these; reading is not learning
5. **Recap**

## Setup (do this first)

You need Windows. Everything in this course ships with the OS — no installs.

```bat
cmd /c ver
```

That prints something like `Microsoft Windows [Version 10.0.26100.xxxx]`. If you can open a **Command Prompt** window (press <kbd>Win</kbd>, type `cmd`, <kbd>Enter</kbd>), you are ready.

A good editor helps a lot: [VS Code](https://code.visualstudio.com/) (set the file language to "Bat"), or [Notepad++](https://notepad-plus-plus.org/). Avoid Windows Notepad for anything non-trivial — it hides line-ending and encoding problems.

!!! note "`.bat` vs `.cmd`"
    They are the same to you 99% of the time. `.cmd` is the modern extension and has one small advantage covered in [Lesson 1](beginner/01-setup-and-first-script.md). This course uses `.bat` in examples because it is what you will see in the wild.

!!! warning "Batch is not PowerShell"
    `cmd.exe` batch and PowerShell are different languages. This course is about `cmd.exe` batch — the language of `.bat` files, `AUTOEXEC.BAT`, most CI scripts on Windows, and countless installers. [Advanced Lesson 4](advanced/04-wmic-where-powershell.md) covers when to call PowerShell from batch and when to just switch languages.

## How to study

- **Type every example.** Copy-paste teaches you nothing.
- **Run it in a Command Prompt window**, not by double-clicking — a double-clicked script closes its window the instant it finishes or errors, and you never see what happened.
- **Break things on purpose.** Change a line, predict the result, run it.
- **Do the exercises before looking at solutions.**
- **One lesson per sitting** is plenty.

## Curriculum at a glance

### Beginner
1. [Setup & your first script](beginner/01-setup-and-first-script.md)
2. [echo, comments & script structure](beginner/02-echo-comments-structure.md)
3. [Variables & `set`](beginner/03-variables-and-set.md)
4. [Command-line arguments](beginner/04-command-line-arguments.md)
5. [Conditionals with `if`](beginner/05-conditionals-with-if.md)
6. [Loops with `for`](beginner/06-loops-with-for.md)
7. [Files & folders](beginner/07-files-and-folders.md)
8. [Redirection & pipes](beginner/08-redirection-and-pipes.md)
9. [Subroutines with `call` & `goto`](beginner/09-subroutines-call-goto.md)
10. [Capstone: file organizer](beginner/10-capstone-file-organizer.md)

### Intermediate
1. [Delayed expansion](intermediate/01-delayed-expansion.md)
2. [String manipulation](intermediate/02-string-manipulation.md)
3. [`for /F` deep dive](intermediate/03-for-f-deep-dive.md)
4. [Error handling & exit codes](intermediate/04-error-handling-exit-codes.md)
5. [`setlocal`, `endlocal` & scope](intermediate/05-setlocal-endlocal-scope.md)
6. [Arrays & pseudo data structures](intermediate/06-arrays-and-data-structures.md)
7. [Date, time & math](intermediate/07-date-time-and-math.md)
8. [User interaction & menus](intermediate/08-user-interaction-and-menus.md)
9. [Scheduling & running scripts](intermediate/09-scheduling-and-running.md)
10. [Capstone: backup utility](intermediate/10-capstone-backup-utility.md)

### Advanced
1. [Recursion & advanced `for`](advanced/01-recursion-and-advanced-for.md)
2. [Robust argument parsing](advanced/02-robust-argument-parsing.md)
3. [Working with the registry](advanced/03-working-with-the-registry.md)
4. [WMIC, `where` & calling PowerShell](advanced/04-wmic-where-powershell.md)
5. [Text processing with `findstr`](advanced/05-text-processing-findstr.md)
6. [Performance & pitfalls](advanced/06-performance-and-pitfalls.md)
7. [Hybrid scripts](advanced/07-hybrid-scripts.md)
8. [Packaging & distribution](advanced/08-packaging-and-distribution.md)
9. [Debugging techniques](advanced/09-debugging-techniques.md)
10. [Capstone: deployment automation](advanced/10-capstone-deployment-automation.md)

## License

Use freely for learning and teaching.
