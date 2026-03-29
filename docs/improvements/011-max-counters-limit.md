# 011 — RISCV_MAX_COUNTERS hardcoded to 64

**File:** `include/linux/perf/riscv_pmu.h`

## What's going on

```c
// include/linux/perf/riscv_pmu.h:22
#define RISCV_MAX_COUNTERS  64
```

This sizes every per-CPU bitmap and array:

```c
struct cpu_hw_events {
    struct perf_event   *events[RISCV_MAX_COUNTERS];      // 512 bytes
    DECLARE_BITMAP(used_hw_ctrs, RISCV_MAX_COUNTERS);     // 8 bytes
    DECLARE_BITMAP(used_fw_ctrs, RISCV_MAX_COUNTERS);     // 8 bytes
    u64 snapshot_cval_shcopy[RISCV_MAX_COUNTERS];         // 512 bytes
    ...
};
```

The driver truncates at probe time if SBI reports more:

```c
// line 1440
if (num_counters > RISCV_MAX_COUNTERS) {
    num_counters = RISCV_MAX_COUNTERS;
    pr_info("SBI returned more than maximum number of counters. "
            "Limiting the number of counters to %d\n", num_counters);
}
```

## Why 64

RISC-V ISA has hpmcounter3..31 (29 counters) + cycle + instret = 31 hardware
counters max. 64 was chosen to leave room for firmware counters on top.
The SBI spec has no hard upper limit on total counters.

## Memory cost

`struct cpu_hw_events` is roughly 1072 bytes per CPU.
On a 128-CPU server that's ~137KB via `alloc_percpu`. Not a problem today.

## Does it need fixing now

Probably not. No platform actually exceeds 64 counters yet.
The main thing worth doing is adding a comment explaining the 64 and what
would need to change if it ever needed to grow.

If it ever does need to be dynamic, `struct cpu_hw_events` would need
its fixed arrays converted to dynamically allocated ones — not trivial.

## Refs

- `RISCV_MAX_COUNTERS` — `include/linux/perf/riscv_pmu.h:22`
- Truncation at probe — `riscv_pmu_sbi.c:1440`
- `struct cpu_hw_events` — `include/linux/perf/riscv_pmu.h:31`
- `alloc_percpu` — `drivers/perf/riscv_pmu.c:396`
