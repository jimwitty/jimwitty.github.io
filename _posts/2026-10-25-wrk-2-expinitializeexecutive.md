---
layout: post
title: "WRK 第 2 批：ExpInitializeExecutive 点火总控（Phase 0 / Phase 1）"
categories: WRK
series_order: 3
---

> **取材说明**：底本为 WRK 源码树 `~/workspace/WRK/WRK/WRK-V1.2`（WS03 SP1 / NT 5.2 代码线）中 `BASE/NTOS/INIT/INITOS.C`（2105 行）全文通读；对位子系统只读到"总控级"（调用顺序与依赖关系），各子系统内部展开留给第 3–7 批。HAL 本体不在 WRK 包内（只有预编译 DLL），凡涉 HAL 内部处均按调用点反读并注明。tpn/pdfs 是第三方文档收集仓库、非微软官方；本批不依赖它。
>
> **写作规范（2026-10-25 用户定稿，后续各批同此标准）**：术语表前置（定义 + 做什么用 + 为什么要有）；每个机制、每次判定、正文里讲清为什么非如此不可（含历史与兼容因素）；32 位与 64 位、单核与多核分叉处讲清为什么只能 / 可以这样；每条机制结论以复核过的源码行号为准，树里查不到的列入文末"未钉死清单"，不编造。
>
> **本批一句话地图**：`ExpInitializeExecutive` 是全系统的"点火顺序表"——它被调两次：Phase 0（P0 在 `KiInitializeKernel` 里调，中断关、单核、只能建静态数据）与 Phase 1（Phase 1 线程里调，调度器已跑、系统完全可运行）。Phase 0 按依赖顺序把各子系统的"静态部分"立起来（HAL → 资源 → NUMA → MM → OB → SE → PS → PP），期间创建 Phase 1 线程；Phase 1 线程提权到最高档附近防抢占，按顺序拉起 AP、做各子系统的"动态部分"、加载 boot 驱动、最后创建 SMSS（第一个用户进程），然后转任零页线程。

---

## 第一部分：术语表（先读这部分，正文不再重复定义）

> 第 1 批术语表已定义的词（PCR/PRCB、IRQL、DPC、APC、IPI、IDT/GDT/TSS、HAL、LoaderBlock、线程/进程对象等）本批直接沿用，不再重复；下面只定义本批新出现的词（自旋锁本批已补完整条目，见第一节）。

### 一、执行体、内核、HAL 的分层

- **Executive（执行体）**：NTOS 里"操作系统服务"的那一层——对象管理器、内存管理器、安全、进程管理、I/O、PnP、电源、缓存等子系统的总称。`ExpInitializeExecutive` 的 "Exp" 就是 Executive 的缩写。为什么要有这一层：内核（Kernel，KE）只管最底层（调度、中断、同步原语），执行体在它之上提供"操作系统"的功能；分层后，执行体的初始化可以按依赖排成一张顺序表。
- **Kernel（KE）**：最底层——调度器、DPC/APC、中断分发、自旋锁。为什么执行体与内核要分开初始化：内核的点火（第 1 批）是"让 CPU 能跑"，执行体的点火是"让操作系统能用"；前者是后者的前提。**DispatcherReadyListHead**：按优先级索引的就绪队列表头数组（数组维度即优先级档数，第 7 步推断 MAXIMUM_PRIORITY 的依据）。
- **自旋锁（Spinlock）**：内核最底层的互斥原语——保护一段临界区，同一时刻只许一个核进入。为什么要有：多核共享数据（就绪队列、PFN 数据库、引用计数）必须互斥，而互斥的第一性原理是"读-改-写必须不可分"，这一点只有硬件能保证。机理（硬件与软件的分工）：硬件提供原子 RMW 指令（`lock bts` / `lock xchg` / `lock cmpxchg`）+ 缓存一致性（MESI）+ 总线/缓存行锁定；软件搭出抢-等-放循环：抢 = `lock bts` 原子测试置位（CF 装旧值，CF=0 抢到、CF=1 去自旋）；等 = 普通读自旋（TTAS）+ `pause` 提示；放 = 写 0。软件另搭三样：IRQL 抬升到 DISPATCH_LEVEL（持锁不能被 DPC/调度打断）、排队公平性（queued spinlock）、UP 退化（单核下获取直接 return，只剩 IRQL）。为什么只能短持、不能等待：DISPATCH_LEVEL 上调度器是关着的，持锁阻塞等于死锁；且等待的核在空转烧 CPU，临界区越长烧得越多——长临界区必须用 ERESOURCE 那种可阻塞的锁。锁字（KSPIN_LOCK）就是一块内存地址里的一个字（32 位 4 字节、64 位 8 字节），必须在非分页内存里，所有核用同一个地址操作同一个字，靠缓存一致性看到同一个值。
- **HAL（硬件抽象层）**：挡主板、芯片组差异的那层。为什么 HAL 也要分两次初始化：Phase 0 的 HAL 初始化只做到"中断控制器能用"（否则 Phase 0 连中断都开不了）；Phase 1 的 HAL 初始化才做全功能（DMA、定时器等平台设备）。（按调用点反读并注明：HAL 本体不在树内，两阶段的内部清单见未钉死清单第 6 条；"中断控制器能用"是从"HalInitSystem(0) 之后就 _enable()"反推的，"DMA、定时器"是平台设备的举例、非树内清单。）

### 二、Phase 0 / Phase 1 与 InitializationPhase

- **Phase 0（第 0 阶段）**：`ExpInitializeExecutive` 第一次被调时的阶段。约束（函数头注释原话）："the only activity that may be performed is the initialization of subcomponent specific data"（只能初始化本子系统的数据），且"performed in the context of the kernel start up routine with interrupts disabled"（在内核启动例程的上下文里、中断关的状态下跑）。为什么约束这么严：此时调度器还没跑（没有线程可切）、分页刚上线、很多结构还是静态的——任何"等待、创建线程、做 I/O"的动作都没有支撑它的机制。
- **Phase 1（第 1 阶段）**：`ExpInitializeExecutive` 第二次被调时的阶段，跑在 Phase 1 线程里。注释原话："the system is fully operational and subcomponents may do any initialization that is necessary"（系统完全可运行，子系统可以做任何必要的初始化）。为什么 Phase 1 就"完全可运行"：调度器已跑（第 1 批尾巴开闸）、MM 已上线（Phase 0 的 `MmInitSystem(0)`）、中断已开——等待、分配、建线程全有了支撑。
- **InitializationPhase**：全局变量，记录当前阶段（Phase 0 里 P0 把它置 0，Phase 1 线程把它置 1，SMSS 建好后加 1 成 2）。为什么要有全局阶段变量：同名 Init 函数要被调两次，函数内部靠它区分"这次是第几阶段、该干哪部分活"；AP 支线调 `HalInitSystem` 时传的也是它（AP 在 Phase 1 期间被拉起，传的就是 1）。
- **"两次调用"约定**：Ob、Ex、Ke、Kd、Se、Po、Pp、Ps 的 Init 函数在 Phase 0 调一次、Phase 1 再调一次；但这张表有两个例外：Mm 调三次（Phase 0 一次、Phase 1 两次：Mm(1) 与 Mm(2)）；Cm 不是"调两次"——CmInitSystem1 只在 Phase 1 调一次，Phase 0 只有 CmpInitSystemVersion 检查点（0、1），Phase 1 另有一串检查点（2、8、3、9、4、6、7）。为什么不是一个函数干完：Phase 0 只能建静态数据，Phase 1 才能做动态的活——同一个子系统的初始化天然劈成两半，劈不开的就没法在启动期活下来。

### 三、点火失败的死因编码（BugCheck 码）

- **PHASE0_INITIALIZATION_FAILED / PHASE1_INITIALIZATION_FAILED**：Phase 0 / Phase 1 里某个子系统初始化失败时的通用死因。为什么要有"阶段 + 通用码"：启动期没有日志系统可写，蓝屏码就是唯一的"死在哪一步"的坐标；逆向时按码先定位到阶段，再按调用顺序缩小到具体子系统。
- **各子系统专属死因**：OBJECT_INITIALIZATION_FAILED（OB）、SECURITY_INITIALIZATION_FAILED（SE）、PROCESS_INITIALIZATION_FAILED（PS）、PP0/PP1_INITIALIZATION_FAILED（PnP 的 Phase 0/1）、HAL/HAL1_INITIALIZATION_FAILED（HAL 的 Phase 0/1，注意 HAL1 的 "1" 即 Phase 1）、MEMORY1_INITIALIZATION_FAILED（MM 的 Phase 1）、CACHE_INITIALIZATION_FAILED、CONFIG_INITIALIZATION_FAILED、FILE_INITIALIZATION_FAILED、LPC_INITIALIZATION_FAILED、IO1_INITIALIZATION_FAILED、MISMATCHED_HAL（加载器 Extension 版本不匹配）、UP_DRIVER_ON_MP_SYSTEM（MP 系统上用了 UP 驱动/HAL）、SESSION3/4/5_INITIALIZATION_FAILED（SMSS 创建三阶段的死因，见正文第 9 步）。**读法**：码名里的阶段号（0/1）与子系统名，就是点火顺序表里的坐标。

### 四、启动期杂项

- **NLS（国家语言支持表）**：ANSI 代码页、OEM 代码页、Unicode 大小写转换三张表。为什么启动期就要它：字符串转换（大小写不敏感比较、ANSI ↔ Unicode）是对象名、注册表路径、文件路径的前提——没有它，连名字都比不了。为什么它要初始化两次：池建好前只能借加载器内存里的表，池建好后拷到池里、把加载器内存还回去（正文第 4 步）。
- **BURNMEMORY / PERFMEM（烧内存）**：启动选项，指定"扣掉多少内存不用"。为什么要有：测试缩水内存配置——不拔内存条、靠启动选项模拟小内存机器，看系统在内存紧张时是否还稳。为什么必须在 MM 上线前扣：扣掉的内存要从加载器的内存描述符里划走，MM 建 PFN 数据库时就看不到它们；MM 建好后再扣就晚了。
- **BBT**：PERFMEM 选项扣下来、专供性能测试（BBT 模式）用的那段内存——`LoaderBBTMemory` 就是这类内存的描述符类型标记（全称未在树内展开，不编造）。**LoaderBad**：标记为坏/不可用的内存描述符类型——烧内存时把扣掉的页标成此类，MM 建 PFN 数据库时跳过它们（`INITOS.C:405` 调用点）。
- **Textmode setup（文本模式安装）**：Windows 安装程序的第一阶段（蓝底白字的文本界面）。为什么点火要先判定它：安装模式下的 I/O 路径与正常启动不同（安装盘、目标盘的角色不同），后续初始化要分叉。
- **Remote boot（远程启动）**：从网络启动的客户机（无盘工作站）。为什么要判定：启动设备是网络（`net(0)`），I/O 路径走网络协议栈而非本地磁盘。
- **Inbv / Boot Video（启动视频驱动）**：启动期显示进度条的驱动（`InbvDriverInitialize`）。为什么要有：从内核接管显示到图形子系统起来之间有很长一段"黑屏期"，进度条是唯一的用户可见反馈——"系统在动"本身就是一种可用性。
- **WinPE 模式**：Windows 预安装环境（内存盘或 U 盘启动的最小系统）。为什么要判定：WinPE 的 I/O 路径（内存盘启动 vs 常规启动）与正常系统不同，初始化要分叉（`INIT_WINPEMODE_INRAM` / `INIT_WINPEMODE_REGULAR`）。
- **RTC / CMOS 时间**：主板上的实时时钟芯片存的时间。为什么系统时间要从它读：启动期没有网络时间、没有用户设置，RTC 是唯一的时间源。**/YEAR 开关**：启动选项，强制指定年份——给"时钟硬件坏了、年份不对"的机器用的 workaround（注释原话：硬件时钟坏了时强制年份）。
- **SharedUserData（KUSER_SHARED_DATA）**：内核与用户态共享的一页内存。为什么要有：版本号、tick 乘数、时区偏置这类"用户态经常要读、内核偶尔才写"的数据，放共享页比每次系统调用快得多。代价：用户态能读到内核版本号（信息泄露面，但这是设计——`GetVersionEx` 这类 API 就读它）。
- **TickCountMultiplier / 49.7 天回绕**：Win32 的 tick 计数是 32 位毫秒值，约 49.7 天绕回 0。为什么启动期要算乘数：毫秒 tick 是从中断时间换算来的，换算要乘数；乘数算好写进 SharedUserData，用户态读 tick 就不用系统调用。**KeMaximumIncrement**：时钟中断的最大增量（定时器间隔），tick 乘数由它算出（`KE/MISCC.C:713`）。**ExpLastTimeZoneBias**：上次生效的时区偏置（-1 表示未设定过，则用注册表里的 programmed bias，`INITOS.C:1143-1151`）。
- **INIT 段 / 可丢弃内存（Discardable）**：内核映像里"只在启动时用、启动完就回收"的段。为什么 Phase 1 的主函数要包一层分页桩（正文第 6 步）：Mm 回收 INIT 内存后，栈回溯走进已回收的地址会错；留一个常驻桩，栈回溯能认出函数边界。
- **ARC 命名**：加载器描述启动设备的约定格式（Advanced RISC Computing 命名），如 `multi(0)disk(0)rdisk(0)partition(1)`（哪块控制器、哪块盘、哪个分区）、`net(0)`（网络启动）。为什么要有统一命名：启动设备可能是 SCSI 盘、IDE 盘、网络——没有统一命名，加载器与内核对"系统在哪"说不到一块去；`CreateSystemRootLink` 就是用加载器的 ARC 设备名与系统路径建 `\SystemRoot` 符号链接。
- **KLDR_DATA_TABLE_ENTRY（加载器数据表项）**：已加载模块链表（`LoaderBlock->LoadOrderListHead`）的节点结构，含映像基址（`DllBase`）、大小、名字（`BaseDllName` / `FullDllName`）。第 0 批加载链地图已详述，此处只提醒：正文第 4 步"ntoskrnl 永远是链的第一项"指的就是这类节点。

