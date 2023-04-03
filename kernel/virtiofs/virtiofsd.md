# virtiofsd

This article is about the backend of virtiofs: virtiofsd. And here I'll
pick up the rust version as an instance, while I'm also new to this languagethough. I'm going to talk about the big picture of virtiofsd as well as
diving deeply into the code to make you have a full sense of it.

### framework

Let's first look at a graph which shows the relationship of the main classes
 of virtiofsd.

![](./images/virtiofsd.drawio.svg)



### cache policy

- guest kernel page cache: GCache
- host kern el page cache: HCache

O_DIRECT: vfs flag, users can set it when opening a file
FOPEN_DIRECT_IO: fuse flag, users can set it when starting virtiofsd by `--cache`

- `--allow-direct-io true`
This means we respect the original IO flag(vfs flag) of how we leverage the host
page cache
```
vfs\fuse    FOPEN_DIRECT_IO(--cache=never)	      !FOPEN_DIRECT_IO(--cache=auto/always)
O_DIRECT        !GCache   !HCache                     !GCache  !HCache
!O_DIRECT	!GCache    HCache                      GCache   HCache
```

- `--allow-direct-io false (default)`
This means we see all the IO requests from guest as buffered IO requests on host
```
vfs\fuse    FOPEN_DIRECT_IO(--cache=never)	      !FOPEN_DIRECT_IO(--cache=auto/always)
O_DIRECT        !GCache    HCache                     !GCache   HCache
!O_DIRECT	!GCache    HCache                      GCache   HCache
```
#### a more clear inclusion

for guest page cache:

`--cache=never`: we don't use guest page cache anyway!
`--cache=auto/always`: we use guest page cache for buffered IO

for host page cache:
`!O_DIRECT`: we use host page cache anyway!
`O_DIRECT`: we use host page cache if `--allow-direct-io=false`

### inode mapping

Think about a problem, how does the frontend(guest kernel) read a file in the share directory
of virtiofs?
Let's first look into how a file is read on host.
	- first you have to open it: fd = open(path, flags)
	- then read it: read(fd, buf, offset, len)

So the key is fd--file descriptor--which can  be seen as a index of process' open file array.
But for read requests from a geust, things become complicated. Because the guest kernel cannnot
don't have the fd information of the virtiofsd process.
You may think about setting a mapping between guest kernel inode and virtiofsd fd in the guest kernel.
That looks good, but I'm not sure if that works out 100%. But that's clear that it's more straightforward
to create a map between guest kernel inode and host filesystem inode. In reality, virtiofs goes for this way.

Let's jump into this topic by reading the code. Here we look at some filesystem operations

#### lookup

lookup is a directory inode operation. It is called in the syscall `open` process.
And it is the slow code path of it. When we iterate to a path component, we first
try to find its dentry in dcache, that's called the fast path. If that fails, we
look into the ondisk filesystem to load the parent dir content by its inode, and
find the dir entry in it, then build the dentry based on the found dir entry, this
is the so-called slow path.
Please refer to [open](../vfs/open.md) for more detail.

For virtiofs, the `ondisk filesystem` is the host backend virtiofsd daemon. so
virtiofsd must support this `LOOKUP` request.

The virtiofs frontend(guest kernel) filesystem structure is below(virtiofs uses fuse filesystem as its filesystem part):
```c
static const struct inode_operations fuse_dir_inode_operations = {
	.lookup		= fuse_lookup,
	.mkdir		= fuse_mkdir,
	.symlink	= fuse_symlink,
	.unlink		= fuse_unlink,
	.rmdir		= fuse_rmdir,
	.rename		= fuse_rename2,
	.link		= fuse_link,
	.setattr	= fuse_setattr,
	.create		= fuse_create,
	.atomic_open	= fuse_atomic_open,
	.tmpfile	= fuse_tmpfile,
	.mknod		= fuse_mknod,
	.permission	= fuse_permission,
	.getattr	= fuse_getattr,
	.listxattr	= fuse_listxattr,
	.get_inode_acl	= fuse_get_inode_acl,
	.get_acl	= fuse_get_acl,
	.set_acl	= fuse_set_acl,
	.fileattr_get	= fuse_fileattr_get,
	.fileattr_set	= fuse_fileattr_set,
};

```

