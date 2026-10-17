# Original Xbox port research

This page summarizes reusable original-Xbox porting context. Findings from other console generations and general Switch optimization measurements are collected in the [prior port research database](../../08-prior-port-research-database.md). Conker's binary analysis, runtime traces, build notes and ongoing hypotheses are in the [Conker folder](conker/).

## Platform characteristics

Original Xbox titles in the supplied material target a single-core, 733 MHz Pentium III-class x86 CPU, NV2A graphics, 64 MB unified RAM and the MCPX audio processor. The graphics path commonly uses Direct3D 8/XDK state and pushbuffers. Xbox 360 PowerPC/Xenos and its PM4 command stream are different targets; their findings are not original-Xbox behavior.

The notes describe `xboxrecomp` as a static x86-to-C recompilation path: parse the XBE, disassemble executable sections, analyze functions and ABI, then build generated code with title glue and a Switch runtime. The recompiled game still expects Xbox kernel, file, memory, input, audio and graphics behavior, which the runtime must provide.

## Compatibility areas to investigate

Treat these as inspection prompts rather than a universal hook checklist:

- GPU fences, vblank timing, PGRAPH/DPC handling and presentation interval.
- DirectSound/APU doorbells, DSP command handshakes and whether audio time advances movies.
- Reachable x86 instruction, flag, x87 precision and SSE semantics; unaligned atomics and indirect function entries.
- RAM mirrors, tiled framebuffer apertures, D3D8 state and pushbuffer behavior actually used by the title.
- Region/video mode, language, filesystem partitions, save initialization, USB/input timing and network waits.
- Fixed-step simulation that may slow when frames are missed; prefer testing elapsed-time behavior in repeatable scenes.
- NT status distinctions that guest code branches on, especially missing parent versus missing final path components.

## Prior original-Xbox port references

The supplied source notes contain four comparison projects. Their FPS descriptions are historical results with differing clocks, builds and workloads, not a directly comparable ranking.

| Port | Title | Reported result / research area |
|---|---|---|
| nfsu2-sw | Need for Speed: Underground 2 | Around 20 FPS in cited race scenes; frame pacing, GIL scheduling, audio and graphics waits |
| nfsu1-sw | Need for Speed: Underground | Steady 30 FPS in cited race run at overclocked clocks; Vulkan threading and frame-lag experiments |
| nfsc-sw | Need for Speed: Carbon | Boots and plays; flip counting, aliasing surfaces and graphics fixes |
| burnout3-sw | Burnout 3: Takedown | Reported 40–55 FPS in cited races; generic fixes in the supplied source comparison |

The source author recommends matching library function bodies and transferring hook mechanisms only after discovering the corresponding title addresses. The [Conker notes](conker/) document one target-specific investigation; its results should not be read as generic Xbox facts.

## Conker case study

Conker's XBE analysis, PAL/region questions, imported kernel APIs and game-specific runtime/render findings are maintained in [target analysis](conker/01-target-analysis.md), [port plan](conker/02-port-plan-and-risks.md) and the [Conker logbook](conker/conker-logbook.md). Preserve those details with the game as the work proceeds; this platform page is intended to remain a stable starting reference.

Original source notes are retained in [Conker source notes](conker/source-notes/).
