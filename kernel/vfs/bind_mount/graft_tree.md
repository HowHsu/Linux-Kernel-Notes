# `graft_tree()`

Ok, let's revise the previous section, after `lock_mount()`, we've got a new
`struct mountpoint` based on the **target directory**, and after `__do_loopback()`,
we've got a new `struct mount` based on the **source directory**.
Now it's time to install them.

```c

static int graft_tree(struct mount *mnt, struct mount *p, struct mountpoint *mp)
{
	if (mnt->mnt.mnt_sb->s_flags & SB_NOUSER)
		return -EINVAL;

	if (d_is_dir(mp->m_dentry) !=
	      d_is_dir(mnt->mnt.mnt_root))
		return -ENOTDIR;

	return attach_recursive_mnt(mnt, p, mp, false);
}

```

Firstly, we can see when we do bind mount, the source and target must be same type
(both directory or both file)

## 1. `attach_recursive_mnt(source_mnt, dest_mnt, dest_mp, moving)`

Let's go through its code(I've removed some code for `mount --move`)
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
	/* Is there space to add these mounts to the mount namespace? */
	err = count_mounts(ns, source_mnt);

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
	
	if (source_mnt->mnt_ns) {
		/* move from anon - the caller will destroy */
		list_del_init(&source_mnt->mnt_ns->list);
	}
	mnt_set_mountpoint(dest_mnt, dest_mp, source_mnt);
	commit_tree(source_mnt);

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

It does:

- do some valid check.
  nothing special.

- do the propagation stuff.
  Let's forget this part for now, mount propagation is a bit complex and deserve
a seperate article.

- `mnt_set_mountpoint(dest_mnt, dest_mp, source_mnt);`
- `commit_tree()`

Next I'll focus on the last two functions.

## 2. `mnt_set_mountpoint()`

```c
void mnt_set_mountpoint(struct mount *mnt,
			struct mountpoint *mp,
			struct mount *child_mnt)
{
	mp->m_count++;
	mnt_add_count(mnt, 1);	/* essentially, that's mntget */
	child_mnt->mnt_mountpoint = mp->m_dentry;
	child_mnt->mnt_parent = mnt;
	child_mnt->mnt_mp = mp;
	hlist_add_head(&child_mnt->mnt_mp_list, &mp->m_list);
}

```

Do you remember what happened in previous `__do_loopback()`

```c
	mnt->mnt.mnt_root = dget(root);
	mnt->mnt_mountpoint = mnt->mnt.mnt_root;
	mnt->mnt_parent = mnt;
```

In `__do_loopback()`, we init the new mount: `mnt_mountpoint = root` and `mnt->mnt_parent = mnt`.
Hmm.. The value are not right, I think this is for convienence of other callers.
Here in `mnt_set_mountpoint()`, they are reset.
Now you may be confused: **why there are both `mnt->mnt_mountpoint` and `mnt->mnt_mp->m_dentry`,
they really look same?**
**Well, my understanding is: yes, they are exactly same thing, `mnt->mnt_mountpoint` is like a cache**


## 3. `commit_tree()`

```c
static void commit_tree(struct mount *mnt)
{
	struct mount *parent = mnt->mnt_parent;
	struct mount *m;
	LIST_HEAD(head);
	struct mnt_namespace *n = parent->mnt_ns;

	BUG_ON(parent == mnt);

	list_add_tail(&head, &mnt->mnt_list);
	list_for_each_entry(m, &head, mnt_list)
		m->mnt_ns = n;

	list_splice(&head, n->list.prev);

	n->mounts += n->pending_mounts;
	n->pending_mounts = 0;

	__attach_mnt(mnt, parent);
/*
	static void __attach_mnt(struct mount *mnt, struct mount *parent)
	{
		hlist_add_head_rcu(&mnt->mnt_hash,
				   m_hash(&parent->mnt, mnt->mnt_mountpoint));
		list_add_tail(&mnt->mnt_child, &parent->mnt_mounts);
	}
*/
	touch_mnt_namespace(n);
}

```

`commit_tree()` inserts the new mount to the mount tree and add it to the global
`mount_hashtable`. Note here `parent` is already the target parent mount since we
have update `mnt_parent` in `mnt_set_mountpoint()`
