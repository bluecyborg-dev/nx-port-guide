# Xbox 360 port research

This page is a platform-level orientation for Xbox 360 titles. Cross-project measurements and reusable methods are indexed in the [prior port research database](../../08-prior-port-research-database.md). The supplied Xbox 360 work is primarily Black Ops (2010); its measurements, unfinished experiments and title-specific decisions stay in the [Black Ops folder](black-ops/).

## Platform and porting boundaries

Xbox 360 titles use a PowerPC-derived Xenon CPU and Xenos GPU. A Switch homebrew port may statically recompile guest PPC code to AArch64 and translate the Xenos command stream (PM4) into a host graphics API, or intercept at a higher D3D-level renderer. These paths have different costs and correctness obligations; do not treat results from an original Xbox NV2A pushbuffer port as Xbox 360 behavior.

For any target, establish from its actual binary and runtime:

- which guest instructions, exceptions, ABI details and synchronization primitives are emitted or used;
- whether the renderer consumes PM4, D3D-level calls, or another engine boundary;
- how guest jobs, events, GPU waits and vblank pacing interact with Horizon scheduling;
- which Xenos render targets, resolves, depth behavior, shaders and state transitions are used;
- which storage, audio, input, video and online services must be emulated, stubbed or intentionally unsupported.

## Prior-project evidence

The supplied survey includes ReXGlue projects (SA2, Futari and related titles), XenonRecomp projects (UnleashedRecomp and MarathonRecomp), and PM4/Mesa projects (The Simpsons Game and Def Jam). Their approaches are useful comparison points, not a single prescribed architecture. The [research database](../../08-prior-port-research-database.md) classifies the cross-project CPU, renderer, GPU, scheduling and testing findings with their reported scope and caveats.

## Black Ops case study

Black Ops remains an active project-specific investigation. Console profiles, renderer measurements, dvar studies, proposed experiments, and current status live in the [Black Ops research folder](black-ops/); the original supplied reports are preserved in that game's `source-notes/` directory. Do not promote its pending findings into platform-wide recommendations without independent evidence.
