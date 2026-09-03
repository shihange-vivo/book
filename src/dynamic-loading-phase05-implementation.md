# BlueOS Dynamic Loading Phase 0.5 详细实施计划

本文是 Phase 0.5 的直接开发清单。它以 2026-09-02 的
`kernel/loader_and_linker@2725c96` 为代码基线，承接
[Phase 0 详细实现方案](./dynamic-loading-phase0-implementation.md)，细化
[总实施计划](./dynamic-loading-implementation-plan.md)中的 C11–C17。

Phase 0.5 的目标是在现有 `blueos_loader` crate 内形成可由 host backend
完整验证的多映像 `DynamicLinker`：从一个 `ET_DYN` root 出发，有界解析
`DT_NEEDED`，闭合依赖图，冻结符号 scope，完成 NOW relocation，收紧所有映像的
权限和 cache，生成生命周期计划，并通过一次不可分割的 host publication 得到
长期持有 allocation lease 的链接产物。

Phase 0.5 不接入真实 VFS、`ApplicationManager`、ThreadGroup、系统 DSO registry、
`librs/libc.so.1` 或应用线程。这些属于 Phase 1。Phase 0.5 也不执行 constructor；
它只生成经过地址和顺序校验的 `InitPlan/FiniPlan`。

## 1. 当前基线与进入条件

### 1.1 已有基础

当前 `kernel/loader` 已具备：

- 统一的 `ET_DYN/ET_EXEC` 单映像流水线；
- `ElfReader::read_exact_at()` 输入抽象；
- checked target address、load bias、逐 `PT_LOAD` region、BSS 清零；
- ARM32、RISC-V32/64 和 AArch64 relative relocation 后端；
- `AllocationLease + ImageLoadTransaction` 单 allocation 回滚；
- `PreparedImage → ReadyImageCommit` 两段式本地提交；
- `SealPlan/PreparedProtectionPlan/ProtectionBatch` 和 cache capability；
- 集中的 `LoadPolicy/PHASE0_LOAD_POLICY`；
- 兼容入口已经转入新 loader，不再维护第二套 ELF 管线。

### 1.2 允许开始但不能掩盖的 Phase 0 欠账

Phase 0.5 的纯 metadata、graph 和 scope 工作现在可以开始，但 C17 合入前仍必须关闭
以下与多映像正确性直接相关的基线问题：

1. dynamic feature 目前在 segment 写入后才判定；必须在第一次 allocation/write 前形成
   `DynamicFeatureSummary`；
2. `ElfReader` 还没有可被 VFS adapter 验证的 snapshot/version contract；
3. ARM/RISC-V `e_flags`、ABI version、Thumb entry 最小指令跨度仍需完成准入；
4. relocation result、owner/writable range 和峰值 metadata budget 仍需收紧；
5. protection granule/slot/alias 与 RISC-V SMP cache scope 仍是 Phase 0 最终门禁；
6. 按当前约定，测试代码可以在生产结构稳定后统一补齐，但 C17 不能在缺少 fault、
   malformed corpus 和 differential oracle 的情况下宣称 Phase 0.5 完成。

这意味着“开始 Phase 0.5”与“跳过 Phase 0 合入门”不是同一件事。C11–C14 中不依赖
目标内存的纯逻辑可以先行；C15–C17 必须建立在 allocation-explicit memory、稳定
snapshot 和最终 protection/cache contract 上。

### 1.3 当前最先阻塞多映像的接口

当前 `ImageMemory` 仍通过 `allocation()` 返回 backend 唯一活动 allocation，且
`read/write/zero/image_span/protect` 只接收 allocation 内 offset。这适合
`MemoryMapper` 的单映像兼容路径，但不能同时表达 main、private DSO 和 system DSO。

Phase 0.5 的第一个代码变更必须让每次访问显式绑定 allocation：

```rust
pub trait ImageMemory {
    fn allocate_image(&mut self, request: AllocationRequest)
        -> LoadResult<AllocationLease>;

    fn read(
        &self,
        allocation: &ImageAllocation,
        offset: AllocationOffset,
        dst: &mut [u8],
    ) -> LoadResult<()>;

    fn write(
        &mut self,
        allocation: &ImageAllocation,
        offset: AllocationOffset,
        data: &[u8],
    ) -> LoadResult<()>;

    fn zero(
        &mut self,
        allocation: &ImageAllocation,
        offset: AllocationOffset,
        len: u64,
    ) -> LoadResult<()>;
}
```

`image_span/protect` 同样必须携带 allocation。`MemoryMapper` 可以继续只允许一个活动
allocation，但要校验传入 descriptor 的 `AllocationId/base/len/ownership` 与其内部
状态完全一致；Phase 0.5 host backend 则使用 `AllocationId → backing` 表支持多个活动
allocation。不得通过“切换 current allocation”全局状态来模拟多映像访问。

## 2. Phase 0.5 范围

### 2.1 本阶段交付

- 一个 root 和有界多个 DSO 的 `ET_DYN` metadata；
- `DT_NEEDED/DT_SONAME` 与可信 resolver contract；
- 文件身份和 SONAME 双重去重；
- 有界 BFS 依赖闭包、稳定 SCC 和生命周期拓扑；
- dynstr/dynsym、GNU hash、SysV hash；
- global/weak/local、default/hidden/protected 的首版语义；
- application scope 与 system scope 的明确区分；
- ARM32 `DT_REL` NOW relocation：
  `R_ARM_RELATIVE/R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT`；
- session-wide relocation preflight，全部成功后才开始写；
- 每个映像的 RELRO、protection 和 cache 收口；
- `InitPlan/FiniPlan/LinkMapEntry/LinkContext`；
- 多资源 rollback 和 host 原子 publication；
- 与固定 Relink revision 的 host differential oracle；
- 原有 `prepare_image/load_elf` 单映像 API 和跨架构 QEMU 回归不退化。

### 2.2 明确不交付

- `PT_INTERP` 和用户态 ld.so handoff；
- `PT_TLS`、TLSDESC 和 native ELF TLS；
- lazy binding；
- IFUNC/IRELATIVE、COPY、TEXTREL；
- symbol version；
- `RPATH/RUNPATH` 或环境变量搜索；
- `dlopen/dlsym/dlclose`；
- constructor/finalizer 的实际调用；
- kernel registry、并发首次装载和 system DSO quiescence；
- `libc.so.1` 真实构建与 ARM32 应用启动；
- AArch64/RISC-V 的符号型 relocation；
- veneer 动态生成和完整 CFI。

