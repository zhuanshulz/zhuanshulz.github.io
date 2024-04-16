---
title: Gem5模拟器配置StreamPrefecher
date: 2024-04-16 21:31:36
tags: [Gem5]
---

Gem5默认不支持StreamPrefetcher，为了在各级缓存使用StreamPrefetcher，需进行如下配置：
<!-- more -->

---

## 一、添加预取算法

文件路径：./src/mem/cache/prefetch/stream.cc

```python

#include "mem/cache/prefetch/stream.hh"

#include <cassert>

#include "base/intmath.hh"
#include "base/logging.hh"
#include "base/random.hh"
#include "base/trace.hh"
#include "debug/HWPrefetch.hh"
#include "mem/cache/prefetch/associative_set_impl.hh"
#include "mem/cache/replacement_policies/base.hh"

#include "params/StreamPrefetcher.hh"


namespace gem5
{

namespace prefetch
{


Stream::Stream(const StreamPrefetcherParams &p)
: Queued(p),
  tableSize(p.tableSize),
  useMasterId(p.use_master_id),
  degree(p.degree),
  distance(p.distance) {
    for(int i=0; i<MaxContexts; i++) {
        StreamTable[i] = new StreamTableEntry*[tableSize];
        for(int j=0; j<tableSize; j++) {
            StreamTable[i][j] = new StreamTableEntry[tableSize];
            StreamTable[i][j]->LRU_index = j;
            resetEntry(StreamTable[i][j]);
        }
    }
}

Stream::~Stream() {
     for (int i = 0; i < MaxContexts; i++) {
            for (int j = 0; j < tableSize; j++) {
                delete[] StreamTable[i][j];
            }
        }
}


// Training and Prefetching of streams
void
Stream::calculatePrefetch(const PrefetchInfo &pfi,
                                    std::vector<AddrPriority> &addresses) {
    uint32_t core_id = 0;
    Addr blk_addr = pfi.getAddr() & ~(Addr)(blkSize-1); // cache block aligned address.
    // assert(core_id < MaxContexts);
    StreamTableEntry** table;
    table = StreamTable[core_id];                          // Per core stream training.
    uint32_t i;
    // Check if there is a stream entry with the same address as blk_addr
    for (i = 0; i < tableSize; i++) {
        switch (table[i]->status) {
        case MONITOR:
            if(table[i]->trainedDirection == ASCENDING) {
                // Ascending order
                if((table[i]->startAddr < blk_addr ) && ( table[i]->endAddr > blk_addr)) {
                    // Hit to a stream, which is monitored. Issue prefetch requests based on the degree and the direction
                    for (uint8_t d = 1; d <= degree; d++) {
                        Addr pf_addr = table[i]->endAddr + blkSize * d;
                        addresses.push_back(AddrPriority(pf_addr,0));
                        DPRINTF(HWPrefetch, "Queuing prefetch to %#x.\n", pf_addr);
                    }
                    if((table[i]->endAddr + blkSize * degree) - table[i]->startAddr <= distance) {
                        table[i]->endAddr   = table[i]->endAddr + blkSize * degree;
                    } else {
                        table[i]->startAddr = table[i]->startAddr + blkSize * degree;
                        table[i]->endAddr   = table[i]->endAddr   + blkSize * degree;
                    }
                    break;
                }
            } else if(table[i]->trainedDirection == DESCENDING) {
                // Descending order
                if((table[i]->startAddr > blk_addr ) && (table[i]->endAddr < blk_addr)) {
                    for (uint8_t d = 1; d <= degree; d++) {
                        Addr pf_addr = table[i]->endAddr - blkSize * d;
                        addresses.push_back(AddrPriority(pf_addr,0));
                        DPRINTF(HWPrefetch, "Queuing prefetch to %#x.\n", pf_addr);
                    }
                    if(table[i]->startAddr - (table[i]->endAddr - blkSize * degree) <= distance){
                        table[i]->endAddr   = table[i]->endAddr - blkSize * degree;
                    } else {
                        table[i]->startAddr = table[i]->startAddr - blkSize * degree;
                        table[i]->endAddr   = table[i]->endAddr   - blkSize * degree;
                    }
                    break;
                }
            } else{
                assert(0);
            }
            break;
        case TRAINING:
            if ((abs(table[i]->allocAddr - blk_addr) <= (distance/2) * blkSize) ){
                // Check whether the address is in +/- of distance
                if(table[i]->trendDirection[0] == INVALID){
                    table[i]->trendDirection[0] = (blk_addr - table[i]->allocAddr > 0) ? ASCENDING : DESCENDING;
                } else {
                    assert(table[i]->trendDirection[1] == INVALID);
                    table[i]->trendDirection[1] = (blk_addr - table[i]->allocAddr > 0) ? ASCENDING : DESCENDING;
                    if(table[i]->trendDirection[0] == table[i]->trendDirection[1]) {
                        table[i]->trainedDirection = table[i]->trendDirection[0];
                        table[i]->startAddr = table[i]->allocAddr;
                        if(table[i]->trainedDirection != INVALID){
                            // Based on the trainedDirection (+1:Ascending, -1:Descending) update the end address of a stream
                            table[i]->endAddr = blk_addr + (table[i]->trainedDirection) * blkSize * degree;
                        }
                        // Entry is ready for issuing prefetch requests
                        table[i]->status = MONITOR;
                    } else {
                        resetEntry(table[i]);
                    }
                }
                break;
            }
            break;
        default:
            break;
        }  // End of Switch
    }  // End of for loop
    uint32_t HIT_index=i;
    int INVALID_index = tableSize;
    for (int i=0; i<tableSize; i++) {
        //find empty entry
        if(table[i]->status==INV) {
            INVALID_index = i;
            break;
        }
    }
    int TEMP_index = -1;
    int LRU_index = -1000000;
    for (int i=0; i<tableSize; i++) {
        //find empty entry
        if(table[i]->LRU_index > TEMP_index) {
            TEMP_index = table[i]->LRU_index;
            LRU_index  = i;
        }
    }
    assert(TEMP_index == tableSize - 1);
    int entry_id;
    if(HIT_index!=tableSize) {  //hit
        entry_id = HIT_index;
    } else if (INVALID_index!=tableSize) {
        //Existence of invalid streams
        assert(table[INVALID_index]->status == INV);
        table[INVALID_index]->status = TRAINING;
        table[INVALID_index]->allocAddr = blk_addr;
        entry_id = INVALID_index;
    } else {
        //Replace the LRU stream-entry
        assert(table[LRU_index]->status!=INV);
        resetEntry(table[LRU_index]);
        table[LRU_index]->status = TRAINING;
        table[LRU_index]->allocAddr = blk_addr;
        entry_id = LRU_index;
    }
    // Shifting the table entries after the eviction of lru-id
    for (int i=0; i<tableSize; i++) {
        if(table[i]->LRU_index < table[entry_id]->LRU_index){
            table[i]->LRU_index = table[i]->LRU_index + 1;
        }
    }
    table[entry_id]->LRU_index = 0;

}

void
Stream::resetEntry(StreamTableEntry *this_entry)
{

    this_entry->status                = INV;
    this_entry->trendDirection[0]     = INVALID;
    this_entry->trendDirection[1]     = INVALID;
    this_entry->allocAddr             = 0;
    this_entry->startAddr             = 0;
    this_entry->endAddr               = 0;
    this_entry->trainedDirection      = INVALID;

}

} // namespace prefetch

// prefetch::Stream *
// StreamPrefetcherParams::create() const
// {
//     return new prefetch::Stream(*this);
// }

} // namespace gem5

```
---

