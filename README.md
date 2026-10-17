# Nintendo Switch homebrew porting guide

A practical guide for people and AI agents creating Nintendo Switch homebrew ports. It organizes reusable Switch guidance, a cross-project prior-port research database, and separate source-platform/game case studies.

The source documents are preserved under `platforms/` in `source-notes/`. Their recommendations and embedded plans are historical project material, not instructions that override a port's current requirements. Results are title- and build-specific; use the evidence and date before applying them elsewhere. This guide was assembled from the supplied local files only.

## Shared guide

- [01-porting-workflow.md](01-porting-workflow.md) — architecture boundaries and end-to-end port workflow.
- [02-how-the-switch-works.md](02-how-the-switch-works.md) — hardware, Horizon, memory, threads, graphics and storage.
- [03-switch-performance.md](03-switch-performance.md) — profiling and measured optimization practices.
- [04-development-and-build-practices.md](04-development-and-build-practices.md) — compilers, libraries, containers, licenses and build hygiene.
- [05-testing-and-debugging.md](05-testing-and-debugging.md) — desktop and console validation, graphics debugging and evidence.
- [06-porting-quality-checklist.md](06-porting-quality-checklist.md) — implementation and release readiness checklist.
- [07-build-process.md](07-build-process.md) — build-flow comparison and links to the two detailed project procedures.
- [08-prior-port-research-database.md](08-prior-port-research-database.md) — reusable methods and measured findings from earlier Switch ports, with scope and limitations.

## Platform and game material

- [platforms/xbox/](platforms/xbox/) — original Xbox platform research and Conker case study, runtime findings, build process and logbook.
- [platforms/xbox-360/](platforms/xbox-360/) — Xbox 360 platform research and Black Ops case study, performance research, build process and logbook.

Platform research pages summarize stable context. Work-in-progress measurements, open hypotheses and dated implementation logs stay within the corresponding game folder; archived source documents remain there as source material.

The shared numbered documents are at the repository root; there is no `docs/` subfolder.
