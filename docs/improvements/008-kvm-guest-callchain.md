# 008 — guest callchain is just a TODO

**File:** `arch/riscv/kernel/perf_callchain.c`

## What's going on

Both `perf_callchain_user()` and `perf_callchain_kernel()` bail immediately
when running in a guest context:

```c
// arch/riscv/kernel/perf_callchain.c:32
void perf_callchain_user(struct perf_callchain_entry_ctx *entry,
                         struct pt_regs *regs)
{
    if (perf_guest_state()) {
        /* TODO: We don't support guest os callchain now */
        return;
    }
    ...
}

void perf_callchain_kernel(struct perf_callchain_entry_ctx *entry,
                           struct pt_regs *regs)
{
    if (perf_guest_state()) {
        /* TODO: We don't support guest os callchain now */
        return;
    }
    ...
}
```

Both TODOs are untouched.

## How bad is it

Inside a KVM guest, `perf record --call-graph fp` collects no callchain at all.
Flame graphs are empty. BPF profilers (bcc `profile`, bpftrace) also get nothing
from `bpf_get_stackid`.

This is frustrating because KVM PMU virtualization is actually quite complete
(`vcpu_pmu.c` is ~900 lines, overflow can be delivered to guests, counters
work). Callchain is the missing last piece for profiling guest workloads.

## Fix

x86 and arm64 both implement guest callchain. Reference: `arch/x86/kernel/perf_callchain.c`
using `kvm_is_in_guest()` and `kvm_get_guest_regs()`.

For RISC-V, kernel callchain in guest would look roughly like:

```c
void perf_callchain_kernel(...) {
    if (perf_guest_state()) {
        struct pt_regs *guest_regs = kvm_get_guest_regs(current);
        if (guest_regs)
            walk_stackframe(NULL, guest_regs, fill_callchain, entry);
        return;
    }
    walk_stackframe(NULL, regs, fill_callchain, entry);
}
```

Guest *user* callchain is harder — needs safe access to guest physical memory.
Could start with just kernel callchain.

## Refs

- TODOs — `arch/riscv/kernel/perf_callchain.c:32,43`
- KVM PMU — `arch/riscv/kvm/vcpu_pmu.c`
- x86 reference — `arch/x86/kernel/perf_callchain.c`
