# BlueOS 普通 namespace 动态加载替换方案

## 1. 文档目标

本文给出一套用于替换当前 manifest-closed package loader 的实现方案。当前阶段以“能够稳定运行动态应用”为首要目标，不引入应用签名、依赖边 allowlist、保留系统 SONAME、`RPATH/RUNPATH` 或环境变量搜索等产品化策略。

新的基本模型是：

- 启动 ELF 可以使用绝对路径或相对路径；
- `DT_NEEDED` 和后续 `dlopen()` 可以按绝对路径、相对路径或名称请求 DSO；
- 纯名称优先查询应用私有库，找不到时查询系统库；
- DSO 可以没有 `DT_SONAME`；
- 应用依赖闭包在运行期从真实 ELF 中预扫描得到；
- 系统库闭包预扫描完成后，通过 registry 一次性原子获取；
- 构建期 package 只负责把文件安装到文件系统，不参与运行期依赖解析。

本文中的“普通 namespace”是路径搜索范围和已加载对象集合，不是安全边界。当前共享特权地址空间、系统 DSO 生命周期和 ELF 格式安全检查仍保持不变。

### 1.1 实现状态（2026-09-10）

本方案的运行期替换已经落地：

- `ApplicationService` 在 `spawn()` 入口捕获 pwd，并把绝对或相对启动参数规范化为稳定的 root path；
- `NamespaceLoadPlanner` 从真实 VFS ELF 做只读 BFS 预扫描，冻结 path、snapshot、identity、`DT_NEEDED` 和 dependency edge；
- `NamespaceArtifactResolver` 在映射前原子获取完整 system path-key 闭包，之后只重放 plan，不在链接中途搜索目录或逐个申请系统库；
- `SystemDsoRegistry` 使用 `(LinkDomainId, canonical system path)` 作为 key，不再依赖 ELF 的 `DT_SONAME`；
- strict package catalog、package resolver、manifest 生成/校验脚本和 `blueos_app_package` 模板已经删除；
- `blueos_app_bundle` 与 boot seed catalog 只描述“构建产物安装到哪个 VFS 路径”，不参与运行期解析；
- 构建 gate 允许无依赖的 PIE 和无 SONAME 的 DSO；`libscope_sys` 已改成无 SONAME 的系统 DSO 纵向 fixture。

当前 MPS2 QEMU 门禁已经覆盖相对路径启动、私有库名称查找、菱形/循环依赖、共享系统库并发首次加载、无 SONAME 系统库的发布/复用/回收，以及 scope、init/fini 和 emutls。本文后续保留设计理由、接口约束和未完成的 `dlopen/dlclose` 扩展边界。

## 2. 本阶段范围

### 2.1 支持

- `ET_DYN` 动态应用；
- 没有 `DT_NEEDED` 的 static PIE；
- 有或没有 `DT_SONAME` 的依赖 DSO；
- 绝对或相对启动路径；
- 三种依赖请求：绝对路径、相对路径、纯名称；
- 应用私有 DSO 和共享系统 DSO；
- 私有库优先、系统库 fallback；
- 依赖环、菱形依赖和同一文件的路径别名；
- 多应用并发首次加载同一系统库；
- 现有 NOW relocation、scope、init/fini、SCC、seal 和回滚流程。

### 2.2 暂不支持

- 应用签名和 build-id 强制校验；
- 构建期逐条 `DT_NEEDED` manifest；
- 系统库名称保留或禁止应用替换；
- `LD_LIBRARY_PATH`；
- `DT_RPATH`、`DT_RUNPATH` 和 `$ORIGIN` token 展开；
- `RTLD_GLOBAL`、lazy binding；
- 任意 DSO 的安全即时卸载；
- 运行时替换已经加载的 `/system/lib` 文件。

不支持上述策略不等于放松 ELF 内存安全检查。文件边界、整数溢出、架构/ABI、segment 权限、relocation 范围、W^X、目标地址和资源上限等校验仍是 loader 正确性的一部分，必须保留。

## 3. 路径和 namespace 模型

### 3.1 推荐目录布局

