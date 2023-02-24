---
{"dg-publish":true,"permalink":"/os/lab0/","dgPassFrontmatter":true,"noteIcon":""}
---

[实验指导零 - 浙江大学22年秋操作系统实验](https://zju-sec.github.io/os22fall-stu/lab0/)

# 学习内容
## GDB
[[OS/gdb常用操作\|gdb常用操作]]

## 内核

### 下载

使用国内镜像

[Linux 内核源码镜像使用帮助 — USTC Mirror Help 文档](http://mirrors.ustc.edu.cn/help/linux.git.html)

### 编译

```C++
$ cd path/to/linux
$ make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- defconfig    # 使用默认配置
$ make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)   # 编译
```

### 运行

```Shell
:~/clmiao/os22fall-stu/src/lab0$ qemu-system-riscv64 -nographic -machine virt -kernel ~/clmiao/linux/arch/riscv/boot/Image \
    -device virtio-blk-device,drive=hd0 -append "root=/dev/vda ro console=ttyS0" \
    -bios default -drive file=rootfs.img,format=raw,id=hd0 -S -s
```

![Pasted image 20230212184516.png](/img/user/OS/attachments/Pasted%20image%2020230212184516.png)
![Pasted image 20230212184523.png](/img/user/OS/attachments/Pasted%20image%2020230212184523.png)
### 调试

编译时没有生成debug信息，无法调试
![Pasted image 20230212184542.png](/img/user/OS/attachments/Pasted%20image%2020230212184542.png)
使用`make ARCH=riscv menuconfig` 进行debug信息的config

# 实验结果

## 使用 `riscv64-linux-gnu-gcc` 编译单个 `.c` 文件

-   code

```C++
#include <stdio.h>

int main() {
    printf("Hello World\n");
}
```

-   compile

```C++
$ riscv64-linux-gnu-gcc code.c -o riscv_out 
```

## 使用 `riscv64-linux-gnu-objdump` 反汇编 1 中得到的编译产物

```C++
$ riscv64-linux-gnu-objdump -d riscv_out > code_asm
```
![Pasted image 20230212184552.png](/img/user/OS/attachments/Pasted%20image%2020230212184552.png)
## 调试 Linux 时:

```C++
$ gdb-multiarch vmlinux
(gdb) target remote :1234
```

-   在 0x80000000 处下断点

```Kotlin
(gdb) break *0x80000000
Breakpoint 3 at 0x80000000
(gdb) continue //运行到断点
```
![Pasted image 20230212184601.png](/img/user/OS/attachments/Pasted%20image%2020230212184601.png)
-   在 GDB 中查看汇编代码

```Kotlin
(gdb) info r pc
pc             0x80000000       0x80000000
(gdb) layout asm
```
![Pasted image 20230212184606.png](/img/user/OS/attachments/Pasted%20image%2020230212184606.png)
-   查看所有已下的断点

```Kotlin
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000080000000
        breakpoint already hit 1 time
```

-   在 0x80200000 处下断点

```Kotlin
(gdb) b *0x80200000
Breakpoint 2 at 0x80200000
(gdb) info b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000080000000
        breakpoint already hit 1 time
2       breakpoint     keep y   0x0000000080200000
```

-   清除 0x80000000 处的断点
![Pasted image 20230212184613.png](/img/user/OS/attachments/Pasted%20image%2020230212184613.png)
-   继续运行直到触发 0x80200000 处的断点
![Pasted image 20230212184620.png](/img/user/OS/attachments/Pasted%20image%2020230212184620.png)

-   单步调试一次

```Kotlin
(gbd) stepi
```
![Pasted image 20230212184643.png](/img/user/OS/attachments/Pasted%20image%2020230212184643.png)
-   退出 QEMU

## `vmlinux` 和 `Image` 的关系和区别是什么？

vmlinux是一个ELF文件，上百M，无法直接flash到板子上。不同架构最终生成的启动镜像略有区别，一般地：

-   通过编译生成`vmlinux`和`System.map`
    
-   通过`objcopy`移除`vmlinux`中不必要段，输出binary格式Image
    
-   再对Image进行压缩，输出不同格式的压缩文件，比如gzip对应的`Image.gz`，
    
-   最后通过工具加上BootLoader可以识别的header用于启动引导。
    
![Pasted image 20230212184652.png](/img/user/OS/attachments/Pasted%20image%2020230212184652.png)
[ref1](https://zhuanlan.zhihu.com/p/466226177) [ref2](https://www.zhihu.com/question/501955271/answer/2705153692)