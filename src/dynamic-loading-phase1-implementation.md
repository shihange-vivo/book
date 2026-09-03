# BlueOS Dynamic Loading Phase 1 详细实施计划

本文是 Phase 1 的直接开发清单。它以 2026-09-03 的以下代码为基线：

- `kernel/loader_and_linker@d034908`：已经公开 Phase 0.5 `DynamicLinker` 入口；
- `build/enable_loader_test@173a130`：已经增加三类 artifact profile 和第一版 ELF gate；
- `librs/loader_and_linker@5639dfa`：仍只生成 `rlib`，动态启动入口仍是无参数版本；
- `book/dynamic_loading_and_linking@1e8d48a`：总方案已经定义 C18–C29，但尚未展开为逐接口实施文档。

本文承接 [Phase 0.5 详细实施计划](./dynamic-loading-phase05-implementation.md)，细化
[总实施计划](./dynamic-loading-implementation-plan.md)中的 C18–C29。目标是在
`qemu_mps2_an385` 上打通第一条真实的 ARM32 Thumb v7-M 动态应用纵向链路：

```text
shell / boot
  → ApplicationManager
  → VFS snapshot + ApplicationArtifactResolver
  → DynamicLinker
  → app.elf + libc.so.1
  → ARM32 NOW relocation + seal + publication
  → blueos_scrt1 + __librs_start_main
  → main(argc, argv, envp)
  → 整个 ThreadGroup 退出、fini 和延迟回收
```

Phase 1 的完成标志不是“能够生成 `.so`”，也不是“内核可以取得一个动态 ELF 的
entry”，而是 C29 的 QEMU 用例确实运行一个带 `DT_NEEDED: libc.so.1` 的 Thumb PIE，
并证明启动、共享、线程归属和退出回收形成闭环。

## 1. 当前基线判断

### 1.1 已经可以直接复用的能力

当前 `kernel/loader` 已经具备 Phase 1 所需的大部分纯链接核心：

- `ElfReader::len/read_exact_at` 和不依赖共享 file offset 的读取模型；
- `LoadProfile/HeaderFlagsPolicy/EntryMode`，可由内核传入 ARM32 soft-float/Thumb profile；
- allocation-explicit `ImageMemory/ImageProtectionMemory`；
- 多映像 `DynamicLinker::begin/link` 和 staged `Building/Scoped/Relocated/SealedSession`；
- 有界 `DT_NEEDED` BFS、identity/SONAME 去重、SCC、application/system scope；
- dynsym、GNU/SysV hash、global/weak/visibility 语义；
- ARM32 `R_ARM_RELATIVE/R_ARM_ABS32/R_ARM_GLOB_DAT/R_ARM_JUMP_SLOT` NOW relocation；
- session-wide rollback、cache/protection seal；
- `InitPlan/FiniPlan/LinkMapEntry/LinkProduct`；
- `LinkPublisher::prepare_batch/commit_batch` 两段式提交边界。

Phase 1 不应在 `kernel/src/application/` 中再实现一套 ELF parser、符号解析器或
relocation engine。内核只实现输入、内存、cache、registry、线程和发布适配。

### 1.2 C18 已有的部分实现

`build@173a130` 已增加：

- `build/config/artifact_profiles.gni`；
- `build/templates/kernel_static.gni`；
- `build/templates/dynamic_app.gni`；
- `build/templates/blueos_dso.gni`；
- `build/scripts/check_blueos_elf.py`；
- ARM32 `kernel_static` 和 `dso` smoke artifact。

这些文件已经建立了正确方向，但 C18 还不能直接标记为完整：

1. `qemu_mps2_an385` 的 kernel link 仍通过 `arm-none-eabi-gcc`，尚未完成 clang + LLD
   收敛和真实 kernel 启动 A/B；
2. `kernel_static_rustflags` 仍显式选择 `-Crelocation-model=static`，尚未证明
   kernel/app/DSO 共用同一份 PIC `core/alloc/compiler_builtins` sysroot；
3. `dynamic_app` 还没有 C19 的 `libc.so` import，因此没有真实 `DT_NEEDED` smoke；
4. 第一版 gate 尚未冻结 `.note.blueos.abi`、精确 SONAME/build-id、完整 NOW/RELRO 和
   输出 stamp 契约；
5. 构建注释中的“target 默认 PIC”必须由实际 rustc invocation、ELF 和 map 证明，不能
   只靠配置名称推断。

因此 C18 后续按 C18-b/C18-c 收口，不回退已经存在的模板。

### 1.3 当前 Phase 1 的重大接口阻塞项

目前 `ArtifactResolver::resolve()` 只能返回需要重新装载的 `ResolvedArtifact<R>`。
`ImageOwnership::SystemCandidate` 虽然会进入 system scope，但第二个应用再次请求
`libc.so.1` 时仍会重新分配、复制、重定位一份 libc；它还不能导入 registry 中已经
Ready 的系统 DSO。

另外，当前 `LinkProduct` 发布时只保留 allocation、sealed state 和 link-map 摘要；
session 中用于后续符号解析的 symbol table、load segment/runtime region、root program
header 视图会被丢弃。这不足以完成：

- 后续应用以 Ready libc 为符号 provider；
- 对导入 provider 的函数/data symbol 做范围校验；
- 构造 `AT_PHDR/AT_PHENT/AT_PHNUM`；
- 让 registry 持有可再次导入的 immutable published descriptor；
- 按 `SessionPrivate/SystemCandidate/ExternalReady` 正确拆分 lease。

C23-a 必须先扩展 loader 的 published/imported image contract。禁止通过让 resolver
重新打开同一个 `libc.so.1`、或让 `FlatImageMemory` 暗中复用上一次 allocation 来伪装
registry 共享。

### 1.4 尚未存在的内核与 librs 能力

当前代码中还没有：

- `kernel/src/application/` 和 `ApplicationManager`；
- `ThreadGroup`、membership、application event queue 和 reaper；
- `SystemDsoRegistry/SystemDsoLease/SystemDsoLoadPermit`；
- VFS `FileOps` 层稳定的 positional-read/snapshot contract；
- kernel 版 `FlatImageMemory` 和 ARM `CodeCache` adapter；
- versioned `ApplicationStartInfo` 和应用生命周期 syscall；
- `libc.so.1`、冻结导出表、`blueos_scrt1.o`；
- `LibcApplicationContext/getauxval/spawn`；
- 动态应用的 init/main/exit/fini runtime。

这些能力正是 C19–C29 的主体，不用预建 Phase 2/3 空壳。

## 2. Phase 1 范围

### 2.1 本阶段必须交付

- 首发 profile 固定为 `thumbv7m-vivo-blueos-newlibeabi`、ELF32、小端、`EM_ARM`、
  EABI5、Thumb-only、soft-float；
- 一个 `ET_DYN` 动态主程序，依赖且仅依赖系统 `libc.so.1`；
- `libc.so.1` 的 SONAME、build-id、BlueOS ABI note 和冻结导出清单；
- 构建期 allowlist 和固定 system library catalog；
- VFS snapshot reader、flat memory、MPS2 cache/barrier adapter；
- 首次 libc 装载的 permit，以及 Ready libc 的真实跨应用复用；
- 单一 `ApplicationManager` 和当前唯一的 `ThreadGroupBackend`；
- versioned start/exit ABI、argv/envp/auxv/init/fini storage；
- 主线程和 pthread 的 ThreadGroup membership；
- emutls 每线程隔离与 destructor；
- 正常退出的两阶段协议和 deferred reaper；
- 两应用并发请求 libc 时只映射、重定位和初始化一次；
- 最后 lease 后受控 quiescence：可以证明静默时卸载，不能证明时明确保持缓存；
- C29 QEMU 纵向用例和错误用例。

### 2.2 Phase 1 明确不交付

