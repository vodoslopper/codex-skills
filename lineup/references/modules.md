# Lineup modules

Use a module before writing an equivalent command directly. Modules package input validation, quoting, defaults, conditional steps, and reusable sequencing.

## Resolution and invocation

A module name such as `apt-get` resolves to `MODULE.toml` in Lineup's configured modules directory. Lineup installs its embedded modules there. A module beginning with `.` or `..` resolves relative to the current manifest; an absolute path remains absolute.

Import tasklines for repeated use:

```toml
[use]
tasklines = ["apt-get", "systemctl"]

[[tasklines.setup]]
run = "apt-get.install"
vars.packages = ["curl", "jq"]
```

The default import prefix is the module/file name. Use `{ module, prefix, items }` to change the prefix or select tasklines:

```toml
[use]
tasklines = [
  { module = "apt-get", items = ["install", "remove"] },
  { module = "./modules/application.toml", prefix = "app" },
]
```

Set `prefix = ""` to import names unchanged. Avoid this when collisions are possible. For variable prefixes, Lineup converts `-` to `_`, and explicit prefixes must contain only alphanumerics and `_`.

Call one taskline without importing it:

```toml
run-taskline = { module = "apt-get", taskline = "install" }
vars.packages = "git-core"
```

## Built-in modules

### `apt-get`

Module variable `update` defaults to `true`. Set task-local `vars.update = false` to suppress cache updates where appropriate. Import it separately through `use.vars` only when the caller itself needs the default as `apt_get.update`.

| Taskline | Required variables | Operation |
| --- | --- | --- |
| `update` | none | Run `apt-get update` |
| `dist-upgrade` | `update: bool` | Optionally update, then dist-upgrade |
| `install` | `packages: array|string`, `update: bool` | Optionally update, then install |
| `reinstall` | `packages: array|string`, `update: bool` | Optionally update, then reinstall |
| `remove` | `packages: array|string` | Remove packages |

When imported with its default prefix, `run = "apt-get.install"` uses the module's `update` default unless the calling task overrides it.

### `apt-repo`

| Taskline | Required variables | Operation |
| --- | --- | --- |
| `add` | `task: u64|string` | Add a task repository |
| `rm` | `task: u64|string` | Remove a task repository |
| `clean` | none | Remove all task repositories |

These tasklines use argument arrays, preserving task values as individual arguments.

### `sed`

The unnamed default taskline requires `file: string` and `expression: string`. It performs extended-regexp in-place editing. If `validate` is defined, it runs that validation command after editing.

```toml
run-taskline = { module = "sed" }
vars.file = "/etc/service.conf"
vars.expression = "s/^port=.*/port=8080/"
vars.validate = "service --check-config"
```

### `systemctl`

Module variable `now` defaults to `true`. Import it separately through `use.vars` only when the caller itself needs the default as `systemctl.now`.

| Taskline | Required variables | Operation |
| --- | --- | --- |
| `enable` | `services: array|string`, `now: bool` | Enable services, with `--now` by default |
| `disable` | `services: array|string`, `now: bool` | Disable services, with `--now` by default |
| `restart` | `services: array|string` | Restart services |
| `start` | `services: array|string` | Start services |
| `stop` | `services: array|string` | Stop services |

### `useradd`

The unnamed default taskline requires `user: string`, `groups: array|string`, `flags: array|string`, and `copy_root_key: bool`. Defaults are empty `groups` and `flags`, and `copy_root_key = true`. It creates the user if missing and can copy root's authorized keys.

Importing it as `tasklines = ["useradd"]` exposes the default taskline as `useradd`:

```toml
[[tasklines.setup]]
run = "useradd"
vars.user = "builder"
vars.groups = ["wheel"]
vars.copy_root_key = false
```

Set `copy_root_key = false` unless copying root's SSH authorization is explicitly required.

### `wait`

The unnamed default taskline polls a command until it succeeds or reaches its iteration limit.

Defaults:

- `command = "systemctl is-system-running --wait"`;
- `count = 120`;
- `sleep = 1`.

Importing it exposes `run = "wait"`. Override `command`, `count`, or `sleep` as task variables. Prefer this module to hand-written polling loops.

## Project-local modules

Extract repeated behavior into a TOML module when it:

- appears in multiple tasklines or manifests;
- contains validation, retries, or conditional sequencing worth centralizing;
- represents a named domain operation rather than incidental glue.

Give modules a small public surface:

- declare required inputs with `ensure.vars`;
- provide safe defaults in `[vars]`;
- keep helper tasklines private by prefixing names with `_`;
- use `shell.cmd` directly for shell expressions rather than wrapping them in `exec.args = ["sh", "-c", ...]`;
- use `exec.args` internally where the executable and all arguments are literal and shell parsing is unnecessary;
- quote values used in shell strings;
- expose taskline names that describe outcomes, not implementation details.

Before creating a module, search project manifests and installed modules for an existing equivalent. Before modifying a shared module, inspect all callers because its variable defaults and taskline names are an API.
