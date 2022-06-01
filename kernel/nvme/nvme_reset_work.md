# nvme_reset_work()

```c
static void nvme_reset_work(struct work_struct *work)
{
    struct nvme_dev *dev =
        container_of(work, struct nvme_dev, ctrl.reset_work);
    bool was_suspend = !!(dev->ctrl.ctrl_config & NVME_CC_SHN_NORMAL);
    int result;

    if (dev->ctrl.state != NVME_CTRL_RESETTING) {
        dev_warn(dev->ctrl.device, "ctrl state %d is not RESETTING\n",
            dev->ctrl.state);
        result = -ENODEV;
        goto out;
    }

    /*
    * If we're called to reset a live controller first shut it down before
    * moving on.
    */
    if (dev->ctrl.ctrl_config & NVME_CC_ENABLE)
        nvme_dev_disable(dev, false);
    nvme_sync_queues(&dev->ctrl);

    mutex_lock(&dev->shutdown_lock);
    result = nvme_pci_enable(dev);
    if (result)
        goto out_unlock;

    result = nvme_pci_configure_admin_queue(dev);
    if (result)
        goto out_unlock;

    result = nvme_alloc_admin_tags(dev);
    if (result)
        goto out_unlock;

    /*
    * Limit the max command size to prevent iod->sg allocations going
    * over a single page.
    */
    dev->ctrl.max_hw_sectors = min_t(u32,
        NVME_MAX_KB_SZ << 1, dma_max_mapping_size(dev->dev) >> 9);
    dev->ctrl.max_segments = NVME_MAX_SEGS;

    /*
    * Don't limit the IOMMU merged segment size.
    */
    dma_set_max_seg_size(dev->dev, 0xffffffff);
    dma_set_min_align_mask(dev->dev, NVME_CTRL_PAGE_SIZE - 1);

    mutex_unlock(&dev->shutdown_lock);

    /*
    * Introduce CONNECTING state from nvme-fc/rdma transports to mark the
    * initializing procedure here.
    */
    if (!nvme_change_ctrl_state(&dev->ctrl, NVME_CTRL_CONNECTING)) {
        dev_warn(dev->ctrl.device,
            "failed to mark controller CONNECTING\n");
        result = -EBUSY;
        goto out;
    }

    /*
    * We do not support an SGL for metadata (yet), so we are limited to a
    * single integrity segment for the separate metadata pointer.
    */
    dev->ctrl.max_integrity_segments = 1;

    result = nvme_init_ctrl_finish(&dev->ctrl);
    if (result)
        goto out;

    if (dev->ctrl.oacs & NVME_CTRL_OACS_SEC_SUPP) {
        if (!dev->ctrl.opal_dev)
            dev->ctrl.opal_dev =
                init_opal_dev(&dev->ctrl, &nvme_sec_submit);
        else if (was_suspend)
            opal_unlock_from_suspend(dev->ctrl.opal_dev);
    } else {
        free_opal_dev(dev->ctrl.opal_dev);
        dev->ctrl.opal_dev = NULL;
    }

    if (dev->ctrl.oacs & NVME_CTRL_OACS_DBBUF_SUPP) {
        result = nvme_dbbuf_dma_alloc(dev);
        if (result)
            dev_warn(dev->dev,
                "unable to allocate dma for dbbuf\n");
    }

    if (dev->ctrl.hmpre) {
        result = nvme_setup_host_mem(dev);
        if (result < 0)
            goto out;
    }

    result = nvme_setup_io_queues(dev);
    if (result)
        goto out;

    /*
    * Keep the controller around but remove all namespaces if we don't have
    * any working I/O queue.
    */
    if (dev->online_queues < 2) {
        dev_warn(dev->ctrl.device, "IO queues not created\n");
        nvme_kill_queues(&dev->ctrl);
        nvme_remove_namespaces(&dev->ctrl);
        nvme_free_tagset(dev);
    } else {
        nvme_start_queues(&dev->ctrl);
        nvme_wait_freeze(&dev->ctrl);
        nvme_dev_add(dev);
        nvme_unfreeze(&dev->ctrl);
    }

    /*
    * If only admin queue live, keep it to do further investigation or
    * recovery.
    */
    if (!nvme_change_ctrl_state(&dev->ctrl, NVME_CTRL_LIVE)) {
        dev_warn(dev->ctrl.device,
            "failed to mark controller live state\n");
        result = -ENODEV;
        goto out;
    }

    if (!dev->attrs_added && !sysfs_create_group(&dev->ctrl.device->kobj,
            &nvme_pci_attr_group))
        dev->attrs_added = true;

    nvme_start_ctrl(&dev->ctrl);
    return;

out_unlock:
    mutex_unlock(&dev->shutdown_lock);
out:
    if (result)
        dev_warn(dev->ctrl.device,
            "Removing after probe failure status: %d\n", result);
    nvme_remove_dead_ctrl(dev);
}
```

