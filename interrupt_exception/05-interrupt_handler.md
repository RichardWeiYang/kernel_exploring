中断的处理其实包含了很多细节，有软件架构上的，也有硬件架构上的。

而我一直以来的困惑是一个中断是怎么样通过**一个中断向量运行到一个中断函数的**。这次我的目的是来解决这个问题。

# 从中断向量初始化开始

在前面的总结中我们已经看到IDT中有部分是用作处理外部中断的中断向量。那我们就来看看这些中断向量是如何设置的。

```
  start_kernel()
      init_IRQ()
          native_init_IRQ()
              idt_setup_apic_and_irq_gates()
                  for_each_clear_bit_from(i, system_vectors, FIRST_SYSTEM_VECTOR) {
                      entry = irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR);
                      set_intr_gate(i, entry);
                  }
```

如果大家仔细看这个循环，就是将外部中断向量填写irq_entries_start对应的内容。那这个irq_entries_start对应的是什么呢？

```
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
	UNWIND_HINT_IRET_REGS
	pushq	$(~vector+0x80)			/* Note: always in signed byte range */
	jmp	common_interrupt
	.align	8
	vector=vector+1
    .endr
END(irq_entries_start)
```

这段就是定义了(FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)个中断向量的函数。都长一个样，最后跳转到了common_interrupt。

```
common_interrupt:
	addq	$-0x80, (%rsp)			/* Adjust vector to [-256, -1] range */
	call	interrupt_entry
	UNWIND_HINT_REGS indirect=1
	call	do_IRQ	/* rdi points to pt_regs */
	/* 0(%rsp): old RSP */
ret_from_intr:
  ...
END(common_interrupt)
```

这个函数太长了，我们就只看重点。重点就是大家都通过do_IRQ来处理！

# do_IRQ后断掉的线索

到此为止，一切顺利，但是好景不长。

```
  do_IRQ()
      handle_irq()
          generic_handle_irq_desc()
              desc->handle_irq(desc);
```

到这里我就抓瞎了，每个desc的handle_irq是哪里设置的呢？难道每个都不同么？

从上面的代码中我们可以看到具体的中断处理由irq_desc的handle_irq来做。那这个handle_irq到底是什么样子呢？

说实话，整个代码的框架暂时我还不知道，这一部分比我想象的要复杂一些。不仅涉及了软件的架构，还涉及到了硬件的部分知识。在这里暂时不做详细的描述，来看看找到的一种实现可能路径。

# pci设备注册中断处理的一种情况

我们以pci设备为例，来看看其中一种情况的代码流程。

## 从request_irq()入手

request_irq()是驱动向内核注册中断处理函数的API。那就从这里切入。

比如e1000的驱动中，e1000_request_irq()中有如下代码：

```
    e1000_request_irq()
        request_irq(adapter->pdev->irq, handler,..., ...)
            desc = irq_to_desc(irq);
```

可以看到，request_irq()前两个参数中一个是irq号，另一个就是处理函数。有了这个irq号则可以找到对应的irq_desc了。

> 也就是当前内核的架构中会通过某些方式实现irq_desc和irq之间的映射。

这么看，这个irq到irq_desc的映射在注册函数之前就存在了。这里不过是添加上具体的处理函数而已。

## pci_host_bridge->map_irq()

到了这里就有点绕了，因为这个映射的方法也是有很多的。我们只看可能的一种方式。

还是从上面的代码入手，adapter->pdev->irq是一个pci设备的irq。那就是要找到pci设备的irq是什么时候设置的，就可以找到这个映射是什么时候建立的了。

找到的一个设置的地方在pci_assign_irq()中。摘取关键代码如下：

```
    int irq = 0;
    struct pci_host_bridge *hbrg = pci_find_host_bridge(dev->bus);
    irq = (*(hbrg->map_irq))(dev, slot, pin);
    dev->irq = irq;
```

可以看到，这里就涉及到硬件了。也就是由pci_host_bridge来决定某个设备获得的irq号是多少。

在代码中drivers/pci/controller目录下保存这pci_host_bridge设备的驱动。其中很多驱动的map_irq都定义为of_irq_parse_and_map_pci()。而这个函数的具体实现则调用了irq_create_of_mapping()。