### 五、Phase 1 里的子系统（总控级一句话）

- **子系统缩写总表（本批顺序表里的两字母代号）**：**MM = 内存管理器**（Memory Manager，管物理页、虚拟地址、池、节对象）；**OB = 对象管理器**（Object Manager，管对象目录、句柄、对象类型）；**SE = 安全子系统**（Security，管令牌、安全描述符、审计）；**PS = 进程管理器**（Process Manager，管进程/线程对象）；**KE = 内核**（Kernel，调度、中断、同步原语）；**KD = 内核调试器**（Kernel Debugger）；**Dbgk = 用户态调试的内核侧支持**（调试对象、调试端口；`DbgkInitialize` 建调试系统，`DBGK/DBGKOBJ.C:74`）；**EX = 执行体支持例程**（Executive Support，资源锁、池 lookaside、句柄表包等基础服务）；**IO = I/O 管理器**（I/O Manager，驱动加载、IRP 分发）；**CM = 配置管理器**（Configuration Manager，即注册表子系统）。PO/CC/PP/LPC/FSRTL 见下面各条。
- **PFN（页帧号，Page Frame Number）**：物理内存页的编号。**PFN 数据库**是每物理页一条记录的数组，记每页的状态（空闲、零页、在用、坏页等）——MM 建 PFN 数据库就是按加载器的内存描述符，给每页发一张"状态卡"（第 3 批展开）。
- **ERESOURCE（执行体资源）**：执行体提供的读写锁——多个读者可同时持有，写者独占。结构比自旋锁重：`INC/EX.H:4558` 的 ERESOURCE 有 15 个字段，内含一个 KSPIN_LOCK（`SpinLock`）、SharedWaiters 信号量、ExclusiveWaiters 事件、OwnerTable 所有者表、ContentionCount——ERESOURCE 是建在自旋锁之上的。但争用时的行为比自旋锁轻：它只在极短时间内持内部自旋锁改状态，争用时挂信号量/事件上用 `KeWaitForSingleObject` 阻塞（`EX/RESOURCE.C:2619`），让出 CPU、不抬 IRQL；自旋锁争用时空转烧 CPU 并抬 IRQL 到 DISPATCH_LEVEL、阻塞本核。"重"量的是机制，"轻"量的是争用代价——两个维度，不矛盾。为什么 MM 等子系统要用它而不用自旋锁：需要"读多写少、可等待、可长持"的场合；短暂、原子、可能在高 IRQL 的操作仍用自旋锁——两者是分工，不是替代（ERESOURCE 内部还用着自旋锁）。
- **Po（电源管理器）**：管处理器闲置、设备电源状态。分两次：`PoInitSystem(0)` 建基础电源结构，`PoInitSystem(1)` 做需要全系统就绪的电源初始化。
- **Cc（缓存管理器）**：文件缓存。为什么在 Cm 之前：注册表的文件操作（hive 文件的读写、Part 2 绑定磁盘文件）走文件缓存，所以缓存管理器必须先于注册表的文件操作就绪——注意：Part 1 开的是内存 hive、不读文件、不碰 Cc（CmInitSystem1 所在文件与 hive 初始化文件里都没有 Cc 调用），所以这不是 Part 1 这个位置的硬性前提，而是注册表子系统整体的文件操作前提；顺序表把 Cc 放在 Cm 之前是为整个注册表服务，不只为 Part 1。
- **CmInitSystem1（注册表 Part 1）**：只开 `\REGISTRY\MACHINE\SYSTEM` 与 `\REGISTRY\MACHINE\HARDWARE` 两个 hive。为什么只开两个：IO 初始化要读这两个 hive 的设备/驱动配置；其余 hive（SOFTWARE 等）晚开（第 6 批展开）。加载器能读 SYSTEM hive 是因为它自带极简只读文件系统代码（见正文第 9 步第 ⑰ 项），不是因为文件系统已经起来了。
- **CcPfInitializePrefetcher（预取器）**：按历史记录预读文件、加速启动。为什么在注册表后：预取参数存在注册表里，没注册表查不到参数。
- **FsRtl（文件系统运行时库）**：文件系统驱动共用的包。为什么在 IO 前：文件系统驱动要 FsRtl 的包才能初始化。
- **Range List（范围链表）**：记录"哪段资源已分配"的链表。为什么在 PnP 前：注释原话——PnP 用范围链表记资源分配。
- **Pp（即插即用管理器）**：枚举设备、建设备树、加载驱动。分两次：Phase 0 建骨架，Phase 1 在 IO 之前完成（注释原话：必须在 I/O 系统初始化之前）。
- **LPC（本地过程调用，Local Inter-Process Communication）**：内核提供的消息传递 IPC 机制——客户与服务线程经端口（Port）传消息、等回复。对话有两端，任一端可能在对话中途死掉，所以必须有"一方死了、另一方怎么办"的清理机制（见正文第 9 步 ⑳）。
- **ExInitSystemPhase2**：执行体第二阶段初始化。为什么叫"Phase2"：注释原话——"Now that system time is running"（系统时间跑起来后才能做的部分），即依赖时间服务的执行体初始化。
- **IoInitSystem**：I/O 系统初始化——boot 驱动加载潮（第 7 批展开）。进度条 25% → 75% 就发生在这里。
- **MmInitSystem(0/1/2)**：MM 的三阶段调用（总控级含义见正文第 8 步，内部展开在第 3 批）。
- **SeRmInitPhase1（安全引用监视器 Phase 1）**：安全审计与安全策略的 Phase 1 部分。为什么这时：审计要注册表 + 对象系统全就绪。
- **VDM（虚拟 DOS 机）**：跑 16 位 DOS 程序的兼容机制。为什么失败不蓝屏：注释原话——VDM 坏了只是 DOS 程序跑不了，不是启动必需。
- **零页线程（Zero Page Thread）**：优先级 0 的后台线程，从空闲链表取页、清零、放零页链表。为什么要有：新分配的页必须清零（防信息泄露——上一任主人的数据不能留给下一任）；清零是重活，放后台慢慢做，不挡正常分配（正常分配先用已清零的页，不够再现清）。
- **SMSS（Session Manager Subsystem，会话管理器）**：第一个用户进程（`\SystemRoot\System32\smss.exe`，路径来自注册表、可配置）。为什么它是"第一个用户进程"而不是"第一个进程"：Idle/System 是内核进程胚胎（第 1 批），SMSS 是第一个跑在用户态的进程——"第一"的定语不同（见第六部分第 6 条）。**NtInitialUserProcessBuffer**：默认 SMSS 路径缓冲区（初值 `\SystemRoot\System32\smss.exe`，128 字符；注册表可覆盖；校验类型 REG_SZ 与长度，`INITOS.C:167-171`）。
- **RtlCreateUserProcess / ZwResumeThread**：创建用户进程、恢复其首线程（开始跑）。SMSS 的诞生就是这两步。
- **KeSetAffinityProcess（亲和性）**：设定进程允许跑在哪些核上。为什么 Phase 1 里要扩系统进程的亲和性：System 进程的线程此前只在 P0 上，AP 全就绪后才放开到全核。
- **HalAllProcessorsStarted**：HAL 的"所有核已拉起"通知。为什么要有：HAL 要在所有核就绪后做 post 初始化（比如 APIC 的最终配置），单个核就绪时做不了。
- **CmpInitSystemVersion(n, …)**：CM（注册表子系统）的分阶段检查点，在 Phase 0 调 0、1，在 Phase 1 调 2、8、3、9、4、6、7。**各编号的具体语义在 WRK 树里查不到定义（见未钉死清单第 1 条）**——本批只记录调用位置与参数，不编造每号的含义。

### 六、实查补漏术语（2026-10-25 全文实查补，凡正文出现而术语表漏的）

- **NUMA（非统一内存访问，Non-Uniform Memory Access）**：多处理器系统中，内存按节点（node）划分，每个处理器访问本地节点的内存快、访问远端节点的内存慢。为什么 MM 之前要先划分节点：MM 建 PFN 数据库时要按节点分，节点划分晚于 MM，PFN 数据库就得返工（正文第 4 步第 ⑩ 项）。
- **池（Pool）**：内核的堆分配器，分两种：非分页池（Non-Paged Pool，常驻内存、永不换出，高 IRQL 也能用）与分页池（Paged Pool，可换出、只能在 < DISPATCH_LEVEL 用）。为什么分两种：有些内核数据在高 IRQL 也要访问（不能缺页），只能放非分页池；其余数据可换出、省物理内存，放分页池。池的 IRQL 红线见第 1 批（分页池 ≤ APC_LEVEL、非分页池 ≤ DISPATCH_LEVEL，超限 BAD_POOL_CALLER）。
- **节对象（Section Object）**：可映射的一段内存的抽象——背后由文件或换页文件支撑，进程把节映射进地址空间就得到文件内容（或共享内存）。为什么 MM Phase 1 要建它：系统 DLL 映射、文件映射、共享内存全靠节对象；它是"文件内容进地址空间"的载体。
- **物理内存对象（Physical Memory Object）**：代表整段物理内存的节对象（`\Device\PhysicalMemory`），内核态可经它读写任意物理页。为什么 Phase 1 建它：调试、诊断等需要直接访问物理内存的场合靠它（第 3 批展开）。
- **工作集（Working Set）**：一个进程（或系统）当前常驻的页的集合。工作集管理器在内存紧张时修剪工作集（把不常用的页换出），在内存宽裕时允许它增长。为什么执行体分页要"纳入系统工作集管理"：PAGE 节代码被换出的时机由工作集管理器按内存压力决定，不是立即换出。
- **IRP（I/O 请求包，I/O Request Packet）**：I/O 管理器发给驱动的请求包——读/写哪个设备、多少字节、数据放哪，全写在包里，驱动处理完把包往下传或完成它。为什么 IO 条目说"IRP 分发"：驱动模型就是"包流过驱动栈"，I/O 管理器的工作就是建包、分发包、回收包（第 7 批展开）。
- **DMA（直接内存访问，Direct Memory Access）**：设备不经 CPU、直接读写内存的机制。为什么 HAL Phase 1 才做它：DMA 需要 HAL 设好的平台设备与中断路由，Phase 0 的 HAL 只做到中断控制器能用（按调用点反读，见 HAL 条目）。
- **句柄（Handle）**：用户态引用内核对象的一串整数凭证——用户态拿句柄号，内核查句柄表找到对象。句柄表包就是这张表的管理结构，Ob 初始化前要先备好（正文第 4 步第 ⑭ 项）。
- **对象类型（Object Type）**：一类对象的模板（每类对象有自己的类型对象，规定这对象的通用操作：创建、删除、权限检查）。LPC 的 Port/WaitablePort 就是两种对象类型，创建它们要 ObCreateObjectType，所以 Lpc 必须在 Ob Phase 1 之后（正文第 9 步第 ⑳ 项）。
- **设备对象（Device Object）**：驱动为每个设备在对象管理器里建的对象——I/O 请求就是发往设备对象的。PnP 枚举出的设备树就是设备对象的层次结构（第 7 批展开）。
- **令牌（Token）**：进程的安全身份凭证——含用户 SID、所属组、特权集。进程创建要令牌，所以 Ps 在 Se 之后（正文第 5 步）。
- **安全描述符（Security Descriptor）**：挂在对象上的访问控制数据——所有者、DACL（自主访问控制表：谁能对这对象做什么）、SACL（系统审计表：什么操作要记审计）。安全描述符依附于对象，所以 Se 在 Ob 之后。
- **换页文件（Pagefile）**：磁盘上 backing 被换出页的文件。页文件支撑的节就是"内容不在文件里、在换页文件里"的节；代码页由映像文件支撑、不走换页文件（正文第 9 步第 ㉓ 项）。
- **prologue / epilogue（函数序言/尾声）**：函数开头建立栈帧的代码（prologue）与结尾拆除栈帧的代码（epilogue）。栈回溯靠识别序言/尾声来认出函数边界，所以 Phase 1 的分页桩必须常驻、带正确序言/尾声（正文第 6 步）。
- **REG_SZ**：注册表值类型之一，表示字符串（以 0 结尾的 Unicode 字符串）。SMSS 路径校验要求类型必须是 REG_SZ，就是要求它是个字符串值（正文第 9 步第 ㉗ 项）。
- **chkdsk（磁盘检查）**：检查并修复磁盘文件系统错误的工具。SMSS 启动后跑 autocheck（chkdsk），跑完才调 NtInitializeRegistry 开其余 hive（QA 第 18 条）。

