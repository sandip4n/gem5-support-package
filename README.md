# Booting OPAL firmware + Linux Kernel on gem5

The below documentation is for bringing up a full system running the OPAL firmware + Linux running a simple userspace shell program.


All of the executable binaries for the linux and skiboot firmware can be found [here](#), to boot the OS right away jump [here](#running-gem5).

To compile from scratch use the corresponding repositories on [power-gem5](https://github.com/power-gem5) and continue reading.

## Compiling the device tree 
A minimal [device tree source](#) has been created and can be converted to a device tree blob using :  

```
$ dtc -I dts -O dtb -o devicetree_file_name.dtb devicetree_file_name.dts
```

## Compiling Linux Kernel
Clone the kernel [source](https://github.com/power-gem5/linux/) and checkout to the `gem5-experimental` branch.

Make sure you have the cross compiler installed for powerpc and that you compile the kernel in `Big Endian` mode.

```
$ make ARCH=powerpc CROSS_COMPILE=powerpc64-linux-gnu- gem5_defconfig
$ make ARCH=powerpc CROSS_COMPILE=powerpc64-linux-gnu- -j16
```

## Compiling Skiboot
Clone the firmware [source](https://github.com/power-gem5/skiboot) and checkout to the `gem5-experimental` branch.

Make sure this too needs to be compiled in the `Big Endian` mode. 

```
$ make CROSS_COMPILE=powerpc64-linux-gnu- -j16
```

## Compiling gem5 for POWER
Clone the gem5 [source](https://github.com/power-gem5/gem5) and checkout to the `gem5-experimental` branch.
```
$ scons CPU_MODELS="AtomicSimpleCPU" build/POWER/gem5.fast -j16
```

For relevant packages and a complete walkthrough of compilation can be found [here](http://learning.gem5.org/book/part1/building.html)

<a name="running-gem5"></a>
##Running gem5

A folder hierarchy must be maintained in the root folder of the gem5 source.

```
dist/
└── m5
    └── system
        ├── binaries
        │   ├── gem5-power9-fs.dtb
        │   ├── objdump_vmlinux
        │   ├── objdump_skiboot
        │   ├── vmlinux
        │   ├── skiboot.elf
        └── disks
            ├── linux-latest.img

```

This [archive]() contains a compiled linux kernel image with a built in initramfs shell, image of the firmware along with it's respective objdump and a minimal device tree blob.

To execute the full system mode in `fast` mode.
```
$ ./build/POWER/gem5.fast configs/example/fs.py
```

A console device also exists for POWER and can be connected to using 

```
$ telnet localhost 3456
```

## Remote Debugging in Gem-5

Remote debugging functionality to help debug the workload running in gem-5.

Steps to attach the remote gdb are as follows:-
> Note: The distro-provided gdb might not come with support for debugging binaries of different architectures. So you may have to install `gdb-multiarch` (or a similar flavor of it).

```
file vmlinux
set directories linux/
set remote Z-packet on
set step-mode on
target remote localhost:7000
```