- `PT_INTERP`、用户态 `ld.so` 和 Linux initial stack；
- 原生 ELF `PT_TLS`、TLSDESC 或 TLS relocation；
- lazy binding；
- `IFUNC/IRELATIVE/COPY/TEXTREL`；
- symbol version、`RPATH/RUNPATH` 和环境变量库搜索；
- 应用私有 `libfoo.so`、菱形依赖和复杂 DSO 循环；
- `dlopen/dlsym/dlclose`；
- Thumb branch veneer 和动态 text patch；
- hard-float、AArch64、RISC-V 的符号型 relocation；
- 外部应用安装、产品签名、OTA ABI 升级；
- 非特权执行、MPU/PMP/MMU application isolation；
- 完整 CFI 或恶意应用运行期 containment。

Phase 1 只运行固件内置或构建期 allowlist 的可信 artifact。签名属于 Phase 3；
`ProtectionLevel::LogicalOnly` 也不能被描述为应用隔离。

### 2.3 首发 artifact policy

| 角色 | 必须满足 | 必须拒绝 |
| --- | --- | --- |
| kernel | fixed `ET_EXEC`、原有 vector/entry/link layout、无运行时 relocation | `PT_INTERP/PT_DYNAMIC/DT_NEEDED` |
| dynamic app | `ET_DYN`、Thumb entry、`PT_DYNAMIC`、NOW、RELRO、build-id、ABI note、`DT_NEEDED: libc.so.1` | `PT_INTERP/PT_TLS/TEXTREL/RPATH/RUNPATH`、其他 SONAME |
| libc | `ET_DYN`、`DT_SONAME: libc.so.1`、NOW、RELRO、build-id、ABI note、冻结 exports | 普通内核 undefined symbol、未知 relocation、未列出的 export |

`libc.so.1` 可以没有 `e_entry`，主程序必须以 `blueos_scrt1::_start` 为 entry。
Phase 1 的 application resolver 对除 `libc.so.1` 以外的 `DT_NEEDED` 返回确定的
`DependencyNotAllowed`，即使 Phase 0.5 图算法本身能够容纳更多依赖。

## 3. 完成定义与进入门禁

### 3.1 开始 C18–C22 的条件

C18–C22 与平台无关或只建立 producer contract，可以在 Phase 0.5 最终测试补齐前推进。
但它们不得绕过以下不变量：

- 生产 loader 只有一套 parser/relocation/seal 路径；
- trusted `LoadProfile/SessionLimits` 由 board/application policy 生成；
- `PHASE0_LOAD_POLICY` 不因 Phase 1 放宽；动态启动只走 `DynamicLinker`；
- `libc.so.1` 不向应用导出内核普通符号；内核能力只通过 SWI ABI 提供。

### 3.2 开始 C23/C24/C26 的硬条件

- Phase 0.5 `DynamicLinker` public facade 稳定；
- multi-allocation、rollback、NOW relocation、seal 和 publication 语义完成；
- `LinkProduct` 增加 C23-a 的 published/imported system image contract；
- VFS reader 能证明同一次 link 观察同一个内容 generation；
- ARM32 profile 的 `e_flags`、Thumb entry/function address gate 已启用；
- board 提供真实 `SessionLimits`，不直接使用 host 巨大默认值。

### 3.3 Phase 1 的唯一完成门禁

只有 C29 的真实 QEMU 纵向链路通过，Phase 1 才完成。host fake、readelf 输出和单独
`.so` smoke 都只是中间门禁。允许按当前开发安排先稳定生产结构、后统一补测试，但
C29 之前不能把测试清单删掉或把“尚未执行”写成“已经验证”。

## 4. 分层和源码布局

建议在现有目录上增量形成：

```text
kernel/loader/src/
  dynamic_linker/
    artifact.rs              # C23-a 增加 imported/published image contract
    publish.rs               # 保留可再次链接所需的 immutable metadata
    session.rs               # loaded 与 external-ready image 分流

kernel/kernel/src/application/
  mod.rs
  model.rs                   # handle、ExecutionModel、公共 ApplicationState
  manager.rs                 # 唯一 ApplicationManager
  registry.rs                # application handle table
  lifecycle.rs               # event queue、abort/exit/reap 编排
  accounting.rs              # quota/usage 和可 copy-in range
  adapters/
    vfs_reader.rs            # 通用 VfsElfReader
    system_paths.rs          # 固定系统库路径/catalog
  thread_group/
    mod.rs
    backend.rs               # ThreadGroupBackend
    group.rs                 # ThreadGroup 与内部状态
    loader.rs                # ApplicationLoader
    system_dso.rs            # catalog、registry、permit、lease
    start_info.rs            # ApplicationStartStorage
    membership.rs            # ThreadGroupMembership
    publication.rs           # LinkPublisher implementation/receipt
    adapters/
      flat_memory.rs         # shared-flat ImageMemory backend handle
      artifact_resolver.rs   # app/system resolver

kernel/kernel/src/arch/arm/
  cache.rs                   # ARM code publication barrier/cache service

kernel/header/src/
  lib.rs                     # 现有导出
  application.rs             # versioned wire ABI（若当前 build 支持子模块）

librs/src/
  start.rs
  application_context.rs
  auxv.rs
  spawn.rs
  pthread.rs
  tls.rs

librs/
  librs.exports
  start/arm/blueos_scrt1.S   # 或等价的最小 Thumb start object

apps/
  shell/
  dynamic/hello/
  dynamic/fixtures/
```

模块职责保持以下边界：

| 层 | 负责 | 不负责 |
| --- | --- | --- |
| `blueos_loader` | ELF、依赖图、scope、symbol、relocation、seal、link product | VFS path、线程、syscall、执行 ctor |
| `ApplicationLoader` | 把 VFS/memory/cache/registry 接到 loader | 重写 ELF 或 relocation 语义 |
| `ApplicationManager` | handle、执行模型分派、公共状态、启动/退出编排 | 在全局锁内做 IO/link/ctor |
| `ThreadGroup` | 当前共享地址空间模型的资源寿命和线程集合 | 冒充 Process 或安全地址空间 |
| `librs/blueos_scrt1` | init/main/atexit/fini、auxv、pthread/emutls | 解析 ELF 或直接管理 registry |

## 5. 必须保持的不变量

1. root、private image、system candidate 和 imported Ready image 的 residency/ownership
   必须显式，不以 SONAME 或地址范围猜测；
2. `AllocationLease` 只存在一个 authority：未提交时在 session rollback log，提交后在
   ThreadGroup 或 SystemDsoRegistry；
3. imported Ready image 不重复 relocate、seal 或执行 constructor；
4. app relocation 可以读取 imported provider 的 exports/ranges，但不能获得其 raw
   allocation release authority；
5. system DSO 自身不能绑定 application-private symbol；
6. `SystemDsoLease` 的 Drop 只减引用并投递 quiescence，不执行 fini 或释放代码；
7. `ApplicationManager`、registry、scheduler 锁内不做 VFS IO、link、cache、ctor/fini；
8. S9 前 entry/link map 不对运行线程可见；S9 后所有地址都有长期 owner；
9. S10 的第一条 constructor 是不可自动回滚的副作用边界；失败进入 Draining/Failed，
   不能宣称外部副作用已撤销；
10. main/fini 运行期间 root 和全部 provider lease 保持有效；
11. 最后一个线程退出前不得释放应用 image；最后 system lease 释放也不能直接卸载；
12. 所有跨 SWI 的结构使用 `abi_version + struct_size`，新增字段只能追加；
13. `TargetAddress` 只有在 ThreadGroup 创建入口或 librs 执行已验证 plan 的极小 unsafe
    边界转换为函数指针；loader 通用层不执行目标地址；
