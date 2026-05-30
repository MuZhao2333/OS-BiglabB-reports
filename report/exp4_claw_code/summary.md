# 任务二 · 实验四：让 claw-code 在 StarryOS 上运行

> 代码分支：[MuZhao2333/tgoskits @ feat/claw-code](https://github.com/MuZhao2333/tgoskits/tree/feat/claw-code)
>
> 工作流：基于 [OSAgent](../exp1_OSAgent/summary.md) 的 `app-port` / `claw-debug` / `claw-robust` 三个 skill 协同完成

## 一、实验目标

任务2 实验4 要求 "基于 qemu，发现并修复 starry 中 bug，增加功能，支持 1~2 个 Linux 大应用基本正常运行"。本实验选择的"大应用"是 **claw-code**-。这是一个 **真实的多进程、多线程、联网调 LLM API、读写文件系统、shell 执行的 Node.js 巨应用**。

要让这样一个应用在 StarryOS 上跑起来，需要完成三件事：

1. **API 兼容层**：补齐 Node.js / glibc 所需但 StarryOS 缺失的 syscall；
2. **沙箱环境**：实现 claw-code 启动时必经的 sandbox 初始化流程（unshare + uid_map/gid_map + setgroups + cgroup）；
3. **集成与健壮性验证**：从 smoke (`--help`) 到多 agent 并发的完整测试链。

最终产出：**24 个提交（含 4 个 PR review 修复），87 个文件改动（+2536/-5 行），5 个 goal 测试 + 1 个 integration 测试 + 12 个 robust 测试**。

## 二、什么是 claw-code 以及它为什么需要沙箱

### 2.1 claw-code 的本质

claw-code 的运行链路是：

```
claw 二进制 (Node.js + V8 + libuv)
  → glibc (pthread, malloc, TLS, rseq, futex...)
    → 100+ 种 Linux syscall
      → StarryOS syscall dispatch layer
```

claw 启动时会经历**沙箱初始化**：它认为自己在 Linux 上跑，就会尝试 `unshare(CLONE_NEWUSER|CLONE_NEWNS|...)` 来降低权限，然后写 `/proc/self/uid_map` 和 `/proc/self/gid_map` 来完成用户命名空间的映射设置，最后读取 `/proc/self/cgroup` 来做容器环境自检。

如果这些 syscall 或 procfs 文件缺失，claw 的启动就会在 sandbox 初始化阶段崩溃。它需要"认为自己处在沙箱中"才能继续运行。

### 2.2 StarryOS "沙箱"的真实含义

- **claw 真正依赖的行为** → 完整实现（如 unshare(NEWUSER)、uid_map / gid_map / setgroups）
- **claw 只是"摸一下"但不依赖实际隔离效果的** → 接受但 no-op（如其他 namespace flags、seccomp）
- **claw 的环境自检路径** → mock 返回合理值（如 cgroup 始终返回 `"0::/"`）

## 三、开发阶段与提交演进

整个实验分三个阶段，对应 OSAgent `app-port` skill 的标准流程。

### Phase 1: 内核沙箱机制（2026-05-11，7 个提交）

**Commit 1** `9066a22` — 实现 `sys_unshare`，支持 `CLONE_NEWUSER` 将凭据重置为 nobody (65534)，新增 goal-01 测试。

**Commit 2** `39590a0` — 在 procfs 注册 `/proc/self/uid_map`、`/proc/self/gid_map`、`/proc/self/setgroups` 三个伪文件；`Thread` 结构体新增 `uid_map_written`、`gid_map_written`、`setgroups_deny` 三个 `AtomicBool` 字段。

**Commit 3-5** `46d0804` ~ `3b2abdc` — 分别为 uid_map、gid_map、setgroups 补上 goal-02/03/04 测试。

**Commit 6** `dad6958` — `cargo fmt` 格式化修复。

**Commit 7** `7d6a15b` — 实现 `/proc/[pid]/cgroup`（始终返回 `"0::/\n"`），新增 goal-05 测试。

### Phase 2: 集成测试与关键修复（2026-05-11 至 2026-05-15，11 个提交）

**Commit 8-11** — 创建 integration 测试套件，先写成 shell 脚本，后重写为 Rust binary（使 claw 二进制通过 `build.rs` 从 GitHub Release 下载并 `include_bytes!` 嵌入）。测试覆盖 smoke (`--help`)、diagnostic (`version`)、functional (`prompt`)、tool (bash)、project (C compile & run) 六个层级。

**Commit 12** `fa97f0e` — **关键修复**：
1. **`sys_mount` 允许 `fs_type` 为 NULL**：claw sandbox 初始化会调 `mount("none", "/proc", NULL, MS_REMOUNT, NULL)`，原代码在 `vm_load_string(NULL)` 时 panic；
2. **`sys_unshare` 扩展为接受所有标准 namespace flags**：最初的实现只接受 `CLONE_NEWUSER` 单一 flag，但 claw 实际调用的是 `unshare(CLONE_NEWUSER|CLONE_NEWNS|...)` 组合 flags。扩展后的实现用 bitmask 验证所有已知 flags（`CLONE_NEWNS | CLONE_NEWCGROUP | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER | CLONE_NEWPID | CLONE_NEWNET`），其中只有 `CLONE_NEWUSER` 真正执行凭据重置，其余作为 no-op 接受。

**Commit 13** `c91d945` — 新增 11 个 robust 测试（robust-01 ~ robust-11），引入 `CLAW_API_KEY` 环境变量注入机制。

**Commit 14-19** — 格式修复、rebasing、清理等后续整理。

### Phase 3: 应用注册（2026-05-29）

**Commit 20** `fe600ad` — 创建 `apps/starry/claw-code/` 目录，包含 `build-x86_64-unknown-none.toml`、`prebuild.sh`（从 GitHub Release 下载 claw 并注入 rootfs）、`qemu-x86_64.toml`（512M 内存，virtio-blk + virtio-net）。

### Phase 4: PR Review 修复与 CI 完善（2026-05-29，4 个提交）

**Commit 21** `b5e1edd` — **优雅跳过**：当 `CLAW_API_KEY` 环境变量未设置时（如 fork 仓库的 CI 环境），robust-01 到 robust-12 全部 12 个测试 graceful skip 而非 FAIL，避免无 API key 的 CI 误报失败。

**Commit 22** `9ca6eac` — 合并上游 dev 分支的 CI workflow 冲突。

**Commit 23** `987833f` — **PR review 修复**：(1) 为 goal-02 到 goal-05 补了 `qemu-x86_64.toml` 配置（此前仅有 riscv64 配置，x86_64 CI 上这些 goal 会因缺少配置而无法运行）；(2) 在 `namespace.rs` 添加注释说明 uid_map 的简化语义；(3) 非 `CLONE_NEWUSER` 的 namespace flags 被 no-op 接受时新增 `warn!` 日志，提升可观测性。

**Commit 24** `f5206a4` — claw 二进制下载 URL 从个人 fork `MuZhao2333/tgoskits` 统一改为 `rcore-os/tgoskits` 组织仓库。

## 四、沙箱环境深度剖析

这是本实验的核心：**StarryOS 为了支持 claw-code，构建了一个多层沙箱环境**。以下从底层到顶层逐层分析。

### 沙箱架构概览

```
┌──────────────────────────────────────────────────────────────┐
│                    claw-code (Node.js)                        │
│  启动时执行: unshare → uid_map → gid_map → setgroups →      │
│              cgroup 检测 → mount proc → 准备就绪              │
├──────────────────────────────────────────────────────────────┤
│              glibc (TLS, rseq, futex, pthread...)             │
├──────────────────────────────────────────────────────────────┤
│          StarryOS Syscall Dispatch Layer                      │
│  ┌───────────────────────┬──────────────────────────────────┐ │
│  │  完整实现              │  Mock / No-op / Stub            │ │
│  │  signal, clone, fork   │  seccomp (完全 no-op)           │ │
│  │  pidfd, eventfd,       │  cgroup (只返回 "0::/")         │ │
│  │  signalfd, timerfd     │  NEWNS/NEWPID/NEWNET (stub)     │ │
│  │  chroot, pivot_root    │  get_mempolicy (mock)           │ │
│  │  mount, umount2        │  open_by_handle_at (ENOSYS)     │ │
│  │  prlimit64             │  sched_getscheduler (mock)      │ │
│  │  uid_map/gid_map       │                                  │ │
│  │  unshare(NEWUSER)      │                                  │ │
│  └───────────────────────┴──────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────┤
│          StarryOS 内核对象层                                  │
│  Thread, ProcessData, Cred, Signal, FutexTable, FsContext    │
├──────────────────────────────────────────────────────────────┤
│          QEMU x86_64 (virtio-blk, virtio-net)                │
└──────────────────────────────────────────────────────────────┘
```

下面逐层深挖具体机制。

### 4.1 第一层：进程隔离 —— `unshare` + 用户命名空间凭证重置

这是**整个沙箱的入口**。

#### 实现（`syscall/task/namespace.rs`，54 行，全新文件）

```rust
pub fn sys_unshare(flags: i32) -> AxResult<isize> {
    if flags == 0 { return Ok(0); }

    const SUPPORTED_FLAGS: i32 = CLONE_NEWNS | CLONE_NEWCGROUP
        | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER
        | CLONE_NEWPID | CLONE_NEWNET;

    if flags & !SUPPORTED_FLAGS != 0 {
        return Err(AxError::InvalidInput);
    }

    if flags & CLONE_NEWUSER as i32 != 0 {
        // 在新的用户命名空间中，uid/gid 初始为 65534 (nobody)
        let mut cred = (*thr.cred()).clone();
        cred.uid = 65534; cred.gid = 65534;
        cred.euid = 65534; cred.egid = 65534;
        cred.suid = 65534; cred.sgid = 65534;
        cred.fsuid = 65534; cred.fsgid = 65534;
        Thread::set_cred(thr, cred);
    }
    Ok(0)
}
```

#### 沙箱层面的意义

这个实现模拟了 Linux 用户命名空间中"初始凭据为 nobody"的语义。在 Linux 上，当你 `unshare(CLONE_NEWUSER)` 时：

1. 内核创建一个新的 user namespace；
2. 在这个新 ns 中，你的 uid/gid 被设为 65534（nobody 的 overflow UID）；
3. 你必须接着写 `/proc/self/uid_map` 和 `/proc/self/gid_map` 来建立映射（把"内部的 uid 0"映射到"外部的一个真实 uid"）；
4. 之后，你在新 ns 中就可以作为 root 执行需要特权的操作——但这些特权只在新 ns 内有效。

StarryOS **没有实现完整的多层级 user namespace**，而是直接将 2 和 3 拆成两步：

- `unshare(CLONE_NEWUSER)` 把凭据置为 nobody；
- 写 `uid_map` / `gid_map` 把凭据"映射"（实际上是直接设置）回需要的 uid/gid。

**claw-code 为什么需要这一步**：claw 在启动时通过 `unshare(CLONE_NEWUSER)` 把自己从 root 降为 nobody，然后再通过写 `uid_map` 把内部 uid 0 映射到外部 uid 0（root）。这个流程在 Linux 上的含义是"我在这个 user namespace 里是 root，但在外面是 nobody"——StarryOS 简化了这个语义为"先降到 nobody，再升回需要的 uid"，保持外部 API 完全兼容。

#### 与其他 namespace flags 的关系

`CLONE_NEWNS`、`CLONE_NEWPID`、`CLONE_NEWNET` 等 flags 被 bitmask 接受但**不执行任何实际隔离**（no-op）。这样做的原因：

- claw 调用 `unshare(CLONE_NEWUSER|CLONE_NEWNS|...)` 组合 flags；
- 如果返回 `EINVAL`，claw 会认为"沙箱不可用"并退出；
- 如果接受但 no-op，claw 的后续初始化继续，最终运行；
- claw 实际上不依赖这些其他 namespace 的隔离效果——它只需要"不报错"。

**证据**：Commit `fa97f0e` 的 diff 中 `sys_unshare` 从只支持 `CLONE_NEWUSER` 扩展到支持全部 7 个 flags，commit message 描述为 "expand to accept all standard namespace flags with bitmask validation"。这就是在发现 claw 的实际调用行为后做的修正。

### 4.2 第二层：凭证映射 —— `/proc/self/uid_map` + `gid_map` + `setgroups`

这四个伪文件（`uid_map`、`gid_map`、`setgroups`、`cgroup`）构成了**沙箱凭证层的完整 API 表面**。

#### `uid_map` / `gid_map`（可读写）

**文件路径**：`os/StarryOS/kernel/src/pseudofs/proc.rs:831-906`

**读行为**：
- 如果已写过，返回 `"         0  <uid> 4294967295\n"`（格式：内部ID、外部ID、长度）
- 否则返回 `"\n"`（空映射）

**写行为**：解析 `"<mapped> <orig> <count>"` 格式（空格分隔的 3 个 u32），将 `orig` 值设置到凭据的 uid/euid/suid/fsuid（或 gid/egid/sgid/fsgid），标记 `uid_map_written`（或 `gid_map_written`）为 true。

**沙箱意义**：在 Linux 上，写 `uid_map` 只能在**你不再具有原 namespace 中的权限**时才能做（即写 uid_map 是"一次性操作"）。StarryOS 简化了这条规则，但保持了相同的外部接口。claw 写的是 `"0 0 1"` 这样的格式，意思是"把新 ns 中的 uid 0 映射到旧 ns 中的 uid 0"——StarryOS 直接把 uid 设成 0，等效于建立了映射。

**证据**：`goal-02-uid-map` 测试用例验证了"写入之前 uid 是 nobody(65534)，写入 `0 0 1` 之后 uid 变回 0"。这与 Linux 行为等价。

#### `setgroups`（可读写）

**读行为**：返回 `"deny\n"` 或 `"allow\n"`。
**写行为**：接受 `"deny"` 或 `"allow"`。

**沙箱意义**：Linux 内核强制要求：在写 `gid_map` **之前**，必须先写 `"deny"` 到 `/proc/self/setgroups`（除非调用者有 `CAP_SYS_ADMIN`）。这条规则防止了"通过 `setgroups()` 在用户命名空间中获取额外组权限"的逃逸路径。StarryOS 提供了完全相同的 two-step 接口：先写 `"deny"` 到 setgroups，再写 gid_map。_虽然 StarryOS 的 `setgroups()` 本身没有做严格的权限检查，但 API 层面的兼容让 claw 的初始化流程能走完。_

**证据**：`goal-04-setgroups` 测试验证了"先 deny 再写 gid_map"的两步流程。

#### `cgroup`（只读）

始终返回 `"0::/\n"`（cgroup v2 unified hierarchy root）。

**沙箱意义**：claw 通过读 `/proc/self/cgroup` 来**检测自己是否在容器中运行**。如果这个文件不存在，claw 可能错误地认为自己在裸机上，调整沙箱策略或产生警告。返回 `"0::/"` 的含义是"处于默认 cgroup v2 层次结构的根"——这是一个正常容器化环境的标准返回值。

**证据**：`goal-05-cgroup` 测试验证了文件存在且可读。

### 4.3 第三层：功能完备的关键 syscall（StarryOS 开箱即用）

以下 syscall 在 StarryOS **已有的 dev 分支上就已经完整实现**，claw-code 能跑起来，很大程度上依赖这些"基础设施"的存在。这里按沙箱相关性分组展示。

#### 进程创建与管理

| syscall | 实现程度 | 沙箱角色 |
|---------|---------|---------|
| `clone` / `clone3` | 完整（含命名空间标志 stub）| 创建子进程，claw 的子 agent 会 fork |
| `fork` / `vfork` | 完整 | 同上 |
| `execve` | 完整 | 执行 `/bin/sh` 和各类命令 |
| `wait4` / `waitid` | 完整 | 回收子进程 |
| `prctl(PR_SET_NO_NEW_PRIVS)` | 完整 | 防止子进程通过 setuid 提权 |
| `prctl(PR_SET_SECCOMP)` | **no-op** | **seccomp 完全不生效** |
| `prctl(PR_SET_NAME)` | 完整 | 线程命名，调试可见性 |
| `prctl(PR_CAPBSET_READ)` | 总返回 1（有权限）| capabilities 查询简化为 euid==0 |

**关键设计**：`PR_SET_SECCOMP` 和 `sys_seccomp()` 被接受为 no-op。这意味着 StarryOS 的沙箱 **不提供任何 syscall 过滤能力**。在 Linux 上，Docker 的默认 seccomp 配置文件会禁用约 44 个"危险" syscall。在 StarryOS 上，沙箱内的进程可以自由发起任何 syscall——不阻塞，但也**不执行实际的 seccomp 过滤语义**。

#### 线程与同步

| syscall | 实现程度 | 沙箱角色 |
|---------|---------|---------|
| `set_tid_address` | 完整 | 线程退出时清零 `clear_child_tid` 并 futex_wake |
| `set_robust_list` / `get_robust_list` | 完整 | robust futex 链表管理，防止死锁 |
| `rseq` | 注册/取消完整，**关键段中止未实现** | glibc 初始化 rseq 但不使用加速路径 |
| `futex` | 完整 | pthread mutex / condvar 底层 |
| `tgkill` / `tkill` | 完整（含权限检查）| 线程信号发送，含 euid 权限验证 |
| `rt_sigaction` / `rt_sigprocmask` | 完整 | 信号处理，拒绝 SIGKILL/SIGSTOP |
| `membarrier` | 完整（5 个命令）| 跨核心内存屏障，QEMU 单核下用 `fence(SeqCst)` |

**`rseq` 的特殊设计**：StarryOS 把 `cpu_id` 设为 `RSEQ_CPU_ID_UNINITIALIZED` (u32::MAX)，让 glibc 知道"rseq 加速不可用"并回退到传统的 `getcpu()` syscall。这比返回 ENOSYS 更好——后者会让 glibc 完全放弃 rseq 注册，而这种方式保留了注册语义，只是禁用了 per-CPU 快速路径。

#### I/O 与事件

| syscall | 实现程度 | 沙箱角色 |
|---------|---------|---------|
| `eventfd2` | 完整（含 SEMAPHORE 模式）| libuv 线程间通知 |
| `signalfd4` | 完整 | 信号集成到 epoll 事件循环 |
| `timerfd_create/settime/gettime` | 完整 | 定时器 fd 化 |
| `epoll_create1/epoll_ctl/epoll_wait` | 完整 | libuv 核心事件循环 |

#### 文件系统隔离

| syscall | 实现程度 | 沙箱角色 |
|---------|---------|---------|
| `chroot` | 完整 | 改变文件系统根目录 |
| `pivot_root` | 完整（含跨任务 cwd/root 传播）| 更安全的根切换 |
| `mount` / `umount2` | 完整（tmpfs、ext4、bind、move、remount、传播类型）| 沙箱内部挂载 |
| `name_to_handle_at` | 完整 | 获取文件句柄用于标识 |
| `open_by_handle_at` | **ENOSYS** | — |

**`pivot_root` 是文件系统隔离的亮点**：不仅实现了基本语义，还做了 `propagate_pivot_root()`——将所有任务（线程）的 cwd 和 root 从旧根传播到新根。这是直接从 Linux 的 `chroot_fs_refs()` 语义借鉴的实现。

**证据**：`mount.rs:285-353` 代码中明确调用了 `ax_fs::FsContext::propagate_pivot_root(&old_root, &new_root_loc)`。

#### 现代进程管理

| syscall | 实现程度 | 沙箱角色 |
|---------|---------|---------|
| `pidfd_open` | 完整（含 `PIDFD_THREAD` 标志）| 稳定的进程引用（不受 PID 回收影响）|
| `pidfd_getfd` | 完整（含权限检查）| 跨进程 fd 复制 |
| `pidfd_send_signal` | 完整 | 向进程发送信号 |
| `getcpu` | 完整（Node 固定为 0）| glibc `sched_getcpu()` 的 fallback |
| `getrandom` | 通过 `/dev/urandom` | TLS 握手和 Node.js crypto |

### 4.4 第四层：资源限制（最薄弱，但正在改进）

| syscall | 实现程度 | 沙箱角色 |
|---------|---------|---------|
| `prlimit64` | 完整 | per-process rlimit（`RLIMIT_NOFILE` 等）|
| `sched_getaffinity` / `setaffinity` | 完整 | CPU 亲和性 |
| `cgroup` | **完全 mock** | 仅 `/proc/self/cgroup` 返回 `"0::/"` |
| `sys_seccomp` | **完全 no-op** | 接受但不执行任何过滤 |
| `get_mempolicy` | **mock** | 只打印日志，返回 0 |
| `sys_sched_getscheduler` | **mock** | 总返回 `SCHED_RR` |

**cgroup 和 seccomp 是最大的安全缺口**：

- **cgroup**：Linux 容器用 cgroup v2 限制 CPU 配额、内存上限、I/O 权重。StarryOS 不实施任何限制——`/proc/self/cgroup` 只是一个字符串 `"0::/"`，没有对应的 cgroup 控制器。
- **seccomp**：Linux 容器用 seccomp BPF 过滤器限制可以发起的 syscall 种类。StarryOS 的 `sys_seccomp()` 直接返回 0 不做任何事。**沙箱内的进程可以自由发起所有已实现的 syscall。**

这两项的缺失决定了 StarryOS 沙箱的**安全边界**：这是"兼容性沙箱"，不是"安全沙箱"。

## 五、StarryOS 沙箱与 Linux 沙箱的系统性对比

| 维度 | Linux (Docker/containerd) | StarryOS (feat/claw-code) | 差距 |
|------|--------------------------|--------------------------|------|
| **syscall 过滤 (seccomp)** | BPF 过滤器，Docker 默认禁用 ~44 个 syscall | 完全 no-op | 最大差距 |
| **资源限制 (cgroup v2)** | CPU/内存/IO 精确配额 | 完全 mock | 无限制 |
| **用户命名空间 (NEWUSER)** | 完整嵌套，5 行 uid_map，支持递归 | 单级，凭据重置 + uid_map 写入 | 基本满足 |
| **挂载命名空间 (NEWNS)** | 完整 + overlayfs/aufs | mount/pivot_root 存在，NEWNS flag 是 stub | 文件系统隔离够用 |
| **PID 命名空间 (NEWPID)** | PID 1 init 管理 | clone flag 接受但 no-op | PID 全局可见 |
| **网络命名空间 (NEWNET)** | 独立网络栈 + veth pair | clone flag 接受但 no-op | 网络全局共享 |
| **capabilities** | 64-bit 位图 + 继承规则 + ambient/bounding 集合 | euid==0 返回全权限 | 二元简化 |
| **Landlock** | 无特权文件系统访问控制 (Linux 5.13+) | 未实现 | 未覆盖 |
| **文件系统隔离** | chroot + pivot_root + mount | chroot + pivot_root + mount（完整） | 基本对齐 |
| **进程间 fd 传递** | pidfd_getfd + SCM_RIGHTS | pidfd_getfd 完整实现 | 对齐 |
| **用户凭证** | 完整 uid/gid + supplementary groups | 完整 uid/gid + fsuid/fsgid | 对齐 |


## 六、测试策略

实验四的测试覆盖遵循 OSAgent `claw-debug` skill 的 6 级 fail-fast 顺序：

### 6.1 Goal 测试（5 个，C 语言）

每个 goal 对应一个独立的内核功能点，用 C 程序调用目标 syscall/procfs，与 Linux 行为对照：

| 测试 | 验证内容 | 测试文件 |
|------|---------|---------|
| goal-01-unshare | `unshare(0)` → OK, `unshare(CLONE_NEWUSER)` → OK, `unshare(0xdeadbeef)` → EINVAL | C 程序, 3 个用例 |
| goal-02-uid-map | `/proc/self/uid_map` 存在、可读、可写、unshare 后可映射 | C 程序, 5 个用例 |
| goal-03-gid-map | `/proc/self/gid_map` 同 uid_map 语义 | C 程序, 5 个用例 |
| goal-04-setgroups | deny/allow 读写、unshare 后可用 | C 程序, 4 个用例 |
| goal-05-cgroup | `/proc/1/cgroup` 和 `/proc/self/cgroup` 存在且可读 | C 程序, 3 个用例 |

每个 goal 带 `qemu-riscv64.toml` 配置，独立运行，独立 PASS/FAIL。

### 6.2 Integration 测试（Rust binary）

**文件**：`test-suit/starryos/normal/qemu-smp1/claw-code/integration/rust/src/main.rs`（123 行）

在 StarryOS guest 中以 Rust binary 运行，测试 6 级：

1. **Smoke**：`claw --help`（验证二进制能启动）
2. **Diagnostic**：`claw version`（验证版本信息输出）
3. **Functional**：`claw prompt 'say just the word ok and nothing else'`（验证 LLM API 调用）
4. **Tool (bash)**：`claw --allowedTools bash prompt 'use bash to echo hello world'`（验证子进程执行）
5. **Project (write)**：`claw --allowedTools bash,write prompt 'create a file ...'`（验证文件系统写操作）
6. **Project (C compile)**：`claw --allowedTools bash,write prompt 'write a C program ... compile it, and run it'`（验证完整工具链）

每个非 smoke 级测试带 3 次重试（因为涉及 LLM API，单次调用有不确定性）。成功标记 `ALL_TESTS_DONE`，失败标记 `ALL_TESTS_FAILED`。

claw 二进制通过 `build.rs` 从 GitHub Release 下载，`include_bytes!` 嵌入到测试 binary 中。API 凭据通过 `option_env!("CLAW_API_KEY")` 编译时注入。

### 6.3 Robust 测试（12 个，Rust）

| 测试 | 内容 | 超时 |
|------|------|------|
| robust-01 | 搜索 LLM agent 论文，写摘要到 `digest.md` | 600s |
| robust-02~11 | 不同类型的文件操作和 shell 任务 | 300s |
| robust-12 | **多 Agent 并行**：2 个子 agent 并行创建文件，合并结果 | 600s |

全部在 StarryOS guest 中以 Rust binary 运行，每个测试创建独立的 git repo（`/tmp/work/.git/`）供 claw 工作。

## 七、遇到的难点与代表性 bug

### Bug 1: `sys_mount` 不接受 NULL `fs_type`

**现象**：claw 在 sandbox 初始化时调用 `mount("none", "/proc", NULL, MS_REMOUNT, NULL)`，StarryOS 的 `vm_load_string(NULL)` 失败 → mount 返回错误 → sandbox 初始化失败。

**根因**：`sys_mount` 假设 `fs_type` 参数永不为 NULL，直接解引用。

**修复**（commit `fa97f0e`）：加了 NULL 指针检查。

```rust
let fs_type = if fs_type.is_null() {
    String::new()
} else {
    vm_load_string(fs_type)?
};
```

**思考**：这个 bug 暴露了一个常见的"假设"— Linux ABI 中很多"看似必须"的指针参数实际上允许 NULL。`vm_load_string` 应该内建 NULL 检查，或者 syscall 层应该有一个 NULL-safe 的 wrapper。这是可以做一个系统性扫描的方向。

### Bug 2: `unshare` 的 flags 限制太窄

**现象**：最初的 `sys_unshare` 只支持 `CLONE_NEWUSER` 单一 flag。但 claw 实际调用的是 `unshare(CLONE_NEWUSER|CLONE_NEWNS|CLONE_NEWCGROUP|...)` 组合 flags → 返回 `EINVAL` → sandbox 初始化失败。

**修复**（commit `fa97f0e`）：扩展 bitmask 到全部 7 个标准 namespace flags，非 NEWUSER 的作为 no-op。这个修复是"接到真实测试输出才发现的"——在写代码时，天然倾向是"先实现最核心的 NEWUSER，其他再说"，但真实应用的行为恰好不允许循序渐进。

## 九、遗留问题与后续方向

### 9.1 seccomp 的真实缺失

当前 `sys_seccomp()` 和 `prctl(PR_SET_SECCOMP)` 都是 no-op。这意味着如果一个恶意脚本被 claw 执行，它可以发起任意 syscall，不受任何限制。在真正的多租户场景中，这需要从以下方向补齐：

- 实现最小 BPF 解释器（至少支持 `RET_ALLOW` / `RET_KILL` 两种 action）
- 或者与现有的 seccomp BPF 库（如 `libseccomp` 的 Rust binding）集成

### 9.2 cgroup 的 mock 局限性

目前 `/proc/self/cgroup` 永远返回 `"0::/"`。如果将来有任何应用依赖 cgroup 的实际资源限制（而不只是"读取 cgroup 做自检"），就需要进一步实现 cgroup 控制器的基本框架。至少需要 `cpu.max` 和 `memory.max` 两个核心控制文件。

### 9.3 其他 namespace 的语义缺口

`CLONE_NEWPID`、`CLONE_NEWNET` 在 `clone` 和 `unshare` 中都是 no-op。如果需要让多个 claw 实例真正隔离：

- PID namespace：需要实现 PID 映射表，让 namespace 内的 init 进程看到 PID 1；
- 网络 namespace：需要支持 `veth` pair 和起码的 NAT 规则。

## 十、验证方式

```bash
# Goal 测试（5 个独立 C 测试，每个 ≤ 5 个用例）
cd test-suit/starryos/normal/qemu-smp1/claw-code
cargo xtask starry test qemu --arch x86_64 --test-case goal-01-unshare
cargo xtask starry test qemu --arch x86_64 --test-case goal-02-uid-map
cargo xtask starry test qemu --arch x86_64 --test-case goal-03-gid-map
cargo xtask starry test qemu --arch x86_64 --test-case goal-04-setgroups
cargo xtask starry test qemu --arch x86_64 --test-case goal-05-cgroup

# Integration 测试（Rust binary，6 级 fail-fast）
cargo xtask starry test qemu --arch x86_64 --test-case claw-code-integration

# Robust 测试（12 个，含多 Agent 并发）
cargo xtask starry test qemu --arch x86_64 --test-case claw-code-robust-01
...
cargo xtask starry test qemu --arch x86_64 --test-case claw-code-robust-12
```

CI 通过 GitHub Actions 注入 `CLAW_API_KEY` secret，在编译时通过 `option_env!` 传入。测试超时最长 1800 秒（integration）和 600 秒（robust），成功标记 `ALL_TESTS_DONE`。

## 十二、总结

本实验通过 24 个提交（87 文件，+2536/-5 行），让 Claw Code 在 StarryOS 上成功运行。关键工作集中在三个方面：

**1. 沙箱 API 兼容层的构建**

实现了 `unshare(CLONE_NEWUSER)` + `/proc/self/{uid_map,gid_map,setgroups,cgroup}` 这套"**兼容性沙箱**"的基本骨架。

**2. 找到并修复 2 个关键 bug**

- `sys_mount` 不接受 NULL `fs_type` → claw sandbox 初始化失败；
- `sys_unshare` 只支持单一 `CLONE_NEWUSER` flag → claw 的组合 flags 调用被拒绝。

这两个 bug 都是典型的"Linux ABI 比内核实现的假设更宽容"模式——我们实现的每个 syscall 都**只考虑了最小合法参数**，但真实应用的调用比我们想象的更复杂（组合 flags、NULL 参数）。这种"假设断裂"只能靠**跑真实应用来发现**，代码审查和单测都抓不到。

**3. 建立了完整的测试链**

5 个 goal C 测试（每个 ≤ 5 个用例）→ 1 个 integration Rust binary（6 级 fail-fast）→ 12 个 robust Rust binary（含多 agent 并发）。