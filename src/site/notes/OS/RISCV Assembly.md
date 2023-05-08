---
{"dg-publish":true,"permalink":"/os/riscv-assembly/","dgPassFrontmatter":true,"noteIcon":""}
---

# RISC-V汇编语句
## 构成

一条典型的 RISC-V 汇编语句由3部分组成：

\[Label:] \[operation] \[comment]

label（标号）: GNU汇编中，任何以冒号结尾的标识符都被认为是一个标号。
-   operation 可以有以下多种类型：
    -   instruction（指令）: 直接对应二进制机器指令的字符串
    -   pseudo-instruction（伪指令）: 为了提高编写代码的效率，可以用一条伪指令指示汇编器产生多条实际的指令(instructions)。 (比如li指令，nop指令)
    -   directive（指示/伪操作）: 通过类似指令的形式(以“.”开头)，通知汇编器如何控制代码的产生等，不对应具体的指令。比如.text指示以下都放入代码段
    -   macro：采用 .macro/.endm 自定义的宏
        -   .marco 可以带多个参数，，如果在宏内部引用参数，可以使用如下方式：

```assembly
.macro plus p0 p1 
mov r1, \p0 @ \p0 调用参数 p0 
mov r2, \p1 @ \p1 调用参数 p1 
.endm
```

-   comment（注释）: 常用方式，“#” 开始到当前行结束。
```
# First RISC-V Assemble Sample

.macro do_nothing	# directive
	nop		# pseudo-instruction
	nop		# pseudo-instruction
.endm			# directive

	.text		# directive
	.global _start	# directive
_start: 		# Label
	li x6, 5	# pseudo-instruction
	li x7, 4	# pseudo-instruction
	add x5, x6, x7	# instruction
	do_nothing	# Calling macro
stop:	j stop		# statement in one line

	.end		# End of file

```

## 伪指令概览
[riscv-asm-manual/riscv-asm.md at master · riscv-non-isa/riscv-asm-manual · GitHub](https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md)

| 伪指令 | 语法           | 等价指令                       | 描述                                | 示例                |
|-----|--------------|----------------------------|-----------------------------------|-------------------|
| NEG | NEG RD, RS   | SUB RD, x0, RS             | 对RS中的值取反，将结果放入RD                  | neg x5, x6        |
| MV  | MV RD, RS    | ADDI RD, RS, 0             | 将RS中的值拷贝到RD                       | mv x5, x6         |
| NOP | NOP          | ADDI x0, x0, 0             | 什么也不做                             | nop               |
| LI  | LI RD, IMM   | LUI和ADDI的组合 如果大数，用LUI+ADDI | RD=IMM                            | li x5, 0x12345678 |
|     |              | 小数ADDI即可                   |                                   |                   |
| LA  | LA RD, LABEL | AUIPC和ADDI的组合              | RD=LABEL                          | la x5, foo        |
| NOT | NOT RD, RS   | XORI RD, RS, -1            | 对RS的值按位取反，结果存于RD                  | not x5, x6        |
| J   | J OFFSET     | JAL X0, OFFSET             | 跳转后不需要返回，可以利用 x0 代替JAL 和JALR 中的RD | j leap            |
| JR  | JR RS        | JALR X0, 0(RS)             | 跳转后不需要返回，可以利用 x0 代替JAL 和JALR 中的RD | jr x2             |

| 伪指令  | 语法 | 等价指令 | 描述                                                  |
|------|----|------|-----------------------------------------------------|
| BLE  |    |      |                                                     |
| BLEU |    |      |                                                     |
| BGT  |    |      |                                                     |
| BGTU |    |      |                                                     |
| BEQZ |    |      |                                                     |
| BNEZ |    |      |                                                     |
| BLTZ |    |      |                                                     |
| BLEZ |    |      |                                                     |
| BGTZ |    |      | Branch if Greater Than Zero，如果RS > 0，跳转到OFFSET      |
| BGEZ |    |      | Branch if Greater or Equal Zero, 如果RS >=0，跳转到OFFSET |

| 伪指令 | 语法           | 等价指令                       | 描述               | 示例                |
|-----|--------------|----------------------------|------------------|-------------------|
| NEG | NEG RD, RS   | SUB RD, x0, RS             | 对RS中的值取反，将结果放入RD | neg x5, x6        |
| MV  | MV RD, RS    | ADDI RD, RS, 0             | 将RS中的值拷贝到RD      | mv x5, x6         |
| NOP | NOP          | ADDI x0, x0, 0             | 什么也不做            | nop               |
| LI  | LI RD, IMM   | LUI和ADDI的组合 如果大数，用LUI+ADDI | RD=IMM           | li x5, 0x12345678 |
|     |              | 小数ADDI即可                   |                  |                   |
| LA  | LA RD, LABEL | AUIPC和ADDI的组合              | RD=LABEL         | la x5, foo        |
| NOT | NOT RD, RS   | XORI RD, RS, -1            | 对RS的值按位取反，结果存于RD | not x5, x6        |

![Pasted image 20230212191646.png](/img/user/OS/attachments/Pasted%20image%2020230212191646.png)
尾调用不再调用函数