```text
/apps/<application>/
├── app.elf
└── lib/
    ├── libfoo.so.1
    └── libbar.so.1

/system/lib/
├── libc.so.1
└── libscope_sys.so.1
```

对于根应用 `/apps/multi/app.elf`：

```text
application_root = /apps/multi
private_lib_dir  = /apps/multi/lib
```

第一版将启动 ELF 的父目录视为应用根目录。若将来需要 `/apps/foo/bin/app.elf` 这类布局，再由 launcher 显式传入 namespace root；本阶段不增加目录向上猜测规则。

### 3.2 启动路径

调用方不需要传绝对路径。启动入口同时接受：

```text
/apps/hello/app.elf  # 绝对路径
apps/hello/app.elf   # 相对于 pwd
app.elf              # 相对于 pwd
```

kernel 在一次 launch 开始时获取一次 `pwd`：

```rust
let launch_pwd = crate::vfs::path::get_working_dir().get_full_path();
```

然后将输入路径与该快照组合并规范化：

```text
pwd   = /apps/hello
input = app.elf
root  = /apps/hello/app.elf
```

“内部使用绝对路径”不等于“用户必须输入绝对路径”。内部解析成稳定绝对路径是为了避免加载过程中 `chdir()` 改变后续依赖含义，同时便于派生应用私有目录、记录诊断和比较系统 catalog 路径。

当前 BlueOS 的 working directory 仍是 VFS 全局状态，而不是 `ThreadGroup` 私有状态。因此必须在 launch 开始时只读取一次，不能在每个 `DT_NEEDED` 解析时重新读取。后续将 cwd 移入 `FsEnv/ThreadGroup` 时，namespace 接口不需要变化。

### 3.3 Namespace 数据结构

```rust
pub struct ApplicationNamespace {
    /// 调用 launch 时捕获的工作目录。
    launch_pwd: String,
    /// 已按 launch_pwd 解析、规范化的根 ELF 路径。
    root_path: String,
    /// root_path 的父目录。
    application_root: String,
    /// application_root/lib。
    private_lib_dir: String,
    /// 当前板级 ELF/ABI 策略。
    profile: LoadProfile,
    /// 系统 DSO 的共享域。
    system_domain: LinkDomainId,
}
```

namespace 在一次启动期间保持不变。应用运行后即使 cwd 改变，也不会改变启动依赖的解析结果。

## 4. 依赖请求分类和搜索顺序

`DT_NEEDED` 的字符串先按是否包含路径分隔符分类。

### 4.1 绝对路径

以 `/` 开头的请求按规范化后的绝对路径精确打开：

```text
/vendor/lib/libfoo.so
```

规则：

1. 只打开这个路径；
2. 若路径与 system catalog 中某个 entry 的路径相同，则按共享系统 DSO 处理；
3. 否则按当前应用的 `SessionPrivate` DSO 处理；
4. 路径不存在或 ELF 加载失败时直接失败，不执行名称 fallback。

### 4.2 相对路径

包含 `/` 但不以 `/` 开头的请求是相对路径：

```text
./lib/libfoo.so
../shared/libbar.so
plugins/libcodec.so
```

对于 ELF 的 `DT_NEEDED`，相对路径以 requester ELF 所在目录为基准：

```text
requester = /apps/foo/app.elf
needed    = ./lib/libfoo.so
result    = /apps/foo/lib/libfoo.so
```

```text
requester = /apps/foo/lib/libfoo.so
needed    = ./libbar.so
result    = /apps/foo/lib/libbar.so
```

这样可以直接表达“相对于当前 ELF”的 bundle 布局，而不依赖加载期间可能变化的全局 cwd。

未来 `dlopen("./plugin.so", ...)` 的显式相对路径则以调用 `dlopen()` 时所属 `ThreadGroup` 的 cwd 为基准，这与文件 API 的直觉一致。启动期 `DT_NEEDED` 与 `dlopen` 必须通过不同的 `ResolveBase` 明确表达基准：

```rust
pub enum ResolveBase<'a> {
    RequesterDirectory(&'a str),
    CurrentWorkingDirectory(&'a str),
}
```

