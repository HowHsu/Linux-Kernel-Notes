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

## Translation

A write request is like write(fd, offset, buf, len), means writing data [buf, buf+len)
to position [offset, offset+len)  in file fd. From [offset, offset+len) we can
get the logical blocks range [offset/blk_sz, (offset+len)/blk_sz). After we get
the inode of this file, we have to translate the logical range to physical ranges.

### ext4 dio write

Pick up ext4 direct write as an example, let's see how the translation goes. As this
is highly related with iomap, please read [IOMAP](./iomap.md) first, there is a big
picture which describes the whole process of ext4 dio write. Here let's focus on
the extent part.






## Extent Status Tree
