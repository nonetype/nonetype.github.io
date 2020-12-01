---
layout: post
title: "Linux Kernel debugging with Qemu"
author: "nonetype"
categories: dev
tags: vm
---

리눅스 커널 빌드 & Qemu를 통한 디버그

---

# 목차

1. TOC
{:toc}

---

# Kernel Configure&Build
일단 리눅스 커널 소스 다운로드 후 압축 해제한다.
여기선 4.17 버전으로 진행하지만, 원하는 버전이 있다면 아래 링크에서 선택하여 다운로드 받을 수 있다.
<https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/>

```sh
$ wget https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.17.tar.xz
$ ls
linux-4.17.tar.xz
$ tar -xvf linux-4.17.tar.xz
```

패키지 업데이트 후 필요한 패키지를 다운로드 받는다.

```sh
$ sudo apt update
$ sudo apt install build-essential libncurses5 libncurses5-dev bin86 kernel-package libssl-dev -y
```

중간에 분홍색 창이 떴는데 엔터(keep current~)를 선택했다.

압축 해제한 리눅스 소스 폴더로 이동한 다음, 아래 명령을 입력한다.

```sh
$ make defconfig
$ make kvmconfig
$ make menuconfig
```

![menuconfig1](/assets/menuconfig1.PNG)

`Kernel hacking > Compile-time checks and compiler options`로 이동 후 `Compile the kernel with debug info`를 체크

**- KASAN이 필요하면**
`Kernel hacking > Memory Debugging`로 이동 후 `KASan: runtime memory debugger` 체크, `KAsan: extra checks` 체크

**2019-09-17 추가**
`Processor type and features`로 이동, `Randomize the address of the kernel image (KASLR)` 체크 해제

**2019-10-11 추가**
`General setup / Enable userfaultfd() systemm call` 체크시 Exploit할 때 `userfault` syscall을 사용할 수 있다.
추가로 `Disable heap randomize` 체크시 힙 디버깅이 편해진다!!!

이후 esc키를 눌러 저장하고 나온다.
make를 시작하자.

```sh
$ make all
```

# Create root filesystem

빌드가 오래 걸리니 그동안 root filesystem을 구성한다.

아래 명령을 통해 `debootstrap` 패키지를 설치한다.

```
$ sudo apt install debootstrap
```

<https://github.com/google/syzkaller/blob/master/tools/create-image.sh> 파일을 긁어서 저장한다.

