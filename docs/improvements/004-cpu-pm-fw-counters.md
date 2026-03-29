# 004 — CPU suspend/resume ignores firmware counters

**File:** `drivers/perf/riscv_pmu_sbi.c`

## What's going on

The CPU PM notifier only checks `used_hw_ctrs` and bails early if it's empty:

```c
// line 1247
bool enabled = !bitmap_empty(cpuc->used_hw_ctrs, RISCV_MAX_COUNTERS);
// used_fw_ctrs is never checked

if (!enabled)
    return NOTIFY_OK;  // fw counters active → still returns here

for (idx = 0; idx < RISCV_MAX_COUNTERS; idx++) {
    ...
    case CPU_PM_ENTER:
        riscv_pmu_stop(event, PERF_EF_UPDATE);
        break;
    case CPU_PM_EXIT:
        riscv_pmu_start(event, PERF_EF_RELOAD);
        break;
    }
}
```

When the CPU enters a sleep state, M-mode firmware may clear or not preserve
firmware counter values. Without stop/restart, the delta computed on resume
is garbage.

## How bad is it

Only triggers if a workload uses exclusively firmware events (e.g., counting
`SBI_PMU_FW_SET_TIMER`) on a system with CPU idle states. In that case
`perf stat` values will have spurious jumps around sleep boundaries.
Non-deterministic and hard to reproduce.

## Fix

One-line change:

```c
bool enabled = !bitmap_empty(cpuc->used_hw_ctrs, RISCV_MAX_COUNTERS) ||
               !bitmap_empty(cpuc->used_fw_ctrs, RISCV_MAX_COUNTERS);
```

The event loop already iterates indices 0..63 covering both HW and FW slots,
so the loop body doesn't need to change.

Worth verifying that `riscv_pmu_stop`/`riscv_pmu_start` issue correct SBI
ecalls for FW counter indices too.

## Refs

- `riscv_pm_pmu_notify()` — line 1242
- `used_hw_ctrs` / `used_fw_ctrs` — `struct cpu_hw_events` in `riscv_pmu.h`
- `pmu_sbi_ctr_is_fw()` — line 405
