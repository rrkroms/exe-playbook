# 01 — Architecture: Why a Windows Program Cannot Run Directly

A program is a contract between software and a machine. The contract has several independent parts, and moving the program to a new host breaks each part in a different way.

This module establishes the vocabulary used by the rest of the book. It is deliberately host-neutral: the same reasoning applies to Android, ARM Linux, x86 Linux, macOS and ChromeOS.

## 1.1 The five contracts of a program

Every non-trivial application depends on five contracts:

| Contract | What it specifies | Example |
|---|---|---|
| Instruction set | Which machine instructions the CPU executes | x86-64, ARM64 |
| ABI | How functions are called, how registers and the stack are used, how binaries are loaded | System V ABI, Windows x64 ABI |
| Operating system | Which system calls, files, processes, registry and services exist | Windows NT, Linux, Android |
| Libraries | Which shared libraries provide APIs | `kernel32.dll`, `libc.so`, `libvulkan.so` |
| Devices and APIs | How graphics, audio, input and storage are accessed | Direct3D, Vulkan, XAudio2, XInput |

A compatibility environment can preserve some contracts and translate others. It rarely preserves all of them perfectly, which is why individual games fail in individual ways.

## 1.2 CPU architectures

A CPU understands a specific **instruction set architecture (ISA)**. Instructions are binary encodings; an ARM64 CPU cannot decode an x86-64 `MOV` or `CALL` the way an x86 CPU does.

| Architecture | Typical devices | Notes |
|---|---|---|
| x86 (IA-32) | Older PCs, 32-bit Windows software | 32-bit; still common in games from the 1990s and 2000s |
| x86-64 (AMD64) | Modern PCs, most Windows games | 64-bit extension of x86 |
| ARM32 (AArch32) | Older Android devices, some embedded systems | 32-bit ARM |
| ARM64 (AArch64) | Modern Android, Apple Silicon, Raspberry Pi, Snapdragon X | 64-bit ARM |
| ARM64EC | Windows on ARM | Microsoft’s ABI that allows ARM64 and x86-64 code to coexist in one process |
| RISC-V | Emerging devices | Not a mainstream target for this book |

The reference tablet uses ARM64. Most Windows games are compiled for x86 or x86-64. That mismatch is the first problem.

### Why “just recompile it” is not a solution

You usually do not have the game’s source code, build system, middleware, or legal right to rebuild it. Even when source is available, the game may depend on Windows-only libraries, Direct3D, or third-party middleware. Translation lets the original binary run.

## 1.3 Dynamic binary translation

**Dynamic binary translation (DBT)** reads guest instructions, translates them into host instructions, and executes the result. A **just-in-time (JIT)** compiler translates blocks of instructions at runtime and caches them.

```text
Guest x86-64 code
       |
       v
+-------------------+
|  Decoder          |  read instruction bytes
+-------------------+
       |
       v
+-------------------+
|  Translator / JIT |  emit ARM64 instructions
+-------------------+
       |
       v
+-------------------+
|  Block cache      |  reuse translated blocks
+-------------------+
       |
       v
Host ARM64 CPU
```

DBT is not a simple one-to-one mapping. The translator must preserve:

- **Flags and condition codes.** x86 has a flags register; ARM64 uses condition flags and different semantics.
- **Memory ordering.** x86 has a relatively strong memory model; ARM64 is weaker. The translator inserts barriers where required.
- **Self-modifying code.** Some software writes and then executes code; the translator must detect and invalidate stale translations.
- **Calling conventions.** Windows x64, System V x86-64 and ARM64 AAPCS pass arguments differently.
- **Thread-local state and signal handling.** Guest threads must map onto host threads.
- **Floating point and SIMD.** x86 SSE/AVX and ARM NEON/SVE are not identical.

This is why translation overhead is workload-dependent and why “FEX vs Box64” has no universal winner.

## 1.4 Operating system contracts

Windows software expects Windows, not merely a CPU that can execute its instructions.

### Windows

