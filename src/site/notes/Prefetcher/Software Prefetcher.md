---
{"aliases":"Profile-guided Optimization Overview","tags":null,"cssclass":null,"source":null,"created":"2023-02-22 13:41","updated":"2023-02-28 13:38","dg-publish":true,"permalink":"/prefetcher/software-prefetcher/","dgPassFrontmatter":true,"noteIcon":""}
---

# [Profile-guided Optimization](https://en.wikipedia.org/wiki/Profile-guided_optimization)(PGO) Overview
## A. Overall Process
Step 1: Running & Profiling
Step 2: Analysis. 
	The profiling and tracing data as well as the unmodified binaries are then fed into a tool
Step 3: Injection
![Pasted image 20230223232525.png|500](/img/user/Prefetcher/attachments/Pasted%20image%2020230223232525.png)

## B. Profiling & Tracing Tools
| tools                               | --                  |
| ----------------------------------- | ------------------- |
| VTune                               | instruction mix     |
| PMU counters                        | instruction mix     |
| PEBS (precise event-based sampling) | execution frequency, AMAT,  | 
| LBR (last branch record)            |                     |
| PT (processor trace)                | execution frequency |

## C. Representative work
[Heiner Litz](https://people.ucsc.edu/~hlitz/)
* [MICRO'22 Best Paper] Whisper: Profile-Guided <font color="#2DC26B">Branch Misprediction</font> Elimination for Data Center Applications
* [ISCA'22] Thermometer: Profile-Guided <font color="#2DC26B">BTB Replacement</font> for Data Center Applications
* [EuroSys'22] APT-GET: Profile-Guided Timely Software <font color="#2DC26B">Prefetching</font>
* [ASPLOS'22] CRISP: Critical Slice <font color="#2DC26B">Prefetching</font>
* [MICRO'21] Twig: Profile-Guided <font color="#2DC26B">BTB Prefetching</font> for Data Center Applications
* [ISCA'21] Ripple: profile-guided <font color="#2DC26B">instruction cache replacement</font> for data center applications
* [MICRO'20] I-SPY: Context-Driven <font color="#2DC26B">Conditional Instruction Prefetching</font> with Coalescing
* [ASPLOS'20] Classifying Memory Access Patterns for <font color="#2DC26B">Prefetching</font>
* [ISCA'19] AsmDB: understanding and mitigating <font color="#2DC26B">front-end stalls</font> in warehouse-scale computers
* [CGO'19] BOLT: A Practical Binary Optimizer for Data Centers and Beyond
* [CGO'16] AutoFDO: automatic feedback-directed optimization for warehouse-scale applications
---

# Classifying Memory Access Patterns
This approach executes the target binracy once to obtain execution traces and miss profiles, performs dataflow analysis, and finally injects prefetches into a new binary.

![Pasted image 20230223133644.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230223133644.png)
[[Excalidraw/classify\|classify]]

---
## Memory Access Pattern -- The Recurrence Relation
$$A_n=f(A_{n-1})$$
* $A$ is a memory address 
* $n$ represents the nth execution of a particular load instruction 
* $f (x)$ is an arbitrary function. The complexity of $f (x)$ determines the capabilities a prefetcher requires to predict and prefetch a certain future cache miss.
---
| Pattern                        | Recurrence relation               | Example                    | Note                                                                             |
| ------------------------------ | --------------------------------- | -------------------------- | -------------------------------------------------------------------------------- |
| Constant                       | $A_n=A_{n-1}$                     | \*ptr                      |                                                                                  |
| Delta                          | $A_n=A_{n-1}+d$                   | streaming, array traversal | d=stride                                                                         |
| Pointer Chase                  | $A_n=Ld(A_{n-1})$                 | next=current->next         | load addr is derived from the value of prev load                                 |
| Indirect Delta                 | $A_n=Ld(B_{n-1}+d)$               | \*(M[i])                   |                                                                                  |
| Indirect Index                 | $A_n=Ld(B_{n-1+c}+d)$             | M\[N\[i]]                  | c=base addr of M, d=stride                                                       |
| Constant plus offset reference | $A_n=B_n+c_1,  B_n=Ld(B_{n-1}+c2)$ | Linked list traversal      | $c_1$=data offset $c_2$=next pointer offset $B_n$=address of the $n^{th}$ struct |

---
To formalize prefetch patterns, define prefetch kernel: 

> For a given load instruction and its function to compute the next address, $f(x)$ can be expressed as a ==prefetch kernel== which consists of a number of <font color="#2DC26B">data sources</font> and <font color="#2DC26B">operations</font> on those sources which generate the delinquent load address.

* <font color="#2DC26B">Data source types</font>
	* constants
	* registers
	* memory locations
* <font color="#2DC26B">Operations</font>
	* instruction or micro-op specified by tehe ISA

---
## Extract Prefetch Kernel
| Machine Classification | Corresponding Pattern            |
| ---------------------- | -------------------------------- |
| Constant               | Constant                         |
| Add                    | Delta                            |
| Add, Multiply          | Complex                          |
| Load, Add              | Linked List, Indirect Index, ... |
| Load, Add, Multiply    | Complex                          | 

---
### Preparation
* program & library binaries
* application trace: PC as well as the memory addr of all load and store insts
* cache miss profile: a ranked list of PCs that cause cache misses

---
### Process
* Dataflow Analysis
	* Locate the execution history window (from miss PC back to the same PC)
	* Build data dependencies
* Compaction
	* store-load bypassing
	* arithmetic distribution and compaction
	* assignment and zero pruning

---
## Analysis

![Pasted image 20230223112632.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230223112632.png)

---
Findings:
* Indirection使得spatial prefetcher效果变差
* 很大一部分程序虽然没有Indirection, 但是reuse distance很大
* simple kernel可能也需要复杂计算
* pattern数量很多

---
# CRISP: Critical Slice Prefetching
---
## Overview
![Pasted image 20230223232525.png|500](/img/user/Prefetcher/attachments/Pasted%20image%2020230223232525.png)

---
## Motivation
Reordering instructions --> critical first

![Pasted image 20230223144004.png|600](/img/user/Prefetcher/attachments/Pasted%20image%2020230223144004.png)

What is CRISP:
> CRISP is a a lightweight mechanism to hide the high latency of irregular memory access patterns by leveraging <font color="#2DC26B">criticality-based scheduling</font>. 
> CRISP executes <font color="#2DC26B">delinquent loads</font> and their load slices as early as possible, hiding a significant fraction of their latency. Furthermore, we observe that the latency induced by <font color="#2DC26B">branch mispredictions</font> and <font color="#2DC26B">other high latency instructions</font> can be hidden with a similar approach.

---

Two Tasks:
* Task1: <font color="#2DC26B">Identifying cirtical instruction</font>. CRISP identifies high-latency load instructions that frequently induce pipeline stalls due to cache misses and <font color="#2DC26B">tracks their load-address-generating instructions</font> (slices).
* Task2: <font color="#2DC26B">Tagging and prioritizing critical instruction's execution</font>. By tagging these instructions as critical and prioritizing their execution, the instruction scheduler can hide a large fraction of the memory access latency, improve memory-level parallelism (MLP) and overall performance.
---

## Task 1: Instruction Criticality
---
### Step1: Determining Delinquent Loads
#### A. What is a critical load
Define: A load is critical if
* its LLC miss rate is higher than a threshold (e.g. 20%)
* its memory addr cannot be easily predicted by the hardware prefetcher
* the number of independent instructions behind the load in the sequential instruction stream is small
---
There are other factors, e.g.
* the load's execution ratio over other loads in the program
* the LLC miss rate of the load
* the pipeline stalls induced by the load
* the baseline IPC and instruction mix of a program
* the MLP of the program at the time where the load occurs
* the time a load becomes ready to be scheduled, determined by its dependency on other high latency instructions
---
#### B. How to obtain these info
How to collect thses info
* IPC: directly measured
* instruction mix: VTune or PMU counters
* the execution frequency of a specific load: Precise event-based sampling (PEBS), Processor trace (PT)
* A load' average memory access time: PEBS
* the pipeline stalls induced by a load and MLP: observing precise back-end stalls and load queue occupancy
---
### Step2a: Load Slice Extraction
* After obtaining a trace, we perform load slice extraction by iterating through the trace until one of the delinquent load instructions (see Section 3.2) is found
* traverse in reverse order
	> 算法上用了一个队列的形式。我感觉相当于一个二叉树的层序遍历？
* flag all load slice instructions as critical by prepending the new 'critical' instruction prefix duing the FDO pass.
---
### Step2b: Branch Slice Extraction
For loops that contain hard-to-predict branches, the execution time of a loop iteration is determined to a large degree by the time required to resolve the branch outcom.
--> Prioritize hard-to-predict branch slices

---
### Step3: Inject into binrary
Rewrite the compiled assembly code adding the new instruction prefix to every critical instruction using post-link-time compilation.

---
## Task 2: Hardware Scheduling
* Extend the instruction decoder to interpret the new latency-critical instruction prerfix and tag all critical instructions as they progress through the CPU pipeline
* Extend the IQ scheduler to priotitize these critical instructions.

![Pasted image 20230223232214.png|400](/img/user/Prefetcher/attachments/Pasted%20image%2020230223232214.png)
# APT-GET: Profile-Guided Timely Software Prefetching
---
## Overview
自动化的过程
1. 用perf record找到cache miss多的loads和他们的basic block
2. 找到对应的LBR profile
3. 计算prefetch distance和prefetch injection site
4. 输入LLVM使用
---
## Background
hwpf无法检测complex pattern比如indirect mem access, s现有的sw prefetcher通过static信息可能可以有比较好的accuracy和coverage, 但是由于缺少dynamic execution info 比如execution time, 很难做到timely。这篇文章认为timely prefetch有两个重要的点：prefetch distance和prefetch injection site, 所以通过profile-guided的方法获取了一些dynamic info来指导timely prefetch 途径

![Pasted image 20230223234223.png|400](/img/user/Prefetcher/attachments/Pasted%20image%2020230223234223.png)
## Motivation
![Pasted image 20230223234333.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230223234333.png)
!![Pasted image 20230223234357.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230223234357.png)
## Step1: Profiling
* Cache miss profile --> 收集比较关心的demand loads的信息，比如这条load的hit/miss ratio
* LBR (Last Branch Record) samples --> 可以得到包含这条load指令的loop的trip count, 单次loop的执行时间
收集这些信息后，就可以分析知道同一条load的两次执行之间的时间(loop latency or execution time)，以便得到timely prefetch
![Pasted image 20230223234533.png|400](/img/user/Prefetcher/attachments/Pasted%20image%2020230223234533.png)

> 文章里面用到Intel CPU上的Last Branch Record (LBR) 如上图所示
> 每个LBR有一个PC，记录这里发生了一个taken branch；有一个Target, 记录跳转地址；有一个branch执行的时间。
> LBR是一个buffer, 其中存了last 32 basic blocks (BBL) executed by the CPU。
> BBL是两个taken branch之间连续执行的一段指令

## Step2: Analysis
### Overview
Loop latency包含两部分的执行时间信息
- <font color="#2DC26B">instruction component (IC) </font>
	- includes all (non-load) instructions implementing the loop
	* IC_latency取决于指令数量和指令的控制流依赖，基本上是常量
- <font color="#2DC26B">memory component (MC)</font> 
	- includes the loads causing frequent cache misses
	* MC_latency取决于内存状态，会有很大变化

IC_latency和MC_latency得到之后，可以通过如下公式计算prefetch_distance。在这个distance下，prefetch可以hide memory latency
$$IC\_{latency}\times prefetch\_{distance}=MC\_{latency}$$
接下来就是怎么去得到IC_latency和MC_latency
### A. 计算loop execution time & loop trip count
* <font color="#2DC26B">loop execution time计算</font>：finding two instances of the same branch PC implementing a loop and subtracting their cycle counts, we can compute the execution time of a loop iteration
* <font color="#2DC26B">loop trip count计算</font>：In the case of a nested loop, if we know the branch PC corresponding to the outer loop and the branch PC corresponding to the inner loop, we can count the number of inner branch PCs within two outer branch PCs in the LBR to compute the number of inner loop iterations.

![Pasted image 20230223235046.png|400](/img/user/Prefetcher/attachments/Pasted%20image%2020230223235046.png)

### B. Determining the Optimal Prefetch Distance
得到loop execution time之后还是得不到IC_latency, MC_latency
为此，这篇文章做了<font color="#2DC26B">loop execution time</font>的latency distribution
* for all LBR samples that contain at least two instances of the BBL containing the delinquent load, measure the loop execution time by subtracting the cycle counts of the two subsequent branches
* analyze the latency distribution of the loop's execution time to predict the latency in the case that the load is served from the L1 or L2 cache
    
![Pasted image 20230223235205.png|500](/img/user/Prefetcher/attachments/Pasted%20image%2020230223235205.png)
这个一个distribution图，显示了4个峰：80, 230, 400, 650 cycles; 对应了serve from L1, L2, LLC, DRAM
计算可得IC_latency=80 cycles, MC_latency=650-IC_latency=570 cycles
最后prefetch_distance=570/80~7

## Step3: Injection -- Finding the Optimal Prefetch Injection Site
inject在outerloop还是innerloop, 由以下公式选择，如果以下成立，就inject在outerloop

$$loop\_trip\_count\times k<prefetch\_distance$$
其中k代表的是？？？
如果决定inject在outerloop, 则需要重新根据outer loop execution latency计算prefetch distance


---

# Resources
**### [Disclosure of H/W prefetcher control on some Intel processors](https://radiable56.rssing.com/chan-25518398/article18-live.html)

| Prefetcher                        | Bit# in MSR 0x1A4 | Description                                                                                                                     |
|-----------------------------------|-------------------|---------------------------------------------------------------------------------------------------------------------------------|
| L2 hardware prefetcher            | 0                 | Fetches additional lines of code or data into the L2 cache                                                                      |
| L2 adjacent cache line prefetcher | 1                 | Fetches the cache line that comprises a cache line pair (128 bytes)                                                             |
| DCU prefetcher                    | 2                 | Fetches the next cache line into L1-D cache                                                                                     |
| DCU IP prefetcher                 | 3                 | Uses sequential load history (based on Instruction Pointer of previous loads) to determine whether to prefetch additional lines |
If any of the above bits are set to 1 on a core, then that particular prefetcher on that core is disabled. Clearing that bit (setting it to 0) will enable the corresponding prefetcher.


* The algorithms found in the L1 cache hardware are the Data Cache Unit (DCU) Prefetcher (INTEL, 2019) and the DCU IP Prefetcher (INTEL, 2019). The DCU Prefetcher, also known as the streaming prefetcher, is triggered by an ascending access to very recently loaded data. The processor assumes that this access is part of a streaming algorithm and automatically fetches the next line. The DCU IP Prefetcher keeps track of individual load instructions (based on their instruction pointer value). If a load instruction is detected to have a regular stride, then a prefetch is sent to the next address which is the sum of the current address and the stride. 
* The L2 Hardware Prefetcher (INTEL, 2019) and the L2 Adjacent Cache Prefetcher (INTEL, 2019) are the prefetcher algorithms found in the real machine L2 cache. The L2 Hardware Prefetcher monitors read requests from the L1 cache for ascending and descending sequences of addresses. Monitored read requests include L1 data cache requests initiated by load and store operations and also by the L1 prefetchers, and L1 instruction cache requests for code fetch. When a forward or backward stream of requests is detected, the anticipated cache lines are prefetched. This prefetcher may issue two prefetch requests on every L2 lookup and run up to 20 lines ahead of the load request. The L2 Adjacent Cache Prefetcher fetches two 64-byte cache lines into a 128-byte sector instead of only one, regardless of whether the additional cache line has been requested or not.

![Pasted image 20230222143800.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230222143800.png)
![Pasted image 20230222144621.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230222144621.png)
[Fetching Title#wnft](https://software.intel.com/sites/default/files/managed/9e/bc/64-ia-32-architectures-optimization-manual.pdf)**