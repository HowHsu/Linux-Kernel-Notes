# virtiofsd

This article is about the backend of virtiofs: virtiofsd. And here I'll
pick up the rust version as an instance, while I'm also new to this languagethough. I'm going to talk about the big picture of virtiofsd as well as
diving deeply into the code to make you have a full sense of it.

## 1. framework

Let's first look at a graph which shows the relationship of the main classes
 of virtiofsd.

![](./images/virtiofsd.drawio.svg)



## 2. cache policy

- guest kernel page cache: GCache
- host kern el page cache: HCache

- `O_DIRECT`: vfs flag, users can set it when opening a file
- `FOPEN_DIRECT_IO`: FUSE flag, FUSE hand over the `bypass pagecache` control to
backend fs of FUSE. Specifically for virtiofs, it all depends on `CachePolicy`,
**once a file is opened, `FOPEN_DIRECT_IO = !is_dir && CachePolicy::Never`**

- `--allow-direct-io true`
This means we respect the original IO flag(vfs flag) of how we leverage the host
page cache

```
vfs\fuse    FOPEN_DIRECT_IO(--cache=never)	      !FOPEN_DIRECT_IO(--cache=auto/always)
O_DIRECT        !GCache   !HCache                      !GCache  !HCache
!O_DIRECT	!GCache    HCache                      GCache   HCache
```

Here note the `O_DIRECT && !FOPEN_DIRECT_IO` case, `FOPEN_DIRECT_IO` is for buffered IO.
**Just regard the `FOPEN_DIRECT_IO` as another restriction comes from FUSE backend
fs, which obeys the frontend `O_DIRECT` semantics**
To figure out the guest pagecache usage, just follow:

- `O_DIRECT` is set, direct IO, bypass pagecache.
- `O_DIRECT` isn't set, buffered IO, wait, check fuse flag
	- `FOPEN_DIRECT_IO` is set, ok, backend restriction, bypass pagecache
	- `FOPEN_DIRECT_IO` isn't set, ok, real buffered IO


### a more clear inclusion

for guest page cache:

`--cache=never`: we don't use guest page cache anyway!
`--cache=auto/always`: we use guest page cache for buffered IO

for host page cache:
`!O_DIRECT`: we use host page cache anyway!
`O_DIRECT`: we use host page cache if `--allow-direct-io=false`

## 3. inode mapping

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

### 3.1 lookup

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
    <br />
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

### 3.2 conclusion

From the lookup code, we can see the inode mapping in virtiofs is straightforward, which
is: when somehow we need to open a file on the backend at the first time(no matter the case
is we want to access it or access files under it or some other cases), we open(O_PATH) it in
backend and store it in `virtiofsd->inodes`, meanwhile we give it a inode number, the rule of
given inode number is simply increment the number each time. At last we return this inode number
to the frontend within the request reply packet. The frontend stores it in `fuse_inode->inodeid`.

This mapping is quite important since the frontend requests backendfiles by this `inodeid`.


## 4. kill_priv_v2

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

### 4.1 `CAP_FSETID`

```
    CAP_FSETID
           * Don't clear set-user-ID and set-group-ID mode bits when a file is modified;
           * set the set-group-ID bit for a file whose GID does not match the filesystem or  any  of  the
             supplementary GIDs of the calling process.
```

According to the man page of `capabilities`, if `CAP_FSETID` is set, the thread doesn't clear
`set-user-ID` and `set-group-ID` bits after modifying a file.

