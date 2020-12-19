---
title: "Golang Mutex源码分析"
date: 2020-12-14T14:27:08+08:00
draft: false
---

Golang标准库中提供了[互斥锁]^(Mutex)的原语来解决并发资源竞争，这篇文章探讨了标准库
中Mutex的实现原理

<!--more-->

## 基础知识

### 信号量

[信号量]^(Semaphore) 是计算机科学家 Dijkstra 发明的数据结构，是在多线程环境下使用的一种设施，是可以用来保证两个或多个关键代码段不被并发调用。其本质是一个整数，有两个基本操作：

1. 申请acquire（也称为 wait、decrement 或 P 操作）:

    将信号量减 1，如果结果值为负则挂起协程，直到其他线程进行了信号量累加为正数才能恢复。如结果为正数，则继续执行。

2. 释放release（也称 signal、increment 或 V 操作）:

    将信号量加 1，如存在被挂起的协程，此时唤醒他们中的一个协程。

### CAS 操作

`CAS操作`是CPU指令提供的一个原子操作， 其全程为 `Compare And Swap`，在Go标准库 `sync/atomic` 中实现了这个方法：

```golang
// CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
```

这个函数会首先比较指针`addr`指向的值是否和`old`是否相等，如果相等则将`addr`指向的值替换为`new`，并返回`true`，
否则不做任何操作，并返回`false`。

## Mutex的第一次提交

