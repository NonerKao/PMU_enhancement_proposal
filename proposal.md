The current facility provided by RISC-V hardware performance monitors (HPM) is not sufficient for general use, e.g. **perf** in Linux or **hwpmc** in BSDs.  In this proposal, we will address what current spec lacks and provide the solutions we have in Andes Technology, taking Linux perf as an example.  We believe that this proposal will serve as a starting point of discussion towards improving RISC-V HPM.

### Background -- Perf Sampling

The basic steps of perf sampling are as follows,
1. Initialization: translating inputs to parameters
   + set up events of interests.
   + allocate a counter and set up the binding with the events.
   + set the value of the counter to `MAX - period`, where `MAX` is its maximum value and `period` is a given sampling period based on specified events.

2. Running
When a target program is being sampled, the counter increases by 1 each time the events happen.

3. Interrupt handling
Eventually, the counter overflows and triggers an interrupt.  The ISR should
   1. reset the counter value to `MAX - period`
   2. (architecture independent) sample the pc of the target program/whole system
   3. return

### Problems

1. Writable counters/controls in S-mode
Tools like perf are the core driver of performance monitoring, and they often reside in kernel space.  Perf tends to take full control of the whole HPM so that it can even profile the performance of its underlying hypervisor if itself is inside a virtualization environment.  However, obeying to the hierarchical philosophy of RISC-V, M-mode is the real owner of HPM CSRs.  Counters are read-only shadow in S-mode, and controls are either shadows (e.g. `hpmevent*`) or M-mode exclusive (`mcounterinhibit`).

2. Mode selections
Perf can profile different privileged modes or their combinations, e.g. sampling the whole system every 1000 cycles in the user space only or in kernel and hypervisor.  The current spec has no such support.

3. No counter-triggered interrupt
The current spec has no such support.

### Implementation in AndeStar™ V5

From our experience, RISC-V HPM can be enhanced by adding critical CSRs.  Note that all following CSRs have the same length as `*counteren` registers:

|CSR name|Description|Related to Problem|
|---|---|---|
|`mcounterwen`|Each counter has one bit to allow/deny write accesses to HPM CSRs from S-mode.  The firmware/M-mode initialization can thus turn this register on or off during boot time so that the OS can directly manipulate any required CSRs.  An alternative is to implement SBI calls to modify HPM counters and controls, but obviously, it introduces large overhead.|write access from S-mode|
|`[m\|s]countermask_[m\|s\|u]`|Each counter has three mode-selection mask bits that profiling software can manipulate.|mode selection|
|`[m\|s]counterinen`(\*)|Each counter has one bit indicating to or not to trigger an interrupt when an overflow happens.|counter-triggered interrupt|
|`[m\|s]counterovf`(\*)|In ISR, scanning this CSR to know which counters have overflowed.|counter-triggered interrupt|
|`scounterinhibit`|While we have `mcounterinhibit` in current spec, we implement this as an alias that can be directly controlled by S-mode software once the corresponding bits in `mcounterwen` is set .|others|
|`shpmevent*`|While we have `mhpmevent*` in current spec, we implement this as an alias that can be directly controlled by S-mode software once the corresponding bits in `mcounterwen` is set .|others|

Note (*): These two are just CSRs, not the counter-triggered interrupt itself.  We implement the interrupt in aligned with the consensus from Fast Interrupts Task Group.

### References

There were two previous presentations.  One is at the [RISC-V microconference of LPC'18](https://www.youtube.com/watch?v=4OKkHCg7El0&t=2h20m53s), and the other is at [RISC-V Workshop Taiwan 2019](https://www.youtube.com/watch?v=Onvlcl4e2IU).  The later one is a summary of this proposal.