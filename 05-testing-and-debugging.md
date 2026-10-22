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

## Automation boundaries

Automate the repeatable work at each layer, but state what evidence the automation produces. A successful
compile proves that a toolchain emitted an artifact; it does not prove that the artifact launched, reached
game code, rendered a scene or ran on the console.

| Work | Good unattended coverage | What still needs a check |
|---|---|---|
| Analysis and builds | Run pinned tools and build scripts; keep exit status and artifact identity. | Review generated-code summaries and warnings about missed functions or unsupported instructions. |
| Host smoke run | Timed launch, fresh save/cache, logs and periodic frames. | Confirm the intended milestone; a timeout-killed run is not a pass by itself. |
| Test-scene route | Script input, load a known save, or use a debug route. | Verify that input reaches the active backend and the route reaches its target scene. |
| Logs and performance | Capture crash output, progress markers, frame timing and profiles. | Parse failure signals and resolve addresses against the exact build. |
| Console upload and launch | Script transfer and launch once the homebrew loader or debugger is ready. | Opening that loader and confirming launch mode or device state may still need a person or hardware control. |
| Console gameplay and visuals | Capture counters and profiles during a repeatable route. | Check image, sound, controls, transitions and performance on real hardware. |
| Evidence retrieval | Stream logs and artifacts the runtime sends over the network. | Transfer files left on the SD card, such as binary profiles or frame dumps. |

For every automated run, define a positive milestone and explicit failure checks: expected scene/progress
marker, no crash or watchdog report, no unresolved calls for the covered route, and a captured frame or
other observable result when graphics matter. Keep capture review separate from the program's exit status.

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

#### Unattended smoke runs on the development machine

A desktop build of the port on the development machine itself (for example macOS through a Vulkan-on-Metal
layer) can check most changes without anyone playing:

1. **Launch in the background with a time limit**, then stop the process: start it, sleep about two
   minutes, send a polite kill and then a forced one. A foreground run once hung the user's terminal.
2. **Reach the test scene with no input**: an environment variable queues one console command (a map load)
   a set number of seconds after start. The game then idles in that scene.
3. **Pass the setting under test as a launch argument**, the same form the console build uses, so one
   build covers both sides.
4. **Read the runtime's own log file**, not the terminal output, and check:
   - periodic present and draw counters (a steady value at the frame cap means no stall);
   - per-rule counters for any renderer skips;
   - a startup line listing the settings the game actually applied;
   - warnings and errors.
5. **Trace one frame after a delay** to log per-pass draw counts and state. Two runs that differ in one
   option then show exactly which draws changed.
6. **Cycle a live setting on a timer** through an environment variable, to test live switching without a
   controller.

What this proves: the build compiles, boots into the scene and accepts the settings, nothing crashes, and the
draw counts per pass and per rule. What it does not prove:

- performance on the Switch (different CPU, GPU and driver);
- busy scenes, since nobody plays and nothing spawns;
- visuals, if the capture path cannot read the whole frame (one translation layer only returned the HUD;
  check a capture before trusting it);
- Horizon behavior: cores, priorities, memory mapping.

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
- Everything else can be scripted: the sender builds the arguments, a debug argument loads the test scene
  by itself, and the log streams back. A person still has to open the NetLoader and play, because the
  measurements and the feel come from real gameplay. Send one build per question, then compare its log with
  the previous session's.
- Apply launch arguments before the runtime reads any setting. In Conker the `KEY=VALUE` arguments were set
  after boot code had already read the profiler, clock and logging switches, so a run "with the profiler
  on" had no profile and nothing said so. Log each applied argument.
- A debug route into the test scene can change what the game allows. A developer map command, for example,
  can enable cheats and so accept settings a normal launch refuses. Check at least once that every setting the
  port changes still takes effect when the game is started the way players start it. Read the value back rather
  than trusting the command.

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

### Unattended test sessions in a container

A coding agent (or a script) can run the whole check-a-change loop on the desktop bench while the developer
is away, and leave evidence a person can review afterwards. The pieces, in the order a session uses them:

1. **Build in the container**, incrementally, and keep only the lines that matter (errors, warnings in the
   files just changed, the final link line). Archive each console build under a content hash next to its
   unstripped ELF.
2. **Start the game detached** in a named container: the mounted repository supplies the binary and game
   data; the game reads presses from a live-input file and dumps a frame every N presents into a folder.
   A loop inside the container deletes all but the newest few hundred dumps: unbounded dumps once filled
   the disk and hung the container runtime, which then needed a forced restart.
3. **Wait on log markers, not on time.** Poll the log every few seconds with a timeout: a draw count above
   a threshold means the menu is up, a file-open line for a level's data means the load reached that
   level. Boot time on a software renderer varies by a minute, and a build running at the same time slows
   the game further; fixed sleeps land in the wrong scene.
4. **Drive it with presses appended to the input file**, each held 300–400 ms, with a pause long enough for
   the menu to react. Keep the route that reaches a test scene as a script (route = waits + presses), and a
   save that starts past the parts not under test.
5. **Look at the frames.** Assemble the newest dumps, or every Nth dump of a phase, into one labelled
   contact sheet and inspect it; open single frames at full size only where the sheet shows something.
   Thumbnails mislead about blur and fine detail.
6. **Trace on purpose.** Arm a frame trace by creating a flag file the runtime deletes when it starts
   tracing; to sample a phase whose timing is unknown (a cutscene), create it again every two seconds and
   classify the traced frames afterwards by their draw counts or passes. Diagnostics that print only for
   unusual frames (for example, frames with thousands of draws) keep the log short enough to read.
7. **Stop the container** at the end of every session (one run at a time), and compare against the
   previous run's log and frames before and after a change.

Console side: the upload can be scripted as a retry loop that keeps trying the homebrew loader every ten
seconds for an hour, so the developer only has to open it. When nobody will watch the log, send the build
without the log listener, so the game neither waits for nor depends on the development machine.

What a person still does: judge images against the original, play on the console, and say whether it feels
right. Switches and log markers for one toolkit: [platform page](platforms/xbox/02-xboxrecomp-debug-reference.md#unattended-bench-session);
exact commands of a real session: [worked example](platforms/xbox/conker/build-process.md#how-claude-runs-a-test-session).

## Graphics correctness

### Find the first wrong operation

Capture one relevant frame or command trace and find the earliest incorrect operation. Compare intermediate
render targets, not only the final screen: dump the off-screen surfaces (shadow maps, glow and composite
targets) next to the presented frame. Check target address and format, tiled layout, aliasing, clear/resolve
order, depth state, texture bytes/mips and shader inputs. A format census, raw guest-memory dump and pixel
readback can distinguish bad source data from bad shader/renderer translation. Dump each generated shader
with the key it was built from.

When the renderer keeps a guest surface in a host image (a depth buffer, a render target) and the title
reads that memory back as a texture, reproduce the bytes, not one interpretation of them. Titles reinterpret
surfaces: Conker samples its Z24S8 depth buffer as two different 32-bit colour formats in the same frame,
and its depth of field takes alpha from the depth's high byte in one of them. Store the converted surface in
the guest's byte order and let each declared format pick its channels (an image-view swizzle on Vulkan);
check every format the frame trace shows reading that address.

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

### A picture that freezes while the game keeps running

A loading screen that never appears (the last frame stays up for a few seconds, then the game goes on) is
often a presentation problem, not a stall. Check whether draws continue during the freeze while no flip
or present call arrives. Consoles scan out a framebuffer by themselves at every vertical blank, so a title
can update the screen by drawing straight into the buffer being displayed and never flip: Conker's loader
draws a progress-bar step into the back buffer and copies it into the front buffer, hundreds of times a
second for several seconds. A renderer that presents only on flips shows nothing new. Count draws per
target surface for the long frame and compare with the scan-out address register; if draws hit the
displayed buffer, present that buffer yourself, rate-limited and only after no flip has come for a while
(a normal frame never draws into the buffer on screen, and the delay keeps a half-drawn frame from being
shown). Details for the original Xbox: [xboxrecomp debug reference](platforms/xbox/02-xboxrecomp-debug-reference.md).

### Image comparisons

Use a deterministic menu/fade or scripted route for image comparisons. If a fast path has a verification
mode, compare it to the original path on first use or for all uses while debugging. Dynamic memory should be
snapshotted once per comparison; reading it twice while the game mutates it can create false mismatches.

## Audio correctness

Players describe broken audio vaguely ("static", "heavy rain", "it cut out"); turn that into data:

1. **Capture the exact output PCM** for a scripted route (menu selections, a cutscene), kept in memory and
   written at the end so the capture does not disturb timing.
2. **Find the bad windows.** For each 20 ms window compute the RMS and the ratio of first-difference energy
   to signal energy: about 2 for white noise, well below 1 for music and speech. Loud windows with a ratio
   above 1 are noise; silent stretches where sound is expected are starvation.
3. **Find the voice.** In the emulated audio processor, run the same measurement on each voice's own
   fetched samples and log the voice's format and buffer addresses when it looks like noise. RMS ≈ 0.5 with
   a ratio near 2 is uniform random data: the voice is reading the wrong memory, not decoding badly.
   In Conker the voice's buffer was above RAM and the device-side address mapping masked it into the
   game's code.
4. **For silence, trace the stream protocol** (segments queued, completions, pauses) next to the frame
   hitches. A stream that drains during a long frame and is then paused for lack of data points at the
   stall, not at the audio code.

Never judge audio from a run with audio tracing on: the trace itself causes gaps.

## Logging and crash evidence

Keep hot paths nonblocking: buffer log lines and flush from a worker. Network logging can block too; avoid holding a shared sink mutex across socket sends. Keep crash handling able to drain buffered logs. Record the exact ELF/NRO pair for every run.

Collect the translator's own signals on every run (unresolved and stub calls, ABI checks, untranslated
instructions, NaN traps; see [debugging recompiled code](09-debugging-recompiled-code.md)), plus guest
call-stack watchdogs, frame traces, surface dumps, crash register/address dumps and sampling profiles. For
file and save problems, a per-open trace is worth having from the start: the guest path, requested access,
create disposition and options, the status returned, and the guest return addresses on the stack. It shows
what the title asked for (create versus open, directory versus file) and which game routine asked, which a
plain path log does not.

If a run hangs, capture thread stacks and distinguish a guest spin, an expected event wait, a lock deadlock,
file I/O, GPU fence wait or logging blockage. Do not infer a deadlock solely from a frozen image.

## Console performance protocol

- Use a known scene and launch configuration; archive the exact build.
- Change one setting per A/B run, keep clocks and device mode fixed, and repeat enough to see normal scene-to-scene variation.
- Record the active settings, frame-time distribution, thread utilization, GPU timing, draw/upload counts and stalls.
- Compare input latency, audio stability and image correctness alongside throughput.
- Treat profiler call chains cautiously if the build lacks frame pointers; the source notes recommend trusting sampled PCs more than deep stacks in that case.
- Have the console profiler summarize into the log (hottest code ranges per thread per window, resolved later against the ELF), so a Wi-Fi run needs no trip to the SD card. The same format from a desktop sampler (a `SIGPROF` timer recording the interrupted PC) lets one report script read both. A debugger-driven sampler is not a substitute: gdb's asynchronous continue/interrupt does not work reliably in batch mode.

## Storage, deployment and saves

The supplied notes document network NRO deployment and live logs as a way to avoid repeatedly moving the SD card (see [the console run loop](#console-run-loop-over-wi-fi)). Before trusting a failure, rule out a stale log listener and verify that the game reads the expected files. SD cards may silently retain stale/corrupt blocks; hash data after copying where possible (against the extracted disc), avoid overwriting suspect files in place, keep a known-good backup and replace a card that fails once. The Black Ops port lost days to a failing card.

Always test a fresh save/cache directory as well as an existing one. Verify nested directory creation, file dispositions, save/load cycles and behavior after restarting the application. A host filesystem error can mean a missing asset or a failed attempt to create parent directories; inspect the requested guest path and operation. Reproduce the original OS's status codes exactly, not only success versus failure: games branch on them. In the Conker port, the game's own recursive directory helper created missing parents only when an attribute query reported "path not found" (missing parent) rather than "name not found" (missing last component); a runtime that reported "name not found" for both sent the game to its fatal-error screen on a fresh cache. Fix the status the game reads instead of working around it in the runtime (for example by creating parent directories the original OS would refuse to create).

## Run evidence and comparison discipline

Give each run a stable identifier and record the source commit, NRO hash, matching ELF, executable/data
identity, runtime options, launch mode, console mode/clocks, save/cache state, route, and capture interval.
Keep raw logs and captures immutable; write interpretations in the logbook with a link to that evidence.
The word **fixed** should name the build and environment where the failure stopped reproducing.

For correctness comparisons, hold the scene, input, save state, renderer options, and timing point constant.
Separate cold-start from warm-cache behavior and fresh-save from existing-save behavior. When a frame trace
is expensive, trigger it from a draw/packet threshold or known in-game event, then keep its unmodified
source data alongside decoded views. Compare intermediate surfaces and state, not only the final screenshot.

Use a hypothesis ledger for visual failures:

| Field | Example of what to write |
|---|---|
| Observation | The presented image dimensions differ from the expected back-buffer dimensions. |
| Suspected layer | Presentation target selection. |
| Competing explanations | Last surface touched versus last surface drawn; stale swapchain image. |
| Discriminating test | Log target address/dimensions for each draw, clear, and flip. |
| Result and limit | Selecting the last-drawn surface restores the expected dimensions; this does not validate all composites. |
| Next step | Verify the target on a frame with no draw or an explicit resolve. |

This keeps a useful observation from becoming an unsupported diagnosis. A temporary override that reveals
geometry proves that a state variable affects visibility; it does not automatically identify the correct
state or prove the override is a valid fix.

For the detailed Switch console validation and deployment procedure, see
[Switch console validation and deployment](#switch-console-validation-and-deployment).

## Switch console validation and deployment

### Validate in increasing cost order

#### Desktop or container correctness pass

Build for the console CPU architecture where possible. An AArch64 Linux container can expose alignment,
calling-convention, register-allocation, and address-space issues that an x86 desktop hides. A software
Vulkan backend such as lavapipe is useful for deterministic logic and many renderer checks, but its timing
is not a Switch performance proxy.

Exercise boot, file reads, cache creation, menu navigation, one representative gameplay route, audio,
input, transitions, save/load, and shutdown. Use a fresh save/cache path in addition to a warmed path. A
fresh cache tests directory creation, error statuses, and first-use work that an existing cache hides.

#### Hardware pass

Use Homebrew Menu title takeover when the port needs the full process memory budget. Keep game data on the
SD card and send only the NRO over the network when the port's NetLoader workflow supports it. Record the
actual console mode and clocks; repeat on hardware after desktop validation for mappings, IPC, unaligned
atomics, audio/input deadlines, NVK, storage, and performance.

Automate menu/gameplay navigation with input scripts, a known save, or a title-specific command hook.
Trigger traces from an in-game event or draw/packet threshold rather than process start time, because load
and boot duration varies. When the bench runs slowly, hold controller inputs long enough to span a game
input-sampling interval.

#### Network upload and log discipline

- Make sure the Homebrew Menu's NetLoader is open before upload. Do not use a port scan as a readiness
  check; one supplied test found that probing the NetLoader disrupted the next transfer.
- Stop stale log listeners before a run and confirm which process owns the log connection. A stale reader
  once left a game blocked while writing output.
- Use line-buffered/pseudo-terminal transport if a pipe batches logs. Confirm the sender is streaming
  before interpreting an empty log as an early game crash.
- Keep socket sends off the shared logging mutex and hot threads. Queue lines for a dedicated sender and
  track queue drops; a network stall must not freeze audio and rendering.
- Keep runtime arguments short and log the effective settings. Store stable defaults on the SD card and
  use per-run arguments for diagnostic overrides when supported.
- Archive the log and exact NRO/ELF for every run. Add frame/surface captures and profiler files with the
  same run identifier.

### Make test state reproducible

Use a run identifier that joins source state, output artifacts, environment, and evidence. Record whether
the run used a fresh or warmed filesystem/cache, deterministic input route, random seed if available,
display mode, save state, active graphics options, and the point at which the capture was taken. Keep
baseline and experimental runs under separate output names.

Change one variable at a time unless the experiment explicitly tests an interaction. For an optimization,
keep the original path available behind a switch, verify image/gameplay correctness first, then measure
performance. A test that improves FPS but changes frame output, blocks loading, breaks held input, or
causes audio gaps is not a successful optimization.

### Console build/run checklist

- [ ] Toolchain, driver, dependencies, patches, source state, and flags are recorded.
- [ ] Code generation was rerun only when relevant, and the generated code corresponds to the current
      executable/seed set.
- [ ] The NRO packages the intended ELF and has the expected startup/link layout.
- [ ] Matching unstripped ELF and NRO are archived by the same build identifier/hash.
- [ ] Desktop/container run passes boot and a reproducible scene route.
- [ ] Hardware launch mode, display mode, clocks, and active options are recorded.
- [ ] Fresh and warmed save/cache runs have both been exercised.
- [ ] Upload and log stream are confirmed; no stale listener can block the game.
- [ ] Final run evidence includes logs, frame dumps/profiles when relevant, and a clear claim about what
      was actually verified.

### Local source trail

This page synthesizes `conker-nx/docs/10-building.md`, `conker-nx/docs/11-testing-on-console.md`,
`conker-nx/docs/13-build-process.md`, `bo1-nx/docs/BUILDING_NRO.md`,
`bo1-nx/docs/SWITCH_OPTIMIZATION_GUIDE.md`, and `bo1-nx/docs/OPTIMIZATION_LOG.md`. Exact commands and
tool paths remain in the corresponding project build guides.
