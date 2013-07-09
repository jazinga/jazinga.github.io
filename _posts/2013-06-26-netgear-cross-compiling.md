---
layout: post
title: Cross-compiling software for Netgear devices
---

{{ page.title }}
================

http://www.myopenrouter.com/download/34858/Original-Firmware-for-WNR3500Lv2-Source/
in the 

...

asterisk - ncurses


http://randomsplat.com/id5-cross-compiling-python-for-embedded-linux.html
    http://www.cnx-software.com/2011/02/04/cross-compiling-python-for-mips-and-arm-platforms/

Telnet access to the netgear router
-----------------------------------

./telnetenable 
./telnetenable 192.168.1.1 28C68E76ABC2 Gearguy Geardog > modpkt.pkt
nc 192.168.1.1 23 < modpkt.pkt 

Manual steps I've preformed:
    - done my python pre-staging
    - installed distribute using my x86 python build

    (A) In a number of cases, like 'lzo', I've noticed that shared libraries
        are not built -- I guess I am linking against static libraries? Perhaps
        I could be adding these libraries to the /sysroot directory to avoid
        having them in my bundle?

    (A) Asterisk still has build errors, the menuselect stuff needs to be build
        for x86, however my included makefile seems to mess that up?


../python/src/x86/usr/local/bin/python -measy_install python-gnupg
../python/src/x86/usr/local/bin/python -measy_install requests==0.6.4

PYTHONPATH=/tmp/dist//opt/jazinga/current/lib/python2.7/site-packages ../python/src/x86/usr/local/bin/python -measy_install --prefix=/tmp/dist//opt/jazinga/current requests==0.6.4
PYTHONPATH=/tmp/dist//opt/jazinga/current/lib/python2.7/site-packages ../python/src/x86/usr/local/bin/python -measy_install --prefix=/tmp/dist//opt/jazinga/current m2crypto


Missing 3rd party libraries? You will fail to build a number of
modules:

    Failed to build these modules:
    _bsddb             _curses            _curses_panel   
    _multiprocessing   _socket            _sqlite3        
    dbm                linuxaudiodev      ossaudiodev     
    select

Cross compiling sqlite3, and ncurses, then adding the destination -L and -I
paths to the CFLAGS and LDFLAGS resolved those.

You will also need to say 'CROSS_COMPILE_TARGET=yes' when calling 'make',
otherwise you will still see things like \_socket failing to build:

    Failed to build these modules:
    _bsddb             _multiprocessing   _socket         
    dbm                linuxaudiodev      ossaudiodev     
    select

Once CROSS_COMPILE_TARGET=yes is set, this entire set disappeared, and
I was instead geeted with a diffrent list that looks to need more 3rd
party libraries:

    Python build finished, but the necessary bits to build these modules were not found:
    _bsddb             _tkinter           bsddb185        
    bz2                dbm                gdbm            
    nis                readline           sunaudiodev     


Trying to get some more complicated modules depending on C libraries and SWIG
to work, like M2Crypto, I ran into:

    >>> from M2Crypto import BIO

    python: symbol '_IO_getc': can't resolve symbol

    python: symbol '__fxstat64': can't resolve symbol

I re-ran the import with python in verbose mode, and tracked it down to the
'io' module using:

    python -v -mM2Crypto.BIO

That allowed me to see that it was the 'io' module throwing an error, as we can
see here:

    >>> import io

    python: symbol '__fxstat64': can't resolve symbol
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/tmp/mnt/usb0/part1/opt/jazinga/current/lib/python2.7/io.py", line 60, in <module>
        import _io
    ImportError: unknown dlopen() error

On a whim, I recompiled python with the '--enable-shared' flag set, and that
resolved the import error.


export DEST_DIR=/tmp/dist/opt/jazinga/current
export DEST_PYTHONPATH=$DEST_DIR/lib/python2.7/site-packages
export X86_PYTHON=/home/wingu/netgear-mipsel/client/python/python/x86/usr/local/bin/python

