---
layout: post
title: "WRK 第 8 批：从内核到用户世界——会话、Win32k 加载与第一个用户进程"
categories: WRK
---

取材说明：底本是 WRK 树（Windows Server 2003 SP1 代码线）`BASE/NTOS/` 下的 `INIT/INITOS.C`（Phase 1 尾）、`RTL/RTLEXEC.C`（`RtlCreateUserProcess`）、`PS/CREATE.C`（`PspCreateProcess` / `PspCreateThread` / `PspUserThreadStartup`）、`PS/PSQUERY.C`（`PsEstablishWin32Callouts` / `PsConvertToGuiThread`）、`PS/PSDELETE.C`（退出侧 callout）、`EX/SYSINFO.C`（会话映像装载入口）、`INC/PS.H`（callout 表结构）。**本批的边界比前几批更重要，先画清**：win32k.sys 本体、CSRSS、SMSS 的用户态代码都不在 WRK 树内——本批只读内核对接面（callout 表、转换函数、会话映像装载入口、进程创建链），用户态一侧（SMSS 如何建 CSRSS、CSRSS 如何触发 win32k 加载、Winlogon 登录流程）只做到架构级陈述，列入未钉死清单，不编造。

主题一句话：Phase 1 的尾巴用 `RtlCreateUserProcess` 捏出第一个用户进程 SMSS，SMSS 的首线程在内核启动例程里被排上一个用户 APC——线程第一次返回用户态时，先跑 ntdll 的 `LdrInitializeThunk`，再跑 exe 的入口点；随后 win32k 以会话空间驱动的身份注册一整张 callout 表，内核从此有了“Win32”这个概念，线程转 GUI 才成为可能。内核到用户世界的交棒，不是某一个函数调用，而是一条“建进程 → 起首线程 → 首跑加载器 → 注册 Win32”的链。

## 第一部分：术语表（按引用级排序，基础在前、派生在后）

### 一、进程与线程的内核对象

- **进程对象（EPROCESS）**：内核代表一个进程的对象，含进程控制块（PCB：页目录基址、亲和性、优先级、时间片）、地址空间、句柄表、令牌、配额、作业指针、映像名、PEB 指针等。第 5 批手建的 Idle/System 进程就是两个静态 EPROCESS；本批的 `PspCreateProcess` 建的是动态 EPROCESS（对象管理器分配，见第 4 批对象机制）。
- **线程对象（ETHREAD）**：内核代表一个线程的对象，含线程控制块（TCB：状态、优先级、等待、APC 队列、陷阱帧指针）、TEB 指针、起始地址、Win32 起始地址等。首线程的内核起点与用户起点是两个不同的字段（见第 ⑤ 步），不要混成一个。
- **PEB（进程环境块）**：用户态每个进程一份的结构，含映像基址、加载器数据（已加载 DLL 链表）、进程参数、堆信息等。内核在 `MmCreatePeb` 里建它（`CREATE.C:1565`），用户态的 ntdll 初始化时填充它。PEB 是内核与用户态加载器的交接面。
- **TEB（线程环境块）**：用户态每个线程一份的结构，含线程本地存储（TLS）数组、异常链表、locale、理想处理器等。首线程的 TEB 在 `PspUserThreadStartup` 里填 locale 与理想处理器（`CREATE.C:2040` 一带）。
- **CID（客户标识）**：进程 ID 与线程 ID 的一对数，第 5 批讲过由 `PspCidTable` 句柄表分配。进程创建时分 CID，线程创建时分 CID，两者是同一张表的两类项。
- **令牌（token）**：进程的安全上下文（用户 SID：安全标识符、组、特权），第 4 批讲过由 Se 注册类型。`PspCreateProcess` 里 `PspInitializeProcessSecurity`（`CREATE.C:1302`）从父进程复制/派生令牌——子进程的安全身份默认继承父进程，SMSS 的父进程是 System，令牌即系统令牌。
- **作业对象（job）**：一组进程的容器，可统一限配额、限优先级、收通知。进程创建时可指定加入作业（`JobMemberLevel` 参数）；作业的会话归属（`Job->SessionId`）是进程归属会话的线索之一（见第九节）。
- **调试端口 / 异常端口**：进程的两个 LPC 端口。调试端口供调试器收进程的调试事件（创建线程、异常等，`DbgkCreateThread` 走这里）；异常端口供未处理异常时找“最后处理人”（如 CSRSS 的异常处理）。SMSS 创建时两者都传 NULL（`INITOS.C:1777` 一带），即 SMSS 不被调试、无异常端口兜底。
- **配额（quota）**：进程可用的池内存与句柄数上限，防单个进程耗尽系统资源。进程创建时从父进程/系统默认值继承配额账本。

### 二、地址空间与映像

- **节对象（section）**：第 4 批讲过的对象类型之一，代表一段可映射内存的后备。用户进程的映像节是 `SEC_IMAGE` 类型（按 PE 格式映射），由 `RtlCreateUserProcess` 用 `ZwCreateSection` 建（`RTLEXEC.C:890` 一带），再以节句柄调 `ZwCreateProcess`——进程的地址空间就是从这个节长出来的。
- **映像信息（section image information）**：从节对象里查出的 PE 头摘要：入口点、栈预留/提交大小、子系统类型（原生/GUI/控制台）、DLL 特征等。`RtlCreateUserProcess` 用 `ZwQuerySection` 取它（`RTLEXEC.C:940` 一带），供后续建线程栈与 PEB 用。
- **系统地址空间（system address space）**：不依赖映像节、直接复用内核地址空间的进程创建方式。`PspCreateProcess` 的文档原话：父进程参数缺省时，“the process has no parent and is created using the system address space”——这就是第 5 批 System/Idle 手建分支的通用化描述。正常用户进程一定带节句柄，不走这条分支。
- **会话空间（session space）**：每个会话独占的一段系统地址空间范围，里面放该会话的会话页池、win32k 映像、会话结构等。内核侧的锚是 `MmSessionSpace`（`MM/MI.H` 一带）。为什么要有会话空间：多用户登录时，每个会话的 win32k 数据、GDI 句柄、窗口站必须隔离，又不能每个会话复制一份内核——会话空间就是“内核地址空间里按会话切分的租界”。
- **会话（session）**：一次登录会话的内核侧身份，含会话号（SessionId）、会话空间、win32k 映像、locale 等。会话 0 是控制台会话，SMSS、CSRSS 都在会话 0 里。进程归属哪个会话，由令牌/作业的 SessionId 决定（`PSJOB.C:1532` 一带的 `Job->SessionId` 即此线索）。

### 三、用户进程诞生链

- **RtlCreateUserProcess**：内核态的“建一个用户进程”的打包函数（`RTL/RTLEXEC.C:774`），文档原话：创建一个带单个线程的用户模式进程，线程挂起计数为 1，地址空间由指定映像文件初始化。它依次做：开映像文件 → 建映像节 → 调 `ZwCreateProcess` → 查映像信息与 PEB → 写进程参数与环境 → 建首线程（挂起）→ 返回进程与线程句柄。调用方随后 `ZwResumeThread` 放行首线程（`INITOS.C:1800` 一带对 SMSS 即如此）。
- **NtCreateProcess / PspCreateProcess**：`NtCreateProcess` 是系统服务入口（`PS/CREATE.C:816`），实干在 `PspCreateProcess`（`:967`）。它建 EPROCESS、初始化 PCB（`KeInitializeProcess`，`:1289`）、建安全上下文（`:1302`）、建地址空间（`MmInitializeProcessAddressSpace`，`:1366` / `:1394` / `:1448` 三分支：带节、克隆父进程、手建）、建 PEB（`MmCreatePeb`，`:1565`，仅父进程存在且需建 PEB 时）。
- **PspCreateThread**：建线程对象的函数（`PS/CREATE.C:243`）。它给线程定的内核起点是 `PspUserThreadStartup`（`:492`），用户起点是上下文里的程序计数器（存 `Thread->StartAddress`，`:468`），Win32 起点是上下文寄存器（x86 的 Eax / AMD64 的 Rdx，存 `Thread->Win32StartAddress`，`:472` / `:476`）。三个起点各司其职：内核起点是线程在内核态的第一站，用户起点是 exe 入口点，Win32 起点是 win32k 眼里的线程起始地址（供调试与窗口过程回溯用）。
- **挂起计数（suspend count）**：线程的“暂停次数”，建线程时为 1（不调度），`ZwResumeThread` 减 1 到 0 才可调度。为什么建时先挂起：进程参数、PEB、首线程上下文还没写完就调度，线程会跑到半成品状态；先挂起、写完、再放行，是进程创建的标配时序。
- **PspUserThreadStartup**：所有用户线程在内核态的统一第一站（`PS/CREATE.C:2003`）。它降到 `PASSIVE_LEVEL`（中断请求级别 IRQL 的 0 级，可等待、可缺页）、填 TEB（locale、理想处理器）、通知调试器（`DbgkCreateThread`）、排初始用户 APC（见下），最后处理系统范围 cookie（`SharedUserData->Cookie`，全局安全 cookie，0 时现算）。
- **LdrInitializeThunk**：ntdll 的加载器初始化入口（用户态函数）。内核侧不叫这个名字——它叫 `PspSystemDll.LoaderInitRoutine`，是第 5 批 `PspInitializeSystemDll` 从已映射 ntdll 里提取的入口点地址（`PSINIT.C:840` 一带）。两者是同一地址的两个叫法：ntdll 侧叫 `LdrInitializeThunk`，内核侧叫 `LoaderInitRoutine`。
- **初始用户 APC**：首线程在 `PspUserThreadStartup` 里排的用户 APC（`KiInitializeUserApc`，`CREATE.C:2115` 一带），参数是陷阱帧、异常帧、`LoaderInitRoutine`、ntdll 基址。为什么是 APC 而不是直接跳：线程从内核返回用户态的正规出口就是 APC 投递（第 5 批讲过的用户 APC 机制：`KiDeliverApc` → 改写陷阱帧 → ntdll 的 `KiUserApcDispatcher` → 跑例程 → `NtContinue` 恢复）。初始 APC 让“第一次进用户态先跑加载器”复用了同一条正规出口，不用为首线程另造一种进入方式。
- **PreviousMode（先前模式）**：线程上一次进入内核时的模式（用户态或内核态），决定系统服务是否做参数探查、APC 投递走用户队列还是内核队列。`PsConvertToGuiThread` 要求先前模式为用户态（内核线程不能转 GUI，见第 ⑦ 步）。

