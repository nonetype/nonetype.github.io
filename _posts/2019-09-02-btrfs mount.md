---
layout: post
title: "btrfs FileSystem mount analysis"
author: "nonetype"
categories: dev
tags: linux_kernel
---



btrfs 드라이버 내 mount 함수 분석기


---

# 목차

1. TOC
{:toc}

---


>본 문서는 LKL(Linux Kernel Library)의 Linux Kernel의 syscall 선언과 동작 방식을 분석합니다. 세부적인 내용은  github.com/torvalds/linux 와 다를 수 있습니다.

<!--
#User application에서 sys_mount syscall이 호출되었을 때 Flows
1. sys_mount가 호출 된 프로세스(User App-)이 sys_mount의 호출 권한이 있는가?
	호출 권한이 있다면(추가적인 검사도 있긴 함) do_mount 함수 호출
2. do_mount 함수 내에서 인자로 받아온 flag를 체크 후 각 flag에 따른 mount를 실행(MS_REMOUNT, MS_MOVE 등의 플래그가 존재 / 해당 플래그에 맞는 mount 함수 호출)
3. do_new_mount 함수의 경우 File system type을 체크(register된 파일 시스템 중 입력한 파일 시스템 타입에 맞는 구조체 주소를 반환)한 후, vfs_kern_mount 함수 호출
4. Vfs_kern_mount 함수 내에서는 file_system_type 구조체 내에 존재하는 파일 시스템 모듈별 초기화 함수 포인터를 호출하여 파일 시스템 모듈별로 다른 마운트 동작을 실행.
-->

# Analysis

sys_mount syscall부터 VFS를 타고 드디어. 마침내. btrfs 드라이버가 마운트 되는 과정까지 왔다.
이 문서에서는 btrfs 파일 시스템이 어떻게 커널에 마운트되는지 확인해 보자!

우선, `super.c`의 mount_fs 함수에서 호출되는 btrfs_mount 함수부터 살펴본다.

``` c
/*
 * Mount function which is called by VFS layer.
 *
 * In order to allow mounting a subvolume directly, btrfs uses mount_subtree()
 * which needs vfsmount* of device's root (/).  This means device's root has to
 * be mounted internally in any case.
 *
 * Operation flow:
 *   1. Parse subvol id related options for later use in mount_subvol().
 *
 *   2. Mount device's root (/) by calling vfs_kern_mount().
 *
 *      NOTE: vfs_kern_mount() is used by VFS to call btrfs_mount() in the
 *      first place. In order to avoid calling btrfs_mount() again, we use
 *      different file_system_type which is not registered to VFS by
 *      register_filesystem() (btrfs_root_fs_type). As a result,
 *      btrfs_mount_root() is called. The return value will be used by
 *      mount_subtree() in mount_subvol().
 *
 *   3. Call mount_subvol() to get the dentry of subvolume. Since there is
 *      "btrfs subvolume set-default", mount_subvol() is called always.
 */
static struct dentry *btrfs_mount(struct file_system_type *fs_type, int flags,
		const char *device_name, void *data)
{
	struct vfsmount *mnt_root;
	struct dentry *root;
	fmode_t mode = FMODE_READ;
	char *subvol_name = NULL;
	u64 subvol_objectid = 0;
	int error = 0;

	if (!(flags & SB_RDONLY))
		mode |= FMODE_WRITE;

	error = btrfs_parse_subvol_options(data, mode,
					  &subvol_name, &subvol_objectid);
	if (error) {
		kfree(subvol_name);
		return ERR_PTR(error);
	}

	/* mount device's root (/) */
	mnt_root = vfs_kern_mount(&btrfs_root_fs_type, flags, device_name, data);
	if (PTR_ERR_OR_ZERO(mnt_root) == -EBUSY) {
		if (flags & SB_RDONLY) {
			mnt_root = vfs_kern_mount(&btrfs_root_fs_type,
				flags & ~SB_RDONLY, device_name, data);
		} else {
			mnt_root = vfs_kern_mount(&btrfs_root_fs_type,
				flags | SB_RDONLY, device_name, data);
			if (IS_ERR(mnt_root)) {
				root = ERR_CAST(mnt_root);
				goto out;
			}

			down_write(&mnt_root->mnt_sb->s_umount);
			error = btrfs_remount(mnt_root->mnt_sb, &flags, NULL);
			up_write(&mnt_root->mnt_sb->s_umount);
			if (error < 0) {
				root = ERR_PTR(error);
				mntput(mnt_root);
				goto out;
			}
		}
	}
	if (IS_ERR(mnt_root)) {
		root = ERR_CAST(mnt_root);
		goto out;
	}

	/* mount_subvol() will free subvol_name and mnt_root */
	root = mount_subvol(subvol_name, subvol_objectid, device_name, mnt_root);

out:
	return root;
}
```
우선 주석을 확인해 보면,
> 1. 나중에 mount_subvol() 함수에서 사용하기 위해 subvol id와 관련있는 옵션을 파싱한다.
> 2. vfs_kern_mount() 를 통해 디바이스의 루트(/)를 마운트 한다.
> [+] `sys_mount()`가 호출되면서 `vfs_kern_mount()` 내에서 `btrfs_mount()`가 호출되었으므로 file_system_type인자로 VFS에 등록된 `btrfs_fs_type`을 준다면 무한 재귀가 발생할 수 있으므로, file_system_type 인자로 VFS에 등록되지 않은 file_system_type인 `btrfs_root_fs_type`을 인자로 준다.
이로 인해 `vfs_kern_mount()`내에서는 `btrfs_mount()`가 재호출 되어 무한 재귀가 발생하는 것이 아니라, `btrfs_mount_root`가 호출된다.
반환 값은 `mount_subvol()` 내의 `mount_subtree()`에서 사용된다.
> 3. subvolume의 dentry를 가져오기 위해 `mount_subvol()`을 호출한다. (`btrfs subvolume set-default`가 걸려있다면, 항상 호출된다.)