文件路径：./src/mem/cache/prefetch/stream.hh

```c++
#ifndef __MEM_CACHE_PREFETCH_STREAM_HH__
#define __MEM_CACHE_PREFETCH_STREAM_HH__

#include "mem/cache/prefetch/queued.hh"
#include "params/StreamPrefetcher.hh"
// Direction of stream for each stream entry in the stream table

#include "base/types.hh"

namespace gem5
{

namespace prefetch
{

enum StreamDirection{
        ASCENDING = 1,                      // For example - A, A+1, A+2
        DESCENDING = -1,                    // For example - A, A-1, A-2
        INVALID = 0
};
// Status of a stream entry in the stream table.
enum StreamStatus{
            INV       = 0,
            TRAINING  = 1,                  // Stream training is not over yet. Once trained will move to MONITOR status
            MONITOR   = 2                   // Monitor and Request: Stream entry ready for issuing prefetch requests
};


class Stream : public Queued {
  protected:
    static const uint32_t MaxContexts = 64; // Creates per-core stream tables for upto 64 processor cores
    uint32_t tableSize;                     // Number of entries in a stream table
    const bool useMasterId;                 // Use the master-id to train the streams
    uint32_t degree;                        // Determines the number of prefetch reuquests to be issued at a time
    uint32_t distance;                      // Determines the prefetch distance

   /* StreamTableEntry 
     Stores the basic attributes of a stream table entry.
   */
  class StreamTableEntry {

      public:
        int  LRU_index;
        Addr allocAddr;                     // Address that initiated the stream training
        Addr startAddr;                     // First address of a stream
        Addr endAddr;                       // Last address of a stream
        StreamDirection trainedDirection;   // Direction of trained stream (Ascending or Descending)
        StreamStatus    status;             // Status of the stream entry
        StreamDirection trendDirection[2];  // Stores the last two stream directions of an entry

  };
  void resetEntry (StreamTableEntry *this_entry);

  /* Creating a StreamTable for each core with 
     Tablesize as the number of stream entries 
  */
  StreamTableEntry **StreamTable[MaxContexts];

  public:
  Stream(const StreamPrefetcherParams &p);
  ~Stream();
  /* Function called by cache controller to initiate 
     the stream training process
  */
  // void calculatePrefetch(const PacketPtr &pkt, std::vector<AddrPriority> &addresses);
  void calculatePrefetch(const PrefetchInfo &pfi, std::vector<AddrPriority> &addresses);
};

} // namespace prefetch
} // namespace gem5

#endif // __MEM_CACHE_PREFETCH_STREAM_HH__
```
---
## 二、修改Prefetcher.py
文件路径：./src/mem/cache/prefetch/Prefetcher.py