### 四、Win32 对接面

- **Win32 子系统**：Windows 的图形与窗口子系统，由三部分组成：用户态的 CSRSS（客户/服务器运行时子系统，管进程、线程、控制台、关机等）、内核态的 win32k.sys（窗口管理、GDI、消息、输入）、用户态的 user32/gdi32（API 壳）。内核本身（Windows NT 执行体）不知道窗口是什么——它只知道进程、线程、对象；“窗口”这个概念是 win32k 注册进来后才有的。
- **SMSS（会话管理器子系统）**：第一个用户进程（`smss.exe`），内核 Phase 1 尾建它（`INITOS.C:1777`）。它的用户态职责：建 CSRSS、建 Winlogon、管会话创建与删除、响应关机。SMSS 是“内核 → 用户世界”的第一棒，也是唯一由内核直接创建的用户进程（其余用户进程都由 SMSS/CSRSS/Winlogon 链式创建）。
- **CSRSS（客户/服务器运行时子系统）**：每个会话一个的 Win32 运行时进程，由 SMSS 创建。它的职责：进程/线程的 Win32 侧管理、控制台窗口、关机处理、异常端口兜底。CSRSS 的代码不在树内，本批只讲内核侧与它的接口（调试端口、异常端口、Win32 callout）。
- **win32k.sys**：Win32 的内核态部分，以驱动形式存在。为什么必须是驱动而不是普通 DLL：它要进内核地址空间（调内核例程、访问内核对象）、要注册对象类型（WindowStation/Desktop）、要填 callout 表、要提供系统服务表（影子服务表）。普通 DLL 进不了内核地址空间，办不到这四件事。
- **Win32 callout 表（WIN32_CALLOUTS_FPNS）**：win32k 向内核注册的 16 个函数指针（`INC/PS.H:1757`）：进程 callout、线程 callout、全局原子表 callout、电源事件 callout、电源状态 callout、作业 callout、批刷例程、Desktop 四过程（打开/可关闭/关闭/删除）、WindowStation 五过程（可关闭/关闭/删除/解析/打开）。注册函数是 `PsEstablishWin32Callouts`（`PS/PSQUERY.C:4300`），它把 16 个指针逐项存进内核全局变量（`PspW32ProcessCallout`、`PspW32ThreadCallout`、`ExDesktop*Callout`、`ExWindowStation*Callout` 等，`:4345` 一带）。
- **影子服务表（shadow service table）**：第二张系统服务分发表，专供 Win32k 服务（`NtUser*` / `NtGdi*`）。普通线程的服务表是 `KeServiceDescriptorTable`（NT 执行体服务）；转 GUI 后，线程的服务表指针换成影子表（AMD64：`Thread->Tcb.Win32kTable`；x86：`Thread->Tcb.ServiceTable`，`PSQUERY.C:4585` 一带）。为什么要两张表：NT 服务与 Win32 服务是两套编号体系，同一系统调用号在两张表里指不同的函数；线程用哪张表，决定它能调哪套服务。
- **GUI 线程转换（PsConvertToGuiThread）**：把普通线程变成 GUI（图形用户界面）线程的函数（`PS/PSQUERY.C:4470`）。步骤：先前模式须为用户态、callout 须已注册（否则 `STATUS_ACCESS_DENIED`）→ 换大内核栈（GUI 线程的内核栈要更大，窗口/GDI 调用栈深）→ 调进程 callout（`Process, TRUE`）→ 换影子服务表 → 调线程 callout（`Initialize`）。退出侧在 `PS/PSDELETE.C:1084` / `:1088`：线程退出调线程 callout（`Exit`），进程退出调进程 callout（`FALSE`）。
- **WindowStation / Desktop**：Win32 的两个对象类型。WindowStation 是“一个登录会话的交互边界”（含剪贴板、原子表、桌面集合），Desktop 是“一张虚拟屏幕”（窗口、菜单、钩子的容器）。两者都由 win32k 创建对象类型（`ObCreateObjectType`，第 4 批机制），类型过程（解析/打开/关闭/删除）就是 callout 表里的 Desktop/WindowStation 五加四过程——内核的对象管理器在解析这些对象时，调的是 win32k 注册的过程，内核自己不知道窗口站与桌面是什么。
- **全局原子表（global atom table）**：全系统共享的字符串 → 整数标识（atom）映射表，供剪贴板格式、窗口类名等跨进程复用字符串。内核侧的取表函数就是 callout 表里的 `GlobalAtomTableCallout`（存 `ExGlobalAtomTableCallout`），内核需要原子表时问 win32k 要，不自己维护。
- **GDI 批刷（batch flush）**：GDI（图形设备接口）调用先攒成一批（减少用户/内核切换），攒满或显式要求时一次刷进内核。刷函数就是 callout 表里的 `BatchFlushRoutine`（存 `KeGdiFlushUserBatch`）。为什么内核要存这个指针：内核在某些点（如线程切换、系统调用返回）要替用户态把未刷的 GDI 批处理掉，刷函数在 win32k 里，内核只存指针、不实现。
- **KeUserModeCallback**：内核主动调用户态代码的通道（`KE/I386/CALLBACK.C` 与 `KE/AMD64/CALLBACK.C`），方向与 APC 相反：APC 是“内核排队、用户态跑”，回调是“内核在系统调用上下文里直接切到用户态跑一段、再回来”。Mandt 全译讲的用户回调攻击面（用户态回调例程被利用提权）走的就是这条通道的用户侧，本批只在对位小结点名。

### 五、会话

- **会话 locale**：会话的区域设置标识（语言、度量衡等），首线程 TEB 的 `CurrentLocale` 从 `MmGetSessionLocaleId()` 取（`CREATE.C:2040` 一带）。为什么 locale 是会话级的不是系统级的：多用户登录时，不同会话的用户语言可以不同，locale 跟会话走，不跟机器走。
- **RTL_USER_PROC_RESERVE_1MB**：SMSS 进程参数的一个标志（`INITOS.C:1665` 一带）：在 SMSS 地址空间里预留低 1MB。注释原话：安装程序用 SMSS 的替代品启动、要用到 VGA（视频图形阵列）驱动的 int10，而 int10 要求低 1MB 预留——代价极低，索性每次都预留。为什么 SMSS 要操心 VGA：启动期的显示（进度条、蓝屏、chkdsk 输出）走 VGA，SMSS 是第一个能碰显示的用户进程。

## 第二部分：正文

### ① Phase 1 尾：从 85% 到 100%，交棒前的最后一串检查点

`ExpInitializeExecutive` 的 Phase 1 尾巴（`INITOS.C:1595` 一带起）按以下顺序收尾，进度条是这里的节拍器：

1. `PoInitSystem(1)`（电源管理第二阶段），失败 → `INTERNAL_POWER_ERROR`。
2. `PsInitSystem(1, LoaderBlock)`（:1604），失败 → `PROCESS1_INITIALIZATION_FAILED`。注释原话：“Okay to call PsInitSystem now that \\SystemRoot is defined so it can locate NTDLL.DLL and SMSS.EXE.”——`PsInitSystem(1)` 要定位 ntdll 与 SMSS，前提是 `\SystemRoot` 已在 Io 波尾巴重指好（第 7 批第 ⑪ 步），依赖链又一次咬合。
3. 进度 85%。`KeLoaderBlock` 清掉（调试器从此看 `PsLoadedModuleList`，不再看加载器块）、`MmFreeLoaderBlock` 释放加载器内存（第 3 批讲过的释放点，至此加载器块彻底退出历史舞台）。
4. `SeRmInitPhase1()`（引用监视器命令服务线程，System 进程的常驻线程，建 LPC 端口收 LSA（本地安全机构）的审计命令），失败 → `REFMON_INITIALIZATION_FAILED`。进度 90%。
5. 捏 SMSS 进程参数（见第 ② 步）、`CmpInitSystemVersion(7, NULL)`（记版本 7）。
6. `RtlCreateUserProcess` 建 SMSS（:1777），失败 → `SESSION3_INITIALIZATION_FAILED`；`ZwResumeThread` 放行首线程，失败 → `SESSION4_INITIALIZATION_FAILED`。进度 100%，开调试输出（为接下来的 chkdsk 显示做准备）。
7. **等 SMSS 5 秒**（:1815 一带）：`ZwWaitForSingleObject(Process, 5 秒)`。注释原话：“Wait five seconds for the session manager to get started or terminate. If the wait times out, then the session manager is assumed to be healthy.”——超时 = SMSS 还活着 = 健康，继续；5 秒内 SMSS 进程对象被触发（即 SMSS 退出了）→ `SESSION5_INITIALIZATION_FAILED` 蓝屏。为什么等 5 秒而不是不等：SMSS 一起就撒手不管，SMSS 若立即崩溃，系统会带着一个死掉的会话管理器继续跑、随后在更难定位的地方出错；5 秒窗口是“SMSS 能活过最初的初始化”的最低健康证明。超时不等于是“SMSS 已就绪”，只是“没死”，真正的就绪由 SMSS 自己的后续步骤保证。
8. 关进程与线程句柄、释放参数内存、`InitializationPhase += 1`、x86 下 `KiInitializeInterruptTimers()`。`ExpInitializeExecutive` 返回，Phase 1 线程的使命结束——内核初始化完成，控制权交给 SMSS 链。

### ② SMSS 参数捏制：第一个用户进程的“出生证明”怎么写

