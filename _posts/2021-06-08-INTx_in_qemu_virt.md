---
title: "qemu如何模拟INTx"
classes: "wide"
tagline: ""
header:
  overlay_color: "#48C9B0"
categories:
  - Virtual Machine Manager
toc: true
toc_label: "qemu如何模拟INTx"
toc_icon: "navicon"
toc_sticky: true
---

本文主要以针对arm架构实现的virt机型为例，分析qemu是如何实现INTx中断的模拟和虚拟化的。qemu在x86架构下的实现可参考这篇[文章](https://www.binss.me/blog/qemu-note-of-interrupt/)。PCI/PCIe设备支持INTx中断或者MSI/MSI-X中断，传统的一些老旧设备仍旧在使用通过中断引脚传递中断请求的INTx中断机制。此外，在arm平台下由于linux内核中GIC驱动的实现，如果有PCI桥且PCI桥下还有多个设备，也必须支持INTx中断。

### irqmap

virt机型中定义了一个中断号数组，可以看到其机型的设备及平台的中断号被固定了。

```c
static const int a15irqmap[] = {
    [VIRT_UART] = 1,
    [VIRT_RTC] = 2,
    [VIRT_PCIE] = 3, /* ... to 6 */
    [VIRT_GPIO] = 7,
    [VIRT_SECURE_UART] = 8,
    [VIRT_ACPI_GED] = 9,
    [VIRT_MMIO] = 16, /* ...to 16 + NUM_VIRTIO_TRANSPORTS - 1 */
    [VIRT_GIC_V2M] = 48, /* ...to 48 + NUM_GICV2M_SPIS - 1 */
    [VIRT_SMMU] = 74,    /* ...to 74 + NUM_SMMU_IRQS - 1 */
    [VIRT_PLATFORM_BUS] = 112, /* ...to 112 + PLATFORM_BUS_NUM_IRQS -1 */
};
```

### GPIO(General Purpose Input/Output)

通用输入输出引脚，顾名思义可以用作输入也可以用于输出电信号，通常在中断控制器上有。qemu中使用结构体NamedGPIOList来表示一个设备上的GPIO控制器。一个设备上可以有多个GPIO控制器，于是可以通过node将多个NamedGPIOList作为链表链接。

```c
struct NamedGPIOList {
    char *name;
    qemu_irq *in;// 输入中断数组
    int num_in;// 输入引脚的个数
    int num_out;// 输出引脚的个数
    QLIST_ENTRY(NamedGPIOList) node;
}
```

qemu_irq是IRQState结构体的别名，用来模拟中断引脚。

```c
struct IRQState {
    Object parent_obj;

    qemu_irq_handler handler;// 中断的处理回调
    void *opaque;// 指向所属设备
    int n;// 引脚号
}
```

### GED(General Event Device)

qemu的virt机型默认是精简硬件的ACPI平台，在4.2版本及以上的该机型中加入了针对GED的支持。GED是ACPI v6.1专门为精简硬件平台引入的设备，用于处理包括热插拔的所有平台事件。创建GED设备的接口函数为create_acpi_ged，qemu将GED作为系统总线上的一个设备。

```c
static inline DeviceState *create_acpi_ged(VirtMachineState *vms)
{
    DeviceState *dev;
    MachineState *ms = MACHINE(vms);
    int irq = vms->irqmap[VIRT_ACPI_GED];
    uint32_t event = ACPI_GED_PWR_DOWN_EVT;

    if (ms->ram_slots) {
        event |= ACPI_GED_MEM_HOTPLUG_EVT;
    }

    if (ms->nvdimms_state->is_enabled) {
        event |= ACPI_GED_NVDIMM_HOTPLUG_EVT;
    }

    dev = qdev_new(TYPE_ACPI_GED);
    qdev_prop_set_uint32(dev, "ged-event", event);

    sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, vms->memmap[VIRT_ACPI_GED].base);
    sysbus_mmio_map(SYS_BUS_DEVICE(dev), 1, vms->memmap[VIRT_PCDIMM_ACPI].base);
    sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0, qdev_get_gpio_in(vms->gic, irq));

    sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);

    return dev;
}
```

qdev_get_gpio_in函数是qdev_get_gpio_in_named的简单封装，用以获取指定设备的对应中断输入引脚，这里就是获取GIC的指定irq的pin。针对输出引脚的对应接口为qdev_get_gpio_out。

```c
qemu_irq qdev_get_gpio_in_named(DeviceState *dev, const char *name, int n)
{
    // 获取指定name的NamedGPIOList，如果找不到就分配一个。
    NamedGPIOList *gpio_list = qdev_get_named_gpio_list(dev, name);

    assert(n >= 0 && n < gpio_list->num_in);
    return gpio_list->in[n];
}
```

sysbus_connect_irq通过将获取的GIC的输入引脚加到GED的link属性中实现模拟GED的0号输出引脚与GIC的指定引脚连接。

### GIC(Generic Interrupt Controller)

GIC是arm平台的中断控制器，和x86平台的IOAPIC/PIC类似，用于接收硬件中断信号并通过中断路由表路由至对应的CPU进行处理。GIC可以由qemu进行模拟，也可以在kvm内核模块中模拟。考虑到性能，我们一般选择在内核中模拟中断控制器。虽然GIC功能是由kvm模块来模拟，qemu还是需要负责创建GIC设备，做一些初始化工作。创建GIC的接口为create_gic函数，其会创建GIC设备，并且初始化NamedGPIOList链表（DeivceState->gpios）。除此之外，还会将vcpu的时钟中断输出引脚、维护中断、PMU中断引脚与GIC的输入引脚相连，以及将GIC的FIQ、IRQ、VFIQ、VIRQ输出引脚与vcpu的输入引脚连接，这些逻辑应该是为了用户态模拟cpu以及GIC实现的，如果由kvm来模拟并不需要。

### PCIe主桥

virt主板的PCIe主桥称为GPEX，在create_pcie函数中将创建和初始化GPEX，其中会将GPEX的四个输出引脚的中断号设为[VIRT_PCIE, VIRT_PCIE + 3]，并将这四个输出中断引脚与 GIC的对应输入引脚相连。

```c
for (i = 0; i < GPEX_NUM_IRQS; i++) {
	sysbus_connect_irq(SYS_BUS_DEVICE(dev), i,
                       qdev_get_gpio_in(vms->gic, irq + i));
    gpex_set_irq_num(GPEX_HOST(dev), i, irq + i);
}
```

同时create_pcie_irq_map函数还会在设备树（FDT）中添加PCI插槽和GIC输入中断引脚的连接关系，即中断路由表。PCI/PCIe规范中推荐了PCI设备的传统中断引脚与中断控制器的映射关系表，可以看到32个插槽被分为了四组。因此，在create_pcie_irq_map中只需要在interrupt-map中设置0-3号slot，其余的slot通过与interrupt-map-mask进行与后再从interrupt-map中查找。

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/sys_int_remap.png){: .align-center}

