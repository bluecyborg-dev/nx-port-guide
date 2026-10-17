# Testing and debugging

No single environment covers every failure mode. Use the fastest target for early correctness, then validate console-specific behavior on a real Switch.

## Test targets and what they can prove

| Target | Good for | Cannot establish |
|---|---|---|
| Linux/desktop build | Boot logic, lifted code, files, most input/audio behavior, graphics captures, debugger and repeatability | Horizon aliases/IPC/thread behavior, NVK-on-Horizon correctness or Switch speed |
| Emulator such as Eden | Some GL-path boot checks and menu behavior in the supplied original-Xbox notes | NVK submissions or representative race/gameplay FPS |
| Nintendo Switch hardware | Horizon memory mappings, unaligned atomics, actual Vulkan/NVK behavior, storage, clocks, input/audio and performance | Fast iteration; each run should have a clear question and captured build identity |

Use full-memory title takeover when the port needs more memory than applet mode provides. Record handheld/docked mode and clocks for performance runs.

## Bring-up test stages

1. Executable parses and all executable sections are covered.
2. Entry point starts; no unresolved indirect calls, invalid guest accesses or accidental runtime-data overlap.
3. Title screen, menus, controller input and audio work.
4. Saves, cache partitions, file creation and directory enumeration work on a fresh data directory.
5. Movies and vblank/frame pacing progress; offline/network paths do not hang.
6. Gameplay scene renders correctly; test multiple views, effects, surfaces, transitions and loading sequences.
7. Long run and scene transitions stay stable; check memory/resource growth, audio, input and save persistence.
8. Hardware performance and frame pacing are measured at known settings and clocks.

## Graphics correctness

Capture one relevant frame or command trace and find the earliest incorrect operation. Compare intermediate render targets, not only the final screen. Check target address and format, tiled layout, aliasing, clear/resolve order, depth state, texture bytes/mips and shader inputs. A format census, raw guest-memory dump and pixel readback can distinguish bad source data from bad shader/renderer translation.

Use a deterministic menu/fade or scripted route for image comparisons. If a fast path has a verification mode, compare it to the original path on first use or for all uses while debugging. Dynamic memory should be snapshotted once per comparison; reading it twice while the game mutates it can create false mismatches.

## Logging and crash evidence

Keep hot paths nonblocking: buffer log lines and flush from a worker. Network logging can block too; avoid holding a shared sink mutex across socket sends. Keep crash handling able to drain buffered logs. Record the exact ELF/NRO pair for every run.

Useful diagnostics mentioned in the supplied notes include unresolved-call and stub traces, ABI checks, generated-code searches for unresolved flags or unsupported instructions, guest call-stack watchdogs, frame traces, surface dumps, pad scripts, crash register/address dumps and sampling profiles. Use a trace flag or delayed trigger to capture the first heavy gameplay frame instead of a loading frame. For file and save problems, a per-open trace is worth having from the start: the guest path, requested access, create disposition and options, the status returned, and the guest return addresses on the stack. It shows what the title asked for (create versus open, directory versus file) and which game routine asked, which a plain path log does not.

If a run hangs, capture thread stacks and distinguish a guest spin, an expected event wait, a lock deadlock, file I/O, GPU fence wait or logging blockage. Do not infer a deadlock solely from a frozen image.

## Console performance protocol

- Use a known scene and launch configuration; archive the exact build.
- Change one setting per A/B run, keep clocks and device mode fixed, and repeat enough to see normal scene-to-scene variation.
- Record the active settings, frame-time distribution, thread utilization, GPU timing, draw/upload counts and stalls.
- Compare input latency, audio stability and image correctness alongside throughput.
- Treat profiler call chains cautiously if the build lacks frame pointers; the source notes recommend trusting sampled PCs more than deep stacks in that case.

## Storage, deployment and saves

The supplied notes document network NRO deployment and live logs as a way to avoid repeatedly moving the SD card. Before trusting a failure, rule out a stale log listener and verify that the game reads the expected files. SD cards may silently retain stale/corrupt blocks; hash data after copying where possible, avoid overwriting suspect files in place, and keep a known-good backup.

Always test a fresh save/cache directory as well as an existing one. Verify nested directory creation, file dispositions, save/load cycles and behavior after restarting the application. A host filesystem error can mean a missing asset or a failed attempt to create parent directories; inspect the requested guest path and operation. Reproduce the original OS's status codes exactly, not only success versus failure: games branch on them. In the Conker port, the game's own recursive directory helper created missing parents only when an attribute query reported "path not found" (missing parent) rather than "name not found" (missing last component); a runtime that reported "name not found" for both sent the game to its fatal-error screen on a fresh cache. Fix the status the game reads instead of working around it in the runtime (for example by creating parent directories the original OS would refuse to create).
