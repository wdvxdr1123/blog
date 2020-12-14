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

### 加锁的实现

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

### 解锁的实现

解锁的实现也很简洁:

```golang
func (m *Mutex) Unlock() {
    if xadd(&m.key, -1) == 0 { // 将标记减1
        // changed from 1 to 0; no contention
        return; // 如果没有其他goroutine持有锁，直接返回
    }
    sys.semrelease(&m.sema); // 通过信号量唤醒被挂起的goroutine
}
```

解锁时，首先将当前标记减一，如果当前锁没有被其他`goroutine`持有，则直接返回，
否则，通过信号量通知运行时，调度被挂起的`goroutine`。

在这个版本，`Mutex`已经实现了基本的功能，但是这个版本有一个问题，所有`goroutine`会排队等待
运行时的调度，虽然这保证了公平性，所有的`goroutine`都会有机会参与调度，但是从性能的角度上看，
这会导致频繁的上下文切换，如果我们把锁直接交给新人(未挂起的`goroutine`)，这样就可以避免上下文的切换，
于是Go 团队再 Go1.0 正式版时对`Mutex`进行了较大的调整。

## Go 1.0中Mutex的实现

咕咕咕...