```c
static void create_pcie_irq_map(const MachineState *ms,
                                uint32_t gic_phandle,
                                int first_irq, const char *nodename)
{
    int devfn, pin;
    uint32_t full_irq_map[4 * 4 * 10] = { 0 };
    uint32_t *irq_map = full_irq_map;

    for (devfn = 0; devfn <= 0x18; devfn += 0x8) {
        for (pin = 0; pin < 4; pin++) {
            int irq_type = GIC_FDT_IRQ_TYPE_SPI;
            int irq_nr = first_irq + ((pin + PCI_SLOT(devfn)) % PCI_NUM_PINS);
            int irq_level = GIC_FDT_IRQ_FLAGS_LEVEL_HI;
            int i;

            uint32_t map[] = {
                devfn << 8, 0, 0,                           /* devfn */
                pin + 1,                                    /* PCI pin */
                gic_phandle, 0, 0, irq_type, irq_nr, irq_level }; /* GIC irq */

            /* Convert map to big endian */
            for (i = 0; i < 10; i++) {
                irq_map[i] = cpu_to_be32(map[i]);
            }
            irq_map += 10;
        }
    }

    qemu_fdt_setprop(ms->fdt, nodename, "interrupt-map",
                     full_irq_map, sizeof(full_irq_map));

    qemu_fdt_setprop_cells(ms->fdt, nodename, "interrupt-map-mask",
                           cpu_to_be16(PCI_DEVFN(3, 0)), /* Slot 3 */
                           0, 0,
                           0x7           /* PCI irq */);
}
```

