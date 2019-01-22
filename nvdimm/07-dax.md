nd_dax设备可以在两个地方创建：

  * nd_region_probe
  * nd_pmem_probe

经过测试，在nd_pmem_probe中通过nd_dax_probe()创建的设备才是有真实信息的。而具maintainer说，nd_region_probe中生成的设备是一个用来配置的接口。这个暂时还没有去研究，这里我们只看nd_pmem_probe中通过nd_dax_probe创建的流程。

# 构造nd_dax的函数

这里我们只看通过nd_dax_probe函数创建的流程。

```
nd_dax_probe()
  nd_dax_alloc()
  nd_pfn_devinit()
  nd_pfn_validate()
  __nd_device_register()
```

主要就是做了点分配、初始化、检测的任务。

其中想要着重突出的是，如果nd_dax_probe()返回0，也就是一切正常的话。那么它的父函数就会报错，而导致驱动对namespace的probe失败。

这也是为什么我们在sysfs上看不到在这种情况下namespace的驱动的原因。

# nd_dax

```
    nd_dax
    +-------------------------------------------------+  
    |nd_pfn                                           |
    |     (struct nd_pfn)                             |
    |     +-------------------------------------------+
    |     |ndns                                       | points to the ns
    |     |    (struct nd_namespace_common*)          |
    |     |dev                                        |
    |     |    (struct device)                        |
    |     |    +--------------------------------------+
    |     |    |name                                  | "dax0.0"
    |     |    |groups                                | nd_dax_attribute_groups
    |     |    |type                                  | nd_dax_device_type
    |     |    |                                      |
    |     |    |driver_data                           | = dax_region
    |     |    |                                      |
    |     |    +--------------------------------------+
    |     |id                                         |
    |     |    (int)                                  |
    |     |uuid                                       |
    |     |    (u8*)                                  |
    |     |mode                                       | PFN_MODE_NONE
    |     |    (enum nd_pfn_mode)                     |
    |     |align                                      | HPAGE_PMD_SIZE / PAGE_SIZE
    |     |npfns                                      | tricky?
    |     |pfn_sb                                     | setup in nd_pfn_init
    |     |    (struct nd_pfn_sb*)                    |
    |     |    +--------------------------------------+
    |     |    |signature[PFN_SIG_LEN]                |
    |     |    |uuid[16]                              | = nd_pfn->uuid
    |     |    |parent_uuid[16]                       |
    |     |    |   (u8)                               |
    |     |    |mode                                  | = nd_pfn->mode
    |     |    |align                                 | = nd_pfn->align
    |     |    |   (__le32)                           |
    |     |    |dataoff                               | meta data
    |     |    |npfns                                 | number of pfn
    |     |    |   (__le64)                           |
    |     |    |start_pad                             |
    |     |    |end_trunc                             |
    |     |    |   (__le32)                           |
    |     |    |                                      |
    |     |    |                                      |
    +-----+----+--------------------------------------+  
```

东西有点长，重要的成员加了备注解释其作用。然后再着重解释两个东西：

  * pfn_sb: pfn superblock
  * pfn_sb: start_pad, end_trunc, dataoff

pfn_sb这个结构的内容将要写回到硬件，作为硬件的配置。这样就说通了作为硬件是如何知道某个内存地址的访问是在硬件的什么位置。

然后还有这么几个长得很奇怪的大兄弟是干什么的呢？且看下面这张图。

```
      |< start_pad >|<    dataoff      >|              |< end_trunc >|
      v             v 8k |page |dax_res v              v             v
      |--------------------------------------------------------------|
      ^                                                              ^
      |                                                              |
  nsio->res.start                                               nsio->res.end
```

这个nsio就是nd_pfn->ndns指向的nd_namespace_io。而这个的res也就是当前namespace所映射的空间范围。

  * start_pad, end_trunc就是为了对其128M，并不和邻居冲突做的调整
  * dataoff是一段用来存放page结构体和其他buffer的空间

好了，这样是不是觉得清楚点了？

# dax_pmem_driver

有了新的设备nd_dax，那么就有对应的驱动来干活。而它的驱动就是dax_pmem_driver。

主要做了这么几件事情：

  * 设置了pfn_sb，并写入硬件
  * 将设备中的空间hotplug到系统内存中
  * 创建了一个dax字符设备，其ops为dax_fops

其中第二件事已经超出了本章节的描述范围，如果有机会将会在内存相关的章节中描述。

而第三件事虽然也可以单独开章节讨论，不过鉴于和本章内容强相关，所以将在下一篇中详细讨论。
