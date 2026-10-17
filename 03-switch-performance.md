# Switch performance and optimization

Treat each optimization as a measured experiment. The supplied port notes repeatedly show that a change that helps one title, scene or device mode can hurt another. Separate correctness changes from performance changes and leave a runtime switch or easy rollback path for risky experiments.

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