### ACPI相关配置

除了FDT以外，arm也可以通过ACPI表给OS传递一些硬件的信息。例如，GED、中断引脚的连接表都在ACPI的DSDT表中配置。qemu为arm平台实现的DSDT表构建函数为build_dsdt。

- GED

```c
build_ged_aml(scope, "\\_SB."GED_DEVICE,
              HOTPLUG_HANDLER(vms->acpi_dev),
              irqmap[VIRT_ACPI_GED] + ARM_SPI_BASE, AML_SYSTEM_MEMORY,
              memmap[VIRT_ACPI_GED].base);
```

- 中断路由表

```c
static void acpi_dsdt_add_pci_route_table(Aml *dev, uint32_t irq)
{
    Aml *method, *crs;
    int i, slot_no;

    /* Declare the PCI Routing Table. */
    Aml *rt_pkg = aml_varpackage(PCI_SLOT_MAX * PCI_NUM_PINS);
    for (slot_no = 0; slot_no < PCI_SLOT_MAX; slot_no++) {
        for (i = 0; i < PCI_NUM_PINS; i++) {
            int gsi = (i + slot_no) % PCI_NUM_PINS;
            Aml *pkg = aml_package(4);
            aml_append(pkg, aml_int((slot_no << 16) | 0xFFFF));
            aml_append(pkg, aml_int(i));
            aml_append(pkg, aml_name("GSI%d", gsi));
            aml_append(pkg, aml_int(0));
            aml_append(rt_pkg, pkg);
        }
    }
    aml_append(dev, aml_name_decl("_PRT", rt_pkg));

    /* Create GSI link device */
    for (i = 0; i < PCI_NUM_PINS; i++) {
        uint32_t irqs = irq + i;
        Aml *dev_gsi = aml_device("GSI%d", i);
        aml_append(dev_gsi, aml_name_decl("_HID", aml_string("PNP0C0F")));
        aml_append(dev_gsi, aml_name_decl("_UID", aml_int(i)));
        crs = aml_resource_template();
        aml_append(crs,
                   aml_interrupt(AML_CONSUMER, AML_LEVEL, AML_ACTIVE_HIGH,
                                 AML_EXCLUSIVE, &irqs, 1));
        aml_append(dev_gsi, aml_name_decl("_PRS", crs));
        crs = aml_resource_template();
        aml_append(crs,
                   aml_interrupt(AML_CONSUMER, AML_LEVEL, AML_ACTIVE_HIGH,
                                 AML_EXCLUSIVE, &irqs, 1));
        aml_append(dev_gsi, aml_name_decl("_CRS", crs));
        method = aml_method("_SRS", 1, AML_NOTSERIALIZED);
        aml_append(dev_gsi, method);
        aml_append(dev, dev_gsi);
    }
}
```

### 其他PCI桥设备

PCI桥规范没有要求桥设备传递其下PCI设备的中断请求，实际上多数PCI桥也没有为下游PCI总线提供中断引脚INTx#，但是PCI桥也推荐使用上面的映射关系表来建立下游PCI设备的INTx信号与上游PCI总线INTx信号之间的映射关系。以标准的PCI-to-PCI桥为例，设置这种中断映射关系的函数为pci_swizzle_map_irq_fn。PCI桥下设备如果使用传统的INTx，则需要将中断根据中断路由表来层层往上传递中断信号。

```c
static void pci_change_irq_level(PCIDevice *pci_dev, int irq_num, int change)
{
    PCIBus *bus;
    for (;;) {
        bus = pci_get_bus(pci_dev);
        irq_num = bus->map_irq(pci_dev, irq_num);
        if (bus->set_irq)
            break;
        pci_dev = bus->parent_dev;
    }
    pci_bus_change_irq_level(bus, irq_num, change);
}
```

### 参考资料

[1] [PCI Express体系结构导读](https://github.com/vvvlan/misc/blob/master/PCI+Express%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84%E5%AF%BC%E8%AF%BB.pdf)

[2] [Advanced Configuration and Power Interface (ACPI) Specification](https://uefi.org/specs/ACPI/6.4/index.html)





