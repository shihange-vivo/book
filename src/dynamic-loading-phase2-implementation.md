# BlueOS Dynamic Loading Phase 2 详细实施计划

> **历史方案提示（2026-09-10）：** 本文中的 manifest-closed package、
> `PackageArtifactResolver`、`blueos_app_package.gni` 和 package catalog 已被
> [普通 namespace 替换方案](./dynamic-loading-namespace-replacement-plan.md)取代并从代码中删除。
> 本文仍用于记录 Phase 2 的链接、scope、生命周期和跨架构设计；涉及依赖发现、路径搜索、
> 系统 registry key 与 boot 安装流程时，应以上述替换方案和当前源码为准。

本文是 Phase 2 的直接开发清单。它以 2026-09-05 的以下代码为基线：

- `kernel@e267a66`：Phase 1 的 `DynamicLinker`、应用控制面、system DSO registry、
  `ThreadGroup`、启动/退出 ABI 和 QEMU 纵向链路已经落地；
- `build@f2d1a10`：已有 ARM32 soft-float 的 `dynamic_app`、`blueos_dso` 和 ELF gate；
- `librs@f771f67`：已有 `libc.so.1`、ARM32 `blueos_scrt1`、应用上下文、init/main/fini
  和 pthread/emutls 退出路径；
- `book@98808e0`：总方案已经定义 C30–C35，本文件将其展开为逐接口、逐提交和逐门禁
  的实施方案。

本文承接 [Phase 1 详细实施计划](./dynamic-loading-phase1-implementation.md)，细化
[总实施计划](./dynamic-loading-implementation-plan.md)中的 C30–C35。Phase 2 分为两个
必须连续完成的里程碑：

```text
P2-A：ARM32 Thumb v7-M 多 DSO

app.elf
  ├─ libfoo.so
  │    └─ libc.so.1
  └─ libbar.so
       └─ libc.so.1

P2-B：复用同一 DynamicLinker 扩展目标 profile

ARM32 v8-M hard-float → RISC-V64 → AArch64 → RISC-V32 IMAC/IMC
```

Phase 2 的完成标志不是“其余架构的 relocator 类型已经存在”，也不是“多生成了几个
`.so`”。同一套 app/libfoo/libbar/libc fixture 必须在每个目标 profile 上得到一致的
依赖图、符号 owner、构造/析构顺序和整组回收结果；每种允许的 relocation 还必须有
真实工具链产物和精确 golden gate。

> **2026-09-09 实现状态。** 本文第 1 节保留 2026-09-05 的历史基线，不能再当成当前
> 代码说明。当前已完成 C30/C31 的主链路：manifest-closed package、requester-aware
> private/system resolver、跨映像 scope、SCC lifecycle、system DSO registry ownership、
> `Initializing → Ready` 批量发布、counted dependency lease、quiescent system fini/unload、
> group reap 和多 DSO emutls 都已接入真实 ELF/QEMU。并发 gate 会同时启动两个复杂应用，
> 验证同一 system DSO 的首次装入/复用和最后 lease 释放。
>
> 当前运行验证如下：
>
> | profile | 当前状态 |
> | --- | --- |
> | Thumb v7-M soft-float / MPS2 | 多 DSO、scope、cycle、并发、emutls 通过；board `check_all` 通过 |
> | Thumb v8-M hard-float / MPS3 | 同一动态 scope 纵向门禁通过 |
> | RV64 / RV32 | 同一动态 scope 纵向门禁通过 |
> | AArch64 | producer、ELF gate 和 backend 已接入；动态运行门禁仍未闭环 |
>
> 这意味着 loader 已经从 Phase 1 的单 app/libc 演示进入可工作的多 DSO 运行时，但
> **Phase 2 尚不能整体标记完成**。主要缺口仍是 AArch64 运行闭环、硬件级页权限/W^X/
> RELRO（当前 flat image backend 只能记录逻辑权限）、完整 fault-injection/golden 矩阵
> 以及 C++ 纵向 fixture。`dlopen/dlsym/dlclose`、原生 ELF TLS、签名与 OTA 仍按本计划
> 留在后续阶段。
>
> 本轮还收口了三项会干扰门禁判断的问题：测试映像不再无条件启动 bootstrap shell；
> QEMU action 使用有界 Ninja pool，避免多个 TCG 实例争抢宿主机后触发假 inactivity
> timeout；动态 ELF 的最大页对齐固定为 4 KiB，避免小 DSO 因 64 KiB segment 间距在
> 4 MiB SRAM 的 MPS3 上并发装载 OOM。

## 1. 当前基线判断

### 1.1 已经存在、应直接复用的链接核心

当前 `kernel/loader/src/dynamic_linker/` 已经具备：

- 按 `DT_NEEDED` encounter order 展开的有界 BFS；
- `ArtifactIdentity` 优先、SONAME 冲突次之的双重去重；
- 菱形边保留、循环依赖 SCC 和 dependency-first 顺序；
- application scope 与 system scope；
- global/weak、hidden/internal、protected self-first 的纯链接语义；
- root `DT_PREINIT_ARRAY`、`DT_INIT/INIT_ARRAY` 和逆序 `FINI_ARRAY/DT_FINI`；
- imported Ready image 不重复 map、relocate、seal 或 init；
- session-wide relocation preflight、rollback、seal 和原子 publication；
- ARM32 NOW relocation：`R_ARM_RELATIVE/R_ARM_ABS32/R_ARM_GLOB_DAT/`
  `R_ARM_JUMP_SLOT`。

因此 C30/C31 的原则是“接入真实应用包和补齐生命周期所有权，再用真实 ELF 证明已有
算法”，不是复制一套 graph、scope 或 relocation engine。若 fixture 暴露纯链接 bug，
应在上述模块中修复通用实现，不能在 kernel resolver 中加入符号解析特例。

### 1.2 当前只支持 Phase 1 的接入边界

当前生产路径仍有以下单一 app/libc 假设：

1. `ApplicationArtifactResolver` 只查询 `SystemLibraryPaths`，没有应用包私有 `lib/`
   catalog；
2. `ApplicationService::prepare()` 固定选择
   `LoadProfile::arm_thumb_soft_float(ElfType::Dyn)`；
3. `ApplicationLoader` 固定构造 `DynamicLinker::new(ArmRelocator)`；
4. `check_blueos_elf.py` 固定解析 ARM relocation 名称并固定 soft-float ABI；
5. `librs/start/` 只有 ARM32 start object；
6. `KernelLinkReceipt` 仍把首次装入的 system allocation 挂在首个发布它的
   `ThreadGroup` 上，而不是交给 registry instance；
7. registry 在 link publication 后立即将 system DSO 标记为 `Ready`，早于
   `__librs_start_main` 真正运行其 constructor；
8. application 的 fini plan 目前仍可能包含本次首次装入的 system DSO fini。多应用
   并发后，首个应用退出时绝不能析构仍被其他组使用的 system DSO。

第 6–8 项在 Phase 1 的单一、保守驻留路径上不一定立即显现，但进入多 DSO 后会造成
错误复用、错误析构或 backing ownership 不清。它们是 C31 的硬修复项。

### 1.3 其余架构目前只是单映像骨架

`LoadProfile` 已经有 ARM hard-float、RV32、RV64 和 AArch64 构造器；单映像兼容入口也
能选择相应 relocator。但动态链接会 fail closed：

- `Riscv32Relocator/Riscv64Relocator/AArch64Relocator` 尚未实现
  `classify_relocation()`；
- session `RelocationPolicy` 只对 ARM 开放 NOW 类型；
- cache contract 虽有架构分支，但 cache-enabled Cortex-M 明确返回不支持；
- 应用内存后端尚未按 AArch64 页粒度证明真实 W^X/RELRO；
- 构建 gate、ABI note、start object、应用文件打包和 QEMU 测试尚未跨架构收口。

