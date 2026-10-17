# Porting workflow

A homebrew port is a compatibility project: preserve the original game's code and data while replacing the original console services with a runtime the Switch can host. First identify which layer owns a failure, then make the smallest change that addresses it.

## Keep three layers separate

| Layer | Typical responsibilities | Port-specific work |
|---|---|---|
| Recompiler/toolkit | Executable parsing, instruction translation, guest register model, function dispatch | Discover missed function entries; report/fix translation bugs with a minimal reproduction |
| Original-platform runtime | Kernel, memory, files, threads, audio/input and original graphics API behavior | Game paths, image layout, stack/data addresses, device hooks and per-title quirks |
| Switch host | libnx startup, Horizon memory/thread services, display/input/audio and host graphics backend | Select the renderer and build configuration; verify real memory behavior, scheduling and performance on hardware |

Keep executable or GPU details within their source platform. Original Xbox x86/NV2A material does not describe Xbox 360 PowerPC/Xenos PM4 behavior. Shared Switch observations can inform both, but a measured gain in one port is not a guarantee for another.

## What you need before you begin

An NRO is the Switch homebrew executable package. Packaging a game file as an NRO does not translate its
instructions or replace the original console's operating system and graphics services. There is no
universal converter that turns an arbitrary XBE, XEX, PC executable, disc image, or ROM into a working
Switch game. The NRO must contain Switch-targeted code: a source port, translated/recompiled game code, or
an emulator/runtime that supports that game's original CPU and devices.

If none of those routes supports the executable, the missing work is the port runtime or translation
support; a packaging command cannot supply it. The no-op NRO below is still useful as a hardware/toolchain
baseline, but it does not run the game.

### Project inputs

| Input | Why you need it |
|---|---|
| Exact game executable and the data/assets it expects | The executable may refer to archives, media, configuration, fonts, saves, or other files that are not embedded in it. Keep the extracted layout intact until traces show which files are needed. |
| Known version, region, and hashes | Recompilers, function maps, patches, and asset paths can depend on the exact revision. A hash identifies the file used for analysis and each build. |
| Original platform and technical format | Identify the CPU/ABI, endianness, executable/container format, memory layout, operating-system calls, and graphics API/GPU. This determines which analysis tools and runtime are compatible. |
| A viable implementation route | Obtain the source and dependencies for a source port, or a recompiler/emulator that supports this exact executable format and CPU. A tool for a different console or instruction set will not become compatible because both projects target Switch. |
| Rights to use the game data and dependencies | Keep proprietary game files out of public source control and check the licenses for code and bundled libraries before distribution. |

### Build and test tools

| Need | Typical choice in the supplied projects | Required when |
|---|---|---|
| Switch compiler and system library | devkitPro's devkitA64 toolchain and libnx | Required for the NRO build; install them on the host or use a build container that includes them. |
| NRO packaging tools | `nacptool` for application metadata and `elf2nro` for packaging; `nxlink` is one upload option | The build system does not already call these tools for you. Prefer the project's build script so metadata, startup objects, and linker settings stay consistent. |
| Host build tools | Git, the project's compiler/toolchain, and its build system (commonly Make or CMake with Ninja); Python or other host runtimes when required by its generator | Running the selected port's build and code-generation steps. Use the versions recorded by that project. |
| Original-game analysis or code generation | A platform-specific parser, disassembler, ABI/function analyzer, recompiler, or asset extractor | The chosen static-recompilation or source-port pipeline requires it. These are not interchangeable across platforms. |
| Container runtime | Docker or another container system used by the project's build instructions | Needed only if the project documents a container build or its dependencies are isolated there. Match the image architecture to the host where practical. |
| Renderer libraries | Only the backend selected by the port. The supplied NVK builds use a Switch Vulkan driver; some ports also build glslang for runtime shader compilation. | The game renderer actually needs these components. A Vulkan or NVK dependency is not required for every NRO. |
| Supporting libraries | Examples include Switch SDL2 for input/audio/window services, zlib/zstd/expat for a chosen driver, and a media library for a title that uses video/audio codecs | The selected runtime calls them on Switch. Keep optional dependencies out until the title path needs them, and check their licenses and target builds. |
| Switch homebrew test setup | A Switch configured to launch homebrew, a usable SD card, and the chosen launch route (for example Homebrew Menu/NetLoader or copying the NRO to SD) | Proving the NRO actually starts on hardware. A successful host build cannot replace this step. |
| Run evidence | Console log capture plus the exact NRO and matching unstripped ELF; add frame/profile captures when relevant | Diagnosing crashes and proving which build reached which milestone. |

