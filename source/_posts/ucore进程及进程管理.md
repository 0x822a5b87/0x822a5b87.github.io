---
title: "uCore进程及进程管理"
date: 2025-12-02 11:09:36
tags:
  - os
  - rust
---

# uCore：进程及进程管理

>我们将开发一个用户 **终端** (Terminal) 或 **命令行** (Command Line Application, 俗称 **Shell** ) ， 形成用户与操作系统进行交互的命令行界面 (Command Line Interface)。
>
>为此，我们要对任务建立新的抽象： **进程** ，并实现若干基于 **进程** 的强大系统调用。

```mermaid
flowchart LR
    subgraph init["初始化阶段：无用户任务"]
        direction LR
        registers_0("registers.cx: idle运行状态"):::pink
        current_0("current: None"):::purple
        idle_task_cx_0("idle_task_cx: zero.ctx（占位）"):::green
    end

    subgraph stage_1["阶段1：fetch_task"]
        direction LR
        registers_1("registers.cx: task.cx（用户任务状态）"):::pink
        current_1("current: Some(task)"):::purple
        idle_task_cx_1("idle_task_cx: 原registers.cx（idle休眠状态）"):::green
    end

    subgraph stage_2["阶段2：suspend"]
        direction LR
        registers_2("registers.cx: idle运行状态（从idle_task_cx_1加载）"):::pink
        current_2("current: None"):::purple
        idle_task_cx_2("idle_task_cx: 原registers_1（用户任务挂起状态）"):::green
    end

    subgraph stage_3["阶段3：exit→用户任务退出"]
        direction LR
        registers_3("registers.cx: idle运行状态（从idle_task_cx_1加载）"):::pink
        current_3("current: None"):::purple
        idle_task_cx_3("idle_task_cx: 原idle_task_cx_1（无变化）"):::green
    end

    subgraph TaskManager["TaskManager：就绪队列"]
        t_task("task.cx: 用户任务挂起状态（仅suspend后存在）"):::coral
    end

    %% 流转逻辑（补充关键动作标注）
    init -->|init| stage_1
    stage_1 -->|suspend| stage_2
    stage_1 -->|exit| stage_3

    %% 状态转移细节
    current_1 -->|take_current后add_task，共享task所有权| TaskManager
    registers_1 -->|__switch保存用户任务状态到task.cx| t_task
    idle_task_cx_1 -->|__switch加载idle状态到寄存器| registers_2
    idle_task_cx_1 -->|__switch加载idle状态到寄存器| registers_3

    %% 样式定义（沿用你的设计）
    classDef pink fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
    classDef green fill: #696,color: #fff,font-weight: bold;
    classDef purple fill:#969,stroke:#333, font-weight: bold;
    classDef error fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5;
    classDef coral fill:#f9f,stroke:#333,stroke-width:4px;
    classDef animate stroke-dasharray: 9,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;
```





## 与进程有关的重要系统调用

### 重要系统调用

#### fork 系统调用

```rust
/// 功能：由当前进程 fork 出一个子进程。

/// 返回值：
/// - 对于子进程返回 0，对于当前进程则返回子进程的 PID 。

/// syscall ID：220
pub fn sys_fork() -> isize;
```

#### exec 系统调用

