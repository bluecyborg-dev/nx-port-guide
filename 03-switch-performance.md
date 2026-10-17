# Switch performance and optimization

Treat each optimization as a measured experiment. The supplied port notes repeatedly show that a change that helps one title, scene or device mode can hurt another. Separate correctness changes from performance changes and leave a runtime switch or easy rollback path for risky experiments.

For prior-project examples and their measurement caveats, see the [prior port research database](08-prior-port-research-database.md). This guide summarizes methods; title-specific conclusions remain with each port.

## Find the limiting stage first

Measure main/guest CPU, renderer CPU, GPU work, storage stalls, synchronization waits and frame pacing separately. A frame pipeline overlaps CPU and GPU work, so do not add stage timings blindly; find which stage is on the critical path for the frames that miss their deadline.

Useful measurements from the supplied notes include per-thread busy/idle time, guest call samples, renderer work per draw, draw and packet counts, texture hashes/uploads, GPU timestamps, slow-frame categories and frame-time histograms. GPU timestamp readings on the cited NVK build were reported about 1.6× low; calibrate against the exact build before comparing them to CPU time.

Use a repeatable scene. A static scene or scripted route is easier to compare than unrelated gameplay rounds. Capture medians and tail behavior: share of frames over 33/50/100 ms, 95th/99th percentile, worst hitch and cadence. At a 20 FPS cap, a 50 ms budget matters; a smoother cadence can feel better than a higher average with frequent missed intervals.

## Common CPU-side wins

- Remove avoidable guest busy-waits; use event-driven waits or bounded sleeps and ensure the producer can run.
- Profile generated code and renderer work before changing code generation. Potential targets include guest register traffic, indirect dispatch, repeated state derivation, repeated hashing/copies and excessive helper calls.
- Cache effective graphics state and emit host state only when it changes. Count requested versus emitted changes to prove the cache is useful.
- Copy only vertex ranges the draw uses, and copy dynamic inputs at submit time if workers would otherwise retain mutable guest pointers.
- Avoid allocations, kernel calls, logging, formatting and expensive lookups in per-draw or per-frame hot paths.
- Consider native replacements only for measured hot routines, and compare their output to the translated function.

Code-generation techniques such as register localization, direct calls, non-volatile guest memory with loop barriers, LTO/PGO and function ordering are workload-specific. Validate guest behavior, branch-to-self polling loops and deterministic output after each change.

## GPU, renderer and memory wins

- Lower internal resolution only when GPU-bound. Pixel count is not GPU time: vertex work, bandwidth, resolves and fixed overhead remain.
- Reduce expensive passes or selected effect-buffer resolution where the game can tolerate it. A setting that removes a system can leave stale shader inputs; verify frame output and game state.
- Track render-target lifetimes, copies/resolves, barriers and attachment load/store behavior. Preserve guest-visible ordering and contents.
- Cache shaders and prewarm known pipelines when cold compilation causes visible hitches. Keep cache keys tied to driver/device and translator versions.
- Suballocate GPU resources from slabs when per-object allocation costs or handle limits are demonstrated. Retire allocations only after their GPU fence completes.
- Cache streaming reads where repeated access is measured; do not shrink working memory blindly if that increases unload/reload churn.

Resolution, ZCULL, prepass skipping, multi-draw, deferred recording, descriptor/push-constant updates and alternative APIs are all experiments. The source notes include cases where a seemingly obvious change increased GPU time, changed pixels, caused input problems or did not improve FPS.

## A/B procedure

1. Record the baseline build, device mode, clocks, scene, settings and frame-time distribution.
2. Change one variable behind a setting; keep the previous path available.
3. Verify correctness first using screenshots, frame hashes or targeted captures, plus input/audio/save checks as relevant.
4. Repeat the same scene on hardware. Do not use desktop or emulator FPS as a Switch performance proxy.
5. Compare CPU/GPU time and frame-time tails, not only averages. Note run-to-run variability.
6. Keep or revert the change based on both measured results and visible/interaction regressions. Update the game logbook with the build and evidence.

