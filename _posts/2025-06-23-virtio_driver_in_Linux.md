---
title: "Linux的virtio设备前端驱动实现"
classes: "wide"
tagline: ""
header:
  overlay_color: "#9EBC8A"
categories:
  - Linux Kernel
toc: true
toc_label: "Linux的virtio设备前端驱动实现"
toc_icon: "navicon"
toc_sticky: true
---

众所周知，virtio协议是虚拟化场景下诞生的一种前后端设备通信协议，极大提升了虚拟化场景下的I/O性能。该协议需要前后端共同配合实现，Guest OS内部需要安装virtio设备的前端驱动，后端VMM则负责模拟对应的virtio设备后端，两者基于virtio规范进行通信。
本文主要基于Linux Kernel 6.6分析virtio设备的前端驱动实现，以求深入全面地理解virtio协议。

# virtio总线

virtio子系统初始化时会向内核注册一条虚拟的virtio总线，并像PCI总线一样在sysfs文件系统下有对应目录。
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/virtio_bus.png){: .align-center}

```c
static struct bus_type virtio_bus = {
    .name  = "virtio",
    .match = virtio_dev_match,
    .dev_groups = virtio_dev_groups, // Default attributes of the devices on the bus.
    .uevent = virtio_uevent,
    .probe = virtio_dev_probe,
    .remove = virtio_dev_remove,
};
```

# virtio驱动结构体

virtio协议定义了设备进行I/O通信的协议规范，衍生出了virtio_net、virtio_blk、virtio_balloon等各种各样的virtio设备。设备驱动模块初始化的时候，首先需要调用register_virtio_driver接口向内核注册。以virtio_net驱动为例，驱动模块在初始化时注册了virtio_net_driver。

```c
static struct virtio_driver virtio_net_driver = {
    .feature_table = features,
    .feature_table_size = ARRAY_SIZE(features),
    .feature_table_legacy = features_legacy,
    .feature_table_size_legacy = ARRAY_SIZE(features_legacy),
    .driver.name =  KBUILD_MODNAME,
    .driver.owner = THIS_MODULE,
    .id_table = id_table,
    .validate = virtnet_validate,
    .probe =    virtnet_probe,
    .remove =   virtnet_remove,
    .config_changed = virtnet_config_changed,
#ifdef CONFIG_PM_SLEEP
    .freeze =   virtnet_freeze,
    .restore =  virtnet_restore,
#endif
};
```

# 设备初始化

当驱动被注册到总线上时，内核会依次调用总线及驱动的probe方法执行初始化。如上节所述，每当发现一个virtio设备触发对应驱动模块插入时对应驱动模块的初始化函数首先会调用register_virtio_driver接口注册virtio_driver结构体。

```c
register_virtio_driver()
    --> driver_register()
        --> bus_add_driver()
            --> driver_attach()
                --> __driver_attach()  // 遍历总线上的所有设备
                    --> driver_probe_device()
                        --> really_probe()
                            --> call_driver_probe()
                                --> virtio_dev_probe()  // dev->bus->probe()
                                    --> drv->probe()
```

virtio_dev_probe()通过读写设备的配置空间完成前后端features的协商，接着调用具体virtio设备驱动如的probe方法基于协商的features完成设备独有的配置以及分配队列，最后设置VIRTIO_CONFIG_S_DRIVER_OK告诉后端初始化已完成。

以virtio_net驱动为例，其probe接口virtnet_probe中首先会根据前面与后端协商的virtio协议定义的features设置net_device结构体中与网络设备强相关的许多属性，接着调用init_vqs()分配与初始化virtio队列。除了发送与接收队列，virtio_net根据VIRTIO_NET_F_CTRL_VQ这个feature来决定是否创建控制队列。控制队列只有一个，发送与接收队列可以有多组，具体组数由后端设置在配置空间中。

