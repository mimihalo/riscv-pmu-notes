# 010 — legacy and SBI PMU initcall ordering has an implicit dependency

**File:** `drivers/perf/riscv_pmu_legacy.c`, `drivers/perf/riscv_pmu_sbi.c`

## What's going on

The two drivers use different initcall levels:

```c
device_initcall(pmu_sbi_devinit);        // level 6
late_initcall(riscv_pmu_legacy_devinit); // level 7
```

The skip mechanism works by the SBI PMU driver calling
`riscv_pmu_legacy_skip_init()` inside its probe, which sets
`pmu_init_done = true` before the legacy `late_initcall` runs:

```c
// riscv_pmu_legacy.c
static int __init riscv_pmu_legacy_devinit(void)
{
    if (likely(pmu_init_done))
        return 0;   // SBI PMU already registered, skip
    ...
}
```

## When it breaks

`pmu_sbi_devinit` creates a platform device via `platform_device_register_simple`.
Normally the platform bus probes and runs the `probe` callback synchronously,
so `pmu_init_done` is set before `late_initcall`.

But if the platform device probe is deferred for any reason (DT dependency,
driver bind order), `probe` won't run by the time `late_initcall` fires.
Both drivers would then try to register as the `"cpu"` PMU with
`PERF_TYPE_RAW`, and the second registration would fail or produce undefined
behavior.

In practice this probably doesn't happen with today's boards, but it's
a fragile assumption to rely on.

## Fix

**Option A** — use an explicit exported flag instead of relying on timing:

```c
// riscv_pmu_sbi.c, set in probe callback on success:
bool riscv_pmu_sbi_available __ro_after_init = false;
EXPORT_SYMBOL_GPL(riscv_pmu_sbi_available);

// riscv_pmu_legacy.c:
extern bool riscv_pmu_sbi_available;
if (riscv_pmu_sbi_available)
    return 0;
```

**Option B (cleaner)** — merge into a single `late_initcall`:

```c
late_initcall(riscv_pmu_unified_init)
{
    if (sbi_probe_extension(SBI_EXT_PMU))
        return pmu_sbi_devinit();
    return pmu_legacy_devinit();
}
```

This removes the implicit ordering dependency entirely.

## Refs

- `riscv_pmu_legacy_skip_init()` — `riscv_pmu_legacy.c:174`
- `pmu_init_done` — `riscv_pmu_legacy.c:18`
- `pmu_sbi_devinit()` — `riscv_pmu_sbi.c:1532`
