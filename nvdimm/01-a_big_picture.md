名字起的大了点，所谓上帝视角就是看看nvdimm总线驱动设备中都有哪些东西。

# 驱动和设备的对应关系

先来看看sysfs下，nvdimm驱动和设备之间的对应关系。

```
                                       /sys
                                        |
               +------------------------+------------------------------------------+
               |                                                                   |
              bus/                                                              devices/
               |                                                                   |
               |                                                                   .
               |                                                                   .
               |                                                                   .
               |                                                                   |
       --------+-------                                                       -----+-----
               |                                                                   |
              nd                                                                   |
        (nvdimm_bus_type)                                          +--------->  ndbus0/
               |                                                   |          (nvdimm_bus)
               |                                                   |               |
        -------+-------                                            |               |
               |                                                   |               |
            drivers/                                               |       +-------+------+
               |                                                   |       |              |
       +-------+----------------+--------------+-------------+     |       |              |
       |       |                |              |             |     |       |              |
       |    nd_pmem         nd_region          |         nd_bus ---+   +-> nmem0    +--> region0/
       |  (nd_pmem_driver) (nd_region_driver)  |     (nd_bus_driver)   |  (nvdimm)  |  (nd_region)
       |    nd_blk                             |                       |            |     |
       |  (nd_blk_driver)       |            nvdimm -------------------+            |     |
       |       |                |        (nvdimm_driver)                            |     |
       |       |	        |                                                   |     |
       |       |                +---------------------------------------------------+     |
       |       |                                                                          |
       |       |                                         +----------------+-------------+-+---------+
       |       |                                         |                |             |           |
       |       +-----------------------------------> namespace0.0         |             |           |
       |                                             (nd_namespace_blk)   |             |           |
       |                                             (nd_namespace_io)    |             |           |
       |                                             (nd_namespace_pmem)  |             |           |
       |                                                                  |             |           |
       dax_pmem   ------------------------------------------------------> dax0.0      btt0.0      pfn0.0
      (dax_pmem_driver)                                                   (nd_dax)    (nd_btt)    (nd_pfn)
```

暂时这个图还不全，后续会再加进来。

可以看到的是在nvdimm_bus_type下有多个驱动分别对应了设备目录下不同类型的nvdimm设备。
而设备这边各自并不是完全独立，而是形成了一颗设备树。

所以说这玩意还真有点复杂。

# 总体流程

最开始抓瞎的就是不知道初始化的流程是从哪里开始的，看到哪觉得都是起始点。终于找到的时候才有种柳暗花明的感觉。

首先需要说的是，nvdimm初始化的地方现在看有三个，也就是有三个不同的来源建立nvdimm。这里列出的是从acpi_nfit获取信息建立的过程。

```
acpi_nfit_init(struct acpi_nfit_desc *acpi_desc)
  nvdimm_bus_register()                            create nvdimm_bus
  add_table()
  nfit_mem_init()
  acpi_nfit_register_dimms()                       create nvdimm
  acpi_nfit_register_regions()                     create nd_region
```

现在看还是挺简单的。

大家可以看到的是在这个函数中创建了上一节设备树中的前三个设备：nvdimm_bus, nvdimm, nd_region。
而挂在nd_region下的其他设备则由其余的驱动来处理了。

对于从acpi_nfit中获取的nvdimm信息，有一个关键的数据结构acpi_nfit_desc。接着我们就围绕着这个数据结构来看看这些设备是怎么来的。
