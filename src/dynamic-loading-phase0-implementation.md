# BlueOS 动态加载 Phase 0 详细实现方案

本文是 [BlueOS 动态加载详细实现计划](./dynamic-loading-implementation-plan.md) 中 Phase 0 的独立实施文档。目标不是预先创建一组“以后可能有用”的类型，而是按照正常工程实现顺序推进：每个 commit 先交付一种可以测试的能力，再引入完成该能力所必需的数据结构和方法。

本文的提交编号使用独立的 `P0-Cxx` 命名空间；它只表示 Phase 0 分支内的实现顺序，不占用总实施计划中 Phase 0.5 以后使用的 `Cxx` 编号。`P0-C00`–`P0-C12` 对应既有 commit，`P0-C13`–`P0-C15` 是本次评审新增的收口提交。

Phase 0 完成后，当前的 `load_elf(buffer, mapper)` 将成为可信单映像 `ImageLoader` 的兼容包装；现有自包含 `ET_DYN`、fixed-address `ET_EXEC` 和 RISC-V64 relative relocation 继续工作，同时补齐 ARM32 relative relocation、BSS、load bias、错误处理、事务所有权、权限与 cache 基线，为 Phase 0.5 的 `DynamicLinker` 提供稳定输入。

## 0. 实施状态（2026-08-27）

Phase 0 的 P0-C00–P0-C12 功能基线已在 `kernel` 仓库的 `feat/loader-phase0` 分支实现。本文本次评审以 `abezbm/feat/loader-phase0@6abe0aa` 为代码基线；当前工作区的 `kernel` HEAD 仍位于 `origin/main`，尚不包含这些文件，因此不能用当前 checkout 中的旧 `loader/src/lib.rs` 判断下表状态。

实现中已经发现旧方案把“static PIE”错误地当作启动方输入，因此 P0-C04 同时修正了 P0-C01–P0-C03 的类型契约：公开请求现在只有 `ExpectedElfType::{Dyn, Exec}`，自包含性在扫描 `PT_DYNAMIC` 后由 feature policy 判定。不过，2026-08-27 对代码、总实施计划和 Relink `1928b38` 的交叉评审又发现了事务提交、fixed 映像失败语义、ABI 准入和真实 protection/cache 能力等设计缺口。因此 P0-C00–P0-C12 只能称为**可运行基线**，不能再称为 Phase 0 已最终完成。

| 阶段 | Commit | 已交付结果 |
| --- | --- | --- |
| P0-C00 | `8b70ca8` | host regression fixture 和 GN test target |
| P0-C01 | `13173cd` | read-at 输入、ELF header/ABI/type 准入和结构化错误 |
| P0-C02 | `13b04db` | 自有 program-header 运行时视图 |
| P0-C03 | `8acde3c` | checked layout、W^X、entry、load bias 和资源上限 |
| P0-C04 | `57ced2a` | `ExpectedElfType` 修正、fallible allocation 和事务回滚 |
| P0-C05 | `d1b9f2b` | 分块复制、BSS/gap 清零和 fixed placement 写前预检 |
| P0-C06 | `8aac272` | 有界 `PT_DYNAMIC` 扫描、Phase 0 策略和 REL/RELA 归一化 |
| P0-C07 | `f5616f6` | RISC-V64 RELA relative relocation engine |
| P0-C08 | `1734d4b` | ARM32 REL 隐式 addend 和 ELF32 地址宽度检查 |
| P0-C09 | `429409b` | cache 同步、RELRO/最终权限、seal 和事务提交 |
| P0-C10 | `46478c9` | `load_image()` 编排、兼容入口切换和旧管线删除 |
| P0-C11 | `9e65653` | 补全 alignment overflow、gap `NONE` seal、动态策略矩阵和逐调用故障回滚 |
| P0-C12 | `6abe0aa` | 让 LLD ARM32 工件稳定产生并在 QEMU 中消费 `R_ARM_RELATIVE` |

P0-C00–P0-C12 的验证覆盖 52 个 host tests，以及 RISC-V64 `ET_DYN`、ARM32 `ET_DYN`、RISC-V32 fixed `ET_EXEC` 三条真实 QEMU 执行路径。P0-C11 修复了 alignment overflow 分类、allocation gap 未进入 `SealPlan` 和故障注入覆盖不足；P0-C12 修复了“ARM QEMU 通过但工件没有真实 relocation”的 oracle 缺口。下面保留按 commit 解释“为什么此时引入这些结构”的实施逻辑，同时以 P0-C13–P0-C15 修正最终接口和验收门禁。

### 0.1 评审后新增的阻塞项

以下问题必须在进入 Phase 0.5 前关闭：

1. `load_image()` 当前先执行 `commit_for()`，兼容包装随后才调用可失败的 `install_sealed()`。这使 entry 安装失败发生在 rollback authority 已解除之后，违反“commit 后不再失败”。
2. `ImageLoadTransaction` 与 staged image 是两个独立参数，`commit_for()` 只比较 `AllocationId`；来自另一个 backend/transaction 的同号 allocation 可能被错误提交。成功后的旧 sealed payload 也只保存 allocation 描述，不持有资源 lease，entry 的存活所有者不明确。
3. owned `ET_DYN` 失败时可以通过释放 allocation 实现强回滚；borrowed fixed `ET_EXEC` 一旦开始写入或修改权限就不能靠 `release()` 恢复旧内容。两者不能继续使用同一句“事务回滚”描述。
4. `RuntimeFeaturePolicy` 目前没有实际决策，`PT_INTERP/PT_TLS` 和 dynamic tag 的拒绝散落在 parser/decoder；`DT_FLAGS/DT_FLAGS_1` 也没有按 bit 白名单校验，Phase 0.5 无法只靠“放宽 policy”复用 parser。
5. byte-range `SealPlan` 没有 protection granule、MPU region 数量或实际应用范围；逐 range `protect()` 可以部分成功。ARM backend 对所有 ARM 仅执行 DSB/ISB 就报告成功，无法区分无 cache 的 MPS2 与 cache-enabled Cortex-M；RISC-V `fence.i` 也只同步当前 hart。
6. `ArtifactProfile` 没有校验 ARM/RISC-V `e_flags`、Cortex-M Thumb entry bit 和最小指令对齐；`ElfReader` 没有不可变快照契约；默认 relocation/映像配额的峰值 RAM 也不适合 MCU。

### 0.2 从 Relink 采用和不采用的事务原则

Relink 的 `PreparedLoad → RelocatedLoad → PublishedLoad`、私有 `PublishSession`、`ModuleId { context, slot, generation }`、非 `Copy` 的 `ModuleLease` 和映射 RAII 证明了以下原则适合 BlueOS：

- 映射、重定位和 seal 期间资源保持私有，stage 对象必须 `#[must_use]`；
- commit 前验证全部身份、容量、依赖和发布条件；commit 段只做 move/swap 或不可失败的安装；
- 成功后由明确的 owner/lease 保证映像地址在 entry、回调和执行流存活期间有效；
- 初始化等外部副作用位于提交之后，不能谎称可以自动回滚。

BlueOS 不照搬 Relink 在 prepare 时预占 committed slot、commit 写入后仍可能分配、`PublishedLoad` 被直接 Drop 不自动回滚，以及 rollback 错误覆盖原始错误的行为。Phase 0 只有一个 allocation，不提前引入多模块 registry、generation table 或 loader lock；先用单映像 guard 把上述不变量做实，Phase 0.5 再扩展为多资源 rollback log。

## 1. 实施前实现的问题

实施前 [`kernel/loader/src/lib.rs`](../../kernel/loader/src/lib.rs) 的主要问题不是缺少动态链接功能，而是单映像装载本身尚未形成可信基线：

- `p_vaddr + p_memsz`、`p_offset + p_filesz` 等 ELF 输入直接参与未检查算术；
- 只复制 `p_filesz`，没有显式清零 `p_memsz - p_filesz`，但底层 `Storage` 并不保证零初始化；
- relocation 使用 `real_start + addend`，最低 `p_vaddr != 0` 时 allocation base 不等于 load bias；
- 只遍历 RELA 和 `R_RISCV_RELATIVE`，其他 relocation 被静默忽略；
- `Elf::parse(buffer)` 强制整个文件连续驻留，无法自然接入 VFS read-at 路径；
- 映像复制完成后立即允许取得 entry，没有 cache 同步和 seal 边界。

实施前 [`memory_mapper.rs`](../../kernel/loader/src/memory_mapper.rs) 还存在以下问题：

- 分配对齐由 loader 自身架构决定，没有采用 ELF 所有 `PT_LOAD` 的最大 `p_align`；
- `start + size` 等目标范围检查仍可能溢出；
- `write_value_at<T>` 混合目标字宽、宿主类型、端序和对齐；
- 分配失败不能可靠地形成结构化 OOM 错误；
- `MemoryMapper` 同时承担布局、分配、访问和成功结果保存，难以表达失败回滚。

因此 Phase 0 必须先修正单映像 loader，不能直接在旧 `load_elf()` 上叠加 `DT_NEEDED`、符号解析和 DSO 生命周期。

## 2. Phase 0 的范围

### 2.1 必须交付

- `ElfReader::read_exact_at()` 随机读取接口和 slice adapter；
- `ElfReader` 的不可变快照契约；
- ELF header、program header、目标 ABI（含 `e_flags`/entry mode）和板级资源上限校验；
- checked arithmetic、正确 load bias、segment 布局和最大对齐；
- 映像分配、分块复制、BSS/gap 清零和失败回滚；
- 现有 RISC-V64 `RELA + R_RISCV_RELATIVE`；
- ARM32 `REL + R_ARM_RELATIVE`；
- relative relocation 的范围、owner、权限、字宽、端序、对齐和溢出校验；
- 未知 relocation fail closed；
- dynamic tag/flag 的显式白名单；
- logical permissions、granule-aware protection plan、逐 range `ProtectionRecord`、指令缓存同步和 seal；
- pre-commit guard、成功后的 allocation lease/owner，以及 borrowed fixed 的 poisoned 失败状态；
- `load_elf()` 兼容入口转调新管线；
- host malformed/fault-injection tests 和现有 QEMU 回归。

### 2.2 明确不做

Phase 0 不实现：

- AArch64 单映像 compatibility relocation/execution；`load_elf()` 在该 target 明确返回 unsupported，正式支持放到 Phase 2 的 `R_AARCH64_*`、页权限和 QEMU oracle 一并交付；
- `DT_NEEDED` 和 DSO 依赖闭包；
- dynsym、GNU/SysV hash 和符号查找；
- `R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT`；
- constructor/destructor 和 init/fini plan；
- `ApplicationManager`、`ThreadGroup`、VFS adapter 和 system DSO registry；
- 路径身份、签名、build-id 和产品 allowlist；
- `libc.so.1`、`dlopen`、原生 ELF TLS、lazy binding、IFUNC 和 COPY relocation。

这些能力分别属于 Phase 0.5、Phase 1 或 Phase 3，不应以空结构、占位 trait 方法或未使用字段的形式提前进入 Phase 0。

### 2.3 ELF 类型与运行时能力的职责边界

