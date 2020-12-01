---
layout: post
title: "IoT Firmware Emulation with QEMU"
author: "nonetype"
categories: pwn
tags: pwn_tips
---

Way to emulate IoT(home router, NAS, ipcam) firmware with QEMU

# 목차


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
<!-- code_chunk_output -->

- [목차](#목차)
- [Installation](#installation)
- [Run QEMU](#run-qemu)
- [Firmware Emulation](#firmware-emulation)
- [References](#references)

<!-- /code_chunk_output -->


# Installation
**주의!!** 설치할 펌웨어를 미리 까본 후(fmk 사용) `file bin/busybox` 명령을 통해 해당 바이너리가 Little/Big Endian인지 확인해야 한다!

Little Endian(LSB)라면 mipsel, Big Endian(MSB)라면 mips vmlinux/hda를 다운로드받으면 된다.

Example) `busybox: ELF 32-bit LSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), ...` -> `https://people.debian.org/~aurel32/qemu/mipsel/`

나는 Little endian binary를 구동하는 mipsel 환경을 구축하므로 본인 환경에 맞게 설정을 조금씩 바꾸면 된다.

```sh
sudo apt install qemu
cd /path/to/download
wget https://people.debian.org/~aurel32/qemu/mipsel/debian_wheezy_mipsel_standard.qcow2
wget https://people.debian.org/~aurel32/qemu/mipsel/vmlinux-3.2.0-4-4kc-malta
```

# Run QEMU
```sh
qemu-system-mipsel -M malta -kernel vmlinux-3.2.0-4-4kc-malta -hda debian_wheezy_mipsel_standard.qcow2 -append "root=/dev/sda1" -nographic -redir tcp:2222::22 -redir tcp:8080::80
[QEMU RUNS]
```
login 창이 뜨는데, default id/pw는 `root/root`다.

# Firmware Emulation
Firmware에 따라 rcS 파일의 위치가 다를 수 있다. `find / -name rcS` 명령을 통해 경로를 찾아 수정하여 진행하자.

```sh
<QEMU> service ssh start
<HOST> sudo tar -zcf rootfs.tar.gz rootfs/
<HOST> sudo scp -P 2222 ./rootfs.tar.gz root@127.0.0.1:/root
<QEMU> tar xvf rootfs.tar.gz
<QEMU> cd rootfs
<QEMU> chroot . ./bin/sh
<QEMU> #
<QEMU> /usr/etc/rcS
[initial settings...]
<QEMU> ps auxwww         (check httpd started)
[...]
12746 root      3852 S    /usr/sbin/mini_httpd -d /www -r NETGEAR R6950 -c **.c
[...]
<QEMU> netstat -antp     (check httpd started)

Active Internet connections (servers and established)
Proto Recv-Q Send-Q Can't open led device
Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      2264/exim4
tcp        0      0 0.0.0.0:47594           0.0.0.0:*               LISTEN      1607/rpc.statd
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      1576/rpcbind
tcp        0      0 0.0.0.0:56688           0.0.0.0:*               LISTEN      12906/miniupnpd
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      2037/sshd
tcp        0      0 ::1:25                  :::*                    LISTEN      2264/exim4
tcp        0      0 :::55116                :::*                    LISTEN      1607/rpc.statd
tcp        0      0 :::111                  :::*                    LISTEN      1576/rpcbind
tcp        0      0 :::80                   :::*                    LISTEN      12746/mini_httpd
tcp        0      0 :::22                   :::*                    LISTEN      2037/sshd

<HOST> 
```


# References
- [Emulating Embedded Linux Systems with QEMU
](https://www.novetta.com/2018/02/emulating-embedded-linux-systems-with-qemu/)
- [firmware 분석 환경 구축하기](https://bachs.tistory.com/entry/firmware-%EB%B6%84%EC%84%9D-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0)