SMSS 的进程参数在 `INITOS.C:1645` 一带起现捏，逐项都有讲究：

- **参数块大小**：`sizeof(RTL_USER_PROCESS_PARAMETERS) + DOS_MAX_PATH_LENGTH × 6 × sizeof(WCHAR)`——6 个路径槽（当前目录、DLL 路径、映像路径、命令行等各一个 `DOS_MAX_PATH_LENGTH`）。为什么按最大路径预留：SMSS 的参数里要放好几条完整路径，现算长度容易溢出，按最大路径一次给足最省事。
- **低 1MB 预留**：`ProcessParameters->Flags = RTL_USER_PROC_PARAMS_NORMALIZED | RTL_USER_PROC_RESERVE_1MB`（术语表第五节）。`NORMALIZED` 表示参数里的指针已是自包含偏移（参数块要跨进程传，指针得先归一化，接收方再反归一化——`RtlDeNormalizeProcessParams` 在 `RtlCreateUserProcess` 调用点，`INITOS.C:1780` 一带）。
- **当前目录与 DLL 路径**：当前目录 = `NtSystemRoot`（即 `\SystemRoot`，系统盘 Windows 目录）；DLL 路径 = 当前目录 + `\System32`。为什么 DLL 路径先定好：SMSS 一起就要加载 DLL（ntdll、kernel32 等），加载器按 DLL 路径找 DLL，路径错了第一个 DLL 就找不到。
- **映像路径名**：从 `NtInitialUserProcessBuffer` 取，第一个空格即截断（注释原话：带空格的可执行名不支持，就当空格是映像名终结）。`NtInitialUserProcessBuffer` 的默认值是 `\SystemRoot\System32\smss.exe`（`INITOS.C:167`），类型须为 `REG_SZ`、长度须合法，否则 `SESSION2_INITIALIZATION_FAILED`（`:1715` 一带）。默认值可改的覆盖来源在树内未找到，列入未钉死清单第 1 条——不要把“注册表某键可改 SMSS”写成树内事实。
- **环境变量**：只放三条：`Path=<SystemRoot>\System32`、`SystemDrive=<系统盘符>`、`SystemRoot=<NtSystemRoot>`。为什么只有三条：SMSS 的环境是最小可用集，完整环境由 SMSS 自己在用户态展开（读注册表、建用户环境块）；内核只给够启动的最低限度。
- **命令行**：整段 `NtInitialUserProcessBuffer` 内容（含空格后的参数部分）——映像名截断到空格，命令行保留全文，SMSS 的参数（如 `/setup` 之类的安装开关）就从这里来。

### ③ RtlCreateUserProcess 全链：从一个路径名到一个挂起的首线程

`RtlCreateUserProcess`（`RTL/RTLEXEC.C:774`）的文档原话已在术语表第三节引过，全链如下：

1. **开映像文件**：`RtlpOpenImageFile` 按路径开 exe 文件（`RTLEXEC.C:850` 一带）。为什么先开文件再建节：节对象的后备就是这个文件，文件句柄是建节的入参。
2. **建映像节**：`ZwCreateSection(SECTION_ALL_ACCESS, PAGE_EXECUTE, SEC_IMAGE, File)`（`:890` 一带），随后关文件句柄——节对象已引用文件，句柄可关（第 7 批讲过的节引用计数规则在此复用）。
3. **建进程**：`ZwCreateProcess`，父进程缺省时默认当前进程（`RTLEXEC.C:905` 一带：`if (!ARGUMENT_PRESENT(ParentProcess)) ParentProcess = NtCurrentProcess()`）。SMSS 的情形：`INITOS.C:1777` 传 NULL → 父进程 = 当前进程 = Phase 1 线程所在的 System 进程。为什么 SMSS 的父进程是 System：Phase 1 尾跑在 System 进程上下文里，SMSS 从它派生，令牌、配额、亲和性都从 System 继承——第一个用户进程的安全身份就是系统身份。
4. **查映像信息与 PEB**：`ZwQuerySection(SectionImageInformation)` 取入口点、栈大小、子系统类型等（`:940` 一带）；`ZwQueryInformationProcess(ProcessBasicInformation)` 取 PEB 基址（`:955` 一带）。为什么要查两遍：节信息是“映像长什么样”，PEB 基址是“进程的 PEB 在哪”，前者供建线程用，后者供写参数用。
5. **写进程参数与环境**：把调用方给的进程参数块与环境块写进新进程地址空间（参数块先在调用方进程里建好、再跨进程写，归一化/反归一化即为此服务）。标准输入/输出句柄如有指定，用 `ZwDuplicateObject` 复制进新进程（`:975` 一带）。
6. **建首线程（挂起）**：调 `ZwCreateThread`，线程挂起计数为 1（文档原话，见术语表第三节）。返回进程句柄、线程句柄、映像信息。调用方 `ZwResumeThread` 后，首线程才可调度——SMSS 的放行点就是 `INITOS.C:1800` 一带。

### ④ PspCreateProcess 全链：建一个进程对象要办的事

`PspCreateProcess`（`PS/CREATE.C:967`）是 `NtCreateProcess` 的实干，建一个进程对象要办的事按执行顺序是：

1. **验参与父进程**：非法标志位直接 `STATUS_INVALID_PARAMETER`；父进程句柄如有，`ObReferenceObjectByHandle(PROCESS_CREATE_PROCESS)` 取 EPROCESS（`:1090` 一带）。为什么要 `PROCESS_CREATE_PROCESS` 权限：建子进程是父进程对象上的一种操作，权限检查在对象管理器层（第 4 批）统一做。
2. **建 EPROCESS 对象**：`ObCreateObject(PsProcessType)`（第 4 批对象机制），随后 `KeInitializeProcess` 初始化 PCB（`:1289`）：页目录、亲和性、基础优先级、时间片——第 5 批手建 Idle/System 时手工填的字段，在这里由函数统一填。
3. **建安全上下文**：`PspInitializeProcessSecurity(Parent, Process)`（`:1302`），从父进程派生令牌。无父进程时走系统令牌路径（手建分支，第 5 批）。
4. **建地址空间**：`MmInitializeProcessAddressSpace` 三分支（`:1366` / `:1394` / `:1448`）：带节句柄 → 按映像节建地址空间（正常用户进程）；无节句柄 → 克隆父进程地址空间；无父进程 → 系统地址空间（手建分支）。为什么分三支：三种创建语义（新映像、克隆、内核手建）对应三种地址空间来源，混用会错。
5. **建 PEB**：父进程存在且需建 PEB 时，`MmCreatePeb(Process, &InitialPeb, &Process->Peb)`（`:1565`）。为什么 PEB 创建有条件：手建进程（无父进程）没有用户态 PEB 可言，PEB 是用户进程的专属结构。
6. **句柄表、作业、配额、CID、映像名**：句柄表新建或继承（`InheritHandles` 参数）；作业归属按 `JobMemberLevel`；配额账本初始化；CID 分配（第 5 批的 `PspCidTable`）；映像文件名从节对象取。进程创建通知例程（`PspCreateProcessNotifyRoutine`，驱动可注册的进程创建回调，`:579` 一带）在此链中被调——安全驱动监视进程创建的钩子就在这里，与 Win32 callout 是两套不同的回调（前者给所有驱动，后者只给 win32k）。
7. **不做的事**：`PspCreateProcess` **不调** Win32 进程 callout（`PspW32ProcessCallout`）。进程创建时 win32k 的进程对象还没建——建它的时机是线程转 GUI 时（第 ⑦ 步）。把“进程创建”与“Win32 进程对象创建”当成同一件事，是本批最容易踩的混淆，见易混点第 1 条。

### ⑤ 首线程全链：从 PspUserThreadStartup 到 LdrInitializeThunk

首线程的诞生分内核侧三段与用户侧一段：

1. **建线程**：`PspCreateThread` 给线程定三个起点（术语表第三节）：内核起点 `PspUserThreadStartup`（`:492`）、用户起点 `StartAddress`（上下文程序计数器，`:468`）、Win32 起点 `Win32StartAddress`（上下文寄存器，`:472` / `:476`）。线程挂起计数为 1，等待 `ZwResumeThread`。
2. **调度与内核第一站**：`ZwResumeThread` 后，线程被 `KiReadyThread` 挂上就绪队列（第 5 批机制）、选中后跑 `PspUserThreadStartup`（`CREATE.C:2003`）。它先 `KeLowerIrql(PASSIVE_LEVEL)`，再填 TEB（`CurrentLocale = MmGetSessionLocaleId()`、理想处理器，`:2040` 一带）、通知调试器（`DbgkCreateThread`，除非线程已死或对调试器隐藏）、处理创建失败（`PS_CROSS_THREAD_FLAGS_DEADTHREAD` 时直接终结线程）。
3. **排初始用户 APC**：升到 `APC_LEVEL` 后，`KiInitializeUserApc(陷阱帧, 异常帧, PspSystemDll.LoaderInitRoutine, NULL, PspSystemDll.DllBase, NULL)`（`:2115` 一带），再降回 `PASSIVE_LEVEL`。注释原话：“All threads start with an APC at LdrInitializeThunk.”——**所有**用户线程（不只是首线程）都以这个 APC 起步，这是用户线程的统一出生仪式。
4. **首次返回用户态**：线程从内核返回用户态时，APC 投递机制（第 5 批：`KiDeliverApc` → 改写陷阱帧 → ntdll 的 `KiUserApcDispatcher`）先跑 `LdrInitializeThunk`（ntdll 基址即 `DllBase` 参数）。`LdrInitializeThunk` 在用户态做的事（ntdll 代码，不在树内，只讲接口）：建堆、加载导入 DLL、跑 DLL 入口与 TLS 回调、处理进程参数——全部完成后调 `NtContinue`，陷阱帧恢复为 exe 入口点，线程才真正跑 exe 的代码。**为什么第一条用户指令是 ntdll 不是 main**：exe 的入口点是 PE 头里的一个地址，但跑它之前，加载器必须先把 exe 依赖的 DLL 全加载、把 PEB 填好、把 TLS 建好；这些事只能由 ntdll 的加载器做，exe 自己做不了——因为 exe 的代码还没跑，跑它的前提就是加载器已就绪。鸡生蛋的解法与第 2 批加载器迷你文件系统同构：内核先把 ntdll 映射好、把入口点地址交给首线程，首线程的第一件事就是跑加载器。

