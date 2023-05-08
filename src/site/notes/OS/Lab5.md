---
{"UID":20230313183923,"aliases":"背景知识,Lab5","tags":null,"source":null,"cssclass":null,"created":"2023-03-13 18:39","updated":"2023-03-13 19:22","dg-publish":true,"permalink":"/os/lab5/","dgPassFrontmatter":true,"noteIcon":""}
---



# 背景知识

## 1. sstatus[SUM] PTE[U]

* 当页表项 PTE[U] 置 0 时，该页表项对应的内存页为内核页，运行在 U-Mode 下的代码**无法访问**。
* 当页表项 PTE[U] 置 1 时，该页表项对应的内存页为用户页，运行在 S-Mode 下的代码**无法访问**。
* 如果想让 S 特权级下的程序能够访问用户页，需要对 sstatus[SUM] 位置 1 。但是无论什么样的情况下，用户页中的指令对于 S-Mode 而言都是**无法执行**的。

## 2. 用户态栈与内核态栈

当用户态程序在用户态运行时，其使用的栈为**用户态栈**，当调用 SYSCALL时候，陷入内核处理时使用的栈为**内核态栈**，因此需要区分用户态栈和内核态栈，并在异常处理的过程中需要对栈进行切换

![Pasted image 20230313185334.png](/img/user/OS/attachments/Pasted%20image%2020230313185334.png)

# 实验步骤
## 1. 创建用户态进程
为每个用户态进程新增
* sepc, sstatus, scratch
	* sepc存用户程序起始地址 (_entry)
	* sstatus存状态，是否开启SUM, 
* 页表。由于多个用户态进程需要保证相对隔离，因此不可以共用页表。我们为每个用户态进程都创建一个页表。

```cpp
// proc.h 

typedef unsigned long* pagetable_t;

struct thread_struct {
    ...
    uint64_t sepc, sstatus, sscratch; 
};

struct task_struct {
    ...
    struct thread_struct thread;

    pagetable_t pgd;
};
```

在`task_init`中修改

```cpp
void task_init() {
	...

    for(int i = 1; i < NR_TASKS; i++) {
        ...
        // 为用户栈分配空间
        uint64* user_stack = (uint64*)alloc_page();
		// 为用户页表分配空间
        pte_t* user_pgtable = (pte_t*)alloc_page();
		/** 为了避免 `U-Mode` 和 `S-Mode` 切换的时候切换页表
		  复制kernel page table的内容
		**/
        for(int i = 0; i < 512; i++) {
            user_pgtable[i] = swapper_pg_dir[i];
        }
		// 为uapp和用户栈做映射
        create_mapping(user_pgtable, USER_START, (uint64)uapp_start - PA2VA_OFFSET, 
                        uapp_end - uapp_start, PTE_V | PTE_R | PTE_W | PTE_X | PTE_U);
        create_mapping(user_pgtable, USER_END-PGSIZE, (uint64)user_stack - PA2VA_OFFSET, 
                        PGSIZE, PTE_V | PTE_R | PTE_W | PTE_U);
        /**修改好 `sstatus` 中的 `SPP` （ 使得 sret 返回至 U-Mode ）， 
        `SPIE` （ sret 之后开启中断 ）， `SUM` （ S-Mode 可以访问 User 页面 ）
        **/
        uint64 sstatus = csr_read(sstatus);
        task[i]->thread.sstatus     = sstatus | 0x40020;
        // 将 `sepc` 修改为 `USER_START`, 以便跳转到用户态entry point
        task[i]->thread.sepc        = USER_START;
        /** `sscratch` 设置为 `U-Mode` 的 sp，其值为 `USER_END` 
        （即 `U-Mode Stack` 被放置在 `user space` 的最后一个页面
        **/
        task[i]->thread.sscratch    = USER_END;
        // 注意va和pa
        task[i]->pgd = (uint64)user_pgtable - PA2VA_OFFSET;

    }

    printk("...proc_init done!\n");
}
```

修改`__switch_to`
加入 保存/恢复 `sepc` `sstatus` `sscratch` 以及 切换页表的逻辑。在切换了页表之后，需要通过 `fence.i` 和 `vma.fence` 来刷新 TLB 和 ICache

