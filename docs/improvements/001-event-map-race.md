# 001 — data race between event_map and check_std_events_work

**File:** `drivers/perf/riscv_pmu_sbi.c`

## What's going on

`pmu_sbi_check_std_events` runs asynchronously via a workqueue *after*
`perf_pmu_register()` returns. Its job is to mark unsupported events
by writing `event_idx = -ENOENT` into the global arrays
`pmu_hw_event_map[]` and `pmu_cache_event_map[][][]`.

Meanwhile, any `perf_event_open(2)` call can invoke `pmu_sbi_event_map()`
on any CPU and read those same arrays. No synchronization at all.

```c
// writer — workqueue context
static void pmu_sbi_check_event(struct sbi_pmu_event_data *edata)
{
    ...
    if (ret.error == SBI_ERR_NOT_SUPPORTED)
        edata->event_idx = -ENOENT;    // plain store, no barrier
}

// reader — perf_event_open() context
static int pmu_sbi_event_map(struct perf_event *event, u64 *econfig)
{
    ...
    return pmu_hw_event_map[config].event_idx;  // plain load, no barrier
}
```

This is a C11 data race. KCSAN should catch it.

## How bad is it

Not catastrophic. The worst case: an unsupported event slips through
`event_map` before the workqueue marks it, then fails later at
`SBI_EXT_PMU_COUNTER_CFG_MATCH` with an error. No crash, no memory
corruption. But it's still a kernel rule violation — concurrent
plain accesses to shared data must use `READ_ONCE`/`WRITE_ONCE`.

## Fix

**Option A (minimal)** — just add `READ_ONCE`/`WRITE_ONCE`:

```c
// writer
WRITE_ONCE(edata->event_idx, -ENOENT);

// reader
ret = READ_ONCE(pmu_hw_event_map[config].event_idx);
```

**Option B** — use a completion to block until the check finishes:

```c
static DECLARE_COMPLETION(std_events_checked);

// end of pmu_sbi_check_std_events:
complete(&std_events_checked);

// start of pmu_sbi_event_map:
wait_for_completion(&std_events_checked);
```

Option B stalls the first `perf_event_open` call until the workqueue
finishes, which is usually unnecessary. Option A is enough.

## Refs

- `pmu_sbi_check_std_events()` — line 378
- `pmu_sbi_check_event()` — line 363
- `pmu_sbi_event_map()` — line 642
- `pmu_hw_event_map[]` — line 124
- `schedule_work(&check_std_events_work)` — line 1513