### ⑥ Win32 callout 表：16 个指针，一张“内核不知道 Win32”的声明

`WIN32_CALLOUTS_FPNS`（`INC/PS.H:1757`）的 16 个指针按用途分四组：

- **生死组**：进程 callout（创建/删除）、线程 callout（初始化/退出）、作业 callout（作业状态变化或进程加入作业）。调用点：进程创建**转换**时（`PSQUERY.C:4574`，`TRUE`）、进程退出时（`PSDELETE.C:1088`，`FALSE`）、线程转换时（`PSQUERY.C:4603`，`Initialize`）、线程退出时（`PSDELETE.C:1084`，`Exit`）、作业变化时（`PSJOB.C:433` / `:707` / `:1960`，经 `PspWin32SessionCallout` 按会话分发）。
- **系统服务组**：全局原子表 callout（存 `ExGlobalAtomTableCallout`）、批刷例程（存 `KeGdiFlushUserBatch`）、电源事件与电源状态 callout（存 `PopEventCallout` / `PopStateCallout`）。内核在需要原子表、刷 GDI 批、电源事件时，调的是这些指针——实现全在 win32k 里。
- **对象过程组**：Desktop 四过程（打开/可关闭/关闭/删除）与 WindowStation 五过程（可关闭/关闭/删除/解析/打开），存 `ExDesktop*Callout` 与 `ExWindowStation*Callout`。对象管理器在创建/打开/关闭/删除 WindowStation 与 Desktop 对象时，调这些过程——类型过程是对象类型的标准钩子（第 4 批），win32k 只是给出了 WindowStation/Desktop 这两类的新实现。
- **注册函数**：`PsEstablishWin32Callouts`（`PSQUERY.C:4300`）把 16 个指针逐项存进上述全局变量后返回，无返回值、无失败路径——注册即生效，没有“注册一半”的状态。谁调它：win32k 的 `DriverEntry`（win32k 代码，不在树内，列入未钉死清单第 2 条）。文档注释说注册函数还“报告结构大小”，但本版结构里没有大小字段——注释与结构的小漂移，实测以结构为准。

为什么是“表”而不是“一个个注册函数”：16 个指针一次交齐，内核要么全有 Win32、要么全无——`PsConvertToGuiThread` 检查的是 `PspW32ProcessCallout` 非空（`PSQUERY.C:4507`），一个指针即代表整张表已注册。半注册的 Win32（进程 callout 有了、桌面过程没了）是不存在的状态，表结构本身排除了它。

### ⑦ GUI 转换：线程怎么变成“能调 Win32 服务”的线程

`PsConvertToGuiThread`（`PS/PSQUERY.C:4470`）是用户线程获得 Win32 能力的唯一入口（系统服务 `NtUser*` 的前置门），全链如下：

1. **门禁两项**：调用线程的先前模式须为用户态（内核线程调它返回 `STATUS_INVALID_PARAMETER`，`PSQUERY.C:4501` 一带）；`PspW32ProcessCallout` 须非空（win32k 未注册返回 `STATUS_ACCESS_DENIED`，`:4507`）。为什么两项都硬：内核线程没有用户态 Win32 上下文可言；win32k 未起时，影子服务表指向的服务实现不存在，切过去即空指针。
2. **换大内核栈**：线程尚未用大栈时，`MmCreateKernelStack(TRUE, NodeNumber)` 建大栈、`KeSwitchKernelStack` 切换（`:4530` 一带），切换期间禁内核 APC（`KeEnterGuardedRegionThread`）——切换内核栈时复制基础陷阱帧，上下文一变就全错，故禁 APC 是保护切换的手段。为什么 GUI 线程要大栈：窗口过程、GDI 调用、消息循环的内核侧调用栈比普通系统服务深，小栈会溢出。
3. **调进程 callout**：`(PspW32ProcessCallout)(Process, TRUE)`（`:4574`）。TRUE = 创建/初始化方向。失败则直接返回失败，影子表不切——进程的 Win32 对象没建好，线程不能先切过去。
4. **切影子服务表**：AMD64 写 `Thread->Tcb.Win32kTable / Win32kLimit`，x86 写 `Thread->Tcb.ServiceTable`（`:4585` 一带），指向 `KeServiceDescriptorTableShadow`。从此该线程的系统调用号查影子表，`NtUser*` / `NtGdi*` 才有实现。
5. **调线程 callout**：`(PspW32ThreadCallout)(Thread, PsW32ThreadCalloutInitialize)`（`:4603`）。失败则把服务表切回去（`:4605` 一带）——转换是原子的，要么全成、要么全回滚，不留“服务表切了、线程对象没建”的半成品。
6. **退出侧**：线程退出时 `PspW32ThreadCallout(Thread, PsW32ThreadCalloutExit)`（`PSDELETE.C:1084`），进程退出时 `PspW32ProcessCallout(Process, FALSE)`（`:1088`）。FALSE = 删除方向，与创建时的 TRUE 配对。

关键纠正（与第 ④ 步第 7 点呼应）：Win32 的进程对象与线程对象**在转换时建，不在进程/线程创建时建**。一个进程可以有多个线程，其中只有转过 GUI 的线程有 Win32 线程对象；一个进程的 Win32 进程对象在第一个线程转换时建。进程创建与 Win32 化是两件分开的事，前者是执行体的事，后者是 win32k 的事。

### ⑧ win32k 作为会话空间驱动：为什么它必须是驱动、每会话一份

win32k 的装载走第 7 批的会话空间路径：

- **装载入口**：`MmLoadSystemImage` 的 `MM_LOAD_IMAGE_IN_SESSION` 标志（`INC/MM.H:1017`，第 7 批第 ⑧ 步已钉流程）。树内能钉的调用点是 `NtSetSystemInformation` 的一个分支（`EX/SYSINFO.C:3456` 一带）：前置模式为内核时，直接以会话标志调 `MmLoadSystemImage` 装指定映像。用户态是谁调的这个系统信息类（SMSS 侧代码不在树内），列入未钉死清单第 3 条。
- **每会话一份**：会话标志装载的映像，每个会话一份副本——代码页共享、数据页写时复制（第 7 批第 ⑧ 步第 6 点的注释原话）。为什么：win32k 的数据（窗口、GDI 句柄、桌面）是会话私有的，代码（窗口过程实现、GDI 绘图）是共有的；共享代码、复制数据是最省内存的隔离方式。
- **注册两件事**：win32k 的 `DriverEntry`（树外）做两件内核侧可见的事：调 `PsEstablishWin32Callouts` 注册 16 指针（第 ⑥ 步），以及用 `ObCreateObjectType` 创建 WindowStation 与 Desktop 对象类型（第 4 批机制），类型过程即 callout 表里的 Desktop/WindowStation 过程。内核侧只看到全局变量被填上、对象类型表里多了两类——类型是谁建的、过程实现在哪，树内无代码，标界。
- **为什么必须是驱动**：四条理由缺一不可——进内核地址空间（调 `ObCreateObjectType`、`PsEstablishWin32Callouts` 都是内核例程）、注册对象类型（对象类型表是内核结构）、填 callout 表（全局变量在内核数据段）、提供系统服务表（影子表是内核的分发表）。普通用户态 DLL 一条都办不到；win32k 以驱动身份进内核，才办得到这四件。

### ⑨ 会话：会话空间里到底装了什么

会话的内核侧由三部分组成：

- **会话空间**：`MmSessionSpace` 指向的结构（`MM/MI.H` 一带），含会话页池、工作集锁、调试计数等。会话空间是系统地址空间里按会话切分的区域，切换进程时若跨会话，页目录与会话空间映射一起换（第 3 批的页表机制在此复用，只是换的范围含会话区）。
- **会话私有数据**：win32k 映像（每会话一份，见第 ⑧ 步）、会话 locale（首线程 TEB 的 `CurrentLocale` 即从会话 locale 取，`CREATE.C:2040` 一带）、会话的 WindowStation/Desktop 对象（win32k 在会话空间里建）。
- **归属**：进程归属哪个会话，由令牌或作业的 SessionId 决定（`PSJOB.C:1532` 一带的 `Job->SessionId`）。SMSS 在会话 0 里，CSRSS、Winlogon、登录后的 Explorer 都在各自会话里——会话是“登录边界”的内核实现：不同会话的窗口、剪贴板、GDI 句柄互不可见，靠的正是会话空间隔离与对象命名空间隔离（WindowStation 名前挂会话号，win32k 侧实现，树外）。

会话创建与删除的用户侧代码（SMSS 的会话管理）在树内没有对应物，本批只讲到“会话在内核侧长什么样”，会话怎么生、怎么死列入未钉死清单第 4 条。

### ⑩ 交棒总时序：从内核 Phase 1 尾到用户登录，树内与树外分界线

| 次序 | 步骤 | 树内 / 树外 |
|---|---|---|
| 1 | Phase 1 尾：85%→90%→100%，SMSS 参数捏制 | 树内（`INITOS.C:1595` 一带） |
| 2 | `RtlCreateUserProcess` 建 SMSS，`ZwResumeThread` 放行，5 秒健康等待 | 树内（`INITOS.C:1777` 一带） |
| 3 | SMSS 首线程：`PspUserThreadStartup` → 初始用户 APC → `LdrInitializeThunk` → exe 入口点 | 内核侧树内，用户态 ntdll 树外 |
| 4 | SMSS 建 CSRSS（会话 0 的 Win32 运行时进程） | 用户态 SMSS 代码树外，架构级陈述 |
| 5 | win32k 以会话空间驱动装载，注册 callout 表、建 WindowStation/Desktop 类型 | 装载入口树内（`SYSINFO.C:3456` 一带），win32k 本体树外 |
| 6 | CSRSS 的线程转 GUI（`PsConvertToGuiThread`），Win32 服务可用 | 树内（`PSQUERY.C:4470`） |
| 7 | SMSS 建 Winlogon → 用户登录 → 建用户会话 → Explorer | 用户态代码树外，架构级陈述 |

