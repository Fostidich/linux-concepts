# Interrupt management

## Device IRQ flow

Interrupt are signals sent from devices initially to a IOAPIC (IO Advances
Programmable Interrupt Controller) pin.
They then forward the interrupt to a CPU's LAPIC (Local APIC), which enable
Linux to run the corresponding IRQ (interrupt request) handlers.
IOAPICs are chips available system-wide, and they can be used by all cores;
each core has a single LAPIC component.

Each IOAPIC has many pins, assignable to multiple devices; a pin belongs to a
single IOAPIC.
A device generally connects to a single IOAPIC pin.
IOAPICs receive hardware interrupts from devices, and they in turn forward
them to the correct LAPIC.
When registering an interrupt in a IOAPIC, the pin is assigned with an
interrupt ID (8-bit vector); it is also mapped with the LAPIC ID to which it
must be sent.
Pins are indexed via their `hwirq` number, but get mapped to a new vector as
this way they become unique among all IOAPICs and they can represent other
information too, such as priority.
After the LAPIC receives the vector from a IOAPIC, Linux eventually runs the
function (`irq_flow_handler_t`) mapped to the interrupt, which can chain many
IRQ handler functions.
These IRQ handler functions are also provided by the driver at setup-time.

## Kernel IRQ flow

1. When an IRQ arrives, the CPU stops listening for other local lower-priority
    interrupts, and context switches.
2. `__do_irq`/`__handle_irq`, the main interrupt dispatcher (entry point) of
    the kernel, gets called after finding IRQ associated data from the 8-bit
    vector.
3. The `irq_flow_handler_t` function associated with the IRQ gets called.
    It abstracts all specific details/protocols for managing the interrupt.
4. A call to `irq_mask_ack` tells interrupt chips (e.g. APICs) that the IRQ
    has been received, and to avoid further deliveries.
5. Finally, the flow handler calls all the ISR actions registered in the driver
    for that interrupt.
6. When finished, `irq_unmask` tells the chip to resume IRQ deliveries.

## Handler registration

To register an interrupt handler, first define the function with a set
prototype (`irq_handler_t`).
Non-critical work should be deferred as handlers should be fast and responsive.
Also, they should not rely on process-specific data, as they are not executed
in process contexts.
The pointer to the function must then be passed to `request_irq`.
More that a single function can be associated with a `irq` line number.

```c
int request_irq(unsigned int irq, irq_handler_t handler,
                unsigned long flags, const char *name, void *dev)
```

### Deferring work

Non-critical work is deferred with the soft IRQs APIs.
Interrupt handlers can register functions to be executed later.
This is useful as the CPU leaves more time available for receiving interrupts.
Pending work gets executed when calling `do_softirq`, happening either after
returning from hard IRQs, or from the `ksoftirqd` kernel threads, run when too
much work piles up.

The "top half" of the interrupt runs minimal work in non-interruptible context,
deferring the rest later.
The "bottom half" finalizes the work at reconciliation points.
Soft IRQs can be of a few types.

#### Tasklets

Tasklets are functions which the kernel ensures run once and securely on a
single CPU, without sleeping.
From the same IRQ, different tasklets with distinct handlers can be registered,
but the same handler cannot be schedules twice, avoiding redundant runs if
interrupts fire rapidly.

Tasklets are enqueued in a list of `tasklet_struct`; during reconciliation
points, the kernel checks this list for scheduled tasks.
The struct is registered with `tasklet_schedule`.

```c
DECLARE_TASKLET(tasklet, handler);
tasklet_schedule(&tasklet);
```

#### Work queues

If deferred actions must block (use locks, perform IO), they can be appended to
a work queue, whose work gets executed later, in process context, by worker
kernel threads (named "events/0", "events/1"...).

```c
DECLARE_WORK(work, handler);
schedule_work(&work);
```

#### Timers

Precise sub-milliseconds timing can be achieved via the `hrtimer` subsystem.

```c
struct hrtimer timer;
hrtimer_setup(&timer, handler, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
hrtimer_start(&timer, ms_to_ktime(100), HRTIMER_MODE_REL);
```

