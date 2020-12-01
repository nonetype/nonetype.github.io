---
layout: post
title: "Linux Kernel sys_write analysis"
author: "nonetype"
---

sys_write 분석기

#### sys_write() 분석

우선 write syscall은 `fs/read_write.c` 위치에 선언되어 있다.

```c
ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_write(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}

	return ret;
}

SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	return ksys_write(fd, buf, count);
}
```
우선, 인자값으로 전달된 fd에 할당된 파일을 일어오고, `vfs_write()` 함수를 호출한다.
이후 return값이 0 이상일 경우, `file_pos_write()` 함수를 호출한다.

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
	ssize_t ret;

	if (!(file->f_mode & FMODE_READ))
		return -EBADF;
	if (!(file->f_mode & FMODE_CAN_READ))
		return -EINVAL;
	if (unlikely(!access_ok(VERIFY_WRITE, buf, count)))
		return -EFAULT;

	ret = rw_verify_area(READ, file, pos, count);
	if (!ret) {
		if (count > MAX_RW_COUNT)
			count =  MAX_RW_COUNT;
		ret = __vfs_read(file, buf, count, pos);
		if (ret > 0) {
			fsnotify_access(file);
			add_rchar(current, ret);
		}
		inc_syscr(current);
	}

	return ret;
}
```
우선 file의 f_mode flag에 FMODE_READ와 FMODE_CAN_READ 비트가 존재하는지 체크한다.

<details>
<summary>FMODE Flag bits(클릭)</summary>
<div markdown="1">

```c
/*
 * flags in file.f_mode.  Note that FMODE_READ and FMODE_WRITE must correspond
 * to O_WRONLY and O_RDWR via the strange trick in __dentry_open()
 */

/* file is open for reading */
#define FMODE_READ		((__force fmode_t)0x1)
/* file is open for writing */
#define FMODE_WRITE		((__force fmode_t)0x2)
/* file is seekable */
#define FMODE_LSEEK		((__force fmode_t)0x4)
/* file can be accessed using pread */
#define FMODE_PREAD		((__force fmode_t)0x8)
/* file can be accessed using pwrite */
#define FMODE_PWRITE		((__force fmode_t)0x10)
/* File is opened for execution with sys_execve / sys_uselib */
#define FMODE_EXEC		((__force fmode_t)0x20)
/* File is opened with O_NDELAY (only set for block devices) */
#define FMODE_NDELAY		((__force fmode_t)0x40)
/* File is opened with O_EXCL (only set for block devices) */
#define FMODE_EXCL		((__force fmode_t)0x80)
/* File is opened using open(.., 3, ..) and is writeable only for ioctls
   (specialy hack for floppy.c) */
#define FMODE_WRITE_IOCTL	((__force fmode_t)0x100)
/* 32bit hashes as llseek() offset (for directories) */
#define FMODE_32BITHASH         ((__force fmode_t)0x200)
/* 64bit hashes as llseek() offset (for directories) */
#define FMODE_64BITHASH         ((__force fmode_t)0x400)

/*
 * Don't update ctime and mtime.
 *
 * Currently a special hack for the XFS open_by_handle ioctl, but we'll
 * hopefully graduate it to a proper O_CMTIME flag supported by open(2) soon.
 */
#define FMODE_NOCMTIME		((__force fmode_t)0x800)

/* Expect random access pattern */
#define FMODE_RANDOM		((__force fmode_t)0x1000)

/* File is huge (eg. /dev/kmem): treat loff_t as unsigned */
#define FMODE_UNSIGNED_OFFSET	((__force fmode_t)0x2000)

/* File is opened with O_PATH; almost nothing can be done with it */
#define FMODE_PATH		((__force fmode_t)0x4000)

/* File needs atomic accesses to f_pos */
#define FMODE_ATOMIC_POS	((__force fmode_t)0x8000)
/* Write access to underlying fs */
#define FMODE_WRITER		((__force fmode_t)0x10000)
/* Has read method(s) */
#define FMODE_CAN_READ          ((__force fmode_t)0x20000)
/* Has write method(s) */
#define FMODE_CAN_WRITE         ((__force fmode_t)0x40000)

/* File was opened by fanotify and shouldn't generate fanotify events */
#define FMODE_NONOTIFY		((__force fmode_t)0x4000000)
```

</div>
</details>

이후 access_ok 함수를 호출하는데, `include/asm-generic/uaccess.h` 파일에 존재하는 내용으로는 무조건 True를 반환한다.

```c
#define access_ok(type, addr, size) __access_ok((unsigned long)(addr),(size))

/*
 * The architecture should really override this if possible, at least
 * doing a check on the get_fs()
 */
#ifndef __access_ok
static inline int __access_ok(unsigned long addr, unsigned long size)
{
        return 1;
}
#endif
```


***

#### References

http://egloos.zum.com/rousalome/v/9992271 ([리눅스커널][가상파일시스템] 파일 객체란)
