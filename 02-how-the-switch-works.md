# How the Switch works for a homebrew port

This is the Switch-facing view collected in the supplied notes. Hardware figures and behavior below are reported by those notes; check the exact libnx, Horizon and Mesa build used by a port before relying on implementation details.

## Hardware shape

The Switch has four Cortex-A57 CPU cores, with a game process commonly able to use three while the system uses the fourth. CPU frequency is reported as 1020 MHz in docked and handheld modes. The shared L2 cache is small relative to a large static recompile, so scattered guest-state accesses and large working sets can be expensive.

CPU and GPU share 4 GB LPDDR4. Copies, vertex conversion, texture decoding and framebuffer work compete for memory bandwidth. The Maxwell GM20B GPU runs at different clocks by mode; the supplied notes report a handheld default of 307.2 MHz, an official handheld profile at 460.8 MHz, and 768 MHz docked. Handheld GPU performance can therefore be the limiting factor even when a desktop build is fast.

A homebrew process launched as an applet has roughly 400 MB available in the supplied observations, while title takeover provides about 3.2 GB. Recompiled console games generally need full-memory launch mode.

The three-core limit comes from the homebrew loader's permissions, not the hardware. The Prototype 2 port (2026-10) raises `highest_cpu_id` from 2 to 3 in both NPDM permission tables (ACID and ACI0), either in a HOME-menu NSP forwarder or in a patched `hbl.nsp`. Thread core masks then decide what runs on core 3; that port keeps audio there and the main thread on cores 1-3. System services still use core 3, so the fourth core is shared, not exclusive. A forwarder needs the user's own `prod.keys`; a patched `hbl.nsp` changes a system file on the SD card.

## Engine, compatibility, and translation layers

Think of a port as a path from game logic to Switch hardware. The game engine owns gameplay, animation,
resource loading, rendering decisions, input interpretation, and audio mixing. It calls APIs supplied by
its original platform or by its engine abstraction layer. A Switch port must either replace those calls
with native Switch services or translate/emulate the behavior the game expects.

| Layer | What it does | What it does not automatically do |
|---|---|---|
| Game engine and game code | Runs gameplay, scripts, menus, animation, resource management, and engine-side rendering/audio/input logic | It does not know that Horizon services or Switch controller/audio behavior are available unless a port connects them. |
| Guest CPU translator or emulator | Executes original CPU instructions, or translates them into host instructions | It does not implement the original operating system, GPU, controller, audio device, or file system. |
| Compatibility runtime | Implements guest memory, threads, synchronization, files, timers, audio/device calls, and error/status behavior | It cannot guess missing contracts safely; unsupported calls need evidence, tracing, and an explicit implementation. |
| Device/API translation | Converts original graphics commands, input state, audio formats, or other device operations into host equivalents | It is separate from CPU translation; translated game code can run while graphics, input, or sound is still broken. |
| Switch host layer | Uses libnx, Horizon services, selected libraries, and a graphics backend to reach the console hardware | libnx is the Switch system interface, not a replacement implementation of every original console API. |

When engine source is available, gameplay and asset code may be reusable while its renderer, operating-
system calls, media paths, and build configuration still need a Switch implementation. The engine name
alone does not establish that a particular game can be built for Switch. With only a binary, CPU
translation still leaves the engine's platform calls and device behavior to be supplied by the port.

A source port usually adapts the engine's platform interface directly. A static recompiler translates CPU
code but still needs the original platform APIs and devices implemented. An emulator interprets or
translates CPU and device behavior as a coordinated runtime. Identify which of these layers a project
already supplies before adding another translation layer. The [porting workflow](01-porting-workflow.md)
describes how to choose the route for a particular executable.

### Graphics: commands, resources, and presentation

A game engine normally asks its graphics API to set state, bind resources, draw, resolve/copy targets,
and present a frame. For a console GPU port, the path is approximately:

1. Game/render code emits original API calls or a guest GPU command stream.
2. The guest graphics layer decodes commands and reconstructs guest state: shader inputs, textures,
   render targets, depth/blend/cull state, and synchronization.
