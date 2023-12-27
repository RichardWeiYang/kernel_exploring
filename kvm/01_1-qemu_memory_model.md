# Qemu中的内存模型

相信看过源代码的朋友会有这样的体会Qemu中有着一个非常庞（奇）大（葩）的内存模型。对于头一次看
代码的小伙伴估计很容易Lost，因为会有很多相互关联有长得很像的小伙伴们

* AddressSpace
* MemoryRegion/MemoryRegionSection
* FlatView/FlatRange
* RAMBlock

反正第一次看得我几乎要吐了。接下来我们就一点点揭开这个庞然大物吧。

# 树形和平面的内存模型

## AddressSpace

AddressSpace是Qemu内存模型最顶层的数据结构，其他的概念都通过它链接。

整个虚拟机中也不只有一个AddressSpace，通过GDB的调试，我们可以看到完整的虚拟机中有如下这些：

```
(gdb) dump_address_spaces 0
AddressSpace : memory(0x5555566ed920)
     Root MR : 0x555556804b00
    FlatView : 0x555557a2e650
AddressSpace : I/O(0x5555566ed8c0)
     Root MR : 0x5555567fbf00
    FlatView : 0x555557a47d30
AddressSpace : cpu-memory-0(0x555556877460)
     Root MR : 0x555556804b00
    FlatView : 0x555557a2e650
AddressSpace : cpu-smm-0(0x555556877640)
     Root MR : 0x5555567e0400
    FlatView : 0x555557a3b800
AddressSpace : i440FX(0x5555569de1a0)
     Root MR : 0x5555569de200
    FlatView : 0x5555568301e0
AddressSpace : PIIX3(0x555557177830)
     Root MR : 0x555557177890
    FlatView : 0x5555568301e0
AddressSpace : VGA(0x555557257c30)
     Root MR : 0x555557257c90
    FlatView : 0x5555568301e0
AddressSpace : e1000(0x7ffff7e08220)
     Root MR : 0x7ffff7e08280
    FlatView : 0x5555568301e0
AddressSpace : piix3-ide(0x5555576e3840)
     Root MR : 0x5555576e38a0
    FlatView : 0x5555568301e0
AddressSpace : PIIX4_PM(0x55555781e7b0)
     Root MR : 0x55555781e810
    FlatView : 0x5555568301e0
```

具体其中的关联和区别，小编现在还不清楚。待我慢慢深究，这里先放下不表。

## MemoryRegion

顾名思义，这是用来表达一段内存区域的。其中重要的两个成员就是：addr和size，表示了这段内存区域
对应的起始地址和大小。

从内存模型的角度出发，MemoryRegion重要的特征是**形成了一棵内存区域树**。

来一个简单的图示意一下：

```
                            struct MemoryRegion
                            +------------------------+                                         
                            |name                    |                                         
                            |  (const char *)        |                                         
                            +------------------------+                                         
                            |addr                    |                                         
                            |  (hwaddr)              |                                         
                            |size                    |                                         
                            |  (Int128)              |                                         
                            +------------------------+                                         
                            |subregions              |                                         
                            |    QTAILQ_HEAD()       |                                         
                            +------------------------+                                         
                                       |
                                       |
               ----+-------------------+---------------------+----
                   |                                         |
                   |                                         |
                   |                                         |

     struct MemoryRegion                            struct MemoryRegion
     +------------------------+                     +------------------------+
     |name                    |                     |name                    |
     |  (const char *)        |                     |  (const char *)        |
     +------------------------+                     +------------------------+
     |addr                    |                     |addr                    |
     |  (hwaddr)              |                     |  (hwaddr)              |
     |size                    |                     |size                    |
     |  (Int128)              |                     |  (Int128)              |
     +------------------------+                     +------------------------+
     |subregions              |                     |subregions              |
     |    QTAILQ_HEAD()       |                     |    QTAILQ_HEAD()       |
     +------------------------+                     +------------------------+
```

> 耳听为虚，眼见为实

那我们来看看一个MemoryRegion的树形结构会是什么样子的。

```
(gdb) dump_memory_region 0x555556804b00
Dump MemoryRegion:system
[0000000000000000-100000000ffffffff]:system
  [00000000fee00000-000000000feefffff]:apic-msi
  [00000000000ec000-000000000000effff]:pam-ram
  [00000000000ec000-000000000000effff]:pam-pci
  [00000000000ec000-000000000000effff]:pam-rom
  [00000000000a0000-000000000000bffff]:smram-region
  [00000000fed00000-000000000fed003ff]:hpet
  [00000000fec00000-000000000fec00fff]:ioapic
  [0000000000000000-00000000007ffffff]:ram-below-4g
  [0000000000000000-100000000ffffffff]:pci
    [00000000000a0000-000000000000bffff]:vga-lowmem
    [00000000000c0000-000000000000dffff]:pc.rom
    [00000000000e0000-000000000000fffff]:isa-bios
    [00000000fffc0000-000000000ffffffff]:pc.bios
```

怎么样，小编没有骗你吧。

## FlatView/FlatRange

再来一次顾名思义，FlatView就是平面视图。那是啥的平面视图呢？我就知道你聪明，不用猜就知道。
是MemoryRegion的平面视图。刚才咱不是看了么，MemoryRegion形成了一棵高大雄伟的树，但是
要用的时候还是得铺平了看起来舒服。

和刚才一样，我们也来瞅一眼这个数据结构的样子。


