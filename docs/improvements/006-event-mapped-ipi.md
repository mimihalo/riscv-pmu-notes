# 006 — event_mapped fires a synchronous IPI to all CPUs

**File:** `drivers/perf/riscv_pmu_sbi.c`

## What's going on

When a process mmaps a perf event fd (to enable direct userspace counter reads),
`pmu_sbi_event_mapped()` sends a synchronous IPI to every CPU in `mm_cpumask`:

```c
// line 1350
if (event->hw.flags & PERF_EVENT_FLAG_USER_ACCESS)
    on_each_cpu_mask(mm_cpumask(mm),
                     pmu_sbi_set_scounteren, (void *)event, 1);
                                                              ^ wait=1 — blocks caller
```

`pmu_sbi_set_scounteren` writes `CSR_SCOUNTEREN` to allow userspace to `csrr`
the counter directly.

## Why it's written this way

The comment explains it:

> We must enable userspace access *before* advertising in the user page
> that it is possible to do so to avoid any race. And we must notify all
> cpus here because threads that currently run on other cpus will try to
> directly access the counter too without calling pmu_sbi_ctr_start.

Fair concern — if `scounteren` isn't set and userspace tries to read the
counter, it gets an illegal instruction trap.

## How bad is it

Every `mmap(perf_fd)` blocks until IPIs complete on all CPUs running
a thread of this process. For a multi-threaded service spread across many
CPUs, this stacks up. `perf record` mmap/munmaps on start/stop and on each
sample period change, so it's not a cold path.

## Fix

Set `scounteren` lazily in `pmu_sbi_ctr_start()` instead — that always runs
on the correct CPU:

```c
static void pmu_sbi_ctr_start(struct perf_event *event, u64 ival)
{
    ...
    if (event->hw.flags & PERF_EVENT_FLAG_USER_ACCESS)
        pmu_sbi_set_scounteren(event);  // on this CPU, no IPI needed
    ...
}
```

`event_mapped` only needs to set the flag, not send any IPIs.

Tradeoff: a thread already running on another CPU at mmap time won't get
`scounteren` updated until its next scheduling turn. That's fine because
`cap_user_rdpmc` in the userspace page isn't set to true until the event
is actually scheduled in on that CPU anyway.

## Refs

- `pmu_sbi_event_mapped()` — line 1322
- `pmu_sbi_set_scounteren()` — line 781
- `pmu_sbi_ctr_start()` — line 799
- `arch_perf_update_userpage()` — `drivers/perf/riscv_pmu.c:30`
