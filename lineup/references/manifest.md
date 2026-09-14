# Lineup manifest reference

Use this reference for the current upstream `LM.toml` vocabulary. Confirm behavior against the installed Lineup version when a project uses newer or older fields.

## Core structure

```toml
[use]
tasklines = ["apt-get"]

[vars]
message = "hello"

[workers.local]
engine = "host"

[[tasklines.show]]
shell.cmd = "printf '%s\\n' {{ message | quote }}"
shell.stdout.print = true

[taskset.show]
run = "show"
workers = ["local"]
```

- `[use]`: import `vars` or `tasklines` from embedded modules or manifest paths. Entries may be module-name strings or `{ module, prefix, items }` tables. Paths beginning with `.` or `/` are treated as paths. An empty prefix imports names unchanged.
- `[vars]`: global variables of any TOML type. Strings normally render as Tera templates.
- `[networks.NAME]`: managed virtual networks; currently supports `engine.incus` with `address` and `nat`.
- `[storages.NAME]`: managed volumes; currently supports `engine.incus` with `pool` and `copy`.
- `[workers.NAME]`: task execution targets.
- `[default.worker]`: default worker configuration.
- `[[tasklines.NAME]]`: sequential arrays of tasks. `[[taskline]]` is the unnamed default taskline.
- `[taskset.NAME]`: tasks scheduled concurrently unless constrained with `requires`.
- `[extend]`: currently supports ordered `vars.maps` overlays.

## Variables and templates

Define a variable as `kind % name: type`. Both kind and type are optional.

Types: `bool|b`, `number|n`, `u64|u`, `i64|i`, `f64|f`, `string|s`, `array|a`, and `object|o`. Join types with `|` for a union.

Kinds:

- `fs`: store the value on the filesystem and read it with `fs(name='x')` or `'x' | fs`;
- `json|j`: decode a JSON string;
- `yaml`: decode a YAML string;
- `raw|r`: suppress template rendering in the value.

Special variables include `item`, `manifest_dir`, `result`, `taskline`, and `worker`.

Lineup uses Tera templates in most strings. Useful additions include:

- filters: `basename`, `dirname`, `cond`, `fs`, `is_empty`, `json|j`, `lines`, `quote|q`, `re_match`, `re_sub`, and `to_list`;
- functions: `confirm`, `fs`, `input`, `host_cmd`, and `tmpdir`.

Quote values inserted into shell commands:

```toml
shell.cmd = "apt-get install -y {{ packages | quote }}"
```

## Workers and engines

Common engine fields are `name`, `setup`, and `exists` (`fail`, `ignore`, or `replace`).

The `host` engine is the sole string-form exception. Define every other engine as a table nested below the worker's `engine` key.

- `host`: run on the local host; define it as `[workers.local] engine = "host"`.
- `ssh`: `host`, `port`, `user`, `key`, `ssh-cmd`.
- `docker`: `image`, `load`, `memory`, `user`.
- `podman`: `image`, `load`, `memory`, `pod`, `user`.
- `incus`: `image`, `copy`, `net`, `nproc`, `memory`, `hostname`, `storages`, `user`.
- `vml`: `vml-bin`, `memory`, `image`, `net`, `nproc`, `parent`, `user`.
- `dbg`: accept and print configuration without provisioning a real worker.

To expose one VM/container as multiple workers, give them the same engine `name` and set `setup = false` on secondary workers.

Incus storage mounts use `path`, `readonly`, `pool`, and `volume`. Network attachment commonly uses `net.network` and `net.address`.

## Tasksets

Taskset entries are tasks. Important scheduling fields:

- `requires = ["NAME"]`: run after named tasks;
- `workers = ["REGEX"]`: restrict workers; default matches all;
- `provide-workers = ["NAME"]`: expose workers to a nested `run-taskset`;
- `when = "before" | "after"`: place a task in an independently ordered leading or trailing phase.

Tasklines run sequentially; taskset tasks and item expansions may run concurrently. State ordering explicitly.

## Common task fields

Tasks accept `condition`, `items`, `items-var`, `parallel`, `vars`, `export-vars`, `clean-vars`, `try`, and `table`. Existing manifests may also use a rendered `if` condition.

`items` forms:

- array of strings or integers;
- `{ start, end, step }` sequence;
- `items.json` containing rendered JSON;
- `items.var` naming an array/object variable;
- `{ command = "..." }`, whose host stdout is split into lines.

Retry configuration uses `try.attempts`, `try.sleep`, and optionally `try.cleanup.task`.

## Task types