启动方不声明“这个应用是 static PIE”。对内核而言，应用在真正启动前只是一个 ELF 工件；`ET_DYN` 也不能区分自包含 PIE、动态链接可执行文件和共享对象。因此公开请求只表达可以由 ELF header 直接、常数时间验证的类型约束：

```rust
pub enum ExpectedElfType {
    Dyn,
    Exec,
}
```

- `Dyn` 要求 `e_type == ET_DYN`，布局采用可移动放置并计算 load bias；
- `Exec` 要求 `e_type == ET_EXEC`，布局采用 fixed placement；
- 不在 header admission 阶段推断“static PIE”或“动态应用”；
- Phase 0 在 P0-C03/P0-C06 分别对 program feature 和 `PT_DYNAMIC` 做有界扫描，并通过 `Phase0ArtifactPolicy` 拒绝 `PT_INTERP/PT_TLS/DT_NEEDED`、符号型 relocation 等本阶段不支持的语义；通过该策略意味着“本次可以由 Phase 0 单映像管线完成”，而不是给工件永久贴上 static PIE 标签；
- Phase 0.5 放宽运行时策略并建立依赖图，但沿用同一个 `ExpectedElfType::Dyn` 请求 ABI。

扫描 `PT_DYNAMIC` 不是为了额外识别应用类别；relocation 本来就必须读取这些元数据。扫描次数受 `max_dynamic_entries` 限制，复杂度与后续 relocation 数量校验相比不是独立的启动成本。

### 2.4 完成标准

完成标准不是“新 API 已经存在”，而是：

1. 旧 parse/copy/relocate 实现已经删除；
2. 所有输入走同一组阶段转换；
3. 所有可以在不写目标内存时完成的 ELF、policy、布局、发布和 backend capability 校验都在副作用前完成；
4. owned 映像在 S2–S8 任一步失败都释放唯一 lease；borrowed fixed 在首次修改后的失败进入不可复用的 `Poisoned` 状态，不能被描述为内容回滚；
5. commit 前 allocation、seal 结果和 entry 均不可见，只有成功接管 lease 的 committed owner 能公开 entry；
6. commit 前完成全部可失败校验，commit/install 本身不可失败，commit 后没有返回 `Result` 的步骤；
7. ARM32 host fixture、现有 RISC-V64 `ET_DYN`、fixed `ET_EXEC` 和 QEMU 回归全部通过；未支持的既有架构必须显式列为范围缩减，不能静默回退。

## 3. 总体实现主线

```text
P0-C00 锁定测试基线
  ↓
P0-C01 从任意来源安全读取并准入 ELF
  ↓
P0-C02 解析自有的 program-header 运行时视图
  ↓
P0-C03 在写内存前生成完整布局计划
  ↓
P0-C04 预留目标内存并建立事务回滚
  ↓
P0-C05 复制 PT_LOAD，确定性清零 BSS/gap
  ↓
P0-C06 从 PT_DYNAMIC 归一化 relative relocation 元数据
  ↓
P0-C07 把现有 RISC-V64 relative relocation 迁入新 engine
  ↓
P0-C08 增加 ARM32 REL 隐式 addend
  ↓
P0-C09 同步 cache、应用最终权限并 seal
  ↓
P0-C10 切换 load_elf()，删除旧管线
  ↓
P0-C11–P0-C12 修复验收 oracle 和已发现的 hardening 缺口
  ↓
P0-C13 收紧事务、所有权和 fixed abort 语义
  ↓
P0-C14 收紧 source snapshot、ABI、policy、配额与 relocation 计划
  ↓
P0-C15 落地 granule-aware protection/cache capability 和最终故障矩阵
```

P0-C00–P0-C09 期间旧 `load_elf()` 暂时保持可用；P0-C10 才切换生产入口。这会使新旧实现短期共存，但旧路径在此期间只做兼容基线，不继续扩展。P0-C10 合入后不得保留第二个 parser、映射算法或 relocation engine。

最终运行时转换为：

```text
ElfReader
  → admit            → AdmittedArtifact<R>
  → inspect + plan   → PlannedArtifact<R>
  → reserve          → ReservedImage<R>
  → copy + zero      → MappedImage
  → decode runtime   → RuntimeImage
  → relocate         → RelocatedImage
  → seal             → PreparedImage<SealedState>（rollback 仍 armed）
  → local commit     → M::CommitReceipt / installed MemoryMapper owner
```

类型设计遵循以下规则：

1. 阶段转换消费上一阶段对象并返回下一阶段对象；
2. 阶段类型的字段和构造函数为 private 或 `pub(crate)`；
3. 不提供公共 `set_state()`；
4. `MappedImage/RuntimeImage/RelocatedImage` 不暴露 entry，pre-commit 的 `SealedState::entry()` 只允许受信 publisher 读取；
5. `PreparedImage` 必须 `#[must_use]`，Drop 时仍拥有唯一 rollback authority；
6. 值类型在第一次出现真实消费者时引入，不按终态类型表一次性创建。

## 4. P0-C00：建立可运行的测试基线

建议提交：

```text
loader: add host regression test scaffolding
```

### 4.1 本提交要解决的问题

后续会依次替换 reader、parser、layout、memory 和 relocation。实现前需要能区分：

- 有意拒绝不安全输入；
- 无意破坏已有自包含 `ET_DYN` 或 `ET_EXEC`；
- 新实现计算结果错误；
- 失败路径泄漏资源。

### 4.2 实现内容

- 在 `kernel/loader/BUILD.gn` 增加 host `loader_unittest` 和 `run_host()` target；
- 增加 test-only `ElfFixtureBuilder`；
- builder 能构造 ELF32/ELF64 header、program header、dynamic table 和 REL/RELA 表；
- fixture 默认不带 section header，避免测试继续依赖完整链接产物；
- 固化现有 RISC-V64 `ET_DYN` 和 fixed `ET_EXEC` 的兼容行为；
- 保留现有 QEMU execution tests；
- 将纯 malformed ELF 测试放到 host debug，不再全部隐藏在 `#[cfg(not(debug_assertions))]` 后。

此时不要增加 `FakeMemory` 或 `FakeCodeCache`：相应生产 trait 尚未存在，测试替身应在 trait 出现的 commit 一起引入。

### 4.3 本提交需要的数据结构

只增加 test-only 结构：

```rust
struct ElfFixtureBuilder {
    // ELF class/endian/machine/header/phdr/dynamic/relocation test data
}
```

不改变生产 API，不增加阶段对象。

### 4.4 Gate

- host test target 可独立运行；
- 当前合法 fixture 在旧实现上通过；
- QEMU loader integration test 不回退；
- 本提交不提交预期失败测试，也不修改生产行为。

## 5. P0-C01：用 read-at 接口完成 ELF 身份准入

建议提交：

```text
loader: admit ELF artifacts through read-at input
```

### 5.1 本提交要解决的问题

当前 loader 必须拿到完整 `&[u8]`。本提交只回答：

> 这个输入是不是当前 artifact profile 允许装载的 ELF？

此时不需要知道 segment 布局，也绝不能接触目标内存。

### 5.2 实现内容

新增：

```text
src/error.rs
src/reader.rs
src/identity.rs
src/limits.rs
```

Reader 接口：

```rust
pub trait ElfReader {
    fn len(&self) -> LoadResult<u64>;

    fn read_exact_at(
        &self,
        offset: u64,
        dst: &mut [u8],
    ) -> LoadResult<()>;
}

pub struct SliceElfReader<'a> {
    bytes: &'a [u8],
}
```

`ElfReader` 的值必须代表一次稳定快照：从 `len()`、header/phdr 读取直到最后一个 segment read，所有调用观察同一组字节。`SliceElfReader` 天然满足该约束；未来 VFS adapter 必须 pin 不可变 vnode/version，或在版本变化时返回 `SourceChanged`，不能把共享 file offset 或可并发覆写的文件直接包装成 reader。签名阶段也必须验证并装载同一个 snapshot，避免 check/use 分离。

解析 header 时先读取 16 字节 `e_ident`，确定 ELF32/ELF64 后再精确读取 52/64 字节 header。不能为了方便固定读取 64 字节并误拒绝较短但合法的 ELF32 输入。

实现：

```rust
impl ImageLoader {
    pub fn admit<R: ElfReader>(
        &self,
        reader: R,
        request: ArtifactRequest,
    ) -> LoadResult<AdmittedArtifact<R>>;
}
```

校验顺序：

1. `reader.len()` 和 `max_file_len`；
2. magic、class、endianness、ELF version；
3. `e_type` 是否匹配调用方要求的 `Dyn/Exec`；
4. machine 和 artifact profile；
5. `e_ehsize/e_phentsize`；
6. `e_phnum` 上限；
7. checked 计算 program-header table 文件范围。

### 5.3 因此才引入的数据结构

```rust
pub type LoadResult<T> = core::result::Result<T, LoadError>;

pub struct LoadError {
    stage: LoadStage,
    kind: LoadErrorKind,
    context: ErrorContext,
}

pub enum ElfClass {
    Elf32,
    Elf64,
}

pub enum Endian {
    Little,
    Big,
}

pub enum ExpectedElfType {
    Dyn,
    Exec,
}

pub struct ArtifactProfile {
    class: ElfClass,
    endian: Endian,
    machine: u16,
    header_flags: HeaderFlagsPolicy,
    entry_mode: EntryMode,
    minimum_image_alignment: u64,
    relative_values: RelativeValuePolicy,
}

pub struct HeaderFlagsPolicy {
    // 由 machine profile 实现 mask/value/forbidden 校验，不能跨架构解释 e_flags。
}

pub enum EntryMode {
    Direct {
        instruction_alignment: u8,
        minimum_instruction_size: u8,
    },
    Thumb {
        instruction_alignment: u8,
        minimum_instruction_size: u8,
    },
}

pub struct LoadLimits {
    max_file_len: u64,
    max_program_headers: u16,
}

pub struct ArtifactRequest {
    expected_elf_type: ExpectedElfType,
    profile: ArtifactProfile,
    limits: LoadLimits,
}

pub struct AdmittedArtifact<R> {
    reader: R,
    header: ElfHeaderInfo,
    request: ArtifactRequest,
    file_len: u64,
}
```

`ArtifactProfile::new()` 不提供架构相关默认值，必须显式接收 `HeaderFlagsPolicy`、`EntryMode`、最小 image alignment 和 `RelativeValuePolicy`；调用方不能只填 class/endian/machine 就得到一个宽松 profile。profile 构造后没有可逐项削弱命名 profile 的 `with_*` API；生产入口优先使用经过评审的命名 machine profile，不能用“通用 profile”隐式放行未知 ABI 位、退化 entry/alignment 约束或扩大 relative relocation 结果范围。

这些类型的当前目的：

- `ArtifactProfile`：描述 loader 能处理的目标 ELF ABI；
- `ArtifactRequest`：描述一次单映像装载的 ELF 类型、ABI 和资源要求，不声明应用链接模型；
- `AdmittedArtifact`：证明 header、ABI 和 phdr table 外层范围已经验证；
- `LoadLimits`：让文件长度和 phdr 数量在 allocation 前受控；
- `LoadError`：让测试和上层能够区分 read/parse/validate 错误。