### 2.3 工件角色

不能继续用一套“所有映像都必须有可执行 entry”的规则处理 root 和 DSO：

```rust
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub enum ArtifactRole {
    ExecutableRoot,
    SharedObject,
}
```

- `ExecutableRoot` 必须有 canonical entry，且最小指令跨度完整位于自己的 X region；
- `SharedObject` 不以 ELF entry 作为调用入口，允许 `e_entry == 0`；若非零，仅做格式和
  范围检查，不将它发布为应用入口；
- root 可无 `DT_SONAME`，DSO 必须提供一个有界、NUL 终止的 SONAME；
- Phase 0.5 的 root 和 DSO 都必须是 `ET_DYN`；固定 `ET_EXEC` 只走 Phase 0
  单映像兼容入口，不能进入 dependency session。

## 3. 必须保持的不变量

1. parser 只产生结构化事实，`LoadPolicy` 决定当前 profile 是否支持；
2. `PHASE0_LOAD_POLICY` 永远不能因为调用方输入而放宽；Phase 0.5 使用独立的
   `PHASE05_LOAD_POLICY`；
3. 开启一个 policy switch 时必须同时存在 metadata consumer，不能“接受后忽略”；
4. root 与每个依赖在 allocation 前完成所有可由 snapshot/header/phdr/dynamic summary
   判定的错误；
5. target address 永远是整数地址加 allocation/region 归属，不等同于 Rust 指针；
6. 每次内存访问显式绑定 `AllocationId`，每次 relocation 写入显式绑定 owner image；
7. dependency discovery order、symbol lookup order 和 init/fini order 是三个独立产物；
8. scope 一旦用于 relocation 就不可变，不在后续装载时反向重绑定已写 GOT；
9. session-wide relocation plan 完整预检后才执行第一笔 relocation write；
10. S0–S8 任意失败都通过同一个 session 回滚，allocation lease 恰好消费一次；
11. partial protection/cache 失败仍属于未发布事务，owned allocation 全部回收；
12. S9 的所有 fallible prepare 在可见状态改变前完成，最终 commit 不返回 `Result`；
13. committed product 或 host registry 是 lease 的唯一长期 owner；
14. loader 不持锁、不执行 constructor、不进入应用 entry；
15. `blueos_loader` 生产依赖中不得出现 `//librs`、VFS、ThreadGroup 或
    `ApplicationManager`。

## 4. 策略模型

### 4.1 沿用统一 `LoadPolicy`

不为了 Phase 0.5 再增加一套散落的 dynamic feature 判断，也不急于引入仅有一个实现的
policy trait。现有 `LoadPolicy` 继续作为统一能力控制器，形成两个 crate 内 preset：

```rust
pub(crate) const PHASE0_LOAD_POLICY: LoadPolicy = LoadPolicy::phase0();
pub(crate) const PHASE05_LOAD_POLICY: LoadPolicy = LoadPolicy::phase05();
```

公开 `prepare_image/load_elf` 仍硬绑定 `PHASE0_LOAD_POLICY`。只有 crate 内的
`DynamicLinker` 可以调用 `ImageLoader::inspect_with_policy()` 和
`decode_runtime_with_policy()` 并传入 `PHASE05_LOAD_POLICY`。不能把一个可任意放宽的
policy 放进公开 `LoadRequest` 后直接交给单映像提交入口。

### 4.2 Phase 0.5 开关矩阵

| ELF 特性 | Phase 0 | Phase 0.5 | 说明 |
| --- | --- | --- | --- |
| `DT_REL/DT_RELA` relative | 允许 | 允许 | 复用现有 relative engine |
| `DT_NEEDED` | 拒绝 | 允许且消费 | 进入 BFS discovery queue |
| `DT_SONAME` | 可识别 | DSO 必需 | 参与双索引去重 |
| dynstr/dynsym/hash | 可识别 | 允许且消费 | 生成 `SymbolTable` |
| `DT_JMPREL/DT_PLTREL` | 拒绝 | 仅 NOW | lazy 仍拒绝 |
| init/fini/array | 拒绝 | 允许且消费 | 只生成 plan |
| `DT_FLAGS` | `DF_BIND_NOW` | 同左 | 其他 bit fail-closed |
| `DT_FLAGS_1` | `DF_1_NOW/DF_1_PIE` | 同左 | 不启用 group/interpose 等语义 |
| `RPATH/RUNPATH` | 拒绝 | 拒绝 | resolver 不读取 ELF 搜索路径 |
| `PT_INTERP/PT_TLS` | 拒绝 | 拒绝 | Phase 1 仍不需要 |
| TEXTREL/W+X/可执行栈 | 拒绝 | 拒绝 | 不作为可放宽阶段开关 |
| symbol version | 拒绝 | 拒绝 | parser 可识别，profile 拒绝 |
| RELR | 拒绝 | 暂时拒绝 | Phase 2 按架构启用 |
| X-only/W-only/NONE segment | 拒绝 | 拒绝 | 只接受 `R/RX/RW` |
| unknown phdr/tag/relocation | 拒绝 | 拒绝 | fail-closed |

### 4.3 两阶段 feature 判定

`LoadPolicy` 的调用分为：

1. S1 `validate_program_features()`：根据 phdr summary 判定 interpreter、TLS、stack、
   segment permissions、unknown phdr；
2. S1 `validate_dynamic_features()`：直接从 snapshot 的 file-backed `PT_DYNAMIC`
   生成 `DynamicFeatureSummary`，判定 needed、PLT、lifecycle、flags 和未知 tag。

S4 才解码完整字符串、符号、hash、relocation 和生命周期表。S4 发现结构损坏仍然失败，
但不能到此时才发现“Phase 0.5 根本不支持此功能”。

## 5. Artifact 与 resolver contract

### 5.1 身份类型

```rust
#[derive(Clone, Copy, Debug, Eq, Ord, PartialEq, PartialOrd)]
pub struct ImageId(u32);

#[derive(Clone, Copy, Debug, Eq, Ord, PartialEq, PartialOrd)]
pub struct LinkDomainId(u32);

#[derive(Clone, Debug, Eq, Ord, PartialEq, PartialOrd)]
pub struct FileIdentity {
    // backend 定义但稳定、可比较的 snapshot identity
    bytes: Box<[u8]>,
}

pub struct ArtifactIdentity {
    file: FileIdentity,
    generation: u64,
    build_id: Option<BuildId>,
}

pub struct DependencyRequest {
    requester: ArtifactIdentity,
    needed: DependencyName,
    domain: LinkDomainId,
}
```