- `run = "TASKLINE"`: run a taskline already loaded in the manifest.
- `run-taskline = { module = "MODULE_OR_PATH", taskline = "NAME" }`: run a taskline directly from a module/file.
- `run-taskset.module = "PATH"`: run another manifest's taskset with provided workers. Worker selection can be `"all"`, named selection, or mappings such as `run-taskset.worker.maps = [["outer", "inner"]]`.
- `run-lineup.manifest = "PATH"`: run a separate Lineup manifest; supports `exists`, `cleanup`, and passed `vars`.
- `shell.cmd` or `shell.command`: execute a shell string. Use it directly for tests, substitutions, pipelines, redirections, compound commands, and other shell syntax; do not put such code behind `exec.args = ["sh", "-c", ...]`.
- `exec.args`: execute a literal argument array without shell parsing. Use it when the executable and every argument are discrete values and no shell syntax is needed.
- `test.commands`: run shell strings, argument arrays, or command tables and return whether all checked commands succeeded.
- `file`: copy `src` or rendered `content` to worker `dst`; optional `chown` and `chmod`.
- `get`: copy worker `src` to host `dst`; default destination is beside the manifest.
- `ensure.vars`: require variables, optionally with types.
- `break`, `dummy`, `debug`, `trace`, `info`, `warn`, `error`: control flow/result and logging.
- `special.start`, `special.stop`, `special.restart`: engine-specific lifecycle operations.

Shell/exec command controls include `check`, `stdin`, `stdout`, `stderr`, `success-codes`, `success-matches`, `failure-matches`, and `result`. Output controls accept `print` and `log`. Result controls include `lines`, `matched`, `return-code`, `stream`, and `strip`.

Match formulas combine `and`/`or` with `err-re`, `out-re`, or `any-re` regex leaves.

## Validate commands natively

Do not build a large shell script merely to inspect exit status or pipe captured output into `grep`. A `shell` or `exec` task checks for return code 0 by default. Declare other accepted codes with `success-codes`:

```toml
[[tasklines.verify]]
exec.args = ["diff", "expected.txt", "actual.txt"]
exec.success-codes = [0, 1]
```

Use `success-matches` when output must contain a regex and `failure-matches` when matching output should fail the task. Select `out-re`, `err-re`, or `any-re`; combine leaves with `and` or `or` when necessary:

```toml
[[tasklines.verify]]
exec.args = ["mytool", "--version"]
exec.success-matches.out-re = '^mytool 2\.4\.1(?:\r?\n)?$'

[[tasklines.verify]]
shell.cmd = "mytool check --verbose"
shell.failure-matches = { or = [
  { out-re = "FAILED" },
  { err-re = "fatal:" },
] }
```

Regexes see the raw captured stream, including its trailing newline. Account for it when anchoring a pattern. TOML literal strings are convenient because regex backslashes remain literal; in a double-quoted TOML basic string, write `\\.` to pass `\.` to the regex engine.

Use `test.commands` to express a short collection of independent command assertions. Strings and `cmd` tables are shell commands; argument arrays and `args` tables execute directly:

```toml
[[tasklines.verify]]
test.commands = [
  ["test", "-s", "/etc/example.conf"],
  { args = ["mytool", "status"], success-matches = { out-re = "ready" } },
  { cmd = "systemctl is-active example", success-matches = { out-re = '^active(?:\r?\n)?$' } },
]
```

With its default checking, `test.commands` fails at the first unsuccessful command. Prefer separate taskline entries when each operation changes state, has distinct retry or condition behavior, should be resumable independently, or benefits from its own error context. Do not split commands that depend on an earlier `cd`, shell variable, trap, or other process-local state unless that state is expressed again in each entry.

## Capture and persist results

Every command task returns a typed `result` for the next taskline entry. Output is a stripped array of stdout lines by default. Change the shape with command `result` fields:

- `lines = false` returns one string instead of an array;
- `stream = "stderr"` captures stderr instead of stdout;
- `strip = false` preserves trailing whitespace;
- `return-code = true` returns the exit code and takes precedence over output;
- `matched = true` returns whether `success-matches` or `failure-matches` matched.

For example, inspect an allowed nonzero return code without reparsing shell output:

```toml
[[tasklines.check]]
exec.args = ["cmp", "expected", "actual"]
exec.check = false
exec.result.return-code = true

[[tasklines.check]]
info.msg = "cmp returned {{ result }}"
```

Set `result-fs-var = "NAME"` on any task to write its result as JSON. Read it later with `fs(name='NAME')` or `'NAME' | fs`. This is useful when nested tasklines, filesystem-backed state, or resume behavior make the immediately preceding `result` insufficient:

```toml
[[tasklines.discover]]
exec.args = ["printf", "api\\nworker\\n"]
result-fs-var = "service_names"

[[tasklines.discover]]
info.msg = "services={{ fs(name='service_names') | json }}"
```

Filesystem-variable names may contain only alphanumeric characters and underscores. With `--resume`, filesystem variables live under the manifest's `.lineup` state; apply the stale-state precautions in [execution-control.md](execution-control.md#resume-safely).

## Drive tasks and workers from data

Use `items` when one task should repeat over scalar values. Sources may be a literal array, a half-open numeric sequence, JSON, an array or object variable, or lines printed by a host command. Change the loop variable with `items-var`:

```toml
[[tasklines.check]]
exec.args = ["systemctl", "is-active", "{{ service }}"]
items-var = "service"
items.var = "services"
parallel = false
```

Items run in parallel by default. Set `parallel = false` for deterministic order, shared state, rate limits, or operations that must not overlap. An item-expanded task returns an object keyed by the rendered item value.

Use `table` when each repetition needs several named fields. Each row is available as `row`, and the task returns an array of row results:

```toml
[[tasklines.deploy]]
exec.args = ["install-service", "{{ row.name }}", "{{ row.port }}"]
table = [
  { name = "api", port = 8080 },
  { name = "metrics", port = 9090 },
]
parallel = false
```

A table may instead be produced by a command and decoded as `json`, `yaml`, `toml`, or `csv`:

```toml
[[tasklines.deploy]]
exec.args = ["install-service", "{{ row.name }}", "{{ row.port }}"]
table = { command = "inventory export --format json", format = "json" }
```

Both `items.command` and `table.command` execute on the Lineup host, not on the selected worker. Do not use them for worker-only discovery or interpolate untrusted data into their shell command. Prefer `items.var` or `items.json` when the data is already available in the manifest context.

Workers support the same scalar expansion plus lookup tables:

- `table-by-item` selects the row whose `item` field matches the current expansion and exposes it as `row_by_item`;
- `table-by-name` selects the row whose rendered `name` matches the final worker name and exposes it as `row_by_name`;
- `[default.worker]` can provide shared `items`, lookup tables, or an engine to workers that omit them.

Use these tables for per-worker image, address, user, or resource differences instead of copying complete worker definitions. Keep lookup keys unique; unmatched lookups do not fail automatically, so validate required row fields through rendering or explicit checks.

## Control context, flow, and retries

Task-local `vars` normally extend the inherited context. Use `clean-vars = true` for an isolated task that must not see preceding or global variables; only item/row data and the task's own variables are then added. Do not enable it on a task whose templates depend on inherited values.

Use `export-vars = ["NAME", ...]` to propagate selected task-local variables to subsequent entries in the same running taskline. It does not export those variables back to the caller of a nested `run` or `run-taskline`; use the nested taskline's returned `result` for that boundary. With `table` expansion, each exported variable is folded into an array; with `items`, it is folded into an object keyed by item.

`condition` (also accepted as `cond` or `if` in Lineup 0.1.1) renders before execution. The literal strings `true` and `false` run or skip directly; any other rendered value is executed as a shell command on the worker, and command failure skips the task while preserving the preceding result. Use it for a worker-side precondition, not for hiding an unexpected failure in the task itself.

Use `break = {}` for an intentional successful exit from the innermost taskline. It returns the preceding result by default; `break.result` can replace it, and `break.taskline` can name an enclosing taskline. Prefer `break` over a failing shell command when reaching a condition means the requested outcome is already satisfied.

Retry behavior is version-sensitive. In the verified Lineup 0.1.1 implementation, the task runs once before the retry loop, and `try.attempts` controls additional executions after failure. Thus `try.attempts = 2` can execute the task three times total. Verify this against the installed version rather than assuming the field is a total-attempt count.

`try.sleep` accepts fractional seconds. `try.cleanup.task` runs after the delay and before each retry; cleanup failure is logged as a warning and does not stop the retry. Keep retries bounded and use cleanup only for idempotent recovery of partial state.

## Path and composition rules

- Resolve manifest-relative modules and file transfers deliberately; use `manifest_dir` for explicit host paths.
- Imported module names receive a default prefix; hyphens become underscores for variable prefixes.
- Use `run` when imports have already loaded a taskline; use `run-taskline` for direct file/module dispatch.
- For nested tasksets, expose outer workers through `provide-workers` and map names when the nested manifest expects different names.
- Quote TOML table keys containing dots, spaces, or templates, for example `[workers.'node-{{ item }}']`.
