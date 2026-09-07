# 7. Hybrid Scripts

## What & why

A hybrid script is a single file that is **valid batch and valid something-else at the same time** — usually PowerShell or JScript. The batch part runs first, then hands the *same file* to the other interpreter. You get one distributable file: no separate `.ps1` to lose, no execution-policy prompt, double-clickable. The trick is a header that both languages tolerate.

When to use this: you want PowerShell's power but must ship a single `.bat` that runs on a locked-down box. When *not* to: if you can ship two files, do — hybrids are clever but harder to read and debug.

## Batch + PowerShell

### The idea

- Line 1 is a batch command that re-invokes PowerShell on this file and then exits.
- To PowerShell, that same line 1 must be a no-op (or a comment).

### A working header

```bat
<# : batch portion
@echo off
setlocal
echo [batch] preparing...
set "NAME=%~1"
powershell -NoProfile -ExecutionPolicy Bypass -Command ^
  "$input | &{ [ScriptBlock]::Create((Get-Content -LiteralPath '%~f0' -Raw)).Invoke() }" -- %*
endlocal
exit /b %errorlevel%
: end batch / begin powershell #>

param()
Write-Host "[powershell] running as $env:USERNAME"
Write-Host "[powershell] arg NAME = $env:NAME"
Get-ChildItem -Path . -File | Select-Object -First 3 Name, Length
exit 0
```

Why it works:

- `<# ... #>` is a **PowerShell block comment**. Everything between `<#` and `#>` is invisible to PowerShell.
- To **batch**, `<# : batch portion` — the `<` is a redirect from a file named `#`... actually `<#` with a label-ish `:` is tolerated; the common robust form is the first line being exactly `<# :` and batch treating `: batch portion` as a label/no-op. The batch body then runs normally until `exit /b`.
- The batch body calls `powershell` on `%~f0` (this file). PowerShell reads the file, skips the `<#...#>` comment, and runs the `param()`-onward part.

