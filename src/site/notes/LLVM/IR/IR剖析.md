---
{"UID":20230227163219,"aliases":"LLVM IR最基本的程序","tags":null,"source":null,"cssclass":null,"created":"2023-02-27 16:32","updated":"2023-02-27 16:40","dg-publish":true,"permalink":"/llvm/ir/ir/","dgPassFrontmatter":true,"noteIcon":""}
---

[llvm-ir-tutorial/LLVM IR入门指南(2)——Hello world.md at master · Evian-Zhang/llvm-ir-tutorial · GitHub](https://github.com/Evian-Zhang/llvm-ir-tutorial/blob/master/LLVM%20IR%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97(2)%E2%80%94%E2%80%94Hello%20world.md)

# LLVM IR最基本的程序
一个macOS 10.15编译的IR

```
; main.ll
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.15.0"

define i32 @main() {
	ret i32 0
}
```

这个程序可以看作最简单的C语言代码：

```cpp
int main() {
	return 0;
}
```

在macOS 10.15上编译而成的结果。

# IR 解读
## 注释

首先，第一行`; main.ll`。这是一个注释。在LLVM IR中，注释以`;`开头，并一直延伸到行尾。
## 目标数据分布和平台
第二行和第三行的`target datalayout`和`target triple`，则是注明了目标汇编代码的数据分布和平台。我们之前提到过，LLVM是一个面向多平台的深度定制化编译器后端，而我们LLVM IR的目的，则是让LLVM后端根据IR代码生成相应平台的汇编代码。所以，我们需要在IR代码中指明我们需要生成哪一个平台的代码，也就是`target triple`字段。类似地，我们还需要定制数据的大小端序、对齐形式等需求，所以我们也需要指明`target datalayout`字段。关于这两个字段的值的详细情况，我们可以参考[Data Layout](http://llvm.org/docs/LangRef.html#id1248)和[Target Triple](http://llvm.org/docs/LangRef.html#id1249)这两个官方文档。我们可以对照官方文档，解释我们在macOS上得到的结果：

```
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
```

表示：
- `e`: 小端序
- `m:o`: 符号表中使用Mach-O格式的name mangling（这玩意儿我一直不知道中文是啥，就是把程序中的标识符经过处理得到可执行文件中的符号表中的符号）
- `i64:64`: 将`i64`类型的变量采用64比特的ABI对齐
- `f80:128`: 将`long double`类型的变量采用128比特的ABI对齐
- `n8:16:32:64`: 目标CPU的原生整型包含8比特、16比特、32比特和64比特
- `S128`: 栈以128比特自然对齐

```
target triple = "x86_64-apple-macosx10.15.0"
```

表示：
- `x86_64`: 目标架构为x86_64架构
- `apple`: 供应商为Apple
- `macosx10.15.0`: 目标操作系统为macOS 10.15

在一般情况下，我们都是想生成当前平台的代码，也就是说不太会改动这两个值。因此，我们可以直接写一个简单的`test.c`程序，然后使用

`clang -S -emit-llvm test.c`

生成LLVM IR代码`test.ll`，在`test.ll`中找到`target datalayout`和`target triple`这两个字段，然后拷贝到我们的代码中即可。

比方说，我在x86_64指令集的Ubuntu 20.04的机器上得到的就是：

```
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"
```

和我们在macOS上生成的代码就不太一样。

# 主程序
主程序是可执行程序的入口点，所以任何可执行程序都需要`main`函数才能运行。所以，

```
define i32 @main() {
	ret i32 0
}
```

就是这段代码的主程序。关于正式的函数、指令的定义，我会在之后的文章中提及。这里我们只需要知道，在`@main()`之后的，就是这个函数的函数体，`ret i32 0`就代表C语言中的`return 0;`。因此，如果我们要增加代码，就只需要在大括号内，`ret i32 0`前增加代码即可。
