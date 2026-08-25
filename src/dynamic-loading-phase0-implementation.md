# BlueOS 动态加载 Phase 0 详细实现方案

本文是 [BlueOS 动态加载详细实现计划](./dynamic-loading-implementation-plan.md) 中 Phase 0 的独立实施文档。目标不是预先创建一组“以后可能有用”的类型，而是按照正常工程实现顺序推进：每个 commit 先交付一种可以测试的能力，再引入完成该能力所必需的数据结构和方法。

Phase 0 完成后，当前的 `load_elf(buffer, mapper)` 将成为可信单映像 `ImageLoader` 的兼容包装；现有自包含 `ET_DYN`、fixed-address `ET_EXEC` 和 RISC-V64 relative relocation 继续工作，同时补齐 ARM32 relative relocation、BSS、load bias、错误处理、权限与 cache 基线，为 Phase 0.5 的 `DynamicLinker` 提供稳定输入。

## 0. 实施状态（2026-08-25）

Phase 0 已在 `kernel` 仓库的 `feat/loader-phase0` 分支按本文顺序完成。实现中发现旧方案把“static PIE”错误地当作启动方输入，因此 C04 同时修正了 C01–C03 的类型契约：公开请求现在只有 `ExpectedElfType::{Dyn, Exec}`，自包含性由 C06 的 `RuntimeFeaturePolicy::Phase0` 在扫描 `PT_DYNAMIC` 后确定。

| 阶段 | Commit | 已交付结果 |
| --- | --- | --- |
| C00 | `8b70ca8` | host regression fixture 和 GN test target |
| C01 | `13173cd` | read-at 输入、ELF header/ABI/type 准入和结构化错误 |
| C02 | `13b04db` | 自有 program-header 运行时视图 |
| C03 | `8acde3c` | checked layout、W^X、entry、load bias 和资源上限 |
| C04 | `57ced2a` | `ExpectedElfType` 修正、fallible allocation 和事务回滚 |
| C05 | `d1b9f2b` | 分块复制、BSS/gap 清零和 fixed placement 写前预检 |
| C06 | `8aac272` | 有界 `PT_DYNAMIC` 扫描、Phase 0 策略和 REL/RELA 归一化 |
| C07 | `f5616f6` | RISC-V64 RELA relative relocation engine |
| C08 | `1734d4b` | ARM32 REL 隐式 addend 和 ELF32 地址宽度检查 |
| C09 | `429409b` | cache 同步、RELRO/最终权限、seal 和事务提交 |
| C10 | `46478c9` | `load_image()` 编排、兼容入口切换和旧管线删除 |

最终验证覆盖 40 个 host tests，以及 RISC-V64 `ET_DYN`、ARM32 `ET_DYN`、RISC-V32 fixed `ET_EXEC` 三条真实 QEMU 执行路径。下面保留按 commit 解释“为什么此时引入这些结构”的实施逻辑；若示例草图与实现细节有差异，以本节 commit 对应的代码和各节标注的修订为准。

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
- ELF header、program header、ABI 和资源上限校验；
- checked arithmetic、正确 load bias、segment 布局和最大对齐；
- 映像分配、分块复制、BSS/gap 清零和失败回滚；
- 现有 RISC-V64 `RELA + R_RISCV_RELATIVE`；
- ARM32 `REL + R_ARM_RELATIVE`；
- relative relocation 的范围、owner、权限、字宽、端序、对齐和溢出校验；
- 未知 relocation fail closed；
- logical permissions、RELRO、指令缓存同步和 seal；
- `load_elf()` 兼容入口转调新管线；
- host malformed/fault-injection tests 和现有 QEMU 回归。

### 2.2 明确不做