分界线的意义：1–3 步与 6 步是内核能自证的；4、5、7 步的用户态部分，树内只有调用点与接口，没有实现——本批不把架构级陈述写成源码事实。

### ⑪ 鸡生蛋再解一次：用户进程的第一条指令为什么是 ntdll

- **结**：用户进程要 ntdll（用户态加载器）才能跑（加载 DLL、填 PEB、建堆），但 ntdll 本身也是一个要加载的 DLL——谁加载加载器？
- **打破循环的办法**：内核先行一步。第 5 批：`IoInitSystem` 里 `PsLocateSystemDll` 把 ntdll 映射进系统空间、`PspInitializeSystemDll` 提取它的入口点（`LdrInitializeThunk`）与基址，存进 `PspSystemDll`。建用户进程时，内核不需要“加载 ntdll”——ntdll 的映像节早已由 `RtlCreateUserProcess` 的 `ZwCreateSection(SEC_IMAGE)` 从文件建好、映射进新进程地址空间；首线程的初始 APC 直接指向已知的入口点地址。
- **打破后的局面**：首线程第一次返回用户态，跑的不是 exe 入口点，而是 ntdll 的加载器初始化；加载器把 exe 依赖的 DLL 全加载、PEB 填好、TLS 建好后，才调 exe 入口点。exe 的 `main`（或 `WinMain`）是加载器初始化完成后的最后一跳，不是第一跳。
- **与第 2 批的同构**：加载器迷你文件系统打破的是“要文件系统才能读内核、要内核才能跑文件系统”的循环；`PspSystemDll` 打破的是“要加载器才能跑进程、要进程才能跑加载器”的循环。两次打破用的都是同一招：内核侧先准备好最小可用品（迷你文件系统 / 已映射 ntdll + 入口点），用户侧 / 内核文件系统侧再在此基础上长出完整版。

## 第三部分：32 / 64 位对照

- **PEB 双份（64 位独有）**：`PspCreateProcess` 的局部变量里有 `INITIAL_PEB32`（`CREATE.C:1032` 一带，`#if defined(_WIN64)`），供 WOW64 进程用——32 位进程在 64 位内核上跑时，有 64 位 PEB（内核看）与 32 位 PEB（32 位 ntdll 看）两份，32 位加载器读的是 32 位那份。32 位内核无此分支，一份 PEB 到底。
- **影子服务表字段名不同**：AMD64 写 `Thread->Tcb.Win32kTable / Win32kLimit`，x86 写 `Thread->Tcb.ServiceTable`（`PSQUERY.C:4585` 一带）。为什么：AMD64 的线程控制块把 Win32 服务表拆成独立字段（64 位服务表项宽、限制值另存），x86 直接复用服务表指针字段——字段名不同，语义相同（“本线程查哪张服务表”）。
- **首线程 APC 机制相同**：`PspUserThreadStartup` → `KiInitializeUserApc` → `KiUserApcDispatcher` → `NtContinue` 的链两边同一套，只差陷阱帧布局（x86 与 AMD64 的上下文结构不同）与 `Win32StartAddress` 的来源寄存器（x86 的 Eax、AMD64 的 Rdx，`CREATE.C:472` / `:476`）。
- **内核栈切换的帧指针要求（x86 独有）**：`PsConvertToGuiThread` 在 x86 下关帧指针省略（`#pragma optimize("y",off)`，`PSQUERY.C:4465` 一带），因为 `KeSwitchKernelStack` 依赖 EBP（帧指针寄存器）帧建栈；AMD64 无此要求（调用约定与栈回溯机制不同）。

## 第四部分：单核 / 多核对照

- **Phase 1 尾是单线程串行**：SMSS 创建走在 Phase 1 线程（System 进程的初始线程）上下文里，全程单线程、无并发——单核与多核在此处无差异，`InitializationPhase` 的递增、加载器块的释放都不需要锁。
- **首线程调度走第 5 批机制**：`ZwResumeThread` 后，首线程经 `KiReadyThread` 进就绪队列、由调度器选中（单核：队列即全部；多核：按亲和性与理想处理器分核，`IdealProcessor` 在 `PspUserThreadStartup` 里写进 TEB 即为此服务）。
- **内核栈切换禁内核 APC**：`PsConvertToGuiThread` 换栈期间 `KeEnterGuardedRegionThread` 禁内核 APC（`PSQUERY.C:4545` 一带）。单核下禁 APC 是“本核不被抢”，多核下还是“本线程不被抢”——禁的是本线程的 APC 投递，与核数无关，只与“切换期间上下文不能变”有关。
- **会话空间切换随 CR3**：跨会话进程切换时，页目录基址寄存器（CR3）一换，会话空间映射跟着换——单核下换 CR3 即换全部，多核下每核各换各核的 CR3，会话隔离不靠核、靠页表。

## 第五部分：逆向 / 安全解读

- **SMSS 是信任链的第一环**：内核直接创建的唯一用户进程，它的映像路径默认值是 `\SystemRoot\System32\smss.exe`（`INITOS.C:167`），类型与长度校验在 `:1715` 一带。SMSS 一旦被替换或劫持，其后整个用户世界（CSRSS、Winlogon、登录会话）都在被劫持的链上——启动链的信任锚从加载器、内核一路传到 SMSS，SMSS 是锚落地的第一站。
- **Win32 callout 未注册 = 无 GUI 的硬门**：`PsConvertToGuiThread` 的 `STATUS_ACCESS_DENIED`（`PSQUERY.C:4507`）意味着 win32k 没起时，任何线程都成不了 GUI 线程、调不了 `NtUser*` 服务。防御侧看，这是“图形子系统未就绪则图形攻击面不存在”的硬保证；攻击侧看，win32k 注册前的窗口期没有 Win32 服务可打。
- **进程创建通知 vs Win32 callout 是两套钩子**：`PspCreateProcessNotifyRoutine`（驱动可注册，所有进程创建都通知，`CREATE.C:579` 一带）是安全驱动监视进程创建的钩子；Win32 callout 只给 win32k。EDR（端点检测与响应）类驱动用前者，窗口管理用后者——混用两者会把“进程创建了”与“进程有了 Win32 对象”当成同一事件，时序上差着一个 GUI 转换。
- **KeUserModeCallback 是反向通道**：APC 是内核排队、用户态在返回时跑；`KeUserModeCallback`（`KE/*/CALLBACK.C`）是内核在系统调用上下文里直接切到用户态跑一段再回来。Mandt 全译讲的用户回调攻击（内核调用户态回调例程、用户态例程被劫持后提权）走的就是这条通道——回调例程的地址与栈都由用户态提供，内核切过去时若校验不足，即成提权点。本批只点名通道，不展开 Mandt 的利用链（全译已交付）。

## 第六部分：易混点

1. **进程创建 ≠ Win32 进程对象创建**：前者在 `PspCreateProcess`，后者在第一个线程 `PsConvertToGuiThread` 时（第 ④ 步第 7 点、第 ⑦ 步）。进程可以存在而没有 Win32 对象（如 SMSS 刚起、未转 GUI 的瞬间）。
2. **首线程的用户起点 ≠ 内核起点**：`StartAddress` 是 exe 入口点（用户态），`PspUserThreadStartup` 是内核起点（内核态），两者经初始用户 APC 衔接——线程先跑内核起点，内核起点排 APC，APC 跑加载器，加载器完事才跑用户起点。三段不要压成一段。
3. **LdrInitializeThunk ≠ exe 入口点**：前者是 ntdll 的加载器初始化，后者是 PE 头里的入口点地址；`NtContinue` 是两者之间的交接，不是加载器的返回。
4. **callout 表注册 ≠ win32k 装载**：装载是 `MmLoadSystemImage`（会话标志）的事，注册是 `DriverEntry` 里调 `PsEstablishWin32Callouts` 的事；装载成功但 `DriverEntry` 失败时，表是空的，`PsConvertToGuiThread` 照样拒绝。
5. **会话空间 ≠ 进程地址空间**：进程地址空间是每进程的用户部分（低 2GB/低 128TB），会话空间是系统地址空间里按会话切的区域（win32k 像、会话页池）；同一会话的多个进程共享会话空间、各自有进程地址空间。
6. **SMSS 的 5 秒等待 ≠ SMSS 就绪**：超时只证明“没死”，不证明“已建好 CSRSS”。把超时当成就绪，会误判启动时序。
7. **影子服务表切换 ≠ 服务表扩容**：切换是“本线程改查另一张表”，不是“往原表里加 Win32 服务”。原表（NT 服务）与影子表（Win32 服务）是两套编号，互不包含。

## 第七部分：自查题

1. SMSS 的父进程是谁、为什么？（答案见第 ③ 步第 3 点：`RtlCreateUserProcess` 传 NULL 时父进程默认当前进程，Phase 1 尾跑在 System 进程上下文里，故父进程是 System，令牌与配额从 System 继承。）
2. 首线程的三个起点各是什么、由谁填？（答案见第 ⑤ 步第 1 点：内核起点 `PspUserThreadStartup` 由 `PspCreateThread` 定，用户起点 `StartAddress` 取上下文程序计数器，Win32 起点取上下文寄存器。）
3. 为什么 `PspCreateProcess` 不调 Win32 进程 callout？（答案见第 ④ 步第 7 点与第 ⑦ 步：Win32 进程对象在第一个线程转 GUI 时建，进程创建时 win32k 的进程对象尚不存在、也无从建起。）
4. `PsConvertToGuiThread` 的两项门禁各防什么？（答案见第 ⑦ 步第 1 点：先前模式防内核线程转 GUI，callout 非空防 win32k 未起时切向不存在的服务实现。）
5. win32k 为什么必须是驱动、不能是普通 DLL？（答案见第 ⑧ 步：进内核地址空间、注册对象类型、填 callout 表、提供系统服务表，四件普通 DLL 一件都办不到。）