| 伪指令 | 语法         | 等价指令          | 描述             | 示例              |
| ------ | ------------ | ----------------- | ---------------- | ----------------- |
| CSRW   | CSRW CSR, RS | CSRRW x0, CSR, RS | x0=CSR, CSR=RS   | csrw mscratch, t6 |
| CSRR   | CSRW RD, CSR | CSRRS RD, CSR, x0 | RD=CSR, CSR\|=RS | csrr t5, mie      | 

| pseudoinstruction | base instructions   | Meaning                      | 
| ----------------- | ------------------- | ---------------------------- |
| csrr rd, csr      | csrrs rd, csr, x0   | Read CSR                     |
| csrw csr, rs      | csrrw x0, csr, rs   | Write CSR                    |
| csrs csr, rs      | csrrs x0, csr, rs   | Set bits in CSR              |
| csrc csr, rs      | csrrc x0, csr, rs   | Clear bits in CSR            |
| csrwi csr, imm    | csrrwi x0, csr, imm | Write CSR, immediate         |
| csrsi csr, imm    | csrrsi x0, csr, imm | Set bits in CSR, immediate   |
| csrci csr, imm    | csrrci x0, csr, imm | Clear bits in CSR, immediate |

## 寄存器
![Pasted image 20230212191638.png](/img/user/OS/attachments/Pasted%20image%2020230212191638.png)
![Pasted image 20230212211104.png](/img/user/OS/attachments/Pasted%20image%2020230212211104.png)

# 内联汇编
内联汇编（通常由 asm 或者 __asm__ 关键字引入）提供了将汇编语言源代码嵌入 C 程序的能力。 内联汇编的详细介绍请参考 [Extended Asm (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html) 。 下面简要介绍一下这次实验会用到的一些内联汇编知识：

内联汇编基本格式为：

    __asm__ volatile (
        "instruction1\n"
        "instruction2\n"
        ......
        ......
        "instruction3\n"
        : [out1] "=r" (v1),[out2] "=r" (v2)
        : [in1] "r" (v1), [in2] "r" (v2)
        : "memory"
    );
其中，三个 : 将汇编部分分成了四部分：
* 第一部分是==汇编指令==，指令末尾需要添加 '\\n'。
* 第二部分是==输出操作数==部分。
* 第三部分是==输入操作数==部分。
* 第四部分是==可能影响的寄存器或存储器==，用于告知编译器当前内联汇编语句可能会对某些寄存器或内存进行修改，使得编译器在优化时将其因素考虑进去。
这四部分中后三部分不是必须的。

## 示例1
```cpp
unsigned long long s_example(unsigned long long type,unsigned long long arg0) {
    unsigned long long ret_val;
    __asm__ volatile (
        "mv x10, %[type]\n"
        "mv x11, %[arg0]\n"
        "mv %[ret_val], x12"
        : [ret_val] "=r" (ret_val)
        : [type] "r" (type), [arg0] "r" (arg0)
        : "memory"
    );
    return ret_val;
}
```

示例一中指令部分，`%[type]`、`%[arg0]` 以及 `%[ret_val]` 代表着特定的寄存器或是内存。

输入输出部分中，`[type] "r" (type)`代表着将 `()` 中的变量 `type` 放入寄存器中（`"r"` 指放入寄存器，如果是 `"m"` 则为放入内存），并且绑定到 `[]` 中命名的符号中去。`[ret_val] "=r" (ret_val)` 代表着将汇编指令中 `%[ret_val]` 的值更新到变量 `ret_val`中。

## 示例2
```cpp
#define write_csr(reg, val) ({ __asm__ volatile ("csrw " #reg ", %0" :: "r"(val)); })
```

示例二定义了一个宏，其中 `%0` 代表着输出输入部分的第一个符号，即 `val`。

`#reg` 是c语言的一个特殊宏定义语法，相当于将reg进行宏替换并用双引号包裹起来。

例如 `write_csr(sstatus,val)` 经宏展开会得到：
```cpp
({
	__asm__ volatile ("csrw " "sstatus" ", %0" :: "r"(val)); })
```

![Pasted image 20230212192329.png](/img/user/OS/attachments/Pasted%20image%2020230212192329.png)

![Pasted image 20230212192354.png|400](/img/user/OS/attachments/Pasted%20image%2020230212192354.png)

# 其他

## 开辟栈区

```
stack_start:
    .rept 12
    .word 0
    .endr
stack_end:
```

**`.equ`** is like `#define` in C:
```
#define bob 10
.equ bob, 10
```
**`.word`** is like `unsigned int` in C:
```
unsigned int ted;
ted: 
.word 0
```
Or initialized with a value:
```
unsigned int alice = 42;
alice:
.word 42
```

`.rept`Repeat the sequence of lines between the .rept directive and the next .endr directive count times.
For example, assembling
```
.rept   3
.long   0
.endr
```
is equivalent to assembling
```
.long   0
.long   0
.long   0
```


```
stacks:
.skip STACK_SIZE * MAXNUM_CPU
```

[`.skip size , fill`](https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/html_chapter/as_toc.html#TOC125)

This directive emits size bytes, each of value fill. Both size and fill are absolute expressions. If the comma and fill are omitted, fill is assumed to be zero. This is the same as `.space'.