`LoadStage` 由执行阶段边界赋值。`TargetAddr/TargetRange/TargetLocation` 等底层算术不能把错误永久写成 `Validate` 或 `Map`，backend 也不能把所有访问失败都写成 `Map`；它们应返回 stage-neutral 的 `RangeError/MemoryError`，由 metadata/relocate/cache/seal/publish 调用点包装。`ElfReader` 自身的原始错误属于 `Read`，但 S1 program-header 消费点和 S3 segment-copy 消费点必须分别重绑定为 `Parse`、`Map`。尤其 `prepare_install` 属于 `Publish`，不能继续借用 `Seal` 标签。否则同一溢出或 I/O 失败会因为复用 helper 而被错误归因，诊断和 fault-injection oracle 都不可信。

`ArtifactProfile` 不能只比较 class/endian/machine：

- ARM Thumb v7-M soft-float profile 至少校验 ARM EABI/float 相关 `e_flags`，要求 entry bit 0 为 1，并用清除 bit 0 后的 canonical 地址及最小指令跨度做 X-range membership；
- RISC-V profile 校验 float ABI、RVE/扩展能力与 entry 指令对齐；
- 机器专用校验通过 profile/arch validator 完成，不把 ARM 常量散落到通用 parser；
- BlueOS ABI note、签名和 build-id 仍属于 Phase 1 产品准入，不因本修订提前进入 Phase 0。

`LoadLimits::default()` 只能用于 host fixture。兼容入口必须由 board/profile 提供限额；64 MiB image 和一百万条 relocation 不能作为 MCU 的生产默认值。后续在真实消费者出现时逐步增加 layout/metadata/operation 的峰值字节配额，而不是在 P0-C01 预建未使用字段。Phase 0 的可变长度 state payload 最终统一保留已经 `try_reserve*` 成功的 `Vec`；不能在其后调用可能隐式收缩分配的 `into_boxed_slice()`，尤其不能在映像字节已修改后引入无法返回 `LoadError` 的 OOM 点。

不要在本提交引入路径、build-id、签名、`ExecutionModel`、`LinkDomainId` 或 `ImageId`。它们在单映像 Phase 0 中没有消费者。

### 5.4 测试与 Gate

- 空文件、截断 header、短读；
- 错误 class/endian/machine/type，以及 System V 下非零 `EI_ABIVERSION`；
- 错误 `e_flags`、ARM even entry、错误指令对齐；
- `e_phoff + e_phnum * e_phentsize` 溢出或越界；
- 文件长度和 phdr 数量超限；
- `SliceElfReader` 的 offset 转换和 `offset + len` 溢出；
- snapshot reader 在一次事务中保持版本不变，版本变化由 adapter 确定失败；
- profile/board 限额覆盖峰值 metadata/scratch，而不是只覆盖最终 image span；
- `ImageLoader::admit()` 不接受 `ImageMemory` 参数，从接口上保证本阶段不会分配；
- 旧 `load_elf()` 行为不变。

## 6. P0-C02：解析自有的 program-header 视图

建议提交：

```text
loader: parse an owned program-header view
```

### 6.1 本提交要解决的问题

只有取得 `PT_LOAD/PT_DYNAMIC` 等 program header 后才能规划内存。完整 `goblin::Elf<'a>` 会借用整个文件并解析 Phase 0 不需要的 section header，因此不能作为长期状态。

### 6.2 实现内容

新增：

```text
src/address.rs
src/image/mod.rs
src/image/parser.rs
```

通过 `read_exact_at()` 读取已经过上限校验的 phdr table。可以使用 goblin 的低层 header/phdr 解码，但解码结果要立即复制为 BlueOS 自有结构。

Phase 0 只保留：

- `PT_LOAD`；
- `PT_DYNAMIC`；
- `PT_GNU_RELRO`；
- `PT_GNU_STACK`；
- 用于 policy 判定的 `PT_INTERP/PT_TLS`。

不读取 section header，也不要求文件存在 section header。

Parser 只做结构解码和范围校验，不在 `match p_type` 中写死 Phase 0 产品策略。`ParsedImage` 需要记录 `ProgramFeatureSummary`；P0-C03 在 allocation 前调用 policy 拒绝 `PT_INTERP/PT_TLS`、可执行栈等仅依赖 program header 的特性，P0-C06 再判定只有读取 dynamic table 后才能知道的特性。这样既保持 fail-early，又不要求 Phase 0.5 重写 parser。

### 6.3 因此才引入的数据结构

```rust
#[repr(transparent)]
pub struct TargetAddr(u64);

pub struct TargetRange {
    start: TargetAddr,
    len: u64,
}

pub struct FileRange {
    offset: u64,
    len: u64,
}

pub struct LoadSegmentInfo {
    file_range: FileRange,
    vaddr: TargetAddr,
    memory_size: u64,
    align: u64,
    permissions: MemoryPermissions,
}

pub struct DynamicSegmentInfo {
    file_range: FileRange,
    vaddr: TargetAddr,
    memory_size: u64,
}

pub struct ParsedImage {
    header: ElfHeaderInfo,
    load_segments: Vec<LoadSegmentInfo>,
    dynamic: Option<DynamicSegmentInfo>,
    relro: Option<TargetRange>,
    stack_policy: StackPolicy,
    program_features: ProgramFeatureSummary,
}

pub struct ProgramFeatureSummary {
    interpreter: Option<(u16, FileRange)>,
    tls: Option<(u16, TargetRange)>,
    executable_stack: Option<u16>,
}
```

`TargetAddr/TargetRange` 到本提交才出现，因为从这里开始需要明确区分 ELF 虚址、目标范围与文件 offset；`PT_GNU_RELRO` 也已经需要表达目标范围。

核心方法：

```rust
impl ImageLoader {
    pub(crate) fn inspect<R: ElfReader>(
        &self,
        artifact: &AdmittedArtifact<R>,
    ) -> LoadResult<ParsedImage>;
}

impl FileRange {
    fn end(&self) -> LoadResult<u64>;
    fn validate(&self, file_len: u64) -> LoadResult<()>;
}

impl TargetRange {
    fn end(&self) -> LoadResult<TargetAddr>;
    fn contains_span(&self, start: TargetAddr, len: u64) -> bool;
    fn overlaps(&self, other: &Self) -> bool;
}
```

`inspect()` 可以作为只读的结构解析边界公开，但不能分配目标内存或产生可变 target access；P0-C03 的 `plan()` 才形成可进入 reserve 的完整阶段证明。

### 6.4 测试与 Gate

- 多个 `PT_LOAD`；
- 无 section header；
- phdr table 截断；
- segment file range 越界；
- 重复 `PT_DYNAMIC/PT_GNU_STACK`；
- `PT_INTERP/PT_TLS` 和 stack policy 被准确记录，结构损坏仍由 parser 拒绝；
- Phase 0 policy 在 P0-C03、allocation 调用次数仍为零时明确拒绝这些 feature；
- `ParsedImage` 不含输入 slice 引用；
- 临时 phdr buffer 在 `inspect()` 返回后即可释放。

## 7. P0-C03：在分配前生成完整布局计划

建议提交：

```text
loader: validate and plan image layout
```

### 7.1 本提交要解决的问题

分配大小、对齐、load bias、entry 和 segment 权限由全部 `PT_LOAD` 共同决定。边解析边分配或边复制会使恶意的后续 phdr 在系统已经产生副作用后才触发失败。

### 7.2 实现内容

新增：

```text
src/image/layout.rs
```

固定检查顺序：

1. 调用 `ArtifactFeaturePolicy::validate_program_features()`，在 allocation 前拒绝本 profile 不支持的 program feature；
2. 至少存在一个非空 `PT_LOAD`；
3. `p_filesz <= p_memsz`；
4. `p_align` 为 0、1 或 2 的幂，0 归一化为 1；
5. `p_offset % p_align == p_vaddr % p_align`；
6. file range、vaddr range 和 image span 全部 checked；
7. segment 数量、span、alignment 和峰值 scratch 不超过 `LoadLimits`；
8. Phase 0 只接受 `R`、`RX`、`RW` 三种非空 segment 权限，拒绝 W+X、X-only、W-only 和未知权限组合；
9. 按 vaddr 排序；Phase 0 拒绝任何非空 overlap；
10. 计算所有 segment 的 `max_align` 并满足 profile 的最小映像对齐；allocation backend 可以使用更高物理对齐，但必须返回满足请求的逻辑 base，并隐藏额外 padding；
11. 计算 `aligned_min_vaddr/aligned_max_vaddr/image_span`；
12. entry 必须满足架构编码/指令对齐，且从 canonical entry 开始的最小指令跨度完整落在真实 X segment，不能只让首字节位于 allocation 包络或 segment 尾部。

ARM32 entry 需要区分：

- 范围验证用 canonical address，即清除 Thumb bit；
- 最终执行地址保留 bit 0。

### 7.3 因此才引入的数据结构

```rust
pub struct SegmentLayout {
    source_index: u16,
    vaddr_range: TargetRange,
    file_range: FileRange,
    align: u64,
    permissions: MemoryPermissions,
}

pub struct ImageLayout {
    aligned_min_vaddr: TargetAddr,
    aligned_max_vaddr: TargetAddr,
    image_span: u64,
    max_align: u64,
    entry_vaddr: TargetAddr,
    canonical_entry_vaddr: TargetAddr,
    segments: Vec<SegmentLayout>,
    relro: Option<TargetRange>,
}

pub struct PlannedArtifact<R> {
    artifact: AdmittedArtifact<R>,
    parsed: ParsedImage,
    layout: ImageLayout,
}

pub(crate) struct SegmentLocation {
    segment_index: usize,
    offset_in_segment: u64,
}
```

`LoadLimits` 在出现消费者后扩展：

```rust
max_load_segments: u16,
max_image_span: u64,
max_segment_alignment: u64,
max_layout_bytes: u64,
```

核心方法：

```rust
impl TargetAddr {
    fn checked_add(self, value: u64) -> LoadResult<Self>;
    fn checked_add_signed(self, value: i64) -> LoadResult<Self>;
    fn checked_sub(self, other: Self) -> LoadResult<u64>;
    fn align_down(self, align: u64) -> LoadResult<Self>;
    fn align_up(self, align: u64) -> LoadResult<Self>;
}

impl ImageLoader {
    pub fn plan<R: ElfReader>(
        &self,
        admitted: AdmittedArtifact<R>,
        policy: &dyn ArtifactFeaturePolicy,
    ) -> LoadResult<PlannedArtifact<R>>;
}

impl ImageLayout {
    fn load_bias_for(
        &self,
        mapped_base: TargetAddr,
        expected_elf_type: ExpectedElfType,
    ) -> LoadResult<TargetAddr>;

    fn locate_vaddr_range(
        &self,
        vaddr: TargetAddr,
        len: u64,
        permissions: MemoryPermissions,
    ) -> LoadResult<SegmentLocation>;
}
```

公式：

