可能是之前有做过pci的关系，对总线不感到陌生。但是如果你头一次接触，可能不太理解总线的定义。

总的来说，总线是一个硬件架构的概念。举一个不恰当的类比，就好像我们马路上的车道：机动车，非机动车和人行道。不同的车、人需要在不同的地方行驶。同理，计算机设备也有这样的要求。不同的设备需要在指定的总线上才能正常运行。所以我们先把路认清，这样有了车就知道在哪里跑了。

# 数据结构

先从数据结构入手，看看总线对应的数据类型。在linux内核中总线用数据结构bus_type表示。

```
    bus_type
    +------------------------------------+
    |name                                |
    |dev_name                            |
    |     (char *)                       |
    +------------------------------------+
    |bus_groups                          |
    |dev_groups                          |
    |drv_groups                          |
    |     (struct attribute_group**)     |
    +------------------------------------+
    |match                               |
    |uevent                              |
    |probe                               |
    |remove                              |
    |shutdown                            |
    |                                    |
    +------------------------------------+
    |p                                   |
    |   (struct subsys_private*)         |
    |   +--------------------------------+
    |   |bus                             |
    |   |    (struct bus_type*)          |
    |   |subsys                          |
    |   |   +----------------------------+
    |   |   |kobj.name                   | = bus->name
    |   |   |kobj.kset                   | = bus_kset
    |   |   |kobj.ktype                  | = bus_ktype
    |   |   +----------------------------+
    |   |devices_kset                    |
    |   |drivers_kset                    |
    |   |    (struct kset)               |
    |   |klist_devices                   |
    |   |klist_drivers                   |
    |   |    (struct klist)              |
    |   |class                           |
    |   |    (struct class*)             |
    +---+--------------------------------+
```

简单解释一下部分成员的含义：

  * name/dev_name:        总线自己和下属设备的名字
  * bus/dev/drv_groups:   相关sysfs的文件
  * match/probe...:       匹配设备，操作设备的接口
  * p.subsys.kobj:        kobj/sysfs 树的节点
  * p.subsys.devices_kset:下属设备的kobj/sysfs节点
  * p.subsys.drivers_kset:下属驱动的kobj/sysfs节点

估计第一次看很多都不明白，不过不要紧因为重点内容基本都在这里了。

# 总线树

通过kobj的连接，总线在内核中形成一棵树。（其实严格来说，这棵树就一层。当然这是我的理解。）

这颗树可以通过ls /sys/bus查看。在这里画一个简图突出一下。

```
                         bus
                          |
                          |
     --+------------------+------------------+-----
       |                                     |     
       |                                     |     
      pci                                    nd        
       |                                     |     
       |                                     |     
       +- drivers                            +- drivers
       |                                     |     
       |                                     |     
       +- devices                            +- devices
```

这张图中我只列出了总线类型中的两种：pci, nd。所以各个总线之前是平级的。

而接下来可以看到，每个总线下面各自带了drivers, devices两目录。其中将会列出该总线上所属的驱动和设备。

所以从这点也可以看出总线是整个设备模型中的纽带了。
