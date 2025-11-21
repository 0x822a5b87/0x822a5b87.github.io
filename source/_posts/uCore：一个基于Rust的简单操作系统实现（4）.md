---
title: uCore：一个基于Rust的简单操作系统实现（4）
date: 2025-11-21 17:11:09
tags:
  - os
  - rust
  - memory
---

>在这个章节，我们会实现虚拟内存机制。

```mermaid
flowchart LR
%% 样式定义：区分目录、新增文件、修改文件、普通文件
    classDef dir fill:#e0f7fa, stroke:#00695c, rounded:8px, font-weight:bold, font-size:13px;
    classDef new_file fill:#e8f5e8, stroke:#2e7d32, rounded:6px, font-size:12px, color:#1b5e20;
    classDef modified fill:#fff3e0, stroke:#ef6c00, rounded:6px, font-size:12px, color:#e65100;
    classDef normal_file fill:#f5f5f5, stroke:#78909c, rounded:6px, font-size:12px;

    root["os/"]:::dir
    os_src["src/"]:::dir

%% 一级文件（src 目录下）
    config["config.rs<br/>(修改：新增内存管理配置)"]:::modified
    linker["linker.ld<br/>(修改：引入跳板页)"]:::modified
    loader["loader.rs<br/>(修改：保留应用数据功能)"]:::modified
    main["main.rs<br/>(修改)"]:::modified
    src_other["..."]:::normal_file

%% 新增 mm 子模块
    mm["mm/<br/>(新增：内存管理子模块)"]:::dir
    mm_address["address.rs<br/>(物理/虚拟地址抽象)"]:::new_file
    mm_frame["frame_allocator.rs<br/>(物理页帧分配器)"]:::new_file
    mm_heap["heap_allocator.rs<br/>(内核动态内存分配器)"]:::new_file
    mm_memory_set["memory_set.rs<br/>(地址空间相关)"]:::new_file
    mm_mod["mod.rs<br/>(mm 模块初始化)"]:::new_file
    mm_page_table["page_table.rs<br/>(多级页表抽象)"]:::new_file

%% syscall 子模块
    syscall["syscall/"]:::dir
    syscall_fs["fs.rs<br/>(修改：基于地址空间的 sys_write)"]:::modified
    syscall_mod["mod.rs"]:::normal_file
    syscall_process["process.rs"]:::normal_file

%% task 子模块
    task["task/"]:::dir
    task_context["context.rs<br/>(修改：构造初始任务上下文)"]:::modified
    task_mod["mod.rs<br/>(修改，详见文档)"]:::modified
    task_switch["switch.rs"]:::normal_file
    task_switch_S["switch.S"]:::normal_file
    task_task["task.rs<br/>(修改，详见文档)"]:::modified

%% trap 子模块
    trap["trap/"]:::dir
    trap_context["context.rs<br/>(修改：扩展 Trap 上下文)"]:::modified
    trap_mod["mod.rs<br/>(修改：基于地址空间的 Trap 机制)"]:::modified
    trap_S["trap.S<br/>(修改：Trap 上下文保存/恢复)"]:::modified

%% 层级连接
    root --> os_src
    os_src --> config
    os_src --> linker
    os_src --> loader
    os_src --> main
    os_src --> src_other
    os_src --> mm
    os_src --> syscall
    os_src --> task
    os_src --> trap

%% mm 子模块连接
    mm --> mm_address
    mm --> mm_frame
    mm --> mm_heap
    mm --> mm_memory_set
    mm --> mm_mod
    mm --> mm_page_table

%% syscall 子模块连接
    syscall --> syscall_fs
    syscall --> syscall_mod
    syscall --> syscall_process

%% task 子模块连接
    task --> task_context
    task --> task_mod
    task --> task_switch
    task --> task_switch_S
    task --> task_task

%% trap 子模块连接
    trap --> trap_context
    trap --> trap_mod
    trap --> trap_S
```

```terminaloutput
➜  ~/code/2025a-rcore-0x822a5b87 git:(ch4) cloc --include-ext=rs,s,S,asm os
     152 text files.
     121 unique files.                              
     126 files ignored.

github.com/AlDanial/cloc v 1.82  T=0.01 s (2328.7 files/s, 184956.7 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Rust                            29            193            386           1866
Assembly                         4             10             26            140
-------------------------------------------------------------------------------
SUM:                            33            203            412           2006
-------------------------------------------------------------------------------
```


## QA

### 内存管理的一些示例

>内存管理的完全抽象

![address translation](/images/20251121/address-translation.png)

>分段内存管理

![simple-base-bound](/images/20251121/simple-base-bound.png)