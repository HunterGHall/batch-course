# 1. Recursion & Advanced `for`

## What & why

Real automation walks trees: find every `.csproj` under a solution, compute a folder's total size, delete empty directories bottom-up. `for /R` handles the simple sweeps; genuine recursion (a subroutine that `call`s itself) handles the cases where you need control over traversal order, pruning, or per-directory state.

## `for /R` recap and limits

```bat
rem Every .log anywhere under C:\logs:
for /R "C:\logs" %%f in (*.log) do echo %%f

rem Every directory (the "(.)" set is special: it yields each dir itself):
for /R "C:\proj" %%d in (.) do echo DIR: %%~fd
```

Limits of `for /R`:

- You can't **prune** a subtree (skip `node_modules\` entirely) — it always descends everywhere.
- Order is fixed (top-down, directory order); you can't do bottom-up.
- No per-directory setup/teardown.
- On a huge tree it enumerates everything before you can stop.

When any of those matter, recurse yourself.

## A recursive directory walk

```bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion

call :walk "%~1"
exit /b 0

:walk
    rem %1 = directory to process
    set "dir=%~1"

    rem --- prune rule ---
    for %%n in ("%dir%") do set "leaf=%%~nxn"
    if /i "!leaf!"=="node_modules" goto :eof
    if /i "!leaf!"==".git" goto :eof

    rem --- act on files in this directory ---
    for %%f in ("%dir%\*") do (
        if not exist "%%~f\" echo FILE: %%~ff
    )

    rem --- recurse into subdirectories ---
    for /D %%s in ("%dir%\*") do call :walk "%%~fs"
    goto :eof
```

`for /D %%s in ("%dir%\*")` lists immediate subdirectories only; the `call :walk` on each is what makes it recursive. Because you control the recursion, you can `goto :eof` early to prune.

## Bottom-up traversal (needed for "remove empty dirs")

Process children *before* the current directory:

```bat
:prune_empty
    set "dir=%~1"
    for /D %%s in ("%dir%\*") do call :prune_empty "%%~fs"

    rem now children are done; is this dir empty?
    for /F %%c in ('dir /b /a "%dir%" 2^>nul ^| find /c /v ""') do set "n=%%c"
    if "!n!"=="0" (
        rd "%dir%" 2>nul && echo removed empty: %dir%
    )
    goto :eof
```

## Recursion needs `setlocal` per level (usually)

Each `call :walk` shares the caller's variables unless the subroutine `setlocal`s. If you keep per-directory state in a variable (`dir`, `leaf`, a running count), either:

- `setlocal` / `endlocal` inside the subroutine so each level has its own copy, **or**
- carefully restore the variable after the recursive call, **or**
- use only the loop variable `%%x` and arguments `%~1` (which *are* naturally per-level).

```bat
:sizeof
    setlocal EnableDelayedExpansion
    set /a total=0
    for %%f in ("%~1\*") do set /a total+=%%~zf
    for /D %%s in ("%~1\*") do (
        call :sizeof "%%~fs"
        set /a total+=!__ret!
    )
    endlocal & set "__ret=%total%"
    goto :eof
```

Here `endlocal & set "__ret=%total%"` returns this level's subtotal past the scope barrier ([Intermediate Lesson 5](../intermediate/05-setlocal-endlocal-scope.md)); the caller adds it in.

## Depth guarding

Runaway recursion (a directory junction that loops, a bug) will pile up `setlocal` scopes — the limit is ~32 nested `setlocal`, after which things fail quietly. Guard it:

```bat
:walk
    set /a depth+=1
    if !depth! gtr 40 ( echo max depth hit at "%~1" 1>&2 & set /a depth-=1 & goto :eof )
    rem ... work + recurse ...
    set /a depth-=1
    goto :eof
```

And skip reparse points (junctions/symlinks) to avoid cycles:

```bat
for /D %%s in ("%dir%\*") do (
    set "attr=%%~as"
    rem the 9th attribute char is 'l' for a reparse point on some builds; more robust:
    dir /a:l "%%~fs" >nul 2>&1 && ( echo skip link: %%~fs ) || call :walk "%%~fs"
)
```

## `for /F` recursion trick (iterate `dir /s` output instead)

Often you don't need real recursion — `dir /s /b` already flattened the tree:

```bat
for /F "delims=" %%f in ('dir /s /b /a:-d "C:\proj\*.cs" 2^>nul') do call :process "%%f"
```

This is simpler and faster than a hand-rolled walk **when** you don't need pruning or ordering. Reach for recursion only when `dir /s` can't express what you need.

## Common mistakes

- **No `setlocal` in the recursive sub** — inner levels clobber `dir`, `total`, counters shared with outer levels.
- **`endlocal` eating the return value** — use `endlocal & set "__ret=%x%"` on one line.
- **Infinite recursion via junctions** — a directory junction pointing at an ancestor. Skip reparse points; cap depth.
- **Hitting the ~32 `setlocal` ceiling** silently — guard depth.
- **Using `for /R` and expecting to prune** — it can't; filter inside the loop body (wastes the enumeration) or recurse.
- **`for /D %%s in ("%dir%\*")` when there are no subdirs** — the loop body doesn't run (unlike plain `for` with a non-matching wildcard, which runs once with the literal). That's actually the convenient behaviour here.
- **Quoting**: always `call :walk "%%~fs"` and `set "dir=%~1"` — paths have spaces.

## Exercises

1. Write `:tree DIR` that prints an indented directory tree (indent by recursion depth). Cap depth at 5.
2. Write `:dirsize DIR outvar` that returns the total size in bytes of a folder and everything under it. Compare against `dir /s`'s total.
3. Write `:find_ext DIR EXT` that prints every file with extension `EXT` under `DIR`, skipping any `node_modules` or `.git` folder.
4. Write `:prune_empty DIR` that removes every empty directory under `DIR`, bottom-up. Test on a tree with nested empties.
5. Create a directory junction (`mklink /J loop ..\..`) inside a test tree and confirm your walker doesn't hang (add link-skipping).
6. Rewrite exercise 3 using only `dir /s /b` + `findstr` and note when that's the better choice.

## Recap

- `for /R` sweeps a whole subtree top-down with no pruning and fixed order.
- Real recursion = a subroutine that `call`s itself on each subdir (`for /D`), giving you pruning (`goto :eof` early), bottom-up order, and per-directory state.
- `setlocal`/`endlocal` per level for isolation; return subtree results with `endlocal & set "__ret=%x%"`.
- Guard recursion depth and skip reparse points (junctions) to avoid runaway loops.
- If you don't need pruning or ordering, `for /F` over `dir /s /b` is simpler and faster.

Next: [Robust argument parsing →](02-robust-argument-parsing.md)
