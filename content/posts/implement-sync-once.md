---
title: "从零实现 sync.Once: 探索并发控制的精髓"
date: 2025-01-22T19:30:00+08:00
draft: false
tags: ["Go", "并发", "sync", "CAS", "Mutex"]
categories: ["技术"]
---

## 前言

`sync.Once` 是 Go 标准库中一个非常简单但精妙的设计,它保证一个函数只执行一次。但你知道吗?要正确实现一个 `Once.Do`,有很多容易被忽视的细节。本文将带你从最简单的 CAS 方案开始,逐步演进到最终的正确实现,探讨每个方案的优缺点和那些"坑"。

## 场景

假设我们要实现一个 `Once` 类型,它的 `Do(f func())` 方法需要保证:
1. 无论调用多少次,`f()` 只执行一次
2. `f()` 执行完后,所有对 `Do()` 的调用都应该立即返回
3. **关键要求**: 当某个 `Do(f)` 调用返回时,必须保证 `f()` 已经完全执行完毕

## 方案一: 使用 CAS (原子操作)

最直观的想法是使用原子操作(CAS - Compare And Swap):

```go
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
- **无锁**: 不使用互斥锁,性能较好
- **简单**: 代码逻辑直观易懂

### 缺点

这个实现有一个**致命问题**: **不能保证 `f()` 执行完后才返回**。

考虑这个场景:
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
    once.Do(func() {}) // 这时 done 已经是 1 了,直接返回
    fmt.Println("第二个 goroutine: Do() 返回了,但 f() 可能还在执行!")
    wg.Done()
}()

wg.Wait()
```

**可能的输出**:
```
第二个 goroutine: Do() 返回了,但 f() 可能还在执行!
第一个 goroutine: Do() 返回了
f() 执行完成
```

可以看到,第二个 goroutine 的 `Do()` 虽然返回了,但 `f()` 还在执行!这违反了 `sync.Once` 的语义。

## 方案二: 使用 Mutex (初版)

既然 CAS 不行,那就用互斥锁:

```go
type Once struct {
    done uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.m.Lock()
        defer o.m.Unlock()
        if o.done == 0 {
            f()
            atomic.StoreUint32(&o.done, 1)
        }
    }
}
```

### 关键细节: 何时设置 done?

这里有一个**非常重要的细节**: **必须在 `f()` 执行完成后才能设置 `done` 标志**。

如果我们在 `f()` 之前设置 `done`:

```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.m.Lock()
        defer o.m.Unlock()
        if o.done == 0 {
            atomic.StoreUint32(&o.done, 1) // ❌ 错误!在 f() 之前设置
            f()
        }
    }
}
```

这样会导致什么问题?

**问题**: 其他 goroutine 看到已经 `done=1`,就会直接返回,但此时 `f()` 可能还在执行!

```go
var once Once
var wg sync.WaitGroup

wg.Add(2)
go func() {
    once.Do(func() {
        fmt.Println("f() 开始执行")
        time.Sleep(100 * time.Millisecond)
        fmt.Println("f() 执行完成")
    })
    wg.Done()
}()

go func() {
    time.Sleep(10 * time.Millisecond)
    once.Do(func() {})
    fmt.Println("第二个 goroutine 返回,但 f() 还在执行!")
    wg.Done()
}()

wg.Wait()
```

输出可能是:
```
f() 开始执行
第二个 goroutine 返回,但 f() 还在执行!
f() 执行完成
```

这违反了 `sync.Once.Do` 的语义: **当 `Do()` 返回时,必须保证 `f()` 已经完全执行完毕**。

## 方案三: 正确的 Mutex 实现

正确的实现是:**先执行 `f()`,然后再设置 `done`**:

```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.m.Lock()
        defer o.m.Unlock()
        if o.done == 0 {
            f()                              // ✅ 先执行 f()
            atomic.StoreUint32(&o.done, 1)   // ✅ 再设置 done
        }
    }
}
```

### 为什么这样是对的?

1. **第一个获取锁的 goroutine**:
   - 执行 `f()`
   - 设置 `done = 1`
   - 释放锁
   - 返回

2. **其他等待的 goroutine**:
   - 等待获取锁
   - 获取锁后,检查 `done == 1`,直接返回
   - **此时 `f()` 一定已经执行完了**,因为第一个 goroutine 只有在 `f()` 执行完后才会设置 `done`

3. **后续的 goroutine**:
   - 在第一个检查 `atomic.LoadUint32(&o.done) == 0` 时就会失败
   - 直接返回,无需获取锁
   - **此时 `f()` 一定已经执行完了**,因为 `done = 1`

## 方案四: 完整的正确实现

为了更完善,我们还需要考虑:
1. `f()` 发生 panic 的情况
2. 重用 `Once` 实例(官方 `sync.Once` 不支持)

```go
type Once struct {
    done uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    // 快速路径:如果已经执行过,直接返回
    if atomic.LoadUint32(&o.done) == 0 {
        // 慢速路径:需要加锁
        o.m.Lock()
        defer o.m.Unlock()
        // 双重检查:可能在等待锁时,其他 goroutine 已经执行了 f()
        if o.done == 0 {
            f()
            // 即使 f() panic,也要标记为已完成
            // sync.Once 的行为是:如果 f() panic,后续调用仍然不会执行 f()
            atomic.StoreUint32(&o.done, 1)
        }
    }
}
```

## 为什么需要双重检查?

看这个场景:

```
时刻 T1: goroutine A 读取 done==0,进入 if
时刻 T2: goroutine B 读取 done==0,进入 if
时刻 T3: goroutine A 获取锁
时刻 T4: goroutine A 执行 f()
时刻 T5: goroutine A 设置 done=1
时刻 T6: goroutine A 释放锁
时刻 T7: goroutine B 获取锁
时刻 T8: goroutine B 检查 done==1,直接返回(双重检查!)
```

如果没有双重检查,goroutine B 会再次执行 `f()`!

## 性能对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| CAS | 无锁,快速 | 不能保证 `f()` 执行完就返回 |
| Mutex (双重检查) | 正确,高性能 | 需要维护锁 |

**快速路径**: 已经 `done` 的情况下,只需要一次原子操作,无锁。
**慢速路径**: 第一次执行时需要获取锁,但只发生一次。

## 官方 sync.Once 的实现

Go 标准库的实现也是类似的思路,但做了一些优化:

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

关键点:
1. 快速路径内联,减少函数调用开销
2. `defer` 保证即使 `f()` panic,也会设置 `done`
3. 双重检查避免重复执行

## 总结

实现 `sync.Once.Do` 需要注意的关键点:

1. ✅ **不能只用 CAS**: 必须用锁,确保 `f()` 执行完后才返回
2. ✅ **双重检查**: 锁内再次检查 `done`,避免重复执行
3. ✅ **先执行 f(),再设置 done**: 这是保证语义的关键
4. ✅ **使用 defer 处理 panic**: 确保 `done` 总是被设置

这些细节看似简单,但每一个都是为了在并发环境下保证正确性。这就是并发编程的精髓所在!

## 参考

- [Go sync.Once 源码](https://github.com/golang/go/blob/master/src/sync/once.go)
- [The Go Memory Model](https://go.dev/ref/mem)
