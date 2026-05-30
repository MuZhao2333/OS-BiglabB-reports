# 任务二 · 实验一：OSAgent —— AI 驱动的 StarryOS 持续迭代框架

> 仓库：[MuZhao2333/OSAgent](https://github.com/MuZhao2333/OSAgent)
>
> 定位：在 Claude Code 平台上为 `rcore-os/tgoskits`（StarryOS 内核 + 测试套件）提供「调试 → 修复 → 测试 → PR」的结构化工作流。

## 一、实验目标

实验1 要求每位同学搭建一套"AI 驱动的内核改进持续运行迭代框架"，把后续的实验2（syscall 修复）、实验3（busybox 修复）、实验4（大应用移植）等具体工作都跑在这套框架之上。OSAgent 就是为此目的而搭建的脚手架，它本身不写一行内核代码，但**任务2 实验2 和 实验3 全部 12 个 merged PR**都是它的工作产物。

## 二、整体定位与设计原则

OSAgent 是一套跑在 Claude Code 之上的 LLM 工作流编排系统。仓库结构：

```
OSAgent/                    ← 框架仓库（本仓库）
├── .claude/
│   ├── skills/            ← 6 个 workflow skill：剧本，定义"做某类任务的 step 序列"
│   └── agents/            ← 7 个 sub-agent：执行单元，每个只做一件事
├── docs/                  ← workflow / environment 速查
├── templates/             ← PR / 测试用例模板
├── config/                ← Docker 助手脚本
└── CLAUDE.md              ← Claude Code 自动加载的全局 prompt
└── tgoskits/              ← 目标仓库，独立 clone，本框架以 .gitignore 屏蔽
```

核心设计原则：

> **Skills define step-by-step workflows. The main session calls sub-agents at each step. No nesting — all agent calls come from the top-level session.**

即"扁平调度"：

- **Skill 层**：每个 skill 是一份 Markdown 剧本，规定主会话按 1, 2, 3, … 顺序做什么；
- **Sub-Agent 层**：每个 agent 是叶子节点，工具白名单收得很紧，做完就把结果写到 `outputs/` 里，只返回简短摘要给主会话；
- **主会话**：所有 sub-agent 调用都从顶层发出，禁止 sub-agent 再嵌套 sub-agent（仅 `test-agent` 例外）。

这种设计避免了 Claude Code 上常见的两类坑：(a) sub-agent 嵌套套娃导致 token 爆炸（其实似乎Claude Code本身就不允许sub-agent嵌套）；(b) "把所有 context 塞给一个万能 sub-agent"导致主会话失去判断力。

主会话保留全部"决策权"（读 strace、定位根因、写 patch），sub-agent 只承担"重 IO + 重计算 + 重文本"。这是 OSAgent 与一般"自主 agent"框架最不同的地方。

## 三、六个 Skill 详解

每个 skill 是一份 `.claude/skills/<name>/SKILL.md`，主会话进入 skill 后按 step 逐条调用 sub-agent。下表给出每个 skill 的定位与流水线骨架。

| Skill | 适用任务 | 流水线骨架 |
|-------|----------|------------|
| `busybox-fix` | busybox applet 修复 | git-sync → 取 issue #13 测试命令 → 加测试 → QEMU 失败确认 → strace → 修内核 → 验证 → fmt → PR |
| `debug-fix` | 通用内核 bug 修复 | git-sync → Linux 基线先行（WSL gcc） → QEMU 确认失败 → code-explorer 同时做"Linux 行为 + 内核源码 + 根因定位" → 修复 → 验证 → fmt → PR |
| `feature-dev` | 新 syscall 或内核功能 | git-sync → 研究 Linux 行为（signature / errno / edge case） → design → test-agent 写 C 测试 → 实现 → 测试 → fmt → PR |
| `app-port` | 大型应用移植 | git-sync → app-profiler 收集 syscall 集合 + gap analysis → plan.md → 逐 goal 循环（test-first + one-commit-per-goal） → 三层 integration test → PR |
| `claw-debug` | claw-code 测试驱动修内核 | 跑测试 → 主会话定位 → 修内核 → 记 progress.md → 重复，直到 ALL_TESTS_DONE（不 commit） |
| `claw-robust` | claw-code 健壮性测试 | 生成 prompt → C 测试 + qemu-x86_64.toml → 注入 rootfs → QEMU 跑 → 失败分类表 → 修对应仓库 → 重跑 |

### 1. `busybox-fix` —— 实验3 的核心 skill

10 个 step，专门解决 busybox applet 在 starry 上失败的问题。关键设计：

- **issue #13 是测例的来源**：用 `WebFetch` 直接读 `rcore-os/linux-compatible-testsuit/issues/13` 的"测试命令"列和"验证方式"列，**逐字符照抄**到 `busybox-tests.sh` 里。Skill 里专门写了一张"verification-pattern → shell `if`"映射表，把 issue 的自然语言验证表达式翻译成可执行的 shell 判断。
- **多阶段 debug**：Step 6 拆成 4 个子步骤（WSL 跑 strace → 读 qemu-failure.log → 主会话直接 Read 内核源码 → **只有定位不到时**才调 `code-explorer-agent` 做窄查询）。这个分级显式写在 SKILL.md 里，强调"不要把所有 context 都甩给 sub-agent"。
- **分支命名规则**：`fix/busybox-<applet>`，由 `git-sync-agent` 创建；commit message 形如 `fix(busybox-<applet>): <brief>`。

实验3 的 4 个 PR（#489 / #491 / #517 / #521）全部走这套流程。

### 2. `debug-fix` —— 实验2 的核心 skill

8 个 step，专门处理"已有 syscall 行为不对"的 bug。关键设计：

- **Linux 基线先行**：必须先在 WSL 上用 gcc 编 C 测试用例，看到 Linux 上 PASS 之后才允许去改内核——这条规定防止 LLM 凭空"修"出非 POSIX 的行为。
- **errno 映射规范**：所有错误要走 `AxError::from(LinuxError::XXX)`，禁止直接返回原生 `AxError`。
- **early validation pattern**：参数校验必须前置，与内部状态分离——这条规范直接体现在实验2 的 6 个 PR 上（fallocate / fadvise64 / truncate / ftruncate 全部走"先校验再操作"）。

实验2 的 6 个 PR（#257 / #430 / #441 / #444 / #466 + 顺带的 #280）全部走这套流程。

### 3. `feature-dev` —— 新增 syscall / 内核功能

TDD 流程：测试用例先写，且 Linux 基线先 PASS，再写内核实现。`test-agent` 在这里被复用，C 测试模板由 `templates/test-case.md` 提供。

### 4. `app-port` —— 大型应用移植（最复杂的 skill）

为实验4（claw-code on StarryOS）准备。关键设计：

1. `app-profiler-agent` 自动 clone 目标 app、build、`strace` 收集系统调用集合、对比 StarryOS 已实现的 syscall 表，产出 gap analysis 列表（含优先级）；
2. 主会话生成 `plan.md`：1:1 sub-goal ↔ test 映射，按依赖关系拓扑排序，定义三层 integration test（smoke / diagnostic / functional）；
3. **逐 goal 循环**：每个 sub-goal 标 `[~]`（进行中）→ 写测试 → WSL baseline PASS → 实现内核改动 → QEMU 验证 → fmt → **一个 sub-goal 一个 commit + push** → push 成功才能进入下一个；
4. 集成测试：smoke → diagnostic → functional 逐层升级，发现的新 bug 作为新 sub-goal 加进 `plan.md`；
5. 全绿后 `pr-writer` 提 PR。

强制 "one commit per sub-goal" 是为了**可 bisect**。`plan.md` 是结构化真源，`progress.md` 是时间轴日志，两者结合让 LLM 在长时间跨会话任务中保持节奏感。

### 5 / 6. `claw-debug` / `claw-robust`

为实验4 后期阶段准备：`claw-debug` 是"迭代修内核到 claw-code/integration 测试全过"（不 commit）；`claw-robust` 是"自动生成 prompt → QEMU 跑 claw → 失败分类（API 4xx 是 claw bug、kernel panic 是 kernel bug、ENOSYS 是 kernel bug、hang 多半是 epoll/tokio bug）→ 修对应仓库 → 重跑"。

## 四、七个 Sub-Agent 详解

每个 sub-agent 是单一职责的叶子节点，工具白名单收得很紧。

| Sub-Agent | 工具 | 职责 | 关键约束 |
|-----------|------|------|----------|
| `git-sync-agent` | `Bash` | 同步 `local/dev = origin/dev = upstream/dev`，创建工作分支 | 分支名严格 `fix/<x>` 或 `feat/<x>` |
| `code-explorer-agent` | Read / Bash / Grep / Glob / WebSearch / WebFetch | **"一次回答一个具体问题"**：定位符号、跑 strace（带 timeout）、查 man page | **明确禁止"全面报告"**，避免膨胀 |
| `test-runner-agent` | `Bash` | 跑 WSL 编 C 基线，或 Docker QEMU 跑 `cargo xtask starry test qemu` | 用 `tee` 保留 raw log，剥离 QEMU boot 噪声只返回 PASS/FAIL 摘要 |
| `test-agent` | Read / Bash / Edit / Write / **Agent** | 写 C 测试用例（**唯一可嵌套调 sub-agent**） | 可调 `code-explorer-agent` 查 Linux 行为，再调 `test-runner-agent` 跑双基线 |
| `pre-commit-agent` | `Bash` | 跑 `cargo fmt --check`（早期还含 clippy + sync-lint + std test，后被简化以避免权限和耗时） | 只做"会阻塞 PR"的检查 |
| `pr-writer` | Read / Bash / Edit / Write | 从 `outputs/` 抽 before/after log → 拼 PR 正文 → `git rebase upstream/dev` → `gh pr create` | **显式禁止 AI 署名行** |
| `app-profiler-agent` | Bash / Read / Write / WebFetch / Grep / Glob | clone 目标 app → build → strace → 对比 syscall 表 → gap analysis | 输出到 `outputs/app-port-<name>/profile.log` |

## 五、模板与文档

| 文件 | 作用 |
|------|------|
| `CLAUDE.md` | Claude Code 自动加载的 system prompt，定义两库结构、git workflow、Docker 限制、**"Fix the kernel, not the test" 硬规则** |
| `docs/workflow.md` | workflow 快速参考 + errno 速查表 |
| `docs/environment.md` | Docker 镜像 `starryos-dev:ubuntu-qemu10.2.1` 说明 |
| `templates/pr-bugfix.md` | Bugfix PR 模板：Bug 概述 → Root Cause → 代码 diff → Before/After QEMU 输出 → Changes 列表 |
| `templates/pr-feature.md` | Feature PR 模板：Motivation → Design → Implementation → Test Results |
| `templates/test-case.md` | C 测试模板 + 11 条覆盖类目 checklist（normal/null/negative/boundary/EBADF/EACCES/EISDIR/…） |
| `config/docker-helper.sh` | `run_test` / `build` / `run_local` 三个 bash 包装函数 |

`pr-writer` 直接引用 `templates/pr-bugfix.md` 和 `templates/pr-feature.md` 作为 PR 正文骨架；`test-agent` 直接引用 `templates/test-case.md` 作为 C 文件骨架。这种模板复用让所有 PR 看起来都来自同一套规范。

## 六、自动化流程

OSAgent 没有传统意义上的 hook 或外部 orchestrator —— **整个 orchestrator 就是 Claude Code 主会话 + Skill 文件**。

```
用户给任务（例如"修复 busybox blockdev"）
    ↓
主会话识别意图，加载对应 skill (busybox-fix)
    ↓
按 SKILL.md 的 Step 1 → Step N 顺序执行
    ↓
每个 Step 是一次 Agent(subagent_type=…, prompt=…) 调用
    ↓
Sub-agent 在隔离 context 下做完工作，把 raw log 写到 outputs/，返回摘要
    ↓
主会话读 outputs/ + 做决策（改哪行、加什么校验）+ 直接 Edit 内核代码
    ↓
最后一步固定是 pr-writer → 提 PR
```

亮点是把"重 IO"（QEMU 跑出来的几 MB 日志、strace 几万行）**软隔离**在 sub-agent context 里，只把"几十字摘要 + outputs/ 文件路径"返回给主会话：

- 主会话 context 不被 boot log 撑爆；
- 主会话需要细节时随时 `Read outputs/<file>`；
- 不同 step 之间靠 `outputs/` 文件做信息接力，**可断点续做**。

`outputs/` 目录约定 + `progress.md` 累积日志 + `plan.md` 真源文件 共同组成了"跨会话可恢复"的工作记忆。

## 七、各 Skill 在任务2 实验2、实验3 中的具体应用

### 实验2（6 个 syscall）—— 全部走 `debug-fix` skill

| PR | syscall | 触发的关键 step |
|----|---------|----------------|
| #257 | `times` | Linux 基线 → 发现 `cutime == utime` → code-explorer 追到 `ProcessData` → 修复 + waitpid 累加 |
| #430 | `clock_getres` | C 测试构造非法 clockid → Linux 返回 EINVAL → starry 假成功 → 加 early validation |
| #441 | `fallocate` | edge case 全覆盖：normal / negative / boundary / 不支持 mode → EOPNOTSUPP → 跨 starry-kernel / axfs-ng / rsext4 三层修 |
| #444 | `fadvise64` | 典型 errno 校验三件套：EBADF / EINVAL / ESPIPE 优先级 |
| #466 | `truncate / ftruncate` | ENOENT / EFBIG / EACCES / EISDIR 多种错误条件 |

这 6 个 PR 都有同一种模版：

1. PR 正文严格遵循 `templates/pr-bugfix.md` 结构（Bug summary → Root Cause → Before/After QEMU 输出）；
2. 修复内容都是 **"early input validation + 错误映射到 `LinuxError::XXX`"** —— 正是 `debug-fix` skill Step 5 强调的范式；
3. 测试用例都放在 `test-suit/starryos/normal/qemu-smp1/test-<name>/c/src/main.c`，由 `test-agent` 按 `templates/test-case.md` 生成；
4. 测试覆盖类目几乎一一对应模板里的 11 条 checklist。

### 实验3（4 个 busybox PR）—— 全部走 `busybox-fix` skill

| PR | applet | 触发的关键 step |
|----|--------|----------------|
| #489 | `blockdev --getss` | strace 发现 loop ioctl 未实现 → 在 `pseudofs/dev/loop.rs` 加 handler |
| #491 | `blkid` | warn 刷屏 → 在 `syscall/fs/ctl.rs` 把块设备 ioctl 加入静默白名单 |
| #517 | `run-parts` | strace 看到 execve 对非 ELF 文件失败 → 在 `sys_execve` 加 ENOEXEC fallback |
| #521 | `hwclock -r` | 删除假的 `/dev/rtc0` 注册，让 busybox 走 Linux 在无 RTC 时的标准失败路径 |

这些 PR 的共性：

1. **测试命令逐字符照抄 issue #13**：如 `_t=$({ timeout 10 sh -c "..."; } 2>&1)` 这个写法直接来自 `busybox-fix` SKILL Step 4 的模板；
2. **修复都从 strace 出发**：因为 SKILL Step 6a 就是"先 strace"；
3. **分支名 `fix/busybox-<applet>`**：与 `git-sync-agent` 创建分支的命名规则一致；
4. **commit message `fix(busybox-<applet>): <brief>`**：是 `busybox-fix` Step 10 规定的格式。

### 工作流命中度复盘

实验2 / 实验3 每个 PR 几乎都完整覆盖了 skill 描述的 step。

唯一持续被"打磨"的是 `pre-commit-agent` —— 它从最初的 `fmt + clippy + sync-lint + std test` 被多次简化（commit `c3d2950 simplify precommit`），最后只留 `cargo fmt --check`。原因是 clippy 等步骤跑得太慢、容易触发 sandbox 权限问题。

## 八、框架的工程进化

从 git log 可以看到 OSAgent 经历了 4 次范式变迁：

```
init (单一主 agent，所有事都自己干)
  → agents-only（把 sub-agent 从 main agent 拆出来）
  → skill-subagent（引入 skill 层，扁平 sub-agent）
  → simplify precommit（pre-commit 简化为 fmt-only）
  → app-port / claw-debug / claw-robust（实验4 阶段大改）
```

每一步都是对实际跑出来的痛点的回应：

- "把所有 context 塞给一个万能 agent" → context 爆炸 → 拆 sub-agent；
- "sub-agent 嵌套" → token 灾难 → 扁平规则；
- "pre-commit 跑 clippy 慢且常崩" → 减到只跑 fmt；
- "大型应用移植没法用单次 prompt 跑完" → app-port + plan.md / progress.md 二元模型。

## 九、创新点与亮点

1. **Skill + flat sub-agent 二分法**：把"决策"留在主会话（保留全局视野），把"重 IO"扔给 sub-agent（隔离 context 爆炸）。完全避开了 sub-agent 嵌套套娃带来的 token 灾难。
2. **`outputs/` 作为跨步骤共享内存**：所有 sub-agent 都把 raw log 用 `tee` 写到约定路径，主会话按需 Read。这是"文件系统级 RAG"，比塞进对话历史更省 token，也支持跨会话断点续做。
3. **"Fix the kernel, not the test" 硬规则**：在 `CLAUDE.md` 末尾用 critical rule 阻止 LLM 走捷径"改测试让它过"。issue #13 测试命令必须逐字符照抄、`...` 占位符也只允许填最小 setup —— 都是为了堵住作弊路径。
4. **issue-driven 工作流**：`busybox-fix` 直接 `WebFetch` issue #13 的"测试命令"列作为真源，把测试规约自动化地从 issue 翻译成 shell test、再翻译成 verification `if` 表达式（提供了一张完整的 pattern → shell 映射表）。
5. **PR 模板 + AI-branding 黑名单**：`pr-writer` 显式禁止 `🤖 Generated with Claude Code` 之类署名，输出贴近"正常人写的 PR"；同时强制从 outputs/ 拉真实 before/after 日志，杜绝占位符。
6. **app-port 的 plan.md + progress.md 二元模型**：plan 是结构化真源，progress 是时间轴日志，两者结合让 LLM 在长时间跨会话任务中保持节奏感。"One commit per sub-goal" + commit push 验证 gate 强制可 bisect。
7. **claw-robust 的跨仓库自动分类**：通过一张 "症状 → 根因 → 修哪个 repo" 表，把 application 层和 kernel 层 bug 的 fix-and-loop 自动化。这是把 OSAgent 自己当成"持续迭代框架"的体现。
