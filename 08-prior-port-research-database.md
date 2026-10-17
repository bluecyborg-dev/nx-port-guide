# Prior port research database

This file collects reusable findings from earlier Switch port and recompilation projects. It is a reference database, not a task log: title-specific progress, open bugs and proposed experiments belong in each game's folder. The evidence below comes from the supplied local research notes, read on 2026-10-17; no new web research was used.

## How to use this database

- Treat every number as tied to its named project, build, scene, device mode and measurement method. Do not compare unlike runs or assume a gain will transfer to another title.
- A measured result is evidence for a hypothesis, not a promise. Repeat it on the target Switch and preserve an unchanged baseline.
- Keep risky optimizations optional until output and behavior match. A fast-path self-check does not replace full-frame comparison.
- Game-specific conclusions and current work items are linked from their platform/game research pages. This document keeps the cross-project lessons.

## Projects and porting approaches

| Project family | Port / rendering approach in the supplied notes | Useful research area |
|---|---|---|
| ReXGlue Xbox 360 recompilations | SA2, Mushihimesama Futari, Banjo-Kazooie: Nuts & Bolts, Peggle 2 and Sonic Transformed; native Vulkan renderer lineage | CPU translation, wait behavior, draw submission, lockstep visual testing |
| XenonRecomp | Sonic Unleashed and MarathonRecomp; D3D-level renderer rather than PM4 translation | CPU scheduling, render passes, resolves, shader work and optimization tracking |
| Xenos PM4 / Mesa NVK | Simpsons, Def Jam and project8-sw lineage | Driver-side changes and their interaction with the exact Mesa revision |
| PC x86 lifted ports | Silent Hill 2/3/4 using D3D8-on-OpenGL; other OpenGL/GLES ports | Resolution targets, shader prewarming, audio/thread constraints and CPU submission cost |
| Decompilation / source ports | OpenGOAL/Jak, OutRun 2006 and assorted source/Android ports | Alternative renderer/API boundaries and the need to profile the host submission layer |
| Other engines / APIs | Twilight Princess and Melee experiments with WebGPU; Ogre-based Torchlight interception | Renderer interception choices; WebGPU/NVK and deko3d claims require direct equivalent benchmarks |

The source survey also listed Rayman Origins Recompiled (ReXGlue with a D3D-level renderer, no Switch build in the notes), GTA III NX, 0 A.D. Switch, Smash Hit NX, Sonic & All-Stars Racing Transformed NX, DBT-NX, openPREYswitch, openmw-nx, GZDoom-NX and AvP Gold. These entries provided limited performance documentation in the supplied material; their existence alone is not performance evidence.

## Reusable measurement and correctness methods

### Make optimization state observable

The SA2-derived method in the notes uses an individual setting for each optimization, CPU/non-CPU group switches, a startup report of enabled settings, and periodic use counters. Missing config keys default to the safe established behavior so old configuration files do not silently disable features. Keep each experiment independently reversible.

For a fast path, run the old and new implementations side by side for early uses or in a verification mode. Compare their outputs and report a mismatch count. For mutable guest memory, take one snapshot and compare against that snapshot; reading it twice can create false differences when the game changes the buffer.

### Compare deterministic output

Use lockstep runs with controlled presentation/vblank timing and compare per-frame image hashes against a known-good build. Menus, fades and repeatable scenes can be more deterministic than gameplay. The supplied notes explicitly report optimizations whose internal self-checks passed while rendered frames still differed. Pixel/frame comparisons therefore remain a separate gate.

### Profile the actual console

Desktop and Mac measurements are useful for finding a code path, but they do not predict Switch timing. On-console stack sampling in the supplied method records at roughly 1 ms intervals, symbolizes against the exact archived ELF, and uses PC-based self time for leaf functions. The notes caution that call chains may be unreliable in binaries without frame pointers and after startup; use PC distributions where appropriate. At capped frame rates, compare thread CPU time and wait time rather than FPS alone.

Preserve the exact tested ELF, configuration, title build, scene and device mode alongside each result so samples and symbols remain attributable.

## CPU translation and game-thread findings

