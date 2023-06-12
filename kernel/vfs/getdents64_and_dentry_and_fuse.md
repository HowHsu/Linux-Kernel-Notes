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

Now Question2: Does it visit ondisk filesystem if there are already all the
dentries for a dir when we do `ls`?
Pick up the above case, dentry for `file` is already there in dcache, does
`ls /path/to` cause reading dir entries of `to` from ondisk filesystem?

The answer is yes(I think), since linux kernel doesn't know if the dentries in
dcache are all what we want. Note, dentries may be destroyed at some time, it's
a **cache** above all, it accelerates some thing, but it is not the thing itself.
So, each time you do `ls`, you visit disk for the content.
The syscall for `ls` to get dir entries of a dir is `getdents64`

## fuse

Ok, let's focus on `getdents64` in fuse filesystem, things are a bit different
there.
