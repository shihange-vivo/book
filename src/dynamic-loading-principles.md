# BlueOS 动态加载原理与方案讲解

## 1. 为什么需要动态加载

blueos目前把应用和内核链接成单一镜像烧录。动态加载解决三类问题：

1. **共享运行时**：每个应用不再静态带一份 libc，全系统共享一个 `libc.so.1`，显著节省无 MMU 设备的 RAM；
2. **独立交付**：应用作为独立工件安装、升级、回滚，不需要重烧内核；
3. **扩展性**：应用可以携带私有 DSO（如 `libfoo.so`）。

BlueOS 的现实起点：内核 loader 已有 `ET_DYN`/`ET_EXEC` 装载原型（goblin 解析、`PT_LOAD` 复制、`R_RISCV_RELATIVE` 重定位），VFS、线程创建、SWI syscall ABI、emutls 均已就绪。缺的是**工程化的链接语义**（依赖、符号、生命周期）和**正确性基线**（BSS、load bias、fail-closed）。

## 2. 原理：装载与链接是两件事

### 2.1 ELF 文件格式：文件视图与运行时视图

ELF 不是把内存内容原样平铺到磁盘的文件，而是用 ELF Header 连接两套视图：**section 是编译、静态链接和调试视图，segment 是装载和运行时视图**。BlueOS 首发动态应用 profile 采用 32 位小端 ARM `ET_DYN`，目标为 `thumbv7m-vivo-blueos-newlibeabi`；loader 首先校验 `ELFCLASS32`、`ELFDATA2LSB`、`EM_ARM`、`e_flags`、文件类型、版本和表项大小，再把已签名 manifest/`PT_NOTE` 携带的 BlueOS ABI note（构建时 section 名为 `.note.blueos.abi`）中的 Thumb v7-M 与 soft-float EABI 声明纳入准入。构建门禁可以额外检查 `.ARM.attributes`，但运行时不能依赖可被 strip 的 section header。Cortex-M 只执行 Thumb 指令，因此入口和函数地址还必须遵守 Thumb bit 规则。

```text
ELF 文件
├─ ELF Header（Ehdr）
│  ├─ e_type / e_machine       文件类型与目标架构
│  ├─ e_entry                  ELF 虚址形式的入口
│  ├─ e_phoff / e_phnum        Program Header Table 的位置与数量
│  └─ e_shoff / e_shnum        Section Header Table 的位置与数量
├─ Program Header Table（运行时视图）
│  └─ PT_LOAD / PT_DYNAMIC / PT_TLS / PT_GNU_RELRO / ...
├─ 各 segment/section 的文件内容
└─ Section Header Table（链接与调试视图，可被 strip）
   └─ .text / .rodata / .data / .bss / .dynsym / .rel.dyn / ...
```

两套视图不是一一对应的。静态链接器通常把多个带 `SHF_ALLOC` 的 section 合并进少量 segment，例如：

```text
PT_LOAD  R-X  ← .text、.plt
PT_LOAD  R--  ← .rodata、.dynsym、.dynstr、.rel.dyn
PT_LOAD  RW-  ← .dynamic、.got、.data、.bss
                  └─ PT_GNU_RELRO 标出 relocation 后应转为只读的子区间
```

section header 可以被裁掉，运行时仍能依靠 program header 和 `PT_DYNAMIC` 完成装载与链接。因此产品 loader 不能通过 `.text`、`.dynsym` 等 section 名称驱动运行时流程，也不能要求 section header 必须存在；section header 只用于离线分析、调试和构建检查。

真正决定映像内存内容的是 `PT_LOAD`。每个表项用以下字段描述文件区间和内存区间：

| 字段 | 含义 |
| --- | --- |
| `p_offset` | segment 内容在 ELF 文件中的偏移 |
| `p_vaddr` | 链接器为该 segment 指定的 ELF 虚址 |
| `p_filesz` | 需要从文件复制的字节数 |
| `p_memsz` | 运行时占用的总字节数，必须满足 `p_filesz <= p_memsz` |
| `p_flags` | 最终的 `PF_R/PF_W/PF_X` 权限 |
| `p_align` | 文件偏移和虚址共同满足的对齐要求 |

对于 `ET_DYN`，所有 `PT_LOAD` 保持同一个 load bias `B`，第一个运行时字节位于 `B + p_vaddr`。loader 把文件中的 `p_filesz` 字节复制到这里，再把尾部 `p_memsz - p_filesz` 字节清零；后者通常对应 `.bss`。`PT_LOAD` 之间的地址空洞不是可随意访问的对象。`e_entry` 同样是 ELF 虚址，实际入口为 `B + e_entry`；发布前必须按架构规范化 entry，并验证最小指令跨度完整落在当前映像的可执行区域。

其他 program header 主要提供附加语义，不应再次分配一份重复映像：`PT_DYNAMIC` 指出 dynamic table，`PT_GNU_RELRO` 标出重定位后收紧为只读的范围，`PT_TLS` 描述每线程 TLS 的初始化模板，`PT_GNU_STACK` 只声明栈权限。它们引用的文件或内存范围必须落入经过验证的文件范围或 `PT_LOAD` backing；如果以后支持 `PT_TLS`，每个线程的 TLS 实例由运行时另行分配，而不是直接把模板当成共享的可写 TLS。

解析时至少要检查整数加法溢出、文件范围、`p_filesz <= p_memsz`、`p_align` 合法且 `p_offset`/`p_vaddr` 同余、segment 重叠不产生冲突，以及 dynamic table、重定位表、符号表、字符串表和构造函数数组最终都落在对应映像的已验证范围内。文件给出的偏移、数量和地址都只是待校验输入，不能直接转成指针。

### 2.2 装载器 ≠ 链接器

- **装载（loading）**：校验 ELF → 分配地址 → 复制/映射 `PT_LOAD` → 清 BSS → 定权限 → 找入口。参考：Embox 的 `load_app` 只有这一层（最小体量样本）。
- **链接（linking）**：解析 `DT_NEEDED` → 建立符号作用域 → 重定位 → 构造/析构。参考：musl `dynlink.c`、Relink 的 `Linker`。

