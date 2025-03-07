---
title: 实现高性能的本地缓存库
date: 2024-08-05 22:40:01
tags:
- GO
- 本地缓存
- LRU
- 高性能
---

> 在日常高流量场景中(读多写少场景)，经常会使用本地缓存来应对热点流量，保障系统的稳定。可是你有没有好奇过它底层是怎么实现的？数据是如何管理的？如果你来设计一个缓存库，你会如何设计?
<!-- more -->


# 他山之石，可以攻玉
在开始之前，借助开源社区了解主流缓存库的种类、设计思想以及适用场景是一个明智的做法。通过这样的调研，可以了解到不同缓存库的特点和优势，并从中汲取经验，以设计出符合自己需求的缓存库。 为了方便学习和理解，我对主流库做了详细调研并整理出以下多维度对比图，帮助你更清晰地了解不同缓存库之间的差异和优势。

![主流缓存库对比](/images/go_localcaches_compare.png)
上述中比较有意思的是Zero-Gc这个概念，我总结下关键信息:  
<strong>如何实现Zero-GC?</strong>
1. 完全避免GC: 采用syscall.MMap申请堆外内存，gc就不会扫描
2. 规避GC扫描策略:
- 数组(固定了指针数量) + map[uint64]uint32(非指针) + []byte(参考freecache) 
- slice + 非指针的map + ringbuffer(参考bigcache)


<strong>如何选择？</strong>
- 读写性能要求? 比如ristretto底层依赖channel,Get很快，但是Set如果是同步模式，会较慢需要评估。
- gc敏感度, 需要压测看业务是否能接受。
- 过期时间配置是否灵活，有些库甚至都不支持过期时间，不过这还得取决于使用场景需要自行评估。
- 业务匹配度，比如大部分业务ristretto更适合，支持泛型、使用门槛低，不过有一定的gc压力；再比如apiCache场景，只是简单的取出缓存写socket，无序序列化，那更适合bigCache，不过bigCache读的时候存在内存拷贝，需要留意;


综上，没有一个缓存库适用于所有场景和问题, 每个缓存库的诞生都是为了解决特定场景下的特定问题, 不过这些问题种类不多主要分为以下几类:
- 锁竞争。全局锁导致大量请求都在抢锁、休眠，严重影响性能
- 数据淘汰。内存资源有限，必须要按特定策略淘汰数据
- GC问题。存储海量对象时，GC扫描的影响不容小觑

----


# 实践出真知
接下来围绕上述三个问题来设计我们自己的高性能本地缓存库。
## 设计目标
- 高性能, 减少锁竞争
- 使用简单，配置不能太复杂，要开箱即用
- 支持按key设置过期时间
- 支持自动淘汰(LRU)
- 不要求Zero-GC, 但也应该尽量减少GC

## 设计思路
- 锁竞争: 读写锁 + 数据分片
- 数据淘汰: LRU
- 高性能: 合并写操作; 批量更新;
- GC优化: 我们的目标是减少GC，尽量减少对象分配

## 详细设计
### API设计
```golang

type Cache interface {
	Set(k string, v any, ttl time.Duration) bool
	Get(k string) (v any, err error)
	Del(k string)
	Len() uint64
	Close()
}
```
### 核心数据结构
#### cache
cache中核心结构为store、policy、expireKeyTimers模块, store负责存储引擎的实现，policy负责淘汰机制，expireKeyTimers管理过期清理的定时任务，这三者共同组成了缓存库基础骨架。
```golang
type cache struct {
	size int

	store            store   // 读写锁 + 数据分片
	policy           policy  //链表淘汰策略，LRU等
	ekt              *expireKeyTimers //维护key的定期清理任务
	accessUniqBuffer *uniqRingBuffer // 合并Get操作，降低对链表的移动

	accessEvtCh chan []*list.Element //批量Get操作，支持批量更新链表
	updateEvtCh chan *entExtendFunc  //合并对链表的Update
	addEvtCh    chan *entExtendFunc  //合并写操作(包含链表和map)
	delEvtCh    chan *keyExtendFunc  //合并对链表的Del

	isSync     bool //同步标识，会阻塞等待至写成功之后
	setTimeout time.Duration //阻塞等待超时时间
}
```
#### store - 存储引擎实现
store 提供增删改查的接口，可以根据自己的需求实现对应的接口，比如我们这里用就是shardedMap, 通过分片来降低锁的粒度, 减少锁竞争。
```azure
type store interface {
    set(k string, v any)
    get(k string) (any, bool)
    del(k string)
    len() uint64
    clear()
}
type shardedMap struct {
    shards []*safeMap
}
    
type safeMap struct {
	mu   sync.RWMutex
	data map[string]any
}

```

#### policy - 淘汰机制
淘汰机制主要是在对数据增删改查时，通过一定的策略来移动链表元素，以保证活跃的缓存项留在内存中，同时淘汰不活跃的缓存项。常见淘汰策略有LRU、LFU等。LRU较简单，可以通过标准库中的list实现policy接口实现。
```golang
// 缓存项，包含 key,value,过期时间
type entry struct {
    key      string
    val      any
    expireAt time.Time
    mu sync.RWMutex
}
type policy interface {
	isFull() bool
	add(*entry) (*list.Element, *list.Element) // 返回新增, victim:淘汰的entry
	remove(*list.Element)
	update(*entry, *list.Element)
	renew(*list.Element)
	batchRenew([]*list.Element)
}
```