함수를 보면 주석의 내용과 동일하게 `btrfs_parse_subvol_options()`를 통해 subvol_objectid와 subvol_name을 가져오고, 이후 `vfs_kern_mount()` 함수를 통해 `btrfs_root_fs_type` 구조체의 `.mount` 포인터인 `btrfs_mount_root()` 함수를 호출한다.
```c
static struct file_system_type btrfs_root_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "btrfs",
	.mount		= btrfs_mount_root,
	.kill_sb	= btrfs_kill_super,
	.fs_flags	= FS_REQUIRES_DEV | FS_BINARY_MOUNTDATA,
};
```

```c
/*
 * Find a superblock for the given device / mount point.
 *
 * Note: This is based on mount_bdev from fs/super.c with a few additions
 *       for multiple device setup.  Make sure to keep it in sync.
 */
static struct dentry *btrfs_mount_root(struct file_system_type *fs_type,
		int flags, const char *device_name, void *data)
{
	struct block_device *bdev = NULL;
	struct super_block *s;
	struct btrfs_fs_devices *fs_devices = NULL;
	struct btrfs_fs_info *fs_info = NULL;
	struct security_mnt_opts new_sec_opts;
	fmode_t mode = FMODE_READ;
	int error = 0;

	if (!(flags & SB_RDONLY))
		mode |= FMODE_WRITE;

	error = btrfs_parse_early_options(data, mode, fs_type,
					  &fs_devices);
	if (error) {
		return ERR_PTR(error);
	}

	security_init_mnt_opts(&new_sec_opts);
	if (data) {
		error = parse_security_options(data, &new_sec_opts);
		if (error)
			return ERR_PTR(error);
	}

	error = btrfs_scan_one_device(device_name, mode, fs_type, &fs_devices);
	if (error)
		goto error_sec_opts;

	/*
	 * Setup a dummy root and fs_info for test/set super.  This is because
	 * we don't actually fill this stuff out until open_ctree, but we need
	 * it for searching for existing supers, so this lets us do that and
	 * then open_ctree will properly initialize everything later.
	 */
	fs_info = kvzalloc(sizeof(struct btrfs_fs_info), GFP_KERNEL);
	if (!fs_info) {
		error = -ENOMEM;
		goto error_sec_opts;
	}

	fs_info->fs_devices = fs_devices;

	fs_info->super_copy = kzalloc(BTRFS_SUPER_INFO_SIZE, GFP_KERNEL);
	fs_info->super_for_commit = kzalloc(BTRFS_SUPER_INFO_SIZE, GFP_KERNEL);
	security_init_mnt_opts(&fs_info->security_opts);
	if (!fs_info->super_copy || !fs_info->super_for_commit) {
		error = -ENOMEM;
		goto error_fs_info;
	}

	error = btrfs_open_devices(fs_devices, mode, fs_type);
	if (error)
		goto error_fs_info;

	if (!(flags & SB_RDONLY) && fs_devices->rw_devices == 0) {
		error = -EACCES;
		goto error_close_devices;
	}

	bdev = fs_devices->latest_bdev;
	s = sget(fs_type, btrfs_test_super, btrfs_set_super, flags | SB_NOSEC,
		 fs_info);
	if (IS_ERR(s)) {
		error = PTR_ERR(s);
		goto error_close_devices;
	}

	if (s->s_root) {
		btrfs_close_devices(fs_devices);
		free_fs_info(fs_info);
		if ((flags ^ s->s_flags) & SB_RDONLY)
			error = -EBUSY;
	} else {
		snprintf(s->s_id, sizeof(s->s_id), "%pg", bdev);
		btrfs_sb(s)->bdev_holder = fs_type;
		error = btrfs_fill_super(s, fs_devices, data);
	}
	if (error) {
		deactivate_locked_super(s);
		goto error_sec_opts;
	}

	fs_info = btrfs_sb(s);
	error = setup_security_options(fs_info, s, &new_sec_opts);
	if (error) {
		deactivate_locked_super(s);
		goto error_sec_opts;
	}

	return dget(s->s_root);

error_close_devices:
	btrfs_close_devices(fs_devices);
error_fs_info:
	free_fs_info(fs_info);
error_sec_opts:
	security_free_mnt_opts(&new_sec_opts);
	return ERR_PTR(error);
}
```
`btrfs_parse_early_options()`는 함수 declaration 위치에 주석으로 `"마운트 작업 초기에 필요한 마운트 옵션을 가져온다. 다른 옵션들은 마운트 작업 이후나 필요할 때에 SuperBlock에 할당될 것이다."`라고 되어있다.

