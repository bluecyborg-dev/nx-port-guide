# Original Xbox port research

This file consolidates original-Xbox findings in the supplied Conker notes. It does not describe Xbox 360 ports.

## Hardware and compatibility implications

Original Xbox titles in these notes target a single-core, 733 MHz Pentium III-class x86 CPU, NV2A graphics, 64 MB unified RAM and the MCPX audio processor. The graphics path is Direct3D 8/XDK state and pushbuffers. This differs from Xbox 360 PowerPC/Xenos and its PM4 command stream.

The source notes describe xboxrecomp as a static x86-to-C recompilation path. The XBE is parsed, executable sections are disassembled, functions and ABI are analyzed, and output is built with the title glue and runtime for Switch. The recompiled game still expects Xbox kernel, file, memory, input, audio and graphics behavior; the runtime supplies those services.

Important original-Xbox issues reported across the supplied ports include GPU fence and vblank behavior, PGRAPH/DPC handling, DirectSound/APU doorbells, x87 precision mode, x86 flag and SSE instruction semantics, unaligned atomics, missed indirect function entries, RAM mirrors, tiled framebuffer apertures, and time-stepping that assumed the Xbox's frame rate. These are mechanisms to investigate, not a fixed list of hooks for every title.

## Runtime debug switches added during the Conker port

These are xboxrecomp runtime features, independent of the title. Set them as environment variables on desktop builds, or as `KEY=VALUE` launch arguments / `*_env.txt` lines on Switch.

| Switch | Effect |
|---|---|
| `RECOMP_FILE_TRACE=1` | logs every `NtCreateFile` the bridge handles: guest path, access mask, create disposition, create options, attributes, returned status and `Information` |
| `RECOMP_FILE_TRACE=2` | the same, plus the code addresses found on the guest stack (innermost first: return addresses and stray code pointers), to find the game routine that issued the open |

## Reference ports in the supplied material

| Port | Title | Reported result / useful evidence |
|---|---|---|
| nfsu2-sw | Need for Speed: Underground 2 | Around 20 FPS in race scenes in the cited run; notes on frame pacing, GIL scheduling, audio and graphics waits |
| nfsu1-sw | Need for Speed: Underground | Steady 30 FPS in the cited race run at overclocked clocks; includes Vulkan threading and frame-lag experiments |
| nfsc-sw | Need for Speed: Carbon | Boots and plays; notes on flip counting, aliasing surfaces and graphics fixes |
| burnout3-sw | Burnout 3: Takedown | Reported 40–55 FPS in races in a tuning run; most recent generic fixes in the source comparison |

Numbers are historical source-note results, with differing clocks, build revisions and workloads. They are not directly comparable benchmarks. The source authors recommend matching library function bodies and transferring hook mechanisms while discovering addresses in each new XBE.

## Shared investigation topics

- Identify D3D device pointer, GPU fence counters, wait routines, vblank/PGRAPH handlers and presentation interval.
- Confirm audio initialization, DSP command handshakes and whether audio time advances movies.
- Audit every translated x86 instruction and flag path used by reachable code; x87 precision and atomics require special care on AArch64.
- Test graphics palette formats, register combiners, texture stages, fog, immediate-mode attributes, surfaces and cube maps as the title uses them.
- Review title region/video mode, language, filesystem partitions, save initialization, USB/input timing and network waits.
- Check the kernel bridge's NT status codes against what titles branch on: `STATUS_OBJECT_PATH_NOT_FOUND` (missing parent, XAPI `ERROR_PATH_NOT_FOUND`) versus `STATUS_OBJECT_NAME_NOT_FOUND` (missing last component, `ERROR_FILE_NOT_FOUND`) for `NtQueryFullAttributesFile` and `NtCreateFile`. Conker's cache setup depends on the difference (see the [Conker logbook](conker/conker-logbook.md)).
- Check the XBE image end against the runtime's fixed guest stack base: xboxrecomp's default `0x00780000` fits images ending below it (Burnout 3), but Conker's ends at `0x00891D20`, so its stacks and runtime data block overlapped `.data` until the base was moved to `0x00900000`.
- On 64-bit hosts with a 48-bit user address space (aarch64 Linux), make sure guest base + `0xFFFFFFFF` fits: an OS-chosen base near the top of the address space leaves no room for the device apertures at `0xFD000000` and above.
- Confirm guest simulation uses wall-clock elapsed time rather than a fixed step that slows when frames are missed.

The Conker-specific target analysis, port plan and logbook live in [conker/](conker/). Full original source notes are preserved in [conker/source-notes/](conker/source-notes/).
