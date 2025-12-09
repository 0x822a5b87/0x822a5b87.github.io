---
title: uCore文件系统
date: 2025-12-09 10:23:14
tags:
  - os
  - rust
  - fs
---

# 文件系统

## `等待处理的问题`

1. 画一下内核，进程，文件描述符，文件之间的关系；

## 文件与文件描述符

### 文件简介

这个接口在内存和I/O资源之间建立了数据交换的通道。其中 `UserBuffer` 是我们在 `mm` 子模块中定义的应用地址空间中的一段缓冲区，我们可以将它看成一个 `&[u8]` 切片。

```rust
// os/src/fs/mod.rs

pub trait File : Send + Sync {
    fn readable(&self) -> bool;
    fn writable(&self) -> bool;
    fn read(&self, buf: UserBuffer) -> usize;
    fn write(&self, buf: UserBuffer) -> usize;
}
```

`UserBuffer` 的声明如下，此外，**我们还让它作为一个迭代器可以逐字节进行读写**，参考 [Iterator和IntoIterator](#iterator和intoiterator)

```rust
// os/src/mm/page_table.rs

pub fn translated_byte_buffer(
    token: usize,
    ptr: *const u8,
    len: usize
) -> Vec<&'static mut [u8]>;

pub struct UserBuffer {
    pub buffers: Vec<&'static mut [u8]>,
}

impl UserBuffer {
    pub fn new(buffers: Vec<&'static mut [u8]>) -> Self {
        Self { buffers }
    }
    pub fn len(&self) -> usize {
        let mut total: usize = 0;
        for b in self.buffers.iter() {
            total += b.len();
        }
        total
    }
}
```

### 标准输入和标准输出

我们定义标准输入和标准输出分别为 `Stdin` 和 `Stdout`

```rust
/// stdin file for getting chars from console
pub struct Stdin;

/// stdout file for putting chars to console
pub struct Stdout;
```

并为他们实现 `File` 接口

```rust
impl File for Stdin {
    fn readable(&self) -> bool { true }
    fn writable(&self) -> bool { false }
    fn read(&self, mut user_buf: UserBuffer) -> usize {
        assert_eq!(user_buf.len(), 1);
        // busy loop
        let mut c: usize;
        loop {
            c = console_getchar();
            if c == 0 {
                suspend_current_and_run_next();
                continue;
            } else {
                break;
            }
        }
        let ch = c as u8;
        unsafe {
            user_buf.buffers[0].as_mut_ptr().write_volatile(ch);
        }
        1
    }
    fn write(&self, _user_buf: UserBuffer) -> usize {
        panic!("Cannot write to stdin!");
    }
}

impl File for Stdout {
    fn readable(&self) -> bool {
        false
    }
    fn writable(&self) -> bool {
        true
    }
    fn read(&self, _user_buf: UserBuffer) -> usize {
        panic!("Cannot read from stdout!");
    }
    fn write(&self, user_buf: UserBuffer) -> usize {
        for buffer in user_buf.buffers.iter() {
            print!("{}", core::str::from_utf8(*buffer).unwrap());
        }
        user_buf.len()
    }
}
```

### 文件描述符与文件描述符表

1. 每个进程包含了一个 `文件描述符表`，包含了所有打开的文件；
2. `文件描述符表` 的每一项都是一个 `文件描述符`，文件描述符是一个非负整数，**表示文件描述符表中一个打开的 文件描述符 所处的位置**；
3. 使用 `open()` 或者 `create()` 打开或创建一个文件时，内核会返回给应用刚刚打开或创建的文件对应的文件描述符；
4. 使用 `close()` 关闭文件时，需要提供相应的文件描述符。

### 文件I/O操作



## QA

### 代码树

```
├── easy-fs(新增：从内核中独立出来的一个简单的文件系统 EasyFileSystem 的实现)
│   ├── Cargo.toml
│   └── src
│       ├── bitmap.rs(位图抽象)
│       ├── block_cache.rs(块缓存层，将块设备中的部分块缓存在内存中)
│       ├── block_dev.rs(声明块设备抽象接口 BlockDevice，需要库的使用者提供其实现)
│       ├── efs.rs(实现整个 EasyFileSystem 的磁盘布局)
│       ├── layout.rs(一些保存在磁盘上的数据结构的内存布局)
│       ├── lib.rs
│       └── vfs.rs(提供虚拟文件系统的核心抽象，即索引节点 Inode)
├── easy-fs-fuse(新增：将当前 OS 上的应用可执行文件按照 easy-fs 的格式进行打包)
│   ├── Cargo.toml
│   └── src
│       └── main.rs
├── os
    ├── build.rs(修改：不再需要将用户态程序链接到内核中）
    ├── Cargo.toml(修改：新增 Qemu 的块设备驱动依赖 crate)
    ├── Makefile(修改：新增文件系统的构建流程)
    └── src
        ├── config.rs(修改：新增访问块设备所需的一些 MMIO 配置)
        ├── ...
        ├── drivers(新增：Qemu 平台的块设备驱动)
        │   ├── block
        │   │   ├── mod.rs(将不同平台上的块设备全局实例化为 BLOCK_DEVICE 提供给其他模块使用)
        │   │   └── virtio_blk.rs(Qemu 平台的 virtio-blk 块设备)
        │   └── mod.rs
        ├── fs(新增：对文件系统及文件抽象)
        │   ├── inode.rs(新增：将 easy-fs 提供的 Inode 抽象封装为内核看到的 OSInode
        │   │            并实现 fs 子模块的 File Trait)
        │   ├── mod.rs
        │   └── stdio.rs(新增：将标准输入输出也抽象为文件)
        ├── loader.rs(移除：应用加载器 loader 子模块，本章开始从文件系统中加载应用)
        ├── mm
        │   ├── address.rs
        │   ├── frame_allocator.rs
        │   ├── heap_allocator.rs
        │   ├── memory_set.rs(修改：在创建地址空间的时候插入 MMIO 虚拟页面)
        │   ├── mod.rs
        │   └── page_table.rs(新增：应用地址空间的缓冲区抽象 UserBuffer 及其迭代器实现)
        ├── syscall
        │   ├── fs.rs(修改：新增 sys_open，修改sys_read、sys_write)
        │   ├── mod.rs
        │   └── process.rs(修改：sys_exec 改为从文件系统中加载 ELF)
        ├── task
            ├── context.rs
            ├── manager.rs
            ├── mod.rs(修改：初始进程 INITPROC 的初始化)
            ├── pid.rs
            ├── processor.rs
            ├── switch.rs
            ├── switch.S
            └── task.rs(修改：在任务控制块中加入文件描述符表的相关机制)
```
### 代码统计

```
➜  2025a-rcore-0x822a5b87 git:(ch6) cloc os easy-fs
     310 text files.
     164 unique files.
     411 files ignored.

github.com/AlDanial/cloc v 2.06  T=0.13 s (1275.0 files/s, 46056.3 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Rust                            45            304            697           3563
D                               43            120              0            666
Assembly                         4             24             26            265
JSON                            67              0              0             67
make                             1             22              7             67
Linker Script                    1              7              0             46
TOML                             3              8              2             33
-------------------------------------------------------------------------------
SUM:                           164            485            732           4707
-------------------------------------------------------------------------------
```

### Iterator和IntoIterator

`IntoIterator` 和 `Iterator` 是 Rust 中遍历集合的两个核心 trait：

- `IntoIterator` 负责 “把一个类型转换成迭代器”
- `Iterator` 负责 “定义迭代的具体行为”

```mermaid
flowchart LR

struct("struct"):::green

IntoIteratorInstance("struct转换为Iterator"):::purple
IteratorInstance("迭代操作"):::purple

struct -.->|IntoIterator| IntoIteratorInstance -.->|Iterator| IteratorInstance

classDef pink 1,fill:#FFCCCC,stroke:#333, color: #fff, font-weight:bold;
classDef green fill: #696,color: #fff,font-weight: bold;
classDef purple fill:#969,stroke:#333, font-weight: bold;
classDef error fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5
classDef coral fill:#f9f,stroke:#333,stroke-width:4px;
classDef animate stroke-dasharray: 9,5,stroke-dashoffset: 900,animation: dash 25s linear infinite;
```

#### Iterator

`Iterator` 的定义相当复杂，但是实际上，我们需要做的事只有两个：

1. `type Item = *mut u8` 来定义 trait 的**关联类型（Associated Type）**，详细解析可以参考 [关联类型和泛型参数](#关联类型和泛型参数)；
2. `next()` 函数； 

```rust
#[doc(notable_trait)]
#[lang = "iterator"]
#[rustc_diagnostic_item = "Iterator"]
#[must_use = "iterators are lazy and do nothing unless consumed"]
pub trait Iterator {
    /// The type of the elements being iterated over.
    #[rustc_diagnostic_item = "IteratorItem"]
    #[stable(feature = "rust1", since = "1.0.0")]
    type Item;

    #[lang = "next"]
    #[stable(feature = "rust1", since = "1.0.0")]
    fn next(&mut self) -> Option<Self::Item>;
}
```

#### IntoIterator

`IntoIterator` 解决的问题是：**如何把一个 “集合类型”（如 `Vec<T>`）转换成 “迭代器类型”（如 `std::vec::IntoIter<T>`）**，我们需要的有三步：

1. `Item` 对迭代的类型进行类型关联；
2. `IntoIter` 对迭代器的类型进行类型关联；
3. `fn into_iter()` 方法进行类型迭代；

```rust
impl IntoIterator for UserBuffer {
    type Item = *mut u8;
    type IntoIter = UserBufferIterator;
    fn into_iter(self) -> Self::IntoIter {
        UserBufferIterator {
            buffers: self.buffers,
            current_buffer: 0,
            current_idx: 0,
        }
    }
}
```

### 关联类型和泛型参数

`关联类型` 和 `泛型参数`有相似的 “类型抽象” 作用，但本质是两种不同的 Rust 语法机制。他们最大的区别在于：

1. **关联类型**：在 trait **内部** 声明（`type Item;`），**一个类型对 trait 的单次实现只能绑定一种具体类型**（比如为 `VecIter` 实现 `Iterator`，`Item` 只能是 `i32` 或 `String`，二选一）；
2. **泛型参数**：在 trait/impl/ 函数 **名称后** 声明（`<T>`），**一个类型对 trait 的单次实现可适配任意类型**（比如 `impl<T> MyTrait<T> for MyType`，`T` 可以是 `i32`/`String`/ 任意类型），编译器会为每种实际使用的类型生成单独的函数版本（单态化）。

> 为什么我们在迭代器里使用关联类型而不是泛型参数呢？

迭代器的语义就是**天然只能产出一种元素**，这个和**关联类型**是天然匹配的，例如，假设我们使用泛型参数来实现迭代器：

```rust
struct Vec<T>(Vec<T>);

impl<T> Iterator<T> for Vec<T> {
    fn next(&mut self) -> Option<T> {
        self.0.pop()
    }
}
```

但是，我们甚至可以为它实现一个单独的迭代器来违反这个语义：

```rust
impl Iterator<String> for Vec<i32> {
    fn next(&mut self) -> Option<String> {
        Some("illegal values".to_string())
    }
}
```

而这种语义在使用关联类型时是非法的，以下的代码没有办法通过编译：

```rust
impl Iterator for UserBufferIterator {
    type Item = usize;

    fn next(&mut self) -> Option<Self::Item> {
        Some(0usize)
    }
}

impl Iterator for UserBufferIterator {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        Some(0u64)
    }
}
```

但是在迭代器中，我们仍然可以为它实现一个泛型的迭代器，只是通过关联类型避免了我们刚才提到的问题：

```rust
struct Hello<T> where T : Copy {
    t: T
}

impl <T> Iterator for Hello<T> where T:Copy {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        Some(self.t.clone())
    }
}
```

