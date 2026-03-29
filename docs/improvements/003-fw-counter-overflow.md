# 003 — firmware counters can't be used for sampling

**File:** `drivers/perf/riscv_pmu_sbi.c`

## What's going on

The overflow IRQ handler checks for hardware counters first. If none are
active, it clears the interrupt and returns `IRQ_NONE` immediately:

```c
// line 1060
/* Firmware counter don't support overflow yet */
fidx = find_first_bit(cpu_hw_evt->used_hw_ctrs, RISCV_MAX_COUNTERS);
if (fidx == RISCV_MAX_COUNTERS) {
    csr_clear(CSR_SIP, BIT(riscv_pmu_irq_num));
    return IRQ_NONE;
}
```

The overflow loop also only iterates `used_hw_ctrs`, never `used_fw_ctrs`:

```c
for_each_set_bit(lidx, cpu_hw_evt->used_hw_ctrs, RISCV_MAX_COUNTERS) {
```

## Root cause

SBI firmware counters are maintained in M-mode software, not mapped to
real HPM registers, so they don't generate hardware overflow interrupts.
This is a fundamental SBI spec limitation.

## How bad is it

Firmware events like `SBI_PMU_FW_MISALIGNED_LOAD`, `SBI_PMU_FW_IPI_SENT`, etc.:

- `perf stat -e r<fw_event>` → **works fine**, counting mode is ok
- `perf record -e r<fw_event>` → **silently produces no samples**

So flame graphs and `perf top` are completely useless for firmware events,
and the user gets no error — just empty output.

## Fix

**Short term (fail fast)** — reject sampling outright in `event_init`:

```c
if (pmu_sbi_is_fw_event(event) && is_sampling_event(event))
    return -EOPNOTSUPP;
```

At least the user sees a clear error instead of silently getting nothing.

**Long term** — emulate sampling via hrtimer, similar to how
`PERF_PMU_CAP_NO_INTERRUPT` PMUs work in other architectures.
Set up a per-CPU hrtimer in `event_init` when fw event + sampling mode,
fire at `sample_period`, read counter delta, call `perf_event_overflow()`
when threshold is crossed. More work but would make fw events first-class.

## Refs

- Overflow handler early exit — line 1060
- Overflow loop — line 1093
- `pmu_sbi_is_fw_event()` — line 631
- `used_hw_ctrs` / `used_fw_ctrs` bitmaps — `struct cpu_hw_events` in `riscv_pmu.h`
