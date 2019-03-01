虽然是一名软件工程师，不过有时候还是要对硬件有一些了解。

# 硬件规范

要了解硬件，第一件事当然是查看手册。还好，APIC的信息在SDM vol3中有比较详细的介绍。这里我做一个个人理解的总结。

![APIC Structure](/interrupt_exception/apic.png)

第一眼看确实有点懵逼，多看几眼就好了。

为了方便理解，我们把这些寄存器按照功能分类：

```
  Timer related:

      CCR: Current Count Register
      ICR: Initial Count Register
      DCR: Divide Configuration Register
      Timer: in LVT

  LVT (Local Vector Table):

      Timer
      Local Interrupt
      Performance Monitor Counters
      Thermal Sensor
      Error

  IPI:

      ICR: Interrupt Command Register
      LDR: Logical Destination Register
      DFR: Destination Format Register

  Interrupt State:

      ISR: In-Service Register
      IRR: Interrupt Request Register
      TMR: Trigger Mode Register
```

这样或许有助于理解和记忆。

# 初始化

初始化还真有点麻烦，说实话我没有验证过，只是扫了一眼代码。

大致上分成两部分：

  * 找到驱动
  * 初始化

因为APIC出现了一段时间了，也有了不同的版本。所以系统在做真正的初始化之前将要先探测出究竟是哪种中断控制器。

系统中定义apic驱动的方式还和常见的驱动不一样。系统使用apic_driver/apic_drivers来定义驱动。而且其实就是一个数据结构。在这里借用驱动这个词，只是为了方便描述。

这个过程分散在多个地方，如apic_boot_init, apic_intr_mode_init。具体之间的关系和顺序本人没有深究。

选好了驱动，就可以做真正的初始化了。初始化的函数还是很清晰的。

> setup_local_APIC

当你打开这个函数之后，你就会发现主要的工作就是配置APIC中的各个寄存器。比如设置LVT0/1。

好了，其实这步还挺简单的。

# 对应的中断向量

相对于初始化，中断向量就要好找很多。

```
start_kernel()
    init_IRQ()
        native_init_IRQ()
            idt_setup_apic_and_irq_gates()
                idt_setup_from_table(idt_table, apic_idts, ARRAY_SIZE(apic_idts), true);
```

我相信，有了之前的经验，这行代码就是小菜了。

等等，这样就结束了么？是不是还差了点什么？

> IDT中的某一项是如何同apic中的寄存器关联的呢？

我们就以Thermal Sensor这个中断为例来看看之间的关联。

先看apic_idts中的定义，找到这个中断处理的定义处：

```
  INTG(THERMAL_APIC_VECTOR,	thermal_interrupt),
```

由此可知，这个中断占用的向量号是**THERMAL_APIC_VECTOR**。

既然如此，按照规范就需要在Thermal Sensor Register中注册这个向量号，这样在中断发生时才能正确找到并触发中断。

```
  /* We'll mask the thermal vector in the lapic till we're ready: */
  h = THERMAL_APIC_VECTOR | APIC_DM_FIXED | APIC_LVT_MASKED;
  apic_write(APIC_LVTTHMR, h);
```

在代码中我们找到这么两行，其中就有我们要找的THERMAL_APIC_VECTOR。而接下来的动作就是写到APIC_LVTTHMR指定的寄存器中。

那这个寄存器地址是什么呢？

```
#define	APIC_LVTTHMR	0x330
```

到这里，我想你也已经清楚了～
