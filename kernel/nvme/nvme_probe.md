# nvme_probe()

When PCIe bus matched nvme device and nvme driver, this function is called to do the initialization.
The code(I removed some trivial code)
<details>
<summary>nvme_probe() code</summary>

```c
static int nvme_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
	int node, result = -ENOMEM;
	struct nvme_dev *dev;
	unsigned long quirks = id->driver_data;
	size_t alloc_size;

	node = dev_to_node(&pdev->dev);
	if (node == NUMA_NO_NODE)
		set_dev_node(&pdev->dev, first_memory_node);

	dev = kzalloc_node(sizeof(*dev), GFP_KERNEL, node);

	dev->nr_write_queues = write_queues;
	dev->nr_poll_queues = poll_queues;
	dev->nr_allocated_queues = nvme_max_io_queues(dev) + 1;
	dev->queues = kcalloc_node(dev->nr_allocated_queues,
			sizeof(struct nvme_queue), GFP_KERNEL, node);

	dev->dev = get_device(&pdev->dev);
	pci_set_drvdata(pdev, dev);

	result = nvme_dev_map(dev);

	INIT_WORK(&dev->ctrl.reset_work, nvme_reset_work);
	INIT_WORK(&dev->remove_work, nvme_remove_dead_ctrl_work);
	mutex_init(&dev->shutdown_lock);

	result = nvme_setup_prp_pools(dev);

	/*
	 * Double check that our mempool alloc size will cover the biggest
	 * command we support.
	 */
	alloc_size = nvme_pci_iod_alloc_size();
	WARN_ON_ONCE(alloc_size > PAGE_SIZE);

	dev->iod_mempool = mempool_create_node(1, mempool_kmalloc,
						mempool_kfree,
						(void *) alloc_size,
						GFP_KERNEL, node);

	result = nvme_init_ctrl(&dev->ctrl, &pdev->dev, &nvme_pci_ctrl_ops,
			quirks);

	nvme_reset_ctrl(&dev->ctrl);
	async_schedule(nvme_async_probe, dev);

	return 0;
}
```

</details>

It basically does:

1. find numa node that the device sticks to, if there isn't one, set the first node for it. A device sticks to a numa node which means allocated memory will be from this node
2. Allocate struct nvme_dev {}
3. Set the number of write queues and poll queues
	These info are from userspacce--arguments of modprobe or grub command line