所以不能把已有类型名或单映像 `R_*_RELATIVE` 支持计入 C33–C35 完成度。

## 2. Phase 2 范围

### 2.1 本阶段必须交付

- 构建期生成、运行期只读的应用包 artifact catalog；
- root 与多个 app-private DSO 的精确路径、SONAME、build-id、目标 profile 和依赖声明；
- requester-aware 的 private/system 解析策略；
- private DSO 只在所属 `ThreadGroup` 中可见、保活和整组回收；
- identity/SONAME 双去重、菱形依赖、私有循环和 system 依赖闭包；
- weak/hidden/protected 与 application/system scope 的真实 ELF 验证；
- 多 DSO init/fini、system DSO `Initializing → Ready` 发布边界和失败隔离；
- 多 DSO emutls control object 保活与所有线程退出后的 destructor 顺序；
- system DSO instance 自己拥有 backing allocation、descriptor、fini plan 和依赖 lease；
- ARM32 v7-M soft-float 多 DSO QEMU 闭环；
- ARM32 v8-M hard-float、RV64、AArch64、RV32 的 artifact profile、relocation、cache、
  start object、文件打包和 QEMU/板上 smoke；
- 所有目标共享一套 graph/scope/lifecycle oracle；
- 每个允许 relocation 的 golden artifact 和每个拒绝项的 negative fixture。

### 2.2 明确不交付

- `dlopen/dlsym/dlclose` 和运行中增量修改 scope；
- 单个 app-private DSO 的提前卸载；
- lazy binding、PLT resolver trampoline；
- `IFUNC/IRELATIVE/COPY/TEXTREL`；
- 原生 ELF TLS、TLSDESC 和 TLS relocation；
- symbol version、audit namespace、`LD_PRELOAD`、`RPATH/RUNPATH`；
- 任意目录搜索、环境变量库路径和当前工作目录 fallback；
- 外部可变应用安装、签名、OTA 与降级策略；这些属于 Phase 3；
- 将当前共享特权地址空间描述成进程隔离；
- 默认 ARM32 profile 的运行期 text patch 或 branch veneer。

Phase 2 的“整组回收”是 `ThreadGroup` 退出时回收 root 和全部 app-private DSO。system DSO
只通过内部 registry quiescence 协议析构/卸载；这不等同于向应用提供 `dlclose`。

## 3. 完成定义和进入门禁

### 3.1 开始 C30 的硬条件

- Phase 1 的 `dynamic_app_vertical` 在 `qemu_mps2_an385` 上稳定通过；
- 缺 root、缺依赖和 relocation 失败均能 clean rollback；
- 第二个应用真实 import 同一个 Ready libc descriptor；
- `ApplicationStartStorage` 在整个应用生命周期内固定地址；
- Phase 0/0.5 loader 单测和 clippy 无回退。

### 3.2 开始 C32–C35 的共同条件

- C30/C31 已在 ARM32 v7-M 完成多 DSO 闭环；
- 架构无关 fixture 已生成期望 graph/symbol/lifecycle snapshot；
- `check_blueos_elf.py` 已改成数据驱动的 target profile gate；
- `ApplicationService` 不再硬编码 ARM profile/relocator；
- start-info 中的 `usize`/pointer 数组由目标产物本身生成和消费，没有 host-width
  序列化；
- system DSO constructor 未完成前不会被第二个 link import。

### 3.3 Phase 2 唯一完成门禁

C35 后，以下 profile 全部通过同一语义测试集，Phase 2 才完成：

| profile id | board/QEMU 主门禁 | ELF/ABI | 动态 relocation |
| --- | --- | --- | --- |
| `thumbv7m-vivo-blueos-newlibeabi` | `qemu_mps2_an385` | ELF32、ARM、Thumb、EABI5 soft | ARM32 基线四类 |
| `thumbv8m.main-vivo-blueos-newlibeabihf` | `qemu_mps3_an547` | ELF32、ARM、Thumb、EABI5 hard | ARM32 基线四类 |
| `riscv64-vivo-blueos` | `qemu_riscv64` | ELF64、LE、RV64/LP64、非 RVE | RELATIVE/64/JUMP_SLOT |
| `aarch64-vivo-blueos` | `qemu_virt64_aarch64` | ELF64、LE、AArch64 | RELATIVE/ABS64/GLOB_DAT/JUMP_SLOT |
| `riscv32-vivo-blueos-{imac,imc}` | `qemu_riscv32` + 对应板 | ELF32、LE、ILP32、ISA 精确匹配 | RELATIVE/32/JUMP_SLOT |

若某块目标板暂时没有可运行环境，该 profile 只能标记为“artifact/backend ready”，不能
替代 Phase 2 完成项；至少需要 QEMU 或真实板中的一个执行门禁以及真实产物 golden。

## 4. Phase 2 不变量

1. 一个启动事务只有一张 closed dependency graph；resolver 不另建可产生不同语义的图。
2. 相同 `ArtifactIdentity` 在一个 session 只 map 一次；相同 SONAME 对应不同 identity
   必须确定性失败。
3. root 和 app-private DSO 均为 `SessionPrivate`，只由所属 `ThreadGroup` receipt 保活。
4. system DSO 由 registry instance 保活，应用组只持 generation-bound lease。
5. system requester 只能在 system catalog 中解析依赖，不能看到 root/private symbols。
6. application requester 按“root → private BFS → system”解析符号；system DSO relocation
   永远使用 system scope。
7. `RPATH/RUNPATH` 即使出现在 ELF 中也不参与搜索，并由 artifact gate/loader 拒绝。
8. 所有 relocation 在首次写入前完成 session-wide preflight；未知类型不跳过。
9. constructor 在 loader、registry 和 manager mutex 外执行。
10. system candidate 只有 constructor 成功并收到 `ApplicationInitComplete` 后才可见为
    `Ready`。
11. app exit 只执行 root/private fini；system fini 只由 registry reaper 执行。
12. private image 在最后组线程完成 pthread key/emutls destructor 前不得释放。
13. lease `Drop` 只改变计数/投递 quiescence，不执行 fini、不做 cache 操作、不 unmap。
14. cache sync 的 scope 必须覆盖之后可能执行代码的全部 hart/core；否则 profile fail
    closed 或明确固定执行 affinity。
15. Phase 2 仍只接受可信固件内置 catalog；build-id/ABI note 不是签名或隔离证明。

## 5. 建议源码布局

在现有结构上增量形成：

```text
build/
  config/dynamic_target_profiles.gni       # profile id → ABI/reloc/start/cache policy
  templates/dynamic_app.gni
  templates/blueos_dso.gni
  templates/blueos_app_package.gni         # root + private DSO + manifest + filesystem layout
  scripts/check_blueos_elf.py               # 数据驱动多架构 gate
  scripts/gen_blueos_app_manifest.py        # 读取真实 ELF 后生成闭包清单
  scripts/check_blueos_app_package.py        # 清单与真实 ELF 双向一致

kernel/loader/src/
  dynamic_linker/
    artifact.rs                 # requester id/ownership，明确解析上下文
    graph.rs                    # 复用现有 BFS/SCC，仅补诊断/只读快照
    scope.rs                    # 复用双 scope，补真实 artifact 语义 gate
    lifecycle.rs                # plan 按 ownership/image 分区
    relocate.rs                 # profile → relocation policy
  relocation/
    arm.rs
    riscv.rs
    aarch64.rs

kernel/kernel/src/application/
  package.rs                    # ApplicationPackageCatalog/Manifest/ImageEntry
  loader.rs                     # profile/backend 分派和 staged link
  publication.rs                # private receipt 与 system instance backing 分离
  registry.rs                   # batch acquire、Initializing/Failed、instance ownership
  reaper.rs                     # private group reap + system quiescence work
  adapters/
    resolver.rs                 # PackageArtifactResolver
    system_paths.rs             # system artifact catalog + declared dependencies
    code_cache.rs               # kernel/board 的 CodeCache adapter
    flat_memory.rs              # 无 MMU profile
    paged_memory.rs             # AArch64 页粒度 profile

librs/start/
  arm/blueos_scrt1.S
  riscv/blueos_scrt1.S
  aarch64/blueos_scrt1.S

apps/example/dynamic/multi_dso/
  app/
  libfoo/
  libbar/
  libcommon/
  expected/
```

