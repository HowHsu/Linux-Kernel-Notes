# mount_lock

## Definition

```c
/*
 * vfsmount lock may be taken for read to prevent changes to the
 * vfsmount hash, ie. during mountpoint lookups or walking back
 * up the tree.
 *
 * It should be taken for write in all cases where the vfsmount
 * tree or hash is modified or when a vfsmount structure is modified.
 */
  __cacheline_aligned_in_smp DEFINE_SEQLOCK(mount_lock);

  #define DEFINE_SEQLOCK(sl) \                                                         
                  seqlock_t sl = __SEQLOCK_UNLOCKED(sl)                                

  #define __SEQLOCK_UNLOCKED(lockname)                                    \        
          {                                                               \        
                  .seqcount = SEQCNT_SPINLOCK_ZERO(lockname, &(lockname).lock), \  
                  .lock = __SPIN_LOCK_UNLOCKED(lockname)                  \        
          }                                                                        

```

### `__SEQLOCK_UNLOCKED`

Expand it to get:
```c
seq_lock_t mount_lock = {
          .seqcount = SEQCNT_SPINLOCK_ZERO(mount_lock, &mount_lock.lock),
          .lock = __SPIN_LOCK_UNLOCKED(mount_lock)
}

```

### `SEQCNT_SPINLOCK_ZERO`

```c
  #define SEQCNT_SPINLOCK_ZERO(name, lock)        SEQCOUNT_LOCKNAME_ZERO(name, lock)
  #define SEQCOUNT_LOCKNAME_ZERO(seq_name, assoc_lock) {                  \ 
          .seqcount               = SEQCNT_ZERO(seq_name.seqcount),       \ 
          __SEQ_LOCK(.lock        = (assoc_lock))                         \ 
  }
```

So seqcount is:
```c
	.seqcount = {
		.seqcount = SEQCNT_ZERO(mount_lock.seqcount),
		__SEQ_LOCK(.lock = &mount_lock.lock)
	}
```

#### `SEQCNT_ZERO`

```c

  #define SEQCNT_ZERO(name) { .sequence = 0, SEQCOUNT_DEP_MAP_INIT(name) }         
  #define SEQCOUNT_DEP_MAP_INIT(lockname)                                \        
                  .dep_map = { .name = #lockname }                                 
```

expand it to be:
```c
#define SEQCNT_ZERO(name) { .sequence = 0, .dep_map = { .name = name} }
```

#### `__SEQ_LOCK()`

```c
#if defined(CONFIG_LOCKDEP) || defined(CONFIG_PREEMPT_RT)                        
  #define __SEQ_LOCK(expr)        expr                                             
  #else                                                                            
  #define __SEQ_LOCK(expr)                                                         
  #endif
```

It is empty in normal case.

So again, seqcount becomes:
```c
	.seqcount = {
		.seqcount = {
			.sequence = 0,
			.dep_map = { .name = mount_lock.seqcount }
		}
	}
```

### `__SPIN_LOCK_UNLOCKED`

```c
  #define ___SPIN_LOCK_INITIALIZER(lockname)      \                                
          {                                       \                                
          .raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,  \                                
          SPIN_DEBUG_INIT(lockname)               \                                
          SPIN_DEP_MAP_INIT(lockname) }                                            
                                                                                   
  #define __SPIN_LOCK_INITIALIZER(lockname) \                                      
          { { .rlock = ___SPIN_LOCK_INITIALIZER(lockname) } }                      
                                                                                   
  #define __SPIN_LOCK_UNLOCKED(lockname) \                                         
          (spinlock_t) __SPIN_LOCK_INITIALIZER(lockname)                           

```

expand it we can see `.lock` is:
```c
	.lock = (spinlock_t) {
		{
			.rlock = {
				.raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,
          			SPIN_DEBUG_INIT(lockname)               \
          			SPIN_DEP_MAP_INIT(lockname)
			}
		}
	}
```

`SPIN_DEBUG_INIT` and `SPIN_DEP_MAP_INIT` are empty in non debug mode.
`__ARCH_SPIN_LOCK_UNLOCKED` is arch related, for x86, it is 0. So `.lock` can
be simplified to:

```c
	.lock = (spinlock_t) {
		{
			.rlock = {
				.raw_lock = 0,
			}
		}
	}
```

