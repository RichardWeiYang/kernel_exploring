[DRGN][1]是一个相对较新的查看内核状态的工具。它是Meta开发的，作为crash的一个替代工具。

# 准备环境

## 安装

参考最新的[安装文档][2]。

有几种安装的方式，感觉通过pip来安装，方式比较统一。

```
sudo pip3 install drgn
```

如果能运行下面的命令，看上去应该是安装好了。

```
python3 -m drgn --help
```

## 内核调试信息

在使用drgn调试内核前，先要保证对应的调试信息在。如果在运行drgn时看到没有调试信息的提示，可以参考[Getting Debug Symbols][3]来处理。

其中包括了自己编译的内核，和使用发行版自带的内核两种情况。

## 内核调试helper

为了方便调试内核，drgn项目已经给我们提供了不少helper来帮助我们。

[Helpers][4]

其中有专门一节是讲述内核数据结构相关的helper。

# 使用

## 简单的例子

下面是一个非常简单的例子，当我们准备好环境后，可以通过drgn来查看某个进程的命令行名字。

```
drgn 0.0.31 (using Python 3.10.12, elfutils 0.192, with debuginfod, with libkdumpfile)
For help, type help(drgn).
>>> import drgn
>>> from drgn import FaultError, NULL, Object, alignof, cast, container_of, execscript, implicit_convert, offsetof, reinterpret, sizeof, stack_trace
>>> from drgn.helpers.common import *
>>> from drgn.helpers.linux import *
>>> task = find_task(6977)
>>> print(task.comm)
(char [16])"a.out"
```

实际上，我已经知道了这个程序的pid，所以说这个例子就是个例子。。。


[1]: https://drgn.readthedocs.io/en/latest/
[2]: https://drgn.readthedocs.io/en/latest/installation.html
[3]: https://drgn.readthedocs.io/en/latest/getting_debugging_symbols.html
[4]: https://drgn.readthedocs.io/en/stable/helpers.html
