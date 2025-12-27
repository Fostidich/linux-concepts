# Device management

## Device categories

Linux supports three kinds of devices: [character devices](#character-devices),
[block devices](#block-devices) and network devices.
Each is accessed in a unique way.

Character devices are peripherals like keyboards and monitors, and they can be
accessed as normal files, directly, without buffering.
Block devices, like hard disks, can be accessed only in multiples of block
size.
Network devices use specific interfaces and subsystems.

Linux tries to hide hardware details as much as possible, by integrating
devices' drivers into the file system as special files, usually found in the
`/dev` folder.
Each driver is assigned with a major device identification number, with each
device using it assigned with a minor device number.

### Character devices

When registering a character device, the diver must implement the
`file_operations` interface, defining functions to manage operations such as
`read`, `write` and `*ioctl`.

```c
struct file_operations {
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    int (*open) (struct inode *, struct file *);
    /* many other operations */
}
```

### Block devices

Data accesses on block devices are achieved using a layered approach.
First, the VFS (Virtual File System) receives requests for file accesses and
translates them to page operations.
If the page is in the page cache, data is sent/returned directly.
Otherwise, the VFS uses the mapping layer to determine which device blocks hold
that page.
Requests are then forwarded to the block IO layer (`bio`), which aggregates
accesses before passing them to the driver.
Requests need to be enqueued, therefore the device driver must implement a
`request_queue`.
When accessing the device in `/dev`, a `gendisk` object can be reached which
holds, among other disk information, a pointer to the `request_queue`.

#### Requests compression

When accessing a non-cached page, the mapping layer first identifies the
segments (contiguous sectors on disk) composing the page.
Each segment is translated to a `bio_vec`, which is its kernel representation
comprehending other data like locations and offsets.

Then, all the `bio_vec` of the page are collected and aggregated where
possible, creating new structures simply called `bio`, describing a contiguous
area in memory.
Each `bio` is then converted to an actual request sent to the block layer.

The block layer tries to further optimize `bio` structs, by merging several
physically adjacent requests, coming from different page accesses, together
waiting in the staging queue.
Staged requests are delayed by a factor depending on system load.
This first queue is called the "software" queue.

Requests get then forwarded to the "hardware" queue, which is directly read
by the driver sending to the device.
Fast storage devices can support multiple hardware queues, which permit
concurrent execution of multiple requests.

#### Scheduling

How `bio` requests must be submitted depends on a IO scheduler algorithm, whose
purpose is to improve performance and ensure fair bandwidth use, by minimizing
delays, sorting requests or reflecting priorities.
The following are the main IO schedulers proposed by Linux.

- Noop: ideal for SSDs, as they don't have slow mechanical parts.
- Budget Fair Queueing: bandwidth is divided equally.
- Kyber: reads, which are faster, are prioritized over writes, depending on
    execution time.
- MQ-Deadline: reads are prioritized based on operation deadlines.

## Device model

Since the Linux kernel runs on many different architectures, it aims at
maximizing code reusability, via various layers of abstraction and device
representations.
The high-level device model has many purposes, such as creating a device
hierarchy, binding drivers to devices, power management, hot-plugging or
device events forwarding.

A driver operates on three main components: `struct device`,
`struct device_driver`, `struct bus_type`.
These structs are often extended with custom data to better represent
underlying hardware details.

### Matching flow

Linux sees devices typically using only one of two firmware interfaces,
depending on architectural choices: ACPI or device tree.
Firmware sees hardware and builds a structure representing the device
topology; this structure is either an ACPI table or a device tree blob, which
the kernel can lookup.
ACPI supports sending events to the kernel, whereas DT exploits interrupts.

The fist element setup is the bus.
A bus receives events of additions and removals of devices and manages their
linkage with the corresponding driver.
Drivers are registered to their bus via `register_driver`.
When a device is connected, its driver sends a `probe` which initializes it.
Devices are discovered by the system, which notifies its bus; devices can
appear at runtime.

### Kobjects

Kobjects are the kernel representation of objects like bus, divers and devices.
They are exposed to user space via the `sysfs`, which maps objects to
directories and their attributes to the files they contain.
Kobjects are hierarchically organized in ksets (folders), mimicking hardware
topology.
Once an object is created (e.g. in `/sys/usb/devices`) the user device manager
(`udev`) detects it and creates a device node in `/dev`.
`udev` is a daemon that manages device nodes based on kernel events from
`sysfs`.

### Bus framework

To simplify interactions with complex bus systems, the Linux kernel uses an
abstraction layer (bus frameworks) to represent different hardware
communication channels.

Examples are the Platform BF, which is used to retrieve fixed hardware blocks
or chips at boot time, USB, which supports hot-plugging by periodically polling
the hubs status, or PCI, which adopts interrupts for notifying
self-enumeration.
Platform chips can be read in the ACPI/DT, USB builds a port-based tree of
devices, PCI assigns bus numbers to nodes organized in a matrix/graph.

#### PCIe

PCI Express is a point-to-point, high-speed bus standard, which may link CPU
and GPU, SSDs and network interfaces.
The PCI topology comprises three types of components, organized in a
tree/hierarchy: the root complex, endpoints and bridges.
These nodes are connected with buses (mainly PCIe).
The root complex is the root of this tree, and is the interface used by the
host (CPU and memory) to access devices.
PCIe leaf devices are called endpoints.
Bridges are internal tree nodes connecting the root complex to endpoints.
They may also connect to some non PCIe device, acting as translators.
Each PCI node is assigned with an ID in the form of `bus:device.function`:
bus and device are just the corresponding hardware IDs; function is used to
differentiate logical partitions of the device.

The identification of devices in the kernel is called enumeration.
At boot time, the firmware reserves space (typically via MMIO) for storing a
configuration structure, maintaining devices data like device ID, vendor ID and
BARs (Base Address Registers).
Linux retrieves PCI configurations and setups the necessary drivers.
Upon detection, the driver sends the `probe` function, reading configurations
end establishing the connection.

```c
int probe(struct pci_dev *pdev, const struct pci_device_id *id) {
    int ret;
    // BAR number (from driver datasheet knowledge).
    int b = 0;
    // Enable PCI device power and memory IO.
    pci_enable_device(pdev);
    // Request exclusive access to BAR b memory region.
    ret = pci_request_region(pdev, b, "pci-driver");
    // Get BAR start address and length from PCI config space.
    resource_size_t mmio_start = pci_resource_start(pdev, b);
    resource_size_t mmio_len = pci_resource_len(pdev, b);
    // Map physical MMIO into kernel virtual address space.
    void __iomem *hwmem = ioremap(mmio_start, mmio_len);
    // Test MMIO write using a magic value to verify hardware access.
    iowrite32(0xbaddcafe, hwmem);
    // Allocate only 1 IRQ vector.
    ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_ALL_TYPES);
    // Get the allocated IRQ vector number.
    int irq = pci_irq_vector(pdev, 0);
    // Register IRQ handler for device interrupts.
    ret = request_irq(irq, irq_handler, 0, "pci-driver", pdev);
    // Here you would save state (`hwmem`, `irq`) in some `pci_set_drvdata`.
    return 0;
}
```

