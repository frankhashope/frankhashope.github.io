---
title: "virtio显卡实现分析"
tagline: ""
header:
  overlay_color: "#48C9B0"
classes: wide
categories:
  - Virtualization
---

virtio-vga与virtio-gpu是qemu模拟的较新的显卡设备，它们都是由Dave Airlie等人[引入](https://virgil3d.github.io/)，避免通过直通GPU来加速虚拟机内部的3D渲染。x86下使用virtio-vga，arm下使用virtio-gpu，guest里使用virtio-gpu作为前端驱动。x86下如果Guest OS中没有virtio-gpu驱动，则使用兼容的标准vga模式。此外为了提供高性能，virtio-vga与virtio-gpu都有对应的vhost-user用户态实现，即在另一个单独的用户态进程中模拟virtio-gpu。

### virtio-vga与virtio-gpu的关系

qemu支持virtio-gpu挂接mmio总线和pci总线，下文以virtio-gpu-pci为例。

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/virtio_gpu_qom.png){: .align-center}

首先，从qemu的QOM类型关系图可以看到virtio-vga和virtio-gpu-pci一样，都是virtio-pci设备，遵循virtio-pci协议。其次，从两者的数据结构定义可以看到他们都实现了VirtIOGPU，同时VirtIOGPU又都实现了VirtIOGPUBase。下文主要介绍virtio-vga的主要实现框架。

```c
struct VirtIOVGA {
    VirtIOVGABase parent_obj;
    VirtIOGPU     vdev;
};

struct VirtIOGPUPCI {
    VirtIOGPUPCIBase parent_obj;
    VirtIOGPU vdev;
};

struct VirtIOGPU {
    VirtIOGPUBase parent_obj;

    uint64_t conf_max_hostmem;

    VirtQueue *ctrl_vq;
    VirtQueue *cursor_vq;

    QEMUBH *ctrl_bh;
    QEMUBH *cursor_bh;

    QTAILQ_HEAD(, virtio_gpu_simple_resource) reslist;
    QTAILQ_HEAD(, virtio_gpu_ctrl_command) cmdq;
    QTAILQ_HEAD(, virtio_gpu_ctrl_command) fenceq;

    uint64_t hostmem;

    bool processing_cmdq;
    QEMUTimer *fence_poll;
    QEMUTimer *print_stats;

    uint32_t inflight;
    struct {
        uint32_t max_inflight;
        uint32_t requests;
        uint32_t req_3d;
        uint32_t bytes_3d;
    } stats;

    struct {
        QTAILQ_HEAD(, VGPUDMABuf) bufs;
        VGPUDMABuf *primary;
    } dmabuf;
};
```

### virtio-vga显卡

- ### realize实现

virtio-vga其实例的初始化函数为virtio_vga_inst_initfn。其中，virtio_instance_init_common会对VirtIOGPU进行初始化。

```c
static void virtio_vga_inst_initfn(Object *obj)
{
    VirtIOVGA *dev = VIRTIO_VGA(obj);

    virtio_instance_init_common(obj, &dev->vdev, sizeof(dev->vdev),
                                TYPE_VIRTIO_GPU);
    VIRTIO_VGA_BASE(dev)->vgpu = VIRTIO_GPU_BASE(&dev->vdev);
}
```

