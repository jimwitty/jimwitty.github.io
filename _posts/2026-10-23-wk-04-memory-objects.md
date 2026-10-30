---
layout: post
title: "Win 内核驱动 第 4 批：内存管理与内核对象"
categories: windows-kernel
series_order: 4
---

- **系列**：Windows 内核驱动循序渐进学习（教材：微软官方《内核模式驱动程序体系结构设计指南》，WDM 中心）
- **本批对应原文档**：「管理内核对象」（Managing Kernel Objects：Object Names / Object Directories / Life Cycle of an Object / Object Handles）+「Windows 驱动程序的内存管理」（Memory Management for Windows Drivers：Overview of Windows Memory Space / Allocating System-Space Memory / Map Registers / Mapping Bus-Relative Addresses / Using the Kernel Stack / Using Lookaside Lists / Making Drivers Pageable / Accessing Read-Only System Memory / Accessing User-Space Memory / No-Execute (NX) Nonpaged Pool / Section Objects and Views / Using MDLs），并交叉引用了指南内 PreviousMode、METHOD_NEITHER 缓冲校验等相关小节。
- **取材说明**：learn.microsoft.com 对抓取工具返回 403，正文取自微软官方同名设计指南 PDF（2019 版，GitHub 镜像 tpn/pdfs）对应章节逐节研读，并与 Learn 搜索索引摘录交叉核对（章节结构与摘录一致）。
- **学习目标**：学完本批，你应该能说清：① 内核对象、对象名、对象目录、句柄、引用计数是怎样一套"户籍制度"；② 系统地址空间里分页池/非分页池/内核栈/节视图各在哪里、各自的脾气；③ 分配内存时的第一性纪律——"谁会在什么 IRQL 碰这块内存"决定它必须住哪个池；④ 为什么访问用户态内存是驱动安全的头号雷区；⑤ MDL 到底在描述什么、为什么直接 I/O 离不开它。

---

## 一、本批地图

### 1. 系统地址空间布局（概念示意，非比例）

```
高地址 ┌─────────────────────────────────────────────────────┐
       │              系统空间（所有进程共享同一份）           │
       │  ┌───────────────────────────────────────────────┐  │
       │  │ 内核映像 ntoskrnl、HAL、已加载的驱动映像        │  │
       │  │   （默认非分页；标了 PAGE* 的节可被换出）        │  │
       │  ├───────────────────────────────────────────────┤  │
       │  │ 非分页池 NonPagedPool(Nx)                       │  │
       │  │   永远驻留物理内存，任何 IRQL 可访问             │  │
       │  │   —— 系统最稀缺的资源之一                       │  │
       │  ├───────────────────────────────────────────────┤  │
       │  │ 分页池 PagedPool                                │  │
       │  │   可被换出到分页文件，只能在低 IRQL 访问          │  │
       │  ├───────────────────────────────────────────────┤  │
       │  │ 系统缓存 / 映射视图（节对象 view 映射到这里）     │  │
       │  │ 内核栈（每线程约 3 页，可被换出，有条件）         │  │
       │  │ MmGetSystemAddressForMdlSafe 给出的系统映射     │  │
       │  └───────────────────────────────────────────────┘  │
       ├─────────────────────────────────────────────────────┤
       │              用户空间（每个进程一份，互不可见）        │
       │   用户代码/数据、堆、映射的文件视图（节）              │
       │   随时可被换出；非当前进程的用户地址根本不存在        │
低地址 └─────────────────────────────────────────────────────┘
              ↑ 虚拟地址背后是离散的物理页，
                页表（给 CPU/软件）与 MDL（给驱动/DMA）各记一份账
```

要点先行：**虚拟连续 ≠ 物理连续**；用户空间和分页池"随时可被换出"，非分页池"永远在"；这条区别就是后面所有纪律的源头。

### 2. 对象世界的关系图

```
对象目录（命名空间，如 \Device、\DosDevices、\KernelObjects）
        │  按名查找（名字是 Unicode 串，逐级相对命名）
        ▼
   内核对象（对象头 + 对象体）────────── 引用计数 ReferenceCount
        ▲   ▲                                归零 → 对象被释放
        │   │ 指针引用（内核内）              临时对象：句柄数归零 → 名字摘除
        │   └ ObReferenceObject / ObDereferenceObject 配对
        │
   句柄（进程句柄表 / 内核句柄表中的条目）── 句柄计数 HandleCount
        ▲
        │ 用户态只能经句柄间接访问对象，永远拿不到指针
   用户进程 / 驱动（ZwClose 关闭句柄，两个计数各减一）
```

---

## 二、核心要点

