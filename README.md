# Booting Linux for POWER on gem5

This describes the process of setting up a full system simulation environment
for booting bare-metal Linux on the gem5 simulator. This includes running the
OpenPower Abstraction Layer (OPAL) firmware followed by the Linux kernel and
eventually dropping to a simple userspace shell program.

## Setup Build Environment

To be able to run the simulations, the firmware, kernel and simulator need to
be built from sources. The source code for each of these components can be
cloned from the repositories under [power-gem5](https://github.com/power-gem5).

### Install Prerequisites

Building each of the components requires certain tools and libraries to be
available. This shows how they can be installed on popular distributions.

#### Ubuntu

```
sudo apt update
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev     \
                 libprotobuf-dev protobuf-compiler libprotoc-dev    \
                 libgoogle-perftools-dev python-dev python expect   \
                 flex bison libncurses-dev openssl libssl-dev       \
                 libelf-dev autoconf xz-utils device-tree-compiler  \
                 gcc-powerpc64-linux-gnu gdb-multiarch valgrind     \
                 telnet
```

#### Fedora

```
sudo dnf update
sudo dnf install gcc gcc-c++ git make m4 python2-scons zlib-devel     \
                 protobuf-devel protobuf-compiler gperftools-devel    \
                 python2-devel expect flex bison diffutils findutils  \
                 ncurses-devel openssl-devel elfutils-libelf-devel    \
                 autoconf dtc gcc-powerpc64-linux-gnu valgrind-devel  \
                 binutils-powerpc64-linux-gnu gdb xz telnet
```

### Build Support Package

We will start with building the device tree blob and setting up the support
package under a distribution directory that must have the following hierarchy
to be able to run the gem5 simulator in full-system mode.

```
dist/
└── m5
    └── system
        ├── binaries
        │   ├── gem5-power9-fs.dtb
        │   ├── objdump_vmlinux
        │   ├── objdump_skiboot
        │   ├── vmlinux
        │   └── skiboot.elf
        └── disks
            └── linux-latest.img
```

The path to this directory must be part of the `M5_PATH` environment variable
to make its contents accessible to the gem5 simulator. This also contains an
`initramfs` image that will embedded into the `vmlinux` binary upon building
the kernel.

The device tree blob can be built and `M5_PATH` can be set as shown below.
For convenience, once all of the components are built, `.bashrc` can be
modified to setup `M5_PATH` automatically.

```
git clone https://github.com/power-gem5/gem5-support-package.git
export M5_PATH=$(pwd)/gem5-support-package/dist/m5/system
cd gem5-support-package
dtc -I dts -O dtb -o $M5_PATH/binaries/gem5-power9-fs.dtb $M5_PATH/binaries/gem5-power9-fs.dts
cd ../
```

### Build Firmware

The `skiboot` firmware binary provides the OpenPower Abstraction Layer (OPAL)
runtime services. This is also the component with which the simulator starts
execution and this eventually hands-off control to the kernel. Some hacks and
workarounds were required to get the firmware up and running on the simulator.
We will be switching to the sources under the `gem5-experimental` branch which
contain these changes. The firmware binary can be built and copied to the
distribution directory as shown below. A cross compiler is used here since the
binary is targeted for `powerpc64` big-endian systems.

```
git clone https://github.com/power-gem5/skiboot.git
cd skiboot
git checkout gem5-experimental
make CROSS=powerpc64-linux-gnu- -j4
powerpc64-linux-gnu-objdump -D skiboot.elf | tee $M5_PATH/binaries/objdump_skiboot 2>&1 > /dev/null
cp skiboot.elf $M5_PATH/binaries/
cd ../
```

### Build Kernel

Like the firmware, the kernel also needs to be cross-compiled. This also has
some hacks and workaround for getting things like the virtual serial console
working. This is why we need to switch to the `gem5-experimental` branch.
The `vmlinux` binary can be built and copied to the distribution directory as
shown below. The simulator also requires an `objdump` of the binary.

```
git clone https://github.com/power-gem5/linux.git
cd linux
git checkout gem5-experimental
make ARCH=powerpc CROSS_COMPILE=powerpc64-linux-gnu- gem5_defconfig
make ARCH=powerpc CROSS_COMPILE=powerpc64-linux-gnu- -j4 vmlinux
powerpc64-linux-gnu-objdump -D vmlinux | tee $M5_PATH/binaries/objdump_vmlinux 2>&1 > /dev/null
cp vmlinux $M5_PATH/binaries/
cd ../
```

### Build Simulator

Finally, we will be building the gem5 simulator, or specifically, the `fast`
variant of the simulator which increases the simulation speed significantly
by avoiding detailed tracing. The official upstream sources of the simulator
currently supports running only simple 32-bit big-endian userspace binaries
in syscall emulation mode. For full-system simulation to work, we need to
switch to the `gem5-experimental` branch. The simulator can be built as shown
below.

```
git clone https://github.com/power-gem5/gem5.git
cd gem5
ln -s $M5_PATH/../../ dist
git checkout gem5-experimental
scons CPU_MODELS="AtomicSimpleCPU" build/POWER/gem5.fast -j4
```

> N.B. On Fedora, use `scons-2` provided by the `python2-scons` package.

If detailed tracing is required, the `debug` variant of the simulator can be
built although simulation will be far slower.

## Run Simulation

A full-system simulation can be started in `fast` mode as shown below.

```
./build/POWER/gem5.fast configs/example/fs.py
```

For `debug` mode with detailed per-instruction traces, the simulation can be
run as shown below. This will require the `debug` variant of simulator to be
built and available.

```
./build/POWER/gem5.debug --debug-flags=ExecAll,Registers configs/example/fs.py
```

After the simulation starts, users can connect to a `telnet`-based virtual
serial console to view the kernel boot logs.

```
telnet localhost 3456
```

### Debug Kernel

The gem5 simulator has a remote debugging facility with which users can step
through and inspect the workload running in the simulator. This is done using
remote `gdb` sessions. Once the simulation has started, a debugger instance
can be attached to it from the `gdb` console as shown below.

```
(gdb) file $M5_PATH/binaries/vmlinux
(gdb) set directories $M5_PATH/../../../../linux/
(gdb) set remote Z-packet on
(gdb) set step-mode on
(gdb) target remote localhost:7000
```

> N.B. The distribution-provided `gdb` package might not come with support
  for debugging `powerpc64` binaries. So `gdb-multiarch` (or a similar
  flavour of it) is required.
