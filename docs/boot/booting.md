# Booting

General purpose PCs and embedded systems have their own booting mechanism
(firmware interfaces) which are part of the firmware and which manage the main
hardware initialization process, before calling bootloaders (like GRUB) which
then launches the OS kernel passing the necessary settings.
On GP systems the most common interface is UEFI, which is a modern alternative
replacing the BIOS (legacy).
On microcontrollers there are many simpler alternatives, with of the most
common one being U-BOOT.
Being just a firmware, UEFI alone is not enough to read and launch the OS,
therefore a bootloader like GRUB is a necessary intermediary providing the
essential flexibility and features (like reading complex file systems) for
loading the kernel.

## UEFI

The Unified Extensible Firmware Interface (UEFI) offers a modular environment
extensible with additional drivers.
Upon powering the computer, UEFI is immediately executed, taking control
initializing hardware and loading firmware settings from nvRAM
(non-volatile RAM).
A dedicated FAT32 EFI System Partition (ESP) holds the bootloader files for
various OSs; for Linux this is called `elilo.efi`.

### GPT

BIOS was constrained to 32 bit addresses, hugely limiting indexable memory.
UEFI uses instead 64 bits, addressing logical blocks in a GUID Partition Table
(GPT).
The GPT is a modern disk partitioning system, occupying sectors 1-33 at the
beginning of the disk.
Sector 0 (protected MBR) is reserved for backward compatibility with systems
not recognizing GPT.
The GPT lists many partition entries, each holding various data.

- The first 16 bytes of each entry tells the partition type (e.g. Linux FS).
- The second 16 stores the GUID (Globally Unique ID) of the partition.
- Location and size are described with the starting and ending LBAs.
- Various feature flags define attributes and permissions.
- The partition is assigned also a descriptive name.

## U-BOOT

On embedded platform, a simpler BootROM is preferred over BIOS/UEFI.
Those typically operate in two stages.
The Fist Stage Bootloader (FSBL) is minimalistic software used for initial
hardware setup.
The second (SSBL) gets called when the FSBL completes and boots the OS.
U-BOOT is one of many SSBLs.
It is a mature, well-known bootloader and supports many CPU architectures (e.g.
ARM, x86) and many peripheral drivers.
It is also able to work on various file systems

