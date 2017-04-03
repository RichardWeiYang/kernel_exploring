第一次总是让人激动的，但是也常常伴随着恐惧和害怕。我相信有不少小白希
望学习内核，但是常常连第一步都没有开始，就直接栽倒在了出发的地方。

这篇文章献给那些向往成为内核大牛的小白们，由我来帮你们破除进入内核世界的第一道障碍。

# 准备环境

编译内核之前有一些基本的条件

* 有一台可以联网的机器（或者虚拟机）
* 安装了linux系统
* 怎么着也得会一点基本的命令操作

除此之外对linux系统还要求一些软件包的安装（可能不全，在编译过程中遇到提示可以使用google搜索是缺了哪个包）

* git   一个软件版本管理工具，我们用它来获得内核源码
* gcc  编译器
* make 编译工具
* libncurse-dev 一个图形库
* openssl-dev   加密库

嗯，差不多了，开始动手吧。

# 获取内核

感谢Linus，感谢git，自从有了git，获取内核代码变得异常的方便，而且时刻都可以是最新的。

在终端输入以下命令即可：

> git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git

没有安装git软件的请重复上一个步骤

# 基本配置

和大部分开源软件类似，linux kernel也是要配置之后才能够编译的。配置方法有好几种，我比较偏爱的是 make menuconfig。

```bash
cd linux
make menuconfig
```

执行完，你可以看到如下的配置界面。

![这里写图片描述](http://img.blog.csdn.net/20170222105426439?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvUmljaGFyZFlTdGV2ZW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

知道为啥我喜欢用这个了吧，因为这有个图形界面。虽然有好多看不懂，但是毕竟有些你是能猜出个大概意思的。

这次只是用来编译第一个内核，所以不需要什么配置，直接按右箭头，走到Exit退出保存即可。

## 可能会遇到的问题

在执行make menuconfig的时候，可能会遇到一些提示说某些包没有安装导致执行失败。大家不要慌，究其原因是因为配置的过程实际上是内核先编译了一个用户态的配置工具，这个过程就需要依赖的软件包有： make, gcc, ld 和图形库libncurse-dev。不用紧张，按照提示，缺什么软件就安装什么软件就好了。

# 开始编译

配置完了之后就可以编译了。

很简单，运行如下命令

```
make -j8
```

如果编译成功，你就会看到目录下有一个文件叫vmlinux。
恭喜～

# 安装内核

也是so easy

```
make modules_install
make install
```

注意： 这两步需要有管理员权限。

另外需要注意的是，安装后有些版本可能要调整引导程序的配置。比如在ubuntu上，配置文件在/etc/default/grub， /boot/grub/grub.cfg。否则有时候下一次重启还是使用旧的内核。

# 重启机器

执行如下命令

```
reboot
```

好了，等下次机器起来，那就是一个崭新的世界了。