Setting up your host python to install via easy_install:

    cd ~/netgear-mipsel/client/python/distribute/src
    ../../python/x86/usr/local/bin/python setup.py install
    ../../python/x86/usr/local/bin/python -measy_install distutilscross
    # I am not sure if it mattered that I easy_install'd distutilscross...


Building M2Crypto:

    # It is in fact important that distutilscross be installed in order for
    # m2crypto to compile, otherwise it will just use plain gcc for
    # compliation!

    cd /home/wingu/netgear-mipsel/client/python/pipable
    export PYTHONXCPREFIX=/tmp/dist//opt/jazinga/current    # <-- unclear if this matters
    export CROSS_COMPILE=arm-brcm-linux-uclibcgnueabi-      # <-- unclear if this matters
    $X86_PYTHON -measy_install --editable --build-directory . m2crypto
    cd m2crypto
    $X86_PYTHON setup.py build -x bdist_egg --plat-name=linux-arm build_ext --openssl=$DEST_DIR
                                                                               ^^^ this was important ^^^
    PYTHONPATH=$DEST_PYTHONPATH $X86_PYTHON setup.py install --prefix=$DEST_DIR
    # Not working VVVV
    #PYTHONPATH=/tmp/dist//opt/jazinga/current/lib/python2.7/site-packages ../python/x86/usr/local/bin/python -measy_install --prefix=/tmp/dist//opt/jazinga/current ./m2crypto


Building psutil:

    svn co http://psutil.googlecode.com/svn/trunk@1137 psutil
    PYTHONPATH=$DEST_PYTHONPATH $X86_PYTHON -measy_install --prefix=$DEST_DIR ./psutil


Building pyzeroconf:

    git clone https://github.com/jazinga/pyzeroconf.git
    PYTHONPATH=$DEST_PYTHONPATH $X86_PYTHON -measy_install --prefix=$DEST_DIR ./pyzeroconf


Building pika:

    git clone https://github.com/jazinga/pika.git
    PYTHONPATH=$DEST_PYTHONPATH $X86_PYTHON -measy_install --prefix=$DEST_DIR ./pika

Building mcp & autoprov:

    cd /home/wingu/netgear-mipsel/client
    PYTHONPATH=$DEST_PYTHONPATH $X86_PYTHON -measy_install --prefix=$DEST_DIR ./mcp
    PYTHONPATH=$DEST_PYTHONPATH $X86_PYTHON -measy_install --prefix=$DEST_DIR ./jazinga-autoprov-1.0


Building cow:

    cd /home/wingu/netgear-mipsel
    PYTHONPATH=$DEST_PYTHONPATH $X86_PYTHON -measy_install --prefix=$DEST_DIR ./common


Asterisk segemntation faults without any explanation

BASE=/tmp/mnt/usb0/part1/
BASE=/tmp/mnt/usb1/part1/
export PATH=$PATH:$BASE/opt/jazinga/current/bin:$BASE/opt/jazinga/current/sbin:$BASE/native-armv5l/bin
export PATH=$PATH:$BASE/usr/local/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$BASE/opt/jazinga/current/lib
cd $BASE/opt/jazinga/current/sbin
ulimit -c unlimited
./asterisk -C 


echo "1" > /proc/sys/kernel/core_uses_pid
echo "/tmp/mnt/usb0/part1" > /proc/sys/kernel/core_pattern

LD_LIBRARY_PATH=/tmp/mnt/usb0/part1/opt/jazinga/current/lib ldd asterisk
LD_LIBRARY_PATH=/tmp/mnt/usb0/part1/opt/jazinga/current/lib asterisk -vvvc
Segmentation fault


ap/gpl/minidlna won't build:
cd libogg*
ARCH=arm CROSS_COMPILE=arm-brcm-linux-uclibcgnueabi- make

samba explodes
apt-get install gawk

iptables-1.4.12 does not compile
absolute reference to /bin/arch should simply be 'arch' in Makefile

make LINUX_VERSION=2_6_36 CROSS_COMPILE=arm-brcm-linux-uclibcgnueabi- PLATFORM=arm

