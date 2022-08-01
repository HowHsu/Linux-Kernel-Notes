# IOMAP

iomap is an abstract layer to do the (file logical block address --> physical block address)
translation. It is friendly to extent based filesystems.

## OVERVIEW

To have a overview, we pick up ext4 dio write as an instance. Below is the big picture
of it.

![](./images/ext4_file_write_iter.png)

Ok, I know what you re thinking, don't panic, this time we just focus on a small
part of it.
First we have to know that an IO request like write(fd, offset, buf, len) may
corresponds to multiple BIOs (in case you're not familiar with this concept, a BIO
represents an IO request to a segment of contiguous disk address). The reason is
the logical file segment [offset, offset+len) may map to multiple physical segments.

Based on the above fact, IOMAP does it in this way: iterate [offset, offset+len), once
find a mapping, generate a BIO for it, then handle the next one. The specific code is:

```c
while ((ret = iomap_iter(&iomi, ops)) > 0) {
	iomi.processed = iomap_dio_iter(&iomi, dio);

	/*
	 * We can only poll for single bio I/Os.
	 */
	iocb->ki_flags &= ~IOCB_HIPRI;
}
```

## CODE ANALYSIS

Let's dive much more deeply into it. In the above code, `iomap_iter()` takes the
iomap iterator and the iomap ops to iterator the logical segment, it returns when
a physical segment is found. And `iomap_dio_iter()`, handles the physical segment
found, it basically generate a BIO for it.

Before analyze these two functions, we should be clear about the below two data
structures.

```c
struct iomap_ops {
	/*
	 * Return the existing mapping at pos, or reserve space starting at
	 * pos for up to length, as long as we can do it as a single mapping.
	 * The actual length is returned in iomap->length.
	 */
	int (*iomap_begin)(struct inode *inode, loff_t pos, loff_t length,
			unsigned flags, struct iomap *iomap,
			struct iomap *srcmap);

	/*
	 * Commit and/or unreserve space previous allocated using iomap_begin.
	 * Written indicates the length of the successful write operation which
	 * needs to be commited, while the rest needs to be unreserved.
	 * Written might be zero if no data was written.
	 */
	int (*iomap_end)(struct inode *inode, loff_t pos, loff_t length,
			ssize_t written, unsigned flags, struct iomap *iomap);
};
```

struct iomap_ops defines how to do the logical->physical translation. In each
iteration, we call iomap_begin() to do the translation and return a single mapping.
iomap_end() is undiscovered for now...
This is implemented by the real filesystem.

```c
struct iomap {
	u64			addr; /* disk offset of mapping, bytes */
	loff_t			offset;	/* file offset of mapping, bytes */
	u64			length;	/* length of mapping, bytes */
	u16			type;	/* type of mapping */
	u16			flags;	/* flags for mapping */
	struct block_device	*bdev;	/* block device for I/O */
	struct dax_device	*dax_dev; /* dax_dev for dax operations */
	void			*inline_data;
	void			*private; /* filesystem private */
	const struct iomap_page_ops *page_ops;
};
```
Just like what I said, iomap_begin() returns a single mapping each time. And the
returned mapping is stored in struct iomap. The comments above are already very
clear, I'm not going to be verbose..., But one thing I have to mention is type,
We'll talk more about it later.

```c
/**
 * struct iomap_iter - Iterate through a range of a file
 * @inode: Set at the start of the iteration and should not change.
 * @pos: The current file position we are operating on.  It is updated by
 *	calls to iomap_iter().  Treat as read-only in the body.
 * @len: The remaining length of the file segment we're operating on.
 *	It is updated at the same time as @pos.
 * @processed: The number of bytes processed by the body in the most recent
 *	iteration, or a negative errno. 0 causes the iteration to stop.
 * @flags: Zero or more of the iomap_begin flags above.
 * @iomap: Map describing the I/O iteration
 * @srcmap: Source map for COW operations
 */
struct iomap_iter {
	struct inode *inode;
	loff_t pos;
	u64 len;
	s64 processed;
	unsigned flags;
	struct iomap iomap;
	struct iomap srcmap;
	void *private;
};
```

The comments above it explain it well. struct iomap_iter is the iterator of iomap,
just think about iov_iter, really the same thing.



### `int iomap_iter(struct iomap_iter *iter, const struct iomap_ops *ops)`

![](./images/iomap_iter.png)


