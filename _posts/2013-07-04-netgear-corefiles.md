---
layout: post
title: Enabling core files on the Netgear WNDR6300v2
---
   
By default, the linux kernel on the device does not have core dumps enabled, so
you will need to rebuild it and re-image the Netgear device.

Go into the netgear source bundle, and find the kernel configuration file. My
last look found it under this path:

    R6300v2-V1.0.1.72_1.0.21_src/components/opensource/linux/linux-2.6.36/

From there, I modified .config_R6300v2, and set:

    CONFIG_ELF_CORE=y

Then rebuilt the router image by following the instructions in the top level
README. At the time of writing, the steps to accomplish this were:

    export PATH=/projects/hnd/tools/linux/hndtools-arm-linux-2.6.36-uclibc-4.5.3/bin:$PATH                                     
    make PROFILE=R6300v2 FW_TYPE=WW ARCH=arm PLT=arm LINUX_VERSION=2_6_36                                                      
    make PROFILE=R6300v2 FW_TYPE=WW ARCH=arm PLT=arm LINUX_VERSION=2_6_36 install

Install the produced image onto the netgear device through their web interface,
and re-load your software.  I was using a USB stick to store my applications,
so rebooting was sufficient.

Getting your application to produce a core file is now a matter of the runtime
environment settings, so use ulimit to enable core files:

    ulimit -c unlimited

This will normally produce a core file in the directory from where the program
is launched. There are some sysctl, and proc controls that can affect this, but
I did not explore them [1][2]:

    # cat /proc/sys/kernel/core_pattern
    core
    # cat /proc/sys/kernel/core_uses_pid
    0
    # echo "1" > /proc/sys/kernel/core_uses_pid
    # echo "/tmp/corefiles/core" > /proc/sys/kernel/core_pattern

    # sysctl -w fs.suid_dumpable = 1
    # sysctl -w kernel.core_uses_pid = 1

[1] http://www.fromdual.com/hunting-the-core
[2] http://en.linuxreviews.org/HOWTO_enable_core-dumps

I have yet to determine how to use GDB 
