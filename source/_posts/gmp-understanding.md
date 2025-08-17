---
title: Go语言GMP调度器深度解析
date: 2023-08-09 13:27:45
tags:
- GO
- GO-GMP
- Go调度原理
---
## 调度器发展历程

Go语言调度器的核心职责是通过高效的线程复用机制来执行大量的Goroutine。当前的GMP模型是经过多次迭代优化的结果。

### 早期GM模型的限制
早期调度器采用GM二元模型，存在以下性能瓶颈：
1. **全局锁竞争**：所有M（Machine）竞争同一个全局运行队列，随着Goroutine数量增长，锁竞争愈发严重
2. **CPU利用率低**：M执行系统调用或阻塞操作时会休眠，绑定在该M上的Goroutine无法被其他M接管
3. **调度开销大**：频繁的全局队列访问导致缓存miss和上下文切换开销

### GMP模型的优势
为解决上述问题，Go团队重新设计了调度器架构，引入Processor（P）概念，形成了当前的GMP三元模型，实现了：
- 本地队列减少锁竞争
- Work-Stealing负载均衡
- 系统调用时的P-M解绑机制

## 调度器核心概念

### Processor (P)
Processor是GMP模型的核心创新，承担以下关键职责：

#### 核心功能
1. **本地运行队列管理**：每个P维护独立的本地运行队列（`runq`），避免全局锁竞争
2. **动态绑定机制**：当M因系统调用或阻塞操作休眠时，P与M解绑，寻找空闲M继续执行队列中的Goroutine
3. **调度上下文**：保存调度相关的元数据和状态信息

#### P的状态转换
P在运行时会在以下状态间转换：
- `_Pidle`：空闲状态，等待绑定M
- `_Prunning`：运行状态，已绑定M并在执行Goroutine
- `_Psyscall`：系统调用状态，M正在执行系统调用
- `_Pgcstop`：GC停止状态，暂停调度等待GC完成
- `_Pdead`：死亡状态，P被销毁

![gmp_p_status](/images/gmp_p_status.png)

### Goroutine (G)
Goroutine是Go语言的用户级线程，具有轻量级和高并发特性。

#### 基本状态模型
从调度器角度，Goroutine具有三种核心状态：
- **Waiting**：阻塞状态，等待I/O操作或系统调用完成
- **Runnable**：就绪状态，位于运行队列中等待调度
- **Executing**：执行状态，正在M上运行

#### 详细状态转换
Goroutine的完整生命周期包含以下状态转换：

**创建阶段**：
`_Gidle`（空闲池） → `_Gdead`（被分配） → `_Grunnable`（初始化完成） → `_Grunning`（开始执行）

**运行阶段**：
- `_Grunning` → `_Gsyscall`（系统调用） → `_Grunning`（调用返回）
- `_Grunning` → `_Gwaiting`（阻塞等待） → `_Grunnable`（条件满足）

**销毁阶段**：
当Goroutine执行完毕时，调用链为：`runtime.goexit1` → `goexit0`
1. 切换到G0栈空间
2. 清理Goroutine数据结构
3. 解除与M的绑定关系
4. 状态从`_Grunning`更新为`_Gdead`
5. 回收到空闲Goroutine池

![g_status](/images/g_status.png)

### 特殊对象与全局管理

#### 系统初始对象
- **M0**：主线程对应的Machine，存储在全局变量`runtime.m0`中
  - 负责执行运行时初始化操作
  - 启动第一个Goroutine（通常是`runtime.main`）
  - 初始化完成后与普通M具有相同行为

- **G0**：每个M的调度Goroutine
  - 专门用于执行调度逻辑，不执行用户代码
  - 拥有固定大小的栈空间（通常8KB）
  - 在执行系统调用或调度切换时提供栈空间
  - 全局G0特指M0的调度Goroutine

- **P0**：首个Processor，与M0绑定完成系统启动

#### 全局管理结构
- **allgs**：全局Goroutine切片，记录系统中所有G的引用
- **allm**：全局Machine切片，管理所有操作系统线程
- **allp**：全局Processor切片，维护所有逻辑处理器
- **sched**：全局调度器结构，包含：
  - 空闲M队列（`midle`）
  - 空闲P队列（`pidle`）
  - 全局运行队列（`runq`）
  - 调度统计信息

![g0-p0-m0](/images/g0-p0-m0.png)


## GMP调度机制详解

### 系统启动流程
Go程序启动时按以下步骤初始化调度器：

1. **M0和G0初始化**：创建主线程M0及其调度Goroutine G0
2. **P初始化**：根据`GOMAXPROCS`（默认为CPU核心数）创建相应数量的P
3. **绑定关系建立**：P0与M0、G0建立绑定关系
4. **空闲队列管理**：剩余P进入全局空闲队列等待分配
5. **启动第一个用户Goroutine**：创建G1执行`runtime.main`函数，加入P0本地队列
6. **调度循环启动**：M0的G0开始执行调度主循环