1. **对象是内核资源的统一户籍**：设备、文件、符号链接、注册表键、进程/线程、事件/互斥体等分发器对象、回调对象、节对象，全都是对象管理器（Ob）管辖的"对象"——一块由系统维护头部、由定义它的组件维护体部的数据。驱动只能通过组件导出的例程与对象打交道，不许直接改系统那份账，这就是第 1 批"基于对象 + 封装"的落点。
2. **名字只解决"找到"，句柄才解决"使用"**：命名对象住在对象目录构成的命名空间里（如 `\KernelObjects\LowMemoryCondition`）；无论用户态还是内核态，都是先凭名字打开句柄，之后一切操作都走句柄。无名对象用户态永远够不着，内核态可凭指针或句柄引用。
3. **对象有两本账：引用计数与句柄计数**：创建时引用计数为 1；每开一个句柄，两个计数各 +1；取指针引用的例程（ObReferenceObject、IoGetDeviceObjectPointer 等）只加引用计数。引用计数归零对象才被释放——**多加一次永不释放（泄漏），少加一次提前释放（UAF 土壤）**，配对纪律没有商量余地。
4. **临时对象 vs 永久对象**：绝大多数对象是临时的——句柄计数归零，名字即从命名空间摘除（指针引用还在则对象本体还活着）；创建时带 `OBJ_PERMANENT` 的是永久对象，对象管理器自己持一份引用，它"赖着不走"，要靠 ZwMakeTemporaryObject 才能改回临时。
5. **驱动私用句柄必须打 OBJ_KERNEL_HANDLE**：内核模式默认**绕过**对内核所建句柄的访问检查；不加这个属性的句柄会被塞进当前进程的句柄表，用户态可猜、可继承、可关闭——是现成的攻击面。`OBJ_FORCE_ACCESS_CHECK` 则相反：强制系统按用户态标准做全套访问检查，处理"替用户办事"的场景要用它。
6. **池就是内核的堆，分两种脾气**：分页池（PagedPool）可被换出、便宜些但只能在 IRQL ≤ APC_LEVEL 访问且不能进文件系统写路径；非分页池（NonPagedPool）永远驻留、任何 IRQL 可碰，但它是全系统稀缺资源。分配函数是 `ExAllocatePoolWithTag`，**返回值必须查 NULL**，失败返回 STATUS_INSUFFICIENT_RESOURCES。
7. **选池只问一个问题：这块内存会被谁在什么 IRQL 下访问**？只要有任何路径可能在 IRQL > APC_LEVEL（尤其 DISPATCH_LEVEL 及以上：ISR、DPC、持自旋锁时）碰它，就必须非分页——没有例外。反过来，只在 PASSIVE/APC 级访问的大块数据硬塞非分页池，是在浪费全系统的命根子资源。
8. **现代非分页池默认应该是 NX 的**：Win8 起有 `NonPagedPoolNx`（不可执行）与 `NonPagedPoolExecute` 两种池类型；老常量 `NonPagedPool` 的数值等于 Execute 版——不改代码的老驱动分到的其实是**可执行**内存。数据池可执行，等于给池溢出利用备好了落地页（详见解读）。
9. **内核栈只有约 3 页，是"过道"不是"仓库"**：大结构体、缓冲区一律走池分配；调用树要扁平、递归要封顶；栈耗尽是致命蓝屏。可以用 IoGetRemainingStackSize 探余量、KeExpandKernelStackAndCallout 临时扩栈，但那是急救不是常态。另一个暗坑：内核栈在线程等待时**可能被换出**，把栈上局部变量地址交给 DMA 或 DISPATCH_LEVEL 的例程，等于交出一个随时消失的地址。
10. **旁视列表是给"定长、高频、并发"分配准备的缓存层**：系统替你维护一小池可复用的定长缓冲，按需求自动增减，内置同步、多线程可无锁共用（初始化/删除仍需自己同步）；存储类驱动分配 SRB 是原例。结构头 `LOOKASIDE_LIST_EX` 本身必须住非分页内存；条目分页还是非分页，仍由"会在什么 IRQL 被访问"决定。
11. **驱动代码默认全部非分页，"可分页"是要挣来的**：节名以 `PAGE` 开头（如 `#pragma alloc_text(PAGE, ...)`）才会被 I/O 管理器做成可分页。红线是硬的：IRQL ≥ DISPATCH_LEVEL 运行的、拿自旋锁的、会碰分页池/用户缓冲的代码**绝不能**被换出——在高 IRQL 触发缺页直接蓝屏。用 `PAGED_CODE()` 宏 + Driver Verifier 的 Force IRQL Checking 来抓违规。串口这类"闲时没人用"的驱动可整盘分页（MmPageEntireDriver），但打开计数、断中断、SMP 同步全要自己管。
12. **只读内存保护在内核同样生效**：NT 4.0 之后内核只读页写保护是真的——往只读段写数据、给自己的代码打补丁做系统调用拦截、改一个指向常量的全局字符串，都会撞上 Bug Check 0xBE（ATTEMPTED_WRITE_TO_READONLY_MEMORY）。逆向时看到老式 hook 靠改代码实现，在现代系统上先想想它是怎么绕过这一关的。
13. **用户内存对驱动是"借来的、随时会变的"**：驱动只能在**发起请求的那个用户线程的上下文里**用它的用户虚拟地址，且只有最高层驱动能保证这一点（第 1、2 批"任意线程上下文"的回响）。所谓安全访问 = ProbeForRead/ProbeForWrite 探测 + try/except 包裹 + （要跨上下文/异步用就）MmProbeAndLockPages 把页锁死再说。PreviousMode 是系统判断"参数来自用户还是内核"的开关，Zw 前缀例程的本质就是把这个开关临时拨到 KernelMode。
14. **节对象是"可共享的内存段"的对象化**：由文件支持（内存映射文件）或分页文件支持（纯共享内存，进程间通信用）；进程看到的是映射进自己地址空间的**视图（view）**，同一节在不同进程里虚拟地址可以不同、物理页是同一批。视图按需调页，访问才真正分配/读入。安全要点全在"谁创建、用什么句柄、在谁的上下文映射"（详解节展开）——节也是第 3 批设备命名空间之外，驱动与用户态共享数据的正规通道。
15. **MDL 是"虚拟缓冲 ↔ 物理页布局"的说明书**：一段虚拟连续的缓冲，物理上散在不连续的页里；MDL = 结构体头 + 紧随其后的物理页帧号数组。I/O 管理器为直接 I/O 建 MDL 挂在 `Irp->MdlAddress` 上（可成链）；可分页内存的 MDL 数组只有在 MmProbeAndLockPages 锁页之后才有效，解锁即作废。驱动只许碰 `Next` 和 `MdlFlags` 两个成员，其余不透明，取值用 MmGetMdlByteCount 等宏。这是第 7 批直接 I/O 与第 9 批 DMA 的共同地基。

