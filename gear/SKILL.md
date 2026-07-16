---
name: gear
description: Build, rebuild, and troubleshoot ALT Linux RPM packages from Gear Git repositories using Gear and Hasher. Use when Codex needs to inspect a spec and `.gear/rules`, run the project's `gear --commit --hasher -- hsh-rebuild --no-sisyphus-check=packager,gpg` workflow, capture and analyze its build log, locate RPM artifacts, or fix Gear export, BuildRequires, RPM, compilation, file-list, or Sisyphus-check failures.
---

# Build ALT Linux Packages with Gear and Hasher

## Inspect before building

1. Work from the package's Gear Git repository root.
2. Inspect `git status`, the spec file, `.gear/rules`, and relevant sources and patches.
3. Preserve unrelated user changes and follow the repository's existing packaging layout.
4. Verify that `gear`, `hsh`, `hsh-rebuild`, and Git are available. Do not silently install packages or change system configuration. Initialize Hasher only under the conditions below.
5. Check whether a reusable Hasher chroot already exists and identify the branch and architecture it was initialized for. `hsh-rebuild` reuses that environment.

## Create the Hasher chroot when required

Create a Hasher chroot only when at least one of these conditions applies:

- no reusable chroot exists;
- the requested branch differs from the existing chroot's branch;
- the requested architecture differs from the existing chroot's architecture;
- the user explicitly requests a new, recreated, or cleared chroot.

Do not initialize or recreate the chroot for a normal rebuild when the existing branch and architecture already match.

Initialize it with `hsh --initroot-only`. For a specific branch or architecture, select an existing matching `~/apt/apt.conf.<branch>.<arch>` and pass it to this initialization command. For example, to initialize a P11 AArch64 chroot:

```bash
hsh --initroot-only --apt-config="$HOME/apt/apt.conf.p11.aarch64" --target=aarch64
```

If a custom Hasher workdir is in use, pass the same workdir to this command. Verify that the APT configuration exists before initialization and do not edit it. Do not pass `--apt-config` to the subsequent `hsh-rebuild`; the initialized chroot is reused for the build.

## Build and capture the log

Run the user-defined build pipeline from the repository root:

```bash
bash -o pipefail -c 'gear --commit --hasher -- hsh-rebuild --no-sisyphus-check=packager,gpg 2>&1 | tee log'
```

This explicitly runs the pipeline with Bash. `2>&1` redirects stderr (file descriptor 2) to stdout (file descriptor 1) before the combined stream is piped to `tee`.

Understand every part of the command:

- `--commit` builds committed Gear state. Inspect the worktree first and do not assume uncommitted edits are included.
- `--hasher` sends the Gear-produced source package to Hasher.
- `hsh-rebuild` reuses the prepared Hasher environment for an iterative rebuild.
- `--no-sisyphus-check=packager,gpg` disables only the packager and GPG Sisyphus checks. All other enabled Sisyphus checks must run. Do not add any other check to this exclusion list unless the user explicitly requests it. Record that these two checks were disabled and never describe this build alone as complete policy validation.
- `2>&1 | tee log` displays combined stdout and stderr and replaces `log` with the current run's output.

Do not report success solely from `tee`'s status. The command uses Bash's `pipefail` option so a failed `gear`/Hasher process cannot look successful because `tee` exited zero.

Do not append extra flags or substitute `gear-hsh` unless the user asks for a different workflow.

## Diagnose a failed build

Read `log` from the first meaningful error, not merely its final summary. Classify the failure before editing:

- **Gear export:** reconcile `.gear/rules`, tracked Git paths, archive names, and spec `Source` or `Patch` entries.
- **Missing build dependency:** determine the ALT package that supplies the missing tool, header, library, or pkg-config module; update `BuildRequires` only when the build genuinely requires it.
- **Spec/RPM:** inspect the failing stage and ALT RPM macro expansion.
- **Compile/test:** fix the source, patch, flags, or test environment; do not weaken tests without justification.
- **Install/file list:** compare `%buildroot` with `%files` and correct destinations, ownership, duplicates, or unpackaged files.
- **Hasher environment:** distinguish stale retained-state problems from packaging problems; recreate the environment only with user authorization.

Do not hide undeclared dependencies by installing them only into the retained chroot. Make persistent fixes in the spec or packaging files.

## Verify and report

1. Confirm the left side of the pipeline exited successfully.
2. Locate the source RPM in the configured Hasher repository's `SRPMS.hasher` and binary RPMs in its architecture-specific `RPMS.hasher` directory. Respect custom Hasher paths.
3. Report the exact command, branch, architecture, artifact filenames, and `log` location. If the chroot was initialized, also report the initialization command and selected APT configuration.
4. Summarize packaging changes and explicitly state that the packager and GPG Sisyphus checks were disabled.
5. If the current host cannot run ALT Gear/Hasher, provide the command for an ALT Linux host and label the build unverified.

## Safety

- Never build as root.
- Do not run `hasher-useradd`, install host packages, modify APT sources, change global Git identity, publish packages, or submit tasks without explicit authorization.
- Do not commit unrelated changes, rewrite history, or create tags.
- Preserve existing Hasher repositories and retained environments unless the user authorizes cleanup.

Use installed manual pages (`man gear`, `man hsh`, and command help) as version-matched references. Consult current ALT Platform Gear and Hasher documentation at `https://docs.altlinux.org/` when broader guidance is required.
