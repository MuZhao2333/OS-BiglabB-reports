# 任务一：ArceOS 五个基础练习

> 仓库：[MuZhao2333/tg-arceos-tutorial](https://github.com/MuZhao2333/tg-arceos-tutorial)（基于 `rcore-os/tg-arceos-tutorial` 的 `test` 分支 fork）
>
> 五个练习的代码改动统一汇总在 `test` 分支的提交 `18d0262`（"task1"）中。每个练习通过 git subrepo 嵌入到 `exercise-*/` 子目录下，对应上游 `arceos-org/exercise-*`。

## 一、整体目标

| 练习 | 体量 | 主要触达的组件 | 难点关键词 |
|------|------|----------------|-----------|
| exercise-printcolor | 1 行 | 应用 `main.rs` | ANSI SGR 转义 |
| exercise-hashmap | 5300+ 行 vendor | `axstd::collections` | no_std 哈希、`hashbrown` |
| exercise-altalloc | ~95 行 | `bump_allocator`、`axalloc` | 双端 bump 分配 + 三 trait |
| exercise-ramfs-rename | 主改 2 文件 | `axfs_ramfs`、`axfs` | VFS rename 递归与挂载点转发 |
| exercise-sysmap | 主改 1 文件 | `src/syscall.rs` | mmap 两种分支 + 用户地址空间 |

整条链路自上而下：

```
应用程序 (println!, std::fs::rename, mmap)
  → axstd（用户态标准库）
    → axfs / axfs_ramfs（虚拟文件系统）
    → axalloc / bump_allocator（堆与页分配）
    → axhal（串口、分页、随机源）
      → QEMU
```

五个练习正好沿这条链覆盖了不同层次。

---

## 二、exercise-printcolor — 让 banner 上色

### 任务

`scripts/test.sh` 跨 `riscv64 / x86_64 / aarch64 / loongarch64` 跑 `cargo xtask run`，对最后 10 行做两个 grep：
- 必须出现 `Hello, Arceos!`
- 必须出现 **非纯 reset** 的 ANSI SGR 序列：`\x1b\[[0-9;]*[1-9][0-9;]*m`

### 实现思路

只改一行应用代码：

```rust
// exercise-printcolor/src/main.rs:9
println!("[WithColor]: \x1b[1;32mHello, Arceos!\x1b[0m");
```

`\x1b[1;32m` 是"粗体 + 绿色"，`\x1b[0m` 复位样式。`axstd::println!` 通过 `axhal` 串口把这段字节原样送到 QEMU 串口，host 终端把它解析成颜色；grep 看的是字节而非视觉，所以管道也能匹配。

选择改应用层而不是 `axstd` / `axhal`：动静最小、不影响 banner 等其他打印、跨架构一份代码搞定。

### 注意

- Rust 字符串里 ESC 必须写 `\x1b`（或 `\033`），不支持 `\e`。
- 测试脚本的正则故意排除了 `\x1b[0m`，模板里只写 reset 是过不了的。

---

## 三、exercise-hashmap — 给 axstd 加上 HashMap

### 任务

crates.io 上的 `axstd@0.3.0-preview.1` 并没有 `collections::HashMap`，但应用源码 `src/main.rs` 直接 `use std::collections::HashMap`（其中 `std` 实为 `axstd`），跑 50 000 次 `insert + iter` 校验。测试脚本需要捕获两行：

```
test_hashmap() OK!
Memory tests run OK!
```

### 实现思路

1. **vendor `axstd` 到本地**：`cargo clone axstd@0.3.0-preview.1`，并修改根 `Cargo.toml`：
   ```toml
   axstd = { path = "./axstd", features = ["defplat", "alloc"], optional = true }
   ```
2. **在 axstd 里 re-export `hashbrown`**（`exercise-hashmap/axstd/src/lib.rs:54-62`）：
   ```rust
   #[cfg(feature = "alloc")]
   pub mod collections {
       pub use alloc::collections::{BTreeMap, BTreeSet, BinaryHeap, LinkedList, VecDeque};
       pub use hashbrown::{HashMap, HashSet};
   }
   #[cfg(feature = "alloc")]
   #[doc(no_inline)]
   pub use collections::{HashMap, HashSet};
   ```
3. **`axstd/Cargo.toml` 加上 `hashbrown = "0.16"`**。0.16 之后 `hashbrown` 默认 hasher 是 `foldhash`，纯 Rust 不依赖 `getrandom`，no_std 下能编过。

### 注意

- 选 `hashbrown` 而非接 `axhal::random()` + 自实现 hasher：成本最低、Rust 官方 `std::HashMap` 内部也是 `hashbrown` 的包装。
- 同时在 axstd crate 根 `pub use collections::{HashMap, HashSet}`，让 `use std::HashMap;` 这种 shortcut 也能编。
- `axstd/Cargo.toml` 头部有自动生成注释提醒"path 依赖被改成版本号"，本质上是 cargo clone 的归一化结果，不要手动还原。
- 必须开 `alloc` feature，否则整个 `collections` 模块被 cfg out。

---

## 四、exercise-altalloc — 实现 bump 分配器

### 任务

`exercise-altalloc/modules/bump_allocator/src/lib.rs` 是一个全 `todo!()` 的骨架：要实现 `EarlyAllocator<const PAGE_SIZE: usize>`，同时满足 `BaseAllocator`、`ByteAllocator`、`PageAllocator` **三个** trait。

`modules/axalloc/` 已经把 default feature 设为 `bump_allocator + level-1 + page-alloc-256m`，根 `Cargo.toml` 通过 `[patch.crates-io]` 切到本地。我们补好分配器后，全局分配器自动启用它。

测试需要捕获：

```
Running bump tests...
Bump tests run OK!
```

应用代码本身：`Vec::with_capacity(3_000_000)` 加 push + sort，强制走全局分配器约 24 MiB。

### 实现思路

经典的双端 bump 布局：

```
[ bytes-used | avail-area | pages-used ]
|            | -->    <-- |            |
start       b_pos        p_pos       end
```

数据结构：

```rust
pub struct EarlyAllocator<const PAGE_SIZE: usize> {
    start: usize, end: usize,
    b_pos: usize,       // 字节分配指针，从 start 向上
    p_pos: usize,       // 页分配指针，从 end 向下
    alloc_count: usize, // 字节区活跃计数
}
```

三个 trait 的关键实现：

- **字节 `alloc`**：`addr = (b_pos + align - 1) & !(align - 1)` 向上对齐；越过 `p_pos` 即 OOM。
- **字节 `dealloc`**：递减 `alloc_count`；归零时整体回退 `b_pos = start`（这是 bump 的本质——不支持单次释放，只能整体回收）。
- **页 `alloc_pages_at`**：`addr = (p_pos - size) & !(align - 1)`，先扣再向下对齐；越过 `b_pos` 即 OOM。
- **页 `dealloc_pages`**：空实现（bump 模式下页永不释放）。

`alloc_count.saturating_sub(1)` 防止下溢，是稳健性细节。

### 与 axalloc 的衔接

- `exercise-altalloc/modules/axalloc/src/default_impl.rs:34-36` 在 `#[cfg(feature = "bump_allocator")]` 分支把 `DefaultByteAllocator` 设为 `EarlyAllocator<PAGE_SIZE>`；
- `level-1` feature 让 `GlobalAllocator::init` 直接把整段堆交给 EarlyAllocator，字节端和页端共用同一段内存；
- `page-alloc-256m` 保证 24 MiB 的 Vec 装得下。

### 注意

- `alloc_pages_at` 的 `base` 参数被故意忽略——bump 不支持指定基址，这与 `level-1` 模式下 `unimplemented!("level-1 allocator does not support alloc_pages_at")` 的另一侧呼应。
- 不要同时启用 `bump_allocator` 和 `tlsf`/`slab`/`buddy`，`cfg_if` 只匹配第一个分支。
- 错误类型是 `axallocator::AllocError`（注意 `axallocator` 是 trait + 各种具体分配器的库，`axalloc` 是 `#[global_allocator]` 包装层，两个 crate 名要分清）。

---

## 五、exercise-ramfs-rename — 让 ramfs 支持 rename

### 任务

crates.io 上的 `axfs` / `axfs_ramfs` 都没有完整实现 `rename`。本练习强制使用 ramfs（`xtask` 故意把 `target/disk.img` 创建成 0 字节，virtio-blk 扫描失败 → fallback 到 `RamFileSystem`），所以 `std::fs::rename` 直接报 `Unsupported`。

应用代码做的事：`create_dir("/tmp")` → `create_file("/tmp/f1", "hello")` → `rename("/tmp/f1", "/tmp/f2")` → `print_file("/tmp/f2")` → 打印 `[Ramfs-Rename]: ok!`。

### 实现思路

调用链：

```
axstd::fs::rename
 → axfs::api::rename
   → axfs::root::rename            （处理"目标已存在则先删"的 POSIX 语义）
     → parent_node_of(old).rename
       → RootDirectory::rename     （挂载点转发：find_best_mount 拆分路径）
         → axfs_ramfs::DirNode::rename
           ├ 同目录 → rename_node（BTreeMap.remove + insert）
           └ 递归 → 子 DirNode.rename
```

关键代码：

**`axfs_ramfs/src/dir.rs:74-91`** — 同目录原子改名：

```rust
pub fn rename_node(&self, old_name: &str, new_name: &str) -> VfsResult {
    let mut children = self.children.write();
    if !children.contains_key(old_name) { return Err(VfsError::NotFound); }
    if children.contains_key(new_name)  { return Err(VfsError::AlreadyExists); }
    let node = children.remove(old_name).unwrap();
    children.insert(new_name.into(), node);
    Ok(())
}
```

**`axfs_ramfs/src/dir.rs:193-217`** — `VfsNodeOps::rename` 路径递归：拆 `split_path(src_path)` / `split_path(dst_path)`，若都在当前目录走 `rename_node`，否则下钻到子目录递归。

**`axfs/src/root.rs:599-607`** — 顶层 `rename` 函数（暴露给 `axfs::api::rename`）：

```rust
pub(crate) fn rename(old: &str, new: &str) -> AxResult {
    if parent_node_of(None, new).lookup(new).is_ok() {
        warn!("dst file already exist, now remove it");
        remove_file(None, new)?;
    }
    Ok(parent_node_of(None, old).rename(old, new).map_err(AxError::from)?)
}
```

### 注意

- 拆分职责：`axfs_ramfs` 只管单一 fs 内的递归；`axfs/root.rs` 管"是否跨挂载 / 目标已存在则删"这种顶层语义。
- 用 `children.write()` 写锁保护 `remove → insert` 的原子性。
- 不支持跨目录移动（应用代码的注释就是 "Only support rename, NOT move."），与测试要求吻合。



- `VfsNodeOps::rename` 在 `axfs_vfs` trait 上往往有默认 `Unsupported` 实现，必须显式 `impl VfsNodeOps for DirNode` 块里 override，否则外部走 trait 调用仍会落到默认实现。
- `target/disk.img` 必须是 0 字节，否则 axfs 不会 fallback 到 ramfs。
- vendor 进来的 axfs 体量很大（仅 `axfs/src/highlevel/file.rs` 就 1000+ 行），只改 `root.rs`，其余文件保留原样。

---

## 六、exercise-sysmap — 实现 sys_mmap

### 任务

内核加载 `/sbin/mapfile`（musl 静态 ELF）跑用户态，C payload 先 `creat` 写 `"hello, arceos!"`，再 `open` + `mmap(..., PROT_READ, MAP_PRIVATE, fd, 0)` 读回。`SYS_MMAP` 没有实现就会卡死。

期望两行输出：

```
Read back content: hello, arceos!
MapFile ok!
```

### 实现思路

**主体在 `src/syscall.rs`**，加 ~160 行：

1. **flags / prot bitflags**（`MmapProt`、`MmapFlags`）+ `From<MmapProt> for MappingFlags`（必带 `MappingFlags::USER`）。
2. **mmap 区向下增长**：`static MMAP_BASE: AtomicUsize = AtomicUsize::new(0x3_0000_0000);` —— 用户地址空间是 `[0, 0x40_0000_0000)`，代码段和堆向上、栈向下、mmap 从 `0x3_0000_0000` 向下扩，与现有布局错开。
3. **两个分支**：
   - **匿名映射**（`addr.is_null() && MAP_ANONYMOUS`）：`allocate_vaddr(num_pages)` → 循环 `USER_ASPACE.lock().map_alloc(vaddr, PAGE_SIZE_4K, uflags, populate=true)` 立即建页。
   - **文件映射**（非匿名）：先 `map_alloc` 建页，再 `read_file_at(fd, offset, read_len)` 通过 `axfs::fops::File::read_at` 分块读 4 KiB chunk，最后 `uspace.write(vaddr_base, &buf)` 拷到用户页。`MAP_PRIVATE` 不要求写回文件，所以"一次性拷副本"就是正确语义。
4. **syscall 号架构差异**：`riscv64/aarch64/loongarch64 = 222`，`x86_64 = 9`，通过 `mod nums` 的 cfg 切换；在 `handle_syscall` 里加 `SYS_MMAP =>` 分支并按 musl ABI `(addr, length, prot, flags, fd, offset)` 解参数。

### 关键决策

- **`populate=true`**：避开 lazy fault 路径，对 32 字节这种小映射性能可忽略。
- **错误返回 `-errno`**：`neg_errno(LinuxError::ENOMEM)` 与其他 syscall 保持一致，用户态 musl 才能正确看到 `MAP_FAILED == (void*)-1`。
- **加锁顺序**：匿名分支是 `USER_ASPACE 锁 → map_alloc`；文件分支是 `先 read_file_at（拿 FD 锁）释放后再 USER_ASPACE 锁` —— 注意文件分支当前实现仍是"持有 USER_ASPACE 锁时请求 FD 锁"，单核串行下不会死锁，多核需重新审视。

### 注意

- `SYS_MMAP` 号在 x86_64 是 9，其他架构是 222，少一个 cfg 就只在 riscv 上 PASS。
- 当前匿名分支要求 `addr.is_null()`，如果未来 musl 给非空 hint 的匿名映射（如 stack/TLS）需要补一条分支。
- `MAP_FIXED` 仅当 `!addr.is_null() && MAP_FIXED` 时才把 hint 当真实地址用，其他情况一律 `allocate_vaddr`，与 `mapfile.c` 的使用方式吻合。
- 需要 `<arch>-linux-musl-gcc` 在 PATH 中，否则 xtask 编 payload 阶段就失败，看似代码 bug 实则是工具链没装。

---

## 七、观察


### 1. 修改最小化原则

每个练习都先问"能不能只改应用层"：printcolor 只改 `main.rs`，hashmap 只在 axstd 里加 collections 模块，sysmap 只在 syscall.rs 加 mmap 实现。只有在 crates.io 版本无法满足时才 vendor 整个 crate（hashmap / ramfs-rename），并通过 `[patch.crates-io]` 把所有间接依赖切到本地。

### 2. 调用链思维

最直观的是 ramfs-rename：从 `std::fs::rename` 到 `axfs_ramfs::DirNode::rename` 中间经过 4 层（axstd / axfs::api / axfs::root / axfs_vfs trait），每一层都要拆好职责。这种"由上到下逐层下钻"的方法在后续修 syscall 时反复用到。

### 3. trait 默认实现是陷阱

`VfsNodeOps::rename` 有默认 `Unsupported` 实现，必须在 `impl VfsNodeOps for DirNode` 块里显式 override，否则外部 trait 调用会绕过 inherent impl。这种陷阱在很多 Rust 内核组件里都存在。

### 4. ABI 差异在 cfg

`SYS_MMAP` 号在 x86_64 和其他架构不同，`PAGE_SIZE` 同理。任何看似"通用"的内核代码，都要先确认架构相关常量是否走 cfg 分支。

### 5. 测试方法的一致性

5 个练习的 `scripts/test.sh` 走的是同一套模板：跨 4 架构跑 `cargo xtask run`，对输出做 grep 校验关键字串。这种"输出字串就是验收契约"的设计让 CI 极其简单，但也意味着 grep 表达式本身就是规范的一部分（如 printcolor 排除 `\x1b[0m`）。

### 6. vendor 与 [patch.crates-io] 是利器

hashmap / altalloc / ramfs-rename 三个练习都用了"`cargo clone` + `[patch.crates-io]`"的组合：拉源码到本地子目录、整库 vendor、修改后通过 patch 把所有依赖图里的间接依赖都切过去。这套机制让"修一个 crate 一行代码"和"重写半个 crate"都用同一种工程姿态完成，对后续在 starry 上做大改也是一样的工作流。

---

## 八、验证清单

每个练习独立的验证命令：

```bash
# printcolor
cd exercise-printcolor && bash scripts/test.sh

# hashmap
cd exercise-hashmap && bash scripts/test.sh

# altalloc
cd exercise-altalloc && bash scripts/test.sh

# ramfs-rename
cd exercise-ramfs-rename && bash scripts/test.sh

# sysmap（额外要求：PATH 里有 <arch>-linux-musl-gcc）
cd exercise-sysmap && bash scripts/test.sh
```

预期所有 4 架构（`riscv64 / x86_64 / aarch64 / loongarch64`）全部 PASS。
