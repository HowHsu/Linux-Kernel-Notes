{:toc}

# Extent Tree

Extent tree is a well used structure in filesystem, it is to replace the legacy
indirect mapping method to look for a physical block mapped by a logical block.
Here I'll take ext4 filesystem as an instance to begin the story. If you are
not familiar with the legacy indirect mapping method, you can refer to this
article: [dentry_inode.md](./dentry_inode.md)

## Overview

Firstly, let's begin with two questions, with the indirect mapping method:
 - how many extra blocks (for storing indirect blocks) we need for a 4TB file?
 - how many times do we do the logical block-->physical block translation when we
   write to a 4TB file.

Assume the block size is 4KB.
For the first one, we know the inode->i_block[14] support 3 layers of indirect
blocks. So the number of indirect blocks are 1 + 2^10 + 2^10 * 2^10 ~= 2^20 blocks.
That's a huge storage overhead.
For the second one, we have to do the translation for each logical block, so
around 2^30 times in total.

Hmm, seems not very good, right? Think about a case, all the physical blocks of
this 4TB file are all contiguous. In such a case, we just need one info like:
(lblk, pblk, len) where lblk = logical block number, pblk = physical block number
and len is the length of contiguous blocks. Then we just need one this kind of
struct to describe the relationship between logical block and physical block,
which is (0, x, 4TB), where x is a specific physical block number.
And for the translation, it becomes very simple, we can just do one visit to
get all the mapping details.

Here the key is we leverage the truth that for a file, contiguous logical blocks
can be contiguous in physical blocks too. For those logical blocks, we don't need
to store the mapping data separately. On the contrary, we use a single info
(lblk, pblk, len) to describe the relationship. And this is called a **extent**.

## Ondisk structure

The extents are organized as a B+ tree on disk. the root is in inode->i_block[].
A node of this B tree is made up of a header and a body.  The body consists of
several entries, either is ext4_extent_idx or ext4_extent. The former is for
non-leaf nodes, and the latter is for leaf nodes. Both the header and an entry
is 12Bytes. Here let's recall the inode->i_block[], it has 15 items with each
one 4Bytes, so 15 * 4 = 60Bytes in total. It can store 60 / 12 = 5 entries.
Minus the header, it can store 4 entries there. So if a file has <= 4 extents,
we just put the extents in inode->i_block[].

Otherwise, the entries in i_block[] are ext4_extent_idx, and point to extent blocks.
extent blocks are data blocks that are full of entries. **here please notice
onething: a data block(extent block) is a B+ tree node, and entries in it all belong
to this node.**

Let's first have a overlook on it:

```c
                                  +-------->+---extent block-----+                                                  +-->+---data blocks--+
                                  |         | ext4_extent_header |                                                 /    |                |
                                  |         +--------------------+                                                /     |                |
                                  |         | ext4_extent_idx    |                                               /      +----------------+
                                  |         +--------------------|                                              /       |                |
                                  |         | ext4_extent_idx    |                                             /        |                |
                                  |         +--------------------|                                            /         +----------------+
                                  |         | ...............    |         +---extent block-----+            /          |                |
                                  |         +--------------------+    +--->| ext4_extent_header |           /           |                |
                                  |         | ext4_extent_idx    |    |    +--------------------+          /            +----------------+
                                  |         +--------------------+    |    | ext4_extent        |---------+             |                |
                                  |         | ext4_extent_tail   |    |    +--------------------+          \            |                |
                                  |         +--------------------+    |    | ext4_extent        |           \           +----------------+
                                  |                                   |    +--------------------+            \          |                |
                                  |                                   |    | ...............    |             \         |                |
+------------------------+        |   +---->+---extent block-----+    |    +--------------------+              +------->+----------------+
|       ext4_inode       |        |   |     | ext4_extent_header |    |    | ext4_extent        |-------+               |                |
+------------------------+        |   |     +--------------------+    |    +--------------------+       |\              |                |
| +------i_block[]-----+ |        |   |     | ext4_extent_idx    |----+    | ext4_extent_tail   |       | +------------>+----------------+
| | ext4_extent_header | |        |   |     +--------------------+         +--------------------+       |               |                |
| +--------------------+ |        |   |     | ext4_extent_idx    |                                      |               |                |
| | ext4_extent_idx    |-|--------+   |     +--------------------+                                      \               +----------------+
| +--------------------+ |            |     | ...............    |                                       \              |                |
| | ext4_extent_idx    |-|------------+     +--------------------+                                        \             |                |
| +--------------------+ |                  | ext4_extent_idx    |-------->+---extent block-----+          +----------->+----------------+
| | ext4_extent_idx    |-|------------+     +--------------------+         | ext4_extent_header |                       |                |
| +--------------------+ |            |     | ext4_extent_tail   |         +--------------------+                       |                |
| | ext4_extent_idx    |-|--------+   |     +--------------------+         | ext4_extent        |                       +----------------+
| +--------------------+ |        |   |                                    +--------------------+                       |                |
+------------------------+        |   +---->+---extent block-----+         | ext4_extent        |                       |                |
                                  |         | ext4_extent_header |         +--------------------+                       +----------------+
                                  |         +--------------------+         | ...............    |
                                  |         | ext4_extent_idx    |         +--------------------+
                                  |         +--------------------+         | ext4_extent        |
                                  |         | ext4_extent_idx    |         +--------------------+
                                  |         +--------------------+         | ext4_extent_tail   |
                                  |         | ...............    |         +--------------------+
                                  |         +--------------------+
                                  |         | ext4_extent_idx    |
                                  |         +--------------------+
                                  |         | ext4_extent_tail   |
                                  |         +--------------------+
                                  |
                                  +-------->+---extent block-----+
                                            | ext4_extent_header |
                                            +--------------------+
                                            | ext4_extent_idx    |
                                            +--------------------+
                                            | ext4_extent_idx    |
                                            +--------------------+
                                            | ...............    |
                                            +--------------------+
                                            | ext4_extent_idx    |
                                            +--------------------+
                                            | ext4_extent_tail   |
                                            +--------------------+

```

