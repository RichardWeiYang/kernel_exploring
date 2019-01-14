创建了nvdimm_bus，读取了nfit中的信息后，接下来驱动就根据这些信息创建nvdimm这个数据结构了。

这个数据结构就对应着一个物理的dimm，这可能也是为什么使用memmap=内核参数时没有这个设备的原因。

# 构造nvdimm的函数

这个过程由acpi_nfit_register_dimms()函数完成，大致的流程如下：

```
list_for_each_entry(nfit_mem, &acpi_desc->dimms, list) {
    __nvdimm_create();
}
```

从这个片段能看出nvdimm结构体是从acpi_desc->dimms的信息构建的。

# 相应的数据结构

看了函数调用，现在来看看对应的数据结构的样子。虽然看了有时候也还是不懂。


```
    acpi_nfit_desc
    +-----------------------------------------------+
    |dimms                                          |  a list of nfit_mem
    |    (struct list_head)                         |  parsed from above spa & memdevs
    |    +------------------------------------------+
    |    |nvdimm                                    |  *
    |    |     (struct nvdimm*)                     |
    |    |     +------------------------------------+
    |    |     |dev                                 |
    |    |     |   (struct device)                  |
    |    |     |   +--------------------------------+
    |    |     |   |name                            |  = "nmem%d"
    |    |     |   |groups                          |  = acpi_nfit_dimm_attribute_groups
    |    |     |   |                                |
    |    |     |   |driver_data                     |  = struct nvdimm_drvdata
    |    |     |   |   (void*)                      |
    |    |     |   |  +-----------------------------+
    |    |     |   |  |dev                          |
    |    |     |   |  |   (struct device*)          |
    |    |     |   |  |ns_current, ns_next          |
    |    |     |   |  |nslabel_size                 |  = 128
    |    |     |   |  |   (int)                     |
    |    |     |   |  |nsarea                       |
    |    |     |   |  |     (nd_cmd_get_config_size)|
    |    |     |   |  |data                         |  config data
    |    |     |   |  |   (void*)                   |  namespace index + namespace label
    |    |     |   |  |dpa                          |  created from namespace label
    |    |     |   |  |   (struct resource)         |  from nvdimm_drvdata->data
    |    |     |   |  +-----------------------------+
    |    |     |   |                                |
    |    |     |   +--------------------------------+
    |    |     |provider_data                       |  = nfit_mem
    |    |     |   (void*)                          |
    |    |     |flush_wpq                           |
    |    |     |   (struct resource*)               |
    |    |     |dwork                               |
    |    |     |   (struct delayed_work)            |
    |    |     |sec.ops                             |
    |    |     |                                    |
    |    |     |                                    |
    |    +-----+------------------------------------+
    |    |memdev_dcr                                |
    |    |memdev_pmem                               |
    |    |memdev_bdw                                |
    |    |     (struct acpi_nfit_memory_map)        |
    |    |                                          |
    |    +------------------------------------------+
    |                                               |
    +-----------------------------------------------+
```

在这个数据结构的部分截取中可以看到，acpi_nfit_desc中包含了一个类型为nfit_mem的链表。而函数acpi_nfit_register_dimms就是根据这个链表来构造硬件相对无关的nvdimm数据。

# nvdimm_driver

nvdimm对应的是一个物理的dimm，这个设备生成之后就需要去检查并初始化。这个工作由它的驱动nvdimm_driver来完成。

驱动做的关键工作就是根据硬件信息设置这个数据结构中的dev->driver_data。

这个结构是一个nvdimm_drvdata类型，其具体的成员也已经在图中显示。主要解释其中两个内容：

  * data: 驱动将硬件上的namespace index和namespace label_size读取到这段内存
  * dpa:  驱动根据上面的data信息，将dimm对应的dpa信息转换成res树

我猜这些信息会留着后续使用，那就等着看吧。