* nvme_dev_disable()
disable the device if it is enabled.
* nvme_sync_queues()
cnacel attached callbacks on queues of this device
* nvme_pci_enable
This is pci related initialization, like fill dev->ctrl with info in bar space. Set up one interruption vector.
* nvme_pci_configure_admin_queue()

    <details>
    <summary>nvme_pci_configure_admin_queue() analysis</summary>
    
    ```c
    static int nvme_pci_configure_admin_queue(struct nvme_dev *dev)
    {
        int result;
        u32 aqa;
        struct nvme_queue *nvmeq;

        result = nvme_remap_bar(dev, db_bar_size(dev, 0));
        if (result < 0)
            return result;

        dev->subsystem = readl(dev->bar + NVME_REG_VS) >= NVME_VS(1, 1, 0) ?
                    NVME_CAP_NSSRC(dev->ctrl.cap) : 0;

        if (dev->subsystem &&
            (readl(dev->bar + NVME_REG_CSTS) & NVME_CSTS_NSSRO))
            writel(NVME_CSTS_NSSRO, dev->bar + NVME_REG_CSTS);

        result = nvme_disable_ctrl(&dev->ctrl);
        if (result < 0)
            return result;

        result = nvme_alloc_queue(dev, 0, NVME_AQ_DEPTH);
        if (result)
            return result;

        dev->ctrl.numa_node = dev_to_node(dev->dev);

        nvmeq = &dev->queues[0];
        aqa = nvmeq->q_depth - 1;
        aqa |= aqa << 16;

        writel(aqa, dev->bar + NVME_REG_AQA);
        lo_hi_writeq(nvmeq->sq_dma_addr, dev->bar + NVME_REG_ASQ);
        lo_hi_writeq(nvmeq->cq_dma_addr, dev->bar + NVME_REG_ACQ);

        result = nvme_enable_ctrl(&dev->ctrl);
        if (result)
            return result;

        nvmeq->cq_vector = 0;
        nvme_init_queue(nvmeq, 0);
        result = queue_request_irq(nvmeq);
        if (result) {
            dev->online_queues--;
            return result;
        }

        set_bit(NVMEQ_ENABLED, &nvmeq->flags);
        return result;
    }
    ```

    nvme_pci_configure_admin_queue() is essencial. It initialize the admin queue.
    1. `nvme_remap_bar(dev, db_bar_size(dev, 0));`
    This function remaps the bar space to virtual memory space.
        ```c
        static int nvme_remap_bar(struct nvme_dev *dev, unsigned long size)
        {
            struct pci_dev *pdev = to_pci_dev(dev->dev);

            if (size <= dev->bar_mapped_size)
                return 0;
            if (size > pci_resource_len(pdev, 0))
                return -ENOMEM;
            if (dev->bar)
                iounmap(dev->bar);
            dev->bar = ioremap(pci_resource_start(pdev, 0), size);
            if (!dev->bar) {
                dev->bar_mapped_size = 0;
                return -ENOMEM;
            }
            dev->bar_mapped_size = size;
            dev->dbs = dev->bar + NVME_REG_DBS;

            return 0;
        }
        ```

        * db_bar_size(dev, 0) is to get the size of the bar space
        
        ```c
        static unsigned long db_bar_size(struct nvme_dev *dev, unsigned nr_io_queues)
        {
            return NVME_REG_DBS + ((nr_io_queues + 1) * 8 * dev->db_stride);
        }
        ```

            NVME_REG_DBS is the offset of doorbell registers in bar space.
            dev->db_stride is set in nvme_pci_enable()
            ```c
        dev->ctrl.cap = lo_hi_readq(dev->bar + NVME_REG_CAP);

        dev->q_depth = min_t(u32, NVME_CAP_MQES(dev->ctrl.cap) + 1,
                    io_queue_depth);
        dev->ctrl.sqsize = dev->q_depth - 1; /* 0's based queue depth */
        dev->db_stride = 1 << NVME_CAP_STRIDE(dev->ctrl.cap);
        dev->dbs = dev->bar + 4096;
            ```
            dev->ctrl.cap is the first variable in bar space, it contains queue depth and db_stride info.
            we can see db_stride means the size of each doorbell register, unit dword(32 bytes)
            (NVME_REG_DBS + number of queues * (dev->db_stride * 4 Bytes) * 2(one queue one pair))

        * pci_resource_start(pdev, 0) is dev->resource[bar = 0].start
        The first argument of ioremap should be a physical memory address, so we don't use the value in
        configuration space directly since that is a bus address. We use dev->resource[bar id] alternatively,
        this is the physical memory address where the bar register points to.
        (notice: in x86, bus address == physical memory address, but the mapping process is still there(1:1 mapping))

        Now we've mapped the bar space(from beginning to admin queue's doorbell) to virtual memory space, let's store
        the start address of doorbell area.
        ```c
            dev->dbs = dev->bar + NVME_REG_DBS;
        ```
    2. `nvme_alloc_queue(dev, 0, NVME_AQ_DEPTH);`
    Alloc memory for a queue. Basically this alloc the dma area for sqring and cqring.

        ```c
        static int nvme_alloc_queue(struct nvme_dev *dev, int qid, int depth)
        {
            struct nvme_queue *nvmeq = &dev->queues[qid];

            // queue_count is the number of queues that we already allocated
            if (dev->ctrl.queue_count > qid)
                return 0;

            // nvmeq->sqes is the size of a sqe entry
            // notice: it is an order, so size = (1 << sqes)
            nvmeq->sqes = qid ? dev->io_sqes : NVME_ADM_SQES;
            nvmeq->q_depth = depth;
            // alloc the dma area for cqring
            // nvmeq->cqes is the cpu address(virtual memory address) and
            // nvmeq->cq_dma_addr will be set to the physical memory address
            // which is used by DMA
            // we can see that nvme has same length of sqring and cqring
            nvmeq->cqes = dma_alloc_coherent(dev->dev, CQ_SIZE(nvmeq),
                            &nvmeq->cq_dma_addr, GFP_KERNEL);
            if (!nvmeq->cqes)
                goto free_nvmeq;

            // alloc the dma area for sqring
            // same thing as above
            // nvmeq->sq_cmds is the cpu address
            // nvmeq->sq_dma_addr will be set to the physical address
            if (nvme_alloc_sq_cmds(dev, nvmeq, qid))
                goto free_cqdma;

            nvmeq->dev = dev;
            spin_lock_init(&nvmeq->sq_lock);
            spin_lock_init(&nvmeq->cq_poll_lock);
            nvmeq->cq_head = 0;
            nvmeq->cq_phase = 1;
            // point to the doorbell registers.
            nvmeq->q_db = &dev->dbs[qid * 2 * dev->db_stride];
            nvmeq->qid = qid;
            dev->ctrl.queue_count++;

            return 0;

        free_cqdma:
            dma_free_coherent(dev->dev, CQ_SIZE(nvmeq), (void *)nvmeq->cqes,
                    nvmeq->cq_dma_addr);
        free_nvmeq:
            return -ENOMEM;
        }
        ```

        I've commented the non-trivial parts above. This function is really important since it
        is about the queues of nvme, which is the key of this driver.
        You should know four members here
        
        ```c
        nvmeq->sq_cmds: cpu address of sqring
        nvmeq->sq_dma_addr: physical address of sqring
        nvmeq->cqes: cpu address of cqring
        nvmeq->cq_dma_addr: physical address of cqring
        ```

        There is still one thing to be done----the cmb(controller memory buffer)
        ```c
        static int nvme_alloc_sq_cmds(struct nvme_dev *dev, struct nvme_queue *nvmeq,
                        int qid)
        {
            struct pci_dev *pdev = to_pci_dev(dev->dev);

            if (qid && dev->cmb_use_sqes && (dev->cmbsz & NVME_CMBSZ_SQS)) {
                nvmeq->sq_cmds = pci_alloc_p2pmem(pdev, SQ_SIZE(nvmeq));
                if (nvmeq->sq_cmds) {
                    nvmeq->sq_dma_addr = pci_p2pmem_virt_to_bus(pdev,
                                    nvmeq->sq_cmds);
                    if (nvmeq->sq_dma_addr) {
                        set_bit(NVMEQ_SQ_CMB, &nvmeq->flags);
                        return 0;
                    }

                    pci_free_p2pmem(pdev, nvmeq->sq_cmds, SQ_SIZE(nvmeq));
                }
            }

            nvmeq->sq_cmds = dma_alloc_coherent(dev->dev, SQ_SIZE(nvmeq),
                        &nvmeq->sq_dma_addr, GFP_KERNEL);
            if (!nvmeq->sq_cmds)
                return -ENOMEM;
            return 0;
        }
        ```

        the cmb stuff in nvme_alloc_sq_cmds remains unclear, I'll analyze it later.

    3. 
        ```c
        nvmeq = &dev->queues[0];
        aqa = nvmeq->q_depth - 1;
        aqa = aqa << 16;

        writel(aqa, dev->bar + NVME_REG_AQA);
        ```

        NVME_REG_AQA stands for Admin Queue Attributes, it stores the length of sqring and cqring here.
        They are same and stored in high 16 bits and low 16 bits in aqa.

    4. 
        ```c
        lo_hi_writeq(nvmeq->sq_dma_addr, dev->bar + NVME_REG_ASQ);
        lo_hi_writeq(nvmeq->cq_dma_addr, dev->bar + NVME_REG_ACQ);
        ```

        Store the dma address to ASQ/ACQ registers

    5. `nvme_init_queue(nvmeq, 0);`
    
        ```c
        static void nvme_init_queue(struct nvme_queue *nvmeq, u16 qid)
        {
            struct nvme_dev *dev = nvmeq->dev;

            nvmeq->sq_tail = 0;
            nvmeq->last_sq_tail = 0;
            // these two were set in nvme_alloc_queue()
            // maybe because this function is called elsewhere?
            nvmeq->cq_head = 0;
            nvmeq->cq_phase = 1;
            nvmeq->q_db = &dev->dbs[qid * 2 * dev->db_stride];
            memset((void *)nvmeq->cqes, 0, CQ_SIZE(nvmeq));
            // TODO: analyze this
            nvme_dbbuf_init(dev, nvmeq, qid);
            dev->online_queues++;
            wmb(); /* ensure the first interrupt sees the initialization */
        }
        ```
        
        After a queue is inited, we increment dev->online_queues
    
    6. `queue_request_irq(nvmeq);`
    This is to allocate irq and install it for the nvme queue, here I'm not going to dive into it deeply
    since the irq subsystem is a complex topic, while I'm not an export of it.

        ```c
        static int queue_request_irq(struct nvme_queue *nvmeq)
        {
            struct pci_dev *pdev = to_pci_dev(nvmeq->dev->dev);
            int nr = nvmeq->dev->ctrl.instance;

            if (use_threaded_interrupts) {
                return pci_request_irq(pdev, nvmeq->cq_vector, nvme_irq_check,
                        nvme_irq, nvmeq, "nvme%dq%d", nr, nvmeq->qid);
            } else {
                return pci_request_irq(pdev, nvmeq->cq_vector, nvme_irq,
                        NULL, nvmeq, "nvme%dq%d", nr, nvmeq->qid);
            }
        }
        ```
        
        use_threaded_interrupts is a module parameter. if you indicate it at the modprobe time, the irq will be
        handled in kernel thread. And in this mode, irq handler function just does thread wake up, the main dish
        is thread_fn.
        [TODO]: 1. threaded interruption
                2. irq installation process in MSI/MSIX case
        
        Ok, let's skip the todo and the handler nvme_irq() for now, it's a separate topic.
        (see [nvme_irq](todo))

    7. `set_bit(NVMEQ_ENABLED, &nvmeq->flags);`
    </details>
    

