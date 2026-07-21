# Moonshine on Nobara Linux

Notes on building/installing [Moonshine](https://github.com/hgaiser/moonshine)
on [Nobara Linux](https://nobaraproject.org/) (Fedora-based, KDE Plasma Desktop
Edition, currently 43 / f43 base).

Moonshine is tested upstream on Arch Linux. Nobara is a Fedora-based distro with
its own curated package set, which makes the install path non-obvious. This
document captures the differences and the workarounds required.

Tested hardware: AMD Radeon RX 6800 (Navi 21 / RDNA2). The `RADV_PERFTEST`
workaround below is AMD-specific; NVIDIA users can skip it.

## Build dependencies

Nobara uses `dnf` (RPM), not `pacman`. The Arch package names in the upstream
README map cleanly to Fedora packages:

```bash
sudo dnf install -y --nogpgcheck \
  clang cmake make pkg-config rust cargo libcxx-devel python3 git \
  libevdev-devel libxkbcommon-devel wayland-devel opus-devel \
  vulkan-headers vulkan-loader-devel mesa-vulkan-drivers \
  'libcxx-devel-21.1.8-4.fc43'
```

### Workaround 1: GPG check failures on `[nobara]` rolling repo

Some packages in the `[nobara]` rolling repo are built for fc44 and signed with
a rolling key that is not distributed to existing Nobara 43 installs. The error
looks like:

```
OpenPGP check for package "libcxx-devel-22.1.8-1.fc44.x86_64" ... has failed:
Import of the key didn't help, wrong key?
```

Fix: use `--nogpgcheck` for the install transaction. Safe because these are
Nobara's own repos that you already trust for system packages.

For `libcxx-devel` specifically, pin to the fc43 version
(`'libcxx-devel-21.1.8-4.fc43'`) — it matches the system, is signed with the
trusted Nobara-43 key, and the LLVM 21 vs 22 difference is irrelevant for the
vendored shaderc build.

### Workaround 2: `mesa-libgbm-devel` cannot be installed

The build fails at link time with:

```
rust-lld: error: unable to find library -lgbm
```

The `mesa-libgbm-devel` RPM cannot be installed on Nobara because
`mesa-libgallium-freeworld` (which Nobara uses for patented H.264/H.265
hardware acceleration) obsoletes `mesa-libgallium`, and the dependency chain
for the newer fc44 mesa-libgbm-devel drags in a newer `mesa-libgallium` that
dnf cannot satisfy.

**Do NOT use `--allowerasing`** — that would uninstall
`mesa-libgallium-freeworld` and break hardware video encode/decode on a
gaming/streaming box.

Fix: create the dev symlink manually. The runtime `libgbm.so.1` is already
installed; the linker just needs the unversioned `.so` symlink:

```bash
sudo ln -s libgbm.so.1 /usr/lib64/libgbm.so
```

This is the only file from `mesa-libgbm-devel` that the build actually needs.

## Runtime workarounds

### Workaround 3: `RADV_PERFTEST=video_encode` is required

Moonshine's video encoder fails with:

```
Failed to create video encoder: Failed to create video context:
No suitable Vulkan physical device found: No device with required video
support found. Ensure your GPU drivers support Vulkan Video extensions
(VK_KHR_video_queue, VK_KHR_video_encode_queue, etc.).
```

RADV (the Mesa AMD Vulkan driver) **gates Vulkan Video encode behind an env
var**: `RADV_PERFTEST=video_encode`. Without it, the device reports 228
extensions but no `VK_KHR_video_*`. With it, 236 extensions including
`VK_KHR_video_encode_h264`, `VK_KHR_video_encode_h265`, etc.

Fix: a systemd drop-in (survives updates, doesn't modify the upstream service
file):

```bash
sudo mkdir -p /etc/systemd/system/moonshine@.service.d
sudo tee /etc/systemd/system/moonshine@.service.d/radv-video-encode.conf <<'EOF'
[Service]
Environment=RADV_PERFTEST=video_encode
EOF
sudo systemctl daemon-reload
sudo systemctl restart moonshine@$USER
```

### Workaround 4: Steam URI forwarding defeats compositor isolation

When Moonshine launches `/usr/bin/steam steam://rungameid/<id>`, the Steam
wrapper script detects any existing Steam instance (e.g. on your main desktop),
forwards the URI to it, and exits within ~33ms. Moonshine sees the launcher
exit and tears down the session before the video stream can start.

Symptom in logs:

```
Application exited shortly after launch. state="inactive"
Failed to launch session, waiting for new session.
```

Fix: in `~/.config/moonshine/config.toml`, add a `pre_command` to both the
`[[application]]` and `[[application_scanner]]` entries to kill any existing
Steam before launching the fresh instance inside Moonshine's compositor:

```toml
pre_command = [
    ["/usr/bin/sh", "-c", "pkill -TERM -f '/usr/bin/st[e]am' || true"],
    ["/usr/bin/sleep", "2"],
]
launch_timeout_secs = 15
```

Notes:
- The character class `[e]` in `st[e]am` prevents `pkill -f` from matching its
  own command line (a literal `[e]` in the regex won't match the literal
  string `[e]` in pkill's process name).
- `|| true` is required because `pkill` returns exit code 1 when no processes
  match, and systemd's `ExecStartPre` treats non-zero as failure.
- `launch_timeout_secs` is bumped from the default `2` to `15` — Steam takes
  5-10s to bootstrap, and 2s causes false failures.

## Update helper

`scripts/moonshine-update` automates pulling the latest upstream release tag,
rebuilding as the source-dir owner (so the cargo cache stays user-owned), and
reinstalling the artifacts. It also recreates the workarounds above if they've
been removed.

Install with:

```bash
sudo install -Dm755 scripts/moonshine-update /usr/local/bin/moonshine-update
```

Usage:

```bash
sudo moonshine-update --check     # dry-run, show what would update
sudo moonshine-update             # apply latest release tag
sudo moonshine-update --main      # track main HEAD instead
sudo moonshine-update --user USER # restart moonshine@USER.service
```

Logs to `/var/log/moonshine-update.log`.

## Repository layout

The fork at <https://github.com/Balakirev1837/moonshine> tracks upstream on the
`main` branch (synced via `git fetch upstream`) and carries Nobara-specific
additions on the `nobara` branch:

- `scripts/moonshine-update` — the update helper
- `NOBARA.md` — this file

No source patches are required for v0.12.0 on Nobara; all the workarounds are
config/drop-in level.

Git remotes (after `scripts/moonshine-update` setup):
- `upstream` → `github.com/hgaiser/moonshine` (source of release tags)
- `origin` → `github.com/Balakirev1837/moonshine` (our fork, for Nobara changes)
