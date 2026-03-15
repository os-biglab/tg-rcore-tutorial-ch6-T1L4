# ch6-T1L4 软件架构总览

本文描述 `tg-rcore-tutorial-ch6-T1L4` 作为独立 crate 的实现结构、执行路径与模块分工。

## 1. 系统定位

`ch6-T1L4` 是运行在 RISC-V S 态的 `no_std` 裸机内核样例，在 ch5 进程管理基础上引入“可持久化文件系统 + 块设备 I/O”。

该 crate 的核心能力：

- 通过 VirtIO MMIO 驱动访问磁盘镜像（`fs.img`）；
- 基于 `easy-fs` 提供 inode 文件系统能力；
- 为每个进程维护文件描述符表（`fd_table`）；
- 支持文件相关 syscall（`open/close/read/write`，以及练习中的 `linkat/unlinkat/fstat`）；
- 用户程序不再内嵌内核，而是按文件名从文件系统动态加载。

与 ch3/ch4/ch5 的主要差异：

- ch3/ch4/ch5 主要围绕任务/进程/地址空间；
- ch6 在此之上新增“磁盘 -> 块设备 -> 文件系统 -> 进程 fd 表”的完整 I/O 路径。

---

## 2. 目录与模块职责

```text
tg-rcore-tutorial-ch6-T1L4/
├── .cargo/config.toml         # 目标平台、QEMU runner（含磁盘挂载）
├── build.rs                   # 生成 linker.ld，并构建文件系统镜像内容
├── Cargo.toml                 # crate 元信息、features、依赖
├── README.md                  # 章节说明文档
├── exercise.md                # 练习要求（硬链接相关 syscall）
└── src/
    ├── main.rs                # 启动、内核映射、调度、syscall 实现
    ├── fs.rs                  # easy-fs 全局管理与 read_all
    ├── process.rs             # Process 定义（含 fd_table）
    ├── processor.rs           # PROCESSOR + ProcManager
    └── virtio_block.rs        # VirtIO 块设备与 Hal 适配
```

---

## 3. 分层架构

```text
用户程序（磁盘文件中的 ELF）
      │
      ▼
syscall 接口层（main.rs::impls）
      │
      ▼
进程层（process.rs：地址空间 + fd_table）
      │
      ▼
文件系统层（fs.rs：FSManager/easy-fs）
      │
      ▼
块设备层（virtio_block.rs：BlockDevice + VirtioHal）
      │
      ▼
QEMU VirtIO MMIO + fs.img
```

### 3.1 `main.rs`：系统编排中心

负责：

1. 初始化内核堆与内核地址空间；
2. 新增 MMIO 区域映射（VirtIO 设备寄存器）；
3. 初始化 syscall 子系统与调度循环；
4. 从文件系统读取 `initproc` ELF，创建首进程；
5. Trap 后按 syscall 类型驱动进程状态迁移。

### 3.2 `fs.rs`：文件系统接入层

- 定义全局 `FS`（lazy 初始化）；
- 用 `EasyFileSystem::open(BLOCK_DEVICE)` 绑定块设备；
- 提供 `open/find/readdir/link/unlink`；
- 提供 `read_all()` 供 `exec/spawn` 读取完整 ELF 数据。

### 3.3 `process.rs`：进程+文件描述符模型

在 ch5 `Process` 基础上新增：

- `fd_table: Vec<Option<Mutex<FileHandle>>>`；
- `from_elf` 初始化 `fd 0/1/2` 为标准输入输出错误；
- `fork` 复制地址空间的同时继承文件描述符表；
- `exec` 替换映像但保留进程对象语义。

### 3.4 `virtio_block.rs`：块设备驱动层

- 用 `virtio-drivers` 的 `VirtIOBlk` 绑定 MMIO 地址 `0x10001000`；
- 实现 `tg_easy_fs::BlockDevice`，把文件系统读写下沉到块设备；
- 实现 `VirtioHal` 负责 DMA 分配与地址转换。

---

## 4. 关键执行路径

### 4.1 启动路径

1. `rust_main` 初始化内核；
2. `kernel_space()` 完成内核段、堆、传送门、MMIO 映射；
3. `FS.open("initproc") + read_all()` 读取初始程序；
4. `Process::from_elf` 创建 init 进程并交给调度器。

### 4.2 文件 I/O 路径

用户 syscall `read/write/open/close`：

- 用户指针先经 `translate()` 权限检查；
- `fd_table` 判断标准流或普通文件；
- 普通文件通过 `FileHandle` 调用 easy-fs；
- easy-fs 进一步调用 `BLOCK_DEVICE` 进行块读写。

### 4.3 程序加载路径

`exec/spawn` 不再从内存表取 app，而是：

1. 通过路径字符串定位文件；
2. 从 FS 读取 ELF 字节；
3. `ElfFile::new` 解析后创建/替换进程地址空间。

---

## 5. 系统调用分工（ch6 重点）

- IO：`read/write/open/close`（fd 表驱动）；
- Process：`fork/exec/spawn/wait/exit/getpid/sbrk`；
- Scheduling：`sched_yield/set_priority`；
- Clock：`clock_gettime`；
- Memory：`mmap/munmap`；
- FS exercise：`linkat/unlinkat/fstat`。

---

## 6. 配置与依赖

关键依赖：

- `tg-easy-fs`：文件系统实现；
- `virtio-drivers`：VirtIO 块设备驱动；
- `tg-kernel-vm`：地址空间与页表；
- `tg-task-manage`：进程管理；
- `tg-syscall`：syscall 分发。

---

## 7. 当前实现边界

当前实现面向教程最小闭环，仍有边界：

- 文件系统功能简化（单级目录语义）；
- 缓存与并发策略简化；
- 错误恢复与崩溃一致性未做完整工程化。

但已经形成“进程 + 文件系统 + 块设备 + 磁盘镜像”的完整教学链路。