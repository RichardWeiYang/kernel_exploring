内核启动的过程不是一蹴而就的，而且其中还包含了不少让人不太好找的小秘密。

每次我看到相关的代码，都要从头再找一遍，感觉非常费事。这次干脆就记录下来，以备后用。

# INIT_CALL

在内核代码中经常会看到core_initcall(), subsys_initcall()这样xxx_initcall()的函数。

这些函数可以理解为c++中的构造函数，只是内核对这些函数做了分类，并且在特定的地方调用他们。

[INIT_CALLS的秘密][1]

# start_kernel之前

一般认为start_kernel是整个内核代码开始的地方。但你有没有好奇过在这个之前都发生了什么。

就好像应用程序main函数在执行之前，加载器会做准备工作一样。在内核真正运行前也要有很多准备工作。
但是因为准备工作确实有点多，我这里只记录以下目前我知道的。

首先我们 用bochs探索[start_kernel之前][2]的黑暗世界

[1]: /bootup/01_init_call.md
[2]: /bootup/02_before_start_kernel.md