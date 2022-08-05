# buffer_head

Ok, you may heard a lot of stories about buffer_head if you've worked on linux
IO stack for a long time. But what indeed is it? In this article, I will explain
it as possible as I can.

## History

In linux kernel, there is a cache called buffer cache, it is to cache filesystem's
ondisk blocks. And `struct buffer_head` is for managing those blocks in this cache.
But that's not all what it is. Before `struct bio` occured, buffer_head is also
used as the granularity of IO, and play a big role in logical block to physical block
translation as well.
So at that time, the structure is big. Nowadays, `struct bio` becomes the basic unit
of IO, thus we support more efficient IO since we can do much bigger IO each time.
So buffer_head today is just for managing a block in buffer cache.
I know you may be unfamiliar with buffer cache, that's right since nowadays buffer
cache is merged with page cache!

## buffer cache

The buffer cache is to cache ondisk blocks of a filesystem. It is useful when
block size < page size (it seems block size <= page size is always true).
In this situation, a page in page cache is divided to multiple blocks.
Why do we need it?
Ok, I'm not very sure... but I guess it is because **the granularity of writeback
when a page is dirty and loading when cache miss is block not page.** The reason
is sometimes a page is too big. For example, in a page cache of a block device
file in bdev filesystem, most data is meta data like extent node blocks which
are small. If we load it from disk in a unit of page, that causes extra overhead.
Setting block size as the smallest IO we can do makes it flexibe.

## buffer_head

```c
struct buffer_head {
	unsigned long b_state;		/* buffer state bitmap (see above) */
	struct buffer_head *b_this_page;/* circular list of page's buffers */
	struct page *b_page;		/* the page this bh is mapped to */

	sector_t b_blocknr;		/* start block number */
	size_t b_size;			/* size of mapping */
	char *b_data;			/* pointer to data within the page */

	struct block_device *b_bdev;
	bh_end_io_t *b_end_io;		/* I/O completion */
	void *b_private;		/* reserved for b_end_io */
	struct list_head b_assoc_buffers; /* associated with another mapping */
	struct address_space *b_assoc_map;	/* mapping this buffer is
						   associated with */
	atomic_t b_count;		/* users using this buffer_head */
	spinlock_t b_uptodate_lock;	/* Used by the first bh in a page, to
					 * serialise IO completion of other
					 * buffers in the page */
};
```

Suppose block size is 1024 Bytes and page size is 4KB, then the relationship is like this:

```c
+-------------+
| struct page |
+-------------+              +--------------------+
|   private   |------------->| struct buffer_head |
|             |              +--------------------+
|   .......   |   +--------->|      b_this_page   |--+
+-------------+   |  +-------|      b_page        |  |
       ^          |  |       |      b_blocknr     |  |
       |          |  |       |      b_size        |  |
       |          |  |       |      b_data        |--|---------+
       |       +--+  |       |      b_bdev        |  |         |
       |       |     |       |      ......        |  |         |
       |       |     |       +--------------------+  |         |
       |       |     |                               |         |                             +---the-disk--+
       |       |     |       +--------------------+  |         |                          +->|             |
       |       |     |       | struct buffer_head |  |         |                         /   |             |
       |       |     |       +--------------------+  |         |                        /    +-------------+
       |       |     |   +---|      b_this_page   |<-+         |                       /     |             |
       +-------------+---|---|      b_page        |            |      +---the-page--+ /      |             |
               |     |   |   |      b_blocknr     |            +----->|             |/       +-------------+
               |     |   |   |      b_size        |                   |             |    +-->|             |
               |     |   |   |      b_data        |------------+      +-------------+    |   |             |
               |     |   |   |      b_bdev        |            |      |             |    |   +-------------+
               |     |   |   |      ......        |            +----->|             |------->|             |
               |     |   |   +--------------------+                   +-------------+    |   |             |
               |     |   |                                            |             |    +   +-------------+
               |     |   |   +--------------------+           +------>|             |\  /    |             |
               |     |   |   | struct buffer_head |           |       +-------------+ \/     |             |
               |     |   |   +--------------------+           |       |             | /\     +-------------+
               |     |   +-->|      b_this_page   |--+        |   +-->|             |/  +--->|             |
               |     +-------|      b_page        |  |        |   |   +-------------+        |             |
               |     |       |      b_blocknr     |  |        |   |                          +-------------+
               |     |       |      b_size        |  |        |   |                          |             |
               |     |       |      b_data        |--|--------+   |                          |             |
               |     |       |      b_bdev        |  |            |                          +-------------+
               |     |       |      ......        |  |            |                          |             |
               |     |       +--------------------+  |            |                          |             |
               |     |                               |            |                          +-------------+
               |     |       +--------------------+  |            |                          |             |
               |     |       | struct buffer_head |  |            |                          |             |
               |     |       +--------------------+  |            |                          +-------------+
               +-----|-------|      b_this_page   |<-+            |
                     +-------|      b_page        |               |
                             |      b_blocknr     |               |
                             |      b_size        |               |
                             |      b_data        |---------------+
                             |      b_bdev        |
                             |      ......        |
                             +--------------------+

```

b_this_page, b_page, b_data are clearly explained in this picture. Here
notice b_this_page, it forms a circular list.

## extent search, miss and load

Now let's dive into the code to feel how it works. Here I pick up the extent related
code as an example.

In article [extent.md](./extent.md) we get to know `ext4_find_extent()` is used to
find the extent for the target logical block number. When we walk through the B+ tree,
we have to fetch the block which contains the extent node. A possible case is the
block is not in memory. In that case, we have to load it from disk.

`ext4_find_extent() --> __read_extent_tree_block()`

Let's look into __read_extent_tree_block()

### __read_extent_tree_block()

```c
static struct buffer_head *
__read_extent_tree_block(const char *function, unsigned int line,
			 struct inode *inode, struct ext4_extent_idx *idx,
			 int depth, int flags)
{
	pblk = ext4_idx_pblock(idx);
	bh = sb_getblk_gfp(inode->i_sb, pblk, gfp_flags);
	if (unlikely(!bh))
		return ERR_PTR(-ENOMEM);

	if (!bh_uptodate_or_lock(bh)) {
		trace_ext4_ext_load_extent(inode, pblk, _RET_IP_);
		err = ext4_read_bh(bh, 0, NULL);
		if (err < 0)
			goto errout;
	}
	if (buffer_verified(bh) && !(flags & EXT4_EX_FORCE_CACHE))
		return bh;
	set_buffer_verified(bh);
	return bh;
errout:
	put_bh(bh);
	return ERR_PTR(err);
}

```

I've removed unrelated code. It basically calls two functions.

#### `bh = sb_getblk_gfp(inode->i_sb, pblk, gfp_flags);`

In short, this function gets the buffer_head structure associated with the block
pblk. The way is searching it in the corresponding blkdev file's page cache.

Why do we have to look it up in blkdev file's page cache?
Since that block is a extent node, it isn't file data, so it doesn't belong to
any file else but the device file!

Onething to notice is the block number we use is the physical block number pblk.
The trick here is that in blkdev file represents the whole device, that being said,
the logical block number in the page cache is same as the physical block number
on disk.




`err = ext4_read_bh(bh, 0, NULL);`
