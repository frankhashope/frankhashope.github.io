---
title: "虚拟机的重启、关机以及重置"
tagline: ""
header:
  overlay_color: "#48C9B0"
classes: wide
categories:
  - Virtualization
---

和物理计算机一样，虚拟机同样需要类似重启、关机、重置等的生命周期操作，本文基于qemu/KVM分析上述生命周期在系统虚拟化的实现。

### 关机

x86下南桥规定了一系列电源管理的I/O端口寄存器，一共128个，且以128字节对齐。可以置于I/O空间的任何位置，起始地址PMBASE由固件或者OS指定。qemu模拟的q35机型由ich9_pm_init()函数注册其内存区间及相关读写的回调。其中，PM1_CNT寄存器注册的回调为acpi_pm_cnt_ops。当linux虚拟机（内核未配置CONFIG_ACPI_REDUCED_HARDWARE_ONLY）下发关机命令时，内核会最终调到acpi_hw_legacy_sleep()写PM1_CNT寄存器，将虚拟机设置成S5状态。

ACPI规定了可以在DSDT表中配置进入Sx状态的方法，即向PM1a_CNT.SLP_TYPE/PM1b_CNT.SLP_TYPE写入指定的值。
![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/acpi_system_state.png){: .align-center}

qemu在build_dsdt()中配置了\\_S5的package，当Guest OS向PM1a_CNT.SLP_TYPE写入0且PM1a_CNT.SLP_EN被置位时将进行关机操作。acpi_pm1_cnt_write()函数是为写PM1_CNT寄存器注册的回调接口。

```c
/* ACPI PM1aCNT */
static void acpi_pm1_cnt_write(ACPIREGS *ar, uint16_t val)
{
    ar->pm1.cnt.cnt = val & ~(ACPI_BITMASK_SLEEP_ENABLE);

    if (val & ACPI_BITMASK_SLEEP_ENABLE) {
        /* change suspend type */
        uint16_t sus_typ = (val >> 10) & 7;
        switch (sus_typ) {
        case 0: /* soft power off */
            qemu_system_shutdown_request(SHUTDOWN_CAUSE_GUEST_SHUTDOWN);
            break;
        case 1:
            qemu_system_suspend_request();
            break;
        default:
            if (sus_typ == ar->pm1.cnt.s4_val) { /* S4 request */
                qapi_event_send_suspend_disk();
                qemu_system_shutdown_request(SHUTDOWN_CAUSE_GUEST_SHUTDOWN);
            }
            break;
        }
    }
}
```

可见当接收到Guest OS下发的关机指令后，qemu调用了qemu_system_shutdown_request()。

```c
void qemu_system_shutdown_request(ShutdownCause reason)
{
    trace_qemu_system_shutdown_request(reason);
    replay_shutdown_request(reason);
    shutdown_requested = reason;
    qemu_notify_event();  // 通知主线程执行qemu_aio_context的下半部任务
}
```

shutdown_requested记录了关机类型，可以有以下几种：

```c
typedef enum ShutdownCause {
    SHUTDOWN_CAUSE_NONE,
    SHUTDOWN_CAUSE_HOST_ERROR,
    SHUTDOWN_CAUSE_HOST_QMP_QUIT,
    SHUTDOWN_CAUSE_HOST_QMP_SYSTEM_RESET,
    SHUTDOWN_CAUSE_HOST_SIGNAL,
    SHUTDOWN_CAUSE_HOST_UI,
    SHUTDOWN_CAUSE_GUEST_SHUTDOWN,
    SHUTDOWN_CAUSE_GUEST_RESET,
    SHUTDOWN_CAUSE_GUEST_PANIC,
    SHUTDOWN_CAUSE_SUBSYSTEM_RESET,
    SHUTDOWN_CAUSE__MAX,
} ShutdownCause;
```

主线程每次循环都会判断有没有收到关机请求，如果收到则调用qemu_system_shutdown()通过qmp发送关机事件以及执行注册的关机钩子函数。

```c
static void qemu_system_shutdown(ShutdownCause cause)
{
    qapi_event_send_shutdown(shutdown_caused_by_guest(cause), cause);
    notifier_list_notify(&shutdown_notifiers, &cause);
}
```

主线程结束后qemu_cleanup()会清理一些资源，等待block设备I/O数据下发完成。其中vm_shutdown()调用do_vm_stop()完成停止vcpu以及flush块设备尚未完成下发的I/O。

```c
static int do_vm_stop(RunState state, bool send_stop)
{
    int ret = 0;

    if (runstate_is_running()) {
        runstate_set(state);
        cpu_disable_ticks();
        pause_all_vcpus();
        vm_state_notify(0, state);
        if (send_stop) {
            qapi_event_send_stop();
        }
    }

    bdrv_drain_all();
    ret = bdrv_flush_all();
    trace_vm_stop_flush_all(ret);

    return ret;
}
```

arm上定义了标准的电源管理接口PSCI(Power State Coordination Interface)，qemu/KVM实现了这套接口以实现arm上的虚拟机的电源管理。当虚拟机内部下发poweroff等关机命令时，vcpu会触发hvc。KVM捕获到异常后，以KVM_EXIT_SYSTEM_EVENT陷出到qemu。qemu最终调用的同样是qemu_system_shutdown_request()接口，后面的流程和x86一样，不再赘述。

