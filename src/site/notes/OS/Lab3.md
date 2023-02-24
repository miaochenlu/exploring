---
{"UID":20230221221115,"aliases":"背景知识,思考题,Lab3","tags":null,"source":null,"cssclass":null,"created":"2023-02-21 22:11","updated":"2023-02-23 20:14","dg-publish":true,"permalink":"/os/lab3/","dgPassFrontmatter":true,"noteIcon":""}
---


[[Excalidraw/os_lab3_flow\|os_lab3_flow]]

![Pasted image 20230221221118.png](/img/user/OS/attachments/Pasted%20image%2020230221221118.png)

# 背景知识
## 线程相关属性
在不同的操作系统中, 为每个线程所保存的信息都不同。在这里, 我们提供一种基础的实现, 每个线程会包括：

- `线程ID`：用于唯一确认一个线程。
- `运行栈`：每个线程都必须有一个独立的运行栈, 保存运行时的数据。
- `执行上下文`：当线程不在执行状态时, 我们需要保存其上下文（其实就是`状态寄存器`的值）, 这样之后才能够将其恢复, 继续运行。
- `运行时间片`：为每个线程分配的运行时间。
- `优先级`：在优先级相关调度时, 配合调度算法, 来选出下一个执行的线程。
## 线程切换流程图

```
           Process 1         Operating System            Process 2
               +
               |                                            X
 P1 executing  |                                            X
               |                                            X
               v Timer Interrupt Trap                       X
               +---------------------->                     X
                                      +                     X
               X                  do_timer()                X
               X                      +                     X
               X                  schedule()                X
               X                      +                     X
               X              save state to PCB1            X
               X                      +                     X
               X           restore state from PCB2          X
               X                      +                     X
               X                      |                     X
               X                      v Timer Interrupt Ret
               X                      +--------------------->
               X                                            |
               X                                            |  P2 executing
               X                                            |
               X                       Timer Interrupt Trap v
               X                      <---------------------+
               X                      +
               X                  do_timer()
               X                      +
               X                  schedule()
               X                      +
               X              save state to PCB2
               X                      +
               X           restore state from PCB1
               X                      +
               X                      |
                 Timer Interrupt Ret  v
               <----------------------+
               |
 P1 executing  |
               |
               v
```

- 在每次处理时钟中断时, 操作系统首先会将当前线程的运行剩余时间减少一个单位。之后根据调度算法来确定是继续运行还是调度其他线程来执行。
- 在进程调度时, 操作系统会遍历所有可运行的线程, 按照一定的调度算法选出下一个执行的线程。最终将选择得到的线程与当前线程切换。
- 在切换的过程中, 首先我们需要保存当前线程的执行上下文, 再将将要执行线程的上下文载入到相关寄存器中, 至此我们就完成了线程的调度与切换。

# 思考题
1. 在 RV64 中一共用 32 个通用寄存器, 为什么 `context_switch` 中只保存了14个?
    保存的是`ra`, `sp`, `s0~s11`
应该是只保存了一些callee saved register和ra sp
![Pasted image 20230213201824.png|500](/img/user/OS/attachments/Pasted%20image%2020230213201824.png)
2. 当线程第一次调用时, 其 `ra` 所代表的返回点是 `__dummy`。那么在之后的线程调用中 `context_switch` 中, `ra` 保存/恢复的函数返回点是什么呢? 请同学用 gdb 尝试追踪一次完整的线程切换流程, 并关注每一次 `ra` 的变换 (需要截图)。

调用schedule函数后，ra变成switch_to+100, 这个值会被存储，如果再次切换到这个线程，restore ra的值就变成switch_to+100
* 第一次
![Pasted image 20230222203951.png|400](/img/user/OS/attachments/Pasted%20image%2020230222203951.png)
* 第二次
![Pasted image 20230222203902.png|400](/img/user/OS/attachments/Pasted%20image%2020230222203902.png)