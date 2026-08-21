# BlueOS 动态加载实施方案

本文概括 BlueOS 动态加载的目标、总体架构、阶段规划、模块设计和关键风险。具体结构体、接口、方法及测试项参见 [BlueOS 动态加载详细实现计划](./dynamic-loading-implementation-plan.md)，技术背景和方案论证参见 [BlueOS 动态加载调研报告](./dynamic-loading-survey.md)。

所有模块、文件和公共类型名称以调研报告 [2.12 BlueOS 模块命名规范与参考词典](./dynamic-loading-survey.md#212-blueos-模块命名规范与参考词典) 为准。

## 1. 方案结论

BlueOS 当前采用 RTOS 线程模型和共享地址空间，已经有一个只能装载单 ELF、仅支持少量 RISC-V 相对重定位的 loader。推荐方案是在现有 `blueos_loader` 内逐步扩充动态链接能力，而不是新建另一套 loader，也不在当前阶段引入用户态 `ld.so`。

评审问题的直接结论如下：

| 问题 | 结论 |
| --- | --- |
| 当前 app 是否就是线程 | 主入口由线程执行，但动态应用实例还拥有映像、子线程、TLS/atexit、system DSO lease、配额和退出状态；`Thread` 是调度单位，`ThreadGroup` 是资源归属与回收单位 |
| 为什么需要应用状态 | 只为准入/装载、可运行、停止回收和最终结果提供粗粒度状态；不重复 ready/running/suspended 等 thread state，也不复制 loader 的 relocation typestate |
| 顶层模块叫什么 | `application/ApplicationManager`；不用容易与 libc/language runtime 混淆的 `AppRuntime` |
| 是否继续使用 `Rtos*` 前缀 | 不使用。当前执行实现放入 `thread_group/`，只对确实表示线程组的类型使用 `ThreadGroup`；通用 loader、registry、memory 和 start-info 使用职责名称 |
| 现在能否用 MPU 保证安全 | 不能。Cortex-M 现有 MPU 只做 stack guard，线程仍为 privileged；RISC-V 没有 S/U mode 与应用 PMP domain；AArch64 EL0 尚不支持。当前没有应用运行期 containment；签名包交付前只能运行内置/受控来源代码 |

最终形成以下链路：

```text
ApplicationManager → thread_group::ApplicationLoader
        ↓
blueos_loader::DynamicLinker
        ↓
ImageLoader
        ↓
ElfReader / ImageMemory / ArchRelocator
```

核心决策如下：

- 主程序和 DSO 使用标准 `ET_DYN + PT_DYNAMIC + DT_NEEDED`。
- 主程序不带 `PT_INTERP`，由内核中的 `ApplicationLoader` 在启动前完成动态链接。
- 保留现有 `blueos_loader` crate，在内部拆分 `ImageLoader + DynamicLinker`。
- 首版优先支持 ARM32 Thumb v7-M soft-float EABI，固定 target 为 `thumbv7m-vivo-blueos-newlibeabi`、验证平台为 `qemu_mps2_an385`；之后扩展 Thumb v8-M hard-float、RISC-V64、AArch64 和 RISC-V32。
- 系统按需装载并共享一份 `libc.so.1`，应用通过 lease 保活。
- 首版采用立即绑定、emutls 和整 ThreadGroup 回收。
- 内核、动态应用和 DSO 共用默认 PIC、支持 dynamic 的 target/toolchain 与 sysroot；内核使用 PIC codegen，但最终以 `-static` 链接为自包含启动映像。

命名按职责与所有权划分：通用链接能力使用 `ImageLoader`、`DynamicLinker`、`LinkSession`、`ImageMemory` 等中性名称；内核控制面使用 `ApplicationManager`，统一 `ApplicationHandle/ApplicationLaunchRequest/ApplicationState`、准入、查询、退出通知和回收；当前实现放入 `thread_group/`，实例为 `ThreadGroup`，线程归属为 `ThreadGroupMembership`。`ApplicationLoader`、`SystemDsoRegistry`、`FlatImageMemory` 和 `ApplicationStartInfo` 已能表达职责，不增加平台前缀。未来在同一 `application` 中增加同级 `process/` 后端、`Process`、`AddressSpace`、`ProcessImageMemory` 和 exec stage0。

Phase 0–2 暂不实现：

- 应用侧 `dlopen/dlclose`
- `dlsym/dlerror` 等运行期链接接口
- lazy binding
- 原生 ELF TLS
- IFUNC、COPY、TEXTREL
- symbol version
- `PT_INTERP` 和用户态 `ld.so`
- 进程、用户态和完整地址空间隔离

Phase 3 增加受限的运行期装载：先支持 `dlopen/dlsym/dlerror/dlclose`、`RTLD_NOW | RTLD_LOCAL` 和可选的 `RTLD_NODELETE`。其中 `dlclose` 首先只减少当前 ThreadGroup 的显式打开计数并使最后一个 handle 失效，不立即执行 fini 或释放映像；映像最迟随整个 ThreadGroup 退出回收。这样可以提供插件 API，又不会因为 `dlsym` 返回的裸函数/数据指针、正在执行的代码、回调或 TLS destructor 造成 use-after-free。`RTLD_GLOBAL/RTLD_NOLOAD`、`dladdr/dl_iterate_phdr` 可在 Phase 3 的兼容扩展中增加；lazy binding、`RTLD_DEEPBIND/RTLD_NEXT` 和任意 DSO 的安全立即卸载仍不在当前交付范围。

## 2. 总体架构

![BlueOS RTOS 动态加载分层与稳定边界](./assets/dynamic-loading/blueos-layered-architecture.svg)

各层职责如下：

| 层次 | 所在位置 | 主要职责 |
| --- | --- | --- |
| `ApplicationManager` | kernel | 唯一应用控制面；统一句柄、准入、粗粒度状态、后端分派、查询、退出事件和回收，不参与线程调度 |
| `ApplicationLoader` | kernel | 为链接核心提供 VFS、内存、cache、系统 DSO registry 和 resolver |
| `DynamicLinker` | `blueos_loader`，静态编入 kernel | 处理依赖图、符号作用域、符号解析、重定位、init/fini 顺序和事务回滚 |
| `ImageLoader` | `blueos_loader` | 处理单 ELF 校验、布局、映射、复制、BSS 清零和基础重定位 |
| `ElfReader/ImageMemory/ArchRelocator` | loader 接口，由 kernel adapter 或架构实现提供 | 隔离具体 VFS、allocator、地址空间、权限和指令缓存实现 |
| `libc.so.1` / blueos_scrt1 | 动态应用运行时 | 初始化 libc、pthread/emutls、执行构造函数并进入 `main` |

这是一种“逻辑分层、当前统一运行在特权态”的结构。`DynamicLinker` 虽然不直接依赖内核对象，但在当前 RTOS 方案中仍作为 rlib 编入内核执行。

## 3. 主要模块设计

### 3.1 `blueos_loader` 内部模块

```text
kernel/loader/src/
  address.rs
  identity.rs
  reader.rs
  memory.rs
  error.rs
  image/
    parser.rs
    metadata.rs
    layout.rs
    policy.rs
    loaded.rs
  relocation/
    model.rs
    arch/                 arm / riscv64 / aarch64 / riscv32
  dynamic_linker/
    linker.rs
    session.rs
    context.rs
    dependency.rs
    scope.rs
    symbol.rs
    lifecycle.rs
```

模块与主要对象一一对应：

| 模块 | 主要对象或接口 | 职责 |
| --- | --- | --- |
| `address.rs` | `TargetAddr`、`TargetWord`、`TargetRange` | 区分目标地址与 loader 本地地址，集中 checked arithmetic |
| `identity.rs` | `ImageId`、`AllocationId`、`LinkDomainId` | 区分映像、内存分配和链接命名空间标识；不与 protection domain 混用 |
| `reader.rs` | `ElfReader` | 提供与 VFS、内存切片或用户态文件无关的按偏移读取接口 |
| `memory.rs` | `ImageMemory`、`CodeCache` | 抽象目标内存分配、复制、清零、权限、回收和指令缓存同步 |
| `error.rs` | `LoadError`、`LoadStage` | 记录失败阶段及 ELF、符号、重定位和底层错误上下文 |
| `image/` | `ImageLoader`、`ParsedImage`、`ImageLayout`、`LoadedImage` | 完成单 ELF 解析、校验、布局、映射和基础重定位 |
| `relocation/` | `ArchRelocator`、`RelocationKind` | 提供各架构唯一的 REL/RELA/RELR 解码、计算和写入实现 |
| `dynamic_linker/` | `DynamicLinker`、`LinkSession`、`LinkContext`、`DependencyGraph`、`SymbolScope`、`LinkProduct`、`InitPlan`、`FiniPlan`、`LoadMetrics` | 组合多个 image，完成依赖闭包、符号绑定、事务回滚、发布、生命周期计划和通用指标采集 |

### 3.2 内核单一 `application`

```text
kernel/kernel/src/application/
  model.rs                ApplicationHandle / ApplicationLaunchRequest / ApplicationState / ExecutionModel
  manager.rs              唯一的 ApplicationManager 与 registry/后端编排
  registry.rs             ApplicationRegistry / ApplicationInstance
  lifecycle.rs            ApplicationEventQueue / 查询与延迟回收
  accounting.rs           ApplicationQuota / ApplicationUsage
  metrics.rs              公共资源记账和失败统计
  protection.rs           ProtectionDomain 接口与能力级别
  adapters/              跨执行模型的内核能力适配
    vfs_reader.rs         VFS → ElfReader
    system_library_paths.rs 系统可信路径策略
  thread_group/                    当前唯一实现的后端
    backend.rs            ThreadGroupBackend
    group.rs              ThreadGroup、成员线程与资源寿命不变量
    loader.rs             DynamicLinker 的内核适配层
    system_dso.rs         系统 DSO registry 和 lease
    start_info.rs         argv/envp/auxv/init/fini 启动信息
    membership.rs         Thread 与 ThreadGroup 的归属关系
    adapters/            仅共享地址空间模型特有
      flat_memory.rs      FlatImageMemory
      artifact_resolver.rs ApplicationArtifactResolver
  process/                 未来由 blueos_user_process 条件编译
    backend.rs
    process.rs
    exec.rs
    address_space.rs
    initial_stack.rs
```

指令缓存维护（`fence.i`、D-clean/I-invalidate）实现在 `arch/*/cache.rs` 作为架构通用服务，`ApplicationLoader` 直接调用，不设 `code_cache.rs` adapter 文件。

| 模块 | 主要对象或接口 | 职责 |
| --- | --- | --- |
| `model.rs` | `ApplicationHandle`、`ApplicationLaunchRequest`、`ApplicationState`、`ExecutionModel` | 定义跨执行模型的公共控制面值对象；公共 Rust 类型不缩写为 `App*` |
| `manager.rs` | `ApplicationManager` | 负责应用准入、后端分派，并协调 registry、启动、查询、退出事件和延迟回收 |
| `registry.rs` | `ApplicationRegistry`、`ApplicationInstance` | 保存已发布实例和 tagged 后端引用，不在表锁内执行 I/O、链接或回收 |
| `lifecycle.rs` | `ApplicationEventQueue`、`ApplicationInfo` | 统一粗粒度状态、执行单元退出通知和 reaper 协议 |
| `accounting.rs` | `ApplicationQuota`、`ApplicationUsage` | 定义跨后端配额与使用量，各后端补充模型特有记账 |
| `protection.rs` | `ProtectionDomain`、`ProtectionDomainId`、`ProtectionLevel` | 表达可用硬件保护能力；不把共享地址空间 `ThreadGroup` 宣称为安全域 |
| `thread_group/backend.rs` | `ThreadGroupBackend` | 把当前 RTOS prepare/start/terminate/reap 接到唯一 `ApplicationManager` |
| `thread_group/group.rs` | `ThreadGroup`、`ThreadGroupMembership` | 保存一次应用实例的成员线程、私有映像、link context、lease、protection-domain id 和回收不变量 |
| `thread_group/loader.rs` | `ApplicationLoader` | 将内核 VFS、内存、cache、resolver 和 registry 接到 `DynamicLinker` |
| `thread_group/system_dso.rs` | `DsoArtifactCatalog`、`DsoInstanceRegistry`、`SystemDsoRegistry`、`SystemDsoLoadPermit`、`SystemDsoLease` | 分离工件与运行实例，管理并发首装、共享实例、租约和静默卸载 |
| `thread_group/start_info.rs` | `ApplicationStartInfo`、`ApplicationStartStorage` | 稳定保存 argc、argv、envp、auxv、init/fini plan 及其跨 ABI 指针 |
| `thread_group/membership.rs` | `ThreadGroupMembership` | 记录不可由应用伪造的线程组归属，并支持创建继承与退出记账 |
| `adapters/` | `VfsElfReader`、`SystemLibraryPaths` | 只依赖 VFS 与系统路径策略的通用内核能力适配，与执行模型无关，thread_group 与未来 process 后端共用 |
| `thread_group/adapters/` | `FlatImageMemory`、`ApplicationArtifactResolver` | 依赖共享地址空间映射与 `SystemDsoRegistry` 实例状态的模型特有适配；未来替换为 `ProcessImageMemory` 与每进程 resolver；指令缓存维护由 `arch/*/cache.rs` 架构服务提供，不设独立 adapter |
| `process/` | `ProcessBackend`、`Process`、`AddressSpace`、`ProcessImageMemory` | 未来实现 exec stage0、initial stack 和 user-thread 生命周期；普通 DSO 链接仍交给用户态 `ld.so` |
| `metrics.rs` | `LinkProduct::metrics`、`LoadMetrics` | 聚合并导出各加载阶段时延、读取字节、重定位数量、峰值内存和失败原因 |

当前只编译 `thread_group/`，不需要提前实现空壳 `process/`。以后由一个权威内核配置生成 `blueos_user_process`，只在模块声明、后端字段和 `ApplicationInstance::Process` 分支处使用条件编译。若通用内核仍兼容 RTOS 应用，可同时编入两个后端，并根据已签名 manifest/ABI note 的 `ExecutionModel` 分派；不能根据 `PT_INTERP` 自动猜测。

### 3.3 `librs` 和应用启动

`librs_swi` 在保留现有 rlib 的同时新增 `libc.so.1` 产物。应用链接 SDK 中的 `libc.so`，运行时由 registry 从固定可信路径加载 `libc.so.1`。

```text
librs/src/
  start.rs
  application_context.rs
  auxv.rs
  pthread.rs
  tls.rs
```

| 模块 | 主要对象或入口 | 职责 |
| --- | --- | --- |
| `start.rs` | `__librs_start_main` | 校验启动信息、初始化运行时、执行 init plan、调用 `main` 并协调退出 |
| `application_context.rs` | `LibcApplicationContext` | 保存应用句柄、auxv、atexit 和当前 thread-group 应用级 libc 状态；类型带 `Libc` 以免与内核或 Android `Context` 混淆 |
| `auxv.rs` | `getauxval()` | 从当前线程关联的 `LibcApplicationContext` 查询 auxv |
| `pthread.rs` | `PthreadTcb` | 让子线程继承应用归属，并管理 pthread destructor 和退出顺序 |
| `tls.rs` | emutls 线程表 | 为当前首版提供无需原生 ELF TLS relocation 的线程局部存储 |

`blueos_scrt1.o` 属于 SDK 启动对象，不属于 `librs` 模块：它接收 `ApplicationStartInfo *`，找到应用 `main` 后调用 `__librs_start_main`。

启动流程：

```text
ApplicationManager 分派到 ThreadGroupBackend 创建线程
        ↓
blueos_scrt1::_start(ApplicationStartInfo *)
        ↓
__librs_start_main(main, info)
        ↓
初始化 LibcApplicationContext / pthread / emutls
        ↓
执行 init plan
        ↓
main(argc, argv, envp)
        ↓
等待组内线程退出、执行 fini、回收 ThreadGroup
```

### 3.4 为什么应用实例不能直接等同于线程

`Thread` 的生命周期由 scheduler 管理，状态是 ready/running/suspended/retired；动态应用的生命周期由 `ApplicationManager` 管理，公共状态只需 `Loading/Running/Stopping/Terminated/Failed`。两者分别解决“何时获得 CPU”和“哪些资源必须一起保活/回收”，不能共用同一状态枚举。

`ThreadGroup` 至少拥有：主线程与 pthread 集合、main/private DSO image、`LinkContext`、`SystemDsoLease`、`ApplicationStartStorage`、应用级 libc context、配额/使用量和退出码。主线程退出时若仍有子线程、TLS destructor、fini 或回调，映像不能释放。因此 manager 不是额外调度器，而是装载实例的 owner/reaper；NuttX 的 `task_group_s`、RT-Thread 的 `rt_dlmodule` 和 CHRE `EventLoop` 持有的 `Nanoapp` 都采用相同分层。

### 3.5 当前安全控制与后续隔离边界

| 级别 | 当前/首版机制 | 交付结论 |
| --- | --- | --- |
| L0：可信特权应用 | MVP 仅内置/构建期 allowlist，产品阶段签名；ELF 防御解析、配额、固定 resolver、生命周期回收 | Phase 0/1 交付开发信任边界，Phase 3 才允许产品分发；两者都不能限制已运行代码的任意指针 |
| L1：装载与映像 hardening | 受限依赖/符号域、`RelocationPolicy`、NOW、`SealPlan`、RELRO、cache sync、原子发布 | 当前 Phase 0–3 必须交付；保护 loader 和映像完整性，不构成运行期 containment |
| L2：非特权 protection domain | unprivileged/U/EL0 + 每应用 MPU/PMP/页表 + context-switch hook + fault attribution + syscall copy-in/out | 当前 Phase 0–3 之后的可选演进；完整闭环后才是应用隔离 |

当前阶段的具体控制如下：

| 控制点 | 现阶段策略 | 边界 |
| --- | --- | --- |
| 工件准入 | 内置/allowlist，产品阶段签名、hash、manifest、ABI note、artifact generation；分配前完成资源上限检查 | 证明来源和兼容性，不证明代码无恶意 |
| ELF/布局 | checked arithmetic；校验 file/vaddr/alignment/overlap/entry；拒绝 W+X、TEXTREL、可执行栈和未支持特性 | 防止畸形 ELF 借 loader 越界读写 |
| 依赖/符号 | 固定 resolver，SONAME+文件身份去重，application/system scope 分离，不导出完整内核符号表 | 防止私有 DSO 冒充 system DSO 和应用 interpose libc relocation |
| relocation | `P` 必须属于当前 owner 的允许写入区；`S` 必须来自冻结 scope；类型、宽度、对齐、addend 和溢出全部校验 | `JUMP_SLOT`、entry、init/fini 目标必须在允许 X region，undefined weak=0 只能由 profile 显式放行；只覆盖 relocation-backed 控制流，不是完整 CFI |
| 权限/cache | S9 前映像不可见；写完后形成 RX/R/RW+NX，RELRO 去 W，cache 同步后才 publish | 无硬件后端时只能记录 `LogicalOnly`，不能宣传为硬件防护 |
| 发布/回收 | typestate、rollback log、原子 publish、ThreadGroup、lease、quiescence | 防止半成品执行、资源泄漏和仍在执行时卸载 |

这些控制不会检查同一 ELF 中已经静态解析的直接分支，也不能约束运行期函数指针、返回地址或应用自行计算的跳转/访存地址。当前 Cortex-M 没有进入 `nPRIV`，RISC-V 没有应用 U-mode/PMP domain，AArch64 没有完整 EL0 路径，因此运行中的 native 应用仍是可信特权代码。

MPU/PMP/MMU 作为后续可选演进另行立项。只有非特权执行、per-domain 内存配置、scheduler 切换、syscall copy-in/out、fault attribution、资源配额和共享库可写状态隔离同时完成，才允许标记 `IsolatedDomain`。现有 `ProtectionDomainId`、`ThreadGroupMembership`、`ImageMemory` 和 `ProtectionDomain` 接口只负责为这条演进保留接入点，不进入当前 MVP/产品化阶段的完成门槛。

竞品与源码依据详见调研报告 [§2.11 应用管理命名与生命周期竞品分析](./dynamic-loading-survey.md#211-应用管理命名与生命周期竞品分析) 和 [§4.3 BlueOS 执行与保护现状](./dynamic-loading-survey.md#43-执行与内存模型是最大的结构性约束)。

## 4. 分阶段实施规划

![BlueOS RTOS 动态加载分阶段实施路线](./assets/dynamic-loading/implementation-roadmap.svg)

| 阶段 | 阶段目标 | 主要实现内容 | 阶段产物 |
| --- | --- | --- | --- |
| Phase 0 | 把当前 loader 变成可信的单映像 loader | 修正 load bias、BSS、对齐和整数溢出；增加 ELF 校验、权限、cache 接口、`ElfReader` 和结构化错误；未知 relocation 明确失败 | 安全、可复用的 `ImageLoader`，现有 static PIE/ET_EXEC 继续运行 |
| Phase 0.5 | 冻结 `DynamicLinker` 架构 | 增加动态表解析、依赖图、符号表、GNU/SysV hash、LinkSession、事务回滚、`RelocationPolicy` 和 ARM32 REL/Thumb relocation 框架 | 可在主机侧测试的多 DSO 链接核心 |
| Phase 1 | 打通 ARM32 `app → libc.so.1` MVP | 以 `thumbv7m-vivo-blueos-newlibeabi`/`qemu_mps2_an385` 为首发 profile；实现单一 ApplicationManager 及 `thread_group/` 后端，生成 `libc.so.1` 和 blueos_scrt1，并接入 registry、lease、VFS/内存/cache、argv/auxv、emutls 和整组退出 | shell 可运行一个真正动态依赖 libc 的 Thumb C PIE |
| Phase 2 | 支持多 DSO 和其余目标 profile | 支持应用私有 DSO、菱形/循环依赖、weak/visibility、ctor/fini；扩展 Thumb v8-M hard-float、RISC-V64、AArch64 和 RISC-V32 relocation/cache backend | 同一 app/libfoo/libc 用例跨目标一致运行 |
| Phase 3 | 运行期装载与产品化加固 | 受限 `dlopen/dlsym/dlerror/dlclose`、运行期增量事务与 local scope、逻辑关闭/整组延迟回收；应用包签名、ABI note、allowlist、并发合并、故障注入、装载策略审计、link map、崩溃符号化和 OTA 兼容 | 面向可信应用的插件装载 API，以及可发布、可诊断、可升级的动态应用机制；MPU/PMP 等 L2 隔离能力另行演进 |

## 5. 各阶段重点功能

### Phase 0：修正现有 loader

当前 loader 的 BSS、load bias、relocation 和范围检查都存在明显缺口。Phase 0 先解决基础正确性，避免动态链接建立在不可靠的映射机制上。

重点功能：

- 使用 checked arithmetic 处理文件范围和目标地址。
- 正确计算 `load_bias = mapped_base - aligned_min_vaddr`。
- 显式清零 `p_memsz - p_filesz`。
- 校验 `p_filesz <= p_memsz`、`p_align`、segment overlap 和入口权限。
- 拒绝 W+X、越界和未知 relocation。
- 把整文件入口抽象为 `ElfReader::read_at()`。
- 保留 `load_elf()` 兼容接口，内部转调 `ImageLoader`。

阶段验收：现有 static PIE 和 ET_EXEC 不回退，所有畸形 ELF 都有确定错误且无内存泄漏。

### Phase 0.5：建立动态链接核心

重点功能：

- 解析 `PT_DYNAMIC`、`DT_NEEDED`、`DT_SONAME`、dynsym/dynstr。
- 建立依赖图和稳定的 BFS symbol scope。
- 支持 global/weak 和必要 visibility 语义。
- 以 ARM32 `DT_REL` 为首发路径，实现有界读取隐式 addend，并冻结最小白名单：`R_ARM_RELATIVE/R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT`；`RelocationPolicy` 同时校验写入 owner/范围、符号来源，以及 `JUMP_SLOT` 等控制流目标的 X region，`ArchRelocator` 负责 Thumb bit 归一化与校验。
- 生成 init/fini plan，不在 loader lock 内执行应用代码。
- `LinkSession` 在失败时统一释放临时 image 和 lease。
- Relink 只作为宿主侧对照测试，不进入固件。

阶段验收：人工 ELF fixture 的依赖图、符号绑定、写值和构造顺序与 oracle 一致。

### Phase 1：ARM32 Thumb v7-M 纵向 MVP

重点功能：

- 固定 `thumbv7m-vivo-blueos-newlibeabi` soft-float EABI，建立内核、应用和 DSO 共用的 PIC/dynamic-capable 工具链与 sysroot，并用 `kernel_static`、`dynamic_app`、`dso` 三种策略控制最终链接。
- `librs_swi` 生成带 SONAME 和导出清单的 `libc.so.1`。
- 主程序生成 `ELFCLASS32 + EM_ARM`、无 `PT_INTERP`、带 `DT_NEEDED: libc.so.1` 的 Thumb PIE；构建门禁检查 `e_flags/.ARM.attributes`、`DT_REL` 和 relocation 白名单，运行期由 ELF header、已签名 manifest 与 `PT_NOTE` 携带的 BlueOS ABI note 校验 soft-float/Thumb profile，不依赖 section header。
- 首个应用触发 registry 装载 libc，后续应用复用 Ready 实例。
- `ApplicationManager` 校验 `ExecutionModel::ThreadGroup`，分派到 `ThreadGroupBackend` 创建 ThreadGroup 和主线程。
- blueos_scrt1/libc 接收 `ApplicationStartInfo`，支持 argv/envp/auxv。
- pthread 子线程继承 ThreadGroup，emutls 在线程退出时清理。
- 应用和私有资源按整个 ThreadGroup 回收。
- `qemu_mps2_an385` 无 cache 的 backend 仍执行 DSB/ISB 并明确报告 cache 能力；后续 cache-enabled Cortex-M 必须增加 D-cache clean/I-cache invalidate，不能沿用空实现。

阶段验收：C hello、文件 IO、heap、argv、auxv、weak symbol、ctor、BSS 和多线程 emutls 全部通过；两个并发应用共享同一 libc 实例。

### Phase 2：多 DSO 与多架构

重点功能：

- 支持 `app → libfoo → libc` 及菱形依赖。
- 应用私有 DSO 只在所属 ThreadGroup 可见。
- 系统 DSO 不允许被应用同名符号 interpose。
- ctor 按依赖优先，fini 逆序执行。
- 分别实现各架构的 relocation、地址宽度、Thumb 和 cache 规则。

目标 profile 扩展顺序：

```text
ARM32 Thumb v7-M soft-float（Phase 1）
    → ARM32 Thumb v8-M hard-float
    → RISC-V64
    → AArch64
    → RISC-V32
```

ARM32 另保留一个**不作为 Phase 1/2 基线门槛**的可选 `LoaderVeneerV1` profile。正常 `ET_DYN` 外部调用仍固定走本地 PLT 和 GOT 中的完整目标地址，Flash 与 PSRAM 相距超过 16 MiB 本身不触发 loader veneer。只有产物门禁证明仍存在 `R_ARM_THM_CALL/R_ARM_THM_JUMP24` 等需要 loader 修补、且目标可能超出分支位移范围的动态重定位时，才启用以下能力：S1 按调用点预规划 branch island，S2 在可达范围内预留 veneer pool，S7 生成/复用 veneer 并修补分支，S8 同步 cache 后封为 RX。不可写 Flash/XIP 中的调用点不能在运行期修补，必须改走链接期 veneer、PLT/GOT，或拒绝该工件。

阶段验收：同一依赖 fixture 在所有架构产生相同依赖图、符号绑定和构造顺序。

### Phase 3：运行期装载与产品化

重点功能：

- 在 `librs` 暴露 `dlopen/dlsym/dlerror/dlclose`；内核只接受属于当前 ThreadGroup 的带代次 handle，并对 path、flags 和 symbol name 做 copy-in/上限检查。名称只能解析到当前签名包 manifest 声明的私有 DSO 或系统只读 catalog，不能用环境变量、工作目录或任意绝对路径绕过准入。
- 每次 `dlopen` 使用独立 `RuntimeLoadSession` 增量发现依赖闭包、建立 group scope、完成 NOW relocation、seal 和原子发布；同一工件的并发首次打开合并成一次事务，constructor 在 loader/registry 锁外执行，并允许嵌套 `dlopen`。
- 基线只支持 `RTLD_NOW | RTLD_LOCAL`（接受 `RTLD_NODELETE`）；`dlsym(handle, name)` 在该加载组及其依赖闭包查找，`dlerror` 保存在调用线程的 libc 状态中。
- `dlclose` 只做引用计数和 handle 失效，不在运行期立即 fini/unmap；最后由 ThreadGroup 统一 fini/reap。需要立即回收的 DSO 必须另行定义 `UnloadSafe` 插件协议和可证明的 quiescence gate。
- 安装期包签名、hash、目标 ABI 和依赖闭包检查。
- 加载期复验 ABI note、SONAME、build-id 和资源配额。
- 并发首次装载只执行一次，失败可安全重试。
- 无法证明系统 DSO 已静默时保留缓存，不执行危险卸载。
- 审计 `ArtifactPolicy/RelocationPolicy/SealPlan`，构建门禁拒绝 W+X、TEXTREL、可执行栈、未知 relocation 和越权符号来源。
- 发布每个 region 的请求权限与 `LogicalOnly/HardwareEnforced` 实际结果；当前阶段不把 MPU/PMP protection domain 作为交付门槛。
- 发布 link map、结构化错误和 build-id 崩溃符号化。
- 建立 ABI diff、A/B 升级和回滚门禁。

阶段验收：并发 `dlopen` 同一 DSO 只映射/初始化一次；local scope 不泄漏；嵌套 constructor 不死锁；失败事务不改变既有 link map；重复 `dlopen/dlclose`、陈旧 handle、OOM、构造失败、反复启动退出、内存碎片和 OTA 兼容测试全部通过。

## 6. 工具链调整原则

同一 ISA/ABI 下，内核、应用和 DSO 统一使用一套默认 PIC、支持 dynamic linking 的 BlueOS target/toolchain 和一份 PIC `core/alloc/compiler_builtins` sysroot。内核采用“PIC codegen + `-static` 最终链接”；`dynamic` 只表示工具链具备生成 PIE/DSO 的能力，不表示内核依赖动态链接器。

| 产物策略 | 用途 | 最终链接与 ELF 契约 |
| --- | --- | --- |
| `kernel_static` | 内核启动映像 | PIC 输入对象 + `-static` + kernel `link.x`；固定入口/内存布局，无 `PT_INTERP`、`DT_NEEDED` 和待处理 dynamic relocation |
| `dynamic_app` | 动态主程序 | 同一 PIC sysroot + PIE；无 `PT_INTERP`，保留 `PT_DYNAMIC/DT_NEEDED`，启用 NOW、RELRO 和 build-id |
| `dso` | libc 和普通共享库 | 同一 PIC sysroot + shared；包含 SONAME、导出清单、NOW 和 RELRO |

配置收敛规则：

- GN toolchain 统一选择 clang + LLD，并固化默认 PIC/dynamic-capable target。
- board config 只决定 ISA、MABI 和 float ABI。
- artifact link policy 决定 static/PIE/shared 及安全链接参数。
- target BUILD 文件不再重复指定 linker、relocation model 和 `-static/-pie/-shared`。
- 动态应用不复用 kernel 的 `link.x`。
- Phase 1 首先将 `thumbv7m-vivo-blueos-newlibeabi` 从现有 `arm-none-eabi-gcc` 链接路径收敛到统一 clang + LLD，并冻结 ELF32/ARM EABI/soft-float/Thumb-only 产物契约；不能同时改变为 hard-float。
- 现有 GNU 风格 kernel `link.x` 逐板迁移到 LLD-compatible；未通过的板只能保留受控过渡 fallback，并标记为工具链未统一。
- 对内核 PIC 化建立尺寸和性能 A/B 门禁，重点检查 GOT、Flash/RAM、启动、调度、中断和 syscall；超预算必须显式评审例外。

## 7. 首个工程切片

建议第一个可演示切片严格控制为：

```text
平台：qemu_mps2_an385（ARMv7-M/Thumb）
target：thumbv7m-vivo-blueos-newlibeabi（32-bit little-endian、soft-float EABI）
应用：Thumb C PIE
依赖：仅 libc.so.1
绑定：RTLD_NOW 等价的启动期立即绑定
TLS：emutls
回收：整个 ThreadGroup
入口：blueos_scrt1::_start(ApplicationStartInfo *)
```

完整流程：

1. shell 执行 `run /apps/hello/app.elf arg1`。
2. `ApplicationManager` 验证构建期 allowlist、manifest 中的 `ExecutionModel::ThreadGroup` 和 ABI note，并由 `ThreadGroupBackend` 创建处于 Loading 的 ThreadGroup；产品签名在 Phase 3 替换开发信任证据。
3. ApplicationLoader 调用 DynamicLinker 装载 main。
4. 解析 `DT_NEEDED`，registry 按需装载或复用 `libc.so.1`。
5. 完成符号解析、ARM32 `DT_REL` relocation、Thumb 函数地址校验和 RELRO；执行与该板 cache 能力匹配的 DSB/ISB 同步。
6. 生成 ApplicationStartInfo 和 init/fini plan。
7. 创建以 blueos_scrt1 为入口的主线程。
8. libc 初始化运行时并调用 `main(argc, argv, envp)`。
9. 应用退出后等待所有线程并回收 ThreadGroup。
10. 最后一个 lease 释放后尝试静默卸载 libc；不能证明安全时保留缓存。

该切片一次验证工具链、ELF、VFS、内存、cache、符号、线程、libc、TLS 和退出生命周期，是整个方案最重要的纵向里程碑。

## 8. 主要风险及控制方式

| 风险 | 影响 | 控制方式 |
| --- | --- | --- |
| 当前应用仍在特权态运行 | 无法安全运行不可信第三方代码 | MVP 只允许内置/构建期 allowlist；产品分发必须签名；明确两者都不是用户态隔离 |
| 把 relocation 目标校验写成“控制所有跳转” | 形成错误安全承诺；直接分支、运行期函数指针和自行计算的地址不经过动态重定位 | 明确它只是 loader-time policy，不宣称 CFI；运行期隔离留给后续非特权 protection domain |
| 把“已有 MPU/MMU”误当成应用隔离 | 形成错误安全承诺，恶意代码仍可访问内核 | MPU/PMP 只作为后续演进；只有非特权级、per-domain 配置、fault/syscall 边界齐备才标记 `IsolatedDomain` |
| 现有 loader 基础正确性不足 | 动态链接后可能出现越界写或随机崩溃 | Phase 0 先修正映射、BSS、load bias 和校验 |
| 共享 libc 有系统级可写状态 | 应用状态可能互相影响 | 将线程/应用状态迁入 TCB 或 LibcApplicationContext，并在 Phase 3 审计 |
| `dlclose` 后仍存在裸函数/数据指针 | 可能跳入已释放代码；当前特权共享地址空间下还可能影响整个系统 | Phase 3 的 `dlclose` 只逻辑关闭并使 handle 失效，映像随 ThreadGroup 延迟回收；真正立即卸载仅允许另行认证的 `UnloadSafe` 协议 |
| ARM32 首发的 REL/Thumb/EABI 细节处理错误 | 隐式 addend、函数地址 bit 0 或 hard/soft-float 混用可能导致静默跳错 | Phase 0.5 冻结 ARM32 白名单；每种 relocation 使用 golden fixture，并对 ELF flags/ABI note、Thumb entry/function pointer 建立产物门禁 |
| 把 Flash/PSRAM 远距调用一律交给运行期 veneer | 增加 text relocation、近地址分配和 cache/W^X 复杂度；Flash/XIP 调用点还可能根本不可写 | 基线强制 PLT/GOT；仅当真实工件留下受支持的 Thumb 分支 relocation 时启用可选 profile，并在 S1/S2 预规划 branch island；不可写调用点 fail-closed |
| 后续架构 relocation 差异较大 | RISC-V/AArch64 可能出现隐蔽错误 | 每个 relocation 使用独立 golden fixture 和产物白名单 |
| 内核统一 PIC 后尺寸或性能回退 | GOT、间接寻址和寄存器压力可能影响资源受限平台 | 各板保留 non-PIC/PIC A/B 数据，门禁 Flash/RAM、启动、调度、中断和 syscall；例外必须评审 |
| linker 配置分散 | 产物可能意外变回 static PIE 或产生不支持 relocation | 使用统一 artifact link policy 和 `llvm-readelf` 构建门禁 |
| 构造函数不可回滚 | 启动失败可能已产生外部副作用 | ctor 前完成所有可失败链接步骤；ctor 失败销毁整个 ThreadGroup |

## 9. 预期交付结果

完成 Phase 1 后，BlueOS 将具备：

- 标准 ET_DYN 动态应用格式。
- 系统按需共享的 `libc.so.1`。
- 可复用的 `ImageLoader/DynamicLinker`。
- ThreadGroup、线程归属和整组回收。
- argv/envp/auxv、ctor/fini 和 emutls。
- ARM32 Thumb v7-M 上可运行的第一条完整动态应用链路。

完成 Phase 2 后，将具备多 DSO，并覆盖 ARM32 soft/hard-float、AArch64 和 RISC-V 目标 profile。

完成 Phase 3 后，将具备受限但安全闭环的 `dlopen/dlsym/dlerror/dlclose`、运行期增量链接事务，以及面向可信应用发布所需的签名、ABI、并发、OTA、调试、装载安全策略审计和生命周期 hardening。该阶段的 `dlclose` 不承诺立即释放内存。MPU/PMP/MMU protection domain 属于后续可选演进；只有另行完成 L2 全部门禁的平台，才可以增加“非特权应用隔离”的产品声明。

## 10. 实施原则

- 先修正单映像 loader，再实现多 DSO。
- 先在 `qemu_mps2_an385` 跑通 ARM32 Thumb v7-M soft-float 纵向链路，再扩展 hard-float ARM32 和其他架构。
- 保留一个 ELF parser 和一个 relocation 实现源。
- loader 核心不依赖 `ApplicationManager`、VFS、allocator 或 librs。
- 不持有 loader/registry 锁执行 ctor、fini 或应用代码。
- 所有阶段均以可自动验证的 ELF、QEMU 和故障注入 gate 收口。
- Phase 0–2 不引入 `dlopen`；Phase 3 只按受限 NOW/local scope/逻辑关闭模型增加运行期装载，不顺带引入原生 TLS、lazy binding 或用户态 `ld.so`。