14. Phase 1 policy 始终拒绝未知 dynamic feature 和 relocation；
15. MPS2 无硬件 cache 仍必须执行代码发布所需 DSB/ISB，不能用空 adapter 冒充完成；
16. 当前代码运行在共享特权地址空间，任何“copy-in”“权限”描述都不能升级为隔离承诺。

## 6. 端到端状态与所有权

### 6.1 一次正常启动

```text
OwnedLaunchRequest
  → ApplicationRegistry 预留 handle（Loading）
  → ThreadGroup（Loading/Unlinked）
  → VfsElfReader snapshot
  → DynamicLinker S0–S8
  → LinkPublisher prepare
  → raw leases 分流：private → group，system → registry
  → LinkProduct 安装（Loading/Linked）
  → ApplicationStartStorage 固定地址
  → 主线程进入 blueos_scrt1
  → system/app init plan
  → ApplicationInitComplete
  → ThreadGroup Active / Application Running
```

### 6.2 正常退出

```text
main returns
  → ApplicationBeginExit（禁止新线程）
  → 等待其他 membership 退出
  → atexit
  → FiniPlan
  → 主线程 emutls/TCB cleanup
  → ApplicationFinishExit
  → scheduler 只投递最终 ThreadExited/ReapReady
  → reaper 取走 ReapResources
  → unpublish app link map
  → release private allocation leases
  → release SystemDsoLease
  → system registry quiescence or KeepCached
  → ThreadGroup Reaped / Application Terminated
```

### 6.3 公共状态与内部状态

公共 `ApplicationState` 只允许：

```text
Loading → Running → Stopping → Terminated
   └──────────────────────────→ Failed
```

当前执行后端内部状态只允许：

```text
Loading(Unlinked → Linked → Initializing)
  → Active
  → Draining
  → Reaped
```

Parsed/Mapped/Relocated/Sealed 是 loader typestate，不增加到应用查询 API。

## 7. C18：ARM32 PIC sysroot 与 artifact profiles

### 7.1 C18-a：保留已有模板和第一版 gate

已有 `build@173a130` 作为起点，不重新创建平行模板。先修正现有 action 的输出 stamp、
target 命名和 board `check_all` 接线，使每个 action 确实产生声明的 output。

### 7.2 C18-b：收敛三层配置

配置只允许来自三层：

```text
GN toolchain       → clang + LLD 驱动
board ABI config   → CPU/march/Thumb/soft-float
artifact profile   → static final link / PIE / shared
```

具体修改：

- `blueos_thumbv7m` 的 C/C++/Rust 最终 link 统一由 clang driver + LLD 完成；
- board config 删除与 artifact 类型重复的 `-static/-pie/-shared/-z` 决策；
- 生成并缓存唯一的 ARM32 PIC `core/alloc/compiler_builtins` sysroot；
- `kernel_static` 保持静态最终链接和固定 linker script，但 codegen 是否 PIC 必须由 A/B
  结果明确决定；若计划坚持共享 PIC sysroot，就不能再用 root crate 的
  `-Crelocation-model=static` 掩盖该目标；
- dynamic app 和 DSO 不复用 kernel `link.x`；
- 所有 template 对调用方禁止重复传 linker/output-kind 安全参数。

### 7.3 C18-c：冻结 build-time contract

扩展 `check_blueos_elf.py`：

- 精确检查 ELF32、little-endian、EM_ARM、EABI5、soft-float 和 Thumb；
- dynamic app 必须 `ET_DYN + PT_DYNAMIC + DF_1_PIE + NOW + GNU_RELRO`；
- app 的 `DT_NEEDED` 在 Phase 1 必须精确等于允许集合 `{libc.so.1}`；
- libc 必须 `DT_SONAME: libc.so.1`；
- app/libc 必须有 build-id 和 `.note.blueos.abi` 对应的 `PT_NOTE`；
- dynamic relocation 只允许四种 ARM32 白名单；
- 明确拒绝 `R_ARM_THM_CALL/R_ARM_THM_JUMP24` 等 text relocation；
- export 与 `librs.exports` 对比；
- kernel 必须无运行时 dynamic relocation，entry/vector/copy-zero table 不变。

### 7.4 C18 gate

- kernel 迁移前后记录 text/rodata/data/bss/GOT/固件尺寸；
- MPS2 kernel 实际启动，调度、中断、syscall 和 allocator smoke 不回退；
- `kernel_static`、`dynamic_app`、`dso` 三类正例和负例都由 GN 执行；
- `llvm-readelf -h -l -d -r -s -n` 输出保存为 CI artifact；
- 不允许在单个 app BUILD 中重新加入被 template 禁止的 linker flags。

## 8. C19：生成 `libc.so.1` 并冻结导出

### 8.1 构建产物

在 `librs/BUILD.gn` 保留现有 `librs`/`librs_swi` rlib，不破坏静态应用；新增独立 DSO
target：

```text
libc.so.1                 # runtime artifact，SONAME=libc.so.1
libc.so                   # SDK link-time import/name
libc.so.1.build-id        # 可选构建清单输出
librs.exports             # 唯一导出来源
```

`libc.so.1` 只依赖 PIC 版本的 `blueos_scal_swi`、`core/alloc/compiler_builtins` 和必要
第三方 rlib。它不得解析或直接引用 kernel Rust/C 普通符号。

### 8.2 导出清单

第一版 export 至少覆盖 C29 fixture 真正使用的 ABI：

- `__librs_start_main` 和必要的 crt/runtime symbol；
- malloc/free/realloc/calloc；
- read/write/open/close/lseek 等 smoke 所需接口；
- pthread/emutls 接口；
- `getauxval`；
- BlueOS `spawn`；
- compiler 真实生成并需要的有限 runtime symbols。

采用 version script 或等价的 LLD export-list，把未列出符号全部 local。禁止依赖
Rust symbol mangling 作为 ABI。export 增删必须由独立 ABI diff 审查。

### 8.3 ABI note 与 build-id

`.note.blueos.abi` 至少包含固定 endian 的 versioned payload：

```text
magic/version
execution_model = ThreadGroup
elf_class = 32
machine = ARM
endianness = little
eabi = 5
float_abi = soft
instruction_set = ThumbOnly
branch_mode = PltGotOnly
kernel_abi_min
librs_abi
feature_bits
```

note 必须位于 `PT_NOTE`，不能只存在于 section header。build-id 记录内容身份，ABI note
记录兼容规则，两者不能互相替代。

### 8.4 C19 gate

- `libc.so.1` 没有普通 kernel undefined symbol；
- SONAME、build-id、ABI note 完整；
- dynsym export 与清单完全一致；
- relocation 全部属于 ARM32 Phase 1 白名单；
- 没有 `PT_INTERP/PT_TLS/TEXTREL/RPATH/RUNPATH`；
- 现有 librs static/unit targets 继续构建。

## 9. C20：版本化应用启动与退出 ABI

### 9.1 wire types

在 `kernel/header` 增加 C/Rust 共用、`#[repr(C)]` 的 wire types。句柄不要直接暴露内核
指针；建议固定为两个 `u32`，避免 32/64 位布局歧义：

```rust
#[repr(C)]
pub struct ApplicationHandle {
    pub slot: u32,
    pub generation: u32,
}

#[repr(C)]
pub struct BlueOsStringView {
    pub data: *const u8,
    pub len: usize,
}

#[repr(C)]
pub struct BlueOsAuxvEntry {
    pub key: usize,
    pub value: usize,
}

#[repr(C)]
pub struct BlueOsFunctionPlan {
    pub abi_version: u32,
    pub struct_size: u32,
    pub entries: *const usize,
    pub count: usize,
}
```

`BlueOsApplicationStartInfo` v1 至少包含：

- `abi_version/struct_size/flags`；
- `ApplicationHandle`；
- `argc/argv/envp`；
- `auxv + auxv_count`；
- `init_plan/fini_plan`；
- 可选的 link-map/debug view 只能作为只读、版本化扩展追加。