Phase 0 不实现：

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
- Phase 0 在 C06 对 `PT_DYNAMIC` 做有界扫描，并通过 `RuntimeFeaturePolicy` 拒绝 `DT_NEEDED`、符号型 relocation 等本阶段不支持的语义；通过该策略意味着“本次可以由 Phase 0 单映像管线完成”，而不是给工件永久贴上 static PIE 标签；
- Phase 0.5 放宽运行时策略并建立依赖图，但沿用同一个 `ExpectedElfType::Dyn` 请求 ABI。

扫描 `PT_DYNAMIC` 不是为了额外识别应用类别；relocation 本来就必须读取这些元数据。扫描次数受 `max_dynamic_entries` 限制，复杂度与后续 relocation 数量校验相比不是独立的启动成本。

### 2.4 完成标准

完成标准不是“新 API 已经存在”，而是：

1. 旧 parse/copy/relocate 实现已经删除；
2. 所有输入走同一组阶段转换；
3. 所有 ELF 范围在产生副作用前完成验证；
4. S2–S8 任一步失败都不泄漏 owned allocation；
5. 只有 `SealedImage` 能暴露 entry；
6. ARM32 host fixture、现有 RISC-V64 `ET_DYN`、fixed `ET_EXEC` 和 QEMU 回归全部通过。

## 3. 总体实现主线

```text
C00 锁定测试基线
  ↓
C01 从任意来源安全读取并准入 ELF
  ↓
C02 解析自有的 program-header 运行时视图
  ↓
C03 在写内存前生成完整布局计划
  ↓
C04 预留目标内存并建立事务回滚
  ↓
C05 复制 PT_LOAD，确定性清零 BSS/gap
  ↓
C06 从 PT_DYNAMIC 归一化 relative relocation 元数据
  ↓
C07 把现有 RISC-V64 relative relocation 迁入新 engine
  ↓
C08 增加 ARM32 REL 隐式 addend
  ↓
C09 同步 cache、应用最终权限并 seal
  ↓
C10 切换 load_elf()，删除旧管线
```

C00–C09 期间旧 `load_elf()` 暂时保持可用；C10 才切换生产入口。这会使新旧实现短期共存，但旧路径在此期间只做兼容基线，不继续扩展。C10 合入后不得保留第二个 parser、映射算法或 relocation engine。

最终运行时转换为：

```text
ElfReader
  → admit            → AdmittedArtifact<R>
  → inspect + plan   → PlannedArtifact<R>
  → reserve          → ReservedImage<R>
  → copy + zero      → MappedImage
  → decode runtime   → RuntimeImage
  → relocate         → RelocatedImage
  → seal             → SealedImage
```

类型设计遵循以下规则：

1. 阶段转换消费上一阶段对象并返回下一阶段对象；
2. 阶段类型的字段和构造函数为 private 或 `pub(crate)`；
3. 不提供公共 `set_state()`；
4. `MappedImage/RuntimeImage/RelocatedImage` 不暴露 entry；
5. 值类型在第一次出现真实消费者时引入，不按终态类型表一次性创建。

## 4. C00：建立可运行的测试基线

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

## 5. C01：用 read-at 接口完成 ELF 身份准入

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

这些类型的当前目的：

- `ArtifactProfile`：描述 loader 能处理的目标 ELF ABI；
- `ArtifactRequest`：描述一次单映像装载的 ELF 类型、ABI 和资源要求，不声明应用链接模型；
- `AdmittedArtifact`：证明 header、ABI 和 phdr table 外层范围已经验证；
- `LoadLimits`：让文件长度和 phdr 数量在 allocation 前受控；
- `LoadError`：让测试和上层能够区分 read/parse/validate 错误。

不要在本提交引入路径、build-id、签名、`ExecutionModel`、`LinkDomainId` 或 `ImageId`。它们在单映像 Phase 0 中没有消费者。

### 5.4 测试与 Gate

- 空文件、截断 header、短读；
- 错误 class/endian/machine/type；
- `e_phoff + e_phnum * e_phentsize` 溢出或越界；
- 文件长度和 phdr 数量超限；
- `SliceElfReader` 的 offset 转换和 `offset + len` 溢出；
- `ImageLoader::admit()` 不接受 `ImageMemory` 参数，从接口上保证本阶段不会分配；
- 旧 `load_elf()` 行为不变。