| Technique / finding | Reported evidence in the supplied notes | Conditions and risks |
|---|---|---|
| Guest memory without `volatile`, with compiler barriers at polling-loop headers | Mass Effect NX reported roughly 15–20% improvement; a Futari investigation found 172 polling loops compiled to branch-to-self when barriers were absent | Barriers are essential for guest polling semantics. Inspect generated code for unintended self-loops and validate synchronization. |
| Direct calls between translated functions | Futari report counted about 178,000 translated calls | Hooked or replaceable functions may need indirect dispatch; regenerate after hook changes. |
| Avoid FPCR changes around pure floating-point moves | SA2 sampling found 20–26% of game-thread samples immediately after FPCR writes | Depends on generated instruction mix and floating-point semantics. |
| Elide guest link-register saves / localize guest registers | Mass Effect reported about 10% lower main-thread cost for its translation “diet” and about 9% from register localization; Peggle 2 reduced one `r11` context use count from 549,939 to 13,488 | `setjmp`, exception handling and split fragments can require saved state. Use caller analysis and per-title validation. |
| LTO, PGO and hot-function layout | SA2 reported about 15% from PGO in its tested single-player case; SA2/Unleashed also used sampled hot-function ordering | PGO workload and build setup matter. Large translated code can exceed instruction-cache capacity; preserve startup sections and compare actual scenes. |
| Native replacements for guest `memcpy`/`memset` or hot routines | Futari reported 40 million verified calls for native memory routines; SA2 reported an 8.5% single-function cost removal from an inlined math routine | Guard replacements by known code identity where applicable and compare exact results. Only optimize measured hot functions. |

### Waits and scheduling

- Futari reported a D3D GPU-wait spin reduction from 96% to 8% busy time on a PC test and from 96% to 59% on its console test after sleeping until the ring advanced. This illustrates the value of finding guest/renderer waits that occupy CPU while another thread must run.
- Futari also attributed 78.6% of a game thread's busy time to a frame-pacing spin. Its SA2 notes report a relaxed-vsync PC test rising from 30 to 54.6 FPS with 18 ms of ring work. Neither number predicts another game's timing or pacing correctness.
- MarathonRecomp notes describe owner-word waits, events that avoid wakeups when no thread waits, brief spin-then-sleep behavior, moving workers away from a core assigned to the game thread, and sleeping to a frame deadline instead of yield-spinning. In one CPU-bound spot, four rounds reportedly moved 47 to 60 FPS and game-thread time from 20.2 to 16.1 ms.
- Futari traced one-second hitches to periodic logging performed under a sink mutex during SD writes. Keep periodic logging off hot threads and measure storage stalls.
- SA2 notes identify `thread_local` access as a function call on Switch; avoid it in frequently executed draw paths unless measurement shows otherwise.

Wait optimizations can change scheduling and starvation behavior. Confirm that lower-priority producers still run, and measure jitter as well as average throughput.

## Renderer CPU findings

| Technique / finding | Reported evidence in the supplied notes | Conditions and risks |
|---|---|---|
| Replace expensive per-draw fingerprints with A57 CRC32 where suitable | SA2 used a CRC32-based fingerprint mode | Hash collisions and the actual byte volume matter; correctness checks must remain. |
| Avoid formatting and string-key construction in draw diagnostics | Mass Effect-lineage renderers attributed about 8% of ring-thread work to per-draw formatting in cited measurements | Keep instrumentation cheap and preferably compile-time or opt-in. |
| Memoize unchanged render-target setup | Reported as about 6–9% of ring-thread work in cited renderers | Count cache hits/misses and invalidate on every relevant guest-state change. |
| Upload only changed constants and only shader-read registers | Unleashed and SA2 findings; SA2 reported vertex constants re-uploaded on 77% of draws | Confirm shader register use and preserve guest-visible constants. |
| Avoid reading back from uncached upload memory | Futari attributed 91% of its ring work to this behavior in the measured case | Rework conversion/copy paths only after exact byte and timing counters. |
| Deferred Vulkan recording on a worker | Mass Effect reported ring-thread use falling from 62% to 40% of a core; per-draw NVK recording costs were reported around 4–5.5 μs, with renderer work around 16–25 μs per recorded draw before later cuts | Futari and SA2 tests found pixel differences; readbacks can stall. Preserve ordering and validate full output. |
| Dynamic blend-state optimization | Futari reduced reported binds from 160/frame to 7 | It was rejected after 212 of 1,098 frames differed. Bind-count improvement did not establish correctness. |

The Mass Effect measurements separate NVK command-recording cost from renderer work. They show why optimizing a host Vulkan call alone may not address guest packet decoding, state translation, hashing or copying.

## GPU, driver and render-target findings