The logic without `CAP_FSETID` is: if we write/change something to a "SUID/SGID" file,
the `SUID`/`SGID` bits are cleared for security reason. Otherwise a malicious normal user
may possibly leverage a changed "suid/sgid" file to access high privilege resources.
So now it's very clear why virtiofsd clears `CAP_FSETID` before `write` operation, because
this cap flag keeps `suid`/`sgid` bits after changing a file, it's not safe if we are change
a executable file.
More detail about `SUID` and `SGID` is here: [SUID/SGID](../user_group_permission.md#3-suid-bit--sgid-bit--sticky-bit)

#### why drop_supplemental_groups()
It's related with `SGID`: https://gitlab.com/virtio-fs/virtiofsd/-/merge_requests/77

The example in the url link is:
- a normal user userA runs virtiofsd, but it has a supplemental group which at least has `CAP_FSETID` capability.
- a directory dirA whose group is root and has `SGID` bit, which means files created in it will has same group as A's
- the guest creates a binary file fileA with `SGID` in dirA(because userA has supplemental group with `CAP_FSETID` capability)
- now a normal user userA can execute a fileA with root group privilege. Because fileA has `SGID` and its group is root.

## 5. FsOptions

`FsOptions` indicates features the filesystem supports. Notice, here it's an intersection
of guest kernel fs and user input argument. Let's look into the `INIT` code
to see where it comes from.

```rust
    fn init(&self, in_header: InHeader, mut r: Reader, w: Writer) -> Result<usize> {
        let InitInCompat {
            major,
            minor,
            max_readahead,
            flags,
        } = r.read_obj().map_err(Error::DecodeMessage)?;

	    // we can see the option value is from frontend.
        let options = FsOptions::from_bits_truncate(flags as u64);

	    // there is flags2 if INIT_EXT member is in FsOption
        let InitInExt { flags2, .. } = if options.contains(FsOptions::INIT_EXT) {
            r.read_obj().map_err(Error::DecodeMessage)?
        } else {
            InitInExt::default()
        };

        // These fuse features are supported by this server by default.
        let supported = FsOptions::ASYNC_READ
            | FsOptions::PARALLEL_DIROPS
            | FsOptions::BIG_WRITES
            | FsOptions::AUTO_INVAL_DATA
            | FsOptions::ASYNC_DIO
            | FsOptions::HAS_IOCTL_DIR
            | FsOptions::ATOMIC_O_TRUNC
            | FsOptions::MAX_PAGES
            | FsOptions::SUBMOUNTS
            | FsOptions::INIT_EXT;

	    // merge flags and flags2 to flags_64 and transform it to FsOption
        let flags_64 = ((flags2 as u64) << 32) | (flags as u64);
        let capable = FsOptions::from_bits_truncate(flags_64);

        match self.fs.init(capable) {
            Ok(want) => {
                /*
                * want: intersection of guest kernel supported and user want features.
                * supported: fuse default features
                * capable: guest kernel supported features.
                *
                * Here want is already what we want, doing an extra & is for safety in
                * future I guess.
                */
                let enabled = (capable & (want | supported)).bits();
                self.options.store(enabled, Ordering::Relaxed);

                let out = InitOut {
                    major: KERNEL_VERSION,
                    minor: KERNEL_MINOR_VERSION,
                    max_readahead,
                    flags: enabled as u32,
                    max_background: u16::MAX,
                    congestion_threshold: (u16::MAX / 4) * 3,
                    max_write: MAX_BUFFER_SIZE,
                    time_gran: 1, // nanoseconds
                    max_pages: max_pages.try_into().unwrap(),
                    map_alignment: 0,
                    flags2: (enabled >> 32) as u32,
                    ..Default::default()
                };

                reply_ok(Some(out), None, in_header.unique, w)
            }
            Err(e) => reply_error(e, in_header.unique, w),
        }
    }

```

I've added some comments in above code, should be quite self-explained.
Let's look into `self.fs.init()`

```rust
    fn init(&self, capable: FsOptions) -> io::Result<FsOptions> {
	...
	...
	...

        let mut opts = if self.cfg.readdirplus {
            FsOptions::DO_READDIRPLUS | FsOptions::READDIRPLUS_AUTO
        } else {
            FsOptions::empty()
        };
        if self.cfg.writeback && capable.contains(FsOptions::WRITEBACK_CACHE) {
            opts |= FsOptions::WRITEBACK_CACHE;
            self.writeback.store(true, Ordering::Relaxed);
        }
        if self.cfg.announce_submounts {
            if capable.contains(FsOptions::SUBMOUNTS) {
                self.announce_submounts.store(true, Ordering::Relaxed);
            } else {
                eprintln!("Warning: Cannot announce submounts, client does not support it");
            }
        }
        if self.cfg.killpriv_v2 {
            if capable.contains(FsOptions::HANDLE_KILLPRIV_V2) {
                opts |= FsOptions::HANDLE_KILLPRIV_V2;
            } else {
                warn!("Cannot enable KILLPRIV_V2, client does not support it");
            }
        }
        if self.cfg.posix_acl {
            let acl_required_flags =
                FsOptions::POSIX_ACL | FsOptions::DONT_MASK | FsOptions::SETXATTR_EXT;
            if capable.contains(acl_required_flags) {
                opts |= acl_required_flags;
                self.posix_acl.store(true, Ordering::Relaxed);
                debug!("init: enabling posix acl");
            } else {
                error!("Cannot enable posix ACLs, client does not support it");
                return Err(io::Error::from_raw_os_error(libc::EPROTO));
            }
        }

        if self.cfg.security_label {
            if capable.contains(FsOptions::SECURITY_CTX) {
                opts |= FsOptions::SECURITY_CTX;
            } else {
                error!("Cannot enable security label. kernel does not support FUSE_SECURITY_CTX capability");
                return Err(io::Error::from_raw_os_error(libc::EPROTO));
            }
        }
        Ok(opts)
    }

```

**Here I'm confused why virtiofsd doesn't check if the host filesystem support those features**

Let's have a look at all the options virtiofsd may support.

```rust
bitflags! {
    /// A bitfield passed in as a parameter to and returned from the `init` method of the
    /// `FileSystem` trait.
    pub struct FsOptions: u64 {
        /// Indicates that the filesystem supports asynchronous read requests.
        ///
        /// If this capability is not requested/available, the kernel will ensure that there is at
        /// most one pending read request per file-handle at any time, and will attempt to order
        /// read requests by increasing offset.
        ///
        /// This feature is enabled by default when supported by the kernel.
        const ASYNC_READ = ASYNC_READ;

        /// Indicates that the filesystem supports "remote" locking.
        ///
        /// This feature is not enabled by default and should only be set if the filesystem
        /// implements the `getlk` and `setlk` methods of the `FileSystem` trait.
        const POSIX_LOCKS = POSIX_LOCKS;

        /// Kernel sends file handle for fstat, etc... (not yet supported).
        const FILE_OPS = FILE_OPS;

        /// Indicates that the filesystem supports the `O_TRUNC` open flag. If disabled, and an
        /// application specifies `O_TRUNC`, fuse first calls `setattr` to truncate the file and
        /// then calls `open` with `O_TRUNC` filtered out.
        ///
        /// This feature is enabled by default when supported by the kernel.
        const ATOMIC_O_TRUNC = ATOMIC_O_TRUNC;

        /// Indicates that the filesystem supports lookups of "." and "..".
        ///
        /// This feature is disabled by default.
        const EXPORT_SUPPORT = EXPORT_SUPPORT;

        /// FileSystem can handle write size larger than 4kB.
        const BIG_WRITES = BIG_WRITES;

        /// Indicates that the kernel should not apply the umask to the file mode on create
        /// operations.
        ///
        /// This feature is disabled by default.
        const DONT_MASK = DONT_MASK;

        /// Indicates that the server should try to use `splice(2)` when writing to the fuse device.
        /// This may improve performance.
        ///
        /// This feature is not currently supported.
        const SPLICE_WRITE = SPLICE_WRITE;

        /// Indicates that the server should try to move pages instead of copying when writing to /
        /// reading from the fuse device. This may improve performance.
        ///
        /// This feature is not currently supported.
        const SPLICE_MOVE = SPLICE_MOVE;

        /// Indicates that the server should try to use `splice(2)` when reading from the fuse
        /// device. This may improve performance.
        ///
        /// This feature is not currently supported.
        const SPLICE_READ = SPLICE_READ;

        /// If set, then calls to `flock` will be emulated using POSIX locks and must
        /// then be handled by the filesystem's `setlock()` handler.
        ///
        /// If not set, `flock` calls will be handled by the FUSE kernel module internally (so any
        /// access that does not go through the kernel cannot be taken into account).
        ///
        /// This feature is disabled by default.
        const FLOCK_LOCKS = FLOCK_LOCKS;

        /// Indicates that the filesystem supports ioctl's on directories.
        ///
        /// This feature is enabled by default when supported by the kernel.
        const HAS_IOCTL_DIR = HAS_IOCTL_DIR;

        /// Traditionally, while a file is open the FUSE kernel module only asks the filesystem for
        /// an update of the file's attributes when a client attempts to read beyond EOF. This is
        /// unsuitable for e.g. network filesystems, where the file contents may change without the
        /// kernel knowing about it.
        ///
        /// If this flag is set, FUSE will check the validity of the attributes on every read. If
        /// the attributes are no longer valid (i.e., if the *attribute* timeout has expired) then
        /// FUSE will first send another `getattr` request. If the new mtime differs from the
        /// previous value, any cached file *contents* will be invalidated as well.
        ///
        /// This flag should always be set when available. If all file changes go through the
        /// kernel, *attribute* validity should be set to a very large number to avoid unnecessary
        /// `getattr()` calls.
        ///
        /// This feature is enabled by default when supported by the kernel.
        const AUTO_INVAL_DATA = AUTO_INVAL_DATA;

        /// Indicates that the filesystem supports readdirplus.
        ///
        /// The feature is not enabled by default and should only be set if the filesystem
        /// implements the `readdirplus` method of the `FileSystem` trait.
        const DO_READDIRPLUS = DO_READDIRPLUS;

        /// Indicates that the filesystem supports adaptive readdirplus.
        ///
        /// If `DO_READDIRPLUS` is not set, this flag has no effect.
        ///
        /// If `DO_READDIRPLUS` is set and this flag is not set, the kernel will always issue
        /// `readdirplus()` requests to retrieve directory contents.
        ///
        /// If `DO_READDIRPLUS` is set and this flag is set, the kernel will issue both `readdir()`
        /// and `readdirplus()` requests, depending on how much information is expected to be
        /// required.
        ///
        /// This feature is not enabled by default and should only be set if the file system
        /// implements both the `readdir` and `readdirplus` methods of the `FileSystem` trait.
        const READDIRPLUS_AUTO = READDIRPLUS_AUTO;

        /// Indicates that the filesystem supports asynchronous direct I/O submission.
        ///
        /// If this capability is not requested/available, the kernel will ensure that there is at
        /// most one pending read and one pending write request per direct I/O file-handle at any
        /// time.
        ///
        /// This feature is enabled by default when supported by the kernel.
        const ASYNC_DIO = ASYNC_DIO;

        /// Indicates that writeback caching should be enabled. This means that individual write
        /// request may be buffered and merged in the kernel before they are sent to the file
        /// system.
        ///
        /// This feature is disabled by default.
        const WRITEBACK_CACHE = WRITEBACK_CACHE;

        /// Indicates support for zero-message opens. If this flag is set in the `capable` parameter
        /// of the `init` trait method, then the file system may return `ENOSYS` from the open() handler
        /// to indicate success. Further attempts to open files will be handled in the kernel. (If
        /// this flag is not set, returning ENOSYS will be treated as an error and signaled to the
        /// caller).
        ///
        /// Setting (or not setting) the field in the `FsOptions` returned from the `init` method
        /// has no effect.
        const ZERO_MESSAGE_OPEN = NO_OPEN_SUPPORT;

        /// Indicates support for parallel directory operations. If this flag is unset, the FUSE
        /// kernel module will ensure that lookup() and readdir() requests are never issued
        /// concurrently for the same directory.
        ///
        /// This feature is enabled by default when supported by the kernel.
        const PARALLEL_DIROPS = PARALLEL_DIROPS;

        /// Indicates that the file system is responsible for unsetting setuid and setgid bits when a
        /// file is written, truncated, or its owner is changed.
        ///
        /// This feature is not currently supported.
        const HANDLE_KILLPRIV = HANDLE_KILLPRIV;

        /// Indicates support for POSIX ACLs.
        ///
        /// If this feature is enabled, the kernel will cache and have responsibility for enforcing
        /// ACLs. ACL will be stored as xattrs and passed to userspace, which is responsible for
        /// updating the ACLs in the filesystem, keeping the file mode in sync with the ACL, and
        /// ensuring inheritance of default ACLs when new filesystem nodes are created. Note that
        /// this requires that the file system is able to parse and interpret the xattr
        /// representation of ACLs.
        ///
        /// Enabling this feature implicitly turns on the `default_permissions` mount option (even
        /// if it was not passed to mount(2)).
        ///
        /// This feature is disabled by default.
        const POSIX_ACL = POSIX_ACL;

        /// Indicates that if the connection is gone because of sysfs abort, reading from the device
        /// will return -ECONNABORTED.
        ///
        /// This feature is not currently supported.
        const ABORT_ERROR = ABORT_ERROR;

        /// Indicates support for negotiating the maximum number of pages supported.
        ///
        /// If this feature is enabled, we can tell the kernel the maximum number of pages that we
        /// support to transfer in a single request.
        ///
        /// This feature is enabled by default if supported by the kernel.
        const MAX_PAGES = MAX_PAGES;

        /// Indicates that the kernel supports caching READLINK responses.
        ///
        /// This feature is not currently supported.
        const CACHE_SYMLINKS = CACHE_SYMLINKS;

        /// Indicates support for zero-message opens. If this flag is set in the `capable` parameter
        /// of the `init` trait method, then the file system may return `ENOSYS` from the opendir() handler
        /// to indicate success. Further attempts to open directories will be handled in the kernel. (If
        /// this flag is not set, returning ENOSYS will be treated as an error and signaled to the
        /// caller).
        ///
        /// Setting (or not setting) the field in the `FsOptions` returned from the `init` method
        /// has no effect.
        const ZERO_MESSAGE_OPENDIR = NO_OPENDIR_SUPPORT;

        /// Indicates support for explicit data invalidation. If this feature is enabled, the
        /// server is fully responsible for data cache invalidation, and the kernel won't
        /// invalidate files data cache on size change and only truncate that cache to new size
        /// in case the size decreased.
        ///
        /// This feature is not currently supported.
        const EXPLICIT_INVAL_DATA = EXPLICIT_INVAL_DATA;

        /// Indicates that the kernel supports the FUSE_ATTR_SUBMOUNT flag.
        ///
        /// Setting (or not setting) this flag in the `FsOptions` returned from the `init` method
        /// has no effect.
        const SUBMOUNTS = SUBMOUNTS;

        /// Indicates that the filesystem is responsible for clearing
        /// security.capability xattr and clearing setuid and setgid bits. Following
        /// are the rules.
        /// - clear "security.capability" on write, truncate and chown unconditionally
        /// - clear suid/sgid if following is true. Note, sgid is cleared only if
        ///   group executable bit is set.
        ///    o setattr has FATTR_SIZE and FATTR_KILL_SUIDGID set.
        ///    o setattr has FATTR_UID or FATTR_GID
        ///    o open has O_TRUNC and FUSE_OPEN_KILL_SUIDGID
        ///    o create has O_TRUNC and FUSE_OPEN_KILL_SUIDGID flag set.
        ///    o write has FUSE_WRITE_KILL_SUIDGID
        ///
        /// This feature is enabled by default if supported by the kernel.
        const HANDLE_KILLPRIV_V2 = HANDLE_KILLPRIV_V2;

        /// Server supports extended struct SetxattrIn
        const SETXATTR_EXT = SETXATTR_EXT;

        /// Indicates that fuse_init_in structure has been extended and
        /// expect extended struct coming in from kernel.
        const INIT_EXT = INIT_EXT;

        /// This bit is reserved. Don't use it.
        const INIT_RESERVED = INIT_RESERVED;

        /// Indicates that kernel is capable of sending a security
        /// context at file creation time (create, mkdir, symlink
        /// and mknod). This is expected to be a SELinux security
        /// context as of now.
        const SECURITY_CTX = SECURITY_CTX;

        /// Indicates that kernel is capable of understanding
        /// per inode dax flag sent in response to getattr
        /// request. This will allow server to enable to
        /// enable dax on selective files.
         const HAS_INODE_DAX = HAS_INODE_DAX;
    }
}

```

I'll analyze these options one by one in the coming few days.

### 5.1 `SECURITY_CTX`

condition: cfg.security_label = true && guest kernel fs supports it

Just like the comment says: the guest kernel sends a secuirty context at file
creation time to backend, and it is stored as xattr in inode. It is SELinux
security context as of now.
Pick up `mkdir` for an example:
- guest kernel:

    ```c
        fuse_mkdir()
            create_new_entry()
                get_create_ext()
    ```

    ```c
        static int get_create_ext(struct fuse_args *args,
                        struct inode *dir, struct dentry *dentry,
                        umode_t mode)
        {
            struct fuse_conn *fc = get_fuse_conn_super(dentry->d_sb);
            struct fuse_in_arg ext = { .size = 0, .value = NULL };
            int err = 0;

            if (fc->init_security)
                err = get_security_context(dentry, mode, &ext);
            if (!err && fc->create_supp_group)
                err = get_create_supp_group(dir, &ext);

            if (!err && ext.size) {
                WARN_ON(args->in_numargs >= ARRAY_SIZE(args->in_args));
                args->is_ext = true;
                args->ext_idx = args->in_numargs++;
                args->in_args[args->ext_idx] = ext;
            } else {
                kfree(ext.value);
            }

            return err;
        }

    ```

    `ext` here is the secctx we need, it will be sent to backend within the
    `mkdir` request and finally be written to the host filesystem by `setxattr()`
    `ext` is derived from `get_security_context()`, this function calls
    `security_dentry_init_security()` in `security/security.c` to init the
    content of secctx. I'm not familiar with SELinux part. Will suspend it for now.

- backend(virtiofsd)
    Let's skip over the request decoding stage and just jump into the
    passthroughfs logic.

    ```rust
        fn mkdir(
        &self,
        ctx: Context,
        parent: Inode,
        name: &CStr,
        mode: u32,
        umask: u32,
        secctx: Option<SecContext>,
        ) -> io::Result<Entry> {

        ...
        ...

        // Set security context on dir.
        if let Some(secctx) = secctx {
            if let Err(e) = self.do_mknod_mkdir_symlink_secctx(&parent_file, name, &secctx) {
            unsafe {
                libc::unlinkat(parent_file.as_raw_fd(), name.as_ptr(), libc::AT_REMOVEDIR);
            };
            return Err(e);
            }
        }

        self.do_lookup(parent, name)
        }

    ```

    ```rust
        fn do_mknod_mkdir_symlink_secctx(
        &self,
        parent_file: &InodeFile,
        name: &CStr,
        secctx: &SecContext,
        ) -> io::Result<()> {
        // Remap security xattr name.
        let xattr_name = self.map_client_xattrname(&secctx.name)?;

        // Set security context on newly created node. It could be
        // device node as well, so it is not safe to open the node
        // and call fsetxattr(). Instead, use the fchdir(proc_fd)
        // and call setxattr(o_path_fd). We use this trick while
        // setting xattr as well.

        // Open O_PATH fd for dir/symlink/special node just created.
        let path_fd = self.open_relative_to(parent_file, name, libc::O_PATH, None)?;

        let procname = CString::new(format!("{path_fd}"))
            .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e));

        let procname = match procname {
            Ok(name) => name,
            Err(error) => {
            return Err(error);
            }
        };

        let _working_dir_guard =
            set_working_directory(self.proc_self_fd.as_raw_fd(), self.root_fd.as_raw_fd());

        let res = unsafe {
            libc::setxattr(
            procname.as_ptr(),
            xattr_name.as_ptr(),
            secctx.secctx.as_ptr() as *const libc::c_void,
            secctx.secctx.len(),
            0,
            )
        };

        let res_err = io::Error::last_os_error();

        if res == 0 {
            Ok(())
        } else {
            Err(res_err)
        }
        }
    ```

    As you can see, virtiofsd finally calls `setxattr` to set the secctx
    to file/inode in the host fs file.
    **one thing to notice is before this, it first checks `xattrmap` and
    do the rename/transforming for the xattr name.**

## 6. user/group mapping

### 6.1 uid/gid of files in shared_dir
This topic is about user and group a file created in virtiofsd's shared_dir belongs to.

- think about one problem: a user in guest creates a file in the shared_dir, there is/isn't a same(uid) user on the host, what does this file belong to when you do `ls -l /path/to/file` on host?

To answer this question, we have to figure out how virtiofsd works when creating a file.
Say virtiofsd is run by root, `user=user1, group=group1` in guest creates a new file in shared_dir, what does virtiofsd do after it receive this request from guest? just create the file?
No, we surely cannot do that directly, since then the user
and group of that created file will be root and root, while it should be user1 and group1.
So virtiofsd has to switch its UID and GID before doing
the create file stuff. A simple function call process in
virtiofsd is like this:

```rust
server.create()
    self.fs.create()
        self.do_create()
            set_creds(ctx.uid, ctx.gid)?;
                SYS_setresgid;
                SYS_setresuid;
            self.open_relative_to();
            set_creds(0, 0)
```

Here `ctx` is from the guest request. `ctx.uid` and `ctx.gid` are the `uid` and `gid` of the task who triggers this mission in guest userspace.
So it clearly shows what happens in virtiofsd:
- 1.switch eUID to `ctx.uid` and eGID to `ctx.gid`
- 2.call create syscall
- 3.switch eUID and eGID back.

**Notice, the step 3 isn't done by `set_cred(0, 0)`, it is
done by Drop() of `ScopedUid` and `ScopedGid`, which happens automatically when the function ends.**

- another quesion: `uid=1000, name=Alice` in guest create a file, and there is `uid=1000, name=Bob` on host.
What shows when you type `ls -l` on host?

**It's Bob not Alice, because Linux store the UID in inode not the user name. And when `ls` goes, UID=1000 maps 'Bob' on host, so it outputs 'Bob'**
**Notice, if the UID doesn't exist on host, `ls -l` shows the UID directly**

```c
struct inode {                                                                   
        umode_t                 i_mode;                                          
        unsigned short          i_opflags;                                       
        kuid_t                  i_uid;                                           
        kgid_t                  i_gid;                                           
        unsigned int            i_flags;                                         
        ...
        ...
        ...
}

```

## 7. announce_submounts

Virtiofsd can only share one directory to the guest, but there can be multiple filesystems in the `shared_dir`. What if two files in two filesystems have same inode number and the guest don't know they are in different filesystems? That may cause issues.
The `announce_submounts` argument is to resolve this. It
tells the guest that a directory in `shared_dir` is a mount point. It is currently turn on by default.(after commit bd5fe483ee482)
It works only in `lookup` because that's the place we first touch a file.

```rust
    fn do_lookup(&self, parent: Inode, name: &CStr) -> io::Result<Entry> {
         let p = self
            .inodes
            .read()
            .unwrap()
            .get(&parent)
            .map(Arc::clone)
            .ok_or_else(ebadf)?;

        let p_file = p.get_file()?;

        let path_fd = {
            let fd = self.open_relative_to(&p_file, name, libc::O_PATH, None)?;
            // Safe because we just opened this fd.
            unsafe { File::from_raw_fd(fd) }
        };

        let st = statx(&path_fd, None)?;

        let mut attr_flags: u32 = 0;

        if st.st.st_mode & libc::S_IFMT == libc::S_IFDIR
            && self.announce_submounts.load(Ordering::Relaxed)
            && (st.st.st_dev != p.ids.dev || st.mnt_id != p.ids.mnt_id)
        {
            attr_flags |= fuse::ATTR_SUBMOUNT;
        }

	...
	...
	...

        Ok(Entry {
            inode,
            generation: 0,
            attr: st.st,
            attr_flags,
            attr_timeout: self.cfg.attr_timeout,
            entry_timeout: self.cfg.entry_timeout,
        })
    }


```

I've delete unrelated code above. From the code you can see the steps are:
- open the requested file with `O_PATH`
- get the state of the file by statx()
- if the file **is a dir** and **announce_submounts is on** and **its mnt id !=parent's mnt id**, mark `fuse::ATTR_SUBMOUNT` on `attr_flags`
- `attr_flags` is sent back to the guest kernel

Let's turn to the guest kernel side:
`fuse_lookup() --> fuse_lookup_name() --> fuse_iget()`

```c
struct inode *fuse_iget(struct super_block *sb, u64 nodeid,
 			int generation, struct fuse_attr *attr,
			u64 attr_valid, u64 attr_version)
{
	struct inode *inode;
	struct fuse_inode *fi;
	struct fuse_conn *fc = get_fuse_conn_super(sb);

	/*
	 * Auto mount points get their node id from the submount root, which is
	 * not a unique identifier within this filesystem.
	 *
	 * To avoid conflicts, do not place submount points into the inode hash
	 * table.
	 */
	if (fc->auto_submounts && (attr->flags & FUSE_ATTR_SUBMOUNT) &&
	    S_ISDIR(attr->mode)) {
		inode = new_inode(sb);
		if (!inode)
			return NULL;

		fuse_init_inode(inode, attr, fc);
		get_fuse_inode(inode)->nodeid = nodeid;
		inode->i_flags |= S_AUTOMOUNT;
 		goto done;
 	}
	...
	...
	...
}

```

`fc->auto_submounts` is always true for virtiofs, `FUSE_ATTR_SUBMOUNT` is same as
`fuse::ATTR_SUBMOUNT`.