命名不要求一次重排现有 application 目录；边界比目录名重要。不要为满足本图移动大量
已经稳定的 Phase 1 文件。

## 6. 应用包和 artifact contract

### 6.1 Phase 2 的包模型

Phase 2 不扫描一个可写目录来猜测库。构建系统为每个内置应用生成只读 manifest，并由
board image 将其与 artifact 放到固定位置：

```text
/apps/multi/app.elf
/apps/multi/lib/libfoo.so.1
/apps/multi/lib/libbar.so.1
/apps/multi/lib/libcommon.so.1
/system/lib/<profile>/libc.so.1
```

建议的内核静态视图：

```rust
pub struct ApplicationPackageManifest {
    pub package_id: &'static [u8],
    pub profile: DynamicTargetProfileId,
    pub root: PackageImageEntry,
    pub private_images: &'static [PackageImageEntry],
}

pub struct PackageImageEntry {
    pub role: PackageImageRole,       // Root | PrivateDso
    pub path: &'static str,
    pub soname: Option<&'static [u8]>,
    pub build_id: &'static [u8],
    pub needed: &'static [DeclaredDependency],
}

pub struct DeclaredDependency {
    pub soname: &'static [u8],
    pub source: DependencySource,     // PackagePrivate | System
}
```

这不是最终安装 manifest，也不承担密码学信任。它只把固件构建时已经验证的闭包和运行时
解析策略冻结下来；Phase 3 再加入签名/hash/最低 ABI/升级 generation。

### 6.2 构建期双向验证

`gen_blueos_app_manifest.py` 必须从最终 ELF 读取而不是从 GN 参数猜测：

- root 的 `DT_NEEDED`；
- 每个 private DSO 的 `DT_SONAME/DT_NEEDED`；
- build-id 和 `.note.blueos.abi`；
- ELF class/data/machine/flags；
- dynamic tags 和 relocation 类型集合。

`check_blueos_app_package.py` 再做双向校验：

- manifest 声明的每个 image 都存在且身份匹配；
- ELF 中每条 `DT_NEEDED` 都有且只有一个 manifest binding；
- manifest 没有 ELF 不使用的幽灵依赖；
- private SONAME 在包内唯一；
- private SONAME 不得与 system catalog 的保留 SONAME 冲突；
- root 不带 SONAME，private DSO 必须带 SONAME；
- private DSO 必须动态依赖唯一 system `libc.so.1`，不能静态嵌入 librs；
- 所有 image 使用同一 target profile 和 start/syscall ABI generation；
- `RPATH/RUNPATH/TEXTREL/PT_TLS` 等 Phase 2 拒绝项不存在。

### 6.3 运行期仍重新验证

运行期不能只信 manifest。`VfsElfReader` 对打开的 snapshot 生成
`ArtifactIdentity`，loader 仍从 `PT_DYNAMIC` 解码 SONAME/NEEDED/note/relocation 并与
manifest 逐项比较。路径、manifest entry、snapshot identity 和 reader 必须来自同一
解析决定；禁止先按路径检查、再重新打开另一个 generation。

## 7. C30：应用私有依赖闭包

### 7.1 扩展 resolver 请求上下文

当前 `DependencyRequest` 只有 requester identity、needed 和 domain。C30 增加只读上下文：

```rust
pub struct DependencyRequester<'a> {
    pub image: ImageId,
    pub identity: &'a ArtifactIdentity,
    pub ownership: ImageOwnership,
}

pub struct DependencyRequest<'a> {
    requester: DependencyRequester<'a>,
    needed: &'a DependencyName,
    domain: LinkDomainId,
}
```

这样 kernel resolver 不需要用错误的 `requester: 0` 填诊断，也无需从 identity 猜测
system/private 边界。该扩展不把 VFS、路径或 registry 类型带入 loader crate。

### 7.2 确定性解析规则

`PackageArtifactResolver::resolve()` 按以下固定顺序工作：

1. 校验 requester 属于本次 prepared package/system batch；
2. 若 requester 是 `SystemCandidate/ExternalReady`，只查 system catalog；
3. 若 requester 是 root/private，查询 manifest 中这条精确 edge；
4. `DependencySource::PackagePrivate` 只打开 manifest 指定的包内绝对路径；
5. `DependencySource::System` 只消费 registry batch 预先给出的 Load/Import 决定；
6. 同一个 private entry 的第二次请求返回相同 snapshot identity，由 loader 记录额外边；
7. 实际 SONAME/NEEDED 与 manifest 不符时中止整个 session。

任何未声明依赖、路径穿越、重复 SONAME、private 对 system 名称的覆盖或 system 对 private
库的反向依赖均返回结构化错误。不得 fallback 到 `/apps/<name>/lib` 扫描、cwd、
`LD_LIBRARY_PATH` 或 `RPATH/RUNPATH`。

### 7.3 system dependency 的原子 batch acquire

多 system DSO 不能继续逐 SONAME“边发现边拿 permit”。两个并发 session 分别先拿到 A/B
再等待 B/A 会形成 ABBA。Phase 2 利用可信 manifest 预先知道 system closure，增加：

```rust
pub enum AcquireBatchOutcome {
    Acquired(PreparedSystemBatch),
    Pending(SystemBatchWait),
}

pub struct PreparedSystemBatch {
    loads: Vec<SystemCandidatePermit>,
    imports: Vec<(SystemDsoLease, PublishedImageDescriptor)>,
}
```

`SystemDsoRegistry::acquire_batch()` 在一个短 mutex 临界区中按 SONAME byte order 检查整个
集合：

- 全部槽为 `Vacant/Ready` 时，原子地产生所有 permit/lease；
- 任一槽为 `Loading/Relocated/Initializing/Quiescing` 时，不改变任何槽和计数，返回
  wait ticket；
- waiter 在锁外等待，醒来后重新计算并重试整个 batch；
- 任一 permit 在 publication 前 drop，整批未提交 candidate 都回到 `Vacant` 并唤醒
  waiter；
- generation 必须逐 slot 验证，不能只保存一个全局 epoch。

manifest 的 system closure 与真实 ELF closure 在 `close_dependencies()` 后再次对比；少边、
多边或角色不同都回滚。这一设计同时覆盖 system SCC，避免为循环依赖引入锁顺序特例。

### 7.4 双重去重和冲突语义

保留现有顺序：

1. 完整 `ArtifactIdentity` 相同：复用已有 `ImageId`，仍记录当前 dependency edge；
2. identity 不同但 SONAME 相同：`IdentityConflict`；
3. identity 相同但声明不同 SONAME：`BadElf`；
4. `DT_NEEDED` 名称与 provider `DT_SONAME` 不同：`SonameMismatch`；
5. root 不因路径别名被第二次装入；若循环边指回 root，只记录边；
6. build-id 相同但 VFS snapshot identity 不同不自动等价。

manifest compiler 应尽早拒绝明显冲突，loader 仍保留上述运行期 fail-closed 检查。

### 7.5 private allocation 所有权

