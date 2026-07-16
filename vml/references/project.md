# VML Project Reference

Use the upstream repository at <https://github.com/Obirvalger/vml> as the authoritative project reference. Use the command syntax in `SKILL.md` directly for routine work. Consult the installed command's narrowest relevant `--help` only for complex or uncovered syntax, when installed and upstream versions may differ, or after a usage error.

## Storage and configuration

- Read the main configuration from `~/.config/vml/config.toml`.
- Read `vms-dir` to locate VM directories; do not assume `~/vml`, which is only the upstream example default.
- Treat `vml.toml` as the marker and per-VM configuration. Expect `disk.qcow2` beside it.
- Allow per-VM fields to override or complement fields from the main `[default]` section.
- When `config-hierarchy = true`, include `vml-common.toml` files in parent directories. Account for inherited settings before modifying a VM.
- Read `[commands]` for configured defaults such as create/pull behavior, existing-VM handling, list folding, removal interactivity, SSH waiting, and already-running behavior.
- Read `[images]` for the default image, image directories, and update policy.

## VM selectors

- Use `--names` (`-n`) for complete VM names. Omit it only for a single positional name.
- Use `--parents` (`-p`) to select all VMs below a path-like parent. Names such as `group/first` express hierarchy.
- Use `--tags` (`-t`) to select tags declared in VM configuration.
- Use list folding only for display; do not confuse a folded parent with a single VM.

## Cloud-init

- Expect VML to use cloud-init's NoCloud data source through an additional metadata drive.
- Use cloud-init-capable QCOW2 cloud images as VM base images. A filename containing `cloud`, `genericcloud`, or `cloudimg` is a useful discovery signal, not proof of compatibility; verify the variant in the publisher's documentation.
- Allow VML to generate metadata when cloud-init is enabled, or pass a prepared seed image through `cloud-init-image` / `--cloud-init-image`.
- Create a custom seed image with `cloud-localds <seed.img> <user-data.yaml>` when requested.
- Never retain placeholder public keys in user-data. Verify the requested user's SSH key and avoid printing private keys.

## Networking

- Prefer user networking for ordinary outbound connectivity; configure `net.type = "user"`.
- Use `net.type = "none"` for explicit isolation.
- Treat tap networking as host-level network administration. It requires a pre-created tap device and explicit address, gateway, and optionally nameservers. Do not create or reconfigure host networking without authorization.
- Remember that tap networking can support guest-to-guest and ICMP behavior that QEMU user networking may not.

## SSH integration and transfers

- VML can generate OpenSSH configuration under paths configured in `[openssh-config]`. If the user's SSH config includes VML's generated `main-config`, ordinary `ssh`, `scp`, and `rsync` may address VM names directly.
- Otherwise use `vml ssh`, `vml rsync-to`, `vml rsync-from`, or the guidance from `vml scp`.
- Respect configured SSH user, key, host, port, and options rather than assuming root or a fixed forwarded port.

## Runtime requirements

When diagnosing setup failures, check QEMU/KVM, `rsync`, `socat`, and `cloud-localds` from cloud-utils. Check that the current user has permission to use KVM, commonly through the system's VM user group.

## Extensions

VML discovers external subcommands as `vml-*` executables on `PATH`. For example, an executable named `vml-xrun` becomes `vml xrun`. Inspect an extension before invoking it; it is not necessarily part of upstream VML and may have independent side effects.