---

## 第二部分：本批地图（两张顺序表 + 两条支线）

> 圈号说明（第 3 批自检时备注）：本批地图的圈号（①-㊷，Phase 1 地图）与正文的圈号是两套引用体系——正文的 ⑳ 在 Phase 0 支线与 Phase 1 尾各用了一次，地图的 ㉒（预取器）与正文的 ㉒（IoInitSystem）也不是同一项。跨区引用时请带区位（如"地图第 ⑩ 步"、"正文 ㉕'"），不要裸用圈号。

```
KiInitializeKernel（第 1 批）
│ P0：ExpInitializeExecutive(0, LoaderBlock) —— Phase 0（中断关、单核）
│ ① 校验加载器 → ② Textmode/远启动 → ③ 烧内存 → ④ NLS 借表
│ ⑤ HalInitSystem(0) → 取翻译表 → _enable() 开中断
│ ⑥ BugCheck 消息块 → ⑦ MP 适用性校验 → ⑧ Cmp(0) → ⑨ ExInitSystem
│ ⑩ KeNumaInitialize → ⑪ MmInitSystem(0) → ⑫ 导符号
│ ⑬ NLS 拷到池 → ⑭ Cmp(1) → ⑮ 栈回溯/异常日志/句柄表包
│ ⑯ Ob → ⑰ Se → ⑱ Ps(0)→ ⑲ Pp(0) → ⑳ Dbgk
│ ㉑ Tick 乘数 + SharedUserData 版本与机型
│ AP：ExpInitializeExecutive(AP#, LoaderBlock) —— 只调 HalInitSystem
▼ 返回 KiInitializeKernel → 汇编 → KiIdleLoop（第 1 批尾巴开闸）
▼ 调度器选中 Phase 1 线程
Phase1Initialization（分页桩，INIT 段外常驻）
│ Phase1InitializationDiscard（INIT 段内，跑完即回收）
│ ① 提权 MAXIMUM_PRIORITY-1 防抢占 → ② HalInitSystem(1)
│ ③ BootVideo → ④ WinPE → ⑤ Cmp(2) → ⑥ Po(0) → ⑦ 系统时间
│ ⑧ Cmp(8, Options) → ⑨ Cmp(3) → ⑩ KeStartAllProcessors（拉 AP）
│ ⑪ 重读 RTC 时间 → ⑫ 系统进程亲和性扩全核 → ⑬ Cmp(9)
│ ⑭ HalAllProcessorsStarted → ⑮ Cmp(4) → ⑯ Ob/Ex/Ke/Kd/Se 的 Phase 1 重初始化 → CreateSystemRootLink（:1281，建 \SystemRoot）
│ ⑰ MmInitSystem(1) → ⑱ NLS 快照 → ⑲ 映射系统 DLL
│ ⑳ Cc → ㉑ CmInitSystem1 → ㉒ 预取器 → ㉓ 时区 → ㉔ FsRtl
│ ㉕ RangeList → ㉖ KdDebuggerInitialize1 → ㉗ Pp(1) → ㉘ Lpc
│ ㉙ ExInitSystemPhase2 → ㉚ IoInitSystem（boot 驱动潮）
│ ㉛ Cmp(6) → ㉜ MmInitSystem(2)（PAGE 节分页开关）
│ ㉝ Vdm（可失败）→ ㉞ 核修订不一记日志 → ㉟ Po(1) → ㊱ Ps(1) → LoaderBlock 释放（KeLoaderBlock 置 NULL + MmFreeLoaderBlock，:1615-1621）
│ ㊲ SeRmInitPhase1 → ㊳ SMSS 路径校验 → ㊴ Cmp(7)
│ ㊵ 建 SMSS 参数与环境 → RtlCreateUserProcess → ZwResumeThread
│ ㊶ 进度条 100% → 等 SMSS 5 秒（5 秒内死 = SESSION5）
│ ㊷ InitializationPhase += 1（→ 2）→ KiInitializeInterruptTimers（x86）
▼ Phase1Initialization 桩：MmZeroPageThread() —— Phase 1 线程转任零页线程
第 3 批：MmInitSystem 三阶段展开；第 4 批：Ob；第 5 批：Ps；第 6 批：Cm；第 7 批：Io 与 boot 驱动潮
```

---

## 第三部分：正文

### 第 1 步：总控函数的契约——为什么一个函数要被调两次

`ExpInitializeExecutive` 的函数头注释把契约写得非常清楚，值得全文记住：

> "Each subcomponent is potentially called twice to perform Phase 0, and then Phase 1 initialization. During Phase 0 initialization, the only activity that may be performed is the initialization of subcomponent specific data. Phase 0 initialization is performed in the context of the kernel start up routine with interrupts disabled. During Phase 1 initialization, the system is fully operational and subcomponents may do any initialization that is necessary."

翻译过来就是两条铁律：**Phase 0 只能建本子系统的数据（静态部分），Phase 1 才能做任何必要的初始化（动态部分）**。为什么必须劈成两半：Phase 0 跑在 `KiInitializeKernel` 的上下文里——没有调度器（线程切不了）、中断关、分页刚上线；任何"等待、创建线程、做 I/O"的动作，在 Phase 0 都没有支撑它的机制。硬要在 Phase 0 里做，等于在脚手架还没搭好时往楼上搬家具。

为什么 Phase 1 就"完全可运行"：到 Phase 1 时，第 1 批的尾巴已经开闸（Idle 优先级归零、调度器跑起来）、Phase 0 的 `MmInitSystem(0)` 已让内存分配可用、中断已开——等待、分配、建线程全有了支撑。所以 Phase 1 的注释敢写 "fully operational"。

**为什么是"同一个函数调两次"而不是"两个函数"**：因为点火顺序表只有一张。`ExpInitializeExecutive(Number, LoaderBlock)` 靠 `Number`（核号）区分 P0 与 AP，靠全局 `InitializationPhase` 区分阶段；各子系统的 Init 函数内部也靠 `InitializationPhase` 区分自己该干哪一半。一张表、两次遍历，比两张表好维护——改顺序只改一个地方。

### 第 2 步：入口第一关——ExpIsLoaderValid 与 MISMATCHED_HAL

**做了什么**：`ExpInitializeExecutive` 一进来先调 `ExpIsLoaderValid(LoaderBlock)`；校验不过 → `KeBugCheckEx(MISMATCHED_HAL, 3, Extension->Size, Extension->MajorVersion, Extension->MinorVersion)`。

**为什么第一关就是校验加载器**：LoaderBlock 是加载器与内核之间的唯一交接单，而 Extension 是其中的"夹层块"——HAL 通过它递翻译表等平台数据（第 1 批）。夹层块带版本自描述（Size / Major / Minor），内核先验版本：版本对不上，说明加载器与内核对"交接单格式"的理解不一致——后面所有"按格式读字段"的操作都会错位。与其让系统在错位的数据上跑出一堆莫名其妙的故障，不如在入口就蓝屏、把版本号打进蓝屏参数里。**这是第 0 批"Extension 夹层块带版本自描述、校验不过 bugcheck"的实物**：版本校验不是走过场，是交接单的第一道防线。

### 第 3 步：Phase 0 顺序表（上）——HAL 之前：先定性、再烧内存、再借表

**① Textmode / 远启动判定**（`Number == 0` 块内最先）。为什么先定性：后续初始化要按"安装模式 / 远程启动 / 正常启动"分叉（I/O 路径不同），定性越早，分叉越干净。`ExpInTextModeSetup` 与 `IoRemoteBootClient` 两个全局标志就是定性的结果。不定性会怎样：安装模式若按正常启动走，安装盘与目标盘角色错位，I/O 初始化会认错系统盘；远启动若按本地磁盘走，启动设备根本不存在。

**② PERFMEM / BURNMEMORY 烧内存**（解析 `LoadOptions` 字符串）。为什么在 MM 上线前：烧掉的内存要从加载器的内存描述符里划走（`ExBurnMemory` 改写描述符类型为 `LoaderBBTMemory` / `LoaderBad`），MM 建 PFN 数据库时按描述符建——被划走的页根本不会进 PFN 数据库。MM 建好后再烧，等于房子盖好了再拆墙。为什么用启动选项而不是拔内存条：测试要可重复、可量化（"扣 128MB"比"拔一根内存条"精确得多）。

**③ NLS 表先借加载器的**（`InitNlsTableBase = LoaderBlock->NlsData->AnsiCodePageData`，算三个偏移，`RtlInitNlsTables` + `RtlResetRtlTranslations`）。为什么这时就要 NLS：字符串转换是对象名、注册表路径的前提——后面的初始化处处要比名字、转字符串，没有 NLS 表寸步难行。为什么是"借"不是"建"：池还没建（MM 还没上线），没地方分配 NLS 表的常驻内存；加载器内存里的表是现成的，先借来用——池建好后再拷（第 4 步第 ⑬ 项）。**"先借后建"是启动期的通用模式**：没分配器时借加载器的，有分配器后建自己的。

### 第 4 步：Phase 0 顺序表（中）——HAL、开中断、校验、资源、NUMA、MM

**④ HalInitSystem(0) → 取翻译表 → `_enable()`**（`INITOS.C:437` 起）。为什么 HAL 是 Phase 0 里第一个真正的子系统初始化：中断控制器是 HAL 的地盘——没有 HAL 初始化，中断连"往哪送"都不知道。`#if defined(_APIC_TPR_)` 下，紧接着从 LoaderBlock Extension 取 `HalpIRQLToTPR` / `HalpVectorToIRQL`（第 1 批的翻译表）——**翻译表的交接就发生在这里**，不是第 1 批：HAL 初始化后，内核才拿到"向量 ↔ IRQL"的换算表。

**为什么 Phase 0 中途就 `_enable()` 开中断**：这里有个注释与实现的张力，必须讲清。函数头注释说 Phase 0 "with interrupts disabled"——那是**入口状态**，不是全程状态。HAL 初始化前开中断等于裸奔（中断控制器没就绪、IDT 虽修好但中断路由未建）；HAL 初始化后，中断控制器可用、第 1 批修好的 IDT 已就绪，开中断是安全的。于是 Phase 0 的实际形态是"**HAL 之前关、HAL 之后开**"——注释写的是起点，实现走的是"条件具备就开"。把注释读成"Phase 0 全程关中断"，是误读（见第六部分第 1 条）。

**⑤ DBG-only 的 tick 前跳**（`#if DBG` 下的 `KeAdjustInterruptTime`）。注释原话："Set the interrupt time forward so the Win32 tick count will wrap within one hour to make rollover errors show up in fewer than 49.7 days." 为什么：32 位毫秒 tick 约 49.7 天绕回 0，绕回边界的 bug 在正常测试里极难撞见；测试版故意把中断时间往前拨，让绕回在一小时内发生——**把 49.7 天才能暴露的 bug 压缩到一小时内暴露**。为什么只在 DBG（测试版）：零售版拨时间会打乱用户可见的时钟——测试手段不能上零售。

