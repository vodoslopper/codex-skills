---
name: vml
description: Run and manage virtual machines with the local `vml` CLI. Use when Codex needs to create or run a VM, inspect VM state, start or stop VMs, execute commands over SSH, transfer files, discover or add cloud images, manage VM images, access the QEMU monitor, or remove and clean up VMs.
---

# VML

Use the installed `vml` command to perform VM work directly. VML manages QEMU/KVM machines as directories containing a required `vml.toml` marker/config and a `disk.qcow2` disk. Inspect live state instead of assuming that a VM or image exists.

Read [references/project.md](references/project.md) when configuring VML, selecting VM groups, using cloud-init or networking, troubleshooting dependencies, or extending the CLI.

## Workflow

1. Verify the CLI with `command -v vml` and, when compatibility matters, `vml --version`.
2. Inspect relevant state with `vml list`, `vml show`, or `vml image list`.
3. Write ordinary commands directly from the syntax below. Do not routinely probe commands with `--help` first.
4. Select the smallest command that satisfies the request.
5. Run it and verify the result with `vml show <name>`, `vml list`, or the command's exit status and output.

Use `vml <subcommand> --help` only when the invocation is complex, its syntax is not covered here, an installed-version difference matters, or a command fails with a usage/option error. For nested image commands, use `vml image <action> --help`. After an error, inspect the narrowest relevant help and retry with the corrected command.

## Command Syntax

VML uses `vml [global-options] <subcommand> [subcommand-options] [name]`. Put global options such as `--host`, `--all-vms`, `--vm-config`, and `--minimal-vm-config` before the subcommand. Put subcommand options after it.

- Single VM: `vml <start|stop|show|remove|ssh|monitor> [options] <name>`.
- Multiple VMs: use the subcommand's `--names <name>...`, `--parents <parent>...`, or `--tags <tag>...` selector; use global `--all-vms` only when explicitly intended.
- Create and run: `vml run [options] <name>`; create without starting: `vml create [options] <name>`.
- Guest command: `vml ssh [--check] --cmd '<command>' <name>`. Without `--cmd`, SSH is interactive.
- Copy to a guest: `vml rsync-to --sources <source>... [--destination <destination>] <name>`; use `--template <template>` instead of `--sources` for a templated source.
- Copy from a guest: `vml rsync-from --sources <source>... [--destination <destination>] <name>`.
- QEMU monitor: `vml monitor --command '<command>' <name>`.
- Images: `vml image <list|available|add|pull|store|remove> [options]`; names and action-specific arguments follow that action's options.

Options generally precede the positional VM name. Preserve shell quoting around guest commands, paths with spaces, and descriptions.

## Run VMs

- Use `vml run <name>` as the create-and-start shortcut.
- Use `vml run --name-same-image <name>` when the VM and image intentionally have the same name.
- Add only requested or necessary options such as `--image`, `--memory`, `--nproc`, networking, display, cloud-init, or SSH behavior.
- Prefer `--exists-ignore` and `--running-ignore` when an idempotent operation is appropriate.
- Use `--snapshot` when the user wants changes discarded after the run.
- Do not guess an image. Inspect `vml image list`, `vml image available`, and the configured `images.default`; ask the user if the choice materially affects the result and cannot be inferred.
- Use `--wait-ssh` when subsequent work requires the guest to be reachable. Use `--ssh` only when an interactive session is desired.

## Manage VMs

- Inspect: `vml list`, `vml show <name>`, or `vml show --format-json <name>`.
- Start an existing VM: `vml start <name>`.
- Stop a VM gracefully: `vml stop <name>`. Use `--force` only when necessary or explicitly requested.
- Run commands inside a VM: `vml ssh <name> --cmd <command>`. Add `--check` when command failure must fail the operation.
- Open an interactive session: `vml ssh <name>`.
- Transfer files with `vml rsync-to` and `vml rsync-from`; specify sources and destination explicitly when ambiguity is possible.
- Use `vml monitor <name> --command <command>` only for QEMU monitor operations.
- Select multiple VMs with `--names`; select hierarchical groups with `--parents`; select machines carrying `vml.toml` tags with `--tags`. Confirm the resulting targets before changing them.

## Images

Use `vml image list` for local images and `vml image available` for images that can be pulled. Write straightforward `add`, `pull`, `store`, and `remove` commands directly when their syntax is known; consult their help only for complex cases, version-sensitive options, or after a usage error. Verify an image exists locally before creating a VM from it.

VML supports one-command creation for ALT, CentOS, Debian, Fedora, openSUSE, and Ubuntu images. Do not assume every advertised image is currently downloadable; check `vml image available`.

When the built-in catalog does not contain the requested system, search the internet for an official cloud image. Prefer the distribution's official image index and a stable direct download URL over mirrors or third-party repacks. Select a QCOW2 image intended for cloud use and supporting cloud-init; filenames commonly contain `cloud`, `genericcloud`, or `cloudimg`, but verify this from the publisher's documentation rather than relying on the name alone. Match the guest architecture to the host and requested VM.

Before adding an internet image:

- Confirm the URL resolves to the image itself, not an HTML download page.
- Record the publisher, release, architecture, image variant, and source URL.
- Prefer HTTPS and verify a publisher-provided checksum or signature when available.
- Avoid daily, testing, or rolling images unless the user requests them; explain when only a mutable URL is available.
- Do not download a large image merely to inspect it when headers, checksums, or publisher metadata are sufficient.

Register an image with `vml image add --name <name> --url <direct-qcow2-url> --description '<description>'`; add `--pull` only when the user asked to download it now. Without `--pull`, verify it appears in `vml image available`; after pulling, verify it appears in `vml image list`. Treat registration and downloading as separate actions when the user's intent is ambiguous. Inspect `vml image add --help` only if this form fails or additional options are needed.

## Safety

- Resolve the target VM explicitly before commands affecting multiple machines. Do not infer `--all-vms`, `--all`, broad tags, or parent selectors from a singular request.
- Treat `remove`, image removal, `clean`, `--exists-replace`, forced stop, and disk replacement as destructive. Explain the affected VM or image and obtain approval when the user's request did not already authorize that exact action.
- Do not expose secrets found in cloud-init data, VM configuration, SSH options, or command output.
- Prefer graceful stop before removal and verify the final state after lifecycle changes.

## Common Examples

```sh
vml list
vml image list
vml run test-vm --image alt-sisyphus --memory 2G --nproc 2 --no-ssh
vml show test-vm
vml ssh test-vm --check --cmd 'uname -a'
vml stop test-vm
```

Adapt examples to the installed CLI and actual image names; never copy placeholder names blindly.
