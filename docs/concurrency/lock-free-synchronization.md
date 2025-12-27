# Lock-free synchronization

Lock-free algorithms allow concurrent accesses to shared data without using
locks, therefore improving overall performance.

## Per-CPU variables

One way to reduce shared data is to use per-CPU variables.
A per-CPU variable is an array indexed by the CPU number, so that each CPU can
access and modify only its own elements.
During variable manipulation, preemption should be disabled to avoid changing
CPUs mid-work.

```c
// `counter` is an array of int, one for each CPU.
DEFINE_PER_CPU(int, counter);

void increase_counter(void) {
    // Get the address of the correct CPU's element.
    // `get_cpu_var` also disables preemption.
    int *__counter = &get_cpu_var(counter);
    // Increase counter.
    // This operation is directly reflected on the actual structure, as it is
    // achieved via reference.
    (*__counter)++
    // `put_cpu_var` re-enables preemption.
    put_cpu_var(counter);
}
```

## Atomic operations

There exists Linux APIs that enforce the use of atomic read‑modify‑write
instructions available from the ISA.
They are based on the `atomic_t`/`atomic64_t` types and architecture's CAS
instructions; they guarantee that operations are not interruptible.

```c
atomic64_t v = ATOMIC_INIT(0);
atomic64_add(2, &v);
atomic64_long_add_unless(&v, 2, 10)
```

Using CAS in concurrent data structures may be challenging or insufficient, as
they are based on a static value of variables at a given time, and do not have
a wide enough view of its history or of other variables to ensure more complex
operations complete securely.

## Read copy update

RCU is mainly used on read-mostly data structures (like lists).
It allows for fast concurrent read accesses without enforcing the use of locks,
but with the downside that readers must be carefully coded to tolerate slightly
inconsistent states.

Readers use `rcu_read_lock` and `rcu_read_unlock` to mark critical sections,
disabling preemption.
Reads are always available, they can execute concurrently and are not subject
to waiting queues.

Regarding writers, they typically still have to acquire spinlocks to make
updates, since only one writer can safely make changes.
For additions or updates, writers prepare new nodes or copies of data
separately and then atomically update pointers.
Deletions follow a similar approach but freeing the memory is deferred until
the end of a grace period.
A grace period ends when all CPUs have reported quiescent states (points where
no reader's critical section is running), signaled by context switches.
`rcu_read_unlock` doesn't directly inform the writer; it just helps the system
track grace periods.
Writers learn that a grace period has ended when the RCU subsystem confirms it.
`synchronize_rcu` is used to block the writer until all pre-existing RCU
readers' critical sections complete, after which the writer knows it is safe to
reclaim memory and thus finalize updates.

```c
void reader() {
    // Disable preemption and inform the RCU subsystem of a new reader.
    rcu_read_lock();
    // Here, reads can be achieved as usual, safely.
    // Re-enable preemption.
    rcu_read_unlock();
    // The end of the grace period for this CPU will be known by the RCU
    // subsystem upon the next preemption.
}

void writer() {
    // A lock over the data structure is still needed by writers.
    spin_lock(&list_lock);
    // Here, operations over the list can be achieved by atomically updating
    // pointers.
    // Upon deletions, frees must be deferred for a grace period.
    // Let's say we here removed node `node` from the list.
    // We do this by calling this list-specific function.
    // `node` is removed, but not freed.
    list_del_rcu(&node->list);
    // Free the lock.
    spin_unlock(&list_lock);
    // Wait for the grace period to conclude, meaning all CPUs have passed
    // through at least one quiescent state since `synchronize_rcu` started.
    // All CPUs are waited, even if they have not read from the data structure
    // this writer updated.
    synchronize_rcu();
    // Now it is possible to free the node.
    kfree(node);
}
```