```c
static struct dentry *fuse_lookup(struct inode *dir, struct dentry *entry,
				  unsigned int flags)
{
	int err;
	struct fuse_entry_out outarg;
	struct inode *inode;
	struct dentry *newent;
	bool outarg_valid = true;
	bool locked;

	if (fuse_is_bad(dir))
		return ERR_PTR(-EIO);

	locked = fuse_lock_inode(dir);
	err = fuse_lookup_name(dir->i_sb, get_node_id(dir), &entry->d_name,
			       &outarg, &inode);
	fuse_unlock_inode(dir, locked);
	if (err == -ENOENT) {
		outarg_valid = false;
		err = 0;
	}
	if (err)
		goto out_err;

	err = -EIO;
	if (inode && get_node_id(inode) == FUSE_ROOT_ID)
		goto out_iput;

	newent = d_splice_alias(inode, entry);
	err = PTR_ERR(newent);
	if (IS_ERR(newent))
		goto out_err;

	entry = newent ? newent : entry;
	if (outarg_valid)
		fuse_change_entry_timeout(entry, &outarg);
	else
		fuse_invalidate_entry_cache(entry);

	if (inode)
		fuse_advise_use_readdirplus(dir);
	return newent;

 out_iput:
	iput(inode);
 out_err:
	return ERR_PTR(err);
}

```

- `fuse_lookup_name()`
	Here we only care about `fuse_lookup_name()`, it search/create the target dentry (and its inode) by sending request to the baackend daemon.

	```c
		struct fuse_mount *fm = get_fuse_mount_super(sb);
		struct fuse_forget_link *forget = fuse_alloc_forget();

		fuse_lookup_init(fm->fc, &args, nodeid, name, outarg);

		err = fuse_simple_request(fm, &args);

		*inode = fuse_iget(sb, outarg->nodeid, outarg->generation, &outarg->attr, entry_attr_timeout(outarg), attr_version);

		if (!*inode)
			fuse_queue_forget(fm->fc, forget, outarg->nodeid, 1);
	```

	- `fuse_lookup_init(fm->fc, &args, nodeid, name, outarg)`

		```c
			static void fuse_lookup_init(struct fuse_conn *fc, struct fuse_args *args,
						     u64 nodeid, const struct qstr *name,
						     struct fuse_entry_out *outarg)
			{
				memset(outarg, 0, sizeof(struct fuse_entry_out));
				args->opcode = FUSE_LOOKUP;
				args->nodeid = nodeid;
				args->in_numargs = 1;
				args->in_args[0].size = name->len + 1;
				args->in_args[0].value = name->name;
				args->out_numargs = 1;
				args->out_args[0].size = sizeof(struct fuse_entry_out);
				args->out_args[0].value = outarg;
			}

		```

		The code explain itself quite well, it initializes the arguments of
		the the `LOOKUP` fuse reuqest. The key ones are `nodeid` and `in_args[0].value`,
		which stands for the parent directory inode and the name string of the dir entry.

	- `fuse_simple_request(fm, &args)`
		This sends the reuqest.
		- `fuse_request_alloc()`
		- `__fuse_request_send(req)`
			- `queue_request_and_unlock(fiq, req)`
			- `request_wait_answer(req)`
	- `fuse_iget()`
		- `inode = iget5_locked(sb, nodeid, fuse_inode_eq, fuse_inode_set, void *data = &nodeid);`
			This tries to find the inode from icache, and create a new one
			if it doesn't exist.

			- `fuse_inode_eq()`
				Used to compare inode when searching icache.
			- `fuse_inode_set(struct inode *inode, void *_nodeidp)`
				Init the new created inode.
				```c
					static int fuse_inode_set(struct inode *inode, void *_nodeidp)
					{
						u64 nodeid = *(u64 *) _nodeidp;
						get_fuse_inode(inode)->nodeid = nodeid;
						return 0;
					}
				```

				**Here we can see the mapping of inode number is stored in fuse_inode->nodeid, and it's
				a simple linear relationship**
			- `data`
				The argument for fuse_inode_eq() and fuse_inode_set().
		- `fuse_init_inode(inode, attr, fc);`
		- `fuse_change_attributes(inode, attr, attr_valid, attr_version);`
	- `fuse_queue_forget()`
		This sends the `FUSE_FORGET` request to backend, it is used to decrease
		the reference of or remove if the reference is 0 open(O_PATH)-ed inode
		in `virtiofsd->inodes` because we somehow fails to open/create the
		corresponding inode in the frontend(guest kernel).

#### conclusion

From the lookup code, we can see the inode mapping in virtiofs is straightforward, which
is: when somehow we need to open a file on the backend at the first time(no matter the case
is we want to access it or access files under it or some other cases), we open(O_PATH) it in
backend and store it in `virtiofsd->inodes`, meanwhile we give it a inode number, the rule of
given inode number is simply increment the number each time. At last we return this inode number
to the frontend within the request reply packet. The frontend stores it in `fuse_inode->inodeid`.

