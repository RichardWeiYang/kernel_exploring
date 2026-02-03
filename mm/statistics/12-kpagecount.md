/proc/kpagecount保存了每个pfn对应的mapcount。

# 代码

对应的内核处理函数在fs/proc/page.c，kpagecount_read()。

这个比较简单，就是取了mapcount返回到用户态。

PS: 做了个实验，这个接口返回的不是folio_mapcount()，而是单个page的映射次数。即便这是一个subpage，也只会算当前subpage的，而不是整个folio被映射的次数。所以对验证folio在从PMD map到PTE map过程中的mapcount变化拿不到想要的结果。

# 格式

就是个引用计数的数字。

# 使用

恩。。。selftests里目前没有人用。让我在自己的测试用例里，加一个来验证以下大页在拆分前后的mapcount变化。
