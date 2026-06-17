---
title: "Running VMs on RISC-V Hardware"
layout: post
---

I've been expecting to run KVM on RISC-V for a while now. The SpacemiT K3 board I got recently has the RISC-V H (hypervisor) extension, which means the hardware can actually run VMs properly, not just emulate everything in software. So let's make it work.

{% include image.md file="running-vms-on-riscv-hardware/IMG_0908.jpeg" description="My desk with a Spacemit K3, a Raspberry Pi 3b+, and all the cables to connect Serial, Network, and Power" %}

## Why this is interesting

RISC-V has had a hypervisor extension spec for a while, but shipping hardware that implements it is another story. The K3 does. That means guest code runs directly on the physical harts (the CPU enforces the guest/host boundary) rather than QEMU interpreting every instruction in software. x86 and ARM have had this forever. For RISC-V it's new.

As far as I know, this is the first commercially available RISC-V hardware where you can do this with a stock upstream guest kernel.

## Getting it to actually boot

First attempt:

```bash
$> qemu-system-riscv64 -accel kvm,kernel-irqchip=split \
    -machine virt,aia=aplic-imsic ...
qemu-system-riscv64: KVM AIA: failed to set number of msi
```

The vendor kernel's KVM doesn't implement the IMSIC configuration ioctls. Fine, drop IMSIC. That gets further: the kernel starts booting, but then crashes with a register dump and `mtvec=0`. It means an M-mode exception fired before the trap vector was set up. The KVM/AIA interaction just isn't stable enough with the vendor kernel yet.

What works is dropping back to APLIC without IMSIC, and using `kernel-irqchip=split` which keeps KVM in the interrupt injection path but handles the controller registers in userspace:

