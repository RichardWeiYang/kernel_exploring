方才我们学习了史上最简单[可能是kbuild中最直接的小目标 – help][1]。这次我们来看看稍微高级那么一点点的目标 -- cscope。

客官可能要着急了，这个help和cscope都还不能算是什么真正的编译目标，讲这个我不爱听啊。嗯，出门右转，点击下一个链接就是你真的爱听的了。不过呢，东西有点多，恐怕你一下子接受不了。反正我第一次写的时候都有一种写到要吐的感觉。

你想我们平时要是很久没有运动，突然让你冲刺一百米是不是会头晕眼花腿抽经？如果在奔跑前能让你做个热身，充分让身体舒展开，你的感觉会不会好很多？所以我特意增加了这篇小进阶，希望能帮助你在进入高难度之前，给你做个脑力上的热身。

# 找到cscope目标

打开根目录下的Makefile文件，搜索cscope关键字。你找到了么？

```
tags TAGS cscope gtags: FORCE
	$(call cmd,tags)
```

看来kbuild把相应的这几个tag类目标都放在了一起。

不过后面这个$(call cmd,tags)是什么鬼？原来这是makefile中定义函数的一种方式。我们来看一下手册中是怎么讲的， [GNU make: call function][2]

```
The call function is unique in that it can be used to create new parameterized functions. You can write a complex expression as the value of a variable, then use call to expand it with different values.

The syntax of the call function is:

	$(call variable,param,param,…)

When make expands this function, it assigns each param to temporary variables $(1), $(2), etc.
```

知道了这个定义，再对照刚才的命令

> $(call cmd,tags)

意思就是有一个变量叫做cmd，需要在这里展开，而$(1)会被替换成tags。嗯，有点像宏定义，对不。

那我们现在要去找一个名字为cmd的变量咯～

# 初次遇见kbuild函数

c语言代码都是有一定的层次结构的，变量定义，函数声明都有各自的地方存放。比如定义要在源文件，而声明要在头文件。那宏定义呢？ 是不是也在头文件中定义的？

makefile中也有类似的用法 -- include。

在根Makefile中搜索include关键字，没几下就找到了这么一行

```
include scripts/Kbuild.include
```

是不是看着眼熟？

而在这个文件中就会发现那个叫cmd的变量了。

```
cmd = @$(echo-cmd) $(cmd_$(1))
```

我们把变量tags代入，就得到了

```
	@$(echo-cmd) $(cmd_tags)
```

先不管echo-cmd，先来看cmd_tags长什么样子。

细心的童鞋可能一开始就看到了，其实它就在刚才cscope规则的上方。

```
# Generate tags for editors
# ----------------------------------
quiet_cmd_tags = GEN     $@
      cmd_tags = $(CONFIG_SHELL) $(srctree)/scripts/tags.sh $@
```

这下明白了，原来cmd就好像一个函数，而这个函数的参数就是一个回调函数～

# tags.sh脚本文件

嗯，这个文件咱就不细讲了，毕竟和编译和kbuild的关系不大。总体来说就是传入什么参数，就执行相应的动作来生成需要的辅助文件。

```
case "$1" in
	"cscope")
		docscope
		;;

	"gtags")
		dogtags
		;;

	"tags")
		rm -f tags
		xtags ctags
		remove_structs=y
		;;

	"TAGS")
		rm -f TAGS
		xtags etags
		remove_structs=y
		;;
esac
```

对cscope目标来讲，就是执行docscope这个动作。

# cscope目标的层次结构

经过了一点小小挣扎，我们弄明白了cscope目标是如何通过kbuild系统生成的。怎么样，是不是和你想象的步骤略有不同？ 是不是有学到一些些kbuild的基本结构？

这里我们来回顾一下整个cscope目标生成的步骤，我把它叫做层级结构。

```
    Makefile             <--- scripts/Kbuild.include
    ---------------
    cscope: FORCE
    	$(call cmd,tags)

    Makefile
    ---------------
    cmd_tags
    	scripts/tags.sh $@
```

在Kbuild.include中定义了一些辅助函数，而整个kbuild系统都构建在这些辅助函数的基础上。这次我们看到的例子着实简单，看上去把规格在本地展开要更加清晰。不过随着内核复杂度增加，每次都本地展开会显得代码冗长且不易维护。

虽然这么些在理解上增加了一些难度，不过也经过了一些些努力就能水落石出。若能真的理解，就已经做好了kbuild系统探索的基本准备了。

# 一个小tip

阅读代码的时候我喜欢用cscope生成代码之间的索引，而我常用的方法就是make cscope。

但是通常重新生成一遍需要比较长的时间，后来我发现了一个加快生成速度的方法。

> 那就是跳过一些我并不想看的目录。

具体怎么做呢？好了，直接上代码：

```
diff --git a/scripts/tags.sh b/scripts/tags.sh
index 4fa070f9231a..5ac0873cfe4d 100755
--- a/scripts/tags.sh
+++ b/scripts/tags.sh
@@ -27,6 +27,7 @@ fi

 # ignore userspace tools
 ignore="$ignore ( -path ${tree}tools ) -prune -o"
+ignore="$ignore ( -path ${tree}drivers/gpu ) -prune -o"
+ignore="$ignore ( -path ${tree}drivers/net ) -prune -o"
+ignore="$ignore ( -path ${tree}drivers/media ) -prune -o"
+ignore="$ignore ( -path ${tree}drivers/scsi ) -prune -o"
+ignore="$ignore ( -path ${tree}drivers/staging ) -prune -o"
+ignore="$ignore ( -path ${tree}drivers/usb ) -prune -o"
+ignore="$ignore ( -path ${tree}drivers/infiniband ) -prune -o"

 # Detect if ALLSOURCE_ARCHS is set. If not, we assume SRCARCH
 if [ "${ALLSOURCE_ARCHS}" = "" ]; then
```

主要是去掉了几个大体积驱动的索引。整个制作索引的时间从117s下降到了57s，超过了50%。

希望能帮到你。

## v6.7 版本更新

最新6.7的版本又看了下，现在忽略生成cscope的方式改变了。现在是通过环境变量IGNORE_DIRS来指定需要忽略的路径。

比如
```
make IGNORE_DIRS="drivers tools" cscope
```

或者把变量定义到.bashrc中
```
export IGNORE_DIRS="drivers tools"
```

这样优化后，生成cscope文件的时间大大缩短。文件大小也从1.8M减小到436K。

[1]: /brief_tutorial_on_kbuild/03_first_target_help.md
[2]: https://www.gnu.org/software/make/manual/html_node/Call-Function.html
