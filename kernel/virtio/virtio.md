# virtio 协议

virtio 协议是一种传输协议，用在虚拟化环境中。virtio 的关键是 vring，vring 的本质是共享内存。
共享内存的实现根据 virtio 设备的不同可以分成三种情况：

- qemu模拟的设备
    guest在qemu进程内，所以qemu天然可以访问
- qemu外模拟的设备
    需要再次映射，通过vhost/vhost-user协议进行沟通
- 真实设备
    需要通过IOMMU辅助

