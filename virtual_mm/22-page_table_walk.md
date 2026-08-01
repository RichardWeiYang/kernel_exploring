之前有一次工作是要打印进程内某一段地址对应的页表项，我自己折腾了半天。结果发现内核中有现成的框架，记录一下已备不时之需。

# walk_page_range

遍历页表的框架有很多变种，这里研究一个常用的 

walk_page_range(mm, start, end, ops, private)

意思就是遍历由mm指定的进程中[start, end]虚拟地址空间，对应的操作由ops决定。

ops是struct mm_walk_ops，这个结构掌握了遍历过程中需要执行的操作。也就是遇到了pgd/pud/pmd时要做些什么，具体的可以看mm_walk_ops的注释，很详细了。

```
walk_page_range(mm, start, end, ops, private)
    check_ops_safe(ops)
    walk_page_range_mm_unsafe(mm, start, end, ops, private)
        walk.ops = ops
        walk.mm = mm

        // 确认相关的锁已经拿到
        process_mm_walk_lock(mm, ops->walk_lock)

        // 找到对应的vma， 并划定本次遍历的范围[start, next)
        vma = find_vma(mm, start)
        process_vma_walk_lock(vma, ops->walk_lock)
        walk.vma = vma
        next = min(end, vma->vm_end)

        __walk_page_range(start, next, &walk)
            walk_pgd_range(start, next, walk)                 // 开始遍历页表
```

# walk_pgd_range

```
walk_pgd_range(addr, end, walk)
    pgd = pgd_offset(walk->mm, addr)
    next = pgd_addr_end(addr, end)                            // 本次遍历范围[addr, next)

    walk_p4d_range(pgd, addr, next, walk)
        p4d = p4d_offset(pgd, addr)
        next = p4d_addr_end(addr, end)                        // 本次遍历范围[addr, next)

	walk_pud_range(p4d, addr, next, walk)
            pud = pud_offset(p4d, addr)
            next = pud_addr_end(addr, end)                    // 本次遍历范围[addr, next)

            walk->action = ACTION_SUBTREE
            ops->pud_entry(pud, addr, next, walk)
            if (walk->acion == ACTION_AGAIN)
            if (walk->acion == ACTION_CONTINUE)


            walk_pmd_range(pud, addr, next, walk)
                pmd = pmd_offset(pud, addr)
                next = pmd_addr_end(addr, end)

                walk->action = ACTION_SUBTREE
                ops->pmd_entry(pmd, addr, next, walk)
                if (walk->acion == ACTION_AGAIN)
                if (walk->acion == ACTION_CONTINUE)
```
# 应用

访问文件/proc/self/numa_maps时就会调用到 walk_pgd_range()。