在对指定的数据类型进行初始化后，qemu会调用device_set_realized函数具现化设备。qemu的[QOM](https://www.binss.me/blog/qemu-note-of-qemu-object-model/)建立了一棵面向对象的树，类型初始化和设备具现化都会通过这棵树由上而下从基类递归调用到设备或类型注册的具体接口。virtio-vga的具现化调用路径为：

```c
-> device_set_realized
  -> virtio_pci_dc_realize
    -> virtio_pci_realize
      -> virtio_vga_base_realize
        -> device_set_realized
          -> virtio_device_realize
    	    -> virtio_gpu_device_realize
```

可以看到由于virtio-vga本身是一个virtio-pci设备，因此首先调用了virtio pci的realize接口，然后调用了virtio-vga注册的realize回调。

```c
/* VGA device wrapper around PCI device around virtio GPU */
static void virtio_vga_base_realize(VirtIOPCIProxy *vpci_dev, Error **errp)
{
    VirtIOVGABase *vvga = VIRTIO_VGA_BASE(vpci_dev);
    VirtIOGPUBase *g = vvga->vgpu;
    VGACommonState *vga = &vvga->vga;
    uint32_t offset;
    int i;

    /* init vga compat bits */
    vga->vram_size_mb = 8;
    // 执行一些vga相关的初始化动作，例如设置vram的大小以及vga相关的回调函数。
    vga_common_init(vga, OBJECT(vpci_dev));
    // 映射128KB的显存至0xA0000h-0xBFFFh这段地址空间，并注册I/O端口。
    vga_init(vga, OBJECT(vpci_dev), pci_address_space(&vpci_dev->pci_dev),
             pci_address_space_io(&vpci_dev->pci_dev), true);
    pci_register_bar(&vpci_dev->pci_dev, 0,
                     PCI_BASE_ADDRESS_MEM_PREFETCH, &vga->vram);

    /*
     * Configure virtio bar and regions
     *
     * We use bar #2 for the mmio regions, to be compatible with stdvga.
     * virtio regions are moved to the end of bar #2, to make room for
     * the stdvga mmio registers at the start of bar #2.
     */
    vpci_dev->modern_mem_bar_idx = 2;
    vpci_dev->msix_bar_idx = 4;
    vpci_dev->modern_io_bar_idx = 5;

    if (!(vpci_dev->flags & VIRTIO_PCI_FLAG_PAGE_PER_VQ)) {
        /*
         * with page-per-vq=off there is no padding space we can use
         * for the stdvga registers.  Make the common and isr regions
         * smaller then.
         */
        vpci_dev->common.size /= 2;
        vpci_dev->isr.size /= 2;
    }

    offset = memory_region_size(&vpci_dev->modern_bar);
    offset -= vpci_dev->notify.size;
    vpci_dev->notify.offset = offset;
    offset -= vpci_dev->device.size;
    vpci_dev->device.offset = offset;
    offset -= vpci_dev->isr.size;
    vpci_dev->isr.offset = offset;
    offset -= vpci_dev->common.size;
    vpci_dev->common.offset = offset;

    /* init virtio bits */
    virtio_pci_force_virtio_1(vpci_dev);
    if (!qdev_realize(DEVICE(g), BUS(&vpci_dev->bus), errp)) {
        return;
    }

    /* add stdvga mmio regions */
    pci_std_vga_mmio_region_init(vga, OBJECT(vvga), &vpci_dev->modern_bar,
                                 vvga->vga_mrs, true, false);

    vga->con = g->scanout[0].con;
    graphic_console_set_hwops(vga->con, &virtio_vga_base_ops, vvga);

    for (i = 0; i < g->conf.max_outputs; i++) {
        object_property_set_link(OBJECT(g->scanout[i].con), "device",
                                 OBJECT(vpci_dev), &error_abort);
    }
}
```

VirtIOGPU具现化调用virtio_gpu_base_device_realize初始化VirtIOGPUBase，可以看到其有两个virtio队列：0号队列为控制队列，1号队列为鼠标队列。virtio-vga、virtio-gpu/virtio-gpu-pci、vhost-user-gpu都基于VirtIOGPUBase实现。

```c
void virtio_gpu_device_realize(DeviceState *qdev, Error **errp)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(qdev);
    VirtIOGPU *g = VIRTIO_GPU(qdev);

    ......

    if (!virtio_gpu_base_device_realize(qdev,
                                        virtio_gpu_handle_ctrl_cb,
                                        virtio_gpu_handle_cursor_cb,
                                        errp)) {
        return;
    }

    g->ctrl_vq = virtio_get_queue(vdev, 0);
    g->cursor_vq = virtio_get_queue(vdev, 1);
    g->ctrl_bh = qemu_bh_new(virtio_gpu_ctrl_bh, g);
    g->cursor_bh = qemu_bh_new(virtio_gpu_cursor_bh, g);
    QTAILQ_INIT(&g->reslist);
    QTAILQ_INIT(&g->cmdq);
    QTAILQ_INIT(&g->fenceq);
}

bool
virtio_gpu_base_device_realize(DeviceState *qdev,
                               VirtIOHandleOutput ctrl_cb,
                               VirtIOHandleOutput cursor_cb,
                               Error **errp)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(qdev);
    VirtIOGPUBase *g = VIRTIO_GPU_BASE(qdev);
    int i;

    ......

    g->virtio_config.num_scanouts = cpu_to_le32(g->conf.max_outputs);
    // 初始化virtio队列等virtio设备相关变量
    virtio_init(VIRTIO_DEVICE(g), "virtio-gpu", VIRTIO_ID_GPU,
                sizeof(struct virtio_gpu_config));

    if (virtio_gpu_virgl_enabled(g->conf)) {
        /* use larger control queue in 3d mode */
        virtio_add_queue(vdev, 256, ctrl_cb);
        virtio_add_queue(vdev, 16, cursor_cb);
    } else {
        virtio_add_queue(vdev, 64, ctrl_cb);
        virtio_add_queue(vdev, 16, cursor_cb);
    }

    g->enabled_output_bitmask = 1;

    // 默认的分辨率为1024*768
    g->req_state[0].width = g->conf.xres;
    g->req_state[0].height = g->conf.yres;

    g->hw_ops = &virtio_gpu_ops;
    for (i = 0; i < g->conf.max_outputs; i++) {
        g->scanout[i].con =
            graphic_console_init(DEVICE(g), i, &virtio_gpu_ops, g);
    }

    return true;
}
```

qemu使用结构体QemuConsole来表示图形化输出终端（例如VNC、SPICE等），可以为虚拟显卡设备绑定这样的多个图形化输出终端。在virtio_vga_base_realize的最后，可以看到virtio_vga_base_ops被注册为第一个scanout的QemuConsole的hw_ops，而其他scanout的终端的hw_ops都被设置为virtio_gpu_ops。virtio_common_init中会设置vga的hw_ops，即vga_ops，virtio_vga_base_ops根据是virtio-vga还是virtio-gpu调用vga_op或者virtio_gpu_ops。回调具体的实现与协议强相关，这里不展开详细分析。

- ### I/O端口访问

VGA协议提供了6组不同功能的寄存器组：图形化寄存器、定序器寄存器、属性控制寄存器、CRT控制寄存器、颜色寄存器、外部寄存器，访问这些寄存器的唯一方法就是通过IO端口。qemu为virtio-vga设置的VGA IO端口访问方法为vga_ioport_read以及vga_ioport_write。

```c
static const MemoryRegionPortio vga_portio_list[] = {
    { 0x04,  2, 1, .read = vga_ioport_read, .write = vga_ioport_write }, /* 3b4 */
    { 0x0a,  1, 1, .read = vga_ioport_read, .write = vga_ioport_write }, /* 3ba */
    { 0x10, 16, 1, .read = vga_ioport_read, .write = vga_ioport_write }, /* 3c0 */
    { 0x24,  2, 1, .read = vga_ioport_read, .write = vga_ioport_write }, /* 3d4 */
    { 0x2a,  1, 1, .read = vga_ioport_read, .write = vga_ioport_write }, /* 3da */
    PORTIO_END_OF_LIST(),
};
```

为了支持提供更高分辨率和颜色深度的SVGA(Super VGA)，vga_init函数中也注册了VBE(VESA BIOS EXTENSION)标准的I/O端口。qemu中所有SVGA设备都支持bochs显示接口。该显示接口使用0x1ce的端口作为寄存器索引，0x1cf的端口作为数据寄存器，在一些非x86平台下，使用0x1d0作为数据寄存器的端口。

```c
static const MemoryRegionPortio vbe_portio_list[] = {
    { 0, 1, 2, .read = vbe_ioport_read_index, .write = vbe_ioport_write_index },
# ifdef TARGET_I386
    { 1, 1, 2, .read = vbe_ioport_read_data, .write = vbe_ioport_write_data },
# endif
    { 2, 1, 2, .read = vbe_ioport_read_data, .write = vbe_ioport_write_data },
    PORTIO_END_OF_LIST(),
};
```

- ### 显存访问

在vga_init函数中不仅注册了I/O端口，同时也注册了起始地址为0xA0000的128KB的内存地址空间，关联的读写回调分别为vga_mem_read和vga_mem_write。

```c
/* Used by both ISA and PCI */
MemoryRegion *vga_init_io(VGACommonState *s, Object *obj,
                          const MemoryRegionPortio **vga_ports,
                          const MemoryRegionPortio **vbe_ports)
{
    MemoryRegion *vga_mem;

    *vga_ports = vga_portio_list;
    *vbe_ports = vbe_portio_list;

    vga_mem = g_malloc(sizeof(*vga_mem));
    memory_region_init_io(vga_mem, obj, &vga_mem_ops, s,
                          "vga-lowmem", 0x20000);
    memory_region_set_flush_coalesced(vga_mem);

    return vga_mem;
}

const MemoryRegionOps vga_mem_ops = {
    .read = vga_mem_read,
    .write = vga_mem_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl = {
        .min_access_size = 1,
        .max_access_size = 1,
    },
};
```

### 参考资料

[1] [VGA Chipset Reference](https://web.stanford.edu/class/cs140/projects/pintos/specs/freevga/vga/vga.htm)

[2] [Graphics in QEMU](https://www.linux-kvm.org/images/b/b2/01x10b-QEMUGfraphics.pdf)
