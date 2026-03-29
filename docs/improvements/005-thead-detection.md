# 005 — T-Head vendor detection is too broad

**File:** `drivers/perf/riscv_pmu_sbi.c`

## What's going on

```c
// line 1201
} else if (IS_ENABLED(CONFIG_ERRATA_THEAD_PMU) &&
           riscv_cached_mvendorid(0) == THEAD_VENDOR_ID &&
           riscv_cached_marchid(0) == 0 &&
           riscv_cached_mimpid(0) == 0) {
```

`marchid == 0 && mimpid == 0` means the hardware left those fields unset,
which was common on early T-Head C906/C908/C910 silicon. Two problems:

1. Future T-Head cores that properly fill `marchid`/`mimpid` won't match
   this condition — they'd silently fall through to no-interrupt mode even
   if they have the same errata.
2. Any early silicon from a T-Head licensee with the same vendor ID and
   empty marchid/mimpid would incorrectly get T-Head workarounds applied.

## How bad is it

Mainly a forward-compatibility concern. New T-Head cores with correct IDs
would silently lose overflow interrupt support and fall back to counting-only
mode, with no error message.

## Fix

Build a positive allowlist of known T-Head marchid values, or — better —
use the existing errata infrastructure in `arch/riscv/errata/thead/errata.c`
instead of open-coding the detection here.

All other T-Head errata go through that central path. The PMU driver should
be consistent.

## Refs

- IRQ setup — `pmu_sbi_setup_irqs()` line 1192
- T-Head detection — line 1201
- `ALT_SBI_PMU_OVERFLOW` macro — line 31
- `arch/riscv/errata/thead/errata.c`
