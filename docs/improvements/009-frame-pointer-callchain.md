# 009 — callchain only works with frame pointers

**File:** `arch/riscv/kernel/perf_callchain.c`

## What's going on

RISC-V callchain is entirely frame-pointer based (`arch_stack_walk_user`,
`walk_stackframe`). Both kernel and user callchain require:
- Kernel compiled with `CONFIG_FRAME_POINTER`
- All userspace binaries and libraries built with `-fno-omit-frame-pointer`

GCC and Clang omit frame pointers by default. Distro-shipped glibc/libstdc++
usually don't have them. Result: callchain gets truncated at the first
library boundary.

```
# Example — cuts off at glibc boundary
perf report --call-graph
  my_program`heavy_compute
  my_program`worker_thread
  # truncated here — no frame pointer in pthread_create
```

## How other arches handle it

| Arch | Kernel callchain | User callchain |
|------|-----------------|---------------|
| x86  | ORC unwinder (no FP needed) | DWARF / LBR / FP |
| arm64 | FP or DWARF | FP or DWARF |
| RISC-V | FP only | FP only |

x86 built ORC specifically to remove the frame pointer requirement.

## Workaround today

```bash
# Recompile with frame pointers
CFLAGS="-fno-omit-frame-pointer" make

# Or use DWARF unwinding (slower, bigger samples, but no recompile needed)
perf record --call-graph dwarf ./my_program
```

`--call-graph dwarf` uses `libunwind` and works on the user side without
frame pointers, but it's slow and produces much larger samples.

## Long-term direction

RISC-V calling convention is regular (ra/s0 in fixed roles), so an ORC-style
unwinder is feasible. LLVM already generates `.debug_frame`/`.eh_frame` which
could be used for DWARF-based unwinding as a step toward that. Big project though.

## Refs

- `perf_callchain_user()` — `arch/riscv/kernel/perf_callchain.c:28`
- `arch_stack_walk_user()` — `arch/riscv/kernel/stacktrace.c`
- x86 ORC — `arch/x86/kernel/unwind_orc.c`
