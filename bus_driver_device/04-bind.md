在内核中驱动和设备发生关联称之为绑定bind。

而且有意思的是绑定的过程可以在两个地方发生。

  * 驱动加载
  * 设备发现

不过这两个地方最后都会通过函数__driver_attach()来处理。而这个过程又可以分成两步：

  * 匹配
  * 探测

接着我们就按照这两个步骤展开来看。

# 匹配

匹配的过程由总线的match函数来处理。

```
static inline int driver_match_device(struct device_driver *drv,
				      struct device *dev)
{
	return drv->bus->match ? drv->bus->match(dev, drv) : 1;
}
```

那具体的操作就由总线来决定。比如pci总线上的match函数就是pci_match_device()。它的过程就是匹配pci的厂商号和设备号。

# 探测

找到了匹配的设备和驱动后，就可以执行探测了。这个过程相对比较复杂，在当前的内核中，大多最后由总线的probe函数执行。

比如pci总线上的probe函数就是pci_device_probe()。它的作用就是去调用pci_driver中的probe函数。

# 流程图

如果我们简单画出绑定过程的流程，大致如下：

```
  __driver_attach
      driver_match_device()
        bus->match()
      driver_probe_device()
        bus->probe()
```

从这点可以看出，总线的重要性。
