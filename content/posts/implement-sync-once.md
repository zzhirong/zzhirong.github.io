---
title: "从零实现 sync.Once: 探索并发控制的精髓"
date: 2026-01-22T19:30:00+08:00
draft: false
tags: ["Go", "并发", "sync", "CAS", "Mutex"]
categories: ["技术"]
---

## 前言

写这篇文章的起因是看到了一篇帖子：[下面代码中 f() 会被重复执行吗？](https://v2ex.com/t/1183833)。`sync.Once` 是 Go 标准库中一个非常简单但精妙的设计，它保证一个函数只执行一次。我在仔细看过这篇帖子和官方实现后，才发现但要正确实现一个 `Once.Do`，有很多容易被忽视的细节。本文将从最简单的 CAS 方案开始，逐步演进到最终的正确实现，探讨其中的一些关键细节。

## 场景

假设我们要实现一个 `Once` 类型，它的 `Do(f func())` 方法需要保证：

1. 无论调用多少次，`f()` 只执行一次
2. `f()` 执行完后，所有对 `Do()` 的调用都应该立即返回
3. **关键要求**：当某个 `Do(f)` 调用返回时，必须保证 `f()` 已经完全执行完毕

## 方案一：使用 CAS（原子操作）

最直观的想法是使用原子操作（CAS - Compare And Swap）：

```go
// 注意: 以下是 Once.Do 的错误实现
type Once struct {
    done uint32
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
            f()
        }
    }
}
```

### 优点

- **无锁**：不使用互斥锁，性能较好
- **简单**：代码逻辑直观易懂

### 缺点

这个实现有一个**致命问题**：**不能保证其他调用 `Do(f)` 的 goroutine 等到 `f()` 执行完后才返回**。

考虑这个场景：
```go
var once Once
var wg sync.WaitGroup

wg.Add(2)
go func() {
    once.Do(func() {
        time.Sleep(100 * time.Millisecond) // 模拟耗时操作
        fmt.Println("f() 执行完成")
    })
    fmt.Println("第一个 goroutine: Do() 返回了")
    wg.Done()
}()

go func() {
    time.Sleep(10 * time.Millisecond)
    once.Do(func() {}) // 这时 done 已经是 1 了，直接返回
    fmt.Println("第二个 goroutine: Do() 返回了，但 f() 可能还在执行！")
    wg.Done()
}()

wg.Wait()
```

**可能的输出**：

```
第二个 goroutine: Do() 返回了，但 f() 可能还在执行！
f() 执行完成
第一个 goroutine: Do() 返回了
```

可以看到，第二个 goroutine 的 `Do()` 虽然返回了，但 `f()` 还在执行！这违反了 `sync.Once` 的语义。

## 方案二：使用 Mutex（初版）

因为 CAS 方案无法阻塞其他 goroutine，那就用互斥锁：

```go
// 注意：Once.Do() 的错误实现
type Once struct {
    done uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.m.Lock()
        defer o.m.Unlock()
        if o.done == 0 {
            atomic.StoreUint32(&o.done, 1) // 错误！在 f() 之前设置
            f()
        }
    }
}
```

以上实现还是错的，因为在 `f()` 执行完之前就设置了 `done`，会导致后续调用 `Do(f)` 的 goroutine 直接返回。

## 方案三：完整的正确实现

为了更完善，我们还需要考虑 `f()` 发生 panic 的情况：

```go
type Once struct {
    done uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    // 快速路径：如果已经执行过，直接返回
    if atomic.LoadUint32(&o.done) == 0 {
        // 慢速路径：需要加锁
        o.m.Lock()
        defer o.m.Unlock()
        // 双重检查：可能在等待锁时，其他 goroutine 已经执行了 f()
        if o.done == 0 {
            defer atomic.StoreUint32(&o.done, 1)
            // 即使 f() panic，也要标记为已完成
            // sync.Once 的行为是：如果 f() panic，后续调用仍然不会执行 f()
            f()
        }
    }
}
```

## 官方 sync.Once 的实现

Go 标准库的实现也是类似的思路，但做了一些优化：

```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

关键点：

1. 快速路径内联，减少函数调用开销
2. `defer` 保证即使 `f()` panic，也会设置 `done`
3. 双重检查避免重复执行

## 总结

实现 `Once.Do` 除了「只执行一次」的语义之外，还有其他隐藏的两点：

1. 当 `f()` 正在执行的时候，所有调用 `Do(f)` 的 goroutine 必须等待 `f()` 执行完毕才能返回，所以没办法单靠原子操作实现。
2. 任一 goroutine 调用 `Do(f)` 返回后，`f()` 必须已经执行完毕。

## 参考

- [Go sync.Once 源码](https://github.com/golang/go/blob/master/src/sync/once.go)