在2008年 Russ Cox 提交了第一版的Mutex实现[[1]](https://github.com/golang/go/blob/bf3dd3f0efe5b45947a991e22660c62d4ce6b671/src/lib/sync/mutex.go)，
当时的实现比较简单，我们先从这一版开始了解Mutex的底层原理，由于当时Go还未发布1.0版本，也没有`sync/atomic`包，与现在的实现有很大的区别，但这并不影响我们理解其中的逻辑

### Mutex的定义

```golang
export type Mutex struct {
    key int32; // 记录当前锁是否被某个goroutine持有
    sema int32; // 信号量
}
```

初版的Mutex定义非常简单，使用`key`记录当前锁是否被持有，`sema`记录当前信号量

### 请求锁的实现

```golang
func (m *Mutex) Lock() {
    if xadd(&m.key, 1) == 1 { // 将标记加1，判断是否有其他goroutine持有锁
        // changed from 0 to 1; we hold lock
        return; // 当前 goroutine 持有锁，直接返回
    }
    sys.semacquire(&m.sema); // 挂起当前goroutine，等待调度
}
```

这个函数实现的很简单，在加锁时，首先将当前锁标记为已持有，如果当前锁没有被其他`goroutine`持有，则直接返回，
否则，挂起当前`goroutine`，等待信号量调度。

{{< admonition note "xadd函数" false >}}

```golang
func xadd(val *int32, delta int32) (new int32) {
    for { // 不断自旋操作
        v := *val;
        if cas(val, v, v+delta) { // 判断是否被其他goroutine修改
            return v+delta; // 返回新值
        }
    }
    panic("unreached")
}
```

`xadd`通过自旋CAS操作，将`val`的值加`delta`，就相当于现在Go语言的`atomic.AddInt32`
{{< /admonition >}}

### 释放锁的实现

释放锁的实现也很简洁:

```golang
func (m *Mutex) Unlock() {
    if xadd(&m.key, -1) == 0 { // 将标记减1
        // changed from 1 to 0; no contention
        return; // 如果没有其他goroutine持有锁，直接返回
    }
    sys.semrelease(&m.sema); // 通过信号量唤醒被挂起的goroutine
}
```

释放锁时，首先将当前标记减一，如果当前锁没有被其他`goroutine`持有，则直接返回，
否则，通过信号量通知运行时，调度被挂起的`goroutine`。

在这个版本，`Mutex`已经实现了基本的功能，但是这个版本有一个问题，所有`goroutine`会排队等待
运行时的调度，虽然这保证了公平性，所有的`goroutine`都会有机会参与调度，但是从性能的角度上看，
这会导致频繁的上下文切换，如果我们把锁直接交给新人(未挂起的`goroutine`)，这样就可以避免上下文的切换，
于是Go 团队再 Go1.0 正式版时对`Mutex`进行了较大的调整。

## 加入唤醒机制

在Go 1.0版本[[2]](https://github.com/golang/go/blob/release-branch.go1/src/pkg/sync/mutex.go)中，
`Mutex`的结构体字段进行了调整

```golang
type Mutex struct {
    state int32 // 复合数据，下文进行解释
    sema  uint32 // 信号量
}

const (
    mutexLocked = 1 << iota // state的第一位，代表当前锁是否被持有
    mutexWoken              // state的第二位，唤醒标记
    mutexWaiterShift = iota // 位移
)
```

在这一版中，将`Mutex`的第一个字段由`key`改为`state`，其含义发生了很大的变化，`state`的第一位表示当前锁是否被持有，
相当于之前的`key`，`state`的第二位是唤醒标记，代表当前是否有唤醒的`goroutine`正在请求锁，剩下的30位用来表示等待中
的Waiter数量。

### Go1.0 请求锁

在这一版中，代码相比第一版有很大变化，其主要优化是给新来的`goroutine`一些机会，让新`goroutine`能不参与休眠就获取锁

```golang
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) { 
        return // 当前锁未被持有
    }

    awoke := false 
    for {
        old := m.state
        new := old | mutexLocked // 新状态加上锁
        if old&mutexLocked != 0 {
            new = old + 1<<mutexWaiterShift // 如果已经有goroutine持有锁，则等待的 Waiter + 1
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            new &^= mutexWoken // 清空唤醒标记
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) { // 置新状态
            if old&mutexLocked == 0 {
                break // 旧状态锁已释放，获取当前锁
            }
            runtime_Semacquire(&m.sema) // 请求信号量
            awoke = true
        }
    }
}
```

我们可以用下面这个状态图表示请求锁的过程

{{< mermaid >}}
stateDiagram
    [*] --> Lock
    Lock --> [*]:Fast path
    Lock --> First
    Semacquire --> Awoken:wait
    First --> Semacquire:failed
    Awoken --> Semacquire: failed
    Awoken --> [*]:attempt
    First --> [*]:attempt
{{< /mermaid >}}

我们可以发现，在这版的`Mutex`中，给新来的`goroutine`会在加入等待队列前就去尝试获取锁，
如果失败则加入等待队列中

### Go1.0 释放锁

同时在Go 1.0中，释放锁的代码也变得更加复杂了

```golang
func (m *Mutex) Unlock() {
    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked) // 去掉持有锁标记
    if (new+mutexLocked)&mutexLocked == 0 {
        panic("sync: unlock of unlocked mutex") // 重复Unlock时panic
    }

    old := new
    for {
        // 如果没有其他的waiter
        // 或者已经有唤醒的goroutine
        if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
            return
        }
        // 挑选一个waiter唤醒
        new = (old - 1<<mutexWaiterShift) | mutexWoken // 打上唤醒标记
        if atomic.CompareAndSwapInt32(&m.state, old, new) { // 置新状态
            runtime_Semrelease(&m.sema)
            return
        }
        old = m.state
    }
}
```

1. 在释放锁时，首先会将锁标记为未锁状态，如果当前锁已经是未锁状态，则会`panic`(第5行)
2. 如果当前已经有唤醒的`goroutine`或者没有等待中的`waiter`，我们就什么都不用做，其他`goroutine`会自己抢夺这把锁
3. 如果没有唤醒的`goroutine`,就从等待队列中唤醒一个`goroutine`

### 给新 Goroutine 更多机会

在 Go1.5 版本中，Go团队又对`Mutex`进行了一次优化[[3]](https://github.com/golang/go/blob/edcad8639a902741dc49f77d000ed62b0cc6956f/src/sync/mutex.go)

在某些临界区，代码执行的速度很快，如果加入等待队列就会浪费很多时间，Go 团队针对这一情景对`Mutex`进行了优化

```golang
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if raceenabled { // race detector 相关
            raceAcquire(unsafe.Pointer(m))
        }
        return
    }

    awoke := false
    iter := 0 // 自旋次数
    for {
        old := m.state
        new := old | mutexLocked
        if old&mutexLocked != 0 { // 当前锁被持有
            if runtime_canSpin(iter) { // 检测是否可以自旋
                // 如果当前的没有其他唤醒的goroutine
                // 尝试将当前goroutine置为唤醒状态
                // 提醒 Unlock 不去唤醒其他goroutine
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true // 设置当前唤醒状态
                }
                runtime_doSpin()
                iter++
                continue // 自旋再次请求锁
            }
            new = old + 1<<mutexWaiterShift // 加入等待队列
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                panic("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 {
                break
            }
            runtime_Semacquire(&m.sema)
            awoke = true
            iter = 0 // 清空自旋计数
        }
    }

    if raceenabled { // race detector 相关
        raceAcquire(unsafe.Pointer(m))
    }
}
```

在某些临界区，代码执行速度很快，如果通过自旋几次就能获取到锁的所有权，就可以避免加入等待队列，
这样就可以减少上下文的切换，在某些情况下能很好的提高性能。

## 饥饿机制

进过几次优化后，`Mutex`的性能已经十分好了，但是由于自旋的存在，在特定情况下，有可能出现新人不断
地获取锁，而等待队列中的`goroutine`一直没有机会获取到锁，Go 团队针对这种情况加入了饥饿机制。

我们拿最新的 Go 1.15 中的源码进行分析

```golang
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken              // 唤醒标记
    mutexStarving           // 饥饿标记
    mutexWaiterShift = iota // 偏移量

    starvationThresholdNs = 1e6 // 1ms
)
```

在这个版本中， state的第三位被用作饥饿标记

```golang
func (m *Mutex) lockSlow() {
    var waitStartTime int64
    starving := false // 初始饥饿标记
    awoke := false
    iter := 0
    old := m.state
    for {
        // Don't spin in starvation mode, ownership is handed off to waiters
        // so we won't be able to acquire the mutex anyway.
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // Active spinning makes sense.
            // Try to set mutexWoken flag to inform Unlock
            // to not wake other blocked goroutines.
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        new := old
        // Don't try to acquire starving mutex, new arriving goroutines must queue.
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        // The current goroutine switches mutex to starvation mode.
        // But if the mutex is currently unlocked, don't do the switch.
        // Unlock expects that starving mutex has waiters, which will not
        // be true in this case.
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
            // If we were already waiting before, queue at the front of the queue.
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            if old&mutexStarving != 0 {
                // If this goroutine was woken and mutex is in starvation mode,
                // ownership was handed off to us but mutex is in somewhat
                // inconsistent state: mutexLocked is not set and we are still
                // accounted as waiter. Fix that.
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                if !starving || old>>mutexWaiterShift == 1 {
                    // Exit starvation mode.
                    // Critical to do it here and consider wait time.
                    // Starvation mode is so inefficient, that two goroutines
                    // can go lock-step infinitely once they switch mutex
                    // to starvation mode.
                    delta -= mutexStarving
                }
                atomic.AddInt32(&m.state, delta)
                break
            }
            awoke = true
            iter = 0
        } else {
            old = m.state
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

(咕了，过几天再写