相对路径同样是精确请求：打开失败后不再按名称搜索其他目录。

### 4.3 纯名称

不包含 `/` 的请求按名称搜索：

```text
libfoo.so.1
libc.so.1
```

root 或应用私有 DSO 发出的请求使用：

```text
1. <application_root>/lib/<name>
2. system catalog 中的 <name>
3. unresolved
```

例如：

```text
DT_NEEDED=libfoo.so.1

先尝试 /apps/foo/lib/libfoo.so.1
不存在时再查询 system catalog
```

只有路径不存在（`ENOENT/ENOTDIR`）才进入下一搜索项。候选文件存在但不是有效 ELF、ABI 不匹配、读取失败或加载中被修改时，当前依赖直接失败，不能用系统版本掩盖错误。

系统 DSO 发出的请求不能进入应用私有目录：

```text
SystemCandidate/ExternalReady requester:
    absolute/relative path -> 必须命中 system catalog entry
    plain name             -> 只查询 system catalog
```

这是共享正确性要求，而不是应用准入安全策略。一个系统 DSO 的 relocation 结果必须与第一个触发它加载的应用无关，才能共享给其他应用。

### 4.4 应用可以替换同名系统库

本阶段不区分 `Reserved` 和 `Fallback`。如果同时存在：

```text
/apps/foo/lib/libcodec.so.1
/system/lib/libcodec.so.1
```

应用 root/private requester 选择前者。它被标记为 `SessionPrivate`，不会注册到 `SystemDsoRegistry`。

即使应用提供 `/apps/foo/lib/libc.so.1`，也允许优先选择；能否运行由它是否真正提供所需 ABI 和符号决定。系统 DSO 自身仍只能绑定系统 catalog 中的库，因此私有替换不会污染共享系统 DSO。

## 5. 无 `DT_SONAME` 支持

### 5.1 三种不同情况

- root ELF 可以没有 `DT_SONAME`；
- 没有依赖的 static PIE 可以没有 `DT_NEEDED`，依赖扫描得到空集合后直接继续运行；
- 依赖 DSO 也允许没有 `DT_SONAME`，但仍需有可用于动态链接的 `PT_DYNAMIC`、动态符号和必要 relocation 信息。

空的 `DT_NEEDED` 仍然无效，因为 resolver 没有任何可查询内容。允许缺少 `DT_SONAME` 不等于允许空依赖请求。

### 5.2 不合成伪 SONAME

不要把文件名或请求字符串写回 ELF 元数据并伪装成 `DT_SONAME`。运行期同时保留三种不同概念：

```rust
pub struct PlannedImage {
    /// VFS snapshot 的稳定身份，用于真正的映像去重。
    identity: ArtifactIdentity,
    /// 已解析的规范化文件路径。
    path: String,
    /// root 或 dependency。
    role: ArtifactRole,
    /// 命中 system catalog 时为规范化 catalog path，否则为 None。
    system_key: Option<DependencyName>,
    /// ELF 实际声明的可选 DT_SONAME 和全部 DT_NEEDED。
    scanned: ScannedArtifact,
}
```

去重顺序为：

1. `ArtifactIdentity` 相同：复用同一映像并记录额外 dependency edge；
2. identity 不同且两者都声明了相同 `DT_SONAME`：保持现有 `IdentityConflict`；
3. 没有 `DT_SONAME` 的不同文件：由各自规范化路径和 identity 区分；
4. 请求名称与实际 `DT_SONAME` 不同不再直接失败，路径请求也可以加载带任意或无 SONAME 的 DSO。

例如，以下两个请求最终指向同一 inode/snapshot 时只加载一次：

```text
libfoo.so
./lib/libfoo.so
```

### 5.3 Loader 修改

`ArtifactRole::SharedObject` 不再要求 `DT_SONAME`。需要删除 `validate_dynamic_features()` 中“SharedObject 且无 SONAME”即失败的规则，并更新注释和测试。

