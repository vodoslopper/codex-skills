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

## Path and composition rules

- Resolve manifest-relative modules and file transfers deliberately; use `manifest_dir` for explicit host paths.
- Imported module names receive a default prefix; hyphens become underscores for variable prefixes.
- Use `run` when imports have already loaded a taskline; use `run-taskline` for direct file/module dispatch.
- For nested tasksets, expose outer workers through `provide-workers` and map names when the nested manifest expects different names.
- Quote TOML table keys containing dots, spaces, or templates, for example `[workers.'node-{{ item }}']`.