**⑥ BugCheck 消息块**（找 ntoskrnl 映像的资源、置 `KiBugCodeMessages`）。为什么这时：蓝屏要打消息，消息资源在 ntoskrnl 映像里；启动期就把指针备好，后面任何一步蓝屏都有消息可打。注释警告：代码假设 ntoskrnl 的 `KLDR_DATA_TABLE_ENTRY` 永远是已加载模块链的第一项——**启动期的很多代码都依赖"加载器链的顺序约定"，这不是偶然，是加载器与内核之间的隐性契约**。

**⑦ MP 适用性校验**（`UP_DRIVER_ON_MP_SYSTEM`）。为什么：MP 内核 + UP HAL（或 UP 驱动）混用，行为会悄悄错（比如自旋锁在 UP HAL 下被改成 RET——第 1 批第五部分——若跑在 MP 机器上等于没锁）。启动期就要拦：混用不是"性能问题"，是"正确性问题"，拦得越早越好。

**⑧ CmpInitSystemVersion(0)**。CM 的第一个检查点（具体语义见未钉死清单第 1 条）。位置意义：它在 MM 之前——位置推断（非注释原话）：这个检查点只需要加载器已加载的 hive 数据（第 0 批：SYSTEM hive 由加载器整段读入），不需要池分配。

**⑨ ExInitSystem（资源包）**。为什么在 MM 之前：ERESOURCE 是执行体最基础的同步原语，后续各子系统初始化都要用它建锁；它本身只需要静态数据，故能排在 MM 之前。**顺序的推导法**：排在前面的，要么是"后面都要用的"，要么是"只需要静态数据的"——ExInitSystem 两条都占。

**⑩ KeNumaInitialize**。为什么在 MM 之前：NUMA 节点划分决定内存分配的归属——MM 建 PFN 数据库时要按节点分；节点划分晚于 MM，PFN 数据库就得返工。

**⑪ MmInitSystem(0)**。为什么这时：池、PFN 数据库、系统空间——后面一切分配都靠它。为什么它在 Ex 之后、Ob 之前：MM 内部要用 ERESOURCE（故在 Ex 后）；Ob 要分配对象（故在 MM 后）。**MM 是 Phase 0 的分水岭**：MM 之前是"借加载器、建静态"，MM 之后是"能分配、建动态"。第 3 批展开它的三阶段。

**⑫ 扫描加载模块链、导符号**（`DbgLoadImageSymbols`）。为什么在 MM 之后：符号加载要分配内存（池已建好才有得分配）。为什么这时就要导：调试器要尽早认得出驱动映像——后面的初始化一旦蓝屏，调试器看到的要是"驱动名"而不是"裸地址"。

**⑬ NLS 表拷到池**（算 `InitNlsTableSize`、池分配、拷、再 `RtlInitNlsTables` + `RtlResetRtlTranslations`）。为什么两次初始化：池建好前借加载器的，池建好后拷到池里、把加载器内存还回去——"先借后建"的收束。Phase 1 里还有一次快照（第 9 步第 ⑭ 项，地图 ⑱）：把 NLS 表快照到页文件支撑的节、unmap 再 map 回缓存——为什么：省系统缓存（启动期缓存利用率低，NLS 表常驻缓存是浪费）。

**⑭ CmpInitSystemVersion(1)、栈回溯库、异常日志、句柄表包**（`RtlInitializeStackTraceDataBase`、`RtlInitializeExceptionLog`、`ExInitializeHandleTablePackage`）。为什么这时：这三样都要分配——池建好后才能建。为什么句柄表包在这里：句柄是对象管理器的基础设施，Ob 初始化之前，句柄表的包要先备好。不备好会怎样：Ob 初始化时无句柄表可用，对象创建后无句柄可发。

### 第 5 步：Phase 0 顺序表（下）——OB → SE → PS → PP → Dbgk → 共享数据

**⑮ ObInitSystem**（→ OBJECT_INITIALIZATION_FAILED）。为什么对象管理器在 Phase 0 偏后、但在 Se/Ps 之前：Ob 要池（故在 MM 后）；Se 的安全描述符要挂在对象上、Ps 的进程本身就是对象（故 Ob 在 Se/Ps 前）。**OB 是"后面所有子系统都要用"的典型**——第 4 批展开它的自举（`\`、`\ObjectTypes`、前四类对象类型）。

**⑯ SeInitSystem**（→ SECURITY_INITIALIZATION_FAILED）。为什么在 Ob 后、Ps 前：安全描述符依附于对象（无对象则无处挂）；进程要令牌（无安全子系统则令牌无处建）。

**⑰ PsInitSystem(0)**（→ PROCESS_INITIALIZATION_FAILED）。Ps 的 Phase 0：立 System/Idle 进程胚胎、创建 Phase 1 线程（`PS/PSINIT.C:507`，第 1 批第 7 步已讲）。为什么 Ps 在 Se 之后：进程创建要令牌。为什么 Phase 0 的 Ps 只干这些：创建真正的系统线程、跑调度，那是 Phase 1 的事；Phase 0 的 Ps 只负责"把 Phase 1 的执行载体（线程）建出来"。

**⑱ PpInitSystem（Phase 0 调用）**（→ PP0_INITIALIZATION_FAILED）。PnP 的 Phase 0：建设备树骨架。为什么在 Ob 之后、IO 之前：设备对象是对象管理器的对象（故在 Ob 后）；IO 的 Phase 1 要靠 PnP 的设备树（故在 IO 前）。Pp（Phase 0 调用）排在 Ps（Phase 0 调用）之后的真实原因：`PpInitSystem` 的函数体在 WRK 树内按名只搜到调用点、没有函数体，树内无据，见未钉死清单第 9 条——旧版用 Ob 的理由回答 Ps 的位置是答非所问。Phase 0 的 Pp 具体清单是第 7 批内容。（注：`PpInitSystem` 签名是 `BOOLEAN PpInitSystem(VOID)`，不带 Phase 参数；此处“Phase 0 调用”指在 Phase 0 阶段被调一次，第 7 批已纠正旧记法。）

**⑲ DbgkInitialize、Tick 乘数、SharedUserData**。Dbgk 是内核调试支持（用户态调试的内核侧）。Tick 乘数（`ExComputeTickCountMultiplier(KeMaximumIncrement)`）写进 SharedUserData——为什么这时：用户态马上要读 tick，乘数必须先备好。SharedUserData 的版本（NtMajor/Minor）与映像机型范围（`ImageNumberLow/High` 按架构分 I386 / AMD64）——为什么：loader 靠机型范围判断"某个映像能不能在本机跑"（第 1 批驱动 ABI：I386 映像不能在 AMD64 内核上原生跑）。

**⑳ AP 支线：只调 HalInitSystem**。`else` 分支（`Number!= 0`）里只有一句：`HalInitSystem(InitializationPhase, LoaderBlock)`。为什么 AP 不用重走 Phase 0/1：执行体子系统是全局的——P0 已经建好了，AP 再建就是重复建、互相踩。AP 缺的只是两样：自己的 HAL 状态（中断控制器、APIC 配置是每核的）与自己的处理器状态（第 1 批）。所以 AP 支线 = "HAL 初始化 + 处理器点火"，执行体点火与它无关。**AP 的 `InitializationPhase` 参数就是当时的全局值**：AP 在 Phase 1 期间被拉起，传的就是 1——AP 的 HAL 初始化按 Phase 1 走。

### 第 6 步：Phase 0 → Phase 1 的交接——为什么 Phase 1 是个线程

**交接链**：`KiInitializeKernel` 返回 → 汇编进 `KiIdleLoop`（第 1 批尾巴开闸）→ 调度器选中 Phase 1 线程 → `Phase1Initialization`（分页桩）→ `Phase1InitializationDiscard`（INIT 段内正文）→ 返回后 `MmZeroPageThread()`。

**为什么 Phase 1 要做成线程**（第 1 批第 7 步已讲，总控视角再强调一次）：Phase 1 要做的是"正常内核代码"（等待、分配、建线程、做 I/O）——这些动作的支撑机制（调度器）本身就是"线程"模型。把 Phase 1 包成线程，调度器就能用统一机制运行它，不用为启动期另写一套驱动逻辑。**线程是内核的通用执行载体，启动后期的一切工作都复用它**。

**为什么 Discard 函数要包一层分页桩**（函数头注释原话）："This is because Mm reuses the discardable routine's (INIT) memory after phase 1 completes, but must keep this routine around so that the stack walker can obtain correct prologues/epilogues." 翻译：Phase 1 完成后，Mm 会回收 INIT 段内存复用；但栈回溯（stack walker）需要一个常驻的、带正确 prologue/epilogue 的桩函数来认出函数边界——如果栈回溯走进已回收的地址，看到的是别的数据、认不出边界。**桩是"给调试器留的路标"**：正文可以回收，路标必须常驻。

### 第 7 步：Phase 1 线程的盾牌——MAXIMUM_PRIORITY - 1

**做了什么**（`INITOS.C:1018`）：

```c
InitializationPhase = 1;
Thread = PsGetCurrentThread();
Priority = KeSetPriorityThread(&Thread->Tcb, MAXIMUM_PRIORITY - 1);
```

注释原话："Set the phase number and raise the priority of current thread to a high priority so it will not be preempted during initialization."（置阶段号、把当前线程提至高优先级，使其在初始化期间不被抢占。）

**为什么是盾牌**：与第 1 批 Idle 顶 HIGH_PRIORITY 同款逻辑——调度器的抢占规则是"只有更高优先级的就绪线程才能抢"。Phase 1 线程提至最高档附近，任何就绪线程（包括 Phase 0 里建好的、以及 Phase 1 期间陆续建好的系统线程）都抢不动它，Phase 1 顺序表才能不受打扰地一口气跑完。

**为什么是 MAXIMUM_PRIORITY - 1 而不是满格**：MAXIMUM_PRIORITY 是档数（32 档，优先级 0–31——`DispatcherReadyListHead[MAXIMUM_PRIORITY]` 的数组维度即档数），-1 即最高档 31，与 HIGH_PRIORITY 同值。**此为按数组维度与档位数推断，DDK 常量原定义不在 WRK 树内**（见未钉死清单第 2 条）。INITOS.C 内没有降回——Phase 1 线程此后转任零页线程（第 9 步），零页线程的文档环境就是优先级 0（`MM/ZEROPAGE.C` 头注释："This thread runs at priority zero"），盾牌是 Phase 1 专用的，用完即卸。

### 第 8 步：Phase 1 顺序表（上）——HAL、视频、时间、拉 AP

**① HalInitSystem(1)**（→ HAL1_INITIALIZATION_FAILED，注意与 Phase 0 的 HAL_INITIALIZATION_FAILED 区分——码名里的 "1" 就是阶段号）。为什么 HAL 也要两次：Phase 0 的 HAL 只做到"中断控制器能用"，Phase 1 的 HAL 才做全功能（DMA、定时器等平台设备）。两次的界线就是"能用"与"全功能"。

**② InbvDriverInitialize（启动视频驱动）**。注释原话："There is now enough functionality for the system Boot Video Driver to run."（现在已有足够功能让启动视频驱动跑起来。）为什么这时：进度条要尽早出来——从内核接管显示到图形子系统起来之间是漫长的黑屏期，进度条是唯一的用户可见反馈。为什么在 HAL(1) 之后：注释原话只说"现在已有足够功能让启动视频驱动跑起来"——"HAL 的显示抽象"是推断，树内没这句；准确说法是 HAL(1) 之后系统功能才够启动视频驱动跑，具体依赖 HAL 的哪部分功能，树内未展开。`/NOGUIBOOT` 开关在这里处理：不要图形启动时，释放显示所有权。

**③ WinPE 判定**。为什么：WinPE 的 I/O 路径（内存盘启动 `INIT_WINPEMODE_INRAM` vs 常规启动 `INIT_WINPEMODE_REGULAR`）与正常系统不同，初始化要分叉。不分叉会怎样：WinPE 用内存盘启动，若按常规磁盘路径走，I/O 初始化会去找不存在的本地系统盘。

**④ CmpInitSystemVersion(2)、PoInitSystem(0)**。电源的 Phase 0：建基础电源结构。为什么电源这么早：设备一枚举就要挂电源状态（Po 管设备电源状态），处理器闲置函数（第 1 批 `PRCB.PowerState.IdleFunction`）的承载结构也要先备好——承载结构必须在设备枚举（Io）之前备好。Po 两阶段的具体分工见未钉死清单第 7 条。

**⑤ 系统时间初始化**。注释原话："This cannot be done until after the phase one initialization of the HAL Layer."（这件事必须在 HAL 的 Phase 1 初始化之后才能做。）为什么：时钟源是 HAL 的——HAL 没全功能前，读 RTC、设系统时间都没有可靠的硬件路径。内容：`/YEAR` 开关强制年份（破时钟硬件的 workaround）；本地时间转 UTC（"If the system stores time in local time"）；时区偏置（此前 NT 设过时间则用 `ExpLastTimeZoneBias`，否则用注册表里的 programmed bias——"next best guess"）；`KeSetSystemTime` 设系统时间并通知各组件。

**⑥ CmpInitSystemVersion(8, Options)、CmpInitSystemVersion(3)**。CM 检查点（语义未钉死，见清单第 1 条）。位置意义：(8) 带 `Options`（启动选项字符串）——说明这个检查点与启动选项有关；(3) 在拉 AP 之前，且它本身在 `#ifndef NT_UP` 块内（UP 版没有这一步）——它是 MP 版在拉 AP 之前的检查点，具体语义见清单第 1 条。

