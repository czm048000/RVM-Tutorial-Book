# RVM-Tutorial-Book

[TOC]

## 选题背景及意义

系统级虚拟化技术已经广泛应用于云计算中的数据中心等领域，是操作系统的重要组成部分。RISC-V架构支持了Hypervisor扩展，可以用于构建虚拟机应用程序，实现在一台计算机上同时运行多个操作系统，提供更好的资源划分与隔离。

另一方面，目前高校中对系统级虚拟化的 OS 教学还比较少，而现有的 OS 教学实验也存在对底层实现关注不足的情况，无法满足学有余力的同学们学习操作系统的提高要求。

因此本项目将首先用 Rust 语言编写的轻量级 hypervisor (虚拟机管理器，也叫 Virtual Machine Monitor、VMM) 模块，支持RISC-V架构的操作系统作为虚拟机运行；再将此实现过程裁剪为操作系统教学实验。一方面作为hypervisor教学，另一方面也作为更加熟悉底层细节的操作系统提高实验。

## 教学实验设计目标

目前国内研究型大学的进行操作系统实验的主要方法是给出一些小型操作系统作为框架代码，由学生在此基础上编写代码完成扩展。这些操作系统教学实验专注于操作系统核心概念（例如异常、页表、进程管理等），会尽可能屏蔽底层的具体操作细节。例如：

> - 从执行内核第一条指令开始到真正启动操作系统并进入用户态程序会经历什么过程，其中硬件完成了什么，软件完成了什么？
> - 当一个用户程序执行系统调用时会发生什么，产生什么指令，操作系统如何处理？
> - ······

在现有的教学实验中，这些实现细节大多会在框架代码中给出，同学们不需要了解这些事实也可以完成实验。此要求对于普通本科生教学要求来说已经足够，但是对于学有余力的同学，如果对操作系统方向更加感兴趣，例如希望能独立开发出一个完整的操作系统，那么需要知道更加细节的过程。

本实验假设同学们对操作系统以及RISC-V体系有基本了解，作为学有余力同学的进阶实验。

## 实验指导书框架

如何从Hyper进入Guest，异常，中断，内存

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