# 2026 OS BigLab-B 报告

### 任务一：ArceOS 5 个基础练习


| 练习 | 体量 | 触达组件 | 关键决策 |
|------|------|----------|---------|
| printcolor | 1 行 | 应用 `main.rs` | 改最上层（应用层），跨架构一份代码即可 |
| hashmap | vendor 整个 axstd | `axstd::collections` | `hashbrown 0.16` + `foldhash` no_std |
| altalloc | 实现 ~95 行 | `bump_allocator` / `axalloc` | 双端 bump + 三 trait + 计数式回收 |
| ramfs-rename | 改 2 文件 | `axfs_ramfs` / `axfs` | VFS rename 递归 + 挂载点转发 + 先删后改 |
| sysmap | 加 ~160 行 | `src/syscall.rs` | 两种映射分支 + 用户地址空间向下增长 |

### 任务二 · 实验一：OSAgent 框架

OSAgent 是搭在 Claude Code 之上的 StarryOS 内核开发流水线。核心设计原则：

> Skills define step-by-step workflows. The main session calls sub-agents at each step. No nesting — all agent calls come from the top-level session.

具体结构：

- **6 个 skill**：`busybox-fix` / `debug-fix` / `feature-dev` / `app-port` / `claw-debug` / `claw-robust`
- **7 个 sub-agent**：`git-sync-agent` / `code-explorer-agent` / `test-runner-agent` / `test-agent` / `pre-commit-agent` / `pr-writer` / `app-profiler-agent`
- **3 个模板**：`pr-bugfix.md` / `pr-feature.md` / `test-case.md`（含 11 条覆盖类目 checklist）
- **核心硬规则**：`Fix the kernel, not the test`、issue #13 测试命令逐字符照抄、`outputs/` 作为跨步骤共享内存

OSAgent 经历了"单 agent → main + sub-agent → skill + flat sub-agent → app-port / claw 系列"的四次范式变迁，每一步都是对实际跑出来痛点的回应。

实验二 / 三 / 四 的全部 PR 都是 OSAgent 的产出，**框架与产出是一体的**。详见 [exp1_OSAgent/summary.md](./exp1_OSAgent/summary.md)。

### 任务二 · 实验二：6 个 syscall 修复