```assembly
    .globl __switch_to
__switch_to:
    # save state to prev process
    # YOUR CODE HERE

	# thread struct 5 * 8 --> a0
	sd ra, 40(a0)
	...
	sd s11, 144(a0)

	# sepc, sstatus, scratch
	csrr t0, sepc
	sd t0, 152(a0)
	csrr t0, sstatus
	sd t0, 160(a0)
	csrr t0, sscratch
	sd t0, 168(a0)
	# csrr t0, satp
	# sd t0, 176(a0)

	# set satp 
    ld t0, 176(a1)
    srli t0, t0, 12 
    li t1, 1
    slli t1, t1, 63
    or t0, t0, t1
    csrw satp, t0
    sfence.vma zero, zero
    
	# sepc, sstatus, scratch
	ld t0, 152(a1)
	csrw sepc, t0
	ld t0, 160(a1)
	csrw sstatus, t0
	ld t0, 168(a1)
	csrw sscratch, t0
	# ld t0, 176(a1)
	# csrw satp, t0
	# sfence.vma zero, zero

    # restore state from next process
	ld ra, 40(a1)
	...
	ld s11, 144(a1)

    ret
```

## 2. 修改中断入口/返回逻辑 ( \_trap ) 以及中断处理函数(trap_handler）

- RISC-V 中只有一个栈指针寄存器( sp )，因此需要我们来完成用户栈与内核栈的切换。
- 由于我们的用户态进程运行在 `U-Mode` 下， 使用的运行栈也是 `U-Mode Stack`， 因此当触发异常时， 我们首先要对栈进行切换 （ `U-Mode Stack` -> `S-Mode Stack` ）。同理 让我们完成了异常处理， 从 `S-Mode` 返回至 `U-Mode`， 也需要进行栈切换 （ `S-Mode Stack` -> `U-Mode Stack` ）。

SP: 当前栈指针
sscratch: 要切换的栈指针

- 修改 `__dummy`。， `thread_struct.sp` 保存了 `S-Mode sp`， `thread_struct.sscratch` 保存了 `U-Mode sp`， 因此在 `S-Mode -> U->Mode` 的时候，我们只需要交换对应的寄存器的值即可。
- 修改 `_trap` 。同理 在 `_trap` 的首尾我们都需要做类似的操作。**注意如果是 内核线程( 没有 U-Mode Stack ) 触发了异常，则不需要进行切换。（内核线程的 sp 永远指向的 S-Mode Stack， sscratch 为 0）**

其余较为直观，略去

```
_traps:
	csrrw sp, sscratch,sp
	bne sp, t0, FROM_USER
	csrrw sp, sscratch,sp
FROM_USER:
        addi sp, sp, -272
		csrr t1, sstatus
        sd t1, 264(sp)
        csrr t1, sepc
        sd t1, 256(sp)

        sd x31, 248(sp)
        ...    
        sd x0, 0(sp) // no use

        # 2. call trap_handler
        csrr a0, scause
        csrr a1, sepc
        mv a2, sp
        call trap_handler

        # 3. restore sepc, sstatus and 32 registers (x2(sp) should be restore last) from stack

        # ld t2, 264(sp) # problem!
        ld t1, 264(sp)
        csrw sstatus, t1
        ld t1, 256(sp)
        csrw sepc, t1
        ld x31, 248(sp)
        ...
        ld x1, 8(sp)
        addi sp, sp, 272

	csrrw sp, sscratch,sp
	bne sp, t0, END
	csrrw sp, sscratch,sp
END:
    sret

	.globl __dummy
__dummy:
	csrrw sp, sscratch, sp
	la t0, 0
	csrw sepc, t0

	sret
```

## 3. 添加ELF支持
ELF header的信息

```
Elf64_Ehdr   // 你可以将 uapp_start 强制转化为改类型的指针，
                然后把那一块内存当成此类结构体来读其中的数据，其中包括：
    e_ident  // Magic Number, 你可以通过这个域来检测自己是不是真的正在读一个 Ehdr,
                值一定是 7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
    e_entry  // 程序的第一条指令被存储的用户态虚拟地址
    e_phnum  // ELF 文件包含的 Segment 的数量
    e_phoff  // ELF 文件包含的 Segment 数组相对于 Ehdr 的偏移量

Elf64_Phdr   // 存储了程序各个 Segment 相关的 metadata
             // 你可以将 uapp_start + e_phoff 强制转化为此类型，就会指向第一个 Phdr,
             // uapp_start + e_phoff + 1 * sizeof(Elf64_Phdr), 则指向第二个‘
    p_filesz // Segment 在文件中占的大小
    p_memsz  // Segment 在内存中占的大小
    p_vaddr  // Segment 起始的用户态虚拟地址
    p_offset // Segment 在文件中相对于 Ehdr 的偏移量
    p_type   // Segment 的类型 
    p_flags  // Segment 的权限（包括了读、写和执行）
```

