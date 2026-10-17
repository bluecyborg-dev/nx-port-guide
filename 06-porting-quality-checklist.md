# Porting quality checklist

## Before implementation

- [ ] Identify original console, executable region/revision, CPU/GPU architecture and graphics command path.
- [ ] Choose a toolchain that translates the actual guest instruction set and runtime model.
- [ ] Inventory executable sections, imports, entry point, image/BSS end, callbacks and indirect-call patterns.
- [ ] Keep title files, generated code and captures out of version control.
- [ ] Set up desktop correctness build and target-console build with separate output folders.
- [ ] Record dependency licenses and exact revisions.

## Compatibility and boot

- [ ] Guest RAM, aliases, stack, runtime data and device apertures have verified non-overlapping layouts.
- [ ] Every executable section is covered and seeds point to real function boundaries.
- [ ] Unsupported instructions and ABI/calling-convention assumptions are reviewed.
- [ ] Every instruction the translator left untranslated is listed (grep the generated code for its fallback comments, whatever prefix they carry; see [debugging recompiled code](09-debugging-recompiled-code.md)) and each one is implemented or shown unreachable. Rare x87 stack operations count: `fincstp`/`fdecstp`, `fcmov*`, `ficom`, `fxtract`.
- [ ] File paths, nested directories, save/cache creation and error codes are exercised from a fresh state.
- [ ] Thread, event, timer, audio and input behavior are tested on the target host scheduler.
- [ ] Network initialization cannot block offline play when the title's network service is unavailable.
- [ ] Movies and the game loop advance at the expected timebase.

## Graphics and gameplay

- [ ] First world frame is inspected with a trace/capture.
- [ ] If geometry is missing or off screen, the transform constants (camera and world matrices) are checked for orthogonal rows before clip, cull or depth handling is changed.
- [ ] Surface formats, tiling, aliases, clears, resolves, depth and shader state are tested.
- [ ] Menus, transitions, split-screen/multiple viewports, effects and representative gameplay areas are checked.
- [ ] Controller labels, analog inputs, rumble, suspend/resume and display modes are reviewed where relevant.
- [ ] Saves survive a full restart.
- [ ] Long sessions and loading transitions do not leak resources or lose input/audio.

## Performance and release readiness

- [ ] CPU, renderer, GPU, I/O and waits are measured separately on Switch hardware.
- [ ] Results include clocks, handheld/docked state, scene, configuration and frame-time distribution.
- [ ] Each optimization has a correctness check, measured result and rollback path.
- [ ] Cold shader/pipeline behavior and warm-cache behavior are both measured.
- [ ] Final build identity matches archived symbols/profiler ELF.
- [ ] Third-party license obligations and repository exclusions are rechecked.
- [ ] Game-specific findings are in that game's logbook; shared guidance only contains verified reusable behavior.
