---
{"UID":20230223202742,"aliases":"背景知识","tags":null,"source":null,"cssclass":null,"created":"2023-02-23 20:27","updated":"2023-03-13 18:52","dg-publish":true,"permalink":"/os/lab4/","dgPassFrontmatter":true,"noteIcon":""}
---

---
UID: 20230223202742
aliases: 背景知识
tags: 
source: 
cssclass: 
created: "2023-02-23 20:27"
updated: "2023-03-13 18:52"
---
[[Excalidraw/os_lab4_flow\|os_lab4_flow]]
![Pasted image 20230313185211.png](/img/user/OS/attachments/Pasted%20image%2020230313185211.png)

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


# 思考题
## 验证 `.text`, `.rodata` 段的属性是否成功设置，给出截图。

```cpp
    // mapping kernel text X|-|R|V
    printk("create mapping for text...\n");
    create_mapping(swapper_pg_dir, _stext, _stext - PA2VA_OFFSET, _etext - _stext, 
                PTE_X | PTE_R | PTE_V);
    

    // mapping kernel rodata -|-|R|V
    printk("create mapping for rodata...\n");
    create_mapping(swapper_pg_dir, _srodata, _srodata - PA2VA_OFFSET, _erodata - _srodata, 
                PTE_R | PTE_V);

    // mapping other memory -|W|R|V
    printk("create mapping for data...\n");
    create_mapping(swapper_pg_dir, _sdata, _sdata - PA2VA_OFFSET, _edata - _sdata, 
                PTE_W | PTE_R | PTE_V);
```

![Pasted image 20230302214159.png|400](/img/user/OS/attachments/Pasted%20image%2020230302214159.png)
## 为什么我们在 `setup_vm` 中需要做等值映射?
完成从物理地址到虚拟地址的平滑过渡，首先会创建VA和PA的相等映射
考虑head.S中的relocate。这里面我们设置satp。
执行中
![Pasted image 20230302220717.png|400](/img/user/OS/attachments/Pasted%20image%2020230302220717.png)
执行后，下面这些指令的地址就变成了虚拟地址。如果不设置等值映射就会找不到物理地址。
![Pasted image 20230302220742.png|400](/img/user/OS/attachments/Pasted%20image%2020230302220742.png)

## 在 Linux 中，是不需要做等值映射的。请探索一下不在 `setup_vm` 中做等值映射的方法。
Linux使用了一种叫做"高端内存"(High Memory)的技术来规避等值映射。所谓高端内存，是指物理内存地址高于内核虚拟地址空间的最大地址（例如，在32位Linux系统中，高端内存地址通常高于3GB）。Linux将这些物理内存区域映射到内核虚拟地址空间之外的专门区域，从而避免了等值映射的需要。这样做的代价是需要通过特殊的内存访问函数来访问这些高端内存区域，因为这些区域无法直接访问。  
  
在32位Linux系统中，这些高端内存区域被映射到内核虚拟地址空间之上的vmalloc区域。在64位Linux系统中，这些高端内存区域被映射到内核虚拟地址空间的后面一个特定区域。无论是哪种情况，这些区域都是由特定的内存分配器来管理的，不同于普通内核虚拟地址空间的管理方式。