## 6. C02：解析自有的 program-header 视图

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
- 用于拒绝的 `PT_INTERP/PT_TLS`。

不读取 section header，也不要求文件存在 section header。

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
    load_segments: Box<[LoadSegmentInfo]>,
    dynamic: Option<DynamicSegmentInfo>,
    relro: Option<TargetRange>,
    stack_policy: StackPolicy,
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

`inspect()` 先保持 `pub(crate)`；C03 的 `plan()` 才形成调用方可使用的完整阶段边界。

### 6.4 测试与 Gate

- 多个 `PT_LOAD`；
- 无 section header；
- phdr table 截断；
- segment file range 越界；
- 重复 `PT_DYNAMIC/PT_GNU_STACK`；
- `PT_INTERP/PT_TLS` 和可执行栈被明确拒绝；
- `ParsedImage` 不含输入 slice 引用；
- 临时 phdr buffer 在 `inspect()` 返回后即可释放。

## 7. C03：在分配前生成完整布局计划

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

1. 至少存在一个非空 `PT_LOAD`；
2. `p_filesz <= p_memsz`；
3. `p_align` 为 0、1 或 2 的幂，0 归一化为 1；
4. `p_offset % p_align == p_vaddr % p_align`；
5. file range、vaddr range 和 image span 全部 checked；
6. segment 数量和 span 不超过 `LoadLimits`；
7. 拒绝 W+X；
8. 按 vaddr 排序；Phase 0 拒绝任何非空 overlap；
9. 计算所有 segment 的 `max_align`；
10. 计算 `aligned_min_vaddr/aligned_max_vaddr/image_span`；
11. entry 必须落在真实 X segment，不能只位于 allocation 包络或 gap 中。

ARM32 entry 需要区分：

- 范围验证用 canonical address，即清除 Thumb bit；
- 最终执行地址保留 bit 0。

### 7.3 因此才引入的数据结构

```rust
pub struct SegmentLayout {
    vaddr_range: TargetRange,
    file_range: FileRange,
    permissions: MemoryPermissions,
}

pub struct ImageLayout {
    aligned_min_vaddr: TargetAddr,
    aligned_max_vaddr: TargetAddr,
    image_span: u64,
    max_align: u64,
    entry_vaddr: TargetAddr,
    segments: Box<[SegmentLayout]>,
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
- entry 位于 gap、只读 segment 或映像外；
- ARM Thumb entry membership；
- allocator 调用次数仍为零。

## 8. C04：预留目标内存并建立事务回滚

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

`ImageMemory` 在本提交只包含当前需要的方法：

```rust
pub trait ImageMemory {
    fn allocate_image(
        &mut self,
        request: &AllocationRequest,
    ) -> LoadResult<ImageAllocation>;

    fn release(&mut self, allocation: AllocationId);
}
```

实现 `MemoryMapper` compatibility adapter：

- allocated mode 使用 fallible `Storage::try_from_layout()`；
- fixed mode 返回 borrowed/fixed allocation；
- 校验 backend 返回的 target base、长度和对齐；
- fixed placement 必须完全位于授权 range；
- load bias 只能从经过验证的 allocation 计算。

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

pub struct ImageAllocation {
    id: AllocationId,
    target_base: TargetAddr,
    len: u64,
    align: u64,
    ownership: AllocationOwnership,
}

pub struct ReservedImage<R> {
    artifact: AdmittedArtifact<R>,
    parsed: ParsedImage,
    layout: ImageLayout,
    allocation: ImageAllocation,
    load_bias: TargetAddr,
}

pub struct ImageLoadTransaction<'a, M: ImageMemory> {
    memory: &'a mut M,
    rollback_allocations: Vec<AllocationId>,
    committed: bool,
}
```

