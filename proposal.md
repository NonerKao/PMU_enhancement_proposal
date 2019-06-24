The current facility provided by RISC-V hardware performance monitors (HPM) requires more capabilities to support **perf** in Linux or **hwpmc** in BSDs.  In this proposal, we will discuss the deficiencies in the privileged spec v1.0, taking Linux perf as an example.  Further, we will share our solution as a reference.

### Perf Sampling in RISC-V

**Scenario:** A user want to sample a program every 1000000 cycles in U-mode and S-mode

**Input:** `period = 1000; events = [cycles]; modes = [u, s]`

**Output:** A trace of sampled pcs of the program

1. Set a counter to its maximum value minus `period`
2. Set the corresponding event mask `events`
3. Set the corresponding mode selector `modes`
4. Start the target program
5. Increase the counter by one each time the events marked by `events` happens in privileged modes `modes`
6. Trigger an interrupt when the counter overflows
7. Sample current pc
8. Reset counter as step 2
9. Resume the program, go to step 4

### Problems

* (Functional) No counter-triggered interrupt
The current spec has no such support.
* (Functional) No mode selection
* (Performance) No writtable HPM counters/controls in S-mode

1. HPM counters/controls are not writable in S-mode

Perf is the core driver of performance monitoring, and they often reside in kernel space.  Perf tends to take full control of the whole HPM so that it can even profile the performance of its underlying hypervisor if itself is inside a virtualization environment.  However, obeying to the hierarchical philosophy of RISC-V, M-mode is the real owner of HPM CSRs.  Counters are read-only shadow in S-mode, and controls are either shadows (e.g. `hpmevent*`) or M-mode exclusive (`mcounterinhibit`).

2. Mode selections
Perf can profile different privileged modes or their combinations, e.g. sampling the whole system every 1000 cycles in the user space only or in kernel and hypervisor.  The current spec has no such support.

3. No counter-triggered interrupt
The current spec has no such support.

### Implementation in Our Prototype

From our experiences, RISC-V HPM can be enhanced by adding critical CSRs.  Note that all following CSRs have the same length as `*counteren` registers:

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
