最近研究nvdimm，发现这玩意还真有点复杂。

简单记录之。

# 使用手册

写了一半发现得先写个使用手册，这样一来自己做个记录，二来也清楚正常使用流程，三来部分内容可以帮助解释代码。

所以加一个[使用手册][5]

# 上帝视角

经过了设备模型的洗礼，那就先从总线，驱动和设备角度看看都有些什么。

[上帝视角][1]

# nvdimm_bus

首先创建的是nvdimm_bus设备，而且从树形结构中可以看到它是nvdimm设备树的根。

[nvdimm_bus][2]

# nvdimm

在整个设备树中，有一个孤零零的存在:nvdimm。这就是是用来表示物理dimm设备的。

[nvdimm][3]

# nd_region

接着我们就来看nvdimm_bus下，另一个子树。而这颗子树的根就是nd_region了。

[nd_region][4]

而在nd_region下，有四个并列的设备：

  * [nd_namespace_X][6]
  * [btt][7]
  * [pfn][9]
  * [dax][8]

[1]: /nvdimm/01-a_big_picture.md
[2]: /nvdimm/02-nvdimm_bus.md
[3]: /nvdimm/03-nvdimm.md
[4]: /nvdimm/04-nd_region.md
[5]: /nvdimm/00-brief_user_guide.md
[6]: /nvdimm/05-namespace.md
[7]: /nvdimm/06-btt.md
[8]: /nvdimm/07-dax.md
[9]: /nvdimm/08-pfn.md
