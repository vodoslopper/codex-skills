---
name: vml
description: Run and manage virtual machines with the local `vml` CLI. Use when Codex needs to create or run a VM, inspect VM state, start or stop VMs, execute commands over SSH, transfer files, discover or add cloud images, manage VM images, use graphical or headless-automated displays, access the QEMU monitor, or remove and clean up VMs.
---

# VML

Use the installed `vml` command to perform VM work directly. VML manages QEMU/KVM machines as directories containing a required `vml.toml` marker/config and a `disk.qcow2` disk. Inspect live state instead of assuming that a VM or image exists.

Read [references/project.md](references/project.md) when configuring VML, selecting VM groups, using cloud-init or networking, troubleshooting dependencies, or extending the CLI.

## Workflow

1. Verify the CLI with `command -v vml` and, when compatibility matters, `vml --version`.
2. Inspect relevant state with `vml list`, `vml show`, or `vml image list`.
3. Write ordinary commands directly from the syntax below. Do not routinely probe commands with `--help` first.
4. Select the smallest command that satisfies the request.
5. Run it and verify the result with the command's exit status and output, using `vml show <name>` or `vml list` only when the command does not already confirm the required state. In particular, start a VM with explicit `--wait-ssh` and do not check it again after that command returns successfully.

Use `vml <subcommand> --help` only when the invocation is complex, its syntax is not covered here, an installed-version difference matters, or a command fails with a usage/option error. For nested image commands, use `vml image <action> --help`. After an error, inspect the narrowest relevant help and retry with the corrected command.

## Command Syntax

VML uses `vml [global-options] <subcommand> [subcommand-options] [name]`. Put global options such as `--host`, `--all-vms`, `--vm-config`, and `--minimal-vm-config` before the subcommand. Put subcommand options after it.

- Single VM: `vml <start|stop|show|remove|ssh|monitor> [options] -n <name>`; `--names <name>` is equivalent.
- Multiple VMs: use the subcommand's `-n/--names <name>...`, `--parents <parent>...`, or `--tags <tag>...` selector; use global `--all-vms` only when explicitly intended.
- Create and run: `vml run [options] -n <name>`; create without starting: `vml create [options] -n <name>`. Omit `--image` unless a specific image is required; VML uses the configured default image.
- Guest command: `vml ssh [--check] --cmd '<command>' -n <name>`. Without `--cmd`, SSH is interactive.
- Copy to a guest: `vml rsync-to --sources <source>... [--destination <destination>] -n <name>`; use `--template <template>` instead of `--sources` for a templated source.
- Copy from a guest: `vml rsync-from --sources <source>... [--destination <destination>] -n <name>`.
- QEMU monitor: `vml monitor --command '<command>' -n <name>`.
- Images: `vml image <list|available|add|pull|store|remove> [options]`; names and action-specific arguments follow that action's options.

Options generally precede the positional VM name. Preserve shell quoting around guest commands, paths with spaces, and descriptions.

## Run VMs

- Use `vml run <name>` as the create-and-start shortcut; when no particular image is required, omit `--image` so VML uses the default from its configuration.
- Use `vml run --name-same-image <name>` when the VM and image intentionally have the same name.
- Add only requested or necessary options such as `--image`, `--memory`, `--nproc`, networking, display, cloud-init, or SSH behavior.
- Prefer `--exists-ignore` and `--running-ignore` when an idempotent operation is appropriate.
- Use `--snapshot` when the user wants changes discarded after the run.
- When a specific image is required, inspect `vml image list`, `vml image available`, and the configured `images.default`; ask the user if the choice materially affects the result and cannot be inferred.
- VM startup can take time. Although `vml start` waits for SSH by default, pass `--wait-ssh` explicitly so configuration cannot override it. Let the command finish and, after a successful return, proceed without a separate readiness or state check.
- Use `--wait-ssh` for commands where it is not already the default and subsequent work requires the guest to be reachable. Use `--ssh` only when an interactive session is desired.

## GTK Displays

Treat the QEMU display backend as independent of the image catalog properties. Any bootable image can be created, run, or started with `--display-gtk`; an image does not need the `gui` property. The flag provides a GTK framebuffer window but does not install a guest desktop, display manager, or login user. Desktop images can show a full session, ordinary cloud images commonly show a graphical console, and serial-only guests may leave the framebuffer blank.

- Persist GTK in a new VM with `vml create --image <image> --display-gtk -n <name>`, then use `vml start --wait-ssh -n <name>`.
- Select GTK for one launch with `vml start --display-gtk --wait-ssh -n <name>`.
- Use `vml run --image <image> --display-gtk --wait-ssh -n <name>` to create and start in one command.
- Do not add `--properties gui` merely to obtain a GTK window. Use that image-specific property only when its separate guest configuration behavior is intended.
- For a visible window, run VML from a graphical host session with a usable `DISPLAY`. Confirm QEMU lists `gtk` in `qemu-system-x86_64 -display help`; install the distribution's QEMU GTK UI package if it does not.
- Keep the display server alive for the entire QEMU process lifetime. Losing the X server terminates a VM using the GTK backend.

## Headless GTK Automation

For automation that does not inspect or manipulate a GUI, prefer `vml start --display-none --wait-ssh -n <name>` and operate through `vml ssh` or the QEMU monitor; it needs no X server. Use Xvfb with GTK only when automation must inspect or manipulate the guest framebuffer. Require Xvfb plus `xdotool`; use ImageMagick's `import` or another capture tool when screenshots are needed. Choose a distinct unused X display for concurrent jobs.

```sh
vm_name=new
Xvfb :99 -screen 0 1600x1000x24 -nolisten tcp &
xvfb_pid=$!
export DISPLAY=:99

cleanup() {
    vml stop -n "$vm_name" || :
    kill "$xvfb_pid" 2>/dev/null || :
}
trap cleanup EXIT
trap 'exit 129' HUP
trap 'exit 130' INT
trap 'exit 143' TERM

vml start --display-gtk --wait-ssh -n "$vm_name"

window_id=$(xdotool search --onlyvisible --name '^QEMU$' | tail -n 1)
xdotool getwindowgeometry "$window_id"
xdotool mousemove --window "$window_id" 640 400
xdotool click --window "$window_id" 1
xdotool key --window "$window_id" Escape
```

Target the visible uppercase `QEMU` window and confirm its geometry; QEMU may also create a lowercase `qemu` helper window as small as 10 by 10 pixels. On bare Xvfb without a window manager, activation or focus requests can fail even though direct `xdotool --window` input works. Capture before-and-after screenshots and verify an observable screen or guest-state change rather than treating successful input commands as proof of interaction.

Do not wrap a daemonized GTK VM in a short-lived `xvfb-run` command: when the wrapper returns and removes Xvfb, QEMU loses its display and exits. Keep the controlling session alive until automation completes, stop the VM first, and terminate Xvfb afterward. Put temporary logs and screenshots under `$TMP/codex` when the `tmp` skill applies; otherwise use a secure task-specific temporary directory.

## Manage VMs

- Inspect: `vml list`, `vml show <name>`, or `vml show --format-json <name>`.
- Start an existing VM: `vml start --wait-ssh <name>`. Pass `--wait-ssh` explicitly even though it is the default because configuration can override the default. A successful return confirms that the guest is reachable; do not follow it with `vml show`, `vml list`, or another SSH readiness probe.
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
vml run -n test-vm --memory 2G --nproc 2 --no-ssh
vml show -n test-vm
vml ssh -n test-vm --check --cmd 'uname -a'
vml stop -n test-vm
```

Adapt examples to the installed CLI and actual image names; never copy placeholder names blindly.
