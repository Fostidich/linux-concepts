# Hardware-based

Hardware-based virtualisation solves many software-based problems by
introducing different execution modes.
Ring aliasing is removed, with the guest supervisor working as a normal
supervisor, letting the VMM decide which privileged instructions to trap (truly
sensitive) and which to let pass without VMM intervention; this also improves
performance.
Then, the hardware offers a dedicated space to store guest context, ditching
most shadow data structures.

HW virtualisation is implemented in different ways depending on the ISA.
x86 offers a VM control block to store the whole guest context, swtiching with
a dedicated instruction.
On other systems like ARM's, special registers hold just minimal VM
information.

## Intel VT-x

With Intel VT-x design it is possible to create a dedicated non-root mode,
which duplicates the entire CPU state.
Root mode is where the hypervisor operates from, while VMs live in this
non-root mode.
The switch between the two can be done anytime.
Executions in non-root mode can be recursively virtualised too.

Each mode uses its own interrupt flags, so that external interrupts, triggering
in root mode, can instigate an easy transition to non-root mode.
Regarding privileged instruction operations to data structures, those happen
in a vCPU represented in the VMCS (Virtual Machine Control Structure), skipping
trapping, being auto-managed.

Intel specifically introduced a set of new instructions for managing
virtualisation.
Here are some of them.

- `vmxon` and `vmxoff` turn VT-x on and off.
- `vmlaunch` enters non-root mode directly.
- `vmresume` enters non-root mode but loading the guest state in the VMCS.
- `vmexit` returns to root mode, saving state.

VT-x also enhances paging, introducing an Extended Page Table (EPT).
Instead of relying on a single virtual-to-physical page mapping register
(`ec3`), it offers two, one for the guest to map guest-virtual to host-virtual,
and one for typical host-virtual to host-physical pages.
This eliminates exits as the EPT can be directly managed by the guest without
requiring VMM intervention.

## KVM

Kernel-based Virtual Machine is a virtualisation module directly built into the
Linux kernel, letting it act as a type 1 hypervisor, having VMs run as threads.
It's efficient as reuses Linux's scheduler, memory management, and drivers
support.
It can be started by opening the virtual `/dev/kvm` device.
Typically, you'd place VM executions in a loop where the kernel takes control
back when guest exits for whichever reasons.

```c
// Enter hardware virtualisation.
int kvm_fd = open("/dev/kvm");
// Create guest memory.
int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);
// Allocate vCPU.
int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);

// Map VMCS KVM abstraction from vCPU.
struct kvm_run *kvm_run = mmap(/* ... */, vcpu_fd, 0);

for (;;) {
    // Run vCPU until exit.
    ioctl(vcpu_fd, KVM_RUN, 0);
    switch(kvm_run->exit_reason) {
        // Emulate IO in user space.
        case KVM_EXIT_IO: /* ... */
        // Check interrupt.
        case KVM_EXIT_HLT: /* ... */
    }
}
```

### Overcommitment

VMs do not use 100% of their available memory, therefore, it is possible to
allocate more VM memory that physically available.
KVM VMs are based on Linux virtual memory, which allows each process to consume
all its address space, enabling overcommitment by default.

### Ballooning

Ballooning is a way to enforce that VMs use of their physical memory stays
under a set threshold.
It is accomplished by adopting a special driver, which forces swapping to
occur when a VM wants to allocate more memory than physically allowed.
This can also be used when needing to free up space for allocating a new VM.

### Same-page merging

Kernel Same-page Merging (KSM) is a KVM tool that can be enabled and which
allows virtual pages with same content to share the same physical page.
Comparing pages can be very resource consuming, but it can be beneficial when
using applications that produce many instances of identical data.
When finding two duplicates, both get marked as CoW and one is redirected to
the physical of the other.

### Qemu

Qemu is a program which can create and manage VMs and emulate devices.
It can also do pure CPU emulation, but with KVM enabled it can unload CPU
execution to KVM directly for near-native speed.
A big challenge for VMMs is managing hardware interactions/pass-through exposed
to VMs.
Qemu offers an ecosystem of emulated devices accessible via the VirtIO
interface.
VirtIO implements a standard set of device interfaces including network
adapters, disk drives and IO controllers.

Through a shared memory region (`virtqueue`), the VirtIO guest driver
communicates directly with the hypervisor, knowing it is in a virtual machine;
this concept is called paravirtualisation.
Unlike full virtualisation, paravirtualisation modifies the guest OS to
efficiently interface the hypervisor through hypercalls, reducing traps and
emulation overhead.

## Xen

Xen is a type 1 hypervisor running on bare metal.
Critical (privileged) instructions requiring ring 0 get replaced by hypercalls.
In this way, user space applications typically remain unaffected, but this
design requires guest OS (DomU) instructions in ring-0 to be ported to ring-1.
Xen also support the use of Dom0, which is a specially privileged domain
running Linux completely natively.
DomU drivers can communicate with Dom0 directly, just like VirtIO.

