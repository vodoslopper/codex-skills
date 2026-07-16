---
name: tmp
description: Require all temporary files, directories, sockets, caches, logs, and intermediate artifacts to be located under `$TMP/codex`. Use whenever a task may create temporary data, when adapting commands or code that uses another temporary location, or when reviewing work for temporary-path compliance.
---

# Use `$TMP/codex`

Create every temporary file, directory, socket, cache, log, and other intermediate artifact under `$TMP/codex`. Do not use `/tmp`, the workspace, `$TMP` directly, or any other location for temporary data.

## Apply the rule

1. Assume `$TMP` is already configured and created. Do not create, inspect, or validate it.

## Handle constraints

- Do not fall back to `/tmp` or another location.
- Do not use symlinks, bind mounts, or other indirection to place temporary content outside `$TMP/codex`.
- Account for third-party tools that choose their own temporary location and override it explicitly.

## Review commands and code

Reject or rewrite any pattern that creates temporary content outside `$TMP/codex`. Confirm that every temporary path resolves beneath `$TMP/codex`.
