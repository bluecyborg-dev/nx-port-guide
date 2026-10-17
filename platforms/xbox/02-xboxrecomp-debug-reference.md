# xboxrecomp debug reference

Runtime switches, log signals and tools of the `xboxrecomp` toolkit as vendored in the antoxa2584x
original-Xbox ports and in conker-nx. They are independent of the game. The methods behind them are in
[debugging recompiled code](../../09-debugging-recompiled-code.md) and
[testing and debugging](../../05-testing-and-debugging.md); game-specific findings stay in each game's folder
(for example [Conker's runtime and renderer findings](conker/03-runtime-debug-and-render-findings.md)).

Set switches as environment variables on desktop builds. On the Switch, put `KEY=VALUE` lines in
`sdmc:/switch/<title>/<title>_env.txt`, or pass them as nxlink arguments (the NetLoader cuts the argument
string at about 256 characters).

## Desktop bench

- Build for AArch64 like the console (an arm64 Linux container on an ARM Mac); Vulkan runs on lavapipe,
  headless, with frame dumps. Good for logic, menus and most render bugs; says nothing about speed.
- `RECOMP_NO_MIRRORS=1` maps no RAM mirrors, as on Horizon, which cannot alias memory: aliasing bugs then
  show up on the bench too.
- On 64-bit Linux the guest window is reserved at a fixed low host address (64 GB, then 256 GB; the log
  prints `mapped 65536 KB at 0x1000000000`), so host address = `0x1000000000` + guest VA for RAM and for the
  extended VMA range.
- Under gdb keep ASLR on (`set disable-randomization off`): with it off, the kernel-VA and tiled-aperture
  windows cannot be mapped.
- Translated functions keep their guest names (`sub_XXXXXXXX`), so host backtraces are guest call chains.
  The runtime's globals have no debug types: cast them (`p *(float*)&g_xmm3`, `p/x *(unsigned*)&g_esp`).

## Runtime switches

| Switch | Effect |
|---|---|
| **Frames and surfaces** | |
| `RECOMP_GL_DUMP=<prefix>[,N]` | Write every Nth presented frame as BMP (both back ends; Vulkan headless also writes the FXAA'd frame). |
| `RECOMP_VK_DUMP_SURF=<va>[,<va>...]` | With the frame dump, also dump these off-screen surfaces (shadow maps, glow targets, cube faces). |
| `RECOMP_FB_DUMP_FLIPS`, `RECOMP_VK_DUMP_RAW` | Guest framebuffer / raw surface dumps. |
| **Frame trace** | |
| `RECOMP_FRAME_TRACE=<flag file>` | One frame of draws, armed when the file appears (the runtime deletes it). Per draw: target, clip, depth/blend/cull state, raw blend registers, textures, texture-shader program, combiner control/factors, final combiner, alpha test, surface format, fog (enable, mode, colour, params), raw `SET_CONTROL0`, `ZMIN_MAX_CONTROL`, clip range, and the first vertices as the CPU interpreter places them. Arm it on the first heavy frame after a delay, not on time alone, or it lands on a loading screen. |
| `[FTRACE POS CPU]` lines (in a frame trace) | The CPU interpreter run over every index of each draw: how many land on screen, behind the eye, in the depth range, and their bounds. An estimate, not GPU output. |
| `[FTRACE VSH PROG/ATTR/CONST]` lines (in a frame trace) | For the first large programmable draw to a full-size target: the vertex-program tokens, three vertices' attributes and every non-zero constant. Enough to disassemble the program and test its matrices. |
| `RECOMP_GL_VSH_TRACE=1` (desktop OpenGL, with a frame trace) | Capture the generated vertex shader's `oPos` and `gl_Position` by transform feedback for large triangle draws to full-size targets (indices replayed as points with rasterisation discarded, then the real draw), printed as `[VSH GPU]`. |
| `RECOMP_PSH_DUMP=1` | Each new pixel-shader key (combiner stages, final combiner) and the GLSL generated for it. |
| `RECOMP_FRAME_TRACE_DRAWS=<lo>-<hi>` (conker-nx) | Frame-trace draws lo..hi of every frame (counted from 0 after each flip). Reaches inside a long frame that never flips, such as a loading screen, which a flag file armed at a flip cannot time. |
| `RECOMP_SURF_STATS=1` (conker-nx) | For every frame over 2,000 draws, `[surfstats]` lines: draws per target surface, the scan-out address (`PCRTC_START`) and how often it changed, control-method counts (`0x0100`–`0x01FF`, semaphores, other subchannels) and the first 48 runs of consecutive draws to one surface. |
| `RECOMP_SCANOUT_PRESENT=0` (conker-nx) | Turn off presenting the scanned-out buffer when a title draws into it without flipping (on by default). |
| **Frame bisection** | |
| `RECOMP_SKIP_DRAWS=<lo>-<hi>` | Drop draws lo..hi of every frame (counted from 0 after each flip). |
| `RECOMP_SKIP_BLEND=<src>/<dst>` | Drop every draw with that NV2A blend-factor pair (hex, e.g. `306/0` = `DST_COLOR`/`ZERO`). |
| `RECOMP_SKIP_INLINE=1` | Drop every inline-vertex draw. |
| `RECOMP_GL_DIRECT=0` | Convert vertex attributes through the interpreter's fetch instead of reading guest buffers directly: an A/B for vertex-format handling. |
| `RECOMP_VK_NO_DEPTH=1`, `RECOMP_VK_DEPTH_ALWAYS=1` | Disable depth test and writes / make enabled depth tests pass, on every draw. Screening probes only: they also change shadow passes and draw order. |
| `RECOMP_VK_DEPTH_ALWAYS_OPAQUE=1`, `RECOMP_VK_DEPTH_GREATER_OPAQUE=1` | The same, scoped to opaque draws to full-size targets. |
| `RECOMP_VK_DEPTH_TRACE=<first>-<last>` | Depth clears and opaque full-size draw depth state for that Vulkan frame range. |
| **Input** | |
| `RECOMP_PAD_SCRIPT=<ms>:<buttons>[:<hold>],...` | Scripted pad 1, timed from the title's first pad read. Buttons joined with `+`: `up down left right start back lthumb rthumb a b x y black white lt rt lsl lsr lsu lsd`. `RECOMP_PAD2_SCRIPT` drives pad 2. |
| `RECOMP_PAD_LIVE=<file>` | Lines appended to the file while the game runs are queued as presses (`buttons:hold_ms`; `&buttons:hold` starts at once; `clear` drops pending). With a frame dump running: look at the newest frame, append a press. Hold longer than one frame at the bench's current frame rate. |
| **Hangs, files, CPU** | |
| `RECOMP_WATCHDOG_SECS=<s>` (+ `RECOMP_WATCHDOG_KEEP=1`) | When the game stops making progress, print each thread's guest call stack (`tools/stackwalk.py` turns a raw guest-stack dump into a backtrace). |
| `RECOMP_FILE_TRACE=1` / `=2` | Every `NtCreateFile`: guest path, access, disposition, options, attributes, NT status and `Information`; `=2` adds the code addresses on the guest stack, i.e. which routine opened it. |
| `RECOMP_PB_EXEC=0` | No GPU executor: boot runs ~50× faster on the software path. Use it to follow file accesses. |
| `RECOMP_NATIVE=0` / `RECOMP_NATIVE_CHECK=1` | Disable / verify the native replacements of hot guest functions. |
| `RECOMP_NX_PROFILE=1\|2` | Sampling profiler on the console (conker-nx: also `[profile]` summaries in the log every 10 s, read by `tools/prof_log_report.py LOG ELF`). |
| `RECOMP_LINUX_PROFILE=1` (conker-nx) | Desktop sampler: `SIGPROF` on process CPU time, `[profile]` lines in 1 s windows plus `[profile-lib]` for library code by symbol; `prof_log_report.py --window-s 1`. |
| **Audio and logging** | |
| `RECOMP_APU_TRACE`, `RECOMP_APU_VOICES`, `RECOMP_AUDIO_DUMP`, `RECOMP_APU_VOICE_DUMP` | Audio traces and dumps. Never judge audio from a traced run: the trace itself causes gaps. |
| `*_LOG_SYNC=1`, `*_NO_LOG=1`, `RECOMP_QUIET` | Logging modes. |
| `XBOX_LOG_LEVEL`, `RECOMP_USB_STATS`, `RECOMP_RUMBLE_TRACE` | Subsystem logs. |

## Log signals and build options

| Signal | Meaning |
|---|---|
| `[ICALL] Failed to resolve VA 0x...` | Indirect call into a function the static pass missed. Seed it in the project's `config/seed_functions.json`, run the full regeneration. `python3 -m tools.seed_from_log run.log default.xbe --functions ... --seeds ...` proposes seeds from a log; review each one. |
| `[ICALL] target 0x00000000 is not code` | Null or wild function pointer, usually after an earlier failure. |
| `[STUB] 0x........` | First hit of a direct call into an undetected address (stubs added by the regeneration's stub-trace pass). |
| `[ABI] sub_X: esi edi esp (epilogue never ran)` | Build with `-DRECOMP_ABI_CHECK`: a callee changed callee-saved registers or the stack pointer, i.e. a split function. |
| `_flags` fallbacks in the generated C | Conditional jumps the translator could not pair with their compare. |
| `/* mnemonic ... */`, `/* FPU: ... */`, `/* SSE: ... */` lines in the generated C | Untranslated instructions: `grep -ohE '/\* (FPU|SSE): [a-z0-9]+' gen/*.c \| sort \| uniq -c`, plus the bare form. |
| `[hitch] frame N: X ms -- P programs compiled, T textures` | A slow frame and what it spent time on (shader compiles, texture uploads, draws). |
| `[fps] ... vblank ... Hz` | Frame rate and the emulated vblank rate (`RECOMP_FPS_LOG=1`). |
| `[VK] texture 0x... format 0x.. WxH not decodable (drawn magenta)` | A texture format without a decoder, once per format. A magenta object on screen is this. |
| `[VK] prewarm: N programs ..., M pipelines` | The shader/pipeline cache read at start-up. 0 on a second run means the cache is not being written (check `XBOXRECOMP_SWITCH_DIR`). |

The full regeneration (`tools/regen.sh`) is needed after seed or analysis changes; `LIFT_ONLY=1` re-lifts only,
which is enough after a translator change.

## Translator facts worth knowing

- XMM registers are shared thread-local state (`g_xmm0..7`), so floats the title passes in XMM registers
  between its own functions survive the call.
- The x87 stack is `g_fp_stack[8]` with `g_fp_top`; `fp_pop` only advances the top and never clears a slot,
  as on the real FPU. `fincstp`/`fdecstp` were missing until 2026-10-17 (they broke Conker's rotation
  matrices), and `fcmov*` and `fxtract` until later the same day; `ficom`, `fnsave`/`frstor` are still emitted
  as comments.
- Device-side physical addresses: the APU model (`apu_phys`) treats `[0x08000000, 0x80000000)` as VAs (memory
  above RAM that `MmGetPhysicalAddress` passed through); before 2026-10-17 it masked them into low RAM. The
  GPU's `dma_resolve` falls back to using such an offset as a VA.
- A texture whose address is a depth buffer (conker-nx, 2026-10-17): the Vulkan back end converts the depth
  image to the guest's Z24S8 bytes in a full-screen pass and samples that (`depth_as_texture`).
- Presentation (conker-nx, 2026-10-17): the back end presents on `FLIP_INCREMENT_WRITE`/`FLIP_STALL`, plus
  the buffer `PCRTC_START` points at when a draw lands in it 100 ms or more after the last flip (at most
  every 33 ms). Titles that draw loading screens into the front buffer without flipping (Conker) need the
  second path.
- Vertex formats (conker-nx, 2026-10-17): with `NV2A_BACKEND_RAW_CMP` the executor hands CMP (11:11:10)
  normals to the Vulkan back end as stored and the shader unpacks them; formats the device cannot fetch are
  fetched as a neighbour (3→4 components with w = 1, D3DCOLOR as RGBA) and fixed up by `nv2a_inputs()` in
  the generated vertex shader. The start-up log lists the NV2A formats the device cannot fetch.
- The Switch shader/pipeline cache folder is `XBOXRECOMP_SWITCH_DIR` (a compile definition, e.g.
  `"sdmc:/switch/conker/"`); the default is Burnout 3's `sdmc:/switch/b3x/`.
- `lahf` after `ucomiss`/`fcomi` and the `test ah, 0x44` + `jp` idiom are translated through explicit flag
  bytes; a missing carry-flag declaration after `lahf` once broke the build (`_cf undeclared`).