Linux 是这两层分离的极端例子：内核 `load_elf_binary()` 只做装载并映射解释器，**所有链接语义都在用户态 ld.so**；而 NuttX/Zephyr/RT-Thread 的模块 binder 则把两者都放进内核。BlueOS 的立场介于两者之间：链接语义放进内核内的独立 crate（`blueos_loader`），但**与内核对象解耦**，可以独立测试。

### 2.3 编译期与运行期是接力关系

静态链接器负责"同一镜像内部能确定"的引用；凡是**装载地址才能决定**或**定义来自另一个 DSO**的引用，都以 dynamic table、dynsym、relocation 的形式留给运行时。运行时主要看 program header 和 `PT_DYNAMIC`，不是 section header。

`DT_*` 条目不是 segment，而是 `PT_DYNAMIC` 这个 segment **内部** dynamic table 的成员。解析顺序是"先按 program header 找到 `PT_DYNAMIC`，再按 `DT_*` 条目找到各表"：

```text
segment 层（program header）
├─ PT_LOAD       哪些文件字节映射到哪些虚址、内存尺寸与 R/W/X
├─ PT_INTERP     进程式系统的解释器 handoff；BlueOS RTOS MVP 不生成、不处理
├─ PT_DYNAMIC    dynamic table 的容器（↓ 下层所有 DT_* 条目都住在这里）
├─ PT_GNU_RELRO  relocation 后应改为只读的 GOT/数据区
└─ PT_TLS        TLS 模板；BlueOS MVP 暂不用

dynamic table 层（PT_DYNAMIC 内部，DT_* 条目）
├─ DT_NEEDED / DT_SONAME      依赖的逻辑名 / 本 DSO 的逻辑名
├─ DT_SYMTAB / DT_STRTAB / hash  动态符号、名称与查找索引
├─ DT_REL* / DT_RELA* / DT_RELR* 隐式 addend / 显式 addend / 压缩 relative 重定位
├─ DT_JMPREL                  PLT 函数调用重定位
└─ DT_INIT_ARRAY / DT_FINI_ARRAY 构造/析构入口
```

### 2.4 重定位的四个量

ELF 文献用四个量描述 relocation：`B` 是装入该映像时的 load bias（运行时地址与 ELF 虚址之间的差值），`S` 是符号解析后的运行时地址，`A` 是 addend，`P` 是被修补存储单元的**运行时地址**。为避免把 `P` 与表项中的原始字段混淆，下面另记 `O = r_offset`。对于 `ET_DYN` 中的一条重定位：

```text
O = relocation.r_offset
P = B_ref + O
```

这里 `B_ref` 是包含该重定位的引用方映像的 load bias。若符号定义来自另一个 DSO，则普通已定义符号的地址为 `S = B_def + st_value`，其中 `B_def` 是定义方映像的 load bias；所以跨 DSO 重定位中，计算 `P` 与计算 `S` 的 load bias 通常不同。

先将 `P` 做映射范围、写入宽度和对齐校验，再按重定位类型计算写入内容：

- `RELATIVE`：`write(P, B_ref + A)`，不查符号；static PIE 通常以这类重定位为主；
- absolute/symbolic：`write(P, S + A)`；
- PC-relative：`write(P, S + A - P)`；
- GOT/PLT 槽：按目标 psABI 执行 `write(P, S + A)` 或 `write(P, S)`，例如 `JUMP_SLOT` 通常直接写入 `S`。

注意 **REL 与 RELA**：RELA 的 `A` 显式存放在 `r_addend`；REL 的 `A` 隐含在 `P` 指向的旧内容中，执行前必须从已验证的目标位置按该 relocation 的宽度读取。ARM32 首发产物以 `DT_REL`/`.rel.dyn`/`.rel.plt` 为主，Phase 1 必须先把 REL addend 的有界读取做正确；AArch64/RISC-V 常用 RELA。链接器只接受当前 artifact profile 明确列出的编码和 relocation 类型，其余全部 fail-closed。

### 2.5 GOT、PLT 与绑定时机

GOT 是位置无关代码间接访问外部地址的表；PLT 是外部函数调用的跳板。lazy binding 让 PLT 首次调用才解析，换取启动更快，代价是可写 GOT、架构汇编和不可预测的首次调用时延。BlueOS 选 **`-z now`（立即绑定）**：启动时一次性写完、RELRO 立即收紧为只读，确定性更强，符合 RTOS 时延可预测要求。

### 2.6 ARM32 Thumb 远跳与 veneer

Thumb 的 `BL`/`B.W` 等直接分支把有限宽度的有符号位移编码在指令中；以 `R_ARM_THM_CALL/R_ARM_THM_JUMP24` 对应的编码为例，可达范围约为 `P` 前后各 16 MiB。限制的是**调用点到目标的距离**，不是“不能跨过某个 16 MiB 地址边界”。当 Flash、SRAM、PSRAM 相距很远时，直接分支才可能超出范围。

标准 `ET_DYN` 外部函数调用应优先由静态链接器生成“调用本映像附近的 PLT → 通过 GOT 读取完整目标地址”的序列。运行时链接器只处理 `R_ARM_JUMP_SLOT`，把解析出的带 Thumb bit 的函数地址写入 GOT；目标 `.so` 是否已常驻、与调用方相距多远，都不要求 loader 改写调用指令。因此 Phase 1 的 `PltGotOnly` profile 继续拒绝 `R_ARM_THM_CALL/R_ARM_THM_JUMP24` 等动态 text relocation，也不因为 Flash 与 PSRAM 相距超过 16 MiB 就默认生成 veneer。

方案仍保留可选的 `LoaderVeneerV1` 能力：只有实际发布工件无法收敛到 PLT/GOT、确实留下可能越界的 Thumb 分支 relocation 时，才在分配前预扫描调用点、在其可达范围内预留 branch island，并在重定位时把越界分支改为跳到 island 内的 veneer；veneer 再以完整 32 位地址尾跳到最终目标。同一目标可能因调用点相距过远而需要多个 veneer，不能假定一个全局 veneer pool 足够。

