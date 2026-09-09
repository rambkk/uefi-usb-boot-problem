# UEFI USB Boot Problem
Investigation and troubleshooting of UEFI USB boot problems, including removable/fixed device detection, EFI fallback paths, and boot entries.

This project started with a simple problem:

> **A Ventoy USB flash drive was created successfully, but the notebook's UEFI firmware would not automatically boot it.**

The USB drive was perfectly usable from the operating system, and the expected EFI boot files were present. However, the firmware did not behave as expected when attempting to boot from the USB device.

This led to an investigation into two related areas:

1. How USB storage devices are classified as **removable** or **fixed/non-removable**, and whether that classification can affect how UEFI firmware discovers bootable USB devices.
2. How **UEFI boot entries and fallback EFI paths** work, including entries that can be inspected or modified with tools such as `efibootmgr` and Windows `bcdedit`, and firmware-specific behavior observed through BIOS/UEFI firmware analysis.

This project documents the investigation, commands used to examine devices and boot entries, experiments, and a case study based on an MSI notebook firmware image.

---

 ## Table of Contents

- Background
- The Initial Problem: Ventoy
- Workaround Solution
- What Is Being Investigated
- USB Storage Device Classification
  - Removable vs Fixed
  - Linux
  - Windows
  - Why This Matters to UEFI
- UEFI Boot Entries
  - UEFI Boot Variables
  - Linux: efibootmgr
  - Windows: bcdedit
- EFI Fallback Boot Paths
  - BOOTX64.EFI
  - Explicit Boot Entries vs Fallback Discovery
  - Firmware-Specific Paths
- MSI Firmware Case Study
  - Firmware Extraction
  - EFI Path Strings
  - What the Strings Do and Do Not Prove
- Boot Discovery Model
- Troubleshooting Methodology
- Experiments
- Repository Structure
- Limitations
- Safety and Caution
- Goals of This Project
- Further Research
- License

---

## Background

UEFI systems provide several mechanisms for discovering and launching EFI applications.
On a conventional system, the firmware can have explicit boot entries stored in UEFI NVRAM, for example:

```
Boot0000* Windows Boot Manager
Boot0001* ubuntu
Boot0002* USB Device
```

A boot entry can contain information describing the device, filesystem, and EFI executable that should be launched.
UEFI also defines conventions for discovering bootloaders without relying entirely on a pre-existing NVRAM boot entry.
For **removable** media, the conventional x86-64 fallback path is:

```
\EFI\BOOT\BOOTX64.EFI
```

In an idealized situation, a bootable USB drive containing:

```
EFI/
└── BOOT/
    └── BOOTX64.EFI
```

should therefore be discoverable by UEFI firmware as removable boot media.

In practice, firmware implementations differ.

Some systems appear to apply additional rules when enumerating USB storage devices, determining which devices should be considered bootable, or deciding which EFI paths should be attempted.

This project investigates some of those behaviors.

