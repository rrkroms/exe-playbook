# 03 — Wine and Proton

This module is about the **Windows API compatibility layer**: the part of the pipeline that answers "where is `CreateFileW`?" and "what does the registry look like?" It does **not** answer "how is an x86 instruction executed on ARM64?" (module 04) or "how does Direct3D become Vulkan?" (module 05).

Everything here applies to generic non-Windows hosts. On x86-64 Linux, Wine runs natively. On ARM64 Linux, Android and Apple Silicon, the same Wine concepts apply, but a CPU translator sits alongside them.

## 3.1 Four things that are often confused

Before going further, separate these categories. They are independent, and most configuration mistakes come from mixing them up.

| Term | Category | What it does | What it does **not** do |
|---|---|---|---|
| [Wine](https://gitlab.winehq.org/wine/wine) | API compatibility | Implements Win32/NT APIs, PE loading, registry, DLLs on a POSIX host | Not a CPU translator; not a Windows kernel; not a graphics translator |
| [Proton](https://github.com/ValveSoftware/Proton) | Distribution | A curated Wine build plus runtimes and Steam integration | Not a separate compatibility engine; it *is* Wine, configured and packaged |
| [WoW64](https://learn.microsoft.com/en-us/windows/win32/winprog64/running-32-bit-applications) | Process/ABI compatibility | Lets 32-bit Windows processes call 64-bit system libraries through thunks | Not CPU emulation; not a replacement for a CPU translator |
| [FEX](https://github.com/FEX-Emu/FEX) / [Box64](https://github.com/ptitSeb/box64) | CPU translation | Executes x86/x86-64 instructions on ARM64 | Do not implement Windows APIs |
| [DXVK](https://github.com/doitsujin/dxvk) / [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) / [WineD3D](https://gitlab.winehq.org/wine/wine) | Graphics API translation | Convert Direct3D calls into Vulkan or OpenGL | Do not implement the rest of Windows; do not translate CPU instructions |

Keep this table in mind whenever a frontend offers you a "Wine version", a "Proton version", a "translator" and a "DXVK version". Those are four different knobs.

## 3.2 Wine's moving parts

Wine is a collection of native host programs and shared libraries, not a single monolith. The important pieces are:

```text
  native host executable:  wine / wine64
                 |
                 v
  +---------------------------------------------+
  |  PE loader                                  |
  |  reads .exe/.dll headers, maps sections,    |
  |  resolves imports, transfers control        |
  +----------------------+----------------------+
                         |
                         v
  +---------------------------------------------+
  |  Wine DLLs (native .so implementing Win32)  |
  |  kernel32, ntdll, user32, advapi32, ...     |
  +-----------+-------------------+-------------+
              |                   |
              v                   v
     host system calls     wineserver process
     (files, threads,      (one per prefix)
      sockets, display)     registry, sync objects,
                            window/process coordination
```

**The loader.** The `wine` command is itself a native ELF or Mach-O program. It reads the PE headers of the Windows executable, maps sections into memory, resolves imports from DLLs, sets up the stack and guest thread environment, and jumps to the guest entry point. "Bad EXE format" usually means this stage rejected the file: wrong architecture, a 16-bit executable, or a .NET assembly that needs a runtime rather than the PE loader.

**The DLLs.** Wine ships native shared objects that export the entry points of Windows DLLs. When the game calls `kernel32!CreateFileW`, it lands in Wine's implementation, which translates the request into host calls. This is *API compatibility*: the function names, calling convention, error codes and side effects are reproduced, but the code executing is Wine's, not Microsoft's.

**`wineserver`.** A separate native process, normally one per prefix, that owns state shared across Wine processes: the registry, synchronization objects, handle tables, and much of the window and process coordination. Wine DLLs talk to it over a local socket. It runs entirely in host userspace — it is not a kernel, and it cannot load Windows kernel drivers.

**The registry.** A Windows-style hierarchical database that many games use for configuration. Wine stores it as plain text files inside the prefix (`system.reg`, `user.reg`, `userdef.reg`), not as Windows registry hives. You can edit it with `wine regedit`.

**The prefix.** A directory tree that looks like a Windows installation to the guest, including drive letters, `windows/`, `Program Files/` and `users/`.

## 3.3 Prefixes, drives and `winecfg`

A **prefix** is the unit of Windows state. It contains the fake `C:\`, the registry files, installed runtimes, DLL overrides and per-game settings. Using one prefix per game is the safest habit: a runtime installed for one title rarely helps another, and a bad override in a shared prefix can break everything.

```text
$WINEPREFIX/
├── drive_c/
│   ├── Program Files/
│   ├── Program Files (x86)/
│   ├── users/
│   └── windows/
├── dosdevices/
│   ├── c: -> ../drive_c
│   └── z: -> /            # host root filesystem
├── system.reg
├── user.reg
└── userdef.reg
```

The `dosdevices` symlinks define drive letters. `z:` normally points at the host root, which is how a guest program reaches host files — subject to the host's permissions and, on Android, the app sandbox and scoped storage.

Three environment variables control the basics:

| Variable | Meaning | Typical value |
|---|---|---|
| `WINEPREFIX` | Which prefix to use | `$HOME/.wine`, `~/games/foo` |
| `WINEARCH` | Architecture a *new* prefix is created as | `win64` (default); `win32` only for a pure 32-bit prefix |
| `WINEDLLOVERRIDES` | Per-launch DLL implementation choices | `d3d9=n,dxgi=n` |

`wineboot -u` updates an existing prefix after a Wine upgrade; `winecfg` is the graphical control panel for the Windows version reported to applications, virtual desktop, drive mappings, audio and DLL overrides. The Windows version reported here matters: some installers refuse to run unless you select XP SP3 or Windows 7 ([legacy DirectX notes](https://github.com/WinterSnowfall/d7vk)).

**What to avoid:** copying a prefix between machines or between very different Wine versions and expecting identical behaviour. Prefix incompatibilities are a common source of "it worked yesterday" reports.

## 3.4 Wine's WoW64 — and what it is not

**WoW64** means "Windows 32-bit on Windows 64-bit". On Windows, it is the layer that lets a 32-bit process run on a 64-bit OS by converting 32-bit API arguments and data structures into 64-bit ones before they reach the kernel. Wine implements an equivalent mechanism so that a 32-bit Windows executable can run inside a 64-bit Wine process.

> **Important — do not call WoW64 a CPU translator or an emulator.**
> WoW64 changes *how a 32-bit program calls a 64-bit API*. It does not decode or recompile instructions. On an x86-64 host, the CPU executes 32-bit x86 code natively. On an ARM64 host, WoW64 solves only the 32-bit-to-64-bit API problem; the x86 or x86-64 instructions still require a CPU translator such as FEX or Box64 (module 04).

A 64-bit prefix can generally host 32-bit applications through WoW64. Use `WINEARCH=win32` only when you deliberately want a pure 32-bit prefix, for example for older games whose installers or wrappers are 32-bit-only.

The [Wine 11.0 announcement](https://www.winehq.org/announce/11.0) lists "New WoW64" among its supported features. On ARM64, WoW64 and [ARM64EC](https://learn.microsoft.com/en-us/windows/arm/arm64ec) support are closely related [NEEDS VERIFICATION — the available sources confirm both features exist but do not spell out the exact dependency between the new WoW64 implementation and ARM64EC thunk infrastructure; check the announcement text before relying on that relationship].

## 3.5 Built-in and native DLLs, and overrides

For any given Windows DLL there may be two candidates:

| Kind | What it is | Example |
|---|---|---|
| **Built-in** | Wine's native shared object implementing that DLL's API | Wine's `d3d9` implementing Direct3D 9 entry points |
| **Native** | A real Windows DLL copied into the prefix, usually from a redistributable or a graphics wrapper | `d3dx9_43.dll`, DXVK's `d3d9.dll` |

**DLL overrides** decide which one wins. The `WINEDLLOVERRIDES` syntax is a comma-separated list of `dll=order`, where `n` means native and `b` means built-in:

```sh
# Use DXVK's Windows DLLs rather than Wine's Direct3D implementations
WINEDLLOVERRIDES="d3d9=n,dxgi=n" wine game.exe

# Prefer a native ddraw.dll but fall back to Wine's if absent
WINEDLLOVERRIDES="ddraw=n,b" wine oldgame.exe
```

The order matters: `ddraw=n,b` tries native first, then built-in. Some legacy wrappers require exactly this ordering ([D7VK notes](https://github.com/WinterSnowfall/d7vk)).

You can also set overrides persistently in `winecfg` under Libraries. The environment variable takes precedence for that launch, which makes it useful for experiments.

Two rules save a lot of time:

1. **Copying a DLL into `system32` without an override usually does nothing.** Wine will keep using its built-in implementation unless you tell it otherwise ([legacy DirectX pitfalls](https://github.com/elishacloud/dxwrapper)).
2. **Do not stack two wrappers that claim the same DLL.** For example, dgVoodoo2's D3D9 output can be serviced by DXVK's `d3d11.dll`/`dxgi.dll`, but you should not override `d3d9` with DXVK while dgVoodoo's `d3d9` is active ([dgVoodoo2 notes](https://github.com/dege-diosg/dgVoodoo2)).

## 3.6 Runtimes games expect

Wine implements the *APIs*, but many games ship expecting specific *redistributables* to be installed. A "missing DLL" error is frequently a missing runtime, not a missing Wine feature.

| Runtime | Why it matters | Wine-side answer |
|---|---|---|
| [Visual C++ Redistributable](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist) | Supplies `msvcr*.dll`, `msvcp140.dll` and friends used by almost every C/C++ game | Install the matching version into the prefix; Proton bundles many of them |
| [.NET Framework](https://dotnet.microsoft.com/) | Managed launchers, installers and WPF-based UI | [Wine Mono](https://gitlab.winehq.org/wine/wine-mono) is an open-source substitute; some WPF applications remain problematic ([compatibility notes](https://github.com/ValveSoftware/Proton)) |
| [Wine Gecko](https://gitlab.winehq.org/wine/wine-gecko) | HTML rendering used by some installers and embedded UI | Install into the prefix; without it, some installers show a blank dialog |
| DirectX 9.0c-era components | Helper DLLs such as `d3dx9_*.dll`, `xaudio2_*.dll`, `xinput*.dll` | Wine implements Direct3D itself; install the *helper* DLLs, and do not override `d3d9` with a random Windows DLL if you intend to use DXVK |
| Media Foundation | Video playback and capture | Proton includes an FFmpeg-backed implementation; coverage is still incomplete ([Proton docs](https://partner.steamgames.com/doc/steamhardware/proton)) |
| PhysX, OpenAL, XNA, ... | Middleware shipped by game installers | Usually installed as a native Windows runtime via winetricks or the game's own installer |

Note the subtlety with DirectX: installing "DirectX" through winetricks typically adds D3DX/XAudio/XInput helper DLLs. It does not replace Wine's Direct3D implementation, and it does not remove the need for a graphics translator such as DXVK or VKD3D-Proton.

## 3.7 winetricks

[winetricks](https://github.com/Winetricks/winetricks) is a community-maintained script that downloads and installs Windows redistributables, helper DLLs and common per-prefix settings. It is **not part of Wine**; it is a convenience tool that operates on a prefix.

```sh
export WINEPREFIX="$HOME/games/my-game"
winetricks vcrun2022 d3dx9_43 xaudio2
```

Practical advice:

- Always set `WINEPREFIX` before running it; otherwise you may modify your default prefix.
- Some components require a 32-bit prefix, some a 64-bit one, and a few require both.
- Changes are cumulative and not cleanly reversible. Keep a throwaway prefix for experiments.
- Availability varies by host. On Debian it is packaged in `contrib` (`apt install winetricks`); in a Termux proot you install it inside the Linux distribution, since Termux itself has no official package ([Termux pipeline notes](https://github.com/termux/proot-distro)).

## 3.8 Synchronization: esync, fsync and ntsync

Windows games coordinate threads with **synchronization primitives** — events, mutexes and semaphores. Wine must implement them using host primitives. The implementation mostly affects *performance*, but it is one of the most misunderstood parts of Proton configuration.

| Mechanism | Host primitive | Status |
|---|---|---|
| **esync** | `eventfd` | An older optimisation; **obsoleted in Proton 11.0** ([Proton repo](https://github.com/ValveSoftware/Proton)) |
| **fsync** | Linux `futex`, specifically `FUTEX_WAIT_MULTIPLE` | Used where available; automatically disabled if the kernel lacks the required futex operation |
| **ntsync** | The `/dev/ntsync` kernel character device | Complete in mainline Linux **6.14**; initial Wine support in 10.15, headline feature in Wine 11.0; Proton 11.0 enables it automatically when the device can be opened ([kernel docs](https://docs.kernel.org/userspace-api/ntsync.html), [Wine 11.0](https://www.winehq.org/announce/11.0)) |

```text
Game threads
     |
     v
Wine synchronization objects  (events, mutexes, semaphores)
     |
     +--> host futex / eventfd        (fsync, esync)
     |
     +--> /dev/ntsync                 (ntsync, kernel 6.14+)
```

Two practical corrections to folklore:

- **There is no `PROTON_NTSYNC=1` variable.** ntsync is automatic in Proton 11.0 when usable. The way to turn it *off* is `PROTON_NO_NTSYNC` ([Proton documentation](https://github.com/ValveSoftware/Proton), [Debian how-to](https://wiki.debian.org/Wine/NtsyncHowto)).
- You can check whether the device is available and in use with `lsof /dev/ntsync`. If your kernel is older than 6.14, ntsync simply is not an option; do not try to force it.

## 3.9 Proton and Proton-GE

**Proton** is Valve's *distribution* of Wine for Steam Play. It combines a Wine build, patches, bundled runtimes (DXVK, VKD3D-Proton, Mono, Gecko), and Steam-specific integration. There is no separate "Proton engine": when you read Proton logs you are reading Wine logs with extra components around them ([Proton](https://github.com/ValveSoftware/Proton), [Steam documentation](https://partner.steamgames.com/doc/steamhardware/proton)).

Because Proton is a curated build, it also sets many environment variables for you: DLL overrides, graphics translator options, sync mode and logging. Steam selects a Proton version per game; tools such as ProtonUp-Qt can install additional versions ([ProtonUp-Qt](https://github.com/DavidoTek/ProtonUp-Qt)).

**Proton-GE** is a community build by GloriousEggroll that adds its own patches, commonly used for non-Steam games and for newer upstream Wine features ([Proton-GE](https://github.com/GloriousEggroll/proton-ge-custom)). Treat it as another *distribution* of Wine, not as a different category of compatibility layer.

A warning for ARM users: do not assume a Proton build from one architecture can be reused under a translator. Match the build to the host and to the translation strategy; an ARM64 Proton build is not a drop-in replacement inside an x86 Steam installation running through FEX.

## 3.10 Wine 11.0 in context

The [Wine 11.0 stable release](https://www.winehq.org/announce/11.0) was announced on 13 January 2026; the development branch subsequently reached 11.22 (check the Wine release pages for current numbers). Relevant headline items for this book are the new WoW64 work and ntsync support. Proton 11.0-1 was the corresponding stable Proton release, with Experimental tracks moving independently.

Wine's release train matters because graphics back ends, WoW64 behaviour and sync support change between versions. When you compare two frontends, record the exact Wine version each one bundles — "Wine" alone is not a reproducible description.

## 3.11 ARM64EC and FEX on ARM

On an ARM64 host you have two broad strategies, and the choice determines what gets translated.

**Strategy A — native ARM64 Wine with backend DLLs.** Wine itself is compiled for ARM64, and only the Windows application's x86/x86-64 code is translated. [Hangover](https://github.com/AndreRH/hangover) provides the integration: `libarm64ecfex.dll` handles the 64-bit x86 side and `libwow64fex.dll` the 32-bit side; a Box64-based `wowbox64.dll` variant also exists ([Hangover README](https://github.com/AndreRH/hangover)).

```text
  32-bit / 64-bit x86 Windows game (PE)
             |
             v
  +--------------------------------------+
  | ARM64 Wine + WoW64 / ARM64EC thunks  |
  +----------------+---------------------+
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
libarm64ecfex  libwow64fex   wowbox64.dll
 (FEX x64)      (FEX 32)      (Box64 path)
       |
       v
Native ARM64 Wine DLLs -> host syscalls -> host GPU driver
```

In a Termux-based setup these are selected with `HODLL64=libarm64ecfex.dll` and `HODLL=libwow64fex.dll` ([Hangover](https://github.com/AndreRH/hangover)). This is the recommended approach where available, because Wine stays native and only game code pays translation cost.

**Strategy B — whole x86-64 Wine under Box64.** You run an ordinary x86-64 Wine build as a guest process under Box64. This reuses existing Windows-oriented build artifacts but translates Wine as well as the game, so more code pays the cost. It is a documented fallback, not the preferred design ([Box64 + Wine notes](https://github.com/ptitSeb/box64)).

ARM64EC itself is a Windows-on-ARM ABI that allows ARM64 and x86-64 code to coexist in one process by thunking calls between them. It is an ABI mechanism, not a CPU translator. FEX supplies the backend that actually executes x86-64 instructions.

**Page size constraint.** Wine ARM64EC requires **4K host pages**; 16K and 64K page hosts are unsupported for that path ([compatibility notes](https://github.com/FEX-Emu/FEX)). Asahi Linux, which uses 16K pages, works around this with [muvm](https://github.com/AsahiLinux/muvm), a microVM that gives the guest a 4K-page view. Module 04 explains the page-size problem in detail.

## 3.12 macOS Wine

macOS is a non-Windows host with a different graphics stack, but the Wine concepts above still apply.

- Native macOS builds of Wine exist. On Apple Silicon, an x86-64 Windows executable needs either [Rosetta 2](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment) running an x86-64 Wine build, or an ARM64EC-aware Wine with a backend such as FEX. Rosetta translates CPU instructions; it does not implement Windows APIs.
- [CrossOver](https://www.codeweavers.com/crossover) is a commercial Wine distribution. CrossOver 26 shipped Wine 11.0 along with D3DMetal 3.0, DXMT 0.72 and vkd3d 1.18 ([host summary](https://www.codeweavers.com/crossover)).
- Apple's [Game Porting Toolkit](https://developer.apple.com/games/game-porting-toolkit) is a developer evaluation environment that includes D3DMetal. Third-party wrappers generally require the user to supply the toolkit components themselves; it is closed source.
- The [Whisky](https://github.com/frankea/Whisky) fork is an active community frontend.
- Apple's OpenGL is frozen at 4.1, which leaves WineD3D as a legacy-only path on macOS. The preferred modern routes go through Metal-backed translators; see module 05.

## 3.13 Common failures

| Symptom | Likely layer | First check |
|---|---|---|
| "Bad EXE format" | Loader / architecture | File architecture, `WINEARCH`, native vs translated Wine |
| Wine starts, game instantly exits | Prefix or runtime | Fresh prefix, `wineboot -u`, VC++/.NET/DirectX helpers |
| "The program can't start because ...dll is missing" | DLL resolution | Overrides, winetricks runtime, DLL bitness (32 vs 64) |
| Installer shows a blank HTML dialog | Gecko | Install Wine Gecko into the prefix |
| .NET setup dialog loops | Mono / .NET | Try Mono, check whether the launcher is WPF-based |
| Works after `winecfg` Windows-version change | Reported OS version | Set XP SP3/Windows 7 for older installers |
| Graphics init crashes | DXVK/VKD3D/driver | Vulkan device creation, wrapper logs |
| Game runs but is extremely slow on one core | CPU translation | Translator choice and settings (module 04) |
| Stutter only at startup | Shader compilation | Shader cache (module 05) |
| Anti-cheat refuses to launch | Vendor component | Kernel anti-cheat generally blocks non-Windows hosts; a Linux EAC v2 effort exists but requires signed kernel/eBPF support not present in typical setups ([Proton docs](https://partner.steamgames.com/doc/steamhardware/proton)) |

## 3.14 What to remember

- Wine implements Windows APIs; Proton is a curated distribution of Wine; neither translates CPU instructions.
- WoW64 is an API/ABI thunking mechanism for 32-bit processes — never a CPU emulator or translator.
- The prefix is the unit of state: registry files, drives, overrides and installed runtimes all live there.
- Built-in versus native DLLs is decided by overrides; copying a DLL without an override changes nothing.
- VC++, Mono/.NET, Gecko and D3DX/XAudio helper DLLs are runtime dependencies, not Wine features.
- ntsync is automatic on kernel 6.14+ with Proton 11.0; disable it with `PROTON_NO_NTSYNC`, not `PROTON_NTSYNC=1`.
- On ARM64, decide whether you translate only the game (ARM64 Wine + FEX/Hangover backend) or the whole Wine process (Box64).

Next: [04 — CPU Translation](04-cpu-translation.md).

## References

- Wine source: https://gitlab.winehq.org/wine/wine
- Wine 11.0 announcement: https://www.winehq.org/announce/11.0
- Proton: https://github.com/ValveSoftware/Proton
- Proton-GE: https://github.com/GloriousEggroll/proton-ge-custom
- Steam Proton documentation: https://partner.steamgames.com/doc/steamhardware/proton
- Linux kernel ntsync documentation: https://docs.kernel.org/userspace-api/ntsync.html
- Debian ntsync how-to: https://wiki.debian.org/Wine/NtsyncHowto
- FEX: https://github.com/FEX-Emu/FEX
- Box64: https://github.com/ptitSeb/box64
- Hangover: https://github.com/AndreRH/hangover
- muvm: https://github.com/AsahiLinux/muvm
- DXVK: https://github.com/doitsujin/dxvk
- VKD3D-Proton: https://github.com/HansKristian-Work/vkd3d-proton
- D7VK (legacy D3D override notes): https://github.com/WinterSnowfall/d7vk
- dxwrapper (override pitfalls): https://github.com/elishacloud/dxwrapper
- dgVoodoo2: https://github.com/dege-diosg/dgVoodoo2
- winetricks: https://github.com/Winetricks/winetricks
- Wine Mono: https://gitlab.winehq.org/wine/wine-mono
- Wine Gecko: https://gitlab.winehq.org/wine/wine-gecko
- Termux proot-distro: https://github.com/termux/proot-distro
- ProtonUp-Qt: https://github.com/DavidoTek/ProtonUp-Qt
- Apple Rosetta 2 documentation: https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment
- Apple Game Porting Toolkit: https://developer.apple.com/games/game-porting-toolkit
- CrossOver: https://www.codeweavers.com/crossover
- Whisky (frankea fork): https://github.com/frankea/Whisky
- Microsoft WoW64 documentation: https://learn.microsoft.com/en-us/windows/win32/winprog64/running-32-bit-applications
- Microsoft ARM64EC documentation: https://learn.microsoft.com/en-us/windows/arm/arm64ec