启动信息不是 Linux initial stack，不能假设 `argc` 位于 SP 顶部。

### 9.2 launch syscall 请求

`ApplicationLaunch` 不扫描无界 C pointer array。librs 先把 POSIX 输入转换为：

```rust
#[repr(C)]
pub struct BlueOsApplicationLaunchRequest {
    pub abi_version: u32,
    pub struct_size: u32,
    pub path: BlueOsStringView,
    pub argv: *const BlueOsStringView,
    pub argc: usize,
    pub envp: *const BlueOsStringView,
    pub envc: usize,
    pub flags: u32,
}
```

handler 按 `struct_size` 先复制固定头，再检查 count/总字节/每字符串上限，最后复制内容
到 kernel-owned `OwnedLaunchRequest`。拒绝 null+非零长度、算术溢出、embedded NUL、路径
逃逸和配额超限。

当前共享特权地址空间没有 fault-safe user page table。Phase 1 至少要求来源范围属于当前
ThreadGroup 已登记的 image/stack/heap region；这只能减少错误指针风险，不能声称阻止
恶意特权代码访问内核。boot 内部启动使用已经 owned 的 kernel request，不走 pointer
copy-in。

### 9.3 syscall 编号

在现有 `NR` 尾部、`LastNR` 之前追加：

- `ApplicationLaunch`；
- `ApplicationInitComplete`；
- `ApplicationBeginExit`；
- `ApplicationFinishExit`。

不得重排旧编号。C20 应把已有值冻结为显式 discriminant 或生成 ABI snapshot，避免以后
在枚举中间插入成员导致全部编号漂移。

### 9.4 C20 gate

- C/Rust 对 `size_of/align_of/offset_of` 一致；
- 32/64 位布局分别有 golden snapshot；
- v1 consumer 接受更大的未来 `struct_size`，拒绝更小的必需前缀；
- 旧 syscall 编号、`SpawnArgs` 和 static start ABI 不变；
- forged generation、错误 membership 和非法状态返回确定错误。

## 10. C21：`blueos_scrt1` 与静态启动兼容

### 10.1 动态 entry

生成最小 Thumb `blueos_scrt1.o`：

```text
_start(ApplicationStartInfo *info)
  → 保持 ABI 要求的栈对齐
  → 取得应用 main
  → __librs_start_main(main, info)
  → 不返回
```

由于当前 kernel `Thread::Entry::Posix` 已能传一个 `void *arg`，Phase 1 可把已验证的 ELF
entry 作为线程 entry，把 pinned `ApplicationStartInfo *` 作为 arg，不需要模拟 Linux
进程初始栈。

地址到函数指针的转换只放在 thread-group backend 的一个 audited unsafe 函数中；调用
前必须同时证明 Thumb bit、canonical X range 和 allocation lease 已经安装到 owner。

### 10.2 保留静态路径

当前无参数 `__librs_start_main()` 不能原地改变成不兼容 ABI。建议：

- 动态 DSO 导出新版 `__librs_start_main(main, info) -> !`；
- 现有静态应用改用 Rust 内部 `__librs_start_main_static()` 或独立 static adapter；
- `kernel/rsrt` 和现有 app 先迁移到 static adapter；
- C29 才迁移 shell 的 artifact 类型，不在 C21 提前改变 boot 行为。

### 10.3 C21 gate

- C hello PIE 真实是 `ET_DYN + PT_DYNAMIC + DT_NEEDED libc.so.1`；
- entry bit 0 为 1，canonical entry 落在 X segment；
- SP 满足 ARM EABI 对齐；
- `_start` 不返回；
- 原静态 shell、kernel image 和 librs unit image 仍启动。

## 11. C22：VFS positional read 与 snapshot

### 11.1 VFS API

底层 `InodeOps` 已有 `read_at`，但 `FileOps` 只公开改变共享 offset 的 `read/seek`。增加：

```rust
pub trait FileOps {
    fn len(&self) -> Result<u64, Error>;
    fn read_at(&self, offset: u64, dst: &mut [u8]) -> Result<usize, Error>;
    fn read_exact_at(&self, offset: u64, dst: &mut [u8]) -> Result<(), Error>;
    // existing read/write/seek...
}
```

`read_exact_at` 循环处理合法短读；遇到零进展、EOF 或 IO error 返回可区分错误。所有
`u64 ↔ usize` 和 `offset + len` 使用 checked conversion。调用不读取或修改 `File.offset`。

### 11.2 snapshot contract

仅持有 `Arc<File>` 不等于内容不可变。增加 VFS 可证明的 snapshot token，例如：

```rust
pub struct FileSnapshotId {
    pub fs_instance: u64,
    pub inode: u64,
    pub content_generation: u64,
    pub len: u64,
}
```

写入、truncate、替换必须改变 `content_generation`。`VfsElfReader` 在创建时固定 token，
每次 read 前后验证 generation；发现改变就返回 `SourceChanged`，让尚未发布的 link
session 完整回滚。不能用 path、mtime 粒度或一次 `stat` 代替内容 generation。

如果某个 filesystem 暂时不能提供稳定 generation，Phase 1 只允许其上的 immutable
build-time catalog artifact；普通可写文件必须拒绝作为 executable source，而不是整文件
读入匿名 buffer 绕过问题。

### 11.3 C22 gate

- positional read 不改变共享 offset；
- 两个 reader 并发交错读取不串扰；
- short read/EOF/IO error/offset overflow 分类稳定；
- 中途写入或 truncate 产生 `SourceChanged`；
- `ArtifactIdentity::FileIdentity` 编码 snapshot identity，不编码显示路径。

## 12. C23：loader 桥接、kernel adapters 与 ARM cache

### 12.1 C23-a：published/imported system image contract

这是当前代码进入 Phase 1 的首要 loader 改动。增加 loader-neutral 的 immutable
descriptor，名称可按最终代码风格调整：

```rust
pub struct PublishedImageDescriptor {
    identity: ArtifactIdentity,
    soname: Option<DependencyName>,
    ownership: ImageOwnership,
    allocation: ImageAllocation,
    load_bias: TargetAddress,
    regions: Vec<PublishedRegion>,
    exports: PublishedSymbolTable,
    program_headers: ProgramHeaderRuntimeInfo,
    sealed: SealedState,
}

pub enum DependencyResolution<R> {
    Load(ResolvedArtifact<R>),
    Import(ImportedImageDescriptor),
}
```

具体语义：

- `Load(SystemCandidate)` 走当前 S0–S8，commit 时可以成为 registry 新实例；
- `Import` 表示 registry 中已经 Relocated/Initialized/Ready 的 provider；
- imported image 加入 graph 和 scope，但跳过 allocation、relocation、seal 和 init plan；
- descriptor 必须带 export value/type/visibility、runtime regions 和 identity，足以完成符号
  lookup 与控制流/data target range 检查；
- root program-header view 保留到 `LinkProduct`，供 start storage 构造 auxv；
- `PreparedLinkManifest/LinkMapEntry` 增加显式 residency/ownership，publisher 不通过 image id
  或 SONAME 猜 lease 去向；
- 所有 metadata copy 计入 `SessionLimits.total_runtime_metadata_bytes`；
- registry 的 `SystemDsoLease` 不进入 loader crate。kernel resolver 在整个 staged link 期间
  持有 pending lease，成功时由 publisher receipt 接管，失败时由 resolver/pending set Drop。

推荐 `ApplicationLoader` 使用 staged API，而不是一口气 `DynamicLinker::link()`：

```text
begin(root)
  → close_dependencies(&mut resolver)
  → resolver.finish_resolution() 取得 permits/pending leases
  → freeze_scopes → relocate → seal
  → publish(&mut KernelLinkPublisher)
```

