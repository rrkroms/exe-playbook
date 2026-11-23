# EXE Playbook

**Running Windows applications and games on Android, ARM Linux, x86 Linux, macOS, and ChromeOS.**

Most guides tell you which buttons to press in one app on one device. This book teaches the whole execution pipeline: what a Windows `.exe` expects, which layer supplies it, how x86 instructions become ARM64 instructions, how Direct3D becomes Vulkan, and how to diagnose the layer that broke when a game does not launch.

The reference lab device is a Lenovo Legion Tab Gen 2 (Snapdragon 8+ Gen 1, Adreno 730, 12 GB, Android 13), but every module is written to be portable. The same model applies to a Raspberry Pi 5, an Apple Silicon Mac running Asahi Linux, a Snapdragon X Elite laptop, an x86_64 Linux desktop, and a Chromebook.

---

## What this book covers

- **Universal non-Windows hosts** — Android ARM64, ARM64 Linux, x86_64 Linux, macOS, ChromeOS. Not a single-device recipe.
- **The full stack** — Wine and Proton, FEX and Box64/Box86, DXVK, VKD3D-Proton, D8VK, D7VK, WineD3D, dgVoodoo2, Zink, Turnip, Freedreno, Panfrost/PanVK, AGX, VirGL, and the display stack.
- **Legacy DirectX 7/8/9** — the dgVoodoo2/D7VK/D8VK/D3D8to9/WineD3D decision tree, including old fixed-function titles such as *IGI 1*.
- **DIY pipelines** — build your own Termux + proot + Wine + Box64/FEX + DXVK + Turnip environment on Android, and the equivalent on ARM and x86 Linux.
- **Frontends and tools** — Winlator, GameHub, DroidDeck, GameNative, Mobox, Termux, Lutris, Bottles, Heroic, and what each one actually does.
- **Performance and diagnosis** — bottleneck analysis, shader caches, esync/fsync, thermal limits, logging, and one-variable-at-a-time troubleshooting.
- **Reference material** — decision trees, compatibility matrices, a glossary, and a verified directory of official repositories.

## What this book does not cover

- Piracy, cracks, or bypassing DRM.
- Bypassing kernel-level anti-cheat or online-game protections.
- Cloud streaming as a substitute for local execution (covered only as a comparison).
- Native Windows-on-ARM execution on Windows itself.

---

## Book map

### Part I — Foundations

| # | Module | What it answers |
|---|---|---|
| 00 | [Preface](book/00-preface.md) | How to use the book, the seven-layer model, safety and legal notes |
| 01 | [Architecture](book/01-architecture.md) | Why x86 Windows software cannot run directly on ARM Android |
| 02 | [The Execution Pipeline](book/02-pipeline.md) | The complete path from `Game.exe` to a rendered frame |
| 03 | [Wine and Proton](book/03-wine-proton.md) | Windows API compatibility, prefixes, DLLs, Proton variants |
| 04 | [CPU Translation](book/04-cpu-translation.md) | FEX, Box64, Box86, dynamic binary translation, RootFS |

### Part II — Graphics

| # | Module | What it answers |
|---|---|---|
| 05 | [Graphics APIs and Translators](book/05-graphics-translators.md) | Direct3D, OpenGL, Vulkan, DXVK, VKD3D-Proton, Zink, WineD3D |
| 06 | [Legacy DirectX 7/8/9](book/06-legacy-directx.md) | dgVoodoo2, D7VK, D8VK, D3D8to9, DirectDraw, old-game recipes |
| 07 | [GPU Drivers](book/07-gpu-drivers.md) | Turnip, Qualcomm, Freedreno, Panfrost/PanVK, AGX, VirGL, runtime vs display drivers |

### Part III — Runtime and Platforms

| # | Module | What it answers |
|---|---|---|
| 08 | [Runtime, RootFS and Prefixes](book/08-runtime-prefixes.md) | Containers, Wine prefixes, dependencies, audio, input, filesystems |
| 09 | [Android Frontends](book/09-android-frontends.md) | Winlator, GameHub, DroidDeck, GameNative, Mobox, and how to choose |
| 10 | [Android DIY: The Termux Pipeline](book/10-android-diy-termux.md) | Build your own Windows execution environment without a frontend |
| 11 | [ARM Linux](book/11-linux-arm.md) | Asahi, Raspberry Pi 5, Snapdragon X Elite, ARM Chromebooks |
| 12 | [x86 Linux, macOS and ChromeOS](book/12-linux-x86-and-other-hosts.md) | Hosts that need less CPU translation but different graphics stacks |

### Part IV — Performance and Troubleshooting

