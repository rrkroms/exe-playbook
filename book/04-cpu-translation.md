# 04 — CPU Translation

This module is about executing x86 and x86-64 machine code on a different CPU — typically ARM64. It is the layer that module 03 deliberately left out. Wine can load a PE file and implement `CreateFileW`, but if the guest binary contains x86 instructions and the host is ARM64, something must turn those instructions into ARM64 instructions. That "something" is a CPU translator.

Nothing in this module implements Windows APIs or Direct3D. Those are separate layers, described in modules 03 and 05.

## 4.1 The underlying mismatch

An **instruction set architecture (ISA)** is the set of machine instructions a CPU can decode. An ARM64 CPU cannot execute an x86-64 `MOV` or `CALL` directly, for the same reason a French sentence is not understood by reading it as English.

| Property | x86 / x86-64 | ARM64 (AArch64) | Consequence for translation |
|---|---|---|---|
| Register file | 16 general-purpose registers in 64-bit mode (`RAX`…`R15`) | 31 general-purpose registers (`X0`…`X30`) | Guest state must be mapped onto host registers and spilled to memory |
| Flags | A complex `EFLAGS`/`RFLAGS` register | NZCV condition flags | Flag semantics must be reproduced; a single x86 instruction may set flags ARM64 does not |
| Memory model | Relatively strong ordering | Weaker ordering | Translator inserts barriers to preserve guest behaviour |
| SIMD / FP | SSE/AVX registers and instructions | NEON/SVE | Vector code often needs per-instruction reworking, not a simple rename |
| Calling convention | Windows x64: `RCX`, `RDX`, `R8`, `R9` + shadow space; System V differs | AAPCS64: `X0`–`X7` | Calls between guest and host code need thunks |
| Instruction length | Variable (1–15 bytes) | Fixed 32-bit | Decoding strategies differ substantially |

Two ABI details are worth memorising because they explain real bugs:

- Windows x64 passes the first four integer arguments in `RCX`, `RDX`, `R8`, `R9` and reserves 32 bytes of shadow space. System V x86-64 uses `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`. A translator must present the *Windows* ABI to guest code even when the host is Linux.
- AAPCS64 passes the first eight integer arguments in `X0`–`X7` and requires 16-byte stack alignment.

## 4.2 DBT, JIT and the translation loop

**Dynamic binary translation (DBT)** reads guest instructions, translates them into host instructions, and executes the result. A **just-in-time (JIT)** compiler performs that translation at runtime rather than ahead of time, and caches the result.

