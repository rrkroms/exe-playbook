# 11 — ARM Linux Hosts

Modules 03–07 built the pipeline in the abstract: Wine for Windows APIs, a CPU translator for x86 instructions on ARM, DXVK/VKD3D-Proton for graphics, and a driver for the GPU. This module applies that pipeline to real machines that run a **native Linux userspace on ARM64 hardware**.

[ARM64](https://developer.arm.com/Architectures/A-Arch64) (also called AArch64) is the 64-bit instruction set used by Apple Silicon, Qualcomm Snapdragon, and modern Raspberry Pi boards. An **ARM Linux host** means a normal Linux distribution — glibc, X11 or Wayland, `/dev/dri`, Mesa — running directly on that hardware. It is not Android.

> **The kernel/userspace boundary matters.** Android also uses a Linux kernel, but its *userspace* is different: Bionic libc instead of glibc, scoped storage and app sandboxing instead of ordinary file permissions, SurfaceFlinger instead of X11/Wayland, and per-app native-library loading. That is why the recipes in modules 09–10 do not transfer unchanged. On ARM Linux you get a conventional Vulkan loader, ICD manifests and display server — but you still pay the ARM64-to-x86 translation cost.

## 11.1 The four layers you must line up

Every host in this module needs the same four layers. They are independent, and a working combination on one machine does not guarantee one on another.

| Layer | What it is | ARM Linux examples | What it is **not** |
|---|---|---|---|
| API compatibility | Implements Win32/NT APIs on Linux | [Wine](https://gitlab.winehq.org/wine/wine), [Proton](https://github.com/ValveSoftware/Proton) | Not a CPU translator; not a graphics translator |
| CPU translation | Executes x86/x86-64 instructions on ARM64 | [FEX](https://github.com/FEX-Emu/FEX), [Box64](https://github.com/ptitSeb/box64), [Box86](https://github.com/ptitSeb/box86) | Does not implement Windows APIs |
| Graphics translation | Converts Direct3D calls to Vulkan/OpenGL | [DXVK](https://github.com/doitsujin/dxvk), [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) | Not a driver; does not know about Wine internals |
| GPU driver | Turns Vulkan/OpenGL into GPU commands | Mesa [AGX](https://docs.mesa3d.org/drivers/asahi.html), [V3D/V3DV](https://docs.mesa3d.org/drivers/v3d.html), [Freedreno/Turnip](https://docs.mesa3d.org/drivers/freedreno.html), [Panfrost/PanVK](https://docs.mesa3d.org/drivers/panfrost.html) | Does not know what Direct3D is |

The layers compose in one direction. DXVK produces Vulkan; the Mesa driver consumes it. FEX executes the game's x86 instructions; Wine supplies the Windows API those instructions call. Confusing the layers is the root of most "it works on my other ARM board" reports.

```text
Windows game (x86/x86-64 PE)
        |
        v
  Wine / Proton          <-- Windows API compatibility
        |
        v
  FEX or Box64           <-- CPU translation (x86 instructions -> ARM64)
        |
        v
  DXVK / VKD3D-Proton    <-- graphics translation (D3D -> Vulkan)
        |
        v
  Mesa GPU driver        <-- Vulkan / OpenGL -> GPU commands
        |
        v
  Wayland / X11
```

This is a **conceptual data path**, not literal runtime order. In a real process, Wine, the translator, game worker threads and the driver's command submission run concurrently, and a single frame crosses the diagram many times. Use it to place blame, not to read a stack trace.

## 11.2 Two concepts that decide everything: page size and RootFS

A **page** is the fixed-size block of memory the kernel maps and protects. Most systems use 4096-byte (4K) pages; some ARM configurations use 16K or 64K. Prebuilt x86 binaries and Wine's ARM64EC path assume 4K pages. On a 16K host that assumption breaks — which is why Asahi needs a workaround (section 11.4).

A **RootFS** is a directory tree containing the loader and libraries a guest binary expects — for example an x86-64 `ld-linux-x86-64.so.2` and glibc. It is not a virtual machine disk image. FEX requires one on AArch64; its location defaults to `$HOME/.fex-emu/RootFS/` and can be selected with `FEX_ROOTFS` ([FEX configuration](https://wiki.fex-emu.com/index.php/Development:Configuration)). Box64 instead relies heavily on wrapping *native host* libraries, so its dependency story differs ([Box64 usage docs](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md)).

### 11.2.1 Finding the right Vulkan driver

[Vulkan](https://www.vulkan.org/) does not come from a single "GL driver". The [Vulkan loader](https://github.com/KhronosGroup/Vulkan-Loader) (`libvulkan.so.1`) discovers drivers through JSON manifests under `icd.d/`. On ARM Linux you can point it at a specific driver with `VK_DRIVER_FILES` (a colon-separated list of absolute manifest paths); `VK_ADD_DRIVER_FILES` adds to the discovered set, and the older `VK_ICD_FILENAMES` is a deprecated alias ([Vulkan loader docs](https://docs.vulkan.org/guide/latest/loader.html)).

```sh
VK_DRIVER_FILES=/usr/share/vulkan/icd.d/v3dv_icd.aarch64.json vulkaninfo --summary
VK_LOADER_DEBUG=error,warn vulkaninfo --summary
```

`vulkaninfo --summary` is the single most useful check on these hosts: it tells you whether DXVK will find a Vulkan 1.3/1.4 device at all. Note that these variables are ignored under `sudo` or setuid binaries ([Mesa env vars](https://docs.mesa3d.org/envvars.html)). Android differs sharply: it has no `VK_DRIVER_FILES` discovery route, and driver substitution is done per app instead.

## 11.3 Integration patterns: FEX, Box64 and muvm with Wine/Proton

Module 04 introduced the translators; module 03 showed the ARM64EC/WoW64 backend idea. Here is how they are actually combined on ARM Linux.

| Pattern | What runs natively | What is translated | Trade-off |
|---|---|---|---|
| **FEX backend DLL** (via [Hangover](https://github.com/AndreRH/hangover)) | Wine itself is ARM64 | Only the game's x86-64 / 32-bit code | Efficient; needs an ARM64EC/WoW64-aware Wine. `HODLL64=libarm64ecfex.dll`, `HODLL=libwow64fex.dll` |
| **Box64 whole-process** | Host libraries (GL, Vulkan, SDL) are wrapped natively | Wine **and** the game | Simple with existing x86-64 builds; more code pays translation cost |
| **Box86 / Box32** | Host libraries | 32-bit x86 code | Needed for old 32-bit games; `box86` needs an `armhf` chroot, `box32` is experimental |
| **[muvm](https://github.com/AsahiLinux/muvm) + FEX or Box64** | A 4K-page microVM with a 4K view of the rootfs | Everything inside the guest | Solves page-size problems; adds a fixed overhead |

**Choosing a translator on ARM Linux.** There is no universal winner; match the host and the game.

| Host / goal | Preferred path | Why |
|---|---|---|
| 4K pages, modern ARM64 Wine | FEX backend DLL (Hangover) | Translates only game code |
| 4K pages, only x86-64 Wine available | Box64 whole-process | Reuses existing builds |
| 32-bit legacy game | FEX `libwow64fex.dll`, or Box86 in `armhf` | Box64 alone handles 64-bit code only |
| 16K pages (Asahi, some Snapdragon) | FEX inside muvm | Requires a 4K guest environment |
| Tight memory budget | Tune Box64's DynaCache with `BOX64_DYNACACHE` after measuring actual RSS | Cache size/reservation is version-dependent |

Two distinctions to keep sharp:

- **ARM64EC** is a Windows-on-ARM *ABI* that lets ARM64 and x86-64 code coexist in one process by thunking calls between them. It is not a CPU translator. FEX supplies the backend that executes the x86-64 instructions.
- **WoW64** adapts 32-bit API calls to 64-bit libraries. It does not make a 32-bit x86 instruction executable on ARM64; `libwow64fex.dll` (FEX) or an experimental Box64 WOW64 DLL handles that.

**muvm** is a microVM built on libkrun. It gives x86 binaries a 4K-page environment, shares the host root filesystem with the guest, and lets you pick the emulator with `--emu=fex` or `--emu=box`. It is **not a CPU translator** — FEX or Box64 inside it performs the translation. The roughly 20% Asahi penalty associated with this setup comes from the microVM boundary, not from FEX itself ([Asahi progress report](https://asahilinux.org/2026/11/progress-report-7-3/)).

> [NEEDS VERIFICATION — one source records an Asahi progress report numbered 7.3 while another records AGX as fully upstream in Mesa 25.2 / Linux 6.16. The former is a report number, not necessarily a kernel version. Confirm the exact kernel version before relying on a number.]

## 11.4 Host 1: Asahi Linux (Apple Silicon)

**What it is.** [Asahi Linux](https://asahilinux.org/) ports Linux to Apple Silicon. Its graphics stack is upstream Mesa's AGX driver for OpenGL/OpenGL ES plus **HoneyKrisp**, the Vulkan driver in the same stack ([Asahi driver docs](https://docs.mesa3d.org/drivers/asahi.html)). Apple Silicon uses **16K pages**, which puts the direct Wine ARM64EC path out of reach; muvm supplies a 4K guest instead.

**Status snapshot (Nov 2026).** Wine ARM64EC support is mature, with FEX usable as the WoW64/ARM64EC backend. FEX is the primary translator and Box64 a fallback, both inside muvm. [Fedora Asahi Remix](https://fedoraproject.org/wiki/Asahi_Linux) is the recommended distribution; it provides a Steam wrapper that installs with `dnf install steam` and launches through muvm ([Fedora Asahi x86 support docs](https://docs.fedoraproject.org/ms/fedora-asahi-remix/x86-support/)). M1/M2 3D support is far along; M3 3D enablement is described as young.

**The full chain** (each arrow is a layer with its own failure modes):

```text
Steam wrapper (Fedora Asahi Remix)
    -> muvm (4K guest, shared rootfs)
    -> binfmt handler
    -> FEX (x86-64 instructions)
    -> x86 RootFS + Mesa overlay
    -> Wine / Proton (Windows APIs)
    -> DXVK / VKD3D-Proton (D3D -> Vulkan)
    -> AGX / HoneyKrisp (Vulkan -> GPU)
    -> host display path (DRM / framebuffer)
```

**Step-by-step notes.**

1. Start from Fedora Asahi Remix and update the kernel and Mesa together; AGX is reverse-engineered and tracks the upstream Mesa release train ([Mesa docs](https://docs.mesa3d.org/drivers/asahi.html)).
2. Install Steam with `dnf install steam`. The wrapper is what enters muvm — launching a raw x86-64 Wine binary from the 16K host will not work.
3. Confirm the Vulkan device from *inside* the 4K environment. `vulkaninfo --summary` ([Vulkan-Tools](https://github.com/KhronosGroup/Vulkan-Tools)) should report the AGX/HoneyKrisp driver; an ICD path that resolves only on the host will fail in the guest.
4. Select the translator: FEX is the default path; `muvm --emu=box` is the fallback ([muvm](https://github.com/AsahiLinux/muvm)). Keep the FEX RootFS available (`FEX_ROOTFS`).
5. For newer games use VKD3D-Proton for D3D12 and DXVK for D3D8–11; for old titles use the legacy wrappers from module 06, remembering they are mostly 32-bit.
6. Clear shader caches after changing Mesa or a translator; the Mesa cache default is 1 GiB and is controlled with `MESA_SHADER_CACHE_DIR` ([Mesa env vars](https://docs.mesa3d.org/envvars.html)).

**Limitations.** muvm has no Wayland support, slow I/O across the shared filesystem, and a roughly 20% overhead. The x86 overlay is immutable and gives no root, so you cannot freely install packages inside it. Anti-cheat is a particular problem on Asahi (see 11.9). M3 3D is still early.

## 11.5 Host 2: Raspberry Pi 4 and 5

**What it is.** The Pi 4 uses a Cortex-A72 and the Pi 5 a Cortex-A76. Their GPU is driven by Mesa's **V3D** OpenGL driver and **V3DV** Vulkan driver; V3DV is Vulkan 1.3 conformant ([V3D docs](https://docs.mesa3d.org/drivers/v3d.html)). Crucially, these boards use **4K pages**, so no muvm is needed — the simplest ARM setup in this book.

**Translation.** [Box64](https://github.com/ptitSeb/box64) with the Raspberry Pi 4/5 build flags is the mature path; it wraps native host GL/Vulkan libraries and uses DynaRec for hot guest code. [Box86](https://github.com/ptitSeb/box86) covers 32-bit games but expects an `armhf` (32-bit ARM) chroot with multilib. Wine's WoW64 under Box64 is experimental, so a 32-bit game is often easier through Box86 or a FEX/WoW64 path.

**Step-by-step notes.**

1. Use a current 64-bit Raspberry Pi OS with V3D/V3DV enabled; verify with `vulkaninfo --summary` that the V3DV device appears.
2. Build or install Box64 for the exact board (`RPI4` / `RPI5` flags). Enable DynaRec with `BOX64_DYNAREC=1`; restrict visible CPUs with `BOX64_SKIPCPU` / `BOX64_MAXCPU` if the scheduler puts guest threads on slower cores ([usage docs](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md)).
3. Launch a 64-bit game as `box64 wine64 game.exe` (whole-process pattern). For 32-bit titles, use an `armhf` chroot plus Box86 and a `WINEARCH=win32` prefix.
4. Keep expectations matched to the hardware: the GPU has low memory bandwidth, so lower resolution and effects before blaming the translator.
5. Diagnose with `BOX64_DYNAREC_TRACE` and `BOX64_DYNAREC_PERFMAP=1` (combine the map with `perf`). Check the current Box64 documentation for DynaCache sizing — memory pressure matters on smaller Pis.

**Limitations.** There is no native ARM build of Steam for these boards, so Steam Play is not a turnkey path. Pi 3 and older have a 2G/2G memory split that makes this class of setup impractical. V3D's GL is conformant for GLES but not full desktop GL, so the WineD3D OpenGL path is weaker than the Vulkan path through DXVK.

## 11.6 Host 3: Snapdragon X Elite

**What it is.** Snapdragon X Elite laptops use Oryon CPU cores and an Adreno GPU. The open-source path is `drm/msm` in the Linux kernel plus Mesa's Freedreno OpenGL and Turnip Vulkan drivers ([Freedreno docs](https://docs.mesa3d.org/drivers/freedreno.html)). Turnip is a Vulkan driver, not a graphics translator: it consumes the Vulkan that DXVK produces.

**Status snapshot.** Linux kernel 6.15 brought support for many X Elite laptops ([kernel](https://www.kernel.org/)). GPU firmware is not freely redistributable and must be extracted from the device's Windows installation — a real setup step, not an optional tweak. FEX demos on this hardware have been shown by Linaro/TUXEDO, with Box64 as an alternative [NEEDS VERIFICATION — the brief names the demo but provides no primary URL; confirm before attributing it]. There is no native ARM Steam release, so Wine/Proton must be arranged manually.

**Page size.** Some Snapdragon configurations use 16K pages. If you hit the Wine ARM64EC 4K requirement, use muvm or a Wine-emulation/P3S-style workaround; this is the same class of problem as Asahi, not a driver bug ([Android page-size docs](https://developer.android.com/guide/practices/page-sizes) for the general 16K constraint).

**Step-by-step notes.**

1. Install a distribution with a kernel that includes your specific laptop's `drm/msm` support; 6.15 covers many but not all models.
2. Extract the Adreno firmware from the Windows installation and place it where the kernel expects it. Without it, the GPU will not initialize.
3. Verify the display and Vulkan device before touching Wine. `vulkaninfo --summary` should name Turnip; note that community reports describe GPU initialization failures on some configurations [NEEDS VERIFICATION: no primary bug report in the briefs].
4. Choose FEX (backend DLL with ARM64EC Wine, or whole-process) or Box64 depending on whether you can supply an ARM64 Wine build.
5. Use `TU_DEBUG=sysmem` or `TU_DEBUG=gmem` to switch render path when diagnosing artifacts; these are Turnip debug controls, not Wine settings ([Mesa env vars](https://docs.mesa3d.org/envvars.html)).

**Limitations.** Firmware extraction, uneven laptop-specific kernel support, and the absence of a packaged Steam path make this the least "install and play" ARM host here. Keep the Qualcomm vendor driver for the desktop UI where possible and treat Turnip as the workload driver.

## 11.7 Host 4: ARM Chromebooks

ARM Chromebooks pair an ARM SoC — often a Mali GPU — with ChromeOS. This is a **virtualized** path rather than a native Linux desktop one.

[Crostini](https://chromeos.dev/en/linux/linux-on-chromeos-deep-dive) runs Linux apps in a VM (crosvm/KVM) plus a container, presenting graphics through Sommelier (a Wayland/X bridge). Graphics acceleration exposure is device-dependent, and Google has deprecated graphics acceleration for Linux-on-ChromeOS; a containerless "Baguette" mode is described from M147 [NEEDS VERIFICATION — confirm the current support statement with ChromeOS documentation]. **Borealis**, the managed Steam-in-an-Arch-VM environment that used Venus virtual Vulkan for near-native performance, ended its beta on 1 January 2026; only DIY Crostini remains ([ChromeOS Steam post](https://chromeos.dev/en/posts/bringing-steam-to-chromeos)).

For Mali GPUs the underlying driver is Mesa [Panfrost/PanVK](https://docs.mesa3d.org/drivers/panfrost.html), but inside the VM the path may instead be [Venus](https://docs.mesa3d.org/drivers/venus.html) (Vulkan streaming) or [VirGL](https://docs.mesa3d.org/drivers/virgl/) (OpenGL paravirtualization). Remember from module 05: VirGL and Venus carry *guest* graphics to the host; they do not replace DXVK. Box64/FEX can run Wine inside Crostini, but this is unsupported by Google.

**Step-by-step notes.** Enable the Linux development environment; confirm whether GPU acceleration is actually exposed; install a distro with Box64 or FEX and Wine; check which Vulkan ICD the container sees before installing DXVK; expect to rely on the Venus path and to lose features when acceleration is unavailable.

## 11.8 Host 5: other ARM single-board computers

Many SBCs use GPUs outside the four families above. The driver decides whether a Vulkan-based pipeline is even possible.

| GPU | Mesa driver | Vulkan availability | Consequence for Wine/Proton |
|---|---|---|---|
| Mali (Bifrost/Valhall) | [Panfrost/PanVK](https://docs.mesa3d.org/drivers/panfrost.html) | Vulkan 1.3 on Bifrost/Valhall, 1.4 on 5th gen | PanVK is experimental on many chips and historically gated by `PAN_I_WANT_A_BROKEN_VULKAN_DRIVER=1`; conformance verified only for G610 in the briefs |
| Vivante | Mesa `etnaviv` | Generally no usable Vulkan | D3D→Vulkan translators are out; WineD3D/OpenGL is the only path |
| PowerVR (Series6+) | [Mesa PVR](https://docs.mesa3d.org/drivers/powervr.html) + [Zink](https://docs.mesa3d.org/drivers/zink.html) | Vulkan 1.2 on supported BVNCs | No native Mesa GL for these chips; GL must come from Zink on Vulkan. Non-conformant GPUs require `PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1`; conformance is only claimed for specific BXM/BXS parts |
| Older Mali (Midgard/T6xx) | [Lima](https://docs.mesa3d.org/drivers/lima.html) | No Vulkan | Legacy OpenGL only; modern DXVK/VKD3D-Proton is not an option |

The general rule: without a conformant Vulkan driver, DXVK and VKD3D-Proton cannot work, and you fall back to WineD3D on OpenGL — with the quality problems module 06 describes for fixed-function content.

## 11.9 Limitations that apply across ARM Linux

| Problem | Why it happens | Practical response |
|---|---|---|
| Anti-cheat refusal | Kernel-level anti-cheat expects Windows; ARM Linux is the worst case, especially Asahi ([Proton docs](https://partner.steamgames.com/doc/steamhardware/proton)) | Single-player only; do not expect a workaround |
| 16K page hosts | Wine ARM64EC and prebuilt x86 binaries assume 4K | muvm (Asahi), Wine emulation or P3S-style workaround |
| 32-bit games | Need `libwow64fex.dll`, Box86/Box32, or QEMU user mode | Check the translator's 32-bit path before committing |
| Low GPU bandwidth | V3D, Panfrost and some Adreno parts | Lower resolution/effects; do not misdiagnose as translator slowness |
| Missing firmware | Adreno needs non-redistributable firmware | Extract from the device's Windows installation |
| No ARM Steam package | No native ARM Linux Steam build | Use Wine/Proton manually, or distribution wrappers such as Fedora Asahi's |
| Vulkan feature gaps | VKD3D-Proton needs Vulkan 1.3 plus descriptor indexing, robustness2 and push descriptor | Use DXVK where the D3D version allows; verify with `vulkaninfo` |

### 11.9.1 Verification checklist

Before you blame a translator, confirm the stack in this order:

1. `uname -m` → `aarch64`. Confirms the host instruction set.
2. `getconf PAGESIZE` → `4096` on most hosts, `16384` on Asahi and 16K Snapdragon configurations. If 16384, plan for muvm when using ARM64EC Wine.
3. `ls /dev/dri` → a render node must exist for hardware acceleration.
4. `vulkaninfo --summary` → a Vulkan device and the expected driver name (AGX, V3DV, Turnip, PanVK).
5. `VK_LOADER_DEBUG=error,warn vulkaninfo` → reveals ICD discovery failures.
6. `lsof /dev/ntsync` → on kernel 6.14+, confirms Wine/Proton synchronization is using the ntsync device (module 03).

Run steps 4–6 in the same environment the game will use. On Asahi that means inside muvm; on ChromeOS it means inside the Crostini VM.

### 11.9.2 Symptom table

| Symptom | Likely layer | First check |
|---|---|---|
| "Bad EXE format" at launch | Translator / prefix | 32-bit path, FEX RootFS, `WINEARCH` |
| No Vulkan device | ICD / driver | `VK_DRIVER_FILES`, firmware, `vulkaninfo` |
| Wine ARM64EC fails on a 16K host | Page size | Run inside muvm; check `getconf PAGESIZE` |
| One core pinned, low GPU use | CPU translation | Try the other translator; check `BOX64_DYNAREC` |
| GPU initialization failure on X Elite | Firmware / kernel | Adreno firmware present, kernel model support |
| Black screen with audio | Presentation | Display server, fullscreen or virtual desktop |

### 11.9.3 32-bit support at a glance

| Need | Native x86-64 host | ARM64 host |
|---|---|---|
| 32-bit game, backend DLL path | Wine WoW64 | `libwow64fex.dll`, or experimental Box64 WOW64 DLL |
| 32-bit game, whole-process path | — | Box86 in an `armhf` chroot, or experimental Box32 |
| 32-bit game, QEMU user mode | possible, slow | possible, slow |

WoW64 solves the 32-bit *API* problem; it never makes a 32-bit x86 instruction executable on ARM64 by itself.

## 11.10 What to remember

- ARM Linux has a normal Linux userspace; Android shares the kernel but not the userspace. That boundary changes drivers, display and library loading.
- Four independent layers are required: Wine/Proton (API), FEX/Box64 (CPU), DXVK/VKD3D-Proton (graphics), Mesa (driver).
- Wine ARM64EC needs 4K pages. Asahi's 16K pages require muvm; muvm is an environment provider, not a translator.
- FEX backend DLLs translate only game code; Box64 whole-process is simpler but translates Wine too.
- Asahi: AGX/HoneyKrisp + muvm + FEX is the mature path; M3 3D and no-Wayland muvm are the main gaps.
- Raspberry Pi 4/5: 4K pages, V3D/V3DV, Box64/Box86; low GPU bandwidth is the real limit.
- Snapdragon X Elite: Freedreno/Turnip, kernel 6.15-era support, firmware extraction required.
- Without a Vulkan driver, a modern D3D→Vulkan translator cannot help; check `vulkaninfo` before debugging Wine.

Next: [12 — x86_64 Linux and Other Hosts](12-linux-x86-and-other-hosts.md).

## References

- Asahi Linux: https://asahilinux.org/ ; progress report 7.3: https://asahilinux.org/2026/11/progress-report-7-3/
- Asahi / Apple AGX Mesa driver docs: https://docs.mesa3d.org/drivers/asahi.html
- Fedora Asahi Remix: https://fedoraproject.org/wiki/Asahi_Linux ; x86 support docs: https://docs.fedoraproject.org/ms/fedora-asahi-remix/x86-support/
- muvm: https://github.com/AsahiLinux/muvm
- FEX: https://github.com/FEX-Emu/FEX ; configuration: https://wiki.fex-emu.com/index.php/Development:Configuration
- Box64: https://github.com/ptitSeb/box64 ; usage docs: https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md ; Wine notes: https://github.com/ptitSeb/box64/blob/main/docs/WINE.md
- Box86: https://github.com/ptitSeb/box86
- Hangover (FEX/Box64 Wine backends): https://github.com/AndreRH/hangover
- Wine: https://gitlab.winehq.org/wine/wine ; Proton: https://github.com/ValveSoftware/Proton
- DXVK: https://github.com/doitsujin/dxvk ; VKD3D-Proton: https://github.com/HansKristian-Work/vkd3d-proton
- Mesa driver index and environment variables: https://docs.mesa3d.org/ ; https://docs.mesa3d.org/envvars.html
- Vulkan loader documentation: https://docs.vulkan.org/guide/latest/loader.html ; Vulkan loader source: https://github.com/KhronosGroup/Vulkan-Loader
- V3D / V3DV: https://docs.mesa3d.org/drivers/v3d.html
- Freedreno / Turnip: https://docs.mesa3d.org/drivers/freedreno.html
- Panfrost / PanVK: https://docs.mesa3d.org/drivers/panfrost.html
- Mesa PVR: https://docs.mesa3d.org/drivers/powervr.html ; Zink: https://docs.mesa3d.org/drivers/zink.html
- Venus: https://docs.mesa3d.org/drivers/venus.html ; VirGL: https://docs.mesa3d.org/drivers/virgl/
- Linux kernel: https://www.kernel.org/
- Vulkan-Tools (`vulkaninfo`): https://github.com/KhronosGroup/Vulkan-Tools
- Android page-size documentation: https://developer.android.com/guide/practices/page-sizes
- ChromeOS Linux deep dive: https://chromeos.dev/en/linux/linux-on-chromeos-deep-dive
- Bringing Steam to ChromeOS: https://chromeos.dev/en/posts/bringing-steam-to-chromeos
- Steam Proton documentation: https://partner.steamgames.com/doc/steamhardware/proton
