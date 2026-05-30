# 任务二 · 实验三：修复 BusyBox 在 StarryOS 上的若干失效功能

> 目标仓库：[rcore-os/tgoskits](https://github.com/rcore-os/tgoskits)，dev 分支
>
> 选题来源：[linux-compatible-testsuit#13](https://github.com/rcore-os/linux-compatible-testsuit/issues/13) 列出的 48 个被从 `busybox-tests.sh` 移除的 FAIL 用例
>
> 工作流：基于 [OSAgent](../exp1_OSAgent/summary.md) 的 `busybox-fix` skill 完成

## 一、实验目标

StarryOS 在 `os/StarryOS` 子模块中通过 `test-suit/starryos/normal/qemu-smp1/busybox/sh/busybox-tests.sh` 完成 BusyBox 通用功能的回归覆盖。但 issue #13 列出了 48 个被从脚本中**剔除**的 FAIL 用例，每个都代表 starry 上未对齐 Linux 行为的一个 applet。这些用例如果不修，starry 就不能声称"通过 busybox 测试套"。

本实验提交 4 个 PR（#489 / #491 / #517 / #521），目标是从 issue #13 列表里逐条勾掉：要么在内核侧补 ioctl / loader / execve 逻辑，要么在测试脚本里把曾经被剔除的用例加回来，**并保证它们 PASS**。

所有 4 个 PR 最终都已合并进 `dev` 分支。

## 二、Pipeline 在本实验中的作用

实验3 的全部修复都走 OSAgent 的 `busybox-fix` skill。关键 step 在每个 PR 上都留下了清晰指纹：

1. **`git-sync-agent`** 同步 `dev` 并创建 `fix/busybox-<applet>` 分支；
2. **`grep`** 确认该 applet 没在测试脚本里；
3. **`WebFetch`** 直接从 issue #13 取"测试命令"列和"验证方式"列，**逐字符照抄**到 `busybox-tests.sh`；
4. **`test-runner-agent`** 在 Docker QEMU 跑 busybox 测试 → 确认失败；
5. **多阶段 debug**（关键 step）：
   - 5a. 主会话直接在 WSL 跑 `strace -f busybox <applet>` 看真实 Linux 上的 syscall 流；
   - 5b. 主会话读 `qemu-failure.log` + `strace.log`，定位失败的 syscall；
   - 5c. 主会话直接 Read 内核源码（`syscall/mod.rs`、`syscall/fs/io.rs` 等）；
   - 5d. **只有定位不到时**才调 `code-explorer-agent` 做窄查询；
6. **主会话实施修复**——补 ioctl handler / 调整 syscall handler / 调整设备注册；
7. **`test-runner-agent`** 验证修复后整个 busybox 套件 PASS；
8. **`pre-commit-agent`** 跑 `cargo fmt --check`；
9. **`pr-writer`** 提 PR，commit message `fix(busybox-<applet>): <brief>`。


## 三、PR 总览

| PR | applet | 内核改动 | 测试改动 | 状态 | 合并日期 |
|----|--------|----------|---------|------|---------|
| [#489](https://github.com/rcore-os/tgoskits/pull/489) | `blockdev --getss /dev/loop0` | `pseudofs/dev/loop.rs` 加 `BLKSSZGET / BLKPBSZGET` ioctl | 失败分支加调试输出 | MERGED | 2026-05-10 |
| [#491](https://github.com/rcore-os/tgoskits/pull/491) | `blkid` | `syscall/fs/ctl.rs` 抑制 `BLKGETSIZE64/BLKRAGET/BLKSSZGET` 在非块设备 fd 上的 warn | 新增 `busybox_blkid` 用例 | MERGED | 2026-05-10 |
| [#517](https://github.com/rcore-os/tgoskits/pull/517) | `run-parts`（无 shebang 脚本）| `syscall/task/execve.rs` 加 ENOEXEC → `/bin/sh` 回退 | 新增 `busybox_run_parts` 用例 | MERGED（5 次提交迭代）| 2026-05-24 |
| [#521](https://github.com/rcore-os/tgoskits/pull/521) | `hwclock -r` | `pseudofs/dev/mod.rs` 移除假的 `/dev/rtc0` 注册 | 新增 `busybox_hwclock` 用例 | MERGED | 2026-05-11 |


## 四、PR #489 —— blockdev: 补 BLKSSZGET / BLKPBSZGET ioctl

### 问题

`busybox blockdev --getss /dev/loop0` 查询 loop 设备的逻辑扇区大小。在 starry 上直接返回 `ENOTTY`，命令失败。

### 根因

`os/StarryOS/kernel/src/pseudofs/dev/loop.rs` 的 loop 设备 ioctl 处理器只识别 `BLKGETSIZE / BLKGETSIZE64 / BLKRAGET / BLKRASET / BLKROGET / BLKROSET / BLKDISCARD`。`blockdev --getss` 走的是 `BLKSSZGET` (0x1268)，`--getpbsz` 走 `BLKPBSZGET` (0x127B)。原代码里 `BLKSSZGET` 实际已有一段 `(arg as *mut u32).vm_write(512)?;`，但 `BLKPBSZGET` 完全没接，所以走 default 分支返回 `ENOTTY`。

### 修复

`os/StarryOS/kernel/src/pseudofs/dev/loop.rs`：

```diff
-    ioctl::{BLKDISCARD, BLKGETSIZE, BLKGETSIZE64, BLKRAGET, BLKRASET, BLKROGET, BLKROSET, BLKSSZGET},
+    ioctl::{
+        BLKDISCARD, BLKGETSIZE, BLKGETSIZE64, BLKPBSZGET, BLKRAGET, BLKRASET, BLKROGET, BLKROSET,
+        BLKSSZGET,
+    },
...
-            BLKSSZGET => {
-                (arg as *mut u32).vm_write(512)?;
-            }
+            BLKSSZGET | BLKPBSZGET => {
+                (arg as *mut u32).vm_write(512)?;
+            }
```

把 `BLKSSZGET` 和 `BLKPBSZGET` 合并为同一个 match arm，都返回 512 字节扇区。`test-suit/.../busybox-tests.sh` 顺手在失败分支加打印 `(rc=$_rc)`，便于后续定位。

### 测试

CI：starry x86_64 / aarch64 / riscv64 / loongarch64 QEMU + Clippy + fmt 全绿。Review：ZR233 直接 APPROVE，赞同把两个 ioctl 合并的重构。

### 思考

这是 4 个 PR 中最朴实的一个：**补一个 match arm**。它说明 starry 的 ioctl 实现不是错的，只是覆盖不全。issue #13 真正的价值就在于"逐条找出未覆盖处"，否则没人会主动想起 `BLKPBSZGET` 这个偏门 ioctl。

## 五、PR #491 —— blkid: 抑制非块设备 fd 上的 ioctl warn

### 问题

`busybox blkid` / `busybox blkid /dev/null` 探测块设备属性。在 QEMU 上没有真正的块设备，BusyBox 会在非块 fd 上探 `BLKGETSIZE64 / BLKRAGET / BLKSSZGET` 等 ioctl，然后回退到读 `/etc/blkid.tab` 等用户态后备方式。整个流程**本来就允许失败**，但 starry 内核每次都 `warn!` 一堆噪声，把 qemu 日志刷得不可读。

### 根因

`os/StarryOS/kernel/src/syscall/fs/ctl.rs` 的 `sys_ioctl` 拿到 `AxError::NotATty` 时只对 `TIOCGWINSZ` 静默处理，其他 ioctl 一律打印 `Unsupported ioctl command: ... for fd: ...`。BusyBox `blkid` 这种合规探测被刷屏。脚本侧因此一直没把 `blkid` 写成正式回归用例（怕日志被淹没看不清）。

### 修复

`os/StarryOS/kernel/src/syscall/fs/ctl.rs`：

```diff
-    ioctl::{FIONBIO, TIOCGWINSZ},
+    ioctl::{BLKGETSIZE64, BLKRAGET, BLKSSZGET, FIONBIO, TIOCGWINSZ},
...
-                // glibc likes to call TIOCGWINSZ on non-terminal files, just ignore it
-                if cmd == TIOCGWINSZ {
+                // Applications commonly probe non-terminal/blobk fds with these ioctls; suppress noise.
+                if matches!(cmd, TIOCGWINSZ | BLKGETSIZE64 | BLKRAGET | BLKSSZGET) {
                     return;
                 }
                 warn!("Unsupported ioctl command: {cmd} for fd: {fd}");
```

`test-suit/.../busybox-tests.sh` 新增 `busybox_blkid`，主要校验 `busybox blkid /dev/null` 的 exit code 为 0（错误信息走 stderr）。

### 测试

CI 全过；ZR233 APPROVE，但指出注释中有 typo `blobk -> block`（合入时仍保留 typo，未阻塞）。

### 思考

"内核 warn 太啰嗦其实只挡测试，没挡功能"——这是个微妙的 case。`warn!` 不是错，但它让回归脚本无法区分"本应有的失败 path"和"内核出问题"。`matches!` 白名单一次拓宽多个 ioctl 是简洁的解。这个 PR 也暴露了"`Unsupported ioctl` 默认 warn"是一条该重新审视的策略——可能应该改成 `debug!` 或者完全静默。

## 六、PR #517 —— run-parts: ENOEXEC fallback（5 次提交迭代）

### 问题

`busybox run-parts /tmp/bb_rp/d` 依次执行目录下所有可执行文件。当文件**没有 shebang 且不是 ELF**（例如纯文本 `rp_ok`）时，Linux 用户态会让 `execve` 返回 `ENOEXEC`，再由 libc `execvp` 或 BusyBox 自身重试 `/bin/sh <文件>`。

starry 之前直接返回 `AxError::InvalidExecutable`，BusyBox 收到后报 `Exec format error` 而不会走 fallback。

### 根因

三层问题嵌套：

1. ELF Loader 在 `os/StarryOS/kernel/src/mm/loader.rs` 的 `load_user_app` 中只识别 ELF 和 shebang；其他情况一律返回 `Err(AxError::InvalidExecutable)`；
2. BusyBox 自身的 ENOEXEC 重试依赖 `/proc/self/exe` 重新 `readlink` 自己再 `execve /bin/sh <file>`，而 starry 当时的 procfs 实现还不能完整支持；
3. `fork` / `vfork` 的 syscall 分发只在 `target_arch = "x86_64"` 下注册，而 `run-parts` 在其他架构上需要 `vfork`。

### 修复（最终合入版本）

**`os/StarryOS/kernel/src/syscall/task/execve.rs`**（在 syscall handler 层捕获 `InvalidExecutable` 并通过 `/bin/sh` 重试）：

```diff
-    let new_name = loc.name();
-    let new_exe_path = loc.absolute_path()?.to_string();
+    let mut new_name = loc.name().to_string();
+    let mut new_exe_path = loc.absolute_path()?.to_string();
...
-    let (entry_point, user_stack_base) =
-        load_user_app(&mut new_aspace, Some(path.as_str()), &args, &envs)?;
+    let (entry_point, user_stack_base) =
+        match load_user_app(&mut new_aspace, Some(path.as_str()), &args, &envs) {
+            Ok(result) => result,
+            Err(AxError::InvalidExecutable) => {
+                // ENOEXEC fallback: retry via /bin/sh.
+                // In Linux this retry is done by user-space (execvp / busybox),
+                // not by the kernel. This is a pragmatic workaround until
+                // musl's execvp or busybox's ENOEXEC handling is available.
+                let shell_path = "/bin/sh";
+                let shell_loc = FS_CONTEXT.lock().resolve(shell_path)?;
+                new_name = shell_loc.name().to_string();
+                new_exe_path = shell_loc.absolute_path()?.to_string();
+                args = iter::once(String::from(shell_path))
+                    .chain(args.iter().cloned())
+                    .collect();
+                load_user_app(&mut new_aspace, None, &args, &envs)?
+            }
+            Err(e) => return Err(e),
+        };
```

`mm/loader.rs` 保留 `InvalidExecutable` 的原始返回，**loader 层没有被污染**——这是 review 反复强调的边界。

测试用例：

```bash
_t=$({ timeout 10 sh -c "busybox sh -c 'mkdir -p /tmp/bb_rp/d && busybox echo rp_ok > /tmp/bb_rp/d/00t && chmod +x /tmp/bb_rp/d/00t && busybox run-parts /tmp/bb_rp/d' 2>&1"; } 2>&1)
if echo "$_t" | grep -qF "rp_ok"; then echo "PASS: busybox_run_parts"; ...
```

### CI 与 Review

#### 第 1 版

在 `load_user_app` 内部直接做 `/bin/sh` fallback。

- **ZR233 CHANGES_REQUESTED**："把 `/bin/sh` fallback 放进 raw execve 会改变 Linux 内核语义，仍需要调整实现边界并补 ENOEXEC 回归。" 原始 Linux 语义是内核返回 `ENOEXEC`、用户态决定怎么处理；把 fallback 塞进内核会让"直接调用 `execve` 期望收到 `ENOEXEC`"的程序拿不到正确错误码；
- **mai-team-app[bot]**（glm / mimo 多模型自动 review）**CHANGES_REQUESTED**：同样指出 ENOEXEC 语义偏离，并补充建议：保持 loader 层返回 `InvalidExecutable`，让 musl 的 `execvp` 或者 `/proc/self/exe` 来重试；同时要求新增 ENOEXEC 回归测试。

#### 第 2 版（commit `cb83a458`）

把 fallback 从 `load_user_app` 上移到 `sys_execve`。

- mai-team-app 二次 review 仍标记 CHANGES_REQUESTED：架构上更合理（syscall 层而非 loader 层）但 fallback 本身的语义问题仍在；同时指出 `cargo fmt --check` 失败、与 `dev` 有冲突需要 rebase。

#### 第 3+ 版（commit `54843ba` 及之后）

- 修复 `cargo fmt`、rebase；
- 把注释改成更诚实的描述："In Linux this retry is done by user-space, not by the kernel. This is a pragmatic workaround"，并显式声明这是临时方案（直到 musl `execvp` 或 BusyBox 自带的 ENOEXEC 处理可用）；
- mai-team-app COMMENTED：肯定改进点 —— loader 语义恢复正确、fallback 移至 syscall 层架构合理、注释诚实、`new_aspace` 复用合理、`shell_loc` 经 FS_CONTEXT 解析（没硬编码）；继续标注 ENOEXEC 语义偏离但接受作为 workaround。最后一条 review 自动 DISMISSED；
- ZR233 最终 APPROVED。

CI：starry 4 架构 QEMU + licheerv-nano + orangepi-5-plus + axvisor + arceos 全部 PASS；fmt、clippy 通过。测试结果：PASS 276 / FAIL 1 → PASS 277 / FAIL 0。

### 思考

这是整个实验里讨论最多、迭代最复杂的 PR，演示了一个典型的"该不该在内核里做用户态行为"问题。即使能让测试 PASS，也要权衡是否破坏了 syscall 的对外语义。

最终合并的版本走了一个折中：

- **loader 层保持原 `InvalidExecutable` 返回**，对外语义不变；
- **syscall handler 内部做一次自动 retry**，效果上让 `execve` 这一个 syscall 看起来"内核帮你 fallback"；
- **注释明确写"临时 workaround"**，等待 musl `execvp` 或 procfs 完善后再清理。

这个折中本身也不完美——如果用户调 raw syscall 期望 `ENOEXEC`，他依然会拿到"成功执行 /bin/sh"。但作为一个临时让 busybox 跑起来的方案，已经是 review 能接受的最远边界。

这个 PR 也是 **mai-team-app[bot]** 第一次大规模介入的 PR。多模型 review（glm-5.1 / mimo-v2.5-pro 等）+ 人工 review 互相印证，对 PR 的语义边界讨论起到了推动作用。

## 七、PR #521 —— hwclock: 删除假的 /dev/rtc0

### 问题

`busybox hwclock -r` 从硬件时钟 `/dev/rtc0` 读取时间。issue #13 中 `busybox_hwclock` 的验证条件是 `grep -qF "hwclock"`，意图是匹配 BusyBox 在没有 RTC 时打印的错误信息（形如 `hwclock: can't open '/dev/misc/rtc': No such file or directory`）。

QEMU 默认没有 RTC 硬件，Linux 上这条命令也会失败并输出含 `hwclock` 的错误。但 starry 反而"成功了"，只输出干净的时间字符串，根本不含 `hwclock` 这个字串，测试因此 FAIL。

### 根因

`os/StarryOS/kernel/src/pseudofs/dev/mod.rs` 注册了一个**伪造的** `/dev/rtc0` 字符设备（由 `rtc` 模块提供），它对 `RTC_RD_TIME` ioctl 返回 wall-clock 时间。`hwclock -r` 看到 `/dev/rtc0` 存在且能读，直接打印时间。**伪实现比没实现还糟**。

### 修复

`os/StarryOS/kernel/src/pseudofs/dev/mod.rs`：

```diff
-mod rtc;
 pub mod tty;
...
-    root.add(
-        "rtc0",
-        Device::new(
-            fs.clone(),
-            NodeType::CharacterDevice,
-            rtc::RTC0_DEVICE_ID,
-            Arc::new(rtc::Rtc),
-        ),
-    );
```

删掉伪 RTC 注册和 `mod rtc`。`test-suit/.../busybox-tests.sh` 补回 `busybox_hwclock` 用例。

### 测试

CI：starry 四架构 QEMU + orangepi-5-plus 自托管板子 + axvisor + arceos + fmt + clippy 全部通过。ZR233 APPROVED 并备注本地复测三组测试通过。套件 PASS 276 / FAIL 1 → PASS 277 / FAIL 0。

### 思考

"返回成功并不一定等于语义正确"。这个 PR 的核心思路反直觉但正确——**删除一个看起来在工作的功能（假 RTC），让 BusyBox 走 Linux 在无 RTC 硬件时的标准失败路径，反而让回归用例对齐了 Linux 行为**。

这是个值得反思的模式：starry 在早期为了让某些命令"不报错"，加了不少 stub / fake 实现。随着兼容性测试的覆盖深入，这些 stub 反而成了对齐 Linux 的障碍。`hwclock` 是被发现的第一个，可能还有其他类似的 stub 等待被审视（fake `/dev/random`？fake socket？）。

## 九、整体心得与共性分析

### 五种典型 bug 模式

1. **缺漏 ioctl 命令字**（#489）：补一个 match arm，常规小修；
2. **驱动 warn 噪声 vs 测试可读性**（#491）：内核 warn 太啰嗦其实只挡测试不挡功能，用 `matches!` 白名单一次拓宽多个 ioctl；
3. **假实现比没实现还糟**（#521）：fake `/dev/rtc0` 让 `hwclock -r` 走不到 Linux 期望的错误路径，删掉它即修复；
4. **内核 vs 用户态语义分工**（#517）：把"用户态本应做的 ENOEXEC 重试"搬进内核能让测试 PASS，但会破坏 raw `execve` 语义。走了"在 syscall handler 内部 retry、不污染 loader 层、注释里写明是临时 workaround"的折中方案；

### Review 流程观察

- **ZR233**：项目内的人工 reviewer，覆盖了全部 4 个 PR；通常先 CHANGES_REQUESTED / COMMENT 一次，作者修完后再 APPROVE；
- **mai-team-app[bot]**（glm / mimo 等模型驱动）：在 #517 这种"语义敏感"的 PR 上提供了多轮意见，精确指出 fmt 失败、合并冲突、回归测试覆盖不足等技术细节；机器人 review 与人工 review 互相印证；
- **4 个 PR 中 3 个一次成型**（同日合并），唯一一个反复迭代的 #517 历时两周（2026-05-10 → 2026-05-24），共 5 次提交，覆盖三次实现路线（loader → execve、补 fmt、补 rebase）。

### 改动落到 starry 的哪些模块

| 模块路径 | 涉及 PR | 修复维度 |
|---------|---------|---------|
| `kernel/src/pseudofs/dev/loop.rs` | #489 | loop 设备 ioctl handler |
| `kernel/src/pseudofs/dev/mod.rs` | #521 | 字符设备注册（移除伪 RTC） |
| `kernel/src/syscall/fs/ctl.rs` | #491 | `sys_ioctl` warn 抑制白名单 |
| `kernel/src/syscall/task/execve.rs` | #517 | `sys_execve` 加 ENOEXEC fallback |
| `kernel/src/mm/loader.rs` | #517 | ELF loader 保持原语义 + 注释修正 |
| `test-suit/.../busybox-tests.sh` | 全部 4 个 | 测试用例补齐与调试输出 |


## 十、验证方式

每个 PR 的标准化验证命令清单（来自 OSAgent `workflow.md`）：

```bash
git diff --check
cargo fmt --check
cargo xtask clippy --package starry-kernel
cargo xtask starry test qemu --arch riscv64 --test-group normal --test-case busybox
```

CI 跑全 4 架构 + 物理板子（orangepi-5-plus / licheerv-nano），所有 PASS 才允许合并。