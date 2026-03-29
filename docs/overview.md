# RISC-V PMU driver — architecture notes

## Driver stack

```
User space
  perf(1), BPF programs
      |
      | perf_event_open(2)
      v
kernel/events/core.c   (perf core)
      |
      | pmu->add / start / stop / read
      v
┌─────────────────────────────────────────────────┐
│         drivers/perf/riscv_pmu.c                │  shared framework
│  struct riscv_pmu  (callback table)             │
│  struct cpu_hw_events  (per-CPU state)          │
└────────────┬──────────────┬─────────────────────┘
             │              │
    ┌────────┴──┐    ┌──────┴──────────┐
    │  legacy   │    │   SBI PMU       │
    │ riscv_pmu │    │ riscv_pmu_sbi.c │  ← main driver
    │ _legacy.c │    └──────┬──────────┘
    └───────────┘           │
  (fallback, no IRQ)        │ sbi_ecall(SBI_EXT_PMU, ...)
                            ▼
                      M-mode firmware (OpenSBI)
                            │
                            │ CSR access
                            ▼
                    HPM counters (hpmcounter3..31)
                    + firmware counters
```

## Key files

| File | What it does |
|------|-------------|
| `drivers/perf/riscv_pmu.c` | Shared framework: add/del/start/stop/read, CSR read helper |
| `drivers/perf/riscv_pmu_legacy.c` | Legacy driver: cycle + instret only, no IRQ, used as fallback |
| `drivers/perf/riscv_pmu_sbi.c` | SBI PMU driver, the main one, 1572 lines |
| `include/linux/perf/riscv_pmu.h` | `struct riscv_pmu`, `struct cpu_hw_events`, public API |
| `arch/riscv/include/asm/perf_event.h` | `perf_arch_fetch_caller_regs`, BPF user regs |
| `arch/riscv/include/uapi/asm/perf_regs.h` | enum of all 32 GPRs exposed to userspace |
| `arch/riscv/kernel/perf_callchain.c` | Frame-pointer callchain (user + kernel) |
| `arch/riscv/kvm/vcpu_pmu.c` | KVM PMU virtualization, ~900 lines |
| `arch/riscv/kvm/vcpu_sbi_pmu.c` | KVM-side SBI PMU ecall handler |

## SBI PMU extension

Needs SBI spec ≥ v0.3 and `SBI_EXT_PMU` (0x504D55) to be present.

| Function ID | Name | What it does |
|-------------|------|-------------|
| 0 | `NUM_COUNTERS` | How many counters are available |
| 1 | `COUNTER_GET_INFO` | Type/CSR/width for a given counter |
| 2 | `COUNTER_CFG_MATCH` | Find and allocate a counter for an event, returns logical index |
| 3 | `COUNTER_START` | Start a counter with an initial value |
| 4 | `COUNTER_STOP` | Stop a counter (optionally snapshot all values at once) |
| 5 | `COUNTER_FW_READ` | Read firmware counter low 32-bit |
| 6 | `COUNTER_FW_READ_HI` | Read firmware counter high 32-bit (SBI v2 only) |
| 7 | `SNAPSHOT_SET_SHMEM` | Set up shared memory page for atomic snapshot (SBI v2) |
| 8 | `EVENT_GET_INFO` | Batch query which events the hardware actually supports (SBI v3) |

## SBI version vs features

| Feature | SBI v0.3 | SBI v2.0 | SBI v3.0 |
|---------|----------|----------|----------|
| Basic counter management | ✓ | ✓ | ✓ |
| FW counter high 32-bit read | — | ✓ | ✓ |
| Snapshot shared memory | — | ✓ | ✓ |
| Batch event availability query | — | — | ✓ |

## Overflow interrupt — what's needed

One of the following must be present for sampling to work:

1. **`Sscofpmf` ISA extension** — standard path, uses `RV_IRQ_PMU`
2. **T-Head C9XX errata** — uses `THEAD_C9XX_RV_IRQ_PMU`, CSR swapped to `THEAD_C9XX_CSR_SCOUNTEROF`
3. **Andes custom PMU** (`xandespmu` vendor extension) — uses `ANDES_SLI_CAUSE_BASE + ANDES_RV_IRQ_PMOVI`

If none of the above: driver sets `PERF_PMU_CAP_NO_INTERRUPT | PERF_PMU_CAP_NO_EXCLUDE`,
counting only, no sampling.

## Supported event types (SBI PMU driver)

**Hardware events (PERF_TYPE_HARDWARE)**
cycles, instructions, cache-references, cache-misses, branch-instructions,
branch-misses, bus-cycles, stalled-cycles-frontend, stalled-cycles-backend, ref-cycles

**Cache events (PERF_TYPE_HW_CACHE)**
L1D / L1I / LL / DTLB / ITLB / BPU / NODE × read/write/prefetch × access/miss

**Raw events (PERF_TYPE_RAW)**
- `config[61:0]`: hardware raw event (SBI v3: bits 56-63 must be 0; older: bits 48-63)
- `config[63:62] = 0b10`: SBI firmware event (low 16 bits = firmware event code)
- `config[63:62] = 0b11`: platform-specific firmware event

**Firmware events (via raw encoding)**
misaligned_load/store, access_load/store, illegal_insn, set_timer, ipi_sent/rcvd,
fence_i_sent/rcvd, sfence_vma_sent/rcvd, sfence_vma_asid_sent/rcvd,
hfence_gvma/hfence_vvma variants (hypervisor)

## Counter state machine

```
event_init()    — validate, set hw.config, hw.idx = -1
  |
  v
pmu->add()      — SBI CFG_MATCH ecall → get logical index
  |               set hw.idx, mark used_hw_ctrs / used_fw_ctrs
  v
pmu->start()    — set_period, SBI COUNTER_START ecall
  |
  v
[counter running]
  |
  v  (overflow IRQ or explicit stop)
pmu->stop()     — SBI COUNTER_STOP ecall, event_update computes delta
  |
  v
pmu->del()      — SBI COUNTER_STOP with RESET flag, clear_idx
```

## Snapshot mechanism (SBI v2)

Each CPU gets one page of shared memory (`snapshot_addr`).
When `COUNTER_STOP` is called with `SBI_PMU_STOP_FLAG_TAKE_SNAPSHOT`,
M-mode atomically writes all counter values + overflow mask into that page.

This avoids one SBI ecall per counter in the overflow handler — just one stop
and all the data is already there.

There's also a shadow copy (`snapshot_cval_shcopy[64]`) to avoid the snapshot
page getting clobbered when the overflow handler needs to restart counters
and makes another SBI call.