```c
case KVM_EXIT_SYSTEM_EVENT:
    switch (run->system_event.type) {
    case KVM_SYSTEM_EVENT_SHUTDOWN:
        qemu_system_shutdown_request(SHUTDOWN_CAUSE_GUEST_SHUTDOWN);
        ret = EXCP_INTERRUPT;
        break;
    ......
    }
    break;
```

除了在虚拟机内部通过命令可以关机外，也可以通过virsh shutdown关闭虚拟机。virsh shutdown实际是通过QMP接口，最终qemu会调用qemu_system_powerdown_request()将powerdown_requested置1，并同样通知到主线程。main_loop_should_exit()中通过qemu_powerdown_requested()函数判断powerdown_requested是否已置1，如果已置1则调用qemu_system_powerdown()接口。

```c
static void qemu_system_powerdown(void)
{
    qapi_event_send_powerdown();
    notifier_list_notify(&powerdown_notifiers, NULL);
}
```

q35主板的南桥往powerdown_notifiers注册了pm_powerdown_req()，其调用acpi_pm1_evt_power_down()模拟按下关机的电源键发送SCI通知Guest OS。为此，ICH9注册了PM1a_EVT_BLK寄存器组，其中包括PM1_STS和PM1_EN两个寄存器。它们和前面的PM1_CNT一样都是ACPI规定的固定硬件寄存器。

```c
static void pm_powerdown_req(Notifier *n, void *opaque)
{
    ICH9LPCPMRegs *pm = container_of(n, ICH9LPCPMRegs, powerdown_notifier);

    acpi_pm1_evt_power_down(&pm->acpi_regs);
}

void acpi_pm1_evt_power_down(ACPIREGS *ar)
{
    if (ar->pm1.evt.en & ACPI_BITMASK_POWER_BUTTON_ENABLE) {
        ar->pm1.evt.sts |= ACPI_BITMASK_POWER_BUTTON_STATUS;
        ar->tmr.update_sci(ar);
    }
}

void acpi_update_sci(ACPIREGS *regs, qemu_irq irq)
{
    int sci_level, pm1a_sts;

    pm1a_sts = acpi_pm1_evt_get_sts(regs);

    sci_level = ((pm1a_sts &
                  regs->pm1.evt.en & ACPI_BITMASK_PM1_COMMON_ENABLED) != 0) ||
                ((regs->gpe.sts[0] & regs->gpe.en[0]) != 0);

    // q35: 调用ich9_set_sci()注入SCI中断
    qemu_set_irq(irq, sci_level);

    /* schedule a timer interruption if needed */
    acpi_pm_tmr_update(regs,
                       (regs->pm1.evt.en & ACPI_BITMASK_TIMER_ENABLE) &&
                       !(pm1a_sts & ACPI_BITMASK_TIMER_STATUS));
}
```

类似的，virt机型通过qemu_register_powerdown_notifier()注册了virt_powerdown_req()这个通知函数。低于4.2版本的virt机型不支持ACPI GED，通过GPIO上报关机事件。否则，调用acpi_ged_send_event()通过ACPI的power button project上报。

```c
static void virt_powerdown_req(Notifier *n, void *opaque)
{
    VirtMachineState *s = container_of(n, VirtMachineState, powerdown_notifier);

    if (s->acpi_dev) {
        acpi_send_event(s->acpi_dev, ACPI_POWER_DOWN_STATUS);
    } else {
        /* use gpio Pin 3 for power button event */
        qemu_set_irq(qdev_get_gpio_in(gpio_key_dev, 0), 1);
    }
}

static void acpi_ged_send_event(AcpiDeviceIf *adev, AcpiEventStatusBits ev)
{
    AcpiGedState *s = ACPI_GED(adev);
    GEDState *ged_st = &s->ged_state;
    uint32_t sel;

    if (ev & ACPI_MEMORY_HOTPLUG_STATUS) {
        sel = ACPI_GED_MEM_HOTPLUG_EVT;
    } else if (ev & ACPI_POWER_DOWN_STATUS) {
        sel = ACPI_GED_PWR_DOWN_EVT;
    } else if (ev & ACPI_NVDIMM_HOTPLUG_STATUS) {
        sel = ACPI_GED_NVDIMM_HOTPLUG_EVT;
    } else {
        /* Unknown event. Return without generating interrupt. */
        warn_report("GED: Unsupported event %d. No irq injected", ev);
        return;
    }

    /*
     * Set the GED selector field to communicate the event type.
     * This will be read by GED aml code to select the appropriate
     * event method.
     */
    ged_st->sel |= sel;

    /* Trigger the event by sending an interrupt to the guest. */
    qemu_irq_pulse(s->irq);
}
```

### 重启

计算机的重启可以有多种方式，比如基于ACPI reset寄存器、键盘控制器、x86平台南桥的0xCF9端口、直接调用固件接口等。同样，qemu也为虚拟机实现了上述多种方式的重启实现。