This mapping is quite important since the frontend requests backendfiles by this `inodeid`.


### kill_priv_v2

This argument determines if we drop FSETID before doing `create`, `open`, `write`
and `setattr`.
For example, for `write`, if the frontend gives `WRITE_KILL_PRIV`, and `kill_priv` argument
is given when booting virtiofsd, then we should do that. The triggerring model is similar
for `create` and `open` and `setattr`.
 - `write`
	- `drop_effective_cap("FSETID")`
		```c
			fn drop_effective_cap(cap_name: &str) -> io::Result<Option<ScopedCaps>> {
			    ScopedCaps::new(cap_name)
			}
		```
		- `ScopedCaps::new(cap_name)`
			```c
			    fn new(cap_name: &str) -> io::Result<Option<Self>> {
				use capng::{Action, CUpdate, Set, Type};

				let cap = capng::name_to_capability(cap_name).map_err(|_| {
				    let err = io::Error::last_os_error();
				    error!(
					"couldn't get the capability id for name {}: {:?}",
					cap_name, err
				    );
				    err
				})?;

				if capng::have_capability(Type::EFFECTIVE, cap) {
				    let req = vec![CUpdate {
					action: Action::DROP,
					cap_type: Type::EFFECTIVE,
					capability: cap,
				    }];
				    capng::update(req).map_err(|e| {
					error!("couldn't drop {} capability: {:?}", cap, e);
					einval()
				    })?;
				    capng::apply(Set::CAPS).map_err(|e| {
					error!(
					    "couldn't apply capabilities after dropping {}: {:?}",
					    cap, e
					);
					einval()
				    })?;
				    Ok(Some(Self { cap }))
				} else {
				    Ok(None)
				}
			    }

			```

			The key calls are `capng::update` and `capng::apply`, these two functions
			in rust's capng lib finally call `capng_update` and `capng_apply` in
			c lib `libcap-ng`, and it then calls syscall `capset` to **drop the current
			thread's `CAP_FSETID` capability**.

#### `CAP_FSETID`

```
    CAP_FSETID
           * Don't clear set-user-ID and set-group-ID mode bits when a file is modified;
           * set the set-group-ID bit for a file whose GID does not match the filesystem or  any  of  the
             supplementary GIDs of the calling process.
```

According to the man page of `capabilities`, if `CAP_FSETID` is set, the thread doesn't clear
set-user-ID and set-group-ID bits after modifying a file.
Here set-user-ID and set-group-ID are also called `setuid` and `setgid`. This article explains
these two bits quite clearly: [setuid/setgid](https://www.cbtnuggets.com/blog/technology/system-admin/linux-file-permissions-understanding-setuid-setgid-and-the-sticky-bit)

In short:
- setuid: a bit that makes an executable run with the privileges of the owner of the file

- setgid: a bit that makes an executable run with the privileges of the group of the file

- sticky bit: a bit set on directories that allows only the owner or root can delete files and subdirectories

We can infer `setuid`/`setgid` are good for un-privilege users to run some programs which
need high privileges. For example, `passwd` is for modifying a user's own password,
it modifies `/etc/shadow` file which requests high root priority, but a normal user can succeed
on this because `/bin/passwd` has `suid` bit set. So when a normal user runs `/bin/passwd`,
it is run as root.

Now you may know how the code logic goes if we write/change something to a "suid/sgid" file.
Yes, the `suid`/`sgid` bits is cleared for security reason. Otherwise a malicious normal user
can change a "suid/sgid" file to a malicious binary and run it with root privilege.
So now it's very clear why virtiofsd clears `CAP_FSETID` before `write` operation, because
this cap flag keeps `suid`/`sgid` bits after changing a file, it's not safe if we are change
a executable file and the user changing it is a normal user.


#### Review some useful knowledge

- How to know if a file is set with `setuid`/`setgid` bit, or if a directory is set with
`sticky` bit?
Just `ls -l` a file, you can see something like `-rwxrwxrwx` for the file's permission.
If `setuid` bit is set: `rwsrwxrwx`
If `setgid` bit is set: `rwxrwsrwx`
If `sticky` bit is set: `rwxrwxrwt`

- What does the permission bits mean for a file

There are three groups: `owner, group, other`. And for each object, there are three
permissions: `read, write, execute`, aka, `r, w, x`.

	- For a file:
		- `r`: you can read the file
		- `w`: you can write the file(not include deleting the file)
		- `x`: you can execute the file

	- For a directory:
		- `r`: you can read the directory structure
		- `w`: you can modify the directory structure(add/remove files)
		- `x`: you can enter this directory

**one thing to notice here is the permission to remove/delete a file is on its parent directory
not itself**
