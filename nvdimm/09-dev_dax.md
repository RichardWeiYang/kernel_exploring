当我们生成了nd_dax设备后，nvdimm驱动的任务就基本结束了。或者应该说，nvdimm驱动最后的任务是创建一个dev_dax设备，由这个设备接过最后的枪。

# 构造dev_dax的过程

从nd_dax构造dev_dax的过程在dax_pmem_probe中完成。

```
dax_pmem_probe()
  nvdimm_setup_pfn(nd_pfn, &pgmap);
  dax_region = alloc_dax_region(dev, region_id, &res,
    le32_to_cpu(pfn_sb->align), addr, PFN_DEV|PFN_MAP);
  dev_dax = __devm_create_dev_dax(dax_region, id, &pgmap, subsys);
```

所以这里一共涉及了三个数据结构，我们来看看他们的样子和关系。

# dev_dax, dax_dev, dax_region

```
    dev_dax
    +-------------------------------------+<----------------------------+
    |id                                   |                             |
    |   (int)                             |                             |
    |dev                                  |                             |
    |   (struct device)                   |                             |
    |   +---------------------------------+                             |
    |   |devt                             |  dax_dev.inode->i_rdev      |
    |   |class                            |  dax_class                  |
    |   |/ bus                            |  dax_bus_type               |
    |   |groups                           |  dax_attribute_groups       |
    |   |parent                           |  dax_region->dev = nd_pfn   |
    |   |                                 |                             |
    |   |kobj->name                       |  = "dax0.0"                 |
    |   +---------------------------------+                             |
    |                                     |                             |
    |dax_dev                              |                             |
    |   (struct dax_device*)              |                             |
    |   +---------------------------------+                             |
    |   |private                      ----| ----------------------------+
    |   |   (void*)                       |
    |   |inode                            |      
    |   |   (struct inode)                |      
    |   |   +-----------------------------+      
    |   |   |i_cdev                   ----|-----+
    |   |   |   (struct cdev*)            |     |
    |   |   +-----------------------------+     |
    |   |cdev                             |<----+
    |   |   (struct cdev)                 |  /dev/dax0.0
    |   |   +-----------------------------+
    |   |   |ops                          |  dax_fops
    |   |   |                             |
    |   |   +-----------------------------+
    |   |                                 |
    |   |host                             |
    |   |   (char*)                       |
    |   |ops                              |
    |   |   (struct dax_operations*)      |
    |   +---------------------------------+
    |                                     |
    |region                               |
    |   (struct dax_region*)              |
    |   +---------------------------------+
    |   |id                               |
    |   |   (int)                         |
    |   |dev                              | = nd_pfn.dev
    |   |   (struct device*)              |
    |   |ida                              |
    |   |   (struct ida)                  |
    |   |res                              |
    |   |   (struct resource)             |
    |   |   +-----------------------------+
    |   |   |start                        | nsio->res.start + start_pad + dataoff
    |   |   |end                          | nsio->res.start + size - end_trunc
    |   |   +-----------------------------+
    |   |                                 |
    +---+---------------------------------+
    |ref                              <---|------------------------------------+
    |     (struct percpu_ref)             |                                    |
    |     +-------------------------------+                                    |
    |     |count                          |                                    |
    |     |  (atomic_long_t)              |                                    |
    |     |percpu_count_ptr               |                                    |
    |     |  (unsigned long)              |                                    |
    |     |release                        | = dev_dax_percpu_release           |
    |     |confirm_switch                 |                                    |
    |     |  (percpu_ref_func_t)          |                                    |
    |     |force_atomit                   |                                    |
    |     |                               |                                    |
    |     |rcu                            |                                    |
    |     |  (struct rcu_head)            |                                    |
    +-----+-------------------------------+                                    |
    |pgmap                                | setup in nvdimm_setup_pfn          |
    |     (struct dev_pagemap)            |                                    |
    |     +-------------------------------+                                    |
    |     |dev                            | = nd_pfn.dev                       |
    |     |   (struct device*)            |                                    |
    |     |ref                            | -----------------------------------+
    |     |   (struct percpu_ref*)        |
    |     |res                            |
    |     |   (struct resource)           |
    |     |   +---------------------------+
    |     |   |start                      |  nsio->res.start + start_pad
    |     |   |end                        |  nsio->res.start + size - end_trunc
    |     |   +---------------------------+
    |     |page_free                      |
    |     |   (dev_page_free_t)           |
    |     |kill                           | = dax_pmem_percpu_kill
    |     |   ()                          |
    |     |                               |
    |     |altmap                         |
    |     |   (struct vmem_altmap)        |
    |     |   +---------------------------+
    |     |   |base_pfn                   | = PFN_SECTION_ALIGN_DOWN(nsio->res.start + start_pad)
    |     |   |reserve                    | = PHYS_PFN(SZ_8K)
    |     |   |   (const unsigned long)   |
    |     |   |free                       | = PHYS_PFN(offset - SZ_8K);
    |     |   |align                      |
    |     |   |alloc                      |
    |     |   |   (unsigned long)         |
    |     |   +---------------------------+
    |     |                               |
    +-----+-------------------------------+
```

着重讲几点：

  * 最终的目标是生成dev_dax设备，该设备对应一个字符类型的文件/dev/dax0.0
  * /dev/dax0.0的fops是dax_fops，所以后面的mmap就靠它了
  * dev_dax生成的信息在dax_region中
  * dax_region中res就是对齐过后，去掉元数据的部分； 

怎么样，这下是不是够爽？