The root [build process](07-build-process.md) describes the shared NRO stages. Follow the selected title's
platform/game build guide for its exact recompiler, library versions, commands, and asset layout. The
Conker and Black Ops examples use different guest CPUs, generators, renderers, and dependency sets.

### Prove the Switch toolchain before connecting the game

Build and launch a tiny no-op or simple-screen NRO from an installed devkitPro Switch example. This checks
the devkitA64/libnx setup, NRO packaging, Homebrew Menu launch route, and console access independently of
the game runtime. Start from the example's Makefile/build files so its `switch_rules`, startup objects,
metadata, and NRO targets are included; a handwritten partial Makefile can fail before compilation or
produce a package that does not boot. Keep this as the known-good baseline while adding the game.

The stages prove different things:

1. Compilation means the host tools emitted an executable.
2. Packaging means the ELF was wrapped into an NRO with the expected metadata and startup layout.
3. Launching the no-op NRO on the Switch proves that this toolchain/package route runs on the device.
4. Observing game entry-point progress proves that translated/source game code started.
5. Seeing the title or a stable game frame proves graphics and presentation for that route.

A valid `NRO0` header proves packaging, not hardware execution. Keep the milestone claims separate in the
log and use [testing and debugging](05-testing-and-debugging.md) to record the evidence.

## A repeatable sequence

1. **Inventory the title.** Record executable identity/hash, region, imports, entry point, sections, executable sections outside the main text section, memory end, data layout and required game files. Keep the extracted tree intact.
2. **Understand the translation pipeline.** Parse the executable, disassemble code, identify functions and calling conventions, review callbacks/vtables/jump tables, then lift. Inspect unsupported instructions and unresolved indirect targets.
3. **Set safe guest memory.** Choose stack, runtime-data and heap regions from the actual image/BSS layout. Trace aliases and device apertures explicitly. Do not assume desktop mappings behave like Horizon.
4. **Build a minimal boot path.** Initialize host services, map guest memory, dispatch the original entry point and route file access to extracted assets. Add only compatibility hooks supported by executable evidence or runtime traces.
5. **Reach the title screen.** Resolve fatal errors, missing functions, graphics initialization, audio startup and movie timing. Check save/cache directories and error handling.
6. **Reach gameplay and verify behavior.** Test input, audio, saves, movies, networking/offline paths, simulation timing, menus, resolution and controller options. Capture graphics state when the first incorrect frame appears.
7. **Profile on the Switch.** Separate game-thread, renderer-thread, GPU, storage and waiting costs. Change one variable at a time and retain the old build for comparison.
8. **Record durable findings.** Keep a per-game logbook of symptoms, evidence, fixes and which build/device verified each result. Move general compatibility discoveries into shared guidance only after validating their scope.

## From a game file to the first NRO run

Use this route to reach a concrete hardware milestone before attempting complete compatibility or
optimization. Keep each stage runnable so a failure can be assigned to the game analysis, runtime, build,
or console layer.

### 1. Inventory the game files

Start from the user's exact executable and the full data set it expects. Record the file names, byte sizes,
hashes, version/region, and directory layout. Identify the executable format, entry point, CPU architecture,
calling convention, memory sections, imports, and graphics API/GPU. Preserve the untouched files and run
analysis against a working copy. Do not assume the largest file is the only required input; logs may reveal
additional archives or configuration later.

If the executable is only one part of a disc/install image, first use the title's documented extraction
path to produce the expected files. Keep extraction, analysis, generated code, build output, and run logs in
separate directories. The project's source-platform/game guide should identify supported input formats and
known version requirements.

### 2. Choose a route that supports this executable

| Route | Starting point | Work required before it can run on Switch |
|---|---|---|
| Source port | Available game/source-port code | Port or replace platform-specific code and dependencies, implement Switch services, and build for AArch64/Horizon. |
| Static recompilation | Executable for a supported guest CPU/format | Parse and analyze the binary, discover functions/ABIs, translate guest code, then provide guest memory, kernel/API behavior, assets, and a renderer. |
| Emulation or dynamic translation | Executable plus an emulator/runtime that supports its CPU and devices | Configure or port the emulator/runtime, supply its firmware/data requirements where applicable, and validate its CPU/GPU/device paths on Switch. |