### Goroutine创建与队列管理

#### 本地队列结构
每个P维护两级本地队列结构：

**队列容量设计**：
- `runnext`：单槽，存储优先执行的Goroutine
- `runq`：环形缓冲区，容量256个Goroutine
- 总容量：257个Goroutine（1 + 256）

**队列语义**：
- `runnext`：高优先级槽位，下次调度优先执行
- `runq`：FIFO环形队列，按先进先出顺序执行

#### 队列溢出处理
当本地队列达到容量上限时：
1. 新创建的Goroutine抢占`runnext`槽位
2. 被抢占的Goroutine与`runq`前半部分（128个）一起转移到全局队列
3. 这种设计平衡了本地调度效率和全局负载均衡

![Goroutine和P交互细节](/images/g_to_p.png)

#### 创建流程
Goroutine创建通过以下调用链完成：
```
go func() -> newproc() -> runqput() -> P.runnext/runq
```


### Goroutine调度策略

调度器核心逻辑位于`runtime/proc.go`的`schedule()`→`findRunnable()`方法，采用多级调度策略：

#### 调度优先级序列
1. **公平性保障**：每61次调度（`SchedTick % 61 == 0`）强制从全局队列获取，防止饥饿
2. **本地队列优先**：从`runnext`和`runq`获取，最大化缓存局部性
3. **全局队列补充**：本地队列为空时从全局队列批量获取
4. **网络轮询集成**：从netpoll获取就绪的网络Goroutine，剩余的放入全局队列
5. **Work-Stealing**：从其他P偷取一半Goroutine，实现负载均衡

#### 公平性机制
为避免全局队列中的Goroutine长期得不到调度，调度器引入公平性计数器：
- `SchedTick`：每次调度递增的全局计数器
- 当`SchedTick % 61 == 0`时，强制优先调度全局队列
- 该机制确保全局队列中的Goroutine最多等待61个调度周期

#### 调度流程图解
![gmp_global_runq_probability](/images/gmp_global_runq_random.png)
![get from local runq](/images/gmp_local_runq.png)
![get_from_global_runq](/images/get_from_global_runq.png)
![get_form_netpoll](/images/get_form_netpoll.png)
![steal_from_other_p](/images/steal_from_other_p.png)


### Work-Stealing负载均衡机制

当P的本地队列为空且全局队列也无可用Goroutine时，启动Work-Stealing机制实现动态负载均衡。

#### 偷取策略
- **随机选择**：最多尝试4次，每次随机选择一个目标P
- **适应性偷取**：优先从繁忙的P偷取，避免影响轻载P
- **批量转移**：一次偷取目标P队列的一半，减少偷取频率

![stealwork](/images/stealwork.png)

#### 核心算法：runqgrab
Work-Stealing的关键实现是`runqgrab`函数，采用无锁并发算法：

```golang
func runqgrab(pp *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {
    for {
        // 原子读取队列头尾指针，确保内存可见性
        h := atomic.LoadAcq(&pp.runqhead) // 消费者同步点
        t := atomic.LoadAcq(&pp.runqtail) // 生产者同步点
        
        // 计算待偷取数量（队列一半）
        n := t - h
        n = n - n/2
        
        // 批量复制Goroutine到偷取者队列
        for i := uint32(0); i < n; i++ {
            g := pp.runq[(h+i)%uint32(len(pp.runq))]
            batch[(batchHead+i)%uint32(len(batch))] = g
        }
        
        // CAS原子更新头指针，提交偷取操作
        if atomic.CasRel(&pp.runqhead, h, h+n) {
            return n
        }
        // CAS失败说明发生竞争，重试
    }
}
```

#### 算法特点
1. **无锁设计**：使用原子操作和CAS避免锁竞争
2. **内存屏障**：LoadAcq/CasRel确保正确的内存顺序
3. **失败重试**：CAS失败时自动重试，处理并发竞争
4. **批量操作**：一次转移多个Goroutine，提高效率


## 抢占式调度机制

Go调度器采用混合调度策略，结合协作式和抢占式调度的优势。

### 协作式与抢占式对比

**协作式调度**：
- Goroutine主动调用`runtime.Gosched()`让出CPU
- 在函数调用时检查栈溢出触发调度点
- 优点：上下文切换开销小，任务执行连续性好
- 缺点：依赖程序配合，可能导致某些Goroutine长期占用CPU

**抢占式调度**：
- 运行时系统强制中断正在执行的Goroutine
- 通过时间片轮转和信号机制实现
- 优点：保证调度公平性，防止饥饿问题
- 缺点：频繁中断增加调度开销

![coop_vs_retake](/images/coop_vs_retake.png)

