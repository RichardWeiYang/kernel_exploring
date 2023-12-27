在nd_region的驱动中我们看到其中一个重要的动作就是创建namespace。这个工作交给了nd_region_register_namespaces()函数处理。

# 构造nd_namespace的函数

先来看看构造namespace的函数流程。

```
nd_region_register_namespaces(struct nd_region *nd_region)
  init_active_labels(struct nd_region *nd_region)
  type = nd_region_to_nstype(nd_region);
  switch (type) {
  case ND_DEVICE_NAMESPACE_IO:
    devs = create_namespace_io(nd_region);
    break;
  case ND_DEVICE_NAMESPACE_PMEM:
  case ND_DEVICE_NAMESPACE_BLK:
    devs = create_namespaces(nd_region);
    break;
  default:
    break;
  }
```

大致看主要做了两件事：

  * 配置nd_regioin->mapping->labels
  * 根据nd_region的NS类型创建对应的namespace

其中labels数据类型看定义就是RFC4122中定义的label了。也算是找到了一个硬件和软件对应的关系。

而create_namespaces函数的工作则就是扫描nd_region->mapping->labels来构造的。具体做了什么，让我们来看看相关的数据结构。

# nd_namespace_X

在上面的代码片段和nd_region的章节中我们都可以看出，namespace有三种类型

  * nd_namespace_blk
  * nd_namespace_io
  * nd_namespace_pmem

所以内核中用了三种类型分别表述。我们一个个来看。

## nd_namespace_blk

下面是nd_namespace_blk这个数据结构的内容。

```
    nd_namespace_blk
    +-------------------------------------------------+  
    |common                                           |
    |    (struct nd_namespace_common)                 |
    |    +--------------------------------------------+
    |    |dev                                         |
    |    |    (struct device)                         |
    |    |    +---------------------------------------+
    |    |    |type                                   | = namespace_blk_device_type
    |    |    |                                       |
    |    |    +---------------------------------------+
    |    |claim                                       |
    |    |    (struct device*)                        |
    |    +--------------------------------------------+  
    |    |force_raw                                   |
    |    |    (int)                                   |
    |    +--------------------------------------------+  
    |    |rw_bytes                                    |
    |    |    ()                                      |
    |    +--------------------------------------------+  
    |    |claim_class                                 | = BTT/BTT2/PFN/DAX
    |    |    (enum nvdimm_claim_class)               |
    +----+--------------------------------------------+  
    |alt_name                                         | = nd_label->name
    |    (char *)                                     |
    |uuid                                             | = nd_label->uuid
    |    (u8*)                                        |
    +-------------------------------------------------+  
    |id                                               |
    |    (int)                                        |
    |lbasize                                          |
    |    (unsigned long)                              |
    |size                                             |
    |    (resource_size_t)                            |
    +-------------------------------------------------+  
    |num_resources                                    |
    |    (int)                                        |
    |res                                              | dpa res array
    |    (struct resource**)                          |
    +-------------------------------------------------+  
```

创建的时候根据nd_label的信息填写了uuid, alt_name以及dpa resource数组。

## nd_namespace_io

下面是nd_namespace_io的数据结构。

```
    nd_namespace_io
    +-------------------------------------------------+  
    |common                                           |
    |    (struct nd_namespace_common)                 |
    |    +--------------------------------------------+
    |    |dev                                         |
    |    |    (struct device)                         |
    |    |    +---------------------------------------+
    |    |    |type                                   | = namespace_io_device_type
    |    |    |                                       |
    |    |    +---------------------------------------+
    |    |claim                                       |
    |    |    (struct device*)                        |
    |    +--------------------------------------------+  
    |    |force_raw                                   |
    |    |    (int)                                   |
    |    +--------------------------------------------+  
    |    |rw_bytes                                    |
    |    |    ()                                      |
    |    +--------------------------------------------+  
    |    |claim_class                                 |
    |    |    (enum nvdimm_claim_class)               |
    +----+--------------------------------------------+  
    |size                                             |
    |    (resource_size_t)                            |
    +-------------------------------------------------+  
    |res                                              |
    |    (struct resource)                            |
    |    +--------------------------------------------+
    |    |start                                       | = nd_region->ndr_start
    |    |end                                         | = res->start + nd_region->ndr_size - 1
    |    |                                            |
    +----+--------------------------------------------+  
    |addr                                             |
    |    (void *)                                     |
    +-------------------------------------------------+  
    |bb                                               |
    |    (struct badblock)                            |
    +-------------------------------------------------+  
```