---
# The Initial Problem: Ventoy
The investigation began while trying to install [Ventoy](<https://www.ventoy.net/>) on a USB flash drive. Ventoy was installed successfully and the USB drive could be accessed normally from the operating system. However, the notebook's UEFI firmware did not automatically boot the Ventoy EFI loader from the USB drive. This was interesting because the problem did not initially look like a corrupted filesystem or missing EFI executable.

The investigation therefore moved through several questions:
```
Is the USB device detected by the OS?
            │
            ▼
How does the OS classify the device?
            │
            ▼
Is it reported as removable or fixed?
            │
            ▼
Does the UEFI firmware enumerate it as bootable?
            │
            ▼
Does a UEFI Boot#### entry exist?
            │
            ▼
Does the firmware attempt \EFI\BOOT\BOOTX64.EFI?
            │
            ▼
Does the firmware have additional
vendor-specific EFI paths or boot rules?
```
The purpose of this project is to document those questions and the experiments used to investigate them.

---
## Workaround Solution
The following workaround was found to make the Ventoy USB drive appear as a **Windows Boot Manager** entry in the UEFI boot selection menu.

This was tested on the affected MSI notebook and allowed the system to boot into Ventoy successfully.

### Steps
On the EFI System Partition of the Ventoy USB drive, create the following directory:
```
EFI/
└── MICROSOFT/
    └── BOOT/
```
The existing Ventoy EFI directory contains several files. The files relevant to this workaround are:
```
EFI/
└── BOOT/
    ├── ...
    ├── BOOTX64.EFI
    ├── ...
    ├── grubx64_real.efi
    ├── ...
    └── fbx64.efi
```

Copy the Ventoy `BOOTX64.EFI` to the Microsoft-style bootloader path:
```
/EFI/BOOT/BOOTX64.EFI
    └──> /EFI/MICROSOFT/BOOT/bootmgfw.efi
```

Then there are **two possible ways** to provide the second EFI executable.
#### Option 1: Copy `grubx64_real.efi`
Copy:
```
/EFI/BOOT/grubx64_real.efi
```
to:
```
/EFI/MICROSOFT/BOOT/grubx64.efi
```

The relevant resulting files are:
```
EFI/
├── BOOT/
│   ├── ...
│   ├── BOOTX64.EFI
│   ├── ...
│   ├── grubx64_real.efi
│   ├── ...
│   └── fbx64.efi
│
└── MICROSOFT/
    └── BOOT/
        ├── bootmgfw.efi
        └── grubx64.efi
```

The resulting relationship is:
```
EFI/BOOT/BOOTX64.EFI
        │
        └──> EFI/MICROSOFT/BOOT/bootmgfw.efi

EFI/BOOT/grubx64_real.efi
        │
        └──> EFI/MICROSOFT/BOOT/grubx64.efi
```

#### Option 2: Copy `fbx64.efi`
Alternatively, instead of copying `grubx64_real.efi`, copy:
```
/EFI/BOOT/fbx64.efi
```
to:
```
/EFI/MICROSOFT/BOOT/grubx64.efi
```

The relevant resulting files are:
```
EFI/
├── BOOT/
│   ├── ...
│   ├── BOOTX64.EFI
│   ├── ...
│   ├── grubx64_real.efi
│   ├── ...
│   └── fbx64.efi
│
└── MICROSOFT/
    └── BOOT/
        ├── bootmgfw.efi
        └── grubx64.efi
```

The resulting relationship is:
```
EFI/BOOT/BOOTX64.EFI
        │
        └──> EFI/MICROSOFT/BOOT/bootmgfw.efi

EFI/BOOT/fbx64.efi
        │
        └──> EFI/MICROSOFT/BOOT/grubx64.efi
                    │
                    └──> launches EFI/BOOT/grubx64_real.efi
```
This second variant also works because `fbx64.efi` acts as Ventoy's fallback mechanism and is able to locate and launch `grubx64_real.efi`.

### Result
After making these changes and rebooting into the UEFI boot selection menu, a new entry appeared:

```
Windows Boot Manager
```
Selecting **Windows Boot Manager** launched the Ventoy EFI bootloader, and the system successfully entered the Ventoy menu.

### Why this is interesting
This suggests that the firmware was willing to recognize and offer an EFI executable located at the Microsoft-style boot path:
```
\EFI\MICROSOFT\BOOT\bootmgfw.efi
```
even though it did not automatically offer the Ventoy removable-media fallback loader:
```
\EFI\BOOT\BOOTX64.EFI
```
The important part of the workaround is that `bootmgfw.efi` is actually a copy of Ventoy's `BOOTX64.EFI`.

### Important limitation
This behavior has only been confirmed on the system tested as part of this investigation. It should not be assumed that the same workaround will work on every motherboard or UEFI implementation.

The workaround is particularly useful as an experimental observation because it provides a clue about how the firmware is discovering EFI bootloaders.

It demonstrates that the firmware can discover an EFI executable through the Microsoft-style boot path even when it does not automatically expose the standard Ventoy removable-media fallback path as a boot option.

---
# What Is Being Investigated
The project focuses on two related but distinct mechanisms.

## USB storage classification
Operating systems expose information about whether a storage device or block device is considered removable.
For example, Linux exposes a `removable` attribute for block devices.
A USB flash drive may therefore appear to the operating system as either:
```
removable = 1
```
 or:
```
removable = 0
```
depending on how the hardware and storage stack report the device.

The second case can be particularly interesting because a device that physically looks like a USB flash drive may be treated as a fixed/non-removable disk.

This does **not** mean that every UEFI implementation will reject such a device. Rather, the question is whether particular firmware implementations use this or related information when deciding how to enumerate and boot USB storage.

---

## UEFI boot configuration

The second part of the investigation concerns UEFI NVRAM boot entries.

These entries can normally be inspected from Linux with:

```
efibootmgr -v
```

 and from Windows using:

```
bcdedit /enum firmware
```

The entries may contain references to EFI executables on particular filesystems.
This raises another question:

> What happens when there is no explicit boot entry for an EFI executable?

That leads to the EFI fallback mechanism.

---

## Firmware-specific discovery

UEFI defines standard mechanisms, but firmware vendors can implement additional discovery logic.
A firmware image may contain strings corresponding to EFI executables or paths such as:
```
\EFI\BOOT\BOOTX64.EFI
```
or vendor-specific locations.

Finding such a path in a firmware image does not automatically prove that the firmware will create a NVRAM boot entry for it.
It does, however, provide a useful starting point for further investigation.
The MSI notebook case study in this project is an example of this type of investigation.

---

# USB Storage Device Classification

## Removable vs Fixed

There are several different meanings of the word "removable" in a computer system.

A device can be:
- physically removable;
- connected through USB;
- reported by the USB device as removable media;
- reported by the operating system as a removable block device;
- treated by firmware as removable boot media.

These concepts are related, but they are not necessarily identical.

For example, a USB flash drive can physically be removed from a computer while the operating system or firmware treats the underlying storage as a fixed disk.

Therefore, this project uses terms such as **removable** and **fixed/non-removable** carefully and specifies which layer is being discussed.

---

## Linux
Linux exposes useful information through `/sys`.
For a block device such as `/dev/sdb`, check:
```
cat /sys/block/sdb/removable
```

Typical results are:
```
1
```

 or:

```
0
```

A value of `1` generally indicates that the kernel considers the block device removable.
A value of `0` indicates that the kernel considers it non-removable.

### Using lsblk
A convenient overview is:

```
lsblk -o NAME,TRAN,RM,RO,TYPE,SIZE,MODEL
```
Example:
```
NAME        TRAN   RM RO TYPE    SIZE MODEL
sda         usb     0  0 disk    1.8T YSUWD10-2TSY
sdb         usb     1  0 disk  116.1G USB Flash Drive
nvme0n1     nvme    0  0 disk  931.5G Samsung SSD 970 EVO Plus 1TB
```

 The important columns are:
- `TRAN` — transport type, such as `usb` or `nvme` or `sata` ...
- `RM` — removable flag
- `RO` — read-only state
- `TYPE` — device type
- `MODEL` — reported device model

### sysfs

For a specific device:
```
cat /sys/block/sdX/removable
```
Replace `sdX` with the actual device.
For example:
```
cat /sys/block/sdb/removable
```

### udev information
Additional device properties can be examined with:
```
udevadm info --query=property --name=/dev/sdb
```
This can be useful when investigating how the Linux device-management stack identifies the hardware.
---

## Windows
Windows exposes storage-device information through several interfaces.
PowerShell can be used to inspect physical disks:
```
Get-Disk
```
and additional information can be obtained with:
```
Get-PhysicalDisk
```
For a particular disk:
```
Get-Disk | Format-List Number,FriendlyName,BusType,MediaType,IsBoot,IsSystem
```
For USB devices, additional information can be obtained through Windows device-management and WMI/CIM interfaces.
For example:
```
Get-CimInstance Win32_DiskDrive |
    Select-Object Index,Model,InterfaceType,MediaType
```
The exact properties and terminology do not map perfectly to Linux's `removable` attribute.

The purpose of these commands is therefore to establish how Windows identifies the storage device, not to claim that one particular Windows property is equivalent to every firmware's definition of removable media.

---

# Why This Matters to UEFI

A simplified boot process might look like:

```
USB device
    │
    ▼
USB storage controller
    │
    ▼
UEFI USB/storage driver
    │
    ▼
Storage device enumeration
    │
    ▼
Device/media classification
    │
    ▼
Filesystem discovery
    │
    ▼
EFI bootloader discovery
    │
    ▼
EFI application
```

Firmware may make decisions at several of these stages.
A USB device that is perfectly usable by an operating system can therefore still behave differently during pre-OS boot.
One hypothesis investigated by this project is that some firmware implementations may treat a USB storage device differently depending on whether the device is presented as removable or fixed/non-removable.

This should be considered a **firmware-specific behavior to be tested**, rather than a universal UEFI rule.
---

# UEFI Boot Entries

## UEFI Boot Variables
UEFI stores boot configuration in NVRAM variables.
Boot entries are commonly named:
```
Boot0000
Boot0001
Boot0002...
```
A firmware can maintain a boot order describing which entries should be attempted first.
Conceptually:
```
BootOrder
    │
    ├── Boot0000
    ├── Boot0001
    └── Boot0002
```

Each entry can contain information identifying an EFI executable and the storage device on which it resides.
This is different from simply having an EFI file on a disk.
For example:
```
EFI file exists:
    \EFI\BOOT\BOOTX64.EFI
```
does not necessarily mean:
```
UEFI NVRAM contains:
    Boot000X -> \EFI\BOOT\BOOTX64.EFI
```
The two mechanisms should be considered separately.

---

# Linux: efibootmgr

On a Linux system booted in UEFI mode, `efibootmgr` can be used to inspect firmware boot variables.

```
sudo efibootmgr
```
For more detailed information:

```
sudo efibootmgr -v
```
Example:
```
BootCurrent: 0001
Timeout: 1 seconds
BootOrder: 0001,0000,0002
Boot0000* Windows Boot Manager
Boot0001* ubuntu
Boot0002* USB Device
```
The verbose output can provide information about the device and EFI executable associated with an entry.

### Changing boot order
For example:
```
sudo efibootmgr -o 0001,0000,0002
```
would set the boot order to:
```
Boot0001
Boot0000
Boot0002
```
The exact entries obviously depend on the system.

### Important caution
`efibootmgr` modifies UEFI NVRAM variables.
Incorrect changes can make an operating system temporarily disappear from the firmware boot menu or alter the normal boot process.
Always inspect the existing configuration before modifying it.

---

# Windows: bcdedit

Windows provides access to firmware boot information through `bcdedit`.
To enumerate firmware entries:
```
bcdedit /enum firmware
```
This can show firmware boot-manager entries known to Windows.
This is useful for comparing the state of UEFI boot configuration between Linux and Windows.
For example:
```
Firmware Boot Manager
---------------------
identifier              {fwbootmgr}
displayorder            {...}
timeout                 0
```
The exact output depends on the system.
Windows and Linux do not necessarily present the information in identical formats, but both can be useful when investigating UEFI NVRAM configuration.

---

# EFI Fallback Boot Paths

## BOOTX64.EFI

UEFI defines a conventional fallback location for **removable** media.
For x86-64 systems, the conventional path is:
```
\EFI\BOOT\BOOTX64.EFI
```
Therefore, a removable UEFI boot disk can contain:
```
EFI/
└── BOOT/
    └── BOOTX64.EFI
```
without necessarily having a pre-existing UEFI NVRAM entry pointing to that exact file.
This is one of the reasons the removable-media boot mechanism is useful for installation media, rescue systems, live Linux distributions, and tools such as Ventoy.

---
## Explicit Boot Entries vs Fallback Discovery

It is useful to think of UEFI booting as having at least two broad mechanisms.

### Mechanism 1: Explicit boot entry
The firmware has an NVRAM entry that effectively says:

```
Device X
    ↓
Filesystem
    ↓
EFI application
```
For example:
```
Boot0001
    ↓
USB device
    ↓
\EFI\Vendor\loader.efi
```

### Mechanism 2: Fallback discovery
The firmware decides to examine a device as removable boot media and looks for:

```
\EFI\BOOT\BOOTX64.EFI
```

These mechanisms are related but are not the same thing.
A USB device can contain a perfectly valid:
```
\EFI\BOOT\BOOTX64.EFI
```
while there is no corresponding:
```
Boot000X
```
entry in NVRAM.

Conversely, a firmware boot entry can point directly to an EFI application without relying on the removable-media fallback path.

---

# Firmware-Specific Paths
One of the more interesting parts of this investigation is the possibility of firmware-specific EFI paths.
Firmware implementations can contain code and data referring to particular EFI executables or paths.
For example, firmware analysis may reveal strings resembling:
```
\EFI\...
```
or complete paths to vendor-specific EFI applications.
These can potentially represent:
- built-in boot targets;
- vendor recovery mechanisms;
- firmware update mechanisms;
- diagnostic environments;
- OS-specific bootloaders;
- default paths used during device discovery;
- other internal firmware functionality.

The exact meaning cannot be determined from the string alone.
That distinction is important throughout this project.

---

# MSI Firmware Case Study
## Firmware Extraction

As part of this investigation, an MSI notebook firmware image (MSI GF75-THIN 9SC)was examined. The firmware image was extracted and searched for strings corresponding to EFI executable paths. This produced a list of paths that appear to be referenced by the firmware.

The extracted list is included under:
```
\EFI\debian\grubx64.efi
\EFI\opensuse\grubx64.efi
\EFI\Centos\shim.efi
\EFI\Fedora\shim.efi
\EFI\Redhat\shim.efi
\EFI\Redhat\grub.efi
\EFI\Redhat\elilo.efi
\EFI\Ubuntu\shimx64.efi
\EFI\Suse\elilo.efi
\EFI\Microsoft\Boot\bootmgfw.efi
\EFI\sles12\grubx64.efi
\EFI\sles12\shim.efi
```

The MSI example is intended as a **case study**, rather than a claim that all MSI systems behave identically.
Different MSI notebook models, firmware versions, and UEFI implementations may behave differently.

---

## EFI Path Strings
The extracted strings are useful because they provide clues about paths that firmware code may know about or use.
For example, if a firmware image contains:
```
\EFI\something\loader.efi
```
this tells us that the string exists in the firmware image.

It does **not**, by itself, establish:
- when the path is used;
- which devices are searched;
- whether the path is searched automatically;
- whether an NVRAM boot entry is created;
- whether the path is only used by a particular firmware module;
- whether the path is used on USB storage;
- whether the path is used on every boot.

Further runtime testing is necessary.

---

## What the Strings Do and Do Not Prove
This distinction is important.

### Observed
The EFI path string exists in the extracted firmware image.

### Experimentally observed
Putting an EFI application (or an empty file) at a particular path causes the firmware to discover or execute it under a particular set of conditions.

### Inferred
The firmware may use the path as part of its boot-device discovery process. These three statements have very different levels of confidence. The project attempts to distinguish them rather than treating firmware string extraction as definitive proof of runtime behavior.

---

 # Boot Discovery Model
 A useful conceptual model for this project is:
```
                         UEFI Firmware
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
       NVRAM Boot Entries              Device Discovery
              │                               │
       Boot0000 / Boot0001...                 │
              │                               │
              │                      ┌────────┴────────┐
              │                      │                 │
              │                 USB device       Other devices
              │                      │
              │                      ▼
              │                Device classification
              │                      │
              │                      ▼
              │                Filesystem discovery
              │                      │
              │                      ▼
              │               EFI boot path search
              │                      │
              │              ┌───────┴────────┐
              │              │                │
              │        Standard fallback   Vendor-specific
              │              │                │
              │              ▼                ▼
              │        \EFI\BOOT\       firmware-specific
              │        BOOTX64.EFI         paths
              │
              └───────────────┬───────────────┘
                              │
                              ▼
                         EFI application
```

This is intentionally a simplified model.
Actual firmware implementations can be considerably more complicated.

---

# Troubleshooting Methodology
When a USB device does not appear as bootable, this project recommends investigating the problem layer by layer.

## Step 1: Confirm the USB device is detected
From Linux:
```
lsusb
```
and:
```
lsblk
```

From Windows:
```
Get-Disk
```

---

## Step 2: Determine how the operating system classifies it
Linux:
```
lsblk -o NAME,TRAN,RM,SIZE,MODEL
```
and:
```
cat /sys/block/sdX/removable
```

Windows:
```
Get-Disk
```
and:
```
Get-CimInstance Win32_DiskDrive | Select-Object Index,Model,InterfaceType,MediaType
```

---
## Step 3: Check the EFI filesystem
Verify that the expected EFI files actually exist and the partition must be in **FAT12 or FAT16 or FAT32** format.
For a removable x86-64 UEFI bootloader:
```
\EFI\BOOT\BOOTX64.EFI
```
For Ventoy, also inspect the filesystem layout produced by the particular Ventoy installation.

---
## Step 4: Check UEFI NVRAM entries
Linux:
```
sudo efibootmgr -v
```
Windows:
```
bcdedit /enum firmware
```
Determine whether the firmware has an explicit entry for the USB device or its EFI loader.

---

## Step 5: Test the firmware's boot menu
Check whether the USB device appears in:
- the normal boot menu;
- the one-time boot menu;
- the firmware setup;
- the list of storage devices.

A device appearing in the firmware's storage list does not necessarily mean it will appear as a bootable EFI device.

---

## Step 6: Investigate firmware-specific behavior
If the standard fallback path is present but the device is still not bootable, investigate whether the firmware has additional rules or paths. For systems where firmware images are available for analysis, firmware extraction and string searching can provide useful clues.

---

# Experiments
The repository can eventually contain reproducible experiments comparing different USB devices.
For example:
```
| Device | USB | Linux `RM` | EFI fallback present | UEFI bootable |
| ------ | --- | ---------- | -------------------- | ------------- |
| USB A  | Yes | 1          | Yes                  | Yes           |
| USB B  | Yes | 0          | Yes                  | No            |
| USB C  | Yes | 1          | Yes                  | Yes           |
```
Such a table would be particularly useful if multiple devices can be tested on the same notebook.
The goal is to determine whether there is a repeatable correlation between:
```
USB device characteristics
        +
OS removable classification
        +
UEFI boot-device enumeration
        +
EFI fallback discovery
```
rather than relying on a single observation.

---
## Potential A/B Tests
A useful experiment is to compare multiple USB storage devices while keeping everything else constant.
For example:
```
Same computer
Same firmware version
Same Ventoy version
Same ISO files
Same partition layout
Different USB device
```
Then record:
```
Device model
USB controller
Reported removable status
Partition table
Filesystem
EFI files
UEFI boot-menu visibility
UEFI NVRAM entries
Boot result
```
This can help distinguish:
```
USB-device-specific behavior
```
from:
```
firmware-specific behavior
```
and:
```
Ventoy or other boot configuration problems
```

---
# Limitations
There are several important limitations to this investigation.

## UEFI implementations differ
UEFI defines standards and interfaces, but firmware implementations can contain vendor-specific behavior.
A result observed on one notebook should not automatically be generalized to every motherboard or firmware version.

---

## "Removable" is not a single universal property
The following can differ:
```
Physical device
       ↓
USB device descriptor
       ↓
Storage protocol
       ↓
Operating-system classification
       ↓
UEFI classification
```

Therefore, Linux's:
```
/sys/block/sdX/removable
```
should not be treated as a universal representation of what every firmware considers "removable."

---

## BOOTX64.EFI does not guarantee bootability
Having:
```
\EFI\BOOT\BOOTX64.EFI
```
on a disk does not guarantee that every firmware will discover or execute it.

Firmware may impose additional conditions on:
- which devices are enumerated;
- which USB storage devices are considered bootable;
- filesystem support;
- partitioning;
- Secure Boot;
- boot policy;
- device classification;
- boot-order configuration.

---
## Firmware strings are not proof of behavior
A path found through firmware extraction is evidence that the string exists in the firmware image.
It is not sufficient evidence to conclude that the firmware always searches that path during boot.
Runtime testing is required to establish actual behavior.

---

# Safety and Caution
Firmware and boot configuration should be modified carefully.
Commands such as:
```
efibootmgr
```
can modify UEFI NVRAM.

Firmware flashing can permanently damage a system if the wrong image or procedure is used.
This project therefore distinguishes between:
```
Read-only investigation
```
and:
```
Configuration modification
```
Whenever possible, experiments should begin with read-only inspection.
Before modifying UEFI boot entries, record the original configuration:
```
sudo efibootmgr -v
```
and save the output.

Similarly, firmware images should be preserved before attempting any analysis or modification.

---

# Further Research
Several questions remain open for further investigation.

## USB device characteristics
Determine whether the behavior correlates with:
- USB controller;
- flash-memory controller;
- device firmware;
- USB descriptors;
- removable-media reporting;
- storage protocol;
- partition layout;
- GPT vs MBR;
- filesystem type.

---

## Firmware behavior

Determine whether particular firmware versions:
- enumerate fixed USB disks differently from removable USB disks;
- search `\EFI\BOOT\BOOTX64.EFI` only for certain device classes;
- automatically create NVRAM boot entries;
- contain vendor-specific fallback paths;
- search those paths before or after the standard removable-media path;
- behave differently between the normal boot menu and the one-time boot menu.

---

## Firmware version comparison
A particularly useful extension would be comparing multiple firmware versions from the same motherboard and checking whether the EFI path strings or boot-discovery behavior change.

---

## Cross-vendor comparison
Eventually the same methodology could be applied to firmware from:
- MSI
- ASUS
- Lenovo
- Dell
- HP
- Acer
- other UEFI implementations

This could help distinguish standardized behavior from vendor-specific behavior.

---

# Conclusion

The original problem appeared simple:
> **Ventoy was installed on a USB flash drive, but the notebook would not automatically boot from it.**
Investigating that problem reveals several layers of the modern UEFI boot process.
A USB flash drive is not necessarily treated as "removable" by every component of the system simply because it is physically removable.
Similarly, the presence of:
```
\EFI\BOOT\BOOTX64.EFI
```
does not necessarily guarantee that firmware will discover and execute that file.

UEFI NVRAM boot entries provide another mechanism for selecting EFI applications, while firmware implementations may also contain additional boot-discovery logic and vendor-specific paths.

This project attempts to document these mechanisms experimentally and to identify where behavior is standardized, where it is implementation-specific, and where further investigation is required.

The ultimate goal is not merely to find a workaround for one Ventoy USB drive, but to understand **why a seemingly valid UEFI USB boot device can be invisible or ignored by particular firmware implementations.**