3. The renderer translates that state and the guest GPU's resource layout into host resources and commands.
   This may include shader translation, texture untile/format conversion, vertex preparation, and resolves.
4. A Switch-supported backend such as Vulkan/NVK submits work through Horizon's GPU services to the
   Maxwell GPU; presentation then selects and displays the completed surface.

The CPU recompiler and graphics translator solve different problems. Correctly translated CPU code can
still feed the renderer wrong addresses or state. A renderer can submit valid Vulkan calls but still draw
the wrong image if it misinterprets the guest format, tiling, aliases, shader constants, or resolve order.
Conversely, a black or stale frame does not by itself prove that the game process crashed. Follow the data
from guest command/address to translated resource, host draw, completed target, and presented surface.

Common graphics responsibilities include:

- **State translation:** convert guest registers/API state into host pipeline and dynamic state. Preserve
  command ordering and invalidate cached state whenever a guest command changes it.
- **Resource translation:** map guest addresses, dimensions, formats, pitches, tiling, and aliases to host
  buffers/images. A copy or resolve must preserve the guest-visible contents and timing.
- **Shader translation:** translate guest shader instructions and resource bindings to a form supported by
  the selected Switch backend. Cache keys must include source code and translator/layout versions.
- **Synchronization:** represent guest fences, render-target feedback, and ownership changes with host
  barriers and fences. Missing synchronization can produce intermittent corruption; overly broad waits
  can stall the game or audio.
- **Presentation:** choose the correct completed image and honor the game's flip/vblank behavior. A valid
  draw call is not proof that the intended image was presented.