4. Increment refcount of pdev->dev->kobj
5. Set pdev->dev->driver_data = nvme_dev
6. nvme_dev_map(nvme_dev)
    <details>
    <summary>nvme_dev_map() analysis</summary>

    <details>
    <summary>nvme_dev_map() code</summary>

    ```c
    static int nvme_dev_map(struct nvme_dev *dev)
    {
        struct pci_dev *pdev = to_pci_dev(dev->dev);

        if (pci_request_mem_regions(pdev, "nvme"))
            return -ENODEV;

        if (nvme_remap_bar(dev, NVME_REG_DBS + 4096))
            goto release;

        return 0;
    }
    ```

    </details>

    * pci_request_mem_regions()
        Linux kernel maintains resource trees for PCI space, for example, PCI mem space(already mapped to PA). In this function, we will
        search in this tree and find the entry, mark it as busy, this way the kernel knows somebody is using this resource(the PA resource)
        (resource tree is another big topic, I'll make a new article to discuss it)

    * nvme_remap_bar(dev, NVME_REG_DBS + 4096)
        This one is super important, it maps the BAR space to memory space. There are core stuff like the doorbell registers.

        <details>
        <summary>nvme_remap_bar() code</summary>

        ```c
        static int nvme_remap_bar(struct nvme_dev *dev, unsigned long size)
        {
            struct pci_dev *pdev = to_pci_dev(dev->dev);

            if (size <= dev->bar_mapped_size) // no need to remap
                return 0;
            // nvme only use bar0, dont exceed [bar0.start, bar0.end)
            if (size > pci_resource_len(pdev, 0))
                return -ENOMEM;
            if (dev->bar)
                iounmap(dev->bar); // unmap it before remap
            // map bus memory address into CPU space(virtual memory address space)
            // to be specific, here we map pci bus space area [bar0.start, bar0.end)
            dev->bar = ioremap(pci_resource_start(pdev, 0), size);
            if (!dev->bar) {
                dev->bar_mapped_size = 0;
                return -ENOMEM;
            }
            // update bar_mapped_size
            dev->bar_mapped_size = size;
            dev->dbs = dev->bar + NVME_REG_DBS;

            return 0;
        }
        ```

        </details>

        - ioremap(physical_address, size) is to map bus memory to virtual memory sapce.
            Here you have to be clear about something:
            a. physical_address should be physical memory address.
            b. The PCI configuration space and BAR space are both PCI bus address space
            c. Since b, the value in BAR is PCI bus address
            d. There is mapping from VA(virtual memory address)-->PA(physical memory address)-->BA(bus address).
            e. It's not certain that PA == BA(in x86, seems they equal, but there are other architectures like PowerPC, Arm...)
            f. Based on c, d and e, you cannot directly read BAR value from PCI config space
            and map it to VA since it is BA.
            g. The correct method is to use pci_resource_start(pdev, bar) {pdev->resource[bar].start}.
            The value of it is already a PA(mapped from BA)

            After mapping the BAR space, we can now set the start address of doorbell registers: `dev->dbs = dev->bar + NVME_REG_DBS;`

    </details>

7. `INIT_WORK(&dev->ctrl.reset_work, nvme_reset_work);`
    nvme_reset_work() is the main dish of nvme device initialization. It is executed in system-wq context.
8. `INIT_WORK(&dev->remove_work, nvme_remove_dead_ctrl_work);`
    To Be Done
9. `result = nvme_setup_prp_pools(dev);`
    PRP means Physical Region Page. In a nvme command, there are PRP1 and PRP2 segment, they either point to a contiguous physical
    memory or another PRP list. A PRP entry point to a physical memory page for DMA. This is the way to let SSD device knows the source
    physical memory address(when writing) and target physical memory address(when reading)
    (For more details about PRP and Scatter Gather, see: [tobedone]())

10. `result = nvme_init_ctrl(&dev->ctrl, &pdev->dev, &nvme_pci_ctrl_ops, quiks);`
    This basically init the nvme_ctrl structure in nvme_dev, to be more specific, it initializes a char dev nvme{instance}, and the info
    is in ctrl->device = &ctrl->ctrl_device.
    
    <details>
    <summary>struct nvme_ctrl {}</summary>

    ```c
    struct nvme_ctrl {
        bool comp_seen;
        enum nvme_ctrl_state state; // = NVME_CTRL_NEW
        bool identified;
        spinlock_t lock;
        struct mutex scan_lock;
        const struct nvme_ctrl_ops *ops; // = nvme_pci_ctrl_ops
        struct request_queue *admin_q;
        struct request_queue *connect_q;
        struct request_queue *fabrics_q;
        struct device *dev; // = pdev->dev
        int instance; // the instance id, for instance, instance = 1, then we get /dev/nvme1
        int numa_node; // = NUMA_NO_NODE
        struct blk_mq_tag_set *tagset;
        struct blk_mq_tag_set *admin_tagset;
        struct list_head namespaces;
        struct rw_semaphore namespaces_rwsem;
        struct device ctrl_device;
        struct device *device;	/* char device */ // point to ctrl->ctrl_device
    #ifdef CONFIG_NVME_HWMON
        struct device *hwmon_device;
    #endif
        struct cdev cdev; // cdev->ops = nvme_dev_fops
        struct work_struct reset_work;
        struct work_struct delete_work;
        wait_queue_head_t state_wq;

        struct nvme_subsystem *subsys;
        struct list_head subsys_entry;

        struct opal_dev *opal_dev;

        char name[12];
        u16 cntlid;

        u32 ctrl_config;
        u16 mtfa;
        u32 queue_count;

        u64 cap;
        u32 max_hw_sectors;
        u32 max_segments;
        u32 max_integrity_segments;
        u32 max_discard_sectors;
        u32 max_discard_segments;
        u32 max_zeroes_sectors;
    #ifdef CONFIG_BLK_DEV_ZONED
        u32 max_zone_append;
    #endif
        u16 crdt[3];
        u16 oncs;
        u16 oacs;
        u16 sqsize;
        u32 max_namespaces;
        atomic_t abort_limit;
        u8 vwc;
        u32 vs;
        u32 sgls;
        u16 kas;
        u8 npss;
        u8 apsta;
        u16 wctemp;
        u16 cctemp;
        u32 oaes;
        u32 aen_result;
        u32 ctratt;
        unsigned int shutdown_timeout;
        unsigned int kato;
        bool subsystem;
        unsigned long quirks;
        struct nvme_id_power_state psd[32];
        struct nvme_effects_log *effects;
        struct xarray cels;
        struct work_struct scan_work;
        struct work_struct async_event_work;
        struct delayed_work ka_work;
        struct delayed_work failfast_work;
        struct nvme_command ka_cmd;
        struct work_struct fw_act_work;
        unsigned long events;

    #ifdef CONFIG_NVME_MULTIPATH
        /* asymmetric namespace access: */
        u8 anacap;
        u8 anatt;
        u32 anagrpmax;
        u32 nanagrpid;
        struct mutex ana_lock;
        struct nvme_ana_rsp_hdr *ana_log_buf;
        size_t ana_log_size;
        struct timer_list anatt_timer;
        struct work_struct ana_work;
    #endif

        /* Power saving configuration */
        u64 ps_max_latency_us;
        bool apst_enabled;

        /* PCIe only: */
        u32 hmpre;
        u32 hmmin;
        u32 hmminds;
        u16 hmmaxd;

        /* Fabrics only */
        u32 ioccsz;
        u32 iorcsz;
        u16 icdoff;
        u16 maxcmd;
        int nr_reconnects;
        unsigned long flags;
    #define NVME_CTRL_FAILFAST_EXPIRED	0
    #define NVME_CTRL_ADMIN_Q_STOPPED	1
        struct nvmf_ctrl_options *opts;

        struct page *discard_page;
        unsigned long discard_page_busy;

        struct nvme_fault_inject fault_inject;

        enum nvme_ctrl_type cntrltype;
        enum nvme_dctype dctype;
    };
    
    ```

    </details>

11. `nvme_reset_ctrl(&dev->ctrl);`
    <details>
    <summary>nvme_reset_ctrl() code</summary>

    ```c
	int nvme_reset_ctrl(struct nvme_ctrl *ctrl)
	{
		if (!nvme_change_ctrl_state(ctrl, NVME_CTRL_RESETTING)) // set the ctrl state to avoid re-enter
			return -EBUSY;
		if (!queue_work(nvme_reset_wq, &ctrl->reset_work)) // queue the nvme_reset_work()
			return -EBUSY;
		return 0;
	}
    ```

    </details>

12. `async_schedule(nvme_async_probe, dev);`
    Queue a simple function nvme_async_probe() to wait for the nvme_reset_work() and nvme_scan_work() done and
    then put ref of dev->ctrl.

13. nvme_reset_work(struct work_struct *work)
    Here comes the main dish of nvme initilization, it's a huge function...
    I've put it in a seperate file: [nvme_reset_work()](./nvme_reset_work.md)


14. nvme_scan_work()
To Be Done