`ImageLoadTransaction` 的当前唯一目的：allocation 产生后，S2–S8 任一步失败都恰好 release 一次。

### 8.4 核心方法

```rust
impl ImageLoader {
    fn reserve<R, M>(
        &self,
        planned: PlannedArtifact<R>,
        transaction: &mut ImageLoadTransaction<'_, M>,
    ) -> LoadResult<ReservedImage<R>>
    where
        R: ElfReader,
        M: ImageMemory;
}

impl<M: ImageMemory> ImageLoadTransaction<'_, M> {
    fn disarm_for_test(self);
}
```

`disarm_for_test()` 只用于 C04–C08 验证成功路径不会触发 rollback，不构成正式发布接口。C09 得到 `SealedImage` 后再把它替换为 `commit_for(self, &SealedImage)`，避免 C04 提前依赖尚不存在的 sealed 类型。

### 8.5 测试与 Gate

本提交再引入 test-only `FakeMemory`：

- OOM；
- backend 返回错误 base；
- allocation 长度不足；
- alignment 不满足；
- fixed range 未授权；
- reserve 成功后注入错误，release 恰好一次；
- reserve 前失败，release 次数为零；
- commit 后 transaction 不再释放 allocation。

## 9. C05：复制 segment 并确定性清零

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
) -> LoadResult<()>;

fn zero(
    &mut self,
    location: TargetLocation,
    len: u64,
) -> LoadResult<()>;
```

复制规则：

- `Dyn` owned allocation：先清零整个 `image_span`，再复制所有 segment 的 `p_filesz`；
- `Exec` fixed placement：只清零每个 segment 的 `p_memsz`，不修改 segment gap；
- 使用固定大小 scratch buffer 分块调用 `read_exact_at()`，建议从 512 字节开始，根据线程栈预算调整；
- `LoadedRegion` 只覆盖真实 `PT_LOAD`；
- allocation 内的 gap 即使已经被清零，也不能被 `locate()` 当成合法映像内存；
- fixed placement 在第一次写前完成全部 source range、target range 和 entry preflight。

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

pub struct MappedImage {
    request: ArtifactRequest,
    allocation: ImageAllocation,
    image_span: u64,
    load_bias: TargetAddr,
    entry: TargetAddr,
    canonical_entry: TargetAddr,
    regions: Box<[LoadedRegion]>,
    dynamic: Option<DynamicSegmentInfo>,
    relro: Option<TargetRange>,
}
```

核心方法：

```rust
impl ImageLoader {
    fn copy_and_zero<R, M>(
        &self,
        reserved: ReservedImage<R>,
        transaction: &mut ImageLoadTransaction<'_, M>,
    ) -> LoadResult<MappedImage>
    where
        R: ElfReader,
        M: ImageMemory;
}

impl MappedImage {
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

1. 所有可预见的 parser/layout/source/target 错误必须在第一次 fixed write 前发现；
2. 只有完整成功并 seal 后才能安装 entry；
3. 若 backend 在写入途中报告不可恢复错误，映像保持 unpublished，调用方必须放弃或重新初始化该 fixed region；
4. 不把“释放 allocation”描述为 fixed range rollback。

### 9.5 测试与 Gate

- `FakeMemory` 初始填充 `0xa5`，BSS 最终为零；
- owned `ET_DYN` allocation gap 为零，但 `locate()` 拒绝 gap；
- fixed exec 不修改 segment gap；
- segment 大于 scratch buffer；
- 短读和每个 write/zero 故障点；
- owned allocation 在所有失败点恰好 release 一次；
- 非零最低 vaddr 下 segment runtime address 正确。

## 10. C06：归一化 Phase 0 所需的动态元数据

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
) -> LoadResult<()>;
```

处理规则：

