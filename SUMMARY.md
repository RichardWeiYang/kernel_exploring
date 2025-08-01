# Summary

* [前言](README.md)
* [支持](support.md)
* [老司机带你探索内核编译系统](brief_tutorial_on_kbuild/00_index.md)
  * [编译出你的第一个内核](brief_tutorial_on_kbuild/01_build_your_first_kernel.md)
  * [内核编译中的小目标](brief_tutorial_on_kbuild/02_common_targets_in_kernel.md)
  * [可能是kbuild中最直接的小目标 – help](brief_tutorial_on_kbuild/03_first_target_help.md)
  * [使用了一个kbuild函数的目标 – cscope](brief_tutorial_on_kbuild/04_one_example_of_kbuild_function_cscope.md)
  * [内核中单个.o文件的编译过程](brief_tutorial_on_kbuild/05_rules_for_single_object.md)
  * [根目录vmlinux的编译过程](brief_tutorial_on_kbuild/06_building_vmlinux_under_root.md)
  * [启动镜像bzImage的前世今生](brief_tutorial_on_kbuild/07_rules_for_bzImage.md)
  * [setup.bin的诞生记](brief_tutorial_on_kbuild/08_rule_for_setupbin.md)
  * [真假vmlinux–由vmlinux.bin揭开的秘密](brief_tutorial_on_kbuild/09_rule_for_vmlinux_bin.md)
  * [bzImage的全貌](brief_tutorial_on_kbuild/14_bzImage_whole_picture.md)
  * [kbuild系统浅析](brief_tutorial_on_kbuild/13_root_makefile.md)
* [启动时的小秘密](bootup/00_index.md)
  * [INIT_CALLS的秘密](bootup/01_init_call.md)
  * [内核参数](bootup/02_command_line_parse.md)
* [内核加载全流程](load_kernel/00_index.md)
  * [bootloader如何加载bzImage](load_kernel/02_how_bzImage_loaded.md)
  * [内核压缩与解压](load_kernel/04_compress_decompress_kernel.md)
  * [内核加载的几个阶段](load_kernel/05_phases_in_loading.md)
  * [保护模式内核代码赏析](load_kernel/03_analysis_protected_kernel.md)
* 内存管理
  * [内核页表成长记](kernel_pagetable/00-evolution_of_kernel_pagetable.md)
    * [未解压时的内核页表](kernel_pagetable/01-pagetable_before_decompressed.md)
    * [内核早期的页表](kernel_pagetable/02-pagetable_compiled_in.md)
    * [cleanup_highmap之后的页表](kernel_pagetable/03-pagetable_after_cleanup_highmap.md)
    * [映射完整物理地址](kernel_pagetable/04-map_whole_memory.md)
    * [启用init_level4_pgt](kernel_pagetable/05-switch_to_init_level4_pgt.md)
  * [自底而上话内存](mm/00-memory_a_bottom_up_view.md)
    * [e820从硬件获取内存分布](mm/01-e820_retrieve_memory_from_HW.md)
    * [原始内存分配器--memblock](mm/02-memblock.md)
    * [页分配器](mm/page_allocator/00_page_allocator.md)
      * [寻找页结构体的位置](mm/03-sparsemem.md)
      * [眼花的页结构体](mm/10-page_struct.md)
        * [Compound Page](mm/page_allocator/01-compound_page.md)
        * [Folio](mm/14-folio.md)
      * [Node-Zone-Page](mm/05-Node_Zone_Page.md)
      * [传说的伙伴系统](mm/06-page_alloc.md)
      * [GFP的功效](mm/12-gfp_usage.md)
      * [页分配器的用户们](mm/11-users_of_buddy.md)
    * [slub分配器](mm/slub_allocator/00_slub.md)
      * [slub的理念](mm/08-slub_general.md)
      * [图解slub](mm/09-slub_in_graph.md)
    * [内存管理的不同粒度](mm/13-physical-layer-partition.md)
    * [挑战和进化](mm/50-challenge_evolution.md)
      * [扩展性的设计和实现](mm/51-scalability_design_implementation.md)
      * [减少竞争 per_cpu_pageset](mm/07-per_cpu_pageset.md)
      * [海量内存](mm/52-where_is_page_struct.md)
      * [延迟初始化](mm/54-defer_init.md)
      * [内存热插拔](mm/53-memory_hotplug.md)
      * [连续内存分配器](mm/55-cma.md)
  * [虚拟内存空间](virtual_mm/00-index.md)
    * [页表和缺页中断](virtual_mm/03-page_table_fault.md)
    * [虚拟地址空间的管家--vma](virtual_mm/05-vma.md)
    * [匿名反向映射的前世今生](virtual_mm/01-anon_rmap_history.md)
      * [图解匿名反向映射](virtual_mm/06-anon_rmap_usage.md)
    * [THP和mapcount之间的恩恩怨怨](virtual_mm/02-thp_mapcount.md)
      * [page mapcount](virtual_mm/09-mapcount.md)
    * [透明大页的玄机](virtual_mm/04-thp.md)
      * [mTHP](virtual_mm/10-mTHP.md)
    * [NUMA策略](virtual_mm/07-mempolicy.md)
    * [numa balance](virtual_mm/08-numa_balance.md)
    * [老版vma](virtual_mm/deprecate-vma.md)
  * [内存的回收再利用](mm_reclaim/00-index.md)
    * [水线](mm_reclaim/02-watermark.md)
    * [Big Picture](mm_reclaim/03-big_picture.md)
    * [手动触发回收](mm_reclaim/05-do_reclaim.md)
    * [Page Fram Reclaim Algorithm](mm_reclaim/04-pfra.md)
    * [swapfile原理使用和演进](mm_reclaim/01-swapfile.md)
  * [内存隔离](memcg/00-index.md)
    * [memcg初始化](memcg/01-init_overview.md)
    * [限制memcg大小](memcg/02-set_memcg_limit.md)
    * [对memcg记账](memcg/03-charge_memcg.md)
  * 通用
    * [常用全局变量](mm/common/00_global_variable.md)
    * [常用转换](mm/common/01_important_transform.md)
  * 测试
    * [功能测试](mm/tests/01_functional_test.md)
    * [性能测试](mm/tests/02_performance_test.md)
