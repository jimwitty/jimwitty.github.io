---
layout: post
title: "WRK 第 3 批：MmInitSystem 三阶段——内存管理器上线"
categories: WRK
---

> **取材说明**：底本为 WRK 源码树 `~/workspace/WRK/WRK/WRK-V1.2`（WS03 SP1 / NT 5.2 代码线）中 `BASE/NTOS` 的实现，主文件 `MM/MMINIT.C`（`MmInitSystem` 定义在 `:337`，Phase 分支在 `:433` / `:2211` / `:2391`）、`MM/I386/INIT386.C`（`MiInitMachineDependent` 定义在 `:763`）、`MM/SECTSUP.C`（`MiSectionInitialization` 定义在 `:958`）、`MM/SYSLOAD.C`（`MiWriteProtectSystemImage` 定义在 `:6689`）。HAL 本体不在 WRK 包内，凡涉 HAL 处均按调用点反读并注明；tpn/pdfs 是第三方文档收集仓库、非微软官方，本批不依赖它。
>
> **本批写作规范（2026-10-25 用户定稿，交付门版）**：
> 1. 术语表前置：凡本批出现的专有名词、缩写，开篇先给定义，并说明它是做什么用的、为什么要有；第 1、2 批已定义的词本批沿用，不再重复（沿用名单见术语表首段）；
> 2. 每个机制、每次判定，正文里讲清为什么非如此不可——包括历史因素与兼容考虑；每个机制讲清"没有会怎样"；
> 3. 凡鸡生蛋结，正文当场讲清来龙去脉三段：结为什么会形成、靠什么打破循环、打破后是什么局面；只给解开机制不算讲透；
> 4. 凡涉 32 位与 64 位、单核与多核分叉处，必须讲清为什么 64 位 / 多核可以这么干、而 32 位 / 单核只能那么干；
> 5. 正文零错漏标准：每条机制结论以本轮复核过的源码行号为准；树里查不到的，列入文末"未钉死清单"，不编造；推断必须标注为推断。
>
> **本批一句话地图**：Phase 0 把地址空间划界、按加载器内存描述符建 PFN 数据库、立起非分页池与分页池、备好系统 PTE 与系统缓存——MM 从此能分配、能换页、能记账；Phase 1 建节对象与物理内存节、起 MM 系统线程、给系统镜像加写保护——MM 从此能映射、能回收、能自保；Phase 2 只干一件事：允许执行体分页（并纠正第 2 批按旧头注释写错的一句话）。

---

## 本批地图（在点火链中的位置）

```
第 2 批：ExpInitializeExecutive Phase 0（INITOS.C:582 调 MmInitSystem(0)）
  │
  ▼
MmInitSystem(0) —— Phase 0（MMINIT.C:433 起）
  │ ① 地址空间划界（用户/系统分界、/3GB 偏置、启动映像尺寸）
  │ ② 临时事件（MiTempEvent，替 Phase 1 才能建的正式事件占位）
  │ ③ 尺寸测算（池、系统 PTE、缓存的权衡；32 位争 VA、64 位争物理页）
  │ ④ 机器相关初始化 MiInitMachineDependent（PFN 建库、非分页池布局、系统 PTE）
  │ ⑤ 页表自举（系统缓存工作集的页表页现取现建）
  │ ⑥ 分页池建成 MiBuildPagedPool（分页池从此可分配）
  │ ⑦ 预算与缓存结构（可用页预算、系统缓存结构）
  ▼
第 2 批 Phase 1（INITOS.C:1286 调 MmInitSystem(1)）
  │
  ▼
MmInitSystem(1) —— Phase 1（MMINIT.C:2211 起）
  │ ⑧ 大页转换（MiInitMachineDependent 的 Phase 1 分支，需全部处理器支持）
  │ ⑨ BBT 映射（MiMapBBTMemory，烧掉的内存映射进系统空间）
  │ ⑩ 节对象类型 + 段引用线程（MiSectionInitialization）
  │ ⑪ 物理内存节（\Device\PhysicalMemory，整段物理空间）
  │ ⑫ 共享用户数据双映射（内核与用户共读一页）
  │ ⑬ 会话支持（Hydra 会话空间）
  │ ⑭ MM 系统线程（已改页写出、平衡集管理、零页工人）
  │ ⑮ 镜像写保护 + NATIVE 标记（Phase 1 尾，遍历 PsLoadedModuleList）
  │ ⑯ 完全初始化旗（MiFullyInitialized = 1，修剪开闸）
  │ ⑰ MmFreeLoaderBlock（MM 的 Phase 1 尾活，不在 MmInitSystem 内）
  ▼
第 2 批 Phase 1 尾（INITOS.C:1547 调 MmInitSystem(2)）
  │
  ▼
MmInitSystem(2) —— Phase 2（MMINIT.C:2391 起）
  │ ⑱ 只干一件事：MiEnablePagingTheExecutive（允许执行体分页）
  ▼
第 4 批：OB 点火（对象管理器展开，节对象类型在此批已立好是前提）
```

关键分界：本批讲的是"内存管理器上线"——PFN 数据库、池、页表、节对象、MM 线程；第 4 批讲的是"对象管理器点火"——对象类型、句柄表、名字空间怎么展开。两批的交接点就是 Phase 1 里 `MiSectionInitialization` 调 `ObCreateObjectType`：MM 的节对象类型必须先在 OB 里登记，OB 点火后才能用节对象。

---

## 第一部分：术语表（先读这部分，正文不再重复定义）

> 第 0、1、2 批术语表已定义的词（LoaderBlock、内存描述符、池、节对象、工作集、自旋锁、IRQL、DPC、APC、HAL、KLDR_DATA_TABLE_ENTRY、NUMA、lookaside、句柄、对象类型、设备对象、令牌、安全描述符、换页文件、prologue / epilogue、REG_SZ、chkdsk 等）本批直接沿用，不再重复；下面只定义本批新出现的词（PFN 与链表是本批核心，虽在第 2 批 QA 第 11 条提过，本批仍给完整定义）。
本表按引用级排序：基础概念在前，派生概念在后——读到某条时，它引用的词前面都已定义；VA 与页表互为前提，VA 条里给了页表的行内解释并指到 PTE 条目；MmVirtualBias 条的 PFN 给了行内解释并指到二、PFN 条。

### 一、地址、页表与硬件基础

- **CPU（中央处理器，Central Processing Unit）**：执行指令的硬件；本批的"CPU 特性"指 CPUID（取 CPU 特征信息的指令）报告的特性位（如 KF_LARGE_PAGE），"某核不支持"指某个处理器核没有该特性。

- **VA（虚拟地址，Virtual Address）**：程序看到的地址，不是物理地址——访问 VA 时，硬件按翻译表把它换成物理地址再访问（翻译表即页表，见 PTE 条目）。为什么要有：虚拟地址让每个进程有独立、连续的地址空间（不用管物理内存碎片），还能隔离（进程 A 的 VA 指向的物理页与进程 B 的不同，互不可见）。**VA 区**就是一段连续的虚拟地址范围——本批"划格子"划的就是 VA 区：先定疆界（用户 / 系统分界），再在内核那半地址空间里给各子系统划一段 VA 区，各区互不重叠。

- **系统空间与用户空间**：VA 空间一分为二——系统空间是归内核的那一半（32 位是高 2GB，/3GB 时高 1GB；64 位是高半部分），用户程序不可直接访问；用户空间是归用户程序的那一半。内核代码、池、页表全在系统空间里，用户程序的代码与数据在用户空间里。

- **PTE / PDE / PPE / PXE**：页表是 PTE 排成的翻译表；表项分四级。PTE（页表项）指向 4KB 页；PDE（页目录项）指向一张页表；PPE（页目录指针项）指向一个页目录；PXE（页映射级 4 项）指向一个页目录指针表。32 位只有 PDE / PTE 两级（`_MI_PAGING_LEVELS < 3`）；64 位四级全有（`_MI_PAGING_LEVELS >= 4`）。为什么分级：VA 空间太大，一张平表放不下，逐级索引省页表内存（稀疏地址空间只建用到的分支）。

- **fault（故障）**：访问无效或受保护页时触发的异常（硬件触发）；MM 的 fault 处理函数负责按需调页、或报错（非法访问）。

- **超页（Superpage）**：用 PDE 直接映射一大块（4MB）而不经过 PTE 的映射。相关注释原话："it will always have a PTE itself instead of being superpaged"——超页没有 PTE 可查，fault 处理函数查 PTE 的假设会被打破，所以要求"永远有个 PTE 可查"的映射不用超页。

- **需求零页 PTE（DEMAND_ZERO_WRITE_PTE）**：一种 PTE 存根：页还没分配，fault 时现取一页清零再映射。系统页目录的双映射等"先占位、用到再给"的场合都用它——"先占位、用到再给"的 lazy 思想。

- **原型 PTE（Prototype PTE）**：模板 PTE——不是页表里的项，而是存在共享映射的描述结构里的一份模板；所有映射同一段内存的进程共用它，改它等于改所有映射的 PTE，不用逐进程改页表。为什么要有：共享映射（文件映射、内存映射）需要"一改全改"——文件内容变时，所有映射进程的 PTE 都要变，改原型 PTE 一处即可。

- **KSEG0_BASE**：32 位系统空间的起始地址（0x80000000，即 2GB 分界）。用户空间在它之下，系统空间在它之上。64 位不用这个常量，改用 `MI_HIGHEST_USER_ADDRESS` / `MI_SYSTEM_RANGE_START` 等。

- **/3GB**：启动开关，把 32 位用户空间从 2GB 扩到 3GB、系统空间压到 1GB。为什么有人要：用户程序需要超过 2GB 的 VA（大数据库、大型 CAD）；代价是系统空间（池、PTE 等）全部缩水（见第三部分）。

- **MmVirtualBias（虚拟偏置）**：/3GB 时系统空间的位移量。为什么要有：/3GB 把用户/系统分界从 2GB 推到 3GB，系统空间只剩 1GB；内核各系统区的 VA 都要按这个偏置算（PTE 地址 = KSEG0_BASE + Bias + PFN（页帧号，见二、PFN 条）× 页大小，见正文第 ④ 步）。

- **缓存（Cache）**：CPU 里的高速小内存，存最近用过的物理页内容；访问命中缓存不用访内存，速度快一个量级。为什么要有：内存太慢，缓存把热数据放 CPU 旁边，程序才跑得动。

- **I/O（输入输出，Input/Output）**：与磁盘、网络等外设的数据往来；"等 I/O"指线程等待外设完成读写。

