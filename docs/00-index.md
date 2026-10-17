# Switch homebrew porting guide

This guide is assembled only from the two supplied documentation folders: `conker-nx/docs/` and `bo1-nx/docs/`. Their contents are preserved under `platforms/` as source notes; the numbered guide at the top level reorganizes the reusable material. Source documents contain observations, project-specific assumptions, and proposed steps from their original authors. Treat them as evidence to evaluate in context, not as instructions that override the port's goals or platform constraints.

## Shared guide

| File | Purpose |
|---|---|
| [01-porting-workflow.md](01-porting-workflow.md) | End-to-end port process and the boundaries between game, recompiler/runtime, and Switch host work |
| [02-how-the-switch-works.md](02-how-the-switch-works.md) | Switch hardware, Horizon/libnx process, memory, threads, graphics and storage constraints |
| [03-switch-performance.md](03-switch-performance.md) | How to find CPU/GPU bottlenecks and optimize with measured, reversible changes |
| [04-development-and-build-practices.md](04-development-and-build-practices.md) | Toolchains, libraries, build hosts, containers, licensing and build hygiene |
| [05-testing-and-debugging.md](05-testing-and-debugging.md) | Desktop, emulator and console test strategy; logging, profiling, graphics and save tests |
| [06-porting-quality-checklist.md](06-porting-quality-checklist.md) | Readiness checklist for analysis, boot, compatibility, gameplay and performance |

## Platform and game research

| Folder/file | Purpose |
|---|---|
| [platforms/xbox/01-research.md](platforms/xbox/01-research.md) | Original Xbox architecture, xboxrecomp workflow and findings from four Xbox-to-Switch ports |
| [platforms/xbox/conker/](platforms/xbox/conker/) | Conker: Live & Reloaded target analysis, port risks and the Conker logbook |
| [platforms/xbox-360/01-research.md](platforms/xbox-360/01-research.md) | Xbox 360 recompilation and renderer research from other Switch ports |
| [platforms/xbox-360/black-ops/](platforms/xbox-360/black-ops/) | Black Ops research, dvar notes and its game-specific optimization logbook |

`source-notes/` folders retain exact copies of the supplied Markdown files for provenance and details that are too specific to repeat in the shared guidance. The supplied logs include historical results and known failed experiments; check their date and title before applying a finding elsewhere.