修改proc.c

```cpp
static uint64_t load_program(struct task_struct* task) {
    pte_t* user_pgtable = (pte_t*)alloc_page();
    for(int i = 0; i < 512; i++) {
        user_pgtable[i] = swapper_pg_dir[i];
    }
    task->pgd = (uint64*)((uint64)user_pgtable - PA2VA_OFFSET);

    Elf64_Ehdr* ehdr = (Elf64_Ehdr*)uapp_start;

    uint64_t phdr_start = (uint64_t)ehdr + ehdr->e_phoff;
    uint64_t phdr_cnt = ehdr->e_phnum;

    Elf64_Phdr* phdr;
    int load_phdr_cnt = 0;

    for (int i = 0; i < phdr_cnt; i++) {
        phdr = (Elf64_Phdr *)(phdr_start + sizeof(Elf64_Phdr) * i);
        if (phdr->p_type == PT_LOAD) {
            uint64 page_count = PGROUNDUP(phdr->p_memsz) / PGSIZE;
            uint64 valloc_addr = alloc_pages(page_count);
            uint64 load_addr = (uint64)uapp_start + phdr->p_offset;
            create_mapping(user_pgtable, USER_START, valloc_addr - PA2VA_OFFSET,
                           phdr->p_memsz, phdr->p_flags | PTE_X);
            memcpy((uint64*)valloc_addr, (uint64*)load_addr, phdr->p_memsz);
        }
    }

    uint64* user_stack = (uint64*)alloc_page();
    create_mapping(user_pgtable, USER_END-PGSIZE, (uint64)user_stack - PA2VA_OFFSET, 
                    PGSIZE, PTE_V | PTE_R | PTE_W | PTE_U);


    uint64 sstatus          = csr_read(sstatus);
    task->thread.sstatus    = sstatus | 0x40020;
    task->thread.sepc       = ehdr->e_entry;
    task->thread.sscratch   = USER_END;

    return user_pgtable;
}

void task_init() {
    ...

    for(int i = 1; i < NR_TASKS; i++) {
        task[i] = (struct task_struct*)kalloc();
        task[i]->state = TASK_RUNNING;
        task[i]->counter = 0;
        task[i]->priority = rand();
        task[i]->pid = i;
        task[i]->thread.ra = (uint64)__dummy;
        task[i]->thread.sp = (uint64)(task[i]) + PGSIZE;
        
        load_program(task[i]);
    }

    printk("...proc_init done!\n");
}
```

# 思考题
## 1. 为什么 Phdr 中，`p_filesz` 和 `p_memsz` 是不一样大的？
`p_filesz` 表示在 ELF 文件中该段（section）的大小，`p_memsz` 表示该段在内存中的大小。
通常，`p_filesz` 的值等于或小于 `p_memsz` 的值。这是因为在文件中可能存在填充字节或未初始化的数据（.BSS段）等，这些数据在内存中不需要占用空间，因此 `p_filesz` 会小于 `p_memsz`。此外，在程序执行时，一些段的大小可能会被动态地更改，因此 `p_memsz` 可能会大于 `p_filesz`

BSS 段是程序中未初始化的全局变量和静态变量所占用的内存空间，它在 ELF 文件中是以一个大小为 0 的段表示的。当程序被加载到内存中时，操作系统会根据程序的需要在内存中分配 BSS 段所需的空间，并将其初始化为 0。因此，BSS 段在文件中并不占用空间，但在内存中会占用一定的空间。

## 2. ELF中的segment和section
在 ELF 文件格式中，有两个重要的概念，即“section”和“segment”。
- Section（节）是 ELF 文件中一个逻辑单位的组织方式，它用于描述代码、数据、符号表、重定位信息等等。每个 Section 都有一个名字和一个类型，它们被存储在 Section Header Table 中，这个表本身也是一个 Section。Section 在文件中的排列通常是任意的，不一定要按照内存布局的方式排列。
    
- Segment（段）是在程序运行时，操作系统为了加载程序而将文件中的一部分按照一定的方式映射到内存中的一个连续的区域。一个段可以包含多个 Section，而一个 Section 只能属于一个段。每个 Segment 都有一个类型和一组属性，它们被存储在 Program Header Table 中，这个表本身也是一个 Segment。Segment 在文件中的排列通常是按照内存布局的方式排列，例如代码段、数据段、堆栈段等等。
    
简单来说，Section 是 ELF 文件中的逻辑单位，用于描述文件本身的组织结构；而 Segment 是程序运行时的内存映射，用于描述程序在内存中的布局。