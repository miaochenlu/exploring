---
{"dg-publish":true,"permalink":"/prefetcher/codes/isb/isb-code/","dgPassFrontmatter":true,"noteIcon":""}
---

[[Prefetcher/codes/Gem5中的Prefetcher基类\|Gem5中的Prefetcher基类]]

# Components
## 1. Training Unit
```cpp
/**
 * Training Unit Entry datatype, it holds the last accessed address and
 * its secure flag
 */
struct TrainingUnitEntry : public TaggedEntry
{
    Addr lastAddress;
    bool lastAddressSecure;
};
/** Map of PCs to Training unit entries */
AssociativeSet<TrainingUnitEntry> trainingUnit;
```

## 2. Address Mapping Caches
![Pasted image 20230130234740.png](/img/user/Prefetcher/codes/ISB/attachments/Pasted%20image%2020230130234740.png)
Entry的设置
```cpp
/** Address Mapping entry, holds an address and a confidence counter */
struct AddressMapping
{
    Addr address;
    SatCounter8 counter;
    AddressMapping(unsigned bits) : address(0), counter(bits)
    {}
};

/**
 * Maps a set of contiguous addresses to another set of (not necessarily
 * contiguos) addresses, with their corresponding confidence counters
 */
struct AddressMappingEntry : public TaggedEntry
{
	/*********************一个entry中有num_mappings个mapping*************/
    std::vector<AddressMapping> mappings;
    AddressMappingEntry(size_t num_mappings, unsigned counter_bits)
        : TaggedEntry(), mappings(num_mappings, counter_bits)
    {
    }

    void
    invalidate() override
    {
        TaggedEntry::invalidate();
        for (auto &entry : mappings) {
            entry.address = 0;
            entry.counter.reset();
        }
    }
};
```

```cpp
/** Physical-to-Structured mappings table */
AssociativeSet<AddressMappingEntry> psAddressMappingCache;
/** Structured-to-Physical mappings table */
AssociativeSet<AddressMappingEntry> spAddressMappingCache;
```



# Process
整个calculatePrefetch函数

```cpp
void
IrregularStreamBuffer::calculatePrefetch(const PrefetchInfo &pfi,
    std::vector<AddrPriority> &addresses)
{
    // This prefetcher requires a PC
    if (!pfi.hasPC()) {
        return;
    }
    bool is_secure = pfi.isSecure();
    Addr pc = pfi.getPC();
    Addr addr = blockIndex(pfi.getAddr());

    // Training, if the entry exists, then we found a correlation between
    // the entry lastAddress (named as correlated_addr_A) and the address of
    // the current access (named as correlated_addr_B)
    TrainingUnitEntry *entry = trainingUnit.findEntry(pc, is_secure);
    bool correlated_addr_found = false;
    Addr correlated_addr_A = 0;
    Addr correlated_addr_B = 0;
    if (entry != nullptr && entry->lastAddressSecure == is_secure) {
        trainingUnit.accessEntry(entry);
        correlated_addr_found = true;
        correlated_addr_A = entry->lastAddress;
        correlated_addr_B = addr;
    } else {
        entry = trainingUnit.findVictim(pc);
        assert(entry != nullptr);

        trainingUnit.insertEntry(pc, is_secure, entry);
    }
    // Update the entry
    entry->lastAddress = addr;
    entry->lastAddressSecure = is_secure;

    if (correlated_addr_found) {
        // If a correlation was found, update the Physical-to-Structural
        // table accordingly
        AddressMapping &mapping_A = getPSMapping(correlated_addr_A, is_secure);
        AddressMapping &mapping_B = getPSMapping(correlated_addr_B, is_secure);
        if (mapping_A.counter > 0 && mapping_B.counter > 0) {
            // Entry for A and B
            if (mapping_B.address == (mapping_A.address + 1)) {
                mapping_B.counter++;
            } else {
                if (mapping_B.counter == 1) {
                    // Counter would hit 0, reassign address while keeping
                    // counter at 1
                    mapping_B.address = mapping_A.address + 1;
                    addStructuralToPhysicalEntry(mapping_B.address, is_secure,
                            correlated_addr_B);
                } else {
                    mapping_B.counter--;
                }
            }
        } else {
            if (mapping_A.counter == 0) {
                // if A is not valid, generate a new structural address
                mapping_A.counter++;
                mapping_A.address = structuralAddressCounter;
                structuralAddressCounter += chunkSize;
                addStructuralToPhysicalEntry(mapping_A.address,
                        is_secure, correlated_addr_A);
            }
            mapping_B.counter.reset();
            mapping_B.counter++;
            mapping_B.address = mapping_A.address + 1;
            // update SP-AMC
            addStructuralToPhysicalEntry(mapping_B.address, is_secure,
                    correlated_addr_B);
        }
    }

    // Use the PS mapping to predict future accesses using the current address
    // - Look for the structured address
    // - if it exists, use it to generate prefetches for the subsequent
    //   addresses in ascending order, as many as indicated by the degree
    //   (given the structured address S, prefetch S+1, S+2, .. up to S+degree)
    Addr amc_address = addr / prefetchCandidatesPerEntry;
    Addr map_index   = addr % prefetchCandidatesPerEntry;
    AddressMappingEntry *ps_am = psAddressMappingCache.findEntry(amc_address,
                                                                 is_secure);
    if (ps_am != nullptr) {
        AddressMapping &mapping = ps_am->mappings[map_index];
        if (mapping.counter > 0) {
            Addr sp_address = mapping.address / prefetchCandidatesPerEntry;
            Addr sp_index   = mapping.address % prefetchCandidatesPerEntry;
            AddressMappingEntry *sp_am =
                spAddressMappingCache.findEntry(sp_address, is_secure);
            if (sp_am == nullptr) {
                // The entry has been evicted, can not generate prefetches
                return;
            }
            for (unsigned d = 1;
                    d <= degree && (sp_index + d) < prefetchCandidatesPerEntry;
                    d += 1)
            {
                AddressMapping &spm = sp_am->mappings[sp_index + d];
                //generate prefetch
                if (spm.counter > 0) {
                    Addr pf_addr = spm.address << lBlkSize;
                    addresses.push_back(AddrPriority(pf_addr, 0));
                }
            }
        }
    }
}
```