依赖 DSO 仍要求存在 `PT_DYNAMIC`。当前缺少 `PT_DYNAMIC` 时使用 `DT_SONAME` 构造错误上下文的做法应改成普通的 missing dynamic-table/BadElf 错误，避免继续暗示 SONAME 是必需项。

`DependencyGraph` 已使用 `Option<DependencyName>` 保存 SONAME，主体结构不需要改为必填。planner 先按规范化路径去重，再按 snapshot identity 合并路径别名；linker 继续保留 identity 优先、SONAME 冲突次之的规则。

## 6. 系统 catalog 和 registry key

系统 DSO 不能再以 ELF 的 `DT_SONAME` 作为唯一 registry key，因为现在允许它缺失。系统 catalog 应提供独立于 ELF 元数据的稳定 key。

第一版直接使用规范化 catalog 路径作为系统库 key：

```rust
pub struct SystemLibraryEntry {
    /// 纯名称搜索时使用，例如 b"libc.so.1"。
    lookup_name: &'static [u8],
    /// 系统库的规范化绝对路径，同时作为 registry key。
    path: &'static str,
    build_id: Option<&'static [u8]>,
    keep_cached: bool,
}
```

`SystemLibraryPaths` 提供两种查询：

```rust
fn resolve_name(&self, name: &[u8]) -> Option<&SystemLibraryEntry>;
fn resolve_path(&self, path: &str) -> Option<&SystemLibraryEntry>;
```

registry slot 从：

```text
(LinkDomainId, DT_SONAME)
```

改为：

```text
(LinkDomainId, normalized system catalog path)
```

这样下面三种请求只要最终命中同一个 catalog entry，就共享同一个系统实例：

```text
libc.so.1
/system/lib/libc.so.1
相对路径规范化后得到 /system/lib/libc.so.1
```

`LoadPermit` 已经标识具体 registry slot。`ApplicationLoader::hand_off()` 不应再通过 published descriptor 的 SONAME 匹配 permit，而应使用 resolver 保存的 `(system key, artifact identity, permit)`，在已发布 system candidate 中按 `ArtifactIdentity` 配对。系统 DSO 没有 SONAME 时也能正确发布。

registry 中现有的 `dependency_names` 相应改为 `dependency_keys`。系统库生命周期日志输出 catalog path，link map 仍单独输出 ELF 的可选 SONAME：

```text
DSO_LOAD path=/system/lib/libscope_sys.so.1
LINK_MAP owner=7 soname=- bias=...
```

## 7. 运行期依赖预扫描

### 7.1 为什么采用预扫描

删除构建期 manifest 后，系统库集合只有读取真实 `DT_NEEDED` 才能知道。若边映射边逐个获取 system permit，两个并发 session 可能分别持有 A/B 再等待 B/A。

本方案直接在第一版进行只读预扫描，避免引入全局 link gate、`WouldBlock`、部分链接回滚和整次重试。预扫描增加一次元数据读取，但应用映像数量小，当前优先保证模型简单和并发行为确定。

### 7.2 Scan API

在 `loader` crate 中增加只读 scanner，并复用现有 ELF header、program header、dynamic table 和 string table 解码逻辑：

```rust
pub struct ScannedArtifact {
    pub declared_soname: Option<DependencyName>,
    pub needed: Vec<DependencyName>,
}

pub fn scan_artifact<R: ElfReader>(
    reader: &R,
    profile: LoadProfile,
    role: ArtifactRole,
    limits: LoadLimits,
) -> LoadResult<ScannedArtifact>;
```

scanner 不做目标地址分配、segment copy、relocation、seal 或 publication。它必须与正式加载共用解析函数，不能在 kernel adapter 中再写一套不一致的 ELF parser。

root 没有 `PT_DYNAMIC` 时返回：

```text
declared_soname = None
needed = []
```

依赖 DSO 没有 `PT_DYNAMIC` 时失败；有 `PT_DYNAMIC` 但没有 `DT_SONAME` 时正常返回。

### 7.3 Planner BFS

```rust
pub struct NamespaceLoadPlanner<'a> {
    namespace: &'a ApplicationNamespace,
    system_catalog: &'static SystemLibraryPaths,
    limits: SessionLimits,
}
```

