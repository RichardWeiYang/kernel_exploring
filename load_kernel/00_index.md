# start_kernel之前

一般认为start_kernel是整个内核代码开始的地方。但你有没有好奇过在这个之前都发生了什么。

就好像应用程序main函数在执行之前，加载器会做准备工作一样。在内核真正运行前也要有很多准备工作。这里我们就详细探索一下内核加载的全流程。

首先我们用bochs探索[bootloader如何加载bzImage][2]。

有了称手的工具后，我们就可以[保护模式内核代码赏析][3]

然后我们看看[内核压缩与解压][4]

[2]: /load_kernel/02_how_bzImage_loaded.md
[3]: /load_kernel/03_analysis_protected_kernel.md
[4]: /load_kernel/04_compress_decompress_kernel.md