## 第八部分：对位小结（与前后批的关系）

- 对第 0 批：`PsLoadedModuleList` 的延续——win32k 以会话标志装载后，表项同样挂进模块表（会话表项另记，本批不展开会话表项结构）。
- 对第 2 批：`INITOS.C:1531` 的 `IoInitSystem` 与本批第 ① 步的 Phase 1 尾是同一条 Phase 1 链的两端；第 2 批地图里的“建 SMSS、等 5 秒”在本批展开为参数捏制、创建链与等待语义。
- 对第 4 批：EPROCESS/ETHREAD 是对象管理器对象，`ObCreateObject` / `ObInsertObject` 的复用是本批第 ④ 步的底座；WindowStation/Desktop 是对象类型机制在 Win32 侧的新实例，类型过程即 callout 表里的对象过程组。
- 对第 5 批：`PspSystemDll.LoaderInitRoutine` 是第 5 批 `PspInitializeSystemDll` 提取的 ntdll 入口点，本批第 ⑤ 步的初始 APC 直接用它；`KiDeliverApc` / `KiInitializeUserApc` 的用户 APC 机制是本批首线程诞生的投递底座；`KiReadyThread` 是首线程被放行后的调度底座。
- 对第 7 批：win32k 的会话空间装载复用第 7 批第 ⑧ 步的 `MmLoadSystemImage` 会话标志流程；`NtSetSystemInformation` 的会话映像入口（`SYSINFO.C:3456` 一带）是第 7 批“一函数三用”（system 波、动态、PnP）之外的第四个装载入口（会话映像专用）。
- 对 Mandt 全译：`KeUserModeCallback`（`KE/*/CALLBACK.C`）是 Mandt 讲的用户回调攻击面的内核侧通道，本批只点名；利用链细节见全译，不在此重复。
- 对 Design Notes《MM Design Note》：节对象 → 地址空间 → PEB 的镜像激活链（image activation），本批第 ③、④ 步是它在 WS03 代码线上的实现落点。

## 第九部分：未钉死清单

1. `NtInitialUserProcessBuffer` 默认值之外的覆盖来源（谁、在何时改写它）在树内未找到，只钉了默认值（`INITOS.C:167`）与校验（`:1715`）。
2. win32k 的 `DriverEntry` 本体（调 `PsEstablishWin32Callouts` 的调用点、建 WindowStation/Desktop 类型的代码）不在树内——callout 表的注册方只到“win32k 调它”这一层。
3. `NtSetSystemInformation` 会话映像装载入口的用户态调用方（SMSS 侧代码）在树内无对应物，只钉了内核入口（`SYSINFO.C:3456` 一带）。
4. 会话创建与删除的用户侧流程（SMSS 的会话管理）在树内无对应物，只讲了会话在内核侧的结构（`MmSessionSpace`、locale、归属）。
5. `LdrInitializeThunk` 的用户态实现（ntdll 代码）不在树内——本批只讲到内核侧的 APC 排布与入口点地址来源，加载器内部步骤（建堆、加载 DLL、TLS 回调顺序）为接口级陈述。
6. `PsConvertToGuiThread` 的大内核栈具体尺寸（`MmCreateKernelStack(TRUE, …)` 的 large 参数对应的字节数）在树内未逐行钉死。

## 第十部分：问答补记（本批交付后追问时追加）

- **排了用户 APC 之后，剩下的代码在做什么（2026-10-27 答疑补记）**：先纠一个前提——首线程的初始 APC **不是经 APC 队列排的**（不是 `KeInsertQueueApc`）。`PspUserThreadStartup` 里的注释写“Queue the initial APC”是借用“排队”这个词，机制上是直接调 `KiInitializeUserApc` 改陷阱帧。所以不存在“排完队、线程继续干别的、APC 不知何时跑”的问题；排完 APC 之后，线程唯一的前进方向就是出内核，而出内核恢复的正是被改写过的陷阱帧。完整链条如下：
  - **`KiInitializeUserApc` 到底改了什么**（`KE/I386/APCUSER.C:29`）：把陷阱帧与异常帧里的当前用户上下文整个拷到用户栈上（`CONTEXT` + `KAPC_RECORD`，`:124`），再改写陷阱帧：`HardwareEsp` 指向新用户栈、段选择子净化为 R3、**`Eip` 改成 `KeUserApcDispatcher`**（即 ntdll 里分发器的地址，`:154-155`），APC 记录（例程地址、上下文、系统参数）留在用户栈顶。原始入口点上下文没有丢，只是被压栈保存了——这就是 `NtContinue` 之后能“恢复”的原因。
  - **排完之后的三步收尾**（`PS/CREATE.C:2003` 的 `PspUserThreadStartup` 内）：`KeLowerIrql(PASSIVE_LEVEL)`；填全局 cookie（`while(1)` 循环，`SharedUserData->Cookie` 非 0 即返回，0 时现算并原子写回）；返回。没有等待、没有空转。
  - **回到汇编桩子**（`KE/I386/THREADBG.ASM:67` 的 `KiThreadStartup`）：它先降到 `APC_LEVEL`、调 `PspUserThreadStartup`，等它返回后检查 `UserContextFlag`——为 0 则蓝屏 `NO_USER_MODE_CONTEXT`（`:88` 的 `kits10` 分支），首线程一定非 0，于是 `ebp` 指向陷阱帧、`jmp KiServiceExit2`（`:84`）。
  - **`KiServiceExit2` 是通用出内核路径**：按陷阱帧恢复用户上下文。而陷阱帧的 `Eip` 已被改成 `KeUserApcDispatcher`、`Esp` 已是新用户栈，所以 CPU 一出内核就落在用户态的 ntdll 分发器里——**不需要再经过 `KiDeliverApc`**。`KiDeliverApc` 是给“已在跑的用户线程新排 APC”用的投递点，初始 APC 不走它。
  - **用户态分发器**：ntdll 的 `KiUserApcDispatcher` 按栈上的 `KAPC_RECORD` 调 `LoaderInitRoutine`（即 `LdrInitializeThunk`），跑完后调 `NtContinue`，参数就是当初拷到用户栈上的原始 `CONTEXT`——恢复成 exe 入口点的上下文，线程才真正跑 exe 代码。
  - **与“函数调用”类比的差别**：`KeInsertQueueApc` 那种排队（给已在跑的用户线程排 APC）确实是“排完就返回、线程继续干别的、APC 在下次回用户态或可警告等待时由 `KiDeliverApc` 投递”。但首线程没有“别的”可干——`PspUserThreadStartup` 就是它在内核的最后一站，排完 APC 之后只隔着一段固定的出内核代码（收尾 → 桩子 → 出内核恢复 → 分发器），不隔着不确定的等待。

## 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，与第十部分问答互补：那里讲初始 APC 排完之后的代码，这里补放行到启动例程之间的调度转移。

- **放行 → 调度 → KiThreadStartup：换栈 + ret，不是跳转**。`ZwResumeThread` 把首线程挂起计数减到 0 → `KiReadyThread` 把它挂上就绪队列 → 调度器选中它 → `SwapContext`（`KE/I386/CTXSWAP.ASM:374`）换栈 + ret——ret 从新线程的栈顶弹出返回地址，新线程第一次即 `KiThreadStartup`（`KE/I386/THREADBG.ASM:67`，建线程时栈就是按它搭的，参数都在栈上）。转移方式：就绪队列 + 换栈 + ret；`ZwResumeThread` 本身不“跑”新线程，它只是让新线程变得可调度，真正跑起来是调度器换栈的那一刻。
- **KiThreadStartup → PspUserThreadStartup → KiServiceExit2 → 分发器 → NtContinue：出内核流水线**。桩子降到 APC_LEVEL、调启动例程（第十部分已详述：降级、cookie、返回）→ 检查 `UserContextFlag`（为 0 则蓝屏 `NO_USER_MODE_CONTEXT`）→ `jmp KiServiceExit2` → 按被改写过的陷阱帧恢复用户上下文 → CPU 落进 ntdll 的 `KiUserApcDispatcher` → `LdrInitializeThunk` → `NtContinue` 恢复原始上下文 → exe 入口点。转移方式：函数调用（桩子调启动例程）+ 跳转（桩子跳出内核路径）+ 陷阱帧改写 + iret（出内核即分发器）+ 系统调用（`NtContinue` 恢复上下文）。没有一段是“不知何时跑起来”的等待。
- **GUI 转换的服务表切换：改指针，不是跳转**。`PsConvertToGuiThread` 切影子服务表，是改线程控制块里的服务表指针字段（AMD64 的 `Win32kTable` / x86 的 `ServiceTable`），此后该线程的系统调用按新表查号——切换本身不转移执行，转移发生在下一次系统调用查表时。特意写明：不要把“切表”读成“跳过去”。
- **win32k 注册：函数指针表填充**。`PsEstablishWin32Callouts` 逐项把 16 个指针存进内核全局变量，后续调用经全局指针间接调用——转移方式是间接函数调用；注册之前指针为空，`PsConvertToGuiThread` 查空即拒绝（`STATUS_ACCESS_DENIED`），这是“无表则无服务”的硬门，不是软检查。

### 追问：GUI 线程查影子表，难道不能调 NT 系统服务了吗（2026-10-27）

能调。影子表不是替换，是扩容：它有两项，[0] 是 NT 服务表的完整拷贝，[1] 才是 win32k 表。分发时服务号的高位选表、低位是表内索引，GUI 线程调 NT 服务走的仍是 [0]，和普通线程走原表 [0] 效果一样。

