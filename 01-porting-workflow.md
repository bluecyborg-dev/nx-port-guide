# Porting workflow

A homebrew port is a compatibility project: preserve the original game's code and data while replacing the original console services with a runtime the Switch can host. First identify which layer owns a failure, then make the smallest change that addresses it.

## Keep three layers separate

| Layer | Typical responsibilities | Port-specific work |
|---|---|---|
| Recompiler/toolkit | Executable parsing, instruction translation, guest register model, function dispatch | Discover missed function entries; report/fix translation bugs with a minimal reproduction |
| Original-platform runtime | Kernel, memory, files, threads, audio/input and original graphics API behavior | Game paths, image layout, stack/data addresses, device hooks and per-title quirks |
| Switch host | libnx startup, Horizon memory/thread services, display/input/audio and host graphics backend | Select the renderer and build configuration; verify real memory behavior, scheduling and performance on hardware |

Keep executable or GPU details within their source platform. Original Xbox x86/NV2A material does not describe Xbox 360 PowerPC/Xenos PM4 behavior. Shared Switch observations can inform both, but a measured gain in one port is not a guarantee for another.

## A repeatable sequence

1. **Inventory the title.** Record executable identity/hash, region, imports, entry point, sections, executable sections outside the main text section, memory end, data layout and required game files. Keep the extracted tree intact.
2. **Understand the translation pipeline.** Parse the executable, disassemble code, identify functions and calling conventions, review callbacks/vtables/jump tables, then lift. Inspect unsupported instructions and unresolved indirect targets.
3. **Set safe guest memory.** Choose stack, runtime-data and heap regions from the actual image/BSS layout. Trace aliases and device apertures explicitly. Do not assume desktop mappings behave like Horizon.
4. **Build a minimal boot path.** Initialize host services, map guest memory, dispatch the original entry point and route file access to extracted assets. Add only compatibility hooks supported by executable evidence or runtime traces.
5. **Reach the title screen.** Resolve fatal errors, missing functions, graphics initialization, audio startup and movie timing. Check save/cache directories and error handling.
6. **Reach gameplay and verify behavior.** Test input, audio, saves, movies, networking/offline paths, simulation timing, menus, resolution and controller options. Capture graphics state when the first incorrect frame appears.
7. **Profile on the Switch.** Separate game-thread, renderer-thread, GPU, storage and waiting costs. Change one variable at a time and retain the old build for comparison.
8. **Record durable findings.** Keep a per-game logbook of symptoms, evidence, fixes and which build/device verified each result. Move general compatibility discoveries into shared guidance only after validating their scope.

## Engineering gates and expected evidence

Treat each step as a gate with an artifact that can be reviewed. This prevents an observation such as
"the title opened" from silently becoming evidence that the renderer, saves, or hardware path also work.

| Gate | Evidence to retain | Exit condition |
|---|---|---|
| Target inventory | Executable/data hashes, region/revision, section map, imports, memory end, asset inventory | The analysis covers every code-bearing section and identifies runtime inputs. |
| Translation | Generator version/options, function/ABI reports, unresolved target list, untranslated-instruction census | Missing entries and instruction fallbacks have an owner and a verified disposition. |
| Runtime boot | Guest stack, host/service logs, mapping map, file-operation trace, build identity | Entry point reaches a stable title state without unresolved calls, invalid memory, or accidental overlaps. |
| Functional paths | Repeatable input route, fresh/warm storage cases, input/audio/movie/save observations | Menus and representative gameplay paths complete without hidden dependencies on stale state. |
| Graphics | Deterministic frame/command capture, intermediate targets, shader/state identifiers | The first incorrect operation is localized; a final screenshot alone is not a diagnosis. |
| Hardware | Exact binary pair, launch mode, logs, device mode, settings, stock clocks | Switch-only behavior and performance claims are supported by a hardware run. |
| Optimization | Baseline and candidate builds, correctness comparison, stage measurements, rollback switch | A candidate improves the measured critical path without unacceptable behavior changes. |

Use explicit status words in logs: **observed** means directly present in the artifact; **inferred** means
an explanation consistent with evidence; **hypothesis** means an experiment remains; **verified** names
the exact build and environment that passed. A result from a desktop renderer must not be described as
hardware verification.

## Subsystem contract inventory

For every host bridge, write down the guest-visible contract before implementing a special case:

- Inputs and outputs, including registers, stack arguments, alignment, sizes, and ownership.
- Whether the call blocks, wakes a thread, releases a guest lock, or changes interrupt/IRQL state.
- How failure is reported, including status-code distinctions and partial completion.
- Whether the guest can observe ordering, timing, persistent state, or a side effect on another device.
- Which addresses are guest RAM, aliases, device apertures, shared mappings, or host-owned buffers.

This contract applies to files, memory, events/timers, graphics fences, audio, controller input, movies,
and save/cache APIs. It helps decide whether a bug belongs in the shared runtime, platform layer, guest
translation, or a title-specific hook.

## Function discovery and manual hooks

Static function discovery can miss thread entry points, callbacks, class-driver tables, code after padding, or indirect-call targets. A seed list is a hypothesis about function boundaries: validate each address against bytes, references and surrounding control flow. A seed inside a real function can split it and create hard-to-understand failures.

A manual native replacement or wrapper must document its calling convention, inputs/outputs, stack cleanup, guest-state effects and evidence. Keep a comparison mode where feasible so the replacement can be checked against the translated implementation before trusting it.

## Graphics-debug order

For a wrong frame, capture the command stream/frame trace and identify the first incorrect target or state. Check guest framebuffer address, format, tiling, aliasing, clear/resolve order and texture contents before changing shaders. Then inspect translated shader state and host API resource use. An image that looks wrong at the final output can originate in an earlier surface or data upload. It can also originate in the CPU: geometry projected off screen on every back end usually means the title computed bad transform constants in translated code. Check that its camera and world matrices are well formed (orthogonal rows, equal row lengths) before changing clip, cull or depth rules ([testing and debugging](05-testing-and-debugging.md#graphics-correctness)).

## Data and provenance

Keep proprietary game images, extracted assets, generated translated code and disc-derived captures out of source control. Keep project logs separate from generic guidance. Preserve whether a statement is a measured result, a title-specific observation, a proposal, or an estimate; do not promote an estimate into a benchmark.
