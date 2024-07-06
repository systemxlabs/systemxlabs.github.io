+++
title = "Rust 无畏并发的基础 Send 和 Sync"
date = 2024-07-06
+++

## 线程不安全的原因
1. 数据竞争，是指多个线程在同一时刻对同一个数据进行读写或者写写。
2. 数据的某些操作跟特定线程绑定在一起。比如 [pthread_mutex_lock](https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_mutex_lock.html) 的 unlock 操作必须在其 lock 线程执行，否则会 [undefined behavior](https://en.wikipedia.org/wiki/Undefined_behavior)，比如窗口管理 winit 中的 EventLoop 必须在主线程创建和执行。

## Send 和 Sync 定义
**Send：Types that can be transferred across thread boundaries**

类型可以在线程间安全地被复制或着移动。
- 几乎所有 primitives 类型都是 Send，比如 bool、i32、array等，而裸指针 [pointer](https://doc.rust-lang.org/std/primitive.pointer.html) 类型并不是
- `unsafe impl<T: Sync + ?Sized> Send for &T {}` 表示如果 T 是 Sync，其不可变引用 &T 可以复制到其他线程被安全地使用

**Sync：Types for which it is safe to share references between threads**

类型可以在线程间安全地共享引用。
- 这里共享的引用指 &T，因为 &mut T 天然地被 Rust 借用规则限制，不可能两个线程同时拥有 &mut T
- 由于 Rust 的[内部可变性模式](https://doc.rust-lang.org/reference/interior-mutability.html)，&T 并不能保证只读数据，因此 Sync 解决的问题就是 &T 跨线程的安全性
- 不具有内部可变性的类型，其引用 &T 只读数据，因此 &T 可以跨线程分享，满足 Sync
  - 几乎所有 primitives 类型都是 Sync，比如 bool、i32、array等，而裸指针 [pointer](https://doc.rust-lang.org/std/primitive.pointer.html) 类型并不是
  - 基于以上 primitives 的 struct、tuple 和 enum 也都会自动实现 Sync
- 内部可变性原语 UnsafeCell 不是 Sync
  - 在其上没有通过同步原语构建的类型（比如 Rc 和 RefCell）也不是 Sync
  - 在其上通过同步原语构建的类型（比如 AtomicBool、Arc、Mutex 等）则可以是 Sync

Send 和 Sync 都是 auto marker trait，编译器会自动给每个符合的类型自动实现该 trait，也可以手动通过 `impl !Send for XXX` 来显示地标识某类型不是 Send。

## 推导结论
**T: Sync <=> &T: Send**
- 如果 T 是 Sync，说明可以在线程间安全的共享引用，所以 &T 可以被安全的 Send。相反，如果 T 不是 Sync，那么其引用 &T 也不能被安全地 Send。
- 如果 &T 是 Send，说明 T 的引用可以在线程间安全是发送，所以 T 是 Sync。相反，如果 &T 不能安全的 Send，那么 T 就不是 Sync。

**T: Sync <=> &T: Sync**
- T 是 Sync，所以 &T 是 Send。&T 是 Sync 的前提是 &&T 为 Send，而 &&T 实际就是 &T（共享是可传递的），所以 &&T 也是 Send。

**T: Sync <=> &mut T: Sync**
- T 是 Sync，所以 &T 是 Send。&mut T 是 Sync 的前提是 &&mut T 为 Send，而 &&mut T 实际就是 &T，所以 &&mut T 也是 Send。

**T: Send <=> &mut T: Send**
- 因为 &mut T 可以通过 [std::mem::replace](https://doc.rust-lang.org/std/mem/fn.replace.html) 将其指向的指 move 走，所以 &mut T 是 Send 的前提是 T 必须可以在线程间安全的 move。

以上结论的两边都是等价的，可以互相推导。

## 实际例子
**Send + Sync**
- 绝大多数基本类型

**Send + !Sync**
- RefCell

**!Send + Sync**
- MutexGuard

**!Send + !Sync**
- Rc

参考：
1. https://users.rust-lang.org/t/example-of-a-type-that-is-not-send/59835/3
2. https://doc.rust-lang.org/nomicon/send-and-sync.html