`DependencyName` 必须是已经验证长度、UTF-8/字节策略和 NUL 终止的 owned 值。第一版可
按 ELF 字节名比较，但 resolver 的 catalog key 必须定义唯一规范化方式，不能在不同
层分别做大小写或路径归一化。

### 5.2 Resolver trait

```rust
pub trait ArtifactResolver {
    type Reader: ElfReader;

    fn resolve(
        &mut self,
        request: &DependencyRequest,
    ) -> LoadResult<ResolvedArtifact<Self::Reader>>;
}

pub struct ResolvedArtifact<R> {
    identity: ArtifactIdentity,
    ownership: ImageOwnership,
    reader: R,
}

pub enum ImageOwnership {
    SessionPrivate,
    SystemCandidate,
}
```

Phase 0.5 的 fake resolver 只查固定内存 catalog。trait 不接收环境变量、当前目录、裸
路径或 VFS 对象。Phase 1 的 `ApplicationArtifactResolver` 才把固定 system catalog
和签名应用包适配到这个接口。

`resolve()` 返回的 reader 与 `ArtifactIdentity` 必须属于同一 snapshot。失败不能留下
registry loading entry；并发 permit/Ready system lease 在 Phase 1 registry 实现时再
加入，Phase 0.5 不建立空占位类型。

### 5.3 去重规则

依赖加入 graph 前按以下顺序判断：

1. 同一 `FileIdentity + generation`：复用已有 `ImageId`；
2. 同一 SONAME 且同一 identity：复用，并增加 edge；
3. 同一 SONAME 但 identity 不同：`IdentityConflict`；
4. 同一 identity 声明不同非空 SONAME：`BadElf`；
5. root 与依赖形成回边：保留 edge，不重复 load；
6. resolver 返回的 artifact role/profile/ABI 不匹配：在 allocation 前拒绝。

不得仅用 SONAME 或路径判断“同一个文件”。

## 6. 多映像内存与所有权

### 6.1 Allocation-explicit access

删除 `ImageMemory::allocation()` 作为通用接口。`ImageLoadTransaction` 从自己的 lease
获取 descriptor，并提供 crate-private access wrapper：

```rust
impl<M: ImageMemory> ImageLoadTransaction<M> {
    fn read(&self, offset: AllocationOffset, dst: &mut [u8]) -> LoadResult<()>;
    fn write(&mut self, offset: AllocationOffset, src: &[u8]) -> LoadResult<()>;
    fn zero(&mut self, offset: AllocationOffset, len: u64) -> LoadResult<()>;
}
```

wrapper 在调用 backend 时总是传入同一 descriptor，并统一维护
`MutationProgress`。后续 image stage 不直接绕过 transaction 调用无 owner 的
`memory.write()`。

### 6.2 从单映像 transaction 向 session 移交

`DynamicLinker` 装入一个映像时只对 session memory 做短暂 reborrow。单映像阶段完成
S4 后，必须结束该 reborrow，再装入下一个依赖：

```rust
pub(crate) struct SessionImage<S> {
    image_id: ImageId,
    artifact: ArtifactIdentity,
    allocation: ImageAllocation,
    rollback_slot: RollbackSlot,
    state: S,
}
```

lease 不能同时出现在 `SessionImage` 和 rollback log 中。唯一 ownership 放在：

```rust
struct RollbackEntry {
    lease: AllocationLease,
    progress: MutationProgress,
}

struct RollbackLog {
    entries: Vec<Option<RollbackEntry>>,
}
```

`SessionImage` 只保存不可伪造的 slot 与可复制 descriptor。吸收顺序固定为：

1. transaction 仍持 lease；
2. rollback log `try_reserve()`；失败则 transaction Drop 自动 abort；
3. 创建空 slot；
4. 从 transaction 取走 lease/progress，并以不再分配的方式写入 slot；
5. transaction 被消费，结束 `&mut M` reborrow；
6. session 保存 `SessionImage<RuntimeState>`。

不得用 `mem::forget()`、复制 `AllocationLease` 或让 session image 与 transaction 同时认为
自己有 abort 权限。

### 6.3 Session 回滚

Phase 0.5 rollback log 只管理本次 session 新分配的 image lease。reader、字符串、graph
和 scope 由普通 Rust ownership 回收。Phase 1 出现 registry permit/system lease 后再扩展
资源枚举，不提前加入未消费字段。

session Drop 按 allocation 创建的逆序调用：

```rust
memory.abort_image(lease, progress)
```

回滚必须无分配、无失败、无 panic。若某个 backend 无法满足，不能实现
`DynamicLinkMemory`。

## 7. Runtime metadata（S4）

### 7.1 输出结构

```rust
pub struct RuntimeImageMetadata {
    dynamic: RuntimeDynamicInfo,
    needed: Box<[DependencyName]>,
    soname: Option<DependencyName>,
    symbols: SymbolTable,
    relocations: RelocationTables,
    lifecycle: ImageLifecycleMetadata,
    program_headers: ProgramHeaderRuntimeInfo,
}

pub struct RuntimeImageState {
    layout: ImageLayout,
    regions: Box<[LoadedRegion]>,
    metadata: RuntimeImageMetadata,
    load_bias: TargetAddress,
}
```

所有 target vaddr 在 metadata 内要么保留明确命名的 `ElfVaddr`，要么已经通过同一个
layout locator 归一化为 `TargetLocation { allocation, offset, runtime }`。不得用裸
`u64` 同时表示 file offset、ELF vaddr 和 runtime address。

### 7.2 Dynamic table

- `PT_DYNAMIC` 必须完整落在一个 file-backed readable `PT_LOAD`；
- entry size 由 ELF class 唯一决定；
- 必须在 `max_dynamic_entries` 内遇到 `DT_NULL`；
- single-value tag 重复时报 `BadElf`；
- `DT_NEEDED` 保留原始出现顺序；
- `DT_STRTAB + DT_STRSZ`、`DT_SYMTAB + DT_SYMENT` 必须成组出现；
- `DT_REL/RELA/JMPREL` 地址、长度、entry size 必须完整且互相一致；
- `DT_PLTREL` 只允许 `DT_REL` 或明确启用的 `DT_RELA`；ARM32 首发只允许 REL；
- `DT_FLAGS/DT_FLAGS_1` 做 bit mask，而不是只识别 tag；
- 所有 unknown tag fail-closed。

