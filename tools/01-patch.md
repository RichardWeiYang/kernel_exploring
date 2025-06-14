发patch是和社区交流的重要过程，这个过程也有相关的规范要遵守。比如检查代码格式和确认需要发给谁。

# 检查patch

内核代码有着自己的规范，在发patch前最好用脚本checkpatch.pl检查一下。

检查某个commit

```
./scripts/checkpatch.pl -g HEAD
```

检查某个patch文件

```
./scripts/checkpatch.pl /patch/to/patch/file
```

# 查要发给谁

写完patch后，我们需要发到社区。具体发到哪里，需要发给谁也是有讲究的。可以用get_maintainer.pl来查看。

```
./scripts/get_maintainer.pl /patch/to/patch/file
```

