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

The detailed procedure documents `MESA_SDK`, `NRO_OUT`, `JOBS`, `PLAIN_MEMORY`, `CMAKE_EXTRA` and `IMAGE`, recommends an arm64 Docker image on Apple Silicon, and checks `_start` before packaging. The container needs enough memory for generated translation units; the source notes use four parallel jobs with Docker Desktop at about 8 GB. See [the Black Ops build process](platforms/xbox-360/black-ops/build-process.md) for setup, linking and troubleshooting details.

## Compare before reusing

The build instructions above are project examples, not universal homebrew requirements. Conker is original Xbox x86 with NV2A; Black Ops is Xbox 360 PowerPC with Xenos PM4. Their generator, CMake targets, dependency licenses, driver configuration, game asset paths and output names differ. For another title, carry over the staged-build idea and verification discipline, then confirm each concrete tool and command against that port's repository.