- **TLB（快表，Translation Lookaside Buffer）**：CPU 里的硬件翻译缓存——存"VA → 物理地址"的翻译结果（即 PTE 的缓存副本），按 VA 索引、并行比对。机理：CPU 访问 VA 时先查 TLB，命中直接得物理地址（不用访内存查页表）；未命中则走页表遍历（64 位四级要访内存 4 次）查 PTE，再把结果填进 TLB。为什么要有：页表在内存里，次次查页表要多访内存 2–4 次，TLB 把热翻译放 CPU 旁边，访问才跑得动；TLB 表项有限（几十到几千个），一个 2MB 映射只占一个表项、顶 512 个 4KB 页的表项——大粒度映射多，TLB 压力小、命中率高。失效规则：PTE 改了、进程切换了，旧表项必须刷掉（单核用 invlpg 指令刷单个表项或换页目录全刷，多核还要 TLB shootdown——发处理器间中断通知其他核刷），不刷 CPU 就会按旧翻译访问、读错页。TLB 属性冲突：同一大粒度映射内缓存属性不一时，TLB 表项的缓存属性与页表不一致，CPU 行为未定义——所以大粒度映射的内存必须缓存属性一致。注释里的 TB 即 TLB。

- **guard 页（护栏页）**：故意留空的一页，越界访问立刻 fault 而不是悄悄踩进隔壁区；池边界减一页就是做 guard 页。

- **大页（Large Page）**：2MB（x86 PAE / x64）或 4MB（x86 非 PAE）的大粒度映射。为什么要有：内核映像、HAL 等大块常驻映射，用 4KB 页要几千个 TLB 表项，用大页只要几个——省 TLB 压力。`KF_LARGE_PAGE` 是 CPU 支持大页的特性位。



### 二、PFN 与页状态链表

- **PFN（页帧号，Page Frame Number）**：物理页的编号——第 N 个 4KB 物理页的 PFN 就是 N。MM 不直接说"第 0x12345000 号物理地址"，而说 PFN 0x12345——页帧号是物理内存的"页为单位的地址"。

- **MMPFN 关键字段**：PteAddress（映射它的 PTE 地址——"谁映射它"）、OriginalPte（换出前 PTE 存根）、ShareCount（共享计数）、ReferenceCount（引用计数）、PteFrame（装着它的 PTE 的页目录帧）、PageLocation（页位置：活动 / 备用 / 已改 / 空闲 / 零页 / 坏页等）、CacheAttribute（缓存属性）。

- **PFN 数据库（MmPfnDatabase）**：每物理页一张状态卡的数组 `MmPfnDatabase[PFN]`，元素是 `MMPFN`。做什么用：MM 的全部记账都查它——某页空闲还是在用、被谁映射、被共享几次、引用计数几、换出前 PTE 存根（OriginalPte）、在哪个链表上。为什么必须有：没有它，分配、换页、共享、回收、清零、隔离全做不了，虚拟内存管理整个不存在（第 2 批 QA 第 11 条已讲，本批讲它怎么建）。

- **空闲链表 / 零页链表 / 备用链表 / 已改链表 / 坏页链表**：PFN 数据库之外的五条"按状态串起来的页链"。空闲链表 = 可立即分配的页；零页链表 = 已清零、可立即分配的页（省掉分配时清零）；备用链表 = 内容还在、但已不在工作集里的页（再访问可无盘 I/O 捡回）；已改链表 = 内容改过、还没写回磁盘的页（等写出线程）；坏页链表 = 硬件坏页（`MmBadPageListHead`，永久隔离、不计入可用页总数）。活动页不在任何链表里，靠 PTE 找。为什么分五条链：分配器按"代价"取页——零页最便宜（不用清零）、空闲次之、备用要"抢回"、已改要先写盘、坏页永不给。

- **转换页（Transition Page）**：正在"状态转换中"的页——比如备用链表上的页被再次访问、或已改页正在写盘。`MiRemoveZeroPage` 取页顺序是零页链表 → 空闲链表 → 备用链表（备用页要先把它的 PTE 从转换态恢复成原内容，见 `MM/PFNLIST.C:2049-2063` 的注释原话）。

- **页颜色（Page Color）**：按"页帧号对缓存组数取模"给页分的颜色。为什么要有：直接映射缓存里，不同物理页可能抢同一个缓存组；分配时按颜色轮换取页，把连续分配的页散到不同缓存组，减少缓存冲突。`MiRemoveZeroPage(Color)` 的 Color 参数就是这个意思。

- **PFN 锁（LOCK_PFN / UNLOCK_PFN）**：保护 PFN 数据库与五条链表的全局自旋锁。为什么是全局一把：链表是全系统共享的，不是每核一份；取页、还页、换链表都必须先拿它。机理见第 2 批术语表自旋锁条目。


- **slush**：PFN 建库时"多余的边角内存"——对不齐大页边界的零头。`MxSlushDescriptor1 / 2` 标出的多余 slush 由 `MiAddExpansionNonPagedPool` 收走（`MM/I386/INIT386.C:3013` 一带），且必须全程映射为 cached（避免 TLB 属性冲突）。


### 三、池与地址空间

- **非分页池（Non-Paged Pool）**：常驻内存、永不换出的内核堆。VA 区紧跟 PFN 数据库之后（`MM/I386/INIT386.C:2113` / `:2235`：`MmNonPagedPoolStart = MmPfnDatabase + MxPfnAllocation 页`）。为什么紧跟 PFN 数据库：PFN 数据库是启动期第一块按页数算大小的系统结构，池区顺着它往后划最省事，不用另找 VA 空洞。

- **扩展非分页池（Expansion Non-Paged Pool）**：非分页池的"扩建区"——池不够用时从空闲链表取页扩进来的部分。启动期第一笔扩建来自 slush（见正文第 ④ 步：多余 slush 转入扩展池）。

- **分页池（Paged Pool）**：可换出的内核堆，只能在 IRQL < DISPATCH_LEVEL 用。VA 区与非分页池分开，PTE 与已提交页由 `MiBuildPagedPool` 建（`MM/MMINIT.C:3301`）。为什么分页池最后才建：它要页表 + PFN 数据库 + 可映射任意物理页的 PTE 池三样齐全，缺一样都建不起来（见正文第 ⑥ 步的鸡生蛋）。

- **特殊池（Special Pool）**：驱动验证器（Driver Verifier，微软的驱动验证工具）的特殊池——给可疑驱动的分配加前后 guard 页、专页独占，越界立刻 fault。`MiBuildPagedPool` 末尾调 `MiInitializeSpecialPool(NonPagedPool)` 初始化它；安全启动（safeboot）时禁用它（`MMINIT.C:1143` 注释：安全启动不启用特殊池、验证器等 corruption 追踪手段）。

- **系统 PTE（System PTE）**：系统空间里预留的"可映射任意物理页的 PTE 池"。驱动、I/O、文件映射要把某物理页临时映射进系统空间时，不自己建页表，而是从系统 PTE 池里 reserve（`MiReserveSystemPtes`）。为什么要有：系统空间映射是高频临时需求，集中建池比每次现建页表便宜，且池的 VA 范围固定、便于管理。系统 PTE 空间的大小 = 非分页系统起始到非分页池起始之间的 VA 空隙换算成 PTE 数（`MM/I386/INIT386.C:3270-3295`，大页情形减一页做 guard 页）。

- **系统空间映射（System Space Map）**：`MiInitializeSystemSpaceMap` 初始化的一套结构，供驱动把物理页映射进系统空间用（`MiBuildPagedPool` 末尾调它）。按名义推断：它是系统 PTE 池之上的映射服务层。



### 四、节、段与控制区

- **节（Section）**：可映射的一段内存的抽象（第 2 批补漏术语已定义，本批展开实现）。节对象类型由 `MiSectionInitialization` 在 OB 里登记（`MM/SECTSUP.C:1028` 的 `ObCreateObjectType`，类型名 "Section"，对象从分页池出，默认 charge 为 `sizeof(SECTION)`）。

- **段（Segment）**：节的"页表与页集合"实现体——节是对象层面的名字，段是 MM 内部管理这段映射的数据结构（哪些页、哪些分段、引用计数）。

- **控制区（Control Area，CONTROL_AREA）**：一段映射的"总账本"——指向段、记节引用数、记标志（如 PhysicalMemory）。物理内存节的控制区由 `MiSectionInitialization` 从非分页池分配、清零、置 `PhysicalMemory = 1`。

- **子段（Subsection）**：控制区下面按"页范围"切的小段（比如映像的每个 PE 节对应一个子段）。页 fault 时先找子段、再找原型 PTE——子段是"文件内容 ↔ 内存页"的翻译单位。

- **段引用线程（MiDereferenceSegmentThread）**：`MiSectionInitialization` 建的系统线程，段引用数掉到零时由它做清理（段的析构不能在任意上下文做，集中到一个线程里做）。为什么要有专门线程：段析构要等页、要拿锁、可能要 I/O，不能在调用者上下文里随便做。

- **物理内存节（\Device\PhysicalMemory）**：代表整段物理内存的节对象，大小 = `(1 << PHYSICAL_ADDRESS_BITS) - 1` 字节（即整个物理地址空间），初始保护 `PAGE_EXECUTE_READWRITE`，以 `SECTION_MAP_READ` 插入对象目录。为什么 Phase 1 建它：调试、诊断等需要直接访问物理内存的场合靠它；用节机制统一做，不用另开一条"物理内存映射"的特殊路径。

- **共享用户数据（Shared User Data，KI_USER_SHARED_DATA）**：内核与用户共读的一页——系统时间、版本号、特性位等。内核在系统空间映射它，用户在固定用户地址映射它，两边读同一物理页。为什么共读一页：用户程序读系统时间不用每次都系统调用（GetTickCount、GetSystemTime 直接读这页），省进内核的开销。



### 五、线程与管理器

- **已改页写出线程（Modified Page Writer，MiModifiedPageWriter）**：把已改链表上的脏页写回磁盘（或换页文件）的系统线程。为什么要有专门线程：写盘要等 I/O，只能在能等待的线程上下文做；MM 不能在 fault 路径里同步等写盘。

- **平衡集管理器（Balance Set Manager，KeBalanceSetManager）**：工作集管理 + 栈换入换出的系统线程。为什么要两个线程：注释原话——"performs stack swapping and working set management and requires two threads"（`KeBalanceSetManager` + `KeSwapProcessOrStack`，一个管工作集、一个管栈换入换出）。

- **零页线程（Zero Page Thread）**：把空闲链表上的页清零、推进零页链表的线程。为什么要有：分配器取零页最便宜（不用现清零），但清零本身要 CPU——放在系统空闲时（Idle 上下文）提前做，分配时就不用等。第 2 批已讲：Phase 1 线程转任零页线程；多核另起零页工人。