* [中断和异常](interrupt_exception/00-start_from_hardware.md)
  * [从IDT开始](interrupt_exception/01-idt.md)
  * [中断？异常？有什么区别](interrupt_exception/02-difference.md)
  * [系统调用的实现](interrupt_exception/03-syscall.md)
  * [异常向量表的设置](interrupt_exception/04-exception_vector_setup.md)
  * [中断向量和中断函数](interrupt_exception/05-interrupt_handler.md)
  * [APIC](interrupt_exception/06-apic.md)
  * [时钟中断](interrupt_exception/07-timer_interrupt.md)
  * [软中断](interrupt_exception/08-softirq.md)
  * [中断、软中断、抢占和多处理器](interrupt_exception/09-irq_softirq_preempt_and_smp.md)
* [设备模型](bus_driver_device/00-device_model.md)
  * [总线](bus_driver_device/01-bus.md)
  * [驱动](bus_driver_device/02-driver.md)
  * [设备](bus_driver_device/03-device.md)
  * [绑定](bus_driver_device/04-bind.md)
* [nvdimm初探](nvdimm/00-brief_navigation.md)
  * [使用手册](nvdimm/00-brief_user_guide.md)
  * [上帝视角](nvdimm/01-a_big_picture.md)
  * [nvdimm_bus](nvdimm/02-nvdimm_bus.md)
  * [nvdimm](nvdimm/03-nvdimm.md)
  * [nd_region](nvdimm/04-nd_region.md)
  * [nd_namespace_X](nvdimm/05-namespace.md)
  * [nd_dax](nvdimm/07-dax.md)
    * [dev_dax](nvdimm/09-dev_dax.md)
* [KVM](kvm/00-kvm.md)
  * [内存虚拟化](kvm/01-memory_virtualization.md)
    * [Qemu内存模型](kvm/01_1-qemu_memory_model.md)
    * [KVM内存管理](kvm/01_2-kvm_memory_manage.md)
* [cgroup](cgroup/00-index.md)
  * [使用cgroup控制进程cpu和内存](cgroup/01-control_cpu_mem_by_cgroup.md)
  * [cgroup文件系统](cgroup/02-cgroup_fs.md)
  * [cgroup层次结构](cgroup/03-hierarchy.md)
  * [cgroup和进程的关联](cgroup/04-cgroup_and_process.md)
  * [cgroup数据统计](cgroup/05-statistics.md)
* [同步机制](synchronization/00-index.md)
  * [内存屏障](synchronization/02-memory_barrier.md)
  * [RCU](synchronization/01-rcu.md)
* [Trace/Profie/Debug](tracing/00-index.md)
  * [ftrace的使用](tracing/03-ftrace_usage.md)
  * [探秘ftrace](tracing/04-ftrace_internal.md)
  * [内核热补丁的黑科技](tracing/05-kernel_live_patch.md)
  * [eBPF初探](tracing/01-ebpf.md)
  * [TraceEvent](tracing/02-trace_event.md)
  * [Drgn](tracing/06-drgn.md)
* [内核中的数据结构](data_struct/00-index.md)
  * [双链表](data_struct/01-list.md)
  * [优先级队列](data_struct/03-plist.md)
  * [哈希表](data_struct/02-hlist.md)
  * [xarray](data_struct/04-xarray.md)
  * [B树](data_struct/05-btree.md)
  * [Maple Tree](data_struct/06-maple_tree.md)
    * [Xarray vs Maple Tree](data_struct/08-xarray_vs_maple_tree.md)
  * [Interval Tree](data_struct/07-interval_tree.md)
* [Tools](tools/handy_tools.md)
  * [发补丁](tools/01-patch.md)
  * [检查文件变化](tools/02-check_file_change.md)
  * [selftest](tools/03-selftest.md)
    * [构建过程](tools/03_01-build.md)
    * [编写测试](tools/03_02-write_test.md)
* [Good To Read](reference/00-reference.md)
  * [内核自带文档](reference/03-kernel_doc.md)
  * [内存相关](reference/01-mm.md)
  * [下载社区邮件](reference/02-mail.md)
