# Conker runtime debugging and renderer findings

This file collects diagnostics and behavior found while investigating Conker. These tools may be reusable by other original-Xbox titles, but the issues and observations below are Conker-specific unless noted. See the [logbook](conker-logbook.md) for dated progress and the [target analysis](01-target-analysis.md) for the XBE and hardware context.

## Runtime debug switches

These `xboxrecomp` runtime features were added during Conker work. Desktop builds accept environment variables; Switch builds accept `KEY=VALUE` launch arguments or lines in `*_env.txt`.

| Switch | Effect |
|---|---|
| `RECOMP_FILE_TRACE=1` | Logs handled `NtCreateFile` calls with guest path, access mask, disposition, options, attributes, returned NT status and `Information`. |
| `RECOMP_FILE_TRACE=2` | Adds guest stack code addresses (return addresses and other code pointers) to locate the routine issuing an open. |
| `RECOMP_SKIP_DRAWS=<lo>-<hi>` | Drops the inclusive draw range for each frame; the counter starts at zero after each flip. Useful for bisecting a wrong frame. |
| `RECOMP_SKIP_BLEND=<src>/<dst>` | Drops draws with the given hexadecimal NV2A blend-factor pair, e.g. `306/0` for `DST_COLOR`/`ZERO`. `RECOMP_SKIP_INLINE=1` drops inline-vertex draws. |
| `RECOMP_PSH_DUMP=1` | Prints each new pixel-shader key (combiner stages and final combiner) and generated GLSL. |
| `RECOMP_FRAME_TRACE=<flag file>` | Emits a one-frame draw trace after the flag file exists, including raw blend registers, texture-shader program, combiner control/factors, alpha test, surface format, fog state, `SET_CONTROL0`, `ZMIN_MAX_CONTROL` and clip range. |

## Findings and open questions

- **Cache file status behavior:** Conker's cache setup depends on distinguishing a missing parent path (`STATUS_OBJECT_PATH_NOT_FOUND`, mapped to XAPI `ERROR_PATH_NOT_FOUND`) from a missing final component (`STATUS_OBJECT_NAME_NOT_FOUND`, mapped to `ERROR_FILE_NOT_FOUND`). Check both `NtQueryFullAttributesFile` and `NtCreateFile` paths. See the dated [logbook](conker-logbook.md).
- **Image and guest-stack overlap:** the XBE image ends at `0x00891D20`, beyond xboxrecomp's default guest stack base `0x00780000`. The base was moved to `0x00900000` so stacks/runtime data no longer overlap `.data`.
- **64-bit guest address reservation:** on aarch64 Linux with a 48-bit user address space, the chosen guest base plus `0xFFFFFFFF` must fit. An OS mapping near the top can leave no room for NV2A device apertures at `0xFD000000` and above.
- **Clear-value format:** Conker traces showed an R5G6B5 target where `0xFFFF` means white. When the backend stores surfaces as ARGB8888, interpret a clear value using the guest surface format before widening it.
- **Which surface to present:** present the last surface drawn in the frame, not merely the last surface touched; Conker clears off-screen surfaces after compositing.
- **Depth-format hypothesis remains unconfirmed:** Conker traces showed `SET_CONTROL0 = 0x00100001`; the vendored header defines `Z_PERSPECTIVE_ENABLE` at bit 16 (clear) and does not define bit 20. Do not label this W-buffering without confirming hardware semantics.

The observations are evidence from this title's investigation, not assertions that every XBE uses the same stack layout, cache behavior or renderer patterns.
