# 7. Files & Folders

## What & why

Most batch scripts exist to move files around: copy build output, archive logs, clean temp folders. This lesson covers the file commands, their failure modes, and when to reach for `robocopy` instead.

## Making and removing folders

```bat
md "C:\work\output"            &rem make; creates intermediate folders too
mkdir "C:\work\a\b\c"          &rem md and mkdir are the same

rd "C:\work\output"            &rem remove — only if empty
rd /s /q "C:\work\output"      &rem /s recurse, /q no "Are you sure?" prompt
```

!!! danger "`rd /s /q` is irreversible"
    It does not use the Recycle Bin. There is no undo. Double-check the path — especially if it's built from a variable that could be empty (`rd /s /q "%target%\"` with empty `%target%` becomes `rd /s /q "\"` = the drive root). Guard it:
    ```bat
    if not defined target ( echo target not set & exit /b 1 )
    if not exist "%target%\" ( echo not a folder & exit /b 1 )
    rd /s /q "%target%"
    ```

## Copying

```bat
copy "src.txt" "dst.txt"                 &rem single file
copy "src.txt" "C:\backup\"              &rem into a folder (keep name)
copy "*.log" "C:\archive\"               &rem multiple files
copy /y "src.txt" "dst.txt"             &rem /y = overwrite without prompting
type a.txt b.txt > combined.txt          &rem concatenate
```

`copy` does not create missing destination folders and does not recurse.

### `xcopy` — folders and trees

```bat
xcopy "C:\proj\src" "C:\proj\bak" /E /I /Y
```

| Switch | Meaning |
| --- | --- |
| `/E` | copy all subfolders, including empty ones (`/S` skips empty) |
| `/I` | assume destination is a folder if it doesn't exist |
| `/Y` | overwrite without prompting |
| `/D` | only copy files newer than the destination |
| `/EXCLUDE:file` | skip paths listed in `file` |

### `robocopy` — the good one

For anything nontrivial (mirroring, retries, logging, large trees), use `robocopy`:

```bat
robocopy "C:\proj\src" "C:\proj\bak" /MIR /R:2 /W:5 /NP /LOG:"%~dp0copy.log"
```

| Switch | Meaning |
| --- | --- |
| `/MIR` | mirror: copy new/changed, **delete** extras in destination |
| `/E` | subfolders including empty (safer than `/MIR` — no deletes) |
| `/R:2` | retry 2 times on a locked file (default is **1 million** — set this!) |
| `/W:5` | wait 5 seconds between retries |
| `/NP` | no per-file percentage spam |
| `/LOG:` / `/TEE` | write a log / also show on console |
| `/XD` `/XF` | exclude dirs / files |

!!! warning "robocopy exit codes are not 0/1"
    `robocopy` returns a bitmask: **0** = nothing to do, **1** = files copied, **2** = extra files, **3** = 1+2, ... anything **≥ 8** is a real error. So:
    ```bat
    robocopy ...
    if %errorlevel% geq 8 ( echo robocopy failed & exit /b 1 )
    ```
    A plain `if errorlevel 1` treats a *successful copy* as failure.

## Moving and renaming

```bat
move "old\report.txt" "new\report.txt"
move /y "*.tmp" "C:\trash\"
ren "report.txt" "report-2026.txt"      &rem rename in place; ren cannot move
ren "*.txt" "*.bak"                       &rem bulk rename by extension
```

## Deleting files

```bat
del "temp.txt"
del /q "*.tmp"                &rem /q quiet (no wildcard confirm prompt)
del /f /q "readonly.txt"      &rem /f force read-only
del /s /q "C:\build\*.obj"    &rem recurse
```

`del` removes files, not folders. It does **not** use the Recycle Bin.

## Listing

```bat
dir                          &rem full listing
dir /b                       &rem bare: names only, one per line
dir /b /s "*.txt"            &rem bare + recurse = full paths, great for for /F
dir /a:d /b                  &rem directories only
dir /a:-d /b                 &rem files only
dir /o:-d                    &rem sort by date descending
dir /b /s *.log | find /c /v ""   &rem count matching files
```

`dir /b /s` piped into `for /F` is the standard "find all files matching X" idiom.

## Checking existence and attributes

```bat
if exist "report.txt" echo present
if exist "C:\logs\" echo folder present

attrib "file.txt"                    &rem show attributes
attrib +r "file.txt"                 &rem set read-only
attrib -r -h "file.txt"              &rem clear read-only and hidden
```

## Common mistakes

- **`rd /s /q` with an empty variable** wiping a drive root. Always guard the path.
- **Trusting `robocopy` exit codes like a normal command** — use `if %errorlevel% geq 8`.
- **`robocopy` without `/R:n`** — one locked file = a million retries = your script hangs "forever."
- **`copy` not creating folders** — `md` the destination first.
- **`del *.tmp` prompting** in a scheduled job with no console — add `/q`.
- **`ren` to move a file** — it can't; use `move`.
- **`move` overwriting silently** without `/y` in scripts, or prompting and hanging with a console — decide and be explicit.
- **Wildcards matching more than you think** — `del log*` also deletes `login.config`. Be specific.

## Exercises

1. Write `mkbak.bat SRC` that copies `SRC` to `SRC.bak` (use `%~1`), overwriting if it exists, and prints `Backed up` or an error if `SRC` doesn't exist.
2. Write a script that creates `demo\a`, `demo\b`, `demo\c`, drops an empty `.txt` in each, then lists them with `dir /b /s`.
3. Mirror `demo` to `demo-mirror` with `robocopy /E /R:1 /W:1` and report success/failure using the `>= 8` rule.
4. Bulk-rename every `.log` in a folder to `.log.old` with one `ren` command.
5. Recursively delete every `.tmp` under `%TEMP%\myapp` (create some first), quietly, and print how many were left (`dir /b /s *.tmp | find /c /v ""`).
6. Guard a `rd /s /q "%target%"` so it refuses to run when `target` is empty or not an existing folder.

## Recap

- `md` creates (with intermediate dirs); `rd /s /q` removes recursively with **no undo** — guard the path.
- `copy` = single files/no recursion; `xcopy /E /I /Y` = trees; `robocopy` = anything serious (`/R:2 /W:5`, check `errorlevel >= 8`).
- `move` moves/renames, `ren` renames only, `del` removes files only — none use the Recycle Bin.
- `dir /b /s pattern` → `for /F` is the find-files idiom.

Next: [Redirection & pipes →](08-redirection-and-pipes.md)
