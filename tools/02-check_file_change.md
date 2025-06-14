# obj文件变化

有时候我们该了代码，比如清理后，代码的大小会发生变化。这时想要展示这部分变化可以用bloat-o-meter。

```
./scripts/bloat-o-meter file1.o file2.o
```

通过这个，可以看到不同symbole在代码变化前后有没有区别，区别是多少。
