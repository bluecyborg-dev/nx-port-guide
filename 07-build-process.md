# Build process

Build commands depend on the original console and the chosen port runtime. The supplied Conker and Black Ops procedures use different recompilers, generated-code formats, graphics paths and build arrangements. Use the matching game procedure; do not combine commands or prerequisites across them.

## Shared build discipline

1. Start from the intended branch and inspect the working tree so source changes, generated files and test evidence are attributable to a build.
2. Keep game assets, generated translation output, build directories and run evidence separate from maintained source. They are usually large and should be excluded from version control.
3. Regenerate guest code only after changes to the executable, function seeds, discovery/ABI analysis or translation pipeline. Ordinary runtime/renderer edits should use the incremental native build.
4. Build a desktop/container correctness target before the console target where available. Desktop results do not prove Horizon memory behavior, target GPU behavior or performance.
5. Archive the exact unstripped ELF and NRO for every console run. Crash addresses and profiler samples refer to that build.
6. Record toolchain, compiler options, driver revision/patches, active feature settings, target mode and test scene with performance results.

## Original Xbox: Conker procedure

The supplied Conker process uses `xboxrecomp` to analyze the XBE and lift x86 functions to generated C. `tools/regen.sh` performs disassembly, function identification, ABI analysis and translation. Do a full regeneration when function seeds change; the source notes describe `LIFT_ONLY=1` for translation-only changes that do not require rediscovery.

The local `conker-dev` container builds an AArch64 Linux correctness target and runs Vulkan through lavapipe. It is useful for startup, guest progress, file/cache behavior, crashes and many graphics checks, but it does not validate Horizon mappings, Switch NVK or console speed.

The documented Switch build is:

```sh
VULKAN=1 STAGE_GAME=0 bash switch/build.sh
```

It uses devkitPro/devkitA64, libnx, switch-sdl2, a Switch NVK library and Switch glslang. `STAGE_GAME=0` avoids copying the large extracted game tree. Verify `_start` in the ELF is at address zero, then keep that ELF beside the NRO used for the test. The detailed Linux build, build outputs, runtime staging and console sequence are in [the Conker build process](platforms/xbox/conker/build-process.md).

The Conker source notes discuss adapting a Docker build path but explicitly say it had not yet been tried for Conker. Treat that route as unverified unless it has since been tested in the target checkout.

## Xbox 360: Black Ops procedure

The supplied Black Ops process builds with Docker. The stages are:

1. Build host tools and fetch the SDK's third-party dependencies.
2. Run `tools/codegen.sh` on the user's own `default.xex`; generated C++ is under `app/generated/` and is not committed.
3. Build the Horizon NVK driver with `mesa/build_mesa_docker.sh` when its source pin, patches or optimization level changes.
4. Run `tools/build_nro.sh` for the incremental CMake/Ninja build inside `devkitpro/devkita64`, producing an NRO and symbol-bearing ELF.

The detailed procedure documents `MESA_SDK`, `NRO_OUT`, `JOBS`, `PLAIN_MEMORY`, `CMAKE_EXTRA` and `IMAGE`, recommends an arm64 Docker image on Apple Silicon, and, after the CMake build (which also packages the NRO with `nacptool` and `elf2nro`), fails the build unless `_start` is at address zero. The container needs enough memory for generated translation units; the source notes use four parallel jobs with Docker Desktop at about 8 GB. See [the Black Ops build process](platforms/xbox-360/black-ops/build-process.md) for setup, linking and troubleshooting details.

## Compare before reusing

The build instructions above are project examples, not universal homebrew requirements. Conker is original Xbox x86 with NV2A; Black Ops is Xbox 360 PowerPC with Xenos PM4. Their generator, CMake targets, dependency licenses, driver configuration, game asset paths and output names differ. For another title, carry over the staged-build idea and verification discipline, then confirm each concrete tool and command against that port's repository.

## Build artifacts and invalidation map

Keep a manifest for each testable binary with source revision, executable/data identity, analysis and
generator revisions, build flags, toolchain, driver provenance, package hash, and output paths. This is
especially important when generated code or a local driver archive lives outside the source tree.

| Input changed | Recompute |
|---|---|
| Executable or function seed | Function analysis and the full dependent guest-code generation, then host build. |
| Translator semantics | Focused translator checks, affected generated code, host build, and behavior comparison. |
| Runtime or renderer source | Incremental host build and the relevant desktop/hardware validation. |
| Driver source, patch, or optimization option | Driver build, then relink the NRO and record the new driver provenance. |
| Link configuration or NRO metadata | Relink/package and verify startup layout, output hash, and that the staged binary is the one sent. |

Never infer which configuration Ninja/CMake used from the command alone. Inspect the selected cache,
driver archive path, compiler, and effective runtime settings. A changed static library must be a declared
link dependency or the target may remain linked to the old archive.

## Build success is not runtime evidence

The validation chain has separate claims:

1. **Compilation:** the compiler and linker completed.
2. **Packaging:** the ELF was converted to an NRO with expected metadata/startup layout.
3. **Desktop execution:** the matching target-architecture build progressed through a known route.
4. **Console execution:** the archived NRO actually started on a Switch and produced a captured log.
5. **Feature verification:** a named behavior (rendering, save, input, audio, performance) was observed in
   that exact environment.

Report only the highest stage actually evidenced. A valid NRO header, a changed timestamp, or a successful
desktop frame does not establish that the console ran that binary.