---

## 三、关键知识点详解

### 1. 内核对象与句柄

**对象名与对象目录。** 对象名是 Unicode 字符串，逐级相对父对象命名，每段以 `\` 开头。能容纳子对象的只有少数类型：对象目录（纯粹用来装命名对象的容器，如 `\Device` 装命名设备对象、`\DosDevices` 装 DOS 设备名的符号链接、`\KernelObjects` 装系统标准事件对象、`\Callbacks` 装回调对象）、磁盘设备对象（其子对象是文件）、目录文件对象等。注意对象目录只是对象管理器内部的虚拟目录，**和磁盘上的文件系统目录毫无对应关系**——文件系统目录本身反而是文件对象。文件路径如 `C:\Directory\File` 在对象世界写作 `\DosDevices\C:\Directory\File`，逐段正好是"目录对象 → 设备对象 → 文件对象 → 文件对象"的链。驱动可 ZwCreateDirectoryObject 自建对象目录。

**生命周期与两种计数。** 对象管理器为每个对象维护引用计数（ReferenceCount）与打开句柄计数（HandleCount）：

- 创建对象：引用计数 = 1；
- 每个打开/创建句柄的例程：两个计数同时 +1，关闭（ZwClose）时同时 −1；
- 每个返回对象指针的例程（如 ObReferenceObjectByHandle、ObReferenceObjectByPointer、IoGetAttachedDeviceReference、IoGetDeviceObjectPointer、ExCreateCallback、IoWMIOpenBlock）：只 +1 引用计数，用完必须 ObDereferenceObject 配对 −1；
- 引用计数归零 → 对象被释放。

临时对象的**名字**活得比对象短：句柄计数一归零，名字就从命名空间摘除（对象可凭指针继续存活）。永久对象（创建时在 OBJECT_ATTRIBUTES 里指定 OBJ_PERMANENT）由对象管理器自持引用，需要 ZwMakeTemporaryObject 才恢复"用完即删"。这个模型与 COM 的引用计数、C++ 的 shared_ptr 是同一思想，但代价完全不同：内核里计数错一次，轻则内存泄漏攒到系统耗竭，重则对象提前释放后被继续使用——UAF。

**句柄与 OBJECT_ATTRIBUTES。** 用户态永远只见句柄（HANDLE，不透明），句柄是进程句柄表（或系统句柄表）中的索引，条目里存对象指针与授予的访问掩码。设备对象与驱动对象是例外，它们不经句柄访问。驱动常用句柄的对象与入口例程大致是：文件（ZwCreateFile/ZwOpenFile）、注册表键（ZwCreateKey/ZwOpenKey）、线程（PsCreateSystemThread）、事件（IoCreateSynchronizationEvent 等）、符号链接、目录对象、节对象（ZwOpenSection）。初始化句柄时用 InitializeObjectAttributes 填 OBJECT_ATTRIBUTES，三个关键属性：

- `OBJ_KERNEL_HANDLE`：句柄只许内核访问，进内核句柄表。**驱动为自己创建的句柄必须带它**（见核心要点 5 与安全三规）；
- `OBJ_INHERIT`：子进程继承句柄副本；
- `OBJ_FORCE_ACCESS_CHECK`：强制做全部访问检查——因为默认规则是"内核模式创建的句柄绕过一切访问检查"，替用户办事时不用它，就等于用自己的满级权限替对方开门。

**与用户态共享句柄的安全三规**（原文给驱动的戒律，审计时反过来就是检查单）：① 只把内核创建的句柄传给用户态，**永远不要信任用户态传入的句柄**；② 必须替用户操作句柄时用 OBJ_FORCE_ACCESS_CHECK 核验权限；③ 对共享句柄用 ObReferenceObjectByPointer 额外持一份内核引用——否则用户把句柄一关，引用计数归零，驱动再用/再关这个句柄就是蓝屏。典型场景：用户经 IOCTL 传来事件句柄，驱动必须在**创建该事件的那个进程的上下文**里、用 ObReferenceObjectByHandle 把句柄换成指针并校验它确实是个事件对象，才能安全等待。

### 2. 池内存：分页 vs 非分页与选用纪律

系统空间的大块可分配内存来自两个池，都由 `ExAllocatePoolWithTag(PoolType, NumberOfBytes, Tag)` 分配（Tag 是四字符标签，专为抓泄漏而设，配合 PoolMon/调试器按标签统计——审计与排障利器）：

| | PagedPool（分页池） | NonPagedPool（非分页池） |
|---|---|---|
| 物理驻留 | 可被换出到分页文件 | 永远驻留 |
| 可访问 IRQL | 必须 ≤ APC_LEVEL | 任意 IRQL，含 DISPATCH_LEVEL 以上 |
| 其他限制 | 不得用于文件系统写路径 | 全系统稀缺、会随运行碎片化 |
| 典型用途 | 只在低 IRQL 碰的大块内部数据、注册表缓冲 | 设备扩展、IRP/上下文、DPC/ISR 要碰的一切、锁内数据 |

**分配经济性**（原文的省钱经，背后是池的实现方式）：非分页池中大于一页的分配按整页取，零头浪费且谁也用不了（Vista 起系统自动补足零头问题有所缓解）；小于一页的分配每块带池头开销。所以纪律是：长命的大缓冲在 DriverEntry 里一次备齐（非分页池越用越碎，晚了可能拿不到连续块），别用一串小额 ExAllocatePoolWithTag 反复切。设备扩展本身就是现成的非分页存储（第 3 批），驱动的"全局区"优先放那里。

另外几个专用分配器只点名定位：MmAllocateNonCachedMemory（非缓存非分页整页）、MmAllocateContiguousMemorySpecifyCache（物理连续内存，给老式 DMA）、AllocateCommonBuffer（DMA 公共缓冲，归第 9 批）。**所有分配返回值必须查 NULL**——内核里解引用空指针和用户态不同，是整机蓝屏而不是进程退出。

### 3. NX 与现代池类型

Win7 及以前，**非分页池的内存全部可执行**。这在利用时代是个巨大的便宜：池溢出/数据损坏只要能把执行流引到池里的数据上，数据即代码。Win8 引入 NX 非分页池后，POOL_TYPE 多了两个成员：

- `NonPagedPoolNx`：不可执行非分页池——**新驱动的默认选择**，绝大多数池内存只存数据，从不执行；
- `NonPagedPoolExecute`：可执行非分页池——只给真正要执行池中代码的少数派（如 JIT 生成代码）。

兼容的坑在常量数值上：`NonPagedPool`（老）与 `NonPagedPoolExecute`（新）**数值相同**。所以一个没做任何移植动作的老驱动在新系统上分到的仍是可执行池。移植机制即三件套：POOL_NX_OPTIN（单二进制，DriverEntry 里调 ExInitializeDriverRuntime(DrvRtPoolNxOptIn)，运行时按系统版本把默认池型拨成 Nx 或 Execute）、POOL_NX_OPTIN_AUTO（多二进制，编译期宏直接把 NonPagedPool 改写为 Nx）、POOL_NX_OPTOUT（个别源文件豁免）。还有个硬性兼容警告：Win8 以前的系统不认识 NonPagedPoolNx，目标含老系统的驱动不能无条件写死它。审计视角：一个现代安全驱动若还在大面积用可执行池分配数据，是明确的减分项。

### 4. 内核栈纪律

内核栈每线程只有约 3 页（平台/版本可变），且与用户栈不同，它服务的是"当前线程正在跑的内核代码全家"——你的驱动、你调用的系统例程、压在你下面的整条调用链共用这一份。

纪律四条：① **不在栈上放大家伙**：局部数组、大结构体一律改池分配——原文把"在栈上声明大结构"列为常见错误之首；② **调用树要扁平**：深层嵌套每层都吃栈，递归必须有硬上限；③ **探余量与扩栈是工具不是设计**：IoGetStackLimits / IoGetRemainingStackSize 可查，KeExpandKernelStackAndCallout 可把后续调用挪到一块更大的新栈上执行（文件系统等深调用链场景），但靠它续命说明设计已经出问题；④ **栈地址不可外传**：内核栈在线程进入用户模式等待时可能被换出（KeSetKernelStackSwapEnable 可临时禁用，但属调试/特殊手段）——把栈缓冲地址交给 DMA、或指望它在 DISPATCH_LEVEL 的异步回调里还有效，都是经典翻车点。栈耗尽的后果原文说得很干脆：fatal system error。

### 5. 旁视列表（Lookaside Lists）

场景画像：驱动要**频繁地分配/释放同一种定长结构**（如存储类驱动的 SRB、网络驱动的包描述符），且事先不知道峰值要多少——预分配浪费、现分配现释放又慢还碎。旁视列表就是系统托管的"该结构的缓存池"：

- 用 ExInitializeLookasideListEx 初始化（Vista+ 的统一接口，LOOKASIDE_LIST_EX 可装分页或非分页条目；老接口分 PAGED_LOOKASIDE_LIST / NPAGED_LOOKASIDE_LIST 两套），指定条目大小与池类型；Allocate/Free 回调传 NULL 时系统自动用 ExAllocatePoolWithTag/ExFreePool 补货；
- 系统**全局地动态调节**每个列表的存量：需求高涨就多囤，需求回落就把多余条目还给系统池——既快又不长占资源；
- **内置同步**：多个线程可并发从同一列表取/还条目而无需驱动加锁（但列表本身的初始化与 ExDeleteLookasideListEx 删除仍须驱动自己同步）；
- 纪律：条目必须定长（取到的永远是列表里那一种尺寸）；Allocate 回调不得擅自改 Tag、不得分页/非分页串味；LOOKASIDE_LIST_EX 结构头本身必须放非分页内存（它在取/还路径上随时被碰）；取与还要配对、尽快归还，归还越快下次命中缓存的概率越高。

与普通池分配的关系可以这样理解：旁视列表不是新内存来源，它是池之上的一层**驱动私有的对象缓存**，思想与用户态 slab/对象池完全一致。

### 6. 可分页驱动的代价与做法

**默认状态**：驱动映像的 .text/.data 节加载后全部非分页、常驻。**做法**：给代码/数据节起以 `PAGE` 开头的名字（`#pragma alloc_text(PAGE, Fn)`、data_seg("PAGEDATA") 等），I/O 管理器加载时据节名将其设为可分页；收益是闲置代码/数据被换出，给系统省常驻内存，对调制解调器、CD-ROM 这类偶发使用设备最划算，一般经验是可分页部分不到 4KB 不值当。