```text
Dyn:
    B = mapped_base - aligned_min_vaddr
    runtime(vaddr) = B + vaddr

Exec:
    mapped_base = aligned_min_vaddr
    B = 0
```

### 7.4 测试与 Gate

- 最低 `p_vaddr != 0`；
- segment 中间存在大 gap；
- 无 `PT_LOAD`；
- `filesz > memsz`；
- 非法 align 或 offset/vaddr 不同余；
- file/vaddr/span 溢出；
- W+X 和 overlap；
- entry 位于 gap、只读 segment、映像外，或最小指令跨度越过 X segment 尾部；
- ARM Thumb entry membership；
- allocator 调用次数仍为零。

## 8. P0-C04：预留目标内存并建立事务回滚

建议提交：

```text
loader: reserve image memory transactionally
```

### 8.1 本提交要解决的问题

只有完成布局计划后才具备安全的 allocation 输入。从第一次产生 allocation 开始，后续任何失败都必须释放已经取得的资源。

### 8.2 实现内容

新增：

```text
src/memory.rs
```

P0-C04 最初只引入 allocation descriptor 和 rollback bookkeeping；P0-C13 将其收紧为下面的最终 contract：

```rust
pub trait ImageMemory {
    fn allocate_image(
        &mut self,
        request: &AllocationRequest,
    ) -> LoadResult<AllocationLease>;

    // 必须无分配、无 panic、不可失败；owned 释放，modified fixed 至少标记 poisoned。
    fn abort_image(
        &mut self,
        allocation: AllocationLease,
        progress: MutationProgress,
    );

    // committed owner 结束寿命时调用：owned 可回收，borrowed fixed 只解除借用，
    // 不恢复内容，也不把一次成功装载标成 poisoned。
    fn release_committed(
        &mut self,
        allocation: AllocationLease,
    );

    // 可被 Map/Metadata/Relocate/Seal 复用，backend 不得预写死阶段。
    fn read(...) -> MemoryResult<()>;
    fn write(...) -> MemoryResult<()>;
    fn zero(...) -> MemoryResult<()>;
    fn validate_access(...) -> MemoryResult<()>;
    fn protect(...) -> MemoryResult<ProtectionLevel>;
}

pub struct MemoryError {
    kind: LoadErrorKind,
    context: ErrorContext,
}

impl MemoryError {
    fn at(self, stage: LoadStage) -> LoadError;
}
```

allocation/prepare/publish 等天然只属于单一阶段的接口继续返回 `LoadResult`；目标内存 access backend 返回无阶段的 `MemoryError`，由 Map、Metadata、Relocate 或 Seal 消费点显式调用 `at(stage)`。这样新增 dynamic/symbol decoder 时不会继承兼容 mapper 的默认错误阶段。

实现 `MemoryMapper` compatibility adapter：

- allocated mode 使用 fallible `Storage::try_from_layout()`；
- fixed mode 返回 borrowed/fixed allocation；
- 校验 backend 返回的 target base、精确逻辑长度和对齐；
- fixed placement 必须完全位于授权 range；
- load bias 只能从经过验证的 allocation 计算。

`AllocationRequest::size` 定义 loader 可寻址的**逻辑 allocation 长度**。backend 即使因页/MPU granule 在物理上多分配，也只能向 loader 返回精确等长的逻辑 view；额外 backing padding 必须由 backend 隐藏并维持 NONE/不可访问。fixed backend 不能用更长 descriptor 扩大调用方授权范围。

事务对象持有 `&mut ImageMemory`。后续阶段通过事务访问 backend，不能让 transaction 持有 backend 的同时又把独立 `&mut memory` 传给阶段函数。

### 8.3 因此才引入的数据结构

```rust
pub enum Placement {
    Anywhere,
    Fixed(TargetRange),
}

pub struct AllocationRequest {
    placement: Placement,
    size: u64,
    align: u64,
}

#[repr(transparent)]
pub struct AllocationId(u32);

pub enum AllocationOwnership {
    Owned,
    BorrowedFixed,
}

#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub struct ImageAllocation {
    id: AllocationId,
    target_base: TargetAddr,
    len: u64,
    align: u64,
    ownership: AllocationOwnership,
}

#[must_use = "allocation lease must be transferred or aborted exactly once"]
pub struct AllocationLease {
    allocation: ImageAllocation,
    // 非 Clone、非 Copy；只有 transaction/committed owner 可以持有。
}

pub enum MutationProgress {
    Reserved,
    BytesModified,
    ProtectionModified,
}

pub struct ReservedState<R> {
    artifact: AdmittedArtifact<R>,
    parsed: ParsedImage,
    layout: ImageLayout,
    allocation: ImageAllocation,
    load_bias: TargetAddr,
}

#[must_use = "dropping an active image transaction aborts its allocation"]
pub struct ImageLoadTransaction<'a, M: ImageMemory> {
    memory: &'a mut M,
    pending: Option<AllocationLease>,
    progress: MutationProgress,
}

#[must_use = "dropping any post-allocation stage aborts its lease"]
pub struct StagedImage<'a, M: ImageMemory, S> {
    transaction: ImageLoadTransaction<'a, M>,
    state: S,
}

pub type ReservedImage<'a, M, R> =
    StagedImage<'a, M, ReservedState<R>>;
```

`ImageAllocation` 只是可复制的范围/身份描述，不授予释放或提交权限；只有非 `Clone`、非 `Copy` 的 `AllocationLease` 是资源 authority。因此 stage 可以保存 descriptor 进行寻址，但不能仅凭相同 `AllocationId` 提交或释放资源。

`ImageLoadTransaction` 的当前唯一目的：allocation 产生后，S2–S8 任一步失败都恰好 abort 一次。Phase 0 一笔事务只有一个 allocation，因此使用 `Option<AllocationLease>`；不要为了 Phase 0.5 提前使用 `Vec<AllocationId>` 并增加一个无业务价值的分配失败点。Phase 0.5 出现多映像、permit 和 system lease 后，再把它扩展为 `RollbackLog<Vec<RollbackAction>>`。

从 reserve 成功开始，transaction 不是转换函数的第二个参数，而是每个 `StagedImage<S>` 的私有字段。`ReservedImage/MappedImage/RuntimeImage/RelocatedImage/PreparedImage` 是围绕不同 state payload 的 alias 或 opaque newtype；下文为可读性省略其 backend/lifetime 参数。转换实现先把上一 stage 拆成本地 `transaction + state`，调用内部操作，再把同一 transaction 装入下一 stage；操作返回 `Err` 时本地 transaction 自然 Drop 并 abort。`StagedImage` 本身不实现自定义 Drop，真正的 Drop authority 只在 `ImageLoadTransaction`，从而避免 consuming transition 依赖不必要的 `unsafe/ManuallyDrop`。

### 8.4 核心方法

```rust
impl ImageLoader {
    fn reserve<'a, R, M>(
        &self,
        planned: PlannedArtifact<R>,
        memory: &'a mut M,
    ) -> LoadResult<ReservedImage<'a, M, R>>
    where
        R: ElfReader,
        M: ImageMemory;
}

impl<M: ImageMemory> ImageLoadTransaction<'_, M> {
    fn allocation(&self) -> &ImageAllocation;
    fn mark_bytes_modified(&mut self);
    fn mark_protection_modified(&mut self);
}
```

transaction 的 accessor 以及 backend 读写 helper 均保持 private/`pub(crate)`；安全主路径只暴露消费 `self` 的 stage transition，不再存在 `reserve/copy/decode/relocate/seal(stage, &mut transaction)` 形式。调用方不能同时拿到任意 stage 和另一个 transaction，也不能调用类似 `commit_for(&sealed)` 的 API。测试成功路径同样走私有 commit fake，不能把 `disarm_for_test()` 发展成生产逃生口。

### 8.5 测试与 Gate

本提交再引入 test-only `FakeMemory`：

- OOM；
- backend 返回错误 base；
- allocation 逻辑长度不等于 request、fixed descriptor 超出授权范围；
- alignment 不满足；
- fixed range 未授权；
- reserve 成功后注入错误，abort 恰好一次；
- reserve 前失败，abort 次数为零；
- lease 不能 Clone/Copy；P0-C13 进一步让错误 transaction/staged payload 的组合无法通过公开 API 表达；
- backend 的 `allocate_image()` 自身必须 failure-atomic：返回 `Err` 时不能遗留只有 backend 知道的 allocation。

## 9. P0-C05：复制 segment 并确定性清零

建议提交：

```text
loader: copy segments and zero owned memory
```

### 9.1 本提交要解决的问题

当前 loader 只复制 file-backed bytes，依赖未初始化 heap 恰好为零。本提交第一次真正写目标内存，因此所有输入、布局和 allocation 必须已经验证完成。

### 9.2 实现内容

为 `ImageMemory` 增加：

```rust
fn write(
    &mut self,
    location: TargetLocation,
    data: &[u8],
) -> MemoryResult<()>;

fn zero(
    &mut self,
    location: TargetLocation,
    len: u64,
) -> MemoryResult<()>;
```

复制规则：

- `Dyn` owned allocation：先清零整个 `image_span`，再复制所有 segment 的 `p_filesz`；
- `Exec` fixed placement：复制每个 segment 的 `p_filesz` 并只清零尾部 `p_memsz - p_filesz`，不修改 segment gap；没有必要在覆盖 file bytes 前先清零整段；
- 使用固定大小 scratch buffer 分块调用 `read_exact_at()`，建议从 512 字节开始，根据线程栈预算调整；
- `LoadedRegion` 只覆盖真实 `PT_LOAD`；
- allocation 内的 gap 即使已经被清零，也不能被 `locate()` 当成合法映像内存；
- fixed placement 在第一次写前完成全部 source range、target range 和 entry preflight。

reader snapshot 与 fixed target 默认不得别名。若未来需要 in-place/XIP backend，必须提供专用 map/alias 能力和重叠复制算法，不能依赖 512 字节 scratch 恰好规避 source/target 覆盖。

### 9.3 因此才引入的数据结构

```rust
pub struct TargetLocation {
    allocation: AllocationId,
    offset: u64,
}

pub struct LoadedRegion {
    vaddr_range: TargetRange,
    runtime_range: TargetRange,
    file_range: FileRange,
    allocation_offset: u64,
    logical_permissions: MemoryPermissions,
}

pub struct MappedState {
    request: ArtifactRequest,
    allocation: ImageAllocation,
    load_bias: TargetAddr,
    entry: TargetAddr,
    canonical_entry: TargetAddr,
    regions: Vec<LoadedRegion>,
    dynamic: Option<DynamicSegmentInfo>,
    relro: Option<TargetRange>,
}

pub type MappedImage<'a, M> =
    StagedImage<'a, M, MappedState>;
```

核心方法：

```rust
impl<'a, M, R> StagedImage<'a, M, ReservedState<R>> {
    fn copy_and_zero(
        self,
    ) -> LoadResult<MappedImage<'a, M>>
    where
        R: ElfReader,
        M: ImageMemory;
}

impl MappedState {
    fn locate_vaddr(
        &self,
        vaddr: TargetAddr,
        len: u64,
        permissions: MemoryPermissions,
    ) -> LoadResult<TargetLocation>;

    fn runtime_address(
        &self,
        vaddr: TargetAddr,
        len: u64,
        permissions: MemoryPermissions,
    ) -> LoadResult<TargetAddr>;
}
```