这条后续路径必须遵守两个边界：调用点若位于不可写 Flash/XIP，运行期不能修补它，只能由静态链接器提前生成 veneer、改走 PLT/GOT，或拒绝工件；运行时生成的 veneer 必须在映像私有且尚未发布时写入，完成 D-cache clean/I-cache invalidate 和指令同步后封为 RX，并随所属映像一起回滚和回收。`__veneer` 一类名字只是链接器实现细节，不进入 BlueOS ABI。

### 2.7 为什么是 `ET_DYN` 而不是 `ET_REL`

`ET_REL`（Theseus crate、Linux `.ko`、Zephyr LLEXT）是**未经最终链接的目标文件**：没有 program header，运行时依据 section header 和**完整静态重定位表**。它的"运行"是宿主现场做一次完整静态链接——分配并复制各 section（text/rodata/data，BSS 清零）→ 遍历全部静态重定位：内部引用按 section runtime 地址计算，未定义符号查**宿主导出表**（Linux ksymtab、Zephyr EDK、Theseus symbol_map、RT-Thread RTM_EXPORT）→ flush cache → 按符号名调用 init/entry。因此 `ET_REL` 路线本质是"宿主导出表 ABI"：模块与内核强耦合，扩展即内核代码，适合"内核向可信扩展暴露宿主符号"。

`ET_DYN` 则依据 program header/`PT_DYNAMIC`/`DT_NEEDED`，是标准共享库 ABI——依赖符号由其他 DSO 提供，不需要宿主导出表。BlueOS 目标是对外发布应用与 `libc.so.1`，选 `ET_DYN` 可以直接复用编译器、链接器与 `llvm-readelf` 等通用工具，内核能力走稳定 syscall，也避免把内核模块 ABI 和应用 ABI 混在一起。

## 3. 调研：参考系统怎么解同一道题

调研（完整调用链和源码路径见 [调研报告](./dynamic-loading-survey.md)）仍按六个问题比较：**输入格式；映射责任；依赖发现；符号来源；重定位/权限/初始化顺序；失败与卸载**。

### 3.1 重点对标总览

| 对标对象 | 输入与执行模型 | 依赖与符号作用域 | 完成链接后的处理 | 对 BlueOS 的价值与边界 |
| --- | --- | --- | --- | --- |
| Linux + musl | 内核映射主程序和 `PT_INTERP`；用户态 ldso 处理 `ET_DYN` | `DT_NEEDED` 闭包；主程序和 DSO 构成确定的全局 scope | TLS、全部 relocation、RELRO、constructor，最后进入 crt | 标准语义和顺序基线；当前 BlueOS 不照搬进程、初始用户栈和用户态解释器 |
| RT-Thread | `dlmodule` 在共享地址空间装入 `ET_REL/ET_DYN`；LWP 另走进程路线 | 不解析 `DT_NEEDED`；模块内部符号加 `RTMSymTab` 内核导出表 | cache 同步后调用 `module_init`；销毁时回收模块创建的内核对象 | 地址模型和资源回收最接近；全局内核符号表不能成为 BlueOS 应用 ABI |
| NuttX | binfmt 统一 exec/insmod，支持 `ET_REL/ET_DYN` 和部分 XIP | 未定义符号先查已安装模块，再查内核 exports；命中时记录 provider 依赖 | 成功后加入 registry；`nopen/dependents` 阻止仍被使用的模块卸载 | registry、显式导出和 task-group 寿命值得复用；它不是 `DT_NEEDED` 自动装载模型 |
| OpenHarmony LiteOS-M | 共享地址空间内核态插件 loader；每次装入一个无依赖的 `ET_DYN` | 拒绝 `DT_NEEDED`；重定位先查当前插件自身定义，未定义符号只查内核导出表；多个已加载插件不能互相提供符号 | 拒绝 TLS/TEXTREL，执行 init/fini；宿主通过 `LOS_FindSym` 取得插件入口，以引用计数控制卸载 | 最接近资源受限实现下限和 fail-closed 基线；不是动态应用运行时，无法表达 `app → DSO → libc.so.1` |
| Relink | `no_std` reader/memory backend 上构造带生命周期的动态模块 | BFS 闭合 `DT_NEEDED`，独立冻结 scope，再生成生命周期顺序 | staged relocation、RELRO、原子 publish、lease；init 在提交后执行 | 组件边界和事务模型最匹配；裁剪后借鉴并用于宿主对拍，不作为产品运行时依赖 |

### 3.2 Linux/musl：标准 `ET_DYN` 语义基线

Linux 的 [`load_elf_binary()`](../../../linux/fs/binfmt_elf.c) 只负责校验 ELF、建立新地址空间、映射主程序与解释器、构造 `argc/argv/envp/auxv`，然后把 PC 交给 `PT_INTERP` 指定的 ldso；内核不读取 `DT_NEEDED`，也不解析 dynsym 或普通 DSO relocation。这条边界说明 **ELF segment loader 与 dynamic linker 是两个职责**，即使 BlueOS 当前把两者都静态链接进内核，也不应把它们混成一层。

musl 的 [`dynlink.c`](../../../musl/ldso/dynlink.c) 给出完整链接顺序：ldso 先只做自身 relative relocation，使最小运行环境可用；随后建立 TLS/DTV，按链尾追加方式 BFS 发现主程序的 `DT_NEEDED` 闭包，冻结符号查找 scope，完成依赖和主程序的 relocation，最后收紧 RELRO 并进入 crt/constructor。运行期 `dlopen` 还展示了 loader lock、失败快照和“释放锁后才运行 constructor”的边界。BlueOS 直接复用这些语义，但不复制 `PT_INTERP` handoff、Linux 用户栈、lazy binding 和 native ELF TLS；首版由内核内 `DynamicLinker` 处理同样的 DSO 图，并以 emutls 降低复杂度。