- `PT_DYNAMIC` 必须完整落入可读 `PT_LOAD`；
- 必须在 file-backed dynamic entries 中找到 `DT_NULL`；
- 收集并验证 `DT_REL/DT_RELSZ/DT_RELENT`；
- 收集并验证 `DT_RELA/DT_RELASZ/DT_RELAENT`；
- 识别 `DT_RELR/DT_RELRSZ/DT_RELRENT`，Phase 0 profile 未启用时返回 `UnsupportedByProfile`；
- 拒绝 `DT_NEEDED`、`DT_JMPREL/DT_PLTREL` 和 `DT_TEXTREL`；
- 拒绝 TLS、symbol version、IFUNC、COPY 等 Phase 0 不支持语义；
- 当前真实 `ET_DYN` 应用中的 `FLAGS_1/DEBUG/SYMTAB/STRTAB/HASH/GNU_HASH` 可以作为“已知但本阶段不消费”的 metadata 接受，不能仅因为当前未使用就拒绝真实基线工件；
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
    relocations: Box<[RelocationRecord]>,
}

pub struct RuntimeImage {
    mapped: MappedImage,
    metadata: RuntimeImageMetadata,
}
```

`LoadLimits` 此时扩展：

```rust
max_dynamic_entries: u32,
max_relocations: u32,
```

接口：

```rust
impl MappedImage {
    fn decode_runtime<M: ImageMemory>(
        self,
        transaction: &mut ImageLoadTransaction<'_, M>,
        policy: &RuntimeFeaturePolicy,
    ) -> LoadResult<RuntimeImage>;
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
- ET_EXEC 或无 `PT_DYNAMIC` 映像可形成空 runtime metadata；
- 本提交不发生 relocation write。

## 11. C07：迁移现有 RISC-V64 relative relocation

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
9. 按目标端序写回。

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
    fn word_width(&self) -> WordWidth;
    fn relative_type(&self) -> u32;
    fn addend_encoding(&self) -> AddendEncoding;
}

pub struct Riscv64Relocator;

pub struct RelocatedImage {
    mapped: MappedImage,
    metadata: RuntimeImageMetadata,
}
```

`ArchRelocator` 只描述架构 ABI 差异；通用 engine 负责先生成全部 `RelocationOperation`，预检完成后再写回。`TargetWord/WordWidth` 到本提交才有实际用途，因为此时第一次按目标 ELF 字宽和端序读写 relocation value。

### 11.4 测试与 Gate

- 现有 RISC-V64 fixture；
- 非零最低 vaddr，证明使用 load bias 而不是 allocation base；
- 正/负 addend；
- symbol index 非零；
- target 越界、gap、只读或未对齐；
- 未知 relocation；
- 结果溢出和 write failure；
- 任一错误都回滚 owned allocation。

旧 `load_elf()` 中的 RISC-V 实现暂时保留以维持兼容入口，但不得继续扩展；C10 必须删除。

## 12. C08：增加 ARM32 REL 隐式 addend

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
6. 写回时保留函数指针 Thumb bit；
7. 只在 executable membership 判断时使用清除 bit 0 的 canonical address。

### 12.3 因此才扩展的数据结构

只增加：

```rust
pub struct ArmRelocator;
```

不增加第二套 relocation engine，复用 C07 的 `TargetWord`、预检操作列表和 `RelocatedImage`。

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

## 13. C09：cache、最终权限与 seal

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

增加：

```rust
pub trait CodeCache {
    fn synchronize(&mut self, runtime_range: TargetRange) -> LoadResult<()>;
}
```

为 `ImageMemory` 增加：

```rust
fn protect(
    &mut self,
    location: TargetLocation,
    len: u64,
    permissions: MemoryPermissions,
) -> LoadResult<ProtectionLevel>;
```

`SealPlan` 固定形成：

- text：RX；
- rodata：R；
- data/BSS：RW+NX；
- RELRO：从 RW 区域中切分为 R；
- gap：逻辑不可访问，有硬件能力时为 NONE；
- 相邻同权限范围合并；
- 拒绝 W+X、TEXTREL、可执行栈和 writable executable alias。

执行顺序：

```text
构造并完整验证 SealPlan
  → 同步所有新写入的 executable range
  → 应用最终 protection
  → 形成 SealedImage
