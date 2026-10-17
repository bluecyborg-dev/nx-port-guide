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