* TODO: something about dma
* nvme_setup_io_queues()
Previously, we set up the admin queue, now let's init the IO queues.

    ```c
    static int nvme_setup_io_queues(struct nvme_dev *dev)
    {
        struct nvme_queue *adminq = &dev->queues[0];
        struct pci_dev *pdev = to_pci_dev(dev->dev);
        unsigned int nr_io_queues;
        unsigned long size;
        int result;

        /*
        * Sample the module parameters once at reset time so that we have
        * stable values to work with.
        */
        dev->nr_write_queues = write_queues;
        dev->nr_poll_queues = poll_queues;

        // previously at the beginning of nvme_probe(), nr_allocated_queues
        // are set to num_possible_cpus() + dev->nr_write_queues + dev->nr_poll_queues
        nr_io_queues = dev->nr_allocated_queues - 1;
        // now we send a nvme command to the admin queue to get the number of real hardware queues
        // and result = min(real number of queues, nr_io_queues)
        // so even if the hardware support more than nr_io_queues, we still have
        // num_cpu + nr_write_queues + nr_poll_queues.
        result = nvme_set_queue_count(&dev->ctrl, &nr_io_queues);
        if (result < 0)
            return result;

        if (nr_io_queues == 0)
            return 0;

        /*
        * Free IRQ resources as soon as NVMEQ_ENABLED bit transitions
        * from set to unset. If there is a window to it is truely freed,
        * pci_free_irq_vectors() jumping into this window will crash.
        * And take lock to avoid racing with pci_free_irq_vectors() in
        * nvme_dev_disable() path.
        */
        result = nvme_setup_io_queues_trylock(dev);
        if (result)
            return result;
        // let's first disable the admin queue
        // and free the irq
        if (test_and_clear_bit(NVMEQ_ENABLED, &adminq->flags))
            pci_free_irq(pdev, 0, adminq);

        if (dev->cmb_use_sqes) {
            result = nvme_cmb_qdepth(dev, nr_io_queues,
                    sizeof(struct nvme_command));
            if (result > 0)
                dev->q_depth = result;
            else
                dev->cmb_use_sqes = false;
        }

        // try to map the doorbell registers for queues
        // it's a try best effort mode, if we cannot map for 5 queues
        // we try the first 3 queues.
        // Thus the nr_io_queues changes in this process
        do {
            size = db_bar_size(dev, nr_io_queues);
            result = nvme_remap_bar(dev, size);
            if (!result)
                break;
            if (!--nr_io_queues) {
                result = -ENOMEM;
                goto out_unlock;
            }
        } while (1);
        adminq->q_db = dev->dbs;

    retry:
        // disable the admin queue at the beginning of retry

        /* Deregister the admin queue's interrupt */
        if (test_and_clear_bit(NVMEQ_ENABLED, &adminq->flags))
            pci_free_irq(pdev, 0, adminq);

        /*
        * If we enable msix early due to not intx, disable it again before
        * setting up the full range we need.
        */
        pci_free_irq_vectors(pdev);

        result = nvme_setup_irqs(dev, nr_io_queues);
        if (result <= 0) {
            result = -EIO;
            goto out_unlock;
        }

        dev->num_vecs = result;
        result = max(result - 1, 1);
        dev->max_qid = result + dev->io_queues[HCTX_TYPE_POLL];

        /*
        * Should investigate if there's a performance win from allocating
        * more queues than interrupt vectors; it might allow the submission
        * path to scale better, even if the receive path is limited by the
        * number of interrupts.
        */
        result = queue_request_irq(adminq);
        if (result)
            goto out_unlock;
        set_bit(NVMEQ_ENABLED, &adminq->flags);
        mutex_unlock(&dev->shutdown_lock);

        result = nvme_create_io_queues(dev);
        if (result || dev->online_queues < 2)
            return result;

        if (dev->online_queues - 1 < dev->max_qid) {
            nr_io_queues = dev->online_queues - 1;
            nvme_disable_io_queues(dev);
            result = nvme_setup_io_queues_trylock(dev);
            if (result)
                return result;
            nvme_suspend_io_queues(dev);
            goto retry;
        }
    }
    ```

    * `result = nvme_setup_irqs(dev, nr_io_queues);`
    This one is simple, we alloc irq vectors for adminq and every non-polled queue.
    Here we ensure poll_queues = min(dev->nr_poll_queues, nr_io_queues-1), which means
    we at least reserve one non-polled queue.
    result is the number of irq we allocated.

    * 
        ```c
        dev->num_vecs = result;
        result = max(result - 1, 1);
        dev->max_qid = result + dev->io_queues[HCTX_TYPE_POLL];
        ```
        dev->num_vecs: number of irqs
        dev_max_qid: max qid(from 0).
    * `result = queue_request_irq(adminq);`
    Let's install nvme_irq() for adminq again since we are going to use it.
    * `result = nvme_create_io_queues(dev);`
    This calls two functions:
        * nvme_alloc_queue(dev, i, dev->q_depth)
        We analyzed it before, it does what it means
        * nvme_create_queue(&dev->queues[i], i, polled)
        We already allocated the dma area of sqring/cqring, right? Then what the hell is
        going on with this function and what does 'create' mean?
        Here 'create' means letting the device know about the queue.
        We have to use admin command to setup some registers in device to let it know the
        queue address(dma area), irq vector num and so on. **Otherwise how could it take sqe and push cqe**?
        And yes, we do it through admin queue command while we do it by manully writing to
        the bar space at the admin queue init time.
        ```c
        adapter_alloc_cq(dev, qid, nvmeq, vector);
        adapter_alloc_sq(dev, qid, nvmeq);
        nvme_init_queue(nvmeq, qid);
        // install the nvme_irq()
        queue_request_irq(nvmeq);
        ```

