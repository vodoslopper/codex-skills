---
name: lineup
description: Create, inspect, edit, and troubleshoot Lineup orchestration manifests (`LM.toml` and `LM.local.toml`) for tasks running on host, SSH, Docker, Podman, Incus, or VML workers, with a strong preference for reusable Lineup modules over ad hoc shell commands. Use for Lineup manifest structure, built-in or project modules, workers, tasklines, tasksets and dependencies, variables and Tera templates, networks, storages, file transfers, retries, conditional execution, nested manifests, cleanup, and resume behavior.
---

# Lineup manifests

Work with Lineup's TOML manifests while preserving the surrounding project's conventions. Prefer the smallest manifest structure that expresses the requested workflow.

## Follow the workflow

1. Locate `LM.local.toml` or `LM.toml` and adjacent Lineup manifests. Lineup selects `LM.local.toml` automatically when it exists; otherwise it uses `LM.toml`.
2. Read the complete target manifest and any local manifests referenced through `[use]`, `run-taskline`, `run-taskset`, or `run-lineup`. Inspect nearby manifests and module files for project conventions.
3. Inventory reusable modules before writing tasks. Check existing `[use]` imports, project-local TOML modules, and Lineup's installed modules. Read [references/modules.md](references/modules.md) for built-in contracts and module selection.
4. Read [references/manifest.md](references/manifest.md) before creating a manifest, introducing an unfamiliar section/task/engine, or diagnosing schema behavior.
5. Model execution explicitly:
   - define execution targets under `[workers]`;
   - put reusable sequential steps in `[[tasklines.NAME]]`;
   - schedule tasks under `[taskset.NAME]` and order them with `requires`;
   - keep shared inputs in `[vars]` and task-local inputs in `vars.*`;
   - import reusable variables or tasklines with `[use]`.
6. Prefer a module taskline whenever its contract covers the operation. Import it once under `[use]`, call it with `run`, and pass its declared variables. Use direct `run-taskline` when importing the whole module would be unnecessary or would cause name collisions.
7. Write a project-local module when a multi-step operation is reusable across tasklines or manifests. Use a pure `shell` or `exec` task only for a genuinely one-off operation or when no suitable module exists.
8. Preserve valid TOML types. Do not stringify arrays, objects, numbers, or booleans unless template rendering requires a string. Quote dotted or otherwise special TOML keys.
9. Treat template-bearing strings as Tera expressions. Shell-escape variable values with `| quote` or `| q` when interpolating them into shell commands.
10. Validate safely. Parse TOML locally when a TOML parser is available, then run Lineup only when execution is authorized and its workers/resources are safe to create. Prefer a debug engine or an isolated test manifest when execution would mutate systems.
11. Report validation performed, runtime assumptions, and anything that was not executed.

## Apply design rules

- Search modules first. Prefer `apt-get.install`, `systemctl.enable`, `useradd`, `sed`, `wait`, or another existing module taskline over spelling out the equivalent command.
- Define `host` as the string exception, `[workers.NAME] engine = "host"`. Define every other worker engine as a nested table such as `[workers.NAME.engine.vml]`, `[workers.NAME.engine.podman]`, or `[workers.NAME.engine.ssh]`.
- Use `shell.cmd` directly for shell expressions, pipelines, redirections, substitutions, compound commands, and tests. Do not wrap these in `exec.args = ["sh", "-c", ...]` or `exec.args = ["sh", "-eu", "-c", ...]`.
- Use `exec.args` for a remaining one-off command only when every argument can be passed literally without shell parsing.
- Set `shell.stdout.print = true` only when output should be user-visible; Lineup otherwise logs command output.
- Use `condition` for worker-side preconditions and `if` for rendered boolean conditions when supported by the existing manifest/version.
- Use `ensure.vars` at taskline boundaries to document required inputs and their types.
- Use `items` for genuine repetition. Set `parallel = false` when ordering or shared state makes concurrent items unsafe.
- Limit taskset execution with `workers` regexes. Remember that taskset tasks otherwise run on all workers.
- Use `when = "before"` or `when = "after"` for global setup/teardown phases instead of large dependency lists.
- Use `try` only for plausibly transient failures, with bounded attempts and cleanup when partial state can remain.
- Resolve relative paths from the directory containing the manifest. Use `{{ manifest_dir }}` when a host-side path must be explicit.
- Avoid embedding secrets in manifests. Pass values through the environment/project mechanism or `--extra-vars` when appropriate, and avoid printing them.
- Do not invent fields. Check the reference and the installed `lineup --help` or version when repository usage conflicts with upstream examples.

## Reuse modules

Prefer importing related tasklines once:

```toml
[use]
tasklines = ["apt-get", "systemctl"]

[[tasklines.setup]]
run = "apt-get.install"
vars.packages = ["nginx"]

[[tasklines.setup]]
run = "systemctl.enable"
vars.services = ["nginx"]
```

Use a taskline directly when only one operation is needed:

```toml
run-taskline = { module = "sed", taskline = "" }
vars.file = "/etc/example.conf"
vars.expression = "s/^enabled=.*/enabled=true/"
```

Do not duplicate a module's internal shell implementation in the calling manifest. Pass its public variables and let the module own validation, quoting, defaults, and sequencing.

Prefer the native shell task when the command is already shell code:

```toml
shell.cmd = '[ "$(id -un)" = builder ]'
```

Do not express the same operation through a shell executable:

```toml
# Avoid
exec.args = ["sh", "-eu", "-c", "[ \"$(id -un)\" = builder ]"]
```

Lineup's shell task already runs a shell command and checks failure by default. Add an explicit shell invocation only when a particular non-default shell or shell-specific option is itself required by the workflow.

## Use the CLI conservatively

- Initialize a basic manifest with `lineup init`; it writes `LM.toml` by default.
- Select another manifest with `lineup --manifest PATH`.
- Override values with repeated `--extra-vars NAME=VALUE` arguments.
- Preserve workers after a successful run with `--no-cleanup`; request cleanup with `--cleanup` or `lineup cleanup`.
- Use `--cleanup-before` only when removing existing Lineup-managed state is intended.
- Use `--resume` to persist filesystem variables and completed-task history under `.lineup` and skip completed tasks on the next run.
- Consult `lineup --help` for task filtering and history options because these are operational controls, not manifest schema.

## Review changes

Check all of the following before handing off:

- every `requires` target, taskline name, worker selector, module, and relative path resolves;
- every module call uses an exported taskline and supplies its required variables with compatible types;
- each task has exactly one intended task type;
- shared workers using the same VM/container coordinate `name` and `setup` correctly;
- task ordering matches data and resource dependencies;
- cleanup behavior will not delete state the workflow expects to retain;
- template expressions receive variables of the expected type;
- shell interpolation uses quoting and does not expose secrets;
- TOML remains parseable and names containing dots or template syntax are quoted.
