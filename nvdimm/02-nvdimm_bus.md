说实话，对于nvdimm_bus中的成员含义还知之甚少。不过暂时还不影响对这个结构体生成过程的理解。

# 函数原型揭示的秘密

首先我们看一下创建nvdimm_bus这个结构调用的函数，看看它能带给我们什么信息：

```
struct nvdimm_bus *nvdimm_bus_register(struct device *parent,
		struct nvdimm_bus_descriptor *nd_desc)
```

从总可以看到，生成nvdimm_bus结构体需要两个成员：

  * parent: 父节点
  * nd_desc:描述信息

父节点含义很明确，那这个描述信息是什么，在哪里呢？

# 数据结构的关联

从数据结构的角度上看，刚才的问题就可以从acpi_nfit_desc结构体中找到：

```
    acpi_nfit_desc
    +-----------------------------------------------+
    |dev                                            | = point to an acpi_device
    |    (struct device*)                           |
    +-----------------------------------------------+
    |acpi_header                                    |
    |    (struct acpi_table_header)                 |
    +-----------------------------------------------+
    |nd_desc                                        | *  <-----------------------------------+
    |    (struct nvdimm_bus_descriptor)             |                                        |
    |    +------------------------------------------+                                        |
    |    |attr_groups                               | = acpi_nfit_attribute_groups           |
    |    |      (struct attribute_group**)          |                                        |
    |    |provider_name                             | = "ACPI.NFIT"                          |
    |    |      (char *)                            |                                        |
    |    |                                          |                                        |
    |    +------------------------------------------+                                        |
    |    |ndctl                                     | = acpi_nfit_ctl                        |
    |    |      (ndctl_fn)                          |                                        |
    |    |flush_probe                               | = acpi_nfit_flush_probe                |
    |    |clear_to_send                             | = acpi_nfit_clear_to_send              |
    |    |                                          |                                        |
    |    +------------------------------------------+                                        |
    |nvdimm_bus                                     | *                                      |
    |    (struct nvdimm_bus*)                       |                                        |
    |    +------------------------------------------+                                        |
    |    |nd_desc                                   | point to the above nd_desc  -----------+
    |    |      (struct nvdimm_bus_descriptor*)     |
    |    |dev                                       |
    |    |      (struct device)                     |
    |    |      +-----------------------------------+
    |    |      |name                               | "ndbus%d"
    |    |      |groups                             | = nd_desc->attr_groups
    |    |      |release                            | = nvdimm_bus_release
    |    |      |                                   |
    |    |      +-----------------------------------+
    |    |list                                      |
    |    |mapping_list                              |
    |    |      (struct list_head)                  |
    |    |                                          |
    +----+------------------------------------------+
```

可以看出，用来创建nvdimm_bus的描述信息nd_desc是在acpi_nfit_desc中的。这么看是不是感觉清楚了一点点？

# nd_bus_driver -> /dev/ndctl0

nvdimm_bus设备生成之后接下去的事情就交给了对应的驱动 nd_bus_driver。这个驱动超级简单，但是作用倒是有点。

```
int nvdimm_bus_create_ndctl(struct nvdimm_bus *nvdimm_bus)
{
	dev_t devt = MKDEV(nvdimm_bus_major, nvdimm_bus->id);
	struct device *dev;

	dev = device_create(nd_class, &nvdimm_bus->dev, devt, nvdimm_bus,
			"ndctl%d", nvdimm_bus->id);

	if (IS_ERR(dev))
		dev_dbg(&nvdimm_bus->dev, "failed to register ndctl%d: %ld\n",
				nvdimm_bus->id, PTR_ERR(dev));
	return PTR_ERR_OR_ZERO(dev);
}
```

可以看到主要的工作就是创建了/dev/ndctl%d设备。这下终于知道这个设备文件是如何来的了。
