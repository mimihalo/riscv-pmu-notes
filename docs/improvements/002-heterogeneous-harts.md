# 002 — pmu_ctr_list is global, breaks on heterogeneous harts

**File:** `drivers/perf/riscv_pmu_sbi.c`

## What's going on

The counter info table and available-counter bitmask are global:

```c
// line 94
static union sbi_pmu_ctr_info *pmu_ctr_list;  // global, not per-CPU
static unsigned long cmask;                    // global
```

The code even has a comment admitting this:

```c
/*
 * RISC-V doesn't have heterogeneous harts yet. This need to be part of
 * per_cpu in case of harts with different pmu counters
 */
```

`pmu_ctr_list` is populated once at boot by querying from the boot CPU only.
All CPUs then share that single table.

## How bad is it

No current RISC-V platform has heterogeneous PMU harts so nothing is broken
today. But as soon as something like a big.LITTLE RISC-V design shows up
(SiFive P870 is moving this direction), it'll break:
- Small core has 4 programmable counters, driver asks for counter 8 → fail
- `cmask` is wrong for some CPUs

`pmu_sbi_snapshot_setup` has a similar admission:

```c
/*
 * We enable it once here for the boot cpu. If snapshot shmem setup
 * fails during cpu hotplug process, it will fail to start the cpu
 * as we can not handle hetergenous PMUs with different snapshot
 * capability.
 */
```

## Fix

Make `pmu_ctr_list` and `cmask` per-CPU:

```c
static union sbi_pmu_ctr_info __percpu *pmu_ctr_list;
static unsigned long __percpu cmask;
```

Move the `pmu_sbi_get_ctrinfo()` call into `pmu_sbi_starting_cpu()` so
each hart queries its own counter configuration independently.

Not a small change — also need to update:
- `pmu_sbi_check_std_events` which currently uses the global `cmask`
- `riscv_pmu_get_hpm_info()` used by KVM (has the same TODO comment)

## Refs

- Global declarations — line 94, 101
- `pmu_sbi_get_ctrinfo()` — line 869
- `pmu_sbi_starting_cpu()` — line 1148
- `riscv_pmu_get_hpm_info()` — line 486