이후 `sget(fs_type, btrfs_test_super, btrfs_set_super, flags | SB_NOSEC,
		 fs_info);`를 통해 superblock을 가져오게 되고, `btrfs_fill_super(s, fs_devices, data);`를 호출하게 된다.

```c { .line-numbers }
static int btrfs_fill_super(struct super_block *sb,
			    struct btrfs_fs_devices *fs_devices,
			    void *data)
{
	struct inode *inode;
	struct btrfs_fs_info *fs_info = btrfs_sb(sb);
	struct btrfs_key key;
	int err;

	sb->s_maxbytes = MAX_LFS_FILESIZE;
	sb->s_magic = BTRFS_SUPER_MAGIC;
	sb->s_op = &btrfs_super_ops;
	sb->s_d_op = &btrfs_dentry_operations;
	sb->s_export_op = &btrfs_export_ops;
	sb->s_xattr = btrfs_xattr_handlers;
	sb->s_time_gran = 1;
#ifdef CONFIG_BTRFS_FS_POSIX_ACL
	sb->s_flags |= SB_POSIXACL;
#endif
	sb->s_flags |= SB_I_VERSION;
	sb->s_iflags |= SB_I_CGROUPWB;

	err = super_setup_bdi(sb);
	if (err) {
		btrfs_err(fs_info, "super_setup_bdi failed");
		return err;
	}

	err = open_ctree(sb, fs_devices, (char *)data);
	if (err) {
		btrfs_err(fs_info, "open_ctree failed");
		return err;
	}

	key.objectid = BTRFS_FIRST_FREE_OBJECTID;
	key.type = BTRFS_INODE_ITEM_KEY;
	key.offset = 0;
	inode = btrfs_iget(sb, &key, fs_info->fs_root, NULL);
	if (IS_ERR(inode)) {
		err = PTR_ERR(inode);
		goto fail_close;
	}

	sb->s_root = d_make_root(inode);
	if (!sb->s_root) {
		err = -ENOMEM;
		goto fail_close;
	}

	cleancache_init_fs(sb);
	sb->s_flags |= SB_ACTIVE;
	return 0;

fail_close:
	close_ctree(fs_info);
	return err;
}
```


