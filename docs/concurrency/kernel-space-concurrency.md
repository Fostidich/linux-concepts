# Kernel space concurrency

Kernel concurrency is crucial for modules to function correctly in
multithreaded environments.
Unlike user‑space programs, the kernel handles interrupt processing, spinning
waits, and re-entrant code.
Concurrency in kernels can have three sources: interrupts, multi-core systems
or preemption.

Note than in preemptive systems, after a interrupt finishes execution, the
kernel may switch to a process in user space different from the last executing
one.

Linux kernel is optionally non-preemptive.
In those cases a task in the kernel explicitly has to call `schedule`
(when knowing it is safe to reschedule).
Therefore, a planned context switch can happen if the process puts itself
knowingly in a wait queue.

## Atomic operations

Atomic contexts are sections of code where a preemption is not safe.

Typical atomic contexts are the following.

- The kernel is within an interrupt handler
- The kernel is holding a spinning lock
    - This could lead to deadlocks or inconsistent state due to half‑completed
    critical sections
- The kernel is modifying a CPU structures
- The kernel state cannot be restored completely with a context switch
    - For example when executing in a FPU, as micro-architectural states cannot
    be simply copied and restored by the kernel.

In kernel code, `preempt_disable` and `preempt_enable` can be used to
define an atomic context.

In real-time patched kernels, threads that must avoid interruption can set
their priority to a value higher than fifty.
Moreover, the interrupt handler is designed to perform minimal tasks, improving
response time.