## Training
[[Drawing 2023-01-30 16.55.02.excalidraw\|Drawing 2023-01-30 16.55.02.excalidraw]]
```cpp
    // This prefetcher requires a PC
    if (!pfi.hasPC()) {
        return;
    }
    bool is_secure = pfi.isSecure();
    Addr pc = pfi.getPC();
    Addr addr = blockIndex(pfi.getAddr());

    // Training, if the entry exists, then we found a correlation between
    // the entry lastAddress (named as correlated_addr_A) and the address of
    // the current access (named as correlated_addr_B)
    TrainingUnitEntry *entry = trainingUnit.findEntry(pc, is_secure);
    bool correlated_addr_found = false;
    Addr correlated_addr_A = 0;
    Addr correlated_addr_B = 0;
    if (entry != nullptr && entry->lastAddressSecure == is_secure) {
        trainingUnit.accessEntry(entry);
        correlated_addr_found = true;
        correlated_addr_A = entry->lastAddress;
        correlated_addr_B = addr;
    } else {
        entry = trainingUnit.findVictim(pc);
        assert(entry != nullptr);

        trainingUnit.insertEntry(pc, is_secure, entry);
    }
    // Update the entry
    entry->lastAddress = addr;
    entry->lastAddressSecure = is_secure;

```

## Physical -> Structural Address Mapping
![Pasted image 20230131001114.png](/img/user/Prefetcher/codes/ISB/attachments/Pasted%20image%2020230131001114.png)

```cpp
IrregularStreamBuffer::AddressMapping&
IrregularStreamBuffer::getPSMapping(Addr paddr, bool is_secure)
{
    Addr amc_address = paddr / prefetchCandidatesPerEntry;
    Addr map_index   = paddr % prefetchCandidatesPerEntry;
    AddressMappingEntry *ps_entry =
        psAddressMappingCache.findEntry(amc_address, is_secure);
    if (ps_entry != nullptr) {
        // A PS-AMC line already exists
        psAddressMappingCache.accessEntry(ps_entry);
    } else {
        ps_entry = psAddressMappingCache.findVictim(amc_address);
        assert(ps_entry != nullptr);

        psAddressMappingCache.insertEntry(amc_address, is_secure, ps_entry);
    }
    return ps_entry->mappings[map_index];
}
```

