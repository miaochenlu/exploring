---
{"aliases":"基本调试流程","tags":null,"cssclass":null,"source":null,"dg-publish":true,"created":"2023-02-17 22:46","updated":"2023-02-24 20:29","permalink":"/os/gdb/","dgPassFrontmatter":true,"noteIcon":""}
---

# 基本调试流程

- 重新编译程序并在编译选项中加入“-g”

```C++
$ gcc -g test.c
```

- 运行 gdb 和程序

```C++
$ gdb a.out
```

- 设置断点

```C++
 (gdb) b 6
```

- 运行程序

```C++
(gdb) r
```

- 程序暂停在断点处，执行查看

```C++
(gdb) p xxx
```

- 继续、单步或者恢复程序运行

```C++
(gdb) s/n/c
```

# 概览

- `(gdb) layout asm`: 显示汇编代码
    
- ctrl+x+a关闭显示
    
- `(gdb) start`: 单步执行，运行程序，停在第一执行语句
    
- `(gdb) continue`: 从断点后继续执行，简写 `c`
    
- `(gdb) next`: 单步调试（逐过程，函数直接执行），简写 `n`
    
- `(gdb) step instruction`: 执行单条指令，简写 `si`
    
- `(gdb) run`: 重新开始运行文件（run-text：加载文本文件，run-bin：加载二进制文件），简写 `r`
    
- `(gdb) backtrace`：查看函数的调用的栈帧和层级关系，简写 `bt`
    
- `(gdb) break` 设置断点，简写 `b`
    
- 断在 `foo` 函数：`b foo`
    
- 断在某地址: `b * 0x80200000`
    
- `(gdb) finish`: 结束当前函数，返回到函数调用点
    
- `(gdb) frame`: 切换函数的栈帧，简写 `f`
    
- `(gdb) print`: 打印值及地址，简写 `p`
    
- `(gdb) info`: 查看函数内部局部变量的数值，简写 `i`
    
- 查看寄存器 ra 的值: `i r ra`
    
- `(gdb) display`: 追踪查看具体变量值
    
- `(gdb) x/nfu addr`: 以`f`格式打印从`addr`开始的`n`个长度单元为`u`的内存值
    

# 关于断点

在gdb中设置断点的方法有以下几种：

1. 设置函数断点：您可以在gdb命令行中使用“break function”命令，其中“function”是您想设置断点的函数名称。例如：

```Kotlin
(gdb) break main
```

这将在“main”函数的开头处设置一个断点。每当执行到该函数时，gdb都会暂停并进入调试状态。

1. 设置行断点：您可以在gdb命令行中使用“break filename:line”命令，其中“filename”是您要调试的文件名，“line”是您想设置断点的行号。例如：

```Kotlin
(gdb) break main.c:10
```

这将在“main.c”文件的第10行设置一个断点。每当执行到该行时，gdb都会暂停并进入调试状态。

1. 设置地址断点：您可以在gdb命令行中使用 **“break *address”** 命令，其中“address”是您想设置断点的内存地址。例如：

```Kotlin
(gdb) break *0xffffffff8100b000
```

这将在内存地址0xffffffff8100b000处设置一个断点。每当执行到该地址时，gdb都会暂停并进入调试状态。

## gdb中断点相关的操作

1. 显示断点 使用“info breakpoints”命令可以显示当前设置的所有断点。每个断点都会列出其编号、位置、是否启用以及其他相关信息。
    
2. 启用/禁用断点 使用“enable breakpoints”和“disable breakpoints”命令可以启用或禁用断点。例如，如果您想禁用第1个断点，您可以使用以下命令：

```Kotlin
(gdb) disable 1
```

1. 删除断点 使用“delete breakpoints”命令可以删除断点。例如，如果您想删除第1个断点，您可以使用以下命令：

```Kotlin
(gdb) delete 1
```

1. 修改断点 使用“condition”选项可以修改断点的条件。例如，如果您想让第1个断点仅在变量“i”的值等于100时才停止，您可以使用以下命令：

```Kotlin
(gdb) condition 1 i == 100
```

# 远程调试

![[Pasted image 20230211202616.png\|Pasted image 20230211202616.png]] 以Qemu调试为例 **qemu侧** `qemu-system-riscv64 ... -kernel ./test.elf -s -S`

- -s: “-gdb tcp::1234” 的缩写，启动gdbserver并在1234 端口号上监听客户端
    
- -S: 在启动时停止CPU (只有到在客户端键入'c' 才会开始执行) **本地gdb侧**

```C++
$ gdb test.elf
(gdb) target remote :1234
```

# gdbinit

gdbinit是一个名为.gdbinit的特殊文件，它包含用于配置和自动执行gdb命令的脚本。当您启动gdb时，它会自动读取并执行gdbinit中的命令。 以下是一个使用gdbinit调试QEMU启动的内核的示例：

1. 创建gdbinit文件：在您的主目录下创建一个名为.gdbinit的文件。
    
2. 编辑gdbinit文件：使用文本编辑器打开gdbinit文件，并输入以下命令：

```C++
file ~/clmiao/linux/vmlinux
target remote localhost:1234
break start_kernel
```

1. 保存并关闭gdbinit文件：保存更改并关闭gdbinit文件。
    
2. 启动QEMU：使用以下命令启动QEMU，并在内核启动时自动断点：

```C++
qemu-system-riscv64 -nographic -machine virt \
    -kernel ~/clmiao/linux/arch/riscv/boot/Image \
    -device virtio-blk-device,drive=hd0 -append "root=/dev/vda ro console=ttyS0" \     
    -bios default -drive file=rootfs.img,format=raw,id=hd0 -S -s
```

1. 启动gdb：启动gdb，并使用以下命令连接到QEMU：

```C++
$ gdb
```

在gdb启动时，gdbinit文件将自动读取并执行，并在内核启动时自动断点。此时，您可以使用gdb命令调试内核。