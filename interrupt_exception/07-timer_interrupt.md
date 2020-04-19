时钟中断牵扯到系统的其他子系统，如调度，RCU。本着最小可用原则，我们只探究一下时钟中断的初始化过程以及最后都做了哪些操作。

# 相似的初始化

时钟中断的IDT初始化也普通中断初始化的隔壁。

```
  start_kernel()
      init_IRQ()
          native_init_IRQ()
              idt_setup_apic_and_irq_gates()
                  idt_setup_from_table(idt_table, apic_idts, ARRAY_SIZE(apic_idts), true);
                      INTG(LOCAL_TIMER_VECTOR, apic_timer_interrupt)
```

所以当个时钟中断来了后，就会调用到apic_timer_interrupt(smp_apic_timer_interrupt)。

貌似也不难嘛。

# 未知的event_handler

我们打开这个中断处理函数 smp_apic_timer_interrupt

```
smp_apic_timer_interrupt(regs)
  local_apic_timer_interrupt()
    evt = this_cpu_ptr(&lapic_events);
    evt->event_handler(evt)
```

此处出现了一个新成员, lapic_events。并且调用了它的回调函数event_handler。这个东西是个啥？

# 掘地三尺

这个event_handler的设置还真实隐藏得很深啊。废了九牛二虎之力才找到了它。

```
setup_boot_APIC_clock()
    calibrate_APIC_clock()
    setup_APIC_timer()
        evt = this_cpu_ptr(&lapic_events);
        memcpy(levt, &lapic_clockevent, sizeof(*levt));
        clockevents_register_device(evt)
            tick_check_new_device(dev);
                tick_setup_device(td, newdev, cpu, cpumask_of(cpu));
                    tick_setup_periodic(newdev, 0);
                        tick_set_periodic_handler(dev, broadcast);
                            dev->event_handler = tick_handle_periodic;
```

反正我是眼花了，还好终于找到了设置的地方。也找到了真正在时钟中断时调用的函数**tick_handle_periodic**。

# 主角登场

```
    tick_periodic(cpu)
        update_process_times(user_mode(get_irq_regs()))
            account_process_tick(p, user_tick)
            run_local_timers()
                raise_softirq(TIMER_SOFTIRQ), if required
            rcu_sched_clock_irq(user_tick)
            scheduler_tick()
        profile_tick()
```

终于看到了时钟中断的庐山真面目。

  * 更新进程的时间
  * profile

其中除了会调用一个timer的软中断，别的都是为了调度和rcu服务的了。

好了，本次的使命到此结束。