planner 从 root 开始按 BFS 扫描：

1. 打开文件并冻结 `FileSnapshotId`；
2. 解析 header、可选 SONAME 和全部 `DT_NEEDED`；
3. 按第 4 节规则将每个请求解析成规范化路径；
4. 根据路径是否命中 system catalog 决定 `SessionPrivate/SystemCandidate`；
5. 按 `ArtifactIdentity` 去重；
6. 记录 requester、needed index、provider edge；
7. 对新 provider 继续扫描；
8. 收集所有 system catalog key，排序并去重。

输出：

```rust
pub struct NamespaceLoadPlan {
    pub root: PlannedImageId,
    pub images: Vec<PlannedImage>,
    pub edges: Vec<PlannedEdge>,
    pub system_keys: Vec<SystemLibraryKey>,
}
```

plan 是本次启动从真实文件生成的临时对象，不是 package manifest，也不写入固件。

### 7.4 Snapshot 一致性

每个 `PlannedImage` 保存扫描时的 `ArtifactIdentity/FileSnapshotId`。正式加载使用同一个已打开 reader，或重新打开后验证 snapshot 完全相同。任何变化返回 `SourceChanged`，本次启动失败；第一版不自动重新规划。

本阶段不支持运行时更新 `/system/lib`。若 batch acquire 返回的 Ready descriptor identity 与 plan 中同一路径的 identity 不一致，释放本次 batch 并返回 `SourceChanged`。以后实现系统库版本切换时再定义跨 generation 更新协议。

## 8. 系统库并发控制

### 8.1 原子获取运行期闭包

planner 得到完整 `system_keys` 后调用：

```rust
registry.acquire_batch(namespace.system_domain(), &plan.system_keys)
```

`acquire_batch()` 从“获取 manifest 声明的闭包”改为“获取本次运行期 plan 解析出的闭包”，状态语义保持：

- 所有 slot 为 `Vacant/Ready`：原子产生全部 `LoadPermit/SystemDsoLease`；
- 任一 slot 为 `Loading/Relocated/Initializing/Unloading`：不修改任何 slot 和计数，返回 `SystemBatchWait`；
- caller 在所有 loader、memory、manager 和 registry 锁之外等待；
- 唤醒后只重试 batch acquire，不重新扫描和映射；
- plan snapshot 在等待期间仍由 reader/generation 检查保护。

因为 caller 在等待时没有持有部分 permit/lease，所以不存在 A/B 的 ABBA。

### 8.2 Resolver 只消费 plan

完成 batch acquire 后才构造 resolver：

```rust
pub struct NamespaceArtifactResolver {
    plan: NamespaceLoadPlan,
    batch_loads: Vec<(DependencyName, LoadPermit)>,
    batch_imports:
        Vec<(DependencyName, SystemDsoLease, PublishedImageDescriptor)>,
    candidates: Vec<SystemCandidateClaim>,
    leases: Vec<SystemDsoLease>,
    imports: Vec<SystemImportClaim>,
    opened_private: Vec<ArtifactIdentity>,
}
```

这里不再保存或执行：

```text
namespace
system_catalog
registry
pending
目录搜索
逐项 acquire/wait
```

目录搜索已经由 planner 完成，batch acquire 在 resolver 构造阶段、linker 分配内存前一次性完成。resolver 收到 `DependencyRequest` 后只查找 plan 中的 edge，并返回对应 reader 或 Ready descriptor。`opened_private` 仅用于让同一 private identity 的诊断日志只输出一次，不参与解析策略。

同一 system key 的多个 dependency edge 只消费一个 permit/lease；额外 edge 复用同一 claim。resolver 成功后把 permit 和 lease 交给现有 publisher/`ThreadGroup`，失败时由 Drop 自动取消或释放。

### 8.3 发布和初始化

以下机制继续保留：

```text
freeze_scopes
relocate
seal
publish
publish_relocated_batch
Initializing
Ready/Failed
```

