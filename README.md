# RISC-V PMU driver notes

Notes from reading `drivers/perf/riscv_pmu*.c` and `arch/riscv/kernel/perf_*.c`.
Trying to understand how the driver stack works and what's worth fixing.

---

## Contents

- [Architecture overview](docs/overview.md) — driver stack, SBI interface, event types
- [Issues](docs/improvements/README.md) — bugs, design problems, things worth improving

## Quick index

| # | Description | Severity |
|---|-------------|----------|
| [001](docs/improvements/001-event-map-race.md) | data race between event_map and check_std_events_work | high |
| [002](docs/improvements/002-heterogeneous-harts.md) | pmu_ctr_list is global, breaks on heterogeneous harts | high |
| [003](docs/improvements/003-fw-counter-overflow.md) | firmware counters can't be used for sampling | high |
| [004](docs/improvements/004-cpu-pm-fw-counters.md) | CPU suspend/resume only handles hw counters, fw counters ignored | high |
| [005](docs/improvements/005-thead-detection.md) | T-Head detection is too broad — relies on marchid==0 && mimpid==0 | medium |
| [006](docs/improvements/006-event-mapped-ipi.md) | event_mapped fires sync IPI to all CPUs unnecessarily | medium |
| [007](docs/improvements/007-cfg-match-ecall.md) | CFG_MATCH ecall on every pmu->add(), M-mode trap per context switch | medium |
| [008](docs/improvements/008-kvm-guest-callchain.md) | guest callchain not implemented, just a TODO | medium |
| [009](docs/improvements/009-frame-pointer-callchain.md) | callchain only works with frame pointers, no ORC/DWARF | low |
| [010](docs/improvements/010-initcall-timing.md) | legacy and SBI PMU initcall ordering has an implicit dependency | low |
| [011](docs/improvements/011-max-counters-limit.md) | RISCV_MAX_COUNTERS hardcoded to 64, SBI spec has no such limit | low |
