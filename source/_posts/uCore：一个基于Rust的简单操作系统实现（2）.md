---
title: uCore：一个基于Rust的简单操作系统实现（2）
date: 2025-11-13 11:35:28
tags:
  - os
  - rust
---

# 批处理系统

>本文描述了第一个简单的批处理系统的实现。代码量约 `600` 行。
>全部的源代码位于
>
>- [rCore kernel - ch2](https://github.com/LearningOS/2025a-rcore-0x822a5b87/tree/ch2)
>- [rCore test](https://github.com/0x822a5b87/rCore-Tutorial-Test)

```
➜  ~/code/2025a-rcore-0x822a5b87 git:(ch2) ✗ cloc --include-ext=rs,s,S,asm os        
     191 text files.
     150 unique files.                                          
     184 files ignored.

1 error:
Unable to read:  os/.gdb_history

github.com/AlDanial/cloc v 1.82  T=0.02 s (1154.8 files/s, 54212.4 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Rust                            15             77            114            515
Assembly                         3             10             16            113
-------------------------------------------------------------------------------
SUM:                            18             87            130            628
-------------------------------------------------------------------------------
```

## 综述

在本章节中，我们实现一个简单的**批处理系统**，内核在启动之后，将多个程序打包输入并按顺序执行，而本章节我们会实现以下几个重要逻辑：

1. 通过加载器在启动时动态的加载代码到并执行；
2. 实现错误处理，保证在单个程序出错时不影响其他的程序。这个位置我们会引入一些新的概念，例如 `privilege`。同时我们也可以看到操作系统和普通的应用程序的区别，例如：操作系统有自己的内核栈，这个栈是完全独立于任何程序的。
3. 操作系统通过trap机制来处理 `interruption` 和 `exception` 事件。

### `rCore` 和 `rCore-test` 是怎么组织代码的？

在 [rCore-test](https://github.com/0x822a5b87/rCore-Tutorial-Test) 这个项目中，构建系统主要包含两个文件分别是 [build.py](https://github.com/0x822a5b87/rCore-Tutorial-Test/blob/main/build.py) 和 [linker.ld](https://github.com/0x822a5b87/rCore-Tutorial-Test/blob/main/src/linker.ld)。

仔细观察我们会发现一个奇怪的问题：

1. build.py 中 通过 `cargo rustc --bin %s %s -- -Clink-args=-Ttext=%x` 这个指令，将build/app下的所有文件编译成了可执行文件。并且以 `0x80400000` 作为基址，每个app的大小是 `0x20000`；
2. 而在 linker.ld 中，我们的 BASE_ADDRESS = `0x0` 。

要想要解释这个原因，我们首先必须理清项目的编译和执行逻辑。在rCore的开发中，总共可以分为三个不同的模块：

1. rCore 的 [kernel](https://github.com/LearningOS/2025a-rcore-0x822a5b87/tree/ch2)
2. [rCore-test的用户态程序](https://github.com/0x822a5b87/rCore-Tutorial-Test/tree/main/src/bin)，也就是我们的 `rCore-Tutorial-Test` 中 `src/bin` 部分；
3. [rCore-Tutorial-Test的用户态程序依赖的库](https://github.com/0x822a5b87/rCore-Tutorial-Test/tree/main/src)，也就是我们的 `rCore-Tutorial-Test` 中 `src` 除了 bin 的部分；

下面我们将简称这三个模块分别为：`kernel`, `user`, `lib`。

1. 编译阶段
    1. `lib` 先被编译为静态库，因其 linker.ld 基址为 0，代码中使用相对跳转访问，不依赖具体加载地址。
    2. `user` 程序编译时，通过 cargo rustc 链接 lib，并通过 --link-args=-Ttext=0x80400000 指定最终加载基址，将 lib 的相对地址代码重定位到该基址，生成独立的 ELF 二进制文件（.bin）。
2. 执行阶段：
    1. 内核通过 `build.rs` 将用户程序的 .bin 文件嵌入到内核镜像中（或在运行时读取），确保启动时可访问。
    2. 内核的 load_app() 函数（如 batch.rs 中）将用户程序二进制数据加载到内存的 0x80400000 及后续地址（与编译时指定的基址一致）。
    3. 内核通过上下文切换将 CPU 权限切换到用户态，并跳转到用户程序的入口地址（0x80400000 附近的 _start 函数，定义在 lib/src/lib.rs 中），开始执行用户程序。

>观察上面的结论，我们发现，当 kernel 加载 user 的代码时，由于user 程序编译时的基址（0x80400000）必须与内核加载时的目标地址一致，否则会因地址错位导致程序崩溃（例如指令或数据访问到错误内存)。
>
>也就是说，每个app都必须加载在 `build.py` 中使用 `cargo rustc` 编译时声明的位置。
>
>而此时，user 代码和 kernel 是共用同一片虚拟内存空间的，内核需要提供一些机制防止 `user` 去访问没有权限的内存！
>
>这个错误也非常好模拟，只需要修改 `batch.rs` 中 `APP_BASE_ADDRESS` 到 0x80200000 即可，这是kernel的代码段，复制会覆盖这段内存导致内核异常。 


## rCore-test

`rCore-test` 包含了项目的测试文件，我们前面提到的 “**批处理系统将多个程序打包输入并按顺序执行**”，那些被打包执行的程序就源自于该仓库中。

整体编译逻辑如下：

1. 根据参数BASE,TEST,CHAPTER生成一个包含所有需要编译文件的列表；
2. pre 执行会将所有的目标文件复制到 `app` 目录下；
3. binary 编译源文件： 
   - 编译源文件得到elf文件放在`$(TARGET_DIR)`目录下；
   - 通过 rust-objcopy 得到二进制文件放在 `$(TARGET_DIR)` 目录下；
4. build 将 `.bin` 和 `.elf` 复制到 `build` 对应的目录下；

```mermaid
---
title: Compiling rCore-test
---
flowchart TB

subgraph desc
    direction TB
    build_system("构建系统"):::purple
    dir("文件夹"):::green
    products("源文件/构建产物"):::animate
end


subgraph 构建过程
    direction LR
    makefile("Makefile"):::purple
    build("build.py"):::purple

    target("target/riscv64gc-unknown-none-elf/release"):::green
    build_dir("build"):::green

    source_code("*.rs"):::animate
    bin("*.bin"):::animate
    elf("*.elf"):::animate
    object("*.object"):::animate


    source_code -->|输入| build -->|编译| object -.->|输出到文件夹| target
    object -->|输入| makefile -->|rust-objcopy| bin -.->|输出到文件夹| target
    object -->|输入| makefile -->|cp| elf -.->|输出到文件夹| target

    bin -->|cp| build_dir
    elf -->|cp| build_dir
end

desc --> 构建过程

classDef pink 0,fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
classDef green fill: #695,color: #fff,font-weight: bold;
classDef purple fill:#968,stroke:#333, font-weight: bold;
classDef error fill:#bbf,stroke:#f65,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
classDef coral fill:#f8f,stroke:#333,stroke-width:4px;
classDef animate stroke-dasharray: 8,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;
```

### makefile

```makefile
# define variables
TARGET := riscv64gc-unknown-none-elf
MODE := release
APP_DIR := src/bin
TARGET_DIR := target/$(TARGET)/$(MODE)
BUILD_DIR := build
OBJDUMP := rust-objdump --arch-name=riscv64
OBJCOPY := rust-objcopy --binary-architecture=riscv64
PY := python3

ifeq ($(MODE), release)
	MODE_ARG := --release
endif

BASE ?= 0
CHAPTER ?= 0
TEST ?= $(CHAPTER)

# Generate a list contains all target files.
# $(TEST) : indicate which tests should run, 0 stands for none, 1 stands for all,
#			and other values stand for tests based on BASE and TEST
# $(BASE) : Indicate which types of test should run, 0 stands for normal, 1 stands for basic,
#			and other values stand for basic and normal.
# $(CHAPTER) : set default values for TEST
ifeq ($(TEST), 0) # No test, deprecated, previously used in v3
	APPS :=  $(filter-out $(wildcard $(APP_DIR)/ch*.rs), $(wildcard $(APP_DIR)/*.rs))
else ifeq ($(TEST), 1) # All test
	APPS :=  $(wildcard $(APP_DIR)/ch*.rs)
else
	# generate a sequential number between BASE and TEST, with both BASE and TEST included.
	TESTS := $(shell seq $(BASE) $(TEST))
	ifeq ($(BASE), 0) # Normal tests only
		APPS := $(foreach T, $(TESTS), $(wildcard $(APP_DIR)/ch$(T)_*.rs))
	else ifeq ($(BASE), 1) # Basic tests only
		APPS := $(foreach T, $(TESTS), $(wildcard $(APP_DIR)/ch$(T)b_*.rs))
	else # Basic and normal
		APPS := $(foreach T, $(TESTS), $(wildcard $(APP_DIR)/ch$(T)*.rs))
	endif
endif

# Use pattern substitution to generate a target directory list by replacing APP_DIR with TARGET_DIR.
# For example : src/bin/hello_world.rs will become target/riscv64gc-unknown-none-elf/release/hello_world
ELFS := $(patsubst $(APP_DIR)/%.rs, $(TARGET_DIR)/%, $(APPS))

# Generate binary file according to $CHAPTER.
# 1. Generate elf file by cargo or build.py;
# 2. Transform elf file to binary file by rust-objcopy, and add .bin as suffix for elf file name as the name of binary file;
# 3. Copy elf file to $BUILD_DIR/elf.
binary:
	@echo $(ELFS)
	@if [ ${CHAPTER} -gt 3 ]; then \
		cargo build $(MODE_ARG) ;\
	else \
		CHAPTER=$(CHAPTER) python3 build.py ;\
	fi
	@$(foreach elf, $(ELFS), \
		$(OBJCOPY) $(elf) --strip-all -O binary $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.bin, $(elf)); \
		cp $(elf) $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.elf, $(elf));)

disasm:
	@$(foreach elf, $(ELFS), \
		$(OBJDUMP) $(elf) -S > $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.asm, $(elf));)
	@$(foreach t, $(ELFS), cp $(t).asm $(BUILD_DIR)/asm/;)

# prepare context and copy all target files
pre:
	@mkdir -p $(BUILD_DIR)/bin/
	@mkdir -p $(BUILD_DIR)/elf/
	@mkdir -p $(BUILD_DIR)/app/
	@mkdir -p $(BUILD_DIR)/asm/
	@$(foreach t, $(APPS), cp $(t) $(BUILD_DIR)/app/;)

build: clean pre binary
	@$(foreach t, $(ELFS), cp $(t).bin $(BUILD_DIR)/bin/;)
	@$(foreach t, $(ELFS), cp $(t).elf $(BUILD_DIR)/elf/;)
```

### build.py

`build.py` 的逻辑比较简单，就是将我们在 `make pre` 中复制到 `build/app` 目录下的所有 `*.rs` 源文件编译到 `可执行文件`（这也是为什么 `bin/*.rs` 的每个源文件中都有main函数的原因），以下是一些值得注意的细节：

1. 代码的基址是 `0x80400000`，在 `rCore-kernel` 中，我们也必须使用相同的地址来处理这个地址；
2. 每段代码我们使用了 `0x20000` 这么多空间，理论上来说这个是有问题的，因为每段程序的大小不一样，这有可能导致内存碎片（对于不足0x20000的程序），或者内存溢出等问题（对于超过0x20000的程序）。

```python
import os

base_address = 0x80400000
step = 0x20000
linker = "src/linker.ld"

for app in apps:
    app = app[: app.find(".")]
    #
    os.system(
        # generate object files by compiling source codes.
        #
        # cargo rustc --bin hello_world --release -- -Clink-args=Ttext=0x80400000
        #
        #       --bin [<NAME>]      Build only the specified binary
        #       --release           Use release mode, which enables -O3 optimization, etc.
        #       --                  Separator indicating the subsequent parameters are passed to `rustc`
        #       -C, --codegen <OPT>[=<VALUE>] Set a codegen option
        #
        #           link-args specifies that it's an argument for linker.
        #           -Ttext=0x80400000 T=template, text=.text section. It explicitly specifies the load address of the code.
        "cargo rustc --bin %s %s -- -Clink-args=-Ttext=%x"
        % (app, mode_arg, base_address + step * app_id)
    )
    print(
        "[build.py] application %s start with address %s"
        % (app, hex(base_address + step * app_id))
    )
    if chapter == '3':
        app_id = app_id + 1
```

### linker.ld

`linker.ld` 的逻辑和 `rCore-kernel` 的逻辑基本没有什么区别，就是老生常谈的指定 `.text`, `.bss`, `.data` 等区域的初始地址，唯一值得注意的是，他这里的基址指定为 `0x0`。



```
OUTPUT_ARCH(riscv)
ENTRY(_start)

BASE_ADDRESS = 0x0;
```

## rCore-kernel

## QA

### 代码树

```terminaloutput
── os
│   ├── Cargo.toml
│   ├── Makefile (修改：构建内核之前先构建应用)
│   ├── build.rs (新增：生成 link_app.S 将应用作为一个数据段链接到内核)
│   └── src
│       ├── batch.rs(新增：实现了一个简单的批处理系统)
│       ├── console.rs
│       ├── entry.asm
│       ├── lang_items.rs
│       ├── link_app.S(构建产物，由 os/build.rs 输出)
│       ├── linker.ld
│       ├── logging.rs
│       ├── main.rs(修改：主函数中需要初始化 Trap 处理并加载和执行应用)
│       ├── sbi.rs
│       ├── sync(新增：包装了RefCell，暂时不用关心)
│       │   ├── mod.rs
│       │   └── up.rs
│       ├── syscall(新增：系统调用子模块 syscall)
│       │   ├── fs.rs(包含文件 I/O 相关的 syscall)
│       │   ├── mod.rs(提供 syscall 方法根据 syscall ID 进行分发处理)
│       │   └── process.rs(包含任务处理相关的 syscall)
│       └── trap(新增：Trap 相关子模块 trap)
│           ├── context.rs(包含 Trap 上下文 TrapContext)
│           ├── mod.rs(包含 Trap 处理入口 trap_handler)
│           └── trap.S(包含 Trap 上下文保存与恢复的汇编代码)
└── user(新增：应用测例保存在 user 目录下)
   ├── Cargo.toml
   ├── Makefile
   └── src
      ├── bin(基于用户库 user_lib 开发的应用，每个应用放在一个源文件中)
      │   ├── ...
      ├── console.rs
      ├── lang_items.rs
      ├── lib.rs(用户库 user_lib)
      ├── linker.ld(应用的链接脚本)
      └── syscall.rs(包含 syscall 方法生成实际用于系统调用的汇编指令，
                     各个具体的 syscall 都是通过 syscall 来实现的)
```

### 如何处理GIT目录安全检查机制引发的编译异常

这里rCore-code和rCore-test是通过两个独立的 git clone引入的，部分情况下（例如通过docker启动并且通过mount关联宿主机的文件地址）可能会触发git的“目录安全检查” 机制导致编译失败，一般异常会如下所示：

```
error: failed to run custom build command for `os v0.1.0 (/mnt/os)`

Caused by:
  process didn't exit successfully: `/mnt/os/target/debug/build/os-bb0e74f81fa807fa/build-script-build` (exit status: 101)
  --- stdout
  cargo:rerun-if-changed=../user/src/
  cargo:rerun-if-changed=../user/build/bin/

  --- stderr
  thread 'main' panicked at build.rs:42:44:
  attempt to subtract with overflow
  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
warning: build failed, waiting for other jobs to finish...
```

这个是因为git的目录安全检查机制，可能间接影响目录的文件访问权限（或脚本的执行上下文），导致 build.rs 无法读取 ../user/build/bin/ 等依赖目录的文件（比如文件大小被误判为 0），进而引发计算溢出。

我们把新的项目加入safe.directory即可：

```bash
git config --global --add safe.directory /mnt
```

不过一般来说，对于这种嵌套依赖的项目，我们使用 `submodule` 可能是一个更好的选择：

```bash
git submodule add https://github.com/LearningOS/rCore-Tutorial-Test.git user
```

### riscv特权级切换

在现代计算机体系下，如果我们把代码的执行看成一个线性的（不考虑多核CPU和乱序指令等）指令执行，那么我们会发现有一些事件我们没有办法处理，例如：

1. 在程序执行的过程中计算机找到了一个新的设备，那我必须要等待当前的这些程序全部执行完才能识别到新设备；
2. 计算机在访问内存的过程中，发现页表中的缓存已经失效，或者发生了缺页异常；
3. 我在某个线程中设置了一个定时器，此时定时器设定的时间到了，我需要去执行定时器的回调函数；
4. 用户需要跳转到更高级的特权等级以便于执行某些需要更高权限的操作：例如处于用户态时的系统调用。

这些事件都是不可预测的，而为了处理这些事件，我们需要有一个方式来通知计算机，也就是 `trap`，而我们根据这些事件的类型，我们将他分为了两个大分类：`interruption` 和 `exception`。

```mermaid
flowchart LR

subgraph program
    instruction1("指令1"):::green --> instruction2("指令2"):::green --> instruction3("指令3"):::green -.-> instruction4("..."):::green
end

subgraph 处理
    instruction3 --> handler("处理异常/中断"):::pink --> instruction4
end

exception("exception or interruption"):::error -.->|通知程序有新的事件需要处理| instruction3

classDef pink 0,fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
classDef green fill: #695,color: #fff,font-weight: bold;
classDef purple fill:#968,stroke:#333, font-weight: bold;
classDef error fill:#bbf,stroke:#f65,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
classDef coral fill:#f8f,stroke:#333,stroke-width:4px;
classDef animate stroke-dasharray: 8,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;
```

#### 异常处理相关的寄存器

- Trap触发与入口配置
  - `stvec` Supervisor Trap Vector Base Address Register
  - `scause` Supervisor Cause Register
  - `sepc` Supervisor Exception Program Counter
  - `sstatus` Supervisor Status Register
- 数据传递与地址记录 CSR
    - `stval` Supervisor Trap Value
    - `sscratch` Supervisor Scratch
- 中断控制相关 CSR（扩展必备）
  - `sie` Supervisor Interrupt Enable Register
  - `sip` Supervisor Interrupt Pending Register

#### 异常处理

>这里我们先不关注中断相关的CSR，因为本章节主要是关注特权优先级的转换。

| CSR      | 功能                                                            |
|----------|---------------------------------------------------------------|
| stvec    | 控制 trap 处理代码的入口地址                                             |
| scause   | 描述 trap 的原因                                                   |
| sepc     | 当 trap 是第一个异常的时候，记录 trap 发生之前的 pc                             |
| sstatus  | SPP 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息                       |
| stval    | Trap 值寄存器：保存与 Trap 相关的补充数据（因 Trap 类型而异）                       |
| sscratch | S 特权级临时 scratch 寄存器：Trap 触发后，内核可临时存储数据（如内核栈指针 sp），避免覆盖用户态寄存器。 |

在这里，我们先抛开寄存器，考虑一下异常触发时整体的流程：

1. 用户态触发trap，通知内核处理trap；
2. 内核trap处理，找到trap的对应处理逻辑，并执行该逻辑；
3. 内核还原现场到触发trap之前，并将控制权交还给用户态程序；

那他整体的流程相当于是可以看做如下伪代码执行：

1. `work()` 触发异常，跳转到异常处理逻辑；
2. `handle_exception()` 处理异常；
3. 跳转回用户态，继续执行 `after_handle_exception()`；

```java
public void run() {
    try {
        work();
        after_handle_exception();
    } catch (Exception1 e) {
        handle_exception_1();
    } catch (Exception2 e) {
        handle_exception_2();
    }
}
```

而在这个程序流转的过程中，我们需要很多信息，例如：

1. 触发的是 exception1 还是 exception2？这样我们才能知道使用哪个异常处理函数；
2. 异常处理完了，我应该跳回到哪里？work还是after_handle_exception？

>当我们把这整个过程扩展开来的时候，我们就可以清楚的了解到，这些寄存器就是连接用户态和内核态的桥梁。

```mermaid
---
title: 一次ecall引发的trap
---
flowchart TB

    subgraph registers
        stvec("stvec"):::pink
        scause("scause"):::pink
        sepc("sepc"):::pink
        sstatus("sstatus"):::pink
        stval("stval"):::pink
        sscratch("sscratch"):::pink
        a0("a0 ~ a7"):::pink
    end

    用户态触发trap("用户态触发trap"):::green
    内核trap处理("内核trap处理"):::error
    返回用户态("返回用户态"):::green


    TrapContext("TrapContext"):::purple

    用户态触发trap -->|保存当前特权级| sstatus
    用户态触发trap -->|保存当前指令地址| sepc
    用户态触发trap -->|设置trap原因| scause
    用户态触发trap -->|设置系统调用参数| a0

    scause -.->|确认trap原因| 内核trap处理
    sepc -.->|记录返回地址| 内核trap处理 -->|记录sepc| TrapContext
    a0 -.->|获取系统调用参数并执行| 内核trap处理

    sepc -->|读取并写入TrapContext| 返回用户态 -->|sepc+4| TrapContext
    sstatus -->|读取并写入TrapContext| 返回用户态 -->|sstatus| TrapContext

    返回用户态 -->|从 sepc 恢复执行，回到用户态| 用户态("用户态")

classDef pink 0,fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
classDef green fill: #695,color: #fff,font-weight: bold;
classDef purple fill:#968,stroke:#333, font-weight: bold;
classDef error fill:#bbf,stroke:#f65,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
classDef coral fill:#f8f,stroke:#333,stroke-width:4px;
classDef animate stroke-dasharray: 8,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;
```

```mermaid
---
title: 跳转
---

flowchart TB


    stvec("stvec"):::pink
    scause("scause"):::pink
    sstatus("sstatus"):::pink
    stval("stval"):::pink



    用户态触发trap("用户态触发trap"):::green
    内核trap处理("内核trap处理"):::error

    stvec -->|1. 读取stvec| 用户态触发trap --> TrapHandler("TrapHandler"):::error

    scause -->|2. 读取trap原因-缺页？系统调用？| 内核trap处理
    sstatus -->|3. 原特权等级-自陷？| 内核trap处理

    stval -->|4. 根据scause解析stval并解析执行| 内核trap处理

classDef pink 0, fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
classDef green fill: #695,color: #fff,font-weight: bold;
classDef purple fill:#968,stroke:#333, font-weight: bold;
classDef error fill:#bbf,stroke:#f65,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
classDef coral fill:#f8f,stroke:#333,stroke-width:4px;
classDef animate stroke-dasharray: 8,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;

```

### rust是如何组织依赖的

>rust中：
>1. 每个项目被称之为crate，crate 可以引用其他人发布的crate，由 `lib.rs` 来管理；
>2. crate下可能包含模块目录（Module Directory），模块目录由 `mod.rs` 来管理；
>3. 模块目录下的每个文件一定是一个单独的mod；
>4. 文件内部可以声明 `内嵌mod（Nested Module）`。

一个典型的rust项目的结构如下所示：

```
dev_os/                   # 项目根目录
├── src/                  # 源代码目录（Rust 约定 src/ 下是 crate 内容）
│   ├── main.rs           # 二进制 crate 入口（顶层 crate 的入口）
│   ├── os/               # 模块目录（Module Directory）
│   │   ├── mod.rs        # 模块目录入口：管理 os/ 下的文件模块
│   │   ├── console.rs    # 文件模块：mod console
│   │   │   └─ 内部可声明：mod inner_console { ... } （内嵌子 mod）
│   │   ├── process/      # 嵌套模块目录（os/ 下的子模块目录）
│   │   │   ├── mod.rs    # 嵌套模块目录入口
│   │   │   └── scheduler.rs  # 文件模块：process::scheduler
│   │   └── sbi.rs        # 文件模块：mod sbi
│   └── lib.rs            # （可选）库 crate 入口（若需对外提供库功能）
└── user/                 # 另一个模块目录（用户程序相关）
    ├── mod.rs
    └── hello_world.rs
```

### rust中`lib.rs`和`mod.rs`的区别是什么？

| 特征	   | lib.rs	                                 | mod.rs                        |
|-------|-----------------------------------------|-------------------------------|
| 核心定位	 | 整个 crate 的「库目标入口文件」	                    | 某个子模块的「目录入口文件」                |
| 适用范围  | 	仅用于 src/ 根目录下	                         | 仅用于「子模块目录」下（如 src/foo/mod.rs） |
| 默认作用  | 	自动成为 crate 根模块（crate::）	               | 自动成为该目录对应的模块（crate::foo::）    |
| 关联目标  | 	对应 Cargo.toml 中的 [lib] 目标	             | 对应某个嵌套子模块（手动通过 mod foo; 引入）   |
| 是否必需  | 	库目标（library）默认必需（可通过 Cargo.toml 配置修改）	 | 子模块目录默认必需（否则无法识别该目录为模块）       |

例如，在我们的 `/user` 项目下，它的整体目录结构如下：

```terminaloutput
──── user(新增：应用测例保存在 user 目录下)
   ├── Cargo.toml
   ├── Makefile
   └── src
      ├── bin(基于用户库 user_lib 开发的应用，每个应用放在一个源文件中)
      │   ├── ...
      ├── console.rs
      ├── lang_items.rs
      ├── lib.rs(用户库 user_lib)
      ├── linker.ld(应用的链接脚本)
      └── syscall.rs(包含 syscall 方法生成实际用于系统调用的汇编指令，
                     各个具体的 syscall 都是通过 syscall 来实现的)
```

我们的 Cargo.toml 定义如下：

```toml
[package]
name = "user_lib"
version = "0.1.0"
authors = ["Yifan Wu <shinbokuow@163.com>"]
edition = "2018"
```

在这里，我们通过 `name="user_lib"` 将所有的文件打包为一个 `crate`。

并且，我们在 `lib.rs` 中，声明如下：

```rust
#[macro_use]
pub mod console;
mod lang_items;
mod syscall;
```

也就是，只有 `console` 是 pub 的。

#### lib.rs

>Crate 的「根入口」（库目标专属）

1. 仅存于 src/ 根目录：不能放在子目录（如 src/foo/lib.rs 无效），只能在 src/lib.rs；
2. 自动关联 crate 根模块：无需任何配置，lib.rs 中的内容直接属于 crate:: 命名空间（比如 lib.rs 中定义 pub fn hello() {}，外部可通过 use user_lib::hello; 访问，其中 user_lib 是 crate 名称）；
3. 控制 crate 对外暴露的内容：lib.rs 中用 pub mod xxx; 导出子模块、#[macro_export] 导出宏、pub use xxx; 重导出内容，外部 crate 只能访问这里公开的内容；
4. 可以在 cargo.toml 中配置修改默认的 `lib.rs`；

#### mod.rs
>子模块目录的「入口文件」：mod.rs 是 Rust 中 嵌套子模块目录 的默认入口文件，核心作用是「定义该目录对应的模块内容、组织该目录下的子文件 / 子目录」。
>简单说：当你想把一个目录（如 src/utils/）当作一个模块时，必须在该目录下创建 mod.rs，否则 Rust 无法识别这个目录是一个模块。

1. 仅存于「子模块目录」下：必须放在子目录中（如 src/utils/mod.rs），不能在 src/ 根目录下用 mod.rs 替代 lib.rs（无效）；
2. 自动关联目录模块：src/foo/mod.rs 直接对应模块 crate::foo，该目录下的其他文件（如 foo/bar.rs）需在 mod.rs 中通过 mod bar; 引入；
3. 控制子模块的对外暴露：mod.rs 中用 pub mod bar; 导出目录下的子模块，外部只能访问这里公开的内容；

#### 总结

1. Rust 以 Crate（包）-> Mod（模块） 为核心层级组织代码 —— 一个 Crate 是最小的编译单元（由 Cargo.toml 的 name 定义），内部通过 mod 拆分逻辑（文件 / 目录对应模块）；
2. Crate 的库目标（library）默认由 src/lib.rs 管理（根模块 crate::）；子模块目录（如 src/foo/）默认由 mod.rs 管理（对应模块 crate::foo::）；
3. 访问规则：
    - 普通成员（fn/struct/enum 等）：需满足「双重公开」—— 自身用 pub 修饰（如 pub fn xxx），且所在模块在「根模块到该成员的路径上」都有 pub 暴露（如 lib.rs 中 pub mod foo，foo/mod.rs 中 pub mod bar，才能让外部访问 crate::foo::bar::xxx）；
    - 宏（macro_rules!）：需用 #[macro_export] 标记（替代普通成员的 pub），且所在模块需通过 pub mod 暴露（否则外部无法识别该模块的宏）；
    - 私有模块 / 成员：仅当前 Crate 内部可访问，外部完全不可见（用于封装内部实现）。


### `#[macro_use]` 和 `#[macro_export]`

- `#[macro_use]` 就是针对于宏的 `import`；
- `#[macro_export]` 就是针对于宏的 `export`；