**不可分页清单**（满足任一即必须常驻或先锁入）：① 在 IRQL ≥ DISPATCH_LEVEL 运行的代码；② 使用自旋锁的代码（拿锁必升 IRQL 至 DISPATCH 级）；③ 调用 Wait=TRUE 的 KeReleaseMutex/KeReleaseSemaphore 等——这类调用返回时 IRQL 已在 DISPATCH_LEVEL。反过来说，运行在 < DISPATCH_LEVEL 的代码若要碰可分页的东西（分页池、别的分页例程、未锁定的用户缓冲），必须保证那一刻不会缺页出事。

**检测与隔离**：在声称可分页的例程首行放 `PAGED_CODE()`（调试版下若在 ≥ DISPATCH_LEVEL 调用会当场抱怨）；把碰自旋锁的几行代码拆去独立的非分页例程，"隔离"往往只省几百字节但性质完全不同；最后用 Driver Verifier 的 **Force IRQL Checking** 验收——它在驱动每次升 IRQL 时把其可分页代码全部换出，违规当场现形。这类 bug 若流到用户手里，特征是"偶发、难复现"，因为它依赖缺页时机。

**按需锁定**：可分页节可以临时"钉"回内存：MmLockPagableCodeSection / MmLockPagableDataSection（返回句柄、带引用计数，重复锁只增计数）、MmLockPagableSectionByHandle（同一节多处锁定用的高效版）、MmUnlockPagableImageSection 解锁；锁一整节是重操作，不要频繁调用。串口/并口驱动是范式：端口没打开时代码可换出，打开期间把服务代码锁住。