A related term you will see in [Box64](https://github.com/ptitSeb/box64) documentation is **DynaRec** — dynamic recompiler. It is the same idea: translate blocks of guest code into native host code while the program runs.

An **interpreter** is the simpler alternative: it examines each guest instruction and performs its effect with host code, one instruction at a time. It is much slower but easier to make correct, which is why many translators keep an interpreter as a fallback.

```text
  Guest x86-64 instruction stream
             |
             v
   +---------------------+
   |  Decoder            |  read bytes, decode one instruction
   +----------+----------+
              v
   +---------------------+
   |  Translator / JIT   |  emit equivalent ARM64 instructions
   +----------+----------+
              v
   +---------------------+
   |  Code block cache   |  reuse translations for hot code
   +----------+----------+
              v
        Host ARM64 CPU
```

The loop has several important properties:

1. **Translation granularity.** Translators work on basic blocks, traces or larger regions, not whole programs. A *basic block* is a straight-line sequence ending in a branch.
2. **Caching.** Hot blocks are translated once and reused. This is why short benchmarks and repeated launches behave differently from long play sessions.
3. **Invalidation.** **Self-modifying code** — software that writes instructions and then executes them — can make a cached translation stale. The translator must detect writes to code pages and discard affected blocks.
4. **Intermediate representation (IR).** Some translators, notably FEX, first decode guest instructions into an internal IR, then optimise that IR and emit host code. This separates "understand x86" from "generate good ARM64" and makes optimisation passes possible.
5. **Signal and thread handling.** Guest signals, exceptions and threads must map onto host equivalents without losing guest state. This is where many difficult bugs live.

A **thunk** is a small bridge that adapts a call from guest code to a host function: it converts arguments and calling conventions. Thunks are not general instruction translation. Translators use them to let guest code call native host libraries directly, avoiding the cost of translating an entire library.

> **Emulation, translation and virtualization are different.** FEX and Box64 translate instructions in userspace. [QEMU](https://www.qemu.org/docs/master/user/main.html) system mode emulates a whole machine including devices. [KVM](https://www.linux-kvm.org/) is hardware virtualization and requires guest code that can run natively on the CPU. Wine implements APIs. None of these is interchangeable with the others.

## 4.3 Where a translator can sit

There are several integration patterns, and they have very different cost profiles.

| Pattern | What is translated | Example | Trade-off |
|---|---|---|---|
| Whole Wine process | Wine **and** the game | `box64 wine64 game.exe` | Simple with existing x86 builds; more code pays translation cost |
| Backend DLL | Only the Windows game | ARM64 Wine + FEX's `libarm64ecfex.dll` via [Hangover](https://github.com/AndreRH/hangover) | Efficient; requires ARM64EC/WoW64-aware Wine |
| MicroVM + translator | Guest runs in a 4K-page VM; FEX or Box64 inside | [muvm](https://github.com/AsahiLinux/muvm) with `--emu=fex\|box` | Solves page-size and environment issues; adds overhead |
| OS-level translator | All x86-64 macOS binaries | [Rosetta 2](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment) | Transparent to apps; does not help with Windows PE or Windows APIs |
| User-mode syscall emulation | CPU plus syscall interception | QEMU user mode under [proot-distro](https://github.com/termux/proot-distro) | Broad compatibility, low speed |

The "backend DLL" pattern is the one most worth understanding. The game is still x86-64, but Wine is native ARM64. When the game calls a Windows API, WoW64/ARM64EC thunks route the call into native ARM64 Wine code. When the game executes its own logic, the backend DLL's translator handles it. The two mechanisms cooperate but remain distinct: WoW64 adapts *calls*, FEX executes *instructions*.

## 4.4 FEX

[FEX](https://github.com/FEX-Emu/FEX) is an MIT-licensed userspace x86/x86-64 dynamic binary translator. Its design has four parts that matter here:

- A custom IR and optimising JIT. Guest instructions decode into FEX's IR, which is optimised before ARM64 code is emitted.
- **Syscall translation**, so guest programs' Linux system calls reach the host kernel.
- A **RootFS** mechanism.
- **Thunks** for selected libraries — GL, Vulkan and SDL — so guest calls can go directly to native host libraries.

FEX is sponsored by Valve and bundled with Proton. It requires ARMv8.0-a with floating point and CRC32 support. The monthly release FEX-2611.1 appeared on 11 November 2026 ([FEX releases](https://github.com/FEX-Emu/FEX)). Known gaps in that release include debug registers and `ptrace`; `RDPID`/`RDTSC` handling was fixed in 2611 but depends on kernel support ([host summary](https://github.com/FEX-Emu/FEX)).

### RootFS

A **RootFS** is a directory tree containing the libraries and loader a guest binary expects — for example an x86-64 `ld-linux.so` and glibc. It is not a virtual machine disk image. On AArch64 a RootFS is mandatory: without it, x86-64 guest binaries cannot find their loader.

```text
$HOME/.fex-emu/RootFS/
    x86_64/
       lib64/ld-linux-x86-64.so.2
       lib/x86_64-linux-gnu/libc.so.6
       ...
$HOME/.fex-emu/Config.json     <-- FEX configuration
```

The [FEX RootFS fetcher](https://github.com/FEX-Emu/FEX) downloads a suitable tree, and `FEX_ROOTFS` selects one. Configuration lives in `$HOME/.fex-emu/Config.json`; most settings also have an `FEX_`-prefixed environment override ([FEX configuration docs](https://wiki.fex-emu.com/index.php/Development:Configuration)). One detail worth flagging: `FEX_LOG_LEVEL` is not part of the current configuration schema — use the documented `Logging.*` keys such as `Logging.ProfileStats` and `Logging.OutputLog` ([configuration docs](https://wiki.fex-emu.com/index.php/Development:Configuration)).

### FEX as a Wine backend

FEX can be built as backend DLLs for ARM64 Wine:

| DLL | Role | Typical selector |
|---|---|---|
| `libarm64ecfex.dll` | Executes 64-bit x86 code in an ARM64EC process | `HODLL64=libarm64ecfex.dll` |
| `libwow64fex.dll` | Handles the 32-bit side through WoW64 | `HODLL=libwow64fex.dll` |

This is the Hangover integration described in module 03, and it is the recommended ARM64 approach when your Wine build supports it ([Hangover](https://github.com/AndreRH/hangover)).

## 4.5 Box64

[Box64](https://github.com/ptitSeb/box64) is an MIT-licensed x86-64 dynamic binary translator aimed primarily at ARM64, with DynaRec back ends for ARM64, RISC-V and LoongArch. Its defining characteristics are:

- **DynaRec** is the default fast path. The documentation reports roughly **5–10× faster** execution than the interpreter for typical workloads ([Box64 usage docs](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md)).
- It **wraps native host libraries**. Instead of translating every instruction of `libGL` or `libvulkan`, Box64 can redirect guest calls to the host's native libraries through thunks. This is a large part of why it performs well in graphics-heavy workloads.
- It supports Wine64 and Proton, usually by running the whole Wine process as a guest ([Box64 Wine notes](https://github.com/ptitSeb/box64/blob/main/docs/WINE.md)).

The latest version confirmed in the research briefs is **v0.4.2 (April 2026)**. A v0.4.4 reference exists in the materials but is **unverified**; do not quote it as a release without checking the upstream release page.

```text
  x86-64 game / Wine process
        |
        v
  +---------------------------+
  | Box64 DynaRec             |  translate hot guest blocks
  +------------+--------------+
               |
      +--------+---------+
      |                  |
      v                  v
 native host libs    translated guest code
 (libGL, Vulkan,     (game logic, engine)
  SDL, libc wrappers)
```

Useful settings, drawn from the [usage documentation](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md):

| Variable | Effect |
|---|---|
| `BOX64_DYNAREC=1` | Enable the dynamic recompiler |
| `BOX64_DYNAREC_BIGBLOCK` | Aggressiveness of block formation |
| `BOX64_DYNAREC_STRONGMEM` | Stronger memory ordering emulation for fragile code |
| `BOX64_DYNACACHE` | Enables the DynaCache translation cache; size/reservation details should be checked in the current Box64 documentation |
| `BOX64_SKIPCPU` / `BOX64_MAXCPU` | Restrict visible CPUs, commonly used on big.LITTLE designs |
| `BOX64_LOG`, `BOX64_DYNAREC_LOG`, `BOX64_TRACE_FILE` | Logging and tracing |

Performance measurement is built in: `BOX64_DYNAREC_TRACE` reports translation activity, and `BOX64_DYNAREC_PERFMAP=1` generates a map that can be combined with `perf` ([usage docs](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md)).

**32-bit limitation:** Box64 handles 64-bit x86-64 code. A 32-bit game needs [Box86](#46-box86-and-box32) or a 32-bit path through FEX/WoW64. Running Wine's WoW64 under Box64 is described as experimental in the research notes.

## 4.6 Box86 and Box32

[Box86](https://github.com/ptitSeb/box86) is the 32-bit companion to Box64: it translates 32-bit x86 instructions. It is considered legacy in this context and typically requires an `armhf` (32-bit ARM) chroot with multilib support.

**Box32** is an experimental mode that runs 32-bit code inside a 64-bit Box64 process; it is enabled at build time with `-DBOX32=ON`, and the Wine WoW64 DLL variant with `-DWOW64=ON` ([Termux pipeline notes](https://raw.githubusercontent.com/ptitSeb/box64/main/docs/WINE.md)). Treat it as an option of last resort for 32-bit-only titles when a FEX/WoW64 path is unavailable.

## 4.7 QEMU user mode

[QEMU](https://www.qemu.org/docs/master/user/main.html) has two distinct modes:

| Mode | What it emulates | Use here |
|---|---|---|
| **User mode** | CPU instructions plus Linux syscall translation; guest processes appear as host processes | Cross-architecture Linux guests, e.g. `proot-distro install debian --architecture amd64` on ARM64 |
| **System mode** | A complete machine: CPU, memory, devices, boot firmware | General virtualization/emulation, rarely used for games in this book |

In a Termux setup, QEMU user mode is what allows a proot distribution of a different architecture to run ([proot-distro](https://github.com/termux/proot-distro)). It is broad but slow, and proot's `ptrace`-based interception adds further filesystem overhead. It is a fallback, not a performance choice for games.

## 4.8 Rosetta 2

[Rosetta 2](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment) is Apple's DBT system for running x86-64 **macOS** applications on Apple Silicon. Conceptually it belongs in this module, but its scope is specific:

- It translates CPU instructions for macOS binaries; it does not implement Windows APIs, so it cannot replace Wine.
- Running an x86-64 Wine build under Rosetta is possible, but the Windows API layer still comes from Wine and the graphics path is separate.
- It does not support AVX-512, and it cannot virtualize x86-64 ([Apple documentation](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment)).
- Apple documents availability through macOS 27, with a reduced subset afterwards ([host summary](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment)).

Keep three translations distinct on macOS: Rosetta 2 translates CPU instructions, Wine translates Windows APIs, and D3DMetal or DXMT translates Direct3D to Metal (module 05).

## 4.9 muvm and the page-size problem

### Why page size matters

A **page** is the unit of memory that the OS maps and protects. Linux desktops commonly use 4096-byte (4K) pages. Some ARM systems use 16K or 64K pages. Guest software that assumes 4K — including Wine's ARM64EC path and many prebuilt x86 binaries — can fail on a 16K host.

| Host | Page size | Consequence |
|---|---|---|
| x86-64 Linux | 4K | No issue |
| Most ARM64 Linux and Android | 4K | No issue |
| Asahi Linux (Apple Silicon) | 16K | Wine ARM64EC unsupported; needs a 4K environment |
| Some Snapdragon configurations | 16K | Same class of problem |
| Android 15+ devices with 16KB pages | 16K | NDK r28+ compiles 16 KB-aligned by default; older NDKs need `-Wl,-z,max-page-size=16384`. Prebuilt `.so` files must be rebuilt or realigned; Play enforcement starts 2027-02-01 ([Android page-size docs](https://developer.android.com/guide/practices/page-sizes)) |

### What muvm does — and does not do

[muvm](https://github.com/AsahiLinux/muvm) is a **microVM** built on libkrun. It gives x86 binaries a 4K-page environment, shares the host root filesystem with the guest, and lets you choose the emulator with `--emu=fex|box`. For graphics it offers DRM Native Context, which reached upstream in virglrenderer 1.x and Mesa 25.2+ ([host summary](https://github.com/AsahiLinux/muvm)).

> **muvm is not a CPU translator.** It supplies the 4K-page execution environment. FEX or Box64 inside it performs the actual instruction translation. The Asahi penalty of roughly 20% comes from the microVM boundary, not from FEX.

The full Asahi chain looks like this:

```text
Steam wrapper
   -> muvm (4K guest, shared rootfs)
   -> binfmt handler
   -> FEX
   -> x86 rootfs + Mesa overlay
   -> Wine / Proton
   -> DXVK / VKD3D-Proton
   -> AGX GPU driver
```

Each arrow is a separate layer with its own failure modes. A crash in this stack is only diagnosable if you know which arrow you are looking at.

## 4.10 32-bit support at a glance

32-bit Windows games remain common, and this is where the layers get confusing.

| Need | Works on x86-64 host | Works on ARM64 host |
|---|---|---|
| 32-bit game, native x86 host | Wine's WoW64 runs it natively | — |
| 32-bit game, ARM64 host, backend DLL path | — | `libwow64fex.dll` (FEX) or experimental Box64 WoW64 DLL |
| 32-bit game, ARM64 host, whole-process path | — | Box86 in an armhf chroot, or experimental Box32 |
| 32-bit game under QEMU user mode | possible but slow | possible but slow |

Remember: WoW64 solves the *32-bit API* problem. It never makes a 32-bit x86 instruction executable on ARM64 by itself.

## 4.11 Choosing a translator

There is no universal winner between FEX and Box64 — the right answer depends on the workload and the host ([architecture overview](01-architecture.md)).

| Host / goal | Preferred path | Why |
|---|---|---|
| x86-64 Linux | None | CPU executes guest code natively |
| ARM64 Linux, 4K pages, modern Wine | FEX backend DLL (Hangover), or Box64 whole-process | Backend path translates only game code; Box64 is simpler with x86 builds |
| Raspberry Pi 4/5 | Box64 with Raspberry Pi flags; Box86 for 32-bit | Mature support on these boards ([host summary](https://github.com/ptitSeb/box64)) |
| Asahi Linux (16K pages) | FEX inside muvm; Box64 as fallback | 4K guest environment required |
| Android ARM64 frontend | Usually bundled: FEX/Hangover preferred, Box64 fallback | Frontend controls the RootFS, translator and driver ([FEX](https://github.com/FEX-Emu/FEX), [Box64](https://github.com/ptitSeb/box64)) |
| Apple Silicon macOS | Rosetta 2 for x86-64 Wine, or ARM64EC Wine | Rosetta handles macOS x86-64 binaries, not Windows APIs |
| Cross-architecture Linux guest | QEMU user mode | Broad compatibility, accept the speed cost |

When comparing two translators, keep everything else identical — same game, same Wine build, same driver, same prefix — and measure. A translator that wins in one engine can lose in another.

## 4.12 Performance characteristics

- **DynaRec dominates interpretation.** Box64's own documentation quotes 5–10× improvement from DynaRec ([usage docs](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md)). An interpreter is a correctness fallback, not a performance option.
- **CPU-bound workloads suffer most.** A classic symptom is one core pinned at 100% with low GPU usage. That points at instruction translation or Wine thread synchronization, not at graphics.
- **Thunks reduce work.** Every call redirected to a native host library is a large block of code that never needs translation. This is why library-heavy games can perform better than their raw instruction count suggests.
- **big.LITTLE matters.** On heterogeneous ARM SoCs, scheduler choices can move guest threads to efficiency cores. `BOX64_SKIPCPU` and `BOX64_MAXCPU` restrict which CPUs are visible; CPU governor and `taskset` are common companion tools ([performance notes](https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md)).
- **Translation cache costs memory.** Box64's DynaCache and FEX's code cache both consume resources; the exact reservation depends on the version and is not a fixed resident-RAM figure. Tight memory budgets on Android make this worth measuring rather than assuming.
- **Measure the right thing.** Use `BOX64_DYNAREC_TRACE`, `BOX64_DYNAREC_PERFMAP=1` with `perf`, and FEX's `Logging.ProfileStats`. Frame generation and upscaling do not fix a CPU-bound translation bottleneck.
- **First runs are not representative.** Shader compilation, translator warm-up and cache population all distort early measurements.

## 4.13 What to remember

- CPU translation executes guest instructions on a different ISA; it is not API compatibility and not graphics translation.
- DBT/JIT decodes, translates, caches and invalidates blocks at runtime; thunks are small calling-convention bridges, not general translation.
- FEX uses a custom IR, an optimising JIT, syscall translation, a RootFS and library thunks, and can serve as ARM64EC/WoW64 backend DLLs.
- Box64 uses DynaRec and wraps host libraries; 32-bit code needs Box86/Box32 or a FEX/WoW64 path.
- WoW64 solves 32-bit API calls; it never makes x86 instructions run on ARM64.
- muvm provides a 4K-page environment for 16K hosts; it is not itself a translator.
- Rosetta 2 translates x86-64 macOS binaries, not Windows programs.
- Choose and tune a translator by measuring the specific game on the specific host.

Next: [05 — Graphics Translators](05-graphics-translators.md).

## References

- FEX: https://github.com/FEX-Emu/FEX
- FEX configuration documentation: https://wiki.fex-emu.com/index.php/Development:Configuration
- Box64: https://github.com/ptitSeb/box64
- Box64 usage documentation: https://github.com/ptitSeb/box64/blob/main/docs/USAGE.md
- Box64 and Wine: https://github.com/ptitSeb/box64/blob/main/docs/WINE.md
- Box86: https://github.com/ptitSeb/box86
- Hangover: https://github.com/AndreRH/hangover
- muvm: https://github.com/AsahiLinux/muvm
- QEMU user-mode documentation: https://www.qemu.org/docs/master/user/main.html
- proot-distro: https://github.com/termux/proot-distro
- Apple Rosetta 2 documentation: https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment
- Android page-size documentation: https://developer.android.com/guide/practices/page-sizes
- Wine: https://gitlab.winehq.org/wine/wine
- Proton: https://github.com/ValveSoftware/Proton
- D7VK (legacy Direct3D context): https://github.com/WinterSnowfall/d7vk
- Linux KVM: https://www.linux-kvm.org/
