内核自带文档在源代码的Documents目录里。

这里面可以说是浩如烟海了。。。但是说实话程序员一般不喜欢写文档，很多内容估计没有及时维护已经过时了。不过就算是过时也是很好的学习资料。

# 文档制作

另外这里要说一下文档的制作。因为现在文档用了rst格式，所以直接用编辑器看不是很友好。最好制作一下文档。

这里说一下如何制作html格式的文档。

```
# make htmldocs
```

完成后文档就在Documentations/output目录下了。

不过这么制作的文档内容太多了，我们可以指定一下文档目录，限制一下制作范围。

比如只制作mm相关的文档，可以执行：

```
# make SPHINXDIRS=mm htmldocs
```

这样会快很多。