修复当前 `locate_file_backed_dynamic()` 对计算出的 `file_end` 未实际使用的问题，确保
dynamic bytes 与对应 `PT_LOAD` 的 file range 完整一致。

### 7.3 Dynstr

- 整个 dynstr 落在 readable region；
- `DT_STRSZ` 受 `max_string_table_bytes` 限制；
- 每个 needed/SONAME/symbol name offset 小于 `DT_STRSZ`；
- 从 offset 到第一个 NUL 的扫描受 `max_symbol_name_len/max_dependency_name_len` 限制；
- 不为每次 lookup 重复分配字符串；可以保存 validated range，在需要形成 graph key 时
  才复制 bounded name；
- 错误 UTF-8 不得 panic。若第一版使用 ELF raw bytes，错误展示进行有界转义。

### 7.4 Dynsym 数量推导

运行时不能依赖 section header。symbol count 按以下来源推导并交叉检查：

1. 有 SysV `DT_HASH` 时使用 `nchain`，同时验证 bucket/chain 总范围；
2. 有 GNU hash 时有界扫描 header、bloom、bucket 和 chain，推导最大 symbol index；
3. 两者同时存在时，推导结果必须兼容；
4. relocation 引用的最大 symbol index 必须小于已证明的 dynsym count；
5. 无法从 hash/table range 证明上界时拒绝，不通过读取到 segment 尾部猜测。

`DT_SYMENT` 必须等于目标 ELF ABI 的 symbol entry size。全部 symbol entry 在解析前先做
总 metadata budget 检查。

### 7.5 SymbolTable

```rust
pub struct SymbolRef {
    index: u32,
    name: SymbolNameRef,
    value: TargetAddress,
    size: u64,
    binding: SymbolBinding,
    visibility: SymbolVisibility,
    symbol_type: SymbolType,
    definition: SymbolDefinition,
}
```

- `STB_LOCAL` 只在 owner 内按 index 使用，不进入外部 scope；
- `STB_GLOBAL/STB_WEAK` 进入 export iterator；
- `STV_HIDDEN/INTERNAL` 不对外导出；
- `STV_PROTECTED` 可对外导出，但 owner 内引用固定绑定 owner 定义；
- `STT_FUNC/STT_OBJECT/STT_NOTYPE` 首版允许；
- TLS、GNU IFUNC、section/common 等未支持类型明确拒绝；
- defined symbol 的 `st_value/st_size` 必须落入 owner 合法 region；
- function symbol canonical target 必须位于 X region，ARM Thumb 地址单独保留/校验 bit 0；
- object symbol 必须位于可读 data region；
- undefined symbol 不把 `st_value` 当成地址使用。

GNU/SysV hash lookup 必须与有界线性扫描 oracle 得到同一 symbol index；hash 损坏不能
回退成无界线性扫描。

### 7.6 Lifecycle metadata

S4 只保存 `DT_INIT/DT_FINI` target 和 init/fini array 的范围，不立即把 array 内容固定
成函数地址，因为 array entry 本身可能在 S7 被 relocation 修改。S7 后构建最终 plan 时
重新通过 owner allocation 读取目标字。

## 8. 依赖图（S5）

### 8.1 数据结构

```rust
pub struct DependencyNode {
    id: ImageId,
    artifact: ArtifactIdentity,
    soname: Option<DependencyName>,
    ownership: ImageOwnership,
    discovery_index: u32,
    depth: u16,
}

pub struct DependencyEdge {
    requester: ImageId,
    provider: ImageId,
    needed_index: u16,
}

pub struct DependencyGraph {
    nodes: Vec<DependencyNode>,
    edges: Vec<DependencyEdge>,
    identity_index: BTreeMap<ArtifactIdentity, ImageId>,
    soname_index: BTreeMap<DependencyName, ImageId>,
}
```

索引和 queue 的每次增长都必须使用 `try_reserve` 并计入 runtime metadata budget。

### 8.2 BFS 算法

1. root 固定为 `ImageId(0)`；
2. 按 root dynamic table 中 `DT_NEEDED` 出现顺序入队；
3. 出队时先查 identity/SONAME index，再调用 resolver；
4. 新 artifact 完成 S0/S1 dynamic summary 后才 allocate/map/decode；
5. 新节点的 needed 按原顺序追加到队尾；
6. 已有节点只增加 edge，不重复 load；
7. queue 为空后 graph 才称为 closed；
8. closed graph 才能进入 scope freeze。

预算至少包括：

- `max_images`；
- `max_dependency_edges`；
- `max_dependency_depth`；
- `max_total_image_bytes`；
- `max_total_runtime_metadata_bytes`；
- `max_dependency_name_len`；
- 单 image 和 session 总 relocation 数；
- resolver 调用次数。

循环依赖不会导致无限 BFS。闭图后运行有界 SCC 算法；SCC 内顺序使用 discovery index，
不能依赖 `BTreeMap` key 或 allocator 地址。

## 9. Symbol scope（S6）

### 9.1 ScopeSet

```rust
pub struct SymbolScope {
    ordered_images: Box<[ImageId]>,
}

pub struct ScopeSet {
    application: SymbolScope,
    system: SymbolScope,
    per_image: Box<[ImageScope]>,
}
```

- application scope：root → session-private BFS → system candidate BFS；
- system image 的自身 relocation scope 不能看到 application-private image；
- per-image scope 记录 protected/self-first 规则；
- scope freeze 后不再添加节点或改变顺序。

Phase 0.5 fake resolver 可以标记 `SystemCandidate` 以验证隔离语义，但不建立全局 registry
或跨 session 共享 allocation；真实 system instance/lease 属于 Phase 1。

### 9.2 Lookup 规则

1. relocation 指定 local symbol：只查 owner index；
2. protected owner-local reference：固定 owner；
3. global/weak undefined：按冻结 scope 顺序查找；
4. hidden/internal 定义不参与外部 lookup；
5. 首个合法 strong definition 胜出；BlueOS 首版若同一 scope 中出现多个不允许的 strong
   export，可按 artifact policy 报 `AmbiguousStrongDefinition`；