新增代码块
```python
class StreamPrefetcher(QueuedPrefetcher):
    type = 'StreamPrefetcher'
    cxx_class = 'gem5::prefetch::Stream'
    cxx_header = "mem/cache/prefetch/stream.hh"
    table_sets = Param.Int(16, "Number of sets in PC lookup table")
    table_assoc = Param.Int(4, "Associativity of PC lookup table")
    tableSize = Param.Int(8, "Number of sets in PC lookup table")
    distance = Param.Int(5, "Associativity of PC lookup table")
    use_master_id = Param.Bool(True, "Use master id based history")
    degree = Param.Int(4, "Number of prefetches to generate")
```

## 三、修改Sconscript
文件路径：./src/mem/cache/prefetch/Sconscript

```python
SimObject('Prefetcher.py', sim_objects=[
    'BasePrefetcher', 'MultiPrefetcher', 'QueuedPrefetcher',
    'StridePrefetcherHashedSetAssociative', 'StridePrefetcher', 
    'TaggedPrefetcher', 'IndirectMemoryPrefetcher', 'SignaturePathPrefetcher',
    'SignaturePathPrefetcherV2', 'AccessMapPatternMatching', 'AMPMPrefetcher',
    'DeltaCorrelatingPredictionTables', 'DCPTPrefetcher',
    'IrregularStreamBufferPrefetcher', 'SlimAMPMPrefetcher',
    'BOPPrefetcher', 'SBOOEPrefetcher', 'STeMSPrefetcher', 'PIFPrefetcher'])
```
修改为
```python
SimObject('Prefetcher.py', sim_objects=[
    'BasePrefetcher', 'MultiPrefetcher', 'QueuedPrefetcher',
    'StridePrefetcherHashedSetAssociative', 'StridePrefetcher', 'StreamPrefetcher',
    'TaggedPrefetcher', 'IndirectMemoryPrefetcher', 'SignaturePathPrefetcher',
    'SignaturePathPrefetcherV2', 'AccessMapPatternMatching', 'AMPMPrefetcher',
    'DeltaCorrelatingPredictionTables', 'DCPTPrefetcher',
    'IrregularStreamBufferPrefetcher', 'SlimAMPMPrefetcher',
    'BOPPrefetcher', 'SBOOEPrefetcher', 'STeMSPrefetcher', 'PIFPrefetcher'])

Source('stream.cc')
```
---
重新编译之后就能使用`StreamPrefetcher`参数配置预取器了。


```json
"prefetcher": {
                "type": "StreamPrefetcher",
                "cxx_class": "gem5::prefetch::Stream",
                "name": "prefetcher",
                "path": "system.l2.prefetcher",
                "block_size": 64,
                "cache_snoop": false,
                "clk_domain": "system.cpu_clk_domain",
                "degree": 1,
                "distance": 5,
                "eventq_index": 0,
                "latency": 1,
                ...
}
```