- **Mesa patches are revision-specific.** The Simpsons/Def Jam work described a direct-draw path for zero draw parameters and cached memory queries against the same Mesa commit used by the surveyed project. A separate UBO descriptor patch was reported to move project8 Free Skate from 2–3 to 7–8 FPS without device loss. Check whether a patch applies to the current driver and workload before adopting it.
- **ZCULL and image usage are coupled.** NFSMW notes reported about 2 ms from enabling a ZCULL path for depth images without `TRANSFER_DST`; copy requirements can prevent that usage configuration. Verify image transitions, copies and correctness together.
- **Passes, barriers and resolves can dominate.** Unleashed notes reported reducing barrier batches from 60 to 37 per frame. Another reported change handed off cleared/overwritten targets instead of copying them, replacing 24 of 26 copies; resolves had been measured at 5.8 ms in that scene. These depend on lifetime and dependency analysis.
- **Depth-only rendering and output pruning** were used in Unleashed when pixel shaders had no effect or outputs were unused. Preserve required depth, stencil, blending and shader side effects.
- **Queue submission has fixed host cost.** NFSMW notes associated two submissions per frame with 3.6 ms of GPU idle and reported a one-submit path. Submission count is workload/driver-specific; batching may increase latency or complicate synchronization.
- **Resolution scaling helps pixel-bound scenes.** SA2 notes describe 720p handheld and 1080p docked modes. SH2/3/4 notes describe screen-sized render targets with fewer rendered lines and later scaling. These examples support testing scale as a GPU-bound lever, not treating resolution as a universal speedup.
- **Prewarming reduces first-use shader stutter.** The SH2-family notes report preparing shader programs ahead of first draw. GTA III NX's `full_preload` option was described as loading textures before play to reduce stutter; this trades memory/startup for streaming behavior.
- **Do not shrink working pools blindly.** The GTA V toolbox notes said smaller memory/model-info pools caused more stutter through frequent unload/reload; density, entity limits, population update rates and cull ranges were the more useful CPU controls, with population density reported as its largest setting knob (~11% from 0 to 10 in that toolbox's measurement).

The supplied research contains no equivalent controlled Vulkan-versus-deko3d benchmark. It records projects using or considering deko3d and claims of expected gains, but these are not comparative measurements. Benchmark equivalent command streams and GPU work before selecting an API rewrite.

## Port-architecture observations

- A translated console title can use a guest-level API reconstruction, a D3D-level renderer, a PM4/pushbuffer translator, or a more direct engine-renderer interception. The notes contrast ReXGlue/PM4 with XenonRecomp/D3D-level rendering, and note that Ogre's D3D9 render system may expose HLSL at a different interception boundary.
- PC ports that submit through OpenGL can still be CPU-bound in driver submission: the Jak/OpenGOAL notes measured 45–67 ms of GL submission in the cited frame. API labels alone do not tell which layer owns the bottleneck.
- SH2/3/4 notes describe OpenGL 4.3 core as their D3D8 target and default guest threads on one core because of an audio bug that could loop music when threads were distributed. Scheduling changes must include audio behavior.
- GTA V Switch toolbox notes describe a PC-build port tuned through `settings.xml` (DX version, MSAA, shadows, reflections), not an Xbox 360 recompiler. Similar visual settings can apply across port types, while CPU/renderer findings may not.

## Failed, conditional or easily misread results

- Dynamic blend state in Futari and deferred recording in Futari/SA2 produced pixel differences in tested configurations. Mass Effect's deferred-recording design had different command ordering/semantics; do not transfer its success claim without checking the target implementation.
- Disabling a game subsystem can leave shaders reading stale data. Reducing its workload may be safer, but must still be validated.
- Texture-fetch exponent bias was identified as necessary for FXAA 360 averaging in the supplied notes; omitting it caused edge flicker. Resolve bias and texture-fetch bias are distinct checks.
- Sampling reports without frame pointers may have unreliable call chains; use the measurement the profiler supports reliably.
- Higher priority, affinity changes and extra pipeline workers can help one workload and starve another. A Banjo report attributed 23% of ring-thread samples to a newlib allocation lock with multiple prewarm workers; more workers were not automatically better.
- Claims that NVK always loses to WebGPU or that deko3d is “2x+” were project expectations in the notes, not controlled cross-API measurements.

## Source trail

This database summarizes the supplied `OTHER_PORTS_RESEARCH.md` and its cited local project notes. The full original survey remains archived under [Xbox 360 / Black Ops source notes](platforms/xbox-360/black-ops/source-notes/OTHER_PORTS_RESEARCH.md). BO1-only profiler findings, dvar experiments and pending tests remain in the [Black Ops research folder](platforms/xbox-360/black-ops/); Conker runtime traces and title-specific renderer findings belong in [Conker's folder](platforms/xbox/conker/).
