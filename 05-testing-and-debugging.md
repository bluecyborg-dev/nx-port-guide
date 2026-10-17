# Testing and debugging

No single environment covers every failure mode. Use the fastest target for early correctness, then validate
console-specific behavior on a real Switch. Bugs that come from the translator itself (missed functions,
untranslated instructions, wrong numeric results) have their own page:
[debugging recompiled code](09-debugging-recompiled-code.md).

## Test targets and what they can prove

| Target | Good for | Cannot establish |
|---|---|---|
| Linux/desktop build | Boot logic, lifted code, files, most input/audio behavior, graphics captures, debugger and repeatability | Horizon aliases/IPC/thread behavior, NVK-on-Horizon correctness or Switch speed |
| Emulator such as Eden | Some GL-path boot checks and menu behavior in the supplied original-Xbox notes | NVK submissions or representative race/gameplay FPS |
| Nintendo Switch hardware | Horizon memory mappings, unaligned atomics, actual Vulkan/NVK behavior, storage, clocks, input/audio and performance | Fast iteration; each run should have a clear question and captured build identity |

Use full-memory title takeover when the port needs more memory than applet mode provides. Record handheld/docked mode and clocks for performance runs.

### Desktop bench

- Build for the console's CPU architecture (an AArch64 Linux container on an ARM machine): code generation,
  alignment and address-space problems then show up on the bench too. A 64-bit host with a 48-bit user
  address space can still place mappings where the guest's device windows do not fit; reserve the guest
  window at a fixed low host address.
- Use a software Vulkan driver (lavapipe) for rendering correctness, headless, with frame dumps. It says
  nothing about speed.
- Reproduce the console's restrictions where the runtime allows it, for example mapping no memory aliases
  when Horizon cannot alias.
- Run the game in the background with a live input file and periodic frame dumps (see
  [Drive the game without hands](#drive-the-game-without-hands)), so a session is "look at the newest frame,
  send a press".
- Close emulators politely: a hard kill corrupted Eden's executable/configuration once.

### Console run loop over Wi-Fi

- Send the NRO with `nxlink` to the Homebrew Menu's NetLoader (hold R on a game for title takeover, then
  press Y) and have the game send stdout back (`nxlinkStdio`). No SD-card swapping between builds; keep the
  game data on the card and replace only the NRO.
- Kill stale log listeners before each run: a leftover listener once blocked the game's stdout at a black
  screen.
- Do not probe the NetLoader port with a scanner: a probe connection made the next upload fail with
  "connection reset". If the upload cannot connect, the NetLoader is simply not open yet.
- A log piped through another program can arrive in 32 KB chunks; run the sender under a pseudo-terminal
  (`script -q`) so it streams line by line.
- Archive the exact NRO and ELF of every run under a content hash, and check the ELF's entry point before
  sending.

## Bring-up test stages

1. Executable parses and all executable sections are covered.
2. Entry point starts; no unresolved indirect calls, invalid guest accesses or accidental runtime-data overlap.
3. Title screen, menus, controller input and audio work.
4. Saves, cache partitions, file creation and directory enumeration work on a fresh data directory.
5. Movies and vblank/frame pacing progress; offline/network paths do not hang.
6. Gameplay scene renders correctly; test multiple views, effects, surfaces, transitions and loading sequences.
7. Long run and scene transitions stay stable; check memory/resource growth, audio, input and save persistence.
8. Hardware performance and frame pacing are measured at known settings and clocks.

Each stage tends to expose new missed functions (code only reached from that scene): grep the log for
unresolved calls after every new scene, not only at boot.

## Drive the game without hands

- **Automate the route to the test scene.** Scripted controller input timed from the title's first pad read
  (so boot-time variance does not shift it), a save file that lands at the scene, or a console command queued
  from a hook (the Black Ops port's approach). Boot times on a headless bench vary by a minute; a script timed
  from process start drifts.
- **Live input.** A file the runtime watches, with presses appended while the game runs, plus a frame dump
  every N flips, turns menu navigation into a loop of "look at the newest frame, append a press".
- **Hold presses longer than a frame.** Games sample the pad once per frame; at 5 FPS (a busy bench) a
  150 ms press can fall between two samples and be lost. Use 300–400 ms.
- **Trigger traces on content, not time.** Arm a frame trace on the first heavy frame after a delay (for
  example, once the draw count passes a threshold), or it lands on a loading screen.

## Graphics correctness

### Find the first wrong operation

Capture one relevant frame or command trace and find the earliest incorrect operation. Compare intermediate
render targets, not only the final screen: dump the off-screen surfaces (shadow maps, glow and composite
targets) next to the presented frame. Check target address and format, tiled layout, aliasing, clear/resolve
order, depth state, texture bytes/mips and shader inputs. A format census, raw guest-memory dump and pixel
readback can distinguish bad source data from bad shader/renderer translation. Dump each generated shader
with the key it was built from.

For command or MMIO traces, keep raw register values and decode each field against its exact mask in the
matching hardware header or emulator source before assigning meaning. An incorrect bit label can create a
false cause: in Conker, `SET_CONTROL0` bit 20 was misread as the NV2A w-buffer flag (which is bit 16), and
a w-buffer fix was planned for a bug that had nothing to do with depth.

### Black or missing geometry

Work through these in order; the early steps are cheap and each one rules out a whole class.

1. **Check what was presented.** A presented image of the wrong size means the wrong surface was shown
   (Conker clears a small glow target after compositing; presenting the last surface *touched* showed it).
   Present the last surface *drawn*.
2. **Bisect the frame.** Skip ranges of draws, or every draw with a given blend mode, and look at the
   result. A few runs name the guilty pass. (Conker's first blackout was three `DST_COLOR × ZERO` shadow
   passes multiplying the scene by shadow maps whose 16-bit clear value had been read as 32-bit.)
3. **Not rasterised, or rasterised black?** Temporarily force the back-buffer clear colour to something loud
   (red). If the frame stays red, the geometry is being clipped or culled: the cause is in transforms,
   clip/depth ranges or depth modes, not in shading.
4. **Check the transform inputs before the renderer.** If geometry is clipped, disassemble the vertex
   program to see which constants hold the world/view/projection matrices, log those constants for every
   draw of one bad frame, and test each 3×3 part: a camera or world matrix has equal row lengths (uniform
   scale) and mutually orthogonal rows. Draws that use other matrices in the same frame (shadow or light
   views) are a control group. A degenerate camera matrix was computed by the title's own CPU code, so the
   bug is in the translated code: follow it there with
   [these steps](09-debugging-recompiled-code.md#prove-a-translated-routine-wrong-or-right). In Conker this
   found an x87 `fincstp` the translator had emitted as a no-op, after a day of depth and clip probes that
   could not have found it ([Conker logbook](platforms/xbox/conker/conker-logbook.md)).
5. **Position diagnostics: label their source.** A CPU-side interpreter of the vertex program can point to
   a suspect transform but does not establish what the generated GPU shader produced. Capture the actual
   vertex-shader outputs for the same draw where possible (desktop OpenGL transform feedback of `oPos` and
   the clip-space position). For indexed lists or strips, replay the index sequence as `GL_POINTS` during
   transform feedback with `GL_RASTERIZER_DISCARD` enabled: this records one shader result per input index,
   without strip/fan expansion duplicates and without changing the rendered frame
   ([Khronos OpenGL 4.3 specification](https://registry.khronos.org/OpenGL/specs/gl/glspec43.core.pdf)).
   If GPU and CPU outputs agree but lie outside the view, the shader is fed bad input: go back to step 4.
   An OpenGL readback does not validate Vulkan or console output. When the renderer has both direct
   vertex-buffer reads and a converted-attribute path, A/B them before blaming the shader inputs.
6. **Depth last, and scoped.** Global depth overrides are screening probes: disabling depth also removes
   draw ordering, and forcing every test to pass changes depth-tested shadow passes. Scope an override to
   the target geometry and leave other passes unchanged before concluding that depth is or is not the
   cause. If a scoped compare change reveals geometry in one view but later views still go black, depth
   affects visibility but the correct compare function is not established; compare an early visible frame
   with the first black frame, and if depth clears and compare state stay the same, look at vertex output
   and raster coverage instead.
7. **Reproduce on every back end.** A failure shared by back ends that use the same shader translation
   points at shared guest state, inputs or translation. It narrows the search without proving that the GPU
   outputs are identical.

### Image comparisons

Use a deterministic menu/fade or scripted route for image comparisons. If a fast path has a verification
mode, compare it to the original path on first use or for all uses while debugging. Dynamic memory should be
snapshotted once per comparison; reading it twice while the game mutates it can create false mismatches.

## Logging and crash evidence

Keep hot paths nonblocking: buffer log lines and flush from a worker. Network logging can block too; avoid holding a shared sink mutex across socket sends. Keep crash handling able to drain buffered logs. Record the exact ELF/NRO pair for every run.

Collect the translator's own signals on every run (unresolved and stub calls, ABI checks, untranslated
instructions, NaN traps; see [debugging recompiled code](09-debugging-recompiled-code.md)), plus guest
call-stack watchdogs, frame traces, surface dumps, crash register/address dumps and sampling profiles. For
file and save problems, a per-open trace is worth having from the start: the guest path, requested access,
create disposition and options, the status returned, and the guest return addresses on the stack. It shows
what the title asked for (create versus open, directory versus file) and which game routine asked, which a
plain path log does not. Never judge audio quality from a run with audio tracing on: the trace itself causes
gaps.

If a run hangs, capture thread stacks and distinguish a guest spin, an expected event wait, a lock deadlock,
file I/O, GPU fence wait or logging blockage. Do not infer a deadlock solely from a frozen image.

## Console performance protocol

- Use a known scene and launch configuration; archive the exact build.
- Change one setting per A/B run, keep clocks and device mode fixed, and repeat enough to see normal scene-to-scene variation.
- Record the active settings, frame-time distribution, thread utilization, GPU timing, draw/upload counts and stalls.
- Compare input latency, audio stability and image correctness alongside throughput.
- Treat profiler call chains cautiously if the build lacks frame pointers; the source notes recommend trusting sampled PCs more than deep stacks in that case.

## Storage, deployment and saves

The supplied notes document network NRO deployment and live logs as a way to avoid repeatedly moving the SD card (see [the console run loop](#console-run-loop-over-wi-fi)). Before trusting a failure, rule out a stale log listener and verify that the game reads the expected files. SD cards may silently retain stale/corrupt blocks; hash data after copying where possible (against the extracted disc), avoid overwriting suspect files in place, keep a known-good backup and replace a card that fails once. The Black Ops port lost days to a failing card.

Always test a fresh save/cache directory as well as an existing one. Verify nested directory creation, file dispositions, save/load cycles and behavior after restarting the application. A host filesystem error can mean a missing asset or a failed attempt to create parent directories; inspect the requested guest path and operation. Reproduce the original OS's status codes exactly, not only success versus failure: games branch on them. In the Conker port, the game's own recursive directory helper created missing parents only when an attribute query reported "path not found" (missing parent) rather than "name not found" (missing last component); a runtime that reported "name not found" for both sent the game to its fatal-error screen on a fresh cache. Fix the status the game reads instead of working around it in the runtime (for example by creating parent directories the original OS would refuse to create).
