# Software-based

Software-based virtualisation adopts a process called deprivileging, for which
guest's supervisor instructions are translated into host user space.
The VMM further intercepts all guest modifications to kernel data structures,
such as the IDT (Interrupt Descriptor Table), redirecting them to internally
managed replicates, when necessary.
Another example of these "shadow tables" are page tables: the hypervisor catches
the guest OS when attempting to set its own page table, and it builds a
corresponding shadow table where guest "physical" pages are translated to host
virtual ones.

## Problems

Some architectures originally couldn't be efficiently virtualised because not
all "sensitive" instructions were privileged.

A guest supervisor could detect deprivileging by reading some hardware
registers seeing mismatched CPL (Current Privilege Level) or host tables,
exposing the VMM presence.
Furthermore, constant emulation (`sysenter`/`sysexit` called for every guest
`syscall`) required too many traps, making software-only VMMs impractical.

