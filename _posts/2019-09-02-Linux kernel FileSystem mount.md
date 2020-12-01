---
layout: post
title: "Linux Kernel sys_mount analysis"
categories: dev
tags: linux_kernel
---



linux kernel sys_mount 분석기

---

# 목차

1. TOC
{:toc}

---

# VFS(Virtual File System)
  VFS는 User Mode 프로세스에 동일한 인터페이스를 제공하고, 다양한 파일 시스템을 효과적으로 구현하기 위한 '표준 유닉스 파일시스템과 관련된 모든 System Call을 수행하는 Kernel Layer'이다.

  VFS는 각 파일, 파일 시스템과 관련된 시스템 콜을 처리하며, 파일 시스템에 관련된 자료구조 관리, 파일 시스템을 순회하는 효율적인 동작 및 특정 파일 시스템 모듈과 상호작용한다.

  예를 들어, 마운트 된 파일 시스템이 여러 종류일지라도 `cp /path/to/btrfs/file /path/to/ext4/file` 명령을 실행해도 VFS가 시스템 콜에 대한 처리를 해주어 copy 명령이 정상적으로 동작한다.




![vfs_structure](https://upload.wikimedia.org/wikipedia/commons/thumb/3/30/IO_stack_of_the_Linux_kernel.svg/440px-IO_stack_of_the_Linux_kernel.svg.png)


# sys_mount() 분석
커널에서 sys_mount syscall이 호출될 경우, `fs/namespace.c`의 `int ksys_mount` 함수가 호출된다.

```c
int ksys_mount(char __user *dev_name, char __user *dir_name, char __user *type,
               unsigned long flags, void __user *data)
{
        int ret;
        char *kernel_type;
        char *kernel_dev;
        void *options;

        kernel_type = copy_mount_string(type);
        ret = PTR_ERR(kernel_type);
        if (IS_ERR(kernel_type))
                goto out_type;

        kernel_dev = copy_mount_string(dev_name);
        ret = PTR_ERR(kernel_dev);
        if (IS_ERR(kernel_dev))
                goto out_dev;

        options = copy_mount_options(data);
        ret = PTR_ERR(options);
        if (IS_ERR(options))
                goto out_data;

        ret = do_mount(kernel_dev, dir_name, kernel_type, flags, options);

        kfree(options);
out_data:
        kfree(kernel_dev);
out_dev:
        kfree(kernel_type);
out_type:
        return ret;
}
```

이후 ksys_mount 함수 내에서 do_mount 함수가 실행된다.
(flag 값과 MAGIC_VALUE값으로 연산하는 부분이 존재한다. (추가 분석 예정))
```c
#define MS_MGC_VAL 0xC0ED0000
```

```c
/*
 * Flags is a 32-bit value that allows up to 31 non-fs dependent flags to
 * be given to the mount() call (ie: read-only, no-dev, no-suid etc).
 *
 * data is a (void *) that can point to any structure up to
 * PAGE_SIZE-1 bytes, which can contain arbitrary fs-dependent
 * information (or be NULL).
 *
 * Pre-0.97 versions of mount() didn't have a flags word.
 * When the flags word was introduced its top half was required
 * to have the magic value 0xC0ED, and this remained so until 2.4.0-test9.
 * Therefore, if this magic number is present, it carries no information
 * and must be discarded.
 */
long do_mount(const char *dev_name, const char __user *dir_name,
                const char *type_page, unsigned long flags, void *data_page)
{
        struct path path;
        unsigned int mnt_flags = 0, sb_flags;
        int retval = 0;

        /* Discard magic */
        if ((flags & MS_MGC_MSK) == MS_MGC_VAL)
                flags &= ~MS_MGC_MSK;

        /* Basic sanity checks */
        if (data_page)
                ((char *)data_page)[PAGE_SIZE - 1] = 0;

        if (flags & MS_NOUSER)
                return -EINVAL;

        /* ... and get the mountpoint */
        retval = user_path(dir_name, &path);
        if (retval)
                return retval;

        retval = security_sb_mount(dev_name, &path,
                                   type_page, flags, data_page);
        if (!retval && !may_mount())
                retval = -EPERM;
        if (!retval && (flags & SB_MANDLOCK) && !may_mandlock())
                retval = -EPERM;
        if (retval)
                goto dput_out;

        /* Default to relatime unless overriden */
        if (!(flags & MS_NOATIME))
                mnt_flags |= MNT_RELATIME;

        /* Separate the per-mountpoint flags */
        if (flags & MS_NOSUID)
                mnt_flags |= MNT_NOSUID;
        if (flags & MS_NODEV)
                mnt_flags |= MNT_NODEV;
        if (flags & MS_NOEXEC)
                mnt_flags |= MNT_NOEXEC;
        if (flags & MS_NOATIME)
                mnt_flags |= MNT_NOATIME;
        if (flags & MS_NODIRATIME)
                mnt_flags |= MNT_NODIRATIME;
        if (flags & MS_STRICTATIME)
                mnt_flags &= ~(MNT_RELATIME | MNT_NOATIME);
        if (flags & MS_RDONLY)
                mnt_flags |= MNT_READONLY;

        /* The default atime for remount is preservation */
        if ((flags & MS_REMOUNT) &&
            ((flags & (MS_NOATIME | MS_NODIRATIME | MS_RELATIME |
                       MS_STRICTATIME)) == 0)) {
                mnt_flags &= ~MNT_ATIME_MASK;
                mnt_flags |= path.mnt->mnt_flags & MNT_ATIME_MASK;
        }

        sb_flags = flags & (SB_RDONLY |
                            SB_SYNCHRONOUS |
                            SB_MANDLOCK |
                            SB_DIRSYNC |
                            SB_SILENT |
                            SB_POSIXACL |
                            SB_LAZYTIME |
                            SB_I_VERSION);

        if (flags & MS_REMOUNT)
                retval = do_remount(&path, flags, sb_flags, mnt_flags,
                                    data_page);
        else if (flags & MS_BIND)
                retval = do_loopback(&path, dev_name, flags & MS_REC);
        else if (flags & (MS_SHARED | MS_PRIVATE | MS_SLAVE | MS_UNBINDABLE))
                retval = do_change_type(&path, flags);
        else if (flags & MS_MOVE)
                retval = do_move_mount(&path, dev_name);
        else
                retval = do_new_mount(&path, type_page, sb_flags, mnt_flags,
                                      dev_name, data_page);
dput_out:
        path_put(&path);
        return retval;
}
```

```c
/*
 * create a new mount for userspace and request it to be added into the
 * namespace's tree
 */
static int do_new_mount(struct path *path, const char *fstype, int sb_flags,
			int mnt_flags, const char *name, void *data)
{
	struct file_system_type *type;
	struct vfsmount *mnt;
	int err;

	if (!fstype)
		return -EINVAL;

	type = get_fs_type(fstype);
	if (!type)
		return -ENODEV;

	mnt = vfs_kern_mount(type, sb_flags, name, data);
	if (!IS_ERR(mnt) && (type->fs_flags & FS_HAS_SUBTYPE) &&
	    !mnt->mnt_sb->s_subtype)
		mnt = fs_set_subtype(mnt, fstype);

	put_filesystem(type);
	if (IS_ERR(mnt))
		return PTR_ERR(mnt);

	if (mount_too_revealing(mnt, &mnt_flags)) {
		mntput(mnt);
		return -EPERM;
	}

	err = do_add_mount(real_mount(mnt), path, mnt_flags);
	if (err)
		mntput(mnt);
	return err;
}
```
do mount 함수에서는 `get_fs_type()` 함수를 통해 파일 시스템의 타입을 구해온 후, 파일 시스템 타입에 맞는 드라이버 모듈의 mount 함수를 호출하기 위해 `vfs_kern_mount()` 함수를 호출한다.
```c
struct file_system_type *get_fs_type(const char *name)
{
        struct file_system_type *fs;
        const char *dot = strchr(name, '.');
        int len = dot ? dot - name : strlen(name);

        fs = __get_fs_type(name, len);
        if (!fs && (request_module("fs-%.*s", len, name) == 0)) {
                fs = __get_fs_type(name, len);
                WARN_ONCE(!fs, "request_module fs-%.*s succeeded, but still no fs?\n", len, name);
        }

        if (dot && fs && !(fs->fs_flags & FS_HAS_SUBTYPE)) {
                put_filesystem(fs);
                fs = NULL;
        }
        return fs;
}
```

```c
struct vfsmount *
vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
{
	struct mount *mnt;
	struct dentry *root;

	if (!type)
		return ERR_PTR(-ENODEV);

	mnt = alloc_vfsmnt(name);
	if (!mnt)
		return ERR_PTR(-ENOMEM);

	if (flags & SB_KERNMOUNT)
		mnt->mnt.mnt_flags = MNT_INTERNAL;

	root = mount_fs(type, flags, name, data);
	if (IS_ERR(root)) {
		mnt_free_id(mnt);
		free_vfsmnt(mnt);
		return ERR_CAST(root);
	}

	mnt->mnt.mnt_root = root;
	mnt->mnt.mnt_sb = root->d_sb;
	mnt->mnt_mountpoint = mnt->mnt.mnt_root;
	mnt->mnt_parent = mnt;
	lock_mount_hash();
	list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);
	unlock_mount_hash();
	return &mnt->mnt;
}
```

그리고 `vfs_kern_mount()` 함수에서 `mount_fs()` 함수를 호출한다.

```c

struct dentry *
mount_fs(struct file_system_type *type, int flags, const char *name, void *data)
{
	struct dentry *root;
	struct super_block *sb;
	char *secdata = NULL;
	int error = -ENOMEM;

	if (data && !(type->fs_flags & FS_BINARY_MOUNTDATA)) {
		secdata = alloc_secdata();
		if (!secdata)
			goto out;

		error = security_sb_copy_data(data, secdata);
		if (error)
			goto out_free_secdata;
	}

	root = type->mount(type, flags, name, data);
	if (IS_ERR(root)) {
		error = PTR_ERR(root);
		goto out_free_secdata;
	}
	sb = root->d_sb;
	BUG_ON(!sb);
	WARN_ON(!sb->s_bdi);

	/*
	 * Write barrier is for super_cache_count(). We place it before setting
	 * SB_BORN as the data dependency between the two functions is the
	 * superblock structure contents that we just set up, not the SB_BORN
	 * flag.
	 */
	smp_wmb();
	sb->s_flags |= SB_BORN;

	error = security_sb_kern_mount(sb, flags, secdata);
	if (error)
		goto out_sb;

	/*
	 * filesystems should never set s_maxbytes larger than MAX_LFS_FILESIZE
	 * but s_maxbytes was an unsigned long long for many releases. Throw
	 * this warning for a little while to try and catch filesystems that
	 * violate this rule.
	 */
	WARN((sb->s_maxbytes < 0), "%s set sb->s_maxbytes to "
		"negative value (%lld)\n", type->name, sb->s_maxbytes);

	up_write(&sb->s_umount);
	free_secdata(secdata);
	return root;
out_sb:
	dput(root);
	deactivate_locked_super(sb);
out_free_secdata:
	free_secdata(secdata);
out:
	return ERR_PTR(error);
}
```
`mount_fs()`에서는 `type->mount()`를 통해 각 파일 시스템 드라이버에 선언되어있는 함수 포인터를 호출함으로써 각 드라이버별 마운트 과정을 시작한다.

***

# 기타 정보

get_sb(get SuperBlock) 함수
```c
static struct dentry *get_sb(struct file_system_type *fs_type,
                  int flags, const char *dev_name,
                  void *data)
{
        return mount_single(fs_type, flags, data, fill_super);
}
```

파일 시스템 판별 및 flags
file_system_type 구조체
```c
struct file_system_type {
    const char *name;
    int fs_flags;
    struct super_block *(*read_super) (struct super_block *, void *, int);
    struct file_system_type *next;
};
```

입력된 이름에 맞는 file_system_type 구조체를 찾을 경우, 반환해주는 함수:
```c
static struct file_system_type **find_filesystem(const char *name, unsigned len)
{
        struct file_system_type ** p;
        for (p = &file_systems; *p; p = &(*p)->next)
                if (strncmp((*p)->name, name, len) == 0 &&
                    !(*p)->name[len])
                        break;
        return p;
}
```

파일 시스템을 등록/등록 해제하는 함수
```c
/**
 *      register_filesystem - register a new filesystem
 *      @fs: the file system structure
 *
 *      Adds the file system passed to the list of file systems the kernel
 *      is aware of for mount and other syscalls. Returns 0 on success,
 *      or a negative errno code on an error.
 *
 *      The &struct file_system_type that is passed is linked into the kernel
 *      structures and must not be freed until the file system has been
 *      unregistered.
 */
int register_filesystem(struct file_system_type * fs)
{
        int res = 0;
        struct file_system_type ** p;

        BUG_ON(strchr(fs->name, '.'));
        if (fs->next)
                return -EBUSY;
        write_lock(&file_systems_lock);
        p = find_filesystem(fs->name, strlen(fs->name));
        if (*p)
                res = -EBUSY;
        else
                *p = fs;
        write_unlock(&file_systems_lock);
        return res;
}
```

```c
/**
 *      unregister_filesystem - unregister a file system
 *      @fs: filesystem to unregister
 *
 *      Remove a file system that was previously successfully registered
 *      with the kernel. An error is returned if the file system is not found.
 *      Zero is returned on a success.
 *
 *      Once this function has returned the &struct file_system_type structure
 *      may be freed or reused.
 */
int unregister_filesystem(struct file_system_type * fs)
{
        struct file_system_type ** tmp;

        write_lock(&file_systems_lock);
        tmp = &file_systems;
        while (*tmp) {
                if (fs == *tmp) {
                        *tmp = fs->next;
                        fs->next = NULL;
                        write_unlock(&file_systems_lock);
                        synchronize_rcu();
                        return 0;
                }
                tmp = &(*tmp)->next;
        }
        write_unlock(&file_systems_lock);

        return -EINVAL;
}
```


btrfs 파일 시스템의 예로, 아래와 같은 file_system_type 구조체를 .../fs/btrfs/super.c 파일 안에 포함하고 있다.
```c
static struct file_system_type btrfs_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "btrfs",
        .mount          = btrfs_mount,
        .kill_sb        = btrfs_kill_super,
        .fs_flags       = FS_REQUIRES_DEV | FS_BINARY_MOUNTDATA,
};
```

btrfs 파일 시스템 init 함수
```c
static int __init init_btrfs_fs(void)
{
        int err;

        btrfs_props_init();

        err = btrfs_init_sysfs();
        if (err)
                return err;

        btrfs_init_compress();

        err = btrfs_init_cachep();
        if (err)
                goto free_compress;

        err = extent_io_init();
        if (err)
                goto free_cachep;

        err = extent_map_init();
        if (err)
                goto free_extent_io;

        err = ordered_data_init();
        if (err)
                goto free_extent_map;

        err = btrfs_delayed_inode_init();
        if (err)
                goto free_ordered_data;

        err = btrfs_auto_defrag_init();
        if (err)
                goto free_delayed_inode;

        err = btrfs_delayed_ref_init();
        if (err)
                goto free_auto_defrag;

        err = btrfs_prelim_ref_init();
        if (err)
                goto free_delayed_ref;

        err = btrfs_end_io_wq_init();
        if (err)
                goto free_prelim_ref;

        err = btrfs_interface_init();
        if (err)
                goto free_end_io_wq;

        btrfs_init_lockdep();

        btrfs_print_mod_info();

        err = btrfs_run_sanity_tests();
        if (err)
                goto unregister_ioctl;

        err = register_filesystem(&btrfs_fs_type);
        if (err)
                goto unregister_ioctl;

        return 0;

unregister_ioctl:
        btrfs_interface_exit();
free_end_io_wq:
        btrfs_end_io_wq_exit();
free_prelim_ref:
        btrfs_prelim_ref_exit();
free_delayed_ref:
        btrfs_delayed_ref_exit();
free_auto_defrag:
        btrfs_auto_defrag_exit();
free_delayed_inode:
        btrfs_delayed_inode_exit();
free_ordered_data:
        ordered_data_exit();
free_extent_map:
        extent_map_exit();
free_extent_io:
        extent_io_exit();
free_cachep:
        btrfs_destroy_cachep();
free_compress:
        btrfs_exit_compress();
        btrfs_exit_sysfs();

        return err;
}
```

btrfs 파일 시스템 exit 함수
```c
static void __exit exit_btrfs_fs(void)
{
        btrfs_destroy_cachep();
        btrfs_delayed_ref_exit();
        btrfs_auto_defrag_exit();
        btrfs_delayed_inode_exit();
        btrfs_prelim_ref_exit();
        ordered_data_exit();
        extent_map_exit();
        extent_io_exit();
        btrfs_interface_exit();
        btrfs_end_io_wq_exit();
        unregister_filesystem(&btrfs_fs_type);
        btrfs_exit_sysfs();
        btrfs_cleanup_fs_uuids();
        btrfs_exit_compress();
}
```

***
# References.

https://en.wikipedia.org/wiki/Virtual_file_system

https://www.win.tue.nl/~aeb/linux/lk/lk-8.html

https://dri.freedesktop.org/docs/drm/filesystems/index.html

https://libsora.so/posts/system-programming-linux-file-system/
