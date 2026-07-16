---
name: bash
description: Write, edit, review, or troubleshoot shell scripts using strict portable POSIX shell conventions. Use for shell-script files and snippets, including requests involving Bash or sh, while preferring POSIX syntax unless the task explicitly requires shell-specific features.
---

# Bash

Apply these rules when creating or modifying shell scripts:

- Start executable scripts with exactly `#!/bin/sh -eu`.
- Prefer syntax and utilities specified by POSIX. Do not introduce Bash-only features such as arrays, `[[ ... ]]`, `(( ... ))`, process substitution, or brace expansion unless the task explicitly requires Bash or another non-POSIX shell.
- Quote expansions unless splitting or pathname expansion is intentional. Prefer `"$name"` to `"${name}"` when braces are unnecessary.
- Use braces only when needed to delimit a parameter name or perform a parameter expansion, for example `"${name}_suffix"` or `"${value:-default}"`.
- Account for `set -e` and `set -u`, which are enabled by the shebang. Structure expected failures inside explicit conditionals and provide defaults for intentionally optional parameters.
- Keep scripts simple and readable. Prefer shell built-ins where they express the operation clearly, and avoid needless pipelines and subprocesses.

Preserve an explicitly requested non-POSIX feature when replacing it would change the intended behavior. If a requirement conflicts with portability or strict-mode behavior, explain the conflict briefly and implement the user's stated requirement.

Validate changed scripts with `sh -n`. Run a relevant safe test when practical, and use a POSIX-oriented linter if one is already available in the project.