```c
static int init_vqs(struct virtnet_info *vi)
{
    int ret;

    /* Allocate send & receive queues */
    ret = virtnet_alloc_queues(vi);
    if (ret)
        goto err;

    // 1. 分配vring共享内存区域。
    // 2. 分配中断，可以为每个队列分配一个中断，也可以多个队列共享一个中断，优先前者。中断类型可以是MSI，MSI-X或者INTx。
    // 3. 注册中断的处理回调为vring_interrupt()，其会调用不同队列的callback。
    // 4. 设置通知后端的回调：vp_notify_with_data()/vp_notify()
    // 5. 使能队列。
    ret = virtnet_find_vqs(vi);
    if (ret)
        goto err_free;

    virtnet_rq_set_premapped(vi);

    cpus_read_lock();
    virtnet_set_affinity(vi);  // 均衡设置队列中断绑核
    cpus_read_unlock();

    return 0;

err_free:
    virtnet_free_queues(vi);
err:
    return ret;
}
```

发送与接收队列的中断处理函数分别为skb_xmit_done()和skb_recv_done()，控制队列无中断处理回调。

# virtio_net实现

下文以virtio_net设备为例分析一下数据面即数据包收发的实现，其他virtio设备类似。virtio协议的具体定义请参见virtio的协议规范文档<sup>[[1](#ref1)]</sup>，这里不再详细展开。

## 数据包发送

内核接口__netdev_start_xmit()会调用virtio_net设备注册的start_xmit接口发送数据包。忽略数据包网络强相关的实现逻辑，这里我们主要关注virtio协议相关的实现。

```c
start_xmit()
    --> xmit_skb()
        // 填充到发送队列
        --> virtqueue_add_outbuf()
            --> virtqueue_add()
                // split布局，若是packed布局则调用virtqueue_add_packed()
                --> virtqueue_add_split()
                    // DMA映射
                    --> vring_map_one_sg()/vring_map_single()
                    // 填充vring的描述符链表vring_map_single
                    --> virtqueue_add_desc_split()
    // 当没有更多的数据包需要发送，或发送被停止。
    --> virtqueue_kick_prepare()
        // 判断是否需要kick后端
        --> virtqueue_kick_prepare_split()
    // 写指定BAR空间的特定地址通知后端。
    --> virtqueue_notify()
```

virtqueue_add_split()做的工作主要是将scatterlist中的数据做DMA映射，获取DMA地址后填充vring的描述符链表，更新avail数组与索引。virtqueue_notify()通知后端通过实际物理设备发送数据包，发送完成后后端再通过中断通知前端驱动，前端驱动于是调用skb_xmit_done()接口做一些性能优化，如合并中断等。

## 数据包接收

接收数据包的流程与发送数据包的流程类似：

```c
try_fill_recv()
    --> add_recvbuf_mergeable()/add_recvbuf_big()/add_recvbuf_small()
        --> virtnet_rq_alloc()
            --> virtqueue_dma_map_single_attrs()
        --> virtnet_rq_init_one_sg()
        --> virtqueue_add_inbuf_ctx()
            --> virtqueue_add()
    --> virtqueue_kick_prepare()
    --> virtqueue_notify() 
```

物理网卡接收到数据包后，后端通过中断通知前端，前端驱动调用skb_recv_done()。

## 控制队列

与其他virtio设备不同的是，virtio_net提供了一个控制队列，可用于设置MAC、VLAN过滤等。控制队列由前端作为发送端，发送一些控制命令给后端。与数据包发送不同，控制命令的发送是同步的。

```c
virtnet_send_command()
    --> sg_init_one()
    --> virtqueue_add_sgs()
        --> virtqueue_add()
    --> virtqueue_kick()
    --> virtqueue_get_buf()                   // 同步等待
        --> virtqueue_get_buf_ctx()
            --> virtqueue_get_buf_ctx_split() // 如果是split布局
```

# 参考资料

[1]<a name="ref1"> [Virtual I/O Device (VIRTIO) Version 1.3](https://docs.oasis-open.org/virtio/virtio/v1.3/virtio-v1.3.html)

[2] [virtio前端驱动通知机制分析](https://blog.csdn.net/qq_41596356/article/details/128441953)

[3] [virtio前端驱动详解](https://www.cnblogs.com/ck1020/p/6044134.html "发布于 2016-11-15 15:48")