When graphics fail, first determine whether the game submitted work, whether the renderer emitted host
commands, whether the GPU completed them, and which surface was presented. See [Switch performance](03-switch-performance.md)
for the cost model and [testing and debugging](05-testing-and-debugging.md#graphics-correctness) for capture
and comparison steps.

### Input: map host controls to guest expectations

The Switch exposes controller input through Horizon/libnx or a library such as Switch SDL2. A port reads
that host state and converts it into the original game's controller representation or the engine's input
abstraction. The mapping must preserve more than button names:

- Button bit assignments and the distinction between pressed-this-frame, released-this-frame, and held.
- Analog stick ranges, center/dead-zone handling, trigger values, and any original pressure-sensitive
  controls.
- Controller connection changes, player assignment, handheld/docked availability, and remapping options.
- The game's input sampling cadence and the order in which input is updated relative to simulation and
  frame presentation.

Wrong mapping often appears as controls that do nothing, repeat unexpectedly, or feel delayed rather than
as a crash. If the port crashes while polling, inspect initialization and controller-handle lifetime,
input buffer sizes, and the guest API call/return contract. If inputs work in menus but not gameplay, check
which engine path consumes the state and whether a held/edge-triggered value was cleared too early.

### Audio: preserve format, timing, and thread behavior

The engine may produce audio through a guest mixer/device API or through a source-port audio backend. A
compatibility layer must turn that output into the format and buffers accepted by the Switch audio path,
then keep them filled on schedule. Record sample rate, channel layout, sample format, buffer size, mixer
cadence, and which thread owns each buffer. Convert formats explicitly rather than interpreting a buffer
with the wrong sample type or stride.

Audio is deadline-driven. A mixer that waits on the guest lock, a low-priority refill thread, a blocked log
write, or a long renderer operation can miss buffer deadlines and cause crackle or silence while the game
continues. A lock-order bug, invalid callback lifetime, or corrupt buffer can instead freeze or crash the
process. Keep the audio callback/refill path bounded, avoid file/network I/O there, count underruns, and
test with diagnostic tracing disabled because tracing itself can add enough work to cause gaps.

### Files, memory, and timing are part of compatibility

The game does not simply read files or allocate memory in abstract terms. It observes the original system's
paths, access modes, error codes, alignment, address aliases, timing, blocking, and wakeup behavior. A
missing directory may be a normal first-run condition; returning the wrong status can send the game into a
fatal path. A guest buffer readable by the CPU may still be invalid as a Horizon IPC buffer. A wait that
was cheap on the original system can become a busy loop that consumes a Switch core.

For each boundary, trace the guest request and arguments, the host operation and result, and the value
returned to the game. Keep the guest-visible contract consistent across success, partial completion,
failure, and wakeup. The runtime details below cover memory mappings, IPC, scheduling, storage, and clocks.

## What can crash, hang, or hide the game

The symptom tells you where to start, but it does not prove the cause. Separate an abrupt process exit from
a thread that stopped making progress and from a live game that is producing the wrong image or no sound.

| Symptom | Common failure classes | First useful evidence |
|---|---|---|
| NRO fails before the game entry point | Invalid NRO/package metadata, wrong startup/link layout, missing library, failed libnx initialization, or memory budget too small for launch mode | Build output identity, loader/console log, startup symbol/layout, and launch mode. |
| Exception or exit at a repeatable guest address | Unsupported/mistranslated instruction, wrong function boundary or ABI, invalid guest mapping, bad stack/TLS, or an incorrect manual hook | Exact NRO plus matching unstripped ELF, guest PC/registers, translated function, address map, and exception report. |
| Crash only during loading or first use | Missing/corrupt asset, wrong file status/path, allocation/commit failure, shader/pipeline creation error, or resource lifetime bug | Per-open file trace, memory/pool counters, first-use shader log, and the last guest/renderer milestone. |
| Process remains alive but stops progressing | Guest busy loop, lost wakeup, deadlock, thread starvation, blocking file/log/network operation, or GPU fence wait | Thread stacks/profiles, lock owner, wait object, last guest PC, queue/fence state, and log queue state. |
| Game code progresses but image is black, stale, or incomplete | No draw submitted, bad guest render state, wrong target/format/tiling, shader/resource translation error, missing resolve, or wrong presentation surface | Guest draw/flip counters, emitted host commands, intermediate targets, dimensions/formats, and present selection. |
| Game runs but controls do nothing or feel wrong | Host-to-guest button/range mapping, edge-versus-held handling, polling cadence, connection state, or input thread starvation | Raw host input next to mapped guest state and the guest poll/update call. |
| Game runs with silence, crackle, or intermittent freeze | Unsupported audio format, buffer underrun, late refill, mixer/guest-lock contention, or logging/I/O on an audio path | Audio format/cadence, underrun count, callback thread timing, lock waits, and comparison with tracing disabled. |

Treat crashes as boundary evidence, not just as a request to add a stub. Capture the exact build and first
failure, reproduce it with the shortest route, and change one boundary at a time. For translated CPU
failures, use [debugging recompiled code](09-debugging-recompiled-code.md); for visual failures, use the
[graphics-debug order](01-porting-workflow.md#graphics-debug-order) and [testing guidance](05-testing-and-debugging.md).

## Horizon and libnx

Horizon is not a desktop OS. Memory mappings, IPC buffers, thread scheduling, clocks, file access and GPU submission have console-specific costs and restrictions. A port may use libnx services and SDL for host startup, controller input, audio and windowing, but the game still expects its original operating-system and device semantics. The runtime supplies those semantics through compatibility bridges.

### Memory and IPC

Original console RAM can appear through aliases or device apertures. Those virtual views may not be possible on Horizon in the same way as Linux or Windows. Test the address layout on actual hardware and trace faults by guest address and region: ordinary RAM, an alias, an uncommitted allocation, or device memory are different problems.

Never assume guest memory is a valid Horizon IPC buffer. Copy through host-owned buffers for file/socket service calls when required by the platform path. Treat mappings and allocations as finite resources; avoid per-draw or per-frame kernel operations.

### Threads and waits

The supplied port notes report that priority 0x3B is time-sliced and most other priorities are cooperative. In a cooperative schedule, a busy guest spin loop can occupy a core and starve audio, rendering or another game thread. Replace known polling loops with bounded waits or event-driven wakeups while retaining the guest's expected checks. A yield is not equivalent to sleeping when the other thread has a different priority.

Keep lock ownership and wake decisions synchronized to avoid lost wakeups. Avoid detached workers where thread lifetime and failure handling are unclear. Profile before changing priority or affinity; a high-priority thread can make input, audio or other work less reliable.

### Storage and logging

microSD is useful for large game data, but random reads and repeated archive access can stall streaming. Cache repeated ranges when evidence shows rereads, read in larger aligned chunks where possible, and keep slow I/O off hot game/renderer paths.

Synchronous SD writes and network logging can stall every thread that shares a logging lock. Queue log lines and let a dedicated worker write or send them. Keep crash handling able to flush buffered diagnostics. Validate card contents after copying when silent card corruption is suspected.

### Graphics submission

The host graphics backend does not execute the original GPU instruction set directly. A compatibility renderer interprets or translates original graphics commands and emits Vulkan/NVK or another homebrew-supported backend. The renderer must reproduce resource formats, tiling, surface aliases, shader behavior, synchronization and presentation semantics.

Vulkan on NVK is the backend used by the original-Xbox port notes and by the supplied BO1 notes. The original-Xbox notes found Eden useful for GL checks, but it did not complete NVK submissions (the BO1 notes do not use Eden); use a real Switch for NVK correctness and performance. Driver behavior can vary with the exact Mesa-on-Horizon fork.

GPU allocations and submissions can involve costly kernel/driver work. Pool or suballocate resources when measurements support it, release only after GPU use completes, keep queues/submits bounded, and avoid global barriers when a narrower dependency is correct. These are design constraints to measure, not universal tuning constants.

### Clocks and presentation

Use stock clocks as the performance baseline and record docked/handheld mode. Clock overrides can mask bottlenecks and interact with system clock tools. The supplied notes report that CPU boost mode sharply lowers the GPU clock, making it suitable for some loading work but not ordinary gameplay. The Prototype ports found that keeping a temperature-sensor (`ts`) session open for the whole game coincided with Horizon OC's clock service crashing under load; avoid long-lived sensor polling.

Presentation cadence is quantized by display refresh. Track frame intervals and missed refreshes, not only average FPS. More frames in flight can improve overlap but may increase input latency. Test queue depth, vsync and pacing in the same demanding scene.

The detailed Horizon mapping, IPC, scheduling, clock, storage, audio, and logging material continues in
[Detailed Horizon runtime and hardware notes](#detailed-horizon-runtime-and-hardware-notes). Renderer and
NVK optimization material is in [Switch performance](03-switch-performance.md#detailed-nvk-renderer-and-gpu-engineering).

## Detailed Horizon runtime and hardware notes

This page collects Switch-specific facts and practices from the supplied project documents. It describes
the Horizon/libnx environment used by those homebrew ports; it is not a substitute
for checking the exact SDK, firmware, driver fork, launch mode, and hardware behavior of a new project.
Values reported by one port are evidence for that build, not universal constants.

For the broader port workflow, see [porting workflow](01-porting-workflow.md),
[how the Switch works](02-how-the-switch-works.md), and
[Switch performance](03-switch-performance.md).

### Hardware budgets that affect runtime design

| Resource | Supplied project observations | Porting implication |
|---|---|---|
| CPU | Four Cortex-A57 cores at about 1020 MHz; the game process commonly uses cores 0–2 while the system uses core 3 | Three application cores are a shared budget for guest work, runtime, audio, renderer, and storage. Do not assume each subsystem gets a core. |
| Cache and code locality | Shared L2 is about 2 MB; L1 instruction cache is about 48 KB | Large translated code and scattered guest-state accesses can miss cache frequently. Measure codegen, function order, and locality rather than assuming compiler size or instruction count predicts speed. |
| Main memory | 4 GB LPDDR4 shared by CPU and GPU; the supplied notes report about 8 GB/s from one CPU core | Vertex conversion, texture decoding, untile/copy work, and GPU traffic compete for the same memory bandwidth. Count bytes moved as well as calls. |
| GPU | Maxwell GM20B; supplied notes report 307.2 MHz default handheld, 460.8 MHz under the official handheld profile, and 768 MHz docked | A desktop or emulator result cannot establish Switch GPU performance. Record handheld/docked mode and effective clocks with each measurement. |
| Process memory | About 400 MB in album applet mode and about 3.2 GB with title takeover in the supplied observations | Memory-heavy ports generally need full-memory launch. Confirm the actual launch mode rather than diagnosing an applet memory ceiling as an allocator bug. |
| Storage | microSD uses UHS-I; sequential throughput is much better than small random reads | Streaming can be limited by rereads, latency, or card health even when sequential bandwidth looks adequate. |

The precise figures above come from the local port notes. Recheck them when changing hardware, firmware,
libnx, Mesa, or measurement method. Keep stock-clock performance as the baseline; overclock results answer
a different question.

### Address spaces, mappings, and memory services

#### Model guest memory before mapping it

An original platform may expose RAM through aliases, contiguous physical windows, or device apertures.
Desktop file mappings can make those views appear identical, while Horizon may reject the same aliasing
approach or impose different commit granularity. Draw the guest virtual ranges before implementation and
classify every range as ordinary RAM, alias, dynamically committed memory, or device memory.

- Reserve host address space deliberately when guest addresses or device windows must fit into a limited
  host user-address range. An AArch64 Linux bench can expose address-space failures hidden by an x86-64
  desktop.
- Where the guest requires multiple views of the same storage, use the platform/runtime's supported
  mapping strategy and explicit pointer/MMIO folding where necessary. Do not assume a successful ordinary
  RAM mapping proves alias correctness.
- Commit large guest ranges in the granularity Horizon can represent efficiently. The supplied projects
  report using 2 MB commit granules and congruent mappings for large guest windows. Commit each granule
  once and retain it; repeatedly extending a range from the same base previously caused remapping errors
  (`0xCE01`) in an Xbox port.
- Use a desktop mode that disables convenient aliases when possible, then verify the final mapping on
  hardware. A desktop no-alias mode is a useful early filter, not proof of Horizon behavior.
- Keep guest stacks, runtime data, TLS, image sections, and BSS in a documented range map. Select stack
  and runtime bases from the actual executable end, not a value copied from another title.

#### Keep guest pointers out of Horizon IPC

Horizon services may require host-owned, aligned IPC buffers. A guest pointer that is readable by the
process is not automatically valid for a filesystem, socket, or other service call. Use a bridge-owned
buffer, copy the requested range in and out, and preserve the guest operation's status and partial-transfer
semantics. One supplied port froze on small unaligned save-file reads when guest memory was passed to the
filesystem service; another could not send GPU-mapped image memory through the socket service.

For every bridge operation, record the guest address, length, alignment, access direction, service result,
and guest-visible status. Treat filesystem errors distinctly: missing parent, missing final component,
invalid handle, and failed directory creation can lead the game's own code down different paths. Close
directory handles deterministically; a leaked handle can become a system-level failure rather than a local
resource warning.

#### Avoid fault-driven memory emulation in hot paths

The supplied ports moved away from page-fault-based emulated accesses where a native renderer or explicit
mapping could be used. Fault handling is expensive and interacts poorly with preemption; one cited
Xbox-360 port observed stale vector-register state when a thread was preempted inside the exception path.
If a guest-memory watch mechanism is required for correctness, keep it narrow, measure its fault rate, and
verify vector state across preemption on the actual console.

Large-page mappings may reduce TLB pressure for broad guest windows. This is a measured optimization, not
a correctness requirement: compare aligned and baseline mappings, record allocation/commit failures, and
avoid confusing Horizon's CPU-side guest-memory pages with the GPU driver's own page layout.

### Threads, waits, and service deadlines

The supplied Horizon observations report that priority `0x3B` is time-sliced while most other priorities
are cooperative. Lower numeric priority values represent higher priority in the cited libnx usage. Verify
the exact scheduler behavior for the SDK and firmware being used before applying these values blindly.

- Assign priorities from deadlines and dependencies. Audio, presentation, storage, and renderer workers
  must run often enough to meet their deadlines, but an elevated busy worker can starve the game even when
  another core appears idle.
- When a thread starves, first inspect which higher-priority thread is consuming its core. Lowering the
  hog is often safer than raising the waiter, which can starve other services.
- A preferred core is not necessarily exclusive affinity. Pinning can improve locality for one workload
  and worsen minimum frame rate for another; compare migrations and worst-case frame times before keeping
  a core mask.
- A guest busy-wait that was cheap on a multi-core original console can consume one of the Switch's
  limited application cores. Replace only identified polling loops with a bounded wait or a wakeable
  event, then run the guest's own condition check. A fixed sleep can add latency to every poll; choose
  spin-then-sleep or event wakeups from measurements.
- Make wait registration and wake decisions under the same synchronization rule. A lost wakeup can look
  like a renderer or game hang. Keep persistent worker lifetimes explicit; early-start detached host
  threads caused startup failures in one supplied project.
- Zero-timeout polling followed by yield is still a spin when yield only hands control to a thread of the
  same priority. Inspect the actual wait path and the owner/producer priority.
- Avoid per-draw or per-frame syscalls, IPC, dynamic allocation, and condition-variable notifications
  without waiters. A few microseconds repeated thousands of times can dominate a frame.
- Horizon builds may implement thread-local access through a helper call. Keep guest register state in
  locals only when the translator/runtime's save/restore rules allow it, and profile before reshaping the
  guest-state model.

#### Timing and clock behavior

Use supported timing APIs such as `armGetSystemTick` in the cited libnx environment; direct reads of
`cntvct_el0` were reported to fault. Use a correct scale for converting ticks to time and test long uptimes
for integer overflow. Clock-query functions may also be frequent guest lock handoffs; count their call rate
before optimizing them.

Record the effective CPU, GPU, and memory clocks at test time. The supplied notes describe an official
handheld GPU profile at 460.8 MHz and a CPU-boost mode that raises CPU frequency while dropping the GPU
to a very low clock. CPU boost may help CPU-bound loading but is not a general gameplay mode. Restore
clock overrides on exit, sleep/wake, and dock transitions, and do not fight a separate overclock service.

### Audio, files, and logging

Treat audio as an independent real-time subsystem. Record audio-thread priority, mixer cadence, underruns,
and interaction with the guest lock. A run with voice or audio tracing enabled is not a clean quality
measurement if tracing adds enough work to create gaps.

Keep storage off the game and renderer hot paths. Cache repeated archive ranges only after logs show
rereads; large sequential reads and small random reads have different costs. Do not reduce working pools
blindly: frequent eviction and reload can make streaming stutter worse.

Use a bounded, non-blocking log queue and a dedicated writer/sender. An SD write or blocked network send
must not hold a mutex used by audio, rendering, or guest threads. Count dropped messages and expose a
crash-time drain path. If a network log sink is used, a separate sender thread should own socket writes;
nonblocking flags alone do not prove that every send path cannot stall.

Validate suspect storage after copying and after remount. One supplied project's notes report stale/corrupt
SD blocks that differed between host and console reads. Hash data on the device when possible, keep a known
good copy, and quarantine suspect files rather than repeatedly overwriting the same allocation.

### Switch runtime review checklist

- [ ] Record launch mode, applet/title-takeover memory budget, handheld/docked state, firmware, libnx, and
      driver revision.
- [ ] Map guest RAM, aliases, committed regions, device windows, stack, runtime data, and TLS explicitly.
- [ ] Confirm no guest buffer is passed to an IPC service without a supported mapping and alignment.
- [ ] Exercise thread priorities, wakeups, audio deadlines, and guest polling on hardware.
- [ ] Measure syscalls, allocations, logging, and file reads in the hot path.
- [ ] Verify long-session clock conversion, sleep/wake, and dock transitions.
- [ ] Check card data against a known-good source when file behavior is inconsistent.

### Local source trail

This page synthesizes `conker-nx/docs/04-switch-platform.md`, `conker-nx/docs/11-testing-on-console.md`,
`conker-nx/docs/12-porting-guide.md`, `bo1-nx/docs/SWITCH_OPTIMIZATION_GUIDE.md`,
`bo1-nx/docs/OTHER_PORTS_RESEARCH.md`, and `bo1-nx/docs/OPTIMIZATION_LOG.md`. Project-specific traces,
addresses, and measured outcomes remain in the corresponding Xbox/Conker and Xbox-360/Black Ops folders.
