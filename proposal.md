
In Section 3.1.11 in Privileged Spec, the note shows that
> A future revision of this specification will define a mechanism to generate an interrupt when a
hardware performance monitor counter overflows.

because current spec doesn't have one. Even if such an interrupt exists, we are still lacking functionalities to handle the interrupts with the current HPM (Hardware Performance Monitor, interchangeable with PMU for Performance Monitoring Unit) CSRs. In this proposal, we will address the problems after tracing a slightly-reduced Linux `perf record` case, by adding a few HPM-related CSRs, which are used in practice since mid-2018 in AndeStar V5 series.

### Background -- Perf Sampling

The basic steps of perf sampling are as follows,

1. Initialization: parse inputs
   1. setup the events of interests.
   2. allocate a counter register and setup the binding with the events.
   3. set the value of the counter to `MAX - period`, where `MAX` is its maximum value and `period` is a given sampling period based on specified events.
   4. enable the interrupt of the counter.
2. Running
When a target program is being sampled, the counter increases by 1 each time the events happen.
3. Interrupt handling
Eventually the counter an overflow, which triggers an interrupt.  The ISR should
   1. check which counter overflows.
   2. stop the counter.
   3. reset the counter value to `MAX - period`.
   4. sample the pc when the interrupt happened.
   5. enable the interrupt of the counter.

### Problems and Solutions

#### Counter-triggered interrupt
Such an interrupt need to be a local one so PLIC is not the way to go. We suggest using [CLIC (Core-Local Interrupt Controller)](https://github.com/riscv/riscv-fast-interrupt) after it is ratified. The foundation should also consider if this feature is worth a reserved interrupt number to be shown in `xcause` registers.

Besides the interrupt mechanism itself, we propose a set of CSRs to help process a counter-overflow interrupt,
+ **`mcounterinen` (Machine Counter Interrupt Enable)**: The steps 1.4 and 3.5 require this feature. 
+ **`mcounterovf` (Machine Counter Overflow)**: The step 3.1 requires this feature. 
Roughly speaking, these two serves like a xie/xip pair but for counter interrupts only.

#### Mode selection
Perf supports profiling in different privileged modes, e.g. sampling the whole system every 1000 cycles in the user space. Current spec has no support to this feature. The step 1.1 in the perf case have to ignore mode-related arguments, and thus always counts the event no matter what mode it is in.

We propose a set of CSRs for this: **`mcountermask_[m|s|u]` (Machine Counter Mask for M-/S-/U-mode)**. These registers are used in the beginning of step 1 to correctly setup profiling modes.  A counter only increases in M-/S-/U-modes if the corresponding bit in these registers are set respectively.

#### Writable counters/controls in S-mode
Tools like perf are the core driver of performance monitoring, and they often reside in kernel space.  Perf tends to take full control of all the hardware counters/control registers in HPM, so that it can even profile the performance of its underlying hypervisor if itself is inside a virtualization environment.  However, obeying to the hierarchical philosophy of RISC-V, M-mode is the real owner of PMU-related CSRs.  Counters are read-only shadows in S-mode, and controls are either shadows (e.g. `hpmevent*`) or M-mode exclusive (`mcounterinhibit`).  As a result, the steps 1.1, 1.3, 3.2 and 3.3 in the perf case should trigger instruction emulation or SBI calls to the higher privileged mode, which is a significant overhead.

We propose a CSR, **`mcounterwen` (Machine Counter-Write-Enable)**, for tools like perf to take full control of HPM CSRs. A HPM counter register (and its related bit in `xhpmevent`, `xcountermask*`, etc) can be directly written from S-mode if the corresponding bit in `mcounterwen` is set. S-mode alias is created according to this: `shpmevent`, `scounteren`, `scounterovf`, and `scounterinhibit`.

For M-S-U configuration this works well, but we need more feedback regarding H-extension.

### Conclusion
We add totally 6 new M-mode CSRs and S-mode alias CSRs. We address three problems that prevent the RISC-V port of perf from being as capable as other architecture's. 
1. The lack of HPM interrupt.
2. For a basic sampling usage: `countermask`, `counterinen` and `counterovf` provide indispensible features.
3. Performance overhead introduced from mode switching: `counterwen` allows S-mode HPM subsystem taking full control.

All comments are appreciated.

### References
There were two previous presentations regarding this solution.  One is at the [RISC-V microconference of LPC'18](https://www.youtube.com/watch?v=4OKkHCg7El0&t=2h20m53s), and the other is at [RISC-V Workshop Taiwan 2019](https://www.youtube.com/watch?v=Onvlcl4e2IU).  The later one is a brief summary of this proposal, except for the fact that `counterinhibit` register has been included in the spec.

We released software implementation of perf in Linux kernel along with AndeStar V5 since two years ago. Related source code can be found on [github](https://github.com/andestech/linux/blob/RISCV-Linux-4.17-bsp-v5_1_0-branch/arch/riscv/kernel/perf_event.c). [Some discussions](https://lkml.org/lkml/2020/6/29/1617) on Linux mailing list suggest the PMU driver needs a total re-write, but the implementation is sufficient as a reference.

Currently there are discussions in Unixplatform Spec Task Group about [PMU SBI extension](https://lists.riscv.org/g/tech-unixplatformspec/message/156). Our proposal is parallel to the PMU SBI extension work.