```bash
$> sudo qemu-system-riscv64 \
    -accel kvm,kernel-irqchip=split \
    -machine virt,aia=aplic \
    -smp 2 -m 512M -nographic \
    -kernel ./linux \
    -initrd ./initrd.gz \
    -append "root=/dev/ram rw console=ttyS0 earlycon=sbi" \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0
[    0.000000] Linux version 6.12.86+deb13-riscv64 (debian-kernel@lists.debian.org) (riscv64-linux-gnu-gcc-14 (Debian 14.2.0-19) 14.2.0, GNU ld (GNU Binutils for Debian) 2.44) #1 SMP Debian 6.12.86-1 (2026-05-08)
[    0.000000] random: crng init done
[    0.000000] Machine model: riscv-virtio,qemu
[    0.000000] SBI specification v3.0 detected
[    0.000000] SBI implementation ID=0x3 Version=0x61203
[    0.000000] SBI TIME extension detected
[    0.000000] SBI IPI extension detected
[    0.000000] SBI RFENCE extension detected
[    0.000000] SBI SRST extension detected
[    0.000000] SBI DBCN extension detected
[    0.000000] earlycon: sbi0 at I/O port 0x0 (options '')
[    0.000000] printk: legacy bootconsole [sbi0] enabled
[    0.000000] efi: UEFI not found.
[    0.000000] OF: reserved mem: Reserved memory: No reserved-memory node in the DT
[    0.000000] NUMA: Faking a node at [mem 0x0000000080000000-0x000000009fffffff]
[    0.000000] NODE_DATA(0) allocated [mem 0x9fff48c0-0x9fff6cff]
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000080000000-0x000000009fffffff]
[    0.000000]   Normal   empty
[    0.000000]   Device   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x000000009fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x000000009fffffff]
[    0.000000] SBI HSM extension detected
[    0.000000] riscv: base ISA extensions acdfimv
[    0.000000] riscv: ELF capabilities acdfimv
[    0.000000] percpu: Embedded 32 pages/cpu s90264 r8192 d32616 u131072
[    0.000000] Kernel command line: root=/dev/ram rw console=ttyS0 earlycon=sbi
[    0.000000] Dentry cache hash table entries: 65536 (order: 7, 524288 bytes, linear)
[    0.000000] Inode-cache hash table entries: 32768 (order: 6, 262144 bytes, linear)
[    0.000000] Fallback order for Node 0: 0
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 131072
[    0.000000] Policy zone: DMA32
[    0.000000] mem auto-init: stack:all(zero), heap alloc:on, heap free:off
[    0.000000] software IO TLB: SWIOTLB bounce buffer size adjusted to 0MB
[    0.000000] software IO TLB: area num 2.
[    0.000000] software IO TLB: mapped [mem 0x000000009fe6f000-0x000000009feef000] (0MB)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] ftrace: allocating 39393 entries in 154 pages
[    0.000000] ftrace: allocated 154 pages with 4 groups
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=64 to nr_cpu_ids=2.
[    0.000000]  Rude variant of Tasks RCU enabled.
[    0.000000]  Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] RCU Tasks Rude: Setting shift to 1 and lim to 1 rcu_task_cb_adjust=1 rcu_task_cpu_ids=2.
[    0.000000] RCU Tasks Trace: Setting shift to 1 and lim to 1 rcu_task_cb_adjust=1 rcu_task_cpu_ids=2.
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] riscv-intc: 64 local interrupts mapped using AIA
[    0.000000] riscv: providing IPIs using SBI IPI extension
[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
[    0.000000] clocksource: riscv_clocksource: mask: 0xffffffffffffffff max_cycles: 0x588fe9dc0, max_idle_ns: 440795202592 ns
[    0.000000] sched_clock: 64 bits at 24MHz, resolution 41ns, wraps every 4398046511097ns
[    0.000020] riscv-timer: Timer interrupt in S-mode is available via sstc extension
[    0.000282] Console: colour dummy device 80x25
[    0.000390] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=96000)
[    0.000417] pid_max: default: 32768 minimum: 301
[    0.000554] LSM: initializing lsm=lockdown,capability,landlock,yama,apparmor,tomoyo,bpf,ipe,ima,evm
[    0.000915] landlock: Up and running.
[    0.000937] Yama: disabled by default; enable with sysctl kernel.yama.*
[    0.001441] AppArmor: AppArmor initialized
[    0.001651] TOMOYO Linux initialized
[    0.002952] LSM support for eBPF active
[    0.003311] Mount-cache hash table entries: 1024 (order: 1, 8192 bytes, linear)
[    0.003348] Mountpoint-cache hash table entries: 1024 (order: 1, 8192 bytes, linear)
[    0.005512] ASID allocator using 16 bits (65536 entries)
[    0.005629] rcu: Hierarchical SRCU implementation.
[    0.005655] rcu:     Max phase no-delay instances is 1000.
[    0.005915] Timer migration: 1 hierarchy levels; 8 children per group; 1 crossnode level
[    0.006454] EFI services will not be available.
[    0.006677] smp: Bringing up secondary CPUs ...
[    0.007307] smp: Brought up 1 node, 2 CPUs
[    0.015244] node 0 deferred pages initialised in 8ms
[    0.015401] Memory: 441788K/524288K available (10724K kernel code, 5221K rwdata, 10240K rodata, 2692K init, 697K bss, 78668K reserved, 0K cma-reserved)
[    0.016087] devtmpfs: initialized
[    0.017813] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.017888] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.018071] pinctrl core: initialized pinctrl subsystem
[    0.018565] DMI not present or invalid.
[    0.018905] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.019348] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.019541] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.019584] audit: initializing netlink subsys (disabled)
[    0.019809] audit: type=2000 audit(0.000:1): state=initialized audit_enabled=0 res=1
[    0.020260] thermal_sys: Registered thermal governor 'fair_share'
[    0.020265] thermal_sys: Registered thermal governor 'bang_bang'
[    0.020293] thermal_sys: Registered thermal governor 'step_wise'
[    0.020310] thermal_sys: Registered thermal governor 'user_space'
[    0.020327] thermal_sys: Registered thermal governor 'power_allocator'
[    0.020388] cpuidle: using governor ladder
[    0.020432] cpuidle: using governor menu
[    0.044187] cpu1: Ratio of byte access time to unaligned word access is 4.94, unaligned accesses are fast
[    0.068216] cpu0: Ratio of byte access time to unaligned word access is 5.23, unaligned accesses are fast
[    0.073447] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages
[    0.073480] HugeTLB: 16380 KiB vmemmap can be freed for a 1.00 GiB page
[    0.073500] HugeTLB: registered 64.0 KiB page size, pre-allocated 0 pages
[    0.073517] HugeTLB: 0 KiB vmemmap can be freed for a 64.0 KiB page
[    0.073544] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages
[    0.073562] HugeTLB: 28 KiB vmemmap can be freed for a 2.00 MiB page
[    0.076792] ACPI: Interpreter disabled.
[    0.077022] iommu: Default domain type: Translated
[    0.077049] iommu: DMA domain TLB invalidation policy: strict mode
[    0.081213] pps_core: LinuxPPS API ver. 1 registered
[    0.081241] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.081269] PTP clock support registered
[    0.082101] NetLabel: Initializing
[    0.082129] NetLabel:  domain hash size = 128
[    0.082147] NetLabel:  protocols = UNLABELED CIPSOv4 CALIPSO
[    0.082205] NetLabel:  unlabeled traffic allowed by default
[    0.082407] vgaarb: loaded
[    0.082542] clocksource: Switched to clocksource riscv_clocksource
[    0.083204] VFS: Disk quotas dquot_6.6.0
[    0.083269] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.087194] AppArmor: AppArmor Filesystem Enabled
[    0.087265] pnp: PnP ACPI: disabled
[    0.092530] NET: Registered PF_INET protocol family
[    0.092786] IP idents hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.118262] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.118343] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.118375] TCP established hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.118415] TCP bind hash table entries: 4096 (order: 5, 131072 bytes, linear)
[    0.118464] TCP: Hash tables configured (established 4096 bind 4096)
[    0.118621] MPTCP token hash table entries: 512 (order: 2, 12288 bytes, linear)
[    0.118694] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.118737] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.118869] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.118923] NET: Registered PF_XDP protocol family
[    0.118952] PCI: CLS 0 bytes, default 64
[    0.119288] Trying to unpack rootfs image as initramfs...
[    0.123205] Initialise system trusted keyrings
[    0.123262] Key type blacklist registered
[    0.123735] workingset: timestamp_bits=44 max_order=17 bucket_order=0
[    0.123800] zbud: loaded
[    0.124052] fuse: init (API version 7.41)
[    0.124387] integrity: Platform Keyring initialized
[    0.124426] integrity: Machine keyring initialized
[    0.147083] Freeing initrd memory: 1276K
[    0.159280] Key type asymmetric registered
[    0.159320] Asymmetric key parser 'x509' registered
[    0.161659] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 247)
[    0.161844] io scheduler mq-deadline registered
[    0.166310] riscv-aplic d000000.interrupt-controller: 96 interrupts directly connected to 2 CPUs
[    0.166953] ledtrig-cpu: registered to indicate activity on CPUs
[    0.167136] pci-host-generic 30000000.pci: host bridge /soc/pci@30000000 ranges:
[    0.167182] pci-host-generic 30000000.pci:       IO 0x0003000000..0x000300ffff -> 0x0000000000
[    0.167216] pci-host-generic 30000000.pci:      MEM 0x0040000000..0x007fffffff -> 0x0040000000
[    0.167242] pci-host-generic 30000000.pci:      MEM 0x0400000000..0x07ffffffff -> 0x0400000000
[    0.167282] pci-host-generic 30000000.pci: Memory resource size exceeds max for 32 bits
[    0.167323] pci-host-generic 30000000.pci: ECAM at [mem 0x30000000-0x3fffffff] for [bus 00-ff]
[    0.167452] pci-host-generic 30000000.pci: PCI host bridge to bus 0000:00
[    0.167480] pci_bus 0000:00: root bus resource [bus 00-ff]
[    0.167500] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    0.167518] pci_bus 0000:00: root bus resource [mem 0x40000000-0x7fffffff]
[    0.167536] pci_bus 0000:00: root bus resource [mem 0x400000000-0x7ffffffff]
[    0.167604] pci 0000:00:00.0: [1b36:0008] type 00 class 0x060000 conventional PCI endpoint
[    0.167979] pci 0000:00:01.0: [1af4:1000] type 00 class 0x020000 conventional PCI endpoint
[    0.168055] pci 0000:00:01.0: BAR 0 [io  0x0000-0x001f]
[    0.168096] pci 0000:00:01.0: BAR 1 [mem 0x00000000-0x00000fff]
[    0.168184] pci 0000:00:01.0: BAR 4 [mem 0x00000000-0x00003fff 64bit pref]
[    0.168224] pci 0000:00:01.0: ROM [mem 0x00000000-0x0007ffff pref]
[    0.168810] pci 0000:00:01.0: ROM [mem 0x40000000-0x4007ffff pref]: assigned
[    0.168839] pci 0000:00:01.0: BAR 4 [mem 0x400000000-0x400003fff 64bit pref]: assigned
[    0.168892] pci 0000:00:01.0: BAR 1 [mem 0x40080000-0x40080fff]: assigned
[    0.168933] pci 0000:00:01.0: BAR 0 [io  0x0020-0x003f]: assigned
[    0.168965] pci_bus 0000:00: resource 4 [io  0x0000-0xffff]
[    0.168986] pci_bus 0000:00: resource 5 [mem 0x40000000-0x7fffffff]
[    0.169003] pci_bus 0000:00: resource 6 [mem 0x400000000-0x7ffffffff]
[    0.169526] SBI CPPC extension NOT detected!!
[    0.170125] virtio-pci 0000:00:01.0: enabling device (0000 -> 0003)
[    0.172732] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    0.174322] printk: legacy console [ttyS0] disabled
[    0.174606] 10000000.serial: ttyS0 at MMIO 0x10000000 (irq = 13, base_baud = 230400) is a 16550A
[    0.174697] printk: legacy console [ttyS0] enabled
[    0.174697] printk: legacy console [ttyS0] enabled
[    0.175380] printk: legacy bootconsole [sbi0] disabled
[    0.175380] printk: legacy bootconsole [sbi0] disabled
[    0.177069] mousedev: PS/2 mouse device common for all mice
[    0.178256] goldfish_rtc 101000.rtc: registered as rtc0
[    0.179018] goldfish_rtc 101000.rtc: setting system clock to 2026-06-17T13:47:56 UTC (1781704076)
[    0.180810] riscv-pmu-sbi: SBI PMU extension is available
[    0.181703] riscv-pmu-sbi: 22 firmware and 18 hardware counters
[    0.182691] riscv-pmu-sbi: SBI PMU snapshot detected
[    0.186837] NET: Registered PF_INET6 protocol family
[    0.190716] Segment Routing with IPv6
[    0.191326] In-situ OAM (IOAM) with IPv6
[    0.191964] mip6: Mobile IPv6
[    0.192396] NET: Registered PF_PACKET protocol family
[    0.193212] mpls_gso: MPLS GSO support
[    0.198620] registered taskstats version 1
[    0.199491] Loading compiled-in X.509 certificates
[    0.231768] Loaded X.509 cert 'Build time autogenerated kernel key: 752d7936e66eebfce23efafbd22072a3eb598c5e'
[    0.237771] Demotion targets for Node 0: null
[    0.238832] Key type .fscrypt registered
[    0.239389] Key type fscrypt-provisioning registered
[    0.243371] Key type encrypted registered
[    0.243964] AppArmor: AppArmor sha256 policy hashing enabled
[    0.244781] ima: No TPM chip found, activating TPM-bypass!
[    0.245546] ima: Allocated hash algorithm: sha256
[    0.246216] ima: No architecture policies found
[    0.246943] evm: Initialising EVM extended attributes:
[    0.247679] evm: security.selinux
[    0.248141] evm: security.SMACK64 (disabled)
[    0.248751] evm: security.SMACK64EXEC (disabled)
[    0.249385] evm: security.SMACK64TRANSMUTE (disabled)
[    0.250084] evm: security.SMACK64MMAP (disabled)
[    0.250724] evm: security.apparmor
[    0.251195] evm: security.ima
[    0.251607] evm: security.capability
[    0.252097] evm: HMAC attrs: 0x1
[    0.263985] clk: Disabling unused clocks
[    0.264586] PM: genpd: Disabling unused power domains
[    0.290006] Freeing unused kernel image (initmem) memory: 2692K
[    0.298916] Run /init as init process
udhcpc: started, v1.37.0
udhcpc: broadcasting discover
udhcpc: broadcasting select for 10.0.2.15, server 10.0.2.2
udhcpc: lease of 10.0.2.15 obtained from 10.0.2.2, lease time 86400


BusyBox v1.37.0 (Ubuntu 1:1.37.0-7ubuntu1) built-in shell (ash)
Enter 'help' for a list of built-in commands.

/bin/sh: can't access tty; job control turned off
~ #
```

