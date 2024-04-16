---
title: Gem5模拟器配置L3缓存
date: 2024-04-16 20:54:38
tags: [Gem5]
---


Gem5默认不支持L3缓存相关配置，如果尝试用`--l3cache`、`--l3_size`等参数会发现找不到，需要进行修改才行，以下是具体的修改细节。
<!-- more -->

---

## 一、修改 Cache.py

文件路径：./configs/common/Caches.py


新增代码块：
```python
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency =32
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16
```
---
## 二、修改 CacheConfig.py
文件路径：./configs/common/Caches.py


```python
dcache_class, icache_class, l2_cache_class, walk_cache_class = ( 
    core.O3_ARM_v7a_DCache, 
    core.O3_ARM_v7a_ICache, 
    core.O3_ARM_v7aL2, 
    None , 
)
```
修改为
```python
dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = (
    core.O3_ARM_v7a_DCache,
    core.O3_ARM_v7a_ICache,
    core.O3_ARM_v7aL2,
    None,
    core.O3_ARM_v7aL3,
)
```
---
```python
else : 
    dcache_class, icache_class, l2_cache_class, walk_cache_class = ( 
        L1_DCache, 
        L1_ICache, 
        L2Cache, 
        None , 
    )
```
修改为
```python
else:
    dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class  = (
        L1_DCache,
        L1_ICache,
        L2Cache,
        None,
        L3Cache,
    )
```
---
```python
if options.l2cache:
    # Provide a clock for the L2 and the L1-to-L2 bus here as they
    # are not connected using addTwoLevelCacheHierarchy. Use the
    # same clock as the CPUs.
    system.l2 = l2_cache_class(
        clk_domain=system.cpu_clk_domain, **_get_cache_opts("l2", options)
    )

    system.tol2bus = L2XBar(clk_domain=system.cpu_clk_domain)
    system.l2.cpu_side = system.tol2bus.mem_side_ports
    system.l2.mem_side = system.membus.cpu_side_ports
```
修改为
```python

if options.l2cache and options.l3cache:
    # Provide a clock for the L2 and the L1-to-L2 bus here as they
    # are not connected using addTwoLevelCacheHierarchy. Use the
    # same clock as the CPUs.
    system.l2 = l2_cache_class(
        clk_domain=system.cpu_clk_domain, **_get_cache_opts("l2", options)
    )
    system.l3 = l3_cache_class(
        clk_domain=system.cpu_clk_domain, **_get_cache_opts("l3", options)
    )

    system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
    system.tol3bus = L3XBar(clk_domain = system.cpu_clk_domain)

    system.l2.cpu_side = system.tol2bus.mem_side_ports
    system.l2.mem_side = system.tol3bus.cpu_side_ports

    system.l3.cpu_side = system.tol3bus.mem_side_ports
    system.l3.mem_side = system.membus.cpu_side_ports

elif options.l2cache:
    # Provide a clock for the L2 and the L1-to-L2 bus here as they
    # are not connected using addTwoLevelCacheHierarchy. Use the
    # same clock as the CPUs.
    system.l2 = l2_cache_class(
        clk_domain=system.cpu_clk_domain, **_get_cache_opts("l2", options)
    )

    system.tol2bus = L2XBar(clk_domain=system.cpu_clk_domain)
    system.l2.cpu_side = system.tol2bus.mem_side_ports
    system.l2.mem_side = system.membus.cpu_side_ports

```
---
## 三、修改 XBar.py
文件路径：./src/mem/XBar.py
新增代码块：
```python
class L3XBar(CoherentXBar):
    # 256-bit crossbar by default
    width = 32

    # Assume that most of this is covered by the cache latencies, with
    # no more than a single pipeline stage for any packet.
    frontend_latency = 1
    forward_latency = 0
    response_latency = 1
    snoop_response_latency = 1
```
---
## 四、修改 BaseCPU.py
文件路径：./src/cpu/BaseCPU.py

```python
from m5.objects.XBar import L2XBar
```
修改为
```python
from m5.objects.XBar import L2XBar, L3XBar
```
---
新增代码块
```python
def addThreeLevelCacheHierarchy(
    self, ic, dc, l3c, iwc = None, dwc = None
):
    self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
    self.toL3Bus = L3XBar()
    self.connectCachedPorts(self.toL3Bus)
    self.l3cache = l3c
    self.toL2Bus.master = self.l3cache.cpu_side
    self._cached_ports = ['l3cache.mem_side']
```
---
## 五、修改 Options.py
文件路径：./configs/common/Options.py
```python
    parser.add_argument("--caches", action="store_true")
    parser.add_argument("--l2cache", action="store_true")
    parser.add_argument("--num-dirs", type=int, default=1)
```
修改为
```python
    parser.add_argument("--caches", action="store_true")
    parser.add_argument("--l2cache", action="store_true")
    parser.add_argument("--l3cache", action="store_true", default=False)
    parser.add_argument("--num-dirs", type=int, default=1)

    parser.add_argument(
        "--l3-hwp-type",
        default=None,
        choices=ObjectList.hwp_list.get_names(),
        help="""
                        type of hardware prefetcher to use with the L3 cache.
                        (if not set, use the default prefetcher of
                        the selected cache)""",
    )
```

---
重新编译之后就能使用`--l3cache`、`--l3_size`、`--l3-hwp-type`等参数配置L3缓存了。

```json
"l3": {
            "type": "Cache",
            "cxx_class": "gem5::Cache",
            "name": "l3",
            "path": "system.l3",
            "addr_ranges": [
                "0:18446744073709551615"
            ],
            "assoc": 16,
            "clk_domain": "system.cpu_clk_domain",
            "clusivity": "mostly_incl",
            "compressor": null,
            "data_latency": 34,
            "demand_mshr_reserve": 1,
            "eventq_index": 0,
            "is_read_only": false,
            "max_miss_count": 0,
            "move_contractions": true,
            "mshrs": 64,
            "power_model": [],
            ...
}
```