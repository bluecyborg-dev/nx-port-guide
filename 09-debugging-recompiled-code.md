# Debugging recompiled code

A statically recompiled game fails in ways a native port does not. The translator can miss code, misread it
or translate one instruction slightly wrong, and the symptom usually shows up far from the cause: a black
3D scene from a matrix routine, a fatal-error screen from a file status code, slow motion from a timer.
This page lists the signals worth collecting on every bring-up step and the techniques that turn a symptom
into the guilty function or instruction. Examples are x86 (original Xbox, `xboxrecomp`); the methods apply to
any lifted instruction set. The switches and log lines of a specific toolkit are listed on its platform page
(original Xbox: [xboxrecomp debug reference](platforms/xbox/02-xboxrecomp-debug-reference.md)).

## Make the runtime report what it could not do

Grep the log and the generated code for these after every regeneration and every new scene. Each one is
cheap to check and has named real bugs.

| Signal | What it means | What to do |
|---|---|---|
| Unresolved indirect call (the runtime logs the target address) | A function the static pass missed: code after alignment padding or a jump table, code right after a `ret` with no padding, thread entry points, callback and class-driver tables | Disassemble around the address: a plausible function start follows `ret`, `int3` padding or a jump table. Add it as a seed, run the **full** regeneration (seeds change function boundaries) and check the seed does not land inside an existing function, which would split it. A tail jump from the new function can lead to a second missed one. In Carbon the first unresolved call exposed a disassembler drift bug; Conker needed a dozen seeds found this way over two days, each by a run that reached a new scene |
| Direct call into an undetected address (a stub that logs its first hit and returns) | Same cause, but the call silently does nothing | Same as above |
| Callee-saved registers or stack pointer wrong after a call (an ABI check build option) | A function split in two by a wrong boundary: the epilogue never ran | Find the real start, fix the seed or boundary |
| Unresolved condition-flag fallbacks in the generated code | A compare and conditional jump the translator could not pair | Each may be a branch that is never (or always) taken; read the original code |
| Untranslated instructions: the translator's fallback comments in the generated code | Instructions emitted as no-ops | List them with every prefix the translator uses (`/* mnemonic */`, `/* FPU: ... */`, `/* SSE: ... */`), count them, and implement each one or show it is unreachable. Odd mnemonics with absurd operands (`fbld tbyte ptr [edx - 0x5cfeffc1]`) are data decoded as code. Conker's black 3D scene was one `fincstp` hiding behind the `FPU:` prefix |
| Allocator fill patterns in pointers (`0xAAAAAAAA` and similar) | A read of memory nobody initialised: a race between guest threads or an unlinked list node | Find the writer that should have run first |
| Null or wild function pointers | A table or object the game never filled, often after an earlier silent failure | Look for the first unresolved call or failed file operation before it |
| NaNs | A float operation translated wrong, or a bad input | Unmask the invalid-operation exception once a flag file exists (so boot noise is skipped): the first trap names the instruction. Burnout 3's port found its `lahf` translation bug this way |

## Hangs and crashes

- A watchdog that prints the guest call stack when the game stops making progress saves most hang
  investigations. Without one, attach a debugger and take every thread's backtrace: translated functions
  keep their guest names (`sub_00046490`), so the host frames are the guest call chain.
- Tell a guest spin from an expected event wait, a lock deadlock, file I/O, a GPU fence wait or a blocked
  log sink before calling it a deadlock. A frozen image alone proves nothing.
- Games have their own fatal-error screens, often a tight `jmp $` loop after drawing a message. When one
  thread spins in a single small function forever, check whether it is that screen and what called it.
- Process hygiene on a desktop bench: run one instance at a time. A runtime may rename its process and
  ignore `SIGTERM`; kill it by PID with `-9`, and never `pkill -f` a pattern that also matches your own shell.
- If the runtime maps guest memory at fixed host addresses, keep address-space randomisation on under the
  debugger (`set disable-randomization off` in gdb); with it off, fixed windows can collide.

## Inspect guest memory while the game runs

- Guest RAM sits at a fixed host base plus the guest address. Read it from outside the process through
  `/proc/<pid>/mem` (unbuffered) or a small peek tool that follows pointer expressions, or from inside the
  runtime with a temporary scan behind an environment switch.
- **Search by value.** Floats rarely match exactly between two reads; search with a tolerance. Search
  matrices both row-major and column-major: a D3D matrix stores the camera's axes as columns.
- **Hardware watchpoints report changes only.** Write a different value to the address first, then watch
  it. They can also miss writes: wide vector stores that start below the watched address, writes through
  an aliased mapping of the same memory, writes by the kernel. If the value changes without a hit, break on
  the suspect writer's entry and read the memory when it returns.
- Without debug information, cast the runtime's globals explicitly (`p *(float*)&g_xmm3`,
  `p/x *(unsigned*)&g_esp`); thread-local ones work the same way once the debugger has attached to the thread.
- **Slow motion or fast motion:** find the game's float clocks in guest memory (values that grow
  steadily) and compare their rate with wall time. A fixed time step that assumes the original frame rate
  shows up as a clock running at the frame rate's ratio.

## Prove a translated routine wrong (or right)

When a value in guest memory is wrong and you know which routine computes it, compare the translated
routine with the original machine code on the same inputs:

1. **Capture the inputs at entry**: argument registers, floating-point and vector registers, the stack
   arguments, and any memory the routine reads. Compilers pass floats in vector registers between their own
   functions (whole-program optimisation), so the documented calling convention is not enough; read the
   disassembly to see which registers are used before they are written.
2. **Emulate the original code** on those inputs. A few dozen lines of Python cover a numeric routine: decode
   with capstone and implement only the instructions it uses. Round to 32-bit floats after every scalar SSE
   operation; implement the x87 register stack (including `fincstp`/`fdecstp` and `fxch`) and the flag idioms
   it uses (`ucomiss`/`fcomi`, `lahf`, `test ah, 0x44`, `jp`).
3. **Read the translated routine's output** when it returns (break at entry, `finish`, dump the
   destination).
4. **Compare.** Equal outputs mean the routine is fine and the inputs or a later writer are wrong: go to the
   next writer. Different outputs name the routine; step both through to the first diverging instruction.
5. If the stored result is a correct value combined with something else, solve for the extra factor. For
   matrices, `S = good⁻¹ · bad`; exact zeros, ones and repeated values in `S` mean a real operation (a
   post-multiply) by a broken matrix, which points at the routine that built it.

The whole chain for Conker's black scene (camera matrix found in RAM → copy routine → look-at, which was
correct → roll-matrix builder, which was not → `fincstp`) is in the
[Conker logbook](platforms/xbox/conker/conker-logbook.md), entry "black front end solved".

## Ground truth from an emulator

An emulator with a debugger stub (xemu's GDB stub for the original Xbox) runs the real game code: diff
guest memory page by page, search for floats and matrices, and compare a routine's output with the port's.
Halt the guest before setting breakpoints and remove them before continuing. Use it when the original
machine code is too long to emulate by hand or depends on hardware state.