| # | Module | What it answers |
|---|---|---|
| 13 | [Performance Engineering](book/13-performance.md) | CPU, GPU, memory, storage, shader and thermal bottlenecks |
| 14 | [Troubleshooting](book/14-troubleshooting.md) | Logs, diagnostic method, crash triage, known failure patterns |
| 15 | [Compatibility, Online Play, DRM and Anti-Cheat](book/15-compatibility-online-drm.md) | What will never work locally, and why |

### Part V — Reference and Lab

| # | Module | What it answers |
|---|---|---|
| 16 | [Decision Trees and Compatibility Matrix](book/16-decision-reference.md) | Pick a translator, driver and profile for a specific game |
| 17 | [Lab: Legion Tab Gen 2](book/17-lab-legion-tab.md) | A complete, testable build on the reference device |
| 18 | [Glossary](book/18-glossary.md) | Key terms used in the book, defined |
| 19 | [References](book/19-references.md) | Official repositories, documentation and licenses |

---

## How to read this book

Pick the path that matches your goal. You do not have to read linearly.

- **"I want to run one game on Android now."**
  Read [09](book/09-android-frontends.md), then [17](book/17-lab-legion-tab.md). If the game is old, read [06](book/06-legacy-directx.md). If it fails, read [14](book/14-troubleshooting.md).
- **"I want to understand the whole system."**
  Read 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08, then the platform module for your host.
- **"I have an ARM Linux laptop or SBC."**
  Read [11](book/11-linux-arm.md), then 03–08 and [13](book/13-performance.md).
- **"I have an x86 Linux desktop."**
  Read [12](book/12-linux-x86-and-other-hosts.md), then 03, 05, 08, 13 and 14.
- **"I want to build my own pipeline."**
  Read [10](book/10-android-diy-termux.md) for Android, [11](book/11-linux-arm.md) for ARM Linux, and [08](book/08-runtime-prefixes.md) for the runtime details.
- **"My game is from 1998–2006."**
  Read [06](book/06-legacy-directx.md) first. The generic modern pipeline is not the right starting point.
- **"I need to debug a crash."**
  Read [14](book/14-troubleshooting.md) and use the symptom map in [16](book/16-decision-reference.md).

## Conventions

- Technology names link to their **official repository or documentation** on first use. The full directory is in [19-references.md](book/19-references.md).
- Commands are shown for a POSIX shell unless marked `cmd` or `PowerShell`.
- Paths use `$HOME` and `$PREFIX` where the exact location depends on the host.
- Version numbers are a snapshot. The book favors **version ranges and selection rules** over one “magic” version, because drivers and translators change quickly.
- Warnings are marked **Warning**; they describe data loss, security, warranty or legal risk.
- This book uses the word *translation* for CPU and graphics conversion, and reserves *emulation* for full-system or hardware emulation.

## Reference hardware

The lab in [17](book/17-lab-legion-tab.md) uses:

| Component | Specification |
|---|---|
| Device | Lenovo Legion Tab Gen 2 |
| SoC | Qualcomm Snapdragon 8+ Gen 1 |
| CPU | Qualcomm Kryo, ARM64 |
| GPU | Adreno 730 |
| RAM | 12 GB |
| OS | Android 13 |

Every lesson generalizes: replace the SoC, GPU and driver with your own, and the layer model stays the same.

## Legal and ethical notes

- Own the software you run. This book does not provide, link to, or endorse pirated games, cracks, or DRM circumvention.
- Kernel-level anti-cheat and many online games will not work in translation environments. Do not attempt to bypass those protections.
- Rooting, unlocking the bootloader, and replacing GPU drivers can void warranties, trip verified-boot checks, and expose your data. Understand the risk before you proceed.
- Emulation and compatibility layers are legal in many jurisdictions, but the law varies. You are responsible for how you use them.
- Respect software licenses. Some tools in this space are open source; others are closed-source freeware. The [references](book/19-references.md) module records the license of each project.

## Contributing

Corrections, tested configurations, and hardware reports are welcome. A useful contribution includes:

1. The exact host, OS build, GPU, driver version, and app/frontend version.
2. The game or application, its store version, and its graphics API.
3. The full set of non-default settings.
4. Logs and a minimal reproduction.
5. Whether the result was tested or inferred.

Do not submit copyrighted game files, license keys, or links to pirated content.

## License

Prose and diagrams are licensed under **Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**. Code samples and scripts are licensed under the **MIT License**. See [LICENSE](LICENSE) for details.

## Acknowledgments

This book builds on the work of the Wine, Proton, FEX, Box64, Mesa, DXVK, VKD3D-Proton, Winlator and Termux communities. Their documentation and source code are the primary sources; the [references](book/19-references.md) module points to them.
