-----------#0 prepare---------------------------------------------------------------
$ sudo apt install automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev pkg-config libpixman-1-dev libglib2.0-dev ninja-build libncurses5-dev autoconf libncursesw5-dev
$ mkdir riscv64-linux
$ cd riscv64-linux
$ find . -name "*" | xargs dos2unix
$ find . -name "configure" | xargs dos2unix -f
$ chmod -R +x *
-----------#1 install riscv-gnu-toolchain-------------------------------------------
$ git clone git://github.com/riscv/riscv-gnu-toolchain
$ cd riscv-gnu-toolchain
$ git rm qemu
$ git submodule update --init --recursive
$ ./configure --prefix=/opt/riscv64 
$ sudo make linux -j $(nproc)
$ export PATH="$PATH:/opt/riscv64/bin"
-----------#1.1 check riscv-gnu-toolchain-------------------------------------------
$ riscv64-unknown-linux-gnu-gcc -v
  >
    Using built-in specs.
    COLLECT_GCC=riscv64-unknown-linux-gnu-gcc
    COLLECT_LTO_WRAPPER=/opt/riscv64/libexec/gcc/riscv64-unknown-linux-gnu/11.1.0/lto-wrapper
    Target: riscv64-unknown-linux-gnu
    Configured with: /home/lazyoung/code/riscv64-linux/riscv-gnu-toolchain/riscv-gcc/configure --target=riscv64-unknown-linux-gnu --prefix=/opt/riscv64 
    --with-sysroot=/opt/riscv64/sysroot --with-system-zlib --enable-shared --enable-tls --enable-languages=c,c++,fortran --disable-libmudflap --disable-libssp
    --disable-libquadmath --disable-libsanitizer --disable-nls --disable-bootstrap --src=.././riscv-gcc --disable-multilib --with-abi=lp64d --with-arch=rv64imafdc 
    --with-tune=rocket 'CFLAGS_FOR_TARGET=-O2   -mcmodel=medlow' 'CXXFLAGS_FOR_TARGET=-O2   -mcmodel=medlow'
    Thread model: posix
    Supported LTO compression algorithms: zlib
    gcc version 11.1.0 (GCC)
-----------#2 qemu-----------------------------------------------------------------
$ cd ..
$ git clone https://github.com/qemu/qemu
$ cd qemu/
$ git submodule update --init --recursive
$ git checkout v6.1.0
$ ./configure --target-list=riscv64-softmmu
$ make -j $(nproc)
$ sudo make install
-----------#2.1 check qemu----------------------------------------------------------
$ qemu-system-riscv64 --version
  >
  QEMU emulator version 6.1.0 (v6.1.0)
  Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
-----------#3 linux-----------------------------------------------------------------
$ cd ..
$ git clone git://github.com/torvalds/linux
$ cd linux/
## rename aux.c/aux.h in windows and cp to linux
$ cp rename_i2c_aux.c drivers/gpu/drm/nouveau/nvkm/subdev/i2c/aux.c
$ cp rename_i2c_aux.h drivers/gpu/drm/nouveau/nvkm/subdev/i2c/aux.h
$ cp rename_arc_aux.h include/soc/arc/aux.h
$ git checkout v5.9
$ make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
$ make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j $(nproc)
-----------#3.1 check linux---------------------------------------------------------
$ls arch/riscv/boot/
  >
  dts  Image  Image.gz  install.sh  loader.lds.S  loader.S  Makefile
-----------#4 busybox---------------------------------------------------------------
$ cd ..
$ git clone https://git.busybox.net/busybox
$ cd busybox
$ CROSS_COMPILE=riscv64-unknown-linux-gnu- make menuconfig
  >
  "Settings" -> "Build Options" -> "Build static binary (no shared libs)"
$ CROSS_COMPILE=riscv64-unknown-linux-gnu- make -j $(nproc)
$ CROSS_COMPILE=riscv64-unknown-linux-gnu- make install
-----------#4.1 rootfs.img by busybox-----------------------------------------------
$ ls _install
  >
  bin  linuxrc  sbin  usr
