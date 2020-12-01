---
layout: post
title: "Linux Kernel debugging with Vmware"
author: "nonetype"
categories: dev
tags: vm
---


VMware를 이용해 리눅스 커널 디버깅을 해보자

---

# 목차

1. TOC
{:toc}

---

# guestOS Setting

## Debug symbol download
`sudo apt-get update && sudo apt-get upgrade` 명령을 통해 패키지 업그레이드를 진행한다.

이후 다음 명령을 실행한다.
```sh
echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list
```

Ubuntu 18.04
```sh
sudo apt install ubuntu-dbgsym-keyring
```
Ubuntu 16.04
```sh
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys F2EDC64DC5AEE1F6B9C621F0C8CAB6595FDFF622
```

심볼 설치!
```sh
sudo apt-get update
sudo apt-get install linux-image-`uname -r`-dbgsym
```

이후 `/usr/lib/debug/boot` 디렉토리를 확인해 보면 다음과 같이 나타난다.
```sh
debug@debug-virtual-machine:~$ ls /usr/lib/debug/boot/
vmlinux-4.15.0-45-generic
debug@debug-virtual-machine:~$
```

이 파일을 hostOS(debugger를 실행할)로 옮겨준다.

## KASLR 해제
`/etc/default/grub`의 `GRUB_CMDLINE_LINUX_DEFAULT`필드에 "nokaslr"를 추가해준다.
```sh
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'

GRUB_DEFAULT=0
GRUB_HIDDEN_TIMEOUT=0
GRUB_HIDDEN_TIMEOUT_QUIET=true
GRUB_TIMEOUT=10
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nokaslr"
GRUB_CMDLINE_LINUX=""

# Uncomment to enable BadRAM filtering, modify to suit your needs
# This works with Linux (no patch required) and with any kernel that obtains
# the memory map information from GRUB (GNU Mach, kernel of FreeBSD ...)
#GRUB_BADRAM="0x01234567,0xfefefefe,0x89abcdef,0xefefefef"

# Uncomment to disable graphical terminal (grub-pc only)
#GRUB_TERMINAL=console

# The resolution used on graphical terminal
# note that you can use only modes which your graphic card supports via VBE
# you can see them in real GRUB with the command `vbeinfo'
#GRUB_GFXMODE=640x480

# Uncomment if you don't want GRUB to pass "root=UUID=xxx" parameter to Linux
#GRUB_DISABLE_LINUX_UUID=true

# Uncomment to disable generation of recovery mode menu entries
#GRUB_DISABLE_RECOVERY="true"

# Uncomment to get a beep at grub start
#GRUB_INIT_TUNE="480 440 1"
```

설정 저장 후 `sudo update-grub` 필수!!!

# vmx configure

guestOS의 VM file 경로로 이동해서 .vmx 파일의 마지막 줄에 다음 라인을 추가한다.

**X32**
```sh
debugStub.listen.guest32 = "TRUE"
debugStub.listen.guest32.remote = "TRUE"
debugStub.hideBreakpoints = "FALSE"
monitor.debugOnStartGuest32 = "TRUE"
```

**X64**
```sh
debugStub.listen.guest64 = "TRUE"
debugStub.listen.guest64.remote = "TRUE"
debugStub.hideBreakpoints = "FALSE"
monitor.debugOnStartGuest64 = "TRUE"
```

.vmx 파일 저장 후 guest OS를 부팅한다.


# remote attach

GDB를 또 컴파일하긴 귀찮으니 IDA로 guestOS에서 옮긴 vmlinux 파일을 깐다.

긴 인내의 시간 후, IDA가 vmlinux 바이너리를 열고 symbol load가 끝나면, 일단 다시 로드하지 않도록 database save를 한번 해주자.

이후 `Debugger - Select Debugger - Remote GDB debugger` 선택, `Debugger - Process options`로 들어간다.
Hostname 필드는 `localhost`, Port는 `32bit- 8832, 64bit- 8864`(VMware debugging service port)로 설정해준다.


설정이 끝났으면 `Debugger - Attach to Process`를 클릭해 Remote attach한 후 디버깅을 시작한다!

# Additional Info
내가 할때는 왜인지는 모르겠지만 IDA로 확인한 vmlinux 심볼과 실제 guestOS에서 실행되는 함수 주소가 달랐다.
간단한 예를 들면, IDA에서 functions window로 확인한 `vfs_kern_mount()`의 주소는 `0xffffffff8129D7F0`이었는데, 실제 guestOS에서 `cat /proc/kallsyms | grep vfs_kern_mount`로 확인한 주소값은 `0xffffffffaea9d7f0`이었다.

이전에 IDA의 심볼 주소만 믿고 BP를 걸었다가 안되서 의문이었는데, 일단 BP는 잡히지만 왜 실제 주소가 다른지는 잘 모르겠다. 더 찾아볼 예정!!

**2019-09-10 추가**
위에서 심볼 주소가 안맞았던 이유가 GRUB 설정으로 nokaslr 옵션을 추가하고 `sudo update-grub` 명령을 입력해주지 않아서였다. 필수적으로 입력해주자!!

![result](/assets/result.PNG)


# References
http://jidanhunter.blogspot.com/2015/01/linux-kernel-debugging-with-vmware-and.html

https://wiki.ubuntu.com/Debug%20Symbol%20Packages

http://www.alexlambert.com/2017/12/18/kernel-debugging-for-newbies.html

https://www.lazenca.net/display/TEC/02.Debugging+kernel+and+modules

https://dorgamza.tistory.com/1
