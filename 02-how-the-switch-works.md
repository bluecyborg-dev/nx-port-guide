# How the Switch works for a homebrew port

This is the Switch-facing view collected in the supplied notes. Hardware figures and behavior below are reported by those notes; check the exact libnx, Horizon and Mesa build used by a port before relying on implementation details.

## Hardware shape

The Switch has four Cortex-A57 CPU cores, with a game process commonly able to use three while the system uses the fourth. CPU frequency is reported as 1020 MHz in docked and handheld modes. The shared L2 cache is small relative to a large static recompile, so scattered guest-state accesses and large working sets can be expensive.

CPU and GPU share 4 GB LPDDR4. Copies, vertex conversion, texture decoding and framebuffer work compete for memory bandwidth. The Maxwell GM20B GPU runs at different clocks by mode; the supplied notes report a handheld default of 307.2 MHz, an official handheld profile at 460.8 MHz, and 768 MHz docked. Handheld GPU performance can therefore be the limiting factor even when a desktop build is fast.

A homebrew process launched as an applet has roughly 400 MB available in the supplied observations, while title takeover provides about 3.2 GB. Recompiled console games generally need full-memory launch mode.

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

Use stock clocks as the performance baseline and record docked/handheld mode. Clock overrides can mask bottlenecks and interact with system clock tools. The supplied notes report that CPU boost mode sharply lowers the GPU clock, making it suitable for some loading work but not ordinary gameplay.

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
