# 10. Capstone: File Organizer

## The goal

Build `organize.bat`, a script that sorts a messy folder into subfolders by file type:

```
organize.bat "C:\Users\you\Downloads"
```

turns

```
Downloads\
  invoice.pdf
  cat.jpg
  cat2.png
  setup.exe
  notes.txt
  archive.zip
```

into

```
Downloads\
  Documents\   invoice.pdf  notes.txt
  Images\      cat.jpg  cat2.png
  Installers\  setup.exe
  Archives\    archive.zip
  Other\       (anything unrecognised)
```

It must use everything from this level: arguments, `if`, `for`, file ops, subroutines, redirection, exit codes.

## Requirements

1. Take the target folder as `%1`. If missing or not a folder, print usage and `exit /b 1`.
2. Support a `--dry-run` flag (in any position) that prints what *would* happen without moving anything.
3. Categorise by extension:
   - Images: `.jpg .jpeg .png .gif .bmp .webp .heic`
   - Documents: `.pdf .txt .md .doc .docx .xls .xlsx .ppt .pptx .csv`
   - Archives: `.zip .rar .7z .tar .gz`
   - Installers: `.exe .msi`
   - anything else → `Other`
4. Create category subfolders only when needed.
5. Never move the script itself, and never recurse into the category folders you create.
6. Write a log (`organize-log.txt` in the target folder) of every move, with a timestamp.
7. Print a summary: how many files moved into each category.
8. Exit `0` on success, non-zero if any move failed.

## Starter skeleton

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion

set "TARGET="
set "DRYRUN="

rem --- parse arguments ---
:parse
if "%~1"=="" goto parsed
if /i "%~1"=="--dry-run" (
    set "DRYRUN=1"
) else (
    set "TARGET=%~1"
)
shift
goto parse
:parsed

if not defined TARGET (
    echo Usage: %~nx0 ^<folder^> [--dry-run]
    exit /b 1
)
if not exist "%TARGET%\" (
    echo ERROR: not a folder: "%TARGET%" 1>&2
    exit /b 1
)

set "LOG=%TARGET%\organize-log.txt"
set "FAILED=0"
set /a nImages=0, nDocs=0, nArch=0, nInst=0, nOther=0

rem --- main loop: files in the top level only ---
for %%F in ("%TARGET%\*") do (
    if /i not "%%~fF"=="%~f0" (
        call :categorize "%%~xF" CATEGORY
        call :move_file "%%~fF" "!CATEGORY!"
    )
)

rem --- summary ---
echo.
echo Images:%nImages%  Documents:%nDocs%  Archives:%nArch%  Installers:%nInst%  Other:%nOther%
if %FAILED% gtr 0 ( echo %FAILED% move(s) failed. & exit /b 1 )
exit /b 0

rem ===================================================================
:categorize
    rem %1 = extension (with dot), %2 = out var name
    set "_ext=%~1"
    set "_cat=Other"
    for %%e in (.jpg .jpeg .png .gif .bmp .webp .heic) do if /i "%_ext%"=="%%e" set "_cat=Images"
    for %%e in (.pdf .txt .md .doc .docx .xls .xlsx .ppt .pptx .csv) do if /i "%_ext%"=="%%e" set "_cat=Documents"
    for %%e in (.zip .rar .7z .tar .gz) do if /i "%_ext%"=="%%e" set "_cat=Archives"
    for %%e in (.exe .msi) do if /i "%_ext%"=="%%e" set "_cat=Installers"
    set "%~2=!_cat!"
    goto :eof

:move_file
    rem %1 = full source path, %2 = category name
    set "_src=%~1"
    set "_cat=%~2"
    set "_dstdir=%TARGET%\%_cat%"

    if defined DRYRUN (
        echo [dry-run] "%~nx1"  ->  %_cat%\
    ) else (
        if not exist "%_dstdir%\" md "%_dstdir%"
        move /y "%_src%" "%_dstdir%\" >nul
        if errorlevel 1 (
            echo [%date% %time%] FAILED  "%~nx1" -> %_cat%>>"%LOG%"
            set /a FAILED+=1
        ) else (
            echo [%date% %time%] moved   "%~nx1" -> %_cat%>>"%LOG%"
        )
    )

    if /i "%_cat%"=="Images"     set /a nImages+=1
    if /i "%_cat%"=="Documents"  set /a nDocs+=1
    if /i "%_cat%"=="Archives"   set /a nArch+=1
    if /i "%_cat%"=="Installers" set /a nInst+=1
    if /i "%_cat%"=="Other"      set /a nOther+=1
    goto :eof
```

## Why the tricky bits are there

- **`EnableDelayedExpansion` + `!CATEGORY!`** — `CATEGORY` is set by a subroutine and read inside the same `for` block; `%CATEGORY%` would be stale ([Intermediate Lesson 1](../intermediate/01-delayed-expansion.md)).
- **`if /i not "%%~fF"=="%~f0"`** — skip the script itself so it doesn't move `organize.bat` into `Other`.
- **`for %%F in ("%TARGET%\*")`** — top level only; it does not descend into `Images\` etc., so re-running is safe.
- **`move /y ... >nul` then `if errorlevel 1`** — detect a failed move (file locked, permissions) and count it.
- **Counters** (`nImages` etc.) are set with `%%~xF` categorisation and read at the end where `%var%` *is* correct.

## Test plan

1. Make a scratch folder with one file of each type plus a weird one (`data.xyz`).
2. Run with `--dry-run`. Confirm the plan looks right and **nothing moved**, no log written.
3. Run for real. Check the subfolders, the summary counts, and `organize-log.txt`.
4. Run **again**. It should find nothing to do (all files are now inside category folders, which the top-level glob skips) and still exit `0`.
5. Lock a file (open it in an app) and run again with a new file of that type — confirm the failure is logged and the exit code is non-zero.
6. Run with no argument, and with a path that doesn't exist — confirm usage message and `exit /b 1`.

## Extensions (do at least two)

1. `--undo`: read `organize-log.txt` and move every "moved" file back to `%TARGET%`.
2. `--by-date`: instead of type, sort into `YYYY-MM` folders using `%%~tF`.
3. Config file: read extension→category mappings from `organize.ini` next to the script instead of hard-coding them ([Intermediate Lesson 3](../intermediate/03-for-f-deep-dive.md)).
4. Collision handling: if `Images\cat.jpg` already exists, rename the incoming file to `cat (2).jpg`.
5. `--quiet` / `--verbose` output levels.

## What you should be able to do now

- Explain every line, including why delayed expansion is required and where `%var%` is still fine.
- Add a new category without breaking anything.
- Predict the exit code for any input.

If all three are true, go to [Intermediate](../intermediate/index.md).