$ cd ..
$ qemu-img create rootfs.img  2g
$ mkfs.ext4 rootfs.img
$ mkdir rootfs
$ sudo mount -o loop rootfs.img  rootfs
$ cd rootfs
$ sudo cp -r ../busybox/_install/* .
$ sudo mkdir proc sys dev etc etc/init.d
$ cd etc/init.d/
$ sudo touch rcS
$ sudo vi rcS
  >
  #!/bin/sh
  mount -t proc none /proc
  mount -t sysfs none /sys
  /sbin/mdev -s
$ sudo chmod +x rcS
$ cd ../../../
$ sudo umount rootfs
$ ll | grep rootfs
  >
  drwxrwxr-x  2 lazyoung lazyoung       4096 9月  25 07:41 rootfs/
  -rw-r--r--  1 lazyoung lazyoung 2147483648 9月  25 13:07 rootfs.img
-----------#5 run---------------------------------------------------------------
$ qemu-system-riscv64 -M virt -m 256M -nographic -kernel linux/arch/riscv/boot/Image -drive file=rootfs.img,format=raw,id=hd0  -device virtio-blk-device,drive=hd0 -append "root=/dev/vda rw console=ttyS0"
-----------#5.1 bootup----------------------------------------------------------
  >
  OpenSBI v0.9
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Platform Features         : timer,mfdeleg
Platform HART Count       : 1
Firmware Base             : 0x80000000
Firmware Size             : 100 KB
Runtime SBI Version       : 0.2

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x0000000080000000-0x000000008001ffff ()
Domain0 Region01          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
Domain0 Next Address      : 0x0000000080200000
Domain0 Next Arg1         : 0x000000008f000000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART ISA             : rv64imafdcsu
Boot HART Features        : scounteren,mcounteren,time
Boot HART PMP Count       : 16
Boot HART PMP Granularity : 4
Boot HART PMP Address Bits: 54
Boot HART MHPM Count      : 0
Boot HART MHPM Count      : 0
Boot HART MIDELEG         : 0x0000000000000222
Boot HART MEDELEG         : 0x000000000000b109
[    0.000000] OF: fdt: Ignoring memory range 0x80000000 - 0x80200000
[    0.000000] Linux version 5.9.0 (lazyoung@lazyoung) (riscv64-linux-gcc.br_real (Buildroot 2020.08-14-ge5a2a90) 10.2.0, GNU ld (GNU Binutils) 2.34) #3 SMP Sat Sep 25 07:24:39 CST 2021
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000080200000-0x000000008fffffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080200000-0x000000008fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080200000-0x000000008fffffff]
[    0.000000] software IO TLB: mapped [mem 0x8b000000-0x8f000000] (64MB)
[    0.000000] SBI specification v0.2 detected
[    0.000000] SBI implementation ID=0x1 Version=0x9
[    0.000000] SBI v0.2 TIME extension detected
[    0.000000] SBI v0.2 IPI extension detected
[    0.000000] SBI v0.2 RFENCE extension detected
[    0.000000] SBI v0.2 HSM extension detected
[    0.000000] riscv: ISA extensions acdfimsu
[    0.000000] riscv: ELF capabilities acdfim
[    0.000000] percpu: Embedded 17 pages/cpu s32040 r8192 d29400 u69632
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 64135
[    0.000000] Kernel command line: root=/dev/vda rw console=ttyS0
[    0.000000] Dentry cache hash table entries: 32768 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.000000] Sorting __ex_table...
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 172940K/260096K available (6899K kernel code, 3742K rwdata, 4096K rodata, 183K init, 318K bss, 87156K reserved, 0K cma-reserved)
[    0.000000] Virtual kernel memory layout:
[    0.000000]       fixmap : 0xffffffcefee00000 - 0xffffffceff000000   (2048 kB)
[    0.000000]       pci io : 0xffffffceff000000 - 0xffffffcf00000000   (  16 MB)
[    0.000000]      vmemmap : 0xffffffcf00000000 - 0xffffffcfffffffff   (4095 MB)
[    0.000000]      vmalloc : 0xffffffd000000000 - 0xffffffdfffffffff   (65535 MB)
[    0.000000]       lowmem : 0xffffffe000000000 - 0xffffffe00fe00000   ( 254 MB)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=8 to nr_cpu_ids=1.
[    0.000000] rcu: 	RCU debug extended QS entry/exit.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=1
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] riscv-intc: 64 local interrupts mapped
[    0.000000] plic: plic@c000000: mapped 53 interrupts with 1 handlers for 2 contexts.
[    0.000000] random: get_random_bytes called from start_kernel+0x312/0x482 with crng_init=0
[    0.000000] riscv_timer_init_dt: Registering clocksource cpuid [0] hartid [0]
[    0.000000] clocksource: riscv_clocksource: mask: 0xffffffffffffffff max_cycles: 0x24e6a1710, max_idle_ns: 440795202120 ns
[    0.000245] sched_clock: 64 bits at 10MHz, resolution 100ns, wraps every 4398046511100ns
[    0.005427] Console: colour dummy device 80x25
[    0.012665] Calibrating delay loop (skipped), value calculated using timer frequency.. 20.00 BogoMIPS (lpj=40000)
[    0.013203] pid_max: default: 32768 minimum: 301
[    0.015382] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.015443] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.056034] rcu: Hierarchical SRCU implementation.
[    0.061658] smp: Bringing up secondary CPUs ...
[    0.061817] smp: Brought up 1 node, 1 CPU
[    0.076502] devtmpfs: initialized
[    0.087599] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.087981] futex hash table entries: 256 (order: 2, 16384 bytes, linear)
[    0.096214] NET: Registered protocol family 16
[    0.193137] vgaarb: loaded
[    0.195090] SCSI subsystem initialized
[    0.198677] usbcore: registered new interface driver usbfs
[    0.199170] usbcore: registered new interface driver hub
[    0.199467] usbcore: registered new device driver usb
[    0.215652] clocksource: Switched to clocksource riscv_clocksource
[    0.242635] NET: Registered protocol family 2
[    0.249784] tcp_listen_portaddr_hash hash table entries: 128 (order: 0, 5120 bytes, linear)
[    0.249964] TCP established hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.250296] TCP bind hash table entries: 2048 (order: 4, 65536 bytes, linear)
[    0.250580] TCP: Hash tables configured (established 2048 bind 2048)
[    0.252246] UDP hash table entries: 256 (order: 2, 24576 bytes, linear)
[    0.252747] UDP-Lite hash table entries: 256 (order: 2, 24576 bytes, linear)
[    0.255140] NET: Registered protocol family 1
[    0.259477] RPC: Registered named UNIX socket transport module.
[    0.259560] RPC: Registered udp transport module.
[    0.259587] RPC: Registered tcp transport module.
[    0.259614] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.259791] PCI: CLS 0 bytes, default 64
[    0.268951] workingset: timestamp_bits=62 max_order=16 bucket_order=0
[    0.294558] NFS: Registering the id_resolver key type
[    0.295999] Key type id_resolver registered
[    0.296069] Key type id_legacy registered
[    0.296669] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.297719] 9p: Installing v9fs 9p2000 file system support
[    0.300216] NET: Registered protocol family 38
[    0.300672] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 252)
[    0.300867] io scheduler mq-deadline registered
[    0.300985] io scheduler kyber registered
[    0.311181] pci-host-generic 30000000.pci: host bridge /soc/pci@30000000 ranges:
[    0.312576] pci-host-generic 30000000.pci:       IO 0x0003000000..0x000300ffff -> 0x0000000000
[    0.313301] pci-host-generic 30000000.pci:      MEM 0x0040000000..0x007fffffff -> 0x0040000000
[    0.313400] pci-host-generic 30000000.pci:      MEM 0x0400000000..0x07ffffffff -> 0x0400000000
[    0.316719] pci-host-generic 30000000.pci: ECAM at [mem 0x30000000-0x3fffffff] for [bus 00-ff]
[    0.318061] pci-host-generic 30000000.pci: PCI host bridge to bus 0000:00
[    0.318362] pci_bus 0000:00: root bus resource [bus 00-ff]
[    0.318560] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    0.318591] pci_bus 0000:00: root bus resource [mem 0x40000000-0x7fffffff]
[    0.318618] pci_bus 0000:00: root bus resource [mem 0x400000000-0x7ffffffff]
[    0.320522] pci 0000:00:00.0: [1b36:0008] type 00 class 0x060000
[    0.446264] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    0.456965] printk: console [ttyS0] disabled
[    0.458440] 10000000.uart: ttyS0 at MMIO 0x10000000 (irq = 2, base_baud = 230400) is a 16550A
[    0.494231] printk: console [ttyS0] enabled
[    0.498069] [drm] radeon kernel modesetting enabled.
[    0.527804] loop: module loaded
[    0.540193] virtio_blk virtio0: [vda] 4194304 512-byte logical blocks (2.15 GB/2.00 GiB)
[    0.540764] vda: detected capacity change from 0 to 2147483648
[    0.575839] libphy: Fixed MDIO Bus: probed
[    0.578381] e1000e: Intel(R) PRO/1000 Network Driver
[    0.578732] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    0.579720] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    0.580115] ehci-pci: EHCI PCI platform driver
[    0.580659] ehci-platform: EHCI generic platform driver
[    0.581159] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    0.581650] ohci-pci: OHCI PCI platform driver
[    0.582142] ohci-platform: OHCI generic platform driver
[    0.584025] usbcore: registered new interface driver uas
[    0.584599] usbcore: registered new interface driver usb-storage
[    0.586211] mousedev: PS/2 mouse device common for all mice
[    0.590777] goldfish_rtc 101000.rtc: registered as rtc0
[    0.592229] goldfish_rtc 101000.rtc: setting system clock to 2021-09-25T03:08:58 UTC (1632539338)
[    0.596699] syscon-poweroff soc:poweroff: pm_power_off already claimed (____ptrval____) sbi_shutdown
[    0.597327] syscon-poweroff: probe of soc:poweroff failed with error -16
[    0.600150] usbcore: registered new interface driver usbhid
[    0.600495] usbhid: USB HID core driver
[    0.603670] NET: Registered protocol family 10
[    0.613457] Segment Routing with IPv6
[    0.614335] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    0.618963] NET: Registered protocol family 17
[    0.621501] 9pnet: Installing 9P2000 support
[    0.622202] Key type dns_resolver registered
[    0.623509] debug_vm_pgtable: [debug_vm_pgtable         ]: Validating architecture page table helpers
[    0.685573] EXT4-fs (vda): mounted filesystem with ordered data mode. Opts: (null)
[    0.686337] VFS: Mounted root (ext4 filesystem) on device 254:0.
[    0.690488] devtmpfs: mounted
[    0.720725] Freeing unused kernel memory: 180K
[    0.722816] Run /sbin/init as init process

Please press Enter to activate this console.
  >
  / # 
  
