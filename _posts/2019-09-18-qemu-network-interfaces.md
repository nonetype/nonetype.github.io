---
layout: post
title: "[SOLVED] Failed to start Raise network interfaces."
author: "nonetype"
categories: dev
tags: vm
---

qemu에서 네트워크 설정.

---

# 목차

1. TOC
{:toc}

---

# Failed to start

qemu에서 커널 실행시 아래와 같은 오류가 나타난다.

```sh
[FAILED] Failed to start Raise network interfaces.
See 'systemctl status networking.service' for details.
[  OK  ] Reached target Network.
         Starting Permit User Sessions...
         Starting OpenBSD Secure Shell server...
[  OK  ] Started Permit User Sessions.
[  OK  ] Started Getty on tty4.
[  OK  ] Started Getty on tty2.
root@syzkaller:~# systemctl status networking.service
● networking.service - Raise network interfaces
   Loaded: loaded (/lib/systemd/system/networking.service; enabled; vendor prese
   Active: failed (Result: exit-code) since Wed 2019-09-18 07:52:12 UTC; 1min 23
     Docs: man:interfaces(5)
  Process: 1650 ExecStart=/sbin/ifup -a --read-environment (code=exited, status=
  Process: 1089 ExecStartPre=/bin/sh -c [ "$CONFIGURE_INTERFACES" != "no" ] && [
 Main PID: 1650 (code=exited, status=1/FAILURE)

Sep 18 07:52:12 syzkaller ifup[1650]: exiting.
Sep 18 07:52:12 syzkaller ifup[1650]: ifup: failed to bring up eth0
Sep 18 07:52:12 syzkaller systemd[1]: networking.service: Child 1650 belongs to
Sep 18 07:52:12 syzkaller systemd[1]: networking.service: Main process exited, c
Sep 18 07:52:12 syzkaller systemd[1]: networking.service: Changed start -> faile
Sep 18 07:52:12 syzkaller systemd[1]: networking.service: Job networking.service
Sep 18 07:52:12 syzkaller systemd[1]: Failed to start Raise network interfaces.
Sep 18 07:52:12 syzkaller systemd[1]: networking.service: Unit entered failed st
Sep 18 07:52:12 syzkaller systemd[1]: networking.service: Failed with result ex
Sep 18 07:52:12 syzkaller systemd[1]: networking.service: cgroup is empty
```

`/etc/network/interfaces`에 등록된 인터페이스와 `ifconfig -a`로 나타나는 인터페이스 이름이 달라서 나는 오류였다.

**ifconfig -a**
```sh
root@syzkaller:~# ifconfig -a
enp0s3: flags=4098<BROADCAST,MULTICAST>  mtu 1500
        ether 52:54:00:12:34:56  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 2244  bytes 219888 (214.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2244  bytes 219888 (214.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

sit0: flags=128<NOARP>  mtu 1480
        sit  txqueuelen 1000  (IPv6-in-IPv4)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

**/etc/network/interfaces**
```sh
root@syzkaller:~# cat /etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

auto eth0
iface eth0 inet dhcp
```

`/etc/udev/rules.d/10-rename-network.rules` 경로에 아래 내용을 추가하여 오류를 고쳤다.
맥 주소는 `ifconfig -a`에서 출력되는 맥 주소를 입력해야 한다.

**/etc/udev/rules.d/10-rename-network.rules**
```sh
root@syzkaller:~# cat /etc/udev/rules.d/10-rename-network.rules
SUBSYSTEM=="net", ACTION=="add", ATTR{address}="52:54:00:12:34:56", NAME="eth0"
```

# References
<https://lux.cuenet.kr/105>