**整盘分页**：MmPageEntireDriver 把驱动**全部节**强制可分页、MmResetDriverPaging 恢复编译期属性。能整盘分页的前提是设备不用时不产生中断；流程纪律是自维护打开句柄计数，归零→断中断→整盘换出，再次打开→换入→MmResetDriverPaging→才许连中断。计数增减与分页状态切换之间的 SMP 同步是驱动自己的责任——一边在换出、另一边 CPU 在打开设备连中断，是原文点名的竞态。win32k.sys、串口、beep、null 驱动都是整盘分页的实例。别忘了 INIT 节：初始化代码本来就是"用完即弃"的可丢弃节。

### 7. 访问用户空间内存的规则

这是本批安全含金量最高的一节。先把地基说清：**用户虚拟地址只在"地址所属进程是当前进程"时有意义**（非当前进程的用户空间压根不存在），而驱动例程的线程上下文是"借来的"（第 1 批）。原文规则：

- 只有**最高层驱动**（如 FSD）能确信自己的 Dispatch 例程运行在发起 I/O 的用户线程上下文里，从而可以直接使用该线程的用户地址——也仍须探测与异常保护；
- 中低层驱动**不得**直接碰 IRP 里可能出现的用户地址；正确姿势是靠 I/O 管理器（缓冲 I/O 给系统缓冲、直接 I/O 给 MDL）或最高层驱动先把用户缓冲锁好再下传；
- 最高层驱动要替下层准备时，用 MmProbeAndLockPages 把用户缓冲的页锁入物理内存并生成 MDL，请求走完再解锁。

