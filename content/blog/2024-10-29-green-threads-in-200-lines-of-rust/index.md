+++
title = "200 行 Rust 代码实现绿色线程 / 有栈协程"
date = 2024-10-29
+++

最初是在清华大学 rCore 内核教程中看到这段[示例代码](https://github.com/rcore-os/rCore-Tutorial-v3/blob/main/user/src/bin/stackful_coroutine.rs)，觉得很有趣，但为了不跟 rCore 内核绑定，我将其移植到 Linux for RISC-V64 上，并且重构了很多代码，修复了一些问题，增强了其可读性，具体见[https://github.com/systemxlabs/green-threads-in-200-lines-of-rust](https://github.com/systemxlabs/green-threads-in-200-lines-of-rust)。

## 前置知识：RISC-V 架构和函数调用规约
RISC-V 是一个模块化的精简指令集架构，这里我们以 RV64I 模块（64 位基本整数指令集）为例
1. 包含基本的整数运算指令（算术/逻辑/移位）、分支和跳转指令、内存交互指令
2. 提供 32 个整数寄存器（x0-x31），其中 x0 硬连线为 0（恒为 0），这些寄存器有一些别名，如 x1 用于存放函数返回地址，也称 ra 寄存器，x2 用于存放栈指针，也称 sp 寄存器
3. 指令长度为 32 位，寄存器位宽是 64

函数调用是通过压栈和出栈的方式，每个函数执行都有其对应的栈帧
<img src="call-stack.png" style="width:800px" />

一个函数可能的栈帧内容
<img src="stack-frame.png" style="width:800px" />
它的开头和结尾分别在 sp 和 fp（栈帧指针）寄存器所指向的地址。按照地址从高到低分别有以下内容，它们都是通过 sp 加上一个偏移量来访问的：
- ra 寄存器保存其返回之后的跳转地址，是一个被调用者保存寄存器
- 父亲栈帧的结束地址 fp ，是一个被调用者保存寄存器
- 其他被调用者保存寄存器 s1 ~ s11 
- 函数所使用到的局部变量

何为调用者保存寄存器和被调用者保存寄存器？
- 被调用者保存(Callee-Saved) 寄存器 ：被调用的函数可能会覆盖这些寄存器，需要被调用的函数来保存的寄存器，即由被调用的函数来保证在调用前后，这些寄存器保持不变
- 调用者保存(Caller-Saved) 寄存器 ：被调用的函数可能会覆盖这些寄存器，需要发起调用的函数来保存的寄存器，即由发起调用的函数来保证在调用前后，这些寄存器保持不变

函数 A 调用函数 B 具体过程：
1. 函数 A 保存“调用者保存寄存器”到其栈帧内
2. 函数 A 设置输入参数到 a0-a7 寄存器
3. 函数 A 跳转到函数 B 入口并设置 ra 寄存器为其下一条指令地址（通常是 pc = pc + 4）
4. 函数 B 保存“被调用者保存寄存器”到其栈帧内
5. 函数 B 读取输入参数并执行
6. 函数 B 从其栈帧内恢复“被调用者保存寄存器”
7. 函数 B 跳转到 ra 寄存器指向的地址
8. 函数 A 从其栈帧内恢复“调用者保存寄存器”
9. 函数 A 继续向下执行

更详细介绍可见[此文章](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter1/5support-func-call.html)。

## 定义一个绿色线程
```rust
struct Thread {
    id: usize,
    stack: Vec<u8>,
    ctx: ThreadContext,
    state: State,
}

enum State {
    Available,
    Running,
    Ready,
}

#[repr(C)]
pub struct ThreadContext {
    ra: u64,
    sp: u64,
    s0: u64,
    s1: u64,
    s2: u64,
    s3: u64,
    s4: u64,
    s5: u64,
    s6: u64,
    s7: u64,
    s8: u64,
    s9: u64,
    s10: u64,
    s11: u64,
    entry: u64,
}
```
一个绿色线程包含：
1. 绿色线程 id
2. 执行状态：Availabe 表示空闲，可分配新任务执行；Running 表示正在执行任务；Ready 表示已分配任务，可以被调度执行，任务可能未开始也可能被中途挂起了
3. 函数上下文：被调用者保存寄存器，用于恢复挂起的任务继续执行；entry 是任务的入口地址，只在首次被调度时使用
4. 栈：分配在进程对应的堆区（固定大小、不会扩容）

## 定义绿色线程运行时
```rust
static mut RUNTIME: usize = 0;

pub struct Runtime {
    threads: Vec<Thread>,
    current: usize,
}

impl Runtime {
    pub fn new() -> Self {
        let base_thread_id = 0;
        let base_thread = Thread::new_with_state(base_thread_id, State::Running);

        let mut threads = vec![base_thread];
        let mut available_threads = (1..MAX_THREADS + 1).map(|i| Thread::new(i)).collect();
        threads.append(&mut available_threads);

        Runtime {
            threads,
            current: base_thread_id,
        }
    }

    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }
}
```
运行时在启动时会预创建若干绿色线程，current 代表正在执行的绿色线程
1. 一个 base thread，用于 Runtime 本身的执行流，此时处于 Runtime 执行流，因此状态为 Running
2. 若干 user threads，用于用户任务的执行流，此时没有绑定任务，因此状态均为 Available

## 创建用户任务
```rust
impl Runtime {
    pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available green thread.");

        println!("RUNTIME: spawning task on green thread {}", available.id);
        let size = available.stack.len();
        unsafe {
            let s_ptr = available.stack.as_mut_ptr().offset(size as isize);

            // make sure our stack itself is 8 byte aligned
            let s_ptr = (s_ptr as usize & !7) as *mut u8;

            available.ctx.ra = task_return as u64; // task return address
            available.ctx.sp = s_ptr as u64; // stack pointer
            available.ctx.entry = f as u64; // task entry
        }
        available.state = State::Ready;
    }

    fn t_return(&mut self) {
        // Mark current thread available, so it can be assigned a new task
        self.threads[self.current].state = State::Available;
        self.t_schedule();
    }
}

fn task_return() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_return();
    }
}
```
用户任务是一个无参数的函数。

创建过程：
1. 首先找到一个空闲的绿色线程，然后初始化其栈顶指针为栈空间 `Vec<u8>` 的最高地址，并确保 8 字节对齐（RISC-V64 要求 ld/sd 访存指令的数据地址是 8 字节对齐）
2. 设定任务入口地址为函数 `f`
3. 设定其返回地址为 `task_return` 函数入口，`task_return` 会调用 `Runtime::t_return()` 来标记当前线程空闲，并调度/切换到下一个 ready 的线程继续执行

## 执行/挂起用户任务
```rust
impl Runtime {
    pub fn run(&mut self) {
        while self.t_yield() {}
        println!("All tasks finished!");
    }

    fn t_yield(&mut self) -> bool {
        self.threads[self.current].state = State::Ready;
        self.t_schedule()
    }

    fn t_schedule(&mut self) -> bool {
        let thread_count = self.threads.len();

        let mut pos = (self.current + 1) % thread_count;
        while self.threads[pos].state != State::Ready {
            pos = (pos + 1) % thread_count;
            if pos == self.current {
                return false;
            }
        }
        println!("RUNTIME: schedule next thread {} to be run", pos);

        self.threads[pos].state = State::Running;
        let old_pos = self.current;
        self.current = pos;
        unsafe {
            switch(&mut self.threads[old_pos].ctx, &self.threads[pos].ctx);
        }

        true
    }
}

pub fn r#yield() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_yield();
    };
}
```

## 切换绿色线程
```rust
#[naked]
#[no_mangle]
unsafe extern "C" fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    naked_asm!(
        "
        sd ra, 0*8(a0)
        sd sp, 1*8(a0)
        sd s0, 2*8(a0)
        sd s1, 3*8(a0)
        sd s2, 4*8(a0)
        sd s3, 5*8(a0)
        sd s4, 6*8(a0)
        sd s5, 7*8(a0)
        sd s6, 8*8(a0)
        sd s7, 9*8(a0)
        sd s8, 10*8(a0)
        sd s9, 11*8(a0)
        sd s10, 12*8(a0)
        sd s11, 13*8(a0)
        sd ra, 14*8(a0)

        ld ra, 0*8(a1)
        ld sp, 1*8(a1)
        ld s0, 2*8(a1)
        ld s1, 3*8(a1)
        ld s2, 4*8(a1)
        ld s3, 5*8(a1)
        ld s4, 6*8(a1)
        ld s5, 7*8(a1)
        ld s6, 8*8(a1)
        ld s7, 9*8(a1)
        ld s8, 10*8(a1)
        ld s9, 11*8(a1)
        ld s10, 12*8(a1)
        ld s11, 13*8(a1)
        ld t0, 14*8(a1)

        jr t0
    "
    );
}
```

思考：
1. 如何动态分配栈？扩容和缩容
2. 协作式调度如何避免糟糕的程序持续占用 CPU？
