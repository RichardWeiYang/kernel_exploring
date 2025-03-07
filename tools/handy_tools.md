内核中有些好用的工具，在发patch、检查变化的时候可以用到。

# 发patch

## 检查patch

内核代码有着自己的规范，在发patch前最好用脚本checkpatch.pl检查一下。

检查某个commit

```
./scripts/checkpatch.pl -g HEAD
```

检查某个patch文件

```
./scripts/checkpatch.pl /patch/to/patch/file
```

## 查要发给谁

写完patch后，我们需要发到社区。具体发到哪里，需要发给谁也是有讲究的。可以用get_maintainer.pl来查看。

```
./scripts/get_maintainer.pl /patch/to/patch/file
```

# 代码效果

## obj文件变化

有时候我们该了代码，比如清理后，代码的大小会发生变化。这时想要展示这部分变化可以用bloat-o-meter。

```
./scripts/bloat-o-meter file1.o file2.o
```

通过这个，可以看到不同symbole在代码变化前后有没有区别，区别是多少。
