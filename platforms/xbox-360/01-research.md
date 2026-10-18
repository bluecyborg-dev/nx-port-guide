# Xbox 360 port research

This page is a platform-level orientation for Xbox 360 titles. Cross-project measurements and reusable methods are indexed in the [prior port research database](../../08-prior-port-research-database.md). The supplied Xbox 360 work is primarily one case-study title; its measurements, unfinished experiments and title-specific decisions stay in [its folder](black-ops/).

## Platform and porting boundaries

Xbox 360 titles use a PowerPC-derived Xenon CPU and Xenos GPU. A Switch homebrew port may statically recompile guest PPC code to AArch64 and translate the Xenos command stream (PM4) into a host graphics API, or intercept at a higher D3D-level renderer. These paths have different costs and correctness obligations; do not treat results from an original Xbox NV2A pushbuffer port as Xbox 360 behavior.

For any target, establish from its actual binary and runtime:

- which guest instructions, exceptions, ABI details and synchronization primitives are emitted or used;
- whether the renderer consumes PM4, D3D-level calls, or another engine boundary;
- how guest jobs, events, GPU waits and vblank pacing interact with Horizon scheduling;
- which Xenos render targets, resolves, depth behavior, shaders and state transitions are used;
- which storage, audio, input, video and online services must be emulated, stubbed or intentionally unsupported.

## Prior-project evidence

The supplied survey includes ReXGlue projects (SA2, Futari and related titles), XenonRecomp projects (UnleashedRecomp and MarathonRecomp), and PM4/Mesa projects (The Simpsons Game and Def Jam). The closest to the case study are the Prototype and Prototype 2 ports (2026-10): ReXGlue with the masseffect-nx SDK, a project8-lineage native renderer and the same mesa-switch commit. Both are playable, and they add the fourth CPU core, console-trained PGO and renderer hashing changes. Their approaches are useful comparison points, not a single prescribed architecture. The [research database](../../08-prior-port-research-database.md) classifies the cross-project CPU, renderer, GPU, scheduling and testing findings with their reported scope and caveats.

## Testing without the console

ReXGlue's runtime also builds for desktop systems, macOS included, so the same translated game can run unattended on the development machine:

- **Reach the test scene from a hook.** The guest engine usually has its own console command queue; a hook that queues one command (a map load) after a delay takes the game to the scene with no input.
- **Count PM4 work per pass.** A one-frame renderer trace that logs draws per render target, depth state and which skip rule fired shows exactly what a renderer change removed. Compare two runs that differ in one setting.
- **Check engine settings in the log.** Settings written as console commands can be silently ignored by a build, so log the applied list. Confirm in a frame trace that a setting changes the draws before trusting it.
- **Limits.** The desktop GPU, driver and CPU say nothing about Xenos-translation cost on the Switch, and an idle scene has none of the effects or crowds that load a busy one.

The [case study's build process](black-ops/build-process.md) has the exact commands.

## Case study

The case-study title remains an active project-specific investigation. Console profiles, renderer measurements, engine-setting studies, proposed experiments, and current status live in [its research folder](black-ops/); the original supplied reports are preserved in its `source-notes/` directory. Do not promote its pending findings into platform-wide recommendations without independent evidence.