All structures:

```c
struct ext4_extent_tail {
	__le32	et_checksum;	/* crc32c(uuid+inum+extent_block) */
};

struct ext4_extent {
	__le32	ee_block;	/* first logical block extent covers */
	__le16	ee_len;		/* number of blocks covered by extent */
	__le16	ee_start_hi;	/* high 16 bits of physical block */
	__le32	ee_start_lo;	/* low 32 bits of physical block */
};

struct ext4_extent_idx {
	__le32	ei_block;	/* index covers logical blocks from 'block' */
	__le32	ei_leaf_lo;	/* pointer to the physical block of the next *
				 * level. leaf or next index could be there */
	__le16	ei_leaf_hi;	/* high 16 bits of physical block */
	__u16	ei_unused;
};

struct ext4_extent_header {
	__le16	eh_magic;	/* probably will support different formats */
	__le16	eh_entries;	/* number of valid entries */
	__le16	eh_max;		/* capacity of store in entries */
	__le16	eh_depth;	/* has tree real underlying blocks? */
	__le32	eh_generation;	/* generation of the tree */
};

```

- ext4_extent_header
  `eh_depth` is counted from 0 and from leaf to root, so leaf->eh_depth = 0 and
  root->eh_depth is the maximum one.

- ext4_extent_idx
  ei_leaf_lo, ei_leaf_hi together stand for the physical block number of next
  layer B+ tree node.
  ei_block is the logical block number. Given the current logical block is x, how
  can we find leaf from it?
  For example: the current non-leaf node where we are is:
  `[lblk0, ..][lblk1, ..][lblk2, ..][lblk3, ..]`
  We can compare x with lblk0, lbk1,2 and 3 to see which idx it locates and then
  go to the next layer, to be more accurate, we can do binary search here (this is
  basic knowledge of B+ tree, you can refer to wiki or some data structure materials)

- ext4_extent_tail
  Just a checksum. Note: we don't need this for i_block in inode, since inode is
  checked somewhere else.

- ext4_extent
  This is the entry in leaf node.

## Extent Status Tree

### data structure

Now that we are clear about the ondisk layout, let's turn to the memory. Like inode
and dentry, extent also has memory cache, it's called extent status tree.

```c
/*
 * fourth extended file system inode data in memory
 */
struct ext4_inode_info {
 	struct rw_semaphore i_data_sem;
	struct inode vfs_inode;
	struct jbd2_inode *jinode;

	/* extents status tree */
	struct ext4_es_tree i_es_tree;
	rwlock_t i_es_lock;
	struct list_head i_es_list;
	unsigned int i_es_all_nr;	/* protected by i_es_lock */
	unsigned int i_es_shk_nr;	/* protected by i_es_lock */
	ext4_lblk_t i_es_shrink_lblk;	/* Offset where we start searching for
					   extents to shrink. Protected by
					   i_es_lock  */
};
```

ext4_inode_info is the memory version ext4_inode. i_es_tree represents the extent
status tree.
```c
struct ext4_es_tree {
	struct rb_root root;
	struct extent_status *cache_es;	/* recently accessed extent */
};
```

The es tree is organized as a red black tree. cache_es is the recent visited item,
AKA a cache of cache. A node in this rb tree is:

```c
struct extent_status {
	struct rb_node rb_node;
	ext4_lblk_t es_lblk;	/* first logical block extent covers */
	ext4_lblk_t es_len;	/* length of extent in block */
	ext4_fsblk_t es_pblk;	/* first physical block */ 
};
```

