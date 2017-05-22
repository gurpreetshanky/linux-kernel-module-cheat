# Linux Kernel Module Cheat

Run one command, get into QEMU Buildroot BusyBox with several minimal Linux kernel 4.9 module example tutorials with GDB debug support. x86 and ARM guests supported. Tested in Ubuntu 14.04 - 16.10 hosts.

Usage:

    sudo apt-get install qemu
    ./run

First build will take a while (GCC, Linux kernel).

QEMU opens up, and you can run:

    root
    insmod /hello.ko
    insmod /hello2.ko
    rmmod hello
    rmmod hello2

This should print to the screen:

    hello init
    hello2 init
    hello cleanup
    hello2 cleanup

which are `printk` messages from `init` and `cleanup` methods of those modules.

Each module comes from a C file under `kernel_module/`. For module usage see:

    head kernel_module/*.c

After the first build, you can also run just:

    ./runqemu

to save a few seconds. `./run` wouldn't rebuild everything, but checking timestamps takes a few moments.

We use `printk` a lot, and it shows on the QEMU terminal by default. If that annoys you (e.g. you want to see stdout separately), do:

    dmesg -n 1

See also: <https://superuser.com/questions/351387/how-to-stop-kernel-messages-from-flooding-my-console>

We use Buildroot's default kernel version, you can confirm it after build with:

    grep BR2_LINUX_KERNEL_VERSION buildroot/.config

## Text mode

Show serial output of QEMU directly on the current terminal, without opening a QEMU window:

    ./run -n

To exit, just do a regular:

    poweroff

This is particularly useful to get full panic traces when you start making the kernel crashing :-)

In case of a panic, you want your terminal back with `Ctrl + C, A` and type `quit`. See also: <http://stackoverflow.com/questions/14165158/how-to-switch-to-qemu-monitor-console-when-running-with-curses>

See also: <https://unix.stackexchange.com/questions/208260/how-to-scroll-up-after-a-kernel-panic>

## Debugging

To GDB the Linux kernel, first run:

    ./runqemu -d

This starts QEMU on the background of the shell, to prepare for running GDB.

If you want to break immediately at a symbol, e.g. `start_kernel` of the boot sequence, run:

    ./rungdb start_kernel

Now QEMU will stop there, and you can use the normal GDB commands:

    l
    n
    c

To skip the boot, run just:

    ./rungdb

and when you want to break, do `Ctrl + C` from GDB.

To have some fun, you can first run inside QEMU:

    /count.sh

which counts to infinity to stdout, and then:

    Ctrl + C
    break sys_write

And now you can control the counting from GDB:

    continue
    continue
    continue

See also: <http://stackoverflow.com/questions/11408041/how-to-debug-the-linux-kernel-with-gdb-and-qemu>

If you are using text mode:

    ./runqemu -d -n

QEMU cannot be put on the background of the current shell, so you will need to open a separate terminal and run:

    ./rungdb

manually.

### Kernel module debugging

Loadable kernel modules are a bit trickier since the kernel can place them at different memory locations depending on load other.

So we cannot set the breakpoints before `insmod`.

However, the Linux kernel GDB scripts offer the `lx-symbols` command, which takes care of that beautifully for us:

    ./runqemu -d
    ./rungdb

In QEMU:

    insmod /fops.ko

In GDB, hit `Ctrl + C`, and note how it says:

    scanning for modules in ../kernel_module-1.0/
    loading @0xffffffffa0000000: ../kernel_module-1.0//fops.ko

That's `lx-symbols` working! Now simply:

    b fop_write
    c

In QEMU:

    printf a >/sys/kernel/debug/kernel_module_cheat/fops

and GDB now breaks at our `fop_write` function!

Just don't forget to remove your breakpoints after `rmmod`, or they will point to stale memory locations.

TODO: why does `break work_func` for `insmod kthread.ko` not break the first time I `insmod`, but breaks the second time?

#### Bypassing lx-symbols

Useless, but a good way to show how hardcore you are. From inside QEMU:

    insmod /fops.ko
    cat /proc/modules

This will give a line of form:

    fops 2327 0 - Live 0xfffffffa00000000

And then tell GDB where the module was loaded with:

    Ctrl + C
    add-symbol-file ../kernel_module-1.0/fops.ko 0xfffffffa00000000

## ARM

The portability of the kernel and toolchains is amazing.

If you already have an x86 build present, first:

    cd buildroot
    mv output output.x86~

First ARM build:

    ./run -a arm

Run without build:

    ./runqemu -a arm

Debug:

    ./runqemu -a arm -d
    # On another terminal.
    ./rungdb -a arm

ARM TODOs:

-   only managed to run in the terminal interface (but weirdly an blank QEMU window is still opened)
-   Ctrl + C kills the emulator, not sent to guest. See:
    - <https://github.com/cloudius-systems/osv/issues/49>
    - <https://unix.stackexchange.com/questions/167165/how-to-pass-ctrl-c-in-qemu>
-   `fops.ko`, `printf a >fops` crashes with `BUG: recent printk recursion!`

## Table of contents

1.  [Introduction](introduction.md)
1.  [Build](build.md)
1.  [kmod](kmod.md)
1.  [Bibliography](bibliography.md)
1.  Examples
    1.  [Host](host/)
    1.  Buildroot
        1.  Debugging
            1.  [hello](kernel_module/hello.c)
            1.  [hello2](kernel_module/hello2.c)
            1.  [debugfs](kernel_module/debugfs.c)
            1.  [panic](kernel_module/panic.c)
        1.  [fops](kernel_module/fops.c)
        1.  Asynchronous
            1. [workqueue](kernel_module/workqueue.c)
            1. [sleep](kernel_module/sleep.c)
            1. [kthread](kernel_module/kthread.c)
            1. [timer](kernel_module/timer.c)
            1. [work_from_work](kernel_module/work_from_work.c)