- **两张表，两份拷贝**。服务表一共两项（`NUMBER_SERVICE_TABLES` 为 2，`INC/KE.H:552`；win32k 表号为 1，`:4359`）。内核启动时 `KiInitSystem` 把 `KeServiceDescriptorTable` 整个拷贝给 `KeServiceDescriptorTableShadow`（`KE/KIINIT.C:408`，注释原话“which is used to record the Win32 system services”）；win32k 注册时调 `KeAddSystemServiceTable`（`KE/MISCC.C:729`），表号为 win32k 时**只填影子表**（`:791-794`，`if (Index != WIN32K_SERVICE_INDEX)` 才同步写原表）。所以原表只有 [0] 有效、[1] 为空；影子表 [0]=NT、[1]=win32k。
- **分发按服务号选表**。`KiSystemServiceRepeat`（`KE/I386/TRAP.ASM:1227`）：服务号右移取表号、与线程的 `ThServiceTable` 指针相加得描述符地址，低位取表内索引，对 `SdLimit` 越界检查。普通线程的 `ThServiceTable` 指向原表 [0]；GUI 线程在 `PsConvertToGuiThread` 里被改成指向影子表 [0]（x86：`Thread->Tcb.ServiceTable = &KeServiceDescriptorTableShadow[0]`，`PS/PSQUERY.C:4585`）。AMD64 没有统一指针，改的是 `Win32kTable` / `Win32kLimit` 两个字段（只存 win32k 表），NT 表仍走全局原表——效果相同。
- **非 GUI 线程调 win32k 服务：不是直接拒绝，是先自动转换**。表内索引越界 → `Kss_ErrorHandler` → 判断是 GUI 服务（表号 == `SERVICE_TABLE_TEST`）→ 调 `PsConvertToGuiThread` → 成功则重走分发（`_KiSystemServiceRepeat`）。所以“第一次调 NtUser* 就自动变 GUI 线程”不是用户代码显式申请的，是分发失败触发的；转换失败（如 win32k 未注册 callout → `STATUS_ACCESS_DENIED`）则走 `Kss_LimitError`，服务调不通。
- **附带细节：GUI 服务调用前先刷 GDI 批处理**。表号为 GUI 服务且 TEB 的 `GdiBatchCount` 非零时，先调 `KeGdiFlushUserBatch` 把用户态攒的 GDI 调用落盘，再执行本次服务，保证顺序。
- **这道门的安全意义**。win32k 服务只对 GUI 线程可达：非 GUI 线程调 NtUser* 会被强制转换，转换要过 win32k 注册、进程 callout、线程 callout 三道关，任何一道失败就拒绝——影子表切换本身就是准入门禁，不是单纯的查表优化。

### 追问：怎么进内核、怎么出内核，陷阱帧是什么、为什么叫陷阱、为什么要有（2026-10-28）

先认错：上一条只讲了“进内核后查哪张表”，没讲“怎么进、怎么出”，还把“陷阱帧”当已知词直接用了，违反术语前置与执行流转两条规范。这里补全。

- **陷阱帧是什么**。`KTRAP_FRAME`（`INC/I386.H:1636`）是建在线程内核栈上的一块定长结构，存的是“进内核那一刻的整套机器状态”：硬件压的部分（Eip、SegCs、EFlags、HardwareEsp、HardwareSegSs——只在特权级切换时才有，注释原话“only here for stacks that involve a ring transition”）；入口代码压的部分（非易失寄存器 Edi/Esi/Ebx/Ebp、段寄存器 SegGs/SegEs/SegDs、SegFs、调试寄存器 Dr0–Dr7、错误码 ErrCode）；内核管理用的部分（ExceptionList 旧异常链、PreviousPreviousMode、旧陷阱帧地址存在 TsEdx 里、TempSegCs/TempEsp 供改写帧时用）。一句话：陷阱帧 = 被中断线程的“快照 + 恢复地址”。
- **为什么叫“陷阱”帧**。Intel 术语里，trap 是“受控地陷入内核”的统称：系统调用、中断、异常进内核都走同一套“trap 入口”（`ENTER_TRAP` / `ENTER_SYSCALL`），入口代码建的帧就叫 trap frame。不要按 Intel 的窄义理解（异常分 fault/trap/abort 三类，trap 是其中“指令执行完后报告”的一类，如断点）——Windows 的 trap frame 是广义的：只要经陷阱入口进内核，建的都是同一结构，所以它不叫 syscall frame 也不叫 interrupt frame，叫 trap frame。
- **为什么要有陷阱帧，三条理由**。① 恢复：出内核时 iret 需要 Eip/SegCs/EFlags/HardwareEsp/HardwareSegSs，没有地方存就不知道回哪里。② 检查与修改：分发要读它（服务号越界检查、参数探查看先前模式）；`KiDeliverApc` 读它判断先前模式；`KiInitializeUserApc` 改它的 Eip/Esp 实现重定向（改完在 SegCs 上打 `FRAME_EDITED` = `0xfff8` 标记，告诉出内核路径 Esp 被改过、用 TempEsp）；`NtContinue` 用它恢复指定上下文。③ 嵌套：内核里再 trap（内核态缺页、内核态异常、中断嵌套）时，新帧压在旧帧上面，旧帧地址存在新帧 TsEdx 里成链；异常链也随帧保存/恢复，异常处理才不会串到被中断线程的链上。
- **进内核，两条路（x86，WS03）**。① int 2Eh：ntdll 桩（`mov eax, 服务号; mov edx, 参数指针; int 2Eh`）→ CPU 查 IDT 0x2E（中断门，`KE/I386/TRAP.ASM:419`）→ 特权级切换，按 TSS 的 Esp0 切到线程内核栈，硬件压 SS、ESP、EFLAGS、CS、EIP（用户态下一条指令地址），清 IF，进内核代码段 → `KiSystemService`（`:1052`）→ `ENTER_SYSCALL`（`KE/I386/KIMACRO.INC:316`）建帧：压 pad、非易失寄存器、FS 切到 PCR、存旧异常链并置空、存旧 PreviousMode 并按 SegCs 置新值、存旧陷阱帧地址、清 Dr7 → `KiSystemServiceRepeat` 分发。② SYSENTER：ntdll 桩（`KiFastSystemCall`，地址由内核启动时从 ntdll 取出填进共享数据区）→ CPU 按 SYSENTER MSR（内核预先编程：CS=内核代码段、EIP=`KiFastCallEntry`、ESP=本核 DPC 栈）直接跳进 `KiFastCallEntry`——**sysenter 不压任何返回帧，这是它与 int 的关键区别** → 入口代码切 ESP 到线程内核栈（TssEsp0），手工压出陷阱帧（SS、ESP、EFLAGS、CS、返回地址 = 共享数据区的 SystemCallReturn、pad、非易失寄存器、FS、旧异常链、新 PreviousMode=user）→ `KiSystemServiceRepeat`。SystemCallReturn 的值也是内核启动时从 ntdll 取的 `KiFastSystemCallRet` 桩地址（`PS/KULOOKUP.C:157`）。
- **出内核：不是函数返回，是状态恢复**。服务例程返回 → `KiServiceExit`：cli → `DISPATCH_USER_APC`（回用户态且有用户 APC 挂起 → `KiDeliverApc`，即第 5 批的投递点）→ `EXIT_ALL`：恢复非易失寄存器、段寄存器、FS、异常链、旧陷阱帧指针 → iretd：按陷阱帧的 Eip/SegCs/EFlags/HardwareEsp/HardwareSegSs 恢复用户上下文。int 2Eh 路径回到 int 2Eh 下一条指令；SYSENTER 路径回到 `KiFastSystemCallRet` 桩，再 ret 回 ntdll 调用者。关键：iret 不从栈上取返回地址“调用回去”，而是把整套上下文弹回 CPU；陷阱帧被改过（APC 重定向、NtContinue）时，恢复的就是改过的值——重定向能成立的全部原理就在这里。

---

## 追问补记（2026-10-30）：进出内核与陷阱帧全解（硬件原理层）

**问**：把进内核、出内核、构造陷阱帧的细节讲清楚，并讲清为什么，以及涉及的 CPU/硬件原理。

**核心约束（一切细节的来源）**：①用户态不可信，入口每一步都不能依赖用户配合；②中断可在任意指令间发生，入口从第 0 条指令起就得成立、不能靠已运行的软件铺垫；③进出必须可逆、且中途可被内核改道（APC/异常/调试全靠改帧）。行号均 2026-10-30 实查。

**硬件地基**：
- CPL（当前特权级）= CS 选择子低 2 位，硬件在每条指令的执行路径上查，软件绕不过去——这是"必须用硬件做裁判"的根源。
- 换栈必须硬件做：新栈指针的来源不能是用户给的（可任意伪造）、也不能靠软件 mov（换栈代码自己也要栈）。解法：CPU 在特权级提升时自动查 TSS 取 SS0:ESP0 换栈并压旧 SS/ESP。TSS 只有内核能改、CPU 按 TR 固定地址找得到、不需要任何指令先行执行。Esp0 必须时刻等于"当前线程的内核栈顶"：`SwapContext` 每次切换时改写它（`KE/I386/CTXSWAP.ASM:594-595`），另有 `Ki386AdjustEsp0`（:1579）处理特殊栈。
- CPU 跨环时自动压五件：SS、ESP、EFLAGS、CS、EIP（同环只压后三件——栈没换，SS/ESP 不必存）。这恰是 iret 弹回所需的全部信息，iret 是唯一能原子降特权级的指令（降级+换栈+恢复 EFLAGS 一步完成，分步做会出现"CPL0 却踩在用户栈上"的危险中间态）。部分异常另压 ErrCode；没有错误码的入口一律补压一个 0 垫片保帧定长。
- 门的类型：中断门进门清 IF；陷阱门不清。NT 静态 IDT 全用中断门——半成品的帧不能被中断踩，重新开中断的时机由内核代码在帧建完后自己 sti。

