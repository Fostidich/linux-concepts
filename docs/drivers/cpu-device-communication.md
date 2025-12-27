# CPU device communication

## Port and memory bus

Connection between the CPU and devices can be achieved via channels named
"bus".
Communication can happen over two buses: the port and the memory bus.
Along with data transfers, you would also write commands into these registers.

In port-based IO (legacy), device registers are accessed using special CPU
instructions.
x86 systems have a UART (Universal Asynchronous Receiver‑Transmitter) serial
hardware interface for mapping device registers to the device port numbers.
Each device uses one port for each of its register, therefore one device may
use several ports: they act as their own address space.

In memory-mapped IO devices use registers mapped to reserved memory addresses.
MMIO is more convenient as it lets the CPU use classic loads and stores as if
it was writing to RAM memory cells.
The hardware is then responsible to deliver this data directly to the device
instead.

Port-based IO and memory-mapped IO both access device registers, but the first
uses a dedicated address space with ad-hoc instructions.

### Device access request

Port-based IO requests access to a port range, mapped using `ioport_map`.
Those ports/registers are accessed via instructions such as `inb`, `outb`,
`inw`, `outw`, `inl`, `outl`, based on their size.
Memory-mapped IO requests a memory region instead, mapping it via `io_remap`.
Linux also provides generic APIs like `iowrite`, which abstracts the bus used,
translating the same call either on PBIO or MMIO.

## Notifications

When a device completes its task, it must notify the CPU.

The simplest way is to use polling, where the CPU continuously checks onto
device registers for events (like spinlocks).
This is efficient for fast devices but very wasteful for slow ones.

On slow devices a better alternatives is to use interrupts.
When waiting for the device to respond, the waiting process is put to sleep and
context is switched; when the device completes, it raises an hardware interrupt
which causes the OS to stop the current execution on the CPU and start the
registered interrupt service routine (ISR).
An ISR is a custom function written by the device driver dynamically registered
within the kernel.

An alternative solution is using DMAs, which let devices write data directly to
mapped RAM addresses, bypassing the CPU, leaving it free of executing something
else.
Typically, after data has been written asynchronously, the device might call an
interrupt to notify the CPU.