**⑦ KeStartAllProcessors（拉 AP）**（`INITOS.C:1187`）。注释原话："If this is an MP build of the kernel start any other processors now"（如果是 MP 版内核，现在启动其余处理器）。**为什么 AP 拉起放在 Phase 1 这个位置**：Phase 0 是单核世界（注释原话：Phase 0 只能建本子系统数据、中断关）；拉 AP 需要调度器、DPC、IPI 全套——这些在 Phase 1 才就绪。为什么在 Ob/Ex/Ke 的 Phase 1 重初始化之前：AP 只需要 HAL + 处理器状态（第 5 步），不需要执行体 Phase 1；且拉 AP 会打乱系统时间（下一项），所以拉 AP 要尽量早、打乱后立刻修正——放在 Phase 1 前段是最合适的。

**⑧ 重读 RTC 时间**。注释原话："Since starting processors has thrown off the system time, get it again from the RTC and set the system time again."（启动处理器打乱了系统时间，从 RTC 再读一次、再设一次。）为什么拉 AP 会打乱时间：多核启动期间时钟中断的归属与计数会乱——**精确机制注释没展开，列入未钉死清单第 3 条**；但"打乱后立刻修正"的处理是确定的：再读一次 RTC、再设一次系统时间。**启动期的很多处理都是这个风格：知道会乱，但先接受、再修正——而不是试图让启动过程本身不乱**。

**⑨ 系统进程亲和性扩全核**（`KeSetAffinityProcess`）。注释原话："Set the affinity of the system process and all of its threads to all processors in the host configuration." 为什么：System 进程的线程此前只在 P0 上（AP 还没起来时只能这样）；AP 全就绪后，亲和性放开到全核，系统线程才能被调度到各核上。

**⑩ CmpInitSystemVersion(9, MPKernelString)**（`INITOS.C:1232`）。CM 检查点（语义未钉死，见清单第 1 条）。位置意义：(9) 带 `MPKernelString`（MP 内核标识字符串）——在 HAL 收尾之前，CM 先确认一次 MP 相关的版本/一致性；且它在 `#ifndef NT_UP` 块内（UP 版没有这一步，见第四部分）。

**⑪ HalAllProcessorsStarted、CmpInitSystemVersion(4, DataTableEntry)**（`INITOS.C:1240`、`:1244`）。HalAllProcessorsStarted（→ HAL1_INITIALIZATION_FAILED）注释原话："Signify to the HAL that all processors have been started and any post initialization should be performed."（通知 HAL：所有处理器已启动，可以做 post 初始化了。）为什么要有这个通知：HAL 的有些初始化必须在"所有核就绪"后才能做（比如 APIC 的最终配置、核间中断路由的收尾）——单个核就绪时做不了。**这是 HAL 的"收尾信号"，与 `HalInitSystem(1)` 的"开始信号"配对**。注意：HalAllProcessorsStarted 在 `#endif` 之后，UP 版照调不误（见第四部分）。Cmp(4) 带 `DataTableEntry`（某模块的数据表项）——HAL 收尾之后、重初始化之前，结合模块信息再确认一次。两次检查点夹的是 HalAllProcessorsStarted，不是重初始化（重初始化在 :1250–1271，在 Cmp(4) 之后）——旧笔记"夹着重初始化"的说法错了。

### 第 9 步：Phase 1 顺序表（下）——重初始化、驱动潮、SMSS

**⑫ Ob/Ex/Ke/Kd/Se 的 Phase 1 重初始化**（`INITOS.C:1250-1271`）。注释原话："Initialize OB, EX, KE, and KD." 为什么同名函数调两次（总控视角）：Phase 0 建静态数据，Phase 1 在"系统完全可运行"后做需要等待、分配、创建线程的活。为什么是 Ob → Ex → Ke → Kd → Se 这个顺序：对象（Ob）是载体 → 执行体资源（Ex）是同步原语 → 内核调度（Ke）是执行机制 → 调试（Kd）是观察手段 → 安全（Se）是策略层——**载体 → 原语 → 机制 → 观察 → 策略**，依赖链从底向上。（此梯子为顺序读法、解读，非源码注释原话；树内注释只写"Initialize OB, EX, KE, and KD"，未给依赖链。）

**⑬ MmInitSystem(1)**（→ MEMORY1_INITIALIZATION_FAILED）。MM 的第二次调用：按 `MM/MMINIT.C` 头注释，Phase 1 初始化节对象、物理内存对象、启动 MM 系统线程（分页池在 Phase 0 已建，不在 Phase 1——旧说法已纠）。为什么 MM 要在 Ob/Ex/Ke/Se 的重初始化之后：Mm(1) 的具体前置条件（节对象、物理内存对象、MM 系统线程需要什么）是第 3 批的内容，正文不拿"比如"搪塞；顺序表层面只确定：重初始化在前、Mm(1) 在后。

**⑭ NLS 快照到页文件支撑的节**。注释原话："Snapshot the NLS tables into a page file backed section, and then reset the translation tables." 为什么：Phase 0 末尾 NLS 表已拷到池里；Phase 1 里再快照到页文件支撑的节、unmap 再 map 回缓存——目的是省系统缓存（启动期缓存利用率低，NLS 表常驻缓存是浪费）。**同一批数据在启动期搬三次家（加载器内存 → 池 → 页文件支撑的节）**：每次搬家都是因为"更合适的存放处"出现了——启动期的内存管理就是一部"不断把东西搬到更合适的地方"的历史。

**⑮ 映射系统 DLL 到用户地址空间**。注释原话："Map the system dll into the user part of the address space"。为什么：每个用户进程都要 ntdll；启动期先把系统 DLL 的节对象建好、映射模板备好，用户进程创建时直接映射（第 8 批展开）。为什么这时：SMSS 马上要创建（第 ㊵ 项），SMSS 是用户进程、需要 ntdll——模板必须在 SMSS 之前备好。

**⑯ CcInitializeCacheManager**（→ CACHE_INITIALIZATION_FAILED）。为什么缓存管理器在 Cm 之前：注册表的文件操作（hive 文件读写、Part 2 绑定磁盘文件）走文件缓存，缓存管理器必须先于注册表的文件操作就绪。**措辞修正**：Part 1 开的是内存 hive、不读文件、不碰 Cc，所以这不是 Part 1 位置的硬性前提，而是注册表整体的文件操作前提——"看到 A 在 B 之前就说 A 是 B 的前提"在这里要收窄：Cc 是注册表文件操作的前提，不是 CmInitSystem1 本身的前提。

**⑰ CmInitSystem1**（→ CONFIG_INITIALIZATION_FAILED）。注释原话："Config management (particularly the registry) gets initialized in two parts. Part 1 makes \\REGISTRY\\MACHINE\\SYSTEM and \\REGISTRY\\MACHINE\\HARDWARE available. These are needed to complete IO init."（注册表分两段初始化。Part 1 只开 SYSTEM 与 HARDWARE——完成 IO 初始化需要它们。）为什么只开两个：IO 初始化要读这两个 hive 的设备/驱动配置；其余 hive（SOFTWARE 等）晚开（第 6 批展开）。**分段的原则**：先开"后面步骤必需的最小集"，其余的晚开——启动期的每一步都只拿它当下必需的东西。**这里有个必须当场解开的结**：SYSTEM hive 明明是磁盘上的文件，文件系统还没起来，加载器凭什么读到它？答案是加载器自带一份极简只读的文件系统代码（FAT + NTFS，只会按路径找文件、沿目录/簇链读数据，不写、不分配、不回放日志、不解压/解密），文件→扇区的翻译在启动时由它做（固件 BIOS INT 13h 只认扇区号、不懂文件）。正式 NTFS 驱动是内核驱动、需要执行体（MM/OB/IO/缓存），内核没起来它跑不了，所以不能复用、必须另有一份启动时用的读法；用完就扔，加载器内存在 MmInitSystem(2) 被释放回收。ntldr 本体不在 WRK 树内，行号给不出；交接点树里钉死：`LoaderBlock->RegistryBase`（`CONFIG/CMSYSINI.C:1636`）。

**⑱ CcPfInitializePrefetcher**。注释原话："Initialize the prefetcher after registry is initialized so we can query the prefetching parameters."（注册表初始化后初始化预取器，这样才能查询预取参数。）为什么：预取参数存在注册表里，没注册表查不到参数——**依赖链的又一例：参数存放处先于参数使用处**。

**⑲ 时区偏置计算、FsRtlInitSystem、RangeList、KdDebuggerInitialize1**。时区偏置与下次切换日期的计算（`Compute timezone bias and next cutover date`）。为什么在这个点：时区数据在注册表里，Cm Part 1（⑰）开了 SYSTEM hive 之后才能读到——算时区偏置的前提是注册表已开。FsRtl（→ FILE_INITIALIZATION_FAILED）：文件系统运行时库，文件系统驱动要它的包。RangeList：注释原话——"this must be before PNP initialization as PNP uses range lists"（必须在 PnP 之前，因为 PnP 用范围链表）。KdDebuggerInitialize1：内核调试器的 Phase 1 部分（具体内容树内未展开；位置推断：放在 Ob Phase 1 与 Cm Part 1 之后，调试观察的对象与配置已就绪——此为位置推断，非注释原话）。

**⑳ PpInitSystem（Phase 1 调用）、LpcInitSystem**。Pp 的 Phase 1（→ PP1_INITIALIZATION_FAILED）：注释原话——"must be done before the I/O system initializes"（必须在 I/O 系统初始化之前）。Lpc（→ LPC_INITIALIZATION_FAILED）：注释原话——"LPC needs to be initialized before the I/O system, since some drivers may create system threads that will terminate and cause LPC to be called."（LPC 必须在 I/O 之前，因为驱动可能创建系统线程、线程退出会调 LPC。）**为什么要有 LPC 清理、没有它会怎样**：线程退出时，`PspExitThread`（`PS/PSDELETE.C:794`）在 `:1197` 无条件调 `LpcExitThread`（`LPC/LPCCLOSE.C:355`，文档原话："called whenever a thread is exiting and need to cleanup the lpc port for the thread"）——每个线程退出都要过 LPC 清理。它干的事：拿 LPC 锁、摘掉本线程的 `LpcReplyChain`、置 `LpcExitThreadCalled = TRUE`、清 `LpcReplyMessageId`、释放待回复消息并解引用对方线程；其它等回复的线程看到 `LpcExitThreadCalled` 就不再死等、改走错误返回（`LPCRECV.C:478`、`LPCREPLY.C:348`、`LPCQUEUE.C:203` 查这个标志）。没有这套清理：对端永远等不到回复（挂死）、端口与消息对象泄露。**为什么 LPC 初始化必须在 IO 之前**：IoInitSystem 加载 boot 驱动，驱动的 `DriverEntry` 会创建系统线程，线程干完活退出就走 `PspExitThread → LpcExitThread`，第一件事就是拿 LPC 锁（`LpcpAcquireLpcpLockByThread`）；`LpcInitSystem`（`LPC/LPCINIT.C:89`）没跑过，锁没初始化（`LpcpInitializeLpcpLock`）、Port/WaitablePort 对象类型没创建、消息区没建——退出路径一碰就是未初始化状态，启动期直接蓝屏。也就是说：一个驱动工作线程仅仅"正常退出"，就能在 LPC 未初始化时把启动中的系统蓝屏。**Lpc 的位置（1507 行）正好卡在 Ob Phase 1（1250 行）之后、IoInitSystem（1531 行）之前的窗口里**：它要 Ob（创建对象类型要 `ObCreateObjectType`），又要在驱动建线程之前——这是依赖链夹出来的位置，不是随便排的。启动顺序里，"退出路径"与"创建路径"一样重要。