### 3.3 RT-Thread：当前执行模型最接近的 RTOS

RT-Thread [`dlmodule_load()`](../../../rt-thread/components/libc/posix/libdl/dlmodule.c) 扫描 `PT_LOAD` 求映像包络，分配并整块清零内存，再复制 segment；内部符号按 load base 换算，未定义符号线性查询 `RTMSymTab`，架构后端完成 relocation，随后同步 D/I-cache 并调用自定义 `module_init`。`dlmodule_destroy()` 不只释放 ELF 内存，还清理模块创建的线程、锁、信号量、队列和定时器。这与 BlueOS L0 阶段“可信代码、共享特权地址空间、以 ThreadGroup 为资源寿命单位”高度相似。

它的局限同样明确：`dlmodule` 不读取 `DT_NEEDED`，依赖宿主导出表，模块 ABI 与内核强耦合；符号域也不是标准 DSO scope。RT-Thread 同仓库的 LWP 路线改为映射 `PT_INTERP`、建立 auxv、由用户态解释器链接，恰好给出未来从 BlueOS ThreadGroup 后端演进到进程后端的分界。因此 BlueOS 借鉴其 flat-memory、cache 和成组清理机制，但应用只通过稳定 syscall 使用内核，不引入 `RTM_EXPORT` 式完整内核符号表。

### 3.4 NuttX：依赖所有权与安全卸载参考

NuttX 将 exec、insmod 和 rmmod 建立在同一套 binfmt/libelf/registry 上。[`libelf_load()`](../../../nuttx/libs/libc/elf/elf_load.c) 根据对象类型分配 text/data 或保持 `ET_DYN` 段间关系；[`libelf_bind()`](../../../nuttx/libs/libc/elf/elf_bind.c) 解析未定义符号时先查已安装模块，再查内核 exports，命中模块后立即记录对 provider 的依赖。新模块只有完成重定位、cache 同步和 init 后才进入 registry；卸载前检查 open count 和 dependents，再执行 fini、撤销依赖并释放映像。

最值得 BlueOS 采用的是“发布后才可解析”“依赖边也是所有权边”和“最后执行流退出后才回收”的控制面，而不是它的符号搜索方式。NuttX 的依赖产生于**符号绑定命中已经安装的模块**；BlueOS 的依赖则来自 `DT_NEEDED`，需要主动搜索、装入并闭合图。两者不能混为同一种依赖发现。NuttX 的只读 XIP 也只在文件映射稳定、目标区域无加载期写入且 cache 属性正确时成立，BlueOS 首版仍以 RAM 映射为正确性基线。

### 3.5 LiteOS-M：无依赖插件 linker 与能力上限

LiteOS-M [`LOS_SoLoad()`](../../../openharmony/kernel/liteos_m/components/dynlink/los_dynlink.c) 是最接近 BlueOS 当前部署位置的实现，但它装入的是供现有内核或静态应用使用的**插件 `.so`**，不是带共享库依赖的动态应用。它在内核态、共享地址空间内限制文件和 program-header 数量，预扫描 `PT_LOAD` 后分配单块内存，复制段并清零 BSS，解析 REL/RELA/JMPREL。重定位时先在当前插件的 dynsym 中解析插件自身定义；未定义符号只查 `SYM_EXPORT` 建立的内核导出表。成功后执行 init、加入全局 DSO 链表并返回 handle，宿主再通过 `LOS_FindSym(handle, name)` 取得插件导出函数并调用；任一步失败都逆序释放临时状态和映像。

系统可以同时保存多个独立插件，相同路径重复加载会增加引用计数，但一次 `LOS_SoLoad()` 只处理当前文件：发现 `DT_NEEDED` 就返回不支持，符号查找也不遍历其他已加载插件。因此它既不能形成插件间依赖图，也不能表达 `app → private DSO → libc.so.1`。此外，它拒绝 `PT_TLS`/`DT_TEXTREL`，单块 RWX 映像无法形成 RELRO/W^X，符号域也不能隔离 system/application scope。BlueOS 可以借鉴其资源上限、单映像装载、引用计数和 fail-closed 基线，但必须增加动态应用入口、依赖闭包、文件身份、双层符号域、逐区域权限和 ThreadGroup 生命周期，不能只在 `LOS_SoLoad()` 外包一层应用管理就宣称支持标准共享库。

### 3.6 Relink：实现架构与事务语义参考

Relink 的价值不在于运行于某个相似 RTOS，而在于其 [`LinkerRun`](../../../Relink/src/linker/run.rs) 已把通用链接过程拆成 reader/memory/resolver、`ResolveSession`、依赖图、scope、relocator、publish 和 lease。它用 BFS 关闭 `DT_NEEDED`，另行建立符号 scope 和依赖优先的生命周期顺序；映射和重定位期间对象保持私有，完整成功后才原子 publish，随后才执行 init。失败时 session 可以撤销映射与引用，但不会虚构 constructor 外部副作用也能自动回滚。

这与 `blueos_loader` 的目标组件边界最接近，因此 BlueOS 复用其**设计语义和测试价值**：固定 revision，在宿主侧用相同 fixture 对拍依赖图、符号绑定、重定位结果与生命周期顺序。产品实现只保留 MVP 所需的 `ET_DYN`、NOW、emutls 和目标架构白名单，不带入 Relink 的 ET_REL、lazy binding、IFUNC、native TLS 和通用平台后端。

### 3.7 次级参考及取舍

Zephyr LLEXT 主要用于补足 region backing、MPU/PMP 对齐、重定位后 cache/权限收紧和 init/fini 指针落入可执行区校验；它是 section-centric extension linker，不作为 `DT_NEEDED` 标准语义对标。Tock 的价值是签名凭证、AppId 与 MPU process，不是 ELF linker。Theseus 展示 namespace 和细粒度依赖，也暴露 Rust ABI/编译哈希作为发布接口的风险。Asterinas/StarryOS 则证明未来进程模式仍应采用“内核 exec stage0 + 用户态 ldso”，而不是继续扩大内核动态链接器。

