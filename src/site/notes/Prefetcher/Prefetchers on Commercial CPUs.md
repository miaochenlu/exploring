---
{"UID":20230313105549,"aliases":"AMD","tags":null,"source":null,"cssclass":null,"created":"2023-03-13 10:55","updated":"2023-03-14 10:14","dg-publish":true,"permalink":"/prefetcher/prefetchers-on-commercial-cp-us/","dgPassFrontmatter":true,"noteIcon":""}
---


# AMD
  AMD family 19h processors implement data prefetch logic for its L1 data cache and L2 cache. In general, the L1 data prefetchers fetch lines into both the L1 data cache and the L2 cache, while the L2 data prefetchers fetch lines into the L2 cache. (意思是L1分流，L2不分流)

The following prefetchers are included:
- L1 Stream: Uses history of memory access patterns to fetch additional sequential lines in ascending or descending order
* L1 Stride: Uses memory access history of individual instructions to fetch additional lines when each access is a constant distance from the previous.
* L1 Region: Uses memory access history to fetch additional lines when the data access for a given instruction tends to be followed by a consistent pattern of other accesses within a localized region.
* L2 Stream: Uses history of memory access patterns to fetch additional sequential lines in ascending or descending order.
* L2 Up/Down: Uses memory access history to determine whether to fetch the next or previous line for all memory accesses.

 [Software optimization guide for amd epyc 7003 processors.](https://www.amd.com/system/files/TechDocs/56665.zip.)

# Intel
## Stream Prefetcher (L2/LLC)
* Train stream using BBL transactions
* Read streams and send prefetch requests to L2Q

---

16 instruction streams, 32 data streams
Each stream is for a 4K page and is core specific
Streams are replaced using round robin when there's no empty stream

| Request is Demand Type | CAM Match | Request got L2 Hit | Action          |
| ---------------------- | --------- | ------------------ | --------------- |
| 0                      | 0         | x                  | Do nothing      |
| 1                      | 0         | 1                  | Do nothing      |
| 1                      | 0         | 0                  | Stream Creation |
| x                      | 1         | x                  | Stream Update                |

最前面表示cache line
![Pasted image 20230313143938.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230313143938.png)
![Pasted image 20230314094355.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230314094355.png)
* (a) Demands hitting the far fall back window twice in a role will put the stream in searching state and set num_prefetch_pending = 0
* (b) Increament the LLC num_prefetch_pending by a creg value
* (c) Increament the L2/LLC num_prefetch_pendings by creg values (common case)
* (d) Increment the L2/LLC num_prefetch_pendings by skip_ahead creg values.

### A Stream Lift Cycle

A stream has 4 states: Searching, Forward, Backward, Done
* Start with Searching (ex. stream allocate)
* Move from Searching to Forward/Backward if there's enough demands to determine the direction
* Ideally, stay in Forward/Backward until the L2 homeline reaches page-end/page-begin, Then move to Done.
* In Done state, the stream can either move back to Searching (cache miss) or be replaced for another 4K page
[[Excalidraw/stream state machine\|stream state machine]]
![Pasted image 20230314101418.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230314101418.png)


# ARM
## APPLE
* DMP
[augury.pdf](https://www.prefetchers.info/augury.pdf)
## X3
The X3 features a dozen prefetch engines.
One new engine looks for sequences of indirect loads while the other seeks three-dimensional(spatial) patterns.

![[CORTEX-X3 POWERS UP.pdf]]