**㉑ ExInitSystemPhase2**。注释原话："Now that system time is running, initialize more of the Executive."（系统时间跑起来后，初始化更多的执行体。）为什么叫 Phase2：执行体初始化里依赖时间服务的部分，只能在系统时间跑起来后做——时间服务是 Phase 1 前段（第 ⑤ 项）才建好的。反过来，没有它会怎样：若在时间服务就绪前做依赖时间的执行体初始化，读到的是未初始化的系统时间，时间基准就是错的。

**㉒ IoInitSystem（boot 驱动潮）**（→ IO1_INITIALIZATION_FAILED）。注释原话："IoInitSystem updates progress bar updates from 25 to 75 %."（IoInitSystem 把进度条从 25% 推到 75%。）为什么进度条的大头在这里：boot 驱动加载潮是启动期最重、最长的一段（枚举设备、加载驱动、调每个驱动的 `DriverEntry`）——第 7 批展开。为什么 Io 在 Pp/Lpc 之后：IO 要靠 PnP 的设备树（Pp 已建好）、驱动线程退出要 LPC（Lpc 已建好）——**依赖链收束到 IO**。

**㉓ CmpInitSystemVersion(6)、MmInitSystem(2)**。注释原话："Begin paging the executive if desired."（如果需要，开始让执行体分页。）**先纠一个歧义："执行体分页"不是按子系统切的——对象管理器不可能整个被分出去；它是按 PE 节切的**。`MmInitSystem(2)` 的第一件事是调 `MiEnablePagingTheExecutive`（`MM/MMINIT.C:4248`），其文档原话："locates the start and end of the pageable code for each loaded module entry. This code resides in the PAGE section of each image."（定位每个已加载模块的可分页代码起止——这些代码位于各映像的 PAGE 节。）它遍历 `PsLoadedModuleList`（ntoskrnl、HAL、boot 驱动），读每个映像的 PE 节表，找名字叫 PAGE（代码里比对小端下的 'EGAP'）或 .edata 的节，对其中锁计数为 0 的段调 `MiEnablePagingOfDriverAtInit` 纳入系统工作集管理——此后工作集管理器可以在内存紧张时把它们换出。

**谁决定哪个函数进 PAGE 节**：源码里的 `#pragma alloc_text(PAGE, 函数名)`。实物例子：`OB/FASTREF.C` 的 `ObFastReferenceObject` 系列、`OB/OBCLOSE.C` 的 `NtClose`、`NtMakeTemporaryObject`、`ObMakeTemporaryObject` 都在 PAGE 节。反例：热路径（调度器、DPC、对象核心引用计数、句柄表查找）不在 PAGE 节，在非分页代码里，永远常驻。约束与第 1 批池红线同源：PAGE 节函数只能在 < DISPATCH_LEVEL 调用——因为一旦被换出，访问它会缺页，而缺页处理不能在 ≥ DISPATCH_LEVEL 做。

**"if desired" 的实义**：`MiEnablePagingTheExecutive` 开头就是两道闸：`MmDisablePagingExecutive & MM_SYSTEM_CODE_LOCKED_DOWN` 就直接 return（注释原话："Don't page kernel mode code if customer does not want it paged or if this is a diskless remote boot client."），对应注册表 `DisablePagingExecutive`（用户不想要执行体分页时）；无盘远启动客户机也不分页（没有本地磁盘可回读）。

**为什么在 IoInitSystem 之后**（修正"系统已稳"的笼统说法，精确到两条）：① 遍历的是 `PsLoadedModuleList`，boot 驱动是 IoInitSystem 才加载进这张表的——之前开，boot 驱动的 PAGE 节根本不在表里；② 换出的页再次访问要缺页回读，回读的源头是映像文件本身（代码页由映像文件支撑，不走换页文件），回读要走文件系统 + 磁盘驱动——这条路是 IoInitSystem 才打通的。之前开，换出后回读无路可走。所以开开关的前提是"回读路径已通"，不是笼统的"系统已稳"。另注意：纳入工作集管理只是"允许换出"，真正换出要等内存紧张时工作集管理器修剪；不紧张时照样常驻。

**"可分页"不是绝对的**：节有锁计数，驱动可用 `MmLockPagableCodeSection` 之类把自己的 PAGE 代码临时锁定（比如某段代码要在高 IRQL 跑时）；`MiEnablePagingTheExecutive` 跳过锁计数非 0 的段，中途发现被锁了会重走一遍。

**纠正（第 3 批查实）**：`MM/MMINIT.C` 头注释（`:346-355`）说 Phase 2 "frees memory used by the OsLoader"，但 Phase 2 的代码只有 `MiEnablePagingTheExecutive()`（`:2392`），并不释放加载器内存——头注释是旧说法。真正释放加载器内存的是 `MmFreeLoaderBlock`，在 Phase 1 尾由 `INITOS.C:1621` 直接调。原句按头注释写错了，已按代码修正。

**㉔ KeI386VdmInitialize（可失败）、核修订不一记日志**。Vdm：注释原话——失败不蓝屏（"If this fails, Vdms may not be able to run, but it isn't necessary to bugcheck the system because of this."）。为什么：VDM 是兼容功能，不是启动必需——**启动期的失败处理分两档：必需的失败蓝屏，非必需的失败记日志继续**。核修订不一记日志：注释原话——"Now that the error log interface has been initialized, write an informational message if it was determined that the processors in the system are at differing revision levels."（错误日志接口已初始化，若发现各核修订级别不同，写一条信息性消息。）——与第 1 批的 CPU 对账呼应：对账时发现的差异，在这里记一笔。

**㉕ PoInitSystem(1)、PsInitSystem(1)**。Po 的 Phase 1（→ INTERNAL_POWER_ERROR）：内容与位置原因见未钉死清单第 7 条（树内未展开），正文不只列死因码。Ps 的 Phase 1：注释原话——"Okay to call PsInitSystem now that \\SystemRoot is defined so it can locate NTDLL.DLL and SMSS.EXE."（现在可以调 PsInitSystem 了，因为 \SystemRoot 已定义、Ps 可以定位 NTDLL.DLL 与 SMSS.EXE。）为什么 Ps(1) 必须在 \SystemRoot 定义后：Ps 要靠 \SystemRoot 定位这两个文件（SMSS 创建与 ntdll 映射的文件来源）；\SystemRoot 符号链接由 Phase 1 在 :1281 的 `CreateSystemRootLink` 建好（用加载器的 ARC 设备名与系统路径；符号链接是对象，故在 Ob Phase 1 :1250 之后），不是 IoInitSystem 期间建的。Ps(1) 的位置约束只是"\SystemRoot 已定义"（:1281 已建好），不是"IO 之后的第一时间"——中间还隔着 Cc、Cm、Pp、Lpc、ExPhase2、Io 一大串。

**㉕' LoaderBlock 释放**（`INITOS.C:1615-1621`）。先判 `LoaderBlock == KeLoaderBlock` 成立则把全局 `KeLoaderBlock` 置 NULL，再调 `MmFreeLoaderBlock` 释放整块加载器内存。为什么这时：注释原话——"Force KeBugCheck to look at PsLoadedModuleList now that it is setup"（PsLoadedModuleList 已建好，蓝屏回退枚举从此改查它，不再需要 LoaderBlock）。

**㉖ SeRmInitPhase1**。安全引用监视器的 Phase 1。为什么这时：审计与安全策略要注册表 + 对象系统全就绪——注册表 Part 1 已开、对象系统 Phase 1 已重初始化，条件刚好齐备。

**㉗ SMSS 路径校验**（`NtInitialUserProcessBuffer`）。为什么从注册表读 SMSS 路径：SMSS 路径是可配置的（注册表值），不是硬编码——校验类型（必须是 REG_SZ）与长度，非法 → SESSION3_INITIALIZATION_FAILED。为什么路径可配置：不同安装（正常系统 / WinPE / 嵌入式）的 SMSS 位置不同，硬编码会断了这些变体。

**㉘ 建 SMSS 参数与环境 → RtlCreateUserProcess → ZwResumeThread → 等 5 秒**。环境字符串拼 Path、SystemDrive、SystemRoot（SMSS 启动后要靠它们找 DLL 与系统目录）。`RtlCreateUserProcess` 失败 → SESSION3_INITIALIZATION_FAILED；`ZwResumeThread` 失败 → SESSION4_INITIALIZATION_FAILED。进度条推到 100%，打开调试输出（"so that we can see chkdsk run"——chkdsk 在 SMSS 之后跑，调试输出要先开）。然后等 SMSS 5 秒：

```c
// Wait five seconds for the session manager to get started or terminate.
// If the wait times out, then the session manager is assumed to be healthy
// and the zero page thread is called.
Status = ZwWaitForSingleObject(ProcessInformation->Process, FALSE, &OldTime /* 5 秒 */);
if (Status == STATUS_SUCCESS) {
    KeBugCheck(SESSION5_INITIALIZATION_FAILED);
}
```

为什么等 5 秒：SMSS 若 5 秒内死掉（`ZwWaitForSingleObject` 返回 SUCCESS 表示进程已终止），说明用户世界根本立不起来——SESSION5。为什么不等更久：启动不能无限等一个可能已死的进程；5 秒是"它要么活、要么死"的判断窗——超时则假设健康、继续。**这是启动期的容错哲学：给关键进程一个有限的"证明自己活着"的窗口，窗口内死了就全系统停机，窗口外活着就放行**。

**㉙ 收尾：InitializationPhase += 1（→ 2）、KiInitializeInterruptTimers（x86）**。阶段号加到 2——Phase 2 是"SMSS 已起、系统进入正常运行"的标记。加到 2 之后谁读它：各子系统 Init 函数的 switch 再进来会走非法分支（`EX/EXINIT.C:123`、`PS/PSINIT.C:175`、`SE/SEINIT.C:63` 的 UNEXPECTED_INITIALIZATION_CALL——2 的意思是"初始化已结束，再调 Init 就是非法调用"）；IO 把"启动卷挂载失败"从"启动期必蓝屏"降为"已启动后不蓝屏"（`IO/IOMGR/INTERNAL.C:5155` 的 `InitializationPhase < 2` 判定）；MM 配额在 DBG 下开始断言"被收费的线程不是 Idle 线程"（`MM/MMQUOTA.C:103` 的 `> 1` 判定）。所以 2 的含义就是"启动结束、进入正常运行"的边界线，各子系统按它切换启动期行为与正常行为。x86 的中断定时器初始化放在最后（为什么只在 x86、为什么放在收尾：条件编译存在，具体原因未注释——见未钉死清单第 5 条；正文不拿"系统已稳"搪塞）。

**㉚ Phase 1 线程转任零页线程**。`Phase1Initialization` 桩的最后一句：`MmZeroPageThread()`。为什么是转任不是退出：Phase 1 线程的使命完成，但线程对象还在；零页线程是优先级 0 的后台活——把"用完的线程"转成"永远有用的后台线程"，不浪费一次线程创建。零页线程干什么：从空闲链表取页、清零、放零页链表——为什么要有零页：新分配的页必须清零（防信息泄露——上一任主人的数据不能留给下一任），清零是重活，放后台慢慢做，不挡正常分配（正常分配先用已清零的页，不够再现清）。

---

## 第四部分：32 位 vs 64 位对照、单核 vs 多核对照

### 32 位 vs 64 位

先总括一句：INITOS.C 是两架构共享的，顺序表本身相同，差异只在下面这些 `#if` 条件块里——AMD64 没有另起一张 Phase 1 顺序表（开中断代码的位置例外，见最后一项）。

- **ExInitPoolLookasidePointers 只在非 AMD64**（`#if !defined(_AMD64_)`）。为什么 AMD64 不用这一步：树里只有条件编译、没有注释说明——列入未钉死清单第 4 条，不编造。能确定的是：PRCB 的池 lookaside 指针初始化在两架构上走不同路径，i386 需要这一步显式初始化。
- **SharedUserData 的映像机型范围按架构分**（`#if defined(_AMD64_)` → IMAGE_FILE_MACHINE_AMD64，`#elif defined(_X86_)` → IMAGE_FILE_MACHINE_I386，否则 `#error "no target architecture"`）。为什么：loader 靠机型范围判断"某个映像能不能在本机跑"——I386 映像不能在 AMD64 内核上原生跑（第 1 批第四部分第 5 项：指针、调用约定、IDT 全变）。`#error` 的意思是：连架构都定不下来，编译直接失败——启动期的机型问题不能留到运行时。
- **KiInitializeInterruptTimers 只在 x86**（`#if defined(_X86_)`）。为什么 AMD64 不用：条件编译存在，原因未注释——列入未钉死清单第 5 条。
- **KeI386VdmInitialize 只在 x86**。为什么：VDM 是 16 位 DOS 兼容，依赖虚拟 8086 模式；AMD64 的 long mode 下没有虚拟 8086 模式——架构上就不可能实现 VDM。这不是"AMD64 没做"，是"AMD64 做不了"。**32 位能做的兼容，64 位因为架构砍掉了底层模式而做不了——这是反向的例子**：不是 64 位总能做得更多，有些旧机制是 64 位主动放弃的。
- **Phase 0 的 `_enable()` 只在 x86**（`#if defined(_X86_)`）。为什么：开中断的指令是架构相关的（x86 的 `sti`）；AMD64 同样用 `_enable()`，但位置在 AMD64 自己的 `KiInitializeKernel`（`KE/AMD64/INITKR.C:718`），不在 INITOS.C 的 x86 条件块里——两架构的区别是"开中断的代码放在哪"，不是"用什么 API"（"KeEnableInterrupts 一系"的说法在树内搜不到这个符号，撤回）。

