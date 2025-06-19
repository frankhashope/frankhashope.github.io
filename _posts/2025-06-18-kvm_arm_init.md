---
title: "KVM在ARM架构上的初始化"
classes: "wide"
tagline: ""
header:
  overlay_color: "#48C9B0"
categories:
  - Virtual Machine Manager
toc: true
toc_label: "KVM在ARM架构上的初始化"
toc_icon: "navicon"
toc_sticky: true
---

KVM是Linux内核里集成的一种type 2虚拟化方案，它作为内核的一个模块负责虚拟化环境初始化，虚拟机和虚拟CPU模拟，以及I/O捕获与转发等功能。

本文分析基于内核的版本为6.6。

# KVM初始化：kvm_arm_init()

![image-center]({{%20site.url%20}}{{%20site.baseurl%20}}/assets/images/kvm_arm_init.png){: .align-center}

1. 检查CPU是否从EL2模式启动。ARM规范中定义了多个处理器模式，其中工作在EL2的模式称为[Hyp mode](https://developer.arm.com/documentation/ddi0406/cb/System-Level-Architecture/The-System-Level-Programmers--Model/ARM-processor-modes-and-ARM-core-registers/ARM-processor-modes?lang=en#BEICAIGC)。

2. 检查内核启动参数中是否设置了kvm-arm.mode=none。若设置为none，则直接报错返回。
   
   ```
   kvm-arm.mode=
   
           [KVM,ARM] Select one of KVM/arm64's modes of operation.
   
           none: Forcefully disable KVM.
   
           nvhe: Standard nVHE-based mode, without support for
                 protected guests.
   
           protected: nVHE-based mode with support for guests whose
                  state is kept private from the host.
   
           nested: VHE-based mode with support for nested
               virtualization. Requires at least ARMv8.3
               hardware.
   
           Defaults to VHE/nVHE based on hardware support. Setting
           mode to "protected" will disable kexec and hibernation
           for the host. "nested" is experimental and should be
           used with extreme caution.
   ```

3. kvm_sys_reg_table_init()：初始化寄存器表。主要是对各寄存器描述符数组做有效性检查，以及调用invariant_sys_regs的reset回调设置初始值。

4. kvm_set_ipa_limit()：读取[ID_AA64MMFR0_EL1](https://developer.arm.com/documentation/ddi0601/2025-03/AArch64-Registers/ID-AA64MMFR0-EL1--AArch64-Memory-Model-Feature-Register-0?lang=en)寄存器，根据PARange设置支持的IPA范围大小。

5. kvm_arm_init_sve()：sve是arm用于支持向量计算的可变长度单指令多数据指令，其特点是向量寄存器长度是可变的。该函数即被用于设置vcpu支持的最大sve向量长度。

6. kvm_arm_vmid_alloc_init()：读取[ID_AA64MMFR1_EL1](https://developer.arm.com/documentation/ddi0601/2025-03/AArch64-Registers/ID-AA64MMFR1-EL1--AArch64-Memory-Model-Feature-Register-1?lang=en)寄存器，分配VMID的位图。

7. init_hyp_mode()：针对所有上线的CPU初始化Hyp模式。
   
   a. kvm_mmu_init()：为hypervisor分配页表pgd，并且基于该pgd为hypervisor的identity段建立identity映射。
   
   b. 为Hyp mode分配per-CPU的栈页及内存。
   
   c.  建立内存的页表映射。
   
   d. kvm_hyp_init_protection()：为hypervisor的保留内存创建el2页表，do_pkvm_init()负责实际的hypervisor初始化。当do_pkvm_init()返回时，已经使用新的内存为hypervisor重新建立的页表，故初始化时建立的页表不再需要。
   ![image-center]({{%20site.url%20}}{{%20site.baseurl%20}}/assets/images/do_pkvm_init.png){: .align-center}
   
   do_pkvm_init()实际做的动作主要有：
   
   1. 调用hvc异常，将异常处理函数从__hyp_stub_vectors切换为idmap时的处理函数__kvm_hyp_init。该异常处理函数只被用于hypervisor初始化流程。
   
   2. 通过hvc异常调用__kvm_hyp_init函数，以初始化hypervisor。它主要包括初始化sp、hcr_el2等系统寄存器，设置前面创建的页表并使能mmu。最后将异常处理函数切换到最终工作时使用的版本__kvm_hyp_host_vector。
   
   3. 由于el2的初始化页表是使用内核内存建立的，而实际上step 2已经为el2保留了页表相关的内存。因此__pkvm_init会使用该内存作为页表重新为el2建立页表，并将其页表切换到新的位置。

8. kvm_init_vector_slots()：vector slots中不同slot中的vector就是根据硬件能力，用于防止不同等级spectre漏洞的向量表集合。

9. init_subsystems()：主要包括对cpu、vgic和timer的初始化流程。
   
   a. cpu_hyp_init()：使能硬件，各子系统可以访问EL2。
   
   b. hyp_cpu_pm_init()：注册CPU的低能耗通知器。
   
   c. kvm_vgic_hyp_init()：初始化VGIC。
   
   d. kvm_timer_hyp_init()：初始化定时器支持。
      ![image-center]({{%20site.url%20}}{{%20site.baseurl%20}}/assets/images/init_subsystems.png){: .align-center}

10. kvm_init()：其他一些初始化。
    
    a. 注册cpu上下线的回调kvm_online_cpu()和kvm_offline_cpu()。
    
    b. 注册kvm_syscore_ops。
    
    c. kvm_irqfd_init()：创建工作队列kvm-irqfd-cleanup，用于处理延迟的所有虚机关闭请求。
    
    d. kvm_vfio_ops_init()：注册kvm_vfio_ops。
    
    e. 创建/dev/kvm虚拟设备文件。

# 参考资料

[1] [基于armv8的kvm实现分析（三）kvm初始化流程](https://zhuanlan.zhihu.com/p/530130205)

[2] [Linux虚拟化KVM-Qemu分析（三）之KVM源码](https://www.cnblogs.com/LoyenWang/p/13659024.html)
