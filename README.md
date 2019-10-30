# Booting OPAL firmware and Linux kernel on gem5

This describes the steps for setting up a full system simulation environment for booting the OPAL firmware followed by the Linux kernel and eventually running a simple userspace shell program.


All of the executable binaries for the linux and skiboot firmware can be found in this [archive](https://github.com/power-gem5/gem5-support-package/raw/master/gem5-support-package.7z), to boot the OS right away jump [here](#running-gem5).

To compile from scratch use the corresponding repositories on [power-gem5](https://github.com/power-gem5) and continue reading.

## Building the device tree blob
A minimal device tree source (which can be found in this [archive](https://github.com/power-gem5/gem5-support-package/raw/master/gem5-support-package.7z)) has been created and can be converted to a device tree blob using:

```
$ dtc -I dts -O dtb -o devicetree_file_name.dtb devicetree_file_name.dts
```
A pre-requisite here is to have the device tree compiler (`dtc`) installed. This can be done easily in commonly used distros as shown below.
```
# For Ubuntu
$ sudo apt-get install device-tree-compiler

# For Fedora
$ sudo dnf install dtc
```

## Building the Linux kernel
Clone the kernel [source](https://github.com/power-gem5/linux/) and checkout to the `gem5-experimental` branch.
Make sure you have the cross compiler installed for powerpc64 and that you compile the kernel in `Big Endian` mode.

```
$ make ARCH=powerpc CROSS_COMPILE=powerpc64-linux-gnu- gem5_defconfig
$ make ARCH=powerpc CROSS_COMPILE=powerpc64-linux-gnu- -j16
```

## Building the OPAL firmware
Clone the firmware [source](https://github.com/power-gem5/skiboot) and checkout to the `gem5-experimental` branch.

Make sure this too needs to be compiled in the `Big Endian` mode. 

```
$ make CROSS_COMPILE=powerpc64-linux-gnu- -j16
```

## Building gem5
Clone the gem5 [source](https://github.com/power-gem5/gem5) and checkout to the `gem5-experimental` branch.
```
$ scons CPU_MODELS="AtomicSimpleCPU" build/POWER/gem5.fast -j16
```

For relevant packages and a complete walkthrough of compilation can be found [here](http://learning.gem5.org/book/part1/building.html)

## Running gem5

A specific hierarchy must be maintained in the root directory of the gem5 source.

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

This [archive](https://github.com/power-gem5/gem5-support-package/raw/master/gem5-support-package.7z) contains a compiled linux kernel image with a built in initramfs shell, image of the firmware along with it's respective objdump and a minimal device tree blob. Everthing is organised in the same directory hierarchy shown above.

To execute the full system mode in `fast` mode.
```
$ ./build/POWER/gem5.fast configs/example/fs.py
```

A console device also exists for POWER and can be connected to using 

```
$ telnet localhost 3456
```

## Remote Debugging in gem5

Remote debugging functionality to help debug the workload running in gem5.

Steps to attach the remote gdb are as follows:-
> Note: The distro-provided gdb might not come with support for debugging binaries of different architectures. So you may have to install `gdb-multiarch` (or a similar flavor of it).

```
file vmlinux
set directories linux/
set remote Z-packet on
set step-mode on
target remote localhost:7000
```

