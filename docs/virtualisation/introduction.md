# Introduction

There are two types of virtual machines:

- process VMs are designed to provide a platform-independent environment to
    applications;
- system VMs provide an environment to multiple operating systems.

A system VM is defined as an isolated duplicate of the real system, managed by
a VMM (Virtual Machine Monitor) or hypervisor which translate accesses to
physical resources.

There are two types of hypervisors.

- Type 1: it runs native on bare metal.
- Type 2: it runs within another OS.

System virtualisation must meet three key requirements.

- Fidelity: the behaviour of the VM should match the real machine.
- Safety: VMs cannot override VMM control.
- Efficiency: only minor speed decreases are tolerated.

## Benefits

Implementing VMs is useful for a variety of reasons.
Multiple VMs can be packed together on a single physical machine, so that its
resources get fully exploited minimizing idles (consolidation).
This also helps with management and administration costs, by allowing the usage
of fewer bigger machines, enabling horizontal scalability and better responding
to variable workloads.
Infrastructure gets also standardized, simplifying software distribution and
isolating networks and storage.
Finally, they help with fault tolerance and sandbox security tests.

## Instruction sensitivity

Instructions can be privileged or unprivileged.
The core principle of virtualisation is to run guest privileged instructions as
unprivileged.
An instruction is virtualisation sensitive if is at least one of the following.

- Control sensitive, meaning it directly modifies system state.
- Behavior sensitive, meaning it acts differently in user and supervisor mode.

If sensitive instructions fall in the set of privileged instructions, a VMM can
be built (sufficient condition).

## SW and HW virtualisation

Virtualisation can be achieved on two levels: hardware and software.

Software-based virtualisation adopts a process called deprivileging.
The hypervisor intercepts (traps) and emulates all privileged instructions
coming from guests.

Hardware-based virtualisation exploits CPU virtualisation built-in support,
limiting the host intervention.
All processors typically come with control blocks on which to store guest state
to be resumed later explicitly by the VMM.
Furthermore, fewer specific events require switching to the hypervisor context.

