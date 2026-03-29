# Issues

Things I found while reading the code, roughly ordered by severity.

| # | Description | Severity | Status |
|---|-------------|----------|--------|
| [001](001-event-map-race.md) | data race between event_map and check_std_events_work | high | open |
| [002](002-heterogeneous-harts.md) | pmu_ctr_list is global, breaks on heterogeneous harts | high | open |
| [003](003-fw-counter-overflow.md) | firmware counters can't be used for sampling | high | open |
| [004](004-cpu-pm-fw-counters.md) | CPU suspend/resume only handles hw counters, fw counters ignored | high | open |
| [005](005-thead-detection.md) | T-Head detection is too broad — relies on marchid==0 && mimpid==0 | medium | open |
| [006](006-event-mapped-ipi.md) | event_mapped fires sync IPI to all CPUs unnecessarily | medium | open |
| [007](007-cfg-match-ecall.md) | CFG_MATCH ecall on every pmu->add(), M-mode trap per context switch | medium | open |
| [008](008-kvm-guest-callchain.md) | guest callchain not implemented, just a TODO | medium | open |
| [009](009-frame-pointer-callchain.md) | callchain only works with frame pointers, no ORC/DWARF | low | open |
| [010](010-initcall-timing.md) | legacy and SBI PMU initcall ordering has an implicit dependency | low | open |
| [011](011-max-counters-limit.md) | RISCV_MAX_COUNTERS hardcoded to 64, SBI spec has no such limit | low | open |