* nvme_start_queues()
* nvme_dev_add()
The last part is init the tagset

    ```c
    static void nvme_dev_add(struct nvme_dev *dev)
    {
        int ret;

        if (!dev->ctrl.tagset) {
            dev->tagset.ops = &nvme_mq_ops;
            dev->tagset.nr_hw_queues = dev->online_queues - 1;
            dev->tagset.nr_maps = 2; /* default + read */
            if (dev->io_queues[HCTX_TYPE_POLL])
                dev->tagset.nr_maps++;
            dev->tagset.timeout = NVME_IO_TIMEOUT;
            dev->tagset.numa_node = dev->ctrl.numa_node;
            dev->tagset.queue_depth = min_t(unsigned int, dev->q_depth,
                            BLK_MQ_MAX_DEPTH) - 1;
            dev->tagset.cmd_size = sizeof(struct nvme_iod);
            dev->tagset.flags = BLK_MQ_F_SHOULD_MERGE;
            dev->tagset.driver_data = dev;

            ret = blk_mq_alloc_tag_set(&dev->tagset);
            if (ret) {
                dev_warn(dev->ctrl.device,
                    "IO queues tagset allocation failed %d\n", ret);
                return;
            }
            dev->ctrl.tagset = &dev->tagset;
        } else {
            blk_mq_update_nr_hw_queues(&dev->tagset, dev->online_queues - 1);

            /* Free previously allocated queues that are no longer usable */
            nvme_free_queues(dev, dev->online_queues);
        }

        nvme_dbbuf_set(dev);
    }
    ```

    What it does is basically allocating the requests for each queue.
    It's important part of the block layer. See: [tagset](../block/tagset.md)
* nvme_start_ctrl()
