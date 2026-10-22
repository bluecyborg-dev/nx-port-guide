# Prior port research database

This file collects reusable findings from earlier Switch port and recompilation projects. It is a reference database, not a task log: title-specific progress, open bugs and proposed experiments belong in each game's folder. The evidence below combines the supplied local research notes with a targeted review of the upstream Xbox-to-Switch port repositories and SwitchPorts directory on 2026-10-17.

## How to use this database

- Treat every number as tied to its named project, build, scene, device mode and measurement method. Do not compare unlike runs or assume a gain will transfer to another title.
- A measured result is evidence for a hypothesis, not a promise. Repeat it on the target Switch and preserve an unchanged baseline.
- Keep risky optimizations optional until output and behavior match. A fast-path self-check does not replace full-frame comparison.
- Game-specific conclusions and current work items are linked from their platform/game research pages. This document keeps the cross-project lessons.

### Applicability test before reusing a result

For each finding, classify what it depends on before copying the implementation:

| Dependency | Questions to answer |
|---|---|
| Switch target runtime | Does it depend on Horizon scheduling, libnx services, NVK, storage, or display pacing? See [the Switch runtime guide](02-how-the-switch-works.md) and [Switch performance guide](03-switch-performance.md). |
| Guest CPU/recompiler | Does it depend on x86, PowerPC, instruction semantics, register state, or generated-code shape? Keep it under the matching source-platform folder. |
| Guest GPU/command stream | Does it depend on NV2A, Xenos/PM4, D3D state, tiling, EDRAM, or a particular renderer boundary? Keep the mechanism with that platform/game. |
| Engine/game behavior | Does it rely on a named dvar, map, level, asset format, or title-owned loading rule? Treat it as a case-study result only. |
| Measurement setup | Were the clocks, game scene, build, and profiling method comparable? Keep the raw run details with the claim. |

An idea can transfer while its numeric result does not. For example, measuring allocation cost before
adding a pool is reusable; one project's slab size is not. State caching is reusable; the cache invalidation
rules depend on the guest command stream. A lower-resolution target is an experiment; the guest GPU's
tile/resolve constraints determine safe dimensions.

For research notes, label each statement as a direct measurement, implementation observation, external
claim summarized from supplied files, inference, or proposed experiment. Include the source path and date,
and preserve contradictory results rather than averaging them into a false consensus.

## Projects and porting approaches

| Project family | Port / rendering approach in the supplied notes | Useful research area |
|---|---|---|
| ReXGlue Xbox 360 recompilations | SA2, Mushihimesama Futari, Banjo-Kazooie: Nuts & Bolts, Peggle 2 and Sonic Transformed; native Vulkan renderer lineage | CPU translation, wait behavior, draw submission, lockstep visual testing |
| ReXGlue + masseffect-nx SDK + project8-lineage renderer | [Prototype](https://github.com/fungalspores/prototype-switch) and [Prototype 2](https://github.com/penettrator/prototype2-nx) (2026-10, both playable); same mesa-switch commit as BO1 | Fourth CPU core, main-thread priority, PGO on console, renderer hashing, GPU bandwidth, release packaging |
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
| Native replacements for guest `memcpy`/`memset` or hot routines | Futari reported 40 million verified calls for native memory routines; SA2 reported an 8.5% single-function cost removal from an inlined math routine; Prototype 2 rewrote its job workers' object-culling routine (5.8% of a core) with NEON, and the first 256 calls run both versions and compare every result | Guard replacements by known code identity where applicable and compare exact results. Only optimize measured hot functions. A mismatch should switch the native path off automatically. |
| PGO trained on the console | Prototype 2 0.2.0 ships 222 GCC profiles covering generated guest code and native code. Its instrumented build dumps and resets the gcov counters to `sdmc:/pgo` every 3 minutes from a helper thread, because the game is closed with HOME and never flushes them at exit | Arc counts only (`-fno-profile-values`); same compiler (GCC 15.2 there) and build-directory layout for GEN and USE. Dumps stall the game, so never benchmark an instrumented build. No FPS gain was published for it. |
| Register localization made safe around setjmp | Prototype 2 uses `cr`/`non_volatile`/`xer`/`ctr_as_local` and keeps registers in the context (`share_registers`) only for guest setjmp, a fiber switch and a non-ABI helper pair found with masseffect-nx's `read_before_write.py` | Ports that skipped `non_volatile_as_local` because of setjmp (BO1's `Com_Error`) can use the same per-function exception. |
| Per-address locks for lwarx/stwcx. regions | The Prototype port replaced one process-wide mutex with per-address locks | Check how the SDK in use serializes reservations before assuming it is cheap. |