Finally we can get:
```c
seq_lock_t mount_lock = {
	.seqcount = {
		.seqcount = {
			.sequence = 0,
			.dep_map = { .name = mount_lock.seqcount }
		}
	},

	.lock = (spinlock_t) {
		.rlock = { .raw_lock = 0 }
	}
}

```

Here you can see a seqlock is a integer count plus a spinlock. seqlock is widely
used in linux kernel, it is for access mode of 'frequent reads, bare writes' and
'write has higher priority'. Here the spinlock is to sync writes, while seqcount
is to sync between reads and writes. An odd number means there is a writer.

## Why we need mount_lock

I'll give the conclusion at the end of this section. To figure out this question
let's read the code step by step.
Note: For some reason, I only analyze bind mount/umount code path for simplicity.

### Writer

From what I get, mount_lock is only writer hold in below two functions:
```c
  static inline void lock_mount_hash(void)                                             
  {                                                                                    
          write_seqlock(&mount_lock);                                                  
  }                                                                                    
                                                                                       
  static inline void unlock_mount_hash(void)                                           
  {                                                                                    
          write_sequnlock(&mount_lock);                                                
  }                                                                                    
 
```

Let's first track the writers.

- `__legitimize_mnt()`
The call chain is `do_loopback` -> `lock_mount` -> `lookup_mnt` -> `legitimize_mnt` -> `__legitimize_mnt`
```c
lock_mount_hash();
if (unlikely(bastard->mnt_flags & MNT_DOOMED)) {
	mnt_add_count(mnt, -1);
	unlock_mount_hash();
	return 1;
}
unlock_mount_hash();

```

- `clone_mnt()`
The call chain is `path_mount` -> `do_loopback` -> `__do_loopback` -> `clone_mnt`
```c
	lock_mount_hash();
	list_add_tail(&mnt->mnt_instance, &sb->s_mounts);
	unlock_mount_hash();
```

- `attach_recursive_mnt()`

The call chain is `path_mount` -> `do_loopback` -> `graft_tree` -> `attach_recursive_mnt`
**This one includes multiple mount_lock holding code, both as writer and as exclusive reader**
**exclusive reader is equal to writer though it only does read.**
```c
static int attach_recursive_mnt(struct mount *source_mnt,
			struct mount *dest_mnt,
			struct mountpoint *dest_mp,
			bool moving)
{
	struct user_namespace *user_ns = current->nsproxy->mnt_ns->user_ns;
	HLIST_HEAD(tree_list);
	struct mnt_namespace *ns = dest_mnt->mnt_ns;
	struct mountpoint *smp;
	struct mount *child, *p;
	struct hlist_node *n;
	int err;

	/* Preallocate a mountpoint in case the new mounts need
	 * to be tucked under other mounts.
	 */
	smp = get_mountpoint(source_mnt->mnt.mnt_root);
	if (IS_ERR(smp))
		return PTR_ERR(smp);

	/* Is there space to add these mounts to the mount namespace? */
	if (!moving) {
		err = count_mounts(ns, source_mnt);
		if (err)
			goto out;
	}

	if (IS_MNT_SHARED(dest_mnt)) {
		err = invent_group_ids(source_mnt, true);
		if (err)
			goto out;
		err = propagate_mnt(dest_mnt, dest_mp, source_mnt, &tree_list);
		lock_mount_hash();
		if (err)
			goto out_cleanup_ids;
		for (p = source_mnt; p; p = next_mnt(p, source_mnt))
			set_mnt_shared(p);
	} else {
		lock_mount_hash();
	}
	if (moving) {
		unhash_mnt(source_mnt);
		attach_mnt(source_mnt, dest_mnt, dest_mp);
		touch_mnt_namespace(source_mnt->mnt_ns);
	} else {
		if (source_mnt->mnt_ns) {
			/* move from anon - the caller will destroy */
			list_del_init(&source_mnt->mnt_ns->list);
		}
		mnt_set_mountpoint(dest_mnt, dest_mp, source_mnt);
		commit_tree(source_mnt);
	}

	hlist_for_each_entry_safe(child, n, &tree_list, mnt_hash) {
		struct mount *q;
		hlist_del_init(&child->mnt_hash);
		q = __lookup_mnt(&child->mnt_parent->mnt,
				 child->mnt_mountpoint);
		if (q)
			mnt_change_mountpoint(child, smp, q);
		/* Notice when we are propagating across user namespaces */
		if (child->mnt_parent->mnt_ns->user_ns != user_ns)
			lock_mnt_tree(child);
		child->mnt.mnt_flags &= ~MNT_LOCKED;
		commit_tree(child);
	}
	put_mountpoint(smp);
	unlock_mount_hash();

	return 0;

 out_cleanup_ids:
	while (!hlist_empty(&tree_list)) {
		child = hlist_entry(tree_list.first, struct mount, mnt_hash);
		child->mnt_parent->mnt_ns->pending_mounts = 0;
		umount_tree(child, UMOUNT_SYNC);
	}
	unlock_mount_hash();
	cleanup_group_ids(source_mnt, NULL);
 out:
	ns->pending_mounts = 0;

	read_seqlock_excl(&mount_lock);
	put_mountpoint(smp);
	read_sequnlock_excl(&mount_lock);

	return err;
}

```