| PR | syscall | 类型 |
|----|---------|------|
| [#257](https://github.com/rcore-os/tgoskits/pull/257) | `times` | 子进程 CPU 时间统计语义建模缺失 |
| [#430](https://github.com/rcore-os/tgoskits/pull/430) | `clock_getres` | 无效 clock_id 假成功 → EINVAL |
| [#441](https://github.com/rcore-os/tgoskits/pull/441) | `fallocate` | 5 类 bug，跨 starry-kernel / axfs-ng / rsext4 三层 |
| [#444](https://github.com/rcore-os/tgoskits/pull/444) | `fadvise64` | EBADF / EINVAL / ESPIPE 校验 |
| [#466](https://github.com/rcore-os/tgoskits/pull/466) | `truncate / ftruncate` | 空路径 / 超大长度 / 只读文件 / 目录 fd 四类边界 |

全部使用 `debug-fix` skill。修复模式高度统一：**early input validation + 错误映射到 `LinuxError::XXX`**。测试覆盖类目对齐 `templates/test-case.md` 里的 11 条 checklist，每个 syscall 新增 130 ~ 360 行 C 测试。

**5 merged**。详见 [exp2_syscall/summary.md](./exp2_syscall/summary.md)。

### 任务二 · 实验三：4 个 BusyBox applet 修复

| PR | applet | 修复方式 |
|----|--------|---------|
| [#489](https://github.com/rcore-os/tgoskits/pull/489) | `blockdev --getss` | loop 设备 ioctl 加 `BLKSSZGET / BLKPBSZGET` |
| [#491](https://github.com/rcore-os/tgoskits/pull/491) | `blkid` | `sys_ioctl` warn 抑制白名单 |
| [#517](https://github.com/rcore-os/tgoskits/pull/517) | `run-parts` | `sys_execve` 加 ENOEXEC fallback（5 次提交迭代） |
| [#521](https://github.com/rcore-os/tgoskits/pull/521) | `hwclock -r` | **删除**假的 `/dev/rtc0` 注册 |

全部使用 `busybox-fix` skill。测试命令严格来自 issue #13，禁止简化。测试套最终 276/0 全绿。


### 任务二 · 实验四：claw-code on StarryOS

让 Claude Code CLI (claw-code) 这个真实的 Node.js 大应用在 StarryOS 上运行。核心工作是**构建了一套"兼容性沙箱"**: 实现 `unshare(CLONE_NEWUSER)` + `/proc/self/{uid_map,gid_map,setgroups,cgroup}` 四个 procfs 伪文件，让 claw 的 sandbox 初始化流程能完整执行。24 个提交（87 文件，+2536/-5 行）分四个阶段推进：

| 阶段 | 提交数 | 关键内容 |
|------|--------|---------|
| Phase 1: 沙箱机制 | 7 | `sys_unshare` + uid_map/gid_map/setgroups/cgroup + 5 个 goal C 测试 |
| Phase 2: 集成测试 | 11 | integration Rust binary（6 级 fail-fast）+ 12 个 robust 测试 + `sys_mount` NULL fix + unshare flags 扩展 |
| Phase 3: 应用注册 | 1 | `apps/starry/claw-code/` 构建配置 + prebuild.sh + QEMU 配置 |
| Phase 4: Review 修复 | 4 | goal x86_64 配置补全 + CLAW_API_KEY 优雅跳过 + release URL 修正 + unshare warn |


详见 [exp4_claw_code/summary.md](./exp4_claw_code/summary.md)。

## 四、方法论小结

跑完任务一加任务二的实验一 / 二 / 三 / 四之后，归纳出以下几条方法论：

### 4.1 调用链分层思维

`ramfs-rename` 那个练习从 `std::fs::rename` 到 `axfs_ramfs::DirNode::rename` 中间过 4 层。这种"由上到下逐层下钻"的思维方式在后续 syscall 修复里反复用到：每个 syscall 都是 syscall 层 → VFS 层 → 具体 fs 层；每一层都要确认职责边界。`fallocate` 的修复需要同步改 syscall / axfs-ng / rsext4 三个 crate，正是因为 EFBIG 的检查需要在每一层都补。

### 4.2 假成功比假失败更危险

`clock_getres` / `fadvise64` / `truncate / ftruncate` 几个 PR 都有"对错误输入返回 0"的 bug。`hwclock` 那个 PR 更极端——starry 注册了 fake `/dev/rtc0` 让命令"成功"返回时间，反而堵死了测试。"返回成功"并不一定等于"语义正确"，**对齐 Linux 的失败路径有时候比加新功能更重要**。

### 4.3 errno 优先级是反复踩坑点

实验二的 4 个 PR 都收到 review 反馈说错误优先级偏差。Linux 的隐含约定是 "fd → 参数 → 权限 → 实际操作"。这是后续应该用一个独立 refactor PR 系统性解决的方向，可能需要抽象出一个 "validation pipeline" 把所有 syscall 的校验顺序统一起来。

### 4.4 内核 vs 用户态语义边界要慎重

实验三 #517 演示了一个完整的边界讨论：把"用户态本应做的 ENOEXEC 重试"塞进内核能让测试 PASS，但会破坏 raw `execve` 的对外语义。最终的折中——"loader 层保持原 `InvalidExecutable`、syscall handler 内部 retry、注释明示临时 workaround"——是 review 能接受的最远边界。这种讨论比单纯修代码更有价值。

### 4.5 issue-driven 工作流的护栏价值

OSAgent `busybox-fix` skill 强制要求测试命令从 issue #13 逐字符照抄。这条规则看起来繁琐，但堵死了"改测试让它过"这种最直接的作弊路径。`templates/test-case.md` 的 11 条 checklist 把"测试要覆盖什么"也规范化了，让 LLM 写测试时不会偷懒。

### 4.6 vendor + `[patch.crates-io]` 是利器

任务一的 hashmap / altalloc / ramfs-rename 三个练习都用了"`cargo clone` + `[patch.crates-io]`"的组合。这套机制让"修一个 crate 一行代码"和"重写半个 crate"都用同一种工程姿态完成。后续在 starry 上跨多个 crate 改动（如 `fallocate` 那种）也是同一种思维。

### 4.7 工程进化是被实际跑出来的

OSAgent 经过4 次范式变迁（context 爆炸 → 拆 sub-agent；clippy 跑得慢 → pre-commit 简化为 fmt-only）。这告诫我们：**框架不能纸面设计，要先跑起来再迭代**。
