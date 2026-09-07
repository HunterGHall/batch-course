# 5. `setlocal`, `endlocal` & Scope

## What & why

By default, a batch script shares the *caller's* environment: every `set`, every `cd`, every `PATH` tweak leaks out and persists after the script ends. `setlocal` creates a private copy that's thrown away at `endlocal` (or when the script ends). Understanding exactly what it saves and restores — and how to smuggle one value back out — is essential.

## What `setlocal` snapshots

```bat
@echo off
setlocal

set "TEMPVAR=only visible in here"
cd C:\Windows
set "PATH=C:\tools;%PATH%"

endlocal
rem After endlocal: TEMPVAR is gone, PATH is restored,
rem and the current directory is back to what it was.
```

`setlocal` saves and later restores:

- **all environment variables** (additions, changes, deletions)
- **the current drive and directory**
- **the state of command extensions and delayed expansion**

It does **not** roll back: files you created, registry changes, running processes, `title`, `color`.

## `setlocal` options

```bat
setlocal EnableExtensions EnableDelayedExpansion
setlocal DisableExtensions
setlocal DisableDelayedExpansion
```

- **Command extensions** (`EnableExtensions`) power almost everything modern: `for /F`, `call :label`, `if defined`, `set /a`, `%~dp0`. They're on by default on modern Windows but making it explicit protects you from a machine where they're disabled globally.
- **Delayed expansion** — see [Lesson 1](01-delayed-expansion.md).

A bare `setlocal` inherits the current state of both.

## Nesting

Each `setlocal` pushes a new scope; each `endlocal` pops one:

```bat
setlocal                    &rem scope 1
set "x=1"
    setlocal                &rem scope 2
    set "x=2"
    echo %x%                &rem 2
    endlocal                &rem pop scope 2
echo %x%                     &rem 1
endlocal
```

If you don't `endlocal`, batch does it for you when the script exits — one auto-`endlocal` per outstanding `setlocal`. Subroutines called with `call :label` do **not** get an automatic `setlocal`; they run in the caller's scope unless they call `setlocal` themselves.

## The `endlocal` barrier — returning a value

Everything set inside a `setlocal` block dies at `endlocal`. To carry one value out, expand it **on the same line as `endlocal`**, before the barrier drops:

```bat
setlocal
set /a result = 6 * 7
endlocal & set "result=%result%"
echo %result%          &rem 42
```

How this works: `cmd` parses the whole `endlocal & set "result=%result%"` line first, so `%result%` becomes `42` *while the inner scope is still active*; then `endlocal` runs, then `set "result=42"` runs in the **outer** scope.

### Multiple values

```bat
endlocal & set "a=%a%" & set "b=%b%" & set "c=%c%"
```

### With delayed expansion (careful)

Inside a delayed-expansion scope, `%result%` on the `endlocal` line still works (it's parse-time). But if the value can contain `!`, you need the classic dance:

```bat
setlocal EnableDelayedExpansion
set "msg=Done!!!"
for %%# in ("!msg!") do (
    endlocal
    set "msg=%%~#"
)
echo %msg%
```

## The subroutine pattern

A well-behaved subroutine isolates itself and returns via the barrier:

```bat
call :get_size "C:\big\file.iso" SIZE
echo Size is %SIZE% bytes
exit /b 0

:get_size
    setlocal
    set "bytes=%~z1"
    endlocal & set "%~2=%bytes%"
    goto :eof
```

`%~2` is the caller-supplied output variable name; the assignment lands in the caller's scope because it's after `endlocal`.

## `cd` and `pushd`/`popd`

`setlocal` restores the directory at `endlocal`. Within a scope, prefer `pushd`/`popd` for temporary directory changes — `pushd` also handles UNC paths by mapping a temp drive letter:

```bat
setlocal
pushd "\\server\share\build"
rem ... work here ...
popd
endlocal
```

## Common mistakes

- **No `setlocal` at all** — your script pollutes the user's shell: leftover variables, changed `PATH`, wrong current directory afterward.
- **Setting a result inside `setlocal` and reading it after `endlocal`** — it's gone. Use `endlocal & set "x=%x%"`.
- **`endlocal & set "x=%x%"` on two lines** — the `%x%` on its own line is too late; the scope already collapsed.
- **Unbalanced `setlocal`/`endlocal` in a loop** — each iteration `setlocal`s but the block never `endlocal`s, so you stack scopes until the script ends (and hit the 32-level limit, or just leak memory/perf).
- **Expecting `call :sub` to auto-isolate** — it doesn't; add `setlocal` inside the sub.
- **`setlocal EnableDelayedExpansion` globally** then wondering why strings with `!` break — scope it.
- **Relying on `endlocal` to undo file/registry changes** — it only touches env vars and the cwd.

## Exercises

1. Write a script that changes `PATH` and `cd`s elsewhere inside `setlocal`; after `endlocal`, print `%PATH%` and `%CD%` to confirm they're restored.
2. Write `:double n outvar` using `setlocal` + the `endlocal & set` barrier. Prove the inner temp variable doesn't leak.
3. Nest three `setlocal` scopes, each redefining `LEVEL`; print `LEVEL` after each `endlocal` and confirm it unwinds `3 → 2 → 1`.
4. Write a `for /L` loop that `setlocal`s each iteration to isolate a per-item variable, and `endlocal`s at the end of the block. Confirm scope count stays flat (add `echo` of a marker; run with `EnableExtensions`).
5. Return a value containing `!` out of a delayed-expansion scope correctly.
6. Convert a script that currently leaks 5 variables into one that leaks nothing but still returns its one real result.

## Recap

- `setlocal` snapshots **all env vars + the current drive/dir + extension/delayed-expansion state**; `endlocal` (or script end) restores them.
- It does not undo files, registry, processes, `title`/`color`.
- `setlocal EnableExtensions EnableDelayedExpansion` is a good explicit default.
- Return a value past `endlocal` with `endlocal & set "out=%out%"` **on one line**.
- `call`ed subroutines don't auto-isolate — `setlocal` inside them.
- Keep `setlocal`/`endlocal` balanced, especially in loops.

Next: [Arrays & pseudo data structures →](06-arrays-and-data-structures.md)