### 单核 vs 多核

- **KeStartAllProcessors 只在 MP 编译**。UP 内核的 Phase 1 没有拉 AP、没有重读 RTC、没有亲和性扩核（这三步在 `#ifndef NT_UP` 块内，约 1178–1233）；但 `HalAllProcessorsStarted`（:1240）在 `#endif` 之后，UP 版照调不误（UP 的 HAL 内部如何处理不在树内，不展开）——旧笔记"三步直接消失"错了，消失的是块内的三步，不是 HalAllProcessorsStarted。为什么 UP 能省块内的三步：没有第二个核，所有"核间"概念（IPI、亲和性、栅栏）都不存在。
- **MP 内核跑在 UP 机器上**：`KeStartAllProcessors` 循环里 HAL 说"没有更多核"，循环零次结束——顺序表本身不变，只是拉 AP 的一步空转。为什么顺序表不为 UP 另写一份：同一张表、零次循环，比两张表好维护（第 1 步的"一张表"原则）。
- **Phase 0 的"单核世界"**：Phase 0 注释说"the only activity that may be performed is the initialization of subcomponent specific data"——这个约束成立的前提就是单核：没有第二个核来抢数据、没有核间中断来打断。拉 AP 放在 Phase 1，正是因为 Phase 1 才有调度器与 DPC 来支撑多核。**单核不是"多核的简化版"，是多核启动的前提阶段**：先在一个核上把世界建好，再把世界复制到其他核上。

---

## 第五部分：逆向 / 安全解读

- **点火顺序表 = 攻击面地图**：每个 Init 函数的失败都对应一个 BugCheck 码——蓝屏码即"死在哪一步"的坐标。逆向时按码定位：先看码名里的阶段号（0/1）与子系统名，再按本批顺序表缩小到具体调用。启动期没有日志系统，蓝屏码就是唯一的现场记录。
- **SMSS 路径来自注册表 = 信任边界**：SMSS 是第一个用户进程，它的路径是注册表值（可配置），不是硬编码。这意味着：能改这个注册表值的人，就能决定"第一个用户进程跑什么"。启动期的持久化机制很多都落在这个点上——读法是"注册表值是信任边界"，不是"利用步骤"（本批只讲机制）。
- **SharedUserData 用户态可读 = 设计上的信息面**：版本号、tick 乘数、时区偏置写在用户态可读的共享页里——用户态程序不经系统调用就能读到内核版本号。这是设计（`GetVersionEx` 这类 API 就读它），不是漏洞；但逆向时要知道：版本号探测不需要任何特权。
- **INIT 段回收 = 取证时效**：Phase 1 完成后 INIT 段内存被 MM 回收复用——转储里 INIT 段的代码已不在。取证时要注意："启动代码只存在于启动期"，事后转储里找不到 `Phase1InitializationDiscard` 的正文是正常的，不是被抹掉了。
- **零页机制 = 纵深防御的一环**：页不清零就复用 = 跨进程信息泄露（上一任主人的数据留给下一任）。零页线程把清零放后台，是"分配快"与"不泄露"的折中——正常分配先用已清零的页，不够再现清。
- **5 秒窗口 = 启动期的容错哲学**：SMSS 5 秒内死了就全系统停机（SESSION5），超时则假设健康。读法：关键进程的"存活证明"窗口是有限的——无限等一个可能已死的进程，比停机更糟。

---

## 第六部分：易混点辨析（QA 内容区，随后续问答补充）

| # | 易混说法 | 正确说法 | 证据 |
|---|---|---|---|
| 1 | "Phase 0 全程中断关" | 注释说的是入口状态；HAL(0) 初始化后就 `_enable()` 开中断了——Phase 0 是"HAL 之前关、HAL 之后开" | `INITOS.C:437` HalInitSystem(0) 后 `_enable()`；函数头注释是入口约束 |
| 2 | "同名 Init 函数调两次是重复" | 两次干不同的活：Phase 0 建静态数据，Phase 1 做需要等待、分配、建线程的活；`InitializationPhase` 区分两阶段 | 函数头注释；Phase 0 的 827–847 是 Ob/Se/Ps/Pp/Dbgk（Ex 在 568，KeNuma 在 576）；Phase 1 的 1250–1271 才是 Ob/Ex/Ke/Kd/Se |
| 3 | "Phase 1 线程跑完就退了" | 转任零页线程：`Phase1Initialization` 桩的最后一句是 `MmZeroPageThread()`，线程对象复用、优先级归 0 | `INITOS.C:922-931` 桩函数；`MM/ZEROPAGE.C` 头注释 "runs at priority zero" |
| 4 | "AP 在 Phase 1 才拉起，所以 Phase 0 是单核"——混淆两个"初始化" | AP 的处理器点火（第 1 批：PCR/PRCB/Idle 线程）与执行体点火是两回事：AP 的 `KiInitializeKernel` 在 Phase 1 里跑，但它跑的是处理器初始化，不是执行体初始化；执行体是全局的，P0 已建好 | AP 支线只有 `HalInitSystem`（`INITOS.C:889-897`）；第 1 批的 AP 分支 |
| 5 | "MmInitSystem 调三次是啰嗦" | 三次对应三个阶段：(0) 分页功能、非分页池、分页池、PFN 数据库 → (1) 节对象、物理内存对象、启动 MM 系统线程 → (2) 执行体分页开关 + 释放加载器内存 | `MM/MMINIT.C` 头注释；`INITOS.C:582`（Phase 0）、`:1286`（Phase 1）、`:1547`（"Begin paging the executive"） |
| 6 | "SMSS 是第一个进程" | SMSS 是第一个用户进程；Idle/System 是内核进程胚胎（第 1 批），比 SMSS 早得多——"第一"的定语不同 | `PS/PSINIT.C:304` 的 `PsIdleProcess`；本批 `RtlCreateUserProcess` 在顺序表尾巴 |
| 7 | "Phase 1 的盾牌提权后要降回" | INITOS.C 内没有降回——盾牌是 Phase 1 专用的；线程转任零页线程后按零页线程的优先级 0 跑 | `INITOS.C` 内仅 1018 行一处 `KeSetPriorityThread`；`MM/ZEROPAGE.C` 头注释 |
| 8 | "执行体分页是把对象管理器分出去" | 不是按子系统切，是按 PE 节切：只有被 `#pragma alloc_text(PAGE, …)` 标记进 PAGE 节的个别函数才可换出，热路径永远在非分页代码里；开开关的前提是模块表已含 boot 驱动、回读路径（文件系统 + 磁盘）已通 | `MM/MMINIT.C:4248` 的 `MiEnablePagingTheExecutive`（找 PAGE / .edata 节）；`OB/FASTREF.C`、`OB/OBCLOSE.C` 的 PAGE 标记实例 |
| 9 | "驱动线程退出要 LPC"（只引注释、没讲为什么） | LPC 是消息传递 IPC，对话两端任一端可中途死掉；每个线程退出都无条件走 `LpcExitThread` 清理（摘回复链、置退出标志、释放待回复消息），否则对端死等、对象泄露；LPC 未初始化时驱动线程正常退出即蓝屏（锁、对象类型、消息区全未建），故 Lpc 必须卡在 Ob Phase 1 之后、IoInitSystem 之前 | `PS/PSDELETE.C:1197`（`PspExitThread` 调 `LpcExitThread`）；`LPC/LPCCLOSE.C:355`；`LPC/LPCINIT.C:89` |
| 10 | "调度器运行在 idle 线程里 / KiIdleLoop 的四步就是调度器 / idle 优先级 0 最低怎么调度别人" | 调度器（dispatcher）不是线程，没有自己的优先级与栈；它是一组在 DISPATCH_LEVEL 运行的内核函数 + 就绪队列，借用当前执行流运行。KiIdleLoop 四步是 idle 线程的等活循环（DPC→NextThread→IdleSchedule→HAL 省电），选线程的是 KiSelectReadyThread、切上下文的是 SwapContext。优先级 0 是 idle 线程这个"被选对象"的属性——垫底恰恰使它成为兜底：任何真实线程都能抢它；调度器在 DISPATCH_LEVEL 运行，不参与线程优先级竞争 | `KE/I386/CTXSWAP.ASM` KiIdleLoop（DPC/NextThread/IdleSchedule/PpIdleFunction）；`KE/IDSCHED.C` KiSelectReadyThread；`KE/THREDSUP.C` KiRequestDispatchInterrupt；`KE/I386/KERNLINI.C:943-944` 开闸降 0 |
| 11 | "PFN 数据库是干嘛的、为什么需要、没有会怎样；加载器内存描述符是干嘛的；坏页怎么产生、'坏'是什么意思" | PFN 数据库 = 每物理页一张状态卡的数组 `MmPfnDatabase[PFN]`（MMPFN：PteAddress/ShareCount/ReferenceCount/OriginalPte/PteFrame/PageLocation）；没有它，分配、换页、共享、回收、清零、隔离全做不了。加载器描述符 = 交接清单（BasePage/PageCount/MemoryType），唯一同时知道固件内存地图与加载布局的一方；MM 逐条翻译建库（LoaderBad→坏页链表，OsloaderStack→PFN 建库时（Phase 0，INIT386.C 空闲链表分支）直接进空闲链表；OsloaderHeap→建库时标在用（default 分支）、Phase 1 尾 MmFreeLoaderBlock 里 free（其中映射注册表的页表页按共享计数排除，不 free））。"坏" = 存储单元物理损坏、不可靠，永久隔离（不计入可用页总数）；产生：启动期固件上报（LoaderBad 描述符）+ 运行期 RemovalRequested 退役（归还时改进坏页链表，置旗点不在树内） | `MM/MI.H` MMPFN（:1706）、MmBadPageListHead（:7478）；`PUBLIC/INTERNAL/BASE/INC/ARC.H` TYPE_OF_MEMORY；`MM/I386/INIT386.C:3013` LoaderBad 建库；`MM/PFNDEC.C:303`、`MM/PFNLIST.C:171`、`MM/PFNLIST.C:830` |
| 12 | "ERESOURCE 比自旋锁重量级，又能在低 IRQL 下等待——这不矛盾吗" | 不矛盾，两个维度："重"量机制（ERESOURCE 15 字段、内含自旋锁 + 信号量 + 事件 + 所有者表，建在自旋锁之上）；"轻"量争用代价（ERESOURCE 争用时阻塞让 CPU、不抬 IRQL；自旋锁争用时空转烧 CPU + 抬 IRQL 到 DISPATCH_LEVEL + 堵核）。自旋锁 = 结构轻、等待贵；ERESOURCE = 结构重、等待便宜；两者分工，不是替代 | `INC/EX.H:4558` ERESOURCE（含 SpinLock）；`EX/RESOURCE.C:82-83` 内部自旋锁、`EX/RESOURCE.C:2619` 等待路径 |
| 13 | "自旋锁的实现机理是什么，是硬件 CPU 本身提供的自旋锁吗" | 不是。硬件提供原子 RMW 指令（lock bts/xchg/cmpxchg）+ 缓存一致性 + 总线/缓存行锁定；自旋锁是软件用它们搭的：抢 = lock bts 原子测试置位；等 = 普通读自旋（TTAS，分两步防总线风暴）；放 = 写 0。软件另搭三样：IRQL 抬升（持锁不能被 DPC/调度打断）、排队公平性（queued spinlock，本地标志自旋）、UP 退化（单核下获取直接 return，只剩 IRQL） | `INC/MAC386.INC:61` ACQUIRE_SPINLOCK（lock bts）、`:111` SPIN_ON_SPINLOCK（普通读自旋）；`KE/I386/SPINLOCK.ASM` NT_UP 分支（UP 直接 return）；`KE/I386/CTXSWAP.ASM:278,1455` lock bts 实物 |
| 14 | "lock bts 读旧值并置 1，那读到 1 怎么分清是你置的还是原来就是 1；读到 0 下一步 test 又说不通" | 分得清：lock bts 一个指令内原子地做三件事——把 bit0 的旧值装进 CF、把 bit0 置 1、两步不可分；CF 装的是置位前的旧值，CF=0 ⟺ 抢到、CF=1 ⟺ 别人持有，两个核同时抢时硬件串行化、不可能都看到 0。test 只出现在"没抢到、正自旋"的分支里，且 test 不改内存、ZF 只是"值得再试一次"的提示、不是判决；test 看到 0 只是回去再做一次 lock bts，判决永远是那次 bts 的 CF，失败则回自旋。流程是带分支的循环，不是抢→等→放的线性三步 | `INC/MAC386.INC:61`（lock bts + jc）、`:111`（test + jz 回抢） |
| 15 | "自旋循环里的 YIELD 是让出 CPU 吗？让出了还自旋什么" | 不是。YIELD = pause 指令（F3 90，`INC/MAC386.INC:31`，UP 下为空），是给 CPU 微架构的提示，不是调度器参与的让 CPU：减推测执行惩罚、省电、超线程下把执行资源让一点给兄弟核（兄弟核上可能正跑着锁持有者）。执行完 YIELD 照样 jmp 回自旋，核仍在执行本线程、无线程切换。真正的让出（阻塞/切换）需要 dispatcher，在 DISPATCH_LEVEL 上不可能——这正是自旋锁存在的理由 | `INC/MAC386.INC:31-35` YIELD（db 0F3h, 090h）；`INC/MAC386.INC:111` SPIN_ON_SPINLOCK 循环 |
| 16 | "指令里的'锁字'是什么东西？一块内存地址吗" | 是，就是一个 KSPIN_LOCK 变量的地址，里面存着一个字（`typedef ULONG_PTR KSPIN_LOCK`，32 位上 4 字节、64 位上 8 字节，指令里的 dword/qword 跟着变）。bit0 = 0 空闲 / 1 持有（调试版里整个字还会写上持有线程指针，用于查谁拿着锁）。实物：全局变量（如 _KiFreezeExecutionLock）、结构体字段（如 PRCB 的 PrcbLock、ERESOURCE 的 SpinLock）——都是普通内存，所有核用同一个地址操作同一个字，靠缓存一致性看到同一个值。锁字必须在非分页内存里：拿锁时在 DISPATCH_LEVEL，缺页在 DISPATCH_LEVEL 上没法处理 | `PUBLIC/SDK/INC/NTKEAPI.H:65` KSPIN_LOCK；`KE/I386/NEWSYSBG.ASM:438` 全局锁字、`KE/I386/CTXSWAP.ASM:1455` PRCB 锁字段、`INC/EX.H:4574` ERESOURCE 锁字段 |
| 17 | "缓存一致性是什么，为什么自旋锁完全依赖它" | 每核有自己的缓存，读写的是副本；没有一致性，各核对同一地址看到不同值，自旋锁互斥彻底失效。一致性 = 单写者-多读者不变式 + 写传播 + 写串行化（最终一致、一旦见新不再见旧）。MESI 四状态（M 独占脏/E 独占净/S 共享/I 无效）+ 窥探/目录两种实现；一致性按缓存行走，故有伪共享，热锁字须独占一行（ALIGNED_SPINLOCK，MP 下 DECLSPEC_CACHEALIGN）。自旋锁正确性 = 一致性 × 原子性 × 屏障 × IRQL × 自旋策略。补定义（上条回答漏的）：TSO（Total Store Order，全存储顺序）= x86 的内存一致性模型（多地址顺序规则）：读与读、写与写、读之后的写都不乱序，唯一允许的是一个读越过它前面的一个写——因为写先进存储缓冲（核内暂存待写出数据的小队列），读不必等它落到全局可见；lock 前缀 = 在此模型上加一道"什么都不许越过我"的屏障。MESIF = MESI + F（Forward，指定一个共享者负责转发）；MOESI = MESI + O（Owned，允许脏共享、省一次写回）。内存屏障 = "访存不许越过"的界线，lock 前缀指令自带全屏障 | `INC/NTOSDEF.H:30-34` ALIGNED_SPINLOCK；`INC/KE.H:4584` KiDispatcherLock；`INC/MAC386.INC:61,111` 锁操作 |
| 18 | "CM 子系统在文件系统之前还是之后初始化" | 分两段：Part 1（CmInitSystem1，SYSTEM + HARDWARE）在文件系统之前（IoInitSystem 之前），因为 SYSTEM hive 映像由加载器读入内存（LoaderBlock->RegistryBase）、HARDWARE 是易失 hive，都不走磁盘文件；IoInitSystem 要读 SYSTEM hive 的驱动配置来加载启动驱动（含文件系统驱动），等文件系统起来再开 SYSTEM 是死循环。Part 2 在文件系统之后：SMSS 起、chkdsk 跑完、页面文件打开后调 NtInitializeRegistry，把内存 hive 绑定到磁盘文件、并打开 SOFTWARE/SAM/SECURITY/DEFAULT 等磁盘文件 hive（走 ZwCreateFile/ZwOpenFile，必须有文件系统） | `INIT/INITOS.C:1437` CmInitSystem1、`:1531` IoInitSystem；`CONFIG/CMSYSINI.C:1636` RegistryBase；`CONFIG/NTAPI.C:1030` NtInitializeRegistry 文档；`CONFIG/CMDAT.C:284` CmpMachineHiveList |
| 19 | "文件系统还没初始化，加载器怎么读磁盘上的 SYSTEM hive，难道加载器里有一份文件系统代码" | 是：加载器（ntldr）里有一份极简只读的文件系统代码（FAT + NTFS），只会按路径找文件、沿目录/簇链读数据，不写、不分配、不回放日志、不解压/解密。分层：固件（BIOS INT 13h）只认扇区号、不懂文件；文件→扇区的翻译在启动时由加载器的迷你文件系统代码做（NTFS：读引导扇区→找 $MFT→读文件记录→跟数据运行表→按簇读扇区）。正式 NTFS 驱动是内核驱动、需要执行体（MM/OB/IO/缓存），内核没起来它跑不了，所以必须另有一份启动时用的读法；用完就扔，加载器内存在 MmInitSystem(2) 被释放回收。ntldr 本体不在 WRK 树内，行号给不出；交接点树里钉死：LoaderBlock->RegistryBase、内存描述符、模块链表胚胎 | `CONFIG/CMSYSINI.C:1636` RegistryBase；`MM/I386/INIT386.C:2964` 内存描述符建 PFN DB；第 2 批 MmInitSystem(2) 释放加载器内存 |