- **系统缓存（System Cache）**：文件系统缓存的常驻工作集——文件内容缓存在系统空间里，读文件先查缓存。系统缓存有自己的工作集链表（`MmSystemCacheWorkingSetList`），Phase 0 给它建页表（见正文第 ⑤ 步）。

- **会话空间（Session Space，Hydra）**：终端服务（多用户远程登录服务）里每个会话独立的 VA 区——会话分页池、会话映像空间、会话工作集。为什么要有：多用户同机登录时，各会话的驱动与映像互不干扰（WS03 是服务器系统，Hydra 是终端服务的开发代号）。

- **完全初始化旗（MiFullyInitialized）**：MM 完全初始化完成的标志。Phase 1 尾置 1，同时 `MiTrimInProgressCount` 减 1——修剪（把不常用页换出）在此之前被挡住，置旗后才开闸。为什么挡修剪：初始化期 MM 还在接线，修剪会把正在建的页换走。

- **LDRP_ENTRY_NATIVE**：已加载模块的"原生"标志。Phase 1 尾遍历 `PsLoadedModuleList`，NT5+ 映像（目标系统版本 >= NT 5.0，即 Windows 2000 及之后；系统版本 >= 5 且映像版本 >= 5）打上此标志，同时做写保护；老映像不打、不保护（兼容，见正文第 ⑮ 步）。



### 六、其他

- **BBT（Burned Block Table，烧掉的内存表）**：`/BURNMEMORY` 烧掉的内存。加载器把烧掉的页标成 `LoaderBBTMemory` 描述符（`ExBurnMemory`，`INITOS.C:376`），Phase 1 由 `MiMapBBTMemory` 映射进系统空间、用户可访问，地址存进 `BBTBuffer`，每个新线程的 TEB（线程环境块）`ReservedForPerf` 指向它（`MM/PROCSUP.C:5578`）。为什么用户可访问：让用户态性能/诊断工具能读到"哪些内存被烧掉了"。注意：BBT ≠ 坏页（见第六部分第 2 条）。

- **流体值（Fluid Value）**：可用页预算里扣掉的"应急预留"——`MmResidentAvailablePages` = 可用页 − 流体值 − 未来非分页池扩建预留 − 系统缓存工作集。为什么扣：池扩建与缓存是刚性需求，预算里不先扣，系统跑起来会饿死。



### 七、通用小词（正文高频、此前未单独立条）

- **DACL（自主访问控制列表，Discretionary Access Control List）**：安全描述符里"谁能干什么"的表。Phase 0 临时事件的原因就是正式事件要带 DACL，而 DACL 要 OB / SE 到 Phase 1 才能建。

- **TEB（线程环境块，Thread Environment Block）**：每个线程用户态可见的一块结构，放线程私有数据；`ReservedForPerf` 是其中留给性能工具的指针槽，BBT 缓冲区地址就存在这里。

- **syscall（系统调用）**：用户程序进内核的门——用户态代码通过中断或快速系统调用指令切到内核态执行系统服务，执行完返回用户态。共享用户数据页的意义就是高频读数不用走 syscall。

- **CSC（客户端缓存，Client Side Caching）**：离线文件功能。Phase 2 的无盘远程启动闸里 `IoCscInitializationFailed` 指 CSC 初始化失败——无盘客户机连映像文件都在网上，CSC 失败时分页等于自杀。

- **Hydra**：终端服务（Terminal Services）的开发代号；WS03 的会话空间机制沿用此名。

- **WOW64（Windows 32-bit on Windows 64-bit）**：64 位系统上跑 32 位用户程序的兼容层。用户态有它，内核态没有（第 1 批已讲）。

- **NT5**：Windows 2000 的内核版本号（NT 5.0）；NT5+ 指 Windows 2000 及之后（XP 为 5.1，WS03 为 5.2）。镜像写保护的版本分界就卡在这里。

- **CR3（页目录基址寄存器）**：当前进程页目录的物理地址；进程切换时换 CR3 即换地址空间（第 1 批已讲）。

- **Owner 位**：PTE / PDE 里的用户访问位；置 1 则用户态可访问，清 0 则仅内核态。共享用户数据的 x86 分支就是手动把 Owner 位置 1。

- **RAM（内存，Random Access Memory）**：随机存取存储器，即物理内存条；本批"按 RAM 线性给"指系统 PTE 数按物理内存大小线性算。

- **API（应用程序编程接口，Application Programming Interface）**：给调用者用的函数约定；共享用户数据页"不是泄漏、是 API"指用户程序按约定读它，不算越权。

- **DLL（动态链接库，Dynamic Link Library）**：多个程序共用的可执行代码文件；"系统 DLL"指系统自带的 DLL（如 ntdll.dll），"重定基址"指 DLL 加载地址与预设地址冲突时整体搬移。

- **SMSS（会话管理器子系统，Session Manager Subsystem）**：启动期第一个用户态进程，负责建会话、跑 Part 2（绑定 hive 到文件）、拉起 CSRSS（客户端/服务器运行时子系统）与 Winlogon（登录进程）；第 2 批已讲其诞生与 5 秒窗口。

- **AP（应用处理器，Application Processor）**：除启动处理器外的处理器核；"拉 AP"指把 AP 从 halt 中唤醒并让它跑内核初始化。

- **P0（0 号处理器）**：启动处理器（bootstrap processor），第一个跑起来的核；Phase 0 只有 P0 在跑。

- **MP / UP（多处理器 / 单处理器，Multi-Processor / Uni-Processor）**：MP 指多核机器，UP 指单核机器；`#ifndef NT_UP` 的代码只在 MP 下编译进内核。

- **NLS（国家语言支持，National Language Support）**：字符集、排序、大小写转换等语言相关的表；启动期 NLS 表由加载器读入、MM 拷入池。

- **ABI（应用二进制接口，Application Binary Interface）**：二进制层面的调用约定（结构体布局、指针宽度、寄存器用法）；"二进制 ABI 全变"指 32 位与 64 位驱动在二进制层面完全不兼容。

- **ACL（访问控制列表，Access Control List）**：安全描述符里"谁能干什么"的表；DACL 是其中"自主"的那种（由对象所有者定）。

- **ROM（只读存储器，Read Only Memory）**：断电不丢、不可写的存储器；XIP（就地执行）指直接在 ROM 上跑代码而不拷入内存。

- **NT64**：64 位 NT（NT 64-bit）的简称；注释里的"Hydra NT64"指 64 位终端服务系统。

- **WS03 SP1**：Windows Server 2003 第一个服务包版本，即本批底本 WRK 的代码线。

- **WRK（Windows Research Kernel）**：微软公开的 Windows 研究内核源码，即本批研读的源码树。

- **CAD（计算机辅助设计，Computer Aided Design）**：/3GB 的例子里"大型 CAD"指需要超大用户地址空间的设计软件。

- **GB / MB / KB**：吉 / 兆 / 千字节，1024 进制（1GB = 1024MB）。
- **QA（问答，Question & Answer）**：第 2 批笔记的问答区；"QA 第 N 条"指该区第 N 条（本批引用第 2 批 QA 第 11 条时即指此处）。


---
## 第二部分：正文

### Phase 0（`MMINIT.C:433` 起）：MM 上线的第一天

#### ① 地址空间划界——先定疆界，再谈分配

Phase 0 第一件事是定地址空间的疆界（`MMINIT.C:433` 起的 Phase 0 分支）：

- 64 位：`MmHighestUserAddress = MI_HIGHEST_USER_ADDRESS`、`MmUserProbeAddress = MI_USER_PROBE_ADDRESS`、`MmSystemRangeStart = MI_SYSTEM_RANGE_START`——三组常量直接取架构定义；
- 32 位：`MmHighestUserAddress = KSEG0_BASE - 0x10000 - 1`、`MmUserProbeAddress = KSEG0_BASE - 0x10000`、`MmSystemRangeStart = KSEG0_BASE`——用户空间顶在系统空间起始之下，还留 64KB 空隙；
- 启动映像尺寸：`MmBootImageSize = LoaderBlock->Extension->LoaderPagesSpanned × PAGE_SIZE`，再按 PDE 映射粒度取整（x86 / AMD64 分支）——加载器占用的 VA 跨度（第 0 批已讲 `LoaderPagesSpanned` 是"加载器占用的虚拟内存跨度，MM 不能回收其 VA 的部分"）；
- x86 的虚拟偏置：`MmVirtualBias = LoaderBlock->u.I386.VirtualBias`——/3GB 时系统空间的位移量，加载器按启动开关填好，MM 直接取。

为什么先划界：后面所有 VA 分配（池区、PTE 区、缓存区、会话区）都是在这张疆界图里划格子；疆界不定，格子没法划。为什么 32 位用户顶要留 64KB 空隙（`MmUserProbeAddress = KSEG0_BASE - 0x10000`）：用户指针探查（probe）到顶时，空隙保证探查越界一定 fault，而不是绕回低地址——64KB 的"无人区"是防绕回的护栏。为什么 64 位不用 KSEG0 算术：64 位 VA 巨大，用户顶与系统起始不是"2GB 对半"这种固定分界，而是架构定义的常量——32 位的 KSEG0 算术在 64 位没有对应物。

没有划界会怎样：池、PTE、缓存的 VA 区会互相重叠，第一个分配就踩坏第二个——启动期直接崩，连蓝屏信息都未必打得出（那时连 bugcheck 的页表都还没全）。

#### ② 临时事件——Phase 1 才能建的东西，Phase 0 先拿个占位符顶着

Phase 0 一开头就建了一个临时事件（`MMINIT.C:114-118` 的注释原话："A temporary unsignaled event for the available pages & pool events to point at early during phase 0. This is because the exportable events with DACLs cannot be created until phase 1."）：

- 建 `MiTempEvent`（NotificationEvent、初始不 signaled）；
- 六个事件指针全指向它：`MiLowMemoryEvent` / `MiHighMemoryEvent`（可用页高低水位）、`MiLowPagedPoolEvent` / `MiHighPagedPoolEvent`（分页池水位）、`MiLowNonPagedPoolEvent` / `MiHighNonPagedPoolEvent`（非分页池水位）。

为什么必须占位：Phase 0 的代码路径里已经有"等水位事件"的地方（比如池耗尽时等事件），指针不能悬空；但正式事件带 DACL（安全描述符）——建 DACL 要 OB 与 SE，而 OB / SE 的 Phase 0 才刚跑、Phase 1 还没到（第 2 批地图：Ob 在 `:827`、Se 在 `:831` 只有 Phase 0 部分，事件的正式创建要等 Phase 1）。为什么临时事件初始"不 signaled"：它永远不会被 signal——等它的线程会一直等到 Phase 1 正式事件替换它；如果初始是 signaled，等水位的线程会误以为"水位已恢复"而继续分配，池耗尽时直接踩空。