make PROFILE=AC1450 FW_TYPE=WW ARCH=arm PLT=arm LINUX_VERSION=2_6_36

CONFIG_ELF_CORE
/home/wingu/cross-environment/AC1450-V1.0.0.8_1.0.4_src/components/opensource/linux/linux-2.6.36
grep ELF_CORE .*
.config:# CONFIG_ELF_CORE is not set
.config_AC1450:# CONFIG_ELF_CORE is not set
.config.old:# CONFIG_ELF_CORE is not set
.config_R6300v2:# CONFIG_ELF_CORE is not set


export PATH=/projects/hnd/tools/linux/hndtools-arm-linux-2.6.36-uclibc-4.5.3/bin:$PATH
make PROFILE=R6300v2 FW_TYPE=WW ARCH=arm PLT=arm LINUX_VERSION=2_6_36
make PROFILE=R6300v2 FW_TYPE=WW ARCH=arm PLT=arm LINUX_VERSION=2_6_36 install

trying to make sense of an elf core dump
    http://stackoverflow.com/questions/12621511/gdb-wont-read-core-file-from-foreign-architecture




    astetcdir       => /tmp/mnt/usb0/part1/opt/jazinga/etc/asterisk
    astmoddir       => /tmp/mnt/usb0/part1/opt/jazinga/lib/asterisk/modules
    astvarlibdir    => /tmp/mnt/usb0/part1/opt/jazinga/var/lib/asterisk
    astdbdir        => /tmp/mnt/usb0/part1/opt/jazinga/var/lib/asterisk
    astkeydir       => /tmp/mnt/usb0/part1/opt/jazinga/var/lib/asterisk
    astdatadir      => /tmp/mnt/usb0/part1/opt/jazinga/var/lib/asterisk
    astagidir       => /tmp/mnt/usb0/part1/opt/jazinga/var/lib/asterisk/agi-bin
    astspooldir     => /tmp/mnt/usb0/part1/opt/jazinga/var/spool/asterisk
    astrundir       => /tmp/mnt/usb0/part1/opt/jazinga/var/run/asterisk
    astlogdir       => /tmp/mnt/usb0/part1/opt/jazinga/var/log/asterisk
    astsbindir      => /tmp/mnt/usb0/part1/opt/jazinga/sbin


Using TFTP from busybox
-----------------------

get: tftp -l local.file -r remote.file -g ip.addr
put: tftp -l local.file -r remote.file -p ip.addr

Building Bash
-------------
 * configure --enable-static-link --without-bash-malloc [1]
 * Patch to configure do set bash_cv_getenv_redef=no instead of =yes when cross compiling [2]


References:
 1. http://roycormier.net/2010/11/03/how-to-cross-compile-bash-for-android/
 2. http://lists.gnu.org/archive/html/bug-bash/2012-03/msg00052.html


BASE=/tmp/mnt/usb1/part1/

BASE=/tmp/mnt/usb0/part1/

export PATH=$PATH:$BASE/opt/jazinga/current/bin:$BASE/opt/jazinga/current/sbin
export PATH=$PATH:$BASE/usr/local/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$BASE/opt/jazinga/current/lib
cd $BASE/opt/jazinga/current/sbin
export ROOT=$BASE/opt/jazinga
export WINGU_ROOT=$ROOT/current
export GNUPGHOME=$ROOT/config/gnupg
export USER=nobody
export HOMEDIR=$ROOT/config
export USERNAME=$USER       # for enrollment to work
./rngd -r /dev/urandom    
cd ../bin

- Have to rewrite all the #!.../python to be just #!python
- Generating GPG keys is taking too long, not enough entropy!

gpg path is hardcoded in common/cow/gpg.py
                         common/cow/gpgstream.py
bash path is hardcoded in common/cow/gpgstream.py

ip path is hardcoded in cow/ipcmd.py

(A) There is still an issue where psutils does not build due to a python
    environment flag that -should- have been exported in some manner from my
    makefile cross-comiples thingy

