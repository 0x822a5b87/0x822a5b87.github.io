---
title: uCore：一个基于Rust的简单操作系统实现（3）
date: 2025-11-18 18:12:28
tags:
    - rust
    - os
---

# 多道程序与分时多任务



## QA

### `__switch` 函数的实现

`__switch` 函数的实现有几点需要注意：

1. 在我们保存`TaskContext`的过程中，我们最好的保存方式是，与 TaskContext 的字段定义顺序一致：`ra -> sp -> s0~s11`；
2. 在恢复TaskContext的过程中，我们最好把 `sp` 放在最后恢复。虽然我们在 `__switch` 中，不会用到 sp，但是这种可能影响全局的操作放在最后做是一个比较好的防御式编程习惯。

```assembly
# Store `sn` into the corresponding register, then add 2 to skip `sp` and `ra`.
.altmacro
.macro SAVE_SN n
    sd s\n, (\n+2)*8(a0)
.endm
.macro LOAD_SN n
    ld s\n, (\n+2)*8(a1)
.endm
    .section .text
    .globl __switch
__switch:
    # __switch(
    #     current_task_cx_ptr: *mut TaskContext,
    #     next_task_cx_ptr: *const TaskContext
    # )
    #
    #   TaskContext
    #       ra: usize,
    #       sp: usize,
    #       s: [usize; 12],
    #

    sd ra, 0(a0)
    # save kernel stack of current task
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr

    # restore ra & s0~s11 of next execution
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    ld sp, 8(a1)
    ret


```

### 为什么 `__switch` 只需要保存 `sp`, `ra`, `s0`~`s11` 寄存器？

这里我之前有个疑问，花了不少时间捋清楚，这里分享一下（不保真），有错误请各位大佬们不吝指教。

>**为什么，在 `trap.S` 中需要保存所有寄存器信息，而 `switch.S` 中则只需要保存 callee-saved registers？**

先说结论：这是因为 `trap.S` 的调用机制和 `switch.S` 的调用机制完全不一样：

- `__alltraps` 和 `__restore` 是在发生trap时，由CPU触发的跳转，是完全不可控的跳转逻辑：
  - 可能发生在任意指令之间（比如函数调用 `sd a0 0(sp)` 时，sp 指针指向的内存块在MMU不可用发生了缺页异常）；
  - 可能发生在函数执行的任意阶段（比如函数还没来得及保存 `a0~a7`，或者正在使用 `a0` 进行计算）。
- `__switch` 是内核主动调用的调度函数，调用时机是**完全可控**的 —— 内核只会在 “安全的时机” 调用 `__switch`，这个 “安全时机” 的核心要求是：调用 `__switch` 之前，当前任务的所有 caller-saved 寄存器（`a0~a7`、`t0~t6`）都已被当前执行的函数保存到栈上。

举个例子，在如下的代码中：

```assembly
foo:
	addi sp, sp, -16
	# 下面的代码可能触发缺页异常进入 __alltraps 和 __restore
	# 然而，不可能出现一个 __switch 调用；
	# call __swtich
	sd   a0, 0(sp)
	sd   a1, 8(sp)
```

#### 具体触发逻辑

在一切开始之前，我们必须搞清楚两个基本概念，`caller-saved registers` 和 `callee-saved registers`：

- caller-saved 由调用者保存，也就是 a0~a7 等，函数在调用时会将这些数据保存到自己的函数栈，后续访问都通过函数栈上保存的值来访问；
- callee-saved 由被调用者保存，也就是 ra, fp, s0~s11 等，被调用的函数需要保存原始值，并且在函数退出之后将这些寄存器恢复到原值。


此外，在操作系统中，定义了两个不同的函数来保存上下文：

1. `__alltraps` 和 `__restore` 用于在trap发生时用来处理中断，此时会保存/恢复所有的寄存器作为上下文；
2. `__switch` 用于在需要发生上下文切换时来处理中断，此时只会保存 callee-saved；

我们举个例子，假设存在两个线程：

>线程一

```asm
foo:
	addi sp, sp, -16
	sd   a0, 0(sp)
```

>线程二

```asm
bar:
	li a0, 0x0
```

对于trap，可能会出现如下的执行顺序：