这种namespace比较简单，创建的时候除了设置了type，就是把res设置成了nd_region中的空间。

## nd_namespace_pmem

```
    nd_namespace_pmem
    +-------------------------------------------------+  
    |nsio                                             |
    |    (struct nd_namespace_io)                     |
    +----+--------------------------------------------+
    |    |common                                      |
    |    |    (struct nd_namespace_common)            |
    |    |    +---------------------------------------+
    |    |    |dev                                    |
    |    |    |    (struct device)                    |
    |    |    |    +----------------------------------+
    |    |    |    |type                              | = namespace_pmem_device_type
    |    |    |    |                                  |
    |    |    |    +----------------------------------+
    |    |    |claim                                  |
    |    |    |    (struct device*)                   |
    |    |    +---------------------------------------+  
    |    |    |force_raw                              |
    |    |    |    (int)                              |
    |    |    +---------------------------------------+  
    |    |    |rw_bytes                               |
    |    |    |    ()                                 |
    |    |    +---------------------------------------+  
    |    |    |claim_class                            |
    |    |    |    (enum nvdimm_claim_class)          |
    |    +----+---------------------------------------+  
    |    |size                                        |
    |    |    (resource_size_t)                       |
    |    +--------------------------------------------+  
    |    |res                                         |
    |    |    (struct resource)                       |
    |    |    +---------------------------------------+
    |    |    |flags                                  | = IORESOURCE_MEM
    |    |    |start                                  | = nd_region->ndr_start + dpa/spa offset
    |    |    |end                                    | = start + size - 1
    |    |    |                                       |
    |    +----+---------------------------------------+  
    |    |addr                                        |
    |    |    (void *)                                |
    |    |--------------------------------------------+  
    |    |bb                                          |
    |    |    (struct badblock)                       |
    +----+--------------------------------------------+  
    |alt_name                                         | = nd_label->name
    |    (char *)                                     |
    |lbasize                                          | = nd_label->lbasize
    |    (unsigned long)                              |
    |uuid                                             | = nd_label->uuid
    |    (u8*)                                        |
    +-------------------------------------------------+  
```

pmem最后整合了一个resource。我猜这个res表示的范围就是内存空间的物理地址了？

# nd_pmem/blk_driver

以上三个namespace都会生成自己的设备，这样也就都有各自对应的驱动。

  * nd_blk_driver
  * nd_pmem_driver

很不好意思，nd_pmem_driver包揽了第二三种namespace的驱动，真实能者多劳的典范。

不过这两个驱动都有一个共同点，就是都调用了nvdimm_namespace_common_probe()做初始化。

这个函数其实也没做啥，主要做了两件事：

  * 检测空间大小是否合适
  * 设置了uuid

经过仔细研读和实验验证，结果发现了几个神奇的事儿。

  * nd_pmem_driver不仅仅是namespace的驱动，还是btt/pfn设备的驱动。
  * nd_region_probe()会也建立btt/pfn/dax设备。
  * nd_pmem_probe也会通过nd_btt/pfn/dax_probe建立对应的设备。

关于这个过程的具体细节可以看[lkml][1]中的这个讨论，或许可以让逻辑清楚一点。

总之当namespace被配置成这三种类型的某一种时，nd_pmem_probe会通过nd_btt/pfn/dax_probe去创建某一种类型的设备。并且有意思的是，这种情况下nd_pmem_probe会**失败**。这种高端的方法我还是头一次见。

接下来的事情就交给了各自的设备和驱动了。

[1]: https://lkml.org/lkml/2019/1/18/1026
