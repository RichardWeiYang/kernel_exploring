[DRGN][1]是一个相对较新的查看内核状态的工具。它是Meta开发的，作为crash的一个替代工具。

# 安装

参考最新的[安装文档][2]。

有几种安装的方式，感觉通过pip来安装，方式比较统一。

```
sudo pip3 install drgn
```

如果能运行下面的命令，看上去应该是安装好了。

```
python3 -m drgn --help
```

# 内核调试信息

在使用drgn调试内核前，先要保证对应的调试信息在。如果在运行drgn时看到没有调试信息的提示，可以参考[Getting Debug Symbols][3]来处理。

其中包括了自己编译的内核，和使用发行版自带的内核两种情况。

[1]: https://drgn.readthedocs.io/en/latest/
[2]: https://drgn.readthedocs.io/en/latest/installation.html
[3]: https://drgn.readthedocs.io/en/latest/getting_debugging_symbols.html