**Neither 方式（METHOD_NEITHER）把全部责任推给驱动**（与 IOCTL 的关系第 7 批展开，这里先记校验三件套，原文在"Errors in Referencing User-Space Addresses"里给了反例代码）：I/O 管理器对 METHOD_NEITHER 的 IOCTL/FSCTL **不做任何校验**，连缓冲里内嵌的指针也不查。驱动必须：① 在请求线程上下文中用 ProbeForRead/ProbeForWrite 校验地址范围与读写权限；② 所有解引用包在 try/except 里——因为探测之后用户线程仍可改权限、解映射，探测不是一劳永逸的保险，缺了异常处理就是 TOCTOU 蓝屏/利用面；③ 或者干脆自建 MDL 锁页、或自己双缓冲。原文反例正是审计名场面：把 Type3InputBuffer 当可信指针直接 `*EntryPoint = ...` 写——用户传什么地址，内核就往什么地址写，这就是教科书级的任意地址写提权原语。

**PreviousMode 与 Zw/Nt 的分工**：线程对象里有个 PreviousMode 字段：用户线程经系统调用陷入内核时被置为 UserMode，系统服务据此决定"参数要按不可信处理"（校验句柄归属、ProbeFor 缓冲）。驱动借任意线程运行，该字段可能残留 UserMode：此时驱动若直接调 NtXxx 例程并传内核句柄/内核缓冲，会被当作用户参数——句柄查无此物（还泄漏了内核句柄）、内核缓冲被 ProbeFor 直接抛异常。解法：驱动调 **Zw 版**例程——它是一层薄封装，调用期间把 PreviousMode 强制拨为 KernelMode、返回时恢复，以此声明"参数来自可信内核源"。而处理来源不定的请求时也可直调 Nt 版、让检查按实际来源走。ExGetPreviousMode 可读当前值，IRP 里的 RequestorMode 字段是请求发出时该值的副本。

顺带收下只读内存一节（与第 5 条核心要点呼应）：内核只读页（含驱动自身代码节、常量字符串所在节）受写保护，违规写触发 0xBE 蓝屏；老式"改自己代码拦截系统调用"的手法在此直接撞墙；要改的全局字符串必须声明为字符数组而非指向常量的指针，否则链接器可能把它放进只读节。

### 8. 节对象与视图

**概念**：节对象表示"一段可共享的内存"；视图是节被映射进某个进程地址空间的那部分——节是仓库，视图是各进程自家开的窗。各进程的视图虚拟地址互不相关，物理页同一批。视图映射时不预分配物理页，首次访问缺页时才调入（文件支持的从文件读、页文件支持的给新页）。

**两种支持**：

- **文件支持节（file-backed）**：就是内存映射文件。读视图 = 透读文件，写视图（PAGE_READWRITE）= 透写回文件；PAGE_WRITECOPY 则是写时复制——改动只在私有副本里，视图撤了就丢。视图本身不占分页文件。
- **页文件支持节（page-file-backed）**：ZwCreateSection 时 FileHandle 传 NULL。改动只活在分页文件里，节销毁即蒸发——进程间共享内存的标准做法。

驱动侧入口是 ZwCreateSection / ZwOpenSection 拿句柄，ZwMapViewOfSection 指定偏移（SectionOffset）、大小（ViewSize）与保护（Protect）映射，用完 ZwUnmapViewOfSection、ZwClose。小纪律：映射完且不再映射新视图时立刻 ZwClose 句柄是推荐做法——视图会替节对象续命，早关句柄少一类泄漏。

**安全协议**（原文分三档，条条是审计点）：

1. **不与用户共享的节**：句柄必须是内核句柄（系统进程中创建或 OBJ_KERNEL_HANDLE）；视图只能从系统线程映射——否则视图会落进"碰巧的当前进程"里，平白把内核数据开给某个用户进程看；
2. **与用户进程共享视图**：节必须由**驱动创建**（绝不能用用户传来的节句柄去映射）；视图在共享进程的线程上下文里映射（最高层驱动在 Dispatch 里做，下层驱动上下文任意、做不了这件事）；驱动对视图的**每一次访问都包 try-except**（恶意应用可随时解映射/改保护）；并且必须校验视图内容——不能假设只有可信组件能碰这段共享内存（用户完全可以躺在共享页上等驱动来读，经典的双映射/竞态利用面）；
3. **让用户应用自己建视图**：驱动先 ObReferenceObjectByHandle 自持一份节对象引用再把句柄交出去，防止用户关句柄把对象抽掉；用完 ObDereferenceObject 释放。

附带一条历史注脚：WS2003 SP1 之后只有内核驱动能打开 `\Device\PhysicalMemory`（它本身就是个节对象），驱动把它的句柄递给用户应用等于递出整机物理内存——只许给可信应用。

### 9. MDL 入门

