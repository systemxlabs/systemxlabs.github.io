+++
title = "Rust 异步"
date = 2024-07-19
draft = true
+++

TODO
1. RISC-V 中断向量
2. io总线和驱动
2. epoll 原理
3. 异步运行时的wake逻辑，如何设置waker，waker如何wake具体task？
4. 如果所有 task 均返回pending，那异步运行时会停止poll，还是继续 poll，如何poll？
5. 为什么说 future 是惰性的，必须要多次poll吗，一次不够吗？
6. task 和 future 关系
7. 异步运行时是如何切分task的？
8. 有栈协程相比于无栈协程为啥没有传染性？

参考：
1. https://rust-lang.github.io/async-book/
2. https://www.zhihu.com/question/19732473
3. https://www.eventhelix.com/rust/