这样 resolver 的临时 registry authority 能在 publish 前显式转交，避免两个独立可变借用
之间使用全局 side channel。

### 12.2 C23-b：`VfsElfReader` 与 resolver

`VfsElfReader` 实现 loader 的 `ElfReader`，持有已经准入的 snapshot，不接受任意 path。

`SystemLibraryPaths` 由 board/product config 提供固定映射：

```text
libc.so.1 → /system/lib/libc.so.1 + expected ABI/build-id policy
```

`ApplicationArtifactResolver`：

- root 只能来自 build-time allowlist；
- system dependency 只能按精确 byte-name 查 catalog；
- Phase 1 不搜索 cwd、`LD_LIBRARY_PATH`、RPATH/RUNPATH 或应用包 `lib/`；
- Ready instance 返回 imported descriptor 并持有 pending lease；
- Vacant instance取得 permit并返回 `Load(SystemCandidate)`；
- Loading instance 在不持 loader memory/application manager lock 时等待；
- identity、reader 和 ABI/build-id 必须来自同一 snapshot。

### 12.3 C23-c：`FlatImageMemory`

`FlatImageMemory` 不是每个 session 孤立的 backing owner，而是指向同一 shared-flat memory
service 的轻量 handle。每个 link 可以持有独立 handle，内部 allocation table 按调用粒度
同步，避免在整个 VFS/link 期间持有全局 spin lock。

实现要求：

- `AllocationId` 全局唯一或绑定 backend instance nonce；
- allocation table 保存稳定、不移动的 backing；
- 每个 read/write/zero/span/protect 校验完整 descriptor；
- `Layout::from_size_align` 和地址计算全部 checked；
- abort/release 无分配、无失败、exactly once；
- MPS2 `protect` 如实返回 `LogicalOnly`；
- reaper 可以通过同一 service 的另一个 handle 释放 committed lease；
- 不依赖“current allocation”或隐含当前 ThreadGroup。

### 12.4 C23-d：ARM code-cache service

在 `kernel/src/arch/arm/cache.rs` 实现 `CodeCache` adapter：

- `prepare` 保留全部 executable ranges、alignment 和 execution scope；
- MPS2/Cortex-M3 明确报告无 D/I cache maintenance 需求；
- publication 前仍执行 DSB/ISB；
- `synchronize` 返回与 prepared token 完全一致的 outcome；
- 对 cache-enabled Cortex-M，如果尚无 D-clean/I-invalidate 实现，capability 必须导致
  `UnsupportedByProfile`，不能沿用 MPS2 空路径；
- cache/barrier 操作在所有 relocation write 完成后、entry 可见前执行。

### 12.5 C23 gate

- 第二个 link 导入 Ready libc 时 allocation/relocation/cache/protect 计数均不增加；
- imported symbol function/data 地址与第一次发布 descriptor 完全一致；
- imported lease 在失败路径释放，成功后移入 receipt；
- wrong backend/allocation descriptor 不访问 backing；
- MPS2 barrier trace 位于最后 write 与 publication 之间；
- adapter error 保留 loader stage/context；
- production kernel 不出现第二套 ELF/relocation 实现。

## 13. C24：System DSO registry、permit 与 lease

### 13.1 artifact 与 instance 分层

```rust
pub struct DsoArtifact {
    pub identity: ArtifactIdentity,
    pub soname: DependencyName,
    pub path: SystemPath,
    pub abi: BlueOsAbiNote,
}

pub struct SystemDsoInstance {
    pub generation: u32,
    pub descriptor: PublishedImageDescriptor,
    pub allocation_owner: OwnedAllocationSet,
    pub fini_plan: FiniPlan,
    pub init_state: InitState,
    pub lease_count: usize,
}
```

artifact catalog 只描述可信文件；instance registry 才描述某个 `LinkDomainId` 中已经映射
的运行实例。不要把 path/build-id cache 和运行时 load-bias/GOT/ctor 状态混成一个对象。

### 13.2 状态机

```text
Vacant
  → Loading(g, permit)
  → Relocated(g)
  → Initializing(g)
  → Ready(g, leases=n)
  → Quiescing(g)
  → Vacant(g+1) / ReadyCached(g)

Loading/Relocated/Initializing
  → Failed(g)
  → Quiescing/KeepPoisoned
```

同一 `(LinkDomainId, SONAME)` 只有一个 loading generation。所有 waiter 记录期待的
generation，不能把上一次失败后的新实例当成原请求结果。

### 13.3 permit/lease 规则

- `acquire_or_begin_load` 返回 `ReadyLease | LoadPermit | WaitHandle`；
- permit 是该 generation 唯一发布权，Drop/fail 必须唤醒 waiter；
- `publish_relocated` 的所有 capacity/identity/generation 检查在 LinkPublisher prepare
  完成；commit 只 move/swap；
- 第一位应用在 init 完成前使 registry 保持 `Initializing`，并发应用等待；
- `mark_ready` 只接受 initializer handle + generation；
- `SystemDsoLease` 含 registry identity/SONAME/generation，不能 Clone 出无记账引用；
- lease Drop 只入队 `TryQuiesceSystemDso`；
- quiescence 需要零 lease、零 initializing/waiter owner、零已登记 code callback/use token；
- 无法证明静默时使用 `KeepCached`。Phase 1 可以保守常驻，不能危险卸载；C29 的最小
  fixture 需要提供可证明静默的路径验证卸载和 generation+1 重载。

### 13.4 初始化失败

constructor 已开始后不能回滚外部副作用。如果 initializer thread 在
`ApplicationInitComplete` 前异常退出：

- application 标记 Failed 并进入 Draining；
- registry generation 不得直接回到 Vacant 让下一应用重试同一内存；
- 没有精确 quiescence/fini 证据时保留 `Failed/KeepPoisoned`；
- waiter 获得同一 generation 的失败原因；
- 后续是否允许新 generation 由显式 retry policy 决定，不在 Drop 中隐式重试。

### 13.5 C24 gate

- 两线程同时请求 Vacant libc 只有一个 permit；
- Ready fast path 只增加 lease，不重新 map；
- permit owner 任意阶段失败都会唤醒 waiter；
- stale permit/lease/generation 被拒绝；
- lease Drop 不执行应用代码、不释放 image；
- 最后 lease 只产生 quiescence event；
- KeepCached 和成功 unload/reload 两条路径都可观察且有结构化原因。

## 14. C25：单一 ApplicationManager 与 ThreadGroupBackend

### 14.1 公共对象

```rust
pub enum ExecutionModel {
    ThreadGroup = 1,
    Process = 2,
}

pub enum ApplicationState {
    Loading,
    Running,
    Stopping,
    Terminated,
    Failed,
}

pub struct ApplicationManager {
    registry: ApplicationRegistry,
    thread_group: ThreadGroupBackend,
    events: ApplicationEventQueue,
    #[cfg(blueos_user_process)]
    process: ProcessBackend,
}
```

Phase 1 不创建 `Process` 空壳。ABI 能识别 `Process`，但未启用唯一配置
`blueos_user_process` 时稳定返回 `UnsupportedExecutionModel`。

### 14.2 launch 编排

`ApplicationManager::launch(OwnedLaunchRequest)`：

1. 验证 allowlist/manifest/ABI note/profile/quota；
2. 在短 registry lock 内预留 slot+generation，状态为 Loading；
3. 创建尚未有运行线程的 ThreadGroup；
4. 释放 manager lock；
5. 调用 `ThreadGroupBackend::prepare` 和 `ApplicationLoader`；
6. 安装 link product 和 start storage；
7. 创建主线程并登记 membership；
8. 发布可调度线程；
9. 返回 handle；
10. 任一步失败把实例标为 Failed，并向 reaper 投递资源包。

外部 syscall 和 boot 自举都调用这一 owned-request 入口。不能维护一条“boot 快速路径”
直接调用 `load_elf()`。