- `unlock_mount`
```c
static void unlock_mount(struct mountpoint *where)
{
	struct dentry *dentry = where->m_dentry;

	read_seqlock_excl(&mount_lock);
	put_mountpoint(where);
	read_sequnlock_excl(&mount_lock);

	namespace_unlock();
	inode_unlock(dentry->d_inode);
}

```

- `do_umount()`

The call chain is: `SYSCALL_DEFINE2(umount...)` -> `ksys_umount` -> `path_umount` -> `do_umount`
```c
	namespace_lock();
	lock_mount_hash();

	/* Recheck MNT_LOCKED with the locks held */
	retval = -EINVAL;
	if (mnt->mnt.mnt_flags & MNT_LOCKED)
		goto out;

	event++;
	if (flags & MNT_DETACH) {
		if (!list_empty(&mnt->mnt_list))
			umount_tree(mnt, UMOUNT_PROPAGATE);
		retval = 0;
	} else {
		shrink_submounts(mnt);
		retval = -EBUSY;
		if (!propagate_mount_busy(mnt, 2)) {
			if (!list_empty(&mnt->mnt_list))
				umount_tree(mnt, UMOUNT_PROPAGATE|UMOUNT_SYNC);
			retval = 0;
		}
	}
out:
	unlock_mount_hash();
	namespace_unlock();
	return retval;

```

### Reader
TO BE DONE

### Conclusion

```c
SYSCALL_DEFINE5(mount...)
do_mount
	path_mout
		do_loopback
			lock_mount
				lookup_mnt               ---> [read] mount_hashtable
					legitimize_mnt
						__legitimize_mnt ---> [write] mnt->mnt_count
				get_mountpoint           ---> [read excl] mountpoint_hashtable
			__do_loopback
				clone_mnt                ---> [write] insert a mount to sb->s_mounts
			graft_tree
				attach_recursive_mnt     ---> [write] the most complex one, explained below
					get_mountpoint       ---> [read excl] ditto
			unlock_mount                 ---> [read excl] mountpoint_hashtable
```

The `mount_lock` in `attach_recursive_mnt()` is for below code:
**(assume the dest_mnt is not MNT_SHARED)**

```c
	lock_mount_hash();
	...
	mnt_set_mountpoint(dest_mnt, dest_mp, source_mnt);
	commit_tree(source_mnt);
	...
	put_mountpoint(smp);
	unlock_mount_hash();
	...
	read_seqlock_excl(&mount_lock);
	put_mountpoint(smp);
	read_sequnlock_excl(&mount_lock);
```
- `mnt_set_mountpoint()`
It mainly sets the new mnt's member and inserts the new mnt to
target mountpoint's mnt list(`mountpoint->m_list`).
- `commit_tree()` sets the new mnt's namespace member and inserts it to mount hash table
and finally inserts it to the mount tree.

The `put_mountpoint()` involves deleting mountpoint from mountpoint hash table.

So I guess `mount_lock` is to protect `mount_hashtable` and `mountpoint_hashtable`
and the mount tree. Not sure about that, still unfamiliar with mount code, I'll
update this article if get more info about it, especially why there is a read excl
mode, why we need to read exclusively.
