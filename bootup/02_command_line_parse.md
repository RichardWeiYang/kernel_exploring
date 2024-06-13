内核除了在编译时的配置选项，还可以在启动时根据不同的命令行参数调整运行时的行为。

比如加上memblock=debug可以打开memblock的调试功能，启动过程中可以看到各区域变化情况。

本节我们来看一下这部分在启动时是怎么处理的。

# 全局流程

```c
start_kernel()
	parse_early_param();
		parse_early_options()
			parse_args("early options", cmdline, NULL, 0, 0, 0, NULL,
				   do_early_param);
	after_dashes = parse_args("Booting kernel",
				  static_command_line, __start___param,
				  __stop___param - __start___param,
				  -1, -1, NULL, &unknown_bootoption);
	print_unknown_bootoptions();
	if (!IS_ERR_OR_NULL(after_dashes))
		parse_args("Setting init args", after_dashes, NULL, 0, -1, -1,
			   NULL, set_init_arg);
	if (extra_init_args)
		parse_args("Setting extra init args", extra_init_args,
			   NULL, 0, -1, -1, NULL, set_init_arg);
```

从上面的流程可以看出，核心的函数就是parse_args()。

而根据不同的参数，形成了内核参数不同层级的解析过程。

# early param

内核首先解析的是early_param，使用的函数是do_early_param。函数补偿，直接拿来放这里。


```c
    const struct obs_kernel_param *p;
	for (p = __setup_start; p < __setup_end; p++) {
		if ((p->early && parameq(param, p->str)) ||
		    (strcmp(param, "console") == 0 &&
		     strcmp(p->str, "earlycon") == 0)
		) {
			if (p->setup_func(val) != 0)
				pr_warn("Malformed early option '%s'\n", param);
		}
	}
```

可以看到，这个是遍历__setup_start到__setup_end，根据情况调用setup_func。

举个例子，比如，memblock中的参数。

```c
static int __init early_memblock(char *p)
{
	if (p && strstr(p, "debug"))
		memblock_debug = 1;
	return 0;
}
early_param("memblock", early_memblock);
```

这两个怎么关联上呢？对了，就是看是不是定义在指定的一个section里。

首先我们看__setup_start/__setup_end在哪里。这个东西有点隐藏，直接搜搜不到。打开看了一下才发现这个是由INIT_SETUP定义的。

最后展开长这个样子。

```
. = ALIGN(16); __setup_start = .; KEEP(*(.init.setup)) __setup_end = .;
```

这里就看到这个循环找的是.init.setup这个section里面的东西。

然后再看early_param的定义，摘出其中重要的部分

```c
	static struct obs_kernel_param __setup_##unique_id		\
		__used __section(".init.setup")				\
		__aligned(__alignof__(struct obs_kernel_param))		\
		= { __setup_str_##unique_id, fn, early }
```

也就是定义了一个obs_kernel_param结构，而且还是放在.init.setup这个section的。

好了，这样就联系起来了。do_early_param会遍历.init.setup这个section中的数组，判断符合条件，就会运行对应的setup_func。对memblock来说，也就是early_memblock了。