---

## 第七部分：自查题（只给题，不给答案）

1. Phase 0 的约束为什么是"只能初始化本子系统的数据"？如果 Phase 0 里某个子系统想创建一个线程，会缺什么支撑机制？
2. `HalInitSystem` 为什么在 Phase 0 与 Phase 1 各调一次？两次的界线是"能用"与"全功能"——各举一个只能在 Phase 1 做的 HAL 初始化内容，并说明为什么 Phase 0 做不了。
3. NLS 表在启动期搬了三次家（加载器内存 → 池 → 页文件支撑的节）。每次搬家的触发条件是什么？为什么不能一步到位？
4. `KeStartAllProcessors` 为什么放在 Phase 1 的前段、而不是 Phase 0，也不是 Phase 1 的尾巴？拉 AP 之后为什么要立刻重读 RTC 时间？
5. `MmInitSystem(2)` 的注释是"Begin paging the executive if desired"——"执行体分页"分出去的到底是什么（按子系统还是按节）？为什么开开关要等到 IO 驱动潮之后（从模块表与回读路径两条讲）？`MmDisablePagingExecutive` 的两道闸分别对应什么场景？

---

## 附录：未钉死清单（树里查不到的，不编造）

1. `CmpInitSystemVersion(n, …)` 各编号（0、1、2、8、3、9、4、6、7）的具体语义：定义不在 WRK 树内（CONFIG 目录无此函数体）；本批只记录调用位置、顺序与参数（LoaderBlock / NULL / Options / MPKernelString / DataTableEntry），不编造每号的含义。
2. `MAXIMUM_PRIORITY` 的数值：DDK 常量原定义不在 WRK 树内；按 `DispatcherReadyListHead[MAXIMUM_PRIORITY]` 的数组维度（32 档，优先级 0–31）推断为 32，`MAXIMUM_PRIORITY - 1` 即最高档 31，与 HIGH_PRIORITY 同值——此为推断，标注如上。
3. 拉 AP 打乱系统时间的精确机制：`INITOS.C:1190` 注释只说 "starting processors has thrown off the system time"，未展开时钟中断归属/计数错乱的细节。
4. AMD64 为什么不用 `ExInitPoolLookasidePointers`：只有 `#if !defined(_AMD64_)` 条件编译，无注释说明。
5. `KiInitializeInterruptTimers` 为什么只在 x86：只有 `#if defined(_X86_)` 条件编译，无注释说明。
6. HAL 的 Phase 0 / Phase 1 各自的内部清单：HAL 本体不在 WRK 树（只有预编译 DLL）；本批只按调用点（`HalInitSystem(0/1)`、`HalAllProcessorsStarted`）陈述分工（UP 版 `HalAllProcessorsStarted` 的 HAL 内部处理亦在此列，见第 10 条）。
7. `PoInitSystem(0)` 与 `PoInitSystem(1)` 的分工细节：只知两阶段调用（`INITOS.C:1096` / `:1595`），内部清单留待电源相关批次展开。
8. `CmpInitSystemVersion(8, Options)` 里 Options（启动选项字符串）对 CM 检查的具体影响：未展开。
9. `PpInitSystem`（Phase 0 调用）排在 `PsInitSystem`（Phase 0 调用）之后的原因：`PpInitSystem` 的函数体在 WRK 树内按名只搜到调用点、没有函数体，无法从树内确认 Ps→Pp 的依赖；正文旧版用 Ob 的理由回答 Ps 的位置是答非所问，已纠。
10. UP 版 `HalAllProcessorsStarted` 的 HAL 内部处理：HAL 本体不在树内，不展开（只确定调用点 :1240 在 `#endif` 之后、UP 版照调）。

---

*下一批预告：第 3 批 MM 上线（MmInitSystem 三阶段）——PFN 数据库按 loader block 的内存描述符建起来、系统地址空间与页表自举、非分页池/分页池诞生、系统缓存与节对象就绪。本批的 MmInitSystem(0/1/2) 在此展开。*

## 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，与正文互补。

- **KiSystemStartup → ExpInitializeExecutive：普通函数调用**。`KE/I386/KERNLINI.C:888`，参数是 Phase 号与 LoaderBlock。Phase 0 与 Phase 1 是同一个函数的两个分支（按 `InitializationPhase` 分派），不是两个函数——“进 Phase 1”不是调另一个函数，是 Phase 1 线程跑同一函数的另一分支。
- **Phase 0 → Phase 1：靠一个新线程转岗，不是靠返回**。Phase 0 内 `PsCreateSystemThread` 创建 Phase 1 线程，起 `Phase1Initialization`（`PS/PSINIT.C:502-507`，参数即 LoaderBlock）→ 调度器跑它 → `Phase1InitializationDiscard`（`INIT/INITOS.C:934`）跑 Phase 1 体（即正文第 ⑰–㉛ 步的全部内容）→ 返回后，同一线程调 `MmZeroPageThread()`，转岗成零页线程。转移方式：线程创建 + 调度 + 函数调用；Phase 1 体跑完，线程不退出，而是换了个工作。
- **Phase 1 的跨核协调：集合确认，不是各跑一份**。Phase 1 体只在 P0 的 Phase 1 线程里跑；`KeStartAllProcessors`（`INITOS.C:1187`）叫醒 AP 后，`HalAllProcessorsStarted`（`:1240`）是集合确认（各核已启动），此后 AP 在 IdleLoop 里等调度，不各跑一份 Phase 1 体。转移方式：核间启动 + 屏障确认。
- **Phase 1 尾 → SMSS：函数调用链，不是跳转**。`Phase1InitializationDiscard` 返回 → `Phase1Initialization` 返回 → 线程转岗零页线程；SMSS 的创建在 Discard 体内（`RtlCreateUserProcess` → `ZwResumeThread`），是函数调用链上的一环，见第 8 批。