`include/uapi/linux/magic.h:26:#define BTRFS_SUPER_MAGIC	0x9123683E`

btrfs_fill_super에서는 superblock 내부의 값과 플래그를 넣어주고, `disk-io.c` 파일에 존재하는 `open_ctree()` 함수를 호출한다.

```c { .line-numbers }
int open_ctree(struct super_block *sb,
	       struct btrfs_fs_devices *fs_devices,
	       char *options)
{
	u32 sectorsize;
	u32 nodesize;
	u32 stripesize;
	u64 generation;
	u64 features;
	struct btrfs_key location;
	struct buffer_head *bh;
	struct btrfs_super_block *disk_super;
	struct btrfs_fs_info *fs_info = btrfs_sb(sb);
	struct btrfs_root *tree_root;
	struct btrfs_root *chunk_root;
	int ret;
	int err = -EINVAL;
	int num_backups_tried = 0;
	int backup_index = 0;
	int clear_free_space_tree = 0;
	int level;

	tree_root = fs_info->tree_root = btrfs_alloc_root(fs_info, GFP_KERNEL);
	chunk_root = fs_info->chunk_root = btrfs_alloc_root(fs_info, GFP_KERNEL);
	if (!tree_root || !chunk_root) {
		err = -ENOMEM;
		goto fail;
	}

	ret = init_srcu_struct(&fs_info->subvol_srcu);
	if (ret) {
		err = ret;
		goto fail;
	}

	ret = percpu_counter_init(&fs_info->dirty_metadata_bytes, 0, GFP_KERNEL);
	if (ret) {
		err = ret;
		goto fail_srcu;
	}
	fs_info->dirty_metadata_batch = PAGE_SIZE *
					(1 + ilog2(nr_cpu_ids));

	ret = percpu_counter_init(&fs_info->delalloc_bytes, 0, GFP_KERNEL);
	if (ret) {
		err = ret;
		goto fail_dirty_metadata_bytes;
	}

	ret = percpu_counter_init(&fs_info->bio_counter, 0, GFP_KERNEL);
	if (ret) {
		err = ret;
		goto fail_delalloc_bytes;
	}

	INIT_RADIX_TREE(&fs_info->fs_roots_radix, GFP_ATOMIC);
	INIT_RADIX_TREE(&fs_info->buffer_radix, GFP_ATOMIC);
	INIT_LIST_HEAD(&fs_info->trans_list);
	INIT_LIST_HEAD(&fs_info->dead_roots);
	INIT_LIST_HEAD(&fs_info->delayed_iputs);
	INIT_LIST_HEAD(&fs_info->delalloc_roots);
	INIT_LIST_HEAD(&fs_info->caching_block_groups);
	INIT_LIST_HEAD(&fs_info->pending_raid_kobjs);
	spin_lock_init(&fs_info->pending_raid_kobjs_lock);
	spin_lock_init(&fs_info->delalloc_root_lock);
	spin_lock_init(&fs_info->trans_lock);
	spin_lock_init(&fs_info->fs_roots_radix_lock);
	spin_lock_init(&fs_info->delayed_iput_lock);
	spin_lock_init(&fs_info->defrag_inodes_lock);
	spin_lock_init(&fs_info->tree_mod_seq_lock);
	spin_lock_init(&fs_info->super_lock);
	spin_lock_init(&fs_info->qgroup_op_lock);
	spin_lock_init(&fs_info->buffer_lock);
	spin_lock_init(&fs_info->unused_bgs_lock);
	rwlock_init(&fs_info->tree_mod_log_lock);
	mutex_init(&fs_info->unused_bg_unpin_mutex);
	mutex_init(&fs_info->delete_unused_bgs_mutex);
	mutex_init(&fs_info->reloc_mutex);
	mutex_init(&fs_info->delalloc_root_mutex);
	mutex_init(&fs_info->cleaner_delayed_iput_mutex);
	seqlock_init(&fs_info->profiles_lock);

	INIT_LIST_HEAD(&fs_info->dirty_cowonly_roots);
	INIT_LIST_HEAD(&fs_info->space_info);
	INIT_LIST_HEAD(&fs_info->tree_mod_seq_list);
	INIT_LIST_HEAD(&fs_info->unused_bgs);
	btrfs_mapping_init(&fs_info->mapping_tree);
	btrfs_init_block_rsv(&fs_info->global_block_rsv,
			     BTRFS_BLOCK_RSV_GLOBAL);
	btrfs_init_block_rsv(&fs_info->trans_block_rsv, BTRFS_BLOCK_RSV_TRANS);
	btrfs_init_block_rsv(&fs_info->chunk_block_rsv, BTRFS_BLOCK_RSV_CHUNK);
	btrfs_init_block_rsv(&fs_info->empty_block_rsv, BTRFS_BLOCK_RSV_EMPTY);
	btrfs_init_block_rsv(&fs_info->delayed_block_rsv,
			     BTRFS_BLOCK_RSV_DELOPS);
	atomic_set(&fs_info->async_delalloc_pages, 0);
	atomic_set(&fs_info->defrag_running, 0);
	atomic_set(&fs_info->qgroup_op_seq, 0);
	atomic_set(&fs_info->reada_works_cnt, 0);
	atomic64_set(&fs_info->tree_mod_seq, 0);
	fs_info->sb = sb;
	fs_info->max_inline = BTRFS_DEFAULT_MAX_INLINE;
	fs_info->metadata_ratio = 0;
	fs_info->defrag_inodes = RB_ROOT;
	atomic64_set(&fs_info->free_chunk_space, 0);
	fs_info->tree_mod_log = RB_ROOT;
	fs_info->commit_interval = BTRFS_DEFAULT_COMMIT_INTERVAL;
	fs_info->avg_delayed_ref_runtime = NSEC_PER_SEC >> 6; /* div by 64 */
	/* readahead state */
	INIT_RADIX_TREE(&fs_info->reada_tree, GFP_NOFS & ~__GFP_DIRECT_RECLAIM);
	spin_lock_init(&fs_info->reada_lock);
	btrfs_init_ref_verify(fs_info);

	fs_info->thread_pool_size = min_t(unsigned long,
					  num_online_cpus() + 2, 8);

	INIT_LIST_HEAD(&fs_info->ordered_roots);
	spin_lock_init(&fs_info->ordered_root_lock);

	fs_info->btree_inode = new_inode(sb);
	if (!fs_info->btree_inode) {
		err = -ENOMEM;
		goto fail_bio_counter;
	}

	...
```
약 800줄 가량의 긴 코드로 이루어진 함수여서 부분만 떼어왔다.
`open_ctree()` 함수에서는 filesystem info 구조체의 값들을 세팅해 준다.