**create-image.sh**
```sh
#!/bin/bash
# Copyright 2016 syzkaller project authors. All rights reserved.
# Use of this source code is governed by Apache 2 LICENSE that can be found in the LICENSE file.

# create-image.sh creates a minimal Debian Linux image suitable for syzkaller.

set -eux

# Create a minimal Debian distribution in a directory.
DIR=chroot
PREINSTALL_PKGS=openssh-server,curl,tar,gcc,libc6-dev,time,strace,sudo,less,psmisc,selinux-utils,policycoreutils,checkpolicy,selinux-policy-default

# If ADD_PACKAGE is not defined as an external environment variable, use our default packages
if [ -z ${ADD_PACKAGE+x} ]; then
    ADD_PACKAGE="make,sysbench,git,vim,tmux,usbutils"
fi

# Variables affected by options
RELEASE=stretch
FEATURE=minimal
PERF=false

# Display help function
display_help() {
    echo "Usage: $0 [option...] " >&2
    echo
    echo "   -d, --distribution         Set on which debian distribution to create"
    echo "   -f, --feature              Check what packages to install in the image, options are minimal, full"
    echo "   -h, --help                 Display help message"
    echo "   -p, --add-perf             Add perf support with this option enabled. Please set envrionment variable \$KERNEL at first"
    echo
}

while true; do
    if [ $# -eq 0 ];then
	echo $#
	break
    fi
    case "$1" in
        -h | --help)
            display_help
            exit 0
            ;;
        -d | --distribution)
	    RELEASE=$2
            shift 2
            ;;
        -f | --feature)
	    FEATURE=$2
            shift 2
            ;;
        -p | --add-perf)
	    PERF=true
            shift 1
            ;;
        -*)
            echo "Error: Unknown option: $1" >&2
            exit 1
            ;;
        *)  # No more options
            break
            ;;
    esac
done

# Double check KERNEL when PERF is enabled
if [ $PERF = "true" ] && [ -z ${KERNEL+x} ]; then
    echo "Please set KERNEL environment variable when PERF is enabled"
    exit 1
fi

# If full feature is chosen, install more packages
if [ $FEATURE = "full" ]; then
    PREINSTALL_PKGS=$PREINSTALL_PKGS","$ADD_PACKAGE
fi

sudo rm -rf $DIR
mkdir -p $DIR
sudo debootstrap --include=$PREINSTALL_PKGS $RELEASE $DIR

# Set some defaults and enable promtless ssh to the machine for root.
sudo sed -i '/^root/ { s/:x:/::/ }' $DIR/etc/passwd
echo 'T0:23:respawn:/sbin/getty -L ttyS0 115200 vt100' | sudo tee -a $DIR/etc/inittab
printf '\nauto eth0\niface eth0 inet dhcp\n' | sudo tee -a $DIR/etc/network/interfaces
echo '/dev/root / ext4 defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'debugfs /sys/kernel/debug debugfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'securityfs /sys/kernel/security securityfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'configfs /sys/kernel/config/ configfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'binfmt_misc /proc/sys/fs/binfmt_misc binfmt_misc defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo "kernel.printk = 7 4 1 3" | sudo tee -a $DIR/etc/sysctl.conf
echo 'debug.exception-trace = 0' | sudo tee -a $DIR/etc/sysctl.conf
echo "net.core.bpf_jit_enable = 1" | sudo tee -a $DIR/etc/sysctl.conf
echo "net.core.bpf_jit_kallsyms = 1" | sudo tee -a $DIR/etc/sysctl.conf
echo "net.core.bpf_jit_harden = 0" | sudo tee -a $DIR/etc/sysctl.conf
echo "kernel.softlockup_all_cpu_backtrace = 1" | sudo tee -a $DIR/etc/sysctl.conf
echo "kernel.kptr_restrict = 0" | sudo tee -a $DIR/etc/sysctl.conf
echo "kernel.watchdog_thresh = 60" | sudo tee -a $DIR/etc/sysctl.conf
echo "net.ipv4.ping_group_range = 0 65535" | sudo tee -a $DIR/etc/sysctl.conf
echo -en "127.0.0.1\tlocalhost\n" | sudo tee $DIR/etc/hosts
echo "nameserver 8.8.8.8" | sudo tee -a $DIR/etc/resolve.conf
echo "syzkaller" | sudo tee $DIR/etc/hostname
ssh-keygen -f $RELEASE.id_rsa -t rsa -N ''
sudo mkdir -p $DIR/root/.ssh/
cat $RELEASE.id_rsa.pub | sudo tee $DIR/root/.ssh/authorized_keys

# Add perf support
if [ $PERF = "true" ]; then
    cp -r $KERNEL $DIR/tmp/
    sudo chroot $DIR /bin/bash -c "apt-get update; apt-get install -y flex bison python-dev libelf-dev libunwind8-dev libaudit-dev libslang2-dev libperl-dev binutils-dev liblzma-dev libnuma-dev"
    sudo chroot $DIR /bin/bash -c "cd /tmp/linux/tools/perf/; make"
    sudo chroot $DIR /bin/bash -c "cp /tmp/linux/tools/perf/perf /usr/bin/"
    rm -r $DIR/tmp/linux
fi

# Build a disk image
dd if=/dev/zero of=$RELEASE.img bs=1M seek=2047 count=1
sudo mkfs.ext4 -F $RELEASE.img
sudo mkdir -p /mnt/$DIR
sudo mount -o loop $RELEASE.img /mnt/$DIR
sudo cp -a $DIR/. /mnt/$DIR/.
sudo umount /mnt/$DIR
```

권한 변경 후 실행하여 root filesystem을 만든다.

