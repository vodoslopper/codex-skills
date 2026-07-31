---
name: tmp
description: Keep temporary files, directories, sockets, caches, logs, and intermediate artifacts created by Codex while performing a task under `$TMP/codex`. Use when Codex runs commands or tools that may create temporary operational data. Do not apply this policy to source code, configuration, examples, or other deliverables Codex creates for the user.
---

# Use `$TMP/codex`

Place Codex's own temporary files, directories, sockets, caches, logs, and other intermediate artifacts under `$TMP/codex`. Do not use `/tmp`, the workspace, `$TMP` directly, or another location for Codex's temporary operational data.

## Apply the rule

1. Assume `$TMP` is already configured and created. Do not create, inspect, or validate it.

## Handle constraints

- Do not fall back to `/tmp` or another location.
- Do not use symlinks, bind mounts, or other indirection to place temporary content outside `$TMP/codex`.
- When running third-party tools, override their temporary location when practical so artifacts from Codex's invocation remain under `$TMP/codex`.

## Limit the scope

- Apply this rule to Codex's actions and execution environment only.
- Do not impose `$TMP/codex` paths on source code, scripts, configuration, documentation, examples, tests, templates, or other files created or edited for the user.
- Preserve the project's own temporary-path conventions in deliverables unless the user explicitly asks to change or review them.
- Before running a created file during the task, configure that invocation's temporary environment under `$TMP/codex` when possible; do not alter the file merely to satisfy this skill.
