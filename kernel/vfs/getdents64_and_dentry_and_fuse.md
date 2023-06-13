# getdents64

## dentry creation

We know there is dcache, which is a cache containing visited directory entries.
Now given an instance, let's think about how many dentries are created. To observe
it, you can print some info in `__d_alloc()` in `fs/dcache.c`, that's what I do.
Say you have a file at `/path/to/file`. When you do `ls /path/to`, how many
dentries are created and inserted to dcache?
It's a simple question, we all know dentries are created during pathname lookup.
Let's consider each component:
- `/`
  The dentry of root dir is created when mount this filesystem.
- `to`
  When we walk to this component, we surely create dentry for it.

Is there anything else? How about `file`? No, there is no dentry for `file` since
we are visiting `to`, not `file`, dentry for `file` won't be there until we visit
it.

## getdents64

#### 1. Now Question2: Does it visit ondisk filesystem if there are already all the dentries for a dir when we do `ls`?
Pick up the above case, dentry for `file` is already there in dcache, does
`ls /path/to` cause reading dir entries of `to` from ondisk filesystem?

The answer is yes(I think), since linux kernel doesn't know if the dentries in
dcache are all what we want. Note, dentries may be reclaimed at some time, for
example: under high memory pressure. It's a **cache** after all, it accelerates
some thing, but it is not the thing itself.
So, each time you do `ls`, you visit disk for the content. The syscall for `ls`
to get dir entries of a dir is `getdents64`

### 2. Is the answer for the previous question completed?

**Don't forget the page cache**. Ok, both directory and regular file have page
cache. So if you `ls` a directory the second time, no need to visit ondisk fs.
Note, page cache is not like dcache, you can always know whether the page cache
is complete or not. For example, there are 3 dentries for directory A, you don't
know if there is a 4th item in A, but if there are 3 dir entries in page cache,
you can always know if the 4th item exists by the size of file A and the status
of the page where the 4th item lies on.

**Note, by reading `ext4_readdir()` which is called by `getdents64()` for ext4, I
found that ext4 directly reads the page(contains block where the dir entry lies on)
in the corresponding bdev file(the block device file) page cache, not the page cache
for that individual directory file.**
So looks like ext4 sees those directory entries as filesystem metadata, (correct
me if I'm wrong), because we know that the page cache of the device file is for
fs metadata access acceleration, e.g. extent tree.

#### 2.1 a experiment for page cache
To confirm which page cache is leveraged, I did a experiment: `ls` a directory
twice and observe the buff cache(bdev device file cache) change:

```bash

root@hao-A29R:~/workspace# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 901384   6768  48460    0    0  2477     7  303  473  1  4 95  0  0
root@hao-A29R:~/workspace# ls dentry_test
tmp_fd0  tmp_fd1
root@hao-A29R:~/workspace# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 901384   6784  48404    0    0  1438     7  182  281  1  2 97  0  0
root@hao-A29R:~/workspace# ls dentry_test
tmp_fd0  tmp_fd1
root@hao-A29R:~/workspace# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 901384   6784  48372    0    0   337     2   45   70  0  1 99  0  0
root@hao-A29R:~/workspace# 

```

We can see after the first `ls`, the buff changes, but after the second try, it
remains same value. It's not scientific, but tells something anyway...

## fuse

Ok, let's focus on `getdents64` in fuse filesystem, things are a bit different
there.
the `iterate_shared` member in fuse dir inode operation is `fuse_readdir()`

```c

int fuse_readdir(struct file *file, struct dir_context *ctx)
{
	struct fuse_file *ff = file->private_data;
	struct inode *inode = file_inode(file);
	int err;

	if (fuse_is_bad(inode))
		return -EIO;

	mutex_lock(&ff->readdir.lock);

	err = UNCACHED;
	if (ff->open_flags & FOPEN_CACHE_DIR)
		err = fuse_readdir_cached(file, ctx);
	if (err == UNCACHED)
		err = fuse_readdir_uncached(file, ctx);

	mutex_unlock(&ff->readdir.lock);

	return err;
}

```