root、libfoo、libbar 和 libcommon 都以 `ImageOwnership::SessionPrivate` 进入现有 rollback
log。成功 publication 后，它们全部进入 `KernelLinkReceipt.private_allocations`；失败时按
allocation 逆序 abort；正常退出时在 group 空、fini 和线程 destructor 完成后一起
`release_committed`。

不增加 private registry、引用计数或单 DSO handle。包内菱形只共享本 group 中的同一个
image；另一个 `ThreadGroup` 启动同一包时仍得到自己的 private RW/GOT/constructor state。

### 7.6 C30 子提交和 gate

| 子提交 | 主要修改 | 必须通过 |
| --- | --- | --- |
| C30-a | build package template、manifest generator/checker、foo/bar/common fixture | 最终 ELF 闭包与 manifest 完全一致；保留 system SONAME 冲突被拒绝 |
| C30-b | `DependencyRequest` requester context、错误上下文和 graph snapshot | 现有 loader tests 不回退；错误包含 requester/image/needed |
| C30-c | `ApplicationPackageCatalog`、composite resolver、registry batch acquire | app→foo→libc、菱形、private cycle、system cycle 并发无死锁 |
| C30-d | receipt/private reap、ARM32 v7-M QEMU fixture | 每个 private identity map 一次；组退出后全部 private allocation exactly once release |

C30 完成时可以暂用已有 scope/lifecycle 实现，但必须已经运行真实多 DSO；不能只提交 fake
reader 图测试。

## 8. C31：多 DSO 可见性、初始化和回收语义

### 8.1 冻结 scope 顺序

application scope 固定为：

```text
root
→ app-private images（BFS discovery_index）
→ system images（BFS discovery_index）
```

system requester 的 scope 只包含 system images。规则如下：

- `STB_GLOBAL`：scope 中第一个可导出的 strong definition 胜出；
- `STB_WEAK`：没有 strong 时选择 scope 中第一个 weak definition；
- undefined weak data relocation 绑定 0；
- undefined weak control-flow relocation继续 fail closed；
- `STV_HIDDEN/STV_INTERNAL` 不进入其他 image 的 lookup；
- `STV_PROTECTED` 对本 image 内引用 self-first，对其他 image 仍可导出；
- `STB_LOCAL` 永远按 owner+symbol index 解析；
- app 可以按 application scope interpose private 默认可见符号，但绝不能 interpose system
  DSO 自己的 relocation。

现有 `ScopeSet` 已覆盖大部分规则。C31 的主要工作是补真实 GNU/SysV hash artifact、
owner oracle 和冲突诊断；只有失败证据出现时才修改算法。

### 8.2 lifecycle plan 必须按 ownership 分区

将当前完整 plan 明确拆成三种用途：

```rust
pub struct LifecyclePlans {
    pub startup: InitPlan,             // 本 session 新装入的 system + 全部 private
    pub group_fini: FiniPlan,          // 仅 root/private
    pub system_fini: Vec<ImageFiniPlan>, // 每个新 system candidate，交给 registry
}
```

顺序保持：

1. root `DT_PREINIT_ARRAY`；
2. dependency-first SCC；
3. 每个 image 的 `DT_INIT`；
4. 同 image 的 `DT_INIT_ARRAY` 正序；
5. fini 为 image 顺序精确逆序，单 image 内 `FINI_ARRAY` 逆序后 `DT_FINI`。

同一个 SCC 内采用现有稳定 discovery order，并把该选择写入 snapshot；不要声称循环中的
constructor 存在工具链之外的语义顺序。imported Ready system image 不进入 startup/system
fini；它已经由所属 registry generation 初始化。

### 8.3 修正 system DSO publication 状态机

registry 状态扩展为：

```text
Vacant
  └─ acquire_batch → Loading
       └─ link/seal/publish_backing_batch → Relocated
            └─ install start storage → Initializing
                 ├─ ApplicationInitComplete → Ready
                 └─ thread fault/abort      → Failed → Quiescing → Vacant|CachedFailed

Ready --last lease/use/dependent--> Quiescing --evidence--> Vacant|Ready(cached)
```

关键接口建议为：

```rust
registry.publish_relocated_batch(permits, backings) -> SystemInitBatch
group.install_pending_system_batch(SystemInitBatch)
registry.finish_initialization_batch(batch) -> Vec<SystemDsoLease>
registry.fail_initialization_batch(batch, reason) -> FailedSystemBatch
```

`ApplicationInitComplete` 必须先完成整个 system batch 的 `Initializing → Ready` 和首个
group lease mint，再把 application manager 从 `Loading` 置为 `Running`。所有 token 已在
link/publish 阶段验证，因此成功路径的最终状态切换应只包含不分配的原子 move；若仍出现
stale generation，视为内核一致性错误并终止该组，不能让 main 继续。

并发 waiter 在 `Initializing` 上等待，绝不能取得 descriptor。constructor 执行期间没有
registry mutex；完成 syscall 才在短锁内一次发布整个 batch，从而不暴露半初始化 SCC。

### 8.4 system instance 拥有 backing，而不是首个应用

引入 registry-owned instance：

```rust
pub struct SystemDsoInstance {
    generation: u32,
    descriptor: PublishedImageDescriptor,
    allocation: AllocationLease,
    fini: FiniPlan,
    dependencies: Vec<SystemDsoLease>,
    users: usize,
    use_tokens: usize,
}
```

publication receipt 的职责改为：

- `KernelLinkReceipt`：private allocations + 当前 group 的 system leases；
- `SystemDsoInstance`：system candidate allocation + descriptor + 自身 fini + system
  dependency leases；
- `SystemInitBatch`：在 constructor 完成前暂存上述待发布 backing；
- reaper：消费 private receipt；registry worker：消费 quiescent system instance。

首次加载 system DSO 的应用也必须取得普通 `SystemDsoLease`，不能以“它拥有 raw
allocation”为隐式引用。这样首发组和后续 import 组遵循同一计数语义。

### 8.5 system 依赖边和 SCC 卸载

一个 cached/Ready system DSO 即使没有 application user，只要仍保留 relocation 到另一
system DSO，就必须持有后者的 dependency lease。registry 保存 system 子图，并按 SCC
作为最小 publication/quiescence 单元：

- batch 内 SCC 原子进入 Ready；
- SCC 内部边只作为结构关系记录，不产生会让环永远无法归零的自持 lease；
- SCC 指向其他 SCC 的外向 dependency lease 在源 SCC 存活期间保持；
- SCC 的 application users、kernel use token 和反向 dependent 全为零后才进入
  `Quiescing`；
- system fini 在专用 lifecycle/reaper thread 上、所有 registry lock 外执行；
- fini 完成、cache/执行静默证据成立后才 release allocation；
- 不能证明 callback/function pointer 不可达时转为 `Ready(cached, users=0)`。

Phase 2 至少为无逃逸的测试 system DSO 证明 unload→generation+1 reload。共享 libc 若仍
存在无法建模的 kernel callback 或全局裸函数指针，则必须明确 KeepCached，不能仅凭
refcount 强制卸载。

### 8.6 constructor 异常边界

C constructor 没有返回错误；Phase 2 的失败来源主要是 fault、主线程在
`ApplicationInitComplete` 前退出，或 start-info 校验失败。此时：

- application 公共状态进入 `Failed/Draining`；
- private fini 只对确认完成初始化的 image/entry 执行；无法确认时跳过并记录原因；
- system batch 不进入 Ready，waiter 被唤醒并可在 generation+1 重试；
- 已发生的 IO/外部副作用不宣称可回滚；
- candidate backing 只有在执行栈和线程都静默后释放；
- fault injection 验证不会留下可 import 的半成品 descriptor。

如需精确知道完成到哪个 constructor，可向 start ABI 尾部追加只读 plan id，并由 librs
逐 entry 报告进度；Phase 2 基线不要求每个 ctor 一个 syscall，但必须选择并记录“保守
跳过 fini”或“精确进度”策略，不能猜测。

