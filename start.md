# RVM-Tutorial-Book

[TOC]

## 教学实验设计目标

本实验假设同学们对操作系统以及RISC-V体系有基本了解，作为学有余力同学的进阶实验。

如何从Hyper进入Guest，异常，中断，内存

## 实验指导书框架

### lab0：实验框架搭建过程

虚拟化层

### lab1：进入和退出GuestOS

#### lab1-0，上下文保存与切换

类比OS进入用户程序的过程，希望能保存与恢复寄存器。

分别保存了Host和Guest的通用寄存器，一些特殊的CSR。

Guest：所有通用寄存器，因为作为Guest并不能感受到Host的存在

Host：只需要保存saved寄存器，temporary寄存器不用保存

#### lab1-1，如何进入GUESTOS？

一个guestos是如何启动起来的？类比OS进入用户程序的过程，通过恰当设置一些寄存器，再调用sret实现。

当执行sret的时候CPU会做什么事情？

- pc被修改为sepc
- 根据hstatue寄存器中的字段修改特权级（SPV、SPVP）

（picture：hstatus，spv）

- 容易被忽略的sstatus.SPP

（picture：privilege level）

##### 实现

- 约定GuestOS的BASEADDRESS是0x90000000
- 在VCPU的init里修改sepc和hstatus，sstatus

（看代码）

#### lab1-2，如何从GUESTOS返回hypervisor

当GUESTOS产生trap的时候，希望能执行上下文保存，然后进入hypervisor继续执行

当发生trapl的时候CPU会做什么事情？

- 发生异常的指令保存在sepc中，pc设置为stvec

也就是说

- 将guest的stvec设置为`__riscv_exit`的地址
- 恢复之前保存的host-ra，执行ret、

#### lab1-3，如何处理GuestOS的请求

我们guestos跳进了host之后其实没有进行任何处理。

希望：根据scause正确处理Guest的请求，处理完成后再返回Guest继续执行。

在lab1中，首先要处理的是调用ecall进行串口输出。ecall的参数放在通用寄存器中，寄存器在上下文切换时保存在了guest-state里，因此只要在hypervisor里再生成一个ecall即可



### lab2：将一些中断交给GuestOS处理

GuestOS在lab2时可以跑用户态程序，并且当用户程序发生不合法行为时可以进行一些处理，例如发生Illegalinstruction和storefault时就将当前运行的任务杀死并执行下一个任务。作为在Host在遇到这些类型的异常时并不能进行如此精细的处理，因此应该将Guest能够自己处理的异常交给Guest处理，就是执行异常委托

实现异常委托有直通和模拟两种方法。

直通指的是直接设置异常委托寄存器hedeleg，其他的工作交给硬件执行。（picture）但是hedeleg生效的条件是已经设置了medeleg，而查看opensbi代码后发现，并没有设置medeleg对上面两类异常的委托，在m态将异常转发给s态进行处理。

模拟指的是在Host里进行异常处理，将异常转发给Guest。由于希望不要改动opensbi代码，在host中对以上两类异常也采用模拟的方式转发给guest。

如何模拟？

- vsstatus：SPP，SIE，SPIE
- vscause,vstval,vsepc