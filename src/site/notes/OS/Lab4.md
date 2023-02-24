---
{"UID":20230223202742,"aliases":"背景知识","tags":null,"source":null,"cssclass":null,"created":"2023-02-23 20:27","updated":"2023-02-23 20:39","dg-publish":true,"permalink":"/os/lab4/","dgPassFrontmatter":true,"noteIcon":""}
---


# 背景知识
## Kernel 的虚拟内存布局

```
start_address           end_address
    0x0                 0x3fffffffff
     │                       │
┌────┘                 ┌─────┘
↓        256G          ↓                                
┌───────────────────────┬──────────┬────────────────┐
│      User Space       │    ...   │  Kernel Space  │
└───────────────────────┴──────────┴────────────────┘
                                    ↑    256G      ↑
                      ┌─────────────┘              │ 
                      │                            │
              0xffffffc000000000          0xffffffffffffffff
                start_address                 end_address
```

通过上图我们可以看到 RV64 将 `0x0000004000000000` 以下的虚拟空间作为 `user space`。将 `0xffffffc000000000` 及以上的虚拟空间作为 `kernel space`。

详细内存布局
![Pasted image 20230223203052.png](/img/user/OS/attachments/Pasted%20image%2020230223203052.png)

> 在 `RISC-V Linux Kernel Space` 中有一段区域被称为 `direct mapping area`，为了方便 kernel 可以高效率的访问 RAM，kernel 会预先把所有物理内存都映射至这一块区域 ( PA + OFFSET == VA )， 这种映射也被称为 `linear mapping`。在 RISC-V Linux Kernel 中这一段区域为 `0xffffffe000000000 ~ 0xffffffff00000000`, 共 124 GB

## RISC-V Virtual-Memory System (Sv39)
### `satp`  Register（Supervisor Address Translation and Protection Register）

```
 63      60 59                  44 43                                0
 ---------------------------------------------------------------------
|   MODE   |         ASID         |                PPN                |
 ---------------------------------------------------------------------
```

ASID (地址空间标识符) 域是可选的，它用于帮助地址空间的转换，降低上下文切换的开销。PPN 存储了根页表的物理页号，以 4 KiB 页面大小为单位，它在内存分页中起了十分重要的作用。MODE 位用于选择地址转换的方式.
* Mode字段的取值

```
                         RV 64
     ----------------------------------------------------------
    |  Value  |  Name  |  Description                          |
    |----------------------------------------------------------|
    |    0    | Bare   | No translation or protection          |
    |  1 - 7  | ---    | Reserved for standard use             |
    |    8    | Sv39   | Page-based 39 bit virtual addressing  | <-- 我们使用的mode
    |    9    | Sv48   | Page-based 48 bit virtual addressing  |
    |    10   | Sv57   | Page-based 57 bit virtual addressing  |
    |    11   | Sv64   | Page-based 64 bit virtual addressing  |
    | 12 - 13 | ---    | Reserved for standard use             |
    | 14 - 15 | ---    | Reserved for standard use             |
     -----------------------------------------------------------
```

- ASID ( Address Space Identifier ) ： 此次实验中直接置 0 即可。
- PPN ( Physical Page Number ) ：顶级页表的物理页号。我们的物理页的大小为 4KB， PA >> 12 == PPN。

### RISC-V Sv39 Virtual Address and Physical Address

```
     38        30 29        21 20        12 11                           0
     ---------------------------------------------------------------------
    |   VPN[2]   |   VPN[1]   |   VPN[0]   |          page offset         |
     ---------------------------------------------------------------------
                            Sv39 virtual address
```

```
 55                30 29        21 20        12 11                           0
 -----------------------------------------------------------------------------
|       PPN[2]       |   PPN[1]   |   PPN[0]   |          page offset         |
 -----------------------------------------------------------------------------
                            Sv39 physical address
```

Sv39 模式定义物理地址有 56 位，虚拟地址有 64 位。但是，虚拟地址的 64 位只有低 39 位有效。通过虚拟内存布局图我们可以发现，其 63-39 位为 0 时代表 user space address， 为 1 时 代表 kernel space address。 * Sv39 支持三级页表结构，VPN[2-0](https://zju-sec.github.io/os22fall-stu/lab4/Virtual%20Page%20Number)分别代表每级页表的`虚拟页号`，PPN[2-0](https://zju-sec.github.io/os22fall-stu/lab4/Physical%20Page%20Number)分别代表每级页表的`物理页号`。物理地址和虚拟地址的低12位表示页内偏移（page offset）。

### RISC-V Sv39 Page Table Entry

```
 63      54 53        28 27        19 18        10 9   8 7 6 5 4 3 2 1 0
 -----------------------------------------------------------------------
| Reserved |   PPN[2]   |   PPN[1]   |   PPN[0]   | RSW |D|A|G|U|X|W|R|V|
 -----------------------------------------------------------------------
                                                     |   | | | | | | | |
                                                     |   | | | | | | | `---- V - Valid
                                                     |   | | | | | | `------ R - Readable
                                                     |   | | | | | `-------- W - Writable
                                                     |   | | | | `---------- X - Executable
                                                     |   | | | `------------ U - User
                                                     |   | | `-------------- G - Global
                                                     |   | `---------------- A - Accessed
                                                     |   `------------------ D - Dirty (0 in page directory)
                                                     `---------------------- Reserved for supervisor software
```

- 0 ～ 9 bit: protection bits
    - V : 有效位，当 V = 0, 访问该 PTE 会产生 Pagefault。
    - R : R = 1 该页可读。
    - W : W = 1 该页可写。
    - X : X = 1 该页可执行。
    - U , G , A , D , RSW 本次实验中设置为 0 即可。

### RISC-V Address Translation

```
```