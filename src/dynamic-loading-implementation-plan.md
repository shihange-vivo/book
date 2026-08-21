# BlueOS 动态加载详细实现计划

本文基于主报告 [BlueOS 动态加载调研报告](./dynamic-loading-survey.md) 中已经确定的实现方案，结合当前 `blueos_loader`、内核线程/VFS/内存接口、`librs`、emutls 和 GN 工具链现状，给出可以直接拆分为工程任务的详细实现计划。

模块、文件和公共类型的命名以调研报告 [2.12 BlueOS 模块命名规范与参考词典](./dynamic-loading-survey.md#212-blueos-模块命名规范与参考词典) 为唯一基准；本计划只展开实现，不另造同义名称。

针对评审提出的应用定义、命名和安全问题，本计划先冻结以下约束：

- 顶层模块命名为 `application/`，控制面对象命名为 `ApplicationManager`。它管理动态装载应用实例的准入、句柄、资源归属、终止与回收，不参与 ready/running/suspended 等线程调度。
- 当前执行模型命名为 `ExecutionModel::ThreadGroup`，实现放在 `application/thread_group/`，资源寿命对象命名为 `ThreadGroup`。不再新增 `Rtos*` 类型，也不使用容易与语言运行时混淆的 `AppRuntime`。
- 公共 `ApplicationState` 只保留 `Loading/Running/Stopping/Terminated/Failed`；ELF 的 Parsed/Mapped/Relocated/Published 等状态由 staged loader 类型表达，不复制到公共应用查询状态中。
- 当前动态应用仍是共享地址空间中的特权代码，系统没有运行期 containment；签名包交付前仅允许内置/受控供应链映像。当前 Phase 0–3 必须交付工件准入、ELF 防御解析、固定 resolver/符号域、受限 relocation、映像 seal、原子发布和生命周期回收，保护 loader 与映像完整性，但不约束已运行代码的任意访存或跳转。
- MPU/MMU/PMP protection domain 不属于当前 Phase 0–3 的完成条件，只保留前向兼容接口。以后若要运行互不信任应用，必须把非特权执行、per-domain 配置切换、fault 归属、syscall copy-in/out、资源配额和共享库可写状态隔离作为一个闭环另行立项，不能只配置若干 region 就宣称隔离。

总体分为五个阶段：

- Phase 0：修正现有单 ELF loader，建立可信基线。
- Phase 0.5：在现有 `blueos_loader` 内扩充 `ImageLoader + DynamicLinker` 架构。
- Phase 1：完成 ARM32 Thumb v7-M `app.elf → libc.so.1` 纵向 MVP。
- Phase 2：支持应用私有多 DSO，并扩展到 Thumb v8-M hard-float、RISC-V64、AArch64 和 RISC-V32。
- Phase 3：增加受限 `dlopen/dlsym/dlerror/dlclose`，并完成签名、并发、回收、调试、OTA 等产品化能力。

Phase 1 的 QEMU/MVP 只允许运行内置或构建期 allowlist 的开发工件；`C39` 在 Phase 3 引入产品签名和安装/加载双重校验。在 `C39` 合入前不得开放外部应用分发，也不能把开发 allowlist 描述为签名能力。

首个真正可运行的里程碑定义为：

> 在 `qemu_mps2_an385` 上使用 `thumbv7m-vivo-blueos-newlibeabi`，由 shell 执行 `run /apps/hello/app.elf arg1`，加载标准 ARM32 Thumb `ET_DYN` 主程序和按需共享的 `libc.so.1`，完成符号解析、NOW relocation、RELRO、cache sync、构造函数、argv/auxv、emutls 和整 ThreadGroup 回收。

本文使用三套互不替代的编号：

- `S0`–`S11` 表示一次应用启动内部的**运行时流水线阶段**，对象只能沿 `admit → plan → map → relocate → seal → publish → initialize → run/reap` 前进。
- Phase 0–3 表示跨模块的**工程交付阶段**，一个 Phase 会实现若干运行时阶段。
- `C01`、`C02`……表示建议的**最小可审查提交单元**；每个提交都必须保持所属仓库可编译，并带对应的单元或集成测试。

第 3–9 节按组件给出类型定义，第 11 节再按 `S0`–`S11` 重新归属这些类型和方法；第 12、13 节分别给出工程 Phase 与实际提交顺序。实现时以第 11 节的阶段输出类型作为接口边界，不能用一个带大量 `Option` 和布尔标志的对象跨越整条流水线。

## 1. 最终代码分层

```text
apps/shell
    │ run <path> [args...]
    ▼
kernel::application::ApplicationManager
    │ 统一句柄、准入、状态、后端分派、退出回收
    ▼
kernel::application::thread_group::ApplicationLoader
    │ VFS、内存、cache、系统 DSO registry 适配
    ▼
blueos_loader::DynamicLinker
    │ 依赖图、scope、符号解析、重定位、init/fini plan
    ▼
blueos_loader::ImageLoader
    │ 单 ELF 解析、布局、复制、清零、映射
    ▼
ElfReader / ImageMemory / CodeCache / ArchRelocator
```

各层边界如下：

- `ApplicationManager` 是唯一的内核应用控制面入口；`ApplicationLoader` 是当前 `thread_group/` 后端的内核适配器。
- `DynamicLinker` 逻辑上不依赖内核，但当前作为 `blueos_loader` rlib 静态编入内核并在特权态运行。
- `libc.so.1` 是被动态加载的应用运行时，不属于 loader，也不能被 `ApplicationLoader` 静态依赖。
- 不新建第二个 loader crate，继续扩充现有 `kernel/loader`。
- `load_elf()` 保留，但降级为兼容包装。
- 内核、动态应用和 DSO 共用默认 PIC、支持 dynamic 的 BlueOS target/toolchain 与 sysroot；内核使用 PIC codegen，但最终以 `-static` 链接为自包含启动映像，不产生运行时 DSO 依赖。

命名按职责与所有权划分：`ImageLoader`、`DynamicLinker`、`LinkSession`、`ImageMemory` 等链接能力不加 OS 类型前缀；`ApplicationManager`、`ApplicationHandle`、`ApplicationLaunchRequest` 和 `ApplicationState` 是跨执行模型的控制面；只有确实表示“一组线程”的对象使用 `ThreadGroup`，例如 `ThreadGroupBackend`、`ThreadGroupMembership`。`ApplicationLoader`、`SystemDsoRegistry`、`SystemDsoLease`、`FlatImageMemory` 和 `ApplicationStartInfo` 的职责已经清楚，不再重复平台前缀。未来通用内核在同一 `application` 下增加 `process/`、`Process`、`AddressSpace` 和 `ProcessImageMemory`，不把 `ThreadGroup` 改名后硬复用。

建议目录最终形成：

```text
kernel/loader/src/
  lib.rs
  error.rs
  address.rs
  identity.rs
  reader.rs
  memory.rs

  image/
    mod.rs
    parser.rs
    metadata.rs
    layout.rs
    policy.rs
    loaded.rs

  relocation/
    mod.rs
    model.rs
    arch/
      arm.rs
      riscv64.rs
      aarch64.rs
      riscv32.rs

  dynamic_linker/
    mod.rs
    linker.rs
    session.rs
    runtime.rs             # Phase 3 RuntimeLoadSession/group scope delta
    context.rs
    dependency.rs
    scope.rs
    symbol.rs
    lifecycle.rs

kernel/kernel/src/application/
  mod.rs
  model.rs
  manager.rs
  registry.rs
  lifecycle.rs
  accounting.rs
  metrics.rs
  protection.rs
  adapters/              # 跨执行模型的内核能力适配
    vfs_reader.rs
    system_library_paths.rs
  thread_group/
    mod.rs
    backend.rs
    group.rs
    loader.rs
    system_dso.rs
    runtime_dso.rs         # Phase 3 per-ThreadGroup registry/handle/state
    start_info.rs
    membership.rs
    adapters/            # 仅共享地址空间模型特有
      flat_memory.rs
      artifact_resolver.rs
  process/                     # 未来由 blueos_user_process 条件编译
    mod.rs
    backend.rs
    process.rs
    exec.rs
    address_space.rs
    initial_stack.rs
```

指令缓存维护（`fence.i`、D-clean/I-invalidate）实现在 `arch/*/cache.rs` 作为架构通用服务，`ApplicationLoader` 直接调用 `sync_instruction_cache`，不再设 `code_cache.rs` adapter 文件。

## 2. 需要修改的现有代码

| 现有文件 | 主要修改 |
| --- | --- |
| [`kernel/loader/src/lib.rs`](../../kernel/loader/src/lib.rs) | 删除单体式 parse/copy/relocate 实现，改成公共 API 和 `load_elf()` 兼容包装 |
| [`kernel/loader/src/memory_mapper.rs`](../../kernel/loader/src/memory_mapper.rs) | 保留为兼容 adapter；修复 unchecked arithmetic、BSS、load bias、权限和对齐 |
| [`kernel/loader/BUILD.gn`](../../kernel/loader/BUILD.gn) | 加入新模块；删除生产依赖 `//librs`，测试依赖继续放在 test target |
| `kernel/loader/tests/` | 保留现有 static PIE/ET_EXEC 测试，增加 malformed ELF、dynamic ELF、relocation golden fixture |
| `kernel/loader/src/dynamic_linker/runtime.rs` | Phase 3 增加 `RuntimeLoadSession`、增量 group scope 和 runtime publish delta |
| [`kernel/kernel/src/lib.rs`](../../kernel/kernel/src/lib.rs) | 导出唯一的 `application` 模块；当前只声明 `thread_group` 后端，未来在模块边界条件编译 `process` |
| [`kernel/kernel/BUILD.gn`](../../kernel/kernel/BUILD.gn) | 内核增加 `//kernel/loader:blueos_loader` 依赖 |
| [`kernel/kernel/src/vfs/file.rs`](../../kernel/kernel/src/vfs/file.rs) | 增加不改变共享 offset 的 `read_at/read_exact_at/len` 内核接口 |
| [`kernel/kernel/src/thread/mod.rs`](../../kernel/kernel/src/thread/mod.rs) | `Thread` 增加 ThreadGroup membership |
| [`kernel/kernel/src/syscall_handlers/mod.rs`](../../kernel/kernel/src/syscall_handlers/mod.rs) | `CreateThread` 继承当前 ThreadGroup membership；增加应用初始化/退出协议及 Phase 3 运行期 DSO handler |
| `kernel/kernel/src/application/thread_group/runtime_dso.rs` | Phase 3 增加 per-ThreadGroup runtime registry、generation handle、load permit 和逻辑关闭状态 |
| [`kernel/kernel/src/scheduler/mod.rs`](../../kernel/kernel/src/scheduler/mod.rs) | thread cleanup 后发送 `ThreadExited` 事件，不在调度器路径直接卸载映像 |
| [`kernel/header/src/lib.rs`](../../kernel/header/src/lib.rs) | 增加版本化 ApplicationStartInfo、通用 ApplicationHandle 和追加式 syscall 编号（应用生命周期族及 Phase 3 的 `DynamicLibraryPrepareOpen/NextInitializer/InitComplete/Lookup/Close`） |
| [`librs/src/lib.rs`](../../librs/src/lib.rs) | 拆分 static start 和 dynamic `__librs_start_main(main, info)`；Phase 3 增加 `dlopen/dlsym/dlerror/dlclose` C ABI 包装 |
| `librs/src/dlfcn.rs` | Phase 3 实现 `dlfcn.h` 接口、init-plan 驱动和线程局部 `dlerror` |
| [`librs/src/pthread.rs`](../../librs/src/pthread.rs) | TCB 增加 `LibcApplicationContext`，子线程继承应用归属 |
| [`librs/src/tls.rs`](../../librs/src/tls.rs) | 保留 emutls，补域归属和退出顺序测试 |
| [`librs/BUILD.gn`](../../librs/BUILD.gn) | 保留 rlib，新增 `libc.so.1` DSO target、SONAME 和导出表 |
| [`kernel/rsrt/src/lib.rs`](../../kernel/rsrt/src/lib.rs) | 更新静态启动兼容入口，避免动态版 start ABI 破坏现有 shell/kernel image |
| [`build/templates/rust.gni`](../../build/templates/rust.gni) | linker/linker-script 参数从 target 中收敛到 artifact link policy |
| [`build/toolchain/blueos.gni`](../../build/toolchain/blueos.gni) | 建立内核、应用和 DSO 共用的 PIC/dynamic-capable target、clang + LLD 驱动与 ELF 检查流程 |
| `build/boards/*/BUILD.gn` | 删除散落的 relocation model、linker 和输出模式参数，只保留板级 ISA/ABI 与 kernel memory-layout 输入 |
| `apps/shell` | shell 本身改为动态应用（PIE + `DT_NEEDED: libc.so.1`，由 boot 自举经完整装载/链接路径启动）；`run.rs` 经 librs 的 `spawn` API 发 `ApplicationLaunch` syscall，由内核 syscall handler 分派到 `ApplicationManager::launch()`；manifest 明确指定 `ExecutionModel::ThreadGroup` |

## 3. `blueos_loader` 核心数据类型

### 3.1 地址、标识和范围类型

核心不能继续使用 `usize` 同时表示文件偏移、目标地址和 loader 指针。

```rust
#[repr(transparent)]
pub struct TargetAddr(u64);

#[repr(transparent)]
pub struct TargetWord(u64);

#[repr(transparent)]
pub struct ImageId(u32);

#[repr(transparent)]
pub struct AllocationId(u32);

#[repr(transparent)]
pub struct LinkDomainId(u64);

pub enum ElfClass {
    Elf32,
    Elf64,
}

pub struct TargetRange {
    pub start: TargetAddr,
    pub len: u64,
}
```

需要实现的方法：

```rust
impl TargetAddr {
    fn checked_add(self, value: u64) -> Result<Self, LoadError>;
    fn checked_add_signed(self, value: i64) -> Result<Self, LoadError>;
    fn checked_sub(self, other: Self) -> Result<u64, LoadError>;
    fn encode(self, class: ElfClass, endian: Endian) -> Result<TargetWord, LoadError>;
}

impl TargetRange {
    fn end(&self) -> Result<TargetAddr, LoadError>;
    fn contains(&self, addr: TargetAddr, width: usize) -> bool;
    fn overlaps(&self, other: &Self) -> bool;
}
```

所有 `p_offset + p_filesz`、`p_vaddr + p_memsz`、relocation offset 和 load bias 都必须使用 checked arithmetic。

### 3.2 结构化错误

替换当前的 `Result<(), &'static str>`：

```rust
pub struct LoadError {
    pub stage: LoadStage,
    pub kind: LoadErrorKind,
    pub image: Option<ImageId>,
    pub context: ErrorContext,
}

pub enum LoadStage {
    Read,
    Parse,
    Validate,
    Plan,
    Allocate,
    Copy,
    Dependency,
    Symbol,
    Relocate,
    Seal,
    Publish,
    Initialize,
    Release,
}
```

`LoadErrorKind` 至少覆盖：

- `BadElf`
- `UnsupportedByProfile`
- `OutOfBounds`
- `IntegerOverflow`
- `ResourceLimit`
- `PermissionConflict`
- `MissingDependency`
- `DuplicateSoname`
- `MissingSymbol`
- `DuplicateStrongSymbol`
- `UnsupportedRelocation`
- `RelocationOverflow`
- `AbiMismatch`
- `Io`
- `InvalidState`

`ErrorContext` 保存 ELF offset、vaddr、dynamic tag、relocation type、symbol、限制值等结构化信息，日志层再负责格式化。

## 4. `ElfReader`、`ImageMemory` 和 `ArchRelocator` 接口

建议冻结为以下等价接口：

```rust
pub trait ElfReader {
    fn len(&self) -> Result<u64, LoadError>;
    fn read_exact_at(
        &self,
        offset: u64,
        dst: &mut [u8],
    ) -> Result<(), LoadError>;
}

pub trait ImageMemory {
    fn allocate_image(
        &mut self,
        request: &AllocationRequest,
    ) -> Result<ImageAllocation, LoadError>;

    fn target_base(
        &self,
        allocation: AllocationId,
    ) -> Result<TargetAddr, LoadError>;

    fn write(
        &mut self,
        allocation: AllocationId,
        offset: u64,
        data: &[u8],
    ) -> Result<(), LoadError>;

    fn zero(
        &mut self,
        allocation: AllocationId,
        offset: u64,
        len: u64,
    ) -> Result<(), LoadError>;

    fn read_word(
        &self,
        location: TargetLocation,
        width: WordWidth,
    ) -> Result<TargetWord, LoadError>;

    fn write_word(
        &mut self,
        location: TargetLocation,
        width: WordWidth,
        value: TargetWord,
    ) -> Result<(), LoadError>;

    fn protect(
        &mut self,
        allocation: AllocationId,
        range: TargetRange,
        perms: MemoryPermissions,
    ) -> Result<ProtectionResult, LoadError>;

    fn release(&mut self, allocation: AllocationId);
}

pub trait CodeCache {
    fn sync_instruction_cache(
        &mut self,
        allocation: AllocationId,
        executable_ranges: &[TargetRange],
    ) -> Result<(), LoadError>;
}

pub enum ProtectionResult {
    LogicalOnly,
    HardwareEnforced,
}
```

关键约束：

- `DynamicLinker` 不能直接解引用 `TargetAddr`。
- relocation 必须先通过 `LoadedImage<S>::locate()` 转换为 `TargetLocation`。
- `FlatImageMemory` 才能把 allocation+offset 转换为当前内核指针。
- `protect()` 在无 MPU/MMU 平台可以返回 `LogicalOnly`，但不能假装已经物理保护。
- `CodeCache` 未实现时加载失败，不能提供空 stub。

架构接口：

```rust
pub trait ArchRelocator {
    fn machine(&self) -> u16;

    fn decode_relocation(
        &self,
        raw_type: u32,
        class: ElfClass,
    ) -> Result<RelocationKind, LoadError>;

    fn apply(
        &self,
        request: &RelocationRequest,
        memory: &mut dyn ImageMemory,
    ) -> Result<(), LoadError>;
}

pub enum RelocationKind {
    Relative,
    Absolute,
    GlobDat,
    JumpSlot,
}
```

统一公式：

- `RELATIVE = B + A`
- `ABS = S + A`
- `GLOB_DAT/JUMP_SLOT = S`，具体 addend 规则按 psABI
- REL 从目标内存读取 addend
- RELA 使用显式 addend
- 未知 relocation 必须报错

## 5. 单映像 `ImageLoader`

### 5.1 元数据结构

```rust
pub struct ParsedImage {
    pub header: ElfHeaderInfo,
    pub program_headers: Vec<ProgramHeaderInfo>,
    pub load_segments: Vec<LoadSegmentInfo>,
    pub dynamic: Option<DynamicInfo>,
    pub abi_note: Option<BlueOsAbiNote>,
    pub gnu_relro: Option<TargetRange>,
    pub stack_policy: StackPolicy,
}

pub struct DynamicInfo {
    pub soname: Option<String>,
    pub needed: Vec<String>,
    pub dynstr: TableRange,
    pub dynsym: TableRange,
    pub gnu_hash: Option<TableRange>,
    pub sysv_hash: Option<TableRange>,
    pub rel: Option<RelocationTable>,
    pub rela: Option<RelocationTable>,
    pub relr: Option<TableRange>,
    pub jmprel: Option<RelocationTable>,
    pub init: Option<TargetAddr>,
    pub init_array: Option<FunctionArray>,
    pub fini: Option<TargetAddr>,
    pub fini_array: Option<FunctionArray>,
}
```

Parser 应识别但不直接拒绝 `PT_INTERP`、`PT_TLS`、symbol version；由 `ApplicationArtifactPolicy` 返回 `UnsupportedByProfile`。

### 5.2 布局结构

```rust
pub struct ImageLayout {
    pub aligned_min_vaddr: TargetAddr,
    pub aligned_max_vaddr: TargetAddr,
    pub image_span: u64,
    pub max_align: u64,
    pub entry_vaddr: TargetAddr,
    pub segments: Vec<SegmentLayout>,
    pub relro: Option<TargetRange>,
}

pub struct SegmentLayout {
    pub file_offset: u64,
    pub file_size: u64,
    pub memory_vaddr: TargetAddr,
    pub memory_size: u64,
    pub final_permissions: MemoryPermissions,
}
```

`ImageLayoutBuilder::build()` 负责：

- `p_filesz <= p_memsz`
- `p_align` 为 0、1 或 2 的幂
- `p_offset % p_align == p_vaddr % p_align`
- segment/file/image 配额
- W+X、重叠权限冲突
- entry 必须落在 X segment
- 非零最低 `p_vaddr`
- segment gap
- 最大对齐和正确 load bias

### 5.3 分阶段 `LoadedImage`

`LoadedImage` 不保存可任意改写的 `ImageState`。映像共有字段放入 `LoadedImageData`，阶段专有数据放入 `S`；只有合法阶段的 `impl` 才暴露下一步方法：

```rust
pub struct LoadedImageData {
    id: ImageId,
    identity: ImageIdentity,
    kind: ImageKind,
    allocation: AllocationId,
    load_bias: TargetAddr,
    entry: TargetAddr,
    regions: Vec<LoadedRegion>,
    debug: ImageDebugInfo,
}

pub struct LoadedImage<S> {
    data: LoadedImageData,
    stage: S,
}

pub struct MappedState {
    parsed: ParsedImage,
}

pub struct RuntimeMetadataState {
    metadata: RuntimeImageMetadata,
}

pub struct RelocatedState {
    metadata: RuntimeImageMetadata,
}

pub struct SealedState {
    metadata: RuntimeImageMetadata,
    protections: Vec<AppliedProtection>,
}

pub type MappedImage = LoadedImage<MappedState>;
pub type RuntimeImage = LoadedImage<RuntimeMetadataState>;
pub type RelocatedImage = LoadedImage<RelocatedState>;
pub type SealedImage = LoadedImage<SealedState>;
```

所有阶段共有的只读方法：

- `target_addr(vaddr)`
- `locate(vaddr, width, required_permission)`
- `contains_executable(addr)`

阶段转换方法：

- `MappedImage::decode_runtime(policy) -> RuntimeImage`
- `RuntimeImage::symbol_address(symbol)`
- `RuntimeImage::into_relocated() -> RelocatedImage`，仅由完成整组 relocation 的 `LinkSession<LinkScopedState>` 调用
- `RelocatedImage::seal(memory, cache, plan) -> SealedImage`
- `SealedImage::build_link_map_entry()`

`RuntimeImageMetadata` 是 S4 的输出，保存已经过 load-bias 和 region 范围校验的 dynamic、symbol、relocation 与 lifecycle 视图；它不同于 S1 中仍以 ELF vaddr/file range 表示的 `DynamicInfo`。这样 relocation 和构造计划不会再次解释原始 dynamic tag。

`ImageLoader` 方法：

- `admit(reader, request, policy) -> AdmittedArtifact`
- `inspect(artifact, policy) -> ParsedImage`
- `build_layout(parsed, policy) -> ImageLayout`
- `reserve(layout, memory) -> ReservedImage`
- `copy_and_zero(artifact, reserved, memory) -> MappedImage`
- `load_unrelocated(...)`
- `load_static_pie(...)`
- `load_fixed_exec(...)`

兼容 `load_elf()` 调用链：

```text
SliceElfReader
  → ImageLoader::admit
  → ImageLoader::inspect
  → ImageLoader::build_layout
  → MemoryMapper compatibility adapter
  → load_static_pie/load_fixed_exec
```

## 6. `DynamicLinker` 代码架构

### 6.1 依赖与符号对象

```rust
pub struct DependencyGraph {
    nodes: Vec<DependencyNode>,
    edges: Vec<DependencyEdge>,
    soname_index: BTreeMap<String, ImageId>,
}

pub struct SymbolScope {
    ordered_images: Vec<ImageId>,
}

pub struct SymbolRef {
    pub name: String,
    pub binding: SymbolBinding,
    pub visibility: SymbolVisibility,
    pub symbol_type: SymbolType,
    pub value: TargetAddr,
    pub size: u64,
}

pub struct ResolvedSymbol {
    pub owner: ImageId,
    pub address: TargetAddr,
    pub size: u64,
    pub binding: SymbolBinding,
}
```

`DependencyGraph` 实现：

- `add_root()`
- `add_image()`
- `add_edge()`
- `find_by_soname()`
- `bfs_scope()`
- `constructor_order()`
- `destructor_order()`
- `strongly_connected_components()`

循环依赖按 SCC 处理：

- SCC 之间依赖优先。
- SCC 内按 BFS 首次发现顺序稳定排序。
- fini 使用 init 的逆序。

`SymbolScope` 实现：

- 应用 scope：main → app-private BFS → system DSO
- system scope：仅 system DSO
- `lookup_global()`
- `lookup_weak()`
- `resolve_for_relocation()`
- 重复 strong export 报错
- undefined weak 返回 0
- hidden 不进入外部 scope
- protected 的对象内引用不能被应用 interpose

### 6.2 分阶段链接事务

会话同样采用有限 typestate，只在四个真正改变可调用 API 的边界上分型；S0–S5 的细分状态由 `BuildingSession` 内部工作队列表达，不为每个解析循环创建一种泛型：

```rust
pub struct LinkBuildingState {
    pending_images: Vec<RuntimeImage>,
    discovery: DiscoveryQueue,
}
pub struct LinkScopedState {
    images: Vec<RuntimeImage>,
    scopes: ScopeSet,
}
pub struct LinkRelocatedState {
    images: Vec<RelocatedImage>,
    scopes: ScopeSet,
}
pub struct LinkSealedState {
    images: Vec<SealedImage>,
    scopes: ScopeSet,
}

pub struct LinkSession<S> {
    data: LinkSessionData,
    stage: S,
}

struct LinkSessionData {
    graph: DependencyGraph,
    rollback_log: Vec<RollbackAction>,
    metrics: LoadMetrics,
    disposition: SessionDisposition,
}

enum SessionDisposition {
    Active,
    Committed,
    RolledBack,
}

pub type BuildingSession = LinkSession<LinkBuildingState>;
pub type ScopedSession = LinkSession<LinkScopedState>;
pub type RelocatedSession = LinkSession<LinkRelocatedState>;
pub type SealedSession = LinkSession<LinkSealedState>;
```

方法只出现在对应阶段：

```text
BuildingSession::load_root(&mut self)
BuildingSession::expand_dependencies_bfs(&mut self)
BuildingSession::freeze_scopes(self)          -> ScopedSession
ScopedSession::relocate(self, arch, memory)   -> RelocatedSession
RelocatedSession::seal(self, memory, cache)   -> SealedSession
SealedSession::build_lifecycle_plans(&self, memory) -> (InitPlan, FiniPlan)
SealedSession::commit(self)                   -> LinkProduct
```

stage payload 独占对应阶段的 typed image，`LinkSessionData` 独占这些资源的 rollback authority；任何阶段的会话在 `Drop` 时如果 `disposition == Active`，自动按逆序：

- 取消 registry load permit
- 释放临时 lease
- 释放 image allocation
- 清除未发布 link map

`freeze_scopes/relocate/seal/commit` 均消费 `self`。转换失败时旧会话直接进入 `Drop` 回滚，调用方拿不到“部分重定位”或“部分 seal”的对象。`commit()` 先把资源移动到 `LinkProduct`，再将 disposition 置为 `Committed`；不得依赖 `mem::forget()` 绕过回滚。

构造函数不在 `LinkSession` 内执行，因为 ctor 已经产生不可回滚的外部副作用。

### 6.3 `DynamicLinker` 和输出

```rust
pub struct DynamicLinker<P, A> {
    policy: P,
    arch: A,
}

pub struct LinkProduct {
    context: LinkContext,
    entry: TargetAddr,
    main_program_headers: ProgramHeaderRuntimeInfo,
    init_plan: InitPlan,
    fini_plan: FiniPlan,
    link_map: Vec<LinkMapEntry>,
    metrics: LoadMetrics,
    publication_guard: PublicationGuard,
}
```

主要入口：

- `DynamicLinker::link(request, resolver, memory, cache)`
- `DynamicLinker::link_system_dso(...)`
- `DynamicLinker::link_application(...)`

`LinkContext` 保存：

- 应用自己拥有的 main/private DSO
- 对 system DSO 的只读实例引用
- dependency graph
- symbol scope
- fini plan
- link map
- 不直接拥有 system DSO allocation

`LinkProduct` 仍是未发布对象。`PublicationGuard` 接管原 session 的 allocation/permit/lease 回滚能力；`ApplicationLoader::publish_link_product()` 成功把资源移动到 ThreadGroup/System DSO registry 后才解除 guard。因而 `SealedSession::commit()` 只表示“冻结成可提交产品”，不表示其他线程已经可见；若后续内核 publish 失败，`LinkProduct` 仍能完整回滚。

### 6.4 Phase 3 的运行期链接状态

启动期 `LinkSession` 面对的是尚未运行、失败时可整体丢弃的依赖闭包；`dlopen` 面对的则是已经被多线程观察的 ThreadGroup，不能直接重新执行一次启动流程。Phase 3 在同一个 `DynamicLinker` 上增加增量事务，不复制 ELF、scope 或 relocation 实现：

```rust
#[repr(C)]
pub struct RuntimeDsoHandle {
    pub slot: u32,
    pub generation: u32,
}

pub struct RuntimeLinkState {
    pub startup: LinkContext,
    pub published: RuntimeLinkSnapshot,
    pub handles: RuntimeDsoHandleTable,
}

pub struct RuntimeLoadSession {
    pub baseline_generation: u64,
    pub root: ArtifactIdentity,
    pub new_group: LinkSession<LinkBuildingState>,
    pub flags: DlopenFlags,
}

pub enum RuntimeDsoState {
    Loading { owner_thread: ThreadId },
    Initializing { owner_thread: ThreadId },
    Ready,
    LogicallyClosed,
    Failed,
}
```

`RuntimeLoadSession` 只记录相对于已发布快照的增量：新映像、新依赖边、group scope、init/fini plan、emutls control object、link-map 条目和待增加的 system DSO lease。它沿用 `LinkSession` 的 map→relocate→seal 与 rollback log；失败时只能撤销本次增量，不能修改已经运行的 startup image 或先前 `dlopen` group。

运行规则冻结如下：

1. `dlopen` 的名称只能解析到当前签名包 manifest 声明的私有 DSO 或系统只读 catalog，不能使用工作目录、环境变量、任意 `RPATH` 或未验签绝对路径。通过 ABI/配额准入后，runtime registry 按签名工件身份、规范化文件身份、artifact generation 和 SONAME 做双重去重。同一身份并发首开由 `begin_open()` 产生一个 load permit，其他线程等待 `Initializing → Ready/Failed`；同 SONAME、不同身份在没有 namespace 能力时明确失败。
2. 首版只接受 `RTLD_NOW | RTLD_LOCAL`，并接受 `RTLD_NODELETE`；拒绝 `RTLD_LAZY/RTLD_DEEPBIND`。新 group 的 relocation scope 由当前 ThreadGroup 的启动全局 scope 加本次依赖闭包构成，system scope 隔离规则保持不变。scope 一旦用于 relocation 就不反向重绑定既有 GOT。
3. 新 group 完成 relocation、seal 和 cache sync 后，以一个 generation 原子发布为 `Initializing`；随后释放 loader/registry 锁，由调用 `dlopen` 的 `librs` 执行 constructor，再以 `DynamicLibraryInitComplete` 切换为 `Ready`。并发 waiter 只在 `Ready` 后返回；同一初始化线程的嵌套 `dlopen` 通过 owner/load-group 标识识别，不能等待自己造成死锁。
4. `dlsym(handle, name)` 先验证 handle 属于当前 ThreadGroup 且 generation 匹配，再按 root→该 group 依赖闭包的冻结顺序查找；返回地址仍经过 symbol type、owner 和 X/data region 校验。`dlerror` 文本保存在调用线程的 `librs` TLS 中，内核只返回稳定错误码和有界错误上下文。
5. 重复 `dlopen` 同一 Ready 实例只增加显式 open count，不再次映射、重定位或执行 constructor；显式 open count 与依赖边保活计数分开记录。首版 `dlclose` 只减少该计数；归零后使 handle 失效并将私有 group 标记为 `LogicallyClosed`，但不执行 fini、不 unmap，已有代码/数据地址仍保活到整个 ThreadGroup 退出。再次打开复用映像并获得新 generation，不重复执行 constructor。ThreadGroup 退出时，runtime group 按提交顺序逆序、每组内部再按依赖逆序执行 fini 后统一回收。
6. `RTLD_GLOBAL/RTLD_NOLOAD`、`RTLD_DEFAULT`、`dlopen(NULL, ...)`、`dladdr/dl_iterate_phdr` 是 Phase 3 的第二级兼容项：只有在 local 基线稳定后才加入。`RTLD_GLOBAL` 只能在完整事务提交时把新 group 原子追加到“未来装载可见”的全局 scope，不能改变既有 relocation；首版不实现 `RTLD_NEXT`。

`dlopen()` 的 libc 包装必须在 constructor 全部完成后才把 handle 返回给应用。建议追加 `DynamicLibraryPrepareOpen/NextInitializer/InitComplete/Lookup/Close` syscall：prepare 返回 opaque handle 和内核持有的 init-plan token，`librs` 在锁外逐项执行初始化函数后确认完成。若 constructor 使线程异常退出，由于外部副作用已经发生且 group 已进入 `Initializing`，当前 ThreadGroup 整体进入停止/回收流程，不能假装只回滚本次 `dlopen` 就能恢复原状态。

真正立即回收的 `dlclose` 不是把上述 `unmap` 打开即可。`dlsym` 已经把裸函数/数据指针交给 C 代码，loader 无法证明它没有逃逸到其他线程、GOT、回调、定时器、atexit、TLS destructor 或 unwind 元数据。在当前特权共享地址空间中，悬空指针还可能破坏整个系统。因此当前交付只保证“逻辑关闭 + ThreadGroup 退出回收”；若以后必须中途回收，需新增受限 `UnloadSafe` 插件 profile、显式调用/回调 lease、停止新调用、线程安全点和完整 quiescence 证明，不能对任意 POSIX C DSO 宣称安全立即卸载。

## 7. 内核单一 `application` 控制面

`application` 是内核唯一的应用控制面，不为 thread-group 和进程各建一套 manager。它不调度线程，也不提供 libc 运行时；公共层只抽象两种模型共有的准入、句柄、查询、终止和回收入口，映像布局、启动 ABI、地址空间和 DSO 生命周期仍由各自后端负责。

### 7.1 `ApplicationManager` 与公共控制面

```rust
pub enum ExecutionModel {
    ThreadGroup,
    Process,
}

pub enum ApplicationState {
    Loading,
    Running,
    Stopping,
    Terminated,
    Failed,
}

pub struct ApplicationLaunchRequest {
    pub path: String,
    pub argv: Vec<String>,
    pub envp: Vec<String>,
    pub execution_model: ExecutionModel,
    pub quota: ApplicationQuota,
}

pub struct ApplicationManager {
    registry: ApplicationRegistry,
    thread_group: ThreadGroupBackend,
    #[cfg(blueos_user_process)]
    process: ProcessBackend,
    events: ApplicationEventQueue,
}

pub struct ApplicationRegistry {
    instances: SpinMutex<BTreeMap<ApplicationHandle, ApplicationInstance>>,
    next_id: AtomicU64,
}

pub enum ApplicationInstance {
    ThreadGroup(Arc<ThreadGroup>),
    #[cfg(blueos_user_process)]
    Process(Arc<Process>),
}
```

`ExecutionModel` 是稳定 manifest/ABI note 的 wire-level 选择，始终能识别 `ThreadGroup` 与预留的 `Process` 值；当前未编译进程后端时，`Process` 请求返回 `UnsupportedExecutionModel`。不能仅根据 ELF 是否带 `PT_INTERP` 自动选择执行模型。

公共类型与职责：

- `ApplicationHandle`：跨执行模型的 opaque 句柄，对外只暴露整数 C ABI。
- `ApplicationState`：对查询接口只暴露 `Loading/Running/Stopping/Terminated/Failed`；scheduler thread state 和 staged loader typestate 不进入此枚举。
- `ApplicationQuota/ApplicationUsage`：统一资源限制和记账维度，由后端填充模型特有项目。
- `ApplicationEventQueue`：接收线程/任务退出、终止请求和延迟回收事件。
- `ApplicationRegistry`：只拥有已发布实例表和句柄代次，不在表锁内执行 VFS、链接、终止或回收。
- `ApplicationInstance`：只保存到后端实例的 tagged reference，不把 RTOS 和 Process 字段合并成巨型结构体。

主要方法：

- `global()`
- `launch(request) -> ApplicationHandle`
- `terminate(handle, reason)`
- `query(handle) -> ApplicationInfo`
- `on_execution_unit_created(handle, id)`
- `on_execution_unit_exited(handle, id, status)`
- `begin_exit(handle, status)`
- `finish_exit(handle)`
- `abort(handle, reason)`
- `reap(handle)`

当前 `launch()` 校验信任证据、manifest、ABI note 和 `ExecutionModel::ThreadGroup` 后调用 `ThreadGroupBackend`：MVP 只接受内置/构建期 allowlist，产品配置只接受有效签名。未来 `blueos_user_process` 启用时再增加 `ProcessBackend` 分支。两个后端可以同时编入，以支持通用内核上的 RTOS 兼容应用。条件编译只允许放在 `mod process`、后端字段和 tagged variant 等模块边界，不能用互斥 `cfg` 定义两套 `ApplicationManager`。开关由一个权威内核配置转换为 `blueos_user_process`，不得在 GN、Cargo feature 和零散 rustflags 中重复决策。

锁要求：

- 全局 instance 表锁只做查找、插入和删除。
- 不持有 `ApplicationManager` 锁执行 VFS、ELF parse、relocation、exec 映射或 ctor/fini。
- 不持有 registry 锁执行文件 IO 和应用代码。
- scheduler cleanup 路径只投递退出事件，真正回收由 application reaper 完成。

### 7.2 thread-group 后端与 `ThreadGroup`

```rust
pub struct ThreadGroupBackend {
    loader: ApplicationLoader,
}

pub enum ThreadGroupState {
    Loading,
    Active,
    Draining,
    Reaped,
}

pub struct ThreadGroup {
    pub id: ThreadGroupId,
    pub handle: ApplicationHandle,
    pub link_domain: LinkDomainId,
    pub abi: AbiProfile,
    pub state: ThreadGroupState,
    pub quota: ApplicationQuota,
    pub usage: ApplicationUsage,
    pub protection_domain: ProtectionDomainId,
    pub link_context: Option<LinkContext>,
    pub threads: BTreeSet<ThreadId>,
    pub system_leases: Vec<SystemDsoLease>,
    pub start_storage: Option<ApplicationStartStorage>,
    pub exit_status: Option<i32>,
}
```

`ThreadGroupBackend` 实现 `prepare/start/terminate/reap` 等内部操作，但不另建第二份全局 manager、handle table 或 event queue。`ThreadGroup` 不是 `Thread` 的包装：它持有本次装载产生的映像、link context、系统库 lease、应用级 libc 状态、全部成员线程和最终退出码。

公共查询状态不暴露 relocation 等实现细节。内部只需维护下面四个资源寿命阶段；`S0`–`S10` 的 staged 类型继续负责更细的链接阶段：

```text
Loading ── publish + init success ──→ Active ── begin_exit ──→ Draining ──→ Reaped
   └──────── link/ctor failure ───────────────────────────────→ Draining ──→ Reaped
```

方法：

- `transition(expected, next)`
- `install_link_product()`
- `register_thread()`
- `unregister_thread()`
- `record_init_progress()`
- `activate_after_init_complete()`
- `begin_exit(status)`
- `can_run_fini()`
- `mark_fini_complete()`
- `can_reap()`
- `take_resources_for_reap()`

`record_init_progress` 只记录 S10 的构造进度，`activate_after_init_complete` 原子完成内部 `Loading → Active` 和公共 `Loading → Running`；两者都不制造 `Relocated/Initialized` 应用状态。`ThreadGroup` 是当前模型的管理和回收单位，不是安全地址空间。未来 `process/` 是其同级后端：内核侧只实现 `Process/AddressSpace`、exec stage0、initial stack 和 user-thread 生命周期；普通 `DT_NEEDED`、符号绑定、relocation 与 ctor 由独立的用户态 `ld.so` 完成。

### 7.3 `ApplicationLoader`

职责是把 BlueOS 能力适配给 `DynamicLinker`：

```rust
pub struct ApplicationLoader {
    system_registry: Arc<SystemDsoRegistry>,
    system_paths: SystemLibraryPaths,
    artifact_policy: ApplicationArtifactPolicy,
    memory: FlatImageMemory,
    cache: ArchCodeCache,   # arch/*/cache.rs 架构服务，实现 blueos_loader 的 CodeCache trait
}
```

方法：

- `link_application(group, request)`
- `resolve_needed(requester, soname)`
- `open_artifact(path)`
- `acquire_system_dso(soname)`
- `publish_link_product(group, product)`
- `rollback_failed_link(session)`
- `release_group_images(group)`
- `try_quiesce_system_dso(soname)`

### 7.4 System DSO registry

artifact 与运行实例必须分开：

```rust
pub struct DsoArtifact {
    pub soname: String,
    pub path: String,
    pub build_id: BuildId,
    pub abi_note: BlueOsAbiNote,
    pub parsed_metadata: Arc<ParsedImage>,
}

pub struct SystemDsoRegistry {
    artifacts: DsoArtifactCatalog,
    instances: DsoInstanceRegistry,
}

pub struct DsoArtifactCatalog {
    entries: BTreeMap<ArtifactIdentity, Arc<DsoArtifact>>,
}

pub struct DsoInstanceRegistry {
    instances: BTreeMap<(LinkDomainId, String), SystemDsoState>,
}

pub struct SystemDsoLoadPermit {
    soname: String,
    generation: u64,
}

pub struct SystemDsoInstance {
    pub artifact: Arc<DsoArtifact>,
    pub image: SealedImage,
    pub link_context: LinkContext,
    pub init_state: InitState,
    pub fini_plan: FiniPlan,
    pub generation: u64,
}

pub enum SystemDsoState {
    Loading,
    Relocated,
    Initializing,
    Ready,
    Quiescing,
    Failed,
}
```

接口：

- `acquire_or_begin_load()`
- `wait_ready()`
- `publish_relocated()`
- `mark_initializing()`
- `mark_ready()`
- `fail_loading()`
- `release_lease()`
- `begin_quiescence()`
- `finish_unload()`
- `keep_cached()`

`SystemDsoLoadPermit` 是 `Loading` generation 的排他提交权限，失败或销毁会唤醒 waiter；`SystemDsoLease` 才是 Ready 实例的长期保活令牌，包含 SONAME、generation 和 registry 引用。lease 的 `Drop` 只能投递 quiescence 事件，不能直接在任意线程栈上执行 fini 或释放代码。

## 8. 线程归属与退出协议

### 8.1 为什么不能只使用现有 `Thread`

现有 `Thread` 和 `ThreadState` 已足够表达调度状态，但不能回答以下问题：一个 pthread 属于哪次动态装载；最后一个线程退出后哪些 image、TLS、atexit、link map 和 system DSO lease 才能释放；应用 abort 时应停止哪些兄弟线程；同一路径重新启动后旧 handle 是否仍有效。因此新增的是一个资源所有权 group，而不是第二套 scheduler。

如果最终 MVP 明确删除 pthread、应用私有 DSO、应用级 TLS/atexit 和独立配额，并规定主线程退出后立即永久驻留映像，则可以把 `ThreadGroup` 缩成主线程上的少量 metadata；当前 Phase 1/2 目标不满足这些前提。

当前 `CreateThread` 已支持 `entry + arg`，但线程没有域归属。需要增加：

```rust
pub struct ThreadGroupMembership {
    pub group_id: ThreadGroupId,
}
```

在 `Thread` 中增加：

```rust
application_membership: Option<ThreadGroupMembership>
```

具体规则：

1. `ApplicationManager` 分派到 `ThreadGroupBackend` 创建主线程时显式设置 membership。
2. 动态应用调用 `pthread_create` 时，内核 `CreateThread` 从当前线程继承 membership，不能信任应用传入任意 group handle。
3. 创建成功后调用 `ApplicationManager::on_execution_unit_created()`。
4. `pthread_exit` 先运行 pthread key/emutls destructor。
5. scheduler 执行现有 cleanup。
6. cleanup 完成后投递 `ThreadExited`。
7. ThreadGroup 最后一个线程退出且 fini 已完成后才释放 image 和 libc lease。

建议实现内部两阶段应用退出：

- `ApplicationBeginExit(handle, status)`：阻止新线程，等待其他线程退出。
- libc 运行 atexit 和应用 fini plan。
- `ApplicationFinishExit(handle)`：标记 fini 完成并退出协调线程。
- 普通子线程继续使用 `ExitThread`。

上述 syscall 必须验证 `handle` 与当前线程 membership 一致，并采用 append-only 编号。

应用启动同样走 syscall：shell 经 librs 的 `spawn`/`posix_spawn` API 发出 `ApplicationLaunch(path, argv, envp)`，内核在 syscall handler 中完成 argv/envp 的 copy-in 与范围校验、S0 准入（allowlist/manifest/配额）后分派到 `ApplicationManager::launch()`，返回 `ApplicationHandle`。`ApplicationLaunch` 与退出协议 syscall 同属应用生命周期 ABI，一并进入 append-only 冻结；结构体按 `abi_version + struct_size` 版本化。唯一例外是系统自举：boot 阶段由内核内部直接函数调用 `ApplicationManager::launch()` 装载第一个动态应用（shell）——此时尚无任何应用能发 syscall。

Phase 3 再追加运行期 DSO ABI：`DynamicLibraryPrepareOpen/NextInitializer/InitComplete/Lookup/Close`。handler 必须从当前线程反查 ThreadGroup，copy-in 有界 path/symbol/flags，拒绝应用指定其他 group；`RuntimeDsoHandle { slot, generation }` 只在所属 ThreadGroup 内有效。`PrepareOpen` 只返回 opaque handle/init-plan token，不把 loader 内部对象或可写 link-map 指针暴露给应用；`Lookup` 仅在相应实例为 `Ready`（或当前初始化线程的受控递归路径）时返回已验证目标地址。

### 8.2 当前阶段的装载安全控制

当前安全策略只在工件、装载、链接、发布和回收边界执行，不做逐 load/store 软件检查，也不声称完整控制流完整性。实现必须形成三类显式对象：`ArtifactPolicy` 管理来源、ABI、ELF feature、资源上限和依赖解析策略；`RelocationPolicy` 管理 relocation 白名单、写入 owner/range、符号来源和控制流目标策略；`SealPlan` 管理 RX/R/RW+NX、RELRO、cache sync 与实际保护结果。它们的具体字段分别由 S0/S1、S7 和 S8 的阶段类型定义，不能在 adapter 中另写一套旁路判断。

强制规则如下：

1. `ArtifactPolicy` 在 allocation 前校验来源、machine/class/endian/ABI note、文件与映像配额；拒绝 W+X、TEXTREL、可执行栈、RPATH/环境搜索及 profile 未支持的 TLS/IFUNC/COPY/version 等特性。
2. system resolver 只接受固定可信路径和 catalog identity；依赖图按 SONAME 与文件身份双重去重；application/system scope 在 relocation 前冻结，应用不能 interpose system DSO 自身解析，也不获得完整内核符号表。
3. 每条 relocation 的 `P` 必须能转换为当前 owner 中带 allocation、offset、width 和权限的 `TargetLocation`；写入前校验范围、对齐、目标字宽、addend 和所有算术，未知类型和缺失 strong symbol fail-closed。
4. `S` 必须来自当前冻结 scope。`JUMP_SLOT` 等明确的控制流 relocation、ELF entry 及 init/fini entry 必须落在允许 owner 的 X region；undefined weak=0 是否允许由 artifact profile 显式决定，不能作为通用 X-range 旁路；符号类型、Thumb bit 等架构规则由 `ArchRelocator` 处理。该规则不覆盖已静态解析的直接分支、运行期函数指针、返回地址或应用自行计算的间接跳转，不能标记为 CFI。
5. S9 前映像不能被线程入口、registry Ready 实例或公共 link map 发现；全部 relocation 完成后才执行 `SealPlan`，形成 text RX、rodata/RELRO R、data/BSS RW+NX，完成 cache 同步后再原子发布。`AppliedProtection` 必须准确记录 `LogicalOnly` 或 `HardwareEnforced`。
6. S0–S8 任一步失败由 rollback log 逆序释放；S9 后由 ThreadGroup、lease 和 deferred reaper 保活，执行流、callback、TLS destructor 或外部引用未静默前不得释放代码。

这些规则的验收以畸形 ELF corpus、人工 relocation fixture、故障注入和状态快照完成，不依赖 MPU/PMP。它们保证“loader 不替应用越界写或制造非法 relocation-backed 跳转”，不保证同特权应用开始执行后不能直接访问系统地址。

### 8.3 后续可选的硬件隔离演进

以下硬件状态只用于说明未来路径，不属于当前 Phase 0–3 的交付门禁：

| 平台 | 仓库现状 | Phase 0/1 可用方式 | 成为安全边界还缺什么 |
| --- | --- | --- | --- |
| Cortex-M v7/v8-M | v8-M MPU 已支持系统/线程栈 guard；线程 `CONTROL` 只选择 PSP，没有设置 `nPRIV`；首发 v7-M profile 没有应用 protection domain | ARM32 MVP 只运行内置/构建期 allowlist 的可信特权代码；v7-M 首发板记录 `LogicalOnly`，v8-M 可保留 guard 并实验性 seal RX/RO/RW | unprivileged Thread mode、每 `ThreadGroup` MPU config、切换时激活、fault attribution、syscall/copy-in/out |
| RISC-V32/64 | 当前只支持 M-mode，源码明确 S/U mode 尚未支持；没有应用 PMP domain | 后续架构 backend 仍只运行内置/构建期 allowlist 的可信特权代码；loader 继续做范围和权限逻辑校验 | U mode、trap/syscall、PMP layout、每线程 protection-domain id、上下文切换和 fault 恢复 |
| AArch64 | EL1 MMU linear map 已存在，但 EL0 vector 明确 unsupported | 可以增加内核映像页的 W^X/RELRO hardening 实验 | EL0 entry/return、独立 address space/ASID、用户页映射、copy-in/out、异常归属 |

未来演进时，`ProtectionDomain::level()` 必须返回实际能力：`LogicalOnly`、`HardwarePermissions` 或 `IsolatedDomain`。只有第三种且配套非特权执行、保护域切换、fault attribution、资源配额和 syscall copy-in/out 全部完成时，产品才可以宣称动态应用被隔离。单个 region 的 `ImageMemory::protect()` 只返回 `ProtectionResult::LogicalOnly` 或 `ProtectionResult::HardwareEnforced`；签名验证证明发布者身份，不等于运行时 containment。

### 8.4 后续演进的 `ProtectionDomain` 预留接口

在 `ThreadGroup` 中保存 `ProtectionDomainId`，线程只保存不可伪造的 membership；scheduler 切换时据此激活保护配置：

```rust
pub enum ProtectionLevel {
    LogicalOnly,
    HardwarePermissions,
    IsolatedDomain,
}

pub trait ProtectionDomain {
    fn level(&self) -> ProtectionLevel;
    fn seal_image(&mut self, regions: &[ImageRegion]) -> Result<(), ProtectionError>;
    fn attach_thread(&mut self, thread: ThreadId) -> Result<(), ProtectionError>;
    fn activate(&self) -> Result<(), ProtectionError>;
    fn validate_user_range(
        &self,
        ptr: TargetAddr,
        len: usize,
        access: Access,
    ) -> Result<(), ProtectionError>;
}
```

后续实现 protection domain 时，不要自己实现逐 load/store 的软件地址检查。当前阶段的软件检查仍只放在装载布局、relocation 写入和资源分配边界；真正引入非特权执行后，syscall pointer 检查和普通访存隔离分别由 copy-in/out 层与 MPU/PMP/MMU 承担。上下文切换只在 protection domain 变化或配置 dirty 时重编程硬件，同一应用的多个线程可复用 image/data region，只更新必要的线程栈 guard。Tock 的 Cortex-M MPU backend 已采用 config ID/dirty 缓存，Zephyr 的 `k_mem_domain` 则让子线程继承 protection domain，可直接参考这两类接口边界。

该演进立项后，每个支持板必须增加以下 benchmark gate：同域线程切换、跨域线程切换、空 syscall、N 个 MPU/PMP region seal、应用启动时间、region 对齐造成的 RAM 浪费。region 数不足时通过预扫描把 RX/RO 和 RW/stack/heap 归并为少量 arena；无法安全表达的映像拒绝加载，不能退化为 silent RWX。

参考：[BlueOS Cortex-M MPU](../../kernel/kernel/src/arch/arm/mpu/mpu_v8m.rs)、[BlueOS Cortex-M CONTROL](../../kernel/kernel/src/arch/arm/mod.rs)、[BlueOS RISC-V privilege 状态](../../kernel/kernel/src/arch/riscv/mod.rs)、[BlueOS AArch64 vector](../../kernel/kernel/src/arch/aarch64/vector.rs)、[Zephyr memory domain](../../../zephyr/kernel/userspace/mem_domain.c)、[Zephyr LLEXT partitions](../../../zephyr/subsys/llext/llext_mem.c)、[Tock Process MPU](../../../tock/tock/kernel/src/process_standard.rs)、[Tock MPU config cache](../../../tock/tock/arch/cortex-m33/src/mpu_v8m.rs)。

## 9. ApplicationStartInfo 与 librs

报告当前草案只有 `init_plan`，但退出流程需要 fini。建议在 ABI v1 冻结前直接补齐：

```c
struct BlueOsFunctionPlan {
    uint32_t abi_version;
    uint32_t struct_size;
    size_t count;
    const void (*const *entries)(void);
};

struct BlueOsApplicationStartInfo {
    uint32_t abi_version;
    uint32_t struct_size;
    uintptr_t app_handle;

    int32_t argc;
    const char *const *argv;
    const char *const *envp;

    const struct BlueOsAuxvEntry *auxv;
    size_t auxv_count;

    const struct BlueOsFunctionPlan *init_plan;
    const struct BlueOsFunctionPlan *fini_plan;
};
```

`ApplicationStartStorage` 负责保证所有指针稳定：

```rust
pub struct ApplicationStartStorage {
    argv_strings: Vec<Box<[u8]>>,
    argv_ptrs: Box<[*const c_char]>,
    env_strings: Vec<Box<[u8]>>,
    env_ptrs: Box<[*const c_char]>,
    execfn: Box<[u8]>,
    random: Option<Box<[u8; 16]>>,
    phdr_copy: Option<Box<[ProgramHeader]>>,
    auxv: Box<[BlueOsAuxvEntry]>,
    init_entries: Box<[BlueOsInitFn]>,
    fini_entries: Box<[BlueOsInitFn]>,
    info: Pin<Box<BlueOsApplicationStartInfo>>,
}
```

`librs` 增加：

```rust
pub struct LibcApplicationContext {
    pub handle: ApplicationHandle,
    pub auxv: Box<[BlueOsAuxvEntry]>,
    pub atexit: RwLock<Vec<AtExitEntry>>,
}
```

`PthreadTcb` 增加：

```rust
libc_application_context: Option<Arc<LibcApplicationContext>>
```

动态入口：

```rust
pub unsafe extern "C" fn __librs_start_main(
    main: extern "C" fn(i32, *mut *mut c_char, *mut *mut c_char) -> i32,
    info: *mut BlueOsApplicationStartInfo,
) -> !
```

执行顺序：

1. 校验 `abi_version/struct_size`。
2. 创建并登记主线程 `LibcApplicationContext/PthreadTcb`。
3. 初始化应用级 stdio/emutls。
4. 顺序执行 init plan。
5. 调用内部 `ApplicationInitComplete` SWI。
6. 调用 `main(argc, argv, envp)`。
7. `ApplicationBeginExit`，等待其他线程退出。
8. 运行 atexit 和 fini plan。
9. 清理主线程 emutls/TCB。
10. `ApplicationFinishExit`，不返回。

为避免破坏当前静态 shell/kernel image：

- 动态 ABI 使用新版 `__librs_start_main(main, info)`。
- 另保留 `__librs_start_main_static()` 或 Rust 内部 static start。
- 修改 `kernel/rsrt` 调用 static 入口。
- `libc.so.1` 的导出表只导出动态启动 ABI。

## 10. 工具链统一方案

目标是同一 ISA/ABI 下只保留一套默认 PIC、支持 dynamic linking 的 BlueOS target/toolchain 和一份 PIC sysroot，同时用于编译内核、动态应用和 DSO。这里统一的是编译器、链接器、target 能力、代码模型和 sysroot；最终链接仍按产物启动契约区分，不能把 `-static`、`-pie` 和 `-shared` 混成一组参数。

应把当前混杂的配置拆成三层：

```text
GN toolchain
  决定统一的编译器/链接驱动：clang + LLD

board ABI config
  决定 march/mabi/float ABI/target feature

artifact link policy
  决定 kernel_static、dynamic_app 或 dso 的 static/PIE/shared 最终契约
```

Rust target 默认 `relocation-model=pic` 并声明支持 dynamic linking。`dynamic=true` 只表示能够生成 PIE/DSO，不表示所有 executable 都包含 `PT_DYNAMIC`，也不表示内核需要先由 `ApplicationLoader` 装载。

### 10.1 三种产物策略

#### `kernel_static`

- 编译阶段使用 `-C relocation-model=pic`；所有 Rust/C/C++ 输入对象和 `core/alloc/compiler_builtins` 都是 PIC；
- 最终链接明确使用 `-static -nostdlib -nostartfiles` 和板级 kernel `link.x`，不保留未定义动态符号；
- 当前输出为 `ET_EXEC` 或板级要求的固件格式，入口、vector、ROM/RAM、copy/zero table 和固定启动地址保持不变；
- 禁止 `PT_INTERP`、`DT_NEEDED`、TEXTREL 和需要启动后处理的 dynamic relocation；允许存在已由静态链接器完全解析的 GOT；
- 若未来要让内核本身成为可重定位 `ET_DYN`，必须另行实现 boot stage0/内核 self-relocator，不属于本动态应用方案。

#### `dynamic_app`

- `relocation-model=pic`
- `-pie`
- `--no-dynamic-linker`
- `-z relro`
- `-z now`
- `-z noexecstack`
- `-z text`
- `--build-id`
- 禁止 `-static`
- 禁止 `-z norelro`
- 链接 `blueos_scrt1.o` 和 SDK `libc.so`

#### `dso`

- PIC
- `-shared`
- `-soname,<name>`
- `-z relro -z now -z noexecstack -z text`
- 导出清单
- 普通 DSO 产生 `DT_NEEDED: libc.so.1`

### 10.2 配置收敛

建议新增：

```text
build/config/artifact_profiles.gni
build/templates/kernel_static.gni
build/templates/dynamic_app.gni
build/templates/blueos_dso.gni
build/scripts/check_blueos_elf.py
```

target 只能这样选择：

```gn
blueos_kernel_static("blueos") {
  linker_script = default_kernel_linker_script
  deps = [ "//kernel/kernel" ]
}

blueos_dynamic_app("hello") {
  sources = [ "main.c" ]
  deps = [ "//sdk:libc_import" ]
}

blueos_dso("foo") {
  soname = "libfoo.so.1"
  exports = "foo.exports"
}
```

不允许 target 自己再传 `-Clinker`、`-fuse-ld`、`-static/-pie/-shared` 或 `-z relro/norelro`。toolchain 固定 clang + LLD，artifact template 独占输出类型和安全链接参数，board config 只追加 ISA/ABI 参数与 kernel linker script 地址配置。

### 10.3 GNU script 与 LLD

处理原则：

- 统一目标是所有板都由 clang 驱动 LLD；kernel `link.x` 必须逐板改成 LLD-compatible，并在 CI 实际链接、启动。
- 尚未通过的 GNU script 可以在迁移期保留原链接器 fallback，但该板标记为“工具链未统一”，不能让 fallback 重新进入 target 自定义参数。
- dynamic app/DSO 不复用 kernel `link.x`；使用独立、简单、标准的 ET_DYN 布局。
- 首先用 `thumbv7m-vivo-blueos-newlibeabi` 打通“同一 target/sysroot 构建 kernel + app + libc.so.1”，冻结 ELF32/`EM_ARM`/Thumb-only/soft-float EABI 契约；然后扩展 Thumb v8-M hard-float、RISC-V64、AArch64 和 RISC-V32。
- LLD 迁移必须验证 MEMORY region、ENTRY/vector、`AT()`、NOLOAD、copy/zero table、KEEP/SORT 和板级导出符号；每个 target 在 CI 实际链接和启动，不凭脚本语法推测兼容。

### 10.4 共享 PIC sysroot 与回归门禁

每个 ISA/ABI 只生成一份 PIC sysroot：

```text
blueos-sdk/<abi>/rustlib/
  core
  alloc
  compiler_builtins
```

内核、应用和 DSO 共同使用这份 sysroot；静态并入 `libc.so.1` 的 `blueos_scal_swi`、spin 等依赖同样必须 PIC。禁止同一 ABI 同时发布 static/PIC 两套 `core/alloc`，避免依赖图因选择错误 sysroot 产生不一致代码模型。

统一 PIC 会让内核也承担 GOT/间接寻址、寄存器压力和潜在 text/data 增量，最终静态链接不会自动把 PIC 恢复为 non-PIC。每个板必须保留迁移前后的 A/B 构建，比较：

- kernel ELF 的 text/rodata/data/bss/GOT 和最终固件尺寸；
- boot 时间、调度/中断/syscall/allocator 等关键路径；
- map 中的 GOT/PLT、dynamic section、未定义符号和 relocation；
- RAM/Flash 布局、vector/entry、copy/zero table 与启动测试。

`kernel_static` 门禁要求无 `PT_INTERP`、`DT_NEEDED` 和待运行时处理的 dynamic relocation；`dynamic_app` 要求 `ET_DYN + PT_DYNAMIC + DT_NEEDED` 且无 `PT_INTERP`；`dso` 要求 SONAME、导出清单和 relocation 白名单。ARM32 基线 `PltGotOnly` profile 还必须确认外部函数调用已经收敛为 PLT/GOT，不允许 `R_ARM_THM_CALL/R_ARM_THM_JUMP24` 等动态 text relocation；只有显式声明并通过第 11.15 节门禁的 `LoaderVeneerV1` profile 才可接受。若某个资源受限板的 PIC 回归超过预算，应把它作为统一工具链决策的显式例外评审，而不是悄悄恢复 target 级 `-Crelocation-model=static`。

## 11. 按运行时阶段划分核心数据类型

### 11.1 阶段边界与总转换关系

一次动态应用启动固定经过 `S0`–`S11`。阶段号描述的是**一笔加载事务的状态**，不是代码目录，也不是 Phase 0–3 的项目排期：

```text
ArtifactRequest
  S0 admit
    → AdmittedArtifact<R>
  S1 inspect + plan
    → PlannedArtifact<R>
  S2 reserve
    → ReservedImage<R>
  S3 copy + zero
    → MappedImage
  S4 decode runtime metadata
    → RuntimeImage
  S5 dependency closure
    → BuildingSession（依赖图闭合）
  S6 freeze scopes
    → ScopedSession
  S7 relocate
    → RelocatedSession
  S8 seal
    → SealedSession
  S9 commit + publish
    → LinkProduct / LinkContext（Initializing）
  S10 initialize
    → ThreadGroup：Loading → Active；ApplicationState：Loading → Running
  S11 enter + exit + reap
    → ThreadGroup：Active → Draining → Reaped
    → ApplicationState：Running → Stopping → Terminated/Failed
```

类型设计遵循四条规则：

1. 阶段转换方法消费上一个阶段对象并返回下一个阶段对象，不提供公开的 `set_state()`。
2. 只在会改变合法 API 集合的边界使用 typestate：映像使用 `MappedImage/RuntimeImage/RelocatedImage/SealedImage`，链接事务使用 `BuildingSession/ScopedSession/RelocatedSession/SealedSession`。解析、BFS 队列等局部步骤使用普通内部状态，避免泛型爆炸。
3. `S0`–`S8` 的 allocation、临时 system DSO permit/lease 和未发布元数据统一归 `LinkSessionData` 的 rollback log 所有；阶段转换失败即随旧对象 `Drop`。
4. `S9` 是资源所有权和可见性提交点，`S10` 是不可自动回滚副作用边界。`LinkSession` 不执行 ctor，`ApplicationManager` 不持全局锁执行 ctor/fini 或应用入口。

所有阶段输出类型的字段和直接构造函数默认是 private 或 `pub(crate)`；调用方只能通过只读 accessor 和本节列出的转换方法使用它们。下面只有 `ArtifactRequest` 等纯输入值允许公开字段，不能通过 struct literal 伪造 `AdmittedArtifact`、`MappedImage`、`ScopedSession`、`SealedImage` 或 `LinkProduct`。

阶段与工程交付的对应关系如下：

| 运行时阶段 | 核心输出类型 | 主要所有者 | 首次实现所在工程阶段 |
| --- | --- | --- | --- |
| S0 准入 | `AdmittedArtifact<R>` | `ImageLoader` | Phase 0 |
| S1 预扫描/规划 | `ParsedImage`、`ImageLayout`、`PlannedArtifact<R>` | `ImageLoader` | Phase 0 |
| S2 reserve | `ImageAllocation`、`ReservedImage<R>` | `ImageLoader`/`ImageMemory` | Phase 0 |
| S3 copy/zero | `MappedImage` | `ImageLoader` | Phase 0 |
| S4 runtime metadata | `RuntimeImageMetadata`、`RuntimeImage` | `ImageLoader` | Phase 0.5；static PIE 子集在 Phase 0 |
| S5 依赖发现 | `DependencyGraph`、闭合的 `BuildingSession` | `DynamicLinker` | Phase 0.5 |
| S6 scope | `ScopeSet`、`ScopedSession` | `DynamicLinker` | Phase 0.5 |
| S7 relocation | `RelocatedImage`、`RelocatedSession` | `DynamicLinker`/`ArchRelocator` | Phase 0 relative 子集；Phase 0.5 完整 MVP |
| S8 seal | `SealPlan`、`SealedImage`、`SealedSession` | `ImageMemory`/`CodeCache` | Phase 0 接口；Phase 1 平台落地 |
| S9 publish | `LinkProduct`、`LinkContext`、`LinkMapEntry` | `ApplicationLoader`/registry | Phase 0.5 host；Phase 1 内核落地 |
| S10 initialize | `InitPlan`、`FiniPlan`、`ApplicationStartStorage` | `ApplicationManager`/`librs` | Phase 1 |
| S11 enter/exit/reap | `ThreadGroup`、`ThreadGroupMembership`、`ReapResources` | `ApplicationManager`/reaper | Phase 1 |

### 11.2 S0：artifact 准入

S0 只确认输入来源、基础 ELF 身份、ABI 与配额，不分配目标内存。核心类型：

```rust
pub struct ArtifactRequest {
    pub path: String,
    pub expected_model: ExecutionModel,
    pub domain: LinkDomainId,
    pub quota: LoadLimits,
}

pub struct ArtifactIdentity {
    pub canonical_path: String,
    pub file_id: FileIdentity,
    pub file_len: u64,
    pub build_id: Option<BuildId>,
    pub abi_note: Option<BlueOsAbiNote>,
}

pub struct AdmittedArtifact<R> {
    identity: ArtifactIdentity,
    reader: R,
    profile: ArtifactProfile,
    limits: LoadLimits,
}
```

需要实现：

- `ArtifactPolicy::validate_source(request, reader)`：可信路径/包归属、信任证据（内置、开发 allowlist 或有效产品签名）和允许的执行模型；产品构建拒绝开发 allowlist。
- `ArtifactPolicy::validate_identity(header, abi_note)`：ELF magic、class、endianness、machine、ABI note 和 artifact profile。
- `LoadLimits::check_file_len/check_image_count/check_total_bytes`。
- `ImageLoader::admit(reader, request, policy) -> Result<AdmittedArtifact<R>, LoadError>`。

输出不含 `AllocationId`；`LoadError.stage` 只能是 `Read/Parse/Validate`。S0 的单测使用截断 header、错误 machine/class/endianness、超限文件和 ABI note 不匹配输入，并断言 memory backend 从未被调用。

### 11.3 S1：预扫描与完整布局规划

S1 将借用型 parser 结果转成自有元数据，并在分配前证明所有范围有界：

```rust
pub struct PlannedArtifact<R> {
    artifact: AdmittedArtifact<R>,
    parsed: ParsedImage,
    layout: ImageLayout,
}
```

复用第 5 节的 `ParsedImage/DynamicInfo/ImageLayout/SegmentLayout`，另实现：

- `ImageLoader::inspect(&AdmittedArtifact<R>, policy) -> ParsedImage`。
- `ImageLayoutBuilder::build(&ParsedImage, &LoadLimits) -> ImageLayout`。
- `ImageLayout::allocation_request()`：生成 S2 唯一允许使用的 `AllocationRequest`。
- `ImageLayout::load_bias_for(mapped_base)`：校验最大 `p_align` 后计算显式 load bias。
- `ImageLayout::locate_vaddr_range(vaddr, len, perms)`：供后续阶段复用同一套 range 判定。

这一阶段拒绝 `p_filesz > p_memsz`、非法对齐、file/vaddr 溢出、冲突重叠、W+X、非 X entry 和资源超限。输出仍未持有 allocation，可安全、廉价地丢弃。

### 11.4 S2：reserve/allocate

S2 只决定目标存储位置和 load bias，不复制文件字节：

```rust
pub struct ReservedImage<R> {
    artifact: AdmittedArtifact<R>,
    parsed: ParsedImage,
    layout: ImageLayout,
    allocation: ImageAllocation,
    load_bias: TargetAddr,
}
```

`ImageAllocation` 至少保存 `AllocationId`、实际 target base、长度、对齐和 backend capability；不暴露 host pointer。需要实现：

- `ImageMemory::allocate_image(&AllocationRequest) -> ImageAllocation`。
- `ImageLoader::reserve(planned, memory) -> ReservedImage<R>`。
- `ImageAllocation::validate_layout(&ImageLayout)`：实际 base、长度和对齐必须满足已规划布局。
- `RollbackLog::push_release_allocation(id)`：分配成功后立即登记，后续任何错误都能逆序释放。

阶段门禁包括 allocator 故障注入、错误对齐 backend、连续区不足和多映像总配额；此时映像仍不可被 registry、link map 或运行线程发现。

### 11.5 S3：copy/zero，形成 `MappedImage`

S3 将每个 `PT_LOAD` 的 file bytes 写入计划位置，再显式清零 BSS 和需要归零的 gap：

```rust
pub struct LoadedRegion {
    allocation: AllocationId,
    target_range: TargetRange,
    file_range: Option<FileRange>,
    logical_permissions: MemoryPermissions,
}

pub type MappedImage = LoadedImage<MappedState>;
```

需要实现：

- `ImageLoader::copy_and_zero(reserved, memory) -> MappedImage`。
- `ElfReader::read_exact_at` 的分块读取路径，禁止依赖共享 file offset。
- `ImageMemory::write/zero` 的 allocation+offset 边界检查。
- `MappedImage::regions()` 和共有的 `target_addr/locate/contains_executable`。

复制顺序不能改变最终语义；重叠 segment 只有在 S1 证明内容和权限兼容后才允许。测试必须用预填充非零的 memory backend 验证 BSS 确实被清零，并在每个 segment/每个 read chunk 注入失败验证 allocation 回收。

### 11.6 S4：运行时元数据归一化

S4 不再保留“dynamic tag 还是未经验证的 ELF vaddr”这种歧义，而是生成 relocation、符号和生命周期阶段直接消费的只读元数据：

```rust
pub struct RuntimeImageMetadata {
    dynamic: Option<RuntimeDynamicInfo>,
    symbols: SymbolTable,
    relocations: RelocationTables,
    lifecycle: ImageLifecycleMetadata,
    program_headers: ProgramHeaderRuntimeInfo,
}

pub struct ImageLifecycleMetadata {
    preinit_array: Option<FunctionArray>,
    init: Option<TargetAddr>,
    init_array: Option<FunctionArray>,
    fini: Option<TargetAddr>,
    fini_array: Option<FunctionArray>,
}

pub type RuntimeImage = LoadedImage<RuntimeMetadataState>;
```

需要实现：

- `MappedImage::decode_runtime(policy) -> RuntimeImage`。
- `MappedImage::runtime_range(table_range)`：应用 load bias，并要求整个表落在可读 region。
- `SymbolTable::from_dynamic/lookup_index/lookup_name`，支持 GNU/SysV hash 的统一查询入口。
- `RelocationTables::decode_rel/decode_rela/decode_relr/decode_jmprel`，只解码成架构无关记录，不在此阶段写内存。
- `ImageLifecycleMetadata::validate_ranges(image)`：直接 `DT_INIT/DT_FINI` 地址位于 X region，array 的位置、长度、目标字宽对齐和条目上限有效；数组元素要等 relocation 完成后在 S9 读取并验证。
- `ApplicationArtifactPolicy::validate_runtime_features`：集中拒绝 `PT_INTERP/PT_TLS/RPATH/RUNPATH/TEXTREL/version/IFUNC/COPY` 等首版不支持项。

Phase 0 只需让 static PIE 获得 relative relocation 所需的最小 `RuntimeImageMetadata`；dynsym/hash/needed/init/fini 的完整实现属于 Phase 0.5。

### 11.7 S5：依赖闭包与稳定身份去重

S5 从 root 的 `DT_NEEDED` 开始 BFS，装入所有 app-private DSO，获取或等待 system DSO，并明确记录所有权边：

```rust
pub struct DependencyKey {
    pub soname: Option<String>,
    pub file_id: Option<FileIdentity>,
    pub build_id: Option<BuildId>,
}

pub struct DependencyNode {
    pub image: ImageId,
    pub identity: DependencyKey,
    pub ownership: ImageOwnership,
    pub discovery_index: u32,
}

pub struct DependencyEdge {
    pub requester: ImageId,
    pub provider: ImageId,
    pub needed_index: u32,
}

pub struct DiscoveryQueue {
    queue: VecDeque<DependencyRequest>,
    depth: BTreeMap<ImageId, u16>,
}
```

需要实现：

- `BuildingSession::load_root()` 和 `expand_dependencies_bfs()`。
- `ArtifactResolver::resolve_needed(requester, soname) -> ResolvedArtifact`，返回固定 system path 或应用包 `lib/` 身份，不搜索环境变量/当前目录。
- `DependencyGraph::add_root/add_image/add_edge/find_by_soname/find_by_file_identity`。
- `DependencyGraph::strongly_connected_components` 及依赖数量、深度、总字节配额。
- `SystemDsoRegistry::acquire_or_begin_load`：返回 Ready lease、唯一 loading permit 或 wait handle；permit/临时 lease 立即登记到 rollback log。
- `DependencyGraph::finish_discovery()`：检查 SONAME 与文件身份冲突，并冻结 discovery index。

阶段输出仍是 `BuildingSession`，但其 discovery queue 必须为空、每个 `DT_NEEDED` 必须有唯一 edge。不能把“同名”直接等同于“同一文件”，也不能让应用私有 DSO 覆盖 system DSO 身份。

### 11.8 S6：冻结符号作用域

S6 从已经闭合的图派生查找顺序；依赖图、符号 scope 和 init/fini 顺序分别建模：

```rust
pub struct ScopeSet {
    application: SymbolScope,
    system: SymbolScope,
}

pub struct SymbolScope {
    ordered_images: Vec<ImageId>,
    owner_domain: LinkDomainId,
}
```

需要实现：

- `BuildingSession::freeze_scopes(self) -> ScopedSession`。
- `DependencyGraph::bfs_scope(root)`：保持 `discovery_index` 的稳定顺序。
- `ScopeSet::for_relocation(owner)`：system image 只能获得 system scope；main/private DSO 获得 application scope。
- `SymbolScope::lookup_global/lookup_weak/resolve_for_relocation`。
- `SymbolTable::exports()` 与 visibility 过滤；hidden 不外露，protected 的对象内引用不被 interpose。
- `ScopeSet::validate_unique_strong_exports()`；undefined weak 产生显式 `ResolvedSymbol::Zero`，不是伪造 owner。

`ScopedSession` 创建后不允许再增加 image 或修改依赖 edge；若发现新依赖，必须在 S5 失败并回滚，不能在 relocation 中递归扩图。

### 11.9 S7：重定位，形成 `RelocatedSession`

S7 的输入只能是冻结 scope 和 `RuntimeImage`。架构无关层负责顺序、符号与范围，架构层负责 relocation 语义：

```rust
pub struct RelocationRecord {
    pub target_vaddr: TargetAddr,
    pub kind: RelocationKind,
    pub symbol_index: Option<u32>,
    pub explicit_addend: Option<i64>,
}

pub struct RelocationRequest {
    pub image: ImageId,
    pub location: TargetLocation,
    pub base: TargetAddr,
    pub symbol: Option<ResolvedSymbol>,
    pub addend: i64,
    pub kind: RelocationKind,
}

pub struct RelocationPolicy {
    pub allowed_kinds: RelocationKindSet,
    pub require_owner_writable_target: bool,
    pub require_symbol_in_frozen_scope: bool,
    pub require_control_target_executable: bool,
}
```

需要实现：

- `ScopedSession::relocate(self, arch, memory) -> RelocatedSession`。
- `RelocationEngine::relocate_relative/relocate_symbols/relocate_plt`，顺序固定且统计分开。
- `ArchRelocator::decode_relocation/read_rel_addend/apply`。
- `RuntimeImage::locate`：把 relocation vaddr 转成带 allocation、offset、width 的 `TargetLocation`。
- `SymbolScope::resolve_for_relocation` 和 `RuntimeImage::symbol_address`。
- `RelocationPolicy::validate_request`：校验 relocation 类型白名单、`P` 的 image owner/允许写入权限、`S` 的冻结 scope owner，以及控制流 relocation 的最终 X region；架构函数地址编码由 `ArchRelocator` 归一化后验证。
- `ImageMemory::read_word/write_word`：目标字宽、endianness、对齐和范围检查。
- `RuntimeImage::into_relocated()` 仅为 session 内部转换；外部不能跳过 relocation。

任何未知类型、溢出、缺失 strong symbol、越界/未对齐写、越权 symbol owner 或非 X 的控制流目标都使整笔 session 回滚；undefined weak=0 只有在 artifact profile 对相应 relocation 明确放行时例外。`R_*_RELATIVE` 的结果按当前 artifact profile 校验为允许的同映像目标；需要支持空值、one-past 或架构特殊编码时必须显式列入 profile，不能用“任意地址”兜底。Phase 0 以 ARM32 单映像 `R_ARM_RELATIVE` 打通 `DT_REL` 隐式 addend 读取；Phase 0.5/1 增加 `R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT`。首发 profile 之外的 relocation 全部拒绝。

这里验证的是 dynamic relocation 和生命周期元数据，不是完整 CFI：同映像直接分支、运行期函数指针、返回地址和应用自行计算的 `jalr` 不经过本阶段。

### 11.10 S8：权限与 cache 收口

S8 在全部写入完成后统一生成并执行 seal 计划：

```rust
pub struct SealPlan {
    protections: Vec<ProtectionRange>,
    relro: Vec<TargetRange>,
    executable_ranges: Vec<TargetRange>,
}

pub struct AppliedProtection {
    pub range: TargetRange,
    pub requested: MemoryPermissions,
    pub result: ProtectionResult,
}
```

需要实现：

- `RelocatedSession::seal(self, memory, cache) -> SealedSession`。
- `RelocatedImage::build_seal_plan()`：合并相邻同权限区间，RELRO 覆盖最终写权限；同时拒绝 W+X、TEXTREL、可执行栈和可执行区域的 writable alias，text 目标为 RX、rodata/RELRO 为 R、data/BSS 为 RW+NX。
- `ImageMemory::protect`：返回 `HardwareEnforced` 或 `LogicalOnly`，不得谎报硬件保护。
- `CodeCache::sync_instruction_cache`：覆盖所有新写入的 X range；ARM32 首发 `qemu_mps2_an385` 明确声明无硬件 cache，但仍执行 DSB/ISB 使代码写入与后续取指有序，不能用通用空 stub。cache-enabled Cortex-M 必须额外 clean D-cache、invalidate I-cache。
- `RelocatedImage::seal(...) -> SealedImage`。
- `SealedImage::build_link_map_entry()`：只有 sealed image 才能形成可发布条目。

没有可靠 cache backend 时本阶段失败；在 SMP 尚未实现远端 hart 同步前，运行时策略必须阻止其他 hart 执行新代码。

### 11.11 S9：原子提交与发布

S9 把私有 session 的资源所有权移动到已发布对象，并一次性更新 ThreadGroup/System DSO registry/link map：

```rust
pub struct LinkProduct {
    context: LinkContext,
    entry: TargetAddr,
    main_program_headers: ProgramHeaderRuntimeInfo,
    init_plan: InitPlan,
    fini_plan: FiniPlan,
    link_map: Vec<LinkMapEntry>,
    metrics: LoadMetrics,
    publication_guard: PublicationGuard,
}

pub struct PublicationBatch {
    images: Vec<SealedImage>,
    system_leases: Vec<SystemDsoLease>,
    link_map: Vec<LinkMapEntry>,
}
```

需要实现：

- `SealedSession::build_lifecycle_plans(memory)`：从已经重定位的 init/fini array 读取函数地址，逐项验证 owner 和 X region，再基于 SCC DAG 生成依赖优先 init 与精确逆序 fini。
- `SealedSession::commit(self) -> LinkProduct`：把 allocation/permit/lease 的回滚能力从 session 移入 `PublicationGuard`，session 自身不再回滚。
- `PublicationGuard::rollback()` 和 `disarm()`：前者按逆序撤销未发布资源，后者只能在 ThreadGroup/System DSO registry/link-map 全部接管成功后调用。
- `ApplicationLoader::publish_link_product(group, product)`：原子安装 context/link map/lease；成功后解除 `PublicationGuard`，失败则由 guard 完整回滚。
- `ThreadGroup::install_link_product(product)`：只允许资源阶段为 `Loading` 且尚无 link product 时调用；安装成功后仍为 `Loading`，并且只能由上述 publish 路径内部调用。
- `SystemDsoRegistry::publish_relocated(permit, instance)`：校验 generation，只唤醒同一 loading generation 的等待者。
- `LinkMapRegistry::publish_batch/unpublish_batch`：读者只能看到旧快照或完整新快照。

发布后 `LinkSession` 已完成 `Relocated/Sealed/Published`，但 `ThreadGroup` 和公共 `ApplicationState` 仍是 `Loading`，不是 `Running`。普通线程尚不能获得业务入口；提交或安装失败必须恢复 registry permit，并由 `LinkProduct::publication_guard` 完成回滚。

### 11.12 S10：初始化与副作用边界

S10 由拥有目标执行上下文的 runtime 执行，不由 `DynamicLinker` 执行：

```rust
pub struct LifecycleEntry {
    pub owner: ImageId,
    pub function: TargetAddr,
}

pub struct InitPlan(Box<[LifecycleEntry]>);
pub struct FiniPlan(Box<[LifecycleEntry]>);

pub struct InitializationFailure {
    pub owner: ImageId,
    pub completed: usize,
    pub reason: ApplicationFailureReason,
}
```

需要实现：

- `InitPlan::validate(&LinkContext)` 和 `FiniPlan::validate(&LinkContext)`：owner 匹配且函数地址位于对应 sealed image 的 X region。
- `ApplicationStartStorage::build(request, product)`：稳定保存 argv/envp/auxv/init/fini 指针目标。
- `ThreadGroup::record_init_progress/activate_after_init_complete`；后者只能由校验过 handle 和启动代次的 `ApplicationInitComplete` 事件触发。
- `SystemDsoRegistry::mark_initializing/mark_ready/fail_loading`，同一 generation 只 init 一次。
- `__librs_start_main` 顺序执行 init plan，随后调用 `ApplicationInitComplete`。

第一条 ctor 调用之前仍可撤销纯 loader 资源；一旦开始调用，只能将整个仍处于 `Loading` 的实例标记 `Failed` 并走受控退出，不能声称外部副作用已回滚。所有 ctor 调用都在 loader/global registry/ApplicationManager table 锁之外。

### 11.13 S11：进入、退出与最终回收

S11 负责把初始化完成的 ThreadGroup 变成运行实例，并在所有执行流和引用失效后回收：

```rust
pub struct ThreadGroupMembership {
    pub group_id: ThreadGroupId,
}

pub enum ApplicationEvent {
    ExecutionUnitCreated { handle: ApplicationHandle, id: ThreadId },
    ExecutionUnitExited { handle: ApplicationHandle, id: ThreadId, status: i32 },
    TerminateRequested { handle: ApplicationHandle, reason: ExitReason },
    ReapReady { handle: ApplicationHandle },
}

pub struct ReapResources {
    private_images: Vec<SealedImage>,
    system_leases: Vec<SystemDsoLease>,
    start_storage: Option<ApplicationStartStorage>,
    link_map: Vec<LinkMapEntry>,
}
```

需要实现：

- `ApplicationManager::on_execution_unit_created/on_execution_unit_exited/begin_exit/finish_exit/reap`。
- `ThreadGroup::register_thread/unregister_thread/begin_exit/can_run_fini/mark_fini_complete/can_reap`。
- `ThreadGroup::take_resources_for_reap() -> ReapResources`，只允许一次且只在 `can_reap()` 后调用。
- `FiniPlan::iter()`；计划本身已经是 init 的精确逆序，librs 在其他线程退出后按存储顺序执行 atexit 和 fini。
- `SystemDsoLease::release()` 只投递 quiescence；`SystemDsoRegistry::begin_quiescence/finish_unload/keep_cached` 由 reaper 执行。
- `LinkMapRegistry::unpublish_batch` 必须早于 allocation release，但晚于所有线程/回调停止。

`ReapResources` 是回收线程的独占所有权包，避免持有 `ApplicationManager`/ThreadGroup 锁逐项释放。正确顺序是：阻止新线程 → 等待执行单元退出 → fini/atexit/emutls destructor → unpublish link map → 释放 private image → 释放 system lease → ThreadGroup 进入 `Reaped`，公共状态进入 `Terminated` 或保留 `Failed` 终态。

### 11.14 跨阶段所有权与失败规则

| 资源 | 首次产生 | S0–S8 失败 | S9 后所有者 | 最终释放点 |
| --- | --- | --- | --- | --- |
| reader/file handle | S0 | `AdmittedArtifact`/session Drop | artifact catalog 或关闭 | S4 后可关闭；缓存 metadata 时由 catalog 管理 |
| image allocation | S2 | rollback log 逆序 `release` | `LinkContext`/system DSO instance | S11 reaper/quiescence |
| optional veneer allocation | S2 | rollback log 逆序 `release` | 产生调用点的 image instance | 与所属 image 一起在 S11/quiescence 释放 |
| loading permit | S5 | rollback 触发 `fail_loading` | S9 转成 Ready/Initializing registry entry | 成功后 permit 消失 |
| system DSO lease | S5 | rollback 释放临时 lease | `LinkContext`/`ThreadGroup` | S11 投递 quiescence |
| symbol scope/graph | S5–S6 | 随 session 丢弃 | `LinkContext` | domain/system instance 回收 |
| link map | S9 构建并发布 | S9 前只在 session 内 | link-map registry + `LinkContext` | S11 先 unpublish 后释放代码 |
| init/fini plan | S9 | S9 前随 session 丢弃 | `ThreadGroup`/system instance | fini 完成后随实例释放 |
| argv/envp/auxv storage | S10 | 启动失败随 domain 回收 | `ThreadGroup` | 最后线程退出后 |

对外 API 不返回 `&mut LoadedImageData`、裸 `AllocationId` 或可写的 `SessionDisposition`。调试和 metrics observer 只能观察事件副本，不能持有能跨阶段修改核心对象的引用。

### 11.15 ARM32 Thumb 远跳 veneer 的可选演进

这一能力是为未来兼容特殊 ARM32 工件预留的条件 profile，不改变当前 `ET_DYN + PLT/GOT + NOW` 主路径。Thumb `BL`/`B.W` 的限制来自调用点 `P` 到目标的有符号位移；`R_ARM_THM_CALL/R_ARM_THM_JUMP24` 对应的常见范围约为前后各 16 MiB，不能把它表述成“跨过 16 MiB 地址边界就必须使用 veneer”。正常跨 DSO 调用先到调用方附近的 PLT，再由 PLT 经 GOT 中的完整 32 位地址跳转，所以即使调用方在 Flash、被调 DSO 在 PSRAM，loader 通常也只需处理 `R_ARM_JUMP_SLOT`。

首发 `Arm32BranchMode::PltGotOnly` 保持 fail-closed：构建门禁和 S0/S4 均拒绝需要改写 text 指令的动态分支 relocation。仅当 `llvm-readelf/llvm-objdump` 对真实发布工件证明无法通过链接参数、PLT/GOT 或链接期 thunk 消除该类 relocation 时，才允许产品 profile 在 ABI note/manifest 中声明 `Arm32BranchMode::LoaderVeneerV1`。`__far_func_veneer`、`__Thumbv7ABSLongThunk_*` 等符号名是 GNU ld/LLD 的实现细节，不能作为识别或 ABI 契约；loader 只依据 relocation 类型、指令编码和已验证的目标 profile 工作。

`LoaderVeneerV1` 复用 S0–S11，但增加以下跨阶段约束：

1. **S0/S1 准入与预规划**：S0 先校验 ABI note/manifest 已声明该 capability；S1 只接受 profile 明确列出的 `R_ARM_THM_CALL/R_ARM_THM_JUMP24`，若以后需要 `R_ARM_THM_JUMP19`，必须作为独立能力再加入。S1 在符号尚未解析、load bias 尚未确定时，按映像内相对位置枚举调用点和可达约束，按可共同覆盖的调用点形成 `VeneerIslandPlan`，并按“每条分支 relocation 最坏占用一个 veneer”计算容量上界；S4 归一化时必须与预扫描结果交叉校验。到 S7 得到最终目标后，才以“目标地址 + ARM/Thumb 状态 + 跳转语义”为 key 在同一 island 内去重；不同 island 即使目标相同也可能各需一份。
2. **S2 近地址预留**：`ImageMemory` 增加可选的 `reserve_executable_near(caller_range, size, alignment, reach)` capability，按 `VeneerIslandPlan` 一次性预留 branch island；单一全局 pool 不足以覆盖跨度较大的 RX 区域。无法满足距离、对齐、数量或内存配额时，在复制和发布前失败；禁止 S7 遇到越界后临时分配。veneer allocation 从创建起登记到 rollback log，并由调用方映像拥有。
3. **S7 解析与修补**：解析并校验最终符号地址 `S` 及 Thumb bit；若 `S + A` 对调用点可达，直接编码原分支；否则选择对 `P` 可达且对目标匹配的 veneer，把分支改为 veneer 地址。veneer 使用能承载完整 32 位目标的架构模板，例如 literal load 到 `PC`，或 `MOVW/MOVT + BX`；模板选择由板级执行/读取属性决定，不能假定所有 X region 都可读。修补前同时验证调用点属于当前映像允许写入的 text staging 区、veneer 落在已预留 allocation、最终目标属于冻结 scope 的 X region。
4. **S8 seal 与 cache**：调用指令和新生成 veneer 都只允许在尚未发布的私有 staging 映射中写入；随后对涉及的 cache line 执行 D-cache clean、I-cache invalidate、DSB/ISB，再把 veneer pool 封为 RX。任何时刻都不能向其他执行流发布 W+X 或 writable alias。S9 以后 veneer 进入 link map，并与所属映像一起保活、回滚和回收。

Flash/XIP 是硬边界：调用指令若位于运行期不可写 Flash，loader 不能靠 RAM 中有一个 veneer 就解决问题，因为它仍需先修改原分支。此时只允许静态链接器在 Flash 中提前放置可达 thunk、让调用方使用本地 PLT/可写 RAM GOT、改用间接调用表，或拒绝工件。未来若支持“Flash RX/RO + PSRAM/RAM RW”的 split-segment backend，还必须由 `ImageMemory` 明确报告 callsite writable、near-allocation 和 execute/read 属性，不能沿用连续 RAM `FlatImageMemory` 的假设。

该能力至少通过以下 gate 后才能启用：分支范围边界内/外和正反向 golden fixture；Flash→PSRAM 与 PSRAM→Flash；跨度需要多个 island 且同目标跨 island 复制；Thumb bit/非 X 目标/错误指令编码；near-allocation OOM 与配额；不可写 callsite；rollback/reload；带 cache 板上的逐行 cache 同步和 RX seal。没有命中这些真实产物和板级需求时只保留接口与文档，不实现运行时代码。

## 12. 分阶段实施计划

### 12.1 Phase 0：可信单映像 loader

#### 实现目标

修复现有 loader 的正确性和安全问题，但暂不装载 `DT_NEEDED`。

#### 阶段类型交付

- 覆盖 S0–S3，交付 `AdmittedArtifact → PlannedArtifact → ReservedImage → MappedImage` 完整转换。
- 交付 S4 的 static PIE 子集：`RuntimeImageMetadata` 至少保存 relative relocation 表。
- 交付 S7/S8 的单映像子集：`RuntimeImage → RelocatedImage → SealedImage`；不建立依赖图和符号 scope。
- `load_elf()` 只作为兼容包装，内部必须走上述阶段类型，不能保留第二套 parse/map/relocate 实现。
- 对应建议提交 `C01`–`C10`。

#### 主要工作

1. 引入 `TargetAddr/TargetRange/LoadError/LoadLimits`。
2. 实现 `SliceElfReader` 和 `ElfReader`。
3. 将 goblin 借用型结果转换成自有 `ParsedImage`，不在 `LoadedImage` 中保存 `Elf<'a>`。
4. 实现 `ImageLayoutBuilder`。
5. 修正非零最低 vaddr 的 load bias、BSS 显式清零、segment gap、最大 `p_align`、entry X 权限和 checked arithmetic。
6. 实现 `ImageMemory` 和 `MemoryMapper` 兼容 adapter。
7. 解析 REL/RELA/RELR 元数据；当前 static PIE 只执行允许的 relative relocation。
8. 未知 relocation fail closed。
9. 增加逻辑权限、RELRO hook 和 cache 接口。
10. `load_elf()` 改为调用 `ImageLoader`。
11. 保留 ET_EXEC fixed mapper 测试。

#### 验收门禁

必须覆盖：

- 最低 `p_vaddr != 0`
- segment 中间有洞
- 大 BSS 且预填充非零内存
- `p_filesz > p_memsz`
- 非法 `p_align`
- offset/vaddr 溢出
- W+X
- entry 不在 X segment
- relocation 越界/未对齐
- 未知 relocation
- 任意步骤失败后 allocation 全部释放

ARM32 `R_ARM_RELATIVE`/`DT_REL` host fixture 必须通过；现有 RISC-V64 static PIE、ET_EXEC 和 QEMU 测试作为兼容回归也必须继续通过，但不再决定 Phase 1 的首发架构。

### 12.2 Phase 0.5：冻结 `DynamicLinker` 架构

#### 实现目标

在同一 `blueos_loader` crate 中形成可单测的多 DSO 核心，但暂不接入完整内核启动。

#### 阶段类型交付

- 补全 S4 的 `RuntimeImageMetadata/SymbolTable/RelocationTables/ImageLifecycleMetadata`。
- 实现 S5/S6 的 `DependencyGraph/DiscoveryQueue/ScopeSet` 和 `BuildingSession → ScopedSession`。
- 实现 S7–S9 的 `RelocatedSession → SealedSession → LinkProduct`，并让 `LinkSessionData` 统一拥有 rollback log。
- host backend 中的 S9 只验证原子 publication snapshot；真正的内核 registry/domain publish 在 Phase 1 接入。
- 对应建议提交 `C11`–`C17`。

#### 主要工作

1. 增加 `DependencyGraph`、`SymbolScope`、`SymbolTable`。
2. 增加 `LinkSession/LinkContext/LinkProduct`。
3. 实现事务回滚状态机。
4. 解析 `PT_DYNAMIC`、`DT_NEEDED`、`DT_SONAME`、dynstr/dynsym、GNU/SysV hash 和 init/fini。
5. 实现 `ApplicationArtifactPolicy`。
6. 冻结拒绝项：PT_INTERP、PT_TLS、RPATH/RUNPATH、lazy binding、IFUNC/IRELATIVE、COPY、TEXTREL、symbol version。
7. 实现通用 relocation engine 与 ARM32 backend：先支持 `DT_REL` 和 `R_ARM_RELATIVE/R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT`，集中处理 Thumb bit，拒绝 `COPY/IRELATIVE` 及白名单外类型。
8. 用人工 ELF fixture 测试依赖图、符号绑定和构造顺序。
9. 固定 Relink revision，仅作为 host differential oracle。

#### 验收门禁

- 同一 fixture 的依赖 BFS、symbol owner、relocation 写值和 init/fini 顺序与 oracle 一致。
- 任意一步故障注入后 session 回到零资源状态。
- `DynamicLinker` 不依赖 VFS、kernel allocator、`ApplicationManager` 或 librs。
- `blueos_loader` 生产依赖中不存在 `//librs`。

### 12.3 Phase 1：ARM32 Thumb v7-M `app → libc.so.1` MVP

#### 实现目标

打通第一条完整动态应用启动和退出链路。

#### 阶段类型交付

- 将 S0–S9 接到 `VfsElfReader/FlatImageMemory/arch/*/cache.rs 服务/ApplicationArtifactResolver/SystemDsoRegistry`。
- 实现 S9 的内核所有权落点：`LinkProduct → ThreadGroup::install_link_product()`。
- 实现 S10 的 `InitPlan/FiniPlan/ApplicationStartStorage/LibcApplicationContext`。
- 实现 S11 的 `ThreadGroupMembership/ApplicationEventQueue/ReapResources/SystemDsoLease`。
- 对应建议提交 `C18`–`C29`，以 `C29` 的 QEMU 纵向测试作为 Phase 1 唯一完成门槛。

#### 1A：构建产物

1. 将已有 PIC/dynamic 能力的 `thumbv7m-vivo-blueos-newlibeabi` 从 `arm-none-eabi-gcc` 链接路径收敛为 clang + LLD；首发 ABI 固定为 32 位小端、`EM_ARM`、Thumb-only、soft-float EABI。
2. 生成由 kernel、app 和 DSO 共用的 PIC `core/alloc/compiler_builtins` sysroot。
3. 建立 `kernel_static`、`dynamic_app` 和 `dso` 三种 artifact link policy，并验证 PIC 内核仍按原地址和入口启动。
4. 将 `librs_swi` 生成 `libc.so.1`。
5. 添加 SONAME、`librs.exports`、`.note.blueos.abi` 和 build-id；链接产物必须用 `PT_NOTE` 携带该 ABI note，note/manifest 明确记录 target profile、Thumb-only 与 soft-float EABI，运行时不依赖 `.ARM.attributes` section 存在。
6. 验证 libc 没有未定义的内核普通符号。
7. 生成 `blueos_scrt1.o`。
8. 编译 Thumb C hello PIE，确保它是 `ELFCLASS32 + EM_ARM + ET_DYN`、有 `PT_DYNAMIC`、无 `PT_INTERP`，包含 `DT_NEEDED: libc.so.1`，并且只产生 ARM32 首发 relocation 白名单。

#### 1B：内核 backend

实现：

- `VfsElfReader`
- `FlatImageMemory`
- ARM `arch/*/cache.rs` 指令缓存服务（`sync_instruction_cache`）
- `SystemLibraryPaths`
- `ApplicationArtifactResolver`
- `SystemDsoRegistry`
- `SystemDsoLease`

`qemu_mps2_an385`/Cortex-M3 没有硬件 cache，backend 仍必须显式返回该能力并在发布前执行 DSB/ISB；不能把空实现当作通用 ARM backend。迁移到带 cache 的 Cortex-M7/M55 等平台时，必须按芯片 cache line 和共享属性补齐 D-cache clean、I-cache invalidate、DSB/ISB，未实现的板拒绝执行新代码。

#### 1C：`ApplicationManager` 与 `ThreadGroupBackend`

1. 实现单一 `ApplicationManager`、`ApplicationHandle/ApplicationLaunchRequest/ApplicationState` 公共控制面，以及当前的 `ThreadGroupBackend/ApplicationLoader/ThreadGroup`。
2. manifest/ABI note 增加稳定 `ExecutionModel`；当前只接受 `ThreadGroup`，识别到 `Process` 返回 `UnsupportedExecutionModel`。
3. 由唯一内核配置生成 `blueos_user_process`；Phase 1 固定关闭，`application` 不创建空壳进程对象。
4. `Thread` 增加 membership。
5. `CreateThread` 继承 membership。
6. 增加应用生命周期 syscall：`ApplicationLaunch/ApplicationInitComplete/ApplicationBeginExit/ApplicationFinishExit`；`ApplicationLaunch` 先做 argv/envp copy-in 与 S0 准入。
7. 构造 `ApplicationStartStorage`。
8. shell 改为动态应用（PIE + `DT_NEEDED: libc.so.1`）并增加 `run` 命令：经 librs `spawn` API 发 `ApplicationLaunch` syscall；boot 自举由内核内部直呼 `ApplicationManager::launch()` 装载 shell。
9. 增加 deferred reaper，禁止 scheduler cleanup 直接卸载。

#### 1D：librs 启动

1. 实现新版动态 `__librs_start_main`。
2. 增加 `LibcApplicationContext`。
3. `getauxval()` 从当前线程 LibcApplicationContext 查询。
4. 子 pthread 继承 LibcApplicationContext。
5. emutls 在线程退出时完成 destructor。
6. 应用 init/fini 在 loader lock 外运行。
7. 首次 libc init 后 registry 进入 Ready。
8. 后续应用复用同一 libc instance。
9. 新增 `spawn(path, argv, envp)` API：内部发 `ApplicationLaunch` syscall，返回应用句柄；随 `librs.exports` 导出。

#### 验收门禁

至少运行：

- C hello
- argc/argv/envp
- `getauxval`
- heap
- 文件 IO
- BSS
- weak undefined
- app 到 libc 的 function/data symbol
- ctor 恰好一次
- 多线程 emutls 值独立
- 两应用并发复用同一 libc
- `spawn` API：经 `ApplicationLaunch` syscall 启动应用并返回有效 `ApplicationHandle`，argv/envp 正确传递
- 缺 libc、缺 symbol、ABI note 不匹配
- app 退出前 image 不被释放
- 最后 lease 释放后受控 quiescence、卸载、再次加载

完整 Rust `std` 应用不阻塞这一阶段；先支持最小 Rust C ABI fixture。

### 12.4 Phase 2：多 DSO 和多架构

#### 实现目标

支持：

```text
app.elf
  ├─ libfoo.so
  │    └─ libc.so.1
  └─ libbar.so
       └─ libc.so.1
```

#### 阶段类型扩展

- S5 增加 app-private resolver、SONAME/file identity 双去重和跨多个私有 DSO 的 SCC。
- S6 完成 weak/hidden/protected 语义及 application/system 双 scope 隔离。
- S7/S8 只增加架构 backend，不复制 `DynamicLinker`、`RelocationEngine` 或 seal 流程。
- S10/S11 验证多 DSO init/fini 与整组回收，仍不增加单 DSO `dlclose` 状态。
- 对应建议提交 `C30`–`C35`；若真实 ARM32 工件触发第 11.15 节条件，再追加可选 `C32V`，它不属于 Phase 2 基线完成门槛。

#### ARM32 多 DSO

1. 应用包 `lib/` resolver。
2. SONAME/file identity 双重去重。
3. app-private DSO 归 ThreadGroup 所有。
4. system DSO 只在 system scope 解析。
5. 菱形依赖和循环依赖。
6. private DSO ctor/fini。
7. weak/hidden/protected。
8. 多 DSO emutls control object。
9. 整组回收，不做单 DSO dlclose。

#### 目标 profile 顺序

1. ARM32 Thumb v7-M soft-float（Phase 1 基线，Phase 2 先完成多 DSO）
2. ARM32 Thumb v8-M hard-float
3. RISC-V64
4. AArch64
5. RISC-V32 IMAC/IMC

每个 backend 实现：

| 架构 | relocation | 额外工作 |
| --- | --- | --- |
| ARM32 | 基线：`R_ARM_RELATIVE/R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT`；可选 veneer profile：`R_ARM_THM_CALL/R_ARM_THM_JUMP24` | ELF32、REL addend、Thumb bit、soft/hard-float ABI、DSB/ISB 与按板 cache 维护；可选分支类型必须预规划 island，不能在重定位途中临时分配 |
| RISC-V64 | `R_RISCV_RELATIVE/R_RISCV_64/R_RISCV_JUMP_SLOT` | fence.i、LP64/ISA note |
| AArch64 | `R_AARCH64_RELATIVE/R_AARCH64_ABS64/R_AARCH64_GLOB_DAT/R_AARCH64_JUMP_SLOT` | D-cache clean、I-cache invalidate、页权限 |
| RISC-V32 | RELATIVE、R_RISCV_32、JUMP_SLOT | ILP32、ISA feature、无 A 扩展同步路径 |

#### 验收门禁

同一 app/libfoo/libc fixture 在所有平台必须得到相同的：

- 依赖图
- symbol owner
- ctor/fini 顺序
- weak symbol 结果
- ThreadGroup 回收结果

并为每个允许 relocation 提供精确 golden fixture。

`LoaderVeneerV1` 若启用，还必须通过第 11.15 节的远距、branch island、不可写 Flash、资源耗尽、cache 和回滚门禁；未启用时仍应有反向 fixture，证明分支动态 relocation 被稳定拒绝。

### 12.5 Phase 3：运行期装载与产品化

#### 实现目标

在 Phase 2 多 DSO 基线上提供受限但闭环的插件装载 API，并将“能运行”提升为“可发布、可升级、可诊断、可安全失败”。本 Phase 明确交付 `dlopen/dlsym/dlerror/dlclose`，但 `dlclose` 的基线语义是逻辑关闭和 ThreadGroup 退出时延迟回收，不承诺运行期立即释放 RAM。

#### 阶段类型加固

- 为每个 ThreadGroup 增加 `RuntimeLinkState/RuntimeDsoHandleTable`，启动 `LinkContext` 作为不可变基础，运行期 group 通过带 generation 的快照增量发布。
- 增加 `RuntimeLoadSession/RuntimeDsoState`，复用 S2–S9 的 map、relocate、seal、cache sync、rollback 和 publish 规则；失败只撤销本次增量。
- `RuntimeDsoState` 经过 `Loading → Initializing → Ready → LogicallyClosed`；constructor 在 loader/registry 锁外执行，初始化完成前普通 waiter 和 `dlsym` 不得观察半成品。
- `RuntimeDsoHandle { slot, generation }` 只属于当前 ThreadGroup；重复打开增加 open count，最后一次逻辑关闭使 handle 失效，但映像、emutls 描述符和已返回地址继续保活到 S11 整组回收。
- S0 为 `ArtifactIdentity` 增加签名、hash、manifest、allowlist 和 ABI generation。
- S1/S4 固化产品 `ArtifactPolicy`，构建期与加载期双重拒绝 W+X、TEXTREL、可执行栈和未支持 dynamic feature。
- S7 固化 `RelocationPolicy` 审计记录，证明每个 `P` 的 owner/range 与每个 `S` 的 scope/control-target 判定。
- S8 发布 `AppliedProtection`，准确区分逻辑权限与平台实际执行的权限，不能把 `LogicalOnly` 升格为隔离证据。
- S2–S9 为 allocation/permit/lease/publication 增加全覆盖故障注入和 metrics。
- S9 为 `LinkMapEntry` 增加 build-id、精确 segment 和崩溃符号化信息。
- S11 为 `ReapResources` 增加 quiescence evidence，不能证明安全时把 system instance 转为 cached 而不是释放。
- 对应建议提交 `C36`–`C42`。

#### 主要工作

1. `DynamicLinker` 增加 runtime delta/group scope，支持 `RTLD_NOW | RTLD_LOCAL` 与 `RTLD_NODELETE`，拒绝 lazy/deepbind；保持 system resolver 与应用 scope 隔离。
2. 内核增加 per-ThreadGroup runtime registry、load permit、generation handle 和 `DynamicLibraryPrepareOpen/NextInitializer/InitComplete/Lookup/Close` syscall。
3. `librs` 导出 `dlopen/dlsym/dlerror/dlclose`；在 loader lock 外执行 constructor，在线程局部状态保存 `dlerror`，并覆盖 constructor 嵌套 `dlopen`。
4. 同一工件的并发首开只装载/初始化一次；重复打开增加计数。`dlclose` 计数归零后只使 handle 失效，映像在 ThreadGroup 退出时统一 fini/reap。
5. local 基线稳定后，可增加 `RTLD_GLOBAL/RTLD_NOLOAD/RTLD_DEFAULT` 与 `dladdr/dl_iterate_phdr`；全局 scope 只影响提交后的未来装载，不反向重绑定既有 GOT，仍拒绝 `RTLD_NEXT`。
6. 应用包 manifest、签名、hash 和 allowlist。
7. ABI note、syscall ABI、librs ABI append-only。
8. SONAME major、导出 ABI diff，以及系统 DSO A/B 升级与回滚兼容检查。
9. registry 并发首装、失败重试、generation、OOM、构造失败、依赖环和全路径故障注入。
10. allocator/stdio/cwd/environment/atexit 的 `LibcApplicationContext` ownership 审计；callback、timer、signal、函数指针、TLS destructor 的 quiescence tracking。
11. 无法证明静默时保留 system DSO 缓存，不卸载；应用运行期 DSO 无论是否显式 `RTLD_NODELETE` 都采用 ThreadGroup 延迟回收。
12. Artifact/relocation/seal policy 的构建门禁、加载期审计记录和恶意 fixture。
13. link map、build-id 崩溃符号化，以及各阶段耗时、读取字节、峰值 RAM、relocation/symbol lookup 计数。
14. 模拟两个 address space 的 host backend 测试，确保 artifact 可共享、instance 状态不共享，为后续隔离演进验证接口边界。

#### 增量投入估算

以下是建立在“Phase 2 的多 DSO、ctor/fini、emutls、registry 和故障回滚已经稳定”之上的新增投入，按一名熟悉 BlueOS 内核、librs 和 ELF 的工程师估算；不包含需求评审等待、板卡排队和上游工具链缺陷修复：

| 工作项 | 人周 |
| --- | ---: |
| `RuntimeLoadSession`、增量依赖图与 local group scope | 3–4 |
| per-ThreadGroup registry、generation handle、syscall ABI | 2–3 |
| `librs` 的四个 `dl*` API、init-plan 执行和线程局部 `dlerror` | 1.5–2 |
| 并发同一 DSO、constructor 重入、失败回滚与逻辑关闭 | 1.5–2 |
| ARM32 host/QEMU fixture、ABI 与压力测试 | 1–2 |
| **ARM32 首发小计** | **9–13** |
| 扩展到 Phase 2 的其余 target、cache/ABI 差异与共同回归 | **+5–8** |
| **完整 Phase 3 基线增量** | **14–21（约 3.5–5 工程师月）** |

若 Phase 3 同时要求较完整的 POSIX 兼容层（`RTLD_GLOBAL/RTLD_NOLOAD/RTLD_DEFAULT`、`dlopen(NULL, ...)`、`dladdr/dl_iterate_phdr`），再增加约 **4–7 人周**。若还要求在 ThreadGroup 运行中真正执行 fini 并立即回收任意 DSO，则不能沿用裸 `dlsym` 指针模型；设计和验证受限 `UnloadSafe` profile、显式 callback/call lease、线程安全点、TLS/atexit/unwind 注销及 reclaim 压力测试，预计再增加 **10–18 人周**。机械地在 refcount 归零后直接 unmap 虽然代码量更小，但在当前共享特权地址空间中会把应用的 stale pointer 变成系统级故障，不作为可交付方案。

#### 验收门禁

- `dlopen` 的依赖闭包全部完成 NOW relocation、seal 和 cache sync 后才可见，constructor 执行期间不持 loader/registry 锁。
- 两个线程并发打开同一工件只映射和初始化一次；constructor 内嵌套打开自身/其他 DSO 不死锁。
- `RTLD_LOCAL` 对象不泄漏到其他 group scope；system DSO 仍不能被应用 runtime group interpose。
- `dlsym` 拒绝跨 ThreadGroup、错误 generation、非 Ready 对象和越权 symbol；`dlerror` 在线程之间互不覆盖。
- 匹配次数的 `dlclose` 使 handle 失效但不释放仍可能被裸指针引用的映像；ThreadGroup 退出后统一 fini/reap 且无泄漏。
- 并发启动不会重复初始化 libc。
- 任意失败点不会泄漏 allocation、lease 或半发布 link map。
- 崩溃 PC 可由 build-id + load bias 精确定位。
- 老应用能被新系统拒绝或兼容，不能误执行。
- 旧 system DSO 活跃时不能被原位覆盖。
- 所有平台完成压力、碎片和时延分位测试。

MPU/PMP/MMU protection domain、非特权入口、跨域 context switch、fault attribution 与 syscall copy-in/out 不属于本 Phase。需要时以独立后续方案和提交序列实现，不能通过扩写 `C41` 的权限检查范围把它隐含并入当前产品化门禁。

### 12.6 后续通用内核：增加 `process/` 后端（不属于当前交付）

当 Process、独立 AddressSpace、用户态和 fault-safe copy 已具备后，在现有 `application` 内增加：

```text
process/
  backend.rs          ProcessBackend
  process.rs          Process / ProcessId / task membership
  exec.rs             ELF 准入与 exec stage0
  address_space.rs    ProcessImageMemory / map / protect / unmap
  initial_stack.rs    argc / argv / envp / auxv
```

`ProcessBackend` 只需向公共控制面提供 `prepare_exec/start/terminate/reap/query_usage` 等内部操作。`prepare_exec()` 创建未发布的 Process/AddressSpace，校验并映射 main 与受信任 `PT_INTERP`，构造 initial stack；`start()` 进入用户态解释器。普通 `DT_NEEDED` 闭包、符号绑定、relocation、RELRO 和 ctor/fini 由独立用户态 `ld.so` 完成，不进入内核 `application/process`。

启用后的验收门禁：

- `ApplicationManager` 的公共类型、handle table、事件队列和外部 API 不产生第二份实现。
- `ThreadGroup` 与 `Process` 后端可同时编译，由已认证 manifest 中的 `ExecutionModel` 明确分派；产品使用签名，开发 MVP 使用构建期 allowlist。
- 未启用 `blueos_user_process` 时，Process 工件得到确定的 `UnsupportedExecutionModel`，不会误走 RTOS loader。
- 启用进程后，现有全部 RTOS 动态应用测试继续通过。

## 13. 推荐的实际提交顺序

这里的 `Cxx` 是跨仓库的逻辑合入序号，不表示要制造一个跨 repository 的原子 commit。每个变更仍提交到自己的 `kernel`、`librs`、`build` 或应用仓库；跨仓库升级使用“先增加兼容能力，后切换调用方，最后删除旧路径”的顺序。表格某行若同时涉及多个仓库，实际拆成 `Cxx-a/Cxx-b`，按 producer → consumer 顺序合入，但共用同一个阶段 gate。相邻的两个或三个同仓库 `Cxx` 可以组成一个 PR，表中的 gate 不应被压到最后一个巨型 PR 才验证。

关键依赖关系为：

```text
C01–C10 可信 ImageLoader
      ↓
C11–C17 DynamicLinker host 闭环
      ├──────────────┐
      ↓              ↓
C23 RTOS adapters   C24 registry
      └──────┬───────┘
             ↓
C25 ApplicationManager/domain → C26 publish/start
                           ├→ C27 membership/reaper
C18 toolchain → C19 libc ──┤
C20 start ABI → C21 Scrt1 ─┤
                           └→ C28 librs runtime → C29 QEMU MVP
```

### 13.1 Phase 0：`ImageLoader` 提交序列

| 提交 | 仓库/子树 | 实现的数据结构和方法 | 本提交必须通过的 gate |
| --- | --- | --- | --- |
| C01 `loader: add typed addresses and structured errors` | `kernel/loader` | `TargetAddr/TargetWord/TargetRange/ImageId/AllocationId/LoadError/LoadStage/LoadErrorKind/ErrorContext`；实现 checked add/sub、range contains/overlaps 和目标字宽 encode | 纯单元测试覆盖 32/64 位边界、signed addend、溢出和错误上下文；不改变现有 `load_elf()` 行为 |
| C02 `loader: add read-at input and artifact admission` | `kernel/loader` | `ElfReader/SliceElfReader/ArtifactRequest/ArtifactIdentity/AdmittedArtifact/LoadLimits`；实现 `ImageLoader::admit` 和最小 `ArtifactPolicy` | 截断 header、错误 machine/class/endianness、超限文件确定失败；fake memory backend 调用次数为 0 |
| C03 `loader: own parsed ELF metadata` | `kernel/loader` | `ParsedImage/ElfHeaderInfo/ProgramHeaderInfo/LoadSegmentInfo/DynamicInfo/TableRange`；实现 `ImageLoader::inspect`，把 goblin 借用结果复制为自有结构 | parser 临时缓冲在 `inspect` 后即可释放，reader handle 仅作为值继续传递；malformed phdr/dynamic range corpus 通过，无 `Elf<'a>` 进入长期对象 |
| C04 `loader: plan image layout before allocation` | `kernel/loader` | `ImageLayout/SegmentLayout/PlannedArtifact/ImageLayoutBuilder`；实现 `build/allocation_request/load_bias_for/locate_vaddr_range` | 非零最低 vaddr、segment gap、最大 `p_align`、重叠权限、W+X、非 X entry 和所有算术溢出测试 |
| C05 `loader: reserve images through ImageMemory` | `kernel/loader` | `ImageMemory/AllocationRequest/ImageAllocation/ReservedImage/TargetLocation`，以及 `MemoryMapper` adapter；实现 `reserve/target_base/release` | fake backend 返回错误 base/长度/对齐时失败并释放；现有 mapper 仍可编译 |
| C06 `loader: copy segments and zero bss` | `kernel/loader` | `LoadedImageData/LoadedRegion/MappedState/MappedImage`；实现 `copy_and_zero/target_addr/locate` | 在预填充 `0xa5` 的内存上验证 BSS/gap；逐 read/write 故障注入无 allocation 泄漏 |
| C07 `loader: normalize runtime ELF metadata` | `kernel/loader` | `RuntimeImageMetadata/RuntimeDynamicInfo/RelocationTables/ImageLifecycleMetadata/RuntimeMetadataState`；实现 `MappedImage::decode_runtime` 和 REL/RELA/RELR table decoder | 所有 runtime table 必须落在 R region；init/fini array 此时只校验 range、不读取未重定位指针；首版拒绝项返回 `UnsupportedByProfile` |
| C08 `loader: apply ARM32 relative relocations and fail closed` | `kernel/loader` | `RelocationRecord/RelocationRequest/RelocationEngine` 与 ARM32 `R_ARM_RELATIVE`；实现 REL addend 的有界读取、保留通用 RELA decoder、`RuntimeImage::into_relocated` | ELF32/小端 golden fixture 精确比较写值；越界、未对齐、32 位溢出和未知 relocation 均失败并回滚 |
| C09 `loader: seal permissions and synchronize code cache` | `kernel/loader` | `CodeCache/SealPlan/ProtectionRange/AppliedProtection/RelocatedImage/SealedImage`；实现 `build_seal_plan/protect/sync_instruction_cache/seal` | 验证 RELRO 和最终逻辑权限顺序；cache 未实现时失败；无硬件保护只返回 `LogicalOnly` |
| C10 `loader: route load_elf through ImageLoader` | `kernel/loader` | 将 `load_static_pie/load_fixed_exec` 组合为新阶段转换；旧 `load_elf()` 只保留参数兼容包装 | ARM32 relative host fixture 和现有 RISC-V64 static PIE、ET_EXEC、QEMU 回归全部通过；删除或禁止旧的第二套 relocation 路径 |

C10 是 Phase 0 的合入门。此时还没有 `DT_NEEDED`，但 S0–S4、relative S7 和 S8 已经由真实类型边界串起来。

### 13.2 Phase 0.5：`DynamicLinker` 提交序列

| 提交 | 仓库/子树 | 实现的数据结构和方法 | 本提交必须通过的 gate |
| --- | --- | --- | --- |
| C11 `loader: add dynamic artifact policy and resolver contracts` | `kernel/loader` | `ArtifactProfile/DependencyKey/DependencyRequest/ResolvedArtifact/ArtifactResolver`；补全 `ApplicationArtifactPolicy::validate_runtime_features` | resolver 使用固定输入，不依赖 VFS/环境变量；PT_INTERP/TLS/RPATH/TEXTREL/可执行栈/IFUNC/COPY/version 拒绝表测试 |
| C12 `loader: discover a bounded dependency graph` | `kernel/loader` | `DependencyNode/DependencyEdge/DependencyGraph/DiscoveryQueue/ImageOwnership`；实现 BFS、SONAME/file identity 双索引、SCC 与配额 | 线性、菱形、重复 SONAME、同文件别名、循环和深度/数量超限 fixture |
| C13 `loader: decode dynamic symbols and hash tables` | `kernel/loader` | `SymbolTable/SymbolRef/SymbolBinding/SymbolVisibility/SymbolType`；实现 GNU/SysV hash lookup 和 export iterator | hash 命中/缺失与线性 oracle 一致；损坏 bucket/chain 不越界 |
| C14 `loader: freeze application and system symbol scopes` | `kernel/loader` | `ScopeSet/SymbolScope/ResolvedSymbol`；实现 `bfs_scope/for_relocation/lookup_global/lookup_weak/resolve_for_relocation` | application/system scope 隔离、undefined weak=0、hidden/protected 和 duplicate strong 测试 |
| C15 `loader: add staged link sessions and rollback` | `kernel/loader` | `LinkSessionData/RollbackAction/BuildingSession/ScopedSession/RelocatedSession/SealedSession`；实现 consuming transitions 和 active Drop rollback | 在每个 allocation、依赖、scope、relocation、seal 点注入失败，allocation/permit/lease/link map 计数归零 |
| C16 `loader: complete the ARM32 NOW relocation engine` | `kernel/loader` | `RelocationPolicy` 与 `ArchRelocator` 的 `R_ARM_RELATIVE/R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT`；实现 relative → symbol → PLT 三 pass，校验 `P` owner/range、`S` scope、Thumb function address 和控制流目标 X region | 人工 ELF 与固定 oracle 对比 symbol owner 和目标字节；REL addend、Thumb bit、越权 owner、非 X `JUMP_SLOT`、未知类型和所有范围错误 fail-closed；明确测试不把该门禁描述成完整 CFI |
| C17 `loader: build lifecycle plans and commit LinkProduct` | `kernel/loader` | `LifecycleEntry/InitPlan/FiniPlan/LinkContext/LinkMapEntry/LinkProduct/PublicationBatch`；实现 SCC DAG init、逆序 fini、`SealedSession::commit` 和 host snapshot publish | ctor/fini 顺序、函数地址 X-range 校验、commit 前失败无可见条目、commit 后快照完整；生产依赖无 `//librs` |

C17 是 Phase 0.5 的合入门。到此 loader 核心可在 host 上走完 S0–S9，但没有线程、SWI 和内核全局 registry 依赖。

### 13.3 Phase 1：ARM32 Thumb v7-M 纵向 MVP 提交序列

| 提交 | 仓库/子树 | 实现的数据结构和方法 | 本提交必须通过的 gate |
| --- | --- | --- | --- |
| C18 `build: add ARM32 PIC sysroot and artifact link profiles` | `build` | `thumbv7m-vivo-blueos-newlibeabi` 的 `kernel_static/dynamic_app/dso` templates、统一 clang+LLD toolchain、PIC sysroot 和 `check_blueos_elf.py` | kernel PIC 前后尺寸/入口/map A/B；自动检查 ELF32、EM_ARM、Thumb-only、soft-float EABI 及三种产物契约 |
| C19 `librs: produce libc.so.1 with a frozen export set` | `librs`/`build` | `libc.so.1` target、SONAME、build-id、ABI note、`librs.exports`（含动态启动 ABI 与 `spawn` API）和 SDK `libc.so` import | 无普通内核 undefined symbol；导出仅来自清单；DSO relocation 全在 loader 白名单 |
| C20 `kernel: add versioned application start and exit ABI` | `kernel/header` | `ApplicationHandle/BlueOsAuxvEntry/BlueOsFunctionPlan/BlueOsApplicationStartInfo`，追加式 `ApplicationLaunch/InitComplete/BeginExit/FinishExit` syscall 编号（`ApplicationLaunch` 携带 path/argv/envp，约定 handler 侧 copy-in） | C/Rust layout、32/64 位 `struct_size`、旧 syscall 编号和旧静态启动 ABI 不变 |
| C21 `librs: add ARM32 blueos_scrt1 and preserve static startup` | `librs`/SDK build | Thumb `blueos_scrt1::_start`、动态 `__librs_start_main` 声明和独立 static start adapter | Thumb C hello PIE 为 `ELFCLASS32 + EM_ARM + ET_DYN + PT_DYNAMIC + DT_NEEDED libc.so.1` 且无 `PT_INTERP`；入口 bit 0/栈对齐正确，现有 shell/kernel image 仍启动（shell 的 dynamic_app 迁移推迟到 C29） |
| C22 `kernel: add positional VFS reads for executable artifacts` | `kernel/vfs` | `File::len/read_at/read_exact_at`，保持共享 offset 不变 | 并发 positional read、短读/EOF/IO error；现有 read/lseek 语义不变 |
| C23 `kernel: implement loader adapters and ARM code-cache service` | `kernel/application/adapters` + `kernel/application/thread_group/adapters` + `kernel/arch` | 通用 `VfsElfReader/SystemLibraryPaths`、模型特有 `FlatImageMemory/ApplicationArtifactResolver`、ARM `arch/*/cache.rs` 的 `sync_instruction_cache`；实现固定路径 resolve、allocation+offset 访问、cache capability、DSB/ISB | adapter contract suite 与 host fake adapter 共用；无 cache 的 MPS2 明确报告能力且执行 barrier，cache-enabled 板缺维护实现时拒绝执行新代码 |
| C24 `kernel: add system DSO registry permits and leases` | `kernel/application/thread_group` | `DsoArtifact/SystemDsoInstance/SystemDsoState/SystemDsoRegistry/SystemDsoLease/SystemDsoLoadPermit`；实现 begin/wait/publish/fail/release | 两个并发请求只有一个 permit；失败唤醒 waiter 且 generation 不混淆；lease Drop 不执行 fini |
| C25 `kernel: add the single ApplicationManager and thread-group backend` | `kernel/application` | `ApplicationManager/ApplicationRegistry/ApplicationInstance/ApplicationLaunchRequest/ApplicationState/ApplicationQuota/ApplicationEventQueue/ThreadGroupBackend/ThreadGroup`；实现 handle table、状态转换和 backend 分派；`launch()` 内核内部可直呼（boot 自举装载 shell），syscall handler 复用同一入口 | 全局锁外执行 fake slow prepare；Process 请求稳定返回 unsupported；状态机非法转换失败 |
| C26 `kernel: publish linked images into ThreadGroup` | `kernel/application/thread_group` | `ApplicationLoader/ApplicationStartStorage/ReapResources`；实现 `link_application/install_link_product/publish_relocated/build start storage` | S0–S9 使用真实 VFS/memory/registry adapter；发布前故障不留下 ThreadGroup/link map/lease，start-info 指针稳定 |
| C27 `kernel: add app membership, two-phase exit and deferred reaping` | `kernel/thread`、syscall、scheduler、`application` | `ThreadGroupMembership/ApplicationEvent`；实现 CreateThread 继承、`ApplicationLaunch`（argv/envp copy-in + S0 准入后转 `ApplicationManager::launch`）与 InitComplete/BeginExit/FinishExit 校验、`can_reap/take_resources_for_reap` | 伪造 handle 被拒绝；非法指针/越界 argv 被拒绝；scheduler 只投递事件；最后线程前不释放 image；reaper 顺序测试 |
| C28 `librs: initialize and exit dynamic applications` | `librs` | `LibcApplicationContext`、`PthreadTcb.libc_application_context`、auxv、atexit、emutls cleanup；实现新版 `__librs_start_main/getauxval`、`spawn(path, argv, envp)`（经 `ApplicationLaunch` syscall，返回应用句柄）与 init/main/exit/fini 序列 | argv/envp/auxv、ctor 一次、pthread 继承、emutls 隔离/destructor、fini 逆序、`spawn` 返回有效句柄；所有调用在 loader lock 外 |
| C29 `shell: run ARM32 Thumb dynamic applications end to end` | shell/app fixture/`qemu_mps2_an385` CI | shell 以动态应用自举；`run` 命令经 librs `spawn` API 走 `ApplicationLaunch` syscall、Thumb hello PIE fixture 和错误 fixture | ELF32/EM_ARM/soft-float/Thumb 契约、C hello、heap、文件 IO、BSS、weak、function/data symbol、两应用共享 libc、缺依赖/符号/ABI、退出回收和重新加载 |

C29 是首个真正可运行里程碑。C18–C21 可以与 C11–C17 后半段并行开发，但 C23 不能绕过 C17 自己实现第二套 ELF/relocation 逻辑，C29 也不能用测试专用静态链接假装动态链路成功。

### 13.4 Phase 2：多 DSO 与多架构提交序列

| 提交 | 仓库/子树 | 实现的数据结构和方法 | 本提交必须通过的 gate |
| --- | --- | --- | --- |
| C30 `loader: load application-private dependency closures` | loader + thread-group resolver | 扩展 `DependencyKey/ImageOwnership/DiscoveryQueue`；实现包内 `lib/` resolve、私有 allocation 所有权和 SONAME/file 双去重 | app→foo→libc、菱形、循环、同名不同文件、整组 rollback/reap |
| C31 `loader: complete multi-DSO visibility and lifecycle semantics` | `kernel/loader` | 扩展 `ScopeSet/SymbolScope/InitPlan/FiniPlan`；实现 weak/hidden/protected、SCC 稳定序和多 DSO emutls control 保活 | graph、symbol owner、init/fini 顺序与 oracle 一致；不新增单 DSO dlclose |
| C32 `loader: extend ARM32 to Thumb v8-M hard-float` | loader + build + kernel arch | 复用 ARM32 relocation backend，增加 `thumbv8m.main-vivo-blueos-newlibeabihf` artifact profile、hard-float ABI 校验和 cache-enabled Cortex-M 维护 | hard/soft-float 拒绝矩阵、Thumb entry/函数地址、MPS3/板上 app/libfoo/libc smoke；soft-float Phase 1 fixture 不回退 |
| C32V（可选）`loader: add planned ARM32 Thumb branch veneers` | loader + build + kernel arch | 仅为声明 `LoaderVeneerV1` 的 profile 增加 `VeneerIslandPlan`、near executable allocation、受控分支修补、veneer ownership 与 RX/cache 收口；不改变 `PltGotOnly` 默认路径 | 第 11.15 节全部 gate；默认 profile 继续拒绝分支动态 relocation；不可写 Flash callsite 不产生运行期修补 |
| C33 `loader: add RISC-V64 relocation and cache backend` | loader + kernel arch | RISC-V64 `R_RISCV_RELATIVE/R_RISCV_64/R_RISCV_JUMP_SLOT`、RELA addend、LP64/ISA note 与 `fence.i` | 每种 relocation golden fixture、当前 hart cache 同步和 QEMU/板上 app/libfoo/libc smoke |
| C34 `loader: add AArch64 relocation and cache backend` | loader + kernel arch | AArch64 `R_AARCH64_RELATIVE/R_AARCH64_ABS64/R_AARCH64_GLOB_DAT/R_AARCH64_JUMP_SLOT` 与 D-cache clean/I-cache invalidate | 每种 relocation golden fixture、页权限、QEMU/板上 app/libfoo/libc smoke |
| C35 `loader: add RISC-V32 relocation backend` | loader + kernel arch | RV32 target word、`R_RISCV_RELATIVE/R_RISCV_32/R_RISCV_JUMP_SLOT`、ILP32/ISA note 和无 A 扩展同步策略 | IMAC/IMC fixture、32 位溢出、QEMU/板上 smoke；跨架构 graph/symbol/lifecycle 快照一致 |

`C32V` 是预留的条件提交号，不顺延 `C33`–`C42`；在产物证据和目标板需求出现前不创建空实现，也不阻塞 Phase 2/3。

### 13.5 Phase 3：运行期装载与产品化提交序列

| 提交 | 仓库/子树 | 实现的数据结构和方法 | 本提交必须通过的 gate |
| --- | --- | --- | --- |
| C36 `loader: add runtime load sessions and local group scopes` | `kernel/loader` | `RuntimeLoadSession/RuntimeLinkSnapshot/DlopenFlags`；复用 staged link/rollback，增量建立依赖闭包与 `RTLD_LOCAL` group scope | host fixture 验证既有映像不被重定位；增量失败只回滚新 group；lazy/deepbind/同 SONAME 不同身份稳定拒绝 |
| C37 `kernel: add per-thread-group runtime DSO handles and syscalls` | kernel/header/application | `RuntimeLinkState/RuntimeDsoHandleTable/RuntimeDsoState` 及 `DynamicLibraryPrepareOpen/NextInitializer/InitComplete/Lookup/Close`；实现 permit、generation、membership 与 copy-in | 两线程并发首开只有一个 owner；waiter 只见 Ready/Failed；跨组/陈旧 handle、过长 path/symbol 和非法 flag 被拒绝；所有阻塞和 constructor 都在全局锁外 |
| C38 `librs: expose NOW-local dl APIs with deferred close` | `librs`/SDK + ARM32 fixtures | 导出 `dlopen/dlsym/dlerror/dlclose`，执行 runtime init plan，维护线程局部错误；重复 open/close 计数和 `LogicallyClosed`，映像延迟到 ThreadGroup 退出回收 | `qemu_mps2_an385` 上插件 load/lookup/call、依赖 ctor、嵌套 dlopen、并发同库、错误 symbol、线程独立 dlerror、stale handle 与退出统一 fini/reap 全部通过 |
| C39 `runtime: enforce signed artifact manifests and ABI policy` | build/install/kernel | 扩展 `ArtifactIdentity/ArtifactRequest/ApplicationArtifactPolicy`，加入签名、hash、allowlist、最低 kernel/librs ABI 和 artifact generation | 安装期与加载期双重校验、篡改/降级/错误模型/ABI 回滚用例；运行期打开不能绕过应用启动准入 |
| C40 `runtime: harden registry concurrency and quiescence` | kernel application | `QuiescenceEvidence/RegistryGeneration/RetryPolicy`；实现启动及运行期并发失败重试、cached fallback 和安全卸载判定 | OOM/ctor failure/依赖环/反复启动退出/回调与 TLS destructor 保活压力测试 |
| C41 `runtime: audit hardening and publish diagnostics` | loader + kernel arch/debug | 扩展 `ArtifactPolicy/RelocationPolicy/AppliedProtection/LinkMapEntry/LoadMetrics`；发布 policy 判定、实际权限结果、build-id 崩溃符号化和逐阶段 metrics | W+X/TEXTREL/可执行栈/越权 relocation 恶意 fixture；`LogicalOnly/HardwareEnforced` 不混淆；PC→image+offset 精确定位和时延/RAM 预算 |
| C42 `release: add ABI, OTA and address-space simulation gates` | build/CI/SDK | SONAME/export ABI diff、A/B compatibility 检查和双模拟 address-space backend | 活跃旧 DSO 不被覆盖；回滚 ABI 检查；相同 artifact 可共享而 load bias/RW/lease/link map 不共享；只验证未来接口，不宣称已实现隔离 |

### 13.6 提交纪律与暂缓项

- 每个 `Cxx` 必须同时加入该层最窄测试；不能先合入没有调用方、没有边界测试的裸 `unsafe` 写内存代码，等到 C29 才第一次运行。
- 新 API 先作为内部或兼容扩展加入；调用方完成迁移并通过 QEMU 后，才删除旧入口。`load_elf()` 在当前交付内继续保留兼容包装。
- 跨仓库 ABI 按 `kernel/header → librs/SDK producer → kernel consumer → shell/app caller` 顺序合入，所有新增字段依赖 `abi_version + struct_size` 尾部扩展。
- 架构提交一次只增加一个 backend，并复用同一 graph/scope/lifecycle fixture；禁止在架构目录复制依赖解析或 symbol lookup。
- Phase 0–2 不插入 `dlopen`；Phase 3 仅按 `C36`–`C38` 引入 NOW/local scope/逻辑关闭模型。原生 ELF TLS、lazy binding、IFUNC、COPY、symbol version、`RTLD_DEEPBIND/RTLD_NEXT` 或用户态 `PT_INTERP` 仍不属于当前提交序列，不能作为“顺手支持”的小改动混入。