依赖获取是原子 batch，system candidates 仍然批量发布。constructor 在 loader/registry 锁外执行。应用在 `ApplicationInitComplete` 前退出时，现有 `SystemInitBatch` 负责将该批次切换到 Failed 并唤醒 waiter。

## 9. 启动主流程

```rust
pub fn spawn(&self, input_path: &str, ...) -> Result<ApplicationHandle, ...> {
    let namespace = ApplicationNamespace::from_launch_path(
        input_path,
        current_pwd_snapshot(),
        board_dynamic_profile(),
        self.system_domain,
    )?;

    manager.launch(request, |group| self.prepare(group, &namespace, ...))
}

fn prepare(&self, group: &ThreadGroup, namespace: &ApplicationNamespace, ...) {
    let plan = NamespaceLoadPlanner::new(
        namespace,
        self.loader.catalog(),
        SessionLimits::DEFAULT,
    ).plan()?;

    // ApplicationLoader 内构造 NamespaceArtifactResolver；其构造函数先
    // acquire_batch/wait，再把 root 和依赖交给 DynamicLinker。
    self.loader.link(plan, namespace.profile(), group)?
}
```

`ApplicationService::prepare()` 不再区分 package 和 bare application，所有路径进入同一流程。

## 10. 构建期 bundle 与 boot seed

运行期不再查询 package catalog。构建系统仍需描述哪些文件被放入固件，但它只生成安装文件表：

```rust
pub struct BootSeedFile {
    pub path: &'static str,
    pub bytes: &'static [u8],
}
```

例如：

```text
/apps/multi/app.elf
/apps/multi/lib/libfoo.so.1
/apps/multi/lib/libbar.so.1
/system/lib/libc.so.1
```

该表不包含 profile、private/system edge、`DT_NEEDED` closure 或 SONAME 校验。真实文件系统安装的应用只要符合路径规则即可运行，不需要编译进 kernel catalog。

GN template 改成 `blueos_app_bundle`，只负责构建工件、安装路径和 seed blob，不再生成运行期 manifest。

## 11. 删除和替换清单

本方案不保留 strict/normal 双路。完成替换后直接删除无调用者代码。

### 11.1 kernel repository

删除文件：

```text
kernel/src/application/package.rs
kernel/src/application/adapters/package_resolver.rs
```

删除：

- `ApplicationLoader::link_package()`；
- `PackageArtifactResolver` 的 `ResolverFinish`；
- `service.rs` 的 `package::find_root()` 分支；
- `application/mod.rs` 的 `pub mod package`；
- `adapters/mod.rs` 的 `pub mod package_resolver`；
- `seed.rs` 对 `package::packages()` 和 `image_blob()` 的依赖；
- `kernel/BUILD.gn` 中 `gen_app_package_catalog`、`app_package_catalog` 及其依赖。

新增或替换：

```text
kernel/src/application/namespace.rs
kernel/src/application/planner.rs
kernel/src/application/adapters/resolver.rs
```

`registry.rs` 保留 batch acquire，key 从 SONAME 改为 system catalog path。若迁移后逐 SONAME 的 `AcquireOutcome/WaitHandle/acquire_or_begin_load()` 没有调用者，直接删除。

### 11.2 build repository

删除 strict manifest 专用脚本：

```text
scripts/gen_app_package_catalog.py
scripts/check_blueos_app_package.py
```

先用只生成 bundle/seed layout 的脚本替代 `gen_blueos_app_manifest.py`，然后删除旧脚本。用 `blueos_app_bundle.gni` 替换 `blueos_app_package.gni`，应用 BUILD 文件不再声明 `system_sonames` 或逐边依赖策略。

### 11.3 不删除

- `SystemLibraryPaths`：仍负责系统名称和路径映射；
- `SystemDsoRegistry`：仍负责跨应用共享、并发和 generation；
- `VfsElfReader` 和 snapshot identity；
- `DynamicLinker` 的 graph、scope、relocation、seal、publication；
- `ThreadGroup`、init/fini、SCC 和 reaper；
- boot seed 功能本身。

## 12. 实现顺序

