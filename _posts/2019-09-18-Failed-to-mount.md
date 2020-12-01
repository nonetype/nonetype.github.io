---
layout: post
title: "[SOLVED] Failed to mount /sys/kernel/config"
author: "nonetype"
categories: dev
tags: vm
---

Qemu 부팅시 emergency mode로 진입해버리는 오류를 고쳐보자.

---

# 목차

1. TOC
{:toc}

---

# Failed to mount
qemu로 커널 부팅시 아래와 같은 에러가 나타난다.

```sh
[FAILED] Failed to mount /sys/kernel/config.
See 'systemctl status sys-kernel-config.mount' for details.
[DEPEND] Dependency failed for Local File Systems.
[DEPEND] Dependency failed for Mark the need to relabel after reboot.
[    5.017700] EXT4-fs (sda): re-mounted. Opts: (null)
[  OK  ] Started Load Kernel Modules.
[FAILED] Failed to start Remount Root and Kernel File Systems.
See 'systemctl status systemd-remount-fs.service' for details.
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Timers.
         Starting Apply Kernel Variables...
[  OK  ] Started Emergency Shell.
[    5.275078] systemd-journald[1073]: Fixed min_use=1.0M max_use=99.7M max_size=12.4M min_size=512.0K keep_free=149.5M n_max_files=100
[  OK  ] Reached target Emergency Mode.
[    5.311387] systemd-journald[1073]: Reserving 22691 entries in hash table.
         Starting udev Coldplug all Devices...
[    5.328541] systemd-journald[1073]: Vacuuming...
[    5.331022] systemd-journald[1073]: Vacuuming done, freed 0B of archived journals from /run/log/journal/8c666b40dbed4f8b80d2d53e4ee371c0.
[    5.344013] systemd-journald[1073]: Flushing /dev/kmsg...
         Starting Load/Save Random Seed...
[  OK  ] Closed Syslog Socket.
[  OK  ] Started Apply Kernel Variables.
[  OK  ] Started Load/Save Random Seed.
         Starting Raise network interfaces...
[  OK  ] Started Create Static Device Nodes in /dev.
         Starting udev Kernel Device Manager...
[  OK  ] Reached target Local File Systems (Pre).
[    6.076871] ifquery (1088) used greatest stack depth: 13440 bytes left
[  OK  ] Started Journal Service.
         Starting Flush Journal to Persistent Storage...
[  OK  ] Started Flush Journal to Persistent Storage.
         Starting Create Volatile Files and Directories...
[  OK  ] Started udev Kernel Device Manager.
[  OK  ] Started Create Volatile Files and Directories.
         Starting Update UTMP about System Boot/Shutdown...
         Starting Network Time Synchronization...
[  OK  ] Started udev Coldplug all Devices.
[  OK  ] Started Update UTMP about System Boot/Shutdown.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Network Time Synchronization.
[  OK  ] Started Update UTMP about System Runlevel Changes.
[  OK  ] Reached target System Time Synchronized.
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or ^D to
try again to boot into default mode.
```

검색을 통해 [Issue](https://github.com/google/syzkaller/issues/760)를 찾을 수 있었다.
해당 이슈 쓰레드에서 커널 컴파일시 `.config` 파일에서 `CONFIG_CONFIGFS_FS=y` `CONFIG_SECURITYFS=y`로 설정[^issue]하라고 한다.

grep으로 확인해보니 not set 상태여서 아래와 같이 수정해줬다.

```sh
nonetype@pwn:~/linux/linux-4.17$ cat .config | grep "CONFIG_CONFIGFS"
CONFIG_CONFIGFS_FS=y
nonetype@pwn:~/linux/linux-4.17$ cat .config | grep "CONFIG_SECURITYFS"
CONFIG_SECURITYFS=y
nonetype@pwn:~/linux/linux-4.17$
```

`make all -j4` 명령을 통해 다시 컴파일하면 해결된다.

# References
<https://github.com/google/syzkaller/issues/760>



[^issue]: <https://github.com/google/syzkaller/issues/760#issuecomment-441162769>