Clean boot, two vCPUs, stock Debian trixie 6.12.86 kernel as the guest.

## Networking took a while

The virtio NIC showed up on the PCI bus immediately - the Debian kernel has virtio_pci built in. But `virtio_net` is a module, and the minimal busybox `initrd` had no modules at all.

I went down a detour trying to use the SpacemiT vendor kernel as the guest to get matching modules. That failed because the vendor kernel has neither `virtio_pci` nor `virtio_mmio`; it's built for real SpacemiT hardware, not QEMU's virtual platform. No transport driver, no virtio bus, no NIC.

Back to Debian: download the matching kernel package, extract `failover.ko`, `net_failover.ko`, and `virtio_net.ko`, decompress from xz, bundle into the `initrd`, load in order at boot. Then DHCP via QEMU's built-in server, with `udhcpc` handling the lease:

```bash
$> ip link set eth0 up
$> udhcpc -i eth0 -s /usr/share/udhcpc/default.script
```

And checking that it works:
```bash
$> ping -c 2 1.1.1.1
PING 1.1.1.1 (1.1.1.1): 56 data bytes
64 bytes from 1.1.1.1: seq=0 ttl=255 time=24.787 ms
64 bytes from 1.1.1.1: seq=1 ttl=255 time=45.875 ms

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 24.787/35.331/45.875 ms
```