### 9.4 Fixed `ET_EXEC` 的失败语义

Fixed range 不属于 loader，因此不能像 owned allocation 一样通过释放恢复旧内容。Phase 0 的规则是：

1. 所有只依赖 header/phdr、policy、范围、ABI、资源上限和 backend capability 的错误必须在第一次 fixed write 前发现；Phase 0 fixed profile 若不需要 runtime relocation，应在此时直接拒绝 `PT_DYNAMIC`，不要写入后才发现 unsupported tag；
2. I/O、copy、REL 隐式 addend、cache 维护和 protection apply 仍可能在首次写入后失败，因此不能承诺通用 fixed load 的强原子性；
3. 只有完整成功、seal 和 local commit 后才能安装 entry；
4. `BorrowedFixed + Reserved` 的 abort 可以恢复为可重试；进入 `BytesModified/ProtectionModified` 后，backend 必须将该 range 标记为 `Poisoned`，阻止 entry、再次 allocate 和静默复用，直到平台显式重置/重新初始化；
5. 如果某个平台需要 fixed range 的强回滚，只能通过全量 staging + 原子切换、A/B region、完整 snapshot restore 或 backend 原生事务实现，不能把 bookkeeping `release()` 描述为内容恢复。

### 9.5 测试与 Gate

- `FakeMemory` 初始填充 `0xa5`，BSS 最终为零；
- owned `ET_DYN` allocation gap 为零，但 `locate()` 拒绝 gap；
- fixed exec 不修改 segment gap；
- segment 大于 scratch buffer；
- 短读和每个 write/zero 故障点；
- owned allocation 在所有失败点恰好 abort/release 一次；
- fixed 在首次修改后的每个失败点均无 entry 且进入 poisoned；显式 reset 前重试被拒绝；
- reader/target alias 未声明时确定失败；
- 非零最低 vaddr 下 segment runtime address 正确。

## 10. P0-C06：归一化 Phase 0 所需的动态元数据

建议提交：

```text
loader: decode relative relocation metadata
```

### 10.1 本提交要解决的问题

Dynamic table 中保存的是 ELF 虚址，不能直接作为宿主指针使用。只有建立 `MappedImage` 和 region 索引后，才能证明 dynamic/relocation table 位于本映像的可读区域。

本提交只解析和验证，不修改 relocation target。

### 10.2 实现内容

新增：

```text
src/image/metadata.rs
```

为 `ImageMemory` 增加有界读取能力：

```rust
fn read(
    &self,
    location: TargetLocation,
    dst: &mut [u8],
) -> MemoryResult<()>;
```

处理规则：

- `PT_DYNAMIC` 必须完整落入可读 `PT_LOAD`；
- 必须在 file-backed dynamic entries 中找到 `DT_NULL`；
- 收集并验证 `DT_REL/DT_RELSZ/DT_RELENT`；
- 收集并验证 `DT_RELA/DT_RELASZ/DT_RELAENT`；
- 识别 `DT_RELR/DT_RELRSZ/DT_RELRENT`，Phase 0 profile 未启用时返回 `UnsupportedByProfile`；
- 先生成不带策略结论的 `DynamicFeatureSummary`，再由与 P0-C03 相同的 `ArtifactFeaturePolicy` 拒绝 `DT_NEEDED`、`DT_JMPREL/DT_PLTREL`、`DT_TEXTREL`、lifecycle、TLS、symbol version 等 Phase 0 不支持语义；
- IFUNC/COPY 等不是单纯 dynamic tag 的语义仍由 symbol/relocation decoder fail closed；
- 当前真实 `ET_DYN` 应用中的 `FLAGS_1/DEBUG/SYMTAB/STRTAB/HASH/GNU_HASH` 可以作为“已知但本阶段不消费”的 metadata 接受，不能仅因为当前未使用就拒绝真实基线工件；
- “接受 `DT_FLAGS/DT_FLAGS_1` tag”不等于接受任意 bit：Phase 0 只允许真实基线所需的 `DF_BIND_NOW`、`DF_1_NOW/DF_1_PIE` 等显式 mask，拒绝 `DF_ORIGIN/DF_SYMBOLIC/DF_STATIC_TLS` 以及 `DF_1_GROUP/NODELETE/INITFIRST/ORIGIN/DIRECT/INTERPOSE/NORELOC` 等会改变未实现语义的 bit；
- `DT_RELCOUNT/DT_RELACOUNT` 只作为经过边界验证的优化 hint；若不消费，则记录“ignored after full scan”，不能用它跳过 relocation type 检查；
- 自包含性在这里由策略结果确定：存在 `DT_NEEDED`、PLT relocation 或需要符号解析的 relocation 时，Phase 0 拒绝；不得回头把该结论缓存成 header admission 所需的应用类别；
- relocation 数量和 dynamic entry 数量均受 `LoadLimits` 控制。

### 10.3 因此才引入的数据结构

```rust
pub enum RelocationTableKind {
    Rel,
    Rela,
    Relr,
}

pub struct RelocationTableDescriptor {
    kind: RelocationTableKind,
    vaddr: TargetAddr,
    byte_len: u64,
    entry_size: u64,
}

pub enum RelocationAddend {
    Implicit,
    Explicit(i64),
}

pub struct RelocationRecord {
    offset: TargetAddr,
    raw_type: u32,
    symbol_index: u32,
    addend: RelocationAddend,
}

pub struct RuntimeImageMetadata {
    relocations: Vec<RelocationRecord>,
    features: DynamicFeatureSummary,
}

pub trait ArtifactFeaturePolicy {
    fn validate_program_features(
        &self,
        features: &ProgramFeatureSummary,
    ) -> LoadResult<()>;

    fn validate_dynamic_features(
        &self,
        features: &DynamicFeatureSummary,
    ) -> LoadResult<()>;
}

pub struct Phase0ArtifactPolicy;

pub struct RuntimeState {
    mapped: MappedState,
    metadata: RuntimeImageMetadata,
}

pub type RuntimeImage<'a, M> =
    StagedImage<'a, M, RuntimeState>;
```

`LoadLimits` 此时扩展：

```rust
max_dynamic_entries: u32,
max_relocations: u32,
max_runtime_metadata_bytes: u64,
max_relocation_operation_bytes: u64,
```

接口：

```rust
impl<'a, M: ImageMemory> StagedImage<'a, M, MappedState> {
    fn decode_runtime(
        self,
        policy: &dyn ArtifactFeaturePolicy,
    ) -> LoadResult<RuntimeImage<'a, M>>;
}
```

本提交不引入 `SymbolTable`、`RuntimeDynamicInfo` 的完整 DSO 视图、`ImageLifecycleMetadata` 或 `DT_NEEDED` graph，因为它们在 Phase 0 中没有消费者。

### 10.4 测试与 Gate

- 合法 REL 和 RELA；
- 错误 `RELENT/RELAENT`；
- table size 不是 entry size 的整数倍；
- table 越过 segment 或进入 gap；
- relocation 数量超限；
- dynamic tag 重复、缺失或冲突；
- `DT_NEEDED/TEXTREL/JMPREL/RELR` 确定失败；
- `DT_FLAGS/DT_FLAGS_1` 允许/拒绝 bit 矩阵，未知 bit fail closed；
- parser 不硬编码 Phase 0 policy；替换 policy 后同一份结构解析结果可以供 Phase 0.5 消费；
- ET_EXEC 或无 `PT_DYNAMIC` 映像可形成空 runtime metadata；
- 本提交不发生 relocation write。

## 11. P0-C07：迁移现有 RISC-V64 relative relocation

建议提交：

```text
loader: apply RISC-V relative relocations
```

### 11.1 本提交要解决的问题

RISC-V64 自包含 `ET_DYN` 是当前已有可执行基线。先把它迁移到通用 engine，可以在增加 ARM32 前证明新 reader/layout/memory 抽象没有破坏旧行为。

### 11.2 实现内容

新增：

```text
src/relocation/mod.rs
src/relocation/riscv.rs
```

支持：

```text
ELF64 + little-endian + EM_RISCV
DT_RELA + R_RISCV_RELATIVE
value = B + A
```

每条 relocation 固定执行：

1. 解码 raw type，未知类型立即失败；
2. relative relocation 的 symbol index 必须为 0；
3. checked 计算 `P = B + r_offset`；
4. `locate(r_offset, word_width, WRITE)`，拒绝 gap、只读区域和其他 owner；
5. 检查目标自然对齐；
6. RELA 使用显式 addend；
7. checked 计算 `value = B + A`；
8. 验证结果可编码为 ELF64；
9. 按 Phase 0 relative-value policy 验证结果：默认只接受当前 allocation 内的地址（允许 profile 明确声明的 null/one-past 例外），不能仅因“能编码成目标字宽”就接受任意内核地址；
10. 拒绝重复或字节范围重叠的 relocation target；未来确有 compound relocation 时只能由 arch profile 显式组成一项 operation；
11. 按目标端序写回。

### 11.3 因此才引入的数据结构

```rust
pub enum WordWidth {
    U32,
    U64,
}

pub enum AddendEncoding {
    Implicit,
    Explicit,
}

pub struct TargetWord {
    width: WordWidth,
    endian: Endian,
}

pub trait ArchRelocator {
    fn machine(&self) -> u16;
    fn class(&self) -> ElfClass;
    fn relative_type(&self) -> u32;
    fn addend_encoding(&self) -> AddendEncoding;
}

pub struct Riscv64Relocator;

pub struct RelocatedState {
    mapped: MappedState,
    metadata: RuntimeImageMetadata,
}

pub type RelocatedImage<'a, M> =
    StagedImage<'a, M, RelocatedState>;
```

`ArchRelocator` 只描述架构 ABI 差异；目标字宽由已经准入的 `ElfClass` 唯一推导，不能让 relocator 再声明一份可能矛盾的 `word_width`。通用 engine 负责先生成全部 `RelocationOperation`，按 target range 排序并检查碰撞，预检完成后再写回。`RelocationRecord` 与 operation 的合计峰值必须进入 `max_runtime_metadata_bytes/max_relocation_operation_bytes`；若后续改成两遍扫描来降低 RAM，必须先证明 reader snapshot 稳定、REL addend 不会被第一遍后的写入改变，并拒绝重复 target。`TargetWord/WordWidth` 到本提交才有实际用途，因为此时第一次按目标 ELF 字宽和端序读写 relocation value。

### 11.4 测试与 Gate

- 现有 RISC-V64 fixture；
- 非零最低 vaddr，证明使用 load bias 而不是 allocation base；
- 正/负 addend；
- symbol index 非零；
- target 越界、gap、只读或未对齐；
- 未知 relocation；
- 结果溢出和 write failure；
- duplicate/overlapping target、结果落入映像外和显式允许的边界例外；
- 任一错误都回滚 owned allocation。

