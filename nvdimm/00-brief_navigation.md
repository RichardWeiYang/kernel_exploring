最近研究nvdimm，发现这玩意还真有点复杂。

简单记录之。

# 上帝视角

经过了设备模型的洗礼，那就先从总线，驱动和设备角度看看都有些什么。

[上帝视角][1]

# nvdimm_bus

首先创建的是nvdimm_bus设备，而且从树形结构中可以看到它是nvdimm设备树的根。

[nvdimm_bus][2]

[1]: /nvdimm/01-a_big_picture.md
[2]: /nvdimm/02-nvdimm_bus.md
