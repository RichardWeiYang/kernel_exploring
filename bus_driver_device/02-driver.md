接下来讲讲设备模型中的驱动，和之前套路一样，先看看驱动的数据结构。

# 数据结构

内核中用结构体device_driver来表示一个驱动。不过大家通常在驱动程序中看到是另外的结构体，比如pci驱动用的是pci_driver。但是我们打开这个pci_driver就可以看到其中包含了device_driver。所以驱动的核心数据结构还是device_driver。

```
    device_driver
    +----------------------------------+
    |name                              |
    |    (char *)                      |
    +----------------------------------+
    |bus                               |
    |    (struct bus_type*)            |
    +----------------------------------+
    |owner                             |
    |    (struct module*)              |
    +----------------------------------+
    |probe                             |
    |remove                            |
    |shutdown                          |
    |suspend                           |
    |resume                            |
    |                                  |
    +----------------------------------+
    |p                                 |
    |    (struct driver_private*)      |
    |    +-----------------------------+
    |    |driver                       |
    |    |    (struct device_driver*)  |
    |    |kobj                         |
    |    |    +------------------------+
    |    |    |name                    | = device_driver->name
    |    |    |kset                    | = device_driver->bus->p->drivers_kset
    |    |    |ktype                   | = driver_ktype
    |    |    +------------------------+
    |    |klist_devices                |
    |    |                             |
    +----+-----------------------------+
```

简单解释一下部分成员的含义：

  * name:            驱动名
  * bus:             对应的总线
  * probe/remove...: 驱动调用接口
  * p->kobj.name:    kobj对应的名字
  * p->kobj.kset:    kobj的父节点

暂时没有更多可以解释的，想强调的一点是p->kobj.kset设置成了对应总线的drivers_kset。这点表明了驱动在sysfs中的根是在对应的总线上。当我们看设备的时候，会发现两者之间有所不同。

针对驱动的图解将在下一节设备中给出。