旧 `load_elf()` 中的 RISC-V 实现暂时保留以维持兼容入口，但不得继续扩展；P0-C10 必须删除。

## 12. P0-C08：增加 ARM32 REL 隐式 addend

建议提交：

```text
loader: support ARM32 relative relocations
```

### 12.1 本提交要解决的问题

ARM32 与 RISC-V64 的关键差异不是 relocation 常量名称，而是：

- ELF32 目标字宽；
- `DT_REL` 没有显式 addend；
- addend 必须从经过验证的 relocation target 读取；
- Thumb 函数地址的 bit 0 必须保留。

### 12.2 实现内容

新增：

```text
src/relocation/arm.rs
```

支持：

```text
ELF32 + little-endian + EM_ARM
DT_REL + R_ARM_RELATIVE
```

执行时：

1. 通过 `RuntimeImage::locate()` 验证 P；
2. 按目标端序读取 32 位隐式 addend；
3. 按 ARM ELF ABI 解释 addend；
4. checked 计算 `B + A`；
5. 验证结果可编码为 32 位目标地址；
6. 应用与 RISC-V 相同的 relative-value policy 和 duplicate-target gate；
7. 写回时保留函数指针 Thumb bit；
8. 只在 executable membership 判断时使用清除 bit 0 的 canonical address。

### 12.3 因此才扩展的数据结构

只增加：

```rust
pub struct ArmRelocator;
```

不增加第二套 relocation engine，复用 P0-C07 的 `TargetWord`、预检操作列表和 `RelocatedImage`。

### 12.4 测试与 Gate

- 人工 ARM32 ELF golden fixture；
- LLD 真实 ARM32 fixture 与 `llvm-readelf -r` 对照；
- 隐式 addend；
- Thumb function pointer bit；
- target 未对齐、跨界、只读；
- symbol index 非零；
- 32 位结果溢出；
- 未知 ARM relocation；
- 明确拒绝 `ABS32/GLOB_DAT/JUMP_SLOT`，不把它们混入 Phase 0。

## 13. P0-C09：cache、最终权限与 seal

建议提交：

```text
loader: seal images after cache and permission updates
```

### 13.1 本提交要解决的问题

relocation 完成只表示映像字节暂时正确。代码可执行前仍需要：

- 同步新写入代码的 D/I cache；
- 将 text、rodata、data 和 RELRO 收敛到最终权限；
- 确保失败或半完成映像不能暴露 entry。

### 13.2 实现内容

`SealPlan` 是 loader 期望的逻辑权限，backend 还必须把它编译为能在真实保护粒度上执行的 prepared plan。建议等价接口如下：

```rust
pub trait CodeCache {
    // 发布环境要求；不属于 ELF artifact 准入属性。
    fn requirements(&self) -> CacheRequirements;

    fn prepare(
        &self,
        executable_ranges: &[TargetRange],
    ) -> LoadResult<PreparedCacheSync>;

    fn synchronize(
        &mut self,
        prepared: PreparedCacheSync,
    ) -> LoadResult<CacheSyncOutcome>;
}

pub trait ImageProtectionMemory: ImageMemory {
    // backend 只报告能力；core 据此构造不可伪造的 PreparedProtectionPlan。
    fn protection_capabilities(&self) -> ProtectionCapabilities;

    // 前置阶段：无可见副作用，检查 backend-specific writable alias。
    fn validate_protection_aliases(
        &self,
        allocation: &ImageAllocation,
        prepared: &PreparedProtectionPlan,
    ) -> LoadResult<()>;

    // core 从 prepared 取出固定长度的结果槽并包装为不可改集合的 capability；
    // backend 只能读取请求并按 index 原位记录 level，
    // 不能另造、遗漏或增加结果记录，apply 中也不得为 metadata 分配内存。
    fn apply_protection(
        &mut self,
        batch: ProtectionBatch<'_>,
    ) -> LoadResult<()>;
}
```

`PreparedProtectionPlan::prepare()` 是 crate-private core 路径：它根据 capability 完成 granule/region 数量编译，调用 alias preflight，并逐项执行 target access 校验。它不能作为带默认实现的 backend trait 方法，否则实现方可以覆盖核心计划生成，而外部实现又无法合法构造 opaque plan，接口契约会前后矛盾。

`SealPlan` 固定形成：

- text：RX；
- rodata：R；
- data/BSS：RW+NX；
- RELRO：从 RW 区域中切分为 R；
- gap：逻辑不可访问，有硬件能力时为 NONE；
- 相邻同权限范围合并；
- 拒绝 W+X、TEXTREL、可执行栈和 writable executable alias。

backend prepared plan 还必须满足：

- 覆盖整个 loader 可寻址 logical allocation，不能遗漏 backend 返回的额外尾部；本方案优先要求 logical allocation 与 request 精确等长，物理 padding 不对 loader 暴露；
- 记录 protection granule、实际 round 后的 range、MPU/PMP region 数量和 alias；
- granule rounding 后若 text/data、RELRO/RW 或 gap/segment 发生权限冲突，必须在 apply 前失败；不能靠调用顺序让“最后一次 protect 获胜”；
- 只有实际范围至少与请求同样严格且没有引入 W+X/越权访问时才能报告 `HardwareEnforced`；否则按 profile 返回 `LogicalOnly` 或确定失败。

执行顺序：

```text
构造并完整验证逻辑 SealPlan
  → prepare 全部 protection 和 cache capability（仍无副作用）
  → 一次同步全部新写入的 executable range
  → apply prepared protection batch
  → 形成 SealedState
```

`PreparedCacheSync` 除 scope/maintenance capability 外，必须原样保存调用方给出的全部 executable ranges；seal 在执行同步前逐项核对，不能接受遗漏、替换或扩大的 backend token。同步返回的 `CacheSyncOutcome` 还必须与已验证 token 的 ranges、scope 和 maintenance 完全一致，backend 不能在副作用完成后改报结果。`CacheRequirements` 在 prepare 前只读取一次并固定，属于 cache/publisher adapter，因此 Phase 0.5 的 `LinkSession` 可以对整批映像共享同一发布要求，而 `ArtifactRequest` 继续只描述 ELF、ABI 和资源上限。

cache backend 由 board/arch adapter 提供，不能仅凭 `target_arch` 假定硬件状态：

- 无 cache/关闭 cache 的 ARMv7-M：明确 capability 后执行 DSB/ISB；cache-enabled 板还需逐 line clean D-cache 到 PoU、invalidate I-cache 到 PoU 和正确 barrier；
- RISC-V：当前 hart 执行 `fence.i`；SMP 下必须广播/逐 hart 同步，或由 policy 保证提交后仅同步过的 hart 可执行；
- AArch64：Phase 0 不提供单映像 relocation/execution；保留的通用 cache primitive 使用正确的 D-cache clean/I-cache invalidate，不能被解释为 compatibility loader 已受支持；
- x86 host：可以报告架构指令缓存一致性，但不能把通用空 stub 当作所有架构实现。

如果 Phase 0 暂不支持某个既有架构（例如当前分支没有 AArch64 `load_elf`），必须在范围和 CI matrix 中显式写明；不能用通用 unsupported stub 同时声称“现有架构不回退”。

### 13.3 因此才引入的数据结构

```rust
pub struct SealPlan {
    ranges: Vec<SealRange>,
}

pub enum ProtectionLevel {
    HardwareEnforced,
    LogicalOnly,
}

pub struct SealRange {
    location: TargetLocation,
    runtime_range: TargetRange,
    permissions: MemoryPermissions,
}

pub struct ProtectionRecord {
    location: TargetLocation,
    requested_range: TargetRange,
    applied_range: TargetRange,
    permissions: MemoryPermissions,
    level: ProtectionLevel,
}

pub struct AppliedProtectionSet {
    ranges: Vec<ProtectionRecord>,
}

pub enum ExecutionScope {
    CurrentExecutionContext,
    AllExecutionContexts,
}

pub struct SealedState {
    mapped: MappedState,
    metadata: RuntimeImageMetadata,
    seal_plan: SealPlan,
    protections: AppliedProtectionSet,
    cache_sync: CacheSyncOutcome,
}
```

`ProtectionRecord` 是 prepared plan 预先分配的中性记录槽；apply 前 enforcement 保守为 `LogicalOnly`，core 始终持有这份固定长度存储，并通过 `ProtectionBatch` 只开放只读请求和按 index 写 level 的能力。backend 不能改写/交换记录，也不能返回另一份集合，因此无法漏报、增报范围或在权限副作用开始后再次分配。`PreparedProtectionPlan` 与 `AppliedProtectionSet` 负责表达同一批记录所处的阶段。

`SealedState` 是只证明 relocation、cache 和 protection 已完成的 state payload，仍处于 unpublished 状态。它沿用 reserve 时建立的同一个 guard：

```rust
pub type PreparedImage<'a, M> =
    StagedImage<'a, M, SealedState>;
```

不要暴露 `transaction.commit_for(&arbitrary_sealed)`。只有 `RelocatedImage::seal(self, cache)` 能形成 `PreparedImage`，所以 payload 与 rollback authority 从 reserve 起就没有分离过；Drop 自动 abort。成功路径把非 `Copy` 的 `AllocationLease` 移交给 committed owner 后才解除 guard。

pre-commit 的 entry 只供 crate 内部 publisher 校验：

```rust
impl SealedState {
    pub fn entry(&self) -> TargetAddr;
    pub fn canonical_entry(&self) -> TargetAddr;
    pub fn entry_instruction_span(&self) -> u64;
    pub fn seal_plan(&self) -> &SealPlan;
    pub fn protections(&self) -> &AppliedProtectionSet;
}
```

`PreparedImage` 不公开取得 state payload 的 accessor，因此普通调用方不能从 staged image 读取 entry；公共 `SealedState::entry()` 是给实现 `ImageCommitMemory` 的受信 publisher 在 `prepare_install()` 回调中校验并预构造 installed state 使用。

`MappedImage/RuntimeImage/RelocatedImage` 不提供同名方法；公共 entry 由成功接管 lease 的 committed owner 提供。

### 13.4 测试与 Gate

- RX/R/RW+NX 切分；
- RELRO 覆盖 RW 子区间；
- 相邻范围合并；
- 页/MPU granule 舍入后的权限冲突、region 数量不足和 writable alias 在 apply 前失败；
- cache 同步一次覆盖全部 X region，并记录 CurrentExecutionContext/AllExecutionContexts scope；
- cache/protect 任一步失败时 owned abort，modified fixed poisoned；
- 逐 range 保存 requested/applied range、唯一的目标 permission 和 `ProtectionLevel`；`HardwareEnforced` 表示 backend 精确落实了该 permission，不能用一份始终等于 requested 的冗余 “applied permission” 冒充后端证据；
- 无硬件保护时逐 range 明确返回 `LogicalOnly`；
- 没有可靠 cache backend 时加载失败；
- cache-enabled ARM 不能用 barrier-only backend 通过，RISC-V SMP 不能把 local `fence.i` 报告为 system-wide；
- `LogicalOnly` 不能被描述为硬件隔离。

