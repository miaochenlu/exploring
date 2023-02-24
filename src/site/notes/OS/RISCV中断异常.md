---
{"UID":20230215125748,"aliases":"CSRs","tags":["OS"],"source":null,"cssclass":null,"created":"2023-02-17 22:46","updated":"2023-02-24 20:28","dg-publish":true,"permalink":"/os/riscv/","dgPassFrontmatter":true,"noteIcon":""}
---


# CSRs

**![](https://lh3.googleusercontent.com/byyA4-Ui9Tu4Iy38Q2Y2QVKvfcRO48nXmCNI7pipSdZws8ZWsgIXphOguOuF4sK6TIE_CPudgyTYFq8aJYvofbApAXBQFeuvYvZHqDD_XDPTtg5PiX9y2jGFUYrFOfps-rDkGzjFGRuDsf91xCPTag6Tvw=s2048)

![Pasted image 20230215192659.png](/img/user/OS/attachments/Pasted%20image%2020230215192659.png)
![Pasted image 20230215192728.png](/img/user/OS/attachments/Pasted%20image%2020230215192728.png)

![Pasted image 20230215192739.png](/img/user/OS/attachments/Pasted%20image%2020230215192739.png)

## Status Registers (mstatus/sstatus)
`mstatus`用于设置当前的机器模式和处理器状态。它还包括一些其他有用的信息，例如是否启用中断和虚拟地址转换。
`sstatus`: 与 `mstatus` 类似，但是用于设置当前的超级用户模式和处理器状态。它与 `mstatus` 相比，具有更少的功能。
![Pasted image 20230215193425.png](/img/user/OS/attachments/Pasted%20image%2020230215193425.png)

Fig 3.4 and Fig 3.5 represent the format of `mstatus` and `sstatus`. The length of these regsiters is 64. Each bit is allocated to a different meaning and we can tell the status to the CPU by setting/unsetting bits.

![Fig 3.4 mstatus register (Source: Figure 3.6: Machine-mode status register (mstatus) for RV64. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-4.png)

Fig 3.4 mstatus register (Source: Figure 3.6: Machine-mode status register (mstatus) for RV64. in Volume II: Privileged Architecture)

![Fig 3.5 sstatus register (Source: Figure 4.2: Supervisor-mode status register (mstatus) for RV64. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-5.png)

Fig 3.5 sstatus register (Source: Figure 4.2: Supervisor-mode status register (mstatus) for RV64. in Volume II: Privileged Architecture)

`MIE` and `SIE` are global insterrupt bits, `M` for M-mode and `S` for S-mode. When these bits are set, interrupts are globally enabled.

## [Trap-vector Base-address Registers (mtvec/stvecc)](https://book.rvemu.app/hardware-components/03-csrs.html#trap-vector-base-address-registers-mtvecstvecc)

![Pasted image 20230215193400.png](/img/user/OS/attachments/Pasted%20image%2020230215193400.png)

## [Machine Trap Delegation Registers (medeleg/mideleg)](https://book.rvemu.app/hardware-components/03-csrs.html#machine-trap-delegation-registers-medelegmideleg)

The trap delegation registers, `medeleg` for machine-level exception delegation and `mideleg` for machine-level interrupt delegation, indicate the certain exceptions and interrupts should be directly by a lower privileged level. 

![Fig 3.7 medeleg register (Source: Figure 3.10: Machine Exception Delegation Register medeleg. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-7.png)

Fig 3.7 medeleg register (Source: Figure 3.10: Machine Exception Delegation Register medeleg. in Volume II: Privileged Architecture)

![Fig 3.8 mideleg register (Source: Figure 3.11: Machine Interrupt Delegation Register mideleg. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-8.png)

Fig 3.8 mideleg register (Source: Figure 3.11: Machine Interrupt Delegation Register mideleg. in Volume II: Privileged Architecture)

By default, all trap should be handled in M-mode (highest privileged mode). These registers can delegate a corresponding trap to lower-level privileged mode.

## [Interrupt Registers (mip/mie/sip/sie)](https://book.rvemu.app/hardware-components/03-csrs.html#interrupt-registers-mipmiesipsie)
1. `mie` : RISC-V 的中断使能寄存器。用于启用中断并控制中断优先级。
2. `mip` : RISC-V 的中断挂起寄存器。用于记录中断是否发生并通知 CPU 进行处理。
3. `sie` ：用于控制中断处理程序是否可以被外部或内部中断中断。在`sie`寄存器中，每个位都与一个特定的中断号相对应，如果该位被设置为1，则允许相应的中断处理程序被触发。如果该位被清除，则禁止相应的中断处理程序被触发。
4. `sip`：用于表示当前哪些中断被触发且尚未处理。在`sip`寄存器中，每个位也与一个特定的中断号相对应，如果该位被设置为1，则表示相应的中断已经被触发但是还未处理。如果该位被清除，则表示相应的中断未被触发或已经被处理。

当硬件检测到某个中断事件发生时，会设置`sip`相应的位为1，如果对应的`sie`位也为1，则CPU会进入中断处理程序进行处理。中断处理程序执行完毕后，需要及时清除`sip`中相应的中断位。否则，如果中断处理程序返回而该中断仍然处于挂起状态，则CPU会立即重新进入中断处理程序，导致死循环。

在操作系统中，当操作系统需要屏蔽某些中断时，可以将对应的`sie`位清除为0，从而禁止该中断被触发；而当操作系统需要响应某些中断时，则需要将对应的`sie`位设置为1，从而允许该中断被触发。而`sip`寄存器则用于记录所有已触发但未处理的中断，方便中断处理程序进行处理。

## [Exception Program Counters (mepc/sepc)](https://book.rvemu.app/hardware-components/03-csrs.html#exception-program-counters-mepcsepc)
![Pasted image 20230215194028.png](/img/user/OS/attachments/Pasted%20image%2020230215194028.png)

![Fig 3.13 mepc register (Source: Figure 3.24: Machine exception program counter register. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-13.png)

Fig 3.13 mepc register (Source: Figure 3.24: Machine exception program counter register. in Volume II: Privileged Architecture)

![Fig 3.14 sepc register (Source: Figure 4.10: Supervisor exception program counter register. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-14.png)

Fig 3.14 sepc register (Source: Figure 4.10: Supervisor exception program counter register. in Volume II: Privileged Architecture)

## [Trap Cause Registers (mcause/scause)](https://book.rvemu.app/hardware-components/03-csrs.html#trap-cause-registers-mcausescause)

The trap cause registers, `mcause` for M-mode and `scause` for S-Mode, contain a code indicating the event that caused the trap.
* mcause
![Fig 3.15 mcause register (Source: Figure 3.25: Machine Cause register mcause. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-15.png)

Fig 3.15 mcause register (Source: Figure 3.25: Machine Cause register mcause. in Volume II: Privileged Architecture)
![Pasted image 20230215194247.png|300](/img/user/OS/attachments/Pasted%20image%2020230215194247.png)
* scause
![Fig 3.16 scause register (Source: Figure 4.11: Supervisor Cause register scause. in Volume II: Privileged Architecture)](https://book.rvemu.app/img/1-3-16.png)

Fig 3.16 scause register (Source: Figure 4.11: Supervisor Cause register scause. in Volume II: Privileged Architecture)
![Pasted image 20230214220853.png|500](/img/user/OS/attachments/Pasted%20image%2020230214220853.png)

## [Trap Value Registers (mtval/stval)](https://book.rvemu.app/hardware-components/03-csrs.html#trap-value-registers-mtvalstval)

`mtval` 和 `stval` 用于保存最后一次发生异常时的导致异常的指令地址。
- `mtval`: mtval 寄存器用于在机器模式下记录触发异常的最后一个内存访问的地址。在软件中，通常使用 mcause 寄存器检查哪个异常正在处理，并使用 mtval 寄存器检查导致异常的访问的地址。
- `stval`: stval 寄存器用于在特权级模式下记录导致当前异常的数据存储地址。它可以被访问或者修改

## [Supervisor Address Translation and Protection Register (satp)](https://book.rvemu.app/hardware-components/03-csrs.html#supervisor-address-translation-and-protection-register-satp)

`satp` 是 RISC-V 架构中用来配置页表的寄存器。

在 RISC-V 的特权级结构中，内核态（Supervisor mode）和用户态（User mode）的地址空间是通过页表进行隔离和映射的。`satp` 寄存器用于设置当前特权级的页表，以实现地址转换和访问权限控制。

`satp` 寄存器的位宽是 32 位或 64 位，其中低 22 位用来存放页表的基地址，高位则包括了许多配置信息。其中最高的 4 位用来指示页表类型，剩下的位则可以用于设置访问权限、调试模式、多核模式等等。

在操作系统中，内核需要通过 `satp` 寄存器来控制进程的页表，以实现内存隔离和访问控制。

## scratch (mscratch/scratch)
`mscratch` 和 `sscratch` 都是 RISC-V 的 CSR（Control and Status Register）寄存器，用于存储 scratch 寄存器的值。

- `mscratch` 存储着机器模式下的 `xscratch` 寄存器的值。
- `sscratch` 存储着监管模式下的 `xscratch` 寄存器的值。

当发生异常时，处理器会自动将当前正在执行的指令的地址保存到一个叫做 `sepc`（Supervisor Exception Program Counter）的 CSR 寄存器中，同时将当前执行的线程的寄存器值保存到相应的保存寄存器（save registers）中，最后进入异常处理程序。在异常处理程序中，需要使用 CSR 寄存器保存一些值。然而，这些 CSR 寄存器也需要被保存下来，以便在异常处理程序结束时将状态还原到异常前的状态。为了避免这个问题，RISC-V 规定 `xscratch` 寄存器为 scratch 寄存器，可以临时存储一些值，并且当异常发生时，这些值可以不被保存。`mscratch` 和 `sscratch` 就是用来保存机器模式下和监管模式下的 `xscratch` 寄存器的值，以便在异常处理程序结束时将 `xscratch` 寄存器的值还原到异常前的状态。

在操作系统中，通常使用 `mscratch` 和 `sscratch` 保存一些上下文信息，比如当前线程的进程控制块指针，这些信息可以在异常处理程序中被保存下来，以便在异常处理程序结束时将状态还原到异常前的状态。

# CSR Insts

| 指令            | 格式                     | 描述                                                           |
| --------------- | ------------------------ | -------------------------------------------------------------- |
| csrrw rd,csr,rs | I 形式（immediate）      | 读取 CSR 值并写入 GPR 中，再将 GPR 值写回 CSR 中               |
| csrrs rd,csr,rs | I 形式                   | 读取 CSR 值并写入 GPR 中，再使用 GPR 中的值对 CSR 进行原子设置 |
| csrrc rd,csr,rs | I 形式                   | 读取 CSR 值并写入 GPR 中，再使用 GPR 中的值对 CSR 进行原子清除 |
| csrrwi rd,csr,u | I 形式                   | 将 CSR 中的值写入 GPR 中，再将 GPR 中的值写回 CSR 中           |
| csrrsi rd,csr,u | I 形式                   | 读取 CSR 值并写入 GPR 中，再使用立即数对 CSR 进行原子设置      |
| csrrci rd,csr,u | I 形式                   | 读取 CSR 值并写入 GPR 中，再使用立即数对 CSR 进行原子清除      |
| csrwi csr,u     | Z 形式（zero immediate） | 将立即数写入 CSR 中                                            |
| csrci csr,u     | Z 形式                   | 使用立即数对 CSR 进行原子清除                                  |
| csrsi csr,u     | Z 形式                   | 使用立即数对 CSR 进行原子设置                                  |
| mret            | N 形式（no operands）    | 从机器模式中的异常处理程序返回，并将处理器模式设置为先前的模式 |
| sret            | N 形式                   | 从超级模式中的异常处理程序返回，并将处理器模式设置为先前的模式 |
| uret            | N 形式                   | 从用户模式中的异常处理程序返回，并将处理器模式设置为先前的模式 |
| ecall           |                          |                                                                |
* 伪指令
| csrr rd, csr   | csrrs rd, csr, x0   | Read CSR                     |
|----------------|---------------------|------------------------------|
| csrw csr, rs   | csrrw x0, csr, rs   | Write CSR                    |
| csrs csr, rs   | csrrs x0, csr, rs   | Set bits in CSR              |
| csrc csr, rs   | csrrc x0, csr, rs   | Clear bits in CSR            |
| csrwi csr, imm | csrrwi x0, csr, imm | Write CSR, immediate         |
| csrsi csr, imm | csrrsi x0, csr, imm | Set bits in CSR, immediate   |
| csrci csr, imm | csrrci x0, csr, imm | Clear bits in CSR, immediate |

# Machine Level

[Arch Lab 2 - Google 幻灯片](https://docs.google.com/presentation/d/1f1xaCxogZAE54MDtoOHoWD30M8d7kbAB81BZAdW8uxE/edit?usp=sharing)

![Pasted image 20230215201939.png](/img/user/OS/attachments/Pasted%20image%2020230215201939.png)

![Pasted image 20230215201955.png](/img/user/OS/attachments/Pasted%20image%2020230215201955.png)

![Pasted image 20230215202011.png](/img/user/OS/attachments/Pasted%20image%2020230215202011.png)

# Supervisor Level







# 中断与异常委托
[RISC-V 特权架构 - 峰子的乐园](https://dingfen.github.io/risc-v/2020/08/05/riscv-privileged.html)

一般情况下，在系统发生异常时，控制权都会被移交到**机器模式**下的 trap 处理程序中。然而，Unix 等系统的大多数 trap 处理程序都**应该在监管者模式下**运行。一个简单的解决方案是，让机器模式下的处理程序指向监管者模式的处理程序。但这样的坏处显而易见：速度过慢，明明可以一步到位地转向监管者模式，非要绕道机器模式然后再到监管者模式。因此，RISC-V 提供了一种委托机制，将一些中断/异常处理程序委托给监管者模式。

CSR 寄存器 `mideleg` (Machine Interrupt Delegation，机器中断委托) 指示哪些**中断**将委托给监管者模式。与 `mip` 和 `mie` 一样，被委托的中断位域位置与该中断在 [mcause 寄存器的事件编码图](https://dingfen.github.io/risc-v/2020/08/05/riscv-privileged.html#mcause)的代码号一致。

例如，`mideleg[5]` 对应于监管者模式的时钟中断，如果把它为 1 ，即 `li a0, 1 << 5 ; csrw mideleg a0`， 监管者模式的时钟中断将由该模式下的异常处理程序，而不是机器模式的异常处理程序处理。当然，委托意味着对中断的一种负责，因此委托给监管者模式的任何中断都会受到监管者模式下 CSR 寄存器（主要是 `sie`、`sip` ）的控制，而没有被委托的中断对应位是无效的。

相应地，`medeleg` 寄存器处理的是**同步异常**委托，其用法与 `mideleg` 寄存器相通，被委托的异常位域位置与该异常在 [mcause 寄存器的事件编码图](https://dingfen.github.io/risc-v/2020/08/05/riscv-privileged.html#mcause)的代码号一致。例如，当 `medeleg[15]` 设置为 1 时，那么 store 过程中的页错误 (page fault) 就会委托给监管者模式。

在 hart 运行的权限级别低于或等于被委托的级别时，此时若 bit `i` 在 `mideleg` 置位，且 `mstatus.SIE` 或 `mstatus.UIE` 中断有效时，中断会被认为是全局有效的。

---

可能有人会问，那么，委托这种情况只会发生在监管者模式下么？**当然不会**。如果这个系统支持在用户模式下处理 trap ，那么监管者模式的 `sedeleg` 和 `sideleg` 寄存器就会开始委托机制：若产生的 trap 可以被委托给用户模式，那么该 trap 会转移给用户模式下的处理程序，而不是监管者模式下的处理程序。
> If U-mode traps are supported, S-mode may in turn set corresponding bits in the sedeleg and sideleg registers to delegate traps that occur in U-mode to the U-mode trap handler.

---

中断委托的具体过程如何？

- 某个 trap 被委托给了模式 `x`，并且在执行过程中触发
- `xcause` 寄存器更新，写入 trap 的起因
- `xepc` 寄存器更新，写入 trap 发生时指令的地址（虚拟地址）
- `xtval` 寄存器更新，写入 trap 对应的处理程序位置
- `mstatus.xPP` 写入在发生 trap 时的特权级别
- `mstatus.xPIE` 写入 `xIE` 的值，而 `xIE` 的值被清除
- **注意：`mcause` 和 `mepc` 以及 MPP MPIE 域不会更新**

trap 永远不会从特权较高的模式过渡到特权较低的模式。例如，如果机器模式下，非法指令异常已经被委托给监管者模式，而此时在机器模式下，软件执行了一条非法指令，则 trap 将采用机器模式的处理程序，而不是委派给监管者模式。然而，还是上面的例子，如果监管者模式下，软件遇到了非法指令异常，trap 由监管者模式的异常处理程序处理。

---