### 14.3 handle 与 table

- handle 使用 slot+generation 防 ABA；
- table lock 只做插入/查询/状态更新；
- 查询返回值复制，不暴露内部 Arc、lease 或可写状态；
- quota 至少覆盖 image bytes、metadata bytes、线程数、argv/envp bytes、heap bytes；
- 状态转换使用明确的 expected→next，非法转换返回错误；
- slow VFS/link/wait/cache/thread start 不在 table lock 内执行。

### 14.4 C25 gate

- fake slow prepare 时其他 query/launch 不被全局 table lock 阻塞；
- 同路径两次 launch 得到不同 generation handle；
- forged/stale handle 失败；
- Process 请求稳定 unsupported；
- link/start 失败只留下可查询的 Failed 终态或按策略回收后的 tombstone；
- 没有 `AppRuntime`、`RtosAppDomain` 或第二个 manager。

## 15. C26：把 LinkProduct 发布到 ThreadGroup

### 15.1 publication receipt

实现 kernel `LinkPublisher`，其 receipt 显式拆分：

```rust
pub struct KernelLinkReceipt {
    private_allocations: OwnedAllocationSet,
    system_leases: Vec<SystemDsoLease>,
    newly_published_system: Vec<SystemDsoGeneration>,
    link_map_generation: u32,
}
```

`prepare_batch` 完成：

- group 尚无 link product；
- manifest image/lease/residency 一一对应；
- system permit identity/generation/SONAME 匹配；
- group、registry、link-map 所需容量已预留；
- entry、root program headers 和所有 start-info 输入存在；
- pending imported leases 与 imported nodes 一一对应。

`commit_batch` 只做 move/swap：private raw lease 进入 group receipt，首次 system candidate
进入 registry instance，当前 group 获得相应 system lease。commit 不分配、不验证、不
调用 cache/ctor，也不返回 `Result`。

### 15.2 `install_link_product`

当前 `LinkProduct` 在 loader commit 后才构造。kernel 必须提前准备唯一 install slot，
使随后 `ThreadGroup::install_link_product(product)` 为不可失败 move；只有 install 完成才
把内部状态变为 `Loading::Linked`。若采用直接让 group publisher 成为可见提交点，也必须
保证读者只能看到旧状态或完整 product，不能看到半张 link map。

### 15.3 `ApplicationStartStorage`

storage 自有并固定：

- argv/envp 字节和尾零；
- argv/envp pointer arrays；
- execfn；
- auxv；
- init/fini target arrays；
- 必要时的 main program-header copy；
- pinned `BlueOsApplicationStartInfo`。

先完成所有 Vec/Box 分配和 pointer fixup，再创建主线程。storage 由 ThreadGroup 持有到
最后线程退出，不能位于 launch 函数栈上。

建议首版 auxv：

- `AT_ENTRY`；
- `AT_PHDR/AT_PHENT/AT_PHNUM`（可以指向 pinned phdr copy，但语义要记录）；
- `AT_EXECFN`；
- `AT_PAGESZ` 或实际 protection granule；
- BlueOS 自定义 application handle/ABI version key；
- `AT_RANDOM` 只有 board 提供真实熵时才加入，禁止伪随机常量冒充安全随机。

### 15.4 C26 gate

- S0–S9 全部通过真实 VFS/memory/cache/registry adapters；
- publisher prepare 每个失败点不留下 group link map、system instance 或 lease；
- commit 后不存在可失败的资源配对；
- start-info 所有嵌套 pointer 在 launch 返回后仍稳定；
- entry/function plan 地址均由仍存活 owner 覆盖；
- 第一次加载 libc 后 registry 为 Relocated/Initializing，第二次只在 Ready 后导入；
- reaper 能从 receipt 恰好一次取走所有资源。

## 16. C27：membership、两阶段退出和 deferred reaper

### 16.1 Thread membership

给 `Thread` 增加不可由应用指定的 `Option<ThreadGroupMembership>`。`Builder` 增加内部
setter 和面向应用的 fallible build 路径。

规则：

- 主线程由 backend 显式登记；
- `CreateThread` 从当前线程继承 membership，忽略/拒绝用户提供的 group id；
- group 在 Draining 后拒绝创建新线程；
- thread count reservation 在进入 ready queue 前完成；
- build/queue 失败撤销 reservation；
- 静态 kernel thread 的 membership 为 None。

### 16.2 新 syscall 校验

- `ApplicationLaunch`：copy-in 后调用统一 manager launch；
- `ApplicationInitComplete`：只允许对应 group 的 initializer/main thread 和当前 generation；
- `ApplicationBeginExit`：只允许 group exit coordinator，原子禁止新线程并等待子线程；
- `ApplicationFinishExit`：只在 fini/TCB cleanup 协议完成后接受；
- 普通 pthread 继续使用 `ExitThread`。

所有 handler 都从 `scheduler::current_thread()` 反查 membership，不能信任应用传入的裸
handle 选择其他 group。

### 16.3 scheduler 边界

当前 scheduler 会在 context-switch cleanup 路径调用 thread cleanup。Phase 1 只允许它：

- 完成现有 stack/cleanup 动作；
- 取出 thread 自带的预分配 event node；
- 投递 `ExecutionUnitExited`；
- 不取得 ApplicationManager/SystemDsoRegistry 长锁；
- 不执行 fini、不删除 link map、不释放 image。

event queue 必须有容量保证。优先把退出 event node 随 Thread 预分配，避免 scheduler
cleanup 遇到 OOM 后丢失最后一次退出通知。

### 16.4 reaper 顺序

`ThreadGroup::take_resources_for_reap()` 只有在以下条件全部满足时成功一次：

- 已禁止新线程；
- membership count 为零；
- 正常路径 fini 已完成，或异常路径已经明确记录 `SkipFini(reason)`；
- 没有启动/初始化 owner；
- resources 尚未被取走。

reaper 在 application/registry 锁外执行：unpublish link map → release private images →
release system leases → 投递 system quiescence → drop start storage。正常状态进入
Terminated；异常状态保留 Failed reason。

### 16.5 C27 gate

- pthread membership 自动继承；
- Draining 后 CreateThread 失败；
- 最后线程退出前 allocation/reaper 调用次数为零；
- scheduler cleanup 不直接调用 release/fini；
- 重复/stale BeginExit/FinishExit 被拒绝；
- 正常、main 异常、子线程异常和 launch failure 都只回收一次；
- event queue 满/OOM 不丢失退出通知。

## 17. C28：librs 动态应用 runtime

### 17.1 `LibcApplicationContext`

```rust
pub struct LibcApplicationContext {
    handle: ApplicationHandle,
    auxv: Box<[BlueOsAuxvEntry]>,
    atexit: RwLock<Vec<AtExitEntry>>,
    exit_coordinator: AtomicBool,
}
```

`PthreadTcb` 增加 `Option<Arc<LibcApplicationContext>>`。主线程从 start info 创建 context；
`pthread_create` 在调用 CreateThread 前继承同一个 Arc，并在失败时撤销。TCB 的 Drop/退出
顺序不能让 emutls destructor 在 context 或 DSO lease 已释放后执行。

### 17.2 动态 `__librs_start_main`

固定顺序：

1. 校验 `abi_version/struct_size` 和所有 count/pointer；
2. 注册主线程 TCB 和 `LibcApplicationContext`；
3. 初始化当前应用需要的 stdio/runtime view；
4. 按顺序执行 init plan；
5. 调用 `ApplicationInitComplete`；
6. 调用 `main(argc, argv, envp)`；
7. 调用 `ApplicationBeginExit(status)`，等待其他线程退出；
8. 执行 application-owned atexit；
9. 按存储顺序执行已经是逆序的 fini plan；
10. 执行主线程 emutls/pthread-key cleanup；
11. 调用 `ApplicationFinishExit`，随后 `ExitThread`，永不返回。

