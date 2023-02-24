---
{"UID":20230213210113,"aliases":"RISC-V汇编","tags":["OS"],"source":null,"cssclass":null,"created":"2023-02-15 18:50","updated":"2023-02-16 20:55","dg-publish":true,"permalink":"/os/lab1/","dgPassFrontmatter":true,"noteIcon":""}
---

[实验指导一 - 浙江大学22年秋操作系统实验](https://zju-sec.github.io/os22fall-stu/lab1/)
[[Excalidraw/os_lab1_flow\|os_lab1_flow]]
![Pasted image 20230214215928.png](/img/user/OS/attachments/Pasted%20image%2020230214215928.png)



# RISC-V汇编
[[OS/RISCV Assembly\|RISCV Assembly]]

# 实验
## 代码结构

```
├── arch
│   └── riscv
│       ├── include
│       │   ├── defs.h
│       │   └── sbi.h
│       ├── kernel
│       │   ├── head.S
│       │   ├── Makefile
│       │   ├── sbi.c
│       │   └── vmlinux.lds
│       └── Makefile
├── include
│   ├── print.h
│   └── types.h
├── init
│   ├── main.c
│   ├── Makefile
│   └── test.c
├── lib
│   ├── Makefile
│   └── print.c
└── Makefile
```

需要完善以下文件：
- arch/riscv/kernel/head.S
- lib/Makefile
- arch/riscv/kernel/sbi.c
- lib/print.c
- arch/riscv/include/defs.h

## 4.2 编写head.S
学习riscv的汇编。
完成 arch/riscv/kernel/head.S 。我们首先为即将运行的第一个 C 函数设置程序栈（栈的大小可以设置为4KB），并将该栈放置在`.bss.stack` 段。接下来我们只需要通过跳转指令，跳转至 main.c 中的 `start_kernel` 函数即可。

* 设置栈指针
* 跳转到`start_kernel`

```
.extern start_kernel

    .section .text.entry
    .globl _start
_start:
    # ------------------
    # - your code here -
    la sp, boot_stack_top
    j start_kernel
    # ------------------

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 # <-- change to your stack size

    .globl boot_stack_top
boot_stack_top:
```

### 4.3 完善 Makefile 脚本
阅读文档中关于 [Makefile](https://zju-sec.github.io/os22fall-stu/lab1/#35-makefile) 的章节，以及工程文件中的 Makefile 文件，根据注释学会 Makefile 的使用规则后，补充 `lib/Makefile`，使工程得以编译。

完成此步后在工程根文件夹执行 make，可以看到工程成功编译出 vmlinux。

```Makefile
C_SRC       = $(sort $(wildcard *.c))
OBJ		    = $(patsubst %.c,%.o,$(C_SRC))

all:$(OBJ)
	
%.o:%.c
	${GCC} ${CFLAG} -c $<
clean:
	$(shell rm *.o 2>/dev/null)

```

![Pasted image 20230212195256.png](/img/user/OS/attachments/Pasted%20image%2020230212195256.png)


# 思考题
## 1. 请总结一下 RISC-V 的 calling convention，并解释 Caller / Callee Saved Register 有什么区别？
Calling Conventions
![Pasted image 20230213201824.png|500](/img/user/OS/attachments/Pasted%20image%2020230213201824.png)
 [为什么要区分caller saved和callee saved registers? - 知乎](https://www.zhihu.com/question/453450905/answer/2328315047)
 函数调用过程中，临时寄存器和非临时寄存器，有着不同的保护责任方。非临时寄存器，如果在callee里面改了，不做保护的话，caller里面声明的变量的值都变了，破坏了功能，所以这类寄存器，callee使用了一个就要保护一个；但针对临时寄存器，临时变量用完即废，callee破坏了也可能没啥影响，callee就没义务保护，但是如果caller确实希望保存这个临时变量，那么必须由caller声明进行保护
![Pasted image 20230213200659.png](/img/user/OS/attachments/Pasted%20image%2020230213200659.png)

![Pasted image 20230213200743.png](/img/user/OS/attachments/Pasted%20image%2020230213200743.png)

## 2. 编译之后，通过 System.map 查看 vmlinux.lds 中自定义符号的值
system.map是通过nm得到的
nm ../../vmlinux >  ../../System.map

```
0000000080200000 t $x
000000008020000c t $x
00000000802000d8 t $x
000000008020016c t $x
000000008020017c t $x
00000000802001f0 t $x
0000000080200000 A BASE_ADDR
0000000080203000 B boot_stack
0000000080204000 B boot_stack_top
0000000080204000 B _ebss
0000000080202000 D _edata
0000000080204000 B _ekernel
0000000080201002 R _erodata
000000008020032c T _etext
0000000080202000 d _GLOBAL_OFFSET_TABLE_
00000000802001f0 T puti
000000008020017c T puts
000000008020000c T sbi_ecall
0000000080203000 B _sbss
0000000080202000 D _sdata
0000000080200000 T _skernel
0000000080201000 R _srodata
0000000080200000 T _start
00000000802000d8 T start_kernel
0000000080200000 T _stext
000000008020016c T test

```