1. addi sp, sp -16
2. `__alltraps`
3. li a0, 0x0
4. `__restore`
5. sd a0, 0(sp)

在这个例子中不会有任何问题，因为`__alltraps`和`__restore`会保存/恢复所有的寄存器。

而 `__switch` 是一个主动调用的函数，并不会触发如下的执行顺序（除非程序或者编译器出现BUG）：

1. addi sp, sp -16
2. `__switch`
3. li a0, 0x0
4. `__switch`
5. sd a0, 0(sp)

而在实际的应用中 `__switch` 的调用往往是如下场景：


>线程一

```asm
foo:
	addi sp, sp, -16
	sd   a0, 0(sp)
	# all caller-saved registers are stored.

	call __switch
```

>线程二

```asm
bar:
	li a0, 0x0
	call __switch
```

在这种情况下，所有的 caller-saved registers 已经使用完毕，所以 `__switch` 只需要处理 callee-saved registers 即可。

总结来说：`__switch` 无需保存 caller-saved 寄存器的原因的是：

1. 这些寄存器的值已被当前函数保存到栈上（栈指针 `sp` 会被 `__switch` 保存，栈数据不会丢失）；
2. 切换到新任务后，新任务的 caller-saved 寄存器会由它自己的函数在执行时保存 / 恢复，与当前任务无关；
3. 当当前任务再次被调度时，`__switch` 恢复 `sp` 后，当前函数会从栈上重新加载 caller-saved 寄存器的原始值，继续执行。

简单说：`__switch` 依赖 “调用前已保存 caller-saved 寄存器” 的约定，只需保存 callee-saved 寄存器（`ra`、`sp`、`s0~s11`）—— 这些寄存器是任务长期状态的载体，且不会被函数主动保存，必须由 `__switch` 负责保存 / 恢复。

#### 补充

内核调度 `__switch` 的场景，本质都是 “任务主动放弃 CPU” 或 “内核在安全点触发调度”：

1. 任务主动调用系统调用（如 `sleep`）：系统调用处理函数会先保存 caller-saved 寄存器，再调用 `__switch`；
2. 时钟中断触发调度：时钟中断的 `__alltraps` 会保存所有寄存器，进入内核后，内核会在 “调度前” 确保当前任务的 caller-saved 已妥善保存（或直接使用 `__alltraps` 保存的完整上下文），再调用 `__switch`；
3. 任务执行完毕：内核会在任务退出函数中保存 caller-saved 寄存器，再调用 `__switch` 切换到其他任务。

#### 一些典型的 `__switch` 调用场景

在下面执行任务的场景中，`__switch` 的调用是保证在所有的 `caller-saved registers` 都被保存到栈上之后才会被调用的。

```rust
pub fn run_tasks() {
    loop {
        let mut processor = PROCESSOR.exclusive_access();
        if let Some(task) = fetch_task() {
            let idle_task_cx_ptr = processor.get_idle_task_cx_ptr();
            // access coming task TCB exclusively
            let mut task_inner = task.inner_exclusive_access();
            let next_task_cx_ptr = &task_inner.task_cx as *const TaskContext;
            task_inner.task_status = TaskStatus::Running;
            // release coming task_inner manually
            drop(task_inner);
            // release coming task TCB manually
            processor.current = Some(task);
            // release processor manually
            drop(processor);
            unsafe {
                __switch(idle_task_cx_ptr, next_task_cx_ptr);
            }
        } else {
            warn!("no tasks available in run_tasks");
        }
    }
}
```

#### 总结

| 对比维度       | `__alltraps` + `__restore`                  | `__switch`                            |
| -------------- | ------------------------------------------- | ------------------------------------- |
| 触发方式       | 被动（硬件 Trap：中断 / 异常 / 系统调用）   | 主动（内核函数调用）                  |
| 执行时机       | 任意时刻（不可控）                          | 安全点（可控：caller-saved 已保存）   |
| 保存寄存器范围 | 所有通用寄存器 + 状态寄存器（如 `mstatus`） | 仅 callee-saved：`ra`、`sp`、`s0~s11` |
| 核心目标       | 原封不动恢复现场（Trap 后继续执行）         | 切换任务上下文（保证任务能恢复执行）  |
| 依赖约定       | 无（需兜底所有情况）                        | 调用前 caller-saved 已保存到栈        |