```c {.line-numbers}
struct btrfs_fs_info {
	u8 fsid[BTRFS_FSID_SIZE];
	u8 chunk_tree_uuid[BTRFS_UUID_SIZE];
	unsigned long flags;
	struct btrfs_root *extent_root;
	struct btrfs_root *tree_root;
	struct btrfs_root *chunk_root;
	struct btrfs_root *dev_root;
	struct btrfs_root *fs_root;
	struct btrfs_root *csum_root;
	struct btrfs_root *quota_root;
	struct btrfs_root *uuid_root;
	struct btrfs_root *free_space_root;

	/* the log root tree is a directory of all the other log roots */
	struct btrfs_root *log_root_tree;

	spinlock_t fs_roots_radix_lock;
	struct radix_tree_root fs_roots_radix;

	/* block group cache stuff */
	spinlock_t block_group_cache_lock;
	u64 first_logical_byte;
	struct rb_root block_group_cache_tree;

	/* keep track of unallocated space */
	atomic64_t free_chunk_space;

	struct extent_io_tree freed_extents[2];
	struct extent_io_tree *pinned_extents;

	/* logical->physical extent mapping */
	struct btrfs_mapping_tree mapping_tree;

	/*
	 * block reservation for extent, checksum, root tree and
	 * delayed dir index item
	 */
	struct btrfs_block_rsv global_block_rsv;
	/* block reservation for metadata operations */
	struct btrfs_block_rsv trans_block_rsv;
	/* block reservation for chunk tree */
	struct btrfs_block_rsv chunk_block_rsv;
	/* block reservation for delayed operations */
	struct btrfs_block_rsv delayed_block_rsv;

	struct btrfs_block_rsv empty_block_rsv;

	u64 generation;
	u64 last_trans_committed;
	u64 avg_delayed_ref_runtime;

	...
```