**MDL 描述什么**：一个 I/O 缓冲在虚拟地址上连续，但其物理页是散的。MDL（Memory Descriptor List）就是这个布局的记录：一个 MDL 结构体，后跟一个数组，逐项写下缓冲各页的物理页帧号，外加缓冲的起始虚拟地址、页内偏移、字节数。软件世界靠页表做虚→实翻译，MDL 相当于把某段缓冲的翻译结果**摘抄成一张清单**，好让不能查页表的角色（设备、其他驱动）也能找到数据。

**生命周期与纪律**：

- 来源两路：I/O 管理器为直接 I/O 请求自动建 MDL 并挂在 `Irp->MdlAddress`；驱动也可 IoAllocateMdl 自建（或 MmInitializeMdl 就地格式化一块非分页内存）。一个 IRP 可挂多个 MDL 组成**链**，靠 Next 成员串联，系统在 IRP 完成时统一解锁、释放挂在 IRP 上的 MDL（但驱动自己发出的请求要自己收尾）；
- IoAllocateMdl 只分配了"表格"，没填内容：描述非分页池缓冲用 MmBuildMdlForNonPagedPool 填表；描述**可分页**内存则必须 MmProbeAndLockPages——它先按指定访问方式（IoReadAccess/IoWriteAccess/IoModifyAccess，只能按探测时声明的方式用）校验并把页锁入物理内存，然后才填表；MmUnlockPages 一解锁，表立即作废；
- MDL 结构是**半不透明**的：驱动只许直接碰 Next 与 MdlFlags；读信息用宏——MmGetMdlVirtualAddress（起始虚拟地址）、MmGetMdlByteCount（字节数）、MmGetMdlByteOffset（首页内偏移）；
- **双重映射**入口：MmGetSystemAddressForMdlSafe 把 MDL 描述的页映射到系统地址空间，给驱动一个随时可用的内核虚拟地址——这是"原始用户地址只能在原上下文用、且随时会消失"问题的标准解。部分 MDL 用 IoBuildPartialMdl 从源 MDL 切一段出来（下层驱动处理大请求的一部分时用），原文特别警告：给 IoBuildPartialMdl 推导偏移时必须用源 MDL 的 MmGetMdlVirtualAddress 而不是系统映射地址，混用会数据损坏或蓝屏；
- 与 DMA 的关系只点一句：设备不认虚拟地址，DMA 时把 MDL 的页清单翻译成设备能用的逻辑地址，正是靠映射寄存器（map registers）——每个映射寄存器对应最多一页，与 MDL 的每个条目一一呼应；适配器对象怎么用留到第 9 批。

### 10. 映射寄存器与总线地址映射（定位）

本批只记概念坐标：系统里并存**三种地址空间**——虚拟地址（CPU/软件）、物理地址（RAM）、逻辑/总线地址（设备眼里）。驱动在 START_DEVICE 时从 PnP 管理器拿到成对的原始（总线相对）与已翻译资源列表，应各存一份到设备扩展；翻译后类型为 CmResourceTypeMemory 的资源必须 MmMapIoSpace 把物理地址映成可访问的虚拟地址才能摸寄存器，STOP/REMOVE 时 MmUnmapIoSpace 对称释放；端口类资源则用翻译地址的低 32 位配合 READ_PORT_XXX / READ_REGISTER_XXX 等 HAL 例程访问。细节（资源仲裁、IRP 流程）归 PnP 批与 DMA 批。

---

## 四、解读（结合逆向与安全背景）

**1. 池是内核利用的"主战场"，本批的每条纪律都对应一类漏洞。** 非分页池里躺着全系统的关键结构（对象、IRP、设备扩展……）且彼此相邻：一个定长结构的池溢出，下一个被踩的往往就是隔壁对象的头部——第 3 批学的对象头（类型、引用计数）被改，类型混淆与引用计数破坏随之而来；Use-after-free 在池里更阴险，释放的块立刻被下一个分配者复用成完全不同的结构。NX 池的意义正在于切断利用链的最后一环：就算劫持了数据指针、把 shellcode 塞进池缓冲，NonPagedPoolNx 让这块地不可执行，攻击者必须再去找可执行内存或转 ROP。审计一个驱动时，"池类型选错"（本该 Nx 的用了 Execute、本该非分页的用了分页池导致高 IRQL 缺页）是最容易落笔的两类发现。

**2. 用户内存校验缺失 = IOCTL 提权的经典配方。** 把本批三件事叠起来看：METHOD_NEITHER 让 I/O 管理器完全撒手（第 2 批预告过）、Type3InputBuffer 是用户完全可控的指针、驱动若漏了 ProbeFor + try/except 就直接解引用——任意地址读/写原语当场成型。再叠上 PreviousMode 的坑：驱动在任意线程上下文里用残留的 UserMode 调 Nt 版例程，轻则功能失效，重则校验被绕过或被反向利用。所以审一个陌生驱动的 DeviceControl，先看三样：TransferType 是什么、用户指针探测了没有、访问包异常处理了没有——这是第 7 批 IOCTL 深挖的先行检查单。

