接着我们来看nd_region这个数据结构。

# 构造nd_region的函数

老规矩，先看看构造这个数据结构的函数样子。

```
acpi_nfit_register_regions()
  list_for_each_entry(nfit_spa, &acpi_desc->spas, list) {
    rc = acpi_nfit_register_region(acpi_desc, nfit_spa);
      struct nd_region_desc *ndr_desc;
      nfit_spa->nd_region = nvdimm_pmem_region_create(nvdimm_bus, ndr_desc);
  }
```

这是一个非常简略的构建流程，有几点含义想要表达：

  * nd_region的创建由一个acpi_desc->spa触发，并最后挂在spa上
  * 构造的过程中由一个中间变量ndr_desc，由它构造了最后的nd_region

# 相关的数据结构

这次的牵扯到的数据结构比较多了。

## acpi_nfit_desc

```
    acpi_nfit_desc
    +-----------------------------------------------+
    |dev                                            | = point to an acpi_device
    |    (struct device*)                           |
    +-----------------------------------------------+
    |acpi_header                                    |
    |    (struct acpi_table_header)                 |
    +-----------------------------------------------+
    |spas                                           |  a list of nfit_spa
    |    (struct list_head)                         |  added from acpi_table
    |    +------------------------------------------+  
    |    |nd_region                                 |  *  created from nfit_spa
    |    |    (struct nd_region*)                   |
    |    |ars_state                                 |
    |    |    (unsigned long)                       |
    |    |spa[0]                                    |  looks just have one spa
    |    |    (struct acpi_nfit_system_address)     |
    |    |    +-------------------------------------+
    |    |    |header                               |
    |    |    |    (struct acpi_nfit_header)        |
    |    |    |range_guid[16]                       |
    |    |    |    (u8)                             |
    |    |    |range_index                          |
    |    |    |flags                                |
    |    |    |address                              |  spa address
    |    |    |length                               |
    |    |    |    (u16)                            |
    |    |    |                                     |
    |    +----+-------------------------------------+
    +-----------------------------------------------+
```

首先来看一下acpi_nfit_desc。其中有一个叫spas的链表。这个的数据在前期从acpi中读取并解析后保存在此。而其中的信息就是构造nd_region的基础。

## nd_region_desc

这个结构就是用来构造nd_region的描述结构。在nvdimm驱动中已经是第二次看到这种描述结构了，看来驱动的作者挺喜欢这种方式。

其中在mapping字段可以看出当前nd_region和nvdimm之间的一个关联。

```
    nd_region_desc
    +-----------------------------------------------+
    |res                                            |
    |     (struct resource*)                        |
    |     +-----------------------------------------+
    |     |name                                     |
    |     |start                                    |  nfit_spa->address
    |     |end                                      |  nfit_spa->address + nfit_spa->length - 1
    +-----+-----------------------------------------+
    |provider_data                                  |  = nfit_spa
    |     (void*)                                   |
    |attr_groups                                    |  = acpi_nfit_region_attribute_groups
    |     (struct attribute_group**)                |
    |num_lanes                                      |
    |numa_node                                      |
    |     (int)                                     |
    |flags                                          |
    |     (unsigned logn)                           |
    |                                               |
    |                                               |
    +-----------------------------------------------+
    |num_mappings                                   |
    |     (u16)                                     |
    |mapping                                        |  created from acpi_nfit_desc->memdev
    |     (struct nd_mapping_desc*)                 |
    |     +-----------------------------------------+
    |     |nvdimm                                   |
    |     |    (struct nvdimm*)                     |
    |     |start/size                               |
    |     |    (u64)                                |
    |     |position                                 |
    |     |    (int)                                |
    +-----+-----------------------------------------+
    |nd_set                                         |  * will be used in nd_region
    |     (struct nd_interleave_set*)               |
    |     +-----------------------------------------+
    |     |cookie1                                  |
    |     |cookie2                                  |
    |     |altcookie                                |
    |     |    (u64)                                |
    |     |type_guid                                |
    |     |    (guid_t)                             |
    +-----+-----------------------------------------+
```

## nd_region

最后就是nd_region这个结构本身了。其中大部分的数据是从刚才的 nd_region_desc 中拷贝过来的。

不过有意思的是，这个nd_region_desc之后就消失了，没有人会记得曾经还有个这么重要的角色。

