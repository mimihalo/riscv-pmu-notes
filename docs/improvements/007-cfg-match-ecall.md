# 007 — CFG_MATCH ecall on every pmu->add()

**File:** `drivers/perf/riscv_pmu_sbi.c`

## What's going on

`pmu_sbi_ctr_get_idx()` always makes a synchronous SBI ecall (M-mode trap)
to allocate a counter:

```c
// line 567
ret = sbi_ecall(SBI_EXT_PMU, SBI_EXT_PMU_COUNTER_CFG_MATCH,
                cbase, cmask, cflags, hwc->event_base, hwc->config, 0);
```

`pmu->add()` runs on every context switch where the task has active perf
events. Plus `ctr_start` and `ctr_stop` each need one ecall too, so each
context switch costs at least 3N SBI ecalls (N = active events).

## Numbers

Say 100k context switches/sec with 4 active events → 1.2M SBI ecalls/sec,
all M-mode traps, all needing CSR save/restore + privilege switch.
SBI snapshot helps with stop/start overhead but CFG_MATCH still fires every time.

## Fix

Cache the counter allocation for well-known events. Fixed-function counters
(cycle=0, instret=2) always map to the same logical index. Most hardware
events also get a stable assignment as long as the counter is free.

Build a small event→counter cache at probe time. On `add()`, try the cached
index first with `SBI_PMU_CFG_FLAG_SKIP_MATCH` (this flag is already used
for legacy mode's cycle/instret handling):

```c
// legacy mode example, line 556-561
cflags |= SBI_PMU_CFG_FLAG_SKIP_MATCH;
cmask = BIT(cached_idx);
```

Fall back to full CFG_MATCH only on cache miss or if the cached counter is
already in use. For well-behaved workloads (few events, not stealing each
other's counters) this eliminates most of the CFG_MATCH overhead.

## Refs

- `pmu_sbi_ctr_get_idx()` — line 538
- `SBI_PMU_CFG_FLAG_SKIP_MATCH` usage — line 556-561
- `pmu_sbi_ctr_start()` — line 799
- `pmu_sbi_ctr_stop()` — line 822