6. 没有 strong 时使用首个 weak；
7. undefined strong 失败；
8. undefined weak 可形成值 0，但 `JUMP_SLOT` 等控制流 relocation 是否允许 0 由
   `RelocationPolicy` 单独决定，默认拒绝。

`ResolvedSymbol` 至少包含 owner、runtime address、canonical function address、size、binding
和 region kind。不能只返回裸地址而丢失 owner，后续 relocation policy 需要验证 `S`。

## 10. LinkSession typestate

### 10.1 状态

只在合法 API 集合真正改变的位置分型：

```rust
pub struct BuildingState {
    images: Vec<SessionImage<RuntimeImageState>>,
    discovery: DiscoveryQueue,
}

pub struct ScopedState {
    images: Vec<SessionImage<RuntimeImageState>>,
    scopes: ScopeSet,
}

pub struct RelocatedState {
    images: Vec<SessionImage<RelocatedImageState>>,
    scopes: ScopeSet,
}

pub struct SealedState {
    images: Vec<SessionImage<SealedImageState>>,
    scopes: ScopeSet,
}

pub struct LinkSession<'a, M: DynamicLinkMemory + ?Sized, S> {
    memory: &'a mut M,
    graph: DependencyGraph,
    rollback: RollbackLog,
    limits: SessionLimits,
    metrics: LoadMetrics,
    state: S,
}
```

公开别名为 `BuildingSession/ScopedSession/RelocatedSession/SealedSession`。字段和构造函数
为 private；转换消费 `self`：

```text
DynamicLinker::begin(root)                 -> BuildingSession
BuildingSession::close_dependencies()      -> BuildingSession（closed）
BuildingSession::freeze_scopes(self)       -> ScopedSession
ScopedSession::relocate(self)              -> RelocatedSession
RelocatedSession::seal(self)               -> SealedSession
SealedSession::prepare_product(self)       -> PreparedLinkProduct
PreparedLinkProduct::prepare_commit(self)  -> ReadyLinkCommit
ReadyLinkCommit::commit(self)              -> LinkProduct
```

`close_dependencies()` 是否完成可用内部 enum 表达，不为每次 BFS iteration 增加泛型状态。

### 10.2 DynamicLinker

```rust
pub struct DynamicLinker<A> {
    arch: A,
    policy: LoadPolicy,
}
```

Phase 0.5 先不为单一 `LoadPolicy` 再加 trait。`ArchRelocator` 保留 trait，因为它确实有
多个平台实现。resolver/memory/publisher 使用 trait，因为它们是 loader 与外部系统的
能力边界。

入口建议为：

```rust
pub fn link<R, Resolver, Memory, Cache, Publisher>(
    &self,
    root: ResolvedArtifact<R>,
    domain: LinkDomainId,
    limits: SessionLimits,
    resolver: &mut Resolver,
    memory: &mut Memory,
    cache: &mut Cache,
    publisher: &mut Publisher,
) -> LoadResult<LinkProduct>
```

实现内部仍按 typestate 拆开，不写成一个包含大量布尔状态的长函数。

## 11. Session-wide relocation（S7）

### 11.1 RelocationPolicy

到 Phase 0.5 才引入独立 `RelocationPolicy`，因为此时首次需要同时判断写入位置 `P`、
symbol 来源 `S` 和控制流目标：

```rust
pub struct RelocationPolicy {
    allowed_types: RelocationTypeSet,
    allow_undefined_weak_data: bool,
    allow_undefined_weak_control_flow: bool,
    require_control_flow_target_x: bool,
    require_target_owner_writable: bool,
}
```

它由 `PHASE05_LOAD_POLICY` 和架构 profile 构造，不允许调用方传入任意类型集合。

### 11.2 ARM32 首发语义

| relocation | 计算 | 约束 |
| --- | --- | --- |
| `R_ARM_RELATIVE` | `B + A` | symbol index 必须为 0；结果属于 owner allocation |
| `R_ARM_ABS32` | `S + A` | `S` 来自冻结 scope；`P` 属于 owner RW/RELRO |
| `R_ARM_GLOB_DAT` | ABI 定义的 `S` 值 | symbol 必须可解析，data/function kind 一致 |
| `R_ARM_JUMP_SLOT` | ABI 定义的函数 `S` 值 | canonical target 位于 provider X region；Thumb bit 正确 |

具体 addend signedness、目标字截断和 Thumb function value 必须以 ARM ELF psABI 为准并在
代码中集中实现，不能在通用 engine 中散落 `as u32`。

### 11.3 全量预检

对所有 image 构造 `RelocationOperation`：

```rust
pub struct RelocationOperation {
    owner: ImageId,
    target: TargetLocation,
    width: WordWidth,
    value: u64,
    source: RelocationSource,
    record_index: u32,
}
```

预检顺序：

1. relocation table 自身位于 owner readable region；
2. 类型在架构/profile 白名单；
3. `P` 的完整目标字属于 owner allocation 和允许写入的 segment；
4. 对齐符合目标字和架构要求；
5. REL implicit addend 在任何 relocation write 前读取并固定；
6. symbol index、binding、visibility 和 owner scope 合法；
7. `S/A/B/P` 算术 checked，结果可编码且符合 result range policy；
8. control-flow target canonical address 位于 provider X region；
9. 按 `(owner, target_offset)` 排序，拒绝 duplicate/overlap target；
10. operation 总数和字节数在 session budget 内。

所有 image 的 operation 都成功后才开始 apply。apply 顺序固定为：

1. relative；
2. data/global；
3. PLT/JUMP_SLOT。

任一 backend write 失败，整个 session 保持未发布并通过 rollback log 回收全部 image。

## 12. Seal、cache 与 lifecycle（S8）

### 12.1 批量 seal

1. 为每个 image 生成逻辑 `SealPlan`；
2. 对所有 plan 完成 granule、slot、alias 和 capability preflight；
3. 固定全部 executable range 形成 cache token；
4. 执行 relocation 后的 D-cache clean/I-cache invalidate；
5. 应用 protection batch；
6. 记录每个 range 的实际权限和 `ProtectionLevel`；
7. 任何 partial apply 失败都不发布，所有 owned allocation abort。