```
int of_irq_parse_and_map_pci(const struct pci_dev *dev, u8 slot, u8 pin)
{
	struct of_phandle_args oirq;
	int ret;

	ret = of_irq_parse_pci(dev, &oirq);
	if (ret)
		return 0; /* Proper return code 0 == NO_IRQ */

	return irq_create_of_mapping(&oirq);
}
```

## irq_domain->map()

在of_irq_parse_and_map_pci()中，如果of_irq_parse_pci()失败，则会使用irq_create_of_mapping()来设置irq_desc的handle_irq函数。其大致流程如下：

```
    irq_create_of_mapping()
        irq_create_fwspec_mapping()
            irq_create_mapping()
                irq_domain_associate()
                    irq_domain->ops->map()
```

到这里我们又接触到了一个新概念, irq_domain。好在爬过了这么多坑，现在也很淡定了。顺藤摸瓜，可以找到这个map函数的一种可能是irq_map_generic_chip()。

```
struct irq_domain_ops irq_generic_chip_ops = {
	.map	= irq_map_generic_chip,
	.unmap  = irq_unmap_generic_chip,
	.xlate	= irq_domain_xlate_onetwocell,
};
```

功夫不负有心人，在这个函数中，我们终于找到了irq_desc->handle_irq设置的地方了。

```
    irq_map_generic_chip()
        struct irq_chip_type *ct;
        irq_domain_set_info(..., ct->handler, NULL, NULL);
```

而这个ct->handler就是最后会赋值给irq_desc->handle_irq的了。

## chip_types->handler

终于是时候看一眼这个handler长什么样子了，在代码中这个handler可以设置为几种情况：

* handle_level_irq
* handle_edge_irq
* handle_edge_eoi_irq
* handle_percpu_irq
* ...

虽然形式上有很多个，但殊途同归大家最后都调用了__handle_irq_event_percpu()，而其中一部分核心代码如下：

```
struct irqaction *action;

for_each_action_of_desc(desc, action) {
  irqreturn_t res;

  trace_irq_handler_entry(irq, action);
  res = action->handler(irq, action->dev_id);
  trace_irq_handler_exit(irq, action, res);

  if (WARN_ONCE(!irqs_disabled(),"irq %u handler %pF enabled interrupts\n",
          irq, action->handler))
    local_irq_disable();

  switch (res) {
  case IRQ_WAKE_THREAD:
    /*
     * Catch drivers which return WAKE_THREAD but
     * did not set up a thread function
     */
    if (unlikely(!action->thread_fn)) {
      warn_no_thread(irq, action);
      break;
    }

    __irq_wake_thread(desc, action);

    /* Fall through to add to randomness */
  case IRQ_HANDLED:
    *flags |= action->flags;
    break;

  default:
    break;
  }

  retval |= res;
}
```

这段就是大家熟知的irqaction的链表了。

我想终于可以说到这里，基本理清了中断处理的软件处理架构。

# 粗略的一张图

```
    IDT (idt_table)      struct irq_desc                                struct irqaction      struct irqaction
    +-----------+        +-------------------------------------+    +----------------+    +----------------+
    |           |        |action (struct irqaction*)         --|--->|         next --|--->|         next --| -> NULL
    +-----------+        |                                     |    |                |    |                |
    |           |        |                                     |    | +->handler     |    | +->handler     |
    +-----------+        +-------------------------------------+    +-|--------------+    +-|--------------+
    .           .   +--->|handle_irq()                         |      +---------------------+
    .           .   |    |= handle_level_irq                   |      |
    .           .   |    |= handle_edge_irq                    |      |
    +-----------+   |    |= handle_edge_eoi_irq                |      |
    |           |   |    |= handle_percpu_irq                  |      |
    +-----------+   |    |  |                                  |      |
    |do_IRQ  ---|---+    |  +->  __handle_irq_event_percpu() --|------+
    +-----------+        |                                     |
    |           |        +-------------------------------------+
    +-----------+        
    .           .        
    .           .        
    .           .        
    +-----------+
    |           |
    +-----------+
    |           |
    +-----------+
```