## What does not transfer automatically

A retail Switch port's resolution or frame rate does not establish the performance of a homebrew renderer. Xbox 360 PM4 work, original Xbox NV2A work and PC-game ports have different guest workloads and renderer translation costs. Do not assume deko3d will beat NVK, that a newer Mesa version contains a needed patch, or that a particular priority, large-page mapping or multi-draw extension will provide a fixed gain. Benchmark equivalent work on the target hardware.

## Build a useful cost model

Capture the work at boundaries that correspond to independently optimizable stages. For a translated game
with a command-stream renderer, a useful per-frame model includes:

| Stage | Counters and timings that help explain cost |
|---|---|
| Guest game | Per-thread busy/wait time, hot guest PCs, indirect-call volume, host bridge count, time in game-owned waits |
| Command translation | Packets/commands, state changes requested, parsing time, lookup/hash time, allocations, draw expansions |
| Resource preparation | Bytes copied, texture/vertex hashes, upload bytes, image creations, cache misses and evictions |
| Host recording | Emitted Vulkan state changes, descriptor/constant updates, draw calls, barriers, command-buffer submits |
| GPU | Pass timing, target dimensions/formats, resolves, bandwidth-heavy work, queue idle/fence wait |
| Storage and media | Read count/size, repeated ranges, decompression/decode time, cold versus warm cache |
| Presentation | Acquire/present wait, frames in flight, input-to-present delay, missed refresh intervals |

Keep CPU timings and GPU timestamps separate. They overlap in a pipelined renderer, and NVK timestamp
calibration depends on the exact build. A renderer can report little CPU work while waiting on a fence, or
large GPU time while the game thread is the limiting stage. Label idle, wait, and busy time distinctly.

## Choose experiments by stage, then by risk

1. **Remove measured waste without changing guest semantics:** avoid per-draw allocation/formatting, cache
   unchanged state, avoid redundant constant/descriptor writes, and copy only the required resource range.
2. **Reduce game-side work through supported settings:** count which passes/objects disappear and check
   that later shaders do not read stale buffers. Prefer narrowing costly effects over globally disabling
   stateful systems when the latter causes flicker or invalid history.
3. **Reduce GPU work only after proving a GPU limit:** lower internal resolution, effect-buffer size, pass
   count, or sample count. If renderer/guest CPU is on the critical path, fewer pixels may do nothing.
4. **Address cold-start work separately:** persist translated shader and driver-pipeline caches; prewarm
   measured common keys outside the first visible draw. Track startup cost and cache corruption/recovery.
5. **Change scheduling or frames in flight last:** compare latency, missed deadlines, audio, and input as
   well as average throughput. A separate presentation worker can raise average FPS while making frame
   times less consistent.

Keep the old implementation as an A/B path for high-risk changes. Record requested-versus-emitted state,
bytes, allocations, and waits so a claimed win has a mechanism. Re-run the same scene in handheld and docked
modes only when both are part of the target; do not blend their results into one number.

