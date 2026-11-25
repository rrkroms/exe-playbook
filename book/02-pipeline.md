# 02 — The Execution Pipeline

This module follows a Windows game from the moment you launch it to the moment a frame appears. It then shows how the same pipeline changes across Android, ARM Linux, x86 Linux, macOS and ChromeOS.

Read this module twice: once for the overview, once for the failure points.

## 2.1 The launch sequence

A simplified launch sequence looks like this:

```text
User launches Game.exe
        |
        v
Frontend or shell resolves the command
        |
        v
Container / RootFS is entered
        |
        v
Wine loads the PE executable
        |
        v
Wine initializes the prefix and wineserver
        |
        v
CPU translator begins executing guest code
        |
        v
Game initializes Direct3D / OpenGL / Vulkan
        |
        v
Graphics translator creates a Vulkan device
        |
        v
GPU driver compiles pipelines and allocates memory
        |
        v
Game loop: input -> simulation -> rendering -> present
        |
        v
Host compositor displays the frame
```

Each arrow is a possible failure point. The rest of this module explains what happens at each stage.

## 2.2 Stage 1: the frontend or shell

On Android, a frontend such as Winlator, DroidDeck, GameHub or GameNative usually launches the game. On Linux, a shell script, Lutris, Bottles, Heroic or Steam launches it. On macOS, Whisky or CrossOver launches it.

The frontend is responsible for:

- selecting the container or prefix;
- selecting the Wine/Proton build;
- selecting the CPU translator;
- selecting the graphics translator and driver;
- setting environment variables;
- mounting storage;
- starting the display server or X11 compatibility layer;
- capturing logs.

The frontend is not the compatibility layer. A bug in the frontend can look like a bug in Wine, and vice versa. When debugging, record the frontend version and the exact components it bundled.

## 2.3 Stage 2: the container or RootFS

A Windows environment needs a Linux userspace with libraries, utilities and a filesystem. On Android this is often a **RootFS** inside a proot or chroot container. On Linux it may be the host itself, a Steam Runtime container, a Distrobox container, or a Flatpak sandbox.

The container provides:

- a Linux loader and libc;
- graphics and audio libraries;
- Vulkan and OpenGL loaders;
- Wine’s dependencies;
- a writable filesystem for the prefix;
- device nodes and shared memory;
- environment variables.

If the container is missing a library, Wine may start but the game may fail later. If the container’s graphics libraries do not match the host driver, Vulkan device creation may fail.

## 2.4 Stage 3: Wine loads the PE

Wine’s loader reads the PE headers, maps sections, resolves imports, and starts the guest entry point. It also initializes:

- the prefix (creating it on first run);
- the registry;
- the `wineserver` process;
- DLL search paths and overrides;
- the Windows version reported to the application;
- environment variables such as `WINEPREFIX`, `WINEARCH`, `WINEDLLOVERRIDES`.

At this stage, common failures include:

- a missing 32-bit or 64-bit Wine component;
- a corrupt or incompatible prefix;
- a missing DLL;
- an unsupported executable format (a 16-bit build on a host/version that lacks support, .NET without a runtime, or a kernel driver);
- a path or permission problem.

## 2.5 Stage 4: CPU translation begins

When the guest executes x86/x86-64 code, the translator takes over. It decodes guest instructions, translates them into host instructions, caches the result, and handles guest signals, threads and system calls.

The translator must cooperate with Wine:

- Wine’s Windows API calls must reach the host kernel;
- guest threads must map to host threads;
- guest memory must be accessible to both guest and host code;
- signals and exceptions must be delivered correctly;
- the guest stack and calling convention must be preserved.

On ARM Linux, FEX or Box64 may run the entire Wine process. On Android, the frontend may bundle a specific translator build and RootFS. On x86 Linux, this stage is absent.

Common failures at this stage:

- an unsupported instruction or CPU feature;
- a translator bug triggered by a specific game engine;
- a 32-bit game in a 64-bit-only environment;
- a self-modifying or anti-debug code path;
- a crash that only appears under translation.

## 2.6 Stage 5: Windows APIs and DLLs

The game calls Windows APIs: `CreateFileW`, `ReadFile`, `Direct3DCreate9`, `XInputGetState`, `CoCreateInstance`, and many others. Wine implements these on top of the host.

Two kinds of DLL matter:

1. **Wine built-in DLLs** implement Windows APIs.
2. **Native Windows DLLs** may be installed into the prefix for compatibility (for example, a specific `d3dx9_43.dll`, `msvcp140.dll`, or `xaudio2_7.dll`).

DLL overrides decide which implementation wins. A wrong override can break a game that would otherwise work.

At this stage, common failures include:

- missing Visual C++ Redistributable;
- missing .NET Framework or Mono;
- missing DirectX 9.0c runtime components;
- missing XAudio2, XInput, Media Foundation or PhysX;
- a DLL override that forces the wrong implementation;
- a game that requires a Windows component Wine does not implement.

## 2.7 Stage 6: the graphics API and translator

The game creates a graphics device. The API determines the translator:

| Game API | Translator candidates |
|---|---|
| Direct3D 7 | D7VK, dgVoodoo2, WineD3D |
| Direct3D 8 | DXVK (`d3d8.dll` + `d3d9.dll`; D8VK standalone obsolete), dgVoodoo2, D3D8to9 + DXVK, WineD3D |
| Direct3D 9 | DXVK, dgVoodoo2 + DXVK, WineD3D |
| Direct3D 10 | DXVK, WineD3D |
| Direct3D 11 | DXVK, WineD3D |
| Direct3D 12 | VKD3D-Proton |
| OpenGL | Wine `opengl32` → host OpenGL, Zink when no native GL exists |
| Vulkan | native Vulkan (rare in Windows games) |
| DirectDraw | DDrawCompat, cnc-ddraw, dgVoodoo2 |

The translator converts API calls into Vulkan or OpenGL commands. It also manages:

- shader translation and compilation;
- pipeline state;
- memory allocation;
- synchronization;
- swapchain presentation;
- feature emulation for APIs the host does not support.

At this stage, common failures include:

- a game feature the translator does not implement;
- a shader the translator cannot compile;
- a swapchain or fullscreen issue;
- a driver bug triggered by a specific Vulkan feature;
- a missing `d3dcompiler` DLL;
- an anti-cheat or DRM component that rejects the translated environment.

## 2.8 Stage 7: the GPU driver

The Vulkan or OpenGL commands reach the GPU driver. On Android, the driver may be:

- Qualcomm’s proprietary Adreno driver;
- Turnip, Mesa’s open-source Vulkan driver for Adreno;
- another vendor driver (Mali, PowerVR);
- a virtualized driver such as VirGL.

On ARM Linux, the driver may be:

- Freedreno/Turnip for Adreno;
- Panfrost/PanVK for Mali;
- V3D for Raspberry Pi;
- AGX for Apple Silicon;
- vendor drivers for other SoCs.

On x86 Linux, the driver may be RADV (AMD Vulkan), ANV (Intel Vulkan), or the NVIDIA proprietary driver.

The driver is responsible for:

- compiling Vulkan/OpenGL pipelines to GPU machine code;
- allocating and managing GPU memory;
- scheduling work;
- handling presentation;
- reporting features and limits.

At this stage, common failures include:

- a missing Vulkan extension;
- an unsupported feature level;
- a driver bug;
- a shader compiler bug;
- memory exhaustion;
- a mismatch between the runtime driver and the display driver;
- a driver that works for Android apps but not for the translated workload.

## 2.9 Stage 8: audio, input and display

A game is not only graphics.

### Audio

Windows games use DirectSound, XAudio2, WASAPI or Media Foundation. Wine maps these to PulseAudio, PipeWire, ALSA, or Android audio APIs. Missing audio libraries, wrong buffer sizes, or a broken audio server can cause silence, crackling, or crashes.

### Input

Windows games use DirectInput, XInput, or raw input. Wine maps these to evdev, SDL, or Android input. Controllers may need mapping; touch controls are usually provided by the frontend.

### Display

The final image must be presented through the host compositor. On Android, SurfaceFlinger composites the app’s surface. On Linux, X11 or Wayland does it. On macOS, the window server does it. A game can render correctly and still show a black screen because presentation failed.

## 2.10 Pipeline variants by host

### Variant A: Android frontend

```text
Frontend app
  -> bundled RootFS (proot/chroot)
  -> bundled Wine
  -> bundled Box64/FEX
  -> bundled DXVK/VKD3D
  -> Turnip or Qualcomm driver
  -> Android SurfaceFlinger
```

Examples: Winlator, DroidDeck, GameHub, GameNative.

### Variant B: Android DIY (Termux)

```text
Termux
  -> proot-distro (Debian/Ubuntu)
  -> Wine (x86_64 under Box64, or ARM64 + FEX)
  -> Box64/FEX
  -> DXVK/VKD3D
  -> Turnip/Zink or system driver
  -> Termux:X11 or XServer XSDL
  -> Android display
```

This gives you more control and more ways to break things.

### Variant C: ARM Linux

```text
Linux userspace
  -> Wine/Proton (ARM64 or x86_64)
  -> FEX/Box64
  -> DXVK/VKD3D
  -> native Mesa driver (AGX, Freedreno, Panfrost, V3D)
  -> Wayland/X11
```

Examples: Asahi Linux, Raspberry Pi 5, Snapdragon X Elite.

### Variant D: x86 Linux

```text
Linux userspace
  -> Wine/Proton (x86_64)
  -> no CPU translation
  -> DXVK/VKD3D
  -> RADV/ANV/NVIDIA
  -> Wayland/X11
```

This is the easiest local path, but it is not the subject of the ARM-specific modules.

### Variant E: macOS

```text
macOS
  -> CrossOver/Whisky/Game Porting Toolkit
  -> Wine (x86_64 under Rosetta 2 on Apple Silicon)
  -> D3DMetal or DXVK/MoltenVK
  -> Metal
```

macOS is a non-Windows host, but its graphics path is different from Linux/Android.

### Variant F: ChromeOS

```text
ChromeOS
  -> Crostini (Linux container) or Borealis (Steam)
  -> Wine/Proton
  -> DXVK/VKD3D
  -> VirGL or native GPU
  -> ChromeOS compositor
```

The GPU path depends on the device and whether GPU acceleration is exposed to the container.

## 2.11 Where the pipeline usually breaks

| Symptom | Most likely layer | First check |
|---|---|---|
| Nothing happens on launch | Frontend, container, Wine loader | Frontend logs, container mounts, prefix |
| “Bad EXE format” | Architecture mismatch or 16-bit installer | File architecture, 32/64-bit Wine |
| Wine error about missing DLL | Windows API / dependency layer | DLL overrides, VC++/.NET/DirectX runtime |
| Game starts, then crashes at graphics init | Graphics translator or driver | DXVK/VKD3D logs, Vulkan device creation |
| Black screen but audio works | Presentation/swapchain or display driver | Fullscreen mode, swapchain, compositor |
| Low FPS with low GPU usage | CPU translation or single-thread bottleneck | Translator, CPU governor, game settings |
| Low FPS with high GPU usage | GPU or resolution | Resolution, effects, driver |
| Stuttering | Shader compilation, storage, memory | Shader cache, storage speed, RAM |
| Missing textures | Graphics translator or driver | DXVK/VKD3D version, driver |
| No sound | Audio layer | Audio server, Wine audio driver, dependencies |
| Controller not detected | Input layer | XInput/DInput, mapping, permissions |
| Crash after minutes | Thermal, memory, or driver instability | Temperatures, RAM/VRAM, logs |

## 2.12 The performance model

The pipeline adds overhead at several points:

1. **CPU translation** adds instructions and reduces effective IPC.
2. **Windows API emulation** adds function-call overhead.
3. **Graphics translation** adds draw-call and shader translation overhead.
4. **Containers and filesystems** add I/O overhead.
5. **Display composition** adds latency and may force an extra copy.

The GPU is often not the bottleneck on ARM. A game can be GPU-light and CPU-heavy, or vice versa. The diagnostic question is always: which layer is saturated?

The [performance module](13-performance.md) covers measurement in detail.

## 2.13 The mental model to keep

```text
Application
  -> Windows API (Wine/Proton)
  -> CPU translation (FEX/Box64) or native
  -> Graphics API (D3D/GL)
  -> Graphics translation (DXVK/VKD3D/...)
  -> Vulkan/OpenGL
  -> GPU driver
  -> Display
```

When something fails, point at a layer. When something is slow, measure a layer. When you change a setting, know which layer it affects.

## 2.14 What to remember

- The launch path is a sequence of independent layers, each with its own failure modes.
- Frontends orchestrate; they do not replace Wine, the translator, or the driver.
- The graphics API determines the translator; the translator determines which driver features matter.
- Audio, input and display are part of the pipeline, not afterthoughts.
- Android, ARM Linux, x86 Linux, macOS and ChromeOS implement the same logic with different components.
- Diagnose by layer, not by guessing.

Next: [03 — Wine and Proton](03-wine-proton.md).