######최종적으로 마운트 과정을 정리하자면,
`mount_fs(super.c) => btrfs_mount(btrfs/super.c) => vfs_kern_mount(namespace.c) => mount_fs(super.c) => btrfs_mount_root(btrfs/super.c) => btrfs_fill_super(btrfs/super.c) => open_ctree(btrfs/disk-io.c)`이다.
1. `mount_fs()`: 파일 시스템 드라이버(모듈)별 Mount 함수 호출 (btrfs, ext4 등등)
2. `btrfs_mount()`: 서브볼륨 id/name 파싱, `vfs_kern_mount(vtrfs_mount_root)` 호출, 서브볼륨 dentry 로드
3. `vfs_kern_mount()`: btrfs_mount_root 호출
4. `btrfs_mount_root()`: 당장 필요한 옵션만 파싱(Frame Build), 더미 root, 더미 fs_info 구조체 할당, device 체크, readOnly 체크 후 `btrfs_fill_super()` 호출
5. `btrfs_fill_super()`: SuperBlock 내 값을 채우고, INode를 받아와 SuperBlock의 Root INode로 지정
6. `open_ctree()`: fs_info 구조체 내 필드 값 할당


***
# 참고
소스 파일별 용도
```
ctree.c: 	Core btree manipulation code
ctree.h: 	Defines keys and most of the data structures used for metadata
dir-item.c: 	Helpers to create and use directory items
disk-io.c: 	Metadata IO operations as well as FS open/close routines
extent-tree.c: 	Tracks and space used by btree blocks and file data extents
extent_io.c: 	Keeps track of state (locked, writeback etc) and implements extent_buffers
extent_map.c: 	Maps logical offset in a file to blocks on logical address on disk
file-item.c: 	Helpers to insert and remove file extents and data checksums
file.c: 	File write routines (kernel only)
inode-item.c: 	Helpers to allocate inode structures in the btree
inode.c: 	Most file and directory operations for the kernel
ordered-data.c:	Maintains lists of inodes for data=ordered
print-tree.c: 	Walks a btree and prints the items it finds
root-tree.c: 	Helpers to manage the items in the tree of tree roots
struct-funcs.c:	Macro magic to instantiate all of the metadata set/get functions
super.c: 	Kernel super block and related functions
transaction.c: 	Handles transaction commits
volumes.c: 	All of the multi-device aware code
tree-log.c: 	Handles logging of tree items for fast fsync
compression.c: 	zlib compression support routines
```


# References-
https://btrfs.wiki.kernel.org/index.php/Code_documentation

https://www.potatogim.net/wiki/Btrfs/%EC%BB%A4%EB%84%90_%EC%BD%94%EB%93%9C_%EB%B6%84%EC%84%9D

https://libsora.so/posts/system-programming-linux-file-system/

https://dri.freedesktop.org/docs/drm/filesystems/vfs.html

https://jiming.tistory.com/359

https://www.quora.com/What-happens-inside-Linux-Kernel-when-you-call-mount-command

https://kukuruku.co/post/writing-a-file-system-in-linux-kernel/