```sh
$ chmod 744 ./create-image.sh
$ ./create-image.sh
```

`stretch.img` 이미지가 정상적으로 생성되었다.

```sh
nonetype@pwn:~/linux$ ls
chroot  create-image.sh  linux-4.17  linux-4.17.tar.xz  stretch.id_rsa  stretch.id_rsa.pub  stretch.img
nonetype@pwn:~/linux$
```

# Run Linux Kernel
커널 빌드가 완료되면, 빌드된 bzImage를 확인해보자.

```sh
nonetype@pwn:~/linux/linux-4.17$ file ./arch/x86_64/boot/bzImage
./arch/x86_64/boot/bzImage: symbolic link to ../../x86/boot/bzImage
nonetype@pwn:~/linux/linux-4.17$ ls -l ./arch/x86/boot/bzImage
-rw-r--r-- 1 nonetype nonetype 14749744  9월 17 14:39 ./arch/x86/boot/bzImage
nonetype@pwn:~/linux/linux-4.17$
```

생성된 `bzImage`를 `stretch.img`와 동일한 디렉터리로 이동시킨다.

```sh
nonetype@pwn:~/linux/linux-4.17$ cp  ./arch/x86/boot/bzImage ../
```

qemu를 통해 커널을 실행한다.
```sh
$ qemu-system-x86_64 -kernel bzImage -append "console=ttyS0 root=/dev/sda debug" -hda stretch.img -net user,hostfwd=tcp::10021-:22 -net nic -nographic -m 2G -smp 2 -s
```

커널 로그가 쭉 뜨다가 쉘이 뜬다.

```sh
root@syzkaller:~#
```

파일시스템 최대 사이즈를 조절하려면 `dd if=/dev/zero of=$RELEASE.img bs=1M seek=2047 count=1` 부분의 `seek=` 값을 변경하면 된다. (2047 = 2GB)

ssh 접속은 hostOS에서 아래와 같은 방식으로 한다.
```sh
$ ssh -p10021 -i stretch.id_rsa root@localhost
```

# Kernel Debugging with Qemu
일단 실행중인 Qemu를 종료하고 아래 명령을 통해 gdb를 실행한다.
```sh
nonetype@pwn:~/linux/linux-4.17$ ls
COPYING        Kbuild    MAINTAINERS     README      block       crypto    fs       ipc     mm               net      security  usr      vmlinux.o
CREDITS        Kconfig   Makefile        System.map  built-in.a  drivers   include  kernel  modules.builtin  samples  sound     virt
Documentation  LICENSES  Module.symvers  arch        certs       firmware  init     lib     modules.order    scripts  tools     vmlinux
nonetype@pwn:~/linux/linux-4.17$ gdb -q vmlinux
Reading symbols from vmlinux...done.
GEF for linux ready, type `gef' to start, `gef config' to configure
77 commands loaded for GDB 8.1.0.20180409-git using Python engine 3.6
[*] 3 commands could not be loaded, run `gef missing` to know why.
gef➤
```

`start_kernel` 함수에 bp를 걸어보자.

```sh
gef➤  b * start_kernel
Breakpoint 1 at 0xffffffff83928d6e: file init/main.c, line 532.
gef➤
```

이제 `target remote` 명령을 통해 gdb attach 대기를 시켜놓고 qemu를 켠다.

**debugger**
```sh
gef➤  target remote localhost:1234
```

**run qemu**
```sh
$ qemu-system-x86_64 -kernel bzImage -append "console=ttyS0 root=/dev/sda debug" -hda stretch.img -net user,hostfwd=tcp::10021-:22 -net nic -nographic -m 2G -smp 2 -s
WARNING: Image format was not specified for 'stretch.img' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.
qemu-system-x86_64: warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
qemu-system-x86_64: warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]

(paused)
```

KASLR이 해제되어 있다면 vmlinux 파일의 심볼과 qemu 심볼이 매치되어 정상적으로 bp가 걸린다.

# References
<https://tistory.0wn.kr/368>


<https://github.com/google/syzkaller/blob/master/tools/create-image.sh>


<https://willow72.tistory.com/>