constructor/finalizer 调用全部位于 loader/application/registry lock 外。

### 17.3 auxv 和 `getauxval`

`getauxval(key)` 从当前 TCB 的 context 查找；未找到返回 0 并按 POSIX 约定设置 errno。
不要使用一个可被两个应用覆盖的全局 auxv pointer。

### 17.4 emutls

现有 `tls.rs` 已通过 pthread key 保存每线程对象数组。Phase 1 需要确认：

- `ERRNO` 等 target 上不会产生 native `PT_TLS`；构建 gate 继续拒绝 `PT_TLS`；
- emutls control object 的 owner image 在对应线程退出前由 group/system lease 保活；
- 子线程的数组彼此独立；
- destructor 发生在 `ExecutionUnitExited` 事件和 image release 之前；
- system-level control index 可以全局递增，但 application-owned value 不能跨线程共享。

### 17.5 application-local 与 system-global 状态审计

当前 librs 有全局 TCB map、pthread key map、emutls key、stdio singleton 等状态。Phase 1
至少逐项分类：

| 状态 | Phase 1 ownership |
| --- | --- |
| errno/emutls value | thread |
| argv/envp/auxv/atexit | application context |
| pthread TCB map/key registry | system libc instance，key value 属于 thread |
| allocator | system service，按 group 记账但不宣称隔离 |
| stdin/stdout/stderr | 首版明确 system-shared 或改为 application context |
| cwd/environment | 若尚未 application-local，文档明确 system-shared 限制 |

不得在没有审计时声称共享 libc 等价于每进程 libc 状态隔离。

### 17.6 `spawn`

新增稳定 C ABI `spawn(path, argv, envp)`：

- 有界扫描调用方的 null-terminated arrays；
- 转为 `BlueOsStringView` 数组和 versioned launch request；
- 调用 `ApplicationLaunch`；
- 返回 `ApplicationHandle` 或 errno；
- 不让 librs 直接访问 kernel `ApplicationManager` symbol。

### 17.7 C28 gate

- argv/envp/auxv 正确；
- ctor 恰好一次，fini 次序正确；
- main 返回值传到 application exit status；
- pthread context 继承；
- 两线程 emutls 值独立且 destructor 各一次；
- `getauxval` 不跨应用串值；
- `spawn` 返回有效 generation handle；
- 所有 target function 调用时对应 image/lease 仍存活。

## 18. C29：shell 和 QEMU 纵向 MVP

### 18.1 artifact 布局

建议生成只读测试 image：

```text
/system/lib/libc.so.1
/apps/shell/app.elf
/apps/hello/app.elf
/apps/args/app.elf
/apps/errors/*.elf
```

每个路径和预期 build-id/ABI note 写入构建期 catalog。Phase 1 不从任意可写路径执行。

### 18.2 boot 与 shell

- boot 初始化 VFS、FlatImageMemory service、SystemDsoRegistry、ApplicationManager 和 reaper；
- 内核用 owned request 调 `ApplicationManager::launch("/apps/shell/app.elf", ...)`；
- shell 本身是动态 PIE，第一次启动触发 libc loading permit；
- shell 初始化完成后 registry libc 进入 Ready；
- `run` 命令调用 librs `spawn`，通过 SWI 启动 hello；
- hello 解析到 imported Ready libc，不产生第二份 libc allocation。

### 18.3 正向用例

- C hello；
- argc/argv/envp；
- `getauxval(AT_ENTRY/AT_EXECFN/AT_PHNUM/BlueOS handle)`；
- malloc/free/heap；
- open/read/write/close 文件 IO；
- data/BSS；
- undefined weak data 的允许语义；
- app → libc function symbol；
- app → libc data symbol；
- app ctor 一次、fini 一次；
- pthread + 两线程 emutls 独立；
- shell 与两个并发 app 复用同一 libc address/build-id/generation；
- 正常退出后 app private image 释放；
- 最后 lease 后 KeepCached 或可证明 quiescent 的 unload/reload generation+1。

### 18.4 负向用例

- app 缺 `libc.so.1`；
- 禁止的额外 `DT_NEEDED`；
- missing strong symbol；
- ABI note/soft-float/Thumb/build-id 不匹配；
- 非 NOW、未知 relocation、TEXTREL、W+X、可执行栈；
- stale/forged application handle；
- argv/envp count、总字节、pointer range 超限；
- init 前主线程异常、运行中子线程异常；
- resolver、allocation、relocation、cache、publication、thread creation 故障。

### 18.5 C29 oracle

QEMU checker 不只匹配“hello”。输出至少带稳定字段：

```text
APP_LAUNCHED handle=<slot:generation> path=<...>
DSO_LOAD soname=libc.so.1 generation=<g> base=<...>
DSO_REUSE soname=libc.so.1 generation=<g> base=<same>
APP_INIT_COMPLETE handle=<...>
APP_EXIT handle=<...> status=<...>
APP_REAP handle=<...> private_images=1
DSO_QUIESCENCE generation=<g> result=<unloaded|cached>
```

断言第二个应用的 libc base/generation 与第一个相同，且 libc allocation/relocation/init
计数仍为 1。这样才能区分真实共享和重复加载。

## 19. 跨提交依赖与推荐顺序

```text
Phase 0.5 C17/public API
          │
          └──────────────→ C23-a imported/published image bridge
                              │
C18-a existing profiles → C18-b/c toolchain + ABI gate
          │                   │
          └→ C19 libc.so.1 ─→ C21 blueos_scrt1
                    │              │
C20 start/exit ABI ─┼──────────────┼─────────────┐
                    │              │             │
C22 VFS snapshot ─→ C23-b/c/d adapters/cache     │
                               │                 │
                          C24 system registry    │
                               │                 │
                          C25 manager/group      │
                               └─────┬───────────┘
                                     ↓
                            C26 publish/start storage
                                     ↓
                            C27 membership/reaper
                                     ↓
                            C28 librs runtime
                                     ↓
                            C29 QEMU vertical MVP
```

实际开工顺序建议：

1. 先完成 C18-b/C18-c，同时冻结 C20 wire ABI；
2. C19 生成真实 libc 后立刻让 C18 dynamic-app gate 吃到真实 import；
3. C22 与 C21 可独立推进；
4. 在写 kernel registry 前完成 C23-a，证明 Ready libc 可以被导入；
5. 再实现 C23 adapters、C24 registry 和 C25 manager；
6. C26 首次打通 `DynamicLinker → ThreadGroup`，此时可用测试 trampoline，不把它当 MVP；
7. C27/C28 完成真实线程和 libc 生命周期；
8. C29 最后迁移 shell 并收口所有 gate。

## 20. 跨仓库提交拆分

BlueKernel 是 repo-managed workspace，每个仓库独立提交。建议拆分：

| 逻辑编号 | 仓库 | 建议 subject |
| --- | --- | --- |
| C18-b | `build`/target spec 所属仓库 | `build: unify the ARM32 PIC toolchain` |
| C18-c | `build` | `build: enforce Phase 1 ELF contracts` |
| C19-a | `librs` | `librs: freeze the shared libc export surface` |
| C19-b | `build`/`librs` | `librs: produce libc.so.1` |
| C20-a | `kernel/header` | `header: add the versioned application ABI` |
| C20-b | `kernel` | `kernel: register application lifecycle syscalls` |
| C21 | `librs` | `librs: add the ARM32 dynamic start object` |
| C22 | `kernel` | `vfs: add executable snapshot reads` |
| C23-a | `kernel/loader` | `loader: import published system images` |
| C23-b | `kernel` | `kernel: implement loader platform adapters` |
| C23-c | `kernel` | `arm: publish executable code with cache barriers` |
| C24 | `kernel` | `application: add system DSO permits and leases` |
| C25 | `kernel` | `application: add the thread-group backend` |
| C26 | `kernel` | `application: publish linked images into thread groups` |
| C27-a | `kernel` | `thread: inherit application membership` |
| C27-b | `kernel` | `application: defer thread-group reaping` |
| C28 | `librs` | `librs: initialize and exit dynamic applications` |
| C29-a | app/shell 所属仓库 | `shell: launch dynamic applications` |
| C29-b | `build`/`kernel` | `test: run the ARM32 dynamic application MVP` |