### Waits and scheduling

- Futari reported a D3D GPU-wait spin reduction from 96% to 8% busy time on a PC test and from 96% to 59% on its console test after sleeping until the ring advanced. This illustrates the value of finding guest/renderer waits that occupy CPU while another thread must run.
- Futari also attributed 78.6% of a game thread's busy time to a frame-pacing spin. Its SA2 notes report a relaxed-vsync PC test rising from 30 to 54.6 FPS with 18 ms of ring work. Neither number predicts another game's timing or pacing correctness.
- MarathonRecomp notes describe owner-word waits, events that avoid wakeups when no thread waits, brief spin-then-sleep behavior, moving workers away from a core assigned to the game thread, and sleeping to a frame deadline instead of yield-spinning. In one CPU-bound spot, four rounds reportedly moved 47 to 60 FPS and game-thread time from 20.2 to 16.1 ms.
- Futari traced one-second hitches to periodic logging performed under a sink mutex during SD writes. Keep periodic logging off hot threads and measure storage stalls.
- SA2 notes identify `thread_local` access as a function call on Switch; avoid it in frequently executed draw paths unless measurement shows otherwise.
- **The fourth core.** The homebrew loader's NPDM allows cores 0-2. Prototype 2's HOME-menu NSP forwarder sets `highest_cpu_id` 3 in both permission tables (ACID and ACI0); its `tools/patch_hbl_core3.py` makes the same 2-byte change to one verified `hbl.nsp`. Its thread masks then put guest threads on cores 0-3, the main thread on 1-3, the audio engine on core 3 only and host threads on 1-2. System services still use core 3, so the core is shared. A forwarder needs the user's own `prod.keys`; a patched `hbl.nsp` replaces a system file.
- **The game's main thread among job workers.** Prototype 2's profile showed its main loop getting about 70% of a core while about 16 job workers shared its priority 0x3B on three busy cores. It raises that one thread to 0x3A (still below the host present and audio threads at 0x2C/0x2B) and pins it to core 0, away from host threads. At 0x3A the thread is no longer time-sliced with the workers, so a yield-based wait in it may not let them run on that core; check its wait loops first.
- **The guest's flip schedule.** Prototype 2's D3D scheduled each flip for "last flip + present interval", or for the next vblank when the swap came late, so 35-60 ms of work became 50/67/83 ms frames. It now flips at the current vblank and lets the host presenter (Vulkan FIFO) pace the screen, keeping a minimum of 2 vblanks.
- **Critical sections held across a wait on the render thread.** In Prototype, the main thread waited for the render thread to drain a full command ring while owning critical sections the render thread needed. On the Xbox 360 the ring never filled at those points; on the Switch the render thread is several times slower. The fix releases the waiting thread's critical sections during the drain and retakes them in order. Look for this pattern when a port hangs in transitions or after menus.

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
| Hash guest buffers less often | Prototype 2's renderer: buffers that change every frame are copied with no hash first; ranges of 16 KB or more that stopped changing are checked by samples plus a full hash every 16 frames; ranges smaller than that but over 1 KB, unchanged for 300 frames, are hashed every 4th frame; ranges on pages its skinning code wrote in the last 3 frames get a new identity instead of a hash | A change can be seen up to N-1 frames late; it kept an exact-check switch for a clothing-twitch diagnostic. No timing was published. |