### 8.7 多 DSO emutls

继续使用编译器 emutls，不引入 `PT_TLS`：

- 每个 `EmutlsControl` 是其定义 image 的普通 data symbol；
- private control object 地址在整个 `ThreadGroup` 生命周期内稳定；
- 所有 app/private DSO 的 `__emutls_get_address` 必须解析到唯一 system libc；
- private DSO 不得静态带第二份 librs/emutls runtime；
- 子线程继承 `LibcApplicationContext`，每线程拥有独立 value；
- pthread key/emutls destructor 全部完成后才能 release private image；
- system DSO control object 只有在所有引用其 generation 的线程清理后才允许 quiesce。

增加两个线程同时访问 foo/bar 中同名 TLS 变量、循环创建/退出线程和整个应用重复启动的
fixture，验证 control identity、value isolation 和 destructor 次数。

### 8.8 C31 子提交和 gate

| 子提交 | 主要修改 | 必须通过 |
| --- | --- | --- |
| C31-a | scope/visibility 真实 ELF corpus 和 owner snapshot | strong/weak/hidden/protected、app interpose、system non-interpose |
| C31-b | lifecycle ownership partition、SCC snapshot | init/fini 与 oracle 完全一致；app exit 不运行 system fini |
| C31-c | registry `Initializing/Failed`、batch publication、instance backing | waiter 只见 Ready；首发组也持 lease；ctor fault 不发布半成品 |
| C31-d | emutls、group/system reaper 和 QEMU 压力测试 | 私有整组回收、system cached/unload/reload、线程 destructor 次数正确 |

## 9. 架构公共改造

### 9.1 profile 由 board policy 唯一选择

新增不来自 ELF 自报值的 `DynamicTargetProfileId`，由 board/product 配置传入：

```rust
pub struct DynamicExecutionProfile {
    pub id: DynamicTargetProfileId,
    pub load: LoadProfile,
    pub relocation: DynamicRelocationProfile,
    pub cache: CacheRequirements,
    pub memory: ImageMemoryProfile,
    pub start_abi: u32,
}
```

`ApplicationService` 从 package manifest 取得 profile id，再要求它等于当前 board 允许的
profile；不能根据 ELF machine 自动选择并放宽 ABI。`ApplicationLoader` 用一个实现
`ArchRelocator` 的小型 enum adapter 或泛型 helper 分派到 ARM/RV32/RV64/AArch64，整个
staged pipeline 只写一次。

### 9.2 relocation policy 数据驱动

把 session 中“只有 ARM 开放”的逻辑改为精确 profile 表：

| profile | kind → raw relocation |
| --- | --- |
| ARM32 | Relative=`R_ARM_RELATIVE`；Absolute=`R_ARM_ABS32`；GlobalData=`R_ARM_GLOB_DAT`；JumpSlot=`R_ARM_JUMP_SLOT` |
| RV64 | Relative=`R_RISCV_RELATIVE`；Absolute=`R_RISCV_64`；JumpSlot=`R_RISCV_JUMP_SLOT` |
| AArch64 | Relative=`R_AARCH64_RELATIVE`；Absolute=`R_AARCH64_ABS64`；GlobalData=`R_AARCH64_GLOB_DAT`；JumpSlot=`R_AARCH64_JUMP_SLOT` |
| RV32 | Relative=`R_RISCV_RELATIVE`；Absolute=`R_RISCV_32`；JumpSlot=`R_RISCV_JUMP_SLOT` |

仍使用通用 `RelocationKind`、symbol scope、target owner/range/permission 校验和三阶段 apply。
各 relocator 只负责 raw type 分类、REL/RELA addend、class/machine 和必要的架构值规范化。

### 9.3 每种 relocation 的共同 golden 内容

每个 raw type 至少包含：

- 正常 `B+A` 或 `S+A/S` 结果；
- 非零正/负 addend；
- 最大合法 target word 和 overflow；
- 未对齐 target；
- target 不属于 owner allocation；
- 写入 RX/RO 而不是 writable region；
- function target 不在 executable region；
- data target 不在 provider readable region；
- undefined strong、undefined weak data、undefined weak control flow；
- imported Ready provider 与本 session provider 的相同结果；
- apply 中途 backend failure 的完整 rollback。

golden 输入优先使用 clang/LLD 生成的最小 C/assembly ELF；手工 byte fixture 只补工具链
难以稳定产生的恶意边界，不能取代真实产物。

### 9.4 cache 与调度 scope

把真正的 cache 操作放在 kernel/board adapter，loader crate 只保留
`CodeCache/PreparedCacheSync` contract：

- ARM v7-M cache-less：全部 X range + DSB/ISB，报告 `BarrierOnly`；
- cache-enabled Cortex-M：逐 line D-cache clean、I-cache invalidate、DSB/ISB；
- RV32/RV64：`fence.i` 只覆盖当前 hart，若线程可迁移则做跨 hart rendezvous，或在
  publication 到首次执行期间固定 affinity；
- AArch64：按 `CTR_EL0` line size 做 D clean to PoU、DSB、I invalidate to PoU、DSB/ISB；
- SMP backend 返回 `AllExecutionContexts` 前必须有 IPI/rendezvous 证据，不能把本地指令
  冒充全局同步。

seal 只有在 cache outcome 精确覆盖全部 executable range 后才允许 publication。

## 10. C32：ARM32 Thumb v8-M hard-float

### 10.1 artifact profile

新增 `thumbv8m.main-vivo-blueos-newlibeabihf`，冻结：

- ELF32、小端、`EM_ARM`、EABI5；
- `EF_ARM_ABI_FLOAT_HARD`，拒绝 soft-float image；
- Thumb entry bit、`.ARM.attributes` 中的 CPU/ISA/VFP/argument ABI；
- app/DSO/sysroot/start object 全部使用同一 hard-float target；
- 基线仍只接受 ARM NOW 四类 relocation。

soft/hard 组合形成 2×2 拒绝矩阵：hard kernel 不装 soft app/DSO，soft kernel 不装 hard
app/DSO；同一 dependency closure 中也不能混用。

### 10.2 cache-enabled Cortex-M

在 `qemu_mps3_an547` 对应能力下实现真实 cache backend。若 QEMU 型号没有启用某级
cache，仍要用 capability probe/board config 明确选择 barrier-only；真实 cache-enabled
板必须覆盖 clean/invalidate。对 range 做 line-size 向下/向上对齐时使用 checked
arithmetic，并验证首尾越界和空 range。

### 10.3 start 和调用 ABI

ARM start object 可复用同一源码，但必须分别以 soft/hard target 组装并检查：

- `_start` 为 Thumb function；
- `r0` 传 start-info，重排后调用 `__librs_start_main(main, info)`；
- SP 满足 AAPCS 对齐；
- hard-float 属性一致，含浮点参数的 foo→libc/app 跨 DSO 调用结果正确；
- function pointer 保留 Thumb bit。

### 10.4 C32 gate

- hard/soft artifact reject matrix；
- float/double 参数与返回值跨 app→foo→libc；
- ARM 四类 relocation golden；
- cache line 首尾、多 X segment、零长度和 backend failure；
- `qemu_mps3_an547` 完整 multi-DSO fixture；
- `qemu_mps2_an385` Phase 1/Phase 2 soft-float fixture不回退。

## 11. C32V：可选 ARM32 Thumb branch veneer

C32V 只有在真实产物仍产生 `R_ARM_THM_CALL/R_ARM_THM_JUMP24`，且无法通过 PIC/GOT/PLT
模板消除时才启动。默认 `PltGotOnly` profile 继续稳定拒绝它们。

启用 `LoaderVeneerV1` 时必须在 relocation apply 前完成全局规划：