## 14. P0-C10：切换兼容入口并删除旧实现

建议提交：

```text
loader: route load_elf through ImageLoader
```

### 14.1 本提交要解决的问题

P0-C01–P0-C09 已经分别证明读取、解析、布局、分配、复制、relocation 和 seal。此时才具备替换生产入口的条件。

### 14.2 新高层 API

```rust
pub fn prepare_image<'m, R, M, C, A>(
    reader: R,
    request: ArtifactRequest,
    memory: &'m mut M,
    cache: &mut C,
    relocator: &A,
) -> LoadResult<PreparedImage<'m, M>>
where
    R: ElfReader,
    M: ImageProtectionMemory,
    C: CodeCache,
    A: ArchRelocator + ?Sized;
```

内部只负责编排到 sealed-but-uncommitted：

```rust
let admitted = loader.admit(reader, request)?;
let planned = loader.plan(admitted)?; // 固定 Phase0ArtifactPolicy

let reserved = loader.reserve_staged(planned, memory)?;
let mapped = reserved.copy_and_zero()?;
let runtime = mapped.decode_runtime(&Phase0ArtifactPolicy)?;
let relocated = runtime.relocate(relocator)?;
let prepared = relocated.seal(cache)?;

Ok(prepared)
```

顶层 `prepare_image/load_image` 是 Phase 0 的可信单映像入口，必须固定使用 `Phase0ArtifactPolicy`；不能因为调用方传入一个更宽松的 policy，就把带 `DT_NEEDED`、TLS 或生命周期要求但尚未完成链接的映像提交。`ArtifactFeaturePolicy` 扩展点只保留在 `plan_with_policy()`、`decode_runtime()` 等未发布阶段，Phase 0.5 由 `LinkSession` 吸收 state 后使用。

提交使用两段式协议：

```text
PreparedImage::prepare_commit(self) -> LoadResult<ReadyImageCommit>
ReadyImageCommit::commit(self) -> CommitReceipt      // 不返回 Result，不再分配
```

`PreparedImage` 已经通过 transaction 独占借用同一个 memory backend，因此 `prepare_commit()` 不能再接收一个可能相同、也可能不同的独立 target 参数。P0-C13 在 `ImageCommitMemory: ImageProtectionMemory` 上增加 `prepare_install()`/`commit_install()`：前者通过 transaction 内部的 backend 校验 allocation 身份、entry、owner 容量和所有将要安装的字段，但不改变可见状态；后者只把预构造状态 move/swap 给 owner、转移 `AllocationLease` 并解除 rollback。`ReadyImageCommit` 和 `PreparedImage` 一样必须 `#[must_use]`，未 commit 的 Drop 仍触发 abort。

建议接口骨架：

```rust
pub trait ImageCommitMemory: ImageProtectionMemory {
    type PreparedInstall;
    type CommitReceipt;

    fn prepare_install(
        &mut self,
        allocation: &ImageAllocation,
        sealed: &SealedState,
    ) -> LoadResult<Self::PreparedInstall>;

    // 契约：不分配、不验证、不 panic、不返回 Result；只安装预构造状态并转移 lease。
    unsafe fn commit_install(
        &mut self,
        prepared: Self::PreparedInstall,
        sealed: SealedState,
        lease: AllocationLease,
    ) -> Self::CommitReceipt;
}

#[must_use = "ready commit still owns rollback authority"]
pub struct ReadyImageCommit<'a, M: ImageCommitMemory> {
    transaction: ImageLoadTransaction<'a, M>,
    sealed: SealedState,
    install: M::PreparedInstall,
}
```

`ReadyImageCommit::commit(mut self)` 先从 `transaction.pending` 中 `take()` 唯一 lease，再通过同一 `transaction.memory` 调用 `commit_install()`；`pending` 变为 `None` 后 transaction Drop 不再 abort。不得提供接收外部 allocation、sealed payload 或 backend 的 commit overload。

`commit_install()` 的返回值可以是持有 lease 的 `CommitReceipt`，也可以是 backend 已接管 lease 后的无权 `CommitReceipt`；backend 对自己的 receipt 只能选择一种所有权模式并写入契约。兼容 `MemoryMapper` 采用后者：installed state 持有唯一 lease，receipt 不能公开 entry。若平台无法把安装实现为不可失败操作，就必须把其全部动作保留在 prepare 阶段并提供失败原子性，不能先 disarm 再返回错误。

持有 lease 的 `M::CommitReceipt` 或 backend 的 installed state 是 allocation 的唯一成功 owner，保证 entry 使用期间 backend 和映射不被复用。Phase 0 可以不提供显式 unload，但必须定义 owner Drop：owned 调用 `release_committed()` 回收，borrowed fixed 只解除借用且不声称恢复旧内容；任何模式都不得按失败路径 poison 已成功提交的 fixed 映像。Phase 0.5 把同一 lease 转移到 `LinkContext`/`ReapResources`，不能把裸 `SealedState` 当成生命周期 owner。

### 14.3 兼容入口

旧接口保持参数兼容：

```rust
pub fn load_elf(
    buffer: &[u8],
    mapper: &mut MemoryMapper,
) -> core::result::Result<(), &'static str>;
```

它只负责：

1. 根据 mapper mode 构造 `ArtifactRequest`；
2. 构造 `SliceElfReader`；
3. 构造 `MemoryMapper` 和 board 明确选择的 cache adapter；
4. 调用 `prepare_image()`，此时 rollback 仍 armed；
5. 通过 `PreparedImage` 内部独占借用的 mapper 调用 fallible `prepare_install()`，预先验证 entry/allocation/owner 状态；
6. 在同一 mapper 上调用不可失败的 `commit_install()`，原子安装 entry、接管 lease 并解除 rollback；
7. 将 commit 前产生的结构化 `LoadError` 映射为兼容静态字符串。

`load_elf()` 内部不得再出现 ELF parse、segment copy 或 relocation 细节。

### 14.4 必须删除的旧代码

- 整文件生产路径 `Elf::parse(buffer)`；
- `build_memory_layout()`；
- `allocate_memory_for_segments()`；
- `copy_content_to_memory()`；
- 旧 `handle_riscv_relative_reloc()/relocate()`；
- 核心路径中的泛型 `write_value_at<T>`；
- 对未知 relocation 的静默跳过。

本提交同时移除旧实现不再需要的生产 `//librs` 依赖。

### 14.5 Gate

- 仓库中只有一个 program-header parser；
- 只有一个 segment mapping/copy 算法；
- 只有一个 relocation engine；
- 旧入口不包含 ELF 实现细节；
- 新入口返回结构化错误；
- `MemoryMapper::real_entry()` 只在 local commit 成功后可用；
- publish/install 故障发生时 rollback 仍 armed；commit 后不存在返回 `Result` 的步骤；
- staged payload 与 transaction 不能交叉配对，成功 owner 持有唯一非 `Copy` lease；
- ARM32 host fixture、现有 RISC-V64 `ET_DYN`、fixed `ET_EXEC` 和 QEMU 回归通过。

## 15. P0-C13–P0-C15：设计评审后的收口提交

P0-C00–P0-C12 不改写历史；新增三个可独立审查的提交关闭评审阻塞项：

| 提交 | 主要改动 | 必须通过的 gate |
| --- | --- | --- |
| P0-C13 `loader: keep image rollback armed through local commit` | `AllocationLease`、单 allocation `Option` guard、从 reserve 起携带 guard 的 `StagedImage<S>`、`PreparedImage/ReadyImageCommit`、同 backend 两段式 install、fixed poisoned 状态；阶段低层接口收为 `pub(crate)` | publisher/prepare-install 每点故障仍 abort；commit 后无 `Result`；wrong-transaction 在安全主路径不可表达；owned exactly-once release；modified fixed 不可重试；entry 存活期 owner 持有 lease |
| P0-C14 `loader: harden artifact and relocation policy` | snapshot reader 契约、ARM/RISC-V `e_flags`/entry mode、`ArtifactFeaturePolicy` 两阶段判定、dynamic flag mask、board limits/峰值 RAM、duplicate relocation 和 relative result policy、stage-neutral range error | source version、ABI flag/Thumb entry、dynamic flags、peak budget、duplicate/overlap target、映像外 value、精确 allocation len 和错误 stage 矩阵 |
| P0-C15 `loader: apply prepared protection and cache plans` | canonical segment permission、granule/slot/alias preflight、batch protection、逐 range `ProtectionRecord`、board cache capability/SMP scope、fuzz/corpus | granule 冲突在副作用前失败；partial apply 触发 owned abort/fixed poison；cache token 必须保持全部 X range；cache-enabled ARM 和 RISC-V SMP 不误报；所有 malformed/fault points 与真实 QEMU oracle 通过 |

P0-C13 只修正单映像事务，不引入 Phase 0.5 的 dependency graph、registry、generation table 或 loader lock。P0-C14/P0-C15 允许调整 P0-C01–P0-C09 已有类型，但不得重新增加第二套 parser、mapping 或 relocation 路径。

## 16. 最终代码布局

```text
kernel/loader/src/
  lib.rs
  error.rs
  address.rs
  identity.rs
  limits.rs
  reader.rs
  memory.rs
  cache.rs

  image/
    mod.rs
    parser.rs
    layout.rs
    loaded.rs
    metadata.rs
    seal.rs

  relocation/
    mod.rs
    arm.rs
    riscv.rs

  memory_mapper.rs       # Phase 0 先保留为兼容 adapter
```

不用为了目录形式提前移动 `memory_mapper.rs`。先完成 P0-C10 的单一路径切换，再决定是否在后续提交移动到 `compat/`，避免把文件移动噪声和语义改动混在一起。

每增加一个 Rust module，都必须同步加入 `kernel/loader/BUILD.gn` 的 `sources`，否则 Ninja 可能不能正确追踪 module 文件变化。

## 17. 类型引入时机

下表用于避免再次出现“先创建大量类型、后寻找用途”的实现方式。