## Generate Prefetch & Get Structural -> Physical Address Mapping
![Pasted image 20230131001603.png](/img/user/Prefetcher/codes/ISB/attachments/Pasted%20image%2020230131001603.png)

```cpp
    // Use the PS mapping to predict future accesses using the current address
    // - Look for the structured address
    // - if it exists, use it to generate prefetches for the subsequent
    //   addresses in ascending order, as many as indicated by the degree
    //   (given the structured address S, prefetch S+1, S+2, .. up to S+degree)
    Addr amc_address = addr / prefetchCandidatesPerEntry;
    Addr map_index   = addr % prefetchCandidatesPerEntry;
    AddressMappingEntry *ps_am = psAddressMappingCache.findEntry(amc_address,
                                                                 is_secure);
    if (ps_am != nullptr) {
        AddressMapping &mapping = ps_am->mappings[map_index];
        if (mapping.counter > 0) {
            Addr sp_address = mapping.address / prefetchCandidatesPerEntry;
            Addr sp_index   = mapping.address % prefetchCandidatesPerEntry;
            AddressMappingEntry *sp_am =
                spAddressMappingCache.findEntry(sp_address, is_secure);
            if (sp_am == nullptr) {
                // The entry has been evicted, can not generate prefetches
                return;
            }
            for (unsigned d = 1;
                    d <= degree && (sp_index + d) < prefetchCandidatesPerEntry;
                    d += 1)
            {
                AddressMapping &spm = sp_am->mappings[sp_index + d];
                //generate prefetch
                if (spm.counter > 0) {
                    Addr pf_addr = spm.address << lBlkSize;
                    addresses.push_back(AddrPriority(pf_addr, 0));
                }
            }
        }
    }
```

# 整体参数

```python
class IrregularStreamBufferPrefetcher(QueuedPrefetcher):
    type = "IrregularStreamBufferPrefetcher"
    cxx_class = 'gem5::prefetch::IrregularStreamBuffer'
    cxx_header = "mem/cache/prefetch/irregular_stream_buffer.hh"

    num_counter_bits = Param.Unsigned(2,
        "Number of bits of the confidence counter")
    chunk_size = Param.Unsigned(256,
        "Maximum number of addresses in a temporal stream")
    degree = Param.Unsigned(4, "Number of prefetches to generate")
    training_unit_assoc = Param.Unsigned(128,
        "Associativity of the training unit")
    training_unit_entries = Param.MemorySize("128",
        "Number of entries of the training unit")
    training_unit_indexing_policy = Param.BaseIndexingPolicy(
        SetAssociative(entry_size = 1, assoc = Parent.training_unit_assoc,
        size = Parent.training_unit_entries),
        "Indexing policy of the training unit")
    training_unit_replacement_policy = Param.BaseReplacementPolicy(LRURP(),
        "Replacement policy of the training unit")

    prefetch_candidates_per_entry = Param.Unsigned(16,
        "Number of prefetch candidates stored in a SP-AMC entry")
    address_map_cache_assoc = Param.Unsigned(128,
        "Associativity of the PS/SP AMCs")
    address_map_cache_entries = Param.MemorySize("128",
        "Number of entries of the PS/SP AMCs")
    ps_address_map_cache_indexing_policy = Param.BaseIndexingPolicy(
        SetAssociative(entry_size = 1,
        assoc = Parent.address_map_cache_assoc,
        size = Parent.address_map_cache_entries),
        "Indexing policy of the Physical-to-Structural Address Map Cache")
    ps_address_map_cache_replacement_policy = Param.BaseReplacementPolicy(
        LRURP(),
        "Replacement policy of the Physical-to-Structural Address Map Cache")
    sp_address_map_cache_indexing_policy = Param.BaseIndexingPolicy(
        SetAssociative(entry_size = 1,
        assoc = Parent.address_map_cache_assoc,
        size = Parent.address_map_cache_entries),
        "Indexing policy of the Structural-to-Physical Address Mao Cache")
    sp_address_map_cache_replacement_policy = Param.BaseReplacementPolicy(
        LRURP(),
        "Replacement policy of the Structural-to-Physical Address Map Cache")

```
