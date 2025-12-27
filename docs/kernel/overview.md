# Overview

An operating system is a program that acts as an intermediate layer between
applications and the hardware.

## Goals

In general, the main goals of a OS are resource management, isolation and
protection, portability and extendability.

The OS manages resources between multiple applications by creating programs as
if they were assigned their own set of resources (CPU, memory, disk), while
ensuring fair utilization.
Reliability and security is ensured by regulating access rights to resources,
such as memory or files.
OSs also hides the complexity of hardware management, adopting a facade
pattern, often allowing the same applications to run on systems equipped with
different physical resources.
Low-level interfaces are uniformly developed so that high-level layers can be
reused: this is achieved by utilizing device drivers which hide the undergoing
complexity of the peripherals, introducing a bridge pattern.

## Techniques

### Resource management

Latency can be reduced by increasing CPU utilization.
Processes are interleaved by applying context switches either after a regular
time quantum or when one gets in a waiting status.
This can happen at the time of interrupts (coming from e.g. timers, the disk or
the network), exceptions and system calls.
The context of the current process is written in the PCB while the PCB of the
next is put into the machine state.

The Process Control Block (PCB) is a kernel structure containing process data,
it contains:

- the Process Identifier (PID);
- the architectural state including the values of registers and counters in the
CPU;
- the memory mappings linking the virtual address space to physical memory;
- open files, to ensure that the process can resume interactions after pausing;
- credentials, like user and group IDs, used to determine access levels for
resources;
- signal handlers which are called to manage asynchronous events like
interrupts;
- the controlling terminal, if any, along priority details;
- statistics queried to monitor performance and usage.

Processes are switched based on the scheduling criteria adopted by the OS,
which balances many indices.
General Purpose OSs (GPOS) weights more fairness and throughput, whilst Real
Time OSs (RTOS) deadlines, priority and efficiency.

Linux OSs usually implement one of the following policies.

- FIFO (RT): a task runs until it finishes or a higher-priority task needs the
CPU (no time slice).
- Round Robin (RT): each task gets a time slice; when its turn ends, the next
equal-priority task runs.
- Completely Fair Scheduler (GP): Linux measures how much CPU time each task
should get based on various dynamic factors.
- EEVDF (GP): the scheduler computes a virtual deadline, and runs the task
having the soonest (improved CFS).
- Earliest Deadline First (RT): the task whose deadline is closest runs first.

### Isolation and protection

The memory a process can use is a Virtual Address Space (VAS).
This region is isolated from other processes, but some locations may be shared.
Virtual and physical space usages do not coincide: virtually, the process has
an abundant contiguous chunk of memory, but physically resources are chopped up
in pages and scattered all around the memory, even with some possibly found in
the swap area on the disk.

### Portability and extendability

A facade pattern is introduced in OSs with the integration of system calls.
This is the way for processes to ask for a privileged service (like reads or
writes in buffers) that are taken care by the kernel.
Another pattern used by devices is the bridge pattern, which states that the
abstraction and the implementation of features must be defined independently.

## Architectures

The design of an OS can follow different architectural approaches.

- A bare metal (no OS) option is chosen when there's only a single running app,
which requires low latency and low power consumption.
- A monolithic OS uses a single large binary for the kernel, which itself
contains all required drivers (Linux).
- There is also the option to have a monolithic OS with modules, which allows
external components to be linked at runtime.
- A micro kernel instead avoids all non-essential components, which need to be
included in user space.
- An hybrid OS resembles micro kernels, but includes some other services to
increase performance (Windows and MacOS).
- Finally, a library OS comes in the form of libraries, which must be selected
and compiled with the corresponding configuration code.