目前物理计算机的电源管理大部分均采用ACPI技术，它是英特尔等公司提出的操作系统应用程序管理所有电源管理接口的规范，包括了软件和硬件方面的规范，操作系统的电源管理功能通过调用 ACPI 接口，实现对符合 ACPI 规范的硬件设备的电源管理。其中，ACPI规定了一个reset register，通过向这个寄存器写入特定值来重置计算机。

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/acpi_reset_register.png){: .align-center}

qemu中只有x86下的q35机型实现了通过该寄存器重置虚拟机的方式：将该寄存器与南桥的RST_CNT寄存器（0xCF9端口）绑定，并通过build_fadt()函数构建到FADT表。

```c
if (lpc) {
    uint64_t smi_features = object_property_get_uint(lpc,
        ICH9_LPC_SMI_NEGOTIATED_FEAT_PROP, NULL);
    struct AcpiGenericAddress r = { .space_id = AML_AS_SYSTEM_IO,
        .bit_width = 8, .address = ICH9_RST_CNT_IOPORT };
    pm->fadt.reset_reg = r;
    pm->fadt.reset_val = 0xf;
    pm->fadt.flags |= 1 << ACPI_FADT_F_RESET_REG_SUP;
    pm->cpu_hp_io_base = ICH9_CPU_HOTPLUG_IO_BASE;
    pm->smi_on_cpuhp =
        !!(smi_features & BIT_ULL(ICH9_LPC_SMI_F_CPU_HOTPLUG_BIT));
    pm->smi_on_cpu_unplug =
        !!(smi_features & BIT_ULL(ICH9_LPC_SMI_F_CPU_HOT_UNPLUG_BIT));
}
```

RST_CNT寄存器的读写回调函数为ich9_rst_cnt_read/ich9_rst_cnt_write，其中Guest OS通过写RST_CNT寄存器来触发重置。

```c
/* reset control */
static void ich9_rst_cnt_write(void *opaque, hwaddr addr, uint64_t val,
                               unsigned len)
{
    ICH9LPCState *lpc = opaque;

    if (val & 4) {
        qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
        return;
    }
    lpc->rst_cnt = val & 0xA; /* keep FULL_RST (bit 3) and SYS_RST (bit 1) */
}
```

与关机一样，主线程每次循环都会判断是否收到了RESET请求。

```c
request = qemu_reset_requested();
if (request) {
    pause_all_vcpus();
    qemu_system_reset(request);
    resume_all_vcpus();
    /*
     * runstate can change in pause_all_vcpus()
     * as iothread mutex is unlocked
     */
    if (!runstate_check(RUN_STATE_RUNNING) &&
            !runstate_check(RUN_STATE_INMIGRATE) &&
            !runstate_check(RUN_STATE_FINISH_MIGRATE)) {
        runstate_set(RUN_STATE_PRELAUNCH);
    }
}
```

qemu_system_reset()里如果对应机型有注册reset接口，则调用注册的reset接口，否则默认就调用qemu_devices_reset()。设备通过qemu_register_reset()注册对应的reset回调到全局链表中。

```c
void qemu_devices_reset(void)
{
    QEMUResetEntry *re, *nre;

    /* reset all devices */
    QTAILQ_FOREACH_SAFE(re, &reset_handlers, entry, nre) {
        re->func(re->opaque);
    }
}
```

arm下的virt机型和关机一样，通过PSCI实现重启，可见其最终也是调用了qemu_system_reset_request()来发送重置请求。

```c
case KVM_EXIT_SYSTEM_EVENT:
    switch (run->system_event.type) {
    case KVM_SYSTEM_EVENT_RESET:
        qemu_system_reset_request(SHUTDOWN_CAUSE_GUEST_RESET);
        ret = EXCP_INTERRUPT;
        break;
    ......
    }
    break;
```

与前两者类似，libvirt也提供了重启虚拟机的管理命令virsh reboot。virsh reboot实质是通过qmp接口先后发送了system_powerdown以及system_reset命令来实现虚拟机的重启，即qemu里会先后调到qmp_system_powerdown()以及qmp_system_reset()。期间还会分别pause和resume虚拟机。

### 重置

virsh reset可以重置虚拟机，其通过qmp接口调用qmp_system_reset()接口通知到主线程。其与virsh reboot的差别就是虚拟机不需要先shutdown，模拟物理上的重置键，因此可能会有数据丢失。

```c
void qmp_system_reset(Error **errp)
{
    qemu_system_reset_request(SHUTDOWN_CAUSE_HOST_QMP_SYSTEM_RESET);
}

void qemu_system_reset_request(ShutdownCause reason)
{
    if (reboot_action == REBOOT_ACTION_SHUTDOWN &&
        reason != SHUTDOWN_CAUSE_SUBSYSTEM_RESET) {
        shutdown_requested = reason;
    } else if (!cpus_are_resettable()) {
        error_report("cpus are not resettable, terminating");
        shutdown_requested = reason;
    } else {
        reset_requested = reason;
    }
    cpu_stop_current();
    qemu_notify_event();
}
```