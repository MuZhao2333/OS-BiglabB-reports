# 任务二 · 实验二：扩展 StarryOS 对 6 个 syscall 的支持

> 目标仓库：[rcore-os/tgoskits](https://github.com/rcore-os/tgoskits)，dev 分支
>
> 选题：第一组（进程时间统计）`times` / `clock_getres`；第二组（文件空间管理）`fallocate` / `fadvise64` / `truncate` / `ftruncate`
>
> 工作流：基于 [OSAgent](../exp1_OSAgent/summary.md) 的 `debug-fix` skill 完成

## 一、实验目标

实验2 要求 "基于 qemu，发现并修复 starry 中 bug，增加功能，支持 2~3 组 syscall 相关的源码级测例"。本次选择了两组共 6 个 syscall，都来自"已经存在但行为不对"的 bug 类型：

- **进程时间统计**：`times` / `clock_getres`。`times` 涉及子进程 CPU 时间统计语义；`clock_getres` 涉及无效 clock_id 的错误码返回。
- **文件空间管理**：`fallocate` / `fadvise64` / `truncate` / `ftruncate`。四个都是 ext4 / VFS 边界 syscall，重点在参数校验、错误码精确化和大文件防溢出。

## 二、Pipeline 在本实验中的作用

实验2 的全部修复都走 OSAgent 的 `debug-fix` skill，关键 step 在每个 PR 上都留下了清晰指纹：

1. **`git-sync-agent`** 同步 `dev` 并创建 `fix/<name>` 分支；
2. **Linux 基线先行**：在 WSL 上用 gcc 编 C 测试用例，先看到 Linux PASS 才允许去改内核；
3. **`test-runner-agent`** 在 Docker QEMU 跑 starry riscv64 测试套，**确认失败现象**；
4. **`code-explorer-agent`** 同时做三件事：研究 Linux 行为（man / strace）、追踪内核源码（grep）、定位根因；
5. **主会话实施修复**——固定模式是 **"early input validation + 错误映射到 `LinuxError::XXX`"**；
6. **`test-runner-agent`** 验证修复后所有 case 都 PASS；
7. **`pre-commit-agent`** 跑 `cargo fmt --check`；
8. **`pr-writer`** 按 `templates/pr-bugfix.md` 拼正文，rebase + push + `gh pr create`。

所有 6 个 PR 的标题和正文格式（Bug 概述 → Root Cause → 代码 diff → Before/After QEMU 输出 → Changes）来自同一份模板。

## 三、PR 总览

| PR | syscall | 类型 | 文件改动 | 状态 | 合并日期 |
|----|---------|------|---------|------|---------|
| [#257](https://github.com/rcore-os/tgoskits/pull/257) | `times` | bugfix（语义） | +288 / -9 | MERGED | 2026-04-18 |
| [#430](https://github.com/rcore-os/tgoskits/pull/430) | `clock_getres` | bugfix（参数校验） | +293 / -4 | MERGED | 2026-05-07 |
| [#441](https://github.com/rcore-os/tgoskits/pull/441) | `fallocate` | bugfix（跨 3 层） | +461 / -1 | MERGED | 2026-05-08 |
| [#444](https://github.com/rcore-os/tgoskits/pull/444) | `fadvise64` | bugfix（errno） | +410 / -19 | MERGED | 2026-05-09 |
| [#466](https://github.com/rcore-os/tgoskits/pull/466) | `truncate` / `ftruncate` | bugfix（边界条件） | +881 / -2 | MERGED | 2026-05-10 |


## 四、第一组：进程时间统计

### 4.1 `sys_times` —— 修正子进程 CPU 时间统计 [PR #257]

#### 问题

POSIX `times(struct tms *buf)` 应当返回当前进程的 user/system 时间和**已被 wait() 回收的子进程**累积 user/system 时间。原 StarryOS 实现直接复制：

```rust
// 原 buggy 代码
tms_cutime = tms_utime;
tms_cstime = tms_stime;
```

这意味着 `cutime == utime`，这在 Linux 上是不可能出现的关系。任何依赖 `tms_cutime - tms_utime` 来判断"子进程时间"的应用都会拿到错误值。

#### 根因

StarryOS 的 `ProcessData`（task 的进程私有数据）从来就没有"已回收子进程 CPU 时间"这个字段；`sys_waitpid` 在回收 zombie 时也没有做累加。

#### 修复（3 个文件）

**1. `os/StarryOS/kernel/src/task/mod.rs`** — `ProcessData` 加字段：

```rust
children_cpu_time: SpinNoIrq<(TimeValue, TimeValue)>,
```

加 `children_cpu_time()` getter 和 `add_child_cpu_time(utime, stime)` setter。

**2. `os/StarryOS/kernel/src/syscall/task/wait.rs`** — `sys_waitpid` 累加：

```rust
let (utime, stime) = child_proc.cpu_time_total();
parent.proc_data.add_child_cpu_time(utime, stime);
```

`cpu_time_total()` 取的是子进程所有线程的 CPU 时间总和（不能只取主线程，否则多线程子进程统计会偏低）。

**3. `os/StarryOS/kernel/src/syscall/time.rs`** — `sys_times` 改为从 `ProcessData` 读取：

```rust
let (cutime, cstime) = current().as_thread().proc_data.children_cpu_time();
tms.vm_write(Tms { tms_utime, tms_stime, tms_cutime: cutime, tms_cstime: cstime });
```

#### 测试

新增 `test-suit/starryos/normal/times/c/src/main.c`（129 行），覆盖：

- 无子进程时 `cutime / cstime == 0`；
- fork + wait 后子进程 CPU 时间正确累加；
- 单调性（多次 wait 之间时间不会回退）；
- Bug 回归（`cutime != utime`）。

CI：starry riscv64 qemu / run_container 全 PASS。

#### 思考

这个 bug 是语义建模问题。POSIX 把 wait 当作子进程时间的"切换时刻"，要求父进程在那一刻快照子进程的 CPU 时间并永久持有。`ProcessData` 模型里缺这一字段意味着 starry 之前从未真正实现过这条 POSIX 语义。

### 4.2 `sys_clock_getres` —— 校验无效 clock_id [PR #430]

#### 问题

POSIX `clock_getres(clockid_t, struct timespec *)` 应当：

- 对合法 clock_id（`CLOCK_REALTIME`、`CLOCK_MONOTONIC`、`CLOCK_MONOTONIC_RAW`、`CLOCK_BOOTTIME`、`CLOCK_PROCESS_CPUTIME_ID`、`CLOCK_THREAD_CPUTIME_ID`、`CLOCK_REALTIME_COARSE`、`CLOCK_MONOTONIC_COARSE`）返回相应精度，ret = 0；
- 对无效 clock_id（如 `-1`、`9999`）返回 `EINVAL`。

原 starry 实现对所有未识别的 clock_id 都只打 `warn!` 然后返回 1 μs 精度 + ret = 0，**假成功**。

#### 根因

`sys_clock_getres` 的 `match` 分支用了 `_ => /* warn + 默认精度 */` 作为兜底，缺少"未知 clock_id 是错误"的判断。

#### 修复

`os/StarryOS/kernel/src/syscall/time.rs`（11+/4-）：

```rust
pub fn sys_clock_getres(clock_id: __kernel_clockid_t, res: *mut timespec) -> AxResult<isize> {
    let resolution = match clock_id as u32 {
        CLOCK_REALTIME | CLOCK_MONOTONIC | CLOCK_MONOTONIC_RAW | CLOCK_BOOTTIME
        | CLOCK_PROCESS_CPUTIME_ID | CLOCK_THREAD_CPUTIME_ID => TimeValue::from_nanos(1),
        CLOCK_REALTIME_COARSE | CLOCK_MONOTONIC_COARSE => TimeValue::from_millis(4),
        _ => return Err(AxError::InvalidInput),  // → LinuxError::EINVAL
    };
    if let Some(res) = res.nullable() {
        res.vm_write(timespec::from_time_value(resolution))?;
    }
    Ok(0)
}
```

精度细分到 ns / ms 两档与 Linux 对齐。

#### 测试

新增 `test-suit/starryos/normal/qemu-smp1/test-clock-getres/c/src/main.c`（173 行）。修复前 23 PASS / 2 FAIL，修复后 25 PASS / 0 FAIL，CI 用时 274.60 s。

#### 思考

这是典型的"假成功比假失败更危险"。`warn!` 给了开发者一种"内核已经处理过了"的错觉，但实际行为悄悄偏离了 POSIX。修复后的 `_ => Err(...)` 是更"诚实"的写法。`workflow.md` 里专门列了 errno 速查表，`InvalidInput → EINVAL` 是默认转换。

## 五、第二组：文件空间管理

### 5.1 `sys_fallocate` —— 参数校验 + 错误码精确化 + 大文件防溢出 [PR #441]

#### 问题

`fallocate(fd, mode, offset, len)` 为文件预分配空间。原实现存在 5 类 bug：

1. `mode != 0` 返回 `EINVAL`（应为 `EOPNOTSUPP`，`FALLOC_FL_KEEP_SIZE` 等不是无效参数而是不支持的操作）；
2. `offset < 0` / `len <= 0` 不校验，`i64 as u64` 之后变成 1.8e19 巨值，返回假成功；
3. `offset + len` 不查溢出，没有 `checked_add`；
4. 超大 `offset`（例如 `2^60`）通过后，ext4 内部 LBN 是 u32，`new_blocks as u32` 包裹回绕，分配 1 个块却把 inode 大小设为 1 EB，**元数据与实际块严重不一致**；
5. rsext4 的 `Errno::EFBIG` 未映射到 `LinuxError::EFBIG`，落入 `_ => EIO` 丢失语义。

#### 修复（跨 3 个 crate）

**1. `os/StarryOS/kernel/src/syscall/fs/io.rs:168-192`**（syscall 校验层）：

```rust
pub fn sys_fallocate(fd, mode, offset, len) -> AxResult<isize> {
    if mode != 0 { return Err(AxError::OperationNotSupported); }
    if offset < 0 || len <= 0 { return Err(AxError::InvalidInput); }
    let end = (offset as u64).checked_add(len as u64)
        .ok_or(AxError::from(LinuxError::EFBIG))?;
    if end > u32::MAX as u64 * 4096 {  // ext4 16TB 上限
        return Err(AxError::from(LinuxError::EFBIG));
    }
    let f = file_or_espipe(fd)?;
    let inner = f.inner();
    let file = inner.access(FileFlags::WRITE)?;
    file.set_len(file.location().len()?.max(end))?;
    Ok(0)
}
```

**2. `components/rsext4/src/file/io.rs:46-55`**（ext4 块号防回绕）：

```rust
let new_blocks = if truncate_size == 0 { 0u64 }
                 else { truncate_size.div_ceil(block_bytes) };
if new_blocks > u32::MAX as u64 {
    return Err(Ext4Error::new(Errno::EFBIG));
}
```

**3. `os/arceos/modules/axfs-ng/src/fs/ext4/rsext4/util.rs:21`**（错误码映射补充）：

```rust
rsext4::error::Errno::EFBIG => ax_errno::LinuxError::EFBIG,
```

#### 测试

新增 `test-suit/starryos/normal/qemu-smp1/test-fallocate/c/src/main.c`（335 行）。修复前 33 PASS / 8 FAIL，修复后 41 PASS / 0 FAIL，与 WSL Linux 基线完全一致。

#### Review 反馈与遗留

ZR233 在 review 中指出**一个未在本 PR 解决的问题**：errno 优先级——当前实现把参数校验放在 `file_or_espipe(fd)` 前，导致 `fallocate(-1, 0xdead, 0, 4096)` 返回 `EOPNOTSUPP`，而 Linux 严格按"先查 fd 再查参数"返回 `EBADF`。这一点在合并版中尚未修复，是后续可优化方向。

#### 思考

这是本组工作中**唯一一个跨 3 个 crate** 的修复，难点在于"找到所有 EFBIG 应该被传递的层"。`fallocate` 的 bug 暴露了一条隐藏假设："syscall 层做了校验就够"——但 ext4 内部的 LBN 上限是 ext4 自己的实现细节，syscall 层无从知晓，必须在 ext4 layer 也加一道防线。这种"防御链每一层都要补"的模式在 ext4 这类老格式上特别常见。

### 5.2 `sys_fadvise64` —— EBADF / EINVAL 校验 [PR #444]

#### 问题

`posix_fadvise(fd, offset, len, advice)` 给内核提示文件访问模式。原实现：

```rust
if Pipe::from_fd(fd).is_ok() {
    return Err(AxError::from(LinuxError::ESPIPE));
}
```

只在 fd 是**有效 pipe** 时拦截；对 `fd = -1` 或已关闭的 fd，`Pipe::from_fd` 返回 `Err`、`is_ok()` 为 false，直接落到 `Ok(0)` —— **假成功**。`len < 0` 也不校验。

#### 修复

`os/StarryOS/kernel/src/syscall/fs/io.rs:235-249`（10+/2-）：

```rust
pub fn sys_fadvise64(fd, offset, len, advice) -> AxResult<isize> {
    if len < 0 { return Err(AxError::InvalidInput); }
    if advice > 5 { return Err(AxError::InvalidInput); }
    let _ = file_or_espipe(fd)?;  // 替换原 Pipe::from_fd 检查
    Ok(0)
}
```

`file_or_espipe(fd)?` 是 starry 已有 helper：无效 / 已关闭 fd → `EBADF`，普通文件 → 通过，pipe → `ESPIPE`。

#### 测试

新增 `test-suit/starryos/normal/qemu-smp1/test-fadvise64/c/src/main.c`（219 行）+ 顺带修订 fallocate 测试 17 行。修复前 21 PASS / 3 FAIL，修复后 24 PASS / 0 FAIL。

#### 思考

`fadvise` 是个"建议性 syscall"——内核完全可以忽略它的 advice 而仅做参数校验。这种 syscall 修复的边界很容易模糊，到底应该"实现所有校验"还是"反正是建议，校验做到差不多就行"？OSAgent 的 `debug-fix` 流程给出的态度是"校验必须对齐 Linux"，但 review 的反馈说明边界还可以更严。

### 5.3 `sys_truncate` / `sys_ftruncate` —— 空路径、超大长度、只读文件、目录 fd [PR #466]

#### 问题

两个 syscall 各有 4 类边界条件 bug：

| 场景 | Linux 行为 | 原 StarryOS |
|------|-----------|-------------|
| 空路径 `""` | `ENOENT` | `EISDIR`（被解析成 `/`）|
| 超大 `length` | `EFBIG` | 假成功（ret = 0）|
| 只读文件 truncate | `EACCES` | 假成功 |
| 目录 fd ftruncate | `EINVAL` | `EISDIR` |

#### 修复

`os/StarryOS/kernel/src/syscall/fs/io.rs:144-166`（35+/2-）：

```rust
pub fn sys_truncate(path: UserConstPtr<c_char>, length: __kernel_off_t) -> AxResult<isize> {
    let path = path.get_as_str()?;
    if path.is_empty() { return Err(AxError::from(LinuxError::ENOENT)); }
    if length < 0 { return Err(AxError::InvalidInput); }
    if (length as u64) > u32::MAX as u64 * 4096 {
        return Err(AxError::from(LinuxError::EFBIG));
    }
    let file = OpenOptions::new()
        .write(true)
        .open(&FS_CONTEXT.lock(), path)?
        .into_file()?;
    let metadata = file.location().metadata()?;
    if !metadata.mode.contains(NodePermission::OWNER_WRITE) {
        return Err(AxError::from(LinuxError::EACCES));
    }
    file.access(FileFlags::WRITE)?.set_len(length as _)?;
    Ok(0)
}

pub fn sys_ftruncate(fd: c_int, length: __kernel_off_t) -> AxResult<isize> {
    if length < 0 { return Err(AxError::InvalidInput); }
    if (length as u64) > u32::MAX as u64 * 4096 {
        return Err(AxError::from(LinuxError::EFBIG));
    }
    let f = File::from_fd(fd).map_err(|e| {
        if e == AxError::IsADirectory { AxError::from(LinuxError::EINVAL) } else { e }
    })?;
    f.inner().access(FileFlags::WRITE)?.set_len(length as _)?;
    Ok(0)
}
```

#### 测试

- `test-suit/starryos/normal/qemu-smp1/test-truncate/c/src/main.c`（308 行）：修复前 41 PASS / 3 FAIL → 44 PASS / 0 FAIL；
- `test-suit/starryos/normal/qemu-smp1/test-ftruncate/c/src/main.c`（360 行）：修复前 49 PASS / 2 FAIL → 51 PASS / 0 FAIL。

#### Review 反馈与遗留

ZR233 给出了**这一组 PR 中反馈最详细的 5 条 review**：

1. **权限检查只看 `OWNER_WRITE` bit 不够**：Linux 真正语义是按当前凭据（fsuid / groups）检查 owner / group / other 写位并允许 root 绕过；当前实现会让 `fsuid=0` 被 0444 拒绝、非所有者却能写 0200。建议复用现有 `faccess`-类抽象；
2. **测试 root 凭据下期望 EACCES 不稳定**：Starry QEMU 默认 root 运行，Linux 和 starry 的 root 都会绕过 0444 写检查，所以这条 assert 在 `cargo xtask` 中实际拿到 ret = 0；
3. **EFBIG 检查放在路径解析前导致 errno 优先级反转**：`truncate("/tmp/no-such", 1<<60)` Linux 返回 `ENOENT`、`truncate("/tmp", 1<<60)` 返回 `EISDIR`，当前实现都先返回 `EFBIG`；
4. **`ftruncate(-1, huge)` 应 `EBADF` 但当前返回 `EFBIG`**：同样的 errno 优先级问题；
5. **O_APPEND 测试缺陷**：小节标题说覆盖 O_APPEND，但 open flags 实际没带，覆盖缺口仍在。

#### 思考

POSIX 的错误码优先级是一种"层叠校验"语义：**fd 错误优先于参数错误，参数错误优先于权限错误**。一旦把检查顺序写反，errno 就会和 Linux 偏差。本 PR 的 5 条 review 几乎每条都指向同一根问题。要彻底对齐，需要把所有 "early validation pattern" 改成 "validate after fd lookup"——这是后续 PR 的统一改造方向。

## 六、共性分析

写完 6 个 syscall 修复，归纳出几类问题模式：

### 1. 假成功比假失败更危险

`clock_getres`、`fadvise64`、`truncate / ftruncate` 都有"对错误输入返回 0"的 bug。这种"假成功"会让上游应用拿到错误状态后继续走逻辑，最终在更深层崩溃，调试链很长。Linux 严格的 errno 返回是一道关键护栏。

### 2. errno 优先级是反复踩坑点

6 个 PR 中有 4 个收到 review 反馈说错误优先级偏差。Linux 的隐含约定（fd → 参数 → 权限 → 实际操作）在 starry 上没有形成统一的代码骨架。这是后续应该抽象出一个 "validation pipeline" 的强信号。

### 3. 跨层防御链

`fallocate` 涉及 starry-kernel → axfs-ng → rsext4 三层，任何一层缺一道 EFBIG 检查都会让 16TB 上限失守。这种"每层都要补一刀"的模式在 ext4 这类老格式上很常见。

### 4. 语义建模缺失

`times` 的 `cutime` bug 不是参数校验，而是 starry 一开始就没建模"已回收子进程 CPU 时间"这个字段。这种语义建模缺失只能靠对照 POSIX spec 逐项核对，自动化工具帮不了多少。

## 七、涉及的 starry / arceos 组件

| 组件 | 修改的文件 | 修改的原因 |
|------|-----------|-----------|
| `starry-kernel/syscall/time.rs` | clock-related syscall | clock_getres 校验、times 读 cutime |
| `starry-kernel/syscall/fs/io.rs` | 文件 I/O syscall | fallocate / fadvise64 / truncate / ftruncate |
| `starry-kernel/syscall/task/wait.rs` | 进程回收 | waitpid 累加子进程时间 |
| `starry-kernel/task/mod.rs` | ProcessData | 新增 `children_cpu_time` 字段 |
| `arceos/modules/axfs-ng` | VFS / ext4 backend | rsext4 错误码映射补 EFBIG |
| `components/rsext4` | ext4 file I/O | LBN u32 防溢出 |

底层依赖：`axhal::time::TimeValue`、`kspin::SpinNoIrq`、`LinuxError / AxError` 转换层。

## 八、验证方式

每个 PR 的标准化验证命令清单（来自 OSAgent `workflow.md`）：

```bash
git diff --check
cargo fmt --check
cargo xtask clippy --package starry-kernel
cargo xtask starry test qemu --arch riscv64 --test-group normal --test-case <test-name>
```

CI 跑全 4 架构（x86_64 / aarch64 / riscv64 / loongarch64），全部 PASS 才允许合并。Linux 基线对照用 WSL 上的 gcc 编 C 文件后直接 `./a.out` 跑。