## The full picture

A VM, running on a K3, connected to a Raspberry Pi, connected to a laptop, connected to WiFi. That ping to `1.1.1.1` goes through QEMU's NAT, out to the K3's network stack, through the Pi, through the laptop, through whatever my home router is doing, and back. Inception but for NAT layers. Cloudflare has no idea.

```bash
traceroute to 1.1.1.1 (1.1.1.1), 30 hops max, 46 byte packets
 1  10.0.2.2 (10.0.2.2)  0.201 ms  0.140 ms  0.128 ms              # QEMU's NAT on K3
 2  192.168.254.17 (192.168.254.17)  0.728 ms  0.782 ms  0.593 ms  # Raspberry Pi
 3  192.168.254.1 (192.168.254.1)  15.339 ms  16.069 ms  1.370 ms  # Laptop
 4  172.20.10.1 (172.20.10.1)  12.947 ms  41.817 ms  8.518 ms      # Router
 5  [...]
```

## What's still rough

The vendor KVM doesn't support full AIA with IMSIC yet, so MSI-native devices won't work in the guest. `kernel-irqchip=on` isn't stable. The guest has to be an upstream or custom-built kernel since the SpacemiT kernel lacks virtio transport support for QEMU's virtual platform.

None of that is a hardware limitation. The H extension works. The gaps are in the vendor kernel's KVM implementation, and those close over time as upstream RISC-V KVM support matures and vendors track it. I could also check that two QEMU instance using KVM can run on the same machine. I haven't checked a nested VM scenario yet.

Next step is a real root filesystem instead of a busybox shell. But that's for another post.
