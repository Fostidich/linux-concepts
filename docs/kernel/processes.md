# Processes

## Tasks

In Linux, both processes and threads fall into the definition of task.

A task consists of

- a task counter,
- a stack in user mode,
- a stack in kernel mode,
- a set of CPU registers,
- an address space.

If the address space is shared with another task, those are called threads,
processes otherwise.
Kernel threads share the whole kernel space.

The PCBs of tasks are represented by the `task_struct`, organized in a list.
The pointer to the current running process' node may be stored either in a
register or in the kernel stack.
This struct contains various data; the most important ones are

- `preempt_count`, that allows context switches to happen only in safe points,
- `thread_struct`, which contains values of registers of non-running tasks,
- `mm_struct`, storing memory mappings.

### States and queues

Tasks can fall into different states.

- Running: task is executing in the CPU
- Ready: task is in queue ready for being executed
- Stopped: task is paused, waiting for a signal to continue.
- Interruptible sleep: task is waiting for a resource, but can receive signals.
- Uninterruptible sleep: task is waiting for the end of an atomic action,
ignoring signals.

When `wait_event` or `wait_event_interruptible` is called, the process is put
into the queue of the corresponding waited event as a `wait_queue_entry`.
When the event occurs, a `wake_up` function is called in order on each process
in the queue, marking processes as running.
However, not all processes are woken up: processes marked with and "exclusive"
flag (which are stored in a separate queue) need exclusive access to the
resource; therefore, only the first is woken up, skipping the others.
All non-exclusive processes gets awoken.

### Kernel initialization

In order to create a new process, the `fork` function (invoking `sys_clone`) is
called.
The process is duplicated, giving the child a new PID and PPID (Parent PID).
The copy differs only for some resources, such as pending signals.
Memory space is not duplicated, as it remains shared until one process writes
on a page, which gets thus copied (Copy on Write).

On startup, the kernel executes a series of initialisation calls in a well
defined trace.

1. `start_kernel`, found in `/init/main.c`, executes architecture setup
routines
2. `rest_init` creates a new kernel thread executing `kernel_init`
3. `rest_init` then evolves into the low-power, PID 0 idle thread
4. `kernel_init` then calls `initrd_load`, responsible for initial RAM setup
5. `kernel_execve` runs `/bin/init` with PID 1, initialising long-running
services

The init task have been implemented in many ways.
Initially, it was managed by the SystemV process, which has now been replaced
by a more modern SystemD.
SystemV worked by dividing startup scripts into levels, for then executing each
one by one (which is slow).
SystemD is more simple, efficient and prone to parallelisation: each service is
defined in its corresponding text file containing all necessary configurations
and dependencies; services can be grouped in targets, and a tool is provided
for managing (start, stop, query) services.
SystemD is declarative: you define what you need and the system figures out how
to achieve it.

## Task scheduling

During the execution of instructions, exceptions may arise.
These differ from interrupts as they are internal and synchronous, and are to
be managed before preemption (context switch).
Interrupts on the other hand are external and asynchronous, as they come from
resources, often hardware device.
Furthermore, interrupts will pause and resume programs, whilst exceptions are
instantly resolved, even if they will make the program crash.
Exceptions are found, for instance, upon a division by zero or invalid memory
address accesses.
When an exception is found, `preempt_count` is increase, and decrease when
treated.
The `switch_to` function can be called only when this count gets to zero.

Developer can implement their own scheduling policy by calling functions of a
scheduling class.
Most important ones are:

- `task_tick` updates the current task time statistics;
- `pick_next_task` selects the next from the queue;
- `select_task_rq` selects the core on which task gets enqueued;
- `enqueue_task` enqueues the task.

All processes have a priority value.
For real-time processes, the `rt_priority` is a value between 0 and 99.
For non-real-time processes, `static_prio` takes value between 100 and 139, but
it is stored as an interval from -20 to +19.

### Completely fair scheduling

CFS assigns time slices based on system load.
Each process has a variable `vruntime` which represents its absolute measure
of dynamic priority.
The lower it is, the higher is the priority.
All processes have consumed their time share if they all have reached the same
value for `vruntime`.
The computation of this variable mainly depends on an assigned process weight
value (relative to all other's).
The higher the weight, the longer is the time slice.

### Earliest eligible virtual deadline first


EEVDF is similar to CFS but with fixed time slices.
Each process tracks its expected CPU time and actual CPU time; their difference
is called lag.
Processes with positive lag are eligible to run.
Among eligible processes, EEVDF selects the one with the earliest virtual
deadline to run next.

Tasks with higher priority are assigned smaller fixed time slices.
Tasks with lower nice values have higher weights and thus higher priority.
Moreover, time slices are also influenced by the process latency-nice value,
which is introduced to prioritize processes which need short time slices but
are latency-sensitive.

### Additional fairness

If a user has more threads than another, it would get more CPU time as it would
have more tasks in the run queue.
To ensure that a user with fewer threads isn't penalized, the scheduler is
modified to account for user‑level shares first.
This is achieved by looking at users as they were single tasks in a root run
queue; then, each user has assigned a specific run queue where sub-tasks take
turn to execute.
Task groups are called CGroups.
Furthermore, CGroups allow to move tasks in two groups: workload and
background.
Weights can be assigned to both to allocate more resources to workload ones.

Multi-CPU systems also allow for a better load balancing.
The designated core is the idlest core which periodically tries to steal
threads from busiest ones, to balance the load.
This process relies on a cache hierarchy tree, representing cores grouped by
shared cache levels.
Each core maintains a load value, and cores propagate load information upward
in the hierarchy so that nearby cores are aware of which have higher loads.