```

兼容路径至少提供：

- ARMv7-M：DSB/ISB；有 cache 的板还需 D-cache clean/I-cache invalidate；
- RISC-V：`fence.i`；
- AArch64：若现有 loader 测试继续执行动态写入代码，则实现正确的 D-cache clean/I-cache invalidate；
- x86 host：可以报告架构指令缓存一致性，但不能把通用空 stub 当作所有架构实现。

### 13.3 因此才引入的数据结构

```rust
pub struct SealPlan {
    ranges: Box<[SealRange]>,
}

pub enum ProtectionLevel {
    Hardware,
    LogicalOnly,
}

pub struct SealRange {
    location: TargetLocation,
    runtime_range: TargetRange,
    permissions: MemoryPermissions,
}

pub struct SealedImage {
    mapped: MappedImage,
    metadata: RuntimeImageMetadata,
    seal_plan: SealPlan,
    protection: ProtectionLevel,
}
```

此时把 C04 的 test-only disarm 收敛为正式事务提交接口：

```rust
impl<M: ImageMemory> ImageLoadTransaction<'_, M> {
    fn commit_for(self, sealed: &SealedImage) -> LoadResult<()>;
}
```

`commit_for()` 必须验证 sealed image 引用的 allocation 正是本事务持有的 allocation，然后才解除 rollback 责任。

只有 sealed 类型提供 entry：

```rust
impl SealedImage {
    pub fn entry(&self) -> TargetAddr;
    pub fn protection(&self) -> ProtectionLevel;
    pub fn seal_plan(&self) -> &SealPlan;
}
```

`MappedImage/RuntimeImage/RelocatedImage` 不提供同名方法。

### 13.4 测试与 Gate

- RX/R/RW+NX 切分；
- RELRO 覆盖 RW 子区间；
- 相邻范围合并；
- cache 同步覆盖全部 X region；
- cache/protect 任一步失败均 rollback；
- 无硬件保护时明确返回 `LogicalOnly`；
- 没有可靠 cache backend 时加载失败；
- `LogicalOnly` 不能被描述为硬件隔离。

## 14. C10：切换兼容入口并删除旧实现

建议提交：

```text
loader: route load_elf through ImageLoader
```

### 14.1 本提交要解决的问题

C01–C09 已经分别证明读取、解析、布局、分配、复制、relocation 和 seal。此时才具备替换生产入口的条件。

### 14.2 新高层 API

```rust
pub fn load_image<R, M, C, A>(
    reader: R,
    request: ArtifactRequest,
    memory: &mut M,
    cache: &mut C,
    runtime_policy: RuntimeFeaturePolicy,
    relocator: &A,
) -> LoadResult<SealedImage>
where
    R: ElfReader,
    M: ImageMemory,
    C: CodeCache,
    A: ArchRelocator + ?Sized;
```

内部只负责编排：

```rust
let admitted = loader.admit(reader, request)?;
let planned = loader.plan(admitted)?;

let mut transaction = ImageLoadTransaction::new(memory);
let reserved = loader.reserve(planned, &mut transaction)?;
let mapped = loader.copy_and_zero(reserved, &mut transaction)?;
let runtime = mapped.decode_runtime(&mut transaction, runtime_policy)?;
let relocated = runtime.relocate(&mut transaction, relocator)?;
let sealed = relocated.seal(&mut transaction, cache)?;

