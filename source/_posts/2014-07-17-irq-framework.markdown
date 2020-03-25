---
layout: post
title: "irq framework"
date: 2014-07-17 10:35
comments: true
categories: [IRQs]
tag: [IRQ]
---

### 中断处理过程：

#### 硬件中断到中断控制器
reg value-->irq(int) ---> struct irq_desc
```
==> 中断时的有一个寄存器会保存中断源的vector值.
==> ==> `arch/x86/kernel/entry_64.S`调用函数`do_IRQ`.
==> ==> ==> `do_IRQ`依据`vector_irq`和vector值, 找到对应的中断号,并调用`handle_irq`.
==> ==> ==> ==> `handle_irq`通过函数irq_to_descdesc,可将中断号转化为`struct irq_desc`.
==> ==> ==> ==> generic_handle_irq_desc(irq, desc);
==> ==> ==> ==> ==> `generic_handle_irq_desc`调用 desc->handle_irq(irq, desc);
```
注:这里的handle_irq不是真正的中断处理函数,而是几大类中断控制器处理函数.
如82599, msi等.
具体分析见:[irq study1](http://martinbj2008.github.io/blog/2014/07/17/irq-study)
#### 中断控制器到具体的中断处理函数
```
==> handle_level_irq
==> ==> irqreturn_t handle_irq_event(struct irq_desc *desc)
==> ==> ==> struct irqaction *action = desc->action
==> ==> ==> ret = handle_irq_event_percpu(desc, action);
==> ==> ==> ==> action->handler(irq, action->dev_id);
```
这里的`action->handler`才是我们使用`request_irq`注册的中断处理函数.
具体分析见:
具体分析见:[irq study2](http://martinbj2008.github.io/blog/2014/07/17/register-irq-handler)

