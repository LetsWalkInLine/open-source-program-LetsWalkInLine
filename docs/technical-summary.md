# 开源奖励计划技术总结报告

作者：`LetsWalkInLine`

目标仓库：[`rcore-os/tgoskits`](https://github.com/rcore-os/tgoskits)

报告时间：2026-05-29

## 1. 摘要

本次成果主要由 7 个已合并的 `rcore-os/tgoskits` PR 组成，覆盖 StarryOS 的信号测试、futex / robust-list Linux ABI、nginx 运行兼容性、cgroup2 初始支持与层级目录能力。另有 1 个公开 Issue 用于持续追踪后续 cgroup 工作。

这些贡献的共同特点是：先用用户态或组件级测试固化可观察行为，再做最小语义修复，并在 PR 中明确说明未覆盖范围。报告不会把当前工作描述成“完整支持 Docker / nginx / futex / cgroup”，而是按已合并代码实际达到的能力边界进行总结。

截至 2026-05-29，`tgoskits` 侧 7 个 PR 全部合并，GitHub 统计合计：

| 指标 | 数值 |
| --- | ---: |
| 已合并 PR | 7 |
| 提交数 | 17 |
| 变更文件 | 50 |
| 新增行 | 4007 |
| 删除行 | 224 |

这些统计包含测试、QEMU 配置和必要的支撑代码，不能简单等价为“核心功能代码规模”。本报告重点解释每个 PR 实际解决了什么问题、怎么验证，以及哪些能力仍未实现。

## 2. 贡献清单

### 2.1 tgoskits PR

| PR | 合并日期 | 主题 | 变更规模 | 主要验证 |
| --- | --- | --- | --- | --- |
| [#468](https://github.com/rcore-os/tgoskits/pull/468) | 2026-05-09 | `starry-signal` restore 测试跨架构假设修复 | 3 files, +12/-4 | `cargo fmt`、`clippy`、`starry-signal` 测试、aarch64 定向验证 |
| [#545](https://github.com/rcore-os/tgoskits/pull/545) | 2026-05-13 | futex wait / requeue / robust-list owner death 修复 | 10 files, +874/-45 | 四架构 grouped syscall，`test-futex-robust-list` 64 pass |
| [#657](https://github.com/rcore-os/tgoskits/pull/657) | 2026-05-18 | futex / robust-list 基础语义继续完善 | 4 files, +802/-93 | 四架构 syscall QEMU，`test-futex-robust-list` 146 pass |
| [#796](https://github.com/rcore-os/tgoskits/pull/796) | 2026-05-22 | socket `FIOASYNC` 与 `F_SETOWN` 支持 | 7 files, +457/-9 | `bug-nginx-fioasync`、相关 ioctl 回归 |
| [#869](https://github.com/rcore-os/tgoskits/pull/869) | 2026-05-25 | TCP socket `FIONREAD` 支持与 nginx smoke | 11 files, +866/-2 | 四架构 bugfix 测例、nginx smoke 阻塞点验证 |
| [#989](https://github.com/rcore-os/tgoskits/pull/989) | 2026-05-28 | cgroup2 初始 root 文件支持 | 11 files, +391/-0 | 四架构 `cgroup-basic`，11 pass |
| [#1015](https://github.com/rcore-os/tgoskits/pull/1015) | 2026-05-29 | cgroup2 hierarchy `mkdir` / `rmdir` | 4 files, +605/-71 | 四架构 `cgroup-basic`，43 pass |

### 2.2 社区相关 axcpu PR

[`arceos-org/axcpu#28`](https://github.com/arceos-org/axcpu/pull/28) 和 [`arceos-org/axcpu#30`](https://github.com/arceos-org/axcpu/pull/30) 是同一 RISC-V FP/SIMD 上下文切换修复分别合入 `dev` 与 `main` 的记录。该贡献不属于比赛要求中的 `tgoskits` 主仓库成果，但与 StarryOS 下游运行问题相关，因此在本报告后文单独说明。

## 3. 技术路线和工作方法

本次贡献的基本方法是：

1. 用最小可复现测试锁定用户态可观察语义。
2. 对照 Linux ABI 或项目已有实现意图，避免仅修复表面错误码。
3. 将修复范围限制在当前问题需要的模块内。
4. 明确记录未实现范围，尤其是 futex PI、完整 nginx 兼容、完整 cgroup controller、Docker 全链路等高风险主题。
5. 对关键路径使用 QEMU 多架构验证，避免 x86_64 下通过但其他架构失败。

这条路线在 futex、nginx 和 cgroup 工作中比较明显：先新增 focused C 测例或 QEMU case，再实现最小语义，最后在 PR 描述里写出不覆盖的能力边界。

## 4. 具体贡献分析

### 4.1 `starry-signal` 跨架构 restore 测试修复：PR #468

PR 链接：[#468](https://github.com/rcore-os/tgoskits/pull/468)

该 PR 处理的是 `starry-signal` restore 相关测试中的跨架构建模问题。原测试在调用 `thr.restore(&mut uctx)` 前无条件执行 `sp += 8`。这个行为只符合 x86_64 handler 返回时通过栈上 `restorer` 和 `ret` 消费 8 字节的情况；对 aarch64、riscv64、loongarch64 等架构并不成立。

实际问题不是生产代码把 ABI 做错，而是测试把 x86_64 的特殊返回路径当作所有架构的通用行为。旧测试会让 aarch64 下 `restore()` 看到错误的 `SignalFrame` 起点，导致稳定失败。

该 PR 的修复方式比较克制：

- 提取 `prepare_restore_context()`。
- 仅在 `target_arch = "x86_64"` 时调整 `sp`。
- 其他架构保持 `sp` 指向真实 `SignalFrame` 起始地址。
- `api_thread::restore` 和 `concurrent_check_signals` 共用同一 helper。

验证上，PR 描述中记录了常规 `cargo fmt`、`cargo xtask clippy --package starry-signal`、`cargo test -p starry-signal`，以及 aarch64 musl + QEMU runner 的定向验证。维护者后续也确认 `cargo fmt --check`、`clippy`、`starry-signal` 测试和多个 no_std lib 目标通过。

该 PR 的价值在于修正测试与架构 ABI 的对应关系，避免后续开发者继续被“随机失败”和“aarch64 稳定失败”两个问题混淆。

### 4.2 futex / robust-list 第一阶段修复：PR #545

PR 链接：[#545](https://github.com/rcore-os/tgoskits/pull/545)

该 PR 修复 StarryOS futex / robust-list 的多个 Linux 兼容性问题，并新增 `test-futex-robust-list` 用户态 syscall 回归测试。测试使用 C 语言 raw `syscall()`，避免依赖 glibc robust mutex 封装，从而直接验证 StarryOS syscall ABI。

该 PR 覆盖的核心问题包括：

- `FUTEX_WAIT` 超时或中断后可能留下 stale waiter。
- `FUTEX_REQUEUE` 后 waiter 身份在目标队列中可能和其他 waiter 冲突。
- `FUTEX_WAIT_BITSET` / `FUTEX_WAKE_BITSET` 未拒绝 `val3 == 0`。
- robust-list owner death 使用内核内部 `owner_dead` 状态，而不是按 Linux ABI 更新用户态 futex word。

修复方案包括：

- 使用 `Arc<WaiterState>` 作为 waiter 身份。
- wait future 在 timeout、interruption、drop 时只清理自己的队列项。
- wake 路径跳过并清理 cancelled waiter。
- bitset futex 操作对 `value3 == 0` 返回 `EINVAL`。
- robust-list owner death 写回用户态 futex word：设置 `FUTEX_OWNER_DIED`，清除 owner TID bits，并唤醒一个 waiter。
- 移除旧的 `FUTEX_WAIT` 返回 `EOWNERDEAD` 的内部标记路径，使行为更接近 Linux robust-list ABI。

新增测试覆盖了 futex wait/wake、timeout、fork 共享 futex、bitset、set/get robust list、owner death 和 `FUTEX_REQUEUE` waiter identity collision。最终 focused case 在四个架构上通过，日志中 `/usr/bin/test-futex-robust-list` 达到 `64 pass, 0 fail`。

该 PR 没有宣称完整 futex 支持。PI futex、`FUTEX_WAKE_OP` 等复杂语义未覆盖，这是合理边界。

### 4.3 futex / robust-list 基础语义继续完善：PR #657

PR 链接：[#657](https://github.com/rcore-os/tgoskits/pull/657)

PR #657 是 #545 的后续，继续完善 futex 与普通 robust-list owner death 的基础语义。该 PR 将 `test-futex-robust-list` 扩展到 146 项，并继续使用 raw syscall 验证 ABI。

主要实现变化包括：

- 新增 `FutexCommand` / `ParsedFutexOp`，集中解析 command、private flag、clock realtime flag 和非法 option bits。
- 显式支持 `FUTEX_PRIVATE_FLAG`，同时保留未带 private flag 时基于 VMA backend 的 shared mapping 自动识别。
- `FUTEX_WAIT` 保持相对 timeout，`FUTEX_WAIT_BITSET` 使用绝对 deadline，并按 realtime / monotonic 选择时钟。
- wake / requeue 路径补充 futex word 地址、对齐、可访问性、负数 count 和 `uaddr2` 校验。
- `CMP_REQUEUE` 的 compare 与 wake/requeue 在队列锁内序列化。
- requeue 后更新 waiter cleanup 元数据，并让 cleanup token 持有目标 `Arc<FutexTable>`，避免目标 futex table 生命周期问题。
- robust-list pending entry 指向 head 时跳过；遇到坏链或环链达到 limit 后停止扫描并正常返回，避免退出路径打印 warning。

测试新增和增强了：

- 未知 command、未知 option、`FUTEX_CLOCK_REALTIME` 合法/非法组合。
- 地址、对齐、不可访问地址、负数 count 等参数校验。
- private/shared key 的跨进程差异。
- `FUTEX_CMP_REQUEUE` compare mismatch、wake + requeue 顺序和 target wake。
- robust-list 查询其他 pthread、pending-only、pending 已在链表、owner TID mismatch、坏链/环链。

最终验证记录包括 `cargo fmt`、`cargo xtask clippy --package starry-kernel`，以及 x86_64、riscv64、aarch64、loongarch64 的 QEMU syscall group。PR 明确列出后续仍未实现 PI futex、`FUTEX_WAKE_OP`、futex2、完整 signal interruption 语义、shared futex key 完整审计、robust-list ptrace 权限和 `execve` 生命周期等内容。

这部分贡献的实际意义是：把 futex / robust-list 从“能覆盖少量 happy path”推进到“有明确 ABI 解析、错误码和取消/requeue 生命周期基础”的状态。

### 4.4 nginx worker channel 所需 `FIOASYNC` 支持：PR #796

PR 链接：[#796](https://github.com/rcore-os/tgoskits/pull/796)

该 PR 来自 StarryOS 运行 nginx 的适配排查。nginx worker 启动时会通过 `socketpair(AF_UNIX, SOCK_STREAM)` 创建 worker channel，并在 channel fd 上设置非阻塞、异步通知和 owner。StarryOS 缺少 `ioctl(FIOASYNC)` 支持时，nginx 初始化路径会在用户态直接失败。

PR 新增 `bug-nginx-fioasync` C 测例，覆盖：

- nginx worker channel 最小复现：`socketpair` 后执行 `FIONBIO` 与 `FIOASYNC`。
- `FIOASYNC` 与 `O_ASYNC` / `FASYNC` 状态一致性。
- 关闭 async 时不能误清除 `O_NONBLOCK`。
- `F_SETFL` 设置/清除 `O_ASYNC` 时保持 `O_NONBLOCK`。
- `FIOASYNC` 后的 `F_SETOWN`、`F_SETFD(FD_CLOEXEC)` 和 `F_GETFD`。
- TCP listen socket 上的 `FIOASYNC` on/off。
- 空指针 `EFAULT` 和非法 fd `EBADF`。

实现层面，该 PR 做的是 nginx 当前路径需要的最小语义：

- 在 `sys_ioctl` 中增加 `FIOASYNC` 分支，并按 Linux 语义从用户态 `int *` 读取开关。
- 在 socket open file description 上保存 async 状态。
- `F_GETFL` 根据 socket async 状态返回 `FASYNC`。
- `F_SETFL` 识别 async 位并同步更新 socket 状态，同时保留 `O_NONBLOCK`、`O_APPEND` 逻辑。
- 为 socket 添加 `F_SETOWN` / `F_GETOWN` owner 状态。
- `FileLike` 默认仍不声明支持 async，避免普通文件和目录被本次修复误标为支持。

PR 明确没有实现真实 `SIGIO` 投递，也没有把普通文件、目录 fd 的 `FIOASYNC` 纳入修复范围。这一点很重要，因为 nginx 初始化需要的是 fd flag / owner 状态可设置，而不是完整异步信号通知机制。

### 4.5 TCP socket `FIONREAD` 和 nginx smoke：PR #869

PR 链接：[#869](https://github.com/rcore-os/tgoskits/pull/869)

该 PR 继续来自 nginx smoke 排查。基础 nginx 链路已经能完成安装、配置检查、单进程启动、master/worker、reload/quit、静态 GET/HEAD、sendfile 大文件、Range、小 POST 和多次短连接。异常集中在 sendfile 配置下的超限 POST：客户端发送 8 KiB body 时没有得到预期 413，而是 empty reply。nginx error log 显示 `ioctl(FIONREAD) failed (25: Not a tty) while waiting for request`。

该 PR 新增 `bug-nginx-fionread-socket` C 测例，使用 loopback TCP 连接验证 Linux `ioctl(FIONREAD)` 关键语义：

- listening TCP socket 即使存在 pending connection，也返回 `EINVAL`。
- accepted socket 初始 `FIONREAD` 成功并返回 0。
- client 写入 payload 后，server 端返回完整未读字节数。
- server 部分读取后返回剩余未读字节数。
- 读空后再次返回 0。
- `arg == NULL` 返回 `EFAULT`。

实现上，PR 在 StarryOS socket ioctl 中新增 `FIONREAD` 分支，并在 `ax-net-ng` 的 `SocketOps` 中增加 `recv_available() -> AxResult<usize>`。TCP stream 通过 smoltcp `recv_queue()` 暴露 receive queue 字节数；listening TCP socket 返回 `EINVAL`。实现还避免了热路径上无条件 `poll_interfaces()`，只在当前队列为空时轮询接口。

该 PR 还新增了 `nginx-smoke` 端到端测试入口，用真实 Alpine nginx / curl 覆盖配置检查、启动、reload/quit、GET/404/HEAD、sendfile、Range、小 POST、超限 POST 和短连接循环。PR 描述说明：修复前超限 POST 触发 `FIONREAD` 的 `ENOTTY` 并导致 empty reply；修复后该探针能返回 413，说明 TCP socket `FIONREAD` 覆盖了原始阻塞点。

边界也写得很清楚：没有实现 UDP、RAW、AF_UNIX datagram 的 `FIONREAD`；没有实现 Linux AIO syscall family；没有重构网络栈读路径；nginx smoke 后续仍暴露 keep-alive 两连请求的独立问题。这些都不是本 PR 的完成项。

### 4.6 cgroup2 初始 root 文件支持：PR #989

PR 链接：[#989](https://github.com/rcore-os/tgoskits/pull/989)

PR #989 是 StarryOS cgroup 支持的第一阶段提交。它的目标不是一次实现 Docker 或 systemd 可用的完整 cgroup，而是先落地可验证的最小 cgroup2 语义。

该 PR 覆盖：

- 新增 `cgroup-basic` 测试用例，固化当前行为和期望行为。
- 支持 `mount("none", target, "cgroup2", 0, NULL)`。
- 暴露 root cgroup 下的 `cgroup.procs`、`cgroup.controllers`、`cgroup.subtree_control`。
- `cgroup.procs` 能读到当前活跃进程 PID 列表。
- 在没有真实 controller 前，`cgroup.controllers` 和 `cgroup.subtree_control` 保持为空。
- 未实现写操作返回明确错误。
- cgroup v1 `mount -t cgroup` 继续返回 `ENODEV`。

实现模块包括：

- `os/StarryOS/kernel/src/cgroup/mod.rs`：提供 root cgroup 最小状态入口。
- `os/StarryOS/kernel/src/pseudofs/cgroup.rs`：实现最小 cgroup2 pseudo filesystem。
- `os/StarryOS/kernel/src/syscall/fs/mount.rs`：接入 `fs_type == "cgroup2"`。
- `lib.rs` 和 `pseudofs/mod.rs` 注册模块。

测试位于 `test-suit/starryos/normal/qemu-smp1/cgroup-basic`，覆盖 mount、root 文件读取、当前 PID 可见、controller 文件为空、写 `+pids` 返回 `EINVAL`、cgroup v1 mount 返回 `ENODEV`。四个平台 QEMU 均通过，guest 输出 `DONE: 11 pass, 0 fail`。

该 PR 有意不实现 cgroup 层级、controller 注册、进程迁移、`/proc/[pid]/cgroup`、pids/cpu/memory controller、cgroup namespace、`clone3.cgroup` 或 `CLONE_INTO_CGROUP`。这个边界对于避免“假的 Docker 支持”很关键。

### 4.7 cgroup2 hierarchy `mkdir` / `rmdir`：PR #1015

PR 链接：[#1015](https://github.com/rcore-os/tgoskits/pull/1015)

PR #1015 基于 #989 继续推进 cgroup 支持。#989 只提供 root 基础接口文件，用户态还不能通过 `mkdir` / `rmdir` 维护 cgroup v2 层级。#1015 实现了 hierarchy 的基础能力，为后续 `cgroup.procs` 迁移、membership、`/proc/[pid]/cgroup` 等工作铺路。

实现层面，该 PR 做了三类工作。

第一，cgroup core：

- 新增全局 cgroup tree。
- 维护 cgroup id、父子关系、child 名称映射和后续 membership 删除保护预留字段。
- 提供 root id、child names、child lookup、create child、remove child、path 等 API。
- `create_child()` 支持创建 child，重复创建和 interface file 名称冲突返回 `EEXIST`。
- `remove_child()` 删除空 child；目录含 child 时返回 `ENOTEMPTY`；预留 populated cgroup 的 `EBUSY` 检查。
- root `cgroup.procs` 继续读取当前进程表，非 root 暂时为空。

第二，cgroup2fs VFS 层：

- 从静态 `SimpleDir + DirMapping` 改为动态 `CgroupDir`。
- 每个 cgroup 目录动态注册 `cgroup.procs`、`cgroup.controllers`、`cgroup.subtree_control`。
- `lookup`、`mkdir`、`rmdir` 都通过全局 cgroup tree 处理。
- `read_dir()` 返回 `.`、`..`、三个 interface files 和 child cgroup 列表。
- `CgroupDir::is_cacheable()` 返回 `false`，避免动态目录使用 per-instance dentry cache。
- `has_children()` 只统计真实 child cgroup，避免 interface files 阻塞空目录 `rmdir`。
- 普通文件创建、hard link、symlink、rename、interface file 删除返回 `EPERM`。

第三，通用 VFS 支撑：

- 在 `DirNodeOps` 新增默认 `has_children()` hook。
- `DirNode::has_children()` 改为调用后端 hook。
- `lookup_locked()` 对 non-cacheable 目录直接调用后端 `lookup()`。
- `create()` / `link()` 只在 cacheable 目录填充 per-instance cache，避免跨句柄删除后的 stale child dentry。

`cgroup-basic` 扩展到 43 项，覆盖 child / nested 创建、重复创建 `EEXIST`、非空 parent `ENOTEMPTY`、空目录删除、删除后 lookup/stat `ENOENT`、cwd stale dentry 回归、普通文件创建 / hard link / symlink / rename 的 `EPERM` 和失败无副作用、cgroup v1 仍 `ENODEV`。四个平台 QEMU 均通过，结果为 `43 pass, 0 fail`。

该 PR 仍未实现 per-process cgroup membership、`cgroup.procs` 迁移、fork/exit 继承与清理、`/proc/[pid]/cgroup`、controller 注册与接口文件。它是 hierarchy 基础，不是完整 cgroup。

## 5. 进展追踪 Issue：#582

Issue 链接：[#582](https://github.com/rcore-os/tgoskits/issues/582)

该 Issue 用于公开记录后续聚焦 cgroup 的工作进展。Issue 中已经记录了 #545、#657、#796、#869、#989、#1015 的阶段性完成情况，并补充了 cgroup 支持调研与实施路线。

当前 Issue 的价值主要在两点：

- 公开说明后续方向从 nginx 适配逐步转向 cgroup，为 Docker 适配打基础。
- 用 slice 方式拆分 cgroup 工作，避免一开始承诺完整 Docker，而是按 `mount cgroup2 -> hierarchy -> cgroup.procs membership -> /proc/[pid]/cgroup -> fork/exit -> pids controller` 的路径推进。

截至本报告时间，该 Issue 仍处于 open 状态，后续会继续作为 cgroup 开发进展记录。

## 6. 社区相关贡献：axcpu RISC-V FP/SIMD 修复

PR 链接：[#28](https://github.com/arceos-org/axcpu/pull/28)、[#30](https://github.com/arceos-org/axcpu/pull/30)

这两个 PR 是同一技术修复分别合入 `dev` 和 `main` 的记录。问题来源是下游 StarryOS 在 riscv64 + RustSBI 下启动时，可能在任务切换 / FP-SIMD context switch 路径触发：

- `Unhandled trap Exception(IllegalInstruction)`
- `stval=0xf2000053`

根因是 `axcpu` RISC-V FP/SIMD 上下文切换中，`restore` / `clear` 路径可能在 `sstatus.FS` 仍为 `Off` 时执行 FP 指令。当 `FS=Off` 时，`fld`、`fmv.d.x` 等 FP 指令会触发 illegal instruction trap。

最终修复根据维护者建议简化为：将 `sstatus::set_fs(next_fp_state.fs)` 移到 restore/clear match block 之前，确保进入 FP register 操作前 FS 已处于允许执行 FP 指令的状态。作者在评论中记录了使用 RustSBI 和 OpenSBI/default BIOS 的下游 StarryOS 启动验证，但也明确说明未在真实硬件开发板上验证。

这部分贡献不是 `tgoskits` 主仓库贡献，因此不计入主仓库 7 个 PR 统计；但它与 StarryOS 下游运行稳定性有关，作为社区相关补充成果保留。

## 7. 综合影响

### 7.1 Linux 兼容性推进

futex / robust-list 相关 PR 将 StarryOS 的同步原语兼容性向 Linux ABI 靠近，尤其是 wait 超时取消、requeue 生命周期、bitset 校验、robust-list owner death 用户态可见状态等问题。这类问题会影响 pthread condition variable、robust mutex、语言运行时和复杂用户态程序。

nginx 相关 PR 修复了两个具体阻塞点：

- worker channel 初始化依赖的 `FIOASYNC` / owner 状态。
- request body 处理路径依赖的 TCP socket `FIONREAD`。

这并不等于 StarryOS 已完整支持 nginx，但它让 nginx smoke 从“基础初始化 / 特定请求路径失败”向更深层问题推进。

### 7.2 cgroup 方向从空白到可验证基础

cgroup 两个 PR 将 StarryOS 从没有 `cgroup2` mount 支持推进到：

- 能显式挂载最小 cgroup2。
- root cgroup 暴露基础 interface files。
- root `cgroup.procs` 能读取当前活跃 PID。
- 能通过 `mkdir` / `rmdir` 维护 child cgroup hierarchy。
- cgroup2fs 目录使用全局 tree，不依赖 stale per-instance dentry cache。
- 四架构 QEMU 测试有独立 `cgroup-basic` case。

这仍然距离 Docker 所需能力很远，但它已经建立了后续 membership、procfs、pids controller 的基础结构和测试入口。

### 7.3 测试资产沉淀

这些 PR 新增或扩展了多类可复用测试：

- `test-futex-robust-list`：raw syscall futex / robust-list 回归。
- `bug-nginx-fioasync`：nginx worker channel 所需 socket async / owner 状态回归。
- `bug-nginx-fionread-socket`：TCP socket `FIONREAD` 回归。
- `nginx-smoke`：真实 nginx 端到端 smoke 入口。
- `cgroup-basic`：cgroup2 mount/root/hierarchy 回归。

这些测试不仅证明当前修复有效，也能在后续迭代中防止回归。

## 8. 未完成边界

本节专门列出未完成内容，避免夸大成果。

### 8.1 futex / robust-list

当前已覆盖基础 wait/wake、bitset、requeue、普通 robust-list owner death 等语义，但尚未实现或系统审计：

- PI futex，例如 `FUTEX_LOCK_PI`、`FUTEX_UNLOCK_PI`、`FUTEX_WAIT_REQUEUE_PI`。
- `FUTEX_WAKE_OP`。
- futex2 / `futex_waitv`。
- 更完整的 signal restart / interruption 细节。
- shared futex key 在文件映射、COW、unmap/remap、inode 生命周期下的完整性。
- robust-list ptrace 权限和 `execve` 生命周期。

### 8.2 nginx

当前修复的是 nginx smoke 暴露出的具体 socket ioctl 阻塞点：

- `FIOASYNC`
- socket owner 状态
- TCP socket `FIONREAD`

尚未完成：

- 完整 `SIGIO` 投递。
- UDP/RAW/AF_UNIX datagram 的 `FIONREAD`。
- Linux AIO syscall family。
- nginx keep-alive 两连请求等后续 smoke 暴露的问题。
- 完整 nginx 生产级兼容性。

### 8.3 cgroup / Docker

当前 cgroup 支持仍处于早期阶段，尚未实现：

- per-process cgroup membership。
- 写 `cgroup.procs` 迁移进程。
- `/proc/[pid]/cgroup`。
- fork/clone 继承、exit 清理。
- pids controller。
- cpu/memory/io/cpuset controller。
- cgroup namespace。
- `clone3.cgroup` / `CLONE_INTO_CGROUP`。
- systemd / Docker 完整兼容。

因此，本次成果只能描述为“cgroup2 初始支持和 hierarchy 基础能力”，不能描述为“已支持 Docker”。

## 9. 后续计划

后续工作将继续沿 Issue #582 的 slice 路线推进，优先级建议如下：

1. `cgroup.procs` 迁移和真实 membership。
2. `/proc/[pid]/cgroup` 与 `/proc/self/cgroup`。
3. fork/clone 继承 membership，exit 清理。
4. pids controller：`pids.current`、`pids.max` 和 fork 超限失败。
5. 再评估 `clone3.cgroup`、cgroup namespace、systemd 兼容文件或 CPU accounting。

继续推进时仍应保持当前原则：新增每个 cgroup 文件都说明是真语义、只读兼容，还是未支持并返回错误；不要用“写入成功但无实际效果”的方式伪装兼容。

## 10. 结论

本次开源奖励计划成果可以概括为：围绕 StarryOS 的 Linux 兼容性和 Docker 前置基础设施，完成了一组已合并的、带测试的上游 PR。

其中 futex / robust-list 工作修复了同步原语和 robust mutex 相关基础 ABI；nginx 工作推进了真实用户态程序的网络 socket ioctl 兼容；cgroup 工作从无到有建立了 cgroup2 mount/root/hierarchy 的最小可验证基础；Issue #582 则承接后续持续贡献。

当前成果有明确边界，没有完成完整 Docker、完整 nginx 或完整 cgroup controller。但每个已合并 PR 都对应真实问题、明确修复、公开 review 和可复核测试，是可以作为比赛成果提交的实际社区贡献。
