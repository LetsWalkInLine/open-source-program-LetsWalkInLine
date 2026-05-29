# 开源奖励计划成果提交仓库

本仓库用于整理本人参加开源奖励计划期间的公开社区贡献成果。

本次提交内容不包含独立研发的开源工具项目，主要成果为对 `rcore-os/tgoskits` 上游仓库的 PR 贡献、围绕这些贡献撰写的技术总结，以及持续跟踪后续 cgroup 开发计划的公开 Issue。

## 主要文档入口

- 详细技术总结报告：[docs/technical-summary.md](docs/technical-summary.md)
- PR 与 Issue 元数据：[data/contributions.json](data/contributions.json)
- 公开进展追踪 Issue：[rcore-os/tgoskits#582](https://github.com/rcore-os/tgoskits/issues/582)

## tgoskits 主仓库贡献

截至 2026-05-29，以下 7 个 `rcore-os/tgoskits` PR 均已合并：

| PR | 主题 | 状态 |
| --- | --- | --- |
| [#468](https://github.com/rcore-os/tgoskits/pull/468) | 修复 `starry-signal` restore 测试中的跨架构栈假设 | Merged |
| [#545](https://github.com/rcore-os/tgoskits/pull/545) | 修复 futex wait / requeue / robust-list owner death 基础语义 | Merged |
| [#657](https://github.com/rcore-os/tgoskits/pull/657) | 进一步完善 futex / robust-list Linux ABI 基础语义 | Merged |
| [#796](https://github.com/rcore-os/tgoskits/pull/796) | 支持 nginx 初始化依赖的 socket `FIOASYNC` / owner 状态 | Merged |
| [#869](https://github.com/rcore-os/tgoskits/pull/869) | 支持 TCP socket `FIONREAD`，修复 nginx 超限 POST 路径 | Merged |
| [#989](https://github.com/rcore-os/tgoskits/pull/989) | 添加 cgroup2 初始支持和 `cgroup-basic` 测试 | Merged |
| [#1015](https://github.com/rcore-os/tgoskits/pull/1015) | 支持 cgroup2 hierarchy 的 `mkdir` / `rmdir` 基础能力 | Merged |

GitHub PR 统计口径下，这 7 个 PR 共涉及 50 个变更文件、17 个提交，合计 `+4007/-224` 行。该统计包含测试代码、QEMU 配置和文档式 PR 描述中的相关实现背景，不等同于净新增功能代码行数。

## 社区其他仓库的相关贡献

比赛要求聚焦 `tgoskits`，但在比赛具体通知主仓库前，本人也对社区相关老仓库 `arceos-org/axcpu` 提交过同一技术修复的两个合入记录：

| PR | 说明 | 状态 |
| --- | --- | --- |
| [arceos-org/axcpu#28](https://github.com/arceos-org/axcpu/pull/28) | RISC-V FP/SIMD 切换路径在 RustSBI 下的 `sstatus.FS` 前置修复，合入 `dev` | Merged |
| [arceos-org/axcpu#30](https://github.com/arceos-org/axcpu/pull/30) | 按维护者建议重新提交同一修复到 `main` | Merged |

报告中将这部分作为“社区相关贡献”单独说明，不计入 `tgoskits` 主仓库贡献数量。

## 持续贡献方向

公开 Issue [rcore-os/tgoskits#582](https://github.com/rcore-os/tgoskits/issues/582) 用于持续跟踪后续工作。比赛结束后，后续重点仍将围绕 StarryOS 的 cgroup 支持推进，包括 `cgroup.procs` 迁移、`/proc/[pid]/cgroup`、fork/exit membership 生命周期、pids controller 等方向。

> 很高兴能够为社区贡献自己的一份力量，并在此过程中收获了许多成长。以上内容恳请各位评委老师审阅，感谢您的时间与指导。