```
    nd_region
    +-------------------------------------------------+  
    |dev                                              |
    |   (struct device)                               |
    |   +---------------------------------------------+
    |   |groups                                       | = acpi_nfit_region_attribute_groups
    |   |    (struct attr_groups**)                   |
    |   |                                             |
    |   +---------------------------------------------+
    |   |type                                         | = &nd_pmem_device_type
    |   |    (struct device_type*)                    |   &nd_blk_device_type
    |   |                                             |   &nd_volatile_device_type
    |   |driver_data                                  | = nd_region_data
    |   |    (void*) nd_region_data                   |
    |   |    +----------------------------------------+
    |   |    |ns_count                                |
    |   |    |ns_active                               |
    |   |    |    (int)                               |
    |   |    |hints_shift                             |
    |   |    |    (unsigned int)                      |
    |   |    |flush_wpq[0]                            |
    |   |    |    (void*)                             |
    +---+----+----------------------------------------+  
    |ns_ida                                           |
    |btt_ida                                          |
    |pfn_ida                                          |
    |dax_ida                                          |
    |     (struct ida)                                |
    +-------------------------------------------------+  
    |ns_seed                                          |
    |btt_seed                                         |
    |pfn_seed                                         |
    |dax_seed                                         |
    |     (struct device*)                            |
    +-------------------------------------------------+  
    |lane                                             |
    |     (struct nd_percpu_lan)                      |
    +-------------------------------------------------+  
    |provider_data                                    | = ndr_desc->provider_data
    |     (void *)                                    |
    |ndr_mappings                                     | = ndr_desc->num_mappings
    |     (u16)                                       |
    |ndr_start                                        | = ndr_desc->res->start
    |ndr_size                                         | = ndr_desc->res:size
    |     (u16/u64)                                   |
    |num_lanes                                        | = ndr_desc->num_lanes
    |numa_node                                        | = ndr_desc->numa_node
    |     (int)                                       |
    |                                                 |
    |nd_set                                           | = ndr_desc->nd_set
    |     (struct nd_interleave_set*)                 |
    |                                                 |
    +-------------------------------------------------+  
    |mapping[0]                                       | copied from ndr_des->mapping
    |     (struct nd_mapping)                         |
    |     +-------------------------------------------+
    |     |start/size/position                        |
    |     |    (u64/int)                              |
    |     |labels                                     | a list of
    |     |    (struct list_head)                     | nd_namespace_label
    |     |    +--------------------------------------+
    |     |    |     uuid[NSLABEL_UUID_LEN]           |
    |     |    |     name[NSLABEL_NAME_LEN]           |
    |     |    |     flags                            |
    |     |    |     nlabel                           |
    |     |    |     position                         |
    |     |    |     dpa                              |
    |     |    |                                      |
    |     |    |                                      |
    |     |    +--------------------------------------+
    |     |nvdimm                                     |
    |     |    (struct nvdimm*)                       |
    |     |ndd                                        | = nvdimm->dev->driver_data
    |     |    (struct nvdimm_drvdata*)               |
    |     |                                           |
    |     |                                           |
    +-----+-------------------------------------------+  
```

补充一点，nd_regioin一共有三种类型:

  * nd_pmem
  * nd_volatile
  * nd_blk

这个类型的区分由dev->type的值来区分，对应上面三种类型各自的type值为：

  * nd_pmem_device_type
  * nd_volatile_device_type
  * nd_blk_device_type

到这里基本算可以结束了，但是我还想再往回推一层。那就是这个type的值是谁来决定的。

在本节的开头我们也看到了，每个nd_region是在acpi_nfit_desc->spa的基础上创建的。而在代码中也正好印证了这一点。nd_region->type也是基于spa的类型决定的。

```
int nfit_spa_type(struct acpi_nfit_system_address *spa)
{
	int i;

	for (i = 0; i < NFIT_UUID_MAX; i++)
		if (guid_equal(to_nfit_uuid(i), (guid_t *)&spa->range_guid))
			return i;
	return -1;
}
```

这个函数将spa->range_guid和一个表比较，从而得出当前spa的类型，也就是nd_region的类型了。

## nd_region_driver

当一个nd_region构造好了之后，其对应的驱动就该登场干活了。这次就是nd_region_driver，而且还干了不少活。

```
nd_region_probe()
  nd_region_activate()
  nd_blk_region_init()
  nd_region_register_namespaces()
  nd_region->btt_seed = nd_btt_create(nd_region);
  nd_region->pfn_seed = nd_pfn_create(nd_region);
  nd_region->dax_seed = nd_dax_create(nd_region);
```

其做了一些初始化的工作后又做了几件重要的事：

  * 创建namespace
  * 创建btt/pfn/dax设备

接下来我们就分步骤看看这几个重要的设备的创建过程。
