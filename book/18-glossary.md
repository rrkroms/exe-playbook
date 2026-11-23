# 18 — Glossary

Key terms are defined as used in this book. Names are linked to official upstream documentation where available. For the full tool directory, see [19 — References](19-references.md).

## A

**ABI (Application Binary Interface)** — The low-level contract between compiled code and its environment: calling conventions, register usage, stack layout, object formats and system-call conventions. Different from an API.

**AdrenoTools** — Community tooling used by Android frontends to load a custom Adreno Vulkan driver (such as [Turnip](https://docs.mesa3d.org/drivers/freedreno.html)) inside a single app process without rooting.

**AGX** — Apple GPU architecture used in Apple Silicon Macs; supported on Linux by the [Asahi](https://asahilinux.org/) Mesa driver.

**ANV** — Mesa’s Vulkan driver for Intel GPUs.

**API (Application Programming Interface)** — A higher-level contract a program calls, such as Win32 or Direct3D. Wine implements Windows APIs; DXVK translates graphics APIs.

**ARM64EC** — A Windows-on-Arm ABI that follows x64 software conventions so ARM64EC code and x64 code can coexist in one process. An x64 or ARM64EC process can load both kinds of binary; a native ARM64 process cannot load x64 binaries. The x64 parts run under emulation — Microsoft's built-in emulator on Windows, or [FEX](https://github.com/FEX-Emu/FEX) when used as a backend DLL in Wine/Hangover. It is not the same as generic ARM64 or WoW64.

## B

**binfmt_misc** — Linux kernel facility that associates executable formats with user-space interpreters. Used by FEX and muvm to launch x86 binaries transparently.

**Box32** — Experimental 32-bit x86 support in [Box64](https://github.com/ptitSeb/box64); not the same as the older Box86, which requires a 32-bit armhf userland.

**Box64** — A user-space x86-64 dynamic binary translator for ARM64 and other hosts that uses dynamic recompilation and wraps native host libraries. It translates CPU instructions, not Windows APIs.

**Box86** — A legacy 32-bit x86 translator for ARM hosts, typically requiring a 32-bit (armhf) userland.

## C

**CSMT** — Command Stream Multi-Threading, a WineD3D feature that moves graphics command submission to a separate thread. Enabled by default in modern Wine.

**Container** — An isolated userland sharing the host kernel (proot, chroot, Flatpak, Steam Runtime). Not a virtual machine.

**Crostini** — ChromeOS’s Linux environment: a virtual machine containing a container, with graphics forwarded through Sommelier.

## D

**DBT (Dynamic Binary Translation)** — Translating guest machine instructions to host instructions at runtime, usually with a JIT and a translation cache.

**D3D / Direct3D** — Microsoft’s 3D graphics API family. Versions 7–12 have very different feature sets and need different translators.

**D3D8to9** — A library that translates Direct3D 8 calls and shaders into Direct3D 9, which can then be handled by DXVK.

**D7VK** — A Vulkan-based implementation of Direct3D 7 immediate mode, built on DXVK’s D3D9 backend and requiring a real DirectDraw implementation underneath.

**D8VK** — A Direct3D 8 to Vulkan translator, now merged into upstream DXVK.

**dgVoodoo2** — Closed-source freeware wrapper for Glide and legacy DirectX (D3D3–9) that outputs Direct3D 11/12. Its authors explicitly do not support Wine/Linux.

**DLL override** — A Wine mechanism (`WINEDLLOVERRIDES` or winecfg) that chooses between Wine’s built-in DLL and a native Windows DLL. Graphics translators are installed as native DLLs.

**DXVK** — A Vulkan-based implementation of Direct3D 8/9/10/11 for Wine/Proton.

**DXVK-NVAPI** — A layer that translates NVIDIA NVAPI calls to DXVK/VKD3D-Proton extension points, enabling features such as DLSS and Reflex on supported hardware.

## E

**ELF** — Executable and Linkable Format, the native binary format on Linux and Android. Windows programs use PE instead.

**Emulation** — Strictly, reproducing a complete machine or platform. Compatibility layers and translators reproduce APIs or instructions, not necessarily a whole machine.

**esync** — An older Wine/Proton synchronization optimisation using eventfd. Obsolete in Proton 11.0.

## F

**Feature level** — A Direct3D capability tier; a game can request a specific feature level even when using a newer API.

**FEX** — An x86/x86-64 dynamic binary translator for ARM64 that can also be built as ARM64EC/WoW64 backend DLLs for Wine. Uses a custom IR, JIT, RootFS and thunks.

**Fixed-function pipeline** — Pre-shader graphics hardware stages used by very old games. Modern translators implement it in shaders.

**fsync** — A Wine/Proton synchronization optimisation based on Linux futexes.

**Fossilize** — A Vulkan pipeline/cache serialization format and replay tool used for shader pre-caching and bug reproduction.

**FSR** — AMD FidelityFX Super Resolution, a family of upscaling (and later frame-generation) technologies. Gamescope implements FSR 1.

## G

**Gamescope** — Valve’s Wayland micro-compositor, used for resolution spoofing, upscaling, frame limiting and low-latency presentation.

**Gamemode** — A daemon/library that applies CPU governor, I/O priority and niceness changes for games.

**GXVK** — An experimental Vulkan implementation of 3Dfx Glide.

## H

**Hangover** — A project/Wine build that integrates CPU emulation as DLLs into a native ARM64 Wine, so only the Windows application (not the whole Wine userland) is translated.

**HoneyKrisp** — Mesa’s Vulkan driver for Apple AGX GPUs; also known as HK.

## I

**ICD (Installable Client Driver)** — A Vulkan driver discovered through a JSON manifest. Multiple ICDs can coexist; the Vulkan loader selects one.

**ISA (Instruction Set Architecture)** — The instruction set a CPU understands, such as x86-64 or ARM64.

## J

**JIT (Just-In-Time compilation)** — Compiling or translating code at runtime, usually with caching so repeated blocks are not translated again.

## K

**KGSL** — Qualcomm’s kernel GPU driver used on Android devices, exposed as `/dev/kgsl-3d0`. Turnip builds for Android target KGSL rather than mainline `drm/msm`.

## L

**LAA (Large Address Aware)** — A flag/patch allowing a 32-bit Windows process to use more than 2 GB of address space.

## M

**MangoHud** — An overlay that shows FPS, frame time, CPU/GPU load, temperatures, power and Wine sync status.

**Mesa** — The open-source graphics stack providing OpenGL and Vulkan drivers such as RADV, ANV, Turnip, Panfrost, Asahi and Zink.

**MoltenVK** — A Vulkan portability layer over Apple Metal; deprecated by Khronos but still used, especially on Intel Macs.

**muvm** — An Asahi Linux microVM that gives x86 code a 4 KiB page-size kernel, enabling Windows games on 16 KiB-page Apple Silicon hosts.

## N

**Native DLL** — In Wine terminology, a Windows DLL placed in the prefix and used instead of Wine’s built-in implementation.

**NTSync** — A Linux kernel mechanism implementing Windows NT synchronization primitives in kernel space. Automatically used by Wine 11 / Proton 11 when `/dev/ntsync` is available.

**NVK** — Mesa’s open-source Vulkan driver for NVIDIA GPUs.

## P

**PE (Portable Executable)** — The Windows executable and DLL file format.

**Phantom process killer** — An Android 12+ mechanism that kills excess/CPU-heavy child processes; a common cause of container crashes. Disabled via Developer options or adb settings.

**Prefix (Wine prefix)** — An isolated Windows-like environment containing `drive_c`, registry files and DLL configuration. Not a full Windows installation.

**Proton** — Valve’s gaming-focused compatibility distribution built from Wine, DXVK, VKD3D-Proton and gaming patches.

**Proton-GE** — A community-maintained Proton variant with additional patches and media codecs.

**proot / proot-distro** — User-space filesystem virtualization used by Termux and frontends to run Linux distributions without root.

## Q

**QEMU user mode** — Translating a single guest process of a different architecture using Linux `binfmt_misc`; much slower than a dedicated translator such as FEX or Box64.

## R

**RADV** — Mesa’s Vulkan driver for AMD GPUs.

**RootFS** — A Linux filesystem tree containing libraries and utilities for a compatibility environment; unrelated to rooting an Android device.

**Rosetta 2** — Apple’s x86-64 to ARM64 macOS translator. Cannot translate kernel extensions or virtualize x86-64 platforms.

## S

**shader cache** — Stored compiled shader/pipeline data that avoids recompilation on later runs. Invalidated by driver or translator changes.

**SurfaceFlinger** — Android’s system compositor. It may use a different GPU driver (system Qualcomm) than the app’s runtime driver (e.g. Turnip).

**sysmem / gmem** — Adreno render paths: direct system-memory rendering versus the on-chip tile buffer (GMEM). `TU_DEBUG` can force one for debugging.

## T

**thunk** — A forwarding layer that maps a guest library call (e.g. OpenGL) to the host’s native library, avoiding full emulation of that API.

**Trench / VKD3D-Proton cache** — Pipeline cache files (`vkd3d-proton.cache`) that store D3D12 shader translations.

**Turnip** — Mesa’s Vulkan driver for Adreno GPUs; packaged as `libvulkan_freedreno.so`.

## V

**VKD3D** — Wine’s Direct3D 12 implementation; upstream library for shader translation.

**VKD3D-Proton** — A Vulkan-based Direct3D 12 implementation for Proton/Wine; distinct from DXVK and from upstream VKD3D.

**VirGL** — A virtual GPU driver that translates guest OpenGL to host OpenGL through virtio-gpu.

**Venus** — A virtual GPU driver that serializes guest Vulkan commands to the host over virtio-gpu.

**Vulkan loader** — The library (`libvulkan.so.1`) that discovers ICDs and layers and dispatches Vulkan calls to the selected driver.

## W

**Wine** — A compatibility layer implementing Windows APIs on POSIX systems. It is not a CPU emulator and not a Windows kernel.

**WineD3D** — Wine’s Direct3D implementation using OpenGL (or an experimental Vulkan renderer for D3D10/11).

**Winetricks** — A helper script for installing DLLs, runtimes and configuring Wine prefixes.

**WoW64 (Windows 32-bit on Windows 64-bit)** — Windows’ subsystem for running 32-bit applications on 64-bit Windows. Wine implements its own WoW64; it is not a CPU translator.

## X

**XInput / DirectInput** — Windows game controller APIs. Wine maps them to Linux/Android input devices; missing mapping causes controller failures.

**XServer / Termux:X11** — X11 display servers used by Android frontends and Termux to present Linux/Wine windows.

## Z

**Zink** — A Mesa Gallium driver implementing OpenGL on top of Vulkan. Useful when no native OpenGL driver exists, but adds a translation layer.