transaction.commit_for(&sealed)?;
Ok(sealed)
```

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
3. 构造 `MemoryMapper` 和当前架构 cache adapter；
4. 调用 `load_image()`；
5. 成功后把 sealed entry 安装到 mapper；
6. 将结构化 `LoadError` 映射为兼容静态字符串。

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
- `MemoryMapper::real_entry()` 只在 seal 成功后可用；
- ARM32 host fixture、现有 RISC-V64 `ET_DYN`、fixed `ET_EXEC` 和 QEMU 回归通过。

## 15. 最终代码布局

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

不用为了目录形式提前移动 `memory_mapper.rs`。先完成 C10 的单一路径切换，再决定是否在后续提交移动到 `compat/`，避免把文件移动噪声和语义改动混在一起。

每增加一个 Rust module，都必须同步加入 `kernel/loader/BUILD.gn` 的 `sources`，否则 Ninja 可能不能正确追踪 module 文件变化。

## 16. 类型引入时机

下表用于避免再次出现“先创建大量类型、后寻找用途”的实现方式。

| 类型 | 首次引入 | 当前用途 |
| --- | --- | --- |
| `LoadError` | C01 | 表达 read/header/admission 错误，后续按消费者扩展 stage/kind/context |
| `ElfReader/SliceElfReader` | C01 | 读取 header；后续同一接口读取 phdr 和 segment |
| `ArtifactProfile/Request` | C01 | 描述单映像 ELF ABI 与 `Dyn/Exec` header 类型要求，不表达链接模型 |
| `TargetAddr/FileRange` | C02 | 区分 ELF 虚址与文件范围 |
| `ParsedImage` | C02 | 持有不借用输入文件的 program-header 视图 |
| `TargetRange/ImageLayout` | C03 | 在 allocation 前证明完整布局有界且合法 |
| `AllocationId/ImageMemory` | C04 | 第一次申请和释放目标内存 |
| `ImageLoadTransaction` | C04 | 第一次产生需要 rollback 的副作用 |
| `TargetLocation/MappedImage` | C05 | 第一次按 allocation+offset 写目标内存 |
| `RelocationRecord/RuntimeImage` | C06 | 第一次解析运行时 relocation metadata |
| `TargetWord/WordWidth` | C07 | 第一次按目标 ELF 字宽读写 relocation value |
| `ArchRelocator/RelocatedImage` | C07 | 执行并证明 relative relocation 已完成 |
| `ArmRelocator` | C08 | 增加 ARM32 REL 隐式 addend |
| `CodeCache/SealPlan/SealedImage` | C09 | 第一次允许把映像变为可执行结果 |

以下类型不在 Phase 0 引入：

- `ImageId`：单映像事务没有身份去重需求，Phase 0.5 依赖图出现时再引入；
- `ArtifactIdentity/FileIdentity/BuildId`：Phase 0.5/1 的 catalog 和准入才消费；
- `LinkDomainId`：Phase 1 ApplicationManager/domain 才消费；
- `SymbolTable/ResolvedSymbol/ScopeSet`：Phase 0.5 symbol relocation 才消费；
- `ImageLifecycleMetadata/InitPlan/FiniPlan`：Phase 0.5/1 生命周期阶段才消费；
- `RelocationPolicy` 完整对象：Phase 0 只有硬编码的单映像 relative 白名单，Phase 0.5 出现跨映像符号 owner/scope 后再形成独立策略对象。

## 17. 测试架构

### 17.1 Host 测试替身

测试替身按生产 trait 出现顺序加入：

- C00 `ElfFixtureBuilder`：构造小型 ELF32/ELF64；
- C01 `FakeReader`：记录 read-at 并在第 N 次读取注入 short read/I/O error；
- C04 `FakeMemory`：以 `0xa5` 预填充，记录 allocate/release；
- C05 扩展 `FakeMemory`：记录 write/zero 并支持逐调用失败；
- C06 扩展 `FakeMemory`：支持有界 read；
- C07 扩展 `FakeMemory`：支持目标 word read/write；
- C09 `FakeCodeCache`：记录同步范围并支持失败。

异常输入优先使用小型人工 fixture；真实工具链 fixture 用于验证 ABI 和 relocation oracle，两者不能相互替代。

### 17.2 单元测试矩阵

| 模块 | 必测场景 |
| --- | --- |
| reader/admission | 空文件、短读、错误 class/endian/machine/type、phdr 范围溢出、资源上限 |
| parser | 无 section header、多 `PT_LOAD`、重复 dynamic/stack、unsupported phdr |
| layout | 非零 min vaddr、gap、非法 align、不同余、filesz>memsz、overlap、W+X、entry 非 X |
| allocation | OOM、错误 base/len/align、fixed 未授权、release exactly once |
| mapping | 大 BSS、预填充非零、分块 copy、gap locate、read/write/zero failure |
| metadata | REL/RELA descriptor、table 越界、错误 entry size、数量上限、unsupported tags |
| relocation | RISC-V64 RELA、ARM32 REL、隐式 addend、未知类型、symbol!=0、越界、未对齐、溢出 |
| seal | RX/R/RW+NX、RELRO、cache range、LogicalOnly、cache/protect failure |
| typestate | 只有 `SealedImage` 提供 entry |

### 17.3 集成回归

- 现有自包含 `ET_DYN` 应用加载并执行；
- fixed `ET_EXEC` 加载并执行；
- fixed range 越界在第一次写之前失败；
- RISC-V64 `ET_DYN` 应用使用新 engine；
- ARM32 `DT_REL/R_ARM_RELATIVE` host fixture 精确比较目标字节；
- 大 BSS 在非零预填充内存上仍为零；
- 最低 vaddr 非零时 entry 和 relocation value 正确；
- QEMU 执行新写入代码前经过真实架构 cache backend。

建议最终运行：

```bash
ninja -C out/qemu_riscv64.release.dsc kernel/loader:check_loader kernel/loader:blueos_loader_clippy
ninja -C out/qemu_mps2_an385.release.dsc kernel/loader:check_loader kernel/loader:blueos_loader_clippy
ninja -C out/seeed_xiao_esp32c3.release.dsc kernel/loader:check_loader kernel/loader:blueos_loader_clippy
```

ESP32-C3 的 image action 会下载官方 bootloader 和 partition table；离线环境应预置这两个输入。若 Python TLS trust store 不完整，可先通过系统下载工具取得同一官方 release 文件，再用 `build/scripts/gen_esp32_image.py build_image` 生成测试镜像。

## 18. Phase 0 最终验收清单

- [x] 所有 ELF 数值先 checked arithmetic，后转换为 host 类型；
- [x] parser 不依赖 section header；
- [x] `p_filesz <= p_memsz`、alignment、同余、overlap 和 entry 权限均校验；
- [x] 非零最低 vaddr 使用正确 load bias；
- [x] owned `ET_DYN` allocation 和 BSS 确定性清零；
- [x] `locate()` 拒绝 allocation gap 和错误权限；
- [x] relative relocation 校验 owner、范围、字宽、端序、对齐和溢出；
- [x] 未知或非白名单 relocation fail closed；
- [x] `DT_NEEDED/TEXTREL/PT_TLS/PT_INTERP` 等 Phase 0 不支持特性被拒绝；
- [x] cache backend 缺失时加载失败；
- [x] 无硬件保护时准确报告 `LogicalOnly`；
- [x] S2–S8 失败由同一个事务 rollback，不泄漏或 double-free；
- [x] 只有 `SealedImage` 公开暴露 entry；
- [x] fixed `ET_EXEC` 在所有 source/target/final-permission preflight 完成前不执行第一次写；
- [x] 旧 `load_elf()` 只剩兼容包装；
- [x] 仓库中没有第二套生产 parser、copy 或 relocation 实现；
- [x] ARM32 host fixture、RISC-V64 `ET_DYN`、RISC-V32 fixed `ET_EXEC` 和 QEMU 回归通过。

达到以上条件后，Phase 0.5 可以直接以 `RuntimeImage/RelocatedImage/SealedImage` 为基础增加 dependency graph、symbol scope 和多映像 `LinkSession`，无需再次修改单映像的地址、布局、复制、relative relocation 和 seal 语义。
