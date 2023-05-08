---
{"UID":20230423152025,"aliases":"1. 成员变量","tags":null,"source":null,"cssclass":null,"created":"2023-04-23 15:20","updated":"2023-04-24 10:11","dg-publish":true,"permalink":"/prefetcher/codes/sms-prefetcher/","dgPassFrontmatter":true,"noteIcon":""}
---


主要参考这份代码[GEM5/sms.cc at 15447114d16165a2c6e4315e8fcd45138d7dc901 · OpenXiangShan/GEM5 · GitHub](https://github.com/OpenXiangShan/GEM5/blob/15447114d16165a2c6e4315e8fcd45138d7dc901/src/mem/cache/prefetch/sms.cc)
这份代码其实已经不是纯粹的sms, 还杂糅了IPCP的GS, CS。
# 1. 成员变量
## i. active generation table
用region_addr进行寻址

```cpp
// active generation table
class ACTEntry : public TaggedEntry {
  public:
	Addr pc;
	bool is_secure;
	uint64_t region_bits;
	bool decr_mode;
	uint8_t access_cnt;
	uint64_t region_offset;
	ACTEntry(const SatCounter8 &conf);
	bool in_active_page() {
		// FIXME: remove hard-code 12
		return access_cnt > 12;
	}
};

AssociativeSet<ACTEntry> act;
```

## ii. pattern history table 
用pc进行寻址

```cpp
// pattern history table
class PhtEntry : public TaggedEntry {
  public:
	std::vector<SatCounter8> hist;
	PhtEntry(const size_t sz, const SatCounter8 &conf)
		: TaggedEntry(), hist(sz, conf)
	{}
};

AssociativeSet<PhtEntry> pht;
```

# 2. 成员函数
1. 在act中找到对应的act entry
	1. 如果hit了，更新访问的数据，返回entry
	2. 如果miss了
		1. 查看前后两个相邻的region是否是访问密集的entry，如果是，则可以推断该page也是active_page
		2. evict act entry到pht，并且allocate新的entry。记录pc, region_offset等信息
`actLookup`函数

2. 如果是active page
	* `Addr pf_tgt_addr = decr ? block_addr - 30 * blkSize : block_addr + 30 * blkSize;` 这里+-30提供了足够的lookahead

```cpp
ACTEntry *act_match_entry = actLookup(pfi, is_active_page);
if (act_match_entry) {
	bool decr = act_match_entry->decr_mode;
	bool is_cross_region_match = act_match_entry->access_cnt == 0;
	if (is_cross_region_match) {
		act_match_entry->access_cnt = 1;
	}
	if (is_active_page) {
		// active page
		Addr pf_tgt_addr =
			decr ? block_addr - 30 * blkSize : block_addr + 30 * blkSize;
		Addr pf_tgt_region = regionAddress(pf_tgt_addr);
		Addr pf_tgt_offset = regionOffset(pf_tgt_addr);

		if (decr) {
			for (int i = (int)region_blocks - 1;
				 i >= pf_tgt_offset && i >= 0; i--) {
				Addr cur = pf_tgt_region * region_size + i * blkSize;
				addresses.push_back(AddrPriority(cur, i));
			}
		} else {
			for (uint8_t i = 0; i <= pf_tgt_offset; i++) {
				Addr cur = pf_tgt_region * region_size + i * blkSize;
				addresses.push_back(AddrPriority(cur, region_blocks - i));
			}
		}
	}
}
```

1. 如果不是active page，先看是不是满足stride pattern，这和IPCP的CS是差不多的

```cpp
void
SMSPrefetcher::strideLookup(const PrefetchInfo &pfi,
                            std::vector<AddrPriority> &address)
{
    Addr lookupAddr = blockAddress(pfi.getAddr());
    StrideEntry *entry = stride.findEntry(pfi.getPC(), pfi.isSecure());
    if (entry) {
        stride.accessEntry(entry);
        int64_t new_stride = lookupAddr - entry->last_addr;
        bool stride_match = new_stride == entry->stride && new_stride != 0;
        if (stride_match) {
            entry->conf++;
        } else {
            if (entry->conf < 2) {
                entry->stride = new_stride;
            }
            entry->conf--;
        }
        entry->last_addr = lookupAddr;
        if (entry->conf >= 2) {
            Addr pf_addr = lookupAddr + entry->stride;
            address.push_back(AddrPriority(pf_addr, 0));
        }
    } else {
        entry = stride.findVictim(0);
        entry->conf.reset();
        entry->last_addr = lookupAddr;
        entry->stride = 0;
        stride.insertEntry(pfi.getPC(), pfi.isSecure(), entry);
    }
}
```

1. 如果不是active page, 再看pht

```cpp
void
SMSPrefetcher::phtLookup(const Base::PrefetchInfo &pfi,
                         std::vector<AddrPriority> &addresses)
{
    Addr pc = pfi.getPC();
    Addr vaddr = pfi.getAddr();
    Addr blk_addr = blockAddress(vaddr);
    Addr region_offset = regionOffset(vaddr);
    bool secure = pfi.isSecure();
    PhtEntry *pht_entry = pht.findEntry(pc, secure);
    if (pht_entry) {
        pht.accessEntry(pht_entry);
        int priority = 2 * (region_blocks - 1);
        // find incr pattern
        for (uint8_t i = 0; i < region_blocks - 1; i++) {
            if (pht_entry->hist[i + region_blocks - 1].calcSaturation() >
                0.5) {
                Addr pf_tgt_addr = blk_addr + (i + 1) * blkSize;
                addresses.push_back(AddrPriority(pf_tgt_addr, priority--));
            }
        }
        for (int i = region_blocks - 2, j = 1; i >= 0; i--, j++) {
            if (pht_entry->hist[i].calcSaturation() > 0.5) {
                Addr pf_tgt_addr = blk_addr - j * blkSize;
                addresses.push_back(AddrPriority(pf_tgt_addr, priority--));
            }
        }
    }
}
```