### 步骤 1：放宽 SONAME

- 允许 SharedObject 缺少 `DT_SONAME`；
- 保持 `PT_DYNAMIC` 要求；
- 更新 graph/publish/log 测试以接受 `soname=None`；
- 增加 static PIE 无依赖和无 SONAME DSO fixture。

### 步骤 2：路径解析与 namespace

- 实现 launch pwd 快照；
- 支持绝对/相对 root；
- 实现绝对路径、requester-relative 路径和纯名称分类；
- 增加 `resolve_name()`/`resolve_path()` system catalog 查询。

### 步骤 3：只读 scanner 和 runtime plan

- 从 loader 现有解析器抽出 scan API；
- 实现 BFS planner；
- 保留 path、identity、可选 SONAME、needed 和 edge；
- 使用 frozen VFS snapshot 保证 scan/load 一致。

### 步骤 4：系统 registry 改为 path key

- batch 输入由 SONAME 改为 system catalog key；
- permit 与 candidate 按 identity 配对；
- system dependency lease 使用 key；
- 更新日志、SCC、quiescence 和 reaper。

### 步骤 5：统一 resolver/loader/service

- 增加只消费 plan 的 `NamespaceArtifactResolver`；
- 所有应用统一进入 `load_application()`；
- 删除 package/bare 分支；
- 保持后半段 link/publish/init 流程不变。

### 步骤 6：替换 boot bundle 并删除 strict 代码

- 先生成新的 seed file table；
- 迁移 multi/cycle/scope/tls 示例；
- 删除 package catalog、resolver、生成器和 GN targets；
- 更新 mdBook 中将旧 manifest-closed 方案描述为现状的章节。

每一步都应保持 GN target 可构建；跨 `kernel`、`build`、`apps`、`book` 仓库分别提交。

## 13. 测试矩阵

### 13.1 路径

- 绝对路径启动 `/apps/hello/app.elf`；
- `cd /apps/hello && run app.elf`；
- 相对多级路径启动 `run apps/hello/app.elf`；
- root 的 `DT_NEEDED=./lib/libfoo.so`；
- DSO 的 `DT_NEEDED=./libbar.so`；
- 绝对路径 `DT_NEEDED=/system/lib/libc.so.1`；
- 纯名称先命中私有库；
- 私有文件不存在后命中 system catalog；
- 私有文件存在但损坏时不 fallback。

### 13.2 SONAME

- static PIE root：无 SONAME、无 NEEDED；
- 私有 DSO：无 SONAME，通过纯名称加载；
- 私有 DSO：无 SONAME，通过相对路径加载；
- 同一无 SONAME 文件经名称和路径请求只映射一次；
- 两个不同无 SONAME 文件可同时加载；
- 两个不同文件声明同一 SONAME 仍报告 identity conflict；
- system catalog DSO 无 SONAME 仍可发布、复用和回收。

### 13.3 并发和生命周期

- 两个应用并发首次加载同一个系统路径：一次 load、一次 reuse；
- 两个 system DSO 存在 A/B 反向边时无 ABBA；
- batch 遇到 Loading/Initializing 时不持有部分 permit；
- 等待唤醒后只重试 acquire；
- source generation 改变时失败并完整回滚；
- constructor 失败时 batch 进入 Failed 并唤醒 waiter；
- system SCC 的 init/fini/unload 顺序保持不变；
- 应用退出释放私有映像和 system leases。

## 14. 验收标准

替换完成至少满足：

1. shell 中使用相对路径和绝对路径都能启动动态应用；
2. 没有 `DT_NEEDED` 的 static PIE 能启动；
3. 没有 `DT_SONAME` 的私有和系统 DSO 能加载；
4. 名称请求按“私有库优先、系统库 fallback”解析；
5. 显式绝对/相对路径按精确路径加载；
6. 系统 DSO 不反向绑定应用私有库；
7. 并发系统库加载不产生 ABBA 或 check timeout；
8. strict package runtime 代码和无调用者生成脚本已删除；
9. `check_all` 和相关 QEMU 动态加载测试通过。
