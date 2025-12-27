# Platform configuration

## ACPI

When booting, the kernel must know the devices present on the machine.
The firmware typically uses one of two standards to provide platform data:
either the Advanced Configuration and Power Interface (ACPI, mainly used on
general purpose systems) or Device Trees (DT, used in embedded devices).

ACPI is a standardized method for initializing system components.
Its abstraction permits a single kernel binary to run on different platforms.
What the kernel receives from the ACPI is a set of tables containing AML (ACPI
Machine Language) and registers.
Tables with static data are obtained via the BIOS/UEFI firmware.
ACPI registers (mapped via MMIO) can be used for control tasks, like checking
the power status or set a sleep state.
The ACPI driver in the kernel provides an interpreter able to execute AML code.
Via the OSPM (Operating System-directed configuration and Power Management)
module, the kernel can set, via ACPI, sleep states for the CPU, reducing
consumption when idle.
In user space, interactions with the ACPI can be done via a daemon like
`acpid`.

### Namespace

One of the ACPI components is the Differentiated System Description Table
(DSDT), a block of memory provided by the firmware containing AML.
It contains a representation of all devices in the system (namespace) and the
interfaces for managing their power states.
For example, the state of a battery named `BAT0` can be obtained with
`cat /proc/acpi/battery/BAT0/state`, which calls the `_BST` control method.

### Power management

High temperatures lower the reliability of hardware.
Each component is labeled with a TDP (Thermal Design Power), describing the
maximum heat generated under standard conditions.
The main source of consumption are the voltage and the frequency.
To ease power management, instead of controlling directly multiple
inter-depending variables, devices are designed with predefined states.

ACPI exposes five different layers of power states (G, S, C, P, D types).

- G-types group S-types and describe the system state in a high-level,
behavioural way.
- S-types (system states) concretely define what G-types do.
- C-types refer to CPU states; via specific instruction, the clock speed can be
reduced.
- P-types allow further control over the CPU performance state based on workload.
- D-types specifically refer to devices.

G/S states are managed by the ACPI daemon, C/P/D by the OSPM.
When the scheduler, controlling workload, requires a P state change, it
communicates it to a OSPM module, which via the ACPI changes a given register
(PCM, Performance Control Machine).
In user space, events are forwarded via the `sysfs`, and the ACPI daemon
(listening to it) transitions to the new state by editing `/sys/power/state`.

## Device tree

Device trees are data structures that provide information about hardware
topology where devices cannot be probed.
They come in two forms: a textual, human-readable version (`.dts`) and a
compile version that OSs can interpret (`.dtb`).
DTs are written by developers, they are not auto-generated, therefore, you'd
write the textual version for then compiling it into the blob.

The DT contains hierarchical definitions for device nodes.
Upon providing a DT specification name and version, the tree start from a root
node, identified by `/`.
Each node may have many properties, such as `compatible`, defining vendor and
model names, used to match it with their specific driver.
The `cpu` field is used to describe each CPU in the system.
Many other nodes can be used to introduce other system components, like serial
devices and flash memories.

DTs also provide configuration capabilities for interrupt controllers and
interrupt signal declarations, expressed as links between nodes.
Interrupts are specified by providing an interrupt number and a set of optional
flags.

