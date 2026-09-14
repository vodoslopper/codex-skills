# Execution control

Use these controls while developing or diagnosing a manifest. The behavior below was verified with Lineup 0.1.1; check `lineup --help` when the installed version differs.

## Run only the needed work

- `--taskset-skip TASK...` omits named taskset tasks. Address a nested task with dot-separated names.
- `--taskset-first TASK` and `--taskset-last TASK` select an inclusive interval of the taskset graph. They can be combined to run one task. Dependencies outside the selected interval are not run automatically, so use this only when their state already exists.
- `--taskline-skip NAME...` bypasses every invocation of the named taskline. `NAME=VALUE` supplies a JSON-encoded replacement result when callers need one; without a value, the current `result` is propagated or null is returned.
- `--skip-history FILE` skips exact structural paths listed in a history file produced by `--completed-tasks-append FILE`.

Prefer taskset filters for top-level workflow selection. Use taskline skipping only when bypassing every call with that name is intentional. Filters do not prove that omitted prerequisites or outputs exist.

## Preserve or clean workers

Lineup normally follows its configured cleanup default. With the standard `cleanup = true` configuration:

- `--no-cleanup` preserves Lineup-managed workers, networks, storages, and `.lineup` state after a successful run.
- `--cleanup` requests cleanup after success when configuration disables it.
- `--cleanup-before` removes existing managed state before execution; do not combine it with an expectation that preserved worker or resume state will be reused.
- `lineup cleanup --manifest PATH` explicitly removes the manifest's managed artifacts and `.lineup` state.

Cleanup after failure is not reached by the normal run path. Inspect partial state before deciding whether to resume or clean it.

## Resume safely

Run with both `--resume --no-cleanup` when completion state must survive a successful development run. `--resume` uses the manifest directory's `.lineup` directory for filesystem variables and appends completed paths to `.lineup/completed-tasks.jsonl`; it also reads that file and skips exact matches. With normal automatic cleanup, a successful `--resume` run removes `.lineup`, so there is nothing to resume next time. A failed run leaves completed entries available.

The history file is JSON Lines. Each line is one complete JSON array describing a structural execution path. Examples:

```json
[{"Taskset":"setup"}]
[{"Taskset":"setup"},{"Taskline":"install"}]
[{"Taskset":"setup"},{"Taskline":"install"},{"TasklineEntry":0}]
```

Taskline entries use zero-based positions. Imported taskline identities are recorded as `module-path:taskline-name` (or the module path when its taskline name is empty). A successful taskline writes entries for its completed positions and then a taskline-level record; a successful taskset also writes a taskset-level record. Duplicate lines are harmless when read because matching uses a set.

History contains no command digest or manifest fingerprint. Renaming a taskset or taskline changes its identity and makes old records stop matching, while changing a command, variables, worker selection, dependencies, or taskline contents under the same structural identity does not invalidate completion. Inserting or reordering taskline entries also changes what a saved numeric position refers to.

After editing a completed portion of the manifest, do one of the following before resuming:

1. Reset all resume and filesystem-variable state with `lineup cleanup --manifest PATH`, then start again.
2. Preserve workers but remove or rewrite stale lines in `.lineup/completed-tasks.jsonl`. Remove the enclosing taskset-level and taskline-level records as well as affected entry records; either broader record can skip the whole edited scope.
3. Keep an explicit, reviewed history file with `--completed-tasks-append FILE`, edit a copy to contain only safe completions, and pass it with `--skip-history FILE`.

Treat manual history editing as exact JSONL data editing: keep one valid array per line and retain only paths whose effects still exist. Back up the file first when its state matters.