以上对标直接塑造了 BlueOS 的五项设计：

1. **装载与链接分开**——`ImageLoader` 负责单映像的 parse/layout/map，`DynamicLinker` 负责依赖闭包、scope、跨映像 relocation 和生命周期计划；
2. **发现序、查找序、初始化序分别建模**——以 `app → {libfoo, libbar} → libc` 为例，BFS 发现序是 `app → libfoo → libbar → libc`，查找序由 main/private/system scope 与 visibility 共同决定，初始化则要求依赖先于使用者；DSO 间退出次序与初始化依赖次序相反，单个 DSO 内再执行 `FINI_ARRAY` 逆序和 `DT_FINI`；
3. **准备、发布、执行分开**——映射和重定位期间对象私有，publish 是原子可见性边界，constructor 是不能承诺完整回滚的副作用边界；
4. **引用计数不等于可卸载性**——计数归零只说明显式所有者释放，不代表没有线程 PC、回调、GOT、函数指针或 TLS destructor 指向映像；
5. **通用语义与平台机制分开**——链接核心决定读取哪些段、如何计算地址和何时 seal；内核 traits 只提供 VFS、目标内存、权限与 cache 服务。

## 4. 核心模型：统一十二阶段流水线

把上一节的调研结果归一化后，所有系统共享同一条骨架。某个系统可以裁剪阶段，但**不能颠倒保留下来的因果顺序**。这就是本方案的总线（运行时阶段 S0–S11）：

![统一十二阶段动态加载与动态链接流水线](./assets/dynamic-loading/blueos-12-stage-pipeline.svg)

> 图中阶段 0–8 构成未发布、可回滚的链接事务；9 是对其他组件可见的原子提交点；10 开始执行构造函数并进入不可自动回滚的副作用区。

三条边界注释：

```text
S0–S8   可回滚事务：任一步失败，随对象 Drop 逆序释放 allocation/permit/lease，系统回到零状态
S9      唯一可见性提交点：发布前对象私有，发布后读者只能看到旧快照或完整新快照
S10 起  副作用边界：ctor 产生的外部副作用（I/O、线程等）不可自动回滚
```

三条铁律：

1. **分配和写入之前先证明一切有界**（S0/S1 的 checked arithmetic、配额）；
2. **完整成功才发布**（S9 之前对象私有，失败随 Drop 逆序回滚）；
3. **初始化晚于重定位、回收晚于最后引用失效**（S10/S11）。

### 4.1 各阶段详解

| 阶段 | 做什么 | 不可破坏的不变量 | 参考系统的同一阶段 |
| --- | --- | --- | --- |
| S0 准入 | 校验来源、ELF 身份（magic/class/endian/machine/ABI note）、资源上限 | 分配和写内存之前完成全部准入 | Tock 包凭证检查；Linux/NuttX/Zephyr 的 ELF 校验 |
| S1 预扫描/规划 | phdr/dynamic 概要 → `ImageLayout`（span/对齐/load bias/段布局） | checked arithmetic；所有 file/virtual 范围可证明有界；拒绝 W+X、entry 不在 X 段 | Linux 先算 PIE 跨度再映射；Relink `elf/layout.rs` |
| S2 reserve | 按布局向 `ImageMemory` 预留存储，得到 load bias | 统一 load bias；映像尚未被其他执行流发现 | musl `map_library` 先 reserve 再 MAP_FIXED |
| S3 copy/zero | 复制 `p_filesz`、**显式清零** BSS 与段间洞 | `p_filesz ≤ p_memsz`；BSS 必须在重定位前清零 | RT-Thread dlmodule 整块清零；LiteOS-M 逐段清零 |
| S4 运行时元数据 | dynamic/symbol/relocation/lifecycle 解码为**经过 bias 与范围校验**的运行时视图 | 不支持的 TLS/符号类型/扩展在执行前明确拒绝 | Relink `RawDynamic`；musl `__dls2` 解码 dynamic |
| S5 依赖发现 | 从主程序 `DT_NEEDED` 出发 BFS，按 SONAME/文件身份去重，装入或获取 system DSO | 有界依赖图；同名 ≠ 同一文件；不允许私有 DSO 冒充系统 DSO | musl 的链尾追加式 BFS；Relink `resolve_pending` |
| S6 冻结作用域 | 由闭合图派生符号查找顺序：应用 scope = main → 私有 BFS → system；system scope 仅 system（libc 自身的重定位只在系统域解析） | relocation 前冻结；weak/hidden/protected 语义唯一；系统 DSO 的解析结果与应用无关、与装载顺序无关（防劫持 + 单实例多应用一致性） | Relink `build_scope`；musl 全局符号链 |
| S7 重定位 | relative 批量先行，再数据/GOT，最后 PLT（NOW）；`RelocationPolicy` 同时校验写入位置 `P`、符号来源 `S` 和控制流目标 | `P` 必须属于当前 owner 的允许写入区且宽度/对齐/算术有界；`S` 只能来自冻结 scope；`JUMP_SLOT` 等控制流目标必须落在允许映像的 X region（undefined weak=0 只能按 profile 显式允许）；未知类型 fail-closed | Relink `Relocator` 的固定 pass 顺序 |
| S8 seal | 所有写完成后统一形成 text RX、rodata/RELRO R、data/BSS RW+NX 的 `SealPlan`，再同步 cache | 拒绝 W+X/TEXTREL/可执行栈；不得存在应用可见的 writable executable alias；无硬件保护时显式记录 `LogicalOnly`，cache 未实现则失败 | Zephyr/RT-Thread 先重定位后 flush cache；Linux 后收紧 RELRO |
| S9 publish | 会话资源原子提交给 ThreadGroup / SystemDsoRegistry / link map | 只发布完整一致的图；读者只能见旧快照或全新快照 | Relink `publish`+`acquire`；Linux/Asterinas 先建新映像再切换 |
| S10 初始化 | 依赖先于使用者顺序执行构造函数；完成后公共状态进入 Running | 在 loader/registry 锁外执行（ctor 是用户代码，持锁执行会死锁/阻塞系统，musl 即为此先放锁再调 ctor）；每个装载生命周期恰好一次（共享 libc 的 ctor 全系统只跑一次，并发等待者不重复触发） | musl `__dls3` 之后；Zephyr `llext_bringup` |
| S11 运行/退出/回收 | 进入 `main`；退出时阻止新线程、等待全部线程、逆序 fini/atexit，最后整组回收 | fini 与 init 逆序；所有执行流与引用失效后才释放映像 | NuttX task-group 最后线程离组回收；RT-Thread dlmodule destroy |

> 阶段号是**一笔加载事务的状态**，不是代码目录也不是排期。实现上，S0–S3 属于 `ImageLoader`（单映像），S4–S9 属于 `DynamicLinker`（多映像链接），S10–S11 属于内核 `ApplicationManager`/`librs`。

## 5. 架构分层：模块职责与交互

![BlueOS RTOS 动态加载分层与稳定边界](./assets/dynamic-loading/blueos-layered-architecture.svg)

> 应用启动统一走 `ApplicationLaunch` syscall（shell 经 libc.so.1 的 `spawn` API 发出）；boot 自举由内核内部直呼 `launch()` 装载 shell 本身——这是唯一的函数调用例外。`blueos_loader` 是独立于内核的 crate，不依赖内核对象，链接所需的 VFS/内存/cache 能力经 trait 由内核 adapters 提供。

### 5.1 各模块职责（谁是谁）

| 模块 | 职责 | 命名/设计参考 |
| --- | --- | --- |
| `ApplicationManager` | 系统级控制面：统一 `ApplicationHandle`、`ApplicationState(Loading/Running/Stopping/Terminated/Failed)`、准入、后端分派、退出事件与延迟回收。**不参与线程调度** | AOSP `ActivityManagerService`、CHRE `EventLoopManager` |
| `ThreadGroup` | 一次应用实例的资源寿命单位：主线程+子线程、私有映像、`LinkContext`、system DSO lease、启动信息、配额、退出码。**不是 `Thread` 的包装** | NuttX `task_group_s`、RT-Thread `rt_dlmodule` |
| `ApplicationLoader` | 启动期装载/链接适配器：把 VFS/内存/cache/registry 能力接给 `DynamicLinker`，负责 publish 与回滚 | Relink 的 platform 适配层 |
| `SystemDsoRegistry` | 工件目录（已验证元数据）与运行实例表（mapped/relocated 状态）分离；并发首装合并为一次事务；lease 保活；归零后静默检查才可卸载 | Tock 凭证与运行实例分离；NuttX/Zephyr module 表 |
| `ImageLoader` | 单映像装载：S0–S4。`StagedImage<'a, M, S>` 把 rollback authority 与 `Mapped→Runtime→Relocated→Sealed` payload 绑定，转换方法消费旧对象 | Zephyr LLEXT loader；Relink loader |
| `DynamicLinker` | 多映像链接：S5–S9。依赖图/作用域/符号/重定位/生命周期计划，`LinkSession` 事务（rollback log） | musl `dynlink.c`；Relink `Linker`+`session` |
| traits（`ElfReader`/`ImageMemory`/`CodeCache`/`ArchRelocator`） | 链接核心与内核的唯一触点：核心只持 `TargetAddr`+`AllocationId`，不解引用裸指针 | `rust-elfloader` 的 trait 分离；Relink input/memory/os |
| `libc.so.1` / `blueos_scrt1` | 动态应用运行时：`_start(ApplicationStartInfo*)` → `__librs_start_main` 初始化、跑 init plan、进 `main`、协调退出 | musl `Scrt1.o` 语义（定制 ABI） |

> **为什么需要公共应用状态**：线程状态（ready/running/suspended/retired）只回答"何时拿到 CPU"；应用实例还拥有子线程集合、映像、lease、TLS/atexit、配额和退出码——这些资源何时保活、何时回收是另一个问题。`ApplicationState` 只保留粗粒度 `Loading → Running → Stopping → Terminated/Failed`（Loading：装载/链接/构造中；Running：InitComplete 后；Stopping：begin_exit 后阻止新线程、等待 fini；Terminated/Failed：终态，Failed 保留失败原因），更细的 Parsed/Mapped/Relocated/Published 由 loader 的 typestate 表达，不复制进公共状态。应用 Running 时线程可能全部 suspended，应用 Stopping 时可能有线程仍在执行 fini——两个状态体系正交。

### 5.2 三条设计原理

1. **链接核心与内核解耦**：`blueos_loader` 是独立 rlib，不依赖 VFS/内核分配器/`ApplicationManager`/librs（Phase 0.5 门禁）。全部能力经 trait 回调获得，因此可以在宿主上单测、未来可复用给 `process/` 后端甚至用户态 ld.so（只换 adapter 不换语义）。
2. **typestate + 事务**：映像阶段（Mapped/Relocated/Sealed）与会话阶段（Building/Scoped/Relocated/Sealed session）都用类型表达合法操作；S0–S8 的任何失败让对象进入 Drop，按逆序释放 allocation/permit/lease。Relink 证明这条边界可行。
3. **副作用边界不可回滚**：构造函数会产生 I/O、线程等外部副作用，因此 ctor 不在 `LinkSession` 内执行，publish 与 initialize 之间是明确的"资源提交点 vs 副作用点"分界（参考 Relink `commit` 后才 `initialize`）。

## 6. 一次完整启动走查：`run /apps/hello/app.elf arg1`

把十二阶段和模块分工合起来看一次真实启动：

