---
{"UID":20230213210113,"aliases":["Experiment Flow","思考题"],"tags":["OS"],"source":null,"cssclass":null,"created":"2023-02-15 18:50","updated":"2023-02-16 20:52","dg-publish":true,"permalink":"/os/lab2/","dgPassFrontmatter":true,"noteIcon":""}
---


# Experiment Flow
![](/img/user/OS/attachments/Pasted image 20230216202855.png)
[os_lab2_flow](os_lab2_flow.md)

---
# RISCV中断异常
[RISCV中断异常](RISCV中断异常.md)

---
# 思考题
在我们使用make run时， OpenSBI 会产生如下输出:

```
    OpenSBI v0.9
     ____                    _____ ____ _____
    / __ \                  / ____|  _ \_   _|
   | |  | |_ __   ___ _ __ | (___ | |_) || |
   | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
   | |__| | |_) |  __/ | | |____) | |_) || |_
    \____/| .__/ \___|_| |_|_____/|____/_____|
          | |
          |_|

    ......

    Boot HART MIDELEG         : 0x0000000000000222
    Boot HART MEDELEG         : 0x000000000000b109

    ......
```

通过查看 `RISC-V Privileged Spec` 中的 `medeleg` 和 `mideleg` 解释上面 `MIDELEG` 值的含义。
`medeleg` 和 `mideleg` 中的每一位对应 `mcause` 中的每一位

---

MIDELEG -> 0x0222 = 0000 0010 0010 0010
也就是1号，5号，9号中断委托给supervisor, 分别是supervisor software interrupt, supervisor timer interrupt, supervisor external interrupt
![Pasted image 20230216200710.png|300](/img/user/OS/attachments/Pasted%20image%2020230216200710.png)
MEDELEG -> 0xb109 = 1011 0001 0000 1001
也就是0号，3号，8号，12号，13号，15号异常委托给supervisor, 分别是
* 0: instruction address misaligned
* 3: breakpoint
* 8: environment call from U-mode
* 12: instruction page fault
* 13: load page fault
* 15: store/amo page fault
![Pasted image 20230216201017.png|300](/img/user/OS/attachments/Pasted%20image%2020230216201017.png)