- 收集全部 branch relocation 和 source PC；
- 为每个 target/可达窗口去重 veneer；
- 预留 near executable island，证明每个 branch 到 island 可编码；
- 先生成/relocate veneer，再一次性 patch writable callsite；
- veneer allocation 进入 session rollback 和最终 ownership；
- island seal 为 RX 并参与 cache sync；
- 资源耗尽、不可写 Flash callsite、超范围和中途 failure 全量回滚。

禁止 relocation 走到一半才临时分配 island，也禁止为支持 veneer 放开普通 TEXTREL。
C32V 不阻塞 C33–C35 和 Phase 2 基线完成。

## 12. C33：RISC-V64

### 12.1 relocator

`Riscv64Relocator` 增加：

- `R_RISCV_RELATIVE → Relative`；
- `R_RISCV_64 → Absolute`；
- `R_RISCV_JUMP_SLOT → JumpSlot`；
- ELF64 target word、RELA explicit addend；
- 64 位 checked `B+A/S+A`；
- 未支持 `R_RISCV_CALL/PCREL_*/HI20/LO12/TLS/IRELATIVE` 确定性拒绝。

### 12.2 ABI/ISA gate

`.note.blueos.abi` 和 ELF attributes 固定 LP64、soft-float 及允许的 ISA extensions。
`EF_RISCV_RVE`、错误 XLEN、错误 float ABI 或 manifest/toolchain ISA 不一致均在 map 前拒绝。
不要只看 `e_machine=RISC-V`。

### 12.3 start/cache/board

新增 RISC-V start object：入口从 `a0` 接收 start-info，将 app `main` 地址和 info 按 ABI
传给 `__librs_start_main`，不依赖 kernel `gp`。启动前后验证 `sp` 16-byte alignment 和
必要的 `gp` 初始化策略。

`fence.i` 必须在最后一次 code write 后、首次执行前发生。`qemu_riscv64` 若多 hart 可迁移，
增加 cross-hart sync 或固定 affinity，并把真实 execution scope 写入 outcome。

### 12.4 C33 gate

- 三类 relocation golden 和全部反向类型；
- LP64/XLEN/float ABI/ISA mismatch；
- direct/indirect function、data、weak、ctor/fini；
- app→foo/bar→common/libc graph snapshot 与 ARM32 相同；
- `qemu_riscv64` 启动、两组并发、整组 reap、system reuse/reload；
- Phase 0/静态 RV64 kernel 回归。

## 13. C34：AArch64

### 13.1 relocator

`AArch64Relocator` 增加：

- `R_AARCH64_RELATIVE → Relative`；
- `R_AARCH64_ABS64 → Absolute`；
- `R_AARCH64_GLOB_DAT → GlobalData`；
- `R_AARCH64_JUMP_SLOT → JumpSlot`；
- ELF64 target word、RELA explicit addend；
- 拒绝 ADRP/ADD/LDST text relocation、TLSDESC、IRELATIVE 和 COPY。

### 13.2 page-granule memory backend

AArch64 不能继续只用 heap allocation 上的逻辑权限来完成 C34。增加页独占的 image
allocation：

1. 按最大 `p_align` 和页粒度预留 VA/物理 backing；
2. map/copy/zero 阶段只允许 loader 写；
3. relocation 完成后将 segment 权限收口为 R/RW/RX，RELRO 页转 R；
4. 禁止一个页混装需要不同最终权限的无关 allocation；
5. 页表更新执行 break-before-make/TLB/cache 所需序列；
6. rollback 恢复/撤销全部部分映射；
7. publication record 区分真实硬件权限与逻辑检查。

当前应用仍运行在共享特权地址空间；页权限硬化不等于进程隔离。

### 13.3 cache/start/SMP

新增 A64 start object，`x0=info`，按 AAPCS64 调用
`__librs_start_main(main, info)`，保持 SP 16-byte alignment。cache backend 按
`CTR_EL0` 计算 line size并执行 D clean/I invalidate 到 PoU。若应用可在其他 core 执行，
必须通过 shareability/cross-call 证明所有执行上下文已同步。

### 13.4 C34 gate

- 四类 relocation golden、负 addend、64 位 overflow；
- 页边界、重叠 segment、RELRO、W^X、rollback 和 TLB/cache 顺序；
- `qemu_virt64_aarch64` multi-DSO 语义 snapshot；
- 单核与多核/迁移策略的 cache capability gate；
- 现有 AArch64 static kernel/MMU/virt 回归。

## 14. C35：RISC-V32 IMAC/IMC

### 14.1 relocator 和 ABI

`Riscv32Relocator` 增加：

- `R_RISCV_RELATIVE → Relative`；
- `R_RISCV_32 → Absolute`；
- `R_RISCV_JUMP_SLOT → JumpSlot`；
- ELF32 target word、RELA explicit addend；
- 所有结果写入前做 32 位 overflow 检查；
- RVE、错误 float ABI、XLEN/ILP32 不一致稳定拒绝。

### 14.2 IMAC 与 IMC 必须分开

profile note 至少记录 XLEN、base ISA、C 和 A extension。IMC artifact 不能装到声明需要 A
的产品；IMAC runtime 也不能把原子指令存在当成通用同步前提。无 A 的 application/registry
同步继续通过 kernel syscall/调度器实现，不能让动态 libc 在用户代码路径直接发 A 指令。

### 14.3 C35 gate

- RV32 三类 relocation golden、最高地址和 32 位截断；
- IMAC/IMC 双向 ISA mismatch；
- `qemu_riscv32` 完整 graph/symbol/lifecycle fixture；
- 至少一个 IMAC 或 IMC 真实板 artifact/执行 smoke；
- 无 A 配置下 registry wait、pthread/emutls 和退出路径；
- RV64/AArch64/ARM snapshot 不回退。

## 15. 构建系统和 SDK 收口

### 15.1 `check_blueos_elf.py` 数据驱动化

增加 `--target-profile`，profile 表至少包含：

- class/data/machine/type；
- machine-specific `e_flags` mask/value；
- entry mode/alignment；
- ABI attribute/note 约束；
- dynamic relocation allowlist 和 text relocation denylist；
- start ABI/librs ABI generation；
- 是否要求 SONAME、PIE、NOW、RELRO、build-id。

relocation parser 不能再只匹配 `R_ARM_*`；应读取任意 `R_[A-Z0-9_]+` 后与当前 profile
精确比较。未知 relocation 必须出现在错误输出中。

### 15.2 模板接口

建议模板显式接收：

```gn
dynamic_app("multi") {
  target_profile = current_dynamic_target_profile
  start_object = "//librs:blueos_scrt1"
  private_dsos = [ ":foo", ":bar" ]
  system_needed = [ "libc.so.1" ]
}

blueos_dso("foo") {
  target_profile = current_dynamic_target_profile
  soname = "libfoo.so.1"
  system_needed = [ "libc.so.1" ]
  export_manifest = "foo.exports"
}
```

GN 参数用于建立 build edge；最终 contract 仍由实际 ELF checker 决定。每个普通 DSO
冻结最小 C export manifest，禁止导出 Rust mangled/internal symbol。

### 15.3 SDK/EDK 产物

每个 target profile 发布一致目录：

```text
sdk/<profile>/
  lib/libc.so                  # link name
  lib/libc.so.1                # runtime SONAME artifact
  lib/blueos_scrt1.o
  include/blueos/application.h
  manifests/libc.exports
  abi/<profile>.json
  debug/<build-id>/...
```

普通 private DSO 链接 SDK `libc.so`，运行时只记录 `DT_NEEDED: libc.so.1`。debug/build-id
归档要能把 QEMU fault PC 映射到 `image + offset + symbol`，但它不是签名数据库。

## 16. 跨架构 start/runtime 适配

`BlueOsApplicationStartInfo` 的 C layout 在每个目标上由目标编译器自然确定；不要把一个
ARM32 二进制 struct 序列化后给 ELF64 使用。稳定字段如 handle 继续用固定 `u32`，指针、
count、auxv value 使用目标 `usize`。

每个 start object 的共同 contract：

- 只接收 `ApplicationStartInfo *`，不假装 Linux initial stack；
- 取得当前 image 的 `main` 和 system libc 的 `__librs_start_main`；
- 使用 GOT/PLT 或当前 profile 允许的 relocation，不产生额外 text relocation；
- 保持目标 ABI 栈对齐、函数地址规范和非易失寄存器约束；
- tail-call 后不返回；
- readelf/disassembly gate 验证入口前若干指令和 relocation 类型。

`librs::__librs_start_main` 主体保持架构无关。架构差异只允许出现在 start object、syscall
shim、必要的 context switch/TLS glue 和 artifact config 中。

## 17. 测试 fixture 和统一 oracle

### 17.1 canonical dependency graph

建议固定一个比最小示例更完整的图：

```text
app
├─ foo ─┬─ common ─→ libc
│       └─ cycle_a ─→ cycle_b ─┐
│                    ↑         │
│                    └─────────┘
├─ bar ─── common
└─ libc
```

oracle 至少记录：

- node：logical name、ownership、SONAME、discovery index；
- edge：requester、needed index、provider；
- SCC 分组和 dependency-first 顺序；
- 每个 relocation 的 requester/symbol/provider owner；
- init 和 group/system fini 序列；
- allocation/release 计数；
- system generation、load/reuse/init/fini 次数。

地址因 ASLR/allocator 和位宽不同不能直接跨架构比较；比较规范化的 image id、symbol name、
owner 和 image-relative offset。

### 17.2 symbol fixture

至少包含：

- app strong 覆盖 private default-visible weak；
- foo strong 与 bar weak 的搜索顺序；
- hidden symbol 只能被 owner 访问；
- protected symbol 的 owner-local self binding；
- system DSO 中与 app 同名符号仍绑定 system owner；
- undefined weak data 为 0；
- undefined weak call 被拒绝；
- function/data symbol 都跨 DSO；
- GNU hash 和 SysV hash 各一份；
- 同 SONAME 不同 build/snapshot 的冲突。

### 17.3 lifecycle fixture

每个 image 的 ctor/fini 向有界 trace buffer 写入 `(generation, image, event, sequence)`；测试
比较完整序列并验证：

- shared dependency constructor 一次；
- private dependency 每个 group 一次；
- imported system constructor不重跑；
- app exit 不运行 system fini；
- 两组都退出后 system quiescence 才可能运行 system fini；
- reload generation 的 constructor 恰好再运行一次；
- private cycle 使用冻结的稳定 SCC 顺序，fini 精确逆序。

### 17.4 并发 fixture

- 两应用同时首次请求同一 system closure；
- 两应用以相反 manifest 顺序声明 system A/B；
- 一个 candidate 在 map/relocate/seal/init 各阶段失败，另一个 waiter 醒来重试；
- 一个组退出，另一个组仍执行 foo/libc；
- last lease 与新 acquire 竞争；
- quiescing 期间新 acquire；
- generation wrap/陈旧 token 的人工单测；
- reaper 与 application launch 并发，锁内不执行 VFS、constructor、fini 或 memory release。

## 18. 故障注入矩阵

至少在以下边界逐点失败：

| 阶段 | 注入点 | 必须保持的结果 |
| --- | --- | --- |
| package | manifest 缺项、重复 SONAME、角色错误 | 尚未获得 permit/allocate |
| batch acquire | reserve/lease capacity、Pending | 无部分计数、无部分 Loading |
| VFS | open、snapshot change、短读 | 全批 permit 取消，Ready instance 不受损 |
| graph | image/edge/depth/SCC quota | 所有新 allocation 逆序 abort |
| relocation | preflight、overflow、apply write | 无半发布 link map；已写 byte 回滚 |
| seal/cache | protect、cache range、scope mismatch | 不进入 Relocated/Initializing |
| publication | descriptor/backing handoff | private/system 所有权恰好一方持有 |
| init | entry fault、线程提前退出 | system batch Failed，waiter不见 Ready |
| exit | pthread destructor/fini 中断 | private backing 保活；记录 skipped/failed disposition |
| quiescence | use token/依赖 lease仍活跃 | KeepCached，不执行 system fini/unmap |
| unload | system fini/cache/unmap failure | generation 不可被新 Ready 复用，错误可审计 |

每个点都断言 allocation id、permit generation、lease count、group state 和 registry state，
不能只断言返回了 `Err`。

## 19. 性能和资源预算

Phase 2 在每个 profile 记录：

- 冷启动：VFS bytes、ELF decode、relocation count、cache time、constructor time；
- 热启动：system DSO import 命中率和避免的 map/relocate/init 次数；
- graph：images、edges、最大深度、SCC 数、symbol probes；
- 内存：private/system allocation、metadata、start storage、peak rollback log；
- 退出：等待线程、private reap、system quiescence/fini/unmap 时间；
- allocator：64 KiB/页/MPU 粒度对齐后的内部碎片。

建议首版 product limits 从实际 fixture 推导并留出明确裕量，例如 image/edge/depth/string/
relocation/symbol probe 各自有独立上限。不能为了让复杂 fixture 通过直接使用无限制
`SessionLimits::DEFAULT`。

回归门禁至少限制：相同 ARM32 Phase 1 hello 的冷启动时间、峰值 RAM 和 libc 热复用时间
不能出现未解释的大幅增长。阈值应由 CI 基线数据确定，不在代码中拍脑袋写死。

## 20. 实施依赖和顺序

```text
Phase 1 稳定基线
  │
  ├─ C30-a package producer/manifest
  ├─ C30-b resolver context/diagnostics
  │       └─ C30-c package resolver + batch acquire
  │               └─ C30-d ARM32 private graph/reap
  │
  └───────────────────────┐
                          v
          C31-a scope oracle
          C31-b lifecycle partition
          C31-c registry init publication/ownership
          C31-d emutls/quiescence/QEMU
                          │
             architecture-common profile/gate/start split
                          │
                          v
                 C32 hard-float ARM
                          │
                          v
                       C33 RV64
                          │
                          v
                       C34 A64
                          │
                          v
                       C35 RV32

C32V 仅由真实 ARM branch relocation 证据触发，可与 C33+ 独立排期。
```

C33–C35 可以并行准备 artifact 和 golden corpus，但生产 backend 的合入顺序保持串行，
每次都要运行之前 profile 的完整回归。禁止复制 ARM session pipeline 后再“以后统一”。

## 21. 跨仓提交拆分

继续保留总计划的 C30–C35 编号，在跨 repo 实际落地时拆成可独立验证的小提交：

| 编号 | repo/子树 | 建议提交主题 |
| --- | --- | --- |
| C30-a | `build` + fixture | `build: add manifest-closed dynamic application packages` |
| C30-b | `kernel/loader` | `loader: expose requester context for dependency resolution` |
| C30-c | `kernel/application` | `kernel: resolve package-private dependency closures` |
| C30-d | `kernel/tests` | `tests: run ARM32 private DSO graphs end to end` |
| C31-a | `kernel/loader` | `loader: freeze multi-DSO scope and lifecycle semantics` |
| C31-b | `kernel/application` | `kernel: publish initialized system DSO batches` |
| C31-c | `librs` + fixtures | `librs: validate multi-DSO emutls and lifecycle` |
| C31-d | `kernel/tests` | `tests: reap private groups and quiesce system SCCs` |
| C32 | loader/build/kernel/librs | `loader: enable Thumb v8-M hard-float dynamic applications` |
| C32V | 可选 | `loader: add planned Thumb branch veneers` |
| C33 | loader/build/kernel/librs | `loader: enable RV64 dynamic applications` |
| C34 | loader/build/kernel/librs | `loader: enable AArch64 dynamic applications` |
| C35 | loader/build/kernel/librs | `loader: enable RV32 IMAC and IMC dynamic applications` |

