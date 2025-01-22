# 知识点

## goroutine为什么比线程快

- 创建调度与销毁: goroutine的创建调度与销毁都在用户态，线程的在内核态
- 内存分配: goroutine栈内存为2-4KB，线程1MB
- 切换: goroutine切换时仅需要保护少数寄存器，而线程则需要保护大量的寄存器，goroutine无需操作系统线程切换

## GMP

- 两级线程模型，内核调度实体kse与线程的对应关系为m:n
- kse -> m -> p ->g
  - m: machine 关联一个内核线程
  - p: processor 作为上下文，衔接m和g
  - g: goroutine 轻量化线程，调用栈、调用信息， 例如channel等

https://github.com/fengyuan-liang/notes/blob/main/GoLang/golang%E5%A4%A7%E6%9D%80%E5%99%A8GMP%E6%A8%A1%E5%9E%8B.md

<https://www.jianshu.com/p/4afa0679851d>

## slice的扩容

golang 1.21.4

- 对于小slice进行1.25倍扩展 逐渐过度
- 对于大slice进行2倍扩展

src/runtime/slice.go

```golang
 newcap := oldCap
 doublecap := newcap + newcap
 if newLen > doublecap {
  newcap = newLen
 } else {
  const threshold = 256
  if oldCap < threshold {
   newcap = doublecap
  } else {
   // Check 0 < newcap to detect overflow
   // and prevent an infinite loop.
   for 0 < newcap && newcap < newLen {
    // Transition from growing 2x for small slices
    // to growing 1.25x for large slices. This formula
    // gives a smooth-ish transition between the two.
    newcap += (newcap + 3*threshold) / 4
   }
   // Set newcap to the requested cap when
   // the newcap calculation overflowed.
   if newcap <= 0 {
    newcap = newLen
   }
  }
 }
```

## g0

持有调度栈的goroutine，作用有

- goroutine的创建
- 大内存分配
- CGO函数的执行

## 三色标记法

https://community.apinto.com/d/34057-golang-gc

