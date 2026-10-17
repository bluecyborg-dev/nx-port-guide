# Development and build practices

## Choose tools for the original platform

The source notes describe two distinct families:

| Original platform | Supplied example | Guest/toolchain shape | Graphics path in the notes |
|---|---|---|---|
| Original Xbox | Conker: Live & Reloaded | x86 static recompilation with `xboxrecomp`; XBE parsing, function discovery, ABI analysis and lifted C | Original D3D8 writes NV2A pushbuffers; executor and `nv2a_vk` translate for Vulkan/NVK |
| Xbox 360 | Call of Duty: Black Ops | PowerPC static recompilation using ReXGlue (XenonRecomp is a separate Xbox 360 recompiler family used by other ports) | Xenos PM4 translation into a native renderer/backend |

Do not select a recompiler or renderer based on the shared console family name. The guest instruction set, graphics hardware and command format differ.

## Original-Xbox build pieces in the supplied notes

The Conker notes use devkitPro's devkitA64, libnx and switch-sdl2, with a Mesa NVK static library and a Switch-built glslang because NV2A shaders are generated and compiled at runtime. Linux/container builds with SDL2 and lavapipe are used as a correctness bench. A build also needs the title's runtime glue, generated translation output and assets staged separately.

For this path, the notes describe `tools/regen.sh` for analysis/lifting and `switch/build.sh` for the NRO build. A full regeneration is needed when function seeds or discovery changes; a lift-only pass is for translator/runtime changes that do not alter discovery. Generated C is a build artifact, not a hand-edited source file.

## Build host and containers

The supplied projects use Linux/WSL or containers when large lifted code or a Linux Vulkan test target is more practical. A container can provide a repeatable compiler environment and isolate tool dependencies; mount the checkout at a stable path so CMake/Ninja caches remain valid. Give the container enough memory: large generated translation units can use 1–2 GB per compiler job in the cited BO1 setup, and the Conker build reports about 2 GB per link job with LTO. Limit parallel jobs based on available RAM.

Use separate build directories for native and container builds, different drivers, or experimental flags. Preserve the exact unstripped ELF corresponding to every console NRO so crash addresses and profiler samples resolve against the correct build. Check the final NRO entry-point/link layout required by the selected loader.

The BO1 notes document its Docker build workflow; the Conker notes adapt that process but mark it as not yet tested for Conker. Treat that distinction as project-specific status, not proof that the two build paths are interchangeable.

## Libraries and renderer dependencies

- **libnx/devkitA64:** Switch process, services and executable packaging for homebrew.
- **SDL2 (Switch build):** used in the Conker notes for window, audio and pad support. Filter or stub unrelated host graphics/ELF pieces when the selected Vulkan build does not use them.
- **Mesa NVK on Horizon:** a Vulkan backend for the Switch GPU in these projects; pin and record the exact source revision and patches.
- **glslang for Switch:** needed when the NV2A backend compiles generated GLSL shaders at runtime.
- **FFmpeg:** optional only when needed for a title's media. The Conker notes explicitly require a license-compatible build and warn that the standard devkitPro package they inspected was GPL-enabled. Verify licenses of every bundled component.
- **Lavapipe/desktop SDL:** useful for correctness and repeatable captures, but not for Switch performance.

Do not add a dependency just because another port uses it. Check its license, target architecture, build configuration and whether it is actually called on Switch.

## Practices that keep builds understandable

- Record compiler/toolchain, driver revision, patches, flags and build directory for each output.
- Keep generated code, game images/assets, NRO staging folders, profiles and captures out of source control unless explicitly suitable for distribution.
- Use named configuration switches for performance experiments and log their effective values at startup.
- Separate host-only stubs from code expected to run on Horizon.
- Keep patches and toolkit changes traceable to an upstream source or local fix; note which fork supplied merged behavior.
- Maintain a small clean build and a repeatable full regeneration path. Do not hand-edit generated translation output.

## Licensing and source boundaries

The supplied Conker notes identify xboxrecomp as MIT with specific xemu-derived LGPL components; they also warn that xemu's GPL-derived renderer code and nfsmw-nx files have different licenses. The BO1 notes likewise call out GPL-licensed renderer code and LGPL-only media alternatives. These are source-document assertions; verify the exact license files for the version actually vendored before distribution. Do not copy code across projects until its license permits it.

Keep game executables, disc images, extracted assets and disc-derived captures out of the repository. This guide's source notes are documentation copies only.
