---
title: build my own docker from scratch
date: 2025-12-18 23:24:46
tags:
  - go
  - docker
---

# build my own docker from scratch

> Docker is an OS‑level virtualization (or containerization) platform, which allows applications to share the host OS kernel instead of running a separate guest OS like in traditional virtualization.

# 前言

> 本文基于 `OpenCloudOS` 操作系统以及 `x86_64` 平台，`go1.24.11` 语言开发测试，全部代码位于 [tiny-docker](https://github.com/0x822a5b87/tiny-docker)。
>
> ```bash mark:1,4,7
> uname -a
> # Linux VM-0-10-opencloudos 6.6.117-45.1.oc9.x86_64 #1 SMP Tue Dec 16 11:49:47 CST 2025 x86_64 x86_64 x86_64 GNU/Linux
> 
> go version
> # go version go1.24.11 linux/amd64
> 
> docker --version
> # Docker version 29.1.3, build f52814d
> ```

[Docker](https://www.docker.com/) 是基于 Linux Namespace 和 Cgroup 实现的轻量级虚拟化平台。相较于传统虚拟机（如 [Oracle VirtualBox](https://www.virtualbox.org/)），Docker 无需模拟硬件和运行独立内核，仅通过内核级的资源隔离与限制实现 “虚拟环境”，因此更轻量级（毫秒级启动）、资源占用更低 —— 同一台宿主机上的所有容器**共享宿主机内核**，这是容器与虚拟机的核心区别。

**容器的本质是：Linux 内核通过隔离 “进程、文件、网络、用户” 等资源视图，并限制资源使用量，构建出的 “类独立主机环境”**。其核心实现可拆解为三大模块：

1. 主机身份标识（UTS Namespace 隔离）
  - 核心对象：hostname（主机名）和 domainname（NIS 域名，现代几乎废弃）；
  - 作用：让容器拥有独立的主机身份标识，模拟 “独立机器” 的特征（如容器内修改 hostname 不会影响宿主机或其他容器）；
  - 本质：内核通过 struct utsname 结构体为不同容器维护独立的身份副本，仅在各自 UTS Namespace 内生效。
2. 资源视图隔离（各类 Namespace 核心能力）
  - 通过 Linux Namespace 为容器隔离核心资源的 “可见范围”，让容器仅能感知到自己的资源：
  - 进程隔离（PID Namespace）：容器内进程 ID 从 1 开始，无法看到宿主机 / 其他容器的进程；Linux 进程天然拥有独立的虚拟内存地址空间，同容器
  - 进程可通过消息队列、共享内存、管道等方式通信，跨容器进程需通过网络 / 挂载卷间接通信。
  - 文件系统隔离（Mount Namespace）：容器拥有独立的根文件系统（/），挂载 / 卸载操作仅影响自身，不波及宿主机（如容器内删除 /bin/sh 不会影响宿主机）。
  - 网络隔离（Network Namespace）：容器拥有独立的网卡、IP、端口和路由表，默认与宿主机 / 其他容器网络隔离，需通过端口映射、网桥等方式对外通信。
  - 用户 / 组隔离（User Namespace）：容器内的 root（UID 0）不等同于宿主机的 root，实现权限隔离（即使容器内以 root 运行，宿主机层面仅拥有普通用户权限）。
  - IPC 隔离（IPC Namespace）：容器内的消息队列、共享内存等 IPC 资源仅对本容器可见，避免跨容器的 IPC 干扰。
3. 资源限制工具（[cgroup](#cgroup)）
  - Cgroup（控制组）并非 “隔离资源视图”，而是限制容器对宿主机资源的使用量，防止单个容器占用过多资源导致宿主机崩溃：
  - 核心管控维度：CPU 使用率、内存上限、磁盘 IO 速率、进程数量、网络带宽等；
  - 作用：保证多容器在同一宿主机上公平调度（如限制容器最多使用 1 核 CPU、2GB 内存）。

**当以上三类能力（身份标识隔离 + 核心资源视图隔离 + 资源用量限制）共同作用时，容器就成为了一个 “对宿主机无感知、对外表现为独立主机” 的轻量级运行环境。**

```mermaid
flowchart BT

subgraph hardware
    cpu("cpu")
    gpu("gpu")
    disk("disk")
    net_card("物理网卡")
    memory("memory")
end

subgraph kernel[Linux Kernel 内核]
    subgraph kernel_core[内核核心能力]
        process("进程管理")
        fs("文件系统")
        net("网络协议栈")
        user("用户/组管理")
        ipc("进程间通信")
    end

    subgraph cgroup[资源限制 cgroup 子系统]
        cgroup1("cpu 限制")
        cgroup3("memory 限制")
        cgroup4("blkio/disk 限制")
        cgroup5("pid 数量限制")
    end
end

subgraph view[Namespace 视图隔离]
    PID("PID Namespace")
    Mount("Mount Namespace")
    Network("Network Namespace")
    User("User Namespace")
    IPC("IPC Namespace")
end

subgraph identity[主机身份标识]
    UTS("UTS Namespace(hostname/domain)")
end

subgraph containers[容器集群]
    mysql("mysql 容器")
    redis("redis 容器")
    nginx("nginx 容器")
end

hardware --> kernel
kernel_core --> view
kernel_core --> identity

cgroup --> containers
view --> containers
identity --> containers
```



```bash
docker run -it -d \
			--network host \
			--name example-container \
			-v ${CURDIR}:/mnt \
			-w /mnt \
			example-container
```

# docker依赖的linux基础特性

## namespace

### UTS namespace

我们的第一个例子，我们使用 `CLONE_NEWUTS` 指定我们要创建一个 `UTS Namespace`。

```go mark:9
// StartContainer start a container with UTS Namespace in linux
func StartContainer(cmd string) error {
	command := exec.Command(cmd)
	command.Stdin = os.Stdin
	command.Stdout = os.Stdout
	command.Stderr = os.Stderr

	command.SysProcAttr = &unix.SysProcAttr{
		Cloneflags: unix.CLONE_NEWUTS,
	}

	return command.Run()
}
```

我们可以验证我们的父进程和子进程是否在同一个 `UTS namespace`。

```shell
echo $$
# 255394

pstree -p
           ├─sshd(938)─┬─sshd(229522)───sshd(229527)
           │           └─sshd(229809)───sshd(229848)───bash(229849)───uts(255388)─┬─sh(255394)───pstree(255537)
           │                                                                      ├─{uts}(255389)
           │                                                                      ├─{uts}(255390)
           │                                                                      ├─{uts}(255391)
           │                                                                      └─{uts}(255392)
           

readlink /proc/255394/ns/uts
# uts:[4026532361]

readlink /proc/255388/ns/uts
# uts:[4026531838]
```

### User namespace

在我们使用 `User namespace` 时：

- `Credential#HostID` 会被映射为 `UidMappings#ContainerID`；
- `Credential#Gid` 会被映射为 `GidMappings#ContainerID`；
- 并且映射的值必须完全一致，否则会抛出异常 invalid argument 。

```go
func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: unix.CLONE_NEWUTS |
			unix.CLONE_NEWIPC |
			unix.CLONE_NEWPID |
			unix.CLONE_NEWNS |
			unix.CLONE_NEWUSER,
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getuid(),
				Size:        1,
			},
		},
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getgid(),
				Size:        1,
			},
		},
	}
	// Explicitly set the UID and GID to prevent tasks from running as root.
	cmd.SysProcAttr.Credential = &syscall.Credential{
		Uid:         uint32(1),
		Gid:         uint32(1),
		NoSetGroups: true,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```

## cgroups

> Cgroup 的使用可以参考 [cgroup](#cgroup)。

**Cgroup 是 Linux 内核提供的资源隔离 / 限制机制**，而 `hierarchy`（层级）、`subsystem`（子系统）是实现这一机制的核心组件，三者的关系可总结为：

- `subsystem` 是 “资源控制能力”（如限制 CPU / 内存）
- `hierarchy` 是 “组织容器 / 进程的树形结构”
- `cgroup` 是 “挂载在层级上、绑定了子系统的具体控制组（进程组）”。

### 核心概念

#### Subsystem

Subsystem 是内核提供的 “资源管控插件”，每个插件只负责一种资源的限制 / 统计，比如 “CPU 子系统” 只能管 CPU，“内存子系统” 只能管内存。我们可以认为他们是一组内核钩子函数，通过挂载到 Cgroup 层级上，对层级内的 cgroup 中的进程进行资源限制、计量或隔离。

```shell
lssubsys -a

# cpuset
# cpu
# cpuacct
# blkio
# memory
# devices
# freezer
# net_cls
# perf_event
# net_prio
# hugetlb
# pids
# rdma
# misc
```

| 子系统名     | 核心作用（通俗版）                                | 典型使用场景                                                |
| ------------ | ------------------------------------------------- | ----------------------------------------------------------- |
| `cpuset`     | 绑定进程到**指定 CPU 核心 / NUMA 节点**，独占核心 | 高性能容器（数据库 / 计算密集型）独占 CPU 核                |
| `cpu`        | 限制 CPU 使用率、调度优先级，控制 CPU 调度周期    | 限制容器 CPU 使用率（如最多用 1 核）、多容器 CPU 权重分配   |
| `cpuacct`    | 统计进程 / 容器的 CPU 占用（用户态 / 内核态）     | 容器 CPU 使用率监控、计费统计                               |
| `blkio`      | 限制块设备（磁盘）IO 速率 / 优先级                | 限制容器磁盘 IO（防止写满磁盘）、数据库 IO 优先级保障       |
| `memory`     | 限制内存使用（物理内存 / 交换分区），统计内存     | 容器内存上限（OOM 触发）、禁用 swap 避免性能下降            |
| `devices`    | 控制进程对设备的访问权限（读 / 写 / 创建）        | 容器隔离设备（如禁止访问宿主机 GPU / 串口）、仅允许访问网卡 |
| `freezer`    | 暂停 / 恢复 cgroup 内的所有进程                   | 容器暂停（如备份时）、资源超限时临时冻结进程                |
| `net_cls`    | 给进程的网络数据包打标记（classid）               | 结合 TC（流量控制）限制容器网络带宽、区分不同容器的流量     |
| `perf_event` | 允许 perf 工具对 cgroup 内进程做性能分析          | 容器性能剖析（如分析容器内进程 CPU 热点）                   |
| `net_prio`   | 控制进程网络数据包的发送优先级                    | 核心业务容器网络优先级高于普通容器                          |
| `hugetlb`    | 限制大页内存（HugeTLB）的使用                     | 数据库 / 虚拟化场景（用大页提升内存性能）的大页限制         |
| `pids`       | 限制 cgroup 内允许创建的最大进程数                | 防止容器 fork 炸弹（无限创建进程）、限制容器进程数          |
| `rdma`       | 限制 RDMA（远程直接内存访问）资源使用             | 高性能计算 / 存储场景（RDMA 网卡）的资源隔离                |
| `misc`       | 杂项资源控制（内核新增，适配小众场景）            |                                                             |

#### Hierarchy

Hierarchy 是一棵 “进程分组树”，树的每个节点是一个 cgroup，进程可以被加入树的任意节点，且会继承父节点的资源限制。我们可以认为挂载在 `/sys/fs/cgroup/` 下的一个独立树形文件系统（Cgroupfs），一个层级可以绑定**零个或多个 subsystem**，但一个 subsystem 只能绑定到**一个层级**（核心约束）

#### cgroup（控制组）

Cgroup 是层级树上的一个 “节点”，本质是一组进程的集合，且绑定了 subsystem 的资源规则（比如 “这个节点的进程最多用 1 核 CPU、2G 内存”）。挂载的 Cgroup 层级下的一个目录（目录即 cgroup），目录内的文件（如 `cpu.cfs_quota_us`）是资源规则配置，`tasks` 文件记录该 cgroup 包含的进程 PID：

- 一个 cgroup 可以包含多个进程，一个进程可以属于多个层级的 cgroup；
- 空 cgroup（无进程）不占用资源，仅保存配置；
- 删除 cgroup 只需删除对应的目录（内核会自动清理关联规则）。

#### 三者的核心关系

假设我们要限制一个容器的 CPU 和内存，操作流程如下：

##### 创建hierarchy

挂载一个 Cgroup 层级到 `/sys/fs/cgroup/my-hierarchy/`

```shell
mkdir /sys/fs/cgroup/my-hierarchy
```

执行后，内核会自动的为我们在 `/sys/fs/cgroup/my-hierarchy` 下创建文件，每个文件都对应了 `subsystem` 的一些功能，下面列出了一些典型的例子：

```shell
ls /sys/fs/cgroup/my-hierarchy
# cgroup.freeze
# cpuset.cpus
# cpu.max
# ...
```

##### 配置 Subsystem

这里，我们通过配置来对我们的CPU使用进行限制。

```shell
cd /sys/fs/cgroup/my-hierarchy
# 限制 CPU 使用率为 50%（100ms 周期内最多用 50ms）
echo "50000 100000" > cpu.max

# 限制 CPU 使用率为 200%（允许使用 2 个核心，100ms 周期内最多用 200ms）
#echo "200000 100000" > cpu.max

# 取消限制（设为 max）
#echo "max 100000" > cpu.max
```

##### 测试我们的cgroup

我们实现一个最简单的忙等待

```shell
while :; do :; done

pkill -f "while :; do :; done"
```

后台执行该脚本

```shell
sh busy.sh &
# [1] 1116447
```

通过 `top` 查看我们的进程的CPU占用：

```shell
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
1116447 root      25   5    7108   3496   3212 R  99.7   0.0   2:11.49 sh
```

随后我们将我们的进程加入到 `cgroup`：

```shell
echo 1116447 >> /sys/fs/cgroup/my-hierarchy/cgroup.procs
```

再次通过 `top` 观察

```shell
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
1116447 root      25   5    7108   3496   3212 R  49.8   0.0   4:43.34 sh
```

### 整体回顾

Linux 的 Cgroup（Control Group）是内核提供的进程资源管控系统，核心挂载路径为 `/sys/fs/cgroup`，系统默认在此路径下为核心 `subsystem`（如 cpu/memory/blkio）创建好预挂载的层级（Hierarchy）。

通常我们使用cgroup，我们会按照如下步骤实现：

1. 在 `/sys/fs/cgroup` 下通过 `mkdir` 新建一个文件夹，这个文件夹是一个虚拟文件系统目录，需要通过 `rmdir` 而非 `rm` 来删除。这个文件夹也就是我们的 cgroup 的 `hierarchy`；在新建hierarchy之后，kernel会自动的添加 `subsystem` 到hierarchy下；
2. 通过 `subsystem` 中的参数，我们定制了一个可以对特定资源做限制的 cgroup，而我们可以通过修改 `cgroup.procs` 来指定被该 cgroup 约束的进程。

### docker如何使用cgroup

`docker` 在启动的时候，会为每个 container 初始化一个对应的 hierarchy 挂载在 `/sys/fs/cgroup/system.slice` 下的特定文件。

```shell
docker run -itd --rm --cpus=1.0 -m 128m ubuntu
# e77265ce43b8b9b95894a1fa07649c4438d22e4d7d3aa254edd0f5786ace1e29

cat /sys/fs/cgroup/system.slice/docker-e77265ce43b8b9b95894a1fa07649c4438d22e4d7d3aa254edd0f5786ace1e29.scope/memory.max
# 134217728

cat /sys/fs/cgroup/system.slice/docker-e77265ce43b8b9b95894a1fa07649c4438d22e4d7d3aa254edd0f5786ace1e29.scope/cpu.max
# 100000 100000
```

我们也可以用代码来模拟这个行为，这里我们需要注意的几个点是：

1. 我们必须在创建进程之前，将进程绑定到我们的cgroup。这里听起来有一点矛盾，实际情况是，`stress` 执行会有两个进程：**调度进程**和**工作进程**。如果我们等进程被拉起之后再将主进程添加到cgroup，那么我们工作进程将不会受到任何该cgroup的限制。我们很多途径可以处理这个问题：
   1. 增加一个函数从 `/proc/$PID/` 去解析所有子进程并加入到 cgroup；
   2. 通过 `UseCgroupFD` 和 `CgroupFD` 预先指定绑定的 cgroup；
2. 如果我们成功的绑定了 cgroup，那程序执行时将会因为使用过量的资源被直接 kill。

```go
const cgroupMemoryHierarchyMount = "/sys/fs/cgroup"
const cgroupMemoryMax = "memory.max"

// 提前创建一个 cgroup 组，指定资源量
func createCgroup() (*os.File, error) {
	hierarchy := path.Join(cgroupMemoryHierarchyMount, "test_memory_limit")
	err := os.Mkdir(hierarchy, 0755)
	if err != nil {
		log.Fatal(err)
	}

	memoryMax := path.Join(hierarchy, cgroupMemoryMax)
	err = os.WriteFile(memoryMax, []byte("100m"), 0644)
	if err != nil {
		log.Fatal(err)
	}

	return os.OpenFile(hierarchy, os.O_RDONLY, 0644)
}

// 在子进程中执行 stress
func runStress() {
	fmt.Printf("current pid %d", syscall.Getpid())
	fmt.Println()
	cmd := exec.Command("sh", "-c", `stress --vm-bytes 2048m --vm-keep -m 1`)
	cmd.SysProcAttr = &unix.SysProcAttr{}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		fmt.Printf("%v\n", err)
		os.Exit(1)
	}
}

func main() {
	// Check if the program is relaunched via /proc/self/exe
	if os.Args[0] == "/proc/self/exe" {
		runStress()
		os.Exit(0)
	}

	cgroup, err := createCgroup()
	if err != nil {
		panic(err)
	}
	defer func(cgroup *os.File) {
		_ = cgroup.Close()
	}(cgroup)

	cmd := exec.Command("/proc/self/exe")
	cmd.SysProcAttr = &unix.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWPID |
			syscall.CLONE_NEWNS,
			UseCgroupFD: true,
			CgroupFD: int(cgroup.Fd()),
	}

	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Start(); err != nil {
		log.Println("start command error : ", err)
		os.Exit(1)
	}
}
```

# 构造容器

## 基于namespace构造容器

在真正开始构造我们的容器之前，我们可以学习一下关于 [/proc](#proc)  的知识。

从linux的层面，想要启动一个容器，我们的设想方向应该是这样的：

1. 通过 `fork()` + `namespace` 来生成一个隔离空间；
2. 在 `fork()` 出来的隔离空间中，通过 `exec()` 来执行用户代码程序。

于是，我们的思路就可以这样设计：

1. 声明 `run` 和 `init` 指令
   1. `run` 指令负责通过 namespace 生成隔离空间；
   2. `init` 指令负责在 `run` 生成隔离空间后，在隔离空间内通过 `exec(command)` 执行用户程序。

```mermaid
sequenceDiagram
    宿主机 ->> docker: 1. docker run ubuntu
    docker ->> runCommand: 2. 解析参数
    runCommand ->> NewParentProcess: 3. 创建namespace隔离的容器进程
    NewParentProcess -->> runCommand: 4. 返回配置好隔离参数的Command
    runCommand -->> docker: 5. 启动容器进程
    docker -->> docker: 6. 在隔离的容器进程中使用参数<br/>（init ${command}）调用自身（/proc/self/exe）
    docker ->> RunContainerInitProcess: 7. mount proc 并通过 exec 执行用户代码
    RunContainerInitProcess ->> 宿主机: 8.创建完成，容器开始运行
```

整体逻辑可以描述为：

1. 宿主机：用户执行 `docker run -it sh`：
   1. 程序解析 `run` 指令后，调用 `NewParentProcess`；
   2. 构造子进程执行参数：`["init", ${command}]`（如 `["init", "sh"]`）；
   3. 创建子进程对象：指定子进程执行 `/proc/self/exe`（当前程序自身），并传入上述参数；
   4. 为子进程配置 UTS/PID/Mount/Network/IPC 五种 Namespace 隔离，若指定 TTY 则绑定宿主机终端；
   5. 启动该子进程（此时子进程进入独立的 Namespace 隔离空间）。
2. 子进程（容器内）
   1. 子进程在**隔离空间**内执行 `/proc/self/exe init ${command}`，触发程序的 CLI 框架逻辑：
   2. 执行 `initCommand` 的 `Action` 函数，调用 `RunContainerInitProcess`；
   3. `RunContainerInitProcess` 先以安全标志挂载 `/proc` 文件系统（适配 PID Namespace），再通过 `syscall.Exec` 系统调用替换当前子进程镜像为用户指定的 `${command}`（如 `sh`）；
   4. `${command}` 成为容器内的 PID=1 进程，容器正式运行（若绑定 TTY 则支持交互式操作）。

```go
// run.go
func Run(tty bool, command string) {
	parent := container.NewParentProcess(tty, command)
	if err := parent.Start(); err != nil {
		log.Error(err)
	}
	parent.Wait()
	os.Exit(-1)
}


// container_process.go
func NewParentProcess(tty bool, command string) *exec.Cmd {
	args := []string{"init", command}
	cmd := exec.Command("/proc/self/exe", args...)
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWPID |
			syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET |
			syscall.CLONE_NEWIPC,
	}
	if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	}
	return cmd
}

// init.go
func RunContainerInitProcess(command string, args []string) error {
	logrus.Infof("command %s", command)
	defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
	syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")
	argv := []string{command}
	if err := syscall.Exec(command, argv, os.Environ()); err != nil {
		logrus.Errorf(err.Error())
	}
	return nil
}
```

## 为构造的容器增加cgroup限制

想要为我们的容器增加 `cgroup` 限制，我们最简单的方式就是，构造一个cgroup字符串，

> docker 中为 container 增加限制的逻辑是：
>
> 1. 在 docker run 的时候，为app生成一个唯一的container-id；
> 2. 在 `/sys/fs/cgroup/system.slice/docker-$containner-id.scope/`  内生成一个 hierarchy；
> 3. 在 `hierarchy` 的文件下写入到对应的数据来限制container的资源使用；
> 4. 在 `hierarchy` 的文件 `cgroup.procs` 添加该容器对应进程的 `pid`。
>

### cgroup的分类

`docker` 中的配置文件，我们可以直接通过以下指令来查看：

```shell
ll /sys/fs/cgroup/system.slice/docker.service/

# -r--r--r-- 1 root root 0 Dec 20 08:14 cgroup.procs
# -rw-r--r-- 1 root root 0 Dec 19 11:41 cgroup.subtree_control
# -rw-r--r-- 1 root root 0 Dec 19 11:41 cpu.max
# -rw-r--r-- 1 root root 0 Dec 19 11:41 memory.max
# -rw-r--r-- 1 root root 0 Dec 19 11:41 pids.max
```

这里，我们只保留了一些最具有代表性的指标：

| 一级分类 | 二级文件                 | 作用简述                     |
| -------- | ------------------------ | ---------------------------- |
| `cgroup` | `cgroup.procs`           | 关联到该 cgroup 的进程 PID   |
| `cgroup` | `cgroup.subtree_control` | 控制子 cgroup 可启用的控制器 |
| `memory` | `memory.max`             | 内存使用上限                 |
| `memory` | `memory.current`         | 当前内存使用量               |
| `pids`   | `pids.max`               | 进程数上限                   |

那么我们实际的分类可以划分如下（我们省略了大部分的实例）：

```mermaid
flowchart TB

linux-cgroup("linux-cgroup"):::pink

linux-cgroup --> cgroup("cgroup"):::purple
linux-cgroup --> memory("memory"):::purple
linux-cgroup --> cpu("cpu"):::purple
linux-cgroup --> cpuset("cpuset"):::purple
linux-cgroup --> io("io"):::purple
linux-cgroup --> pids("pids"):::purple
linux-cgroup --> others("..."):::purple

cgroup --> procs("cgroup.procs"):::green
cgroup --> subtree_control("cgroup.subtree_control"):::green

memory --> memory.max("memory.max"):::green
memory --> memory.swap.max("memory.swap.max"):::green

cpu --> cpu.max("cpu.max"):::green


cpuset --> cpuset.cpus("cpuset.cpus"):::green

io --> io.max("io.max"):::green
io --> io.latency("io.latency"):::green

pids --> pids.max("pids.max"):::green
pids --> pids.current("pids.current"):::green


classDef pink   fill:#FFCCCC,stroke:#333,ont-weight: bold;
classDef green  fill:#696,color: #fff,font-weight: bold;
classDef purple fill:#969,stroke:#333;
classDef dotted fill:#bbf,stroke:#f66,stroke-width:2px,color:#fff,stroke-dasharray: 5 5

```

## 使用cgroup隔离资源

我们目前的思路是：将整个引擎分为多层：

1. 将指标分为两个部分，`Value` 表示指标本身，`Item` 表示指标内部的值，例如：
    - `Value` 是一个接口，提供了 `From` 和 `Into` 在 `Value` 和 `string` 类型之间的转换；
    - `Item` 是一个 `any` 类型；
    - 对于 `cgroups.procs` 类型，`Value` 对应整个 pid 数组， `Item` 对应数组内的每个 pid；
    - 对于 `cpu.max`  类型，`Value` 和 `Item` 均对应同一个值，因为他们是一个整体；
2. 抽象了 `Subsystem[I Item, V Value]` 接口，每个不同的类型都需要实现这个接口来实现修改/删除等功能：
    - `Set(I)`
    - `Del(I)`
    - `Empty() bool`
3. 最底层为 `CgroupFileSystem`，这一层负责和操作系统交互。更明确来说，就是负责读/写 `/sys/fs/cgroup/system.slice` 文件夹下的配置文件，并向上提供读/写接口；
4. `CgroupFileSystem` 的上层为 `CgroupManager`，他包含了：
    - 对 `CgroupFileSystem` 的引用，用于读/写底层文件；
    - 对多个 `Subsystem` 的引用，用于操作指标值；

整体结果逻辑如图所示：

```mermaid
classDiagram
    %% 严格按要求定义样式

    %% 核心抽象层（接口/泛型）
    class Item:::typeAlias {
        <<type alias>>
        any
    }
    class Value:::interface {
        <<interface>>
    }
    class BaseSubsystem:::interface {
        <<interface>>
    }
    class Subsystem:::genericInterface {
        <<generic interface>>
        BaseSubsystem
    }

    %% 基础设施层（具体实现）
    class CgroupFileSystem:::concrete {
        Path string
        AutoCreate bool
    }
    class CgroupManager:::concrete {
        fs *CgroupFileSystem
        procsSubsystem *ProcsValueSubsystem
        cpuMaxSubsystem *cpu.MaxValueSubsystem
        memoryMaxSubsystem *memory.MaxValueSubsystem
    }

    %% 外部依赖子系统（CPU/Procs/Memory）
    class cgroup.ProcsValueSubsystem:::external {
        <<external>>
        Procs 子系统
    }
    class cpu.MaxValueSubsystem:::external {
        <<external>>
        CPU 子系统
    }
    class memory.MaxValueSubsystem:::external {
        <<external>>
        Memory 子系统
    }

    %% 接口继承/实现关系
    BaseSubsystem <|-- Subsystem : 继承
    BaseSubsystem <|-- cgroup.ProcsValueSubsystem : 实现
    BaseSubsystem <|-- cpu.MaxValueSubsystem : 实现
    BaseSubsystem <|-- memory.MaxValueSubsystem : 实现
    Subsystem <|-- cgroup.ProcsValueSubsystem : 实现
    Subsystem <|-- cpu.MaxValueSubsystem : 实现
    Subsystem <|-- memory.MaxValueSubsystem : 实现
    Value <|-- cgroup.ProcsValue : 实现（隐含）
    Value <|-- cpu.MaxValue : 实现（隐含）
    Value <|-- memory.MaxValue : 实现（隐含）

    %% 组合/依赖关系
    CgroupManager o-- CgroupFileSystem : 包含
    CgroupManager o-- cgroup.ProcsValueSubsystem : 包含
    CgroupManager o-- cpu.MaxValueSubsystem : 包含
    CgroupManager o-- memory.MaxValueSubsystem : 包含
    CgroupFileSystem <-- newSubsystem : 读写依赖
    Subsystem <-- newSubsystem : 实例化依赖

    classDef typeAlias fill:#f0f8ff,stroke:#2196f3,stroke-width:1px,rounded:8px,font-style:italic;
    classDef interface fill:#fef7fb,stroke:#9c27b0,stroke-width:1.5px,rounded:8px,font-weight:600;
    classDef genericInterface fill:#e8f5e8,stroke:#4caf50,stroke-width:1.5px,rounded:8px,font-weight:600;
    classDef concrete fill:#fff8e1,stroke:#ff9800,stroke-width:1px,rounded:8px;
    classDef external fill:#f5f5f5,stroke:#607d8b,stroke-width:1px,rounded:8px,dashed:true;

```

# 构造镜像

## 什么是构造镜像

“让容器跑在有镜像的环境中”，核心是解决当前容器的两个关键问题 ——**目录环境不隔离**、**挂载点继承父进程**，本质是让容器运行在一个**由镜像提供的、独立且标准化的文件系统环境**中，而非直接复用宿主机的目录和挂载。

### 有镜像的环境

>   先通俗理解：“有镜像的环境” = 容器的 “独立根文件系统”

可以把镜像理解为：**一个打包好的、包含完整操作系统 / 应用运行所需的 “文件系统快照”**；而 “有镜像的环境”，就是让容器不再复用宿主机的 `/` 根目录、不再继承宿主机的挂载点，而是将这个 “文件系统快照” 作为容器自己的 `/` 根目录（即 `chroot`/`pivot_root` 切换根），容器内看到的所有文件、目录、挂载点，都来自这个镜像，和宿主机完全隔离。

>   实现 “有镜像的环境” 的核心步骤

要让容器跑在 “有镜像的环境” 中，核心要加 3 步：

#### 准备镜像

先制作一个简单的镜像，这里我们使用 `busybox` 作为我们的镜像系统：

```shell
docker pull busybox
docker run -d busybox top -b
# 16423fa6038a4ad688f88fa21e38b54ef5a6d357453a2efe90f6d9248e236e81

# 通过docker export将镜像打包成tar
docker export -o busybox.tar 16423fa6038a4ad688f88fa21e38b54ef5a6d357453a2efe90f6d9248e236e81
tar -xvf busybox.tar -C busybox
```

#### 在容器初始化时切换根目录（pivot_root/chroot）

在你 `fork` 出的子进程中（`init` 进程），先执行 `pivot_root` 切换到镜像的根目录，这段代码中做了这些事情： 

1. 将一个新的目录root挂载到了挂载点； 
2. 创建 root/.pivot_root 并执行 pivot_root，执行完成之后，root 变成了我们的参数 root，宿主机的root被挂载到 root/.pivot_root 下；
3. 删除对 root/.pivot_root 的绑定，这个指向了我们的宿主机根目录。删除临时文件。 

至此，我们完成了对文件目录的隔离，进程再也无法访问宿主机的目录了。

```go
func pivotRoot(root string) error {
	// 这一步总的来说就是为了在不改变root本身内容的情况下，将它绑定到挂载点，以便于后续的pivot_root使用。
	// 1. pivot_root 要求新的根目录必须是一个独立的挂载点，而 mount --bind 就是实现这一要求的标准方法。
	// 2. mount --bind source target 是一个特殊的挂载方式，可以把它理解为文件系统级别的硬链接。
	// 		它的作用是将 source 目录的内容，原封不动地 “镜像” 到 target 目录。
	// 3. syscall.MS_BIND: bind 挂载的标志。
	// 4. syscall.MS_REC: 递归（Recursive）标志。这个标志非常重要，它会确保 root 目录下的所有
	//		子挂载点也一同被 bind 挂载过来，保证了新根文件系统的完整性。
	if err := syscall.Mount(root, root, "bind", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
		return fmt.Errorf("Mount rootfs to itself error: %v", err)
	}

	// 创建 rootfs/.pivot_root 存储 old_root
	pivotDir := filepath.Join(root, ".pivot_root")
	if err := os.Mkdir(pivotDir, 0777); err != nil {
		return err
	}

	// 在使用 pivot_root 切换根目录的时候，需要两个目录：
	//
	// 1. new_root，这个是我们切换的目标目录；
	// 2. old_root，这个并不是指的我们的宿主机的根目录，而是一个目录用来mount根目录的。
	if err := syscall.PivotRoot(root, pivotDir); err != nil {
		return fmt.Errorf("pivot_root %v", err)
	}
	// 修改当前的工作目录到根目录
	if err := syscall.Chdir("/"); err != nil {
		return fmt.Errorf("chdir / %v", err)
	}

	pivotDir = filepath.Join("/", ".pivot_root")
	// umount rootfs/.pivot_root
	if err := syscall.Unmount(pivotDir, syscall.MNT_DETACH); err != nil {
		return fmt.Errorf("unmount pivot_root dir %v", err)
	}
	// 删除临时文件夹
	return os.Remove(pivotDir)
}
```

#### 重新初始化容器内的挂载点

在执行 `mount` 命令之前，`/proc` 和 `/dev` 在我们的容器根文件系统中，仅仅是两个**普通的、空的目录**：

- `syscall.Mount("proc", "/proc", "proc", ...)` **明确地**告诉内核：“我要把 `/proc` 这个目录，和 `proc` 类型的虚拟文件系统关联起来。”内核收到指令后，会启动 `proc` 文件系统的驱动程序。这个驱动程序会开始动态地将内核中的进程信息、系统状态等，实时地生成为文件和目录，并 “填充” 到 `/proc` 目录中。
- `syscall.Mount("tmpfs", "/dev", "tmpfs", ...)` **明确地**告诉内核：“我要把 `/dev` 这个目录，和 `tmpfs` 类型的内存文件系统关联起来。”内核创建一个基于内存的临时文件系统，并将其挂载到 `/dev`。

>   注意，通常我们在 `mount` 的时候需要五个参数：
>
>   1. `source`
>   2. `target`
>   3. `fstype`
>   4. `flags`
>   5. `data`
>
>   表示将 `target` 以 `fstype` 的类型挂载到 `source`（其他两个参数我们暂时忽略）。
>
>   而 `proc` 和 `tmpfs` 是虚拟文件系统，它们的数据**不是来自一个物理设备**，而是由**内核动态生成的**。
>
>   - `proc` 的内容是内核中进程、内存等信息的实时映射。
>   - `tmpfs` 的内容存放在内存或交换分区中。
>
>   因此，它们不需要一个传统的 source（物理设备路径）。内核在处理这些文件系统类型时，会直接忽略 source 参数。

```go
func setUpMount() {
	pwd, err := os.Getwd()
	if err != nil {
		log.Errorf("Get current location error %v", err)
		return
	}
	log.Infof("Current location is %s", pwd)
	pivotRoot(pwd)

	//mount proc
	defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
	syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")

	syscall.Mount("tmpfs", "/dev", "tmpfs", syscall.MS_NOSUID|syscall.MS_STRICTATIME, "mode=755")
}
```

需要注意的一点是，**在`Exec()` 或者 `Start()` 的时候，我们可以指定代码的环境变量，为了使得 `busybox` 的命令能正常执行，我们必须指定它的环境变量：**

```go mark:4,5
func NewParentProcess(tty bool, commands []string, env []string) *exec.Cmd {
	// ...
	cmd.Dir = constant.DefaultPwd
    // env = "PATH=/bin"
	cmd.Env = env
	return cmd
}
```

## 镜像的write layer

### COW(Copy-On-Write)

`COW` 是现代软件系统中最为常用的几个性能优化思想之一，在 `linux进程创建`，`LSM数据结构` 等地方都有广泛应用，但是不论是在哪个场景下的应用，他的思想可以最简单被概括为：

1.  **分层划分**：将存储或资源划分为 **只读层（Read Layer）** 和 **可写层（Write Layer）**。
    -   只读层可以是多层（比如容器的基础镜像层 + 业务镜像层），具备**只读、可共享**的特性；
    -   可写层只有一层，是**专属、可修改**的，所有变更都只发生在这里。
2.  **统一视图层（Merge Layer）**：通过一个抽象的 “合并逻辑层”，对外提供**单一、透明的访问入口**，屏蔽底层分层的复杂性。
    -   **读操作**：**自上而下**搜索各层。优先读取可写层的数据；若可写层不存在，则依次向下读取只读层；最终返回找到的第一个结果。
    -   **写操作**：**不修改只读层**。若数据在只读层已存在，先将其**复制到可写层**，再修改可写层的副本；若数据不存在，直接在可写层创建。
    -   **删除操作**：**不删除只读层数据**。而是在可写层创建一个 **“删除标记”**（比如 OverlayFS 的 whiteout 文件），合并视图时会识别这个标记，对外表现为 “文件已删除”。

**这是CoW 在 “分层存储” 场景下的核心形态**，而 Linux 进程 fork 的 CoW 虽然原理一致，但表现形式略有不同（没有显性的 “Merge Layer”，而是通过内核内存页的权限标记 + 复制来实现）。

```mermaid
flowchart BT
    %% 样式定义：区分不同层级和操作
    classDef writeLayer fill:#FFE0B2,stroke:#E65100,stroke-width:2px,rounded:8px,font-weight:600;
    classDef readLayer fill:#E1F5FE,stroke:#0288D1,stroke-width:2px,rounded:8px,font-weight:600;
    classDef operation fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px,rounded:8px,font-weight:600;
    classDef step fill:#E8F5E8,stroke:#2E7D32,stroke-width:1px,rounded:6px;
    classDef arrow stroke:#555555,stroke-width:1.5px;

    %% 1. 分层结构
    subgraph LayerStructure["1. 分层结构"]
        direction BT
        WLayer(可写层 / Write Layer):::writeLayer
        RLayer(只读层 / Read Layer):::readLayer
        RLayer --> WLayer
    end
    class LayerStructure operation

    %% 2. 读操作
    subgraph ReadOperation["2. 读操作 (自上而下)"]
        direction LR
        RStart(读请求):::step
        RCheckWrite{检查可写层?}:::step
        RHitWrite(找到数据<br/>直接返回):::step
        RMissWrite(未找到):::step
        RCheckRead{检查只读层?}:::step
        RHitRead(找到数据<br/>直接返回):::step
        RMissRead(未找到<br/>返回不存在):::step

        RStart --> RCheckWrite
        RCheckWrite -->|"是"| RHitWrite
        RCheckWrite -->|"否"| RMissWrite
        RMissWrite --> RCheckRead
        RCheckRead -->|"是"| RHitRead
        RCheckRead -->|"否"| RMissRead
    end
    class ReadOperation operation

    %% 3. 写操作 (Copy-on-Write)
    subgraph WriteOperation["3. 写操作 (Copy-on-Write)"]
        direction LR
        WStart(写请求):::step
        WCheckWrite{数据在可写层?}:::step
        WUpdate(直接更新<br/>可写层数据):::step
        WMissWrite(数据不存在):::step
        WCheckRead{数据在只读层?}:::step
        WCopy(复制数据到<br/>可写层):::step
        WUpdateNew(在可写层<br/>创建新数据):::step

        WStart --> WCheckWrite
        WCheckWrite -->|"是"| WUpdate
        WCheckWrite -->|"否"| WMissWrite
        WMissWrite --> WCheckRead
        WCheckRead -->|"是"| WCopy
        WCheckRead -->|"否"| WUpdateNew
        WCopy --> WUpdate
    end
    class WriteOperation operation

    %% 整体布局连接
    LayerStructure -.->|为以下操作提供基础| ReadOperation:::arrow
    LayerStructure -.->|为以下操作提供基础| WriteOperation:::arrow
```



### Layer的原理

现在，通过pivot_root，我们已经可以在启动的时候隔离宿主机的文件系统了。现在还存在的一个问题是，我们对镜像的任何变更，都会影响原本的镜像，这个是不正常的。 

在老版本的docker中，通过AUFS实现了一个特性：在启动的时候会启动两个layer，一个write layer和一个container-init layer。现代容器技术（包括新版 Docker）已经不再推荐使用 AUFS，而是采用了更通用、更强大的 ** 存储驱动（Storage Driver）** 模型，其中最核心的技术是 **[Union File System (UnionFS)](#unionfs)** 的各种现代实现。这个特性的实现原理在现代容器中被称为 **Copy-on-Write (写时复制，CoW)**。

这个特性的逻辑可以如下描述：

1.   **只读层 (Read-only Layers)**：
     -   `busybox` 镜像本身就是一个**只读层**。它包含了容器运行所需的所有基础文件和目录；
     -   这个层永远不会被修改。如果多个容器基于同一个镜像启动，它们会共享这同一个只读层，极大地节省了磁盘空间。
2.   可写层 (Writable Layer)：
     -   当我们启动一个容器时，容器运行时会在只读的镜像层之上，**挂载一个新的、可写的层**。
     -   这个可写层通常是一个 `tmpfs`（内存文件系统）或者 `overlayfs` 的上层目录。

而这些特性都是基于 `UnionFs` 以及 `COW` 实现的：

1.   **UnionFS 的作用**：
     -   **UnionFS** 技术（如 `overlay2`, `devicemapper` 等）能够将多个不同的目录（可写层和只读层）“合并” 成一个统一的文件系统视图。
     -   从容器内部看，它只能看到一个统一的 `/` 根目录，但实际上这个目录是由多个层 “联合” 而成的。
2.   **Copy-on-Write (写时复制) 机制**：
     -   **读操作**：当容器需要读取一个文件时，UnionFS 驱动会先在可写层查找。如果找不到，它会继续向下在只读的镜像层中查找，并返回找到的文件。
     -   **写 / 修改操作**：当容器需要创建或修改一个文件时，UnionFS 驱动会：
         -   如果文件不存在：直接在**可写层**创建。
         -   如果文件已存在于只读层：UnionFS 驱动会**先将该文件从只读层复制到可写层**，然后再对可写层中的副本进行修改。原始的只读层文件保持不变。
     -   **删除操作**：删除一个文件时，UnionFS 驱动并不会真的删除只读层里的文件，而是在**可写层创建一个特殊的 “白名单” 文件（whiteout file）**，标记该文件已被删除。这样，在统一视图中，该文件就 “消失” 了。

### Layer的实现

整体的逻辑可以如图描述：

```mermaid
flowchart LR
    subgraph Phase1["父进程（tiny-docker run）"]
        direction TB
        A["解析命令行参数<br/>(-m/-c/镜像路径/用户程序)"]
        B["初始化read/write/work/merge文件夹"]
        C["解压镜像.tar到read文件夹（只读层）"]
        D["创建Cgroup并设置资源限制<br/>(内存/CPU)"]
        E["fork()创建子进程<br/>(带CLONE_NEWNS/PID/UTS等Namespace)"]
        
        A --> B --> C --> D --> E
    end

    subgraph Phase2["子进程（init逻辑）"]
        direction TB
        F["读取父进程传递的路径参数/环境变量"]
        G["初始化UnionFS(OverlayFS)<br/>(mount lowerdir=read,upperdir=write)"]
        H["初始化proc/tmpfs等挂载<br/>(mount proc /proc)"]
        I["pivot_root切换根目录<br/>(根目录指向merge)"]
        J["exec替换为用户进程<br/>(/bin/ash等)"]
        
        F --> G --> H --> I --> J
    end

    subgraph Phase3["容器运行"]
        K["用户进程执行<br/>(/bin/ash/stress等)"]
        L["基于UnionFS读写文件<br/>(写操作落write层)"]
        
        K --> L
    end

    Phase1 -.->|fork创建子进程（带Namespace）| Phase2
    Phase2 -.->|exec覆盖地址空间（无新进程）| Phase3
```



我们的实现思路是：

1.   `UnionFS` 的工作最少需要如下几个文件
     -   只读层
     -   可写层
     -   工作层
     -   对外暴露的挂载点
2.   在启动时，我们可以指定一个 `busybox.tar` 作为输入，这个输入模拟的就是docker中只读的镜像，将 `busybox.tar` 解压到 `/root/tiny-docker/busybox/read` 下；这个对应于我们的只读层；
3.   创建文件夹 `/root/tiny-docker/busybox/write`；这个对应于我们的可写层。
4.   工作层是一个临时文件目录，我们将他放到 `/root/tiny-docker/busybox/work`；
5.   对外暴露的挂载点我们使用 `/root/tiny-docker/busybox/merged`；
6.   执行 `mount -t overlay overlay -o lowerdir=${lower},upperdir=${upper},workdir=${work} ${merged}` ，这条命令的功能是：
     -   `-t overlay` 指定使用 overlay 文件系统；
     -   `overlay` 指定设备名称
     -   `-o lowerdir=${lower},upperdir=${upper},workdir=${work}` 指定了只读层，可写层，工作层；
     -   `${merged}` 指定了挂载点。

>   因为目前我们还没有实现镜像管理，那么我们可以直接使用 `busybox.tar` 作为我们的镜像，我们会将它解压后作为我们的只读层，那么我们的命令就可以修改到如下：
>
>   ```bash mark:5
>   ./tiny-docker run -it \
>                    -e PATH=/bin/ \
>                    -m 2000m \
>                    -c '10000 100000' \
>                    /root/busybox.tar /bin/ash
>   ```

1.   **`tiny-docker run` 启动**：父进程开始执行。
2.   **父进程初始化**：
     -   解析命令行参数。
     -   初始化 Cgroup 配置。
     -   **准备文件系统目录**：确保 `read`, `write`, `work`, `merged` 目录存在。
3.   **`fork()` 创建子进程**：
     -   子进程继承了父进程的资源。
     -   **在子进程中，立即设置 Mount Namespace** (`syscall.Unshare(syscall.CLONE_NEWNS)`)。这一步至关重要，它让子进程的后续挂载操作**不会影响到宿主机和父进程**。
4.   **子进程内部准备**：
     -   **执行 `mount` 命令**：在子进程中，执行 `mount -t overlay overlay -o lowerdir=...,upperdir=...,workdir=... /root/tiny-docker/busybox/merged`。此时，在子进程中 `ls /root/tiny-docker/busybox/merged`，应该能看到 `busybox` 的文件列表。
     -   **执行 `pivot_root`**：现在，将子进程的根目录切换到这个刚刚挂载好的 `merged` 目录。
     -   **执行 `chdir("/")`**：切换当前工作目录到新的根目录。
     -   **清理旧的挂载点**：卸载掉之前的 `merged` 目录的旧挂载点，它现在位于 `.pivot_root` 目录下。
5.   **`exec()` 执行用户程序**：在子进程中，执行 `execve("/bin/ash", ...)`。

#### EnsureDirectoryExists

确保文件存在的实现很简单，因为go已经为我们封装好了所有的函数，我们需要做的是，在初始化container之前去执行对应的逻辑即可。

```go mark:3, 18
func EnsureDirectoryExists(path string) error {
	logrus.Infof("Ensuring directory exists: {%s}", path)
	if err := os.MkdirAll(path, 0755); err != nil {
		logrus.Infof("Failed to create directory %s: %v", path, err)
		return err
	}

	logrus.Infof("Directory %s is ready.", path)
	return nil
}

// init read-layer, write-layer, work-layer, merge-layer for container
func setupLayer() error {
	name := conf.GlobalConfig.Meta.Name
	// ...

	var err error
	if err = util.EnsureDirectoryExists(readPath); err != nil {
		logrus.Errorf("ensure directory error : %s", err.Error())
		return err
	}

    // ...

	return nil
}
```

#### 子进程内部准备

子进程的内部，我们只需要按照前面描述的逻辑去执行即可，但是这里存在一个问题是，我们的逻辑是：通过 exec 创建一个子进程，并切换namespace，那么：

1.   在通过exec创建子进程之后，`GlobalConfig` 不再可用，因为 `exec` 会使用一个全新的内存地址空间替换父进程的内存地址空间；
2.   在子进程中，由于我们切换了`namespace`，它也没有办法读取到我们的配置文件；

为此，我们需要有一个方式来作为父进程和子进程中间传递信息的桥梁，有三个方式：

1.   通过命令行参数传递，它的实现最为简单，但是当命令行过长时维护困难；
2.   通过环境变量传递，他的实现也很简单，并且非常灵活：
     -   比命令行参数更灵活，适合传递多个配置。
     -   信息不会暴露在 ps 输出中，更安全。
     -   仍然不适合传递非常复杂的结构化数据。
     -   环境变量的命名需要全局唯一，以避免冲突。
3.   通过文件描述符传递，它是最健壮也最灵活的方式，但是实现相对比较复杂。

>   这里，我们选用**环境变量传参**的方式来实现我们的逻辑。

先读取我们的配置文件：

```go mark:2,6,10,14,18
type FsConfig struct {
	Root string `yaml:"root"`
}

func (c Config) ReadPath(containerId string) string {
	return filepath.Join(c.Fs.Root, containerId, "read")
}

func (c Config) WritePath(containerId string) string {
	return filepath.Join(c.Fs.Root, containerId, "write")
}

func (c Config) WorkPath(containerId string) string {
	return filepath.Join(c.Fs.Root, containerId, "work")
}

func (c Config) MergePath(containerId string) string {
	return filepath.Join(c.Fs.Root, containerId, "merge")
}
```

随后将它添加到我们的环境变量中

```go mark:2,3,9,10
func Run(commands RunCommands) error {
	parent := container.NewParentProcess(commands.Tty, commands.Args, commands.UserEnv)
	setupEnv(parent)
    // ...
	return nil
}

func setupEnv(cmd *exec.Cmd) {
	util.AppendEnv(cmd, constant.MetaName, conf.GlobalConfig.Meta.Name)
	util.AppendEnv(cmd, constant.FsBasePath, conf.GlobalConfig.Fs.Root)
    // ...
}
```

这里需要注意的是，由于我们的程序是基于 `UnionFS` 上的，所以我们需要修改我们的工作目录：

```go mark:3,4
func NewParentProcess(tty bool, commands []string, env []string) *exec.Cmd {
	// ...
	cmd.Dir = conf.GlobalConfig.MergePath()
	cmd.Env = env
	return cmd
}
```

此时，我们只需要在 `pivot_root` 之前，通过 `UnionFS` 挂载我们的文件系统即可：

```go mark:4-6
func RunContainerInitProcess(command string, args []string) error {
	logrus.Infof("init process command: {%s}, args: {%v}", command, args)
	var err error
	if err = setupUnionFs(); err != nil {
		return err
	}
	logrus.Info("setup layer success.")
	if err = setupMount(); err != nil {
		return err
	}
	// ...
	return nil
}
```

# QA

## Proc

`/proc` 是 Linux 系统中**核心的伪文件系统（pseudo-filesystem）**，由内核实时生成和维护，并非存储在磁盘上的真实文件，而是内核暴露系统 / 进程状态、配置的 “接口”—— 通过读写 `/proc` 下的文件，可查看 / 调整内核、进程、硬件的运行参数，是运维和容器 / 内核开发的核心工具。

### 核心特性

1. **无磁盘存储**：所有文件 / 目录由内核动态生成，占用磁盘空间为 0（`df -h` 查看 `/proc` 挂载点会显示 0 大小）；
2. **实时性**：文件内容随系统 / 进程状态实时变化（如 `/proc/1/status` 会实时反映 PID=1 进程的资源使用）；
3. **权限管控**：大部分文件仅 root 可修改，普通用户仅能读取（部分进程文件仅属主可访问）；
4. **挂载类型**：默认以 `proc` 类型挂载（`mount -t proc proc /proc`），每个 PID Namespace 可挂载独立的 `/proc`（容器隔离的核心）。

### 目录结构

>第一类是进程专属目录：`/proc/$<PID>/`：每个运行的进程对应 `/proc` 下一个以 PID 命名的目录（如 `/proc/1/` 对应 PID=1 的 `systemd` 进程），包含该进程的所有核心信息：

| 关键文件 / 目录       | 作用                                                        |
| --------------------- | ----------------------------------------------------------- |
| `/proc/<PID>/status`  | 进程核心状态（PID、UID/GID、内存、CPU、信号、Namespace 等） |
| `/proc/<PID>/cmdline` | 进程启动的完整命令行（以 `\0` 分隔参数）                    |
| `/proc/<PID>/cgroup`  | 进程所属的 Cgroup 组（v1/v2 路径）                          |
| `/proc/<PID>/ns/`     | 进程所属的所有 Namespace（符号链接）                        |
| `/proc/<PID>/fd/`     | 进程打开的文件描述符（符号链接到真实文件）                  |
| `/proc/<PID>/root`    | 进程的根目录（chroot / 容器的根文件系统）                   |
| `/proc/<PID>/mounts`  | 进程可见的挂载点（文件系统 Namespace 隔离关键）             |

> 第二类是系统级信息目录 / 文件（全局状态）：这类文件 / 目录反映整个系统的运行状态，是运维和开发的核心参考

| 关键文件 / 目录      | 作用                                           |
| -------------------- | ---------------------------------------------- |
| `/proc/cpuinfo`      | CPU 硬件信息（核心数、架构、频率等）           |
| `/proc/meminfo`      | 内存使用状态（总内存、空闲、缓存、交换分区等） |
| `/proc/mounts`       | 系统全局挂载点（等价于 `mount` 命令）          |
| `/proc/sys/`         | 内核参数配置（可动态修改，无需重启）           |
| `/proc/self/`        | 指向当前进程的 PID 目录（符号链接）            |
| `/proc/thread-self/` | 指向当前线程的 TID 目录                        |
| `/proc/uptime`       | 系统运行时间（秒）                             |
| `/proc/net/`         | 网络状态（TCP/UDP 连接、网卡、路由等）         |

> 第三类是特殊文件（内核交互接口）：

| 文件            | 作用                            |
| --------------- | ------------------------------- |
| `/proc/kmsg`    | 内核日志（等价于 `dmesg` 命令） |
| `/proc/loadavg` | 系统负载均值（1/5/15 分钟）     |
| `/proc/version` | 内核版本 + 编译信息             |
| `/proc/cgroups` | Cgroup 子系统状态（v1）         |
| `/proc/cmdline` | 系统启动时的内核参数            |

## Namespace

### Namespace 是什么？

Namespace 本质是 Linux 内核为进程创建的「**资源隔离边界**」—— 不同 Namespace 内的进程，只能看到自己 Namespace 内的资源，无法感知其他 Namespace 或宿主机的同名资源。

例如：

- 宿主机的 PID 1 是 `systemd`，但容器内的 PID 1 可以是 `sh`（PID Namespace 隔离）；
- 容器内修改主机名（`hostname`），宿主机完全不受影响（UTS Namespace 隔离）；
- 容器内的 `eth0` 网卡，和宿主机的 `eth0` 是两套独立网络栈（Network Namespace 隔离）。

### Linux Namespace 的分类

| Namespace 类型 | 标识常量（Go 中 `unix.CLONE_XXX`） | 隔离资源                         | 核心作用                                                     |
| -------------- | ---------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| UTS            | `CLONE_NEWUTS`                     | 主机名、NIS 域名                 | 让容器有独立的主机名（如 `docker run --hostname`）           |
| PID            | `CLONE_NEWPID`                     | 进程 ID 空间                     | 容器内 PID 从 1 开始，隔离进程树                             |
| Mount          | `CLONE_NEWNS`                      | 文件系统挂载点                   | 容器有独立的根文件系统（`/`），挂载 / 卸载不影响宿主机       |
| Network        | `CLONE_NEWNET`                     | 网络栈（网卡、端口、路由）       | 容器有独立的网卡、IP、端口，实现网络隔离（如 `docker run -p`） |
| User           | `CLONE_NEWUSER`                    | 用户 / 组 ID 空间                | 容器内的 root（UID 0）≠ 宿主机的 root，实现权限隔离          |
| IPC            | `CLONE_NEWIPC`                     | 进程间通信（消息队列、共享内存） | 容器内的 IPC 资源仅对本 Namespace 可见                       |
| Cgroup         | `CLONE_NEWCGROUP`（4.6+ 内核）     | Cgroup 根目录                    | 容器有独立的 Cgroup 视图，限制资源更精细                     |

### Namespace 的核心工作原理

创建新进程时，通过 `clone()` 系统调用指定 `CLONE_NEWXXX` 标志，新进程会进入新的 Namespace：

```c
pid_t clone(int (*fn)(void *), void *stack, int flags, void *arg);

clone(child_func, stack, CLONE_NEWUTS | CLONE_NEWPID | SIGCHLD, arg);
```

## cgroup

> `cgroup` 分为 `v1` 和 `v2` 两个版本，我们这里只简单介绍一些 `v2` 版本的实现逻辑。

### cgroup的核心文件

Cgroup v2 的所有操作都基于 `/sys/fs/cgroup`（根 Cgroup 组），根目录核心文件：

| 文件 / 目录              | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| `cgroup.controllers`     | 只读，列出内核支持的所有资源子系统（如 cpu、memory、io、pids、hugetlb 等） |
| `cgroup.subtree_control` | 可写，控制「哪些子系统权限能传递给子组」（子组需父组授权才能配置对应资源） |
| `cgroup.procs`           | 可写，该组内的进程 PID 列表（每行一个，写入 PID 即绑定进程） |
| `cgroup.threads`         | 只读，该组内的线程 TID 列表                                  |
| `cgroup.max.descendants` | 可写，该组允许创建的最大子组数量（默认无限制）               |

### 使用cgroup限制CPU 50% + 内存 1GB

#### 一、启用子系统传递权限

```bash
cd /sys/fs/cgroup

# 启用 cpu + memory 子系统（+ 表示启用，- 表示禁用）
echo "+cpu +memory" > cgroup.subtree_control

cat cgroup.subtree_control
```

我们可以看到输出为：

```
cpuset cpu io memory net_cls hugetlb pids rdma misc
```

| 子系统    | 核心含义            | 管控能力                                           | 典型配置场景                                                | Cgroup v2 核心配置文件                                       |
| --------- | ------------------- | -------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------ |
| `cpuset`  | CPU 集合 / 节点绑定 | 限制进程可使用的**物理 CPU 核心**、NUMA 节点       | 多核服务器上，将进程固定到指定 CPU 核心（避免跨核调度开销） | `cpuset.cpus`（指定 CPU 核心）、`cpuset.mems`（指定 NUMA 节点） |
| `cpu`     | CPU 使用率 / 权重   | 限制 CPU 时间片配额、资源竞争时的优先级            | 限制进程 CPU 使用率 ≤50%，或给核心进程更高 CPU 权重         | `cpu.max`（时间片配额）、`cpu.weight`（竞争权重）            |
| `io`      | 磁盘 IO 管控        | 限制块设备的读写速率、IOPS、优先级                 | 限制容器读写磁盘的速率（如 1MB/s），避免磁盘 IO 抢占        | `io.max`（速率 / IOPS 限制）、`io.weight`（IO 优先级）       |
| `memory`  | 内存 / 交换分区管控 | 限制内存使用上限、交换分区使用、OOM 行为           | 限制容器内存 ≤1GB，禁止使用交换分区                         | `memory.max`（内存上限）、`memory.swap.max`（交换分区上限）  |
| `net_cls` | 网络类别标记        | 为进程网络包打上分类标签（配合 tc/iptables 管控）  | 区分不同进程的网络流量，实现带宽限制 / 优先级               | `net_cls.classid`（打标签）                                  |
| `hugetlb` | 大页内存管控        | 限制进程使用的**大页内存（HugeTLB）** 用量         | 数据库 / 虚拟化场景，限制大页内存占用（避免大页耗尽）       | `hugetlb.2MB.max`（2MB 大页上限）、`hugetlb.1GB.max`（1GB 大页上限） |
| `pids`    | 进程数管控          | 限制 Cgroup 组内可创建的最大进程 / 线程数          | 防止进程无限 fork 导致系统进程耗尽（如挖矿程序、异常服务）  | `pids.max`（最大进程数）                                     |
| `rdma`    | RDMA 资源管控       | 限制进程使用的 RDMA（远程直接内存访问）资源        | 高性能计算 / 存储场景，管控 RDMA 设备的资源占用             | `rdma.max`（RDMA 资源上限）                                  |
| `misc`    | 杂项资源管控        | 管控不归属其他子系统的杂项资源（如 GPU、特殊设备） | 厂商扩展的资源管控（如 NVIDIA GPU 限制）                    | `misc.max`（杂项资源上限）                                   |

#### 二、创建自定义 Cgroup 子组

Cgroup v2 中「创建目录 = 新建子组」，内核会自动为新目录生成该组的规则文件：

```shell
# 创建名为 "my-app" 的子组（管控我的应用进程）
mkdir /sys/fs/cgroup/my-app

# 查看子组自动生成的文件（核心规则文件已出现）
# 关键文件：cpu.max、memory.max、cgroup.procs、cgroup.controllers 等
ls /sys/fs/cgroup/my-app
```

#### 三、配置资源限制规则

```shell
# 配置CPU限额

# 规则文件：cpu.max（格式：[配额(微秒)] [周期(微秒)]）
# 配额：50000μs = 50ms，周期：100000μs = 100ms → 50% 使用率
echo "50000 100000" > /sys/fs/cgroup/my-app/cpu.max
# 验证配置（应输出 50000 100000）
cat /sys/fs/cgroup/my-app/cpu.max

# 配置内存限额

# 规则文件：memory.max（支持 K/M/G/T 单位，或直接写字节数）
echo "1G" > /sys/fs/cgroup/my-app/memory.max

# 验证配置（应输出 1073741824，即 1GB 的字节数）
cat /sys/fs/cgroup/my-app/memory.max
```

#### 四、绑定进程到 Cgroup 子组

将需要管控的进程 PID 写入子组的 `cgroup.procs` 文件，进程立即被该组的规则限制：

```shell
# 1. 先启动一个测试进程（后台无限循环，高 CPU 占用，记录 PID）
while :; do :; done &
# 输出示例：[1] 12345 （12345 是进程 PID）

# 2. 将该进程绑定到 my-app 组
echo 12345 > /sys/fs/cgroup/my-app/cgroup.procs

# 3. 验证绑定结果（应输出 12345）
cat /sys/fs/cgroup/my-app/cgroup.procs
```

#### 五、其他限制

除了CPU和内存，我们还有非常多的限制项目，例如：

1. 磁盘 IO 限制
2. 进程数限制
3. CPU 权重（资源竞争时的优先级）
4. 监控 Cgroup 组的资源使用（只读统计）

## UTS

> `UTS` 是**Unix Time-Sharing System** 的缩写，本质是 Linux Namespace 的一种类型（标识常量 `CLONE_NEWUTS`），专门用于**隔离进程对「主机名（hostname）」和「NIS 域名（domainname）」的视图**。
>
> hostname 和 domainname 被归属到 **UTS Namespace** 下，核心原因是：**二者是 Unix 系统中描述「主机身份」的核心属性，在早期 Unix Time-Sharing System（UTS）规范中被定义为全局系统参数，Linux 内核延续了这一设计并将其纳入同一 Namespace 进行隔离**。

### hostname

`hostname` 是**单台主机的本地标识**，用于在**局域网 / 单机环境**中区分不同设备，相当于给主机起的 “名字”，例如：

- 本地进程间通信（如 `localhost` 对应 `127.0.0.1`）；
- 局域网内设备互访（如通过 `ubuntu-pc` 代替 IP 地址）；
- 容器隔离（每个容器的 `hostname` 独立，由 UTS Namespace 保证）。

### domainname

`domainname` 本质是 **NIS（Network Information Service）域名**，是早期 Unix 系统用于**分布式用户 / 主机管理**的标识，和我们现在常用的 **DNS 域名（如 `baidu.com`）不是一个概念**。

- **作用范围**：NIS 服务集群内，用于统一管理多台主机的用户、密码、主机映射等信息。
- **使用场景**：仅适用于老旧的 NIS 服务架构，**现代 Linux / 容器环境几乎不用**，主要是为了兼容 UTS 标准而保留。

## pivot_root和chroot

`pivot_root` 和 `chroot` 都是为了改变当前进程及其子进程的**文件系统根目录（`/`）**。他们的核心区别可以如下概括：

| 特性               | `chroot` (Change Root)                                       | `pivot_root` (Pivot Root)                                    |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **概念**           | 改变当前进程的根目录视图。                                   | 将整个进程的根目录 “切换” 到一个新的文件系统，并将旧的根目录 “隐藏” 起来。 |
| **安全性**         | **较低**。存在 “逃脱” 的可能。                               | **更高**。几乎不可能从标准环境中逃脱。                       |
| **对旧根的处理**   | **旧根目录仍然可达**。如果在执行 `chroot` 之前，进程持有一个指向旧根目录（如 `/`）的文件描述符（fd），那么它可以通过操作这个 fd 来 “逃脱” chroot 环境。 | **旧根目录被彻底隔离**。`pivot_root` 会将旧的根目录移动到新根目录下的一个临时目录中，并使其成为一个私有的挂载点。原始的根目录对新的根目录进程来说是**不可见且不可访问**的。 |
| **对父进程的影响** | **无影响**。父进程的根目录依然是原来的根目录。`chroot` 只影响当前进程及其子进程。 | **无影响**。同样只影响当前进程及其子进程。                   |
| **使用前提**       | 新的根目录必须是一个**已存在的目录**。                       | 新的根目录必须是一个**独立的挂载点（mount point）**。这意味着你通常需要先执行 `mount --bind newroot newroot` 来将一个目录变成它自身的挂载点。 |

## mount

>       All files accessible in a Unix system are arranged in one big tree, the file hierarchy, rooted at /. These files can be spread out over several devices. The `mount` command serves to attach the filesystem found on some device to the big file tree. Conversely, the umount(8) command will detach it again. The filesystem is used to control how data is stored on the device or provided in a virtual way by network or other services.

### mount的作用

linux的文件系统是一颗以根目录（"/"）作为起始节点的树，任何对文件系统的访问都可以从根目录开始向下搜索得到。 而我们的mount，就是将我们的目标文件作为树上的一个节点，挂载（mount）到这颗树上。 

例如，在最开始我们的树的结构是：

```
/
├── bin
├── usr
├── dev
│   └── ... # 设备文件
├── mnt
│   └── ... # 挂载路径
└── ...
```

假设我们现在有一个移动硬盘，他是一个以ext4文件系统格式化好的文件系统，此时我们没有办法访问它，因为当前linux的文件树上没有硬盘的任何信息，**如果我们想要访问这个移动硬盘，我们就必须先把移动硬盘挂载到linux的目录树**。大概得逻辑可以概括如下：

1. 移动硬盘接入后，Linux 内核识别为物理设备，在 `/dev` 目录下创建设备文件 `/dev/sda1`（这是内核给设备的 “标识”，不是可访问文件的目录），此时目录结构为：

```
/
├── bin
├── usr
├── dev
│   ├── sda1  # 移动硬盘
│   └── ...
├── mnt
│   └── ...   # 挂载路径
└── ...
```

2. `/dev/sda1` 是一个 `设备节点`，我们需要执行 `mount /dev/sda1 /mnt/usb`：

-   `device`：`/dev/sda1`（要接入的 ext4 文件系统，对应移动硬盘）；
-   `mountpoint`：`/mnt/usb`（目录树上的空目录，作为访问硬盘的入口）；

此时，我们才可以访问我们的移动硬盘：

``` bash mark:4,5,7,8,9,10
/
├── bin
├── usr
├── dev
│   ├── sda1           # 移动硬盘，仍为设备文件（不变）
│   └── ...
├── mnt
│   └── usb            # 挂载点（接入了硬盘的 ext4 文件系统）
│       ├── movie.mp4  # 硬盘里的文件
│       ├── photo.jpg
│       └── ...
└── ...
```

### mount的使用

一个标准的mount调用形式如下：

```bash
mount -t type device dir
```

这条命令指示内核将设备上找到的文件系统（类型为 type）挂载到目录 dir。选项 -t type 是可选的。mount 命令通常能够检测到文件系统。默认情况下，挂载文件系统需要 root 权限。

我们也可以只使用 `dir` 作为参数，这种情况下，mount 会搜索 `/etc/fstab` 来挂载。

```bash
mount /dir
```

`/etc/fstab` 类似于配置文件，例如：我们约定好，U盘我们就挂载到 `/mnt/usb`，那么 `/etc/fstab` 中就可能有一条这样的配置。

```
/dev/sdb1  /mnt/usb xfs  defaults  0  2
```

那么，我们下面这两条指令就是等价的：

```bash
mount /mnt/usb
# mount 会在 /etc/fstab 中搜索挂载点为 /mnt/usb 的文件

mount /dev/sdb1 /mnt/usb
```

## UnionFS

**UnionFS（联合文件系统）** 是一种**分层、轻量级**的文件系统技术，核心能力是 **将多个独立的目录（称为 “层”）以只读或读写的方式联合挂载，对外呈现为一个统一的文件系统视图**，而不需要实际合并这些目录的物理数据。

### UnionFS的核心概念

1.  **层（Layer）**

    UnionFS 的核心是**多层目录的联合**，这些目录被分为两类：

    -   **只读层（Read-only Layer）**：作为基础层，内容不可修改，可被多个容器共享（比如 Docker 镜像的基础层、业务层）。
    -   **可写层（Writable Layer）**：作为最上层，所有的写操作（创建、修改、删除文件）都只发生在这一层，不会影响只读层。

2.  **统一视图（Unified View）**

    所有层被联合后，用户看到的是一个**单一的、无缝的文件系统目录树**，无法感知底层的分层结构。

    例如：只读层有 `/bin/ls`，可写层有 `/etc/my.conf`，联合后的视图里就同时存在这两个文件。

3.  **写时复制（Copy-on-Write, CoW）**

    这是 UnionFS 实现**只读层共享 + 可写层隔离**的关键机制，和我们之前讨论的 CoW 思想完全一致：

    -   **读操作**：优先从上层（可写层）读取文件；若上层不存在，则向下遍历只读层，找到后返回。
    -   **写操作**：
        -   新建文件：直接在**可写层**创建，不会影响任何只读层。
        -   修改只读层已有的文件：先把该文件**复制到可写层**，再修改可写层的副本；原始只读层的文件保持不变。
    -   **删除操作**：不会真正删除只读层的文件，而是在**可写层创建一个 “白名单文件（whiteout）”**，标记该文件已被删除；联合视图中会识别这个标记，对外表现为文件不存在。

### UnionFS 的工作流程

假设我们有两个目录：

-   只读层 `lowerdir`：包含 `file1.txt`（内容：`hello`）、`bin/ls`
-   可写层 `upperdir`：初始为空
-   联合挂载点 `merged`：对外的统一视图

1.  **挂载联合文件系统**

    通过挂载命令将 `lowerdir` 和 `upperdir` 联合到 `merged`，此时 `merged` 目录下能看到 `file1.txt` 和 `bin/ls`。

2.  **读操作**

    读取 `merged/file1.txt`：UnionFS 发现可写层没有该文件，于是从只读层读取，返回内容 `hello`。

3.  **写操作（修改只读层文件）**

    修改 `merged/file1.txt` 内容为 `hello world`：

    -   UnionFS 先将 `lowerdir/file1.txt` 复制到 `upperdir`；
    -   然后修改 `upperdir/file1.txt` 的内容；
    -   此时 `merged/file1.txt` 实际指向的是 `upperdir` 里的副本，只读层的 `file1.txt` 依然是 `hello`。

4.  **写操作（新建文件）**

    在 `merged` 下创建 `file2.txt`：文件会直接保存在 `upperdir`，只读层无任何变化。

5.  **删除操作**

    删除 `merged/file1.txt`：UnionFS 不会删除 `lowerdir` 里的文件，而是在 `upperdir` 创建一个 whiteout 文件；此时 `merged` 下看不到 `file1.txt`，但 `lowerdir/file1.txt` 依然存在。

### UnionFS的优势和劣势

-   优势
    -   **节省磁盘空间**：多个容器可以共享同一个只读镜像层，只有可写层是容器专属的，极大降低了重复数据的存储开销。
    -   **实现镜像分层复用**：Docker 镜像的分层构建（`Dockerfile` 的每一行生成一个层）就是基于 UnionFS 思想：每一层都是只读的，上层可以基于下层构建，且可以被多个镜像共享。
    -   **隔离容器变更**：容器的所有修改都局限在可写层，删除容器时只需删除可写层即可，只读镜像层可以保留复用，实现 “镜像只读、容器可写” 的核心需求。
-   劣势
    -   **多层嵌套的性能损耗**：当只读层数量过多时，文件查找需要遍历多层，会带来一定的性能开销；
    -   **内核兼容性**：早期的 UnionFS 不是 Linux 内核原生支持的，需要额外安装补丁；
    -   **不适合大文件频繁修改场景**：对于大文件的频繁修改，CoW 机制需要复制整个文件到可写层，会产生额外的 IO 开销。

### UnionFS中的大文件

>   **不适合大文件频繁修改场景**：对于大文件的频繁修改，CoW 机制需要复制整个文件到可写层，会产生额外的 IO 开销。

这句话，初看其实是有问题的：从操作系统的角度来讲，一个文件可以被拆分为多个`block`，那为什么我们修改大文件时，要把整个文件都复制到可写层而不是将被修改的那个block复制到可写层呢？举个简单的例子，操作系统中的进程复制在最初所有的内存都是共享的，在修改时**操作系统只会复制被修改的内存页数据并修改。**

这是因为这种块粒度的 CoW 是 **Btrfs、ZFS** 这类高级文件系统的实现方式 —— 它们本身就是 “感知块” 的文件系统，因此可以做到精细的块级复制。而 UnionFS、OverlayFS 这类联合文件系统的定位是 **“用户态目录的联合”**，它们的设计目标是**简单、轻量、兼容所有底层文件系统**（ext4、xfs 等），而非深度介入块管理。它们的 CoW 逻辑是**基于文件的**，原因有两点：

-   **不感知底层块结构**：UnionFS 只处理目录和文件的 “视图合并”，它不知道一个文件的哪些 block 被修改了 —— 对它来说，只要文件被打开并写入，就判定为 “文件需要 CoW”。
-   **实现复杂度低**：块粒度 CoW 需要跟踪文件的每一个 block 的归属（属于只读层还是可写层），这会引入大量的元数据管理开销；而文件粒度 CoW 只需要记录 “这个文件是否被复制过”，逻辑简单。

## 为什么需要pivot_root

在我们的代码逻辑中，我们创建进程时已经指定了 `unix.CLONE_NEWNS`

```go mark:7
func NewParentProcess(tty bool, commands []string, env []string) *exec.Cmd {
	// ...
	cmd := exec.Command(constant.UnixProcSelfExe, args...)
	cmd.SysProcAttr = &unix.SysProcAttr{
		Cloneflags: unix.CLONE_NEWUTS |
			unix.CLONE_NEWPID |
			unix.CLONE_NEWNS |
			unix.CLONE_NEWNET |
			unix.CLONE_NEWIPC,
		Unshareflags: unix.CLONE_NEWNS,
	}
	// ...
}
```

而在我们的用户进程执行之前，我们又执行了一次 `pivot_root`：

```go mark:8,9,10
func pivotRoot(root string) error {
	// ...
	pivotDir := filepath.Join(root, ".pivot_root")
	if err := os.Mkdir(pivotDir, 0777); err != nil {
		return constant.ErrMountRootFS.Wrap(err)
	}

	if err := syscall.PivotRoot(root, pivotDir); err != nil {
		return constant.ErrMountRootFS.Wrap(err)
	}

    // ...
	return os.Remove(pivotDir)
}
```

这是因为：

-   **unix.CLONE_NEWNS**只是让子进程拥有**私有 Mount Namespace**（挂载视图），但**初始时完全继承父进程的挂载树**—— 子进程既可以访问父进程的所有文件系统路径，也可以执行 `mount/umount` 操作（这些操作仅影响自己的挂载视图，不会污染父进程 / 宿主机）；
-   **pivot_root/chroot**：这才是让子进程「脱离父进程目录树」的关键操作 —— 执行后，子进程的根目录 `/` 被切换，才会无法访问宿主机的原始路径；

`CLONE_NEWNS` 的核心作用是**隔离挂载操作**，而非**隔离路径访问**：

| 操作                          | 子进程（CLONE_NEWNS）        | 父进程（宿主机） |
| ----------------------------- | ---------------------------- | ---------------- |
| 访问 `/root/tiny-docker/read` | ✅ 可以访问（初始挂载树继承） | ✅ 可以访问       |
| 执行 `mount overlay ...`      | ✅ 仅修改自己的挂载视图       | ❌ 无任何影响     |
| 执行 `umount /tmp`            | ✅ 仅卸载自己的 `/tmp`        | ❌ `/tmp` 仍可用  |
| 执行 `pivot_root merge`       | ✅ 根目录切换到 merge         | ❌ 无影响         |
