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

## Current Diagnostic Result

The server-side frame path is confirmed healthy with temporary trace logging:

```text
Wayland surface committed
Rendering frame
Exported compositor frame to video pipeline
Received frame
DMA-BUF import + color conversion + hardware encode + packet send
```

The Fedora 44 Moonlight client negotiates:

```text
video_format=Hevc
```

It then sends repeated IDR requests. Moonshine repeatedly emits HEVC keyframes
but the client stays black while audio and input continue to work. This is now
a client decode/HEVC acceptance problem, not a Moonshine compositor, DMA-BUF,
encoder, networking, or pairing problem.

## Next Test: Client H.264 Fallback

In Moonlight on the Fedora laptop, disable HEVC/H.265 and AV1, disable HDR,
Diagnostic` and confirm the server log reports:

```text
Client negotiated video format video_format=H264
```

If H.264 displays correctly, inspect/update the laptop's Moonlight Qt,
FFmpeg/VA-API, Mesa, and hardware-decoding stack for HEVC support. If H.264
also remains black while producing the same server-side frame trace, inspect
the Moonlight client logs and GameStream packet handling next.

## Temporary Diagnostics to Remove Later

The current candidate includes compositor frame-handoff trace logs and RTSP
codec logging. The service may have a temporary
`trace-compositor.conf` drop-in. Remove the trace logging and the temporary
X11/Wayland diagnostic applications once a stable codec path is confirmed.

## Rollback

The prior manual v0.12.0 install was backed up under:

```text
/usr/local/lib/moonshine-backup/v0.12.0-nobara/
```

The active candidate branch is `nobara-v0.14.5` in:

```text
https://github.com/Balakirev1837/moonshine/tree/nobara-v0.14.5
```