### 性能特征分析
1. **执行延迟**：协作式调度下短任务执行延迟更低
2. **抢占频率**：抢占式调度中断次数较多，增加调度开销  
3. **公平性权衡**：抢占虽然增加了长任务的延迟，但保证了短任务的及时响应

### 系统监控线程（Sysmon）

Sysmon是Go运行时的系统级监控线程，运行在独立的操作系统线程上，不绑定任何P，负责全局系统监控任务。

#### 核心职责
1. **网络轮询（netpoll）**：检查网络文件描述符事件，将就绪的网络Goroutine加入调度队列
2. **抢占控制（retake）**：监控长时间运行的Goroutine，触发抢占调度
3. **垃圾回收（forcegc）**：定期触发垃圾回收，防止内存积累
4. **内存清理（scavenge）**：回收未使用的内存页面给操作系统

#### 工作模式
- **独立线程**：不参与GMP调度，避免被阻塞影响监控功能
- **周期性执行**：采用指数退避算法调整监控间隔，平衡监控效果和CPU开销
- **动态间隔**：系统空闲时增加监控间隔，繁忙时缩短间隔


#### 抢占机制详解

##### 抢占触发条件
Sysmon遍历所有P，对于处于`_Prunning`和`_Psyscall`状态的P，当同时满足以下条件时触发抢占：

1. **时间阈值**：P对应的M运行时间超过10ms（forcePreemptNS）
2. **队列非空**：P的本地运行队列中有待调度的Goroutine
3. **系统繁忙**：没有空闲的P和自旋的M，系统处于满负载状态

这些条件确保抢占只在必要时发生，避免不必要的调度开销。

##### 抢占执行流程
**对于`_Prunning`状态的P**：
1. 调用`preemptone()`设置抢占标志
2. 设置`gp.stackguard0 = stackPreempt`
3. 如果支持异步抢占，发送`SIGURG`信号

**对于`_Psyscall`状态的P**：
1. 执行基本抢占设置
2. 调用`handoffp()`将P移交给其他M
3. 原M继续执行系统调用，P可立即投入调度

#### 关键源码实现

```golang
func retake(now int64) uint32 {
    n := 0
    lock(&allpLock)
    // 遍历所有的P
    for i := int32(0); i < gomaxprocs; i++ {
        _p_ := allp[i]
        if _p_ == nil {
            continue
        }
        // 用于sysmon线程记录被监控P的系统调用时间和运行时间
        pd := &_p_.sysmontick
        s := _p_.status
        sysretake := false
        
        if s == _Prunning || s == _Psyscall {
            // P处于运行状态，检查是否运行得太久了
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
            } else if pd.schedwhen+forcePreemptNS <= now {
                // pd.schedtick == t 说明这段时间未发生过调度
                // 同一个goroutine一直在运行，检查是否连续运行超过了10ms
                preemptone(_p_)
                sysretake = true
            }
        }
        
        if s == _Psyscall {
            // 系统调用状态的特殊处理
            t := int64(_p_.syscalltick)
            if !sysretake && int64(pd.syscalltick) != t {
                pd.syscalltick = uint32(t)
                pd.syscallwhen = now
                continue
            }
            
            // 满足以下条件之一则抢占该P：
            // 1. P的运行队列里面有等待运行的goroutine
            // 2. 没有空闲的P
            // 3. 系统调用时间超过10ms
            if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && 
               pd.syscallwhen+10*1000*1000 > now {
                continue
            }
            
            unlock(&allpLock)
            incidlelocked(-1)
            if atomic.Cas(&_p_.status, s, _Pidle) {
                if trace.enabled {
                    traceGoSysBlock(_p_)
                    traceProcStop(_p_)
                }
                n++
                _p_.syscalltick++
                // 寻找新的M接管P
                handoffp(_p_)
            }
            incidlelocked(1)
            lock(&allpLock)
        }
    }
    unlock(&allpLock)
    return uint32(n)
}

func preemptone(_p_ *p) bool {
    mp := _p_.m.ptr()
    if mp == nil || mp == getg().m {
        return false
    }
    gp := mp.curg
    if gp == nil || gp == mp.g0 {
        return false
    }
    
    gp.preempt = true
    
    // 设置抢占标志：将stackguard0设置为stackPreempt
    // 每次goroutine函数调用都会检查栈溢出，通过这种方式实现抢占检查
    gp.stackguard0 = stackPreempt
    
    // 如果支持异步抢占，发送抢占信号
    if preemptMSupported && debug.asyncpreemptoff == 0 {
        _p_.preempt = true
        preemptM(mp)
    }
    
    return true
}
```

### P-M解绑机制（Handoff）

当Goroutine发生阻塞、系统调用或被抢占时，采用P-M解绑机制最大化资源利用率。