| 类型 | 首次引入 | 当前用途 |
| --- | --- | --- |
| `LoadError` | P0-C01 | 表达 read/header/admission 错误，后续按消费者扩展 stage/kind/context |
| `ElfReader/SliceElfReader` | P0-C01 | 读取 header；后续同一接口读取 phdr 和 segment |
| `ArtifactProfile/Request` | P0-C01 | 描述单映像 ELF ABI 与 `Dyn/Exec` header 类型要求，不表达链接模型 |
| `TargetAddr/FileRange` | P0-C02 | 区分 ELF 虚址与文件范围 |
| `ParsedImage` | P0-C02 | 持有不借用输入文件的 program-header 视图 |
| `TargetRange/ImageLayout` | P0-C03 | 在 allocation 前证明完整布局有界且合法 |
| `AllocationId/ImageMemory` | P0-C04 | 第一次申请和释放目标内存 |
| `ImageLoadTransaction` | P0-C04，P0-C13 收紧 | 第一次产生需要 rollback 的副作用；最终只持一个 `Option<AllocationLease>` |
| `AllocationLease/MutationProgress` | P0-C13 | 唯一资源 authority，以及 owned abort/fixed poison 的进度依据 |
| `TargetLocation/MappedImage` | P0-C05 | 第一次按 allocation+offset 写目标内存 |
| `RelocationRecord/RuntimeImage` | P0-C06 | 第一次解析运行时 relocation metadata |
| `TargetWord/WordWidth` | P0-C07 | 第一次按目标 ELF 字宽读写 relocation value |
| `ArchRelocator/RelocatedImage` | P0-C07 | 执行并证明 relative relocation 已完成 |
| `ArmRelocator` | P0-C08 | 增加 ARM32 REL 隐式 addend |
| `CodeCache/SealPlan/SealedState` | P0-C09 | 第一次允许把映像变为 sealed 但仍未提交的结果 |
| `StagedImage<S>/PreparedImage/ReadyImageCommit/M::CommitReceipt` | P0-C13 | 从 reserve 起把每个 state payload 与同一 rollback authority 绑定，并把 lease 转给 backend 定义的成功 owner，保证 commit 后无失败点 |
| `ArtifactFeaturePolicy` | P0-C14 | 在 S1/S4 统一执行 program/dynamic feature 判定 |
| `PreparedProtectionPlan/ProtectionRecord/AppliedProtectionSet` | P0-C15 | 区分逻辑计划、预分配记录、backend 实际范围和真实保护级别 |

以下类型不在 Phase 0 引入：

- `ImageId`：单映像事务没有身份去重需求，Phase 0.5 依赖图出现时再引入；
- `ArtifactIdentity/FileIdentity/BuildId`：Phase 0.5/1 的 catalog 和准入才消费；
- `LinkDomainId`：Phase 1 ApplicationManager/domain 才消费；
- `SymbolTable/ResolvedSymbol/ScopeSet`：Phase 0.5 symbol relocation 才消费；
- `ImageLifecycleMetadata/InitPlan/FiniPlan`：Phase 0.5/1 生命周期阶段才消费；
- `RelocationPolicy` 完整对象：Phase 0 只有固定的单映像 relative 类型/value 规则，Phase 0.5 出现跨映像符号 owner/scope 后再形成独立策略对象。

## 18. 测试架构

### 18.1 Host 测试替身

测试替身按生产 trait 出现顺序加入：

- P0-C00 `ElfFixtureBuilder`：构造小型 ELF32/ELF64；
- P0-C01 `RecordingReader`：记录 read-at 并在第 N 次读取注入 short read/I/O error；
- P0-C13 `VersionedReader`：验证一次 load 只能观察同一 snapshot；
- P0-C04 `FakeMemory`：以 `0xa5` 预填充，记录 allocate/abort；
- P0-C05 扩展 `FakeMemory`：记录 write/zero 并支持逐调用失败；
- P0-C06 扩展 `FakeMemory`：记录有界 read 并支持逐调用失败；
- P0-C07 扩展 `FakeMemory`：支持目标 word read/write；
- P0-C09/P0-C15 `FakeCodeCache`：记录全部 X range、execution-context scope/capability，支持遗漏 range、capability 不足和同步失败；
- P0-C15 `FakeProtectionBackend`：配置 granule/region 数量、enforcement level 和逐调用失败；
- P0-C13 `FakeCommitTarget`：分别在 preflight、install 前后验证 rollback authority 和不可失败 commit。

异常输入优先使用小型人工 fixture；真实工具链 fixture 用于验证 ABI 和 relocation oracle，两者不能相互替代。

P0-C15 在 `kernel/loader/fuzz/loader_fuzz.rs` 增加 parser/layout/dynamic/relocation 共用的 fuzz target，并在 `kernel/loader/fuzz/corpus/` 保存空输入、截断 magic、ARM32 合法最小映像、RISC-V64 合法最小映像和带 `R_RISCV_RELATIVE` 的深路径种子。target 在 `cfg(fuzzing)` 下导出 libFuzzer ABI；普通 GN host 构建则对 corpus、边界截断和定点 bit flip 作确定回归，`check_loader`/`check_loader_host` 都会执行。fuzz 只负责发现 panic、越界、非确定错误和资源上限绕过；每个发现仍须收敛为可独立运行的 deterministic unit test，不能用“跑过一段时间”替代下表语义 oracle。

### 18.2 单元测试矩阵

| 模块 | 必测场景 |
| --- | --- |
| reader/admission | 空文件、短读、snapshot 改变、错误 class/endian/machine/type/e_flags/entry mode、phdr 范围溢出、board/峰值字节上限 |
| parser/policy | 无 section header、多 `PT_LOAD`、重复 dynamic/stack；parser 记录 interp/tls，Phase 0 policy 在 allocation 前拒绝 |
| layout | 非零 min vaddr、gap、非法 align、不同余、filesz>memsz、overlap、非 canonical 权限、entry 非 X/未对齐/ARM even/最小指令跨度越界 |
| allocation | OOM、错误 base/exact len/align、fixed 越权、owned abort exactly once、modified fixed poison/reset |
| mapping | 大 BSS、预填充非零、分块 copy、gap locate、reader/target alias、read/write/zero failure |
| metadata | REL/RELA descriptor、table 越界、错误 entry size、数量/字节上限、dynamic tag/flag bit matrix |
| relocation | RISC-V64 RELA、ARM32 REL、隐式 addend、未知类型、symbol!=0、越界、未对齐、溢出、duplicate/overlap target、映像外 value |
| seal | RX/R/RW+NX、RELRO、gap NONE、granule/slot/alias 冲突、requested/applied range 与唯一目标 permission、cache capability/scope、partial apply failure |
| transaction | publisher preflight failure、wrong payload、stage Drop、commit 后无 fallible step、成功 owner/lease 存活 |
| errors | 每个阶段的 arithmetic/backend failure 保留当前 `LoadStage` 和 primary error；不可失败 abort 的异常只能进入 backend diagnostics，不能覆盖 primary error |

### 18.3 集成回归

- 现有自包含 `ET_DYN` 应用加载并执行；
- fixed `ET_EXEC` 加载并执行；
- fixed range 越界在第一次写之前失败；
- fixed 在 post-write 故障后保持无 entry 且 poisoned，显式 reset 后才能重试；
- RISC-V64 `ET_DYN` 应用使用新 engine；
- ARM32 `DT_REL/R_ARM_RELATIVE` host fixture 精确比较目标字节；
- LLD ARM32 `ET_DYN` 工件包含 `R_ARM_RELATIVE`，QEMU 执行时解引用 relocation 后的指针；
- 大 BSS 在非零预填充内存上仍为零；
- 最低 vaddr 非零时 entry 和 relocation value 正确；
- QEMU 执行新写入代码前经过真实架构 cache backend；
- 对 AArch64 做明确决策：补齐兼容执行回归，或在 Phase 0 支持矩阵中正式移除；不能静默返回 unsupported 同时宣称现有路径不回退。

建议最终运行：

```bash
ninja -C out/qemu_riscv64.release.dsc kernel/loader:check_loader kernel/loader:blueos_loader_clippy
ninja -C out/qemu_mps2_an385.release.dsc kernel/loader:check_loader kernel/loader:blueos_loader_clippy
ninja -C out/seeed_xiao_esp32c3.release.dsc kernel/loader:check_loader kernel/loader:blueos_loader_clippy
```

ESP32-C3 的 image action 依赖 `esptool` 4.x API，并会下载官方 bootloader 和 partition table；已验证版本为 `esptool==4.8.1`，5.x 不再导出脚本当前使用的 `arg_auto_chunk_size`。可以把依赖安装到临时目录并以 `PYTHONPATH` 传给 Ninja，避免修改系统 Python。离线环境还应预置两个镜像输入；若 Python TLS trust store 不完整，可先通过系统下载工具取得同一官方 release 文件，再用 `build/scripts/gen_esp32_image.py build_image` 生成测试镜像。

## 19. Phase 0 最终验收清单

下列 `[x]` 表示 P0-C00–P0-C12 基线已经满足；`[ ]` 是 P0-C13–P0-C15 合入前仍需关闭的门禁：

- [x] 所有 ELF 数值先 checked arithmetic，后转换为 host 类型；
- [x] parser 不依赖 section header；
- [x] `p_filesz <= p_memsz`、alignment、同余、overlap 和 entry 权限均校验；
- [x] 非零最低 vaddr 使用正确 load bias；
- [x] owned `ET_DYN` allocation 和 BSS 确定性清零；
- [x] `locate()` 拒绝 allocation gap 和错误权限；
- [x] relative relocation 除 target owner/范围/字宽/端序/对齐/溢出外，还拒绝 duplicate/overlap target，并按 profile 校验结果地址；
- [x] 未知或非白名单 relocation fail closed；
- [x] parser 与 policy 分离；`PT_INTERP/PT_TLS/DT_NEEDED/TEXTREL` 及 `DT_FLAGS/DT_FLAGS_1` bit mask 由同一 `ArtifactFeaturePolicy` 判定；
- [x] `ElfReader` 固化不可变 snapshot 契约，VFS/source 变化不能形成 TOCTOU；
- [x] ARM/RISC-V `e_flags`、Thumb entry bit、指令对齐和 board-specific limits 全部进入 admission gate；
- [x] cache backend 缺失时失败；cache-enabled ARM 和 RISC-V SMP 不把 barrier/local-hart 同步误报为完整同步；
- [x] 逐 range 记录 requested/applied protection；granule、MPU slot、alias 和 RELRO 冲突在 apply 前失败；
- [x] segment gap、alignment padding 和整个 logical allocation 均进入 `SealPlan`；backend 物理 padding 不对 loader 暴露；
- [x] owned S2–S8 失败 exactly-once abort；modified borrowed fixed 明确 poisoned，不能声称恢复旧内容；
- [x] reserve 后每个 `StagedImage<S>` 都携带同一 transaction，转换 API 不再接受独立 transaction 参数；错误 transaction/payload 不可配对或提交；
- [x] reader/write/zero/read/cache/protection/publisher 每个调用点均有故障注入，错误保留正确 `LoadStage`；
- [x] 只有成功接管 `AllocationLease` 的 committed owner 公开 entry，且 owner 保证 entry 使用期间映射存活；
- [x] 所有 fallible publish/install preflight 在 commit 前完成，commit 本身不分配、不返回 `Result`；
- [x] 旧 `load_elf()` 只剩兼容包装；
- [x] 仓库中没有第二套生产 parser、copy 或 relocation 实现；
- [x] ARM32 host fixture、RISC-V64 `ET_DYN`、RISC-V32 fixed `ET_EXEC` 和 QEMU 回归通过。
- [x] AArch64 compatibility relocation/execution 正式移出 Phase 0，并由 `load_elf()` 明确拒绝；
- [x] malformed corpus/fuzz target 纳入 `check_loader` 与 `check_loader_host`。

达到以上条件后，Phase 0.5 才可以直接以 `RuntimeImage/RelocatedImage/PreparedImage` 为基础增加 dependency graph、symbol scope 和多映像 `LinkSession`，并把单 allocation guard 扩展为多资源 rollback log，而无需再次推翻单映像的地址、布局、复制、relative relocation、seal 和 ownership 语义。