Phase 0.5 `DynamicLinker` 只接受 allocated `ET_DYN`，不处理 borrowed fixed image，因此
session rollback 不新增 fixed poison 的组合；Phase 0 单映像 fixed 语义继续回归。

### 12.2 InitPlan/FiniPlan

最终函数地址在 relocation 完成后读取和校验：

- root `DT_PREINIT_ARRAY` 位于整个 DSO init 之前；DSO 的 preinit array 拒绝；
- SCC DAG 按 dependency-first 排序；
- SCC 内按 BFS discovery index 稳定排序；
- 单 image init：`DT_INIT`，再 `DT_INIT_ARRAY` 正序；
- fini 为 init image 顺序逆序；单 image 内 `DT_FINI_ARRAY` 逆序，再 `DT_FINI`；
- 0 或 ABI 定义的 sentinel entry 按明确规则跳过；
- 每个非空 function canonical address 必须位于其 owner X region；
- ARM function value 保留 Thumb bit，校验时使用 canonical address；
- plan 保存 owner lease 间接归属，不能只有裸函数指针数组。

Phase 0.5 只生成 plan。测试调用 fake observer，不直接把 target address 转成 host function
pointer。

## 13. 原子 publication（S9）

### 13.1 两段式 batch commit

沿用 Phase 0 的“fallible prepare + infallible commit”，但对象扩展为整批：

```rust
pub trait LinkPublisher {
    type PreparedBatch;
    type Receipt;

    fn prepare_batch(
        &mut self,
        manifest: &PreparedLinkManifest,
    ) -> LoadResult<Self::PreparedBatch>;

    unsafe fn commit_batch(
        &mut self,
        prepared: Self::PreparedBatch,
        product: CommittingLinkProduct,
    ) -> Self::Receipt;
}
```

`prepare_batch()` 完成容量、identity、generation、entry、link-map slot 和所有可能失败的
校验，但不改变可见 snapshot。`commit_batch()` 只能 move/swap 已准备数据，不分配、不
校验、不失败、不 panic。

### 13.2 输出对象

```rust
pub struct LinkContext {
    graph: DependencyGraph,
    scopes: ScopeSet,
    images: Box<[CommittedImage]>,
}

pub struct LinkProduct<Receipt> {
    context: LinkContext,
    entry: TargetAddress,
    init_plan: InitPlan,
    fini_plan: FiniPlan,
    link_map: Box<[LinkMapEntry]>,
    metrics: LoadMetrics,
    publication: Receipt,
}
```

`CommittedImage` 或 publisher receipt 必须长期持有 allocation lease。`LinkProduct::Drop`
通过 host backend 释放 committed images；Phase 1 则把它们移动到 ThreadGroup/system
registry owner。不能在 commit 时丢弃 `SealedState`，也不能只返回 entry 和裸地址。

host publisher 使用 immutable generation snapshot 验证：commit 前读者看到旧 snapshot，
commit 后一次看到全部 image/link-map/context，不存在部分条目。

constructor 位于 S10，commit 后才可能执行；Phase 0.5 不实现该副作用边界。

## 14. 错误、预算与可观测性

### 14.1 LoadStage 扩展

建议保留现有单映像 stage，并增加链接阶段：

```rust
pub enum LoadStage {
    // existing image stages ...
    Discover,
    Scope,
    LinkRelocate,
    LinkSeal,
    Publish,
}
```

底层 range/memory/parser 错误保持 stage-neutral，由 session 调用点附加 stage。resolver
错误保留 dependency requester/name 上下文；symbol 错误保留 requester image、symbol
index/name 和 relocation record。

### 14.2 SessionLimits

`LoadLimits` 保留单 image 限制，新增 session 总量：

```rust
pub struct SessionLimits {
    per_image: LoadLimits,
    max_images: u32,
    max_dependency_edges: u32,
    max_dependency_depth: u16,
    max_total_image_bytes: u64,
    max_total_runtime_metadata_bytes: u64,
    max_total_relocations: u64,
    max_symbol_lookups: u64,
    max_symbol_name_len: u32,
    max_dependency_name_len: u32,
}
```

每个 `Vec/BTreeMap/String/Box` 的增长都必须先计入 budget，再 `try_reserve`。不得用
`LoadLimits::DEFAULT` 的巨大值作为产品板级配置；Phase 1 由 board/application profile
传入实际限制。

### 14.3 Metrics

Phase 0.5 至少记录：

- resolver calls、images、edges、最大 BFS depth；
- file bytes read、mapped bytes、BSS/gap zero bytes；
- dynstr/dynsym/hash/relocation metadata bytes；
- symbol lookup 次数与 hash probe 次数；
- relocation operation 数；
- protection ranges、cache ranges；
- 各 stage 耗时由可选 observer 在 host 记录，核心不直接依赖时钟。

observer 只能接收事件副本，不能获得 session、lease 或可写 state 引用。

## 15. 源码布局

保持一个 crate 和一套 parser/relocation 核心。建议增量形成：

```text
kernel/loader/src/
├── image/                     # 现有单映像 S0–S4/S7/S8
├── dynamic_linker/
│   ├── mod.rs                 # DynamicLinker 入口
│   ├── artifact.rs            # identity/request/resolver contract
│   ├── metadata.rs            # RuntimeImageMetadata 聚合
│   ├── symbol.rs              # dynsym/hash/lookup
│   ├── graph.rs               # BFS、双索引、SCC
│   ├── scope.rs               # ScopeSet/ResolvedSymbol
│   ├── session.rs             # typestate、absorb、rollback
│   ├── relocate.rs            # session-wide operation plan
│   ├── lifecycle.rs           # init/fini plan
│   └── publish.rs             # batch prepare/commit
├── relocation/               # 现有通用/架构代码，按能力扩展
├── identity.rs               # profile/limits/LoadPolicy，暂不为搬文件制造噪声
├── memory.rs                 # allocation-explicit access/session handoff
└── lib.rs                    # 稳定公开入口与兼容包装
```

当单文件超过清晰边界再拆分；不要在 C11 先做无语义的批量移动。`memory_mapper.rs` 继续是
兼容 adapter，不作为 Phase 0.5 多映像 backend。

## 16. 逐提交实施顺序

总计划的 C11–C17 保持不变；每一项拆成同仓库可独立审查的 a/b 子提交。

### 16.1 C11-a：让 memory access 显式绑定 allocation

**修改**

