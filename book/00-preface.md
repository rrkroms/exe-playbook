# 00 — Preface: Why This Book Exists

A Windows game is not a self-contained object. `Game.exe` is a set of instructions and data that expects a specific CPU, a specific operating system, a specific graphics API, a specific audio stack, and a specific filesystem layout. When you move that file to an Android tablet or an ARM Linux laptop, almost none of those expectations are true.

Most guides solve this by giving you a recipe: install this app, pick this driver, tick these boxes. Recipes work until the game is old, the driver changes, the device is different, or something breaks. Then the user has no model to reason from.

This book is the model.

## The central idea

Running a Windows application on a non-Windows host is a **translation pipeline**, not a single emulator. Each layer answers one question:

| Layer | Question | Typical components |
|---|---|---|
| 1. Application | What does the program contain? | PE `.exe`/`.dll`, game data, launcher |
| 2. Windows API | How does it talk to Windows? | Wine, Proton |
| 3. CPU | What instruction set does it execute? | FEX, Box64, Box86, native ARM64 |
| 4. Graphics API | What rendering API does it use? | Direct3D 7–12, OpenGL, Vulkan |
| 5. Graphics translation | How does that API reach the GPU? | DXVK, VKD3D-Proton, D8VK, D7VK, dgVoodoo2, WineD3D, Zink |
| 6. GPU driver | How does the GPU receive commands? | Turnip, Qualcomm, Freedreno, Panfrost/PanVK, AGX, VirGL |
| 7. Host runtime | Where do files, libraries, audio, input and display live? | RootFS, container, prefix, Android/Linux userspace |

If a game fails, it fails at one of these layers. The job of the book is to help you identify which one, instead of changing five settings at once.

## The seven-layer model in one diagram

```text
+-------------------------------------------------------------+
|  7. HOST RUNTIME                                            |
|     Android / Linux / macOS / ChromeOS                      |
|     filesystem, audio, input, display, sandbox              |
+-------------------------------------------------------------+
|  6. GPU DRIVER                                              |
|     Turnip / Qualcomm / Freedreno / Panfrost / AGX / VirGL  |
+-------------------------------------------------------------+
|  5. GRAPHICS TRANSLATION                                    |
|     DXVK / VKD3D-Proton / D8VK / D7VK / WineD3D / dgVoodoo2 |
+-------------------------------------------------------------+
|  4. GRAPHICS API                                            |
|     Direct3D 7-12 / OpenGL / Vulkan                         |
+-------------------------------------------------------------+
|  3. CPU TRANSLATION                                         |
|     FEX / Box64 / Box86 / native ARM64                      |
+-------------------------------------------------------------+
|  2. WINDOWS API COMPATIBILITY                               |
|     Wine / Proton                                           |
+-------------------------------------------------------------+
|  1. APPLICATION                                             |
|     Game.exe + DLLs + assets                                |
+-------------------------------------------------------------+
```

The stack is conceptual. In a real system, CPU translation, Windows API calls, graphics submission and display presentation run concurrently, and several layers may be provided by the same process or the same package. But the separation is what makes diagnosis possible.

## Why “emulator” is usually the wrong word

A traditional emulator reproduces a complete machine: CPU, memory map, devices, firmware, and often an operating system. It can run software for a different architecture without the host OS providing any of the guest’s APIs.

The tools in this book are mostly **compatibility layers** and **translators**:

- [Wine](https://gitlab.winehq.org/wine/wine) implements Windows APIs on top of POSIX/Linux facilities. It does not emulate a CPU.
- [FEX](https://github.com/FEX-Emu/FEX) and [Box64](https://github.com/ptitSeb/box64) translate x86 instructions to ARM64. They do not emulate a complete PC.
- [DXVK](https://github.com/doitsujin/dxvk) translates Direct3D calls to Vulkan. It does not emulate a GPU.
- [Turnip](https://gitlab.freedesktop.org/mesa/mesa) is a real Vulkan driver for Adreno GPUs, not an emulator.

Some environments combine these pieces with a container or a lightweight virtual machine. That is why the user experience can feel like emulation even when no component is a full emulator.

## What changed since the first draft

The first version of this material was a single Markdown file with twelve modules, written around one tablet. This book expands it in eight directions:

1. **Legacy DirectX.** Direct3D 7, 8 and 9 need their own decision tree: WineD3D, D8VK, D7VK, D3D8to9, dgVoodoo2, and DirectDraw wrappers. Modern DXVK advice alone is not enough for a 1999 game.
2. **Universal hosts.** Android is one target. ARM64 Linux, x86_64 Linux, macOS and ChromeOS each have different drivers, containers and performance ceilings.
3. **The full component map.** FEX, Box64/Box86, GPU drivers, runtime and display drivers, Wine, Proton, and the way they are packaged.
4. **DIY pipelines.** A frontend is convenient, but you can build the same stack yourself in Termux or on a Linux host. The book shows how, and how to choose drivers for the host.
5. **Missing topics.** Anti-cheat, DRM, online play, audio, input, filesystems, thermal limits, benchmarking, licensing and security.
6. **Frontend and tool survey.** Winlator, GameHub, DroidDeck, GameNative, Mobox, Termux, Lutris, Bottles and Heroic are compared by architecture, not by marketing.
7. **Modular repository.** One module per file, with a landing page, glossary, decision reference and a verified link directory.
8. **Official links.** Every technology name points to its upstream repository or documentation, so you can verify the source instead of trusting a screenshot.

## How to use the book

The book has five parts:

- **Foundations (00–04):** the model and the two hardest translation problems, CPU and Windows APIs.
- **Graphics (05–07):** the APIs, the translators, legacy DirectX, and the drivers.
- **Runtime and Platforms (08–12):** containers, prefixes, Android frontends, the Termux pipeline, ARM Linux, and the easier x86/macOS/ChromeOS hosts.
- **Performance and Troubleshooting (13–15):** how to measure, how to isolate, and what will never work.
- **Reference and Lab (16–19):** decision trees, a complete lab, a glossary, and the link directory.

Each module ends with a short “What to remember” section and cross-links. The [decision reference](16-decision-reference.md) is designed to be used while configuring a game, not read cover to cover.

## Reference baseline and portability

The lab uses a Lenovo Legion Tab Gen 2:

- Snapdragon 8+ Gen 1, ARM64
- Adreno 730
- 12 GB RAM
- Android 13

When the book says “your GPU” or “your driver”, substitute your own. The questions stay the same:

- Which CPU instruction set does the game use?
- Which Windows APIs and graphics API does it call?
- Which translator and driver are available on this host?
- Which layer is failing?

## Version volatility

This field moves quickly. A driver that is stable today may regress next month; a frontend may bundle a different Wine build; a game patch may change the graphics API. The book therefore teaches:

- how to identify versions;
- how to read logs;
- how to test one variable at a time;
- how to roll back;
- when a pinned, known-good version is better than “latest”.

The [references](19-references.md) module records a snapshot date and points to the upstream release pages, which are always more current than any book.

## Safety, warranty and law

**Warning:** Rooting an Android device, unlocking a bootloader, replacing system GPU drivers, and disabling verified boot can void warranties, erase data, and weaken device security. Some banking, payment and DRM apps refuse to run on modified devices. Make a backup and understand the recovery path before you start.

**Warning:** Thermal limits are real. Long gaming sessions on a thin tablet or phone can degrade battery health and cause throttling. A stable 30 FPS is often a better experience than an unstable 60 FPS.

**Legal:** Own your software. This book does not support piracy, cracks, or anti-cheat circumvention. Compatibility layers are legal in many jurisdictions, but the details vary; you are responsible for your use.

## What to remember

- The pipeline has seven layers; failures belong to a layer.
- Wine is not a CPU emulator; FEX/Box64 are not Windows API layers; DXVK is not a GPU driver.
- The host matters: Android, ARM Linux, x86 Linux, macOS and ChromeOS expose different drivers and containers.
- Legacy DirectX needs a different decision tree from modern DirectX.
- Measure, isolate, and change one variable at a time.

Next: [01 — Architecture](01-architecture.md).
