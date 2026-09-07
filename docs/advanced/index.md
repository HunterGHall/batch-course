# Advanced

You maintain batch scripts in production — build steps, installers, scheduled jobs. This level is about doing that well: recursion, the registry, interop with `wmic` and PowerShell, real text processing, performance, hybrid scripts, packaging, and debugging the cryptic failures.

## Lessons

| # | Lesson | You'll learn |
| --- | --- | --- |
| 1 | [Recursion & advanced `for`](01-recursion-and-advanced-for.md) | recursive subroutines, `for /R`, walking trees, guarding depth |
| 2 | [Robust argument parsing](02-robust-argument-parsing.md) | `--flags`, `/opt:value`, optional/positional args, quoting |
| 3 | [Working with the registry](03-working-with-the-registry.md) | `reg query`/`add`/`delete`, parsing `reg` output with `for /F` |
| 4 | [WMIC, `where` & calling PowerShell](04-wmic-where-powershell.md) | `wmic` (and its deprecation), `where`, one-line PowerShell escapes |
| 5 | [Text processing with `findstr`](05-text-processing-findstr.md) | `findstr` regex, filtering, counting, its many quirks |
| 6 | [Performance & pitfalls](06-performance-and-pitfalls.md) | pipe subshells, `for` overhead, when batch is the wrong tool |
| 7 | [Hybrid scripts](07-hybrid-scripts.md) | batch + PowerShell and batch + JScript in one file |
| 8 | [Packaging & distribution](08-packaging-and-distribution.md) | self-contained scripts, `%~f0`, config files, signing note |
| 9 | [Debugging techniques](09-debugging-techniques.md) | `echo` tracing, `cmd /k`, reading the real error |
| 10 | [Capstone: deployment automation](10-capstone-deployment-automation.md) | a production-grade multi-step deploy script |

## When you're done

You know batch as well as it is worth knowing. The most valuable advanced skill is judgment: [Lesson 6](06-performance-and-pitfalls.md) is about when to stop writing batch and reach for PowerShell or a real language.
