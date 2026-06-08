---
title: "The Boot Chain of a RISC-V Board: From Silicon to Ubuntu 26.04"
layout: post
---

When you press the power button on the SpacemiT K3 Pico-ITX, Ubuntu 26.04 appears on the serial console about 30 seconds later. Between those two events, five distinct software layers run in sequence, each one handing off to the next. Understanding what each layer does - and why it exists - matters the moment something goes wrong, or the moment you want to put a different OS on the board.

This post walks through the complete boot chain of the K3 running Ubuntu 26.04, from the mask ROM in silicon to the GRUB menu. It is specific to this board, but the architecture generalises to most modern RISC-V single-board computers.

## The hardware

The SpacemiT K3 Pico-ITX is a compact RISC-V board built around the SpacemiT K3 SoC: 8 × X100 RVA23 cores, 16 GB LPDDR5, 128 GB UFS storage. Two persistent storage media matter for boot: a **Winbond W25Q64 NOR flash** (8 MB) soldered on the PCB, and the UFS drive. The NOR flash holds the firmware stack; the UFS drive holds the OS.

## Layer 0: BROM - the silicon anchor

Every boot starts in the **Boot ROM**, a small blob of code burned into the SoC at manufacture that cannot be modified. The BROM runs first, in M-mode (the highest RISC-V privilege level), before any RAM is initialised. Its only job is to locate the first-stage bootloader on NOR flash and hand off to it. Because it lives in silicon, the BROM is also the **unbrick path**: holding the FDL button while pressing RST forces the SoC back into BROM download mode, where it speaks a USB protocol and accepts new firmware regardless of what state NOR flash is in.

## Layer 1: U-Boot SPL - initialising the machine

The BROM loads **U-Boot SPL** (Secondary Program Loader) from NOR flash into the on-chip SRAM. At this point DRAM does not yet exist - the LPDDR5 has not been initialised.

SPL is minimal by design. It initialises DRAM, then loads the next stage (OpenSBI) from NOR flash into that freshly-initialised DRAM, and jumps to it. The binary in the flashing layout is called `FSBL.bin` (First Stage Boot Loader) which is another name for the same thing. The NOR partition is called `fsbl`.

## Layer 2: OpenSBI - the M-mode runtime

**OpenSBI** is the open-source implementation of the RISC-V Supervisor Binary Interface. This layer is conceptually different from all the others: it does not just run and hand off. It stays resident in M-mode for the entire lifetime of the system.

RISC-V splits privilege into three levels: M-mode (machine), S-mode (supervisor), and U-mode (user). The OS kernel runs in S-mode. Anything that requires M-mode (hardware timers, inter-processor interrupts, system power control) has to cross the privilege boundary via an `ecall` instruction. OpenSBI handles those calls. It is roughly analogous to PSCI on ARM or SMM on x86: a small M-mode monitor that stays in the background fielding requests from above.

OpenSBI is compiled in `FW_DYNAMIC` mode here, meaning it receives a pointer to the next stage from SPL at the moment it is invoked, and jumps there once its own initialisation is done.

## Layer 3: EDK2/UEFI - the firmware interface

After OpenSBI, execution passes to **EDK2** - the open-source UEFI firmware implementation, the same codebase that provides the virtual UEFI in QEMU (`OVMF`). SpacemiT ships a vendor port of it (`SPACEMIT v0.0.1`), stored in the NOR partition confusingly named `uboot` (a legacy name from the Bianbu installation) and flashed as `edk2.itb`.

This is where the boot process gains the full UEFI environment: EFI variables stored in NVRAM, EFI System Partition scanning, EFI protocol tables, and the ability to run EFI binaries. EDK2 finds the ESP on the UFS drive (`/dev/sda7`, 105 MB, vfat) and looks for the GRUB EFI binary at `EFI/ubuntu/grubriscv64.efi`.

The reason EDK2 replaced full U-Boot in this slot is instructive. The board originally shipped with Bianbu, which used U-Boot proper in NOR. U-Boot has a partial UEFI implementation built in, and it exposes a `bootefi` command to run EFI binaries from that environment. When the Ubuntu image was written to the UFS drive, the Bianbu U-Boot (Apr 29 2026 build) crashed with `Unhandled exception: Load access fault` every time it tried to run Ubuntu's GRUB EFI binary. U-Boot's UEFI subsystem is a compatibility shim, not a first-class implementation, and it had a bug on this specific firmware version.

EDK2 is a proper UEFI firmware. Once it replaced U-Boot in the NOR `uboot` partition, GRUB loaded without issue.

## Layer 4: GRUB - the OS selector

**GRUB** (`grubriscv64.efi`) runs as a UEFI application. It reads the UEFI environment from EDK2 - filesystem access via EFI protocols, memory map, device handles - and presents the boot menu. On this board it runs in serial-only mode (the `error: no suitable video mode found` message at the GRUB stage is normal and harmless; the display initialises correctly once the kernel is up).

GRUB loads the kernel (`/boot/vmlinuz-6.18.3-5-spacemit-generic`), the initrd, and the DTB from the writable partition (`/dev/sda9`, ext4), passes the kernel command line, and calls the EFI stub entry point.

## Layer 5: Linux kernel + Ubuntu

The kernel takes over in S-mode, with OpenSBI still resident in M-mode fielding privileged calls. The DTB (`k3-pico-itx.dtb`) describes the hardware to the kernel. The initrd runs early userspace; systemd starts; Ubuntu 26.04 is up.

## Why this particular stack

Two choices here are non-obvious:

**Why EDK2 and not U-Boot directly?** Ubuntu's RISC-V port is designed around UEFI. GRUB is compiled as an EFI binary; `grub-install` writes EFI variables; Secure Boot infrastructure sits on top of UEFI. A full UEFI implementation is required, not a compatibility shim.

**Why the vendor kernel?** The K3 DTS was merged into mainline Linux in 7.0, but only as a skeleton - CPUs and interrupt controller, nothing else. Peripheral drivers (Ethernet, I2C, GPIO, PMIC) are queued for 7.1. Ubuntu 26.04 ships a 6.18 kernel, older than 7.0, which has no K3 DTS at all. The SpacemiT vendor kernel (`6.18.3-5-spacemit-generic`) carries all the out-of-tree K3 drivers and is what makes the hardware actually work. This is the kernel in the SpacemiT Ubuntu 26.04 image, and it is the kernel this boot chain runs.

## The NOR flash layout at a glance

| Partition | Content |
|-----------|---------|
| `bootinfo` | Metadata (partition offsets), read by BROM |
| `fsbl` | U-Boot SPL (FSBL.bin) |
| `env` | U-Boot environment variables (legacy, unused in EDK2 path) |
| `opensbi` | OpenSBI FW_DYNAMIC (fw_dynamic.itb) |
| `uboot` | EDK2/UEFI firmware (edk2.itb) |
| `esos` | Unknown - SpacemiT-specific, purpose undocumented publicly |

The UFS drive holds everything else: ESP with GRUB and DTBs on `sda7`, and the Ubuntu rootfs plus vendor kernel on `sda9`.
