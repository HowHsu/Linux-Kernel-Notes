# Catalog

This blog is based on github pages, comments are welcome through github issues :)

* ### [PCIe protocol](./kernel/pcie/pcie.md)
* ### Nvme Driver
    * #### [nvme driver init](./kernel/nvme/driver_init.md)
    * #### [nvme device init](./kernel/nvme/nvme_probe.md)
        * ##### [nvme_reset_work](./kernel/nvme/nvme_reset_work.md)
        * ##### [nvme_scan_work](./kernel/nvme/nvme_scan_work)
* ### Generic Block Layer
    * #### [tagset](./kernel/block/tagset.md)
    * #### [life cycle of bio](./kernel/block/bio.md)
    * #### [life cycle of request](./kernel/block/request.md)
* ### Scheduler
    * #### [preempt](./kernel/scheduler/preempt.md)
    * #### [schedule() function](./kernel/scheduler/schedule.md)
* ### VFS
    * #### [dentry_inode](./kernel/vfs/dentry_inode.md)
    * #### [mount](./kernel/vfs/mount.md)
    * #### [mount_lock](./kernel/vfs/mount_lock.md)
    * #### [extent](./kernel/vfs/extent.md)
    * #### [iomap](./kernel/vfs/iomap.md)
    * #### [open](./kernel/vfs/open.md)
    * #### [bind_mount(WIP)](./kernel/vfs/bind_mount/bind_mount.md)
    	* ##### [lock_mount()(WIP)](./kernel/vfs/bind_mount/lock_mount.md)
    	* ##### [__do_loopback()(WIP)](./kernel/vfs/bind_mount/__do_loopback.md)
    	* ##### [graft_tree()(WIP)](./kernel/vfs/bind_mount/graft_tree.md)
    * #### [selinux(WIP)](./kernel/vfs/selinux.md)
    * #### [posix_acl(WIP)](./kernel/vfs/posix_acl.md)
    * #### [getdents64_and_dentry_and_fuse(WIP)](./kernel/vfs/getdents64_and_dentry_and_fuse.md)
* ### virtiofs
    * #### [virtiofs](./kernel/virtiofs/virtiofs.md)
    * #### [virtiofsd](./kernel/virtiofs/virtiofsd.md)
* ###cloud-hypervisor 
    * #### [vm create and boot](./kernel/cloud-hypervisor/vm_create_boot.md)
    * #### [cpu initialization](./kernel/cloud-hypervisor/cpu.md)
    * #### [mem initialization](./kernel/cloud-hypervisor/mem.md)
    * #### [device initialization](./kernel/cloud-hypervisor/device.md)
    * #### [fs initialization](./kernel/cloud-hypervisor/fs.md)
* ### [fuse](./kernel/FUSE/fuse.md)
* ### virtio
    * #### [virtio](./kernel/virtio/virtio.md)
    * #### [vhost](./kernel/virtio/vhost.md)
    * #### [vhost-user](./kernel/virtio/vhost-user.md)
* ### [dax](./kernel/dax/dax.md)
* ### vfio
    * #### [iommu](./kernel/vfio/iommu.md)
    * #### [sriov](./kernel/vfio/sriov.md)
* ### io_uring
    * #### [uringlet](./kernel/io_uring/uringlet/uringlet.md)
* ### [user && group && permission](./kernel/user_group_permission.md)