Phase 1 里 `MiInitializeMemoryEvents`（`MMINIT.C:2315` 一带，在起平衡集管理器之前，注释原话："This must be done before starting the working set manager"）把六个指针换成正式事件——因为工作集管理器要等这些事件，换晚了它等的是个永不 signal 的占位符。

#### ③ 尺寸测算——32 位争 VA，64 位争物理页

Phase 0 的中段在算账（`MMINIT.C:1114` 起的测算区）：算系统有多少页、池要多大、系统 PTE 要多少、缓存留多大。算账的几条硬规矩：

- **哪些内存算"系统页"**：`IncludeType[LoaderMaximum]` 数组列出计入的加载器内存类型（`MMINIT.C:1117` 一带的注释："include all memory types ... except these ... If you change this list, please make the corresponding changes over in boot\lib\blmemory.c!BlDetermineOSVisibleMemory()"）——加载器与内核必须对"哪些内存是操作系统可见的"口径一致，否则两边算出的总页数对不上，PFN 数据库建出来就错位。
- **64 位不争 VA**：注释原话（`MMINIT.C:1177-1186`）——"64-bit NT is not constrained by virtual address space - but there is physical cost for the preallocated page table pages used by system PTEs. No other tradeoffs between nonpaged pool, paged pool and system PTEs need to be made. So just allocate PTEs on a linear scale as a function of the amount of RAM. For example on a Hydra NT64, 4gb of RAM gets 128gb of PTEs."——64 位 VA 多到不用争，系统 PTE 数直接按 RAM 线性给（4GB 内存给 128GB 的 PTE 空间——PTE 空间本身是 VA，不占物理页，代价只是预先建好的页表页占的物理内存）。
- **32 位处处争**：系统空间只有 2GB（/3GB 时 1GB），非分页池、分页池、系统 PTE、系统缓存全挤在这点 VA 里；池大了 PTE 就小、缓存大了池就小。注册表可以定"系统 PTE 最大数"（`MMINIT.C:1224` 一带），可疑驱动名单（Driver Verifier 的 suspect list）还要额外加系统 PTE（`MMINIT.C:1277` 一带：特殊池要给每个可疑分配独占页 + guard 页，PTE 消耗翻倍）。
- **分配偏好键**：注册表 `AllocationPreference`（top-down 分配）要小心管（`MMINIT.C:1292-1309` 的注释）：每次分配都从高地址往下取，ntdll 启动期分配（比如栈回溯数据库）拿到高地址后，后续系统 DLL 重定基址会撞车；32 位只在 /3GB 下这个键才有用（因为 2GB 模式下系统 DLL 就在 2GB 之下，问题不存在）；64 位 VA 充足，根本不是问题。

为什么算账必须在 Phase 0 做：池区、PTE 区、缓存区的 VA 格子要按算出的尺寸划，划晚了后面没法改——VA 一旦分出去，收不回来。没有算账会怎样：池区划小了，驱动一加载就池耗尽；PTE 区划小了，第一个大 I/O 映射就没 PTE 可用（`MiIssueNoPtesBugcheck`）；缓存区划大了，池被挤死——三种死法都是启动期蓝屏。

**MiMemorySan 的位置（未钉死清单第 1 条）**：`MiMemorySan(LoaderBlock)` 在 Phase 0 测算区开头被调（`MMINIT.C:1114`），前向声明在 `:100`，但定义不在 WRK 树内（全树搜不到实现）。从位置推断：它在"算哪些内存类型计入、算总页数"之前跑，应是对加载器内存描述符做一次清理/校验；但内部清单树内无据，本批不写它的具体行为。

#### ④ 机器相关初始化与 PFN 建库——本批的核心

`MiInitMachineDependent(LoaderBlock)` 在 Phase 0 被调（`MMINIT.C:1389`），定义在 `MM/I386/INIT386.C:763`（AMD64 版在 `MM/AMD64/INITAMD.C:1131`）。它在 Phase 0 干三件大事：PFN 建库、非分页池布局、系统 PTE 初始化；Phase 1 再进同一个函数时走另一条分支（`:854`，见 Phase 1 第 ⑦ 步）。先讲 Phase 0 的 PFN 建库——这是整批最关键的一段。

**建库的输入**：加载器的内存描述符链表（`LoaderBlock->MemoryDescriptorListHead`，每条含 BasePage、PageCount、MemoryType，第 2 批 QA 第 11 条已讲）。建库的循环从链表**尾**往头走（`INIT386.C:2964` 起，`NextMd = MemoryDescriptorListHead.Blink`）：

- **超界跳过**：帧号超过 `MmHighestPhysicalPage` 的描述符跳过——注释原话："> 16GB 的内存 + /3GB 启动时，顶部的处理会把最高物理页压下来再建 PFN 数据库，但加载器块里还留着 16GB 之上的描述"。为什么：/3GB 把系统空间压到 1GB，PFN 数据库本身占 VA，物理内存太多时 PFN 数据库放不下，只能先压最高页（后面的页暂时不用）——不跳过，PFN 数组就越界。
- **slush 转扩展池**：`MxSlushDescriptor1 / 2` 标出的多余 slush（对不齐大页边界的零头）调 `MiAddExpansionNonPagedPool` 转入扩展非分页池，且必须全程映射为 cached——注释原话："to ensure that it is always mapped fully cached since the rest of the large page is inserted as fully cached into the TB"（TLB 属性冲突：同一大页里有的页 cached、有的不 cached，TLB 会乱）。
- **坏页进坏页链表**：`LoaderBad` → 逐页 `MiInsertPageInList(&MmBadPageListHead)`（`INIT386.C:3013-3018` 一带）。坏页直接 return，不计入可用页总数（第 2 批 QA 第 11 条已钉死：`MM/PFNLIST.C:830`）。
- **可释放的进空闲链表**：`LoaderFree / LoaderLoadedProgram / LoaderFirmwareTemporary / LoaderOsloaderStack` → 从区段顶往下逐页进空闲链表（`MiInsertPageInFreeList`），每页置 `CacheAttribute = MiCached`、调 `MiDetermineNode` 定 NUMA 节点、只在 `ReferenceCount == 0` 时进（已在用的不重复进）。为什么从顶往下：`MiInsertPageInFreeList` 是尾插（`MM/PFNLIST.C:112` 注释原话："inserts a page at the end of the free list"），从区段顶往下登记，链表里页的次序就是从高到低——登记顺序即链表次序，不用再排序。为什么 `LoaderOsloaderStack`（加载器栈）能进空闲链表：加载器栈在交接后就没用了，栈页直接还给系统——这是"加载器用完的东西立即还"的第一例。
- **三类跳过**：`LoaderFirmwarePermanent / LoaderSpecialMemory / LoaderBBTMemory` → "Skip these ranges"——固件永久内存（ACPI 表等，碰不得）、特殊内存、BBT 内存（烧掉的内存，Phase 1 另行映射，不进 PFN 链表）。
- **其余全部标在用**：`default` 分支（系统代码、HAL 代码、启动驱动、注册表映像、NLS 表、加载器堆等）→ 逐页标 `ActiveAndValid`：`PteFrame` 取自该 PTE 所在页表页的帧（即 PDE 里记的帧号）、`PteAddress` 取自 PTE 地址（PTE 地址的算法是 `KSEG0_BASE + Bias + PFN × PAGE_SIZE`——系统空间里物理页的 PTE 地址是算出来的，不是查表查出来的）、`ShareCount += 1`、`ReferenceCount = 1`、`CacheAttribute = MiCached`。XIP（就地执行 ROM）另有特殊处理（`Rom = 1`、`PrototypePte = 1`）。

**建库收尾——PFN 数据库描述自己**：描述符循环走完后（`INIT386.C:3115` 一带），如果 PFN 数据库不是大页映射，且 `MmVirtualBias == 0`，MM 再扫一遍"映射 PFN 数据库的 PTE"，把其中有效的 PTE 对应的 PFN 条目标成 `ShareCount = 1`、`ReferenceCount = 1`——PFN 数据库自己的页，在 PFN 数据库里有自己的状态卡。这就是"PFN 数据库描述自己"：它的页不是法外之地，同样被记账、同样不能被重复分配。

**非分页池布局（同一函数里）**：PFN 数据库放在系统 PTE 区的顶部（"where nonpaged pool would have started"），非分页池起始往上滑（`INIT386.C:2113` / `:2235`：`MmNonPagedPoolStart = MmPfnDatabase + MxPfnAllocation 页`）。大页情形（`MI_LARGE_PFN_DATABASE_AND_NONPAGED_POOL`）要求 PFN 数据库 + 非分页池取自一块大页对齐的空闲描述符区、全部映射为 cached（又是 TLB 属性冲突的原因）。

**系统 PTE 初始化（同一函数尾，`INIT386.C:3270-3295`）**：`MmNumberOfSystemPtes` = 从 `MmNonPagedSystemStart` 的 PTE 到非分页池起始的 PTE 之间的 PTE 数（非 /3GB 减 1；/3GB 大页情形对齐 PFN 数据库后减 1 做 guard 页）；调 `MiInitializeSystemPtes(PointerPte, MmNumberOfSystemPtes, SystemPteSpace)` 建池。为什么减 1 做 guard 页：池的边界留一页无人区，越界访问立刻 fault 而不是悄悄踩进隔壁区。

> **鸡生蛋（1）：PFN 数据库与内存分配——来龙去脉**
>
> **结为什么会形成**：MM 的分配器（取空闲页、取零页）完全依赖 PFN 数据库的链表——没链表就没法分配；但 PFN 数据库本身是个大数组（每物理页一个 `MMPFN`），它自己也要占内存。先有数据库还是先有分配，表面看是死循环。
>
> **靠什么打破循环**：三样东西让它不用分配器就能建库。第一，PFN 数据库的 VA 地址是**算出来的**，不是分配出来的：架构初始化按"系统 PTE 区顶部"定 `MmPfnDatabase`，非分页池再顺着它往后滑——VA 格子是地址算术，不是分配结果。第二，PFN 数据库的**物理页取自加载器的空闲描述符**，不用分配器：建库循环直接走描述符链表，把空闲描述符的页登记进链表、把在用描述符的页标成在用——登记动作本身不需要先有分配器。第三，PFN 数据库的 PTE 地址也是**算出来的**：`KSEG0_BASE + Bias + PFN × PAGE_SIZE`，因为加载器预建的页表已经把系统空间按"物理页连续映射"搭好了，MM 不用先建页表就能算出任意物理页的 PTE 地址。
>
> **打破后是什么局面**：建库循环走完，空闲 / 零页 / 坏页链表都有了，PFN 数据库自己的页也在数据库里标成在用（自描述）；从此分配器只跟链表打交道，不再碰加载器描述符。加载器描述符链表的使命结束——Phase 1 尾 `MmFreeLoaderBlock` 把它连同整块 LoaderBlock 一起释放（第 2 批的 LoaderBlock 释放，`INITOS.C:1615-1621`）。