#### expireKeyTimers - 过期时间
这个模块主要维护过期key的定时清理任务。底层主要依赖第三方[时间轮库](https://github.com/RussellLuo/timingwheel)来管理定时任务
```golang
type expireKeyTimers struct {
	mu     sync.RWMutex
	timers map[string]*timingwheel.Timer

	tick      time.Duration
	wheelSize int64
	tw        *timingwheel.TimingWheel
}
```
### hash函数选型
[常见hash函数压测对比](https://github.com/smallnest/hash-bench)
![常见hash函数](/images/hash_func.png)

----
fnv64 vs xxhash  
测试机器: mac-m1, go benchmark结果

| hash函数 | fnv64a  | github.com/cespare/xxhash/v2  |
|--------|---------|---------|
| 8字节    | 5.130 ns/op | 8.817 ns/op |
| 16字节   | 7.928 ns/op|   7.464 ns/op |
| 32字节   | 17.17 ns/op  | 14.22 ns/op|


### 高性能优化
#### 写操作
<strong>隔离:</strong>  按channel隔离增、删、改  
<strong>同步转异步:</strong>  链表并发写操作，改为异步单协程更新
<strong>支持非阻塞</strong>

#### 读操作
<strong>批量操作:</strong> 采用ringbuffer，批量更新链表

#### 内存优化
采用sync.Pool池化ringbuffer对象，避免频繁创建对象

## 压测对比
[代码地址](https://github.com/codingWhat/armory/tree/main/cache/localcache)
同步模式:


| 压测case                                 | 操作次数   | 单次耗时 (ns/op) | 内存分配 (B/op) | 分配次数 |
|------------------------------------------|------------|------------------|-----------------|----------|
| BenchmarkSyncMapSetParallelForStruct-10  | 1,576,032  | 719.3            | 76              | 5        |
| **BenchmarkRistrettoSetParallelForStruct-10** | 716,690    | 1,642            | 369             | 11       |
| BenchmarkFreeCacheSetParallelForStruct-10| 2,122,884  | 562.7            | 61              | 4        |
| BenchmarkBigCacheSetParallelForStruct-10 | 2,206,600  | 546.9            | 200             | 4        |
| **BenchmarkLCSetParallelForStruct-10**   | 914,626    | 1,279            | 282             | 9        |
| BenchmarkSyncMapGetParallelForStruct-10  | 3,933,157  | 305.5            | 24              | 1        |
| BenchmarkFreeCacheGetParallelForStruct-10| 2,159,518  | 577.2            | 263             | 7        |
| BenchmarkBigCacheGetParallelForStruct-10 | 2,218,573  | 539.1            | 279             | 8        |
| **BenchmarkRistrettoGetParallelForStruct-10** | 3,195,711  | 379.0            | 31              | 1        |
| **BenchmarkLCGetParallelForStruct-10**   | 2,233,429  | 530.5            | 31              | 2        |

总结:
- 读取性能: LC 和 SyncMap 在读取操作中表现最佳，具有较低的耗时和内存分配。
- 写入性能: BigCache 和 FreeCache 在写入操作中表现较好，LC、Ristretto因为channel缘故，写入性能较差。
- 内存效率: SyncMap/Ristretto 在Get操作中的内存分配最低，FreeCache在Set操作中内存分配最低, 整体上syncMap占用最低。

非同步模式:
读、写、耗时、内存分配逐渐接近主流库的, 但是写存在失败的概率，需要按场景权衡。

| 压测case                                   | 操作次数  | 单次耗时 (ns/op) | 内存分配 (B/op) | 分配次数 (allocs/op) |
|-------------------------------------------|----------|-----------------|-----------------|---------------------|
| BenchmarkSyncMapSetParallelForStruct-10   | 1256974  | 958.8           | 78              | 5                   |
| **BenchmarkRistrettoSetParallelForStruct-10** | 2372764  | 505.6           | 143             | 4                   |
| BenchmarkFreeCacheSetParallelForStruct-10 | 2117694  | 554.2           | 61              | 4                   |
| BenchmarkBigCacheSetParallelForStruct-10  | 2130927  | 547.5           | 206             | 4                   |
| **BenchmarkLCSetParallelForStruct-10**         | 2115037  | 567.1           | 158             | 6                   |
| BenchmarkSyncMapGetParallelForStruct-10   | 3854450  | 305.2           | 23              | 1                   |
| BenchmarkFreeCacheGetParallelForStruct-10 | 2152428  | 560.6           | 263             | 7                   |
| BenchmarkBigCacheGetParallelForStruct-10  | 2202607  | 539.5           | 279             | 8                   |
| **BenchmarkRistrettoGetParallelForStruct-10**  | 3445798  | 349.7           | 31              | 1                   |
| **BenchmarkLCGetParallelForStruct-10**         | 2453848  | 505.4           | 30              | 2                   |


## 未来展望
继续优化写场景下，临时对象的管理，减少耗时操作和频繁的内存申请。