- `ImageMemory/ImageProtectionMemory` 的 read/write/zero/span/protect 接收 allocation；
- 删除通用 `allocation()` current-slot 假设；
- `ImageLoadTransaction` 提供 owner-bound access wrapper；
- `MemoryMapper` 校验 descriptor identity，保持单映像兼容；
- 增加支持多个 active allocation 的 host memory backend contract；
- 定义 transaction → rollback slot 的 crate-private 无重复所有权移交。

**门禁**

- 原有单映像 ARM/RISC-V/AArch64 测试和 QEMU 不变；
- wrong allocation descriptor 确定失败且不访问 backing；
- 两个 allocation 交错 read/write 不串扰；
- absorption 前任何失败仍由 transaction abort，吸收后只由 session abort。

### 16.2 C11-b：增加 Phase 0.5 policy 与 resolver contract

**修改**

- `ArtifactRole/ArtifactIdentity/FileIdentity/DependencyRequest/ResolvedArtifact`；
- `ArtifactResolver`；
- `PHASE05_LOAD_POLICY`；
- `DynamicFeatureSummary` 和 S1 两阶段 policy；
- public Phase 0 entry 仍固定 Phase 0 policy；
- root/DSO entry 与 SONAME 规则；
- `SessionLimits` 基础。

**门禁**

- `DT_NEEDED` 在 Phase 0 仍拒绝，在 Phase 0.5 被记录而非忽略；
- interpreter/TLS/RPATH/TEXTREL/version/lazy/unknown feature 拒绝矩阵；
- policy 失败发生在 allocation/write 次数为零时；
- resolver 无 VFS、环境变量、当前目录和 `librs` 依赖。

### 16.3 C12：有界闭合 dependency graph

**修改**

- `ImageId/DependencyNode/Edge/Graph/DiscoveryQueue`；
- identity 和 SONAME 双索引；
- 稳定 BFS、深度/数量/总字节配额；
- SCC 和 dependency-first DAG；
- 逐个依赖走同一 `ImageLoader` S0–S4 并吸收到 session。

**门禁**

- 线性、菱形、重复 needed、同文件别名、SONAME 冲突、循环；
- graph 顺序与 resolver map/container 顺序无关；
- 每个 resolve/allocation/map/decode 故障后所有 lease 归零；
- root 和依赖均复用同一 parser/layout，不出现第二套 loader。

### 16.4 C13：补全 runtime dynamic symbols/hash/lifecycle metadata

**修改**

- typed `RuntimeDynamicInfo`、dynstr、dynsym；
- SysV/GNU hash validation/lookup；
- relocation table 和 lifecycle range；
- symbol binding/visibility/type/definition；
- metadata 总量预算。

**门禁**

- hash lookup 与有界线性 oracle 一致；
- bucket/chain/bloom/offset/终止符损坏不越界；
- 不依赖 section header 推导 dynsym；
- invalid symbol region/type/visibility 确定失败；
- init array 在 relocation 前只保存范围，不提前固化函数值。

### 16.5 C14：冻结 application/system symbol scopes

**修改**

- `ScopeSet/SymbolScope/ImageScope/ResolvedSymbol`；
- local/global/weak、hidden/protected；
- application/system 隔离；
- undefined strong/weak 和 duplicate strong 策略；
- lookup budget 与诊断上下文。

**门禁**

- scope 顺序与 BFS fixture 一致；
- system image 无法绑定 application-private symbol；
- hidden 不泄漏、protected owner-local 不被 interpose；
- unresolved strong 失败；undefined weak data/control-flow 按 relocation policy 区分；
- freeze 后 graph/scope 不再可变。

### 16.6 C15：增加 staged LinkSession 和 rollback

**修改**

- `Building/Scoped/Relocated/SealedSession`；
- rollback slot 唯一持有 lease/progress；
- consuming transitions；
- session Drop 逆序回滚；
- metrics 和 stage error rebinding；
- multi-image host memory backend。

**门禁**

- allocation、absorb、graph、scope、relocation-plan、write、cache、protect 每点故障；
- active session Drop 后 allocation 计数归零；
- lease 不 Clone/Copy，不存在 image/log 双 owner；
- transition 失败后调用方拿不到部分新状态；
- session rollback 不分配、不返回错误。

### 16.7 C16：完成 ARM32 NOW relocation engine

**修改**

- `RelocationPolicy`；
- ARM32 四类 relocation；
- `ResolvedSymbol` owner-aware operation；
- session-wide preflight、排序、duplicate/overlap gate；
- relative → data/global → PLT 三 pass；
- Thumb function address/control-flow target 验证。

**门禁**

- REL implicit addend 在第一笔 write 前全部读取；
- operation value 与 ARM psABI/golden oracle 一致；
- owner 越权、非写 target、非 X jump target、Thumb bit、溢出和未知类型 fail-closed；
- 任一 write 失败回滚整个 session；
- 不把“目标位于 X region”描述成完整 CFI。

### 16.8 C17-a：生成 lifecycle/link map/product

**修改**

- `LifecycleEntry/InitPlan/FiniPlan`；
- SCC DAG dependency-first init 和逆序 fini；
- post-relocation function array decode；
- `LinkMapEntry/LinkContext/PreparedLinkManifest`；
- owner/range/Thumb 校验。

**门禁**

- preinit/init/init-array/fini-array/fini 顺序固定；
- cycle 内稳定顺序；
- invalid lifecycle function 不进入 plan；
- plan 不把 target address 转成 host function pointer。

### 16.9 C17-b：host 原子 publication 与最终 gate

**修改**

- `LinkPublisher::prepare_batch/commit_batch`；
- `PreparedLinkProduct/ReadyLinkCommit/LinkProduct`；
- host immutable snapshot publisher；
- committed lease owner 和 Drop release；
- Relink fixed-revision differential harness；
- GN check group、metrics/size baseline 和 dependency audit。

**门禁**

- prepare 的每个失败点仍完整 rollback；
- commit 不分配、不验证、不失败；
- reader 只观察旧 snapshot 或完整新 snapshot；
- committed product Drop 恰好释放所有 owned image；
- graph、symbol owner、relocation bytes、init/fini order 与 oracle 一致；
- 生产依赖无 `//librs`、VFS、application/thread-group；
- 所有 Phase 0 单映像回归继续通过。

依赖关系如下：

