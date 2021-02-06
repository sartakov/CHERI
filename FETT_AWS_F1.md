# This is a step-by-step guide to run CheriBSD on top of FPGA using an AWS F1 instance. 

This guide sumarises several other guides [1](https://github.com/CTSRD-CHERI/FETT/wiki/AWS-F1-Connectal%E2%80%91Based-Platform),[2](https://github.com/acceleratedtech/ssith-aws-fpga)
You start with a fresh F1 instance with  18.04 Ubuntu inside. You need to increase the default storage size from 8 up to 30 or 50 GiB. 

Log into the VM , update it and reboot. 
```
sudo apt update
sudo apt ugprade
<reboot>
```

## Build CheriBSD with FETT kernel and BBL
```
sudo apt install autoconf automake libtool pkg-config clang bison cmake ninja-build samba flex texinfo libglib2.0-dev libpixman-1-dev libarchive-dev libarchive-tools libbz2-dev libattr1-dev libcap-ng-dev

git clone https://github.com/CTSRD-CHERI/cheribuild.git
cd cheribuild/
./cheribuild.py disk-image-riscv64-hybrid --cheribsd-riscv64-hybrid/build-fett-kernels -d
./cheribuild.py bbl-fett-baremetal-riscv64-purecap
cd ..
```

## Build AWS FPGA utility to manipulate the FPGA board

```
git clone https://github.com/aws/aws-fpga.git
. ./sdk_setup.sh
sudo chmod u+s /usr/local/bin/fpga-local-cmd
```

Load RISCV CHERI image into FPGA
```
sudo fpga-load-local-image -S 0 -I agfi-026d853003d6c433a
```

## Build and setup ssith-aws-fpga

```
sudo apt-get install cmake device-tree-compiler build-essential libssl-dev libcurl4-openssl-dev libsdl-dev libelf-dev
cd ssith-aws-fpga/
git submodule update --init --recursive
mkdir build
cd build
cmake -DFPGA=1 ..
make
cd ..
dtc -I dts -O dtb -o build/devicetree.dtb src/dts/devicetree-cheri.dts
cd ..
```

## build and run connectal driver
```
sudo add-apt-repository -y ppa:jamey-hicks/connectal
sudo apt-get update
sudo apt-get install connectal
sudo modprobe portalmem
sudo modprobe pcieportal
```

So at this point, you can run the ready-to-use system:

```
sudo ./ssith-aws-fpga/build/ssith_aws_fpga --dtb ./ssith-aws-fpga/build/devicetree.dtb --elf ./cheri/output/sdk/bbl-fett/riscv64-purecap/bbl --elf ./cheri/output/rootfs-riscv64-hybrid/boot/kernel.CHERI-FETT/kernel --debug-log --block ./cheri/output/cheribsd-riscv64-hybrid.img
```

It starts almost instant, but need time to boot. The expected output when everything is ok: 
```
romBuffer=7f405db08010
ethernet device 0x565370dae850 virtio net device 0x565370daecf0 at addr 40000000
virtio entropy device 0x565370daeff0 at addr 40001000
block device ./cheri/output/cheribsd-riscv64-hybrid.img (0x565370db1300)
virtio block device 0x565370db1360 at addr 40002000
Copied 2340 bytes 2340 bytes total readsize 4096
Read 2340 bytes from ./ssith-aws-fpga/build/devicetree.dtb
asserting haltreq
dmi state machine status 0
loadElf: ./cheri/output/sdk/bbl-fett/riscv64-purecap/bbl is a 64-bit ELF file
loadElf: entry point c0000000
Section .text           : addr         c0000000 to addr         c00003c0; size 0x     3c0 (= 960) bytes
Section .text           : addr         c00003c0 to addr         c000571e; size 0x    535e (= 21342) bytes
Section .rodata         : addr         c0005720 to addr         c0005c8a; size 0x     56a (= 1386) bytes
Section __cap_relocs    : addr         c0005c90 to addr         c0007e00; size 0x    2170 (= 8560) bytes
Section .captable       : addr         c0007e00 to addr         c0008960; size 0x     b60 (= 2912) bytes
Section .htif           : addr         c0009000 to addr         c0009010; size 0x      10 (= 16) bytes
Section .data           : addr         c000a000 to addr         c000a2df; size 0x     2df (= 735) bytes
Section .sdata          : addr         c000a2e0 to addr         c000a2e4; size 0x       4 (= 4) bytes
Section .bss            : addr         c000b000 to addr         c0014024; size 0x    9024 (= 36900) bytes
Section .debug_loc      : addr                0 to addr             7352; size 0x    7352 (= 29522) bytes
Section .debug_abbrev   : addr                0 to addr             13b6; size 0x    13b6 (= 5046) bytes
Section .debug_info     : addr                0 to addr             633e; size 0x    633e (= 25406) bytes
Section .debug_str      : addr                0 to addr             1212; size 0x    1212 (= 4626) bytes
Section .riscv.attributes: addr                0 to addr               3f; size 0x      3f (= 63) bytes
Section .debug_frame    : addr                0 to addr              d58; size 0x     d58 (= 3416) bytes
Section .debug_line     : addr                0 to addr             3397; size 0x    3397 (= 13207) bytes
Section .debug_ranges   : addr                0 to addr              d00; size 0x     d00 (= 3328) bytes
Section .debug_aranges  : addr                0 to addr               60; size 0x      60 (= 96) bytes
Section .symtab         : addr                0 to addr             aa58; size 0x    aa58 (= 43608) bytes
Section .shstrtab       : addr                0 to addr               d0; size 0x      d0 (= 208) bytes
Section .strtab         : addr                0 to addr             1570; size 0x    1570 (= 5488) bytes
phdr 0 type 1 flags 5 va c0000000 pa c0000000
phdr 1 type 1 flags 4 va c0005720 pa c0005720
phdr 2 type 1 flags 6 va c0007e00 pa c0007e00
phdr 3 type 1 flags 6 va c0009000 pa c0009000
phdr 4 type 6474e552 flags 4 va c0007e00 pa c0007e00
phdr 5 type 6474e551 flags 6 va 00000000 pa 00000000
elf_entry=c0000000
loadElf: ./cheri/output/rootfs-riscv64-hybrid/boot/kernel.CHERI-FETT/kernel is a 64-bit ELF file
loadElf: entry point ffffffc00000002e
Section .text           : addr ffffffc000000000 to addr ffffffc000553128; size 0x  553128 (= 5583144) bytes
Section .rodata         : addr ffffffc000553128 to addr ffffffc000560afc; size 0x    d9d4 (= 55764) bytes
Section .rodata.str1.1  : addr ffffffc000560afc to addr ffffffc0005d7072; size 0x   76576 (= 484726) bytes
Section .rodata.cst32   : addr ffffffc0005d7078 to addr ffffffc0005d71d8; size 0x     160 (= 352) bytes
Section .rodata.cst16   : addr ffffffc0005d71d8 to addr ffffffc0005d7258; size 0x      80 (= 128) bytes
Section .rodata.str4.4  : addr ffffffc0005d7258 to addr ffffffc0005d7270; size 0x      18 (= 24) bytes
Section .rodata.cst8    : addr ffffffc0005d7270 to addr ffffffc0005d7288; size 0x      18 (= 24) bytes
Section .eh_frame       : addr ffffffc0005d7288 to addr ffffffc0005d7694; size 0x     40c (= 1036) bytes
Section .interp         : addr ffffffc0005d7694 to addr ffffffc0005d76a1; size 0x       d (= 13) bytes
Section .hash           : addr ffffffc0005d76a4 to addr ffffffc0005ec4d4; size 0x   14e30 (= 85552) bytes
Section .dynsym         : addr ffffffc0005ec4d8 to addr ffffffc00062af50; size 0x   3ea78 (= 256632) bytes
Section .dynstr         : addr ffffffc00062af50 to addr ffffffc000657a2d; size 0x   2cadd (= 183005) bytes
Section .gnu.hash       : addr ffffffc000657a30 to addr ffffffc000668af8; size 0x   110c8 (= 69832) bytes
Section .rela.dyn       : addr ffffffc000668af8 to addr ffffffc000668b70; size 0x      78 (= 120) bytes
Section .note.gnu.build-id: addr ffffffc000668b70 to addr ffffffc000668b94; size 0x      24 (= 36) bytes
Section .rela.plt       : addr ffffffc000668b98 to addr ffffffc000668bb0; size 0x      18 (= 24) bytes
Section .plt            : addr ffffffc000668bb0 to addr ffffffc000668be0; size 0x      30 (= 48) bytes
Section .data           : addr ffffffc00066a000 to addr ffffffc0006d5acb; size 0x   6bacb (= 441035) bytes
Section set_sysctl_set  : addr ffffffc0006d5ad0 to addr ffffffc0006d92d8; size 0x    3808 (= 14344) bytes
Section set_sysinit_set : addr ffffffc0006d92d8 to addr ffffffc0006db1c8; size 0x    1ef0 (= 7920) bytes
Section set_sysuninit_set: addr ffffffc0006db1c8 to addr ffffffc0006dc450; size 0x    1288 (= 4744) bytes
Section set_kdb_dbbe_set: addr ffffffc0006dc450 to addr ffffffc0006dc460; size 0x      10 (= 16) bytes
Section set_modmetadata_set: addr ffffffc0006dc460 to addr ffffffc0006dd0e0; size 0x     c80 (= 3200) bytes
Section set_ofw_set     : addr ffffffc0006dd0e0 to addr ffffffc0006dd0e8; size 0x       8 (= 8) bytes
Section .data.read_frequently: addr ffffffc0006dd0e8 to addr ffffffc0006dd111; size 0x      29 (= 41) bytes
Section .data.read_mostly: addr ffffffc0006dd118 to addr ffffffc0006de498; size 0x    1380 (= 4992) bytes
Section set_uart_fdt_class_and_device_set: addr ffffffc0006de498 to addr ffffffc0006de4b0; size 0x      18 (= 24) bytes
Section set_cons_set    : addr ffffffc0006de4b0 to addr ffffffc0006de4c8; size 0x      18 (= 24) bytes
Section set_pcpu        : addr ffffffc0006de500 to addr ffffffc0006e0180; size 0x    1c80 (= 7296) bytes
Section .data.exclusive_cache_line: addr ffffffc0006e0180 to addr ffffffc0006e66c8; size 0x    6548 (= 25928) bytes
Section set_vnet        : addr ffffffc0006e66c8 to addr ffffffc00070f768; size 0x   290a0 (= 168096) bytes
Section set_compressors : addr ffffffc00070f768 to addr ffffffc00070f770; size 0x       8 (= 8) bytes
Section .got            : addr ffffffc00070f770 to addr ffffffc0007125b0; size 0x    2e40 (= 11840) bytes
Section .dynamic        : addr ffffffc0007125b0 to addr ffffffc0007126b0; size 0x     100 (= 256) bytes
Section .data.rel.ro    : addr ffffffc0007126b0 to addr ffffffc00071ba98; size 0x    93e8 (= 37864) bytes
Section .sdata          : addr ffffffc00071ba98 to addr ffffffc00071c46c; size 0x     9d4 (= 2516) bytes
Section .sbss           : addr ffffffc00071c470 to addr ffffffc00071e88e; size 0x    241e (= 9246) bytes
Section .bss            : addr ffffffc00071e8c0 to addr ffffffc00083f098; size 0x  1207d8 (= 1181656) bytes
Section .comment        : addr                0 to addr              124; size 0x     124 (= 292) bytes
Section .riscv.attributes: addr                0 to addr               3f; size 0x      3f (= 63) bytes
Section .symtab         : addr                0 to addr            9dcf8; size 0x   9dcf8 (= 646392) bytes
Section .shstrtab       : addr                0 to addr              216; size 0x     216 (= 534) bytes
Section .strtab         : addr                0 to addr            7ce4e; size 0x   7ce4e (= 511566) bytes
Section .gnu_debuglink  : addr                0 to addr               14; size 0x      14 (= 20) bytes
phdr 0 type 3 flags 4 va ffffffc0005d7694 pa c07d7694
phdr 1 type 1 flags 5 va ffffffc000000000 pa c0200000
phdr 2 type 1 flags 4 va ffffffc000553128 pa c0753128
phdr 3 type 1 flags 5 va ffffffc000668bb0 pa c0868bb0
phdr 4 type 1 flags 6 va ffffffc00066a000 pa c086a000
phdr 5 type 1 flags 6 va ffffffc00071ba98 pa c091ba98
phdr 6 type 2 flags 6 va ffffffc0007125b0 pa c09125b0
phdr 7 type 6474e552 flags 4 va ffffffc00070f770 pa c090f770
phdr 8 type 6474e551 flags 6 va 00000000 pa 00000000
phdr 9 type 4 flags 4 va ffffffc000668b70 pa c0868b70
setting pc val c0000000
reading pc val c0000000
bbl loader
---<<BOOT>>---
KDB: debugger backends: ddb
KDB: current backend: ddb
Physical memory chunk(s):
  0xc0000000 - 0xfeffffff,  1008 MB ( 258048 pages)
Excluded memory regions:
  0xa0000000 - 0xc01fffff,   514 MB ( 131584 pages) NoAlloc NoDump
  0xc0200000 - 0xc0a5afff,     8 MB (   2139 pages) NoAlloc 
Found 1 CPUs in the device tree
Copyright (c) 1992-2020 The FreeBSD Project.
Copyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994
	The Regents of the University of California. All rights reserved.
FreeBSD is a registered trademark of The FreeBSD Foundation.
FreeBSD 13.0-CURRENT #0 ea692111dc-c1(master)-dirty: Sat Feb  6 13:16:36 UTC 2021
    ubuntu@ip-172-31-39-122:/home/ubuntu/cheri/build/cheribsd-riscv64-hybrid-build/home/ubuntu/cheri/cheribsd/riscv.riscv64/sys/CHERI-FETT riscv
clang version 11.0.0 (https://github.com/CTSRD-CHERI/llvm-project.git ef88c21e79a3b27da20b9e80ad651339439cb988)
WARNING: WITNESS option enabled, expect reduced performance.
CHERI hybrid kernel.
Preloaded elf kernel "kernel" at 0xffffffc000839360.
SBI: Unknown (Legacy) Implementation
SBI Specification Version: 0.1
CPU(0): Unknown Implementer Unknown Processor
real memory  = 1056964608 (1008 MB)
Physical memory chunk(s):
0x00000000c0a5b000 - 0x00000000fd4cafff, 1017577472 bytes (248432 pages)
avail memory = 1012531200 (965 MB)
No static device mappings.
random: no preloaded entropy cache
arc4random: WARNING: initial seeding bypassed the cryptographic random device because it was not yet seeded and the knob 'bypass_before_seeding' was enabled.
VIMAGE (virtualized network stack) enabled
hostuuid: using 00000000-0000-0000-0000-000000000000
ULE: setup cpu 0
random: entropy device external interface
mem: <memory>
crypto: <crypto core>
null: <full device, null device, zero device>
openfirm: <Open Firmware control device>
WARNING: Device "openfirm" is Giant locked and may be deleted before FreeBSD 13.0.
ofwbus0: <Open Firmware Device Tree>
simplebus0: <Flattened device tree simple bus> on ofwbus0
plic0: <RISC-V PLIC> mem 0xc000000-0xc3fffff irq 4,5 on simplebus0
timer0: <RISC-V Timer>
Timecounter "RISC-V Timecounter" frequency 100000000 Hz quality 1000
Event timer "RISC-V Eventtimer" frequency 100000000 Hz quality 1000
riscv_syscon0: <RISC-V syscon> mem 0x50000000-0x50000fff on ofwbus0
rcons0: <RISC-V console>
cpulist0: <Open Firmware CPU Group> on ofwbus0
cpu0: <Open Firmware CPU> on cpulist0
cpu0: Nominal frequency 100Mhz
riscv64_cpu0: register <0>
uart0: <8250 or 16450 or compatible> mem 0x62300000-0x62300fff irq 6 on simplebus0
uart0: console (115200,n,8,1)
uart0: fast interrupt
uart0: PPS capture mode: DCD
virtio_mmio0: <VirtIO MMIO adapter> mem 0x40000000-0x40000fff irq 0 on ofwbus0
vtnet0: <VirtIO Networking Adapter> on virtio_mmio0
virtio_mmio0: host features: 0x100000020 <Version1,MacAddress>
virtio_mmio0: negotiated features: 0x100000020 <Version1,MacAddress>
virtio_mmio0: virtqueue 0 (vtnet0-0 rx) requested indirect descriptors but not negotiated
virtio_mmio0: virtqueue 1 (vtnet0-0 tx) requested indirect descriptors but not negotiated
vtnet0: bpf attached
vtnet0: Ethernet address: 02:00:00:00:00:00
virtio_mmio1: <VirtIO MMIO adapter> mem 0x40001000-0x40001fff irq 1 on ofwbus0
vtrnd0: <VirtIO Entropy Adapter> on virtio_mmio1
virtio_mmio1: host features: 0x100000000 <Version1>
virtio_mmio1: negotiated features: 0
random: registering fast source VirtIO Entropy Adapter
virtio_mmio2: <VirtIO MMIO adapter> mem 0x40002000-0x40002fff irq 2 on ofwbus0
vtblk0: <VirtIO Block Adapter> on virtio_mmio2
virtio_mmio2: host features: 0x100000000 <Version1>
virtio_mmio2: negotiated features: 0
virtio_mmio2: virtqueue 0 (vtblk0 request) requested indirect descriptors but not negotiated
vtblk0: vtblk_poll_request: IO error: 5
vtblk0: error getting device identifier: 5
vtblk0: 5353MB (10962949 512 byte sectors)
virtio_mmio3: <VirtIO MMIO adapter> mem 0x40003000-0x40003fff irq 3 on ofwbus0
virtio_mmio3: Unsupported version: 0
device_attach: virtio_mmio3 attach returned 6
syscon_power0: <Syscon poweroff> on ofwbus0
syscon_power1: <Syscon reboot> on ofwbus0
cryptosoft0: <software crypto>
crypto: assign cryptosoft0 driver id 0, flags 0x6000000
Device configuration finished.
procfs registered
Timecounters tick every 10.000 msec
GEOM: new disk vtbd0
lo0: bpf attached
vlan: initialized, using hash tables with chaining
IPsec: Initialized Security Association Processing.
tcp_init: net.inet.tcp.tcbhashsize auto tuned to 8192
Trying to mount root from ufs:/dev/ufs/root []...
WARNING: WITNESS option enabled, expect reduced performance.
Warning: no time-of-day clock registered, system time will not be set accurately
start_init: trying /sbin/init
No suitable dump device was found.
/etc/rc: WARNING: hostid: unable to figure out a UUID from DMI data, generating a new one
Setting hostuuid: 75c1d48f-687e-11eb-a4f3-5dd5f6a7fd67.
Setting hostid: 0xe6f36f92.
Starting file system checks:
/dev/ufs/root: FILE SYSTEM CLEAN; SKIPPING CHECKS
/dev/ufs/root: clean, 314746 free (10 frags, 39342 blocks, 0.0% fragmentation)
Mounting local filesystems:.
Setting hostname: cheribsd-riscv64-hybrid.
Setting up harvesting: PURE_VIRTIO,[UMA],[FS_ATIME],SWI,INTERRUPT,NET_NG,[NET_ETHER],NET_TUN,MOUSE,KEYBOARD,ATTACH,CACHED
Feeding entropy: random: unblocking device.
.
ELF ldconfig path: /lib /usr/lib /usr/lib/compat /usr/local/lib
ldconfig: /usr/local/lib: ignoring group-writable directory
devmatch: Can't read linker hints file.
lo0: link state changed to UP
vtnet0: link state changed to UP
Starting Network: lo0 vtnet0.
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
	inet6 ::1 prefixlen 128
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2
	inet 127.0.0.1 netmask 0xff000000
	groups: lo
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
vtnet0: flags=8963<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=28<VLAN_MTU,JUMBO_MTU>
	ether 02:00:00:00:00:00
	media: Ethernet 10Gbase-T <full-duplex>
	status: active
	nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
Starting devd.
devmatch: Can't read linker hints file.
devmatch: Can't read linker hints file.
devmatch: Can't read linker hints file.
Starting dhclient.
DHCPDISCOVER on vtnet0 to 255.255.255.255 port 67 interval 5
DHCPOFFER from 10.0.2.2
DHCPREQUEST on vtnet0 to 255.255.255.255 port 67
DHCPACK from 10.0.2.2
bound to 10.0.2.15 -- renewal in 43200 seconds.
add host 127.0.0.1: gateway lo0 fib 0: route already in table
add host ::1: gateway lo0 fib 0: route already in table
add net fe80::: gateway ::1
add net ff02::: gateway ::1
add net ::ffff:0.0.0.0: gateway ::1
add net ::0.0.0.0: gateway ::1
Creating and/or trimming log files.
Updating motd:.
Updating /var/run/os-release done.
Clearing /tmp (X related).
Starting syslogd.
NFS access cache time=60
Mounting late filesystems:.
Performing sanity check on sshd configuration.
Starting sshd.
Starting background file system checks in 60 seconds.

Sat Feb  6 13:28:18 UTC 2021

FreeBSD/riscv (cheribsd-riscv64-hybrid) (ttyu0)

login: 
```

If you want to restart the FPGA board, you need to unload the drivers first. 

```
sudo rmmod portalmem
sudo rmmod pcieportal
sudo fpga-load-local-image -S 0 -I agfi-026d853003d6c433a
sudo modprobe portalmem
modprobe pcieportal
sudo ./ssith-aws-fpga/build/ssith_aws_fpga --dtb ./ssith-aws-fpga/build/devicetree.dtb --elf ./cheri/output/sdk/bbl-fett/riscv64-purecap/bbl --elf ./cheri/output/rootfs-riscv64-hybrid/boot/kernel.CHERI-FETT/kernel --debug-log --block ./cheri/output/cheribsd-riscv64-hybrid.img
```

### Troubleshooting 
If you see this:
```
fpga-clear-local-image -S 0
/usr/local/bin/fpga-clear-local-image: line 18: 9693 Segmentation fault   fpga-local-cmd ClearFpgaImage "$@"
```
Then you didn't unload the drivers prior calling `fpga-clear-local-image`. You need to restart the VM, check that there are no drivers and repeat the procedure.

If your system is stuck with this output: 
```
setting pc val c0000000
reading pc val c0000000
bbl loader
```
You probably passed wrong ELF binaries: BBL and/or kernel

