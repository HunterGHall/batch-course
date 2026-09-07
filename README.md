# The Batch Scripting Course

A self-paced course on **Windows batch scripting** (`.bat` / `.cmd`), beginner to advanced. Made by Claude.

**Read it online:** https://hunterghall.github.io/batch-course/

## Structure

- **Beginner** (10 lessons) — first script, `echo`, `set`, arguments, `if`, `for`, file operations, redirection, subroutines, capstone (file organizer).
- **Intermediate** (10 lessons) — delayed expansion, string manipulation, `for /F`, error handling, `setlocal` scope, pseudo-arrays, date/time/math, menus, scheduling, capstone (backup utility).
- **Advanced** (10 lessons) — recursion, argument parsing, the registry, WMIC/PowerShell interop, `findstr`, performance & pitfalls, hybrid scripts, packaging, debugging, capstone (deployment automation).

Each lesson: *What & why → Examples → Common mistakes → Exercises → Recap*.

## Building locally

```bash
pip install mkdocs-material
mkdocs serve
```

Then open http://127.0.0.1:8000. `mkdocs build --strict` produces the static `site/`.

Pushing to `main` deploys to GitHub Pages via `.github/workflows/deploy.yml`.

## License

Use freely for learning and teaching.