```text
C11-a allocation-explicit memory
   ↓
C11-b policy/resolver/snapshot summary
   ↓
C12 graph ──→ C13 symbols/hash ──→ C14 scope
                                     ↓
                              C15 session/rollback
                                     ↓
                              C16 ARM32 relocation
                                     ↓
                       C17-a lifecycle/link product
                                     ↓
                       C17-b host atomic publication
```

## 17. 测试与验证计划

本节描述 Phase 0.5 合入门；当前可以按既定安排先实现生产结构，之后统一补测试，但不能
删除这些 oracle 或把它们降级为建议项。

### 17.1 Fixture 集合

- root 无依赖；
- root → libc；
- root → foo → libc；
- root → foo/bar → 同一 libc 的菱形；
- A ↔ B 循环；
- 同 identity 不同路径/needed alias；
- 同 SONAME 不同 identity；
- global/weak/local/hidden/protected；
- undefined strong/weak；
- SysV hash、GNU hash、两者同时存在；
- 四种 ARM32 relocation；
- RELRO/GOT/PLT；
- preinit/init/init-array/fini-array/fini；
- 损坏 dynamic/dynstr/dynsym/hash/relocation/lifecycle 表；
- 超过 image/edge/depth/bytes/metadata/lookup budget。

人工 fixture 必须能精确控制 ELF bytes。编译器生成的真实 ARM32 DSO 作为额外 oracle，
不能替代畸形边界 fixture。

### 17.2 Fault injection

记录每种 backend 调用的第 N 次失败：

- resolver；
- allocate；
- reader read；
- memory write/zero/read；
- rollback-log reserve/metadata allocation；
- symbol/hash lookup budget；
- relocation plan allocation；
- 每笔 relocation write；
- cache prepare/sync；
- protection prepare/第 N 个 apply；
- publisher prepare。

每个失败点断言：无 published entry、无悬挂 lease、无重复 abort/release、原始错误 stage 和
context 未被 rollback 覆盖。

### 17.3 Differential oracle

- 固定 Relink revision，不追随上游漂移；当前调研基线为 `1928b38`；
- 只比较 BlueOS 支持子集：依赖 BFS、symbol owner、relocation target bytes、SCC 生命周期
  顺序和错误类别；
- musl/LLVM readelf 作为 ELF 语义和产物结构的第二 oracle；
- 不把 Relink 作为产品 dependency，也不为通过对拍复制其超出范围的功能。

### 17.4 GN 门禁

建议新增/扩展：

```text
kernel/loader:check_loader_host
kernel/loader:check_dynamic_linker
kernel/loader:check_loader_by_clippy
```

Phase 0.5 host gate 在所有 board 上只构建一次 host fixture；目标架构仍运行现有
`check_loader`，至少覆盖：

- `qemu_mps2_an385.release.dsc`；
- `qemu_riscv32.release.dsc`；
- `qemu_riscv64.release.dsc`；
- `qemu_virt64_aarch64.release.dsc`；
- `seeed_xiao_esp32c3.release.dsc`。

真实 `app → libc.so.1` QEMU 属于 Phase 1，不作为 C17 的伪门禁。

## 18. 代码量与审计门禁

C11 开始记录 `blueos_loader`：

- `.text/.rodata/.data/.bss` 增量；
- host peak metadata bytes；
- 单 fixture 最大 allocation 数和总映像字节；
- `unsafe` block 数量、位置和理由；
- 新增生产依赖；
- 单次 load 的 reader calls、symbol probes 和 relocation operations。

以下情况阻止 C17：

- 为 parser/graph/scope 引入 VFS、线程、锁或全局 singleton；
- 生产依赖 `librs` 或完整 Relink；
- 通过整文件 `Box<[u8]>` 快照规避 read-at/snapshot contract；
- `unsafe` 裸指针进入 graph/symbol/session 通用层；
- 未受预算控制的 `String/Vec/BTreeMap` 增长；
- 为 ARM32 复制一套独立于现有 relocation core 的第二 engine；
- commit 后仍存在返回 `Result` 的 install/link-map 操作。

## 19. Phase 0.5 完成定义

只有以下条件全部满足，才能进入 Phase 1：

- [ ] memory access 显式绑定 allocation，多活动 allocation 不串扰；
- [ ] Phase 0 与 Phase 0.5 policy preset 分离，Phase 0 public API 不可被放宽；
- [ ] 所有 program/dynamic feature policy 在 allocation/write 前判定；
- [ ] resolver identity 与 reader snapshot 绑定；
- [ ] dependency graph 有界、稳定并按 identity/SONAME 正确去重；
- [ ] dynstr/dynsym/GNU/SysV hash 不依赖 section header且全部有界；
- [ ] application/system scope、weak/hidden/protected 语义冻结；
- [ ] 所有 relocation operation 在第一笔 write 前完成 session-wide preflight；
- [ ] ARM32 四类 NOW relocation 的 owner、范围、Thumb 和值 oracle 通过；
- [ ] 所有 image protection/cache 收口后才可准备 publication；
- [ ] init/fini plan 在 relocation 后读取，顺序和地址验证通过；
- [ ] session 任意失败点使 allocation/lease/published entry 回到零；
- [ ] batch prepare 后的 commit 不分配、不失败，snapshot 原子可见；
- [ ] committed product 是全部 allocation lease 的长期 owner；
- [ ] loader 不执行 constructor，不依赖 kernel application/VFS/librs；
- [ ] Relink differential、malformed corpus、fault matrix 和 Clippy 通过；
- [ ] Phase 0 原有 ARM/RISC-V/AArch64/ESP32 回归全部通过。

## 20. 开工顺序

当前应从 C11-a 开始，不先写 `DependencyGraph`：

1. 把 `ImageMemory` 改成 allocation-explicit；
2. 让所有现有 image stage 只通过 transaction 的 owner-bound access；
3. 保持 `MemoryMapper` 单映像兼容测试/QEMU 全部通过；
4. 增加 transaction → rollback slot 的安全移交；
5. 再实现 `PHASE05_LOAD_POLICY + ArtifactResolver + DynamicFeatureSummary`；
6. 完成上述基础后进入 C12 BFS graph。

这样 Phase 0.5 的第一个多映像对象出现时，底层地址归属和 lease authority 已经能真实表达
它，而不是先用一个隐含 current allocation 的临时实现跑通，再在 C15 推翻。
