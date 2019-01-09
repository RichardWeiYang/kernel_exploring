最后来看看设备。

# 数据结构

内核中用结构体device来表示一个设备。不过大家通常在驱动程序中看到是另外的结构体，比如pci设备用的是pci_dev。但是我们打开这个pci_dev就可以看到其中包含了device。所以设备的核心数据结构还是device。

```
    device
    +----------------------------------------+
    |init_name                               |
    |    (char *)                            |
    |devt                                    |
    |    (dev_t)                             |
    |id                                      |
    |    (u32)                               |
    +----------------------------------------+
    |kobj                                    |
    |    (struct kobject)                    |
    |    +-----------------------------------+
    |    |name                               | = device->init_name or
    |    |                                   |   device->bus->dev_name + id
    |    |kset                               | = devices_kset
    |    |ktype                              | = device_ktype
    +----+-----------------------------------+
    |type                                    |
    |    (struct device_type*)               |
    +----------------------------------------+
    |bus                                     | = [pci_bus_type|nvdimm_bus_type]
    |    (struct bus_type)                   |
    +----------------------------------------+
    |driver                                  |
    |    (struct device_driver*)             |
    +----------------------------------------+
    |p                                       |
    |    (struct device_private *)           |
    +----------------------------------------+
```

简单解释一下部分成员的含义：

  * init_name:       设备名（如果有的话）
  * id:              设备号
  * bus:             对应的总线
  * driver:          对应的驱动
  * kobj.name:       kobj对应的名字，真正的设备名
  * kobj.kset:       kobj的父节点

这里着重强调一点kobj.kset，这个值和驱动的父节点有所不同。驱动的父节点指向了总线，而设备的父节点是另一个根节点。这个时候我们可以来看看有了驱动和设备后，kobj树形结构的样子了。

# 总线，驱动和设备的树

```
           bus(bus_kset)      <-------------+     devices(devices_kset)
                |                           |            |
  --+-----------+------+-----               |   ----+----+-------------
    |                  |                    |       |
    +- drivers         +- devices           |       |
       |                  |                 |       |
       +- drvA   <- - -   +- devA   - * - * |  >    +- devA
       |                  |  |              |       |  
       |                  |  +- subsystem --+       |  
       |                  |                 |       |  
       +- drvB   <- - -   +- devB   - * - * |  >    +- devB
       |                  |  |              |       |  
       |                  |  +- subsystem --+       |  
       |                  |                 |       |  
       +- drvC   <- - -   +- devC   - * - * |  >    +- devC
       |                  |  |              |       |  
       |                  |  +- subsystem --+       |  
       |                  |                         |  
       +- drvD   <- - -   +- devD   - * - * -  >    +- devD
```

从这张图我们可以看到

  * 总线下有驱动和设备
  * 设备和驱动存在着关联
  * 总线下的设备实际是一个软链接
  * 设备真正的根在devices_kset