producer → consumer 顺序应为：build profile/fixture 先能生成并被 gate 拒绝或接受；loader
backend 其次；kernel adapter/start/runtime 再次；最后接 board filesystem 和 QEMU checker。
中间提交可以不运行新 profile，但不能破坏已有 target。

## 22. 推荐验证入口

最终至少形成：

```text
//build:check_dynamic_target_profiles
//build:check_dynamic_app_packages
//kernel/loader:check_dynamic_linker
//kernel/kernel:check_application
//librs:check_librs
//kernel/tests/dynamic_test:run_dynamic_test
//kernel/tests/dynamic_multidso_test:run_dynamic_multidso_test
//kernel/tests/dynamic_arch_test:run_<profile>_dynamic_test
```

每个 board 的常用命令保持 GN/Ninja：

```bash
gn gen out/<board>.release.dsc --args='build_type="release" board="<board>"'
ninja -C out/<board>.release.dsc check_all
./out/<board>.release.dsc/bin/<dynamic-runner>.sh
```

host fake、readelf 和单元测试都是前置门禁，不能替代最终目标工具链和 QEMU/板上执行。

## 23. 主要风险与控制

| 风险 | 后果 | 控制 |
| --- | --- | --- |
| resolver 通过目录搜索私有库 | 同名劫持、不可复现闭包 | build-generated exact manifest；运行期逐 edge 对比 |
| 逐 SONAME 拿 system permit | 并发 system cycle ABBA | manifest closure + atomic batch acquire |
| system DSO 在 ctor 前 Ready | 第二个应用进入半初始化代码/数据 | `Initializing` 状态；InitComplete 后 batch publish |
| 首发 group 隐式拥有 system backing | group 退出后 descriptor 悬空或永久泄漏 | registry-owned `SystemDsoInstance`；首发组也只持 lease |
| app fini 包含 system fini | 其他应用仍在使用时析构 libc | lifecycle 按 ownership 分区；system reaper 独占 fini |
| cached system DSO 不保活依赖 | provider 先被卸载 | instance 持 system dependency leases；SCC quiescence |
| 把现有 graph 单测当多 DSO 完成 | VFS/ABI/ctor/reap 问题未覆盖 | ARM32 真实 ELF/QEMU 是 C30/C31 gate |
| hard/soft float 混装 | 调用约定破坏且症状随机 | e_flags + attributes + ABI note + closure一致性矩阵 |
| `fence.i`/I-cache 只同步当前 core | 迁移后执行旧指令 | execution scope capability + rendezvous/affinity |
| AArch64 heap 页混装权限 | RELRO/W^X 无法真实执行 | 页独占 image backend + page permission gate |
| RV32 IMC 误用 A 扩展 | 无 A 设备非法指令 | profile note/ELF attributes/toolchain gate；无 A 回归 |
| 架构 backend 复制 session | 平台语义逐渐分叉 | 只扩 `ArchRelocator/CodeCache/ImageMemory` adapter |
| 仅凭 refcount 卸载 system DSO | callback/TLS/function pointer UAF | use token + dependency edge + group/thread quiescence；否则 KeepCached |
| build-id 被当成签名 | 错误安全承诺 | 文档/代码明确 built-in trust；签名留到 Phase 3 |

## 24. Phase 2 完成清单

- [ ] ARM32 v7-M 的 app→foo/bar→common/libc 真实 ELF/QEMU 闭环通过；
- [ ] 应用包 manifest 由最终 ELF 生成并双向验证；
- [ ] private/system SONAME namespace 冲突稳定拒绝；
- [ ] resolver 不搜索 cwd、环境变量、RPATH/RUNPATH 或未声明目录；
- [ ] requester id/ownership 进入解析和错误上下文；
- [ ] identity/SONAME 双去重覆盖菱形、别名和冲突；
- [ ] private 循环与 system SCC 不死锁，稳定输出 SCC snapshot；
- [ ] system closure 通过原子 batch acquire，无部分 permit/lease；
- [ ] application/system scope 和 strong/weak/hidden/protected oracle 通过；
- [ ] system requester 无法绑定 app/private symbol；
- [ ] lifecycle plan 分为 startup、group fini 和 system fini；
- [ ] system DSO 在 constructor 完成前保持 `Initializing`；
- [ ] `ApplicationInitComplete` 原子发布 system batch 并转 application Running；
- [ ] ctor fault 不留下 Ready descriptor，waiter 能重试新 generation；
- [ ] system instance 自己拥有 allocation/descriptor/fini/dependency leases；
- [ ] 首发组与 import 组使用相同的 counted lease 语义；
- [ ] app exit 只 fini/reap root 和 private DSO；
- [ ] system fini 只在 quiescence 后由 registry worker 运行；
- [ ] 至少一个可证明静默的 system fixture 完成 unload/reload；不能证明者 KeepCached；
- [ ] 多 DSO emutls 每线程隔离、destructor 和 image 保活通过；
- [ ] ELF gate 已数据驱动并检查 target-specific ABI note/relocation；
- [ ] ApplicationService/Loader 不再硬编码 ARM soft-float；
- [ ] ARM v8-M hard-float profile 和 soft/hard reject matrix 通过；
- [ ] cache-enabled Cortex-M 实现真实维护或明确 capability fail closed；
- [ ] RV64 relocation、LP64/ISA、start、fence.i 和 QEMU gate 通过；
- [ ] AArch64 relocation、页权限、cache、start 和 QEMU gate 通过；
- [ ] RV32 relocation、ILP32、IMAC/IMC、无 A 同步和 QEMU/板 gate 通过；
- [ ] 每个允许 relocation 都有真实 artifact golden；
- [ ] 各架构 graph/symbol/lifecycle/reap 规范化 snapshot 一致；
- [ ] 全故障注入矩阵证明无半发布、无 dangling permit/lease、无 double release；
- [ ] Phase 0/0.5/1、静态应用和各架构静态 kernel 回归不退化；
- [ ] 未把 built-in allowlist、页权限或特权 thread group 描述成签名/进程隔离。

## 25. 建议的第一个实现切片

不要先写四个架构 backend。第一个切片应只聚焦 ARM32 v7-M 的多 DSO 和正确生命周期：

1. 增加 `libfoo.so.1`，只依赖 `libc.so.1`；
2. 为 `/apps/multi/app.elf + lib/libfoo.so.1` 生成最小只读 manifest；
3. 扩展 `DependencyRequest`，实现 package/system composite resolver；
4. 让 foo 以 `SessionPrivate` 进入现有 graph/scope/relocation；
5. 将 startup plan 与 group/system fini 分区；
6. 把 libc allocation ownership 从首发 group 移入 registry instance；
7. 增加 `Initializing → Ready`，由 `ApplicationInitComplete` 完成发布；
8. 在 `qemu_mps2_an385` 证明：foo ctor 一次/组、libc ctor 一次/generation、foo fini 随组
   执行、libc fini 不随首个组执行、private allocation exactly once 回收；
9. 再扩成 foo/bar/common 菱形和 cycle fixture；
10. ARM32 多 DSO 稳定后才启动 C32–C35。

这个切片优先消除 Phase 2 最大的返工风险：artifact 解析边界和 system/private 生命周期
所有权。若先扩架构，后续修正 `Ready` 时机、fini 分区和 backing owner 将要求同时修改五套
fixture，定位失败也会混入 ABI/cache 差异。
