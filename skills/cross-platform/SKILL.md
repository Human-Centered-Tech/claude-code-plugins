---
name: cross-platform
description: Audit and fix a Node.js project for cross-platform compatibility (Windows + macOS + Linux). Use when setting up a new project or when someone reports scripts not working on their OS.
---

Audit the current project for cross-platform issues and fix them. Check each of the following:

## 1. npm scripts using inline environment variables

Look in `package.json` for scripts that set env vars with bare `KEY=value` syntax (e.g. `NODE_ENV=production node ...`). This fails on Windows PowerShell.

**Fix**: Install `cross-env` as a devDependency and prefix with `cross-env`:
```
"start": "cross-env NODE_ENV=production node dist/index.cjs"
```

## 2. npm scripts using Unix shell operators

Look for scripts that use `&&`, `||`, `|`, or `;` to chain commands, or use Unix-only commands like `rm -rf`, `cp`, `mv`, `mkdir -p`, `cat`, `chmod`.

**Fix**: Use `rimraf` instead of `rm -rf`, `cpy-cli` or `copyfiles` instead of `cp`, `mkdirp` instead of `mkdir -p`, or use `shx` as a general-purpose wrapper. For complex scripts, move logic into a `.ts` or `.js` build script that uses Node.js APIs (`fs`, `path`, `child_process`).

## 3. Hardcoded path separators

Search the codebase for hardcoded `/` or `\\` in file paths constructed in code.

**Fix**: Use `path.join()` or `path.resolve()` from Node's `path` module instead.

## 4. Shell-specific shebangs or scripts

Look for `.sh` scripts referenced in package.json or config without `.cmd`/`.ps1` equivalents.

**Fix**: Rewrite as Node.js scripts (`.ts`/`.js`) or use `cross-env`/`shx` wrappers.

## Output

After auditing, report:
- A list of issues found (with file and line number)
- Fixes applied
- Any remaining items that need manual attention
