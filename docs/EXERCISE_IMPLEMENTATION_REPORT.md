# ch6-T1L4 Exercise 实现报告（linkat / unlinkat / fstat）

本文对应 `exercise.md`，说明当前 `tg-rcore-tutorial-ch6-T1L4` 中硬链接相关功能的实现思路、代码落点与行为语义。

## 1. 练习目标

chapter6 练习要求实现三个 syscall：

1. `linkat`（ID 37）：为已有文件创建新硬链接；
2. `unlinkat`（ID 35）：删除一个路径到 inode 的链接；
3. `fstat`（ID 80）：通过文件描述符写回文件状态结构体 `Stat`。

目标是建立“目录项 ↔ inode”的链接计数语义，并与进程 fd 表、用户地址翻译机制协同。

---

## 2. 代码落点

- `src/main.rs`：`impl IO for SyscallContext` 中实现 syscall 入口语义；
- `src/fs.rs`：`FS.link` / `FS.unlink` 作为内核文件系统接口；
- `src/process.rs`：`fd_table` 提供 `fstat` 查询对象来源；
- `tg-easy-fs`（本地可改时）：inode/link count 元数据维护。

---

## 3. linkat 实现思路

## 3.1 参数处理

- 忽略 `olddirfd/newdirfd/flags`（实验固定兼容参数）；
- 通过当前进程地址空间读取 `oldpath/newpath` 的用户字符串；
- 路径解析可复用 `open` 的用户字符串读取方式。

## 3.2 语义检查

- 若 `oldpath == newpath`，返回 `-1`；
- 若源文件不存在，返回 `-1`；
- 其他错误按 `-1` 处理。

## 3.3 文件系统动作

- 调用 `FS.link(src, dst)`；
- 在 easy-fs 中新增目录项指向同一 inode，并更新 `nlink`；
- 成功返回 `0`。

---

## 4. unlinkat 实现思路

## 4.1 参数处理

- 忽略 `dirfd/flags`；
- 通过用户地址翻译读取路径字符串。

## 4.2 语义检查

- 文件不存在返回 `-1`；
- 存在则执行 unlink。

## 4.3 文件系统动作

- 调用 `FS.unlink(path)`；
- 删除目录项并减少 inode 链接计数；
- 当 `nlink` 归零时回收 inode 与数据块；
- 成功返回 `0`。

---

## 5. fstat 实现思路

## 5.1 入参与对象定位

- 通过 `fd` 从当前进程 `fd_table` 获取目标文件句柄；
- `fd` 无效或空槽返回 `-1`。

## 5.2 用户指针写回

- 将用户 `st` 指针经 `translate::<Stat>(..., WRITEABLE)` 转换；
- 若不可写或不可见返回 `-1`。

## 5.3 状态填充

按题目定义写入：

- `dev = 0`；
- `ino` 为 inode 编号；
- `mode` 为目录或普通文件；
- `nlink` 为当前硬链接计数。

成功返回 `0`。

---

## 6. 与 ch6 架构的集成要点

- 这些 syscall 不改变调度模型，仍走普通 `UserEnvCall` 分发路径；
- 与 `open/close/read/write` 共用 fd_table 与地址翻译机制；
- 对 inode 元数据的真实修改落在 easy-fs 层，内核层保持接口清晰。

---

## 7. 验证方式

在 `tg-rcore-tutorial-ch6-T1L4` 目录执行：

```bash
cargo run --features exercise
```

按提示在终端输入：

```text
tg-rcore-tutorial-ch6_usertest
```

或运行：

```bash
./test.sh exercise
```

重点验证：

- `linkat` 创建后同文件多路径可访问；
- `unlinkat` 后链接计数变化与最终回收行为正确；
- `fstat` 返回的 `nlink/ino/mode` 与文件状态一致。

---

## 8. 已知边界

当前实现遵循教程实验目标，保持最小可用：

- 路径与目录语义简化；
- 错误码和边界行为对齐测例优先；
- 更复杂 POSIX 兼容细节可在后续版本扩展。