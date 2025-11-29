在mmap系统调用中有一个参数flags。这个参数表达的含义有很多，但是我之前一直没有注意到的是，在这么多含义中可以分为两类：

  * 必选其一的
  * 可有可无的

其中必选其一的参数有三个，那就是MAP_SHARED/MAP_SHARED_VALIDATE/MAP_PRIVATE。

所以，对于一个vma空间，它要么是私有映射，要么是共享映射。

好了，我这感觉是说了依据废话。但是，看了这么多年的内核代码，我现在才发现这个里面藏着一个玄机。因为映射的类型决定了最后页面的类型是匿名页还是文件页。而不是根据MAP_ANONYMOUS来判断的。

  * MAP_SHARED: 背后用的是文件页
  * MAP_PRIVATE: 背后用的是匿名页

其中有一个例外: 当MAP_PRIVATE的文件区域第一次读取，且没有再写入的情况下，还是用的文件页。

后来仔细想想也很有道理。只要是共享映射的，我们就需要背后有一个东西做保存，让所有共享的人能够统一访问。而匿名页就无法做到这一点。对于私有映射的，反正就你自己用到这个部分内容，你就用匿名页就可以了。

PS： 这一点我们可以通过/proc/self/pagemap读取对应地址的内存属性来验证。


那接下来的问题是 vma_is_anonymous() 下的内存，都是匿名页么？


# vma_is_anonymous()

曾经一直一位匿名映射的区域，对应的内存一定是匿名的。现在看来这是个错误的理解。现在来看看这个判断到底是什么意思。

```
static inline bool vma_is_anonymous(struct vm_area_struct *vma)
{
	return !vma->vm_ops;
}
```

这么看就清楚了，凡是没有vm_ops的，就是匿名映射。那我们来看看mmap的时候是怎么处理的吧。

首先在vm_flags上，对共享映射都加上了VM_SHARED | VM_MAYSHARE标识。

接下来是不同情况的讨论，根据共享/私有和匿名/文件两个纬度，一共有四种组合：

  * anon shared
  * anon private
  * file shared
  * file private

## anon shared

因为设置了VM_SHARED标识，所以会调用shmem_zero_setup()来设置vma。

```
    vma->vm_file = file;
    vma->vm_ops = &shmem_anon_vm_ops;
```

最终内核会给共享匿名映射分配一个shmem文件和操作函数集shmem_anon_vm_ops。

这样，后续的操作就和文件保持一致了。

那这样的话，vma_is_anonymous()对于共享匿名映射，返回的是假。也就是不认为这是一个匿名映射。

## anon private

新分配的vma会被设置为vma_set_anonymous()，也就是将vm_ops设置为空。

## file shared/private

文件在mmap的时候都是通过__mmap_new_file_vma()来初始化的。相对匿名映射来说，要复杂一一些，我们展开看看。

```
__mmap_region(file, )
    call_mmap_prepare()
        vfs_mmap_prepare()                        // 找到对应的vm_ops
    __mmap_new_vma()
        __mmap_new_file_vma()
            vma->vm_file = get_file(file)
        vma_link_file()                           // 把vma加到file->f_mapping
    set_vma_user_defined_fields()
        vma->vm_ops = map->vm_ops;
```

这里主要设置了vma的两个成员：

  * vm_file
  * vm_ops

对应到文件系统上，我们举个例子：

  * vm_file->f_op:  就是ext4_file_operations
  * vm_ops:         就是ext4_file_vm_ops

好了，到这里其实并没有区分文件映射是共享还是私有的。我觉得这个问题要等到缺页中断的时候再处理。
