---
title: 你了解GoGC么？
date: 2023-07-08 16:46:25
tags:
- GO
- GC
---
> 本文是关于Golang的GC演进以及GC原理总结。
<!-- more -->


# GC演进历史
关键节点:
- Go1：串行三色标记清扫
- Go1.5：并发标记清扫，停顿时间在一百毫秒以内
- Go1.8：引入混合写屏障,停顿时间在半个毫秒左右
- Go1.17: 采用内存归还策略:MADV_DONTNEED(立即归还), 在这之前是MADV_FREE(延迟归还，会导致内存误报)
详细可以看:
https://www.topgoer.cn/docs/goquestions/goquestions-1cjh5mftsd3dm


# GC原理
GC主要分为两部分
- mutator, 用户态的代码，对GC来说就是做引用的插入或删除，所以叫赋值器
- collector, 垃圾回收器, 扫描+清理。
Go在1.8之前是基于三色标记和插入屏障来回收垃圾对象，之后引入了混合写屏障，消除了插入屏障在栈空间的重扫和STW损耗，在性能上有了进一步提升。

## GC触发时机:
- 定时调用: sysmon线程定期执行, 依据是否满足debug.SetGCPercent阈值执行gc
- 申请堆空间时调用: mallocgc
- 手动触发: runtime.GC()

## 三色标记算法
会将所有对象分为三种颜色，白灰黑，分别代表三种不同状态。白色：未扫描，灰色：已扫描，黑色：已标记。
1. GC启动时，所有对象初始状态都是白色，GC会对根对象集合遍历，会将遍历到的对象置为灰色，并将其放到灰色队列中，直到遍历完所有根对象。
2. 接下来会遍历灰色队列，会将灰色对象变为黑色，此时如果灰色对象有next节点，就将next节点变为灰色，写入灰色队列中, 重复这个步骤直到灰色队列为空。
3. 最后剩余的白色对象就是垃圾对象，在sweep阶段stw被清除。

tips: 
1. 三色标记不是Go特有的，Java也有, 也算是一个主流的垃圾回收算法。
2. 根对象集合: 全局变量、协程栈中变量、分配到堆空间的变量

## 为什么需要屏障？
在早期，GC会将所有用户态G停止运行(STW)，开始GC的扫描和清除工作，之后再唤醒用户态G,如果扫描对象或者清理对象过多, GC占用时间就过长，这对耗时敏感的服务来说是不可接受的， 因此Go团队着手优化GC，引入屏障机制，来最大化的让用户态goroutine和GCGoroutine并发执行。 为了能让mutator和collector并发执行(扫描阶段)，需要满足以下两个之一条件:

强三色不变式：
- 黑色对象不能插入白色对象，只能将白色对象置灰

弱三色不变式:
- 黑色对象可以引用白色对象，但是白色对象必须被灰色对象引用(直接或者间接的，中间隔白色对象)

为什么需要满足这俩条件? 如果不满足，GC可能会把正在引用的对象给误清理
举例:
1. 比如栈空间已经扫描完了，此时栈空间都是黑色对象，此时插入一个引用(白色对象)，GC会继续扫描进入sweep阶段，最后会直接把白色对象给清理。
2. 如果白色对象被灰色对象引用，那就好办了，会在遍历灰色对象时一定能遍历到白色对象保证其不会被抛弃(即使白色被黑色引用)

### 删除屏障(了解即可,go未采用):
- 启动前，会做快照，被删对象如是白色会变灰色，灰色的话会变黑色
- 回收精度低，一个被删除对象就算没有被引用，本次GC不会被清理，下一轮GC才会被清理

### 插入屏障(go1.8之前)
- 触发场景:  堆空间(栈空间不会触发) 。
- 满足强三色，如果黑色对象引用白色对象，会触发插入屏障
优点: 精度高
缺点: 需要对栈空间STW，栈空间重新扫描一遍，防止新插入的对象被清理。

### 混合写屏障(go1.8)
1. 优点
- GC启动时，会将栈空间的对象变为黑色，之后新增对象均为黑色
- 栈空间不触发屏障( 栈对象之间操作插入删除，不会有屏障效果,直接删除或插入)。
- [堆空间] 中插入删除对象[堆空间]，被插入、删除对象都会变为灰色
- 交叉的这种，比如栈(黑对象)→堆(白)，无屏障效果直接插入白色；堆(灰对象)→栈(黑色)，无效果。
- 解决了栈重扫的问题

2. 缺点: 还是存在精度问题，当删除对象引用时，被删除对象只能下一轮被清理

tips: 堆空间对象→栈对象，插入或删除都无作用
Golang中的混合写屏障满足`弱三色不变式`，结合了删除写屏障和插入写屏障的优点，只需要在开始时并发扫描各个goroutine的栈，使其变黑并一直保持，这个过程不需要STW，而标记结束后，因为栈在扫描后始终是黑色的，也无需再进行re-scan操作了，减少了STW的时间。
插入屏障→混合写屏障



# GC调优

Gc关注指标
- CPU使用率, GC的使用率不能过高，要提升mutator的使用率
- GC频率, 如果频率过快，需要调整GCPercent
- GC的STW时间, 需要关注mallocgc 查看分配对象的逻辑是否可以优化


因此GC调优主要从两方面着手:
- GC频率控制
- 内存管理

## GC频率控制:
- (不推荐)[ballast](https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap/): 压舱石技术，开辟一个大内存，可以一定程度避免频繁GC
- (不推荐)[GOGCTuner](https://github.com/cch123/gogctuner) 每次GC时动态调整GCPercent.
- (推荐)GOGC或者`debug.SetGCPercent()` + `debug.SetMemoryLimit()`

如何选择合适的GCPercent值？
1. 先确定线上平均NextGC值
2. 设置合理的`debug.SetMemoryLimit()`。（机器内存 / 程序占用内存 * 2 )* 100%  一定不能超过这个值!
3. 动态调整GCPercent、MemoryLimit，观察GC频次, 选最优的GCPercent值

Next_GC(下次触发GC阈值) = liveset(上次GC之后内存空间) + liveset * GCPercent
举例: 当前程序100M, `debug.SetGCPercent(100)`, NextGC = 100 + 100 = 200M

### ！！注意别忘了设置 `debug.SetMemoryLimit()` or `GOMEMLIMIT` 防止OOM

## 内存管理
1. sync.Pool 对象池复用对象
2. map/slice 一次性申请提前分配好
ok:

```go
ret := make([]int, 0 ,10)
for i :=0; i < 10; i++ {
	ret[i] = i
}
```

Wrong:
```go
var ret []int
for i :=0; i < 10; i++ {
	ret = append(ret, i)
}
```
3. 字符串处理 string.Builder优先
4. 控制Goroutine数量，Goroutine太多，会导致GC扫描的栈很多，也会影响mutator的CPU使用率


# 当前GC存在的问题：
如果Goroutine很多，
- 并且牵扯内存申请(mallocgc),MarkAssist 停顿时间长, 随着而来sweep内存清理也会长，
- 扫描的goroutine栈就多，cpu使用率也高


# 和其他语言GC对比
感兴趣可以看:[GC对比](https://www.topgoer.cn/docs/goquestions/goquestions-1cjh5nmtkbc4o)


# Resources:
[刘丹冰GC文章,值得细读](https://github.com/aceld/golang/blob/main/5、Golang三色标记+混合写屏障GC模式全分析.md)