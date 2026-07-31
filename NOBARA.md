# Moonshine on Nobara 44

This branch is a reproducible deployment branch for one Nobara 44 host, not
an upstream contribution proposal. It tracks Moonshine v0.14.5 with only the
local compatibility changes that were required and demonstrated on this host.

## Environment

| Component | Value |
| --- | --- |
| Host OS | Nobara 44 / Fedora-based |
| Kernel | `7.1.4-200.nobara.fc44.x86_64` |
| GPU | AMD Radeon RX 6800 (Navi 21 / RDNA2) |
| Mesa/RADV | 26.1.5, Vulkan 1.4.354 |
| Server | Moonshine v0.14.5 plus the patches in this branch |
| Client under test | Fedora 44 Moonlight laptop |

## Verified Host Health

Moonshine v0.14.5 `healthcheck` passes render-node access, EGL/GLES, Vulkan,
H.264/HEVC codecs, DMA-BUF external memory and modifiers, WSI layer discovery,
Xwayland, runtime directory, input devices, sleep inhibition, and `kcmp(2)`.
The expected port conflict appears only when running healthcheck while the
Moonshine service owns GameStream TCP ports.

## Required Host Configuration

### Build prerequisites

The current upstream source uses Rust features unavailable in the original
Nobara 44 Rust 1.92 toolchain. A user-local Rustup stable 1.97.1 toolchain was
installed under `~/.rustup` to build v0.14.5; no system Rust package was
replaced.

### RADV Vulkan Video encode

RADV gates Vulkan Video encode behind an environment variable. Use the
current spelling in a persistent systemd drop-in:

```ini
# /etc/systemd/system/moonshine@.service.d/radv-video-encode.conf
[Service]
Environment=RADV_EXPERIMENTAL=video_encode
```

`RADV_PERFTEST=video_encode` still works but is deprecated.

### Steam launch isolation

Steam must not be running on the desktop when Moonshine starts it. Steam URI
forwarding otherwise sends the launch request to the desktop instance, causes
the Moonshine transient unit to exit, and produces a 503 on Moonlight.

Both Steam entries in `~/.config/moonshine/config.toml` use:

```toml
pre_command = [
    ["/usr/bin/sh", "-c", "pkill -TERM -f 'st[e]am' || true"],
    ["/usr/bin/sh", "-c", "rm -f \"$HOME/.steam/steam.pid\""],
    ["/usr/bin/sleep", "2"],
]
launch_timeout_secs = 15
```

The broad `st[e]am` regex matches Steam autostart processes below
`~/.local/share/Steam/`, unlike a `/usr/bin/steam`-only match. Removing
`~/.steam/steam.pid` avoids a stale desktop PID being passed to the fresh
Steam instance during systemd scope cleanup.

### Steam 32-bit Vulkan ICD selection

Nobara's user systemd environment sets `VK_ICD_FILENAMES` to only the 64-bit
RADV manifest. Steam's primary client binary is 32-bit, so it returned
`VK_ERROR_INCOMPATIBLE_DRIVER (-9)` during GPU topology detection.

Moonshine's Steam commands use `/usr/bin/env` to pass both manifests:

```toml
"VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.i686.json:/usr/share/vulkan/icd.d/radeon_icd.x86_64.json"
```

This is scoped to Moonshine-launched Steam and its children.

## Local WSI Compatibility Patches

### Pressure-vessel loader chain

Steam pressure-vessel supplies newer Vulkan loader function values in the
`VkInstanceCreateInfo` `pNext` chain. Moonshine's original two-value Rust enum
treated an unknown value as a link-info entry in optimized builds and
dereferenced a null `p_layer_info`, crashing `vulkandriverquery` and
steamwebhelper.

`moonshine-wsi/src/dispatch.rs` now represents the field as raw `u32` and
follows only `VK_LAYER_LINK_INFO` (`0`), skipping all unknown/future values.

### Xlib surface extension

The WSI layer already routes `vkCreateXlibSurfaceKHR` through the XCB
Xwayland-bypass path. It now injects `VK_KHR_xlib_surface` as well as the
existing surface/Wayland/XCB extensions.

## Verified Result

The server-side frame path is confirmed healthy with temporary trace logging:

```text
Wayland surface committed
Rendering frame
Exported compositor frame to video pipeline
Received frame
DMA-BUF import + color conversion + hardware encode + packet send
```

The Fedora 44 Moonlight client initially negotiated:

```text
video_format=Hevc
```

It then sent repeated IDR requests. Moonshine repeatedly emitted HEVC keyframes
but the client stayed black while audio and input continued to work. This
isolated the fault to the laptop's HEVC decode/acceptance path, not Moonshine
compositing, DMA-BUF export/import, encoding, networking, pairing, or input.

Disabling HEVC and AV1 on the Fedora laptop forced:

```text
Client negotiated video format video_format=H264
```

H.264 displayed successfully. The server encoder's observed frame latency was
roughly 3-5 ms, so light remaining stutter during this validation is most
likely wireless-network jitter rather than host-side rendering or encoding.

## Client Codec Guidance

In Moonlight on the Fedora laptop, disable HEVC/H.265 and AV1, disable HDR,
Diagnostic` and confirm the server log reports:

```text
Client negotiated video format video_format=H264
```

For this Fedora 44 laptop, retain the H.264-only setting until its Moonlight
Qt, FFmpeg/VA-API, Mesa, and hardware-decoding stack can be investigated.
If HEVC is needed later, gather these client details first:

```bash
rpm -q moonlight-qt mesa-va-drivers mesa-va-drivers-freeworld libva libva-utils
vainfo
lspci -nn | grep -iE 'vga|3d|display'
flatpak list | grep -i moonlight
```

## Bazzite Client Pairing

Bazzite is Fedora Atomic-derived and commonly carries a more complete
gaming/media stack than plain Fedora. Pair it normally through Moonlight, but
use a controlled codec sequence:

1. Start with H.264, HDR disabled, 1080p60, and a moderate 10-20 Mbps bitrate.
2. Confirm the host log says `video_format=H264` and that video appears.
3. Enable HEVC only after the H.264 baseline works.
4. Enable HDR last, after HEVC is known-good.

The Bazzite client should be tested independently: the Fedora laptop's HEVC
failure does not prove Bazzite will fail, but H.264 is the stable baseline.

## Temporary Diagnostics to Remove Later

The current candidate includes compositor frame-handoff trace logs and RTSP
codec logging. The `trace-compositor.conf` systemd drop-in should be removed
after this investigation so production sessions do not emit frame-level logs.
The temporary `X11 Diagnostic` and `Wayland Diagnostic` application entries
can also be removed from `~/.config/moonshine/config.toml`; retain
`hdr = false` while using SDR/H.264 clients.

## Update Safety

Do not run the older `/usr/local/bin/moonshine-update` helper. It targets the
old `/opt/moonshine-src` v0.12 worktree and can overwrite the active v0.14.5
candidate with an obsolete build. Future upgrades should start from this
branch/worktree, deliberately rebase the local WSI patches onto a selected
upstream release, build with Rustup stable, run `moonshine healthcheck`, then
install the resulting binary and WSI layer together.

## Rollback

The prior manual v0.12.0 install was backed up under:

```text
/usr/local/lib/moonshine-backup/v0.12.0-nobara/
```

The active candidate branch is `nobara-v0.14.5` in:

```text
https://github.com/Balakirev1837/moonshine/tree/nobara-v0.14.5
```