producer 先于 consumer：header → kernel/librs，build template → librs/app，loader public
contract → kernel adapter。跨仓库升级采用“先增加兼容能力、再切调用方、最后删除旧路径”，
不要求制造无法单仓库回滚的巨型提交。

## 21. 测试与验证计划

### 21.1 测试分层

| 层 | 重点 |
| --- | --- |
| build contract | ELF type、ABI note、SONAME、exports、relocation、NOW/RELRO |
| loader host | imported Ready image、symbol/range、lease handoff、fault rollback |
| kernel unit | registry generation、manager/group 状态、membership、event/reap 顺序 |
| librs unit | start-info、auxv、TCB inheritance、emutls、atexit/fini |
| QEMU integration | 真实 app→libc、SWI、线程、IO、退出、共享和重载 |

当前可以先实现生产结构，但每个 commit 的 gate 仍应记录为 TODO issue/清单；C29 合入前
必须全部转为 GN 可执行 target，并接入 `check_all`。

### 21.2 fault injection

至少覆盖第 N 次失败：

- VFS open/len/read/snapshot validation；
- resolver/registry permit/wait；
- memory allocate/write/zero/read/protect/release；
- relocation plan/write；
- cache prepare/synchronize；
- publisher prepare；
- start-storage allocation/fixup；
- thread build/queue；
- event enqueue/reaper；
- application init complete 前异常退出。

每个失败点断言原始错误保留、无半张 link map、无 dangling permit/lease、无重复
abort/release，且已有 Ready libc 不被新应用失败破坏。

### 21.3 推荐 GN 验证入口

最终至少形成或接入：

```text
//build:check_arm32_artifact_profiles
//kernel/loader:check_dynamic_linker
//kernel/kernel:check_application
//librs:check_librs
//apps/dynamic:check_dynamic_apps
//apps/shell:run_dynamic_shell
```

常用验证命令以 GN/Ninja 为主：

```bash
gn gen out/qemu_mps2_an385.release.dsc \
  --args='build_type="release" board="qemu_mps2_an385"'
ninja -C out/qemu_mps2_an385.release.dsc check_all
```

开发过程中可单独构建 producer target，但不能用 Cargo 结果替代最终 GN toolchain 和
QEMU artifact 验证。

## 22. 主要风险与控制

| 风险 | 后果 | 控制 |
| --- | --- | --- |
| 把 SystemCandidate 当成已共享 | 两应用实际映射两份 libc | C23-a imported descriptor；C29 比较 base/generation 和 load count |
| LinkProduct 丢失 export/phdr metadata | Ready libc 无法再次提供符号，auxv 不完整 | published descriptor 长期保留最小 immutable metadata |
| VFS reader 只保存 path 或普通 file handle | link 读到混合版本 | content generation snapshot；不支持者只允许 immutable catalog |
| kernel/app/DSO code model 不一致 | text relocation、远跳或启动回退 | 三层配置、真实 readelf/map/QEMU A/B |
| libc 导出过多 | 应用形成不受控内部 ABI | allowlist/version script + ABI diff |
| 在 registry lock 内 link/ctor | 死锁、长关中断和不可预测延迟 | permit/wait handle；所有慢操作在锁外 |
| lease Drop 直接卸载 | 当前栈或 callback 跳入已释放代码 | Drop 只投递 quiescence；reaper+evidence 决策 |
| 当前特权模型被误写成隔离 | 错误安全承诺 | 仅可信 allowlist；记录 LogicalOnly；Phase 3 签名，另立硬件隔离项目 |
| librs 全局状态跨应用串扰 | cwd/stdio/atexit/TLS 行为错误 | C28 ownership inventory；必要状态迁入 LibcApplicationContext |
| ctor 中途失败 | 已发生副作用无法回滚 | 明确 S10 边界；Failed/Draining；system generation fail-closed |
| scheduler 路径执行回收 | 锁序、延迟、栈上代码释放 | 预分配退出事件 + deferred reaper |
| unload libc 时仍有函数指针 | use-after-free | callback/use token + conservative KeepCached |

## 23. Phase 1 完成清单

- [ ] C18 三类 artifact profile 由同一 ARM32 toolchain/sysroot 产生并通过真实 gate；
- [ ] kernel clang+LLD/PIC 决策有尺寸、map、启动 A/B 证据；
- [ ] `libc.so.1` 的 SONAME、build-id、ABI note、exports 和 relocation 已冻结；
- [ ] dynamic app 是无 `PT_INTERP` 的 Thumb PIE，且只依赖 `libc.so.1`；
- [ ] start/exit ABI 版本化且旧 syscall 编号不变；
- [ ] VFS reader 不改变共享 offset并绑定不可变 snapshot；
- [ ] loader 可以导入 Ready system image，不重复 map/relocate/seal/init；
- [ ] published product 保留 external linking 和 auxv 所需 metadata；
- [ ] FlatImageMemory 支持并发多 allocation、descriptor 校验和 exactly-once release；
- [ ] MPS2 code publication 明确执行 DSB/ISB并报告真实 capability；
- [ ] registry permit、wait、Ready lease 和 generation 状态机完成；
- [ ] 两应用只共享一个 libc runtime instance；
- [ ] ApplicationManager/ThreadGroup 公共与内部状态分离；
- [ ] LinkProduct 原子安装，start storage 所有 pointer 稳定；
- [ ] 主线程和 pthread membership 完成，Draining 后不能创建线程；
- [ ] init/main/atexit/fini/emutls/exit 顺序完成且不在 loader lock 内执行；
- [ ] scheduler cleanup 只投递事件，reaper 负责最终释放；
- [ ] 缺依赖、缺符号、ABI 不匹配和所有 fault point 不破坏 Ready registry；
- [ ] C hello、argv/envp/auxv、heap、IO、BSS、weak、function/data symbol、ctor 通过；
- [ ] pthread emutls 隔离和 destructor 通过；
- [ ] 正常退出在最后线程前不释放 image；
- [ ] 最后 system lease 后产生可审计的 unload 或 KeepCached 结论；
- [ ] `qemu_mps2_an385` C29 完整纵向 gate 进入 `check_all`；
- [ ] Phase 0/0.5 和现有静态应用回归不退化；
- [ ] 文档没有把 allowlist、LogicalOnly 或 relocation target 检查描述为签名、隔离或 CFI。

## 24. 建议的第一个实现切片

不要从完整 ApplicationManager UI 或 shell 命令开始。第一个能降低最大风险的切片是：

1. 收口 C18，生成真实 `libc.so.1` 和最小 hello PIE；
2. C22 实现只读 VFS snapshot reader；
3. C23-a 让 host 上“第一次 Load libc、第二次 Import libc”使用同一 descriptor；
4. C23-c 实现 MPS2 FlatImageMemory/cache adapter；
5. 用一个尚未公开给 shell 的 kernel integration harness 执行
   `DynamicLinker → KernelLinkReceipt`；
6. 断言第二次 link 的 libc allocation/relocation/init 计数为零。

这个切片先证明 Phase 1 最容易出现根本性返工的部分——真实 artifact、snapshot、
published metadata 和 system DSO reuse。它稳定后再叠加 ApplicationManager、线程 ABI 和
librs 生命周期，能避免到了 C29 才发现 registry 只能重复装载 libc。