> 关于 `exec` 指令的详细说明，请参考 [exec](#exec)

```rust
/// 功能：将当前进程的地址空间清空并加载一个特定的可执行文件，返回用户态后开始它的执行。

/// 参数：
/// - 字符串 path 给出了要加载的可执行文件的名字；

/// 返回值：
/// - 如果出错的话（如找不到名字相符的可执行文件）则返回 -1，否则不应该返回。
/// - 注意：path 必须以 "\0" 结尾，否则内核将无法确定其长度

/// syscall ID：221
pub fn sys_exec(path: &str) -> isize;
```

利用 `fork` 和 `exec` 的组合，我们能让创建一个子进程，并令其执行特定的可执行文件。

#### waitpid 系统调用

```rust
/// 功能：当前进程等待一个子进程变为僵尸进程，回收其全部资源并收集其返回值。

/// 参数：
/// - pid 表示要等待的子进程的进程 ID，如果为 -1 的话表示等待任意一个子进程；
/// - exit_code 表示保存子进程返回值的地址，如果这个地址为 0 的话表示不必保存。

/// 返回值：
/// - 如果要等待的子进程不存在则返回 -1；否则如果要等待的子进程均未结束则返回 -2；
/// - 否则返回结束的子进程的进程 ID。

/// syscall ID：260
pub fn sys_waitpid(pid: isize, exit_code: *mut i32) -> isize;
```

`sys_waitpid` 在用户库中被封装成两个不同的 API， `wait(exit_code: &mut i32)` 和 `waitpid(pid: usize, exit_code: &mut i32)`， 前者用于等待任意一个子进程，后者用于等待特定子进程。它们实现的策略是如果子进程还未结束，就以 yield 让出时间片：

```rust
 // user/src/lib.rs
 
pub fn wait(exit_code: &mut i32) -> isize {
    loop {
        match sys_waitpid(-1, exit_code as *mut _) {
            -2 => { sys_yield(); }
            n => { return n; }
        }
    }
}
```

### 应用程序示例

`initproc` 和 `user_shell` 两个程序，他们分别是内核的初始进程以及初始进程启动的一个 `shell` 进程：

1. `initproc` 是 `pid == 0` 的进程，它负责处理内核中的 `zombie` 和 `orphan` 进程；
2. `user_shell` 从命令行读取命令，并启动一个子进程执行该命令。

#### 用户初始程序-initproc

在内核初始化完毕后创建的第一个进程，是 **用户初始进程** (Initial Process) ，它将通过 `fork+exec` 创建 `user_shell` 子进程，并将被用于回收僵尸进程。

下面的程序有几个指的注意的点：

1. `pid == -1` 表示 **要等待的子进程不存在**（符合 rCore 定义）：比如 `fork()` 后父进程先执行，子进程还未完成创建（此时父进程无有效子进程），或后续无任何子进程时，`wait` 会返回 `-1`；
2. `wait()` 的阻塞性由 OS 实现决定：代码中父进程在 `pid == -1` 时主动调用 `yield_()`，说明 rCore 的 `sys_waitpid` 是 **非阻塞式** 的（无符合条件的僵尸进程时立即返回 `-1` 或 `-2`，不挂起父进程），这与 Linux 默认阻塞的 `waitpid` 形成差异；
3. 代码中，显示的处理了 `pid == -1` 以及隐式的处理了 `pid > 0` 的所有情况，但是未处理 `pid == -2` 的情况：
   1. 对于 `pid == -1` 的子进程，此时子进程尚未创建成功，无需任何处理，我们直接continue继续循环；
   2. 对于 `pid > 0`（子进程终止，成功回收僵尸进程）：会执行 `println!` 打印回收日志，之后继续循环；
   3. 未显式处理 `pid == -2`（子进程存在但均未结束）：会跳过 `pid == -1` 分支，直接重新进入循环调用 `wait`（因无 `yield_()`，会高频空转）；

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

use user_lib::{exec, fork, wait, yield_};

#[no_mangle]
fn main() -> i32 {
    /// fork() 创建一个进程
    if fork() == 0 {
        /// 返回值为0，说明是子进程，我们在进程里启动shell程序
        exec("ch5b_user_shell\0", &[0 as *const u8]);
    } else {
        loop {
            /// 0 指定不保存子进程返回值的地址
            let mut exit_code: i32 = 0;
            let pid = wait(&mut exit_code);
            /// pid == -1 表示返回的进程不存在，因为在fork()之后，父进程的可能先执行
            /// 此时子进程的创建尚未完成。
            if pid == -1 {
                yield_();
                continue;
            }
            
            /// 这里我们只处理了-1（进程未创建），而没有处理-2（进程尚未结束）
            println!(
                "[initproc] Released a zombie process, pid={}, exit_code={}",
                pid, exit_code,
            );
        }
    }
    0
}
```

#### shell程序-user_shell

user_shell 需要捕获用户输入并进行解析处理，为此添加一个能获取用户输入的系统调用：

```rust
/// 功能：从文件中读取一段内容到缓冲区。

/// 参数：fd 是待读取文件的文件描述符，切片 buffer 则给出缓冲区。

/// 返回值：如果出现了错误则返回 -1，否则返回实际读到的字节数。

/// syscall ID：63
pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize;
```

实际调用时，我们必须要同时向内核提供缓冲区的起始地址及长度：

```rust
// user/src/syscall.rs

pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize {
    syscall(SYSCALL_READ, [fd, buffer.as_mut_ptr() as usize, buffer.len()])
}
```

下面是在 `initproc` 中启动的一个 `shell` 程序，**该程序从命令行输入指令并创建子进程，在子进程中执行输入的指令**。

```rust
#![no_std]
#![no_main]

extern crate alloc;

#[macro_use]
extern crate user_lib;

const LF: u8 = 0x0au8;          /// Line Feed
const CR: u8 = 0x0du8;          /// Carriage Return
const DL: u8 = 0x7fu8;          /// Delete
const BS: u8 = 0x08u8;          /// Backspace

use alloc::string::String;
use user_lib::console::getchar;
use user_lib::{exec, flush, fork, waitpid};

#[no_mangle]
pub fn main() -> i32 {
    println!("Rust user shell");
    let mut line: String = String::new();
    print!(">> ");
    flush();
    loop {
        let c = getchar();
        match c {
            // 如果读到一个换行符，我们就执行当前行的命令
            LF | CR => {
                print!("\n");
                if !line.is_empty() {
                    if line.eq("exit") {
                        break 1;
                    };
                    // 前面提到的， exec 需要一个以 '\0' 结尾的字符串
                    line.push('\0');
                    let pid = fork();
                    if pid == 0 {
                        /// 子进程逻辑
                        if exec(line.as_str(), &[0 as *const u8]) == -1 {
                            println!("Error when executing!");
                            return -4;
                        }
                        unreachable!();
                    } else {
                        // 父进程逻辑等待子进程结束
                        let mut exit_code: i32 = 0;
                        let exit_pid = waitpid(pid as usize, &mut exit_code);
                        assert_eq!(pid, exit_pid);
                        println!("Shell: Process {} exited with code {}", pid, exit_code);
                    }
                    line.clear();
                }
                print!(">> ");
                flush();
            }
            // 如果读到一个退格符或删除符
            BS | DL => {
                if !line.is_empty() {
                    print!("{}", BS as char);
                    print!(" ");
                    print!("{}", BS as char);
                    flush();
                    line.pop();
                }
            }
            _ => {
                print!("{}", c as char);
                flush();
                line.push(c as char);
            }
        }
    }
}

```

## 进程管理的核心数据结构

- 基于应用名的应用链接/加载器
- 进程标识符 `PidHandle` 以及内核栈 `KernelStack`
- 任务控制块 `TaskControlBlock`
- 任务管理器 `TaskManager`
- 处理器管理结构 `Processor`

### 基于应用名的应用链接/加载器

**基于应用名的应用链接/加载器** 和之前的加载器并无本质区别：

- 除了老的加载器中生成的指针 `_num_app` 用来解析**app数量和函数地址**之外；

- 新的加载器会额外的生成一个指针 `_app_names`，用来解析线上全部的应用名 。
- 两个指针的区别在于：
  - `_num_app` 会用来解析 `_num_app + 2` 个元素，`_app_names` 会用来解析 `_num_app` 个元素；
  - `_num_app` 指向的第一个元素为app数量，后面的 `[ptr + 1, ptr + _num_app + 1]` 总共 `_num_app + 1` 个元素表示的是函数地址；
  - `_app_names` 指向的 `[0, _num_app - 1]` 个元素都是应用名； 

最终生成的文件格式如下：唯一需要注意的是 `.string "ch2b_bad_address"` 会自动的在 `.string` 后面增加 `\0` 作为结尾，**但是这个和 exec 的参数必须以 '\0' 没有关系，可以参考 [exec](#exec)**

```nasm
    .global _num_app
_num_app:
    .quad 19
    .quad app_0_start
	# 省略17个元素
    .quad app_18_start
    .quad app_18_end

    .global _app_names
_app_names:
    .string "ch2b_bad_address"
	# 省略17个元素
    .string "ch5b_usertest"

    .section .data
    .global app_0_start
    .global app_0_end
    .align 3
app_0_start:
    .incbin "../user/build/elf/ch2b_bad_address.elf"
app_0_end:
```

### 进程标识符和内核栈

#### 进程标识符

涉及进程有关的struct和对象有三个：

1. `PidHandle` 表示进程时生成的唯一标识符；
2. `PidAllocator` 用于管理 `pid` -- 负责为新创建的进程分配 `pid`，在进程退出时回收 `pid`；
3. `PID_ALLOCATOR` 全局的唯一 `PidAllocator` 实例；

```mermaid
flowchart LR

PidHandler("PidHandler")

PidAllocator("PidAllocator")
PID_ALLOCATOR("PID_ALLOCATOR")


PidAllocator --o|延迟加载唯一实例| PID_ALLOCATOR


Kernel("Kernel") -.->|请求分配PID| PID_ALLOCATOR
PID_ALLOCATOR -.->|分配PID| PidHandler 

Kernel --> Process("Process")
PidHandler --> Process
```

#### 内核栈

![kernel-as-high](/images/20251121/kernel-as-high.png)

### 进程控制块

#### PCB

```rust
pub struct TaskControlBlock {
    pub pid: PidHandle,
    pub kernel_stack: KernelStack,
    inner: UPSafeCell<TaskControlBlockInner>,
}

pub struct TaskControlBlockInner {
    pub trap_cx_ppn: PhysPageNum,
    pub base_size: usize,
    pub task_cx: TaskContext,
    pub task_status: TaskStatus,
    pub memory_set: MemorySet,
    pub heap_bottom: usize,
    pub program_brk: usize,

    pub parent: Option<Weak<TaskControlBlock>>,
    pub children: Vec<Arc<TaskControlBlock>>,
    pub exit_code: i32,
}
```

#### TCB

```rust
pub struct TaskControlBlock {
    pub trap_cx_ppn: PhysPageNum,
    pub base_size: usize,
    pub task_cx: TaskContext,
    pub task_status: TaskStatus,
    pub memory_set: MemorySet,
    pub heap_bottom: usize,
    pub program_brk: usize,
}
```

#### 实现

> 对比一下 `PCB` （虽然名字还是 TaskControlBlock） 和 `TCB` 的实现，我们可以总结以下几点：
>
> 1. 新的 `PCB` 新增了四个
> 2. `base_size`, `heap_bottom`, `program_brk` 的解析请参考 [应用进程内存模型](#应用进程内存模型)

### 任务管理器

#### 老版本

在老版本的 `TaskManager` 中，它需要负责的功能很杂：

1. 执行任务；
2. 调度任务；
3. 查询任务元数据；
4. 代理当前任务的堆调整操作；

```mermaid
flowchart LR

TaskManager("TaskManager"):::pink

run("执行任务"):::purple
schedule("调度任务"):::purple
meta("查询任务元数据"):::purple
heap("堆调整操作"):::purple

run_first_task("run_first_task"):::green
run_next_task("run_next_task"):::green
find_next_task("find_next_task"):::green
mark_current_suspended("mark_current_suspended"):::green
mark_current_exited("mark_current_exited"):::green
get_current_token("get_current_token"):::green
get_current_trap_cx("get_current_trap_cx"):::green
change_current_program_brk("change_current_program_brk"):::green

TaskManager --> run
run -->|执行首个任务| run_first_task
run -->|执行下一个任务| run_next_task

TaskManager --> schedule
schedule -->|查找下一个待执行任务| find_next_task
schedule -->|标记当前任务挂起| mark_current_suspended
schedule -->|标记当前任务退出| mark_current_exited

TaskManager --> meta
meta -->|获取当前任务分页访问信息| get_current_token
meta -->|获取当前陷阱上下文| get_current_trap_cx

TaskManager --> heap -->|调整当前程序堆指针| change_current_program_brk

classDef pink 1,fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
classDef green fill: #696,color: #fff,font-weight: bold;
classDef purple fill:#969,stroke:#333, font-weight: bold;
classDef error fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
classDef coral fill:#f9f,stroke:#333,stroke-width:4px;
classDef animate stroke-dasharray: 9,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;
```

```rust
pub struct TaskManager {
    num_app: usize,
    inner: UPSafeCell<TaskManagerInner>,
}

struct TaskManagerInner {
    tasks: Vec<TaskControlBlock>,
    current_task: usize,
}

impl TaskManager {
    fn run_first_task(&self) -> ! {}
    fn run_next_task(&self) {}
    
    fn find_next_task(&self) -> Option<usize> {}
    fn mark_current_suspended(&self) {}
    fn mark_current_exited(&self) {}
    
    fn get_current_token(&self) -> usize {}
    fn get_current_trap_cx(&self) -> &'static mut TrapContext {}

    pub fn change_current_program_brk(&self, size: i32) -> Option<usize> {}
}
```

#### 新版本

在新的版本中，我们将它拆分为了 `TaskManager` 和 `Processor` 两个独立的结构：

- `TaskManager` 负责管理所有 `TaskStatus::Ready` 的结构，只包含了两个方法：
  - `add` 添加一个 `TaskStatus::Ready` 的 Task；
  - `fetch` 使用先进先出的顺序，获取一个可执行的 Task；
- `Processor` 只负责 `Task` 的执行：
  - 内部包含两个变量：
    - `current` 包含了当前执行的Task的 `TCB`，这里值得注意的是，`TCB` 对象被包在 `Arc` 中，可能为None；
    - `idle_task_cx` 用于中转逻辑；
  - 内部提供了三个方法：
    - `get_idle_task_cx_ptr` 返回内部持有的 TaskContext
    - `take_current` move `TCB`，在 `suspend` 和 `exit` 这两个会放弃执行时间片的情况下才调用；
    - `current` 共享 `TCB`，用于查询 `TCB` 信息时的调用。

```mermaid
flowchart LR
    subgraph init["初始化阶段：无用户任务"]
        direction LR
        registers_0("registers.cx: idle运行状态"):::pink
        current_0("current: None"):::purple
        idle_task_cx_0("idle_task_cx: zero.ctx（占位）"):::green
    end

    subgraph stage_1["阶段1：fetch_task"]
        direction LR
        registers_1("registers.cx: task.cx（用户任务状态）"):::pink
        current_1("current: Some(task)"):::purple
        idle_task_cx_1("idle_task_cx: 原registers.cx（idle休眠状态）"):::green
    end

    subgraph stage_2["阶段2：suspend"]
        direction LR
        registers_2("registers.cx: idle运行状态（从idle_task_cx_1加载）"):::pink
        current_2("current: None"):::purple
        idle_task_cx_2("idle_task_cx: 原registers_1（用户任务挂起状态）"):::green
    end

    subgraph stage_3["阶段3：exit→用户任务退出"]
        direction LR
        registers_3("registers.cx: idle运行状态（从idle_task_cx_1加载）"):::pink
        current_3("current: None"):::purple
        idle_task_cx_3("idle_task_cx: 原idle_task_cx_1（无变化）"):::green
    end

    subgraph TaskManager["TaskManager：就绪队列"]
        t_task("task.cx: 用户任务挂起状态（仅suspend后存在）"):::coral
    end

    %% 流转逻辑（补充关键动作标注）
    init -->|init| stage_1
    stage_1 -->|suspend| stage_2
    stage_1 -->|exit| stage_3

    %% 状态转移细节
    current_1 -->|take_current后add_task，共享task所有权| TaskManager
    registers_1 -->|__switch保存用户任务状态到task.cx| t_task
    idle_task_cx_1 -->|__switch加载idle状态到寄存器| registers_2
    idle_task_cx_1 -->|__switch加载idle状态到寄存器| registers_3

    %% 样式定义（沿用你的设计）
    classDef pink fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
    classDef green fill: #696,color: #fff,font-weight: bold;
    classDef purple fill:#969,stroke:#333, font-weight: bold;
    classDef error fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5;
    classDef coral fill:#f9f,stroke:#333,stroke-width:4px;
    classDef animate stroke-dasharray: 9,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;
```

> 整体可以总结为：
>
> 1. 当由内核态（idle）切换到用户态时，利用 `idle_task_cx` 保存内核上下文，从 `task.task_cx` 加载用户上下文；
> 2. 当由用户态切换回内核态（idle）时，根据是否需要继续执行，决定是否将 `task_cx` 保存到 `PCB`；
>    1. `suspend` 要复用上下文 → 存到 `PCB`；
>    2. `exit` 不复用 → 存到临时变量丢弃。

假设存在以下状态：

- `kernel_cx`：内核态上下文（如 `idle_task` 运行时的上下文，包含合法的 `ra`/`sp`，不是 `zero_init`）；
- `zero_init`：仅用于 “临时占位” 的全零值，仅在初始化或任务退出时出现，不会被 CPU 执行；
- `task.cx`：用户任务的合法上下文（已初始化，可被 CPU 执行）。

整个的执行流程可以如下描述：

1. **初始化 `Processor`**：

   - `current = None`（无用户任务）；
   - `idle_task_cx = zero_init`（占位用，无实际执行意义）；
   - `registers = kernel_cx`（CPU 运行 `idle_task`，寄存器是 `idle` 的合法内核态上下文）；

2. **`fetch_task`（首次调度用户任务）**：

   - `current = Some(task)`（绑定当前用户任务）；
   - `idle_task_cx = kernel_cx`（通过 `__switch` 保存之前 `idle_task` 的内核态上下文，覆盖初始的 `zero_init`）；
   - `registers = task.cx`（通过 `__switch` 加载用户任务上下文，CPU 开始执行用户任务）；

3. **分叉场景**：

   3.1 **`suspend`（挂起当前任务）**：

   - `current = None`（`take_current_task` 移动 `task` 所有权）；
   - `idle_task_cx = task.cx`（通过 `__switch` 保存用户任务的运行状态）；
   - `registers = kernel_cx`（通过 `__switch` 加载 `idle_task` 的内核态上下文，CPU 切回 `idle`）；
   - 额外：`TaskManager` 中添加 `task`（`task.cx` 保留用户任务状态，后续可恢复）；

   3.2 **`exit`（当前任务退出）**：

   - `current = None`（`take_current_task` 移动 `task` 所有权，后续 `drop(task)` 释放 TCB）；
   - `idle_task_cx = kernel_cx`（保持不变，仍为 `idle_task` 的内核态上下文）；
   - `registers = kernel_cx`（通过 `__switch` 加载 `idle_task` 的内核态上下文，CPU 切回 `idle`）；
   - 额外：用 `zero_init` 临时变量接收退出任务的状态（会被丢弃，不影响 `idle_task_cx` 和寄存器）；

```rust
pub struct TaskManager {
    ready_queue: VecDeque<Arc<TaskControlBlock>>,
}

impl TaskManager {
    pub fn new() -> Self {}
    pub fn add(&mut self, task: Arc<TaskControlBlock>) {}
    pub fn fetch(&mut self) -> Option<Arc<TaskControlBlock>> {}
}
pub struct Processor {
    current: Option<Arc<TaskControlBlock>>,
    idle_task_cx: TaskContext,
}

impl Processor {
    pub fn new() -> Self {}
    fn get_idle_task_cx_ptr(&mut self) -> *mut TaskContext {}
    pub fn take_current(&mut self) -> Option<Arc<TaskControlBlock>> {}
    pub fn current(&self) -> Option<Arc<TaskControlBlock>> {}
}
```

## 进程管理机制的设计实现

### 初始进程的创建

> 关于 `elf` 解析可以参考 [elf解析](#elf解析)

> 与老版本一致的逻辑

1. 解析 `elf` 并初始化内存模型：`.data`, `.rodata`, `trampoline`, `trap_context` 等；
2. 初始化必要信息，例如 `trap_cx_ppn`，内核栈等；
3. 为用户态 `TrapContext` 赋值；

> 与老版本不一致的逻辑

1. 需要为每个task生成一个唯一的 pid；
2. 使用 `kstack_alloc()` 代替了 `KERNEL_SPACE`；具体的逻辑请参考 [进程的内存模型](#进程的内存模型)；

```rust
impl TaskControlBlock {
    /// Create a new process
    ///
    /// At present, it is only used for the creation of initproc
    pub fn new(elf_data: &[u8]) -> Self {
        // memory_set with elf program headers/trampoline/trap context/user stack
        let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
        let trap_cx_ppn = memory_set
            .translate(VirtAddr::from(TRAP_CONTEXT_BASE).into())
            .unwrap()
            .ppn();
        // alloc a pid and a kernel stack in kernel space
        let pid_handle = pid_alloc();
        let kernel_stack = kstack_alloc();
        let kernel_stack_top = kernel_stack.get_top();
        // push a task context which goes to trap_return to the top of kernel stack
        let task_control_block = Self {
            pid: pid_handle,
            kernel_stack,
            inner: unsafe {
                UPSafeCell::new(TaskControlBlockInner {
                    trap_cx_ppn,
                    base_size: user_sp,
                    task_cx: TaskContext::goto_trap_return(kernel_stack_top),
                    task_status: TaskStatus::Ready,
                    memory_set,
                    parent: None,
                    children: Vec::new(),
                    exit_code: 0,
                    heap_bottom: user_sp,
                    program_brk: user_sp,
                })
            },
        };
        // prepare TrapContext in user space
        let trap_cx = task_control_block.inner_exclusive_access().get_trap_cx();
        *trap_cx = TrapContext::app_init_context(
            entry_point,
            user_sp,
            KERNEL_SPACE.exclusive_access().token(),
            kernel_stack_top,
            trap_handler as usize,
        );
        task_control_block
    }
}
```

### 进程调度机制

进程的调度主要通过 `suspend_current_and_run_next` 来实现，该函数在执行时通常会包含如下三个重要状态：

1. `registers` 是当前执行的 task 的 `cx`；
2. `TaskControlBlock#task_cx` 它的当前状态不重要，因为我们要用他来存储目前 `registers` 的状态，这样在下次调度的时候可以加载到当前状态；
3. `Processor#idle_task_cx` 保存了上次从 `内核态` 切换到 `用户态` 的时候的 `registers`；

当我们执行 `__switch(TaskControlBlock#task_cx, Processor#idle_task_cx)`，实际是将当前状态存储到 `TCB`，并且恢复到内核态。

具体的内部状态流转可以参考 [任务管理器](#任务管理器) 的说明。

### 进程的生成机制

#### fork 系统调用的实现

`fork` 系统调用的实现存在两个需要注意的地方：

1. 修改用户进程的信息：为子进程生成一个接近一模一样的内存空间和布局（包括堆和栈）。需注意：这里未实现 COW（写时复制），所有 MapArea 都重新分配了物理内存；实现逻辑不复杂，直接将原进程的地址空间 MemorySet 复制到新进程即可。其中有两个特殊的 MapArea 需要单独处理：
   1. `trampoline`：全局共享的物理页，属于 “特殊映射”（固定虚拟地址、仅可执行权限、全进程共享物理页），不会被包含在普通 MapArea 的遍历复制中，因此无法通过 for 循环完成 VA→PA 映射，需额外调用 `map_trampoline`（无需重新分配物理页，仅复用全局共享页完成映射）；
   2. `trap_ctx`：完整复制父进程的 `trap_ctx` 内容，仅需修改 `kernel_sp` 字段（指向子进程新分配的内核栈顶，这个是内核态才会用到的），其余字段保持不变。
2. **修改PCB信息**：
   1. 重新分配了 `pid`；
   2. 重新分配了 `kernel_stack`，这里是基于我们新实现的 `KSTACK_ALLOCATOR` 分配的；
   3. 因为重新分配了 `kernel_stack`，所以 `kernel_stack_top` 也改变了；
   4. 初始化 `task_cx`，**这里非常重要，因为当该进程初始化完毕被调度的时候，会直接将 `task_cx` 作为第一次的上下文；所以它的 `ra = trap_return`，`sp = kernel_stack_top`* *
      - `ra = trap_return`：表示子进程首次执行时，会先跳转到 `trap_return` 函数（完成内核态→用户态的切换）；
      - `sp = kernel_stack_top`：表示子进程的内核栈指针指向新分配的内核栈顶，这也体现了我们的栈从上往下扩展的特性；
   5. **隐式的修改 `sepc`，父进程在执行完 `fork()` 之后CPU 会自动将 sepc 设置为 ecall 下一条指令的地址（在riscv中是 sepc = sepc + 4），而我们完整的复制了全部的 `trap_cx`，也就是相当于隐式的修改了 `sepc`，这样才保证子进程在第一次被调度的时候不会陷入死循环**。
   6. 修改父进程和子进程的其他关联信息；


##### MemorySet#from_existed_user

```rust
impl MemorySet {
    pub fn from_existed_user(user_space: &Self) -> Self {
        let mut memory_set = Self::new_bare();
        // map trampoline
        memory_set.map_trampoline();
        // copy data sections/trap_context/user_stack
        for area in user_space.areas.iter() {
            let new_area = MapArea::from_another(area);
            memory_set.push(new_area, None);
            // copy data from another space
            for vpn in area.vpn_range {
                let src_ppn = user_space.translate(vpn).unwrap().ppn();
                let dst_ppn = memory_set.translate(vpn).unwrap().ppn();
                dst_ppn
                    .get_bytes_array()
                    .copy_from_slice(src_ppn.get_bytes_array());
            }
        }
        memory_set
    }
}
```

##### MapArea#from_another

```rust
impl MapArea {
    pub fn from_another(another: &Self) -> Self {
        Self {
            vpn_range: VPNRange::new(another.vpn_range.get_start(), another.vpn_range.get_end()),
            data_frames: BTreeMap::new(),
            map_type: another.map_type,
            map_perm: another.map_perm,
        }
    }
}
```

##### sys_fork的实现

随后，我们还有一点需要注意的是：`sys_fork` 进程会存在两个返回值：

1. 对于父进程会返回子进程的 `pid`；
2. 对于子进程会返回 `0`。

这里我们的实现逻辑是：

1. 对于父进程，我们直接返回 `pid`；
2. 对于子进程，我们修改它的 `trap_cx` 中的 `x10`，也就是 `trap_cx.x[10]`；

```rust
pub fn sys_fork() -> isize {
    trace!("kernel:pid[{}] sys_fork", current_task().unwrap().pid.0);
    let current_task = current_task().unwrap();
    let new_task = current_task.fork();
    let new_pid = new_task.pid.0;
    // modify trap context of new_task, because it returns immediately after switching
    let trap_cx = new_task.inner_exclusive_access().get_trap_cx();
    // we do not have to move to next instruction since we have done it before
    // for child process, fork returns 0
    trap_cx.x[10] = 0;
    // add new task to scheduler
    add_task(new_task);
    new_pid as isize
}
```

#### exec 系统调用的实现

> 这里注意，我们前面提到的：
>
> 1. 如果出错的话（如找不到名字相符的可执行文件）则返回 -1
> 2. 否则不应该返回。
>
> 这里在执行不出错的时候，`exec` 函数会直接用新的应用的内存空间去覆盖老的应用的内存空间。

`exec` 方法的实现会比 `fork` 要简单很多，我们需要如下操作：

1. 从 `elf` 中读取新的 `memory_set`；

2. **修改 `PCB` 中加载前和加载后发生变化的变量**，参考下面的模型图很容易得出：

   1. 发生了变化的包括：
      1. `memory_set` 发生了变化；
      2. 由于新应用中的 `.text/.rodata/.data/.bss` 和原应用的长度变化，所以 `user_stack_top` 发生了变化；
      3. `base_size`，`heap_bottom`，`program_brk` 这几个值初始和 `user_stack_top` 一直，所以也需要变化；
      4. `trap_cx_ppn` 重新分配了物理页；
      5. `task_ctx`
   2. 未发生变化的：
      1. `parent`
      2. `children`

   可以明显看到，和 `fork` 的最大区别就是，fork完全继承了父进程的memory_set。

```mermaid
block-beta
    columns 3
        block
            columns 1
            space:6
            user_stack_top("user_stack_top")
            base_size("base_size")
            heap_bottom("heap_bottom")
            program_brk("program_brk")
            space:5
        end
        block
            columns 1
            space:8
            eUserStack("User Stack End")
            mUserStack("...")
            sUserStack("User Stack Start")
            sGuardPage("Guard Page")
            sbss(".bss")
            sdata(".data")
            srodata(".rodata")
            stext(".text")
        end

    block:group4:1
        columns 1
        trampoline("Trampoline")
        TrapContext("TrapContext")
        space:14
    end

    user_stack_top --> eUserStack
    base_size --> eUserStack
    heap_bottom --> eUserStack
    program_brk --> eUserStack
    
    style user_stack_top fill:linear-gradient(to top, #fff7e6, #fff3e0),stroke:#ff7a45,stroke-width:3px,color:#d4380d,padding:8px,font-weight:bold,border-radius:4px 4px 0 0
    style base_size fill:linear-gradient(to top, #fff7e6, #fff3e0),stroke:#ff7a45,stroke-width:3px,color:#d4380d,padding:8px,font-weight:bold,border-radius:4px 4px 0 0
    style heap_bottom fill:linear-gradient(to top, #fff7e6, #fff3e0),stroke:#ff7a45,stroke-width:3px,color:#d4380d,padding:8px,font-weight:bold,border-radius:4px 4px 0 0
    style program_brk fill:linear-gradient(to top, #fff7e6, #fff3e0),stroke:#ff7a45,stroke-width:3px,color:#d4380d,padding:8px,font-weight:bold,border-radius:4px 4px 0 0

    style eUserStack fill:linear-gradient(to bottom, #e6f7ff, #f0f8ff),stroke:#1890ff,stroke-width:3px,color:#0047ab,padding:8px,border-radius:0 0 4px 4px
    style mUserStack fill:linear-gradient(to bottom, #e6f7ff, #f0f8ff),stroke:#1890ff,stroke-width:3px,color:#0047ab,padding:8px,border-radius:0 0 4px 4px
    style sUserStack fill:linear-gradient(to bottom, #e6f7ff, #f0f8ff),stroke:#1890ff,stroke-width:3px,color:#0047ab,padding:8px,border-radius:0 0 4px 4px

    style sbss fill:linear-gradient(to bottom, #f0fff4, #f5fffa),stroke:#52c41a,stroke-width:2px,color:#237804,padding:8px
    style sdata fill:linear-gradient(to bottom, #fffbe6, #fffdf2),stroke:#faad14,stroke-width:2px,color:#aa5800,padding:8px
    style srodata fill:linear-gradient(to bottom, #f9f0ff, #fcf1ff),stroke:#722ed1,stroke-width:2px,color:#531dab,padding:8px
    style stext fill:linear-gradient(to bottom, #f5f5f5, #fafafa),stroke:#8c8c8c,stroke-width:2px,color:#333333,padding:8px

    style sGuardPage fill:#f0f0f0,stroke:#666666,stroke-width:2px,stroke-dasharray:5,5,color:#333333,padding:8px,font-weight:bold


    style trampoline fill:#f0f0f0,stroke:#666666,stroke-width:2px,stroke-dasharray:5,5,color:#333333,padding:8px,font-weight:bold
    style TrapContext fill:#e8e8e8,stroke:#555555,stroke-width:2px,stroke-dasharray:5,5,color:#222222,padding:8px,font-weight:bold
```

```rust
impl TaskControlBlock
    /// Load a new elf to replace the original application address space and start execution
    pub fn exec(&self, elf_data: &[u8]) {
        // memory_set with elf program headers/trampoline/trap context/user stack
        let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
        let trap_cx_ppn = memory_set
            .translate(VirtAddr::from(TRAP_CONTEXT_BASE).into())
            .unwrap()
            .ppn();

        // **** access current TCB exclusively
        let mut inner = self.inner_exclusive_access();
        // substitute memory_set
        inner.memory_set = memory_set;
        // update trap_cx ppn
        inner.trap_cx_ppn = trap_cx_ppn;
        // initialize base_size
        inner.base_size = user_sp;
        // initialize trap_cx
        let trap_cx = inner.get_trap_cx();
        *trap_cx = TrapContext::app_init_context(
            entry_point,
            user_sp,
            KERNEL_SPACE.exclusive_access().token(),
            self.kernel_stack.get_top(),
            trap_handler as usize,
        );
        // **** release inner automatically
    }
end
```

#### 系统调用后重新获取 Trap 上下文

```rust {16}
// os/src/trap/mod.rs

#[no_mangle]
pub fn trap_handler() -> ! {
    set_kernel_trap_entry();
    let scause = scause::read();
    let stval = stval::read();
    match scause.cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            // jump to next instruction anyway
            let mut cx = current_trap_cx();
            cx.sepc += 4;
            // get system call return value
            let result = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]);
            // cx is changed during sys_exec, so we have to call it again
            cx = current_trap_cx();
            cx.x[10] = result as usize;
        }
        ...
    }
    trap_return();
}
```

在执行 `exec` 之后，`trap_cx` 已经失效了，所以执行完之后需要重新获取 `trap_cx`。
```diff
             let result = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]);
+            // cx is changed during sys_exec, so we have to call it again
+            cx = current_trap_cx();
```

## 进程资源回收机制

#### 进程的退出

进程的退出通过 `exit_current_and_run_next` 函数来执行：

1. `take_current_task()` 获取当前 Processor 中 current 对象的所有权，以便于 `PCB` 能被正常的回收；这里需要注意的是：`current` 是一个 `Option<Arc<TaskControlBlock>>` 对象，它可能会被其他的 `PCB` 引用。例如，他的父线程的 `child` 和他子线程的 `parent` 都会有对他的引用；
2. 修改 `PCB` 的状态 **task_status** 为 `TaskStatus::Zombie`，设置 `exit_code`；这里设置状态为 `zombie` 的目的是，为了让父进程在执行 `wait` 的时候能够返回对应的信息。
3. 在退出之前，将它内部所有的 `child` 的父进程修改为 `INITPROC`，避免资源泄露；
4. 清空 `children` 和 `map_area`，其实理论上来说，这里不清空理论上也是可以的。但是存在其他引用一直引用到 `self` 导致自身不能被正确回收，从而导致这些引用也无法被回收的风险；

```rust
/// Exit the current 'Running' task and run the next task in task list.
pub fn exit_current_and_run_next(exit_code: i32) {
    // take from Processor
    let task = take_current_task().unwrap();
    // **** access current TCB exclusively
    let mut inner = task.inner_exclusive_access();
    // Change status to Zombie
    inner.task_status = TaskStatus::Zombie;
    // Record exit code
    inner.exit_code = exit_code;
    // do not move to its parent but under initproc

    // ++++++ access initproc TCB exclusively
    {
        let mut initproc_inner = INITPROC.inner_exclusive_access();
        for child in inner.children.iter() {
            child.inner_exclusive_access().parent = Some(Arc::downgrade(&INITPROC));
            initproc_inner.children.push(child.clone());
        }
    }
    // ++++++ release parent PCB

    inner.children.clear();
    // deallocate user space
    inner.memory_set.recycle_data_pages();
    drop(inner);
    // **** release current PCB
    // drop task manually to maintain rc correctly
    drop(task);
    // we do not have to save task context
    let mut _unused = TaskContext::zero_init();
    schedule(&mut _unused as *mut _);
}
```

#### 父进程回收子进程资源

父进程对子进程的资源回收，是通过 `sys_waitpid` 来实现的：

1. 查找是否存在目标进程：目标进程与输入参数 `pid` 有关，如果不存在任何目标进程返回 `-1`：
   1. `pid == -1` 匹配任意一个进程；
   2. `pid != -1` 匹配与 `pid` 一致的进程；
2. 查找是否存在**已经结束（task_status == TaskStatus::Zombie）**的目标进程，如果不存在则返回 `-2`：
   1. `pid == -1` 匹配任意一个进程；
   2. `pid != -1` 匹配与 `pid` 一致的进程；
3. 找到目标进程，我们开始回收子进程资源：
   1. 从 `children` 中移除当前子进程；
   2. 保证当前 `child` 只有一个强引用，这个位置的设计很有意思可以参考 [children和parent](#children和parent)；
   3. 设置 `exit_code` 并写入 `exit_code_ptr`  指针指向的位置，这个是在系统调用时给用户态应用返回的；
   4. 函数返回 `pid`，这个是内核态自己函数调用的返回值。

```rust
/// If there is not a child process whose pid is same as given, return -1.
/// Else if there is a child process but it is still running, return -2.
pub fn sys_waitpid(pid: isize, exit_code_ptr: *mut i32) -> isize {
    trace!("kernel::pid[{}] sys_waitpid [{}]", current_task().unwrap().pid.0, pid);
    let task = current_task().unwrap();
    // find a child process

    // ---- access current PCB exclusively
    let mut inner = task.inner_exclusive_access();
    if !inner
        .children
        .iter()
        .any(|p| pid == -1 || pid as usize == p.getpid())
    {
        return -1;
        // ---- release current PCB
    }
    let pair = inner.children.iter().enumerate().find(|(_, p)| {
        // ++++ temporarily access child PCB exclusively
        p.inner_exclusive_access().is_zombie() && (pid == -1 || pid as usize == p.getpid())
        // ++++ release child PCB
    });
    if let Some((idx, _)) = pair {
        let child = inner.children.remove(idx);
        // confirm that child will be deallocated after being removed from children list
        assert_eq!(Arc::strong_count(&child), 1);
        let found_pid = child.getpid();
        // ++++ temporarily access child PCB exclusively
        let exit_code = child.inner_exclusive_access().exit_code;
        // ++++ release child PCB
        *translated_refmut(inner.memory_set.token(), exit_code_ptr) = exit_code;
        found_pid as isize
    } else {
        -2
    }
    // ---- release current PCB automatically
}
```

## QA

### user_stack和kernel_stack

`user_stack`  和 `kernel_stack` 都是与进程绑定的栈，每一个进程都会有唯一的 `user_stack` 和 `kernel_stack`，分别负责他们在用户态和内核态的函数执行时的栈操作，区别在于：

1. `user_stack` 映射在用户内存空间，而 `kernel_stack` 映射在内核内存空间；
2. 用户程序可以操作 `user_stack` 的指针，但是不能操作 `kernel_stack` 的指针；
3. 当用户通过 `trap` 或 `exception` 切换到内核态时，由 `__alltraps` 负责将 `sp` 切换到指向 `kernel_stack`；当从内核态切换回用户态时，由 `__restore` 负责将 `sp` 切换到指向 `user_stack`。

| 特性         | user_stack（用户栈）                        | kernel_stack（内核栈）                       |
| ------------ | ------------------------------------------- | -------------------------------------------- |
| 内存空间     | 用户虚拟地址空间（如 0x0~0x7FFFFFFF）       | 内核虚拟地址空间（如 0x80000000~0xFFFFFFFF） |
| 访问权限     | 进程可读写（用户态直接操作 `sp`）           | 进程不可访问（仅内核态可操作）               |
| 作用         | 支撑用户态函数调用（如 `main`/`clear_bss`） | 支撑内核态逻辑（如 `sys_exit`/ 异常处理）    |
| 生命周期     | 随进程创建而分配，进程退出而释放            | 同进程生命周期（和用户栈绑定）               |
| 栈指针控制权 | 用户程序可修改 `sp`（如 `addi sp,sp,-16`）  | 用户程序无法修改，仅内核通过汇编切换         |

### 进程的退出

> 仔细观察我们的 `sys_exit` 函数，我们很多的程序并没有显示的调用该系统调用，那他们会正常的退出吗？

答案是肯定！每个进程都会退出，但是他们是怎么退出的？先说结论：进程的退出，可能是多种情况。例如：显示的调用 `eixt`，预见未捕获的异常（例如除零异常），发起 `SIGKILL` 信号。但是如果我们以上操作都没有发生进程会怎么退出呢？答案是：

1. 在 `_start` 函数的结尾，我们通过 `exit(main(argc, v.as_slice()))` 来获取 `main` 函数的返回值并作为进程的退出码；
2. 但是，如果我们在 `main` 中显示的调用 `exit()`，那么程序将直接结束而不会进入到 `exit(main(argc, v.as_slice()))` 这里；

但是这是如何实现的呢？

#### config.toml

在 `config.tml` 中，我们定义了编译参数为 **-Clink-args=-Tsrc/linker.ld**

```rust
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-args=-Tsrc/linker.ld",
]
```

#### Makefile

在 `Makefile` 编译文件的过程中，将会使用前面提到的参数来指定链接文件。

```makefile
binary:
	@echo $(ELFS)
	@if [ ${CHAPTER} -gt 3 ]; then \
		cargo build $(MODE_ARG) ;\
	else \
		CHAPTER=$(CHAPTER) python3 build.py ;\
	fi
	@$(foreach elf, $(ELFS), \
		$(OBJCOPY) $(elf) -O binary $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.bin, $(elf)); \
		cp $(elf) $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.elf, $(elf));)
```

#### linker.ld

在 `linker.ld` 中，我们指定的入口函数为 `_start`。注意，这里我们将 `*(.text.entry)` 加入到了 `.text`，否则我们的程序将无法正常执行。

```assembly
OUTPUT_ARCH(riscv)
ENTRY(_start)

BASE_ADDRESS = 0x0;

SECTIONS
{
    . = BASE_ADDRESS;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
}
```

#### _start

最后，我们在 `lib.rs` 下定义了 `_start`：

```rust
#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start(argc: usize, argv: usize) -> ! {
    clear_bss();
    unsafe {
        HEAP.lock()
            .init(HEAP_SPACE.as_ptr() as usize, USER_HEAP_SIZE);
    }
    let mut v: Vec<&'static str> = Vec::new();
    for i in 0..argc {
		// process args
    }
    exit(main(argc, v.as_slice()));
}
```

这样，我们就将用户态程序的入口通过 `_start` 引导到了 `main` 函数，并且在 `main` 函数不调用 `exit()` 的情况下，保证程序可以正常的退出。

#### 汇编代码

我们先需要删除 `Makefile` 中的 `--strip-all` 来保留 `elf` 文件中的符号表：

```assembly
binary:
	@echo $(ELFS)
	@if [ ${CHAPTER} -gt 3 ]; then \
		cargo build $(MODE_ARG) ;\
	else \
		CHAPTER=$(CHAPTER) python3 build.py ;\
	fi
	@$(foreach elf, $(ELFS), \
		$(OBJCOPY) $(elf) -O binary $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.bin, $(elf)); \
		cp $(elf) $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.elf, $(elf));)
```

随后，使用 `riscv64-unknown-elf-objdump` 和 `rustfilt` 对elf文件进行反汇编：

```bash
riscv64-unknown-elf-objdump -d ../user/build/elf/ch5b_getpid.elf --disassemble=_start | rustfilt
```

我们可以得到实际的 `_start` 函数如下：

```assembly
../user/build/elf/ch5b_getpid.elf:     file format elf64-littleriscv


Disassembly of section .text:

0000000000000000 <_start>:
       0:	7165                	addi	sp,sp,-400
       # ...
       c:	00002097          	auipc	ra,0x2
      10:	b7e080e7          	jalr	-1154(ra) # 1b8a <user_lib::clear_bss>
      
000000000000009a <.Lpcrel_hi6>:
	   #: ...
     112:	ae8080e7          	jalr	-1304(ra) # 1bf6 <user_lib::exit>
```

我们可以看到，`exit()` 的兜底调用是在这里实现的。

### 进程状态机

```mermaid
flowchart LR

Ready("Ready"):::green
Running("Running"):::purple
Zombie("Zombie"):::error


Running -->|suspend_current_and_run_next| Ready
Running -->|fork| Ready

Ready -->|run_tasks| Running

Running -->|exit_current_and_run_next| Zombie



    classDef pink fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
    classDef green fill: #696,color: #fff,font-weight: bold;
    classDef purple fill:#969,stroke:#333, font-weight: bold;
    classDef error fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5;
    classDef coral fill:#f9f,stroke:#333,stroke-width:4px;
```



### KernelStack

```rust
/// Kernel stack for a process(task)
pub struct KernelStack(pub usize);
```

`KernelStack` 是进程管理中，用于在**Kernel Adress Space(High)**分配用户的应用内核栈的数据结构。

简单来说就是，在一个os中，每个函数的调用都必须有自己的函数栈，不论是在 `S` 模式还是在 `U` 模式，而 `KernelStack` 就是在 `S` 模式下的函数栈指针。

该指针的生命周期如下描述：

1. 进程在初始化时会映射虚拟地址 `TRAP_CONTEXT_BASE` 用于存储 `TrapContext`；
2. 随后初始化 `KernelStack`，可能有以下三种情况：
   1.  `INITPROC` 在初始化时通过 `TaskControlBlock::new()` 初始化，并通过 `TaskManager::add()` 加入等待调度；
   2. 在系统调用 `TaskControlBlock::exec()` 中通过 `TrapContext::app_init_context()` 初始化；
   3. 在系统调用 `TaskControlBlock::fork()` 中复制父进程的 `TrapContext` 并修改初始化；
3. 三种情况在初始化完成之后的调度略微有一些区别：
   1. `INITPORC` 的是直接将自身加入 `TASK_MANAGER`，等待内核调度；
   2. `exec()` 本身是系统调度，在初始化完成之后通过 `__restore` 来返回用户态；
   3. `fork()` 父进程的状态不变，子进程在 `fork()` 完得到子进程的地址空间后通过 `TaskManager::add()` 加入 `TASK_MANAGER` 等待调度。
4. 不管是任何情况，当发生 `trap` 时，内核通过 `__alltraps` 加载 `TrapContext` -- 而 `KernelStack` 也是 其中的一部分，随后内核态的所有函数调用的 `sp` 都是用 `KernelStack` 作为指针来操作自己的函数栈。

```mermaid
graph TD
    %% 样式定义：按流程阶段区分，突出逻辑层级
    classDef init fill:#e8f4f8, stroke:#2563eb, rounded:8px, font-weight:600;
    classDef branch fill:#fdf2f8, stroke:#9f7aea, rounded:8px, font-weight:600;
    classDef runtime fill:#e8f5e8, stroke:#2e7d32, rounded:8px, font-weight:600;
    classDef exit fill:#fef2f8, stroke:#f43f5e, rounded:8px, font-weight:600;
    classDef arrow stroke:#64748b, stroke-width:1.2px;

    %% 节点定义：特殊字符用双引号包裹，[]改为()
    A("进程初始化"):::init
    B{"初始化场景"}:::branch
    B1("INITPROC new()"):::branch
    B2("exec() 系统调用"):::branch
    B3("fork() 系统调用"):::branch
    C("分配内核栈，初始化 TrapContext（记录内核栈指针）"):::init
    D{"进程运行"}:::runtime
    D1("Trap 发生：__alltraps 切换到内核栈执行"):::runtime
    D2("进程调度：保存内核栈 sp 到 TrapContext，切换到其他进程"):::runtime
    E("进程退出（exit()）"):::exit
    F("释放内核栈占用的物理页，回收 TrapContext 映射"):::exit

    %% 逻辑连接：添加样式，保持原有流程
    A --> B:::arrow
    B --> B1:::arrow
    B --> B2:::arrow
    B --> B3:::arrow
    B1 & B2 & B3 --> C:::arrow
    C --> D:::arrow
    D --> D1:::arrow
    D --> D2:::arrow
    D1 & D2 --> E:::arrow
    E --> F:::arrow
```



### KSTACK_ALLOCATOR

`KSTACK_ALLOCATOR` 是负责 [KernelStack](#KernelStack) 的分配和回收的模块：这里非常值得注意的是，每一个 `KernelStack` 会分配固定大小的物理地址空间，而这个空间需要在进程退出的时候销毁，我们通过 RAII 实现：

```rust
impl Drop for KernelStack {
    fn drop(&mut self) {
        let (kernel_stack_bottom, _) = kernel_stack_position(self.0);
        let kernel_stack_bottom_va: VirtAddr = kernel_stack_bottom.into();
        KERNEL_SPACE
            .exclusive_access()
            .remove_area_with_start_vpn(kernel_stack_bottom_va.into());
        KSTACK_ALLOCATOR.exclusive_access().dealloc(self.0);
    }
}
```

### 内存对齐

> 内存对齐的本质是**适配 CPU 硬件的访存规则，平衡内存访问效率、硬件兼容性和内存空间利用率**，核心作用有三：

#### 内存对齐的根本作用

1. 避免 CPU 访问内存时的硬件异常（比如 RISC-V 的对齐故障）；
2. 最大化内存访问效率（CPU 按 “对齐块” 批量读写，而非拆分操作）；
3. 保证硬件 / 编译器的兼容性（比如 DMA、结构体布局跨平台一致）。

#### 为什么 CPU 要求内存对齐

现代 CPU 的内存访问不是 “字节级” 的，而是按**数据总线宽度 / 缓存行大小**（比如 64 位 CPU 总线宽度 8 字节，缓存行 64 字节）批量读取，对齐的核心是适配这种 “批量访存” 规则：

- CPU 访存时，会把内存划分为固定大小的 “对齐块”（比如 8 字节、16 字节）；
- 若数据起始地址是对齐块大小的整数倍（比如 8 字节对齐→地址是 8 的倍数），CPU 只需 1 次访存就能读取完整数据；
- 若未对齐，CPU 需要 2 次访存，再拼接数据（甚至直接触发硬件异常）。

| 场景           | 地址               | 对齐状态 | CPU 操作                                                     | 效率 |
| -------------- | ------------------ | -------- | ------------------------------------------------------------ | ---- |
| 对齐（8 字节） | 0x1000（8 的倍数） | 是       | 1 次访存：读取 0x1000~0x1007，直接拿到完整的 8 字节数据      | 高   |
| 未对齐         | 0x1001             | 否       | 2 次访存：先读 0x1000~0x1007（取后 7 字节），再读 0x1008~0x100F（取前 1 字节），拼接后得到 8 字节 | 低   |

#### 内核场景的对齐策略

| 数据类型                     | 对齐值          | 原因                                               |
| ---------------------------- | --------------- | -------------------------------------------------- |
| 基础类型（u8/u16/u32/u64）   | 自身大小        | 适配 CPU 访存宽度                                  |
| 指针（*mut T）               | 8 字节（64 位） | 64 位 CPU 的地址总线宽度是 8 字节                  |
| 普通结构体（FileDescriptor） | 最大成员对齐值  | 平衡效率与空间（比如包含 u64 的结构体 8 字节对齐） |
| 核心结构体（TCB / 页表）     | 4KB（页对齐）   | 适配物理页分配，简化内存管理，避免跨页访问         |
| DMA 缓冲区                   | 64/128 字节     | 适配外设的批量传输规则                             |

举个例子，假设存在如下结构体，那么它的对齐逻辑应该是：

1. `data` 自身是一个数组，他的内部按照二字节对齐；同时 `data` 的大小是 `14` 字节，会被填充 `2` 字节后扩充到 `16` 字节；
2. `big_number` 按照 `8` 字节对齐；
3. `small_number` 自身按照 `4` 字节对齐，但因结构体整体对齐值为 `8` 字节，最终会填充 `4` 字节让总大小满足 8 的倍数
4. 整个结构体的大小为 `32` 字节。

```rust
struct AlignedStruct {
    d: [u16; 7],
    big_number: u64,
    small_number: u32
}
```

### 对象在物理地址和虚拟地址中的转换

转换的核心其实在于：

当从Vec<&'static mut [u8]>读取数据到对象时：

1. 通过vec中包含的数据长度和对象的实际长度对比，判断vec中数据的合法性；
2. 通过VA得到 mut 指针，在这一步，CPU的MMU会帮我们做VA到PA的转换；
3. 判断内存是否对齐。

当反过来从对象写入到Vec<&'static mut [u8]>时，我们才需要分段式的处理它。

总结就是，由物理内存到虚拟内存不需要分段处理，因为我们使用的虚拟内存是连续的，CPU的MMU会帮我们做这个VA到PA的转换。而虚拟内存到物理内存需要分段处理，因为我们此时要写的物理内存是不连续的。

> `(tcb_ptr as usize) % align_of::<Self>() == 0` 这行代码是**强制校验 TCB 结构体的起始虚拟地址是否满足「内存对齐要求」**，是避免 CPU 访问错误、内存布局错乱的 “保命检查”。
>
> - 每个数据类型（尤其是结构体）都有「最小对齐值」（`align_of::<T>()` 返回的值），比如：
>   - `u8`：对齐值 1（任意地址都可）；
>   - `u64`/`usize`（64 位）：对齐值 8（地址必须是 8 的倍数）；
>   - 你的 `TCB` 结构体（`#[repr(C, align(4096))]`）：对齐值 4096（地址必须是 4KB 页大小的倍数）；
> - CPU 访问内存时，要求数据的起始地址必须是其对齐值的整数倍 —— 这是硬件设计决定的（为了提升访存效率，或避免部分架构直接报错）。

```rust
impl TaskControlBlock {
    /// 从分散的物理内存切片（Vec<&'static mut [u8]>）转换为TCB引用
    /// 前提：切片拼接后能覆盖整个TCB的内存，且虚拟地址连续
    pub unsafe fn from_phys_slices(slices: &mut Vec<&'static mut [u8]>) -> &'static mut Self {
        // 步骤1：校验长度和对齐
        let tcb_size = size_of::<Self>();
        let total_len: usize = slices.iter().map(|s| s.len()).sum();
        assert!(total_len >= tcb_size, "物理内存切片长度不足");
        assert!(align_of::<Self>() <= PAGE_SIZE, "TCB对齐要求超过页大小");

        // 步骤2：获取第一个切片的起始虚拟地址（作为TCB的基地址）
        // 内核中：所有物理页已映射为连续的内核虚拟地址
        let first_slice_ptr = slices[0].as_mut_ptr();
        assert!(!first_slice_ptr.is_null(), "空指针");

        // 步骤3：强转为TCB引用（关键：unsafe，需保证内存布局匹配）
        let tcb_ptr = first_slice_ptr as *mut Self;
        // 校验地址对齐（必做！否则会触发未定义行为）
        assert!(
            (tcb_ptr as usize) % align_of::<Self>() == 0,
            "TCB地址未对齐"
        );

        &mut *tcb_ptr
    }

    /// 把TCB转换为分散的物理内存切片（Vec<&'static mut [u8]>）
    pub unsafe fn to_phys_slices(&mut self) -> Vec<&'static mut [u8]> {
        let mut slices = Vec::new();
        let tcb_ptr = self as *mut Self;
        let tcb_size = size_of::<Self>();
        let tcb_base = tcb_ptr as usize;

        // 步骤1：遍历TCB占用的所有物理页
        for offset in (0..tcb_size).step_by(PAGE_SIZE) {
            let current_va = tcb_base + offset;
            // 步骤2：虚拟地址 → 物理页号（内核页表查询）
            let ppn = va_to_ppn(current_va); // 需实现：虚拟地址转物理页号
            // 步骤3：物理页号 → 字节切片
            let slice = phys_page_to_slice(ppn);
            slices.push(slice);
        }

        slices
    }
}
```

### memory_set中的trampoline和trap_cx

### children和parent

在 `PCB` 中，我们的 `children` 和 `parent` 的定义如下：

```rust
pub impl TaskControlBlockInner {
    /// Parent process of the current process.
    /// Weak will not affect the reference count of the parent
    pub parent: Option<Weak<TaskControlBlock>>,

    /// A vector containing TCBs of all child processes of the current process
    pub children: Vec<Arc<TaskControlBlock>>,
}
```

可以看到，我们的 `parent` 和 `children` 被分别定义为 `Arc` 和 `Weak`。首先，我们需要知道为什么需要这么设计：

- 父进程的 `children` 字段需要引用子进程的 TCB（保证子进程运行时不被销毁）；
- 子进程的 `parent` 字段需要引用父进程的 TCB（方便子进程查询父进程信息）。

所以，他们之间形成了一个引用依赖的关系，如果双方都用 `Arc` 来进行引用，那么会发生如下情况：

1. 父进程退出，但是由于子进程仍然持有他的引用。在这种情况下，只有等待子进程退出之后才能回收父进程的资源；
2. 子进程退出，但是由于父进程仍然持有他的引用。在这种情况下，只有等待父进程退出之后才能回收子进程的资源；

引入弱引用就是为了打破这个循环依赖关系，那问题在于，弱引用是如何在保证PCB的生命周期的呢？

1. 一个进程，他对他父进程的引用是**弱引用**，而对他的子进程是 **强引用**；
2. 当子进程先于父进程退出的时候，父进程正常的回收子进程的资源，这个逻辑很简单；
3. **当父进程先于子进程退出的时候，父进程会将自身所有的子进程的父进程设置为 `INITPROC`，并且将他们添加到`INITPROC`的子进程列表。这保证了在任意时刻，一个进程不会因为缺少强引用而被错误的回收**。

### elf解析

1. 解析 ELF；
2. 映射虚拟内存高256GiB；
   1. map tramploine；
   2. map TrapContext；
3. 映射虚拟内存低256GiB；
   1. map elf（包括 .text/.rodata/.data/.bss），这里值得注意的是，我们在映射的过程需要将elf中的数据复制到映射的目标虚拟内存；
   2. map user stack；
   3. map program break；
4. 返回用户地址集，`user_stack_top`，以及入口地址；`user_stack_top` 的作用很多，可以参考 [应用进程内存模型](#应用进程内存模型) 的解析。

```rust
impl MemorySet {
    /// Include sections in elf and trampoline and TrapContext and user stack,
    /// also returns user_sp_base and entry point.
    pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
        let mut memory_set = Self::new_bare();
        // map program headers of elf, with U flag
        let elf = xmas_elf::ElfFile::new(elf_data).unwrap();

        Self::map_higher_addr(&mut memory_set);
        let user_stack_top = Self::map_lower_addr(&mut memory_set, &elf);

        (
            memory_set,
            user_stack_top,
            elf.header.pt2.entry_point() as usize,
        )
    }
}
```

### 进程的内存模型

1. `KernelStack`
2. `KERNEL_SPACE`
3. `UserStack` 在老版本的代码下，的确存在一个 `UserStack`，但是在实现虚拟内存 之后，UserStack 已经被抽象为一段 MapArea，同时记录了 `user_stack_top` 作为 `sp`。

### 应用进程内存模型

#### base_size

> `base_size` **直接存储 “`user_stack` 高地址边界” 的地址常量**，这也是 `base_size` 这个名字的来源，他指向 `user_stack` 的高地址边界，因为栈是从高地址向低地址增长的。

在操作系统中，`user_stack` 是用户态程序的虚拟地址的一个指针，指向了用户栈。

用户的虚拟地址中，一般按照 .text -> .rodata -> .data -> .bss -> guard page -> user_stack 的顺序**从低到高递增**。 

假设  `user_stack`  的地址范围是 `[user_stack_bottom, user_stack_bottom + USER_STACK_SIZE)`，这里  user_stack_bottom + USER_STACK_SIZE 是用户进程的 `base_size`，那么用户程序的所有数据都只能出现在 base_size 之下。

- `user_stack_bottom`：栈的 **低地址边界**（栈的起始地址）；
- `user_stack_bottom + USER_STACK_SIZE`：栈的 **高地址边界**（也就是 `base_size`）；
- 栈的增长方向：**从高地址向低地址增长**（栈顶指针 `sp` 初始指向 `base_size`，每次压栈 `sp--`，出栈 `sp++`）。

![base_size](/images/20251203/base_size.png)

#### heap_bottom

`heap_bottom` 是用户堆（`heap`）的 **低地址边界**（堆从低地址向高地址增长，这个地址是堆的 “起始起点”）；初始值和 `base_size` 完全相同。

#### program_brk

`program_brk`  是堆的**当前高地址**（`brk` 系统调用可修改，用于扩展 / 收缩堆）；

#### 总结

>刚初始化时，`base_size`/`heap_bottom`/`program_brk` 都指向 **End of User Stack**

```mermaid
block-beta
    block:1
        columns 1
        space:2
        eUserStack("End of User Stack")
        sUserStack("Start of User Stack")
        eGuardPage("End of Guard Page")
        sGuardPage("Start of Guard Page")
        ebss("ebss")
        sbss("sbss")
        edata("edata")
        sdata("sdata")
        erodata("erodata")
        srodata("srodata")
        etext("etext")
        stext("stext")
    end

    block 
        columns 1
        space:2
        init("user_stack_top/base_size/heap_bottom/program_brk")
    end
    eUserStack --> init

    style eUserStack fill:#e6f7ff,stroke:#1890ff,stroke-width:2px,color:#0047ab,padding:8px
    style sUserStack fill:#e6f7ff,stroke:#1890ff,stroke-width:2px,color:#0047ab,padding:8px
    style eGuardPage fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style sGuardPage fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style ebss fill:#f0fff4,stroke:#52c41a,stroke-width:2px,color:#237804,padding:8px
    style sbss fill:#f0fff4,stroke:#52c41a,stroke-width:2px,color:#237804,padding:8px
    style edata fill:#fffbe6,stroke:#faad14,stroke-width:2px,color:#aa5800,padding:8px
    style sdata fill:#fffbe6,stroke:#faad14,stroke-width:2px,color:#aa5800,padding:8px
    style erodata fill:#f9f0ff,stroke:#722ed1,stroke-width:2px,color:#531dab,padding:8px
    style srodata fill:#f9f0ff,stroke:#722ed1,stroke-width:2px,color:#531dab,padding:8px
    style etext fill:#f5f5f5,stroke:#8c8c8c,stroke-width:2px,color:#333333,padding:8px
    style stext fill:#f5f5f5,stroke:#8c8c8c,stroke-width:2px,color:#333333,padding:8px
    style init fill:#fff7e6,stroke:#ff7a45,stroke-width:2px,color:#d4380d,padding:8px,font-weight:bold

```

> 第一次分配堆，我们需要增加 `program_brk`，为我们的堆分配空间，此时 `program_brk` 和其他两个参数分离了，这里 `EOUS` 代表 `End of User Stack`，`SOH` 代表 `Start Of Heap`

```mermaid
block-beta
    block:1
        columns 1
        space:2
        eHeap("End Of Heap")
        eUserStack("EOUS/SOH")
        sUserStack("Start of User Stack")
        eGuardPage("End of Guard Page")
        sGuardPage("Start of Guard Page")
        ebss("ebss")
        sbss("sbss")
        edata("edata")
        sdata("sdata")
        erodata("erodata")
        srodata("srodata")
        etext("etext")
        stext("stext")
    end

    block 
        columns 1
        space:2
        program_brk("program_brk")
        init("base_size/heap_bottom")
    end

    eHeap --> program_brk
    eUserStack --> init

    style eUserStack fill:#e6f7ff,stroke:#1890ff,stroke-width:2px,color:#0047ab,padding:8px
    style sUserStack fill:#e6f7ff,stroke:#1890ff,stroke-width:2px,color:#0047ab,padding:8px
    style eGuardPage fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style sGuardPage fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style ebss fill:#f0fff4,stroke:#52c41a,stroke-width:2px,color:#237804,padding:8px
    style sbss fill:#f0fff4,stroke:#52c41a,stroke-width:2px,color:#237804,padding:8px
    style edata fill:#fffbe6,stroke:#faad14,stroke-width:2px,color:#aa5800,padding:8px
    style sdata fill:#fffbe6,stroke:#faad14,stroke-width:2px,color:#aa5800,padding:8px
    style erodata fill:#f9f0ff,stroke:#722ed1,stroke-width:2px,color:#531dab,padding:8px
    style srodata fill:#f9f0ff,stroke:#722ed1,stroke-width:2px,color:#531dab,padding:8px
    style etext fill:#f5f5f5,stroke:#8c8c8c,stroke-width:2px,color:#333333,padding:8px
    style stext fill:#f5f5f5,stroke:#8c8c8c,stroke-width:2px,color:#333333,padding:8px
    style init fill:#fff7e6,stroke:#ff7a45,stroke-width:2px,color:#d4380d,padding:8px,font-weight:bold

```

> 既然 `base_size` 和 `heap_bottom` 一直是一样的，为什么我们不直接合并这两个参数呢？因为我们可能在后续的扩展中会分离这两个变量。例如，我们可能希望在 `Heap` 和 `Stack` 之间插入一个 `Guard Page`。

```mermaid
block-beta
    block:1
        columns 1
        space:2
        eHeap("End Of Heap")
        sHeap("Start Of Heap")
        eGuardPage1("End of Guard Page")
        sGuardPage1("Start of Guard Page")
        eUserStack("End of User Stack")
        sUserStack("Start of User Stack")
        eGuardPage("End of Guard Page")
        sGuardPage("Start of Guard Page")
        ebss("ebss")
        sbss("sbss")
        edata("edata")
        sdata("sdata")
        erodata("erodata")
        srodata("srodata")
        etext("etext")
        stext("stext")
    end

    block 
        columns 1
        space:2
        program_brk("program_brk")
        heap_bottom("heap_bottom")
        space:2
        init("base_size")
        space:11
    end

    eHeap --> program_brk
    sHeap --> heap_bottom
    eUserStack --> init

    style eUserStack fill:#e6f7ff,stroke:#1890ff,stroke-width:2px,color:#0047ab,padding:8px
    style sUserStack fill:#e6f7ff,stroke:#1890ff,stroke-width:2px,color:#0047ab,padding:8px
    style eGuardPage fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style eGuardPage1 fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style sGuardPage fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style sGuardPage1 fill:#fff2f0,stroke:#ff4d4f,stroke-width:2px,color:#c5001a,padding:8px
    style ebss fill:#f0fff4,stroke:#52c41a,stroke-width:2px,color:#237804,padding:8px
    style sbss fill:#f0fff4,stroke:#52c41a,stroke-width:2px,color:#237804,padding:8px
    style edata fill:#fffbe6,stroke:#faad14,stroke-width:2px,color:#aa5800,padding:8px
    style sdata fill:#fffbe6,stroke:#faad14,stroke-width:2px,color:#aa5800,padding:8px
    style erodata fill:#f9f0ff,stroke:#722ed1,stroke-width:2px,color:#531dab,padding:8px
    style srodata fill:#f9f0ff,stroke:#722ed1,stroke-width:2px,color:#531dab,padding:8px
    style etext fill:#f5f5f5,stroke:#8c8c8c,stroke-width:2px,color:#333333,padding:8px
    style stext fill:#f5f5f5,stroke:#8c8c8c,stroke-width:2px,color:#333333,padding:8px
    style init fill:#fff7e6,stroke:#ff7a45,stroke-width:2px,color:#d4380d,padding:8px,font-weight:bold

```





### 内核堆

> 关于内核堆的分配可以参考之前的解析：[堆的初始化](https://0x822a5b87.github.io/2025/11/21/uCore%EF%BC%9A%E4%B8%80%E4%B8%AA%E5%9F%BA%E4%BA%8ERust%E7%9A%84%E7%AE%80%E5%8D%95%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%AE%9E%E7%8E%B0%EF%BC%884%EF%BC%89/#%E5%A0%86%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96)

### link_app.S

1. `link_app.S` 由 `build.rs` 生成；
2. 在 `main.rs` 中由 `global_asm!(include_str!("link_app.S"));` 引入后被加载到内核空间中的 `.text` 段；
3. `MemorySet#new_kernel`  中，将 `.text` 段通过 `MapType::Identical` 加载到内核内存空间，此后我们可以使用虚拟地址来访问这个地址了 -- 只不过因为是恒等映射，所以 VA = PA。

### exec

#### 寻址逻辑

> 具体 `app_name` 的生成请参考 [基于应用名的应用链接/加载器](#基于应用名的应用链接/加载器)。

1. `load.rs` 下的 `APP_NAMES` 会加载 `link_app.S` 中的全部 `_app_names`
2. 通过 `APP_NAMES` 可以查找到对应的 `index`；
3. 通过 `index` 可以在 `_num_app` 下找到代码段的 `start` 和 `end`；
4. 通过 `start` 和 `end` 直接从 `.text` 段读取到数据。

#### 执行逻辑

`exec` 指令的调用如下：

```rust
exec("ch5b_user_shell\0", &[0 as *const u8]);
```

输入必须以 `'\0'` 结尾，这是因为 `exec` 函数的参数是 `&str`，**Rust 的 `&str` 类型本身不会在字符串末尾自动添加 `\0`（null 终止符）**：

```rust
pub fn exec(path: &str, args: &[*const u8]) -> isize {
    sys_exec(path, args)
}

pub fn sys_exec(path: &str, args: &[*const u8]) -> isize {
    syscall(
        SYSCALL_EXEC,
        [path.as_ptr() as usize, args.as_ptr() as usize, 0],
    )
}
```

而 `sys_exec` 中，将 `path.as_ptr()` 转换为一个字符串指针并以该指针作为参数传递给kernel，内核收到数据后，如果没有 `'\0'` 是无法正确结束循环的。

```rust
/// Translate&Copy a ptr[u8] array end with `\0` to a `String` Vec through page table
pub fn translated_str(token: usize, ptr: *const u8) -> String {
    let page_table = PageTable::from_token(token);
    let mut string = String::new();
    let mut va = ptr as usize;
    loop {
        let ch: u8 = *(page_table
            .translate_va(VirtAddr::from(va))
            .unwrap()
            .get_mut());
        // 读到 '\0' 退出
        if ch == 0 {
            break;
        } else {
            string.push(ch as char);
            va += 1;
        }
    }
    string
}
```

### 进程的资源回收

> 本描述仅针对于 `linux` 

需要先了解的一点是，`linux` 下的 `waitpid` 和 `rCore` 下的资源回收存在区别：

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *status);

pid_t waitpid(pid_t pid, int *status, int options);

int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

> The **wait**() system call `suspends` execution of the calling process until one of its children terminates. The call *wait(&status)* is equivalent to:
>
> ```c
> waitpid(-1, &status, 0);
> ```
>
> The **waitpid**() system call suspends execution of the calling process until a child specified by *pid* argument has changed state. By default, **waitpid**() waits only for terminated children, but this behavior is modifiable via the *options* argument, as described below.
>
> - The value of *pid* can be:
>   - `< -1` meaning wait for any child process whose process group ID is equal to the absolute value of *pid*.
>   - `-1` meaning wait for any child process.
>   - `0` meaning wait for any child process whose process group ID is equal to that of the calling process.
>   - `> 0` meaning wait for the child whose process ID is equal to the value of *pid*.

主要区别有两点：

1. `linux` 下的 waitpid 是阻塞式的；
2. 它的返回值除了可能是 `pid` 或者其他状态之外，还有可能是 `pid_group`。

在 `os` 中，父进程在创建子进程时，内核会为子进程分配一些运行必要的信息 - `pid`，`TCB`（进程控制块）等，父进程必须通过 `wait` 来查询子进程状态，这是因为 “父进程是否需要子进程的这些信息（如退出码、终止原因）” 这个条件内核是不知道的。对于内核来讲，它需要负责把子进程的状态变化消息交给父进程。

这个过程可能出现几种不同的情况：

1. 父进程正常调用 `wait`/`waitpid`，拿到退出子进程的状态信息（如退出码），此时内核会彻底回收子进程的 `pid`、`TCB`（仅保留的关键字段）等资源，子进程正常退出；
2. 在子进程退出前，父进程未调用 `wait` 就提前退出（因为异常或代码 BUG），此时子进程的状态变为 `Orphan（孤儿进程）`，内核维护的 `initproc` 进程（PID=1）会负责收养这些孤儿进程，待其终止后回收相关资源 —— 因为父进程已退出，内核明确知道子进程的状态信息无需再交给父进程，可由 init 进程统一回收；
3. 在子进程退出后，父进程未调用 `wait`，此时子进程的状态变为 `Zombie（僵尸进程）`。内核仅保留 `TCB` 中的关键字段（如 PID、退出码），不会回收这些信息 —— 因为内核无法判断父进程是 “代码中未调用 `wait`” 还是 “已调用但未执行到 `wait` 指令”，只能一直保留，直到父进程调用 `wait`（主动回收）或父进程退出（子进程变孤儿，由 init 回收）。

## 代码统计

```reStructuredText
➜  2025a-rcore-0x822a5b87 git:(ch5) ✗ cloc os
     264 text files.
     137 unique files.
     435 files ignored.

github.com/AlDanial/cloc v 2.06  T=0.06 s (2231.6 files/s, 70042.7 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Rust                            32            232            493           2335
D                               39            108              0            598
Assembly                         4             24             26            265
make                             1             20              7             59
JSON                            58              0              0             58
Linker Script                    1              7              0             46
TOML                             2              3              1             18
-------------------------------------------------------------------------------
SUM:                           137            394            527           3379
-------------------------------------------------------------------------------
```

## 代码树

本章中主要新增的几个模块为：

1. `fs.rs` 新增了 `sys_read`；
2. `process.rs` 新增了 `sys_getpid`/`fork`/`exec`/`waitpid`；
3. `manager.rs` 新增任务管理器；
4. `pid.rs` 新增标志符和内核栈的rust抽象；
5. `processor.rs` 新增处理器管理结构 `Processor`

```
├── os
   ├── build.rs(修改：基于应用名的应用构建器)
   ├── ...
   └── src
       ├── ...
       ├── loader.rs(修改：基于应用名的应用加载器)
       ├── main.rs(修改)
       ├── mm(修改：为了支持本章的系统调用对此模块做若干增强)
       │   ├── address.rs
       │   ├── frame_allocator.rs
       │   ├── heap_allocator.rs
       │   ├── memory_set.rs
       │   ├── mod.rs
       │   └── page_table.rs
       ├── syscall
       │   ├── fs.rs(修改：新增 sys_read)
       │   ├── mod.rs(修改：新的系统调用的分发处理)
       │   └── process.rs（修改：新增 sys_getpid/fork/exec/waitpid）
       ├── task
       │   ├── context.rs
       │   ├── manager.rs(新增：任务管理器，为上一章任务管理器功能的一部分)
       │   ├── mod.rs(修改：调整原来的接口实现以支持进程)
       │   ├── pid.rs(新增：进程标识符和内核栈的 Rust 抽象)
       │   ├── processor.rs(新增：处理器管理结构 ``Processor`` ，为上一章任务管理器功能的一部分)
       │   ├── switch.rs
       │   ├── switch.S
       │   └── task.rs(修改：支持进程机制的任务控制块)
       └── trap
           ├── context.rs
           ├── mod.rs(修改：对于系统调用的实现进行修改以支持进程系统调用)
           └── trap.S
```





