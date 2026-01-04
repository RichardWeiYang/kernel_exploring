反向映射是一种知道物理页，要找到哪些进程映射了本物理页的方法。

```
Reverse map is one of the feature in linux kernel, which with given a physical
page, can return a list of PTEs which point to that page.
```

反向映射包括了"[匿名页反向映射][1]"和"文件页反向映射"，原因是匿名页没有后盾，而文件页有。

除了内核中是如何实现反向映射的，通常我们看到的是如何[使用反向映射][2].

[1]: /virtual_mm/01-anon_rmap_history.md
[2]: /virtual_mm/20-rmap_walk.md