For detailed Switch NVK allocation/lifetime, submission, ZCULL, and shader-cache practices, continue to
[NVK renderer and GPU engineering](#detailed-nvk-renderer-and-gpu-engineering).

## Detailed NVK renderer and GPU engineering

This page covers Switch-specific renderer and GPU performance lessons found in the supplied local port
documents. It focuses on Horizon with Mesa NVK on Maxwell GM20B. The examples include original-Xbox NV2A
and Xbox-360 Xenos ports, whose guest graphics commands and render-target rules are different. Reuse the
Switch-side cost model; keep guest-GPU semantics in the relevant platform and game pages.

Shared measurement practices are in [Switch performance](03-switch-performance.md) and
[testing and debugging](05-testing-and-debugging.md). Xbox-specific renderer findings stay under
[`platforms/xbox/`](platforms/xbox/); Xenos/EDRAM findings stay under
[`platforms/xbox-360/`](platforms/xbox-360/).

### First establish the critical path

The game CPU, renderer/command translator, Vulkan driver, GPU, storage, and synchronization overlap. Stage
durations cannot simply be added to predict frame time; identify which stage is preventing the next
presented frame from meeting its deadline.

Instrument independent counters/timers for:

1. Game-thread execution, host calls, and time waiting for the guest lock.
2. Command parsing and guest-state translation.
3. Renderer state comparison, descriptor/constant writes, vertex/index preparation, and upload bytes.
4. Vulkan command recording, queue submission, fence waits, and present waits.
5. GPU execution by pass or frame, with the exact driver timestamp behavior recorded.
6. Asset reads, texture creation, shader translation/compilation, and pipeline creation.
7. Frames that are idle because the game submitted nothing versus frames blocked behind renderer or GPU work.

Use both sampled profiles and low-cost exact counters. Sampling identifies hot PCs; byte counters and per-
stage timers distinguish hashing, copying, parsing, allocation, or driver calls. Keep instrumentation out
of the hottest path where possible and quantify its overhead. On some supplied NVK builds GPU timestamps
read about 1.6 times low; calibrate the exact Mesa build rather than applying that ratio universally.

Run the same static scene or scripted route at stock clocks. Report the frame-time histogram, p50/p90/p95/
p99 where available, worst hitch, count over the target frame budget, and vblank cadence. A capped average
can hide CPU or renderer headroom, and a single average FPS cannot show uneven pacing or input delay.

### Horizon NVK costs and lifetime rules

#### GPU memory allocation

On the supplied NVK/Horizon builds, a GPU allocation can entail several kernel/driver operations: handle
creation, virtual-address reservation, mapping, memory initialization, and cache maintenance. Per-texture
dedicated allocation therefore scales poorly, and the number of NvMap handles is finite. Allocation costs
varied substantially across the supplied driver builds, so measure the exact path and build in use rather
than treating a prior timing as a prediction.

- Suballocate many images from a small number of larger slabs when per-object allocation is measurable.
  Supplied examples used slab sizes in the 16–32 MB range; choose size and headroom from the title's peak
  working set and the memory budget.
- Grow pools before a burst or loading transition where practical. A large new slab can itself trigger a
  costly clear/cache operation and hitch if it is allocated at the moment the scene needs it.
- Budget by bytes and expected reuse. Evict least-recently-used resources by time/access, not only frame
  count, because low frame rates make a fixed frame count represent a long time.
- Retire image memory, descriptors, and pooled blocks only after the fence for the last GPU use has
  completed. Freeing from a background worker can move host overhead away from recording, but does not
  remove synchronization or memory-pressure costs.
- Track pool committed bytes, live bytes, largest free block, handle count, allocation failures, and
  eviction/recreation rates.

#### Submission, fences, and barriers

Fence queries and queue submission cross the Horizon/driver boundary and can cost more than their desktop
counterparts. The supplied notes describe one queue, three swapchain images, FIFO and IMMEDIATE present
modes, and queue submission involving IPC. Avoid per-draw fence polling or unnecessary submissions; batch
work only when doing so preserves the guest command order and does not create larger latency or resource
hazards.

The local research characterizes broad barriers as a full GPU idle on the tested NVK path. Express only the
dependencies required by the next access, and measure barrier batches and wait time. Do not remove a
barrier merely because it is frequent: resolve, render-target feedback, aliasing, and attachment reuse
must preserve the original graphics API's ordering.

Keep the number of frames in flight a measured trade-off. More work slots may improve CPU/GPU overlap but
can make input arrive at the display later; fewer slots may reduce latency but leave the GPU idle. Compare
one and two frames in flight in the same high-load scene while tracking both throughput and
input-to-present delay.

#### Depth, clears, and feature compatibility

ZCULL/early-Z behavior depends on image usage and shader behavior. The supplied Mesa notes report that
depth images requesting transfer-destination usage could lose ZCULL eligibility; a render-pass clear via
`loadOp CLEAR` was used to preserve a compatible usage path. Results differed across the supplied
title/build configurations: some measurements showed a benefit, while another hardware A/B showed no
repeatable gain and coincided with an input anomaly that disappeared when the feature was disabled.
Therefore:

1. Record the exact image creation flags and clear/copy path.
2. Confirm that the target NVK build actually allocates a ZCULL plane.
3. Compare identical captures and GPU timing on hardware.
4. Check input, audio, and stability as well as FPS.
5. Keep the option off by default unless the target build demonstrates a repeatable benefit.

Shader writes to depth, discard/kill behavior, and alpha testing may also affect early-Z. Preserve correctness
first; do not infer the benefit from a feature flag alone.

GPU timestamp calibration, format support, dynamic rendering, push descriptors, geometry shaders, and
extensions are properties of the exact Mesa-on-Horizon build. Enumerate and log what the runtime exposes.
An emulator or desktop Vulkan driver does not prove Horizon support.

### Reduce renderer work without changing guest behavior

The cheapest host command is one that was never required. First measure whether the translator spends time
parsing, deriving state, allocating resources, copying data, or issuing redundant Vulkan commands.

#### Cache state and avoid redundant updates

Maintain a canonical guest-state shadow and a separate view of state already emitted to Vulkan. Mark only
the state affected by each command as dirty. Before a draw, compare desired state to emitted state and
skip unchanged pipeline-compatible binds, descriptors, viewport/scissor, blend/depth settings, and
constant uploads. Count requested changes and emitted commands; a cache with poor invalidation is a
correctness bug, while a cache that rarely hits is overhead.

Do not equate reducing `vkCmdDraw` calls with eliminating guest work. The translator may still have to
decode every source command and update state. Multi-draw, instancing, deferred recording, and command reuse
are only promising where traces show compatible consecutive draws and the design preserves ordering,
resource hazards, query results, blending, and dynamic state. Keep a baseline path and compare output
frame-by-frame.

Avoid allocations, formatting, string-key construction, and verbose logging in command/draw loops. Use
indexed tables or pre-sized storage for common lookups when measurements show hash/map or allocator cost.
Track the cost of instrumentation separately from production.

#### Copy and hash only what the draw needs

- Use index bounds and the vertex shader's actual fetch range to copy only the source bytes a draw reads.
  A supplied project found that avoiding unused vertex data sharply reduced copy work. The exact
  shader-analysis method is guest-GPU-specific; verify fetch semantics before rebasing addresses.
- Snapshot mutable guest buffers at submit time if a worker will consume them later. Holding guest pointers
  and reading them after the game reuses the memory produced corrupted dynamic UI/movie geometry in a
  supplied port.
- Avoid reading back from uncached upload memory. Prefer staging/copy plans that respect the memory
  architecture and count bytes moved in each direction.
- For static textures, a content hash or sampled detector can identify unchanged data. For renderer-written
  guest memory, invalidate only textures overlapping the modified pages/ranges. Scanning every texture
  layout on each copy can grow with all assets ever seen.
- Do not read dynamic memory twice to compare it: take one snapshot and use that same snapshot for both
  checks to avoid false mismatches caused by concurrent mutation.

#### Resolve, attachment, and target lifetime

Trace render-target creation, attachment load/store, resolves, copies, and barriers. A clear or target that
will be overwritten before its next read may be eligible for a handoff instead of a copy, but only after
the guest API's observable contents and aliasing are represented. Distinguish image allocation identity
from guest address/format: several guest views can refer to the same underlying storage, and depth and
color data may have different lifetimes even at the same guest address.

When diagnosing a wrong image, dump intermediate targets and resolve destinations, not just the final
presentation. Keep the source and destination formats, dimensions, offsets, pitches, swizzles/tiling,
clear values, and conversion scales in the trace. A missing world can originate in vertex positions or
guest input, while a black composite can originate in shader state, a resolve, a stale texture, or a
surface chosen for presentation. Avoid applying a resolution or API fix before identifying the first wrong
operation.

Apply the source GPU's render-target ownership, tiling, format reinterpretation, resolve, and alignment
rules inside the matching renderer adapter. Do not carry assumptions from one guest GPU into another
guest architecture or a Switch-native graphics path.

### Shader and pipeline warm-up

First-use shader compilation can be a visible hitch. Maintain two complementary persistent caches:

1. A translated shader cache, keyed by shader kind, source microcode/content hash, translator revision,
   and any modifications that affect output. Store metadata created during translation, such as resource
   bindings; restoring only binary code can leave runtime binding tables incomplete.
2. A driver pipeline cache, keyed by GPU vendor/device and driver version. Include every pipeline input
   needed to reproduce a pipeline: shader identities, layout counts, fixed-function state, and any
   geometry-stage key.

Use checksums or record lengths so a partial write can be detected. Stop at the first corrupt record and
rewrite the tail instead of trusting truncated cache data. Version the cache when the shader translator or
pipeline layout changes. Keep cache writes off hot threads and ensure a failed cache load can fall back to
normal compilation.

If gameplay uses a known set of pipelines, collect their complete keys and prewarm them on a worker before
the scene needs them. A supplied project reduced compile-linked hitches after adding translated-shader
caching and a complete pipeline prewarm list. This illustrates the difference between a translation cache
and a driver cache; measure the result in the target title and scene. Avoid starting many workers without
profiling allocator/driver locks, and do not let a first-use asynchronous pipeline appear as missing
geometry or pop-in.

### Game-side settings and GPU work

When the engine exposes supported graphics settings, use them as measured workload controls. Count the
draws/passes each option removes and verify that shaders do not continue to consume stale shadow, effect,
or history data. Disabling a whole system can produce flicker or inconsistent lighting; reducing light
count, range, caster set, or effect resolution may preserve required state with less work. A draw-count
heuristic can misclassify a 3D menu as gameplay, and a setting changed through a game command queue may
not execute while the game is inside a blocking load. Use the game's own state as the test for whether a
setting is active and restore any setting needed for loading.

Lower internal resolution only when GPU/pixel work is the limiting stage. It does not reduce command
decoding, draw submission, asset decompression, or game CPU time. Align dimensions to the source renderer's
documented requirements. Some original GPU architectures impose alignment or tile constraints; those
rules are specific to that guest renderer and do not establish a Switch-wide resolution grid.
Consider lower-resolution effect buffers or fewer passes when traces show they dominate GPU time, then
compare the complete image and game behavior.

### Evidence and transfer limits

For each renderer optimization, retain:

- A baseline build and a deterministic scene/frame range.
- Exact driver source revision, patches, compiler options, and exposed feature list.
- Requested versus emitted command counters, copies/bytes, image allocations, barriers, submits, and
  calibrated GPU timings.
- Image comparisons for intermediate and final targets, plus input/audio checks.
- Frame-time distribution and input latency on hardware at recorded stock clocks.
- A runtime switch or easy rollback until correctness is established.

The supplied reports show that ZCULL, depth-prepass skipping, deferred recording, dynamic blend state,
extra frames in flight, and different resolution choices can be neutral or harmful on a different title or
scene. Re-benchmark every change; do not transplant a percentage, optimal pool size, Mesa patch, or API
choice as a universal result.

### Local source trail

Synthesis of `bo1-nx/docs/SWITCH_OPTIMIZATION_GUIDE.md`, `bo1-nx/docs/OPTIMIZATION_LOG.md`,
`bo1-nx/docs/OTHER_PORTS_RESEARCH.md`, `conker-nx/docs/04-switch-platform.md`,
`conker-nx/docs/05-renderer.md`, and `conker-nx/docs/06-performance.md`. BO1 EDRAM/PM4 and Conker NV2A
details remain in their respective game/platform folders.
