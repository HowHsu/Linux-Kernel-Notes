# dentry && inode

An essencial question about filesystem is: what happens when we open a file. To
figure it out, we have to first understand two concepts: dentry and inode. But
this article is not just about this two concepts, it aims to show the big picture
of filesystem from this two.

## ondisk data layout

Let's first go over the data layout of filesystem on the storage. Here I pick up
ext4 as an example.

![](./images/ext4_layout.png)

ext4 divides the disk partition(a disk can be splited to several partitions, each one with a filesystem instance) to many block groups.(the boot sector only exists in the first partition if that disk is where the operation system installed).
Let's look into a block group briefly.
 * superblock contains info about the whole filesystem instance.
 * Group Descriptor Table contains info about this block group.
 * block bitmap indicates the state of each block in this group. Ok, one thing forgot to say, a group is divided to many blocks (that's why it is called block device).
 * inode bitmap is like block bitmap, it indicates which inode is in use
 which else is not.
 * inode table contains all the inodes in this group. An inode represents a
 file.
 * others are data blocks.

Here let's focus on inode, this is the key to a file. Below is the layout of ondisk
inode structure.

```c
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
	__le32	i_flags;	/* File flags */
	union {
		struct {
			__le32  l_i_version;
		} linux1;
		struct {
			__u32  h_i_translator;
		} hurd1;
		struct {
			__u32  m_i_reserved1;
		} masix1;
	} osd1;				/* OS dependent 1 */
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high;
	__le32	i_obso_faddr;	/* Obsoleted fragment address */
	union {
		struct {
			__le16	l_i_blocks_high; /* were l_i_reserved1 */
			__le16	l_i_file_acl_high;
			__le16	l_i_uid_high;	/* these 2 fields */
			__le16	l_i_gid_high;	/* were reserved2[0] */
			__le16	l_i_checksum_lo;/* crc32c(uuid+inum+inode) LE */
			__le16	l_i_reserved;
		} linux2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__u16	h_i_mode_high;
			__u16	h_i_uid_high;
			__u16	h_i_gid_high;
			__u32	h_i_author;
		} hurd2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__le16	m_i_file_acl_high;
			__u32	m_i_reserved2[2];
		} masix2;
	} osd2;				/* OS dependent 2 */
	__le16	i_extra_isize;
	__le16	i_checksum_hi;	/* crc32c(uuid+inum+inode) BE */
	__le32  i_ctime_extra;  /* extra Change time      (nsec << 2 | epoch) */
	__le32  i_mtime_extra;  /* extra Modification time(nsec << 2 | epoch) */
	__le32  i_atime_extra;  /* extra Access time      (nsec << 2 | epoch) */
	__le32  i_crtime;       /* File Creation time */
	__le32  i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
	__le32  i_version_hi;	/* high 32 bits for 64-bit version */
	__le32	i_projid;	/* Project ID */
}
```

It's well commented, one member we should focus on is `__le32	i_block[EXT4_N_BLOCKS];`
This array points to data blocks of this file.

But how? EXT4_N_BLOCKS is 15, say a block is 4KB, so a max size of a file is 15*4KB=60KB?
Surely no, in the legacy way, the first 12 items point to a block each, while the last 3
are indirect pointers. This means they point to blocks which contain pointers.

![](./images/indirect_inode.png)

There is a little bit difference between the last 3 items(indirect pointers).
12nd can only have one layer of indirect block. 13th can have two layers as you can see in the picture. 14th can have three layers.
This significantly expand the size of file it supports. Let's do some simple
math, say a block is 4KB, then a block contains 4KB / 4B = 2^10 items. For
the i_block[14], there are 1\*2^10\*2^10\*2^10 = 2^30 blocks. So i_block[14] supports 2^30\*4KB = 4TB. Yes, ext4 supports a single file large as 4TB...in this way.
Just like what I said, the above is called the legacy way. Modern ext4 filesystem these days use a new way to index file blocks----extent.
Extent deserves a detail analysis in a seperate article.
[ext4 extent](./extent.md)