The Mass Effect measurements separate NVK command-recording cost from renderer work. They show why optimizing a host Vulkan call alone may not address guest packet decoding, state translation, hashing or copying.

## GPU, driver and render-target findings

- **Mesa patches are revision-specific.** The Simpsons/Def Jam work described a direct-draw path for zero draw parameters and cached memory queries against the same Mesa commit used by the surveyed project. A separate UBO descriptor patch was reported to move project8 Free Skate from 2–3 to 7–8 FPS without device loss. Check whether a patch applies to the current driver and workload before adopting it.
- **ZCULL and image usage are coupled.** NFSMW notes reported about 2 ms from enabling a ZCULL path for depth images without `TRANSFER_DST`; copy requirements can prevent that usage configuration. Verify image transitions, copies and correctness together.
- **Passes, barriers and resolves can dominate.** Unleashed notes reported reducing barrier batches from 60 to 37 per frame. Another reported change handed off cleared/overwritten targets instead of copying them, replacing 24 of 26 copies; resolves had been measured at 5.8 ms in that scene. These depend on lifetime and dependency analysis.
- **Depth-only rendering and output pruning** were used in Unleashed when pixel shaders had no effect or outputs were unused. Preserve required depth, stencil, blending and shader side effects.
- **Queue submission has fixed host cost.** NFSMW notes associated two submissions per frame with 3.6 ms of GPU idle and reported a one-submit path. Submission count is workload/driver-specific; batching may increase latency or complicate synchronization.
- **Resolution scaling helps pixel-bound scenes.** SA2 notes describe 720p handheld and 1080p docked modes. SH2/3/4 notes describe screen-sized render targets with fewer rendered lines and later scaling. These examples support testing scale as a GPU-bound lever, not treating resolution as a universal speedup.
- **Prewarming reduces first-use shader stutter.** The SH2-family notes report preparing shader programs ahead of first draw. GTA III NX's `full_preload` option was described as loading textures before play to reduce stutter; this trades memory/startup for streaming behavior.
- **Bandwidth and engine switches on GM20B.** Prototype 2's renderer reports that every switch between drawing and compute makes this GPU go idle, so it can do colour resolves and its shadow-atlas conversion with fragment passes instead of compute dispatches. It also offers 32-bit depth instead of 32-bit float with stencil (which NVIDIA keeps at 64 bits per pixel), the HDR scene as R11G11B10 instead of RGBA16F, and caps or alternates the far shadow cascades. Settings, not published measurements; test each against the title's formats and stencil use.
- **Do not shrink working pools blindly.** The GTA V toolbox notes said smaller memory/model-info pools caused more stutter through frequent unload/reload; density, entity limits, population update rates and cull ranges were the more useful CPU controls, with population density reported as its largest setting knob (~11% from 0 to 10 in that toolbox's measurement).

### NV2A pushbuffer state and clip-space pitfalls

- **Keep method shadows class/subchannel-specific.** In [NFSU2 Switch](https://github.com/antoxa2584x/nfsu2-sw/blob/main/CLAUDE.md), a black loading screen came from writing methods from every pushbuffer subchannel into the 3D register shadow. An NV062 2D blit method at offset `0x0308` aliases NV097 `SET_CULL_FACE_ENABLE`; its value enabled front-face culling and removed all CW loading quads. The fix was to update the 3D shadow only for its 3D subchannel (subchannel 0 in that port). Preserve the subchannel in traces and check the class before interpreting overlapping method offsets.
- **Check winding at each graphics-backend boundary.** The same NFSU2 port had a Vulkan front-face inversion that removed front-facing menu-car, wall and object draws. Its GL and Vulkan paths need different orientation handling because their framebuffer conventions differ. Compare a deterministic frame and per-draw cull state when one backend loses geometry; do not assume matching API enum values imply matching on-screen winding.
- **Do not clamp clip-space z per vertex across the eye plane.** The [NFSU2](https://github.com/antoxa2584x/nfsu2-sw/blob/main/CLAUDE.md) and [Carbon](https://github.com/antoxa2584x/nfsc-sw/blob/main/CLAUDE.md) notes describe large floor/reflection triangles near or behind the eye whose interpolated depth was bent by per-vertex clamping. Their Vulkan path uses hardware depth clamp when available and avoids the invalid behind-eye clamp otherwise. Compare depth at the same pixels across back ends, and treat this as a targeted eye-plane case rather than a blanket fix for vertices far outside the viewport.
- **A black final image can originate in different pipeline stages.** [Burnout 3's bring-up log](https://github.com/antoxa2584x/burnout3-sw/blob/switch/docs/boot.md) records early black/unlit output from missing combiners, texture stages, fog and alpha test, plus a later missing scene composite caused by an ignored immediate-mode attribute. First determine whether world geometry rasterizes; shader/composite fixes do not explain vertices whose generated positions are outside the view.
- **Check for Vulkan render feedback loops.** [FlatOut 2's notes](https://github.com/antoxa2584x/flatout2-sw/blob/main/CLAUDE.md) traced bloom shimmer to a draw sampling the scene image while rendering into that same image. Its renderer snapshots the source before the draw. For a black or corrupted composite, compare the sampled image address with the active color target; copy/snapshot only when the trace proves a same-image read/write.

The supplied research contains no equivalent controlled Vulkan-versus-deko3d benchmark. It records projects using or considering deko3d and claims of expected gains, but these are not comparative measurements. Benchmark equivalent command streams and GPU work before selecting an API rewrite.

## Port-architecture observations

- A translated console title can use a guest-level API reconstruction, a D3D-level renderer, a PM4/pushbuffer translator, or a more direct engine-renderer interception. The notes contrast ReXGlue/PM4 with XenonRecomp/D3D-level rendering, and note that Ogre's D3D9 render system may expose HLSL at a different interception boundary.
- PC ports that submit through OpenGL can still be CPU-bound in driver submission: the Jak/OpenGOAL notes measured 45–67 ms of GL submission in the cited frame. API labels alone do not tell which layer owns the bottleneck.
- SH2/3/4 notes describe OpenGL 4.3 core as their D3D8 target and default guest threads on one core because of an audio bug that could loop music when threads were distributed. Scheduling changes must include audio behavior.
- The Prototype ports ship HOME-menu NSP forwarders (Forwarder-Mod, built with the user's own keys) and `rex-start`, a small libnx launcher that shows the game for 5 seconds and offers a settings screen writing straight into the game's TOML. Their SDK fixes worth checking in any ReXGlue port: `XFile::Write`/`SetLength` returned the size cached at open, so saves looked corrupt; the console signature stubs (`XeKeysConsolePrivateKeySign`, `...SignatureVerification`) returned "invalid"; and a temperature-sensor session left open for the whole game coincided with Horizon OC's clock service crashing under load. Reported FPS (25-30 in gameplay) is on an overclocked OLED (CPU 1785 / GPU 768-1152 / RAM 2333 MHz), "lower at stock clocks".
- GTA V Switch toolbox notes describe a PC-build port tuned through `settings.xml` (DX version, MSAA, shadows, reflections), not an Xbox 360 recompiler. Similar visual settings can apply across port types, while CPU/renderer findings may not.

## October 2026 port scan (code read, 2026-10-22)

Source code read, not only the documents, for: [F.E.A.R. NX](https://github.com/infriker/fear-nx), [ds2-nx](https://github.com/VasilkovND/ds2-nx) and [dantes-inferno-nx](https://github.com/VasilkovND/dantes-inferno-nx), [usf4-nx](https://github.com/Thorhax/usf4-nx), [skate3recomp_nx](https://github.com/SubterraneanHomesickAyan/skate3recomp_nx) and [project8recomp_nx](https://github.com/SubterraneanHomesickAyan/project8recomp_nx) (one shared platform layer), [Bakugan-NX](https://github.com/Wiredlainn-afk/Bakugan-NX), [LostOdysseyRecomp](https://github.com/Ad0ky/LostOdysseyRecomp) (PC/Android/macOS, not Switch), and the newer ChanseyIsTheBest-family ports. [BO2-Requiem-NX](https://github.com/HDErick/BO2-Requiem-NX) has no source yet (planned for November 2026). Numbers are each project's own, at the clocks it states.

**Direction of the scene.** Several ports have left Vulkan/NVK for deko3d, the Switch's own low-level API:
- Dead Space 2 and Dante's Inferno use a PM4 ring reader with the universal deko3d renderer of `nx-mod/rexglue-sdk-nx`, started in its Halo 3 port. Their shaders are translated offline (Xenos microcode → HLSL → SPIR-V → precompiled DKSH), so the console never compiles a shader.
- Bakugan and ChanseyIsTheBest's ports draw through reNut's D3D-level renderer on deko3d.
- Bakugan studied deko3d for its PM4 path and concluded that it fixes CPU cost per draw and pipeline stalls, not shader or EDRAM cost. So it pays most where the renderer thread and the driver are the limit.
- BO2 Requiem takes yet another route: the Ghidra decompilation of BO2's PC executables, compiled for AArch64 with 32-bit pointers (ILP32), with D3D11 translated to Vulkan by DXVK.

| Finding | Port and evidence | Conditions |
|---|---|---|
| Fixed flush-to-zero on Switch | Dead Space 2: recompiled code toggles FPCR.FZ at every scalar/VMX border (45,000 sites, 7,400 unconditional); each `msr fpcr` serializes the A57 pipeline. The SDK now keeps FZ on and makes the toggles no-ops | Scalar results below 1.2e-38 become 0. A runtime change in the SDK's FPSCR wrapper, not a codegen option |
| Yield backoff | Dead Space 2: 40,000-100,000 yields per second. A thread that yields again within 20 µs, more than 16 times in a row, sleeps 100 µs per call instead | Isolated yields keep their behavior |
| XMA kick without the context lock | Dead Space 2: the mixer's kick took the decoder's per-context lock, held for a whole decode, just to set a flag, and signaled the worker's event about 1,500 times per second. Now it signals only when the worker is about to sleep | Audio thread lock time fell from 18 % to 6 % |
| No-op guest priority calls | Dead Space 2: its job workers raise and drop their priority around every job. With `ignore_thread_priorities` the host priority never changes, yet each call took the object-table lock several times. The call now just returns success | Hook the title's own priority wrapper |
| Clamp the guest clock after sleep or HOME | Dead Space 2: the first clock read after a suspend carried the whole nap as one frame, and its audio crashed. Gaps are capped at 1/4 s | — |
| `mtmsrd` regions per thread | Dead Space 2: the codegen's process-wide recursive mutex serialized its job scheduler (main thread from ~76 % to ~97 % useful work once removed). USF4 hit the same problem | `skip_msr` removes the mutex altogether. Check it on any ReXGlue port slow with idle cores |
| Object-table lookups without the global lock | Skate 3 / Project 8: every guest handle operation took the SDK's global recursive mutex, and the main thread spent ~20 % of its time queued there. Lookups now use a table-local reader gate (two atomic operations uncontended) | Mutations keep the global lock, taken in the order global → gate |
| Barging futex lock for the wait coordinator | Skate 3: Horizon mutexes hand ownership straight to the next waiter, creating convoys. The main thread spent a third of its sampled time in `ArbitrateLock` inside event signaling. The fix is a Drepper-style lock over `svcWaitForAddress` / `svcSignalToAddress`, waking threads after the unlock | Platform-layer rewrite |
| Thread-caching malloc | Skate 3: newlib's single-lock allocator was ~5 % of all CPU samples and half of all mutex blocking, with the render, GPU and audio threads the main victims. The fix is a tcmalloc-like allocator via `--wrap`, with a 64 MiB emergency reserve | Large blocks still go to newlib |
| Job completion epoch | Skate 3: the main thread's "wait for job group" loop slept a fixed 30-100 µs and rescanned. It now waits on a word that every completed job bumps (with the same timeout), and that wait was ~14 % of the main thread in the city. USF4 did the same with a condition variable (main thread 88 % → 81 %) | Keep a timeout as the safety net |
| D3D GPU-wait sleep | USF4 (from sa2-nx): the XDK wait iteration sleeps until the ring thread reports progress, at most 1 ms. Main thread 70 % → 55 % | Same XDK function body checked by hash |
| Guest memcpy as host memcpy | USF4: the game's two CRT copy routines were ~13 % of the main thread, then 3.5 %. Code checked by FNV hash; overlapping ranges go to the original | An audit mode compared 14.5 M copies with 0 differences |
| Vertex ranges trusted between checks | USF4: a 24 MB ring of kept vertex copies, with unchanged ranges trusted for 2, 4 then 8 frames. Fingerprint (CRC32 on Switch) went from 15.8 % to 5.1 % of the ring thread | An audit mode found 0 changes in ~80,000 trusted ranges |
| Fewer ring-thread allocations | USF4: kept vectors per nesting level and a linear-probing table instead of a growing map. malloc/mutex had been ~7 % of the ring thread | — |
| Merge tile copies | USF4: NVK records each colour `vkCmdCopyImage` region through `vk_meta`, and the meta objects are destroyed at pool reset (~12 % of the ring thread). Merging neighbouring tiles took regions per call from 9.4 to 2.5. Bakugan: do colour copies as fragment passes | — |
| Game frame limiter sleeps | USF4 and Bakugan: the limiter's `Sleep(0)` spin sleeps for the missing time minus 1 ms (USF4 main thread 81 % → 54 %) | Keep the final spin for accuracy |
| Video recording off | USF4: `appletSetGamePlayRecordingState(false)` at start. Holding Capture had crashed the game, with application memory at 3,281 of 3,285 MB | Screenshots still work |
| Faster save opens | USF4: `FILE_OVERWRITE_IF` truncates in place instead of stat + delete + create (18-31 ms → 9-23 ms per file) | — |
| Codegen measured on Switch | Bakugan: set 1 (cr/ctr/xer/reserved locals, `skip_msr`, `dform_disp_split`, `fctiwz_inline`) plus set 2 (`non_volatile_as_local`, `elide_lr_stores`, `fp_moves_modeless`) took the main thread from 65-72 % to 50-59 %. Hot-function ordering saved ~4 points; LTO gained nothing | Movie decoders broke on ARM only with `non_volatile_as_local`; test cutscenes on the console. **Judge CPU changes by thread CPU %, not FPS, when the game is at a frame cap** |
| Dynamic render scale | Bakugan: 1.5 s windows of GPU busy fraction, a pending gate, step-up on the p90 of 8 windows, and floors of 450p handheld / 720p docked. Heavy arenas hold 29.6 FPS at 540-630p | Its D3D-level renderer can resize targets; a PM4 translator with EDRAM-sized targets cannot easily |
| Shared resolve destinations, 2× anisotropy | F.E.A.R.: tiled resolves write straight into one shared destination. Its release ships D24 depth, no MSAA and 2× anisotropic filtering, and reports ~30 FPS at a 614 MHz GPU | — |
| BO2 Requiem architecture (binary inspection of Beta 1) | The release NRO is a 64-bit libnx host. Its RomFS holds `t6zm_guest.elf`, a 32-bit AArch64 (ILP32) PIE built with devkitA64 GCC 16.1.0 and **not stripped** (25,260 function symbols). The host maps the guest's code with `svcMapProcessCodeMemory`, keeps guest memory below 4 GB, and provides Windows API stand-ins (threads, critical sections, waits, `VirtualAlloc`, files, sockets) through ~96 `nxh_*` calls. D3D11 calls cross an "NxCom" COM bridge to DXVK-Sarek (config `dxvk-sarek.conf`) on NaGa's NVK. It runs on the Steam PC game files | A third route besides recompiling the 360 executable: a native build of a PC decompilation |
| Yield backoff, again (BO2 Requiem) | The guest's `yield_cpu` keeps a per-thread timestamp and counter in its TLS: when 32 yields in a row each come less than 1 ms (19,200 timer ticks) after the previous one, the 32nd sleeps 200 µs (`nxh_sleep_ns(200000)`) and the count restarts; otherwise it yields. `Sleep(0)` and `SleepEx(0)` take the same path | The same remedy Dead Space 2 reached independently |
| CPU boost keyed on the engine's state (BO2 Requiem) | `CL_Client_Frame` calls `nx_load_boost` whenever the client's connection state changes: boost on for states 1-9 (connecting, loading), off in game | Boost mode drops the GPU clock, so only during loading |
| BO2 Requiem status (README and Beta 1 notes, 2026-10-22) | Zombies and multiplayer only (the campaign is not fully decompiled). Source release planned for early November 2026. "Playable at stock speeds", better overclocked; first runs show pop-in while DXVK compiles shaders. Credits: a community decompilation of BO2's PC executables as the base, NaGa for Mesa and the DXVK work | Its host would be the reusable part for other PC-decompilation ports |
| Open-source PC reimplementations of the same engine family | [KisakCOD](https://github.com/SwagSoftware/KisakCOD) (CoD4) and [KisakBlack](https://github.com/SwagSoftware/KisakBlack) (Black Ops multiplayer executable, GPL-3.0): fully buildable, Windows/MSVC/x86/D3D9 only. KisakCOD has several community WebAssembly ports, and a closed-source WebAssembly build of a KisakBlack-based Zombies tree runs in browsers: both engines survive 32-bit-pointer, non-x86, non-D3D builds | No Switch port of either was found (GitHub search, 2026-10-22) |
| Lock-wait records (BO2 Requiem; this and the rows below come from a REA + Ghidra decompilation of the guest ELF, 2026-10-22) | `EnterCriticalSection` tries the lock first and, only when that fails, writes a 32-byte record into a 64-slot table indexed by thread: thread, lock, current owner, caller's return address, lock word, start tick and kind (1 = critical section, 2 = wait with a timeout). `WaitForMultipleObjects` records its waits the same way. The host reads the table (`nxh_diag_waits`) and has a watchdog that logs "stuck for 15 s at" and ends the process | No cost when the lock is free. It names the owner of the lock a stalled thread waits on, which sampled stacks do not |
| Windows priorities on Horizon (BO2 Requiem) | Every thread starts at 0x2C with a host core argument of -1 and a 1 MB stack (at least 256 KB when a size is given). `SetThreadPriority` maps time-critical → 0x28, highest → 0x2A, above normal → 0x2B, normal → 0x2C, below normal → 0x2D, lowest → 0x2E, idle → 0x3B. The sound mixer thread is time-critical (0x28), the decoder above normal (0x2B) | These priorities are cooperative. That is workable for native code that idles through sleeps and the yield backoff; recompiled code that spin-waits needs the time-sliced 0x3B instead (see [How the Switch works](02-how-the-switch-works.md)) |
| Waiting and idling (BO2 Requiem) | One global wait lock; each waiting thread links a node holding its own condition variable into every object it waits on, so `SetEvent` wakes only that object's waiters and an auto-reset release wakes one. The render thread's idle loop polls its events with `Sleep(0)`, so it relies on the yield backoff. Left as in the PC game: `jqWorkerSleep` (waits on the new-job event with a 1 ms timeout), `Sys_AssistAndWaitWorkerCmd` (helps with jobs, then `Sleep(0)`), `R_WaitEndTime` and the `com_maxfps` limiter (`Sleep(1)` loops) | — |
| Present after releasing the frame (BO2 Requiem) | `RB_RenderThreadFrameArm` executes the frame's commands, then sets `renderCompletedEvent` and `backendEvent`, and only then calls `RB_EndFrame` (the present). The main thread can start the next frame while the present runs | Whether the PC game orders it the same way was not checked |
| Dynamic buffer uploads (BO2 Requiem) | `Map` hands the engine a CPU-side copy. The engine's writers (`R_SetVertexData`, `RB_UnmapMeshData`) report each written range through `nx_d3d_buffer_written`, and `Unmap` sends only the lowest-to-highest written range to DXVK. Constant buffers go as one write-discard call carrying a pointer and a size (`R_FlushDirtyConstantBuffers`). Counters: KB uploaded, uploads over 256 KB, largest upload | — |
| Guest heap (BO2 Requiem) | 80 size classes (16-byte steps up to 256 bytes, then steps of about 1/8), one lock, refills of 256 KB, blocks over 64 KB straight from the page arena (zeroed on free) | — |
| Host settings and diagnostics (BO2 Requiem) | DXVK config: `dxgi.maxFrameLatency=1; d3d11.maxFrameLatency=1`, 2 compiler threads, asynchronous shader compilation, `d3d11.enableContextLock=True`. Mesa's shader cache on the SD card capped at 256 MB. Handheld GPU clock raised through clkrst or pcv; CPU boost for "loading", "loaded" and "focus". A 50 Hz pc/lr sampler for 120 s (`nx-profile.txt`). A hitch log (`BO2_HITCH_LOG`) that times 11 steps of `Com_Frame` and reports frames of 81 ms to 2 s, at most 100 times. Same Mesa as other ports (NaGa's 26.2.3) | — |
| Faster block compare for vertex matching | Lost Odyssey (ARM64 Android lessons): `memcmp` was 7 % of its render thread. A 64-byte sampled block compare was 9-15× faster in isolation (vertex match 2.28 → 0.49 ms) but FPS rose only 0.6 % because the GPU was the limit | Measure the whole frame |

## Failed, conditional or easily misread results

- Dynamic blend state in Futari and deferred recording in Futari/SA2 produced pixel differences in tested configurations. Mass Effect's deferred-recording design had different command ordering/semantics; do not transfer its success claim without checking the target implementation.
- Disabling a game subsystem can leave shaders reading stale data. Reducing its workload may be safer, but must still be validated.
- Texture-fetch exponent bias was identified as necessary for FXAA 360 averaging in the supplied notes; omitting it caused edge flicker. Resolve bias and texture-fetch bias are distinct checks.
- Sampling reports without frame pointers may have unreliable call chains; use the measurement the profiler supports reliably.
- Higher priority, affinity changes and extra pipeline workers can help one workload and starve another. A Banjo report attributed 23% of ring-thread samples to a newlib allocation lock with multiple prewarm workers; more workers were not automatically better.
- Claims that NVK always loses to WebGPU or that deko3d is “2x+” were project expectations in the notes, not controlled cross-API measurements.

## Source trail

This database summarizes the supplied `OTHER_PORTS_RESEARCH.md`, its cited local project notes, and the upstream repositories linked above. The [NFSU1 notes](https://github.com/antoxa2584x/nfsu1-sw/blob/main/CLAUDE.md) also document title-specific projection/frustum adjustments for aspect-ratio handling; treat those as a later display-tuning example, not a general black-screen fix. The full original survey remains archived under [Xbox 360 / Black Ops source notes](platforms/xbox-360/black-ops/source-notes/OTHER_PORTS_RESEARCH.md). BO1-only profiler findings, dvar experiments and pending tests remain in the [Black Ops research folder](platforms/xbox-360/black-ops/); Conker runtime traces and title-specific renderer findings belong in [Conker's folder](platforms/xbox/conker/).