The Switch-facing dependency, cache, title-takeover, network-upload, and log-capture practices continue in
[the detailed Switch build and package workflow](#detailed-switch-build-and-package-workflow) and
[Switch console validation and deployment](05-testing-and-debugging.md#switch-console-validation-and-deployment).

## Detailed Switch build and package workflow

This is a reusable Switch-facing build and validation process based on the supplied Conker and Black Ops
project procedures. It describes common stages and checks, not a single universal build command. Follow
the selected recompiler, runtime, graphics backend, and game's own build instructions for exact tools and
flags.

The root [build process](07-build-process.md) compares the supplied project pipelines. The game
folders contain their concrete build recipes:
[Conker](platforms/xbox/conker/build-process.md) and
[Black Ops](platforms/xbox-360/black-ops/build-process.md).

### 1. Keep build inputs and outputs attributable

Before building, record the intended branch/commit and inspect the working tree. Separate maintained
source from game data, generated guest code, build trees, SD staging, caches, profiles, and run evidence.
Do not clean or regenerate away someone else's local work. Store large and proprietary data outside source
control unless distribution is explicitly allowed.

Use distinct build directories for materially different configurations: Linux versus Switch, GL versus
Vulkan, driver revisions, optimization variants, and diagnostic builds. A CMake cache remembers absolute
paths and toolchains; reusing one across different container mounts or compilers can silently keep stale
settings.

Record a build manifest for every hardware artifact:

| Field | Why it matters |
|---|---|
| Source commit and local diff identity | Identifies the exact source state. |
| Executable/data identity or hash | Distinguishes source versions and asset sets. |
| Recompiler/generator revision and generated-code status | Shows whether the guest code matches the current translator/seeds. |
| Compiler, target triple, flags, link options, and toolchain image | Explains code generation and packaging. |
| Mesa/NVK revision, patches, build options, and dependency versions | Driver behavior can change independently of the game. |
| Build directory/configuration and active runtime options | Prevents stale-cache or disabled-option confusion. |
| NRO hash and matching unstripped ELF | Needed to resolve crash PCs, profiler samples, and build identity. |
| Test log, launch mode, device mode/clocks, save/cache state, and scene | Makes the run reproducible and interpretable. |

Keep the unstripped ELF and NRO as a pair. A profiler report or crash address from one build cannot be
reliably symbolized with the ELF from another build, even if the filenames are similar.

### 2. Rebuild only the layers affected by the change

Use the dependency graph rather than rerunning everything by habit:

| Changed input | Typical required work |
|---|---|
| Game executable, analysis options, function boundaries/seeds, or ABI discovery | Reparse/re-disassemble as required, redo function discovery/ABI analysis, regenerate all affected guest code, then rebuild. A new seed can change neighboring boundaries. |
| Instruction translator or code-generation semantics | Run the translator's focused correctness coverage, regenerate the affected guest output, rebuild both diagnostic and console targets, and retain the previous build for comparison. |
| Runtime hook, kernel bridge, filesystem, scheduler, input/audio, or renderer source | Incremental native build, then the relevant desktop correctness pass and hardware validation. No full guest-code generation is needed unless the pipeline input changed. |
| Driver source pin, patch, compiler options, or driver dependency | Rebuild the driver in its own environment, record the exact source/patch identity, then relink the homebrew target. |
| NRO metadata, link order, startup objects, or packaging script | Re-run the package step and verify the produced executable metadata and entry layout. |
| Game assets or save/cache contents | Keep the asset build/data identity separate; repeat both fresh-state and existing-state tests where behavior depends on caches. |

Generated source should be treated as reproducible output. Fix the translator or its inputs, not the
generated C/C++ by hand, unless the project explicitly uses a maintained patch step.

### 3. Build host and dependency controls

Large static recompiles can generate translation units that use 1–2 GB of compiler memory per parallel
job in the supplied examples. Select `JOBS` from the memory limit of the actual host/container rather than
the CPU core count. An out-of-memory kill may not produce a clear compiler diagnostic.

Containers help keep the compiler and Switch SDK consistent. On Apple Silicon, prefer an ARM64 image so
build tools do not run under emulation. Mount the repository at a stable absolute path if build-system
caches record source paths. Use a separate build directory when changing the container/toolchain path.

Pin and document the graphics driver, shader compiler, portlibs, linker, and any local patches. Rebuild a
custom Mesa/NVK library only when its source pin, patches, or optimization configuration changes; otherwise
use the exact existing driver package. Keep a provenance file beside a built driver describing its source
revision, patches, and options.

Check licenses before adding or copying code. The supplied projects have different license boundaries for
runtime code, GPU emulator references, renderer implementations, media libraries, and third-party
patches. Behavioral study and code copying have different obligations; retain required notices and do not
assume two ports share the same license just because they target the same console.

### 4. Package and inspect the NRO

Switch homebrew toolchains commonly link an ELF and package it with NRO metadata/icon tools. The specific
projects use `nacptool` and `elf2nro`; retain the unstripped ELF and package a separate stripped copy when
the NRO should omit symbols.

Where the selected linker/loader path requires it, verify the startup symbol's placement (`_start` at
address zero in the supplied examples) after linking. Also verify that the NRO timestamp, size, and hash
change when a new build is expected, and that the upload script selects the intended NRO. A successful
link or valid NRO header proves packaging only; it does not prove the app starts on a Switch.

For a custom Vulkan runtime, check the actual static-link dependencies and archive order. The supplied
NVK builds link a static Vulkan archive and dependencies such as expat, zlib/zstd, and libnx; some SDK
config packages pull optional libraries that are unavailable on Switch, requiring small host-only stubs
where those APIs are never executed. Keep such stubs isolated from the code that runs on Horizon.
