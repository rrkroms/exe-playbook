# 08 — Runtimes, Containers and Wine Prefixes

A Windows program does not just need a CPU translator and a GPU driver. It needs a place to live: a filesystem that looks vaguely like Windows, a registry, installed runtimes, an audio server, input devices and a display. This module defines those layers and shows how they nest.

The words here are often mixed together. A **RootFS** is a Linux userspace — libraries, binaries and directories — without necessarily being a virtual machine. A **container** uses host-kernel features (or the `ptrace`-based imitation used by [proot](https://github.com/PRoot/proot)) to give that userspace an isolated view of the system. A **[Wine](https://gitlab.winehq.org/wine/wine) prefix** is a directory plus registry files that impersonates one Windows installation. These are three different things.

## 8.1 Three nested environments

```text
Host: Android / Linux / macOS
  |
  +-- Linux userspace (RootFS: libc, ld.so, graphics + audio libraries)
  |     |
  |     +-- Container (chroot / proot / LXC / Flatpak / Steam Runtime)
  |     |     |
  |     |     +-- Wine prefix (WINEPREFIX)
  |     |           |
  |     |           +-- C:\ -> drive_c, dosdevices, registry, DLLs, saves
  |     |
  |     +-- Translators and drivers (Box64/FEX, DXVK/VKD3D, Vulkan ICD)
  |
  +-- Host services (SurfaceFlinger/Wayland/X11, audio server, input devices)
```

This is a conceptual map, not a literal process tree. On native x86 Linux the container may be absent and the RootFS is simply the host distribution. On Android, proot is not a real kernel container: it intercepts syscalls with `ptrace`. A prefix provides *no* process isolation of its own; it is data consumed by Wine inside the container. The host compositor — [SurfaceFlinger](https://source.android.com/docs/core/graphics) on Android, [Wayland](https://wayland.freedesktop.org/) or [X11](https://www.x.org/) on Linux — and the audio server are separate processes outside all three layers. Translators such as [Box64](https://github.com/ptitSeb/box64) and [FEX](https://github.com/FEX-Emu/FEX) live in the userspace layer, not in the prefix.

## 8.2 Containers and Roots

| Mechanism | What it isolates | Where | Key limitation |
|---|---|---|---|
| [chroot](https://man7.org/linux/man-pages/man8/chroot.8.html) | Filesystem root only | Linux, needs root | No namespaces, no resource control |
| [proot](https://github.com/PRoot/proot) | Filesystem view via `ptrace` syscall interception | Android Termux, rootless Linux | Slower filesystem-heavy work; no real root, cgroups or namespaces; fixed `/proc` values |
| [proot-distro](https://github.com/termux/proot-distro) | proot plus a packaged distro RootFS | Termux | Inherits proot limits; images are Docker/OCI-based |
| [LXC](https://linuxcontainers.org/lxc/introduction/) | Real namespaces + cgroups | Linux | Needs privileges or careful unprivileged setup |
| [Flatpak](https://flatpak.org/) | Desktop app sandbox + portals | Linux desktop | Confinement can hide drivers, devices and files without overrides |
| [Steam Runtime](https://github.com/ValveSoftware/steam-runtime) | Curated userspace libraries in a bubblewrap container | Steam for Linux | Only applies to Steam-launched processes |
| [Distrobox](https://github.com/89luca89/distrobox) | Podman/Docker container integrated with the host | Linux desktop | Container runtime required |

A typical [Termux](https://termux.dev/) RootFS setup (F-Droid build; do not mix sources — builds share a `sharedUserId`):

```sh
pkg update && pkg upgrade
pkg install proot-distro termux-api android-tools
pkg install x11-repo
pkg i termux-x11-nightly
pkg install vulkan-tools mesa-vulkan-icd-freedreno
termux-setup-storage
termux-wake-lock

proot-distro install debian:bookworm --architecture aarch64
proot-distro login debian --shared-tmp --shared-x11
export PULSE_SERVER=127.0.0.1 container=proot-distro DISPLAY=:1
```

`--shared-tmp` and `--shared-x11` let the guest see the host's X11 socket and temporary files. Cross-architecture guests can be run through [QEMU](https://www.qemu.org/) user mode, but that adds another translation layer on top of Box64/FEX. [FEX](https://github.com/FEX-Emu/FEX) has its own RootFS requirement on AArch64: it fetches one with `FEXRootFSFetcher` into `$HOME/.fex-emu/RootFS/` and reads `$HOME/.fex-emu/Config.json`.

Two rules save time: keep the RootFS under `$HOME` (Android's scoped storage makes other locations awkward), and do not expect `ptrace`-based proot to behave like a VM — some tools detect it and some syscalls cannot be faked.

## 8.3 The Wine prefix

A prefix is created on first run or with `wineboot`. Its layout:

```text
$WINEPREFIX/
├── drive_c/
│   ├── Program Files/
│   ├── Program Files (x86)/
│   ├── users/<user>/AppData/...   # per-user config and shader caches
│   └── windows/
│       ├── system32/              # 64-bit DLLs
│       └── syswow64/              # 32-bit DLLs
├── dosdevices/
│   ├── c: -> ../drive_c
│   └── z: -> /                    # host root, mounted as a drive
├── system.reg                     # HKLM registry hive
├── user.reg                       # HKCU registry hive
└── userdef.reg
```

Notes:

- `dosdevices` is where drive letters are defined. `c:` is the prefix's `drive_c`; `z:` normally maps to the host root so the game can read files outside the prefix.
- `system32` holds 64-bit DLLs and `syswow64` holds 32-bit DLLs — the opposite of what the names suggest from a 32-bit-era Windows perspective. Getting this backwards is a common cause of “DLL loaded but wrong architecture” errors.
- The `.reg` files are plain text registry hives. A prefix can be copied as a template, but copies made while Wine is running can be corrupt; always close the prefix first.

**Isolation.** Separate prefixes per game are the simplest way to prevent one game's DLL overrides, registry tweaks or installed runtimes from breaking another. They cost disk space (a few hundred MB each, more with runtimes) and duplicate shader caches. A shared prefix is fine for a single game family with identical requirements.

## 8.4 The environment variables that matter

| Variable | Effect |
|---|---|
| `WINEPREFIX` | Absolute path to the prefix. If unset, Wine uses `~/.wine`. |
| `WINEARCH` | Set to `win32` only for a pure 32-bit prefix; modern Wine creates 64-bit prefixes by default and supports 32-bit apps through WoW64. Forcing `win32` makes a prefix that cannot run 64-bit games. |
| `WINEDLLOVERRIDES` | Comma-separated DLL override list, e.g. `d3d9=n,dxgi=n`. |
| `WINEDEBUG` | Log channels, e.g. `err,+loaddll`; `-all` silences logging. |
| `DISPLAY` | X11 display number on Linux/Termux, e.g. `:1` for [Termux:X11](https://github.com/termux/termux-x11). |
| `PULSE_SERVER` | Audio server address; `127.0.0.1` is typical inside proot. |
| `VK_DRIVER_FILES`, `DXVK_HUD` | Driver and translator settings from module 07. |

**Windows version.** Wine reports a Windows version to applications; you change it with Wine's `winecfg` (Applications tab) or the registry. Some older games behave differently under Windows XP versus Windows 7/10 mode — installers may check the version and refuse to run, or a middleware DLL may select a different code path. Change it only to solve a documented problem; “newer” is not always better for a 2003 game. [Proton](https://github.com/ValveSoftware/Proton) does not provide a first-class per-title Windows-version switch in the Steam client; change the prefix's version with `winecfg`, the registry, or a prefix tool such as protontricks.

## 8.5 DLL overrides

A Windows DLL can come from two places:

1. **Built-in** (`b`): Wine's implementation of that Windows API.
2. **Native** (`n`): an actual Windows DLL installed into the prefix, usually from a redistributable or game installer.

`WINEDLLOVERRIDES` decides which wins, per DLL. Examples:

```sh
# Let DXVK supply D3D9 and DXGI instead of Wine's built-ins
export WINEDLLOVERRIDES="d3d9=n,dxgi=n"

# Use Wine built-ins for everything, disabling a broken native DLL
export WINEDLLOVERRIDES="dinput8=b"
```

The same choice can be made persistently in `winecfg` (Libraries tab). For [DXVK](https://github.com/doitsujin/dxvk), the usual install step is to copy the x64 DLLs into `system32` and the x32 DLLs into `syswow64`, then override `d3d8`, `d3d9`, `d3d10core`, `d3d11` and `dxgi` as needed. [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) supplies the D3D12 path. Legacy D3D7/8 games may instead use [D7VK](https://github.com/WinterSnowfall/d7vk), [D8VK](https://github.com/AlpyneDreams/d8vk), D3D8to9, or WineD3D — the override list must match the translator you actually installed.

A wrong override is one of the most common self-inflicted failures: forcing a native DLL that is missing, 16-bit, or from the wrong Windows version will break a game that worked with the built-in. Add overrides one at a time and read `WINEDEBUG=+loaddll` output.

## 8.6 Windows runtime dependencies

Windows games assume runtimes are present system-wide. Wine does not ship every one. Install them into the prefix as needed.

| Dependency | What it provides | Symptom when missing |
|---|---|---|
| [Visual C++ Redistributable](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist) | `msvcr*.dll`, `msvcp*.dll` C/C++ runtime | “The program can't start because MSVCP140.dll is missing” |
| [.NET Framework](https://dotnet.microsoft.com/) / [Mono](https://www.mono-project.com/) (Wine Mono: https://gitlab.winehq.org/wine/wine-mono) | Managed-code runtime for C# games and launchers | .NET install dialog, launcher fails |
| [Gecko](https://gitlab.winehq.org/wine/wine-gecko) | HTML rendering in Wine | Installer windows render blank |
| DirectX 9.0c runtime / `d3dx9_*.dll` ([SDK archive](https://learn.microsoft.com/en-us/windows/win32/direct3d9/directx-sdk-archive-content)) | Legacy D3D helpers, effects, math | Missing `d3dx9_43.dll` and similar errors |
| [XAudio2](https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-introduction) / [DirectSound](https://learn.microsoft.com/en-us/windows/win32/directshow/directsound) / [WASAPI](https://learn.microsoft.com/en-us/windows/win32/coreaudio/wasapi) | Game audio APIs | No sound, or audio initialization crash |
| [Media Foundation](https://learn.microsoft.com/en-us/windows/win32/medfound/media-foundation) | Video playback | Cutscenes fail; Wine's implementation is incomplete |
| [PhysX](https://developer.nvidia.com/physx-sdk) | Physics middleware | PhysX games crash on load |

[winetricks](https://github.com/Winetricks/winetricks) automates most installs: `winetricks vcrun2022 dotnet48 d3dx9 xaudio2_7`. On Debian it is packaged as `winetricks` (contrib). Run winetricks with the same `WINEPREFIX` as the game, and keep the downloads it fetches — reinstalling into a fresh prefix is a normal rollback step.

Do not install everything “just in case”. Each install changes the registry and can overwrite DLLs. .NET/WPF is a known weak spot in Proton; Media Foundation is incomplete, with VP9/AV1 preferred, and Proton offers workarounds such as `nomfdxgiman`. Install what the error message or log asks for.

## 8.7 Audio

Wine implements Windows audio APIs and forwards them to a host audio server.

| Windows side | Host side |
|---|---|
| WASAPI (modern low-level), DirectSound (legacy), XAudio2 (games), Media Foundation (video) | [PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/), [PipeWire](https://pipewire.org/) (often via its PulseAudio compatibility layer), [ALSA](https://www.alsa-project.org/), or Android audio APIs |

Checklist:

- Set `PULSE_SERVER=127.0.0.1` inside proot so Wine connects to Termux's PulseAudio. PipeWire-pulse is a common alternative.
- Confirm the audio server is actually running before blaming Wine. On Android, battery optimisation and the phantom process killer (§8.10) can kill the server.
- Silence, crackling and crashes at audio init often trace to a missing `xaudio2_*` DLL, wrong buffer size, or a DLL override forcing the wrong implementation.
- Wine exposes several audio drivers; if one fails, try another. Record which one worked in your prefix template.

## 8.8 Input

| Windows API | Host path | Notes |
|---|---|---|
| XInput | `xinput1_3.dll` / `xinput9_1_0.dll` built-ins → evdev/SDL | Xbox-style controllers; most modern games |
| DirectInput | `dinput8.dll` → evdev/SDL | Older games and peripherals; force-feedback support is weaker |
| Raw input / Win32 messages | X11/Wayland/Android input events | Mouse and keyboard, pointer lock |
| Touch / on-screen controls | Frontend-provided overlay | Not a Windows API; the frontend injects synthetic input |

A controller may work in one input API but not the other: some games only poll XInput, others only DirectInput. Mapping tools and SDL controller mappings live at the host/container layer, not inside Wine. On Android, touch controls are supplied by the frontend; Wine sees them as synthetic mouse or keyboard events. If a game ignores controllers entirely, check the prefix's input DLL overrides before hunting a kernel issue.

## 8.9 Filesystems, paths and permissions

- Windows software expects drive letters, backslashes and case-insensitive paths. Wine provides these; the host filesystem underneath remains case-sensitive on Linux. A game that opens `SAVE.DAT` after writing `save.dat` can fail on a case-sensitive host.
- `z:` maps to the host root, which makes shared folders easy but also exposes the host filesystem to the game. Keep sensitive data out of paths the game can browse.
- On Android, [scoped storage](https://developer.android.com/training/data-storage#scoped-storage) restricts access to shared folders. `termux-setup-storage` grants Termux access to public storage; the prefix itself should live under `$HOME`, where app-private storage applies.
- Device permissions matter: per-app Turnip injection needs `/dev/kgsl-3d0` to be accessible, but does not need root.
- Termux and its plugins share a `sharedUserId`. Never mix F-Droid, GitHub and Play Store builds; signature mismatches break the whole installation.
- On macOS, case-insensitivity depends on the volume's filesystem; do not assume Windows path semantics.

## 8.10 Android phantom process killer

Android 12+ monitors apps that spawn many child processes and kills them to save power. A Wine container with a `wineserver` plus game processes looks exactly like that pattern, so the game can die after a few minutes with no error in its own log.

Mitigations, from least to most invasive:

1. On Android 14+, Developer options → **Disable child process restrictions**.
2. On Android 12L/13 without root:
   ```sh
   adb shell "settings put global settings_enable_monitor_phantom_procs false"
   ```
3. On Android 12, the older count limit:
   ```sh
   adb shell device_config set_sync_disabled_for_tests persistent
   adb shell device_config put activity_manager max_phantom_processes 2147483647
   ```
   Order matters. [NEEDS VERIFICATION — these switches are widely reported in the community, but availability and exact names vary by device and Android build; check your device before relying on them.]

Also disable battery optimisation for the frontend and use `termux-wake-lock`. A wake lock keeps the CPU on; it does **not** keep the screen on. Keep Termux:X11 in the same app family so off-screen throttling does not kill rendering.

## 8.11 Prefix templates

Create a clean 64-bit prefix:

```sh
export WINEPREFIX="$HOME/.wine-game" WINEARCH=win64
WINEDEBUG=-all wineboot --init
```

Create a 32-bit-only prefix (only for games that truly need it):

```sh
export WINEPREFIX="$HOME/.wine-game32" WINEARCH=win32
WINEDEBUG=-all wineboot --init
```

A per-game launcher for the reference pipeline:

```sh
#!/bin/sh
export DISPLAY=:1
export PULSE_SERVER=127.0.0.1
export WINEPREFIX="$HOME/.wine-adreno"
export VK_DRIVER_FILES=/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json
export WINEDLLOVERRIDES="d3d9=n,dxgi=n"
export DXVK_HUD=devinfo,fps
exec wine "$HOME/Games/Game/game.exe"
```

Once a prefix is configured and verified, copy it to a template directory (after shutting Wine down) and restore that copy for a clean retry. This is the practical rollback for a broken dependency install.

## 8.12 Dependency checklist

Before blaming the translator, confirm:

| Check | How |
|---|---|
| Prefix exists and matches architecture | `WINEPREFIX` set; `wineboot` completed; `WINEDLLOVERRIDES` from template |
| Required runtimes installed | VC++ version named in the error, .NET/Mono, `d3dx9_*`, XAudio2 |
| Correct DLLs in correct folders | x64 in `system32`, x32 in `syswow64`; overrides match the installed translator |
| Audio server reachable | `PULSE_SERVER` set; server process alive |
| Input devices visible | evdev/SDL sees the controller; correct XInput vs DirectInput |
| Driver selected | `vulkaninfo --summary`, `DXVK_HUD=devinfo` (module 07) |
| Storage visible | `dosdevices` mappings; `z:` path; Android storage permission granted |
| Android background survival | phantom-process mitigations applied; wake lock held |

## 8.13 Non-Android hosts

The prefix concept is identical on x86 Linux, ARM Linux and macOS: a directory plus registry hives and installed DLLs. What changes is the surrounding container.

- On Steam for Linux, Proton creates and manages prefixes; the Steam Runtime supplies the userspace libraries, and no manual RootFS is needed.
- Tools such as Lutris, Bottles and Heroic expose the same controls (`WINEPREFIX`, Wine version, overrides) through a GUI.
- Flatpak-packaged frontends add a sandbox layer; grant filesystem and device access explicitly if the game cannot see the prefix or the Vulkan ICD.
- On ARM Linux, FEX/Box64 and the Mesa driver live in the host userspace, so the prefix still contains only Windows-side data — not the translator (module 12).
- On macOS, frontends manage Wine prefixes the same way; exact default paths are frontend-specific.

## 8.14 What to remember

- RootFS, container and prefix are three different layers; a prefix has no process isolation of its own.
- proot is a `ptrace`-based imitation of a container, not a real namespace sandbox.
- Choose `WINEARCH` once, at prefix creation; use separate prefixes to isolate games.
- `system32` is 64-bit, `syswow64` is 32-bit; DLL overrides decide built-in vs native.
- Install runtimes in response to actual errors, not in bulk.
- Audio, input and display are part of the prefix's working environment, not optional extras.
- On Android, fix storage permissions and the phantom process killer before debugging the game.

Next: [09 — Android Frontends](09-android-frontends.md).

## References

- Wine: https://gitlab.winehq.org/wine/wine ; Proton: https://github.com/ValveSoftware/Proton
- Containers: proot https://github.com/PRoot/proot ; proot-distro https://github.com/termux/proot-distro ; chroot https://man7.org/linux/man-pages/man8/chroot.8.html ; LXC https://linuxcontainers.org/lxc/introduction/ ; Flatpak https://flatpak.org/ ; Steam Runtime https://github.com/ValveSoftware/steam-runtime ; Distrobox https://github.com/89luca89/distrobox
- Termux: https://termux.dev/ ; F-Droid package https://f-droid.org/packages/com.termux/ ; Termux:X11 https://github.com/termux/termux-x11 ; QEMU https://www.qemu.org/
- FEX: https://github.com/FEX-Emu/FEX ; Box64: https://github.com/ptitSeb/box64
- Windows runtimes: VC++ https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist ; .NET https://dotnet.microsoft.com/ ; Mono https://www.mono-project.com/ ; Wine Mono https://gitlab.winehq.org/wine/wine-mono ; Wine Gecko https://gitlab.winehq.org/wine/wine-gecko ; DirectX SDK archive https://learn.microsoft.com/en-us/windows/win32/direct3d9/directx-sdk-archive-content ; Media Foundation https://learn.microsoft.com/en-us/windows/win32/medfound/media-foundation ; PhysX https://developer.nvidia.com/physx-sdk
- Audio APIs: WASAPI https://learn.microsoft.com/en-us/windows/win32/coreaudio/wasapi ; DirectSound https://learn.microsoft.com/en-us/windows/win32/directshow/directsound ; XAudio2 https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-introduction ; PulseAudio https://www.freedesktop.org/wiki/Software/PulseAudio/ ; PipeWire https://pipewire.org/ ; ALSA https://www.alsa-project.org/
- Input APIs: XInput https://learn.microsoft.com/en-us/windows/win32/xinput/getting-started-with-xinput ; DirectInput https://learn.microsoft.com/en-us/windows/win32/directinput ; evdev https://www.kernel.org/doc/html/latest/input/evdev.html ; SDL https://www.libsdl.org/
- Graphics translators referenced: DXVK https://github.com/doitsujin/dxvk ; VKD3D-Proton https://github.com/HansKristian-Work/vkd3d-proton ; D8VK https://github.com/AlpyneDreams/d8vk ; D7VK https://github.com/WinterSnowfall/d7vk
- winetricks: https://github.com/Winetricks/winetricks
- Android scoped storage: https://developer.android.com/training/data-storage#scoped-storage ; graphics stack https://source.android.com/docs/core/graphics
- Frontends named: Lutris https://lutris.net/ ; Bottles https://github.com/bottlesdevs/Bottles ; Heroic https://heroicgameslauncher.com/