- PE/COFF executable format (`.exe`, `.dll`, `.sys`).
- Win32 and NT APIs: `kernel32`, `user32`, `gdi32`, `ntdll`, `advapi32`, and hundreds of others.
- The Windows registry.
- Drive letters and Win32 path rules.
- Windows process, thread, handle and security models.
- Services, COM, RPC, and the Windows event model.
- DirectX and the Windows graphics stack.

### Linux

- ELF executable format.
- POSIX/Linux system calls.
- `/proc`, `/sys`, `/dev`, shared libraries, `ld.so`.
- X11/Wayland display servers, PulseAudio/PipeWire audio, evdev input.
- A different filesystem permission model.

### Android

Android uses the Linux kernel but a different userspace:

- Bionic libc instead of glibc.
- A sandboxed app model with scoped storage.
- SurfaceFlinger for display composition.
- AudioFlinger/AAudio for audio.
- A permission system and SELinux policies.
- No traditional X11 or Wayland desktop by default.
- A linker namespace model that restricts which native libraries an app can load.

A compatibility layer such as [Wine](https://gitlab.winehq.org/wine/wine) implements Windows APIs on top of the host’s POSIX facilities. It is not a Windows kernel. It translates API behavior, not every Windows internal.

### PE and ELF

| Format | Host | Purpose |
|---|---|---|
| PE/COFF | Windows | `.exe`, `.dll`, `.sys` |
| ELF | Linux, Android | executables, shared objects |
| Mach-O | macOS | executables, dylibs |

Wine loads PE files and provides a Windows-like environment. The host loader loads Wine itself as an ELF/Mach-O binary. The CPU translator sits between the guest PE code and the host CPU.

## 1.5 Graphics contracts

Graphics is the second major mismatch.

### Windows graphics APIs

| API | Era | Typical use |
|---|---|---|
| Direct3D 7 | 1999 | Early 3D games, fixed-function pipeline |
| Direct3D 8 | 2000 | Programmable shaders introduced |
| Direct3D 9 | 2002 | Long-lived; many games through the 2000s |
| Direct3D 10 | 2006 | Vista-era; shader model 4 |
| Direct3D 11 | 2009 | Modern baseline for many PC games |
| Direct3D 12 | 2015 | Low-level API; newer games |
| OpenGL | 1992– | Cross-platform; older Windows games and tools |
| Vulkan | 2016 | Low-overhead cross-platform API |

### Android and Linux graphics APIs

- **Vulkan** — low-overhead, explicit API; the main target for modern translation layers.
- **OpenGL** — desktop Linux exposes full OpenGL; Android primarily exposes OpenGL ES.
- **OpenGL ES** — mobile variant of OpenGL; not identical to desktop OpenGL.

A GPU driver implements one or more of these APIs. The driver is not the same thing as the API, and the API is not the same thing as the translator.

### The translation chain

```text
Game calls Direct3D 9
        |
        v
DXVK translates D3D9 to Vulkan
        |
        v
Vulkan loader
        |
        v
GPU driver (Turnip, Qualcomm, Freedreno, ...)
        |
        v
GPU hardware
        |
        v
Display compositor
```

If the game uses Direct3D 12, the translator is VKD3D-Proton instead of DXVK. If it uses OpenGL, the path may be native OpenGL, Zink (OpenGL on Vulkan), or WineD3D’s OpenGL backend. If it uses Direct3D 7 or 8, additional translators or wrappers are involved.

## 1.6 Emulation, compatibility, translation, virtualization

These words are often used interchangeably. They are not the same.

| Approach | What it does | Example |
|---|---|---|
| Full-system emulation | Emulates a complete machine, including CPU and devices | QEMU system mode, game console emulators |
| CPU translation | Translates guest instructions to host instructions | FEX, Box64, Box86, Rosetta 2 |
| API compatibility | Implements one OS’s APIs on another OS | Wine, Proton |
| API translation | Translates one graphics/audio API to another | DXVK, VKD3D-Proton, WineD3D |
| Virtualization | Runs a guest OS in a hardware-assisted VM | KVM, Hyper-V, Android virtual machines |
| Container | Shares the host kernel but isolates userspace | proot, chroot, LXC, Docker |
| Streaming | Runs the game elsewhere and sends video | Moonlight, Steam Link |

A typical Android Windows-gaming setup combines **API compatibility** (Wine), **CPU translation** (FEX or Box64), **graphics translation** (DXVK/VKD3D), a **container** (proot/chroot), and a **frontend** (Winlator, DroidDeck, GameHub).

## 1.7 The universal pipeline

The same logical pipeline appears on every host:

```text
+---------------------+
| Windows application |
+----------+----------+
           |
           v
+---------------------+
| PE loader + Wine    |
+----------+----------+
           |
     +-----+------+
     |            |
     v            v
+---------+  +------------------+
| CPU     |  | Windows APIs     |
|translation| | (Wine libraries) |
+----+----+  +--------+---------+
     |                |
     v                v
+---------------------+
| Game logic          |
+----------+----------+
           |
           v
+---------------------+
| Graphics API        |
+----------+----------+
           |
     +-----+------+
     |            |
     v            v
+---------+  +------------------+
| DXVK /  |  | VKD3D-Proton /   |
| WineD3D |  | D8VK / D7VK      |
+----+----+  +--------+---------+
     |                |
     +-------+--------+
             |
             v
+---------------------+
| Vulkan / OpenGL     |
+----------+----------+
           |
           v
+---------------------+
| GPU driver          |
+----------+----------+
           |
           v
+---------------------+
| GPU + display       |
+---------------------+
```

The host determines which boxes exist:

- On **x86 Linux**, the CPU translation box is absent (native x86-64).
- On **ARM Linux**, the CPU translation box is FEX or Box64.
- On **Android**, the runtime box includes the Android app sandbox and display stack.
- On **macOS**, the graphics stack may use MoltenVK or Apple’s Game Porting Toolkit instead of a native Vulkan driver.

## 1.8 ABI details that explain real bugs

You do not need to memorize ABIs, but a few details explain common failures.

### Windows x64 vs System V x86-64

- Windows x64 passes the first four integer arguments in `RCX`, `RDX`, `R8`, `R9`; System V uses `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`.
- Windows x64 has a 32-byte shadow space; System V does not.
- Different registers are caller-saved vs callee-saved.
- Exception handling and unwind metadata differ.

A translator must emulate the Windows ABI for guest code, even when the host is Linux.

### ARM64

- AAPCS64 passes the first eight integer arguments in `X0`–`X7`.
- The stack must be 16-byte aligned.
- Different SIMD/FP register conventions.
- Memory ordering is weaker than x86; barriers matter.

### Why this matters to users

These differences surface as:

- crashes in specific games but not others;
- missing audio or input;
- graphics corruption;
- “works on one frontend, fails on another”;
- performance differences between FEX and Box64 for the same game.

The frontend’s build flags, Wine version, translator version and driver all change the contract.

## 1.9 Filesystems and paths

Windows software expects drive letters (`C:\`), backslashes, case-insensitive paths, and a registry. Linux and Android use `/`, case-sensitive paths, and different permission rules.

Wine maps a prefix to a directory and exposes drive letters through `dosdevices`:

```text
$WINEPREFIX/
├── drive_c/
│   ├── Program Files/
│   ├── Program Files (x86)/
│   ├── users/
│   └── windows/
├── dosdevices/
│   ├── c: -> ../drive_c
│   └── z: -> /
├── system.reg
├── user.reg
└── userdef.reg
```

On Android, scoped storage and app sandboxes complicate access to shared folders. On Linux, permissions and mount options matter. On macOS, case-insensitivity depends on the filesystem.

## 1.10 What to remember

- A program depends on five contracts: ISA, ABI, OS, libraries, devices.
- x86/x86-64 code on ARM64 requires CPU translation; this is not the same as Windows API compatibility.
- Wine implements Windows APIs; it does not translate x86 to ARM64.
- Direct3D must be translated to Vulkan or OpenGL; the GPU driver then talks to the hardware.
- Android is Linux-kernel-based but not a normal Linux desktop; its sandbox, linker and display stack change the rules.
- The same seven-layer pipeline applies to every host, but the components and their availability differ.

Next: [02 — The Execution Pipeline](02-pipeline.md).
