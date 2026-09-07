# 6. Arrays & Pseudo Data Structures

## What & why

Batch has no arrays, lists, maps, or objects. But it has variables with arbitrary names, and `!var!` can build a name dynamically. That's enough to fake indexed arrays, associative arrays, stacks, and records — which is how real scripts track "the list of servers" or "settings by key."

## Indexed "arrays"

Convention: name variables `arr[0]`, `arr[1]`, ... and keep a count.

```bat
@echo off
setlocal EnableDelayedExpansion

set "fruit[0]=apple"
set "fruit[1]=banana"
set "fruit[2]=cherry"
set "fruit.count=3"

rem read one:
echo First: !fruit[0]!

rem iterate:
for /L %%i in (0,1,2) do echo %%i = !fruit[%%i]!

rem iterate using the count:
set /a last=fruit.count-1
for /L %%i in (0,1,!last!) do echo !fruit[%%i]!
```

The `[` `]` are just characters in the variable name — not syntax. `fruit.count` works too; pick a style.

### Appending

```bat
set "n[.count]=0"
call :push n apple
call :push n banana
call :push n "cherry pie"
for /L %%i in (0,1,%n[.count]%) do echo !n[%%i]!   & rem off-by-one, see below
exit /b 0

:push
    rem %1 = array name, %2 = value
    set /a "%~1[.count]+=0"           &rem ensure it exists as a number
    set "idx=!%~1[.count]!"
    set "%~1[!idx!]=%~2"
    set /a "%~1[.count]+=1"
    goto :eof
```

!!! note "Count vs last index"
    Decide once: does `count` mean "number of items" (iterate `0` to `count-1`) or "highest index used" (iterate `0` to `count`)? Mixing the two is the classic array bug. This course uses **count = number of items**.

## Associative "arrays" (maps)

Use the key as part of the variable name:

```bat
setlocal EnableDelayedExpansion

set "color[red]=#FF0000"
set "color[green]=#00FF00"
set "color[blue]=#0000FF"

set "key=green"
echo !color[%key%]!            &rem #00FF00

rem does a key exist?
if defined color[%key%] echo yes

rem iterate keys + values:
for /F "tokens=2,3 delims=[]=" %%k in ('set color[') do echo %%k -> %%l
```

`set color[` lists every variable starting with `color[`; parsing its output with `for /F` and `delims=[]=` splits `color[green]=#00FF00` into `color`, `green`, `#00FF00`.

!!! warning
    Keys can't contain characters that are illegal in variable names or that break your `for /F` delims: `[`, `]`, `=`, and leading/trailing spaces are trouble. Sanitize keys, or hash them.

## Records / structs

Group related fields with a shared prefix:

```bat
set "user1.name=Ada"
set "user1.role=admin"
set "user1.active=1"

set "u=user1"
echo !%u%.name! is a !%u%.role!
```

## Stack (LIFO)

```bat
setlocal EnableDelayedExpansion
set "sp=0"

call :push_s 10
call :push_s 20
call :push_s 30
call :pop_s  &  echo popped !popped!     &rem 30
call :pop_s  &  echo popped !popped!     &rem 20
exit /b 0

:push_s
    set "stack[!sp!]=%~1"
    set /a sp+=1
    goto :eof
:pop_s
    set /a sp-=1
    set "popped=!stack[%sp%]!"
    set "stack[!sp!]="
    goto :eof
```

## Building a list from command output

```bat
setlocal EnableDelayedExpansion
set "count=0"
for /F "delims=" %%f in ('dir /b /a:-d *.log') do (
    set "log[!count!]=%%f"
    set /a count+=1
)
echo Found !count! logs:
for /L %%i in (0,1,!count!) do if %%i lss !count! echo   !log[%%i]!
```

## Sorting

No built-in sort for arrays. Options:

1. Write the items to a temp file, `sort` it, read it back.
2. Use `dir /o:n` / `dir /o:-d` when the items are filenames.
3. Implement bubble/insertion sort over the pseudo-array (fine for small N).

```bat
rem Sort a list of strings via a temp file:
> "%TEMP%\items.txt" (
    for /L %%i in (0,1,%count%) do if %%i lss %count% echo(!item[%%i]!
)
set "count=0"
for /F "usebackq delims=" %%L in (`sort "%TEMP%\items.txt"`) do (
    set "item[!count!]=%%L"
    set /a count+=1
)
del "%TEMP%\items.txt"
```

## Common mistakes

- **`%arr[%i%]%`** — you can't nest `%...%`. Use delayed expansion: `!arr[%%i]!` in a loop, `!arr[%idx%]!` otherwise, or `call set`.
- **Count/last-index confusion** — one off-by-one and you skip the last item or read an empty slot.
- **Forgetting `setlocal EnableDelayedExpansion`** — `!arr[0]!` prints literally.
- **Keys with spaces/`=`/`[`** — corrupt the name or the `for /F` parse.
- **Leaked pseudo-arrays** — 500 `set "x[i]=..."` without `setlocal` pollute the shell. Always `setlocal`.
- **Assuming order** from `set prefix` enumeration — it's alphabetical by name, so `x[10]` sorts before `x[2]`. Zero-pad indices if you enumerate that way, or iterate with `for /L`.
- **No real sort** — don't fight it; use a temp file + `sort`.

## Exercises

1. Build an indexed array of the words in `"the quick brown fox"` (split with `for`), store the count, and print each as `[i] word`.
2. Implement `:contains arrayName value` returning `0`/`1` via `exit /b`. Test it.
3. Build an associative array from a `.ini` file (`key=value` lines) into `cfg[key]`; then look up three keys by name.
4. Implement a queue (FIFO) with `enqueue`/`dequeue` subroutines and a head/tail index.
5. Read all `*.txt` filenames into an array, sort them via a temp file, and print the sorted list.
6. Zero-pad indices to width 3 so that enumerating with `set arr[` returns them in numeric order; verify `arr[002]` comes before `arr[010]`.

## Recap

- No native arrays — fake them with `name[i]` variables + a count, read via `!name[%%i]!`.
- Associative arrays: put the key in the name (`map[key]`); enumerate with `set prefix` + `for /F "delims=[]="`.
- Records: shared prefix (`user1.name`). Stacks/queues: an array plus a pointer index.
- Nested `%...%` is impossible — delayed expansion or `call set` for dynamic names.
- Decide "count = item count" and stick to it. Sort via a temp file + `sort`.
- Always `setlocal` — pseudo-arrays create a lot of variables.

Next: [Date, time & math →](07-date-time-and-math.md)