**陷阱帧三问（补全版）**：是什么 = `KTRAP_FRAME`（`INC/I386.H:1636`），线程内核栈顶固定位置（栈顶先留 FPU 保存区 NPX，帧紧贴其下，快调用入口以 `InitialStack − NPX长 − 帧长` 算出应然地址与实际 ESP 比对，不符即拒绝）的一整套机器状态快照，分三区：硬件区（SegSs/HardwareEsp/EFlags/SegCs/Eip/ErrCode，地址最高端）、入口代码区（Ebp/Ebx/Esi/Edi 非易失寄存器、SegFs、ExceptionList、PreviousPreviousMode、易失寄存器与段寄存器槽）、内核管理区（TempSegCs/TempEsp 编辑暂存、Dr0–Dr7、Dbg* 调试字段，含魔数 DbgArgMark=0xBADB0D00 供退出时校验"这真是陷阱帧"）。为什么叫陷阱：trap 是"受控陷入内核"的架构统称，系统调用/中断/异常三路共用同一套建帧逻辑，故帧统称陷阱帧。为什么要有：①恢复（iret 的弹药）；②检查与修改（帧字段由硬件与入口代码填写、在内核栈上，用户伪造不了——SegCs 判 PreviousMode 这条安全边界因此可信；KiInitializeUserApc 改 Eip/Esp 改道）；③嵌套成链（帧存旧 ExceptionList 与旧 PreviousMode，进出如括号配对）；④标准表示（异常分发把帧转 CONTEXT 给 SEH/调试器，全内核只认这一种"现场"）。

**三条入口的建帧分工**：
- int 2Eh：CPU 走中断门——查 DPL（0x2E 号门 DPL=3 用户才敲得动，硬件中断不查 DPL）、清 IF、按 TSS 换栈、压五件套；入口宏 `ENTER_SYSCALL`（`KE/I386/KIMACRO.INC:359`）补齐：压 0 垫片占 ErrCode、压非易失寄存器、存用户 FS 并把 FS 换成 PCR 选择子（用户 FS=TEB、内核 FS=PCR）、存旧 ExceptionList 并置空、存旧 PreviousMode。**不存**易失寄存器与段寄存器（宏注释明示）：系统调用是自愿调用，ABI 本就约定易失寄存器可毁，存了白存。
- SYSENTER（`KiFastCallEntry`，`KE/I386/TRAP.ASM:1136`）：CPU 按 MSR 跳转，只换 CS/EIP/SS、ESP 取自 SYSENTER_ESP MSR——MSR 是每 CPU 定值、不知道当前哪个线程在跑，NT 把它指向本核 DPC 栈当落脚垫（无主、恒有效）；桩子随即从 TSS 读 Esp0 二次跳板到当前线程内核栈。然后**手工逐 push 伪造硬件区**：用户 SS、用户 ESP（ntdll 桩子预先放进 EDX）、净化过的 EFLAGS（用户那份存帧，活动 EFLAGS 强制清 DF/NT/IF——DF 污染字符串指令方向、NT 位污染 iret 行为，进门必须净化；存帧那份单独把 IF 位置上留给 iret 恢复）、用户 CS、**假返回地址 SystemCallReturn**（共享用户数据区里的 ntdll `KiFastSystemCallRet` 桩地址，内核启动时从 ntdll 取，`PS/KULOOKUP.C:155-157`——因为 SYSENTER 硬件什么都没压，iret 需要的 Eip 只能伪造，伪造目标选一个"回来后立刻 ret 回调用点"的桩子，把硬件的缺失在用户态补齐）。再压 ErrCode 垫片、非易失寄存器、用户 FS（快速入口里用户 FS 是定值 TEB 选择子，直接压常数）、ExceptionList、PreviousMode（恒为用户，直接压 MODE_MASK 常数），sub 预留管理区。帧建完才 sti。
- 中断/异常：硬件压帧同 int 路（异常带真 ErrCode 的由 CPU 压、没有的由 KiTrap 桩补 0）；`ENTER_INTERRUPT`（KIMACRO.INC:477）**全存**：易失寄存器、全部段寄存器都进帧，PreviousPreviousMode 填 −1（中断没有"上一个模式"可存，语义不同）。为什么中断必须全存：被中断的代码是被偷袭的、没有任何调用约定保护它，少存一个寄存器就等于篡改它的现场。三路入口最后都产出同一种帧、汇入同一分发与退出代码——这是帧布局统一的回报。

**编辑协议（改帧为什么有规矩）**：要改 ESP 不能直改帧的 HardwareEsp（内核态帧根本没存 ESP，头文件注释明示"想要 ESP 必须自己物化"）；规矩是：新 ESP 写 TempEsp、真 SegCs 挪到 TempSegCs、SegCs 本位打 FRAME_EDITED 标记。`EXIT_ALL`（KIMACRO.INC:876）退出时专查此标记，命中走"编辑过的帧"弹出路径。KiInitializeUserApc 改 Eip/Esp 给新线程重定向走的正是这套协议（第 8 批正文）。

**出内核（状态恢复，不是函数返回）**：`KiServiceExit`（TRAP.ASM:1386）：cli（恢复是精密对表，半途被打断就帧/寄存器错位）→ DISPATCH_USER_APC（出门最后一米查挂起的用户 APC——APC 的投递点就设在这条必经之路上，要改帧也只剩这个窗口）→ `EXIT_ALL` 逆序恢复（还 PCR 的 ExceptionList、还线程 PreviousMode、按需恢复调试寄存器、FS 换回 TEB、弹非易失寄存器）。快出口 KiServiceExit 用 EXIT_ALL 的 NoRestoreSegs/NoRestoreVolatile 变体（段与易失寄存器按 ABI 本就可毁，不恢复省时间），KiServiceExit2 用全恢复（帧被编辑过、从 APC/异常回来的走这条）。最后 ESP 正指硬件区，iretd 一次弹完 EIP/CS/EFLAGS（+SS/ESP），特权级落回 3，IF 由帧里的 EFLAGS 恢复——IF 的恢复靠 iret 不靠 sti（第 5 批答疑）。

**AMD64 变体**：SYSCALL 指令连 SYSENTER 的落脚垫都没有——只按 LSTAR MSR 跳 `KiSystemCall64`（`KE/AMD64/TRAP.ASM:1682`），返回 RIP 存 RCX、RFLAGS 存 R11（正是 SYSRET 恢复所需的两样，硬件只存"回程票据"），栈不换、GS 还是用户的。第一件事 `swapgs`（:1684）：x64 段寄存器退休后只剩 FS/GS 当基址指针，用户 GS=TEB、内核 GS=PCR，必须有一条原子换 GS 的指令让入口第 0 步就摸到 PCR、进而取当前线程内核栈——swapgs 就是为这个瞬间存在的。AMD64 帧（`INC/AMD64.H:2241`）自带 ExceptionActive 字段自报家门（0 中断/1 异常/2 服务），且带 PreviousIrql（中断帧要记"进来前 IRQL"以便退出还原，i386 靠 PCR 软件字段记）。IST 专用栈见补遗《中断向量全表》。

## 追问补记（2026-10-30 续）：内核栈是每线程一本，不是全系统共用一本

**问**：维护义务说"TSS 的 Esp0 必须永远等于当前运行线程的内核栈顶、每次线程切换 SwapContext 都要改写"——难道不是所有线程共用同一个内核栈吗，还是每个线程一个内核栈？

**答**：每个线程一本自己的内核栈，全系统共用一本的理解是误会。理由两条：① NT 内核可阻塞——线程进内核办一件事可以中途睡过去（等 I/O、等事件），它那半截调用链（局部变量、返回地址链、栈上的陷阱帧）必须原样留着等它醒；若共用一本，A 睡下后 B 进内核就会踩掉 A 的现场。② 线程切换的物理动作就是换栈（SwapContext 换 ESP、ret 落进对方栈里冻着的返回地址，第 5 批）：一人一本，"冻结/解冻一个线程的内核世界"才等于"搁置/拿起一本栈"，零拷贝；只有一本栈，切换就得整本搬运，不可能。

生灭：线程创建时由 MM 分配（`MmCreateKernelStack`，普通/大型两档，第一个参数即 LargeStack 布尔），栈顶地址记在线程对象 InitialStack、当前位置记 KernelStack 字段，线程销毁回收。PsConvertToGuiThread"换大内核栈"（第 8 批正文）换的就是这个线程自己的那本——win32k 调用链深，普通本不够写。idle 线程、系统线程也都是真线程，各有各的栈（`KE/I386/ALLPROC.C:331` 给每核建 idle 线程时同时分栈）。

于是 Esp0 的维护义务顺理成章：Esp0 是 CPU 手里的登记值——"下一次特权级提升（系统调用/中断）落到哪本栈"。跑着的线程换了人，登记就必须跟着改，所以 SwapContext 换栈时顺手把新线程栈顶写进 TSS.Esp0（`KE/I386/CTXSWAP.ASM:594-595`）。推论：中断在用户态时到达，CPU 按 Esp0 落到的正是被中断线程自己的内核栈——"中断借用被中断线程的执行流"连栈都是借它的，这就是借用的物质基础。

不按线程分的栈只有例外三类：① 每核一张 DPC 栈（PRCB.DpcStack，`KE/I386/KERNLINI.C:918` 与 ALLPROC.C 分配）——DPC 不属于任何线程，KiDispatchInterrupt 排空 DPC 队列时临时换过去、办完换回（第 5 批）；SYSENTER 的落脚垫借的也是这张（见上一篇补记）。② AMD64 的 IST 专用栈（NMI/#DF/#MC 每核一张，见补遗《中断向量全表》）。③ 启动早期尚无线程时的启动栈，过渡性质。误解来源：把"内核栈"听成"内核这个整体的一本栈"；但 NT 里内核不是执行主体，只是代码与数据，执行永远属于某个线程（或借线程流的中断），栈按执行主体分，自然按线程分。本次未钉死：普通/大型两档栈的确切字节数（MM 分配参数，未在树内找到定义行，不编造）。
