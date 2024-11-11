有很多值得一看的材料，对理解内核和社区给了我很大的帮助。

其中非常重要的一个来源就是[lwn][1]

在这里我列一些，一方面是可以以后查找，另一方面是后续研究学习的方向。

  * [设计理念][2]
  * [内核开发者书单][6]
  * [git][3]
  * [调试][4]

还有些可以整理成专题的：

[内存相关][5]

PS: 这个lwn kernel index页面做的不太友好，找专题不一定能看清楚。写了个破脚本把主题先搂出来一遍。不是很准，凑合可以看。

```
wget --no-check-certificate https://lwn.net/Kernel/Index/
grep "href=\"#" index.html | grep "name=" | grep -Po 'href="\K[^"]*' > kernel_index
awk -F "-" '{print $1}' kernel_index | uniq > kernel_index.first
```

这样可以筛出来一级标题，虽然也挺多的。。。

忘了还有一个重量级的宝库[内核自带文档][7]。

[1]: https://lwn.net/Kernel/Index/
[2]: https://lwn.net/Kernel/Index/#Development_model-Patterns
[3]: https://lwn.net/Kernel/Index/#Development_tools-Git
[4]: https://lwn.net/Kernel/Index/#Development_tools-Kernel_debugging
[5]: /reference/01-mm.md
[6]: https://lwn.net/Kernel/Index/#Kernel_Hackers_Bookshelf
[7]: /reference/03-kernel_doc.md