| 阶段 | 发生什么（谁负责） | 失败处理 |
| --- | --- | --- |
| S0 | shell 经 librs `spawn` API 发 `ApplicationLaunch` syscall；内核校验信任证据（MVP：内置/构建期 allowlist）、manifest 的 `ExecutionModel::ThreadGroup`、ABI note、配额，argv/envp 先 copy-in | 直接拒绝，不分配任何资源 |
| S1–S2 | `ApplicationLoader` 经 `VfsElfReader` 打开工件；`ImageLoader` 解析出 `ParsedImage`+`ImageLayout`，`FlatImageMemory` 按 span+最大对齐预留连续 RAM，得到 load bias | 丢弃解析结果；reserve 失败释放已分配 |
| S3 | 整块清零后逐段复制 `app.elf` 的 `PT_LOAD`，BSS 确定性为零 | 释放 allocation |
| S4 | 解码 dynamic 表：`DT_NEEDED: libc.so.1` 进入待解析队列；relocation 表转为运行时记录 | 未知类型 fail-closed |
| S5 | BFS 发现依赖：`ApplicationLoader::resolve_needed` 查 `SystemDsoRegistry`——首次请求触发 `libc.so.1` 从固定可信路径装入（并发请求等待同一 Loading 事务），已 Ready 则直接拿 `SystemDsoLease` | 缺依赖 → MissingDependency；并发失败唤醒 waiter 可重试 |
| S6 | 冻结两个作用域：应用 scope = main → 私有 DSO → system；system scope 仅 system DSO（应用不能 interpose 系统符号） | 重复 strong export 报错 |
| S7 | 全部 NOW 重定位：relative 批量 → 数据/GOT → JUMP_SLOT；`ArchRelocator` 按 psABI 写值 | 未知/溢出/越界 → 整 session 回滚 |
| S8 | RELRO 收紧为只读、逻辑权限收口；ARM32 首发板执行 DSB/ISB，存在 cache 的板再执行对应的 D-cache clean 与 I-cache invalidate | cache 能力未声明或所需维护未实现 → 失败 |
| S9 | `LinkProduct` 原子发布：映像/lease/init/fini 计划装入仍处 Loading 的 `ThreadGroup`；link map 原子更新 | publish 失败由 `PublicationGuard` 完整回滚 |
| S10 | 主线程以 `blueos_scrt1::_start` 为入口、`ApplicationStartInfo*` 为参数创建；`__librs_start_main` 校验 ABI → 建 `LibcApplicationContext` → 执行 init plan → `ApplicationInitComplete` → 公共状态 Running | ctor 失败 → 停止后续 ctor、按"已执行前缀的逆序"执行对应 fini、终止组内线程并释放映像/lease/link map，整组走 Draining→Reaped；已发生的外部副作用不承诺撤销 |
| S11 | `main(argc,argv,envp)` 运行；退出时 `ApplicationBeginExit` 阻止新线程、等待组内线程、逆序 fini/atexit、`ApplicationFinishExit`；reaper 释放私有映像与 lease；最后一个 lease 归零后 registry 静默检查，通过才卸载 `libc.so.1`，否则缓存复用 | 最后线程退出前映像绝不释放 |

## 7. 关键设计决策及理由

| 决策 | 理由 | 参考系统 |
| --- | --- | --- |
| 标准 `ET_DYN`，不带 `PT_INTERP` | `PT_INTERP` 是进程式 ldso handoff 契约，不是动态链接的必要条件；内核直接链接省掉解释器自举 | Linux/musl（对照：需要它）、StarryOS static-PIE 特例 |
| 扩充现有 `blueos_loader` 而非新 crate/Relink 部署 | 保留已验证代码与测试；Relink 4.2 万行、含 ET_REL/lazy/TLS 等不需要的能力，只做 host 对照 oracle | Relink（借鉴语义）、RT-Thread 双轨演进 |
| NOW + RELRO + 逻辑 `dlclose`/整组回收 | 启动和运行期装载都立即绑定、权限尽早收紧；Phase 3 可关闭 handle，但避免中途释放裸指针仍可能引用的映像 | LiteOS-M（单 DSO 最小实现）、musl（`dlclose` no-op） |
| 共享一个 `libc.so.1`（lease 保活） | 无 MMU 下省 RAM、单一运行时实现；代价是 libc 全局状态系统级共享——按 system/thread/`LibcApplicationContext` 分层审计 | NuttX 模块 registry、Zephyr LLEXT use-count |
| trait 化 `ElfReader`/`ImageMemory`/`ArchRelocator` | 核心可在宿主单测、未来进程后端只换 adapter | `rust-elfloader`、Relink input/memory/os |
| typestate + 事务回滚 | 类型层面杜绝半链接对象；失败自动逆序回滚 | Relink `LinkSession`、Asterinas 先建后切 |
| emutls | 六个 target 已统一生成 `__emutls_get_address` 调用，避免 PT_TLS/DTV/TLSDESC 全家桶 | LLVM emutls、Tock per-task TLS |
| C ABI 为唯一发布 ABI | Rust rlib/dylib 符号与编译配置耦合，不是稳定系统 ABI | Theseus out-of-tree ABI 问题 |
| Phase 0 先修正确性 | BSS/load bias/静默 relocation 在单映像下已致错，多 DSO 会放大 | 调研 §4.2 逐条源码佐证 |

### 7.1 当前阶段必须交付的安全控制

当前 Phase 0–3 面向的是**受信任的特权应用**。安全目标不是约束应用开始执行后的每一次访存和跳转，而是保证“不可信文件字节不能借 loader 越界写系统、错误依赖不能劫持 system DSO、半完成映像不能被执行、退出时不会释放仍被引用的代码”。这些控制是当前方案的必交付项，不依赖 MPU/PMP：

| 控制面 | 当前策略 | 必须满足的门禁 |
| --- | --- | --- |
| 工件来源 | MVP 仅接受内置/构建期 allowlist；产品阶段校验签名、hash、manifest、ABI note 和 artifact generation | 准入失败发生在 allocation 之前；固定可信 system path，不能用环境变量、当前目录或任意 RPATH 改写解析 |
| ELF 与布局 | checked arithmetic、文件/虚址范围和总量配额；校验 `p_filesz ≤ p_memsz`、对齐、重叠、entry；拒绝 W+X、TEXTREL、可执行栈及当前 profile 不支持的 ELF 特性 | 畸形输入只产生结构化错误；任何失败不留下 allocation 或可见 link-map 条目 |
| 依赖与符号域 | SONAME 与文件身份双重去重；应用 scope 与 system scope 分离；system DSO 只从冻结导出集解析 | 私有 DSO 不能冒充 system DSO；应用符号不能 interpose libc 自身 relocation；不向应用发布完整内核符号表 |
| 重定位写入 | `RelocationPolicy` 对每条记录验证 owner、`P` 的 allocation/range/width/alignment、addend 和目标字宽溢出；类型采用架构白名单 | relocation 只能通过 `ImageMemory` 的已验证位置写入；未知类型、越界、未对齐、缺失 strong symbol 全部 fail-closed |
| 重定位控制流 | `JUMP_SLOT`、入口和 init/fini 函数地址必须来自允许 scope，并落在对应 owner 的 X region；undefined weak=0 是否允许由 artifact profile 明确决定；符号类型和架构函数地址规则由 `ArchRelocator` 校验 | data/BSS/gap/内核普通地址不能成为 loader 写入的控制流目标；该检查只覆盖 relocation 和生命周期元数据，不宣称完整 CFI |
| 映像权限与 cache | 映像在 S9 前不可见；全部写入结束后按 `SealPlan` 收为 RX/R/RW+NX，RELRO 去 W，随后完成 D/I-cache 同步再发布 | `ProtectionRecord` 必须区分 `LogicalOnly` 与 `HardwareEnforced`，不能把逻辑权限谎报为硬件隔离；NOW 绑定，不保留运行期可写 lazy PLT |
| 发布与回收 | typestate、rollback log、原子 publish、ThreadGroup 所有权、system DSO lease、quiescence 后卸载 | 读者只见旧快照或完整新快照；最后执行流、回调、TLS destructor 和 lease 失效前不得释放映像 |

重定位目标校验不是控制流完整性：同一 ELF 内已被静态链接器解析的直接分支、运行期函数指针、返回地址和应用自行计算的 `jalr` 目标都不出现在 dynamic relocation 表中。因此当前控制能防止 loader 替应用制造非法跳转，不能阻止同特权 native 代码运行后自行访问任意系统地址。

### 7.2 后续可选的应用隔离演进

MPU/PMP/MMU 不作为当前 Phase 0–3 的完成条件。以后若产品需要运行互不信任的 native 应用，再单独立项实现 L2 protection domain；最小闭环必须同时包括：

1. 应用进入 unprivileged/U/EL0，不能修改中断、页表、PMP/MPU 或异常入口；
2. 每个 ThreadGroup/Process 拥有独立内存保护配置，scheduler 在跨域切换时激活；
3. syscall 成为唯一特权入口，所有用户指针执行 range check 与 copy-in/out，内核对象只通过带代次的 handle 暴露；
4. 用户态 fault 归属到当前应用，只终止并回收对应资源组；
5. CPU、RAM、线程、句柄和设备访问有强制配额；共享 libc 只共享 RX/RO，所有可写状态按 protection domain 隔离。

`ProtectionDomain`、`ThreadGroupMembership` 和 `ImageMemory` 继续保留前向兼容接口，但当前后端允许明确返回 `LogicalOnly`。只有上述闭环通过越权访问、坏指针、特权指令、无限循环和 fault-containment 门禁后，才允许返回 `IsolatedDomain` 或对外宣称应用隔离。

### 7.3 Phase 3 的 `dlopen/dlclose` 边界

启动动态链接发生在 `main` 之前，整个依赖闭包尚未运行；`dlopen` 则发生在应用已经多线程运行之后，所以它是同一链接核心上的**增量事务**，不是给启动函数简单包一层 C API：

```text
librs::dlopen(path, RTLD_NOW | RTLD_LOCAL)
  → 内核校验当前 ThreadGroup、路径、签名/ABI 和配额
  → 同一工件并发首开合并为一个 RuntimeLoadSession
  → 发现新增依赖 → 冻结 group scope → NOW relocation
  → seal/同步 cache → 原子发布为 Initializing
  → 释放 loader lock，由 librs 执行 constructor
  → 标记 Ready，dlopen 才向调用方返回 handle
```

Phase 3 的运行期装载基线支持 `RTLD_NOW | RTLD_LOCAL` 和 `dlsym(handle, name)`；名称只能解析到当前签名包 manifest 声明的私有 DSO 或系统只读 catalog。重复打开同一 Ready DSO 只增加计数，不重复装载或运行 constructor。`RTLD_LOCAL` 表示该 group 的导出不会自动进入其他后续装载的全局查找域，但 group 自己仍可从启动时已经建立的应用/libc scope 解析符号。`dlerror` 使用调用线程自己的 libc 状态，避免不同线程互相覆盖错误。

这里必须区分两种“close”：

| 语义 | Phase 3 是否交付 | 原因 |
| --- | --- | --- |
| 逻辑 `dlclose` | 是 | 减少显式 open count；最后一个 close 使带 generation 的 handle 失效，但代码、数据、emutls 描述符和已返回地址仍保活到 ThreadGroup 退出 |
| 立即 fini + unmap | 否，需单独的 `UnloadSafe` 协议 | `dlsym` 返回的是裸地址，loader 无法证明其他线程、GOT、回调、timer、atexit、TLS destructor 或 unwind 表已经不再引用它 |

因此 Phase 3 的 `dlclose` 不是“没有实现”，而是采用与当前共享特权地址空间相匹配的保守实现：API 和引用计数语义存在，陈旧 handle 会被拒绝，内存回收推迟到能够证明所有组内执行流都结束的 ThreadGroup 回收点。以后若业务要求在运行中立即归还 RAM，就必须限制插件 ABI、跟踪调用与回调 lease、建立线程安全点并完成 quiescence 证明；单纯在引用计数归零时 `unmap` 会把一个普通 stale pointer 升级成系统级故障。