#### ⑤ 页表自举——页表页现取现建，PFN 条目与有效 PTE 一起落袋

Phase 0 后段给系统缓存工作集建页表（`MMINIT.C:1760` 起）。模式是固定的四步，PXE / PPE / PDE 逐级：

1. 该级表项无效（`Valid == 0`）才动手——已有效的跳过（共享用户数据那页在 4 级情形下已有效，断言它有效）；
2. `LOCK_PFN` 后调 `MiRemoveZeroPage(颜色)` 取一页——取页顺序是零页链表 → 空闲链表 → 备用链表；不是从零页链表取的，当场清零（`MM/PFNLIST.C:2049-2063` 的注释原话，见术语表"转换页"）；
3. `MiInitializePfnAndMakePteValid(帧号, 表项地址, 有效 PTE)`——**PFN 条目初始化与 PTE 置有效是同一个函数里一起做的**；
4. `MmResidentAvailablePages -= 1`——可用页预算扣一页。

为什么 PFN 条目与 PTE 置有效必须一起做：页表页是"映射别人的页"，它自己也得被记账；如果先置 PTE 有效、后补 PFN 条目，中间窗口里这页是"已映射但未记账"——别的 CPU 或一次中断里分配器可能把同一页又分出去，两张映射指向同一物理页，数据互相覆盖。一起做，窗口不存在。为什么取页要按颜色：页表页也要散到不同缓存组（术语表"页颜色"）。

> **鸡生蛋（2）：页表与 PFN 数据库——来龙去脉**
>
> **结为什么会形成**：要映射 PFN 数据库（以及系统缓存、池区），需要页表；页表本身是物理页，也需要 PFN 条目记账。先有页表还是先有 PFN 条目，表面看又是死循环。
>
> **靠什么打破循环**：加载器预建的页表是第一块垫脚石——内核启动时已经跑在加载器搭好的页表上，系统空间的基本映射（内核映像、PFN 数据库预留区）已经有效；MM 扩展页表时不从零建，而是"缺哪级补哪级"：逐级检查表项，无效才取页。取页不依赖页表——取页走的是 PFN 链表（已建好），页的 PFN 条目与 PTE 置有效是同一个函数（`MiInitializePfnAndMakePteValid`）里原子般一起落袋，中间没有"已映射未记账"的窗口。
>
> **打破后是什么局面**：系统空间的页表与 PFN 数据库互相咬合——每张页表页都有 PFN 条目，每个 PFN 条目都指着映射它的 PTE（`PteAddress`）；`MiBuildPagedPool` 里再把系统页目录双映射（`MmSystemPageDirectory[]` + `MmSystemPagePtes`），MM 从此能按 VA 算术找到任意 PTE，页表自举完成。

#### ⑥ 分页池建成——Phase 0 的收尾，分页池从此可分配

`MiBuildPagedPool` 在 Phase 0 末尾被调（`MMINIT.C:2092`），定义在 `:3301`。它的头注释原话就是它的交付标准："This function is called to build the structures required for paged pool and initialize the pool. Once this routine is called, paged pool may be allocated."——调完它，分页池就能分配了。

它干的事：

- **定大小**：`MmSizeOfPagedPoolInBytes`——默认 2 倍于非分页池最大值、有下限 `MM_MINIMUM_PAGED_POOL_NTAS`、受 VA 空间封顶（`MMINIT.C:3392-3415` 一带）。为什么默认 2 倍：分页池可换出，不占物理内存，VA 上可以比非分页池大方；但 VA 终归有限（32 位尤其），封顶不能少。
- **双映射系统页目录**（2 级页表情形，`_MI_PAGING_LEVELS < 3`）：`MmSystemPageDirectory[]` 逐项取系统页目录页的帧号、其 OriginalPte 置 `DEMAND_ZERO_WRITE_PTE`；再从系统 PTE 池 reserve `PD_PER_SYSTEM` 个 PTE，`MmSystemPagePtes` 指向它们映射出的 VA——系统页目录既被页表机制映射，又被一段连续 VA 映射，MM 能按 VA 直接读写页目录项。4 级情形走 PXE / PPE 分支，不双映射（页表层级够，算术可达）。
- **提交初始页**：按 PTE / PDE 范围逐页取页、置有效 PTE、计 `AdditionalCommittedPages`，最后 `MmTotalCommittedPages` 累加——分页池的初始已提交页就此落袋；后续分配在池内切块，不够再向 MM 要页（池扩建）。
- **收尾两件**：`MiInitializeSpecialPool(NonPagedPool)`（特殊池初始化，见术语表；安全启动时禁用）、`MiInitializeSystemSpaceMap(NULL)`（系统空间映射结构初始化，按名义推断：供驱动把物理页映射进系统空间用）。
- **描述符初始化**：分页池描述符由 `InitializePool(PagedPool, 0L)` 初始化（`MMINIT.C:3656`，调 `EX/POOL.C:894` 的 `InitializePool`，内调 `ExInitializePoolDescriptor` 逐个初始化描述符，`EX/POOL.C:660` 的注释原话："Note that this routine is called directly by the memory manager"）——池的"块管理"（空闲块链表、大小统计）至此就绪。非分页池描述符同理，由架构初始化里的 `InitializePool(NonPagedPool, 0)` 初始化（`MM/I386/INIT386.C:3263`）。

> **鸡生蛋（3）：池与 PFN 数据库——来龙去脉**
>
> **结为什么会形成**：池分配要页，页要 PFN 链表；池的 VA 区、PTE、已提交页又要页表与 PFN 条目。池要是先建，没页表没链表；链表要是先建，池的描述符（`POOL_DESCRIPTOR`）又要从池里出（描述符本身是内核堆对象）——表面看池与链表互相等。
>
> **靠什么打破循环**：靠**分层建、每层只用已建好的层**。顺序是：PFN 数据库（描述符直登记，不用池）→ 系统 PTE 池（VA 算术划区，不用池）→ 非分页池区（VA 紧跟 PFN 数据库划，不用池；描述符由架构初始化直接调 `InitializePool` 初始化，描述符是全局静态量 `NonPagedPoolDescriptor`，不是从池里分配的）→ 分页池（最后建，此时页表、链表、系统 PTE 全齐，`InitializePool(PagedPool, 0L)` 在 `MiBuildPagedPool` 里调）。描述符不是池分配物，而是全局静态结构——这是打破循环的最后一块拼图。
>
> **打破后是什么局面**：`MiBuildPagedPool` 返回后，`ExAllocatePoolWithTag(PagedPool / NonPagedPool)` 全可用；Phase 1 的 `MmSharedUserDataPte`（从分页池出）、`MmFreeLoaderBlock` 的 `RunBase`（从非分页池出）都是这时才敢调的——第 2 批里这些调用点的位置（Phase 1 而非 Phase 0）正是这个顺序的证据。

#### ⑦ 预算与缓存结构——Phase 0 的算账收尾

Phase 0 后段还有一笔预算账（`MMINIT.C:1700-1760` 一带）：

- `MmResidentAvailablePages` = 可用页 − 流体值（应急预留）− 未来非分页池扩建预留 − 系统缓存工作集大小。为什么扣三笔：非分页池扩建是刚性需求（驱动加载、I/O 高峰都要扩池），系统缓存工作集是文件缓存的底线，两者都不扣，系统跑起来先饿死池与缓存，再饿死用户程序。

- 工作集扩展自旋锁初始化、节扩展资源初始化、系统缓存结构建立——工作集管理器与缓存管理器 Phase 0 先把自己的锁与结构立好，线程 Phase 1 再起（锁先于线程，见第 2 批"先建锁、后起线程"的惯例）。

Phase 0 至此结束：MM 能分配（链表）、能记账（PFN 数据库）、能换页（页表与池）、能缓存（系统缓存结构）。但它还不能映射（节对象类型未登记）、不能回收（MM 线程未起）、不能自保（镜像未写保护）——那是 Phase 1 的事。

### Phase 1（`MMINIT.C:2211` 起）：MM 从"能用"到"能管"

#### ⑧ 大页转换——为什么等到 Phase 1：因为要全体处理器的特性位

Phase 1 一开头又进了 `MiInitMachineDependent`（`MMINIT.C:2219`），这次走 `:854` 的 Phase 1 分支（注释原话）："If *ALL* the booted processors support large pages, and the number of physical pages is greater than the threshold, then map the kernel image, HAL, PFN database and initial nonpaged pool with large pages."——代码检查 `(KeFeatureBits & KF_LARGE_PAGE) && (MxMapLargePages != 0)`，然后对 `MiLargeVaRanges[]` 里每段 VA 调 `MxConvertToLargePage`（内核映像、HAL、PFN 数据库、初始非分页池四段）。

为什么等到 Phase 1：Phase 0 只有 P0 在跑，"全体处理器是否支持大页"根本无从知道；Phase 1 里 `KeStartAllProcessors`（第 2 批地图第 ⑩ 步，`INITOS.C:1187`）已把 AP 拉起，各核特性位齐了，才能下"全体支持"的结论。为什么只转这四段：它们又大（内核映像、PFN 数据库按内存规模算）、又常驻（永不换出），大页省 TLB 的收益最大；普通驱动、池块大小不一、生命周期不定，转大页反而浪费（大页粒度粗，零头难处理）。为什么"全体支持"才转：大页映射是 CPU 特性，某核不支持时那核访问大页映射会 fault——全体通过是硬条件，缺一核都不行。为什么 Phase 0 的 `:875` 断言 `MxMapLargePages == MI_LARGE_ALL`、`:900` 一带又在芯片不支持时清零：Phase 0 先按"全转"做准备（PFN 数据库与非分页池取大页对齐的空闲区），芯片不支持则 Phase 0 就把开关关掉，Phase 1 的转换分支自然不进——准备与执行分两阶段，开关在两阶段之间定。

#### ⑨ BBT 映射——烧掉的内存，映射出来给用户态看

`MiMapBBTMemory(LoaderBlock)` 在 Phase 1 被调（`MMINIT.C:2222`），定义在 `:2400`。头注释原话："walks through the loader block's memory descriptor list and maps memory reserved for the BBT buffer into the system. The mapped PTEs are PDE-aligned and made user accessible."

