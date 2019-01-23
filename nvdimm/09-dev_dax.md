当我们生成了nd_dax设备后，nvdimm驱动的任务就基本结束了。或者应该说，nvdimm驱动最后的任务是创建一个dev_dax设备，由这个设备接过最后的枪。

# 构造dev_dax的过程

从nd_dax构造dev_dax的过程在dax_pmem_probe中完成。

```
dax_pmem_probe()
  dax_pmem = devm_kzalloc(dev, sizeof(*dax_pmem), GFP_KERNEL);
  dax_region = alloc_dax_region(dev, region_id, &res,
    le32_to_cpu(pfn_sb->align), addr, PFN_DEV|PFN_MAP);
  dev_dax = devm_create_dev_dax(dax_region, id, &res, 1);
```

所以这里一共涉及了三个（四个）数据结构，我们来看看他们的样子和关系。

# dax_pmem, dax_region, dev_dax

```
                                                                                                dax_pmem                                          nd_dax
                                                                                                +----------------------------------------+        +--------------+
                                                                                                |dev                                     |        |nd_pfn        |
                                                                                                |     (struct device*)               ----|--------|--> dev       |
                                                                                                +----------------------------------------+        +--------------+             
                                                                                                |ref                                 <---|------------------------------------+
                                                                                                |     (struct percpu_ref)                |                                    |
                                                                                                |     +----------------------------------+                                    |
                                                                                                |     |count                             |                                    |
                                                                                                |     |  (atomic_long_t)                 |                                    |
                                                                                                |     |percpu_count_ptr                  |                                    |
                                                                                                |     |  (unsigned long)                 |                                    |
    dev_dax                                                                                     |     |release/confirm_switch            |                                    |
    +-------------------------------------+<----------------------------+                       |     |  (percpu_ref_func_t)             |                                    |
    |id                                   |                             |                       |     |force_atomit                      |                                    |
    |   (int)                             |                             |                       |     |                                  |                                    |
    |dev                                  |                             |                       |     |rcu                               |                                    |
    |   (struct device)                   |                             |                       |     |  (struct rcu_head)               |                                    |
    |   +---------------------------------+                             |                       +-----+----------------------------------+                                    |
    |   |devt                             |  dax_dev.inode->i_rdev      |                       |cmp                                     |                                    |
    |   |class                            |  dax_class                  |                       |     (struct completion)                |                                    |
    |   |groups                           |  dax_attribute_groups       |                       +----------------------------------------+                                    |
    |   |parent                           |  dax_region->dev = nd_pfn   |                       |pgmap                                   | setup in nvdimm_setup_pfn          |
    |   |                                 |                             |                       |     (struct dev_pagemap)               |                                    |
    |   +---------------------------------+                             |                       |     +----------------------------------+                                    |
    |                                     |                             |                       |     |dev                               |                                    |
    |num_resources                        |                             |                       |     |   (struct device*)               |                                    |
    |res[]                                |  copy from dax_region.res   |                       |     |ref                               | -----------------------------------+
    |   (struct resource)                 |                             |                       |     |   (struct percpu_ref*)           |
    |                                     |                             |                       |     |res                               |
    |dax_dev                              |                             |                       |     |   (struct resource)              |
    |   (struct dax_device*)              |                             |                       |     |   +------------------------------+
    |   +---------------------------------+                             |                       |     |   |start                         |  nsio->res.start + start_pad
    |   |private                      ----| ----------------------------+                       |     |   |end                           |  nsio->res.start + size - end_trunc
    |   |   (void*)                       |                                                     |     |   +------------------------------+
    |   |inode                            |                                                     |     |page_free                         |
    |   |   (struct inode)                |                                                     |     |   (dev_page_free_t)              |
    |   |   +-----------------------------+                                                     |     |kill                              | = dax_pmem_percpu_kill
    |   |   |i_cdev                       |                                                     |     |   ()                             |
    |   |   |   ops                       |  dax_fops                                           |     |                                  |
    |   |   |                             |                                                     |     |altmap                            |
    |   |   +-----------------------------+                                                     |     |   (struct vmem_altmap)           |
    |   |cdev                             |                                                     |     |   +------------------------------+
    |   |   (struct cdev)                 |                                                     |     |   |base_pfn                      | = PFN_SECTION_ALIGN_DOWN(nsio->res.start + start_pad)
    |   |                                 |                                                     |     |   |                              |
    |   |host                             |                                                     |     |   |reserve                       | = base_pfn + 8K
    |   |   (char*)                       |                                                     |     |   |                              |
    |   |ops                              |                                                     |     |   |   (const unsigned long)      |
    |   |   (struct dax_operations*)      |                                                     |     |   |free/align/alloc              |
    |   +---------------------------------+                                                     |     |   |   (unsigned long)            |
    |region                           ----|------> dax_region                                   |     |   +------------------------------+
    |   (struct dax_region*)              |        +---------------------------+                |     |                                  |
    +-------------------------------------+        |id                         |                +-----+----------------------------------+  
                                                   |   (int)                   |                                                      |
                                                   |dev                        |  = nd_pfn                                            |
                                                   |   (struct device*)        |                                                      |
                                                   |ida                        |                                                      |
                                                   |   (struct ida)            |                                                      |
                                                   |res                        |                                                      |
                                                   |   (struct resource)       |                                                      |
                                                   |   +-----------------------+                                                      |
                                                   |   |start                  |  nsio->res.start + start_pad + dataoff               |
                                                   |   |end                    |  nsio->res.start + size - end_trunc                  |
                                                   |   +-----------------------+                                                      |
                                                   |                           |                                                      |
                                                   |                           |                                                      |
                                                   |base                       |  devm_memremap_pages(nd_pfn, &dax_pmem->pgmap) <-----+
                                                   |   (void *)                |
                                                   |                           |
                                                   +---------------------------+  
```

这大概是我今年画的最庞大的一张图了，得缩小好几倍才能看到全貌。着重讲几点：

  * 最终的目标是生成dev_dax设备，该设备对应一个字符类型的文件/dev/dax0.0
  * /dev/dax0.0的fops是dax_fops，所以后面的mmap就靠它了
  * dev_dax生成的信息在dax_region中
  * dax_region中重要的信息有两个：res就是对其过后，去掉元数据的部分； base是根据pgmap做memremap后的返回

怎么样，这下是不是够爽？
