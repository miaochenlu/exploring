---
{"UID":20230313105549,"aliases":"AMD","tags":null,"source":null,"cssclass":null,"created":"2023-03-13 10:55","updated":"2023-03-16 11:25","dg-publish":true,"permalink":"/prefetcher/prefetchers-on-commercial-cp-us/","dgPassFrontmatter":true,"noteIcon":""}
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

### D-Side State Machine
![Pasted image 20230314101418.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230314101418.png)

* New stream create (eb_allocate_new_ms58h)
	* page begin kickstart: Forward
	* page end kickstart: Backward
	* Otherwise: Searching
* Searching
	* within_initwin_fwd: Forward
	* Within_initwin_bck: Backward
	* Otherise: Searching
* Done
	* cache miss: Searching
	* Otherwise: Done
* Forward/Backward
	* within_new_skipahead_ms58h
		* increment perf_pending
		* Move home lines
	* Stable_strm_prefpending_ms58h
		* increment pref_pending
	* Within_trig_win_ms58h
		* increment pref_pending
		* Move home lines
	* Within_fallback_ms58h
		* Increment pref_pending (LLC only)
		* Move home lines
	* If detects reverse direction twice (Within_far_fallback_ms58h) and cache miss
		* Searching

### I-Side State Machine
* The only states are Forward and Done (no Searching, no Backward).
* Kick start to forward state during stream allocate, no searching state.
* L2 prefetch only (no LLC prefetch)
* Never overwrite L2 prefetch pending with a smaller value
* When hitting the fallback windows, no update to the stream (also no update to pref_pending).
![Pasted image 20230316102906.png|400](/img/user/Prefetcher/attachments/Pasted%20image%2020230316102906.png)

### Stream's Key Fields
* Page address[51:12]
* State // Searching/Forward/Backward/Done
* L2 home offset[11:6] (page offset 减去 cache line offset)
* L2 prefetch pending // number of L2 prefetches pending to be sent out
* LLC home offset[11:6]
* LLC prefetch pending // number of LLC prefetches pending to be sent out
* AMP last offset[11:6]
* AMP delta1 // index to prediction table (SPT0, SPT1)
* AMP delta2 // index to prediction table (SPT1)
* AMP new page

### Stream Prefetcher: LLC
* The LLC home line maintains a min/max distance from the L2 prefetch home line (max distance is disabled).
* The LLC home line typically finishes (ex. end of page) before the L2 home line. The stream moves to the Done state when the L2 home line finishes
* LLC prefetch is only for data and not instruction (I-side). 
	* Implementing I-side LLC would require an entirely new windowing scheme.
	* I-side doesn't typically need to prefetch as far ahead anyway
	* No plan to implement I-side LLC at this point


### Stream Prefetcher: SendStream (Send Prefetch Requests to L2Q)
### Processes
* Read the 48 streams one-by-one using the round robin method
* If the stream has L2/LLC prefetch pending number != 0
	* Decrement the prefetch pending number by 1
	* Increment the L2/LLC home line (prefetch offset) by 1 forward state
	* Decrement the L2/LLC home line (prefetch offset) by 1 backward state
	* Send out 1 prefetch with address = {page_addr, prefetch_offset}
	* Move on to the next stream
* For each stream, ping-pong between L2/LLC prefetch
* For Forward state, set the stream state = DONE whe the L2 home line reaches the end of the 4K page
* For Backward state, set the stream state = DONE when the L2 home line reaches the begining of the 4K page.
### Bitmap Filter (For each stream)
* Before a prefetch request is sent to L2Q, check againsga
* Update the bitmap filter bit corresponding the prefetch addres
### L2Q credit
* only send prefetch request to L2Q when there's L2Q credit available (eb_pref_req_credit_avail_mnnnh)

### Prefetch Accelation and Throttling 
![Pasted image 20230314161207.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230314161207.png)
### Prefetcher Blocking
![Pasted image 20230314161154.png](/img/user/Prefetcher/attachments/Pasted%20image%2020230314161154.png)

### Intel Stream Prefetcher Reverse
[ReadPaper](https://readpaper.com/pdf-annotate/note?pdfId=4731820407082975233&noteId=1688252371593961984)

#### Structure
* A table of stream entries tagged by a page number. Each entry contains the **last fetched line** in the page, **a direction state**, and **a prefetcher confidence state**. The last fetched line (L) is the last prefetch candidate or the last request when lacking such a candidate. 
* The **prefetch candidate** logic block takes as an input the miss from the L1 cache and the stream entry for the same page. It uses the stream direction state and last fetched line to suggest prefetch candidates and update the stream entry.
* The **prefetch confidence** logic block computes and updates the confidence prefetcher for this stream based on requests made and whether they fit the stream.
* The **prefetch arbitration** logic uses the confidence and candidates from all prefetchers to pick which to issue.
![Pasted image 20230316105924.png|500](/img/user/Prefetcher/attachments/Pasted%20image%2020230316105924.png)

The Stream prefetcher treats in a special way streams that first access a page in its first or last two lines (如果access了一个page中的前两个或者后两个line 会触发prefetch). 
Otherwise, if confident enough, it prefetches a pair of consecutive lines starting on the **last fetched line** or the **current line**, whichever is furthest along the direction of the stream. (有两个起始地点)
Accesses 32 or more lines away from the last fetched line are treated differently. (current line和last fetched line不能差距太远)
Prefetches issued will safely wrap around page limits, which may issue pointless prefetches but causes no potentially dangerous prefetches across page limits.
The prefetcher seems to output a confidence metric used to decide whether to prefetch, but suppressed prefetches may update the prefetcher state. Lastly, the prefetcher is reluctant to start streams too close to the page end. 

# ARM
## APPLE
* DMP
[augury.pdf](https://www.prefetchers.info/augury.pdf)
## X3
The X3 features a dozen prefetch engines.
One new engine looks for sequences of indirect loads while the other seeks three-dimensional(spatial) patterns.

![[CORTEX-X3 POWERS UP.pdf]]