- 输入：`BBTPagesToReserve`——`/BURNMEMORY` 烧掉的页数（`INITOS.C:376`：`BBTPagesToReserve = ExBurnMemory(LoaderBlock, ...)`）；`<= 0` 直接返回（没烧内存，没事干）。
- 做法：按 PDE 对齐 reserve 足够的系统 PTE（注释原话："Request enough PTEs such that protection can be applied to the PDEs"——保护要按 PDE 粒度加，PTE 数先按 PDE 对齐取整），把烧掉的内存区映射进去，用户可访问；`BBTBuffer = Va`（`:2575`）。
- 消费者：每个新线程的 TEB `ReservedForPerf = BBTBuffer`（`MM/PROCSUP.C:5578`）——用户态性能/诊断工具通过 TEB 读到"哪些内存被烧掉了"。

为什么烧掉的内存还要映射：烧掉 ≠ 扔掉——`/BURNMEMORY` 是"假装这段内存不存在"（模拟低内存、绕开坏内存），但诊断工具需要知道"被假装不存在的是哪段"；映射出来、用户可读，工具不用进内核就能查。为什么 PDE 对齐：保护（用户可访问的读写属性）是按 PDE 加的，PTE 数不对齐 PDE，加保护时会跨 PDE 污染隔壁区。

#### ⑩ 节对象类型 + 段引用线程——MM 的映射能力在 OB 里登记

`MiSectionInitialization()` 在 Phase 1 被调（`MMINIT.C:2224`），定义在 `MM/SECTSUP.C:958`。头注释原话："creates the section object type descriptor at system initialization and stores the address of the object type descriptor in global storage."

- 登记节对象类型：`ObCreateObjectType`（`SECTSUP.C:1028`），类型名 "Section"、对象从分页池出、默认 charge `sizeof(SECTION)`、删除过程 `MiSectionDelete`、通用映射 `MiSectionMapping`、`UseDefaultObject = TRUE`，结果存 `MmSectionObjectType`。
- 建段引用线程：`PsCreateSystemThread` 起 `MiDereferenceSegmentThread`（`SECTSUP.C:1046` 一带）——段引用数掉零时的清理工（见术语表）。
- （物理内存节见下一步，同属本函数。）

为什么 Phase 1 才登记：`ObCreateObjectType` 要 OB（对象管理器 Phase 1 在 `INITOS.C:1250` 已重初始化，见第 2 批），节对象本身从分页池出（Phase 0 末才可用）——两个前提都在 Phase 1 才齐。为什么节对象类型要 OB 登记：节是对象（有句柄、有名字、有安全描述符），不登记就进不了对象系统，`NtCreateSection` / `MapViewOfSection` 无从谈起——MM 的映射能力必须先在 OB 里挂号，第 4 批 OB 点火展开时这就是现成的类型。

#### ⑪ 物理内存节——整段物理空间装进一个节对象

同属 `MiSectionInitialization`（`SECTSUP.C:1091` 起）：

- 控制区从非分页池分配、清零、`ControlArea->Segment = Segment`、`NumberOfSectionReferences = 1`、`u.Flags.PhysicalMemory = 1`；
- 节名 `\Device\PhysicalMemory`，`ObCreateObject(KernelMode, MmSectionObjectType, ...)` 建节对象；
- `Section->SizeOfSection = (1 << PHYSICAL_ADDRESS_BITS) - 1`——整个物理地址空间；`InitialPageProtection = PAGE_EXECUTE_READWRITE`；`ObInsertObject(SECTION_MAP_READ)` 插入对象目录；关句柄，返回 TRUE。

为什么用节对象装物理内存：映射物理内存与映射文件、映射页面文件在 MM 内部走同一套"节 → 段 → 子段 → 原型 PTE"的机制；物理内存节只是其中"控制区标 PhysicalMemory、大小等于整个物理空间"的一个特例——不用另开一条特殊路径，调试器、诊断工具调 `NtOpenSection(\Device\PhysicalMemory)` + `MapViewOfSection` 就能读写物理内存（对象安全检查照常，见第五部分）。为什么初始保护是 `PAGE_EXECUTE_READWRITE`：物理内存里有代码（内核映像）也有数据，节的初始保护取最宽，后续映射时再按需收紧——节保护是"上限"，不是"每次都按这个来"。

#### ⑫ 共享用户数据双映射——内核与用户共读一页

`MiSectionInitialization` 返回后立即做共享用户数据双映射（`MMINIT.C:2224` 之后）：

- `MmSharedUserDataPte = ExAllocatePoolWithTag(PagedPool, sizeof(MMPTE), '  mM')`——**PTE 本身故意从分页池出**，注释原话："it will always have a PTE itself instead of being superpaged. This way, checks throughout the fault handler can assume that the PTE can be checked without having to special case this."——超页映射没有 PTE 可查，fault 处理函数里"查 PTE"的假设会被打破；从分页池出 PTE，保证它永远有个 PTE 可查。
- 取 `KI_USER_SHARED_DATA` 的 PTE、`MI_MAKE_VALID_KERNEL_PTE` 做有效内核 PTE 拷进 `MmSharedUserDataPte`、对应 PFN 条目的 `OriginalPte = DEMAND_ZERO_WRITE_PTE`（页还没真正分配时先占位，fault 时再给零页）。
- x86 专属分支：`MmHighestUserAddress < MM_SHARED_USER_DATA_VA` 时，把 PTE 与 PDE 的 `Owner` 置 1（用户可访问）、`MI_WRITE_VALID_PTE` 把映射装进用户地址空间——共享用户数据的 VA 在 32 位某些布局下会落在系统空间一侧，此时要手动把它对用户开放。

为什么双映射：系统时间、版本号、特性位是用户程序高频读的数据（`GetTickCount`、`GetSystemTime`）；共读一页，用户直接读内存，不用每次系统调用——syscall 开销省了，时间精度还更高（不用进内核取）。为什么 PTE 从分页池出而不是静态量：静态 PTE 没有"自己的 PTE"（它不在页表映射的 VA 里），fault 处理函数查它要特殊处理；分页池出的 PTE 自己也在页表里，查 PTE 的代码不用分情况。

#### ⑬ 会话支持——Hydra 的会话空间就绪

Phase 1 接着调三个会话初始化：`MiSessionWideInitializeAddresses`、`MiInitializeSessionWsSupport`、`MiInitializeSessionIds`——会话范围地址、会话工作集支持、会话 ID。为什么 Phase 1 才做：会话空间（会话分页池、会话映像空间、会话工作集）是"每个登录会话一份"的 VA 结构，要 OB（会话对象）、池（会话分页池）、节（会话映像）全齐——三者都在 Phase 1 才就绪。为什么 WS03 要会话空间：WS03 是服务器系统，终端服务多用户同机登录，各会话的驱动映像与池互不干扰是硬需求；工作站版 Windows 会话空间只有 0 号会话，本质是同一套机制的不同规模。

#### ⑭ MM 系统线程——能等待的活，必须是线程

Phase 1 起三组 MM 系统线程（`PsCreateSystemThread` 调用点在 `MMINIT.C:2301` / `:2330` / `:2342`）：

- **已改页写出线程** `MiModifiedPageWriter`（`:2301`）：把已改链表上的脏页写回磁盘或换页文件。为什么必须是线程：写盘要等 I/O，只能在能等待的上下文做；fault 路径里不能同步等写盘（会卡住 fault 的线程，甚至死锁）。
- **平衡集管理器** `KeBalanceSetManager`（`:2330`）+ **栈换入换出** `KeSwapProcessOrStack`（`:2342`）：注释原话——平衡集管理器"performs stack swapping and working set management and requires two threads"（一个管工作集修剪与平衡、一个管栈的换入换出）。起之前先调 `MiInitializeMemoryEvents`（注释原话："This must be done before starting the working set manager"）——工作集管理器要等低/高内存事件，事件必须先从临时占位符换成正式事件（见第 ② 步）。
- **零页工人**：多核才起（`#ifndef NT_UP` 的 `MiStartZeroPageWorkers`）——把空闲页清零推进零页链表。单核不起：Phase 1 线程转任零页线程（第 2 批已讲），一核一个清零工就够；多核起多个工人，清零并行做（见第四部分）。

为什么这些线程 Phase 1 才起：线程要 PS（进程/线程对象）、要 OB（线程对象类型）、要能等待（事件、队列）——Phase 0 连进程对象都没全（`PsInitSystem(0)` 只有 Phase 0 部分），起线程无从谈起。为什么 Phase 0 不能直接"顺手"把脏页写了、把零页清了：Phase 0 是"接线期"，只能做不等待的直连操作；一涉及等待（I/O、事件、队列），就必须有线程——这是 Phase 0 / Phase 1 分工的根本原因（"能等的活 Phase 1 做"）。

#### ⑮ 镜像写保护 + NATIVE 标记——Phase 1 尾，给系统代码上锁

Phase 1 尾遍历 `PsLoadedModuleList`（`MMINIT.C:2380` 一带，持 `PsLoadedModuleResource` 排他锁）：

- 每个模块取 PE 头；`MajorOperatingSystemVersion >= 5` 且 `MajorImageVersion >= 5`（NT5+ 映像）→ 打 `LDRP_ENTRY_NATIVE` 标志 + 调 `MiWriteProtectSystemImage(DllBase)`；
- `MiWriteProtectSystemImage`（`MM/SYSLOAD.C:6689`，头注释原话："sets the protection of a system component as specified"）：按 PE 节表的 `IMAGE_SCN_MEM_WRITE / READ / EXECUTE` 给各节的页加保护——代码节变执行读、只读数据节变只读；
- 老映像（NT5 之前）跳过：注释原话——"Images prior to Win2000 were not protected from stepping all over their (and others) code and readonly data. Here we somewhat preserve that behavior"——老映像当年就是不保护的，节表保护位未必可靠，强行加保护会打断它们（兼容）。

为什么 Phase 1 尾才做：要遍历全部已加载模块——boot 驱动是 `IoInitSystem`（Phase 1 的 `:1531`）才加载进 `PsLoadedModuleList` 的，之前做，boot 驱动不在表里，漏保护；且函数本身是 `PAGED_CODE`（在 PAGE 节里），要分页机制已上线（Phase 0 末）才能跑。为什么保护：运行时代码页只读，驱动 bug 或攻击改写内核代码会立刻 fault，而不是悄悄篡改成功——这是内核自保的第一道锁（后来的 PatchGuard 在此之上再加巡查）。为什么老映像例外：兼容压倒安全——老驱动按"代码可写"的假设写成，强行只读会让它启动失败；安全与兼容的权衡点卡在 NT5 这个版本分界上。

#### ⑯ 完全初始化旗——修剪开闸

Phase 1 尾：`InterlockedDecrement(&MiTrimInProgressCount)`、`MiFullyInitialized = 1`、返回 TRUE。为什么修剪要挡到此时：工作集修剪（把不常用页换出）是平衡集管理器的日常活；初始化期 MM 还在接线（页表、池、节都在变），修剪会把正在建的页当成"不常用"换走——建到一半被换，等于拆自己的脚手架。`MiTrimInProgressCount` 就是这道闸：初始化期计数 > 0，修剪跳过；Phase 1 尾减到 0，`MiFullyInitialized` 置 1，修剪正式开闸。

#### ⑰ MmFreeLoaderBlock——MM 的 Phase 1 尾活（不在 MmInitSystem 内，但属 MM）

加载器内存的释放不在 `MmInitSystem` 里，而在 `MmFreeLoaderBlock`（`MM/MMINIT.C:2886`，头注释原话："This function is called as the last routine in phase 1 initialization. It frees memory used by the OsLoader."），由 `Phase1Initialization` 在 Phase 1 尾直接调（`INITOS.C:1621`，`KeLoaderBlock` 先置 NULL 在 `:1615`）。它干两件：

- **注册表页 pageable 化**（`:2972-3066` 一带）：遍历描述符，把 `LoaderRegistryData` 的页记下来（`MiBootRegistryRuns`，除非 `MmKeepBootRegistry` 另有要求）；逐页把 PTE 置入转换态（transition），PTE 起始地址存好，供以后释放——"make the page pageable and put the PTE into transition. If it is a PFN without a PTE then leave it as nonpaged until MmFreeBootRegistry is called."——意思是：内存 hive 的页先变成"可换出"，等 CM Part 2 把 hive 绑定到磁盘文件时（第 2 批 QA 第 18 条），`MmFreeBootRegistry` 再把它们彻底释放。为什么分两步：Phase 1 尾 hive 还没绑定文件（文件系统刚起来、SMSS 还没跑 Part 2），页不能直接 free（hive 数据还在用）；先 pageable 化是"预告可回收"，绑定完成后再真回收。
- **其余加载器内存进空闲链表**（`:3114` 一带："Walk through the memory descriptors and add pages to the free list in the PFN database"）："Remove PTE and insert into the free list"（`:3215` 一带）——加载器堆、栈等用完的页，拆 PTE、进空闲链表。**注意一个坑**（`:3185-3188` 的注释原话）："Note LoaderOsloaderHeap is used to describe page table pages used to map the registry so care must be taken not to blindly free those. Distinguish them via the share count check above."——`LoaderOsloaderHeap` 描述符里混着两样东西：真正的加载器堆页（可 free）和"映射注册表用的页表页"（不能 free，注册表映像还靠它们映射）；区分靠共享计数检查——共享计数高的（被多处引用）是页表页，留着；低的是堆页，free。盲目全 free 会把注册表映像的页表拆掉，hive 立刻不可访问。

> **本批对第 2 批的纠正（第 2 批笔记后台同步修正，对话里同步输出）**：
>
> 1. **MmInitSystem(2) 不释放加载器内存**：第 2 批地图（第 122 行）与正文（第 284 行）按 `MM/MMINIT.C` 头注释（`:346-355`）写了"Phase 2 还释放加载器占用的内存"——头注释是旧说法，与代码不符。Phase 2 的代码只有 `MiEnablePagingTheExecutive()`（`MMINIT.C:2392`）；真正释放加载器内存的是 `MmFreeLoaderBlock`，在 Phase 1 尾由 `INITOS.C:1621` 直接调（其头注释原话就是"called as the last routine in phase 1 initialization"）。第 2 批两处已改成正确说法。
> 2. **OsloaderHeap / Stack 的回收路径写错了**：第 2 批 QA 第 11 条写"OsloaderHeap/Stack → Phase 2 回收"——两处都不对。`LoaderOsloaderStack`（加载器栈）在 PFN 建库时（Phase 0，`INIT386.C` 的空闲链表分支）就直接进了空闲链表；`LoaderOsloaderHeap` 在建库时标在用（default 分支），在 `MmFreeLoaderBlock`（Phase 1 尾）里才 free，且其中映射注册表的页表页按共享计数排除、不 free。两处都不经过 Phase 2。第 2 批 QA 第 11 条已改成正确说法。

---

### Phase 2（`MMINIT.C:2391` 起）

#### ⑱ 执行体分页——只干一件事

```c
if (Phase == 2) {
    MiEnablePagingTheExecutive ();
    return TRUE;
}
```

Phase 2 的代码就是这一行（`MMINIT.C:2392`）。`MiEnablePagingTheExecutive`（定义在 `MM/MMINIT.C:4248`）的机制第 2 批已讲透（遍历 `PsLoadedModuleList`、读各映像 PE 节表、找 `PAGE` 节与 `.edata` 节、锁计数为 0 的段调 `MiEnablePagingOfDriverAtInit` 纳入系统工作集管理——只是"允许换出"，不是"立即换出"），本批不再重复，只补三点：

- **两道禁用闸**（`MM/MMINIT.C:4248` 起的函数头）：`MmDisablePagingExecutive & MM_SYSTEM_CODE_LOCKED_DOWN`（注册表 `DisablePagingExecutive`）→ 直接返回；无盘远程启动客户机且 CSC 初始化失败（`REMOTE_BOOT` 下 `IoRemoteBootClient && IoCscInitializationFailed`）→ 直接返回。为什么：用户明令不分页、或无盘启动时换出源（映像文件）本身在网络上不可靠，分页等于自杀。
- **为什么 Phase 2 在 IoInitSystem 之后**：第 2 批已钉死的两条——boot 驱动是 `IoInitSystem` 才加载进 `PsLoadedModuleList` 的，之前开，boot 驱动的 PAGE 节不在表里；换出后回读源头是映像文件本身，回读要文件系统 + 磁盘驱动路径，之前开，回读无路。
- **头注释与代码的出入（再强调一次）**：`MmInitSystem` 的头注释（`:346-355`）说"Phase 2 frees memory used by the OsLoader"，但 Phase 2 的代码没有释放动作；释放是 `MmFreeLoaderBlock` 在 Phase 1 尾干的。头注释是旧版本残留——以代码为准，不以注释为准。本批正文与第 2 批笔记均按代码修正。

---

## 第三部分：32 位 vs 64 位对照

| 分叉点 | 32 位 | 64 位 | 为什么只能这样 |
|---|---|---|---|
| 用户 / 系统分界 | `KSEG0_BASE`（0x80000000）对半分，2GB / 2GB；/3GB 时 `MmVirtualBias` 把分界推到 3GB，系统空间剩 1GB | `MI_HIGHEST_USER_ADDRESS` / `MI_SYSTEM_RANGE_START` 常量，用户空间巨大，系统空间不挤 | 32 位 VA 总共 4GB，分界是零和博弈，用户多拿系统就少拿；64 位 VA 多到分界位置不构成约束，索性用架构常量定死 |
| 池 / PTE / 缓存的权衡 | 处处争：池大了 PTE 小、缓存大了池小，注册表要定 PTE 上限、可疑驱动还要加 PTE | 不争：PTE 按 RAM 线性给（注释原话：Hydra NT64 上 4GB 内存给 128GB 的 PTE 空间），代价只是预建页表页占的物理内存 | 32 位系统空间是稀缺 VA，每块区都要精打细算；64 位 VA 不稀缺，稀缺的是物理内存——权衡的维度从 VA 换成了物理页 |
| 页表级数 | 两级（PDE / PTE，`_MI_PAGING_LEVELS < 3`），`MiBuildPagedPool` 里双映射系统页目录 | 四级（PXE / PPE / PDE / PTE，`_MI_PAGING_LEVELS >= 4`），逐级建表，不双映射 | 32 位 VA 浅，两级够用；双映射是两级下"按 VA 直达页目录项"的补丁。64 位 VA 深，必须四级；级数够，算术可达，不用双映射 |
| PFN 数据库的物理代价 | `MMPFN` 较小（指针 4 字节），数据库占内存相对少 | `MMPFN` 较大（指针 8 字节、PFN 号 64 位），同样内存规模下数据库占更多物理页 | 结构体里的指针与 PFN 号宽度跟随架构——64 位数据库天然更贵，这是 64 位"物理代价"的一部分（与 PTE 预建页表的物理代价同理） |
| 大页映射 | 支持（4MB 大页），但 PFN 数据库 + 非分页池的大页映射要求大页对齐的空闲区、全部 cached（防 TLB 属性冲突） | 支持（2MB 大页），同理；Phase 1 转换分支在两架构共用（`MiInitMachineDependent` 各有实现） | 大页是 CPU 特性，与位数无关；但"整段全 cached"是 TLB 的硬件要求——同一大页内属性不一，TLB 会乱，与位数无关 |
| 共享用户数据 | 有 x86 专属分支：`MmHighestUserAddress < MM_SHARED_USER_DATA_VA` 时手动把 PTE / PDE 的 Owner 置 1 对用户开放 | 无此分支：共享用户数据 VA 天然在用户空间 | 32 位某些布局下共享用户数据 VA 会落在系统空间一侧（用户顶被 /3GB 等因素推高），需要手动开放；64 位用户空间巨大，共享页 VA 永远在用户侧，不用动手 |

**为什么 64 位不能跑 32 位驱动（本批视角的补丁）**：第 1 批已讲二进制 ABI 全变（结构体布局、指针宽度、调用约定、IDT / IST、HAL 接口、IRQL 编号）；本批补一条 MM 视角的：PFN 数据库条目大小、页表级数、系统空间布局全变——驱动里任何"按 32 位算 PTE 地址、按 32 位算 PFN 条目"的代码在 64 位全错位。用户态有 WOW64 翻译，内核态没有翻译层，因为内核态跑在地址空间与页表的最底层，翻译成本就是重写 MM。

---

## 第四部分：单核 vs 多核对照

| 分叉点 | 单核（UP） | 多核（MP） | 为什么只能这样 |
|---|---|---|---|
| PFN 锁 | 照拿（IRQL 照抬），但没人抢 | 全系统一把锁，所有 CPU 共用同一批链表，核越多抢得越凶 | PFN 链表是全系统共享的，不是每核一份——取页、还页、换链表必须串行化。UP 下串行化没有代价（没人抢）；MP 下这把锁就是可扩展性瓶颈（WS03 时代的固有限制） |
| NUMA 节点 | 所有页归 0 号节点，`MiDetermineNode` 走同一条路 | 每页在建库时按物理地址定节点（`MiDetermineNode`），`KeNumaInitialize`（Phase 0，第 2 批）先建节点结构 | 节点是物理拓扑的反映——单节点机器没有"远近"之分，分节点无意义；多节点机器上页归属节点决定访问延迟，建库时就得定死 |
| 零页工人 | 不起 `MiStartZeroPageWorkers`；Phase 1 线程转任零页线程（第 2 批） | 起多个零页工人（`#ifndef NT_UP`），清零并行做 | 清零是纯 CPU 活、可并行；一核时一个清零工就是全部算力，多起工人没意义；多核时清零跟不上分配速度，工人必须多个 |
| 大页转换 | "全体处理器" = P0 一人，条件自然成立（只要 P0 支持且页数过阈值） | 必须全体 AP 都支持（Phase 1 分支，AP 已拉起后才判断） | 大页映射是 CPU 特性，某核不支持则该核访问大页映射会 fault——UP 下"全体"无歧义，MP 下必须等最慢的那核表态 |
| 系统页目录 | 全核共用 `MmSystemPageDirectory`（全局量） | 同样共用；各核当前进程的 CR3 各自不同（进程切换时换 CR3，第 1 批） | 系统空间是全系统一张图，不分核；用户空间是每进程一张图，随 CR3 换。系统页目录共用是"系统空间全局一致"的前提 |

---

## 第五部分：逆向 / 安全解读

- **PFN 数据库是取证金矿**：每页一张状态卡（PteAddress、ShareCount、ReferenceCount、OriginalPte、PageLocation）——内存取证时，PFN 数据库就是"物理内存全图"：哪页被谁映射、哪页被共享几次、哪页换出前长什么样，全能反推。坏页链表是硬件故障史（哪页物理损坏过），已改链表是"还没落盘的脏数据"（断电取证时这部分最值钱）。逆向时找隐藏页（比如 rootkit 藏的页）也可以从 PFN 数据库入手：有 PTE 映射但 PFN 条目异常的页、或 PFN 条目在用但无 PTE 的页，都是可疑点。
- **镜像写保护是内核自保的第一道锁**：Phase 1 尾代码页变只读后，驱动 bug 或攻击改写内核代码会立刻 fault，而不是悄悄成功。后来的 PatchGuard 在此之上再加"定时巡查代码完整性"，但第一道锁就是这里的写保护。缺口在老映像例外：NT5 之前的驱动不保护（兼容），老驱动的代码页仍可写——这是"兼容压倒安全"留下的攻击面（老驱动 + 写代码页 = 绕过写保护的已知路径）。
- **物理内存节是特权窗口**：`\Device\PhysicalMemory` 把整段物理空间装进节对象，走对象安全检查——打开它要过对象安全那一关（具体权限模型第 4 批 OB 点火展开）。逆向视角：拿到它的映射等于拿到整机内存读写，是提权链的终点之一；防守视角：它的 ACL 与打开权限是审计重点。
- **BBT / TEB 暴露是"设计如此的信息披露"**：烧掉的内存信息通过 TEB `ReservedForPerf` 给用户态读——敏感度低（只是"哪段内存被烧了"），不是漏洞；但取证时它能告诉你启动参数（/BURNMEMORY 烧了多少、哪段），是启动配置的旁证。
- **共享用户数据是"设计如此的共享"**：用户可读的系统页（时间、版本、特性位）不是泄漏，是 API（`GetTickCount`、`GetSystemTime` 直接读它）；但逆向时它也是"内核态时间源"的用户态观测点——时间类反调试（比如检测时间是否被加速）常读这页。

---

## 第六部分：易混点辨析

| # | 易混点 | 辨析 |
|---|---|---|
| 1 | "Phase 2 释放加载器内存" | 头注释旧说法，与代码不符。Phase 2 代码只有 `MiEnablePagingTheExecutive()`（`MMINIT.C:2392`）；释放是 `MmFreeLoaderBlock` 在 Phase 1 尾（`INITOS.C:1621`）。见正文 ⑰ 纠正框，第 2 批笔记已同步修正。 |
| 2 | BBTMemory 与 LoaderBad | 烧掉 ≠ 坏掉。`LoaderBBTMemory` 是 /BURNMEMORY 主动烧掉的内存：PFN 建库时跳过（不进链表），Phase 1 由 `MiMapBBTMemory` 映射出来给用户态看；`LoaderBad` 是硬件坏页：PFN 建库时进坏页链表、永久隔离、不计入可用页。烧掉的是"假装不存在"，坏掉的是"真的不能用"。 |
| 3 | "执行体分页" = 立即换出 | 不是。第 2 批已钉死：`MiEnablePagingTheExecutive` 只是把 PAGE 节的段"允许换出"（纳入系统工作集管理），换不换出由工作集管理器按需决定；锁计数非 0 的段跳过。 |
| 4 | "分页池建成" = 描述符初始化 | 不全对。分页池建成 = `MiBuildPagedPool` 建 VA 区 + PTE + 已提交页（`MMINIT.C:3301`）+ `InitializePool(PagedPool, 0L)` 初始化描述符（`:3656`）。前者是"地盘与页"，后者是"块管理"；两者合起来，头注释"paged pool may be allocated"才成立。 |
| 5 | 非分页池的"建成"函数 | 没有单一函数。非分页池 = VA 区（紧跟 PFN 数据库划，`INIT386.C:2113` / `:2235`）+ 描述符（架构初始化里 `InitializePool(NonPagedPool, 0)`，`INIT386.C:3263`）+ 扩建（slush 转入 + 按需从空闲链表取页）。找一个"建成"函数是找错了对象——它是一块区 + 一个描述符 + 一套扩建机制。 |
| 6 | PFN 数据库"建库" = "可用" | 不是。建库（描述符循环登记）只是第一步；可用还要页表（第 ⑤ 步）、锁（PFN 锁）、链表（空闲 / 零页 / 备用 / 已改 / 坏页）全就绪。Phase 0 的顺序就是"建库 → 页表 → 池"，缺一环，分配器都不能跑。 |
| 7 | 零页 = 空闲页 | 不是。零页链表的页已清零，分配时不用现清；空闲链表的页内容未知，分配时要清零（或走零页路径）。`MiRemoveZeroPage` 的取页顺序（零页 → 空闲 → 备用）就是按"分配代价"排的序。 |

---

## 第七部分：自查题

1. **PFN 数据库的内存从哪来、为什么不需要分配器？**（提示：VA 地址算术 + 加载器空闲描述符直登记 + PTE 地址算术 + 自描述收尾。）
2. **为什么分页池最后才建？**（提示：它要页表 + PFN 数据库 + 系统 PTE 三样齐全；分层建、每层只用已建好的层。）
3. **大页转换为什么等到 Phase 1？**（提示：Phase 0 只有 P0，"全体处理器是否支持"无从知道；AP 在 Phase 1 才拉起。）
4. **BBTMemory 与 LoaderBad 的处理路径有何不同？**（提示：烧掉的跳过建库、Phase 1 映射出来；坏掉的进坏页链表、永久隔离。）
5. **Phase 2 到底干什么？为什么头注释说"释放加载器内存"是旧说法？**（提示：代码只有 `MiEnablePagingTheExecutive()`；释放是 `MmFreeLoaderBlock` 在 Phase 1 尾干的，以代码为准。）

---

## 附录：未钉死清单

1. **`MiMemorySan` 的内部清单**：Phase 0 测算区开头调它（`MMINIT.C:1114`），前向声明在 `:100`，但定义不在 WRK 树内（全树搜不到实现）。从位置推断它在"算哪些内存类型计入、算总页数"之前对加载器内存描述符做清理/校验；内部清单树内无据，不编造。
2. **大页转换的页数阈值**：Phase 1 分支注释说"the number of physical pages is greater than the threshold"，但阈值变量本轮未定位到具体符号与数值；条件本身（全体支持 + 过阈值）已钉死，阈值数未钉死。
3. **BBT 缓冲区的用户态消费者**：树内只见 TEB `ReservedForPerf = BBTBuffer`（`MM/PROCSUP.C:5578`）；具体哪些用户态工具读它，树内无据，不展开。
4. **32 位 /3GB 下池 / PTE / 缓存尺寸的完整算术表**：机制与调整点已给（`:801`、`:853` 的池起始调整、`:3270-3295` 的 PTE 计数、`:3392-3415` 的分页池尺寸），但尺寸随 RAM 大小与注册表设置变化，无固定表；本批不造一张"通用表"充数。

---

*第 3 批交付门自检记录（交付前执行，见对话）：术语反查（正文逐词扫回术语表，沿用词已列名单）、行号复核（每条结论重查源码行号，不凭记忆）、推断标注（MiInitializeSystemSpaceMap 用途、MiMemorySan 作用、PFN 锁瓶颈等已标推断或列入未钉死）、前向引用核对（第 1 / 2 批既有内容、第 4 批路线规划均真实存在）、数字重算（64KB 护栏、2GB / 1GB 分界、4GB→128GB PTE 引文、2 倍池尺寸均为源码原值或引文，无自算数）。*

## 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，与正文互补。

- **缺页异常：fault 的转移，iret 回的是同一条指令**。CPU 取指或访存遇到页表项 P=0 → #PF（向量 14）→ IDT → `KiTrap0E`（`KE/I386/TRAP.ASM:5480`）：从 CR2 取故障虚拟地址 → 调 `MmAccessFault`（`MM/MMFAULT.C:51`）换入页（可能走 I/O 并等待，等待期间线程被换出，换入完成后再调度回来）→ 成功 → `KiExceptionExit` → iret 回到故障指令的地址，指令重执行。关键：缺页是 fault（可恢复），不是 trap——iret 回的是同一 EIP，不是下一条；页换入后指令第二次执行即成功。失败则走异常分发（访问违例），不再重执行。
- **Phase 0 → 1 → 2：普通函数调用，Phase 2 是开关转移**。`INITOS.C` 调 `MmInitSystem(0/1/2)` 都是函数调用；Phase 2 体内只有 `MiEnablePagingTheExecutive()`——它的转移意义是“执行体从常驻变可换页”：此后执行体页面可被换出，缺页路径（上一条）开始对执行体本身生效。不是跳转，是状态开关。
- **加载器内存释放：函数调用 + 指针失效**。Phase 1 尾 `MmFreeLoaderBlock`（`INITOS.C:1621`）是函数调用，释放后 LoaderBlock 指针不再可用——此后任何再读 LoaderBlock 的代码都是野指针访问，不是“交接完成”这种模糊说法。
- **池分配与页表操作：函数调用，不转移执行流**。本批其余机制（PFN 库建库、页表搭建、池初始化）全是函数调用链，没有中断、没有换栈、没有陷阱帧改写——特意写明，以免把“建库”误读成某种执行转移。
