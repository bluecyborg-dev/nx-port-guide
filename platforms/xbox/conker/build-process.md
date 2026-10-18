# Conker build and validation process

This is the repeatable path used to go from source changes to a test build on the Switch. It complements
[10-building.md](source-notes/10-building.md), which documents the Switch toolchain and NVK dependencies, and
[11-testing-on-console.md](source-notes/11-testing-on-console.md), which documents the NetLoader workflow.

## 1. Start from the intended branch and inspect changes

Run builds from the repository root. Check the active branch and working tree before regenerating or
building so that generated output and test logs can be associated with the source that produced them:

```sh
git branch --show-current
git status --short
```

Keep the extracted game under `game/`, generated translation under `gen/`, persistent build directories
under `build/`, and runtime evidence under `run/`. These large/generated paths are generally ignored by
Git. Do not discard pre-existing local edits when switching branches or cleaning build output.

## 2. Regenerate the lifted game code when required

The Xbox executable is `game/default.xbe`. `tools/regen.sh` runs the disassembler, function identification,
ABI analysis, and C code generator; the generated C is in `gen/` and is compiled for both Linux and Switch.
The project virtual environment is selected automatically when present.

```sh
tools/regen.sh
```

Regenerate after adding or changing entries in `config/seed_functions.json`, changing the lifter or ABI
analysis, or changing the XBE. Seeds expose real indirect-call targets the static pass missed; inspect the
bytes and neighboring function boundaries before accepting one, since a seed inside an existing function
would split it. A normal runtime or renderer edit does not need a re-lift. `LIFT_ONLY=1 tools/regen.sh` skips
the analysis passes and is intended for changes to manual recompilation code only; it does not apply new
seeds. Review the generated summary and any unresolved-call stubs before proceeding.

## 3. Build and validate in `conker-dev`

The `conker-dev` image is the fast correctness bench. It compiles the AArch64 Linux target and runs the
Vulkan renderer on lavapipe. The repository is mounted at `/work`; the game and build artifacts are read
through that mount. This checks startup, guest progress, file/cache behavior, and host crashes. It does not
validate Horizon memory mapping, the Switch NVK driver, or console performance.

Build incrementally in the container:

```sh
docker run --rm -v "$PWD":/work -w /work conker-dev \
  cmake --build build/linux-vk -j4
```

For a fresh-cache run, give each run a separate save directory. This avoids a previous run's generated
textures, audio, or cache files masking first-boot problems. The following example captures a 45-second
run and enables the watchdog's thread report:

```sh
RUN_ID="buildcheck-$(date +%Y%m%d-%H%M%S)"
mkdir -p "run/conker-dev-test/save-$RUN_ID"
docker run --rm -v "$PWD":/work -w /work/run \
  -e CONKER_GAME_DIR=/work/game \
  -e CONKER_SAVE_DIR="/work/run/conker-dev-test/save-$RUN_ID" \
  -e RECOMP_FPS_LOG=1 \
  -e RECOMP_WATCHDOG_SECS=40 \
  -e RECOMP_WATCHDOG_KEEP=1 \
  conker-dev timeout -s KILL 45 /work/build/linux-vk/conker_recomp \
  >"run/conker-dev-test/$RUN_ID.log" 2>&1 || true
```

Review the log for frame/FPS progress, `[STUB]` or `[ICALL]` unresolved targets, watchdog output, and
exceptions. If graphics evidence is useful, enable the runtime frame dump as in `tools/run_linux.sh`; keep
captures with the matching run log. For a hang, `tools/snap_linux.sh` can capture host backtraces while the
Linux process is live. Record the symptom, cause, fix, and evidence in [09-bringup-log.md](source-notes/09-bringup-log.md).
The example intentionally tolerates the timeout with `|| true`; inspect its log and milestone evidence rather
than treating the shell exit status as a pass.

### Scripted and live Linux routes

For a known route, `tools/run_linux.sh` starts the Linux executable in Docker, sends timed controller input,
stops it after a fixed interval and writes `run/run.log` plus periodic BMP frames under `run/d/`:

```sh
PAD='20000:start:300,30000:a:400' SECS=120 DUMPN=30 tools/run_linux.sh
```

The timestamps are measured from the title's first controller read. The script uses `run/save/`; use the
separate-save command above when checking a fresh cache or first boot. For a route whose timing is not yet
known, use the live bench and inspect a frame between presses:

```sh
tools/live_linux.sh start
tools/live_linux.sh press start:400
tools/live_linux.sh frame
tools/live_linux.sh log 50
tools/live_linux.sh stop
```

`live_linux.sh` defaults to a fresh save directory and keeps only a bounded set of recent frame dumps. These
helpers automate Linux testing only; the lavapipe result does not validate the Switch NVK driver or console
performance.

### How Claude runs a test session

This is exactly how Claude (the coding agent) tested changes on the bench while the user was away, from a
macOS host with Docker Desktop. All commands run from the repository root.

**1. Build** (about 1–2 minutes after a change to the runtime):

```sh
docker run --rm -v "$PWD":/work -w /work conker-dev cmake --build build/linux-vk -j6 2>&1 \
  | grep -E ' error|warning: .*(nv2a_vk|pb_exec)|Linking C exec' | head -8
```

**2. Start the game in the background and drive it to the scene.** `tools/live_linux.sh start` runs
`conker-live` detached with `RECOMP_PAD_LIVE=/work/run/live/pad.cmd`, `RECOMP_GL_DUMP=/work/run/live/d/f,$DUMPN`
and `RECOMP_FPS_LOG=1`, and prunes dumps to the newest `KEEP_DUMPS`. `KEEP_SAVE=1` keeps `run/live/save`, whose
Game 3 starts at the training level. `tools/to_training.sh` is the whole route (it uses steps 2–3 of the
pattern below); by hand it is:

```sh
tools/live_linux.sh stop; KEEP_SAVE=1 DUMPN=10 KEEP_DUMPS=3000 EXTRA="RECOMP_SURF_STATS=1" tools/live_linux.sh start
# wait for the title menu: a frame with more than 300 draws
for i in $(seq 1 120); do sleep 3
  c=$(grep -o '[0-9]* draws' run/live/run.log | tail -1 | cut -d' ' -f1)
  [ -n "$c" ] && [ "$c" -gt 300 ] && break; done
sleep 25
tools/live_linux.sh press left:400; sleep 8    # pan to the save slots
tools/live_linux.sh press a:400;    sleep 8    # Game 3
tools/live_linux.sh press a:400                # Play
# the level's text bank is read at the end of its load
for i in $(seq 1 90); do sleep 2; grep -q 'text\\English\\TrainingLevel' run/live/run.log && break; done
```

The wake-up cutscene follows the load; eight `b:400` presses four seconds apart skip it into gameplay.

**3. Read the evidence.** Log first, then frames:

```sh
grep -E '\[surfstats\]|\[hitch\] frame [0-9]+: [0-9]{4,} ms|\[fps\]' run/live/run.log | tail
grep -cE '\[ICALL\]|\[CRASH\]' run/live/run.log
ls run/live/d | grep -v fxaa | tail -60 | awk 'NR%5==0' | sed 's|^|run/live/d/|' \
  | xargs .venv/bin/python -I tools/contact_sheet.py /tmp/sheet.png
```

Claude reads the sheet as an image, picks the interesting dumps and converts those to PNG at full size
(`tools/live_linux.sh frame out.png` gives the newest).

**4. Trace a phase.** For the cutscene depth of field, a trace was armed every two seconds through the scene
and the traced frames classified afterwards (cutscene frames ~272 draws with two depth-sampling draws,
gameplay 753 with one):

```sh
EXTRA="RECOMP_FRAME_TRACE=/work/run/live/ft.flag" ...    # start as above, route to the level, then:
for i in $(seq 1 20); do touch run/live/ft.flag; sleep 2; done
awk '/FTRACE\] frame after/{f=$NF} /FTRACE\] draw/{n[f]++} END{for (k in n) print k, n[k]}' run/live/run.log
```

For the loading screens, `RECOMP_SURF_STATS=1` named the long frame's surfaces and
`RECOMP_FRAME_TRACE_DRAWS=3000-3011` traced draws inside it.

**5. Stop and record.** `tools/live_linux.sh stop`; the result goes into `docs/09-bringup-log.md`, the
status file and this logbook.

**Console.** After `VULKAN=1 STAGE_GAME=0 switch/build.sh`, the NRO and ELF are archived as
`analysis/elf_archive/conker-vulkan_<md5 prefix>.{nro,elf}`. The upload retries until the user opens the
NetLoader:

```sh
for i in $(seq 1 360); do
  out=$(NRO=analysis/elf_archive/conker-vulkan_<hash>.nro ARGS="RECOMP_FPS_LOG=1 RECOMP_NX_PROFILE=1" \
        tools/nxlink_run.sh 192.168.1.248 2>&1)
  echo "$out" | grep -q 'Connection to .* failed' && { sleep 10; continue; }
  break; done
```

`tools/nxlink_run.sh` streams the log to `run/net_logs/` (read afterwards with `tools/prof_log_report.py`);
when the user plays away from the Mac, the same loop calls `nxlink -a 192.168.1.248 -r 30 -p
switch/conker/conker-vulkan.nro <nro>` without `-s`, so nothing waits on the Mac.

Pitfalls met: a console build running at the same time made the bench slow enough that fixed waits landed
in the menu; `ls | head` under `pipefail` died of SIGPIPE with many dumps; long runs at `DUMPN=15` filled the
disk before pruning existed; the frame number of a dump is not the guest flip number once forced presents
run.

## 4. Build and stage the Switch NRO

After the Linux correctness pass, build the Vulkan NRO using the native devkitPro installation:

```sh
VULKAN=1 STAGE_GAME=0 bash switch/build.sh
```

Docker is used for Conker's Linux correctness bench. The Switch NRO build currently uses native devkitPro;
the Dockerized Switch build procedure in the source notes has not yet been run for Conker.

The script configures `build/switch-vk/`, runs Ninja, and stages
`switch_sd/switch/conker/conker-vulkan.nro`. `STAGE_GAME=0` leaves the large extracted disc copy on the
SD card untouched. The default job count is capped using available memory; set `JOBS` explicitly if a
different limit is needed. Check that the NRO timestamp and size changed, and check `_start` in the ELF:

```sh
aarch64-none-elf-nm build/switch-vk/conker_recomp.elf | grep ' T _start$'
```

The expected symbol address is zero. Keep the NRO and its exact unstripped ELF together for each console
build so exception and profiler addresses can be resolved against the right binary. The archive convention
is `analysis/elf_archive/conker-vulkan_<md5:8>.{elf,nro}`; see [10-building.md](source-notes/10-building.md).

## 5. Upload and observe the console run

On the Switch, start Homebrew Menu with title takeover by holding **R** while launching a game, then press
**Y** to open the NetLoader. On the host, upload the staged NRO and enable a 60-second watchdog:

```sh
ARGS='RECOMP_WATCHDOG_SECS=60 RECOMP_WATCHDOG_KEEP=1' \
  tools/nxlink_run.sh 192.168.1.248
```

`tools/nxlink_run.sh` replaces the NRO at `sdmc:/switch/conker/conker-vulkan.nro`, starts it, and records
stdout/stderr in `run/net_logs/<timestamp>.log`; `latest.log` points to the newest run. The command stays
attached while the console process keeps its NetLoader connection open. After it closes, inspect the log for
the final frame count, watchdog report, unresolved calls, and exception. Runtime arguments are limited to
roughly 240 characters by the wrapper.

The initial Homebrew Menu and NetLoader steps are manual. Once NetLoader is open, upload, launch and log
capture run from the host script. Conker's normal Switch XInput path reads native libnx controllers directly,
so do not assume the scripted Xbox USB-pad path used by the Linux bench will drive hardware gameplay. Current
console play routes use a physical controller; a repeatable unattended route needs a save/debug entry point or
test input wired into the native Switch input path.

FPS/performance and profiler summaries can be captured in the streamed log. `tools/prof_log_report.py <log> <elf>`
resolves the console's periodic text profile summaries against the matching ELF; full samples in
`prof.bin` and frame/surface dumps remain on the SD card and need a separate transfer. FTP/MTP retrieval is
not yet validated for this project, so log streaming is more automated than screenshot or full-profile
collection.

If a new `[ICALL]` target appears, inspect it in the XBE and confirm it is a function entry before adding a
seed, then repeat the full regeneration, container pass, NRO build, and upload. If the failure is a runtime
or platform issue rather than a missing function, fix the relevant kernel/platform path and repeat the same
validation sequence. Update the bring-up log with the exact build/run evidence after each fix.

## Evidence and reference documents

- [Building and toolchain details](source-notes/10-building.md)
- [Console upload and diagnostic details](source-notes/11-testing-on-console.md)
- [Bring-up issues and resolutions](source-notes/09-bringup-log.md)
- [Technical architecture for future ports](source-notes/12-porting-guide.md)