Each node is a cache of a disk extent. The rb tree is sorted by es_lblk----the start
block address this extent covers.

### EXTENT_STATUS_xxx flags

Ok, the name \<extent status\> may be confusing, the word 'status' here means one of:

```c
#define EXTENT_STATUS_WRITTEN	(1 << ES_WRITTEN_B)
#define EXTENT_STATUS_UNWRITTEN (1 << ES_UNWRITTEN_B)
#define EXTENT_STATUS_DELAYED	(1 << ES_DELAYED_B)
#define EXTENT_STATUS_HOLE	(1 << ES_HOLE_B)
#define EXTENT_STATUS_REFERENCED	(1 << ES_REFERENCED_B)
```

The status is represented by     . Now let's look into each of them, this is related
with the initative of the es tree.

 - `WRITTEN`
    **pages exists in page cache, ondisk blocks exists too.**

    this is the normal case, an es node is a cache version of the ondisk extent

 - `UNWRITTEN`
    **pages exists in page cache, ondisk blocks exists but not filled (they contains junk data).**

    this is also cache version of an ondisk extent. The difference is the ondisk
    data blocks hasn't been touched yet. This is useful for syscall like fallocate().
    fallocate() allows you to allocate blocks to a file without filling zeroes.
    Think about the situation when there is no es tree: if you want to generate
    a file of 4GB, you have to fill all the 4GB blocks to zeroes. With es tree,
    you can just set up metadata like the extent and extent status, and mark it as
    UNWRITTEN, and allocate data blocks in block map, no need to do any data IO.

 - `HOLE`
    **no pages in page cache, and no ondisk blocks either.**

    stands for a hole in a file, in this case, the pblk is meaningless. The file
    logical range doesn't have it's physical range ondisk.
 - `DELAYED`
    **pages exits in page cache, no ondisk blocks.**
    **(This is why es tree is invented)**

    This is used by a feature called delay allocation. Normally, when a buffered
    write request writes some data to the page cache and expands the file size or
    fill a hole area, it should allocate blocks ondisk immediately.
    But actually we can delay the allocation to the writeback period, this will
    improve the latency. Think about the situation of no es tree: to distinguish
    HOLE and DELAYED extent, we have to look into the page cache to see if there
    are pages in the range. That makes things complicated and may cause bugs.
    (comments are welcome here: why it's complicated? I just get this from mail list)
    With this flag, we can do all these by just checking this flag.
    The advantage of this feature is we can gether neighbor logical blocks and in
    the writeback period we try to allocate contiguous physical blocks to get better
    performance.

 - `REFERENCED`
    undiscovered. TOBEDONE.

The above flags are stored in the highest (five) bits in extent_status->es_pblk.

## Translation

A write request is like write(fd, offset, buf, len), means writing data [buf, buf+len)
to position [offset, offset+len)  in file fd. From [offset, offset+len) we can
get the logical blocks range [offset/blk_sz, (offset+len)/blk_sz). After we get
the inode of this file, we have to translate the logical range to physical ranges.

### ext4 dio write

Pick up ext4 direct write as an example, let's see how the translation goes. As this
is highly related with iomap, please read [IOMAP](./iomap.md) first, there is a big
picture which describes the whole process of ext4 dio write. Here let's focus on
the extent part. As being said in [IOMAP](./iomap.md), iomap_begin() does the translation,
which is ext4_iomap_begin() in ext4 filesystem.

### ext4_iomap_begin()

ext4_iomap_begin() is simple:
 - it defines a struct ext4_map_blocks and initializes its members, then calls
   ext4_map_blocks().

```c
struct ext4_map_blocks {
	ext4_fsblk_t m_pblk;
	ext4_lblk_t m_lblk;
	unsigned int m_len;
	unsigned int m_flags;
};
```

struct ext4_map_blocks is used to store a mapping, you can see it as 'ext4\'s private
struct iomap'. And it indeed will deliver its info to struct iomap at last.

 - ext4_map_blocks()
   this is the main dish, we'll dive into it later.

 - ext4_set_iomap()
   After we get the mapping, we need to fill the struct iomap by struct ext4_map_blocks.
   This basically involves setting up:
   - iomap->flags

	```c
	/*
	 * Writes that span EOF might trigger an I/O size update on completion,
	 * so consider them to be dirty for the purpose of O_DSYNC, even if
	 * there is no other metadata changes being made or are pending.
	 */
	iomap->flags = 0;
	if (ext4_inode_datasync_dirty(inode) ||
	    offset + length > i_size_read(inode))
		iomap->flags |= IOMAP_F_DIRTY;

	if (map->m_flags & EXT4_MAP_NEW)
		iomap->flags |= IOMAP_F_NEW;

	```
        TOBEDONE: analyze IOMAP_F_DIRTY and IOMAP_F_NEW.

   - iomap->bdev

	```c
	if (flags & IOMAP_DAX)
		iomap->dax_dev = EXT4_SB(inode->i_sb)->s_daxdev;
	else
		iomap->bdev = inode->i_sb->s_bdev;
	```

   - iomap->offset && iomap->length

	```c
	iomap->offset = (u64) map->m_lblk << blkbits;
	iomap->length = (u64) map->m_len << blkbits;
	```

   - iomap->type && iomap->addr

	```c
	if (map->m_flags & EXT4_MAP_UNWRITTEN) {
		iomap->type = IOMAP_UNWRITTEN;
		iomap->addr = (u64) map->m_pblk << blkbits;
		if (flags & IOMAP_DAX)
			iomap->addr += EXT4_SB(inode->i_sb)->s_dax_part_off;
	} else if (map->m_flags & EXT4_MAP_MAPPED) {
		iomap->type = IOMAP_MAPPED;
		iomap->addr = (u64) map->m_pblk << blkbits;
		if (flags & IOMAP_DAX)
			iomap->addr += EXT4_SB(inode->i_sb)->s_dax_part_off;
	} else {
		iomap->type = IOMAP_HOLE;
		iomap->addr = IOMAP_NULL_ADDR;
	}
	```

### ext4_map_blocks()

I have to say that I haven't figured out all the detail of it. I'll explain it as
possible as I can.
Generally speaking, ext4_map_blocks() does two things:
 - look up the mapping
	- fast path: look it up in es tree
	- slow path: look it up in extent tree
		- look up needed blocks (extent node) in per cpu array bh_lrus
		- look up needed blocks in the page cache of the device file
		- page cache miss, load needed blocks from disk(I'm not sure here)
 - handle the mapping

#### ext4_es_lookup_extent

This one is super simple, we give it the desired logical block number, it search
the rb tree for the right node. I'm not gona do code analysis for it here, the code
explains itself well.

After get the extent, we can update struct ext4_map_blocks now. This includes
`map->pblk`, `map->m_flags` and `map->m_len`. Here we must be clear about the m_flags:

```
EXTENT_STATUS_WRITTEN --> EXT4_MAP_MAPPED
EXTENT_STATUS_UNWRITTEN --> EXT4_MAP_UNWRITTEN
EXTENT_STATUS_DELAYED --> ?
EXTENT_STATUS_HOLE --> ?
```
#### ext4_ext_map_blocks()

If we fail to find the target extent in es tree, we have to search the extent tree.
And we have to hold the rw semaphore ext4_inode_info->i_data_sem.

```
in ext4_map_blocks():

	down_read(&EXT4_I(inode)->i_data_sem);
	if (ext4_test_inode_flag(inode, EXT4_INODE_EXTENTS)) {
		retval = ext4_ext_map_blocks(handle, inode, map, 0);
	} else {
		retval = ext4_ind_map_blocks(handle, inode, map, 0);
	}
```

One thing we can obviously see is there are two ways to organize the mapping.
One is the legacy mode----indirect mapping, the other one is the extent tree.
And a flag EXT4_INODE_EXTENTS in inode controls which way to use.
(So theoretically a filesystem can have two methods in use at the same time?)

Ok, let's jump into the function. I'll list the main operation it does in order.

 - `path = ext4_find_extent(inode, map->m_lblk, NULL, 0);`
	the first thing ext4_ext_map_blocks() does is searching the target extent
	in the extent tree.
	```c
	struct ext4_ext_path {
		ext4_fsblk_t			p_block; // physical block number
		__u16				p_depth;
		__u16				p_maxdepth;
		struct ext4_extent		*p_ext; // to the extent if it's leaf
		struct ext4_extent_idx		*p_idx; // to the idx node if it's non-leaf
		struct ext4_extent_header	*p_hdr; // to the extent header
		struct buffer_head		*p_bh; // to the block
	};

	```
	the return value is an array `path[]`
	`struct ext4_ext_path *path` records the path from root to leaf during
	the walking. Meaning of members are clear, here I want to mention `struct buffer_head`.
	It stands for a disk block, and is used to manage the block. Here it points to
	the corresponding block which is a extent node. Notice that this block is
	in a page in the blkdev page cache.
	The detail of buffer_head is complex, I've wrote another article for it.
	[buffer_head](./buffer_head.md)
	It's ok not to know the detail of buffer_head, you just need to know that
	**page cache consists of pages, and those pages each contains one or more
	blocks(if block size < page size). These blocks are caches of filesystem's
	ondisk blocks.**

