内核除了在编译时的配置选项，还可以在启动时根据不同的命令行参数调整运行时的行为。

比如加上memblock=debug可以打开memblock的调试功能，启动过程中可以看到各区域变化情况。

本节我们来看一下这部分在启动时是怎么处理的。

# 全局流程

```c
start_kernel()
	parse_early_param();
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