```
FlatView (An array of FlatRange)
+----------------------+
|nr                    |
|nr_allocated          |
|   (unsigned)         |         FlatRange             FlatRange
+----------------------+         
|ranges                | ------> +---------------------+---------------------+
|   (FlatRange *)      |         |offset_in_region     |offset_in_region     |
+----------------------+         |                     |                     |
                                 +---------------------+---------------------+
                                 |addr(AddrRange)      |addr(AddrRange)      |
                                 |    +----------------|    +----------------+
                                 |    |start (Int128)  |    |start (Int128)  |
                                 |    |size  (Int128)  |    |size  (Int128)  |
                                 +----+----------------+----+----------------+
                                 |mr                   |mr                   |
                                 | (MemoryRegion *)    | (MemoryRegion *)    |
                                 +---------------------+---------------------+
```

FlatRange是FlatView的一个成员，这个成员是一个一维数组，每个项目代表了平面视图上的一段内存空间。

```
(gdb) dump_flatview 0x555557a2e650
[00000000000000000-000000000000a0000], offset_in_region 0000000000000000
[000000000000a0000-000000000000c0000], offset_in_region 0000000000000000
[000000000000c0000-000000000000e0000], offset_in_region 0000000000000000
[000000000000e0000-00000000000100000], offset_in_region 0000000000020000
[00000000000100000-00000000008000000], offset_in_region 0000000000100000
[000000000fec00000-000000000fec01000], offset_in_region 0000000000000000
[000000000fed00000-000000000fed00400], offset_in_region 0000000000000000
[000000000fee00000-000000000fef00000], offset_in_region 0000000000000000
[000000000fffc0000-00000000000000000], offset_in_region 0000000000000000
```

嗯，眼花了，大家凑合看一下～

## 大串联

看过了这么几个鼎鼎有名的数据结构，接下来我们来看看这几者之间的关联。这样你就能有一个比较全局的
视野了。

```
AddressSpace               
+-------------------------+
|name                     |
|   (char *)              |          FlatView (An array of FlatRange)
+-------------------------+          +----------------------+
|current_map              | -------->|nr                    |
|   (FlatView *)          |          |nr_allocated          |
+-------------------------+          |   (unsigned)         |         FlatRange             FlatRange
|                         |          +----------------------+         
|                         |          |ranges                | ------> +---------------------+---------------------+
|                         |          |   (FlatRange *)      |         |offset_in_region     |offset_in_region     |
|                         |          +----------------------+         |                     |                     |
|                         |                                           +---------------------+---------------------+
|                         |                                           |addr(AddrRange)      |addr(AddrRange)      |
|                         |                                           |    +----------------|    +----------------+
|                         |                                           |    |start (Int128)  |    |start (Int128)  |
|                         |                                           |    |size  (Int128)  |    |size  (Int128)  |
|                         |                                           +----+----------------+----+----------------+
|                         |                                           |mr                   |mr                   |
|                         |                                           | (MemoryRegion *)    | (MemoryRegion *)    |
|                         |                                           +---------------------+---------------------+
|                         |
|                         |
|                         |
|                         |          MemoryRegion(system_memory/system_io)
+-------------------------+          +----------------------+
|root                     |          |                      | root of a MemoryRegion
|   (MemoryRegion *)      | -------->|                      | tree
+-------------------------+          +----------------------+
```

丑是丑了点，但是中心思想还是比较清晰的。

> 每个AddressSpace关联着一棵MemoryRegion树，而随着这棵树的变化对应着一个内存空间的平面视图FlatView.

这样是不是知道这么一大坨玩的是什么花样了？

# 瞅一眼代码逻辑

上来看了这么动（呆）人（板）的图片，接着来电代码接接地气。

```
memory_region_add_subregion()  // 添加subregion
    ...
    memory_region_transaction_commit()
        ...
        flatviews_reset()      // 重新生成FlatView
        address_space_set_flatview()
            ...
            address_space_update_topology_pass()
                ...
                MEMORY_LISTENER_UPDATE_REGION(region_add/del)  // 调用相应的region_add/del
```

在实际的编程过程中，直接操作的是MemoryRegion这棵树，而其对应的FlatView将会根据需要生成。

在理解了这些内容之后，就可以来看看之前没有提及的内容了。也就是上述代码片段中最后一行，在这一行中
我们将会看到Qemu的内存模型是如何逐渐和KVM联系在一起的。

# 握手KVM

当内存树发生变化，Qemu就会对相关的FlatView调用相应的回调函数，而Qemu的内存模型就是在这个过程中
和KVM发生联系的。

在Qemu初始化的过程中，通过kvm_memory_listener_register()注册了相应的回调函数kvm_region_add/del。
所以这个对应的回调函数就是kvm_region_add/del。

具体的细节就不在这里深究了，我们来看一下究竟Qemu告诉了KVM什么。

```
struct kvm_userspace_memory_region {
	__u32 slot;
	__u32 flags;
	__u64 guest_phys_addr;
	__u64 memory_size; /* bytes */
	__u64 userspace_addr; /* start of the userspace allocated memory */
};
```

Qemu和KVM之间的通信格式就是上面的这段数据结构。很清楚，我们看到他们交换了这么几个重要的信息

* GPA: guest_phys_addr
* HVA: userspace_addr

好了，你懂了。

# NOTE

本节使用到的GDB脚本可以在[gdb-script][1]中获得。

[1]: https://github.com/RichardWeiYang/qemu/blob/mytest/gdb-script