!!! note "Use a known-good header"
    The exact incantation is finicky and there are several published variants (DosTips' "batch + PowerShell hybrid" is the reference). Copy a working one verbatim rather than reconstructing it from memory. Test on the actual target PowerShell version.

### Simpler, less magic: heredoc-style

If you don't need the PowerShell part to be huge, skip the true hybrid and just embed:

```bat
@echo off
setlocal
set "PS=%TEMP%\_%~n0_%RANDOM%.ps1"
> "%PS%" (
    echo param($Name^)
    echo Write-Host "Hello, $Name from PowerShell"
    echo Get-Date -Format o
)
powershell -NoProfile -ExecutionPolicy Bypass -File "%PS%" -Name "%~1"
set "rc=%errorlevel%"
del "%PS%" 2>nul
endlocal & exit /b %rc%
```

This writes a temp `.ps1`, runs it, deletes it. Easier to read and debug than a true hybrid; the only downside is a transient temp file. For most "single file" needs this is the pragmatic choice.

## Batch + JScript (`cscript`)

Windows ships the Windows Script Host (`cscript.exe`) with JScript — no PowerShell startup cost (~10 ms vs ~200 ms), useful for fast string/number work or when PowerShell is blocked.

```bat
@if (@CodeSection == @Batch) @then
@echo off
setlocal
echo [batch] calling JScript engine...
cscript //nologo //E:JScript "%~f0" %*
endlocal & exit /b %errorlevel%
@end

// ==== JScript from here ====
var args = WScript.Arguments;
WScript.Echo("[jscript] got " + args.length + " argument(s)");
for (var i = 0; i < args.length; i++) WScript.Echo("  " + i + ": " + args(i));

// JScript has real regex, real numbers, JSON via eval, etc.
var s = "2026-09-07";
var m = s.match(/^(\d{4})-(\d{2})-(\d{2})$/);
if (m) WScript.Echo("year=" + m[1] + " month=" + m[2] + " day=" + m[3]);
WScript.Quit(0);
```

Why it works:

- `@if (@CodeSection == @Batch) @then` — to **batch**, `@if` runs `if` with weird-but-tolerated tokens and the `@then`/`@end` act as harmless markers; the batch body executes.
- To **JScript**, `@if (@CodeSection == @Batch) @then ... @end` is **conditional compilation** syntax (`@if`/`@end` are real JScript conditional-compilation directives), and `@CodeSection` is undefined so the block is skipped. `@echo off`, `cscript ...` etc. sit inside that skipped block.
- `//E:JScript` forces the engine; `//nologo` hides the banner.

### Batch + JScript for regex replace (a classic use)

```bat
@if (@X)==(@Y) @end /* JScript multiline comment trick
@echo off
cscript //nologo //E:JScript "%~f0" "%~1" "%~2" "%~3"
exit /b %errorlevel%
*/
// args: file, pattern, replacement
var fso = new ActiveXObject("Scripting.FileSystemObject");
var file = WScript.Arguments(0), pat = WScript.Arguments(1), rep = WScript.Arguments(2);
var text = fso.OpenTextFile(file, 1).ReadAll();
text = text.replace(new RegExp(pat, "g"), rep);
var out = fso.CreateTextFile(file, true);
out.Write(text);
out.Close();
```

Call: `sar.bat notes.txt "colou?r" "color"` — real regex replace, no PowerShell, ~instant.

!!! warning "WSH can be disabled by policy"
    Locked-down environments sometimes disable Windows Script Host entirely (`HKLM\...\Windows Script Host\Settings\Enabled = 0`). Then `cscript` fails. PowerShell is more likely to be available, if slower.

## Debugging hybrids

- Run the batch part alone: it should reach the interpreter call. Add `echo` breadcrumbs before it.
- Run the other-language part directly: `powershell -File script.bat` or `cscript //E:JScript script.bat` — see its syntax errors without the batch wrapper in the way.
- Line endings **must** be CRLF or the header parsing breaks in one language or the other.
- Keep the boundary header at the very top; don't put `setlocal`/comments above it.

## Common mistakes

- **Reconstructing the header from memory** — use a tested one; the tokens matter.
- **LF line endings** — breaks the dual-parse.
- **Assuming execution policy is fine** — pass `-ExecutionPolicy Bypass` (works for `-File`/`-Command` without admin).
- **WSH disabled** — the JScript hybrid just fails; have a fallback or use PowerShell.
- **`%` and `!` in the embedded code** — the batch pass still expands `%var%` and (if delayed) `!var!` inside the file before the other language sees it. Escape or avoid.
- **Forehead-slap**: shipping a hybrid when two files would've been fine and far more maintainable.
- **Not deleting the temp `.ps1`** in the heredoc approach (and not using `%RANDOM%` → concurrent runs collide).

## Exercises

1. Build the "write temp `.ps1`, run, delete" wrapper. Pass it two arguments and have the PowerShell part use them. Confirm the temp file is gone afterward and that two concurrent runs don't clash.
2. Get a known-good batch+PowerShell hybrid header working. Have the PowerShell part print `$args` and an environment variable set by the batch part.
3. Build a batch+JScript hybrid that takes a string and prints whether it matches an email regex (real regex — the thing `findstr` can't do).
4. Write `sar.bat FILE PATTERN REPLACEMENT` as a batch+JScript hybrid doing an in-place regex replace. Test it on a file with `colour`/`color`.
5. Time 100 runs of a batch+JScript hybrid vs 100 runs of an equivalent batch+PowerShell hybrid. Note the startup difference.
6. Deliberately break the header's line endings (convert to LF) and observe how each language fails. Restore CRLF.

## Recap

- A hybrid file is valid in two languages at once via a header that one language treats as code and the other as a comment / skipped block.
- Batch + PowerShell: wrap the batch part in a PowerShell `<# ... #>` block comment; the batch part re-invokes `powershell` on `%~f0`. Use a tested header.
- Batch + JScript: `@if (@CodeSection == @Batch) @then ... @end` — JScript conditional-compilation skips it, batch runs it. `cscript //nologo //E:JScript "%~f0"`.
- JScript gives real regex/numbers/JSON with ~10 ms startup; PowerShell gives more power at ~200 ms; WSH may be policy-disabled.
- Simpler alternative: write a temp `.ps1` (`%RANDOM%` name), run `-File`, delete it.
- CRLF endings are mandatory; prefer two files unless single-file distribution is a hard requirement.

Next: [Packaging & distribution →](08-packaging-and-distribution.md)
