# Locking synchronization

## Basic lock types

- Sleeping locks are employed when a task is put to sleep if a lock isn't
    available, like during file system or network operations
    - [Mutexes](#mutexes) provide mutual exclusion, where only a single task
        can hold the lock at a time
    - Semaphores allow for a set number of tasks to own the lock, via an
        internal counter
- [Spinlocks](#basic-spinlock) are utilized in atomic and interrupt contexts to
    protect shared kernel data structures, without sleeping but spin-waiting
    - [Read‑write locks](#rwlocks) allow readers to hold the lock
        simultaneously, but writers to get exclusive access
    - [Seqlocks](#seqlocks) are used when there are many readers but few
        writers, allowing readers to avoid locking, but checking that data
        wasn't updated during the read

### Sleeping locks

#### Mutexes

```c
static DECLARE_MUTEX(lock);

// Calling `down_interruptible` puts the process in waiting status on the
// lock's queue.
// The function returns when the process is awaken.
if (down_interruptible(&lock)) {
    // Lock not acquired: if a signal is received while in waiting queue, the
    // process is kicked from the queue, as the process is awaken early for
    // executing the signal handler.
} else {
    // Lock acquired: critical region is executing.
    // Lock is released at the end.
    up(&lock);
}
```

### Spinlocks

#### Basic spinlock

```c
DEFINE_SPINLOCK(lock);

spin_lock(&lock);
// Lock acquired: critical region is executing.
// Lock is released at the end.
spin_unlock(&mr_lock);
```

All spinlocks have a `irq` variant which prevents the execution of local
interrupts, which in some cases can cause deadlocks.

```c
DEFINE_SPINLOCK(lock);

unsigned long flags;

spin_lock_irqsave(&lock, flags);
// Lock acquired: critical region is executing.
// No interrupt can execute here.
// Lock is released at the end.
spin_unlock_irqrestore(&mr_lock, flags);
```

Spinlocks are often implemented exploiting machine‑provided atomic instructions
such as the atomic compare‑and‑swap (CAS, `_atomic_compare_xchg`).
This guarantees that only one thread can update the variable at a time in SMP
systems.

#### RWlocks

```c
DEFINE_RWLOCK(lock);

read_lock(&lock);
// Lock acquired: can read.
// Many tasks can own the read lock simultaneously.
read_unlock(&lock);

write_lock(&lock);
// Lock acquired: can read and write.
// Only one task can own the lock when writing.
write_unlock(&lock);
```

When having many readers, writers may be starved.
Seqlocks try to fix this issue.

#### Seqlocks

```c
DEFINE_SEQLOCK(lock);

write_seqlock(&lock);
// Lock acquired: can read and write.
// Only one task can own the lock when writing.
// Sequence counter is increased both when locking and unlocking.
write_sequnlock(&lock);

do {
    // Store current sequence number.
    // It will always be even, as the function will spin until it becomes such.
    seq = read_seqbegin(&lock);
    // Read data.
    // Loop if sequence number has changed.
} while (read_seqretry(&lock, seq));
```

## Deadlocks

When configured, the kernel can run with a runtime mechanism (called `lockdep`)
for checking deadlocks, by analyzing locking sequences.
This usually is a debugging option, as it adds a big overhead.
It keeps track of locking sequences, comparing them, and checks spinlocks
acquired during interrupts or interrupts-enabled contexts.

Deadlocks can appear during interrupt handlers.
If the executing process takes a spinlock over a resource, and an interrupt
takes over before it is released, a deadlock can appear if the handler waits
for the same resource.
Moreover, in multi-processors architecture, deadlocks can also be caused by
interrupts received on other CPUs.
This involves two resources, and happens when the first CPU is locking A and
requesting B, while the second CPU is locking B and an interrupt requests A.
These are the cases in which to use `irq` variants.

## Cache aware spinlocks

### The ping pong problem

The ping pong problem occurs when multiple CPUs want to acquire a spinlocks
over the same resource, therefore they spin over the same lock.
Doing so, they repeatedly access and update the same lock memory location,
continuously invalidating the cache line for that variable in all other CPUs.
While spinning, when CPUs find the lock variable invalid in the cache they need
to reload it, which is slow.
This happens in every CPU upon each write of the variable, happening at minimum
twice per previous lock holder (when acquiring and when releasing).
The ping pong problem is not a deadlock problem, but a performance degradation
problem.

### MCS lock

The MCS (Mellor-Crummey and Scott) locks solve this problem by using a
queue-based locking mechanism.
Instead of all CPUs spinning on a single shared variable, each CPU spins on a
separate local variable.
When a CPU fails to acquire the lock, it enqueues itself in a linked list and
spins only on its own node's lock.
It also asks the previous task in line (which could/will be lock owner) to hand
off the lock when releasing it, updating the local lock and making the next in
queue exit the spinning.

### Qspinlocks

Qspinlocks are MCS locks optimized for specific hardware and performance goals.
In Linux qspinlocks, the first spinning CPU waiting in line for the lock will
not enqueue itself, but will keep spinning on the same lock of the holder, as
it won't cause ping pong effects.
This is because the first write on the lock will be the one freeing it, and a
reload from the next CPU will be required anyway.