**3. MDL 的"双重映射"是设计出来的风险口。** MmGetSystemAddressForMdlSafe 把同一批物理页再映一份到系统空间：原缓冲若是用户页，用户进程仍持有自己的映射且能随时改内容、改保护、甚至解映射——驱动在系统映射上读到的数据可以与校验时不同（double-fetch），MmProbeAndLockPages 锁住的只是"页不被换出"，锁不住"内容不被改"。历史上围绕 MDL 的滥用（重复映射、partial MDL 偏移算错）多出于此。第 7、9 批会看到：正确的姿势是把需要信任的数据先拷进驱动自己的池缓冲再校验使用，而不是反复读共享页。

**4. 引用计数与句柄是审计的"资产负债表"。** 逆向时看到 ObReferenceObjectByHandle 之后没有配对的 ObDereferenceObject，或事件/节句柄在错误路径上漏 ZwClose，短期是泄漏（句柄表、非分页池缓慢膨胀，PoolMon 按 Tag 一抓一个准），长期在共享句柄场景里就是对象生命周期被用户态操纵的 UAF。反过来，第 3 批的设备对象、文件对象在本批的对象账本下才算闭环：`!object`、`!handle` 在 WinDbg 里看到的引用/句柄计数，正是本批两本账的实景。还有内核句柄属性这一条：审计中发现驱动创建的句柄没打 OBJ_KERNEL_HANDLE，或者把句柄经 IOCTL 回吐给用户态而没另持引用，基本可以直接写成发现项。

**5. 可分页性是把"性能/内存优化"做成"可用性漏洞"的开关。** 把本该常驻的代码标成 PAGE、或在持自旋锁路径上调用分页例程，平时不犯、压力一来缺页时机对上就是蓝屏——这类问题在竞品/恶意场景下可被刻意构造负载触发。Force IRQL Checking 因此在驱动质量门禁里地位很高；逆向时在 IDA 里看到驱动映像的 PAGE 节里躺着 ISR/DPC 路径函数，也是一眼可见的红旗。

---

## 五、易混点辨析

1. **引用计数 vs 句柄计数**：前者数"指针引用 + 句柄引用"的总和，归零对象才死；后者只数打开的句柄，归零只摘名字（临时对象）。把句柄全关了对象还活着（有人持指针引用），是正常现象而非泄漏。
2. **PagedPool 能不能分页 ≠ 代码能不能分页**：前者是数据池的可换出属性，后者是驱动映像节的属性；共同红线是 IRQL——访问分页池、执行分页代码，本质都要求当前不能处于"一缺页就蓝屏"的高 IRQL。
3. **NonPagedPool 与 NonPagedPoolNx 不是一回事**：老常量 NonPagedPool 数值等于 NonPagedPoolExecute（可执行）。"用了非分页池"不等于"用了 NX 池"，尤其在审计老驱动时。
4. **锁页 ≠ 校验**：MmProbeAndLockPages 同时做探测（probe）和锁页（lock），但探测通过只说明"这一瞬间地址范围合法且已驻留"；用户对已锁定页内容的并发修改防不住，数据可信度问题要靠拷贝 + 再校验解决，别把锁页当成消毒。
5. **节（section） vs 内存节/段（section of image）**：同名两物——节对象是可共享内存的对象化实体（ZwCreateSection）；驱动映像的"节"是 PE 段（.text、PAGE 节）。前者在对象目录里，后者在加载映像里，语境完全不同。
6. **Zw 版 vs Nt 版例程**：不是"一个给驱动一个给应用"——区别只在 PreviousMode 的处理：Zw 版在调用期强制 KernelMode 再转调 Nt 版实现，Nt 版按当前线程的 PreviousMode 如实检查。在任意线程上下文里拿不准时，驱动应选 Zw 版。

## 六、自查问题（只给问题，自己回笔记里找答案）

1. 一个对象被用户态打开了 2 个句柄、又被某驱动用 IoGetDeviceObjectPointer 取了一次指针引用，此时它的引用计数与句柄计数分别是多少？两个句柄都关闭后，对象会立即被释放吗？为什么？
2. 某驱动在 DPC 例程里要读一段在 Dispatch 例程中分配的数据，这段数据应该从哪个池分配？如果分配时写的是 `NonPagedPool`，在 Win8+ 上它实际落进哪种池、可执行吗？要改成不可执行应该怎么写或怎么移植？
3. 驱动想把一个 200 字节的固定结构放在函数局部变量里传给内部例程，为什么这在本批是被点名的坏习惯？栈上地址为什么不能留给稍后运行的 DPC 使用？
4. 某 IOCTL 是 METHOD_NEITHER，用户传入的 Type3InputBuffer 指向一个输出地址。驱动在解引用它之前必须做哪三件事？少做任何一件分别会出什么问题？
5. 驱动要给用户态应用共享一大块数据用节对象实现：节由谁创建、句柄要带什么属性、驱动每次读共享页时要防哪两件事？另外，若这块节改为由分页文件支持而不是文件支持，数据的"寿命"有什么不同？
