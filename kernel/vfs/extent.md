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
  ei_block is the logical block number. Given the target logical block is x, how
  can we find leaf from root?
  For example: a non-leaf node is:`[lblk0, ..][lblk1, ..][lblk2, ..][lblk3, ..]`
  We can compare x with lblk0, lbk1,2 and 3 to see which idx it locates and then
  go to the next layer.(this is basic knowledge of B+ tree, you can refer to wiki
  or some data structure materials)

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
    this is the normal case, an es node is a cache version of the ondisk extent
 - `UNWRITTEN`
    this is also cache version of an ondisk extent. The difference is the ondisk
    data blocks hasn't been touched yet. This is useful for syscall like fallocate().
    fallocate() allows you to allocate blocks to a file without filling zeroes.
    Think about the situation when there is no es tree: if you want to generate
    a file of 4GB, you have to fill all the 4GB blocks to zeroes. With es tree,
    you can just set up metadata like the extent and extent status, and mark it as
    UNWRITTEN, and allocate data blocks in block map, no need to do any data IO.
 - `HOLE`
    stands for a hole in a file, in this case, the pblk is meaningless. The file
    logical range doesn't have it's physical range ondisk.
 - `DELAYED`
    This is used by a feature called delay allocation. Normally, when a buffered
    write request writes some data to the page cache and expands the file size,
    it should allocate blocks ondisk immediately.
    But actually we can delay the allocation to the writeback period, this will
    improve the latency. Think about the situation of no es tree: to distinguish
    HOLE and DELAYED extent, we have to look into the page cache to see if there
    are pages in the range. That makes things complicated and may cause bugs.
    (comments are welcome here: why it's complicated? I just get this from mail list)
    With this flag, we can do all these by just checking this flag.
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
struct iomap'. And it indeed will deliver its info to struct iomap.