#### 核心思想
- **P的连续性**：P作为调度上下文，应尽可能保持忙碌状态
- **M的灵活性**：M作为执行载体，可以在阻塞时释放资源
- **动态绑定**：根据系统负载动态调整P-M绑定关系

#### 实现机制
```golang
func handoffp(_p_ *p) {
    // 如果本地或全局队列有工作，立即分配新的M
    if !runqempty(_p_) || sched.runqsize != 0 {
        startm(_p_, false)
        return
    }
    
    // 系统繁忙时启动自旋M寻找工作
    if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 {
        startm(_p_, true) // 启动自旋M
        return
    }
    
    // 无工作时将P放入空闲队列
    pidleput(_p_)
}
```

#### 关键优化
1. **工作检测**：优先检查是否有待处理的Goroutine
2. **自旋机制**：在系统繁忙时启动自旋M主动寻找工作
3. **资源回收**：空闲时及时回收P到全局池，避免资源浪费

## 两种抢占机制对比

Go调度器实现了两种抢占机制，从协作式逐步演进到支持异步抢占。

### 基于协作的抢占式调度

协作式抢占是Go早期采用的抢占机制，依赖函数调用时的栈检查实现。

#### 实现原理
编译器在每个函数入口插入栈溢出检查代码，通过复用这一机制实现抢占：

1. **栈检查复用**：利用现有的`runtime.morestack`栈检查逻辑
2. **抢占标志**：将`gp.stackguard0`设置为`stackPreempt`特殊值
3. **主动让出**：检测到抢占标志时调用`gopreempt_m()`让出CPU

#### 触发条件
- Sysmon检测到Goroutine运行时间超过10ms
- 函数调用时触发栈检查，发现抢占标志


#### 局限性
协作式抢占存在明显缺陷：
- **依赖函数调用**：如果Goroutine中包含长时间循环且无函数调用，无法被抢占
- **抢占延迟**：只能在函数调用时检查，抢占时机不够灵活
- **GC阻塞**：垃圾回收时可能因为无法抢占某些Goroutine而延迟

### 基于信号的异步抢占调度

Go 1.14引入异步抢占机制，解决协作式抢占的局限性。

#### 实现原理
异步抢占通过操作系统信号机制实现强制中断：

1. **信号注册**：注册`SIGURG`信号处理函数`doSigPreempt`
2. **信号发送**：Sysmon通过`preemptM()`向目标M发送抢占信号
3. **上下文修改**：信号处理函数修改被中断线程的执行上下文
4. **异步切换**：将执行流程重定向到`asyncPreempt`函数完成调度

#### 核心代码
```golang
func doSigPreempt(gp *g, ctxt *sigctxt) {
    if wantAsyncPreempt(gp) {
        if ok, newpc := isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()); ok {
            // 修改执行上下文，注入asyncPreempt调用
            ctxt.pushCall(abi.FuncPCABI0(asyncPreempt), newpc)
        }
    }
}

func asyncPreempt2() {
    gp := getg()
    gp.asyncSafePoint = true
    if gp.preemptStop {
        mcall(preemptPark)  // GC抢占
    } else {
        mcall(gopreempt_m)  // 常规抢占
    }
    gp.asyncSafePoint = false
}
```



#### 触发场景
异步抢占主要用于以下场景：
1. **GC阶段**：垃圾回收需要暂停所有Goroutine进行栈扫描
2. **运行时监控**：Sysmon检测到长时间运行的Goroutine
3. **紧急抢占**：系统资源紧张时的强制调度

#### 优势与意义
- **真正异步**：不依赖用户代码配合，可在任意执行点抢占
- **GC效率**：大幅提升垃圾回收的响应速度
- **调度公平性**：确保所有Goroutine都能获得执行机会
- **系统响应性**：提高整体系统的实时性和响应性

## 总结

Go语言的GMP调度器经过多年演进，已成为高并发场景下的优秀调度系统：

### 核心优势
1. **高效调度**：本地队列 + Work-Stealing实现负载均衡
2. **混合抢占**：协作式与异步抢占相结合，保证调度公平性
3. **动态适应**：P-M解绑机制最大化资源利用率
4. **垃圾回收集成**：与GC深度集成，支持低延迟垃圾回收

### 性能特征
- **低延迟**：Goroutine创建和切换开销极小
- **高吞吐**：支持百万级Goroutine并发
- **公平调度**：防止饥饿，保证调度公平性
- **自适应**：根据系统负载动态调整调度策略

## 参考资料

1. [Go语言设计与实现 - 调度器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)
2. [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw)
3. [Go: Asynchronous Preemption](https://medium.com/a-journey-with-go/go-asynchronous-preemption-b5194227371c)
4. [了解go在协程调度上的改进](https://cloud.tencent.com/developer/article/1938510)