# Memory consistency models

Modern CPUs can incorporate instruction reordering in their design, for
performance reasons.
This means that operations may be executed in a order different from the one
appearing in the program, but it is ensured that the execution logic is left
unchanged.

However, on SMP systems with CPUs executing concurrently, these operations may
be perceived differently across threads.
For instance, the reordering of writes in a thread can produce a theoretically
impossible trace in another CPU.

```c
// Initial state
x = 0;
done = 0;

// Thread 1
x = 1;
done = 1;

// Thread 2
while(done == 0);
print(x);

```

Here, thread 2 should only print 1, but if thread 1 swaps its writes, it is
possible to see 0 printed.

## Sequential consistency

Sequential consistency is a property that programs must satisfy when compiled.
It enforces that instruction ordering follows the "happens‑before" relation,
meaning that the order of instructions in a program is the same as the order
in which they are visible in shared memory.

Execution between threads now is always consistent with how programs are
defined, but performance can be reduced.

## TSO model

In a TSO model (Total Store Order), which is the one x86 use, each CPU has its
own buffer in which it temporarily stores writes, allowing reordering in
instructions but showing shared memory changes as the program's order would.
However, by enqueuing writes, consistent (instant) visibility across threads
may be broken, and thus the locking algorithms depending on it.

## PSO model

ARM's Partial Store Order model has an even more relaxed architecture than TSO,
as each CPU has its own shared memory copy, with writes propagating between
processors independently, and without enforcing writes ordering.
Such flexibility means higher concurrency but can complicate reasoning about
memory consistency.

## Data races

A data race between threads can be produced if executing instructions comprise
at least one write operation.
When writes order isn't guaranteed, potential misalignment between intended
and actual execution can appear in a data race context.

CPUs fulfilling the DRF (Data Race Free) synchronization model provide
operations to coordinate reads and writes.

#### Fences

Fences (barriers) ensure that memory operations can be reordered
within, but not across them.
DRF programs set those barriers so that the program ensures "happens-before"
relations.
Two threads accessing the same memory location will either read or have
synchronization operations that ensure one's memory access occurs before the
other's.

#### Store release and load acquire

The `smp_store_release` and `smp_load_acquire` functions are used to enhance
basic load and store instructions.
SR ensures that before writing to memory, all the thread's previous operations
are pushed for visibility in other threads.
LA instead observes SRs, waiting until the read operation can see the latest
value.

#### Read and write once

`READ_ONCE` and `WRITE_ONCE` are primitives in C which can be used to read and
write variables while disallowing the compiler to reorder them, omit reads if
values appear unchanged, or insert excessive reads due to register spills.