Do not start code generation until the toolchain explicitly supports the input executable and guest CPU.
For a static recompiler, review function boundaries, callback/thread entry points, ABI reports, and
unsupported instructions before treating generated code as usable. For a source port or emulator, confirm
its own documented Switch path and dependency set. See [the Conker and Black Ops build procedures](07-build-process.md)
for examples of platform-specific pipelines.

### 3. Prepare the workspace and identify generated outputs

Keep the original executable and assets unchanged. Use separate directories for inputs, generated code,
host/console build trees, and run evidence (for example `game/`, `generated/`, `build/`, and `run/`). Mark
generated and proprietary content so it is not accidentally committed. Record the source commit, executable
hash, generator revision/options, build flags, driver revision, and final artifact paths in a build/run
manifest. The [development and build practices](04-development-and-build-practices.md) page describes
invalidation and reproducible build records.

### 4. Build the no-op NRO and verify the hardware route

Before game integration, use the devkitPro sample described above to build an NRO and launch it on the
actual Switch through the same Homebrew Menu, title-takeover, SD, or NetLoader path intended for the game.
Record the NRO hash, device/launch mode, and a console log or other direct evidence. This confirms the
toolchain and delivery path, not the game's compatibility.

### 5. Build the smallest game boot path

Add the game's entry point and only enough runtime to start it: initialize libnx and logging, reserve and
map guest memory according to the binary's real sections/BSS, establish stack and runtime data, dispatch
the guest entry point, and route file reads to the extracted assets. Implement guest service calls as the
game reaches them. Log unresolved imports/indirect calls and unsupported instructions with enough address
and register context to reproduce them. A safe placeholder is useful only when evidence shows the tested
route can tolerate it; a silent stub for a required file, thread, memory, or graphics operation can hide
the actual blocker.

For static recompilation, regenerate only when the executable, function seeds/discovery, ABI analysis, or
translator inputs change. Runtime, renderer, or platform-layer edits normally use the incremental native
build. Keep generated code reproducible and do not hand-edit it as the permanent fix. Use the
[recompiled-code debugging guide](09-debugging-recompiled-code.md) to distinguish translation errors from
runtime failures.

### 6. Get repeatable progress on a desktop or container target

Where the project supplies an AArch64 Linux/container target, use it to find guest startup, file, ABI,
threading, and many renderer-logic failures before deploying. A lavapipe/software-renderer path can provide
repeatable correctness captures. It cannot prove Horizon memory mappings, Switch services, NVK behavior,
or hardware performance.

Choose one short, reproducible route: process initialization, first asset reads, first menu/load, and a
known input or game-state event. Capture the log from process start, count frames or guest calls, and
record where progress stops. If the image is black, use logs and intermediate surfaces to determine
whether guest code reached rendering; the final screen alone does not distinguish a paused game, missing
draws, or a presentation error.

### 7. Build and package the Switch NRO

Use the title's Switch build script or documented CMake/Make target. That target should compile the
Switch/AArch64 runtime and generated game code, link the target libraries, create application metadata,
and package the ELF as an NRO. With the common devkitPro toolchain, `nacptool` creates NACP metadata and
`elf2nro` packages the ELF; project scripts should supply the exact arguments. Do not copy commands from a
different title without checking its linker script, startup objects, dependencies, and metadata.

Retain the unstripped ELF beside the exact NRO that will be tested. Confirm that the build output timestamp,
size, and hash correspond to the intended source/configuration, that the upload or SD-staging step selects
that file, and that the required entry/startup layout is correct for the project's loader path. Some
supplied build scripts check `_start` at address zero; treat that as a project/toolchain requirement to
verify, not a rule to assume for every linker setup. The detailed package checks are in
[the build process](07-build-process.md#build-success-is-not-runtime-evidence).

### 8. Launch that binary on the Switch and prove game progress

Deploy the exact NRO through the tested SD or network route. Use title takeover when the port requires its
larger process memory budget. Start log capture before launch, record handheld/docked mode and effective
settings, and follow a repeatable route. Add a heartbeat or milestone log around guest entry, asset load,
menu initialization, and frame/present so a black output can still be separated from a process that never
started.

Report the strongest result actually observed:

| Milestone | Evidence | What it establishes |
|---|---|---|
| Homebrew smoke test | No-op NRO launched on the physical console; log/screen confirms its own loop | Toolchain, packaging, and console launch route work. No game code has been proven. |
| Guest-code run | The game entry point or a named game initialization milestone appears in the console log, then execution continues | The tested game-code path runs on hardware. Graphics, controls, and full compatibility remain separate checks. |
| Visible title/menu | A frame from the game is presented, ideally with repeatable controller input | The tested boot/render/input route works; gameplay, audio, saves, and performance are not implied. |

For the first goal, aim to launch the NRO and show sustained guest-code progress on the Switch. A blank or
incorrect image can still accompany proven guest execution, but it is not evidence that rendering works.
Keep logs, the NRO/ELF pair, route/settings, and unresolved calls together, then fix the earliest failed
stage before widening the route. Continue with [console validation and deployment](05-testing-and-debugging.md#switch-console-validation-and-deployment).

## Engineering gates and expected evidence

Treat each step as a gate with an artifact that can be reviewed. This prevents an observation such as
"the title opened" from silently becoming evidence that the renderer, saves, or hardware path also work.

| Gate | Evidence to retain | Exit condition |
|---|---|---|
| Target inventory | Executable/data hashes, region/revision, section map, imports, memory end, asset inventory | The analysis covers every code-bearing section and identifies runtime inputs. |
| Translation | Generator version/options, function/ABI reports, unresolved target list, untranslated-instruction census | Missing entries and instruction fallbacks have an owner and a verified disposition. |
| Runtime boot | Guest stack, host/service logs, mapping map, file-operation trace, build identity | Entry point reaches a stable title state without unresolved calls, invalid memory, or accidental overlaps. |
| Functional paths | Repeatable input route, fresh/warm storage cases, input/audio/movie/save observations | Menus and representative gameplay paths complete without hidden dependencies on stale state. |
| Graphics | Deterministic frame/command capture, intermediate targets, shader/state identifiers | The first incorrect operation is localized; a final screenshot alone is not a diagnosis. |
| Hardware | Exact binary pair, launch mode, logs, device mode, settings, stock clocks | Switch-only behavior and performance claims are supported by a hardware run. |
| Optimization | Baseline and candidate builds, correctness comparison, stage measurements, rollback switch | A candidate improves the measured critical path without unacceptable behavior changes. |

Use explicit status words in logs: **observed** means directly present in the artifact; **inferred** means
an explanation consistent with evidence; **hypothesis** means an experiment remains; **verified** names
the exact build and environment that passed. A result from a desktop renderer must not be described as
hardware verification.

## Subsystem contract inventory

For every host bridge, write down the guest-visible contract before implementing a special case:

- Inputs and outputs, including registers, stack arguments, alignment, sizes, and ownership.
- Whether the call blocks, wakes a thread, releases a guest lock, or changes interrupt/IRQL state.
- How failure is reported, including status-code distinctions and partial completion.
- Whether the guest can observe ordering, timing, persistent state, or a side effect on another device.
- Which addresses are guest RAM, aliases, device apertures, shared mappings, or host-owned buffers.

This contract applies to files, memory, events/timers, graphics fences, audio, controller input, movies,
and save/cache APIs. It helps decide whether a bug belongs in the shared runtime, platform layer, guest
translation, or a title-specific hook.

## Function discovery and manual hooks

Static function discovery can miss thread entry points, callbacks, class-driver tables, code after padding, or indirect-call targets. A seed list is a hypothesis about function boundaries: validate each address against bytes, references and surrounding control flow. A seed inside a real function can split it and create hard-to-understand failures.

A manual native replacement or wrapper must document its calling convention, inputs/outputs, stack cleanup, guest-state effects and evidence. Keep a comparison mode where feasible so the replacement can be checked against the translated implementation before trusting it.

## Graphics-debug order

For a wrong frame, capture the command stream/frame trace and identify the first incorrect target or state. Check guest framebuffer address, format, tiling, aliasing, clear/resolve order and texture contents before changing shaders. Then inspect translated shader state and host API resource use. An image that looks wrong at the final output can originate in an earlier surface or data upload. It can also originate in the CPU: geometry projected off screen on every back end usually means the title computed bad transform constants in translated code. Check that its camera and world matrices are well formed (orthogonal rows, equal row lengths) before changing clip, cull or depth rules ([testing and debugging](05-testing-and-debugging.md#graphics-correctness)).

## Data and provenance

Keep proprietary game images, extracted assets, generated translated code and disc-derived captures out of source control. Keep project logs separate from generic guidance. Preserve whether a statement is a measured result, a title-specific observation, a proposal, or an estimate; do not promote an estimate into a benchmark.
