---
layout: post
title: "WRK 第 7 批：IoInitSystem 与驱动加载潮"
categories: WRK
series_order: 8
---

取材说明：底本是 WRK 树（Windows Server 2003 SP1（WS03 SP1）代码线）`BASE/NTOS/` 下的 `IO/IOMGR/IOINIT.C`、`IO/IOMGR/INTERNAL.C`、`MM/SYSLOAD.C`、`CONFIG/CMSYSINI.C`、`CONFIG/CMBOOT.C`；ntldr、HAL 本体不在树内（见第 0 批）。**本批有一条树内边界，必须先说清**：`IO/PNPMGR/` 目录下只有头文件，没有 `.C` 文件，因此 `IopInitializeBootDrivers`、`IopInitializeSystemDrivers`、`IopInitializePlugPlayServices`、`PpInitSystem`、`PpLastGoodDoBootProcessing`、`IopDriverLoadingFailed`、`IopPnpDriverStarted` 这七个函数的本体都不在树内——本批只能钉死它们的调用点、入参与失败码，波内细节列入未钉死清单，不编造。另纠路线原话一处：路线写“`MmLoadSystemImage` 装映像 → `IopInitializeDriverModule` → `DriverEntry`”，但树内**没有名为 `IopInitializeDriverModule` 的函数**，对应环节的实名是 `IopLoadDriver`（`IO/IOMGR/INTERNAL.C:3689`），下文一律用实名。

主题一句话：Phase 1 的 `IoInitSystem` 先把 I/O 管理器自己的数据库、队列与 IRP 弹药库建好，然后按“boot 波 → system 波 → 再初始化 → 重指 SystemRoot”的顺序把驱动一个一个装起来、调起 `DriverEntry`，让内核从“只有执行体”变成“能碰硬件、能读磁盘、能挂卷”的完整系统。

## 第一部分：术语表（按引用级排序，基础在前、派生在后）

### 一、驱动与设备：先分清“谁是谁”

- **驱动（driver）**：一个内核态的 PE（可移植可执行格式，Portable Executable）映像（`.SYS` 文件），是“我会操作某类硬件/提供某类服务”的一段代码。它本身不直接被用户调用，用户与驱动之间隔着 I/O 管理器。
- **驱动对象（DRIVER_OBJECT）**：内核为每个已装载驱动建的对象（对象管理器第 4 批讲过的对象机制在此复用），代表“这个驱动”本身。它上面挂着两样关键东西：派遣表（见下）与它创建的设备对象链表。驱动对象由 `IopLoadDriver` 建（`INTERNAL.C:4080` 一带），不是驱动自己建的。
- **设备对象（DEVICE_OBJECT）**：代表“一个可被 I/O 的设备实例”（如一块盘、一个分区、一个文件系统卷）。由驱动在 `DriverEntry` 里调 `IoCreateDevice` 创建，挂在所属驱动对象的设备链表上。设备对象是 IRP 投递的目标。
- **派遣表（MajorFunction 表）**：驱动对象里一张按 IRP 主功能码索引的函数指针表，`IRP_MJ_READ`、`IRP_MJ_WRITE`、`IRP_MJ_DEVICE_CONTROL` 等各占一项。I/O 管理器收到请求后查这张表，把 IRP 交给对应的派遣例程。表项默认全填 `IopInvalidDeviceRequest`（`INTERNAL.C:4100` 一带），驱动在 `DriverEntry` 里把自己支持的项换成自己的例程——不换的项，请求来了就返回“非法请求”，而不是崩溃。
- **设备栈（device stack）**：同一设备上叠起来的多个设备对象：最底下是总线驱动建的物理设备对象（PDO），中间是功能驱动建的功能设备对象（FDO），上面还可叠过滤驱动建的过滤设备对象（FiDO）。IRP 从栈顶往下传，每层驱动处理完往下递。这就是 WDM 第 3 批讲的堆栈，本批只在内核侧看到它的落点：栈的每一层都是一个设备对象，IRP 的投递不关心驱动是谁，只关心栈顶是谁。
- **功能驱动 / 过滤驱动 / 总线驱动**：WDM 对驱动按角色的三分类。功能驱动是真正操作设备的；过滤驱动插在栈里改请求或加功能（如杀毒过滤）；总线驱动负责枚举子设备、建 PDO。分类只决定驱动在栈里的位置与 `AddDevice` 的行为，不改变装载流程。
- **即插即用（PnP）**：系统在运行时发现设备、找驱动、建设备栈、调 `AddDevice` 的整套机制。本批里 PnP 管理器只出现调用点（本体缺失），但要先有这个概念：boot 波与 system 波装的是“服务键里写明的”驱动，PnP 枚举波装的是“总线汇报上来的设备”对应的驱动，两者最后都汇到同一个 `IopLoadDriver`。

### 二、服务键与启动类型：驱动从哪来、按什么顺序装

- **服务键（service key）**：注册表 `HKLM\SYSTEM\CurrentControlSet\Services\<驱动名>` 下的一组值，是驱动的“档案”：`ImagePath`（映像路径，缺省则按默认路径找）、`Start`（启动类型）、`Group`（分组）、`Tag`（组内序号）、`DependOnGroup` / `DependOnService`（依赖）、`ErrorControl`（失败处置）。`IopLoadDriver` 的第一个参数就是这个服务键的句柄（`INTERNAL.C:3689`），它做的第一件事就是读键名拼出驱动文件名。
- **Start 值**：`0 = Boot`（启动驱动，加载器在内核起来前就读进内存）、`1 = System`（系统驱动，`IoInitSystem` 的 system 波装）、`2 = Auto`（自动驱动，用户态服务控制管理器 SCM 在系统起来后装）、`3 = Demand`（按需，`NtLoadDriver` 随时调）、`4 = Disabled`（禁用）。第 6 批的 `CmpFindDrivers` 就是按这个值筛的（`CMBOOT.C:715` 一带注释原话：`Start=BootLoad` 才是 boot 驱动）。
- **Group / Tag**：`Group` 是驱动的分组名（如 `SCSI miniport`），同组驱动按 `Tag` 数值从小到大装；组与组之间按 `ServiceGroupOrder` 表排（第 6 批第 ⑱ 步已讲排序算法，本批不重复）。为什么要有分组：磁盘驱动必须在文件系统驱动之前，文件系统驱动必须在卷驱动之前——顺序错了，文件系统驱动一启动就找不到盘。
- **ErrorControl**：驱动装载失败时怎么办的档案值。树内只看到 `IopLoadDriver` 失败出口调 `IopDriverLoadingFailed` 并随后读 `ErrorControl` 值（`INTERNAL.C:4357-4365` 一带），处置本体缺失；公开文档口径是 0=忽略、1=记日志继续、2=严重（非 LastKnownGood 时按失败启动处理）、3=关键（蓝屏），本批只把这张表当作“公开口径、未逐行钉死”引用，见未钉死清单第 2 条。

### 三、驱动入口与生命周期：DriverEntry 到底是什么

- **DriverEntry**：每个驱动导出的初始化例程，是驱动的“主函数”。内核侧不叫它 `DriverEntry` 这个名字——`IopLoadDriver` 从 PE 头里取入口点地址，存进驱动对象的 `DriverInit` 字段（`INTERNAL.C:4116`），再以 `driverObject->DriverInit(driverObject, registryPath)` 调起它（`:4225`）。所以 `DriverEntry` 是驱动侧的导出名，`DriverInit` 是内核侧的字段名，指同一个函数指针。为什么分两个名字：内核不依赖驱动的符号名，只依赖 PE 入口点——驱动改名、换导出表都不影响装载。
- **AddDevice**：PnP 驱动在 `DriverEntry` 里填进驱动对象 `DriverExtension->AddDevice` 的例程，PnP 管理器每枚举到一个该驱动负责的设备就调一次，驱动在里面建 FDO 并挂进设备栈。Legacy（非 PnP）驱动没有 `AddDevice`，直接在 `DriverEntry` 里建设备对象。区分标志是 PE 特征位：`DllCharacteristics` 缺 `IMAGE_DLLCHARACTERISTICS_WDM_DRIVER` 位时，`IopLoadDriver` 给驱动对象打上 `DRVO_LEGACY_DRIVER`（`INTERNAL.C:4113-4115` 一带）。
- **DriverUnload**：驱动的卸载例程（可选）。`DriverEntry` 失败后，内核把驱动对象标为临时并解引用，对象删光即卸载（见正文第 ⑨ 步）；成功后若 PnP 启动失败且驱动有 `DriverUnload`，也会调它再卸（`INTERNAL.C:4285` 一带）。
- **DriverReinitialize**：驱动向内核登记的“以后再调我一次”的例程。为什么要有：`DriverEntry` 运行时，它依赖的其他驱动可能还没装好；登记再初始化后，内核在 system 波结束后统一重跑一遍（`IopCallDriverReinitializationRoutines`，`INTERNAL.C:10165`），这时全波驱动都已就位。
- **DO_DEVICE_INITIALIZING**：设备对象的一个标志位，设备对象刚建好时立着，表示“还没初始化完，别人别用”。`IopReadyDeviceObjects`（`INTERNAL.C:6286`）在驱动成功后清掉它，并给驱动对象打 `DRVO_INITIALIZED`。为什么要有这道门闩：`DriverEntry` 还没跑完时，别的驱动或用户请求可能已经拿到设备对象指针；门闩立着时 I/O 管理器会挡下发向它的请求，避免用到半成品设备。
- **DRVO_INITIALIZED / DRVO_UNLOAD_INVOKED / DRVO_REINIT_REGISTERED**：驱动对象的状态标志，分别是“已初始化完成”、“已调过卸载”、“已登记再初始化”。`IopCallDriverReinitializationRoutines` 跑完一项就清 `DRVO_REINIT_REGISTERED`，防止重复跑。

### 四、IRP 与请求包弹药库

- **IRP（I/O 请求包）**：I/O 管理器内部描述一次 I/O 请求的结构：主功能码、设备栈位置数组（每层一个 `IO_STACK_LOCATION`）、缓冲区描述（MDL）、完成例程等。请求从栈顶往下传，每过一层用掉一个栈位置——所以 IRP 的大小取决于设备栈深度，这就是为什么要有“大 IRP”（栈位置多）与“小 IRP”（栈位置少）两种规格。
- **lookaside 表**：为高频小对象准备的预分配缓存池，分配时先从池里拿、不用时还回池里，避免每次都走通用池分配。`IoInitSystem` 按系统规模建了四套：完成包、大 IRP、小 IRP、MDL（见正文第 ③ 步）。为什么要分“系统级”与“每核级”：每核一份可避免多核抢同一把锁。
- **MDL（内存描述符表）**：描述一块用户缓冲区的物理页集合，让驱动能直接 DMA（直接内存访问，设备不经 CPU 直接读写内存） 而不用逐字节拷。IRP 里挂 MDL 是“直接 I/O”的载体。
- **IopInitFailCode**：`IoInitSystem` 内部的失败码（0、5、6、7、8、9、11、12），每一步失败前先写码再返回 FALSE，供调试器与无头日志定位“死在哪一步”。它不是返回给用户的错误码，是启动失败的验尸编号。

### 五、映像装载：.SYS 文件怎么变成内存里的代码

- **节对象（section）**：第 4 批讲过的对象类型之一，代表“一段可映射的虚拟内存的后备”。`MmLoadSystemImage` 给驱动文件建的是映像节（`SEC_IMAGE`），即“按 PE 格式映射”的节，映射后代码段可执行、数据段按 PE 节属性保护。
- **重定位（relocation）**：PE 文件的首选基址可能已被占用，装载器要按实际装载地址修正代码里所有绝对地址引用，这一步叫 `LdrRelocateImage`（`SYSLOAD.C:1409`）。为什么驱动也需要重定位：驱动不是固定地址的固件，它和内核、其他驱动共享系统地址空间，基址由装载时现定。
- **导入表（import table）**：PE 文件里声明的“我要调哪些别的模块的函数”（驱动调的几乎全是 `ntoskrnl.exe` 导出的内核例程，如 `IoCreateDevice`）。装载时要把导入表里的名字解析成实际地址、填进导入地址表（IAT）。加载器预装的 boot 驱动不走这一步现解析——它们的导入在加载器阶段已处理，`LoadedImports` 标为 `LOADED_AT_BOOT`（`SYSLOAD.C:4009` 一带）。
- **INIT 段（可丢弃段）**：PE 里标记为 `IMAGE_SCN_MEM_DISCARDABLE` 的段，驱动把只在初始化时用的代码（如 `DriverEntry` 本体）放这里。`MmFreeDriverInitialization`（`SYSLOAD.C:2575`）在驱动成功后把尾部的可丢弃段从地址空间里删掉——为什么：初始化代码跑完就再也不用了，留着白占系统页表与内存，删掉即回收。注意它只删“尾部连续的可丢弃段”，中间夹着不可丢弃段就停（`:2600` 一带的 break 逻辑）。
- **PsLoadedModuleList**：内核已装载模块的链表（`KLDR_DATA_TABLE_ENTRY` 串起来的），第 0 批讲过它是加载器链表的延续。`MmLoadSystemImage` 装完一个驱动就把它的表项挂进这条链；`IopLoadDriver` 查这条链判断“是不是已经装过了”。
- **会话空间驱动（session-space driver）**：用 `MM_LOAD_IMAGE_IN_SESSION` 标志装载的驱动（如 win32k 相关的显示驱动），每个会话一份副本、代码页共享、数据页写时复制。本批只点名这个概念，第 8 批讲 win32k 时再展开。
- **大页驱动（large-page driver）**：`MiUseLargeDriverPage` 把驱动映像挪到大页（2MB/4MB 页）上，减少 TLB（快表，虚拟地址到物理地址的翻译缓存）占用。触发条件是注册表值，树内未逐行查触发值，见未钉死清单第 5 条。

### 六、文件系统注册与挂载：驱动怎么变成“能读的盘”

- **文件系统队列**：`IoInitSystem` 建的四条链表：磁盘文件系统队、CD-ROM 队、磁带队、网络文件系统队（`IOINIT.C:232` 一带）。文件系统驱动在 `DriverEntry` 里调 `IoRegisterFileSystem` 把自己挂进对应的队。为什么要分队：卷挂载时，I/O 管理器要按介质类型找候选文件系统——磁盘卷只问磁盘队，不问网络队。
- **卷参数块（VPB）**：挂在设备对象上的“卷已挂载”标记与挂载后的文件系统设备对象指针。卷第一次被打开时，I/O 管理器走 VPB 挂载流程：通知各候选文件系统试认卷，认上的就把 VPB 指向该文件系统的卷设备对象。挂载是按需的、不是启动时全挂的（第 6 批问答已讲“卷第一次被打开时才挂载”，本批第 ⑮ 步纠正那里的一个表述）。
- **\SystemRoot**：指向系统盘 Windows 目录的符号链接。`IoInitSystem` 尾巴的 `IopReassignSystemRoot`（`IOINIT.C:776`）把它从加载器阶段的 ARC 路径形式重指为 NT 设备路径形式。为什么要重指：ARC 路径是固件命名空间的叫法，内核态打开文件要走 NT 命名空间的设备路径；重指后 `\SystemRoot\System32\Drivers\xxx.sys` 才能被 `ZwOpenFile` 解析。
- **引导日志（ntbtlog.txt）**：驱动装载记录文件。`IopBootLog`（`INTERNAL.C:9549`）的注释原话：`NtInitializeRegistry` 被调之前，日志条目先假脱机进注册表；SM（会话管理器，smss.exe）调 `NtInitializeRegistry` 后，日志文件才被创建/截断，注册表里的条目被拷进文件再删掉。为什么这样绕：日志文件本身在系统卷上，而系统卷的写盘能力要等 SM 阶段的 hive 绑定之后才完整——日志不能写进一个还没准备好的文件系统，只好先寄存在注册表里。

### 七、通用小词

- **ERESOURCE**：第 2 批讲过的读写锁，`IopDriverLoadResource`（驱动装载互斥）与 `IopDatabaseResource`（I/O 数据库锁）都是它。驱动装载期间全程拿 `IopDriverLoadResource` 的互斥写锁，任意核、任意线程调 `IopLoadDriver` 都要排队——一次只装一个驱动。
- **安全模式（SafeBoot）**：只装白名单驱动的启动方式。`IopLoadDriver` 里 `CheckForSafeBoot` 为真且 `InitSafeBootMode` 立着时，驱动的 Group 或文件名不在白名单里就跳过装载、返回成功（`INTERNAL.C:3810` 一带）——注意返回的是成功不是失败，跳过不是错误。
- **LastKnownGood（LKG）**：上一次成功启动的控制集。加载器按它选控制集（第 6 批），`PpLastGoodDoBootProcessing` 在 boot 波后做 LKG 的账本处理（本体缺失，只钉调用点）。

## 第二部分：正文

### ① 总闸：IoInitSystem 在 Phase 1 的什么位置、失败意味着什么

调用点是 `INITOS.C:1531`，在 `CmInitSystem1`（:1437）之后、`MmInitSystem(2)`（:1547）之前。进度条语义写在注释里：`IoInitSystem` 负责 25%→75% 这一段（`INITOS.C:1526`），`MmInitSystem(2)` 之后才到 80%。失败总闸只有一行（`:1531-1533`）：

```c
if (!IoInitSystem(LoaderBlock)) {
    KeBugCheck(IO1_INITIALIZATION_FAILED);
}
```

为什么是蓝屏而不是“跳过继续”：`IoInitSystem` 失败意味着 I/O 管理器本身没立起来——没有它就没有驱动、没有文件系统、没有卷，内核连系统盘都读不了，继续跑只会死得更难看。蓝屏码 `IO1_INITIALIZATION_FAILED` 是 Phase 1 的 I/O 初始化失败，与 Phase 0 的 PnP 失败码 `PP1_INITIALIZATION_FAILED`（`INITOS.C:1495` 一带）是两道不同的闸。`IoInitSystem` 内部每一步失败前写 `IopInitFailCode`（0、5、6、7、8、9、11、12），这是给调试用的验尸编号，不是给用户的错误信息。

### ② I/O 数据库：先建锁、队列头与函数指针表

`IoInitSystem`（`IOINIT.C:141`）开头先建四把 ERESOURCE 锁（`IopDriverLoadResource`、`IopDatabaseResource`、`IopSecurityResource`、`IopCrashDumpLock`，`:229` 一带）与一串链表头（`:232` 一带）：四条文件系统队列（磁盘、CD-ROM、磁带、网络）、两条再初始化队列（boot 波用一条、system 波用一条，分开记）、两条关机通知队列、一条文件系统变化通知队列。为什么锁与队列头要在任何驱动之前：驱动一启动就要往这些队里挂（`IoRegisterFileSystem`）、就要拿这些锁（`IopLoadDriver` 拿 `IopDriverLoadResource`），先建后用是启动顺序的铁律。

接着 `IopSetIoRoutines`（`:240`，本体 `:863`）把三张函数指针填上：`pIofCompleteRequest = IopfCompleteRequest`、`pIoAllocateIrp = IopAllocateIrpPrivate`、`pIoFreeIrp = IopFreeIrp`，且只在指针为空时填（`if (pIofCompleteRequest == NULL)`）。为什么用指针而不是直接调：这是给硬件抽象层（HAL）或调试器留的替换点——指针可被改写，调用方永远走指针，换实现不用改调用处。`IopUniqueDeviceObjectNumber` 清零（`:243` 一带），供 `IoCreateDevice` 自动编号设备对象用。

### ③ IRP 弹药库：按系统规模缩放的 lookaside

`IoInitSystem` 按 `MmQuerySystemSize()` 分四档建 lookaside（`:250` 一带起）：小系统完成包 6 / 小 IRP 6 / 大 IRP 8 / MDL 16；中系统 24 / 24 / 32 / 90；大系统分两种——AS（Advanced Server）系统 96 / 96 / 128 / 256，非 AS 的大系统 32 / 32 / 64 / 128。为什么按规模缩放：lookaside 是预分配的常驻内存，小机器上建多了是浪费，大机器上建少了是瓶颈；规模档位来自内存大小（第 3 批 `MmQuerySystemSize` 的档位），不是 CPU 档位。

每核再分一份：总信用额除以 `KeNumberProcessors` 记进各核处理器控制块（PRCB）的 `LookasideIrpFloat`，每核的 `PPLookasideList` 指向系统级表与本核表（`:340` 一带起）。为什么每核一份：IRP 分配是全系统最高频的操作之一，每核一份池子，分配时不抢全局锁；用超了（float 信用额花光）才回系统级表取。单核下这套机制退化为“一份池子”，逻辑不变。

大 IRP 的栈位置数默认 `DEFAULT_LARGE_IRP_LOCATIONS`，且可自动调大（`IOP_ENABLE_AUTO_SIZING` 标志，`:255` 一带）：设备栈叠得越深，IRP 需要的栈位置越多；栈位置不够时 I/O 管理器要现配更大的 IRP，自动调大就是把“上次不够”的教训记下来，下次直接按更大的规格建。

### ④ 波次总览：IoInitSystem 内部的顺序表

`IoInitSystem` 的主干按以下顺序走（行号均为 `IOINIT.C` 内调用点，失败码见括号）：

| 次序 | 步骤 | 失败码 |
|---|---|---|
| 1 | `WMIInitialize(0, LoaderBlock)`，WMI（Windows 管理规范）第一阶段 | — |
| 2 | 存 `IopLoaderBlock`（供 PnP 枚举用，boot 波后清掉） | — |
| 3 | `IopInitializePlugPlayServices(LoaderBlock, 1)`，PnP 服务第一阶段（跑固件映射器） | 5 |
| 4 | `IopInitializeBootDrivers(LoaderBlock, &driverObject)`，boot 波 | 6 |
| 5 | `PpLastGoodDoBootProcessing()`，LKG 账本处理 | — |
| 6 | `PsLocateSystemDll(FALSE)`，定位并映射 ntdll（第 5 批纠正过的调用点，`:743`） | 7 |
| 7 | `CcPfBeginBootPhase(PfSystemDriverInitPhase)`，通知启动预取器进入 system 驱动阶段 | — |
| 8 | `IopInitializeSystemDrivers()`，system 波 | 8 |
| 9 | `IopCallDriverReinitializationRoutines()`，再初始化队列 | — |
| 10 | `IopReassignSystemRoot`，重指 `\SystemRoot` | 9 |
| 11 | `IoAssignDriveLetters`，分配盘符 | 11、12（字符串转换失败） |
| 12 | `WMIInitialize(1, NULL)`，WMI 第二阶段；`PoInitDriverServices(1)`，电源管理第二阶段 | — |

为什么 WMI 与电源管理都要分两阶段：第一阶段在驱动起来前（建自己的驱动对象、准备接收），第二阶段在驱动全起来后（这时驱动才能发 WMI 请求、电源策略才能管到已装设备）。为什么 `PsLocateSystemDll` 卡在 boot 波与 system 波之间：system 波的驱动文件要靠它定位的 ntdll 里的用户态入口吗？不——`PsLocateSystemDll` 是给 Phase 1 尾建 SMSS 做准备（第 5 批已钉：`PsSystemDllDllBase` 的写入点），它放在这里是因为 boot 波已把卷驱动栈立起来、文件系统已可读，而 system 波还没开始抢 I/O 资源。

### ⑤ PnP 的两段：Phase 0 与 Phase 1 各调一次，但本体都不在树内

`PpInitSystem` 声明在 `INC/PNP.H:441`，签名是 `BOOLEAN PpInitSystem(VOID)`——**不带 Phase 参数**。它被调两次：Phase 0 一次（`INITOS.C:839`，失败 → `PP0_INITIALIZATION_FAILED`，第 2 批笔记里写成 `PpInitSystem(0)` 是简写，严格说函数不取参数，见本批末尾的纠正）、Phase 1 一次（`INITOS.C:1495`，失败 → `PP1_INITIALIZATION_FAILED`，注释原话“must be done before the I/O system initializes”）。为什么 PnP 要在 I/O 之前：设备对象是对象管理器的对象（故在 Ob 之后），而 I/O 的驱动波要靠 PnP 的设备树与服务——boot 波与 system 波装的驱动，其设备对象最终都要挂进 PnP 维护的设备树里。

Phase 1 的 `IopInitializePlugPlayServices(LoaderBlock, 1)`（`IOINIT.C:670` 一带，失败码 5）是 PnP 服务的第一阶段，注释原话是“execute firmware mapper”——把固件（BIOS：基本输入输出系统；ACPI：高级配置与电源接口）汇报的设备翻译成 PnP 设备节点，为后续枚举打底。固件映射器的本体同样不在树内，本批只钉到“它在 boot 波之前、失败码 5”这一层。

### ⑥ boot 波：加载器已装好的驱动，在这里被“点火”

boot 波的调用点是 `IOINIT.C:686`（失败码 6）。它的输入是加载器阶段的成果：加载器按 SYSTEM hive 的 boot 清单（第 6 批第 ⑱ 步的 `CmpFindDrivers(BootLoad)` 逻辑在加载器里的对应物）把 boot 驱动映像读进内存、挂进 `PsLoadedModuleList`。所以 boot 波**不需要** `MmLoadSystemImage` 现读文件——映像已经在内存里了；boot 波要做的是给每个已装映像建驱动对象、调 `DriverEntry`、让驱动建设备对象。

boot 波的本体 `IopInitializeBootDrivers` 不在树内，因此“它内部是按什么循环调 `DriverEntry`、失败一个 boot 驱动时是继续还是停”这两点**树内钉不死**，列入未钉死清单第 1 条。但有一点可以从树内另一处反推钉死：boot 波**不可能**走 `IopLoadDriver` 的“已装”分支——因为 `IopLoadDriver` 查 `PsLoadedModuleList` 发现已装时，返回 `STATUS_IMAGE_ALREADY_LOADED` 并直接退出，**不调 `DriverEntry`**（`INTERNAL.C:3890` 一带）。如果 boot 波走这条分支，boot 驱动的 `DriverEntry` 永远不会被调，系统不可能起来。所以 boot 波必有另一条“已装映像 → 建驱动对象 → 调 DriverEntry”的路径，只是本体缺失。

为什么 boot 驱动必须由加载器预装、不能等 system 波现读：鸡生蛋——system 波读驱动文件要走文件系统栈，而文件系统驱动（ntfs.sys、fastfat.sys）与磁盘驱动（disk.sys、atapi.sys）本身就是 boot 驱动；没有它们，内核连系统盘的一个扇区都读不到。加载器的迷你文件系统代码（第 2 批）就是为打破这个循环而存在的：它不依赖内核文件系统，只依赖固件扇区读写，把 boot 驱动读进来，boot 波点火后，内核才第一次拥有“能读盘”的能力。

### ⑦ system 波的清单来源：CmGetSystemDriverList

system 波的清单不是现算的，是 `CmGetSystemDriverList`（`CONFIG/CMSYSINI.C:1747`）算好的。它的步骤：

1. 打开 `\Registry\Machine\System`（`:1790` 一带），从键体（`KeyBody`）的键控制块（KCB）里**直接**取出 SYSTEM hive 指针与根 cell（`:1815` 一带）——不走 `NtOpenKey` 逐层解析，而是像加载器一样直接在 hive cell 上走（注释原话：“we can use all the same code that the OS Loader does”）。为什么：这时注册表锁（`CmpLockRegistryExclusive`）已拿着，直接走 cell 最快，也避免在启动关键路径上重走对象解析。
2. `CmpFindControlSet` 找 `Current` 控制集（第 6 批第 ⑨ 步的符号链接解析在此复用）。
3. `CmpFindDrivers(Hive, ControlCell, SystemLoad, NULL, &DriverList)`——注意第三个参数是 `SystemLoad`，即只筛 `Start=1` 的驱动（第 6 批讲的是同一函数筛 `BootLoad` 的加载器侧用法，函数本体同一套，参数不同）。
4. `CmpSortDriverList`（按 Group/Tag 排序）→ `CmpResolveDriverDependencies`（解 DependOnGroup/DependOnService）→ 返回句柄数组，末尾 NULL 封口。
5. 函数头注释原话：“The caller will traverse this array and load and initialize the drivers described by the keys.”（调用方遍历数组、装载并初始化这些键描述的驱动。）——这就是 system 波循环体的**间接证据**：`IopInitializeSystemDrivers` 本体虽缺失，但它必定是“取数组 → 逐个调 `IopLoadDriver`”的结构，否则 `CmGetSystemDriverList` 的这句契约无人履行。这一步标为“契约反推”，不是逐行实测，见未钉死清单第 1 条。

### ⑧ 映像装载：MmLoadSystemImage 全链

`MmLoadSystemImage`（`MM/SYSLOAD.C:761`）是“把 `.SYS` 文件变成内存里可执行映像”的唯一入口，`IopLoadDriver` 与动态装载（`NtLoadDriver`）都调它。全链如下：

1. **进锁与禁挂起 APC**：拿 `MmSystemLoadLock`、调 `KeEnterCriticalRegionThread`（`:1000` 一带）。为什么：函数文档原话是“APC_LEVEL or below, arbitrary process context”（APC_LEVEL 即 IRQL 1 级；任意进程上下文）——调用方可能是任意进程，挂起 APC 可能把当前线程挂在持有装载锁的状态下，造成死锁；进临界区禁掉挂起 APC（APC：异步过程调用，内核投递给线程的回调机制）是保护锁的手段，与核数无关。
2. **重查已装表**：扫 `PsLoadedModuleList`，同名已装则返回 `STATUS_IMAGE_ALREADY_LOADED`（`:1040` 一带）。为什么先查表再开文件：同名驱动可能正被另一个线程装着（`ReCheckLoaderList` 标签即为此——开节之后还要再查一次，防“两线程同时装同一个”的竞态）。
3. **开文件**：`ZwOpenFile`，权限 `FILE_EXECUTE`，共享 `FILE_SHARE_READ | FILE_SHARE_DELETE`（`:1157` 一带）。为什么共享模式留 DELETE：驱动文件在装载期间可被替换/删除（升级场景），读完映像后文件句柄即关，映像页由页文件后备，不再依赖原文件。
4. **验像**：`MmCheckSystemImage`，三项硬错直接失败：校验和不符（`STATUS_IMAGE_CHECKSUM_MISMATCH`）、UP/MP 不匹配（`STATUS_IMAGE_MP_UP_MISMATCH`——单核内核上装多核驱动或反之）、映像保护非法（`STATUS_INVALID_IMAGE_PROTECT`）。为什么验 UP/MP：单核与多核内核的同步原语实现不同（第 1 批：单核下自旋锁可退化），驱动若按错的模型编译，锁语义全错。
5. **建映像节**：`ZwCreateSection`，`SEC_IMAGE`，`PAGE_EXECUTE`（`:1215` 一带）。节对象建好后引用之（`ObReferenceObjectByHandle`），句柄即关——节对象靠引用计数活着，不靠句柄。
6. **映射与读入**：`MiLoadImageSection` 把节映射进系统空间、把映像页读进内存（`:1320` 一带）。注释原话：普通驱动的映像“已经整个读进内存，从此到重启（或卸载）之前由页文件后备”，所以随后就把节引用放掉（`ObDereferenceObject`）；会话空间驱动例外——仍由文件后备，引用留着（文件被删会出问题，故不放）。
7. **重定位**：`LdrRelocateImage`（`:1409`），按实际基址修正绝对引用，失败则整个装载失败。
8. **导入解析与登记**：导入表解析（boot 驱动标 `LOADED_AT_BOOT`，现装驱动现解析，`:1500`、`:1658`、`:1715`、`:1868` 一带）；建 `KLDR_DATA_TABLE_ENTRY`，`LoadCount = 1`，PE 主版本与映像版本均 ≥5 时打 `LDRP_ENTRY_NATIVE`（`:1501` 一带起），表项挂进 `PsLoadedModuleList`。返回节指针（`ImageHandle`）与基址（`ImageBaseAddress`）——调用方 `IopLoadDriver` 拿节指针当驱动对象的 `DriverSection`，拿基址找 PE 头、取入口点。

### ⑨ 驱动装载：IopLoadDriver 全链（本批最核心的一条链）

`IopLoadDriver`（`INTERNAL.C:3689`）的入参是服务键句柄、安全模式开关、是否过滤驱动、出参 `DriverEntryStatus`。全链按执行顺序：

1. **拼文件名**：`NtQueryKey` 读服务键名，拼出 `<服务名>.SYS`（`:3790` 一带）。为什么用键名而不是 `ImagePath`：键名是驱动的身份，`ImagePath` 只是“去哪找它”的提示；先拼默认名，再由 `IopBuildFullDriverPath` 决定全路径。
2. **安全模式过滤**：`CheckForSafeBoot` 且 `InitSafeBootMode` 时，查 `Group` 值与文件名是否在白名单（`IopSafebootDriverLoad`），不在则记“Not Loaded”日志、**返回成功**（`:3810` 一带）。为什么返回成功：跳过是策略决定，不是错误；调用方（system 波循环）不应因一个被策略跳过的驱动而停下整波。
3. **已装检查**：共享锁下扫 `PsLoadedModuleList`，同名已装（加载器预装的 boot 驱动走这里）→ 记“Loaded”日志、返回 `STATUS_IMAGE_ALREADY_LOADED`（`:3880` 一带）。注意：这条分支**不建驱动对象、不调 `DriverEntry`**——boot 驱动的点火不在此函数内（见第 ⑥ 步）。
4. **定全路径**：`IopBuildFullDriverPath`（`:3908`）。有 `ImagePath` 值就用它，没有则拼默认路径 `\SystemRoot\System32\Drivers\<服务名>.sys`（`INTERNAL.C:3397-3398`）。为什么默认路径是这个：Windows 的驱动约定目录，`ImagePath` 是例外机制。
5. **取驱动对象名**：`IopGetDriverNameFromKeyNode`（`:3915` 一带），对象名形如 `\Driver\<服务名>`，`OBJ_PERMANENT` 属性（驱动对象常驻，不随句柄关闭消失）。
6. **装映像**：互斥锁 `IopDriverLoadResource` 下调 `MmLoadSystemImage`（`:3935` 一带）。为什么要拿互斥锁：一次只装一个驱动，避免两个线程同时装同一驱动、避免驱动对象名冲突。`MmLoadSystemImage` 返回 `STATUS_IMAGE_ALREADY_LOADED` 时走第二分支（见第 7 步）；其他失败时记“Not Loaded”、按 64 位非本机映像规则处理（见 32/64 对照）后退出。
7. **已装第二分支（复活路径）**：`MmLoadSystemImage` 说已装，说明驱动对象可能已存在但曾被卸载——`ObOpenObjectByName` 开驱动对象、`ObReferenceObjectByHandle` 取指针、`IopResurrectDriver` 复活它（`:4000` 一带）。为什么有复活：驱动卸了但映像还在（`DriverSection` 未放），重装时不用重读文件，只要把驱动对象从“待删”状态拉回来。复活细节未逐行读，见未钉死清单第 4 条。
8. **准备 Enum 子键**：`IopPrepareDriverLoading`（`:4055` 一带），按服务名与映像基址准备服务键下的 `Enum` 子键（PnP 枚举用的账本）。失败则 `MmUnloadSystemImage` 卸映像后退出。
9. **建驱动对象**：`ObCreateObject`，类型 `IoDriverObjectType`，大小 `sizeof(DRIVER_OBJECT) + sizeof(DRIVER_EXTENSION)`（`:4080` 一带）。清零后：`DriverExtension` 指到对象尾部的扩展区；派遣表全填 `IopInvalidDeviceRequest`；`Type = IO_TYPE_DRIVER`；`Size = sizeof(DRIVER_OBJECT)`；从 PE 头取入口点存 `DriverInit`；`DriverSection`、`DriverStart`、`DriverSize` 照映像填；`DllCharacteristics` 缺 WDM 位则打 `DRVO_LEGACY_DRIVER`（`:4113` 一带）。`ObInsertObject` 插进对象表（`:4125` 一带）。
10. **填档案字段**：`HardwareDatabase` 指到 `\Registry\Machine\Hardware\Description\System`（第 6 批建的硬件描述树）；`DriverName` 拷服务名；`NtQueryObject` 取服务键的注册表路径（`:4180` 一带）——这条路径就是 `DriverEntry` 的第二个参数 `RegistryPath`，驱动靠它读自己的服务键值（如 `Parameters` 子键）。
11. **调 DriverEntry**：`status = driverObject->DriverInit(driverObject, &registryPath->Name)`（`:4225`），出参 `*DriverEntryStatus = status`；失败则把返回码换成 `STATUS_FAILED_DRIVER_ENTRY`（`:4230` 一带）。调试版里 DriverEntry 超过 5 秒打印一条超时日志（`:4236` 一带）——启动慢的驱动在这里现形。
12. **零表项回填**：DriverEntry 返回后，把仍为 NULL 的派遣表项填回 `IopInvalidDeviceRequest`（`:4255` 一带），注释原话是 workaround for broken NT 4.0 3D labs driver（某驱动误把表项清零）。为什么内核要兜这个底：派遣表是 I/O 管理器查的，NULL 项一查就空指针蓝屏；填回非法请求至少让请求体面失败。
13. **成功后收尾**：非 legacy 驱动且成功 → `IopPnpDriverStarted`（`:4279`，本体缺失，接口语义是“通知 PnP 该驱动已起，触发 AddDevice”）；失败 → 驱动对象标临时并解引用（对象删光即卸，`:4295` 一带）；成功 → 记“Loaded”日志、`MmFreeDriverInitialization` 释放尾部可丢弃段（`:4304`）、`IopReadyDeviceObjects` 清 `DO_DEVICE_INITIALIZING`、打 `DRVO_INITIALIZED`（`:4305`）。
14. **失败出口**：除 `STATUS_PLUGPLAY_NO_DEVICE`（被硬件配置文件禁用的设备，不是错误）外，一律调 `IopDriverLoadingFailed` 向配置管理器（CM）报告失败，并读 `ErrorControl` 值（`:4357` 一带）。单驱动失败不中断整波——循环继续装下一个，处置（忽略/记日志/按 LKG 失败/蓝屏）由 `ErrorControl` 与 LKG 逻辑在波外决定（本体缺失，见未钉死清单第 2 条）。

### ⑩ 再初始化：为什么 DriverEntry 之后还要跑第二遍

`IopCallDriverReinitializationRoutines`（`INTERNAL.C:10165`）在 system 波之后、重指 SystemRoot 之前跑（`IOINIT.C:765` 一带）。它从 `IopDriverReinitializeQueueHead` 上逐个摘包（另有一条 boot 驱动专用的队列与登记函数，见第十部分问答补记）：给驱动对象的 `DriverExtension->Count` 加一、清 `DRVO_REINIT_REGISTERED`、调包里登记的 `DriverReinitializationRoutine(DriverObject, Context, Count)`、释放包。为什么要有这第二遍：`DriverEntry` 运行时，它依赖的驱动可能还没装好（组顺序只能保证粗粒度先后，不能保证“某个具体驱动已起”）；登记再初始化就是把“依赖别人”的工作推迟到全波结束——这时所有 system 驱动都已就位，再初始化例程可以安全地打开别人的设备、下 IRP、建连接。`Count` 参数告诉例程“这是第几次被调”（同一驱动可登记多次）。

### ⑪ 尾巴：重指 SystemRoot、分配盘符、第二阶段收尾

`IopReassignSystemRoot`（`IOINIT.C:776`，失败码 9）把 `\SystemRoot` 从加载器阶段的 ARC 路径形式重指为 NT 设备路径形式，随后 `IoAssignDriveLetters`（`:812`）按加载器的 ARC 磁盘信息与已挂载卷分配盘符（C: 等），字符串转换失败时失败码 11、12。为什么盘符分配放在 IoInitSystem 尾巴：盘符是“卷 → 字母”的映射，卷要先挂载（system 波里驱动文件一打开，系统卷即挂载，见第 ⑮ 步），映射才有东西可指。最后 `WMIInitialize(1, NULL)`（`:847`）、`PoInitDriverServices(1)`（`:852`），返回 TRUE。`INITOS.C` 随后清进度条子集模式、记版本 6、进 `MmInitSystem(2)`（`:1547`）、进度到 80%。

### ⑫ 失败与回滚总表

- **IoInitSystem 级**：任一波失败（失败码 5/6/7/8/9/11/12）→ 返回 FALSE → `INITOS.C:1531` 蓝屏 `IO1_INITIALIZATION_FAILED`。没有“降级启动”——I/O 管理器没立起来，系统无从继续。
- **单驱动级**：`IopLoadDriver` 失败 → 映像已卸（`MmUnloadSystemImage` 在准备失败分支）或驱动对象标临时删光（DriverEntry 失败分支），波继续；`IopDriverLoadingFailed` 报告 + `ErrorControl` 读值（处置本体缺失）。
- **引导日志**：`IopBootLog` 的假脱机机制（术语表第六节）保证“启动日志”本身不依赖文件系统先就绪——日志条目先寄存在注册表里，SM 阶段 `NtInitializeRegistry` 之后才落盘成 `ntbtlog.txt`。为什么这值得单列：它复用了第 6 批的两阶段结构（先内存态、后文件态），是同一解法在 I/O 侧的再现。
- **LastKnownGood**：加载器按 LKG 选控制集（第 6 批），`PpLastGoodDoBootProcessing` 在 boot 波后做账本处理（本体缺失）；“启动被接受”（`CmpHoldLazyFlush` 清零的那个点，第 6 批第 ⑭ 步）之后，当前控制集才会被记为新的 LKG——这就是为什么中途崩溃的启动不会污染 LKG。

### ⑬ system 波之外：Auto 波与 PnP 枚举波（不在 IoInitSystem 内，但走同一函数）

`Start=2` 的自动驱动不在 `IoInitSystem` 里装——它们由用户态的服务控制管理器（SCM，`services.exe`，会话管理器子系统 SMSS 与 Winlogon 起来后才运行）通过 `NtLoadDriver` 调起，最终汇到同一个 `IopLoadDriver`。`Start=3` 的按需驱动同理，任何时候调 `NtLoadDriver` 都是这条链。PnP 枚举波（新设备插上、总线汇报设备实例）也是 `IopLoadDriver`，只是调用方换成 PnP 管理器、服务键来自设备实例的驱动匹配结果（本体缺失，架构级陈述）。所以 `IopLoadDriver` 是“一函数三用”：system 波、动态装载、PnP 枚举——这也是为什么它的入参里有 `IsFilter`（PnP 过滤驱动用）与 `CheckForSafeBoot`（安全模式用）这两个“波外”开关。

### ⑭ 与 WDM 对位：内核侧提供什么、驱动侧写什么

WDM 第 2 批的“驱动骨架”落到本批是：`DriverEntry` 的两个参数（驱动对象、注册表路径）就是 `IopLoadDriver` 第 ⑨ 步第 10、11 点给的；派遣表的默认填充与回填是内核侧的兜底，驱动侧只填自己支持的项。WDM 第 3 批的“设备对象与堆栈”落到本批是：`IoCreateDevice` 的实现与 `DO_DEVICE_INITIALIZING` 门闩在内核侧，`DriverEntry` 里建设备、挂栈、建符号链接是驱动侧——驱动侧代码（如 ntfs.sys、disk.sys）不在 WRK 树内，树内只有内核侧的接待代码。Design Notes 的《I/O System》与《Driver Model》两份规范讲的是同一套契约的设计意图，本批是契约在 WS03 代码线上的实现。

### ⑮ 鸡生蛋再解一次：卷挂载到底发生在何时（并纠正第 6 批问答一处表述）

第 6 批问答里有一句需要纠正。原话是：“系统卷第一次被打开就是 SM 阶段 `NtInitializeRegistry` 打开 hive 文件时，所以‘文件系统真正能读系统盘’落在 SM 阶段。”**这句把两件事混在一起了，现纠正如下**：

- **卷挂载的触发点**是对的：卷第一次被打开时才挂载（建 VPB、文件系统驱动认卷），不是启动时全挂。
- **“系统卷第一次被打开在 SM 阶段”是错的**：`IoInitSystem` 的 system 波里，`MmLoadSystemImage` 用 `ZwOpenFile` 打开 `\SystemRoot\System32\Drivers\xxx.sys`（第 ⑧ 步第 3 点），`PsLocateSystemDll` 在 boot 波与 system 波之间打开 ntdll（第 ④ 步第 6 点）——这些打开都发生在 `IoInitSystem` 期间、远早于 SM 阶段。系统卷的挂载就发生在这些打开中的第一次，至迟在 system 波读第一个驱动文件时。
- **SM 阶段打开 hive 文件时，卷早已挂载**：那次打开只是“hive 文件第一次被打开”，不是“卷第一次被挂载”。第 6 批问答的后半句应改为：“hive 文件的打开落在 SM 阶段，但此时卷早已在 `IoInitSystem` 期间挂载。”

为什么这个纠正重要：它决定了第 6 批两阶段分界的准确含义——分界不是“文件系统何时可用”（文件系统在 `IoInitSystem` 期间就可用了），而是“hive 何时绑文件”（要等 autocheck 与页面文件，见第 6 批第 ⑫ 步）。文件系统可用 ≠ hive 可绑，前者早、后者晚，混淆两者会把第 6 批的整个分界逻辑读反。第 6 批笔记的问答补记已在后台按此修正。

## 第三部分：32 / 64 位对照

- **驱动字长硬门槛（64 位独有）**：`IopLoadDriver` 里 `_WIN64` 分支（`INTERNAL.C:3960` 一带）：`MmLoadSystemImage` 返回非本机映像错误时，`IopCheckIfNotNativeDriver` 判定为 32 位驱动则拦截——过滤驱动返回 `STATUS_DRIVER_BLOCKED`，非过滤驱动返回 `STATUS_DRIVER_BLOCKED_CRITICAL`，并记事件日志。为什么 64 位内核不能跑 32 位驱动（第 1 批已讲）：用户态有 WOW64（32 位程序在 64 位系统上的兼容层）翻译层，内核态没有；驱动与内核共享地址空间、直接调内核例程，字长一错，指针与结构布局全错。32 位内核无此分支，32 位驱动即本机。
- **映像节与重定位**：`SEC_IMAGE`、`LdrRelocateImage`、导入解析的流程两边相同；差异只在 PE 格式本身（PE32+ 与 PE32 的可选头字段宽度）与指针宽度，`MmLoadSystemImage` 的代码是同一套、按编译目标分化。
- **lookaside 规模档**：四档数值与系统规模（内存档位）挂钩，与字长无关；64 位大系统上同样的档位表适用。
- **INIT 段释放**：`MmFreeDriverInitialization` 按 PE 节头从尾部删可丢弃段，逻辑与字长无关；64 位下节对齐粒度按页（4KB）不变。

## 第四部分：单核 / 多核对照

- **驱动装载串行化（两边相同，原因不同）**：`IopDriverLoadResource` 是 ERESOURCE 互斥锁，`IopLoadDriver` 全程持有——任意核、任意线程调装载都要排队。单核下排队只是“同一核上的线程互斥”，多核下还是“跨核互斥”；锁的存在与核数无关，因为装载要改全局表（`PsLoadedModuleList`、驱动对象名空间、文件系统队列），并发装载会互相踩。
- **每核 lookaside 信用额**：总信用额除以 `KeNumberProcessors`（第 ③ 步）。单核下除数为 1，全额归本核；多核下每核一份，分配不抢锁。为什么除法而不是每核全额：全额乘核数会让预分配内存随核数线性膨胀。
- **装载锁与挂起 APC**：`MmLoadSystemImage` 的临界区（禁挂起 APC）与核数无关——它防的是“当前线程被挂起时持有装载锁”，单核下挂起同样死锁（锁的持有者不跑，等待者永等）。
- **再初始化队列**：`IopDriverReinitializeQueueHead` 是全局链表，摘包用 `IopInterlockedRemoveHeadList`（原子操作，第 ⑩ 步）。单核下原子操作退化为普通操作加中断屏蔽，多核下靠硬件原子指令——队列本身一份，不按核分。

## 第五部分：逆向 / 安全解读

- **驱动装载是内核攻防的主战场**：`IopLoadDriver` 的每一步都是检查点——服务键可写即等于“可指定任意 `.SYS` 路径”（`ImagePath` 可指向任意位置），`ErrorControl` 与 `Start` 可改即等于“可改启动顺序”。BYOVD（自带 vulnerable 驱动）攻击的落点正是这里：一个有签名的脆弱驱动被 `IopLoadDriver` 正常装载后，`DriverEntry` 里的漏洞即成为内核代码执行。防御侧看，64 位的非本机拦截、映像校验和与 UP/MP 检查、WDM 标志判定，都是装载时的静态门禁——但它们只验“格式与字长”，不验“驱动代码有没有洞”。
- **`DriverEntry` 的 5 秒超时日志是取证线索**：调试版里超 5 秒的驱动会打印 `IOLOAD: Driver %wZ took %d.%ds`（第 ⑨ 步第 11 点），启动缓慢的根因定位可从这里起；正式版无此日志，排障时需借助引导日志与事件日志。
- **引导日志是启动期的黑匣子**：`ntbtlog.txt` 的“Loaded / Not Loaded”逐行记录（第 ⑫ 步）是判断“哪个驱动没起来”的第一手证据；假脱机机制意味着：即使系统在 SM 阶段前崩溃，日志条目仍在注册表里（只要 hive 内存像还在），不依赖文件系统落盘。
- **INIT 段释放是“用后即焚”的正面例子**：`DriverEntry` 代码跑完即从地址空间删掉，攻击者事后无法再调已释放的初始化代码；反向看，逆向分析时 `DriverEntry` 的代码只在启动瞬间存在，事后内存转储里找不到它——要分析初始化逻辑，需在装载时下断点或静态分析映像文件。

## 第六部分：易混点

1. **boot 波 ≠ IopLoadDriver 的已装分支**：已装分支返回 `STATUS_IMAGE_ALREADY_LOADED` 且不调 `DriverEntry`（第 ⑥ 步），boot 驱动的点火必走另一条本体缺失的路径。把两者等同，会得出“boot 驱动从不跑 DriverEntry”的荒谬结论。
2. **system 波 ≠ 全波**：`IopInitializeSystemDrivers` 只装 `Start=1`；`Start=2` 的自动驱动由 SCM 在用户态后装（第 ⑬ 步）。“系统启动时所有驱动都在 IoInitSystem 里装完”是错的。
3. **DriverEntry ≠ 驱动的全部初始化**：DriverEntry 只建设备、填派遣表；PnP 驱动的设备真正“起动”在 `AddDevice` 与 `IopPnpDriverStarted` 之后；依赖别人的工作在再初始化队列里。把 DriverEntry 当成“驱动已完全就绪”是错的。
4. **映像装载 ≠ 驱动装载**：`MmLoadSystemImage` 只把文件变成内存映像、挂进模块表；驱动对象、派遣表、DriverEntry 调用全是 `IopLoadDriver` 的活。只调前者，驱动永远不会跑。
5. **挂载 ≠ 文件系统驱动装载**：文件系统驱动装好（挂进队列）只是“候选就绪”；卷挂载发生在第一次打开卷时（第 ⑮ 步）。驱动装了、卷没打开，VPB 还是空的。
6. **\SystemRoot 重指 ≠ 盘符分配**：前者是命名空间路径形式的切换（ARC → NT 设备路径），后者是卷 → 字母的映射；两者都在 IoInitSystem 尾巴，但做的是不同的事（第 ⑪ 步）。
7. **失败码 ≠ 错误码**：`IopInitFailCode` 是启动失败的验尸编号（5/6/7/8/9/11/12），`STATUS_FAILED_DRIVER_ENTRY` 是单驱动失败的返回码，`ErrorControl` 是失败处置策略——三者各司其职，不可互换。

## 第七部分：自查题

1. boot 驱动的映像是谁读进内存的、挂在哪条链表上？boot 波点火时为什么不需要 `MmLoadSystemImage` 现读文件？（答案见第 ⑥ 步：加载器读的，`PsLoadedModuleList`；现读要走文件系统栈，而文件系统驱动本身就是 boot 驱动。）
2. `IopLoadDriver` 查 `PsLoadedModuleList` 发现已装时，为什么不直接调 `DriverEntry`、而是返回 `STATUS_IMAGE_ALREADY_LOADED`？（答案见第 ⑨ 步第 3 点与第 ⑥ 步：已装分支是给“重复装载请求”准备的，调 DriverEntry 是 boot 波另一条路径的职责；重复调 DriverEntry 会建重复设备对象。）
3. 驱动成功后，为什么 `MmFreeDriverInitialization` 只删“尾部连续的可丢弃段”、中间夹着不可丢弃段就停？（答案见术语表第五节：删段是按页删地址空间，中间断开会留下空洞与对齐问题；实现上从尾部往前扫、遇不可丢弃即停，保证删掉的是一段连续尾巴。）
4. `DO_DEVICE_INITIALIZING` 门闩清掉的时机为什么是 `IopReadyDeviceObjects`（DriverEntry 成功后）、而不是 `IoCreateDevice` 当时？（答案见术语表第三节：设备对象建好时驱动还没填完派遣表、还没准备好，门闩立着期间发向它的请求会被挡；DriverEntry 成功才算准备好。）
5. 系统卷的挂载最早可能发生在 `IoInitSystem` 的哪一步、为什么不是 SM 阶段？（答案见第 ⑮ 步：`PsLocateSystemDll` 读 ntdll 或 system 波读第一个驱动文件时；SM 阶段只是 hive 文件第一次被打开。）

## 第八部分：对位小结（与前后批的关系）

- 对第 0 批：`PsLoadedModuleList` 从加载器链表延续到内核装载表，本批的 `MmLoadSystemImage` 与 `IopLoadDriver` 都在往这条链上挂表项、查这条链判重。
- 对第 2 批：`INITOS.C:1531` 的 `IoInitSystem` 调用点与本批第 ① 步的失败总闸是同一处；第 2 批地图里的“Io（1531）在 Cm（1437）之后、Ps(1)（1604）之前”在本批得到展开：Io 波内部还有 boot/system 两波与 ntdll 定位。
- 对第 4 批：驱动对象与设备对象都是对象管理器对象，`ObCreateObject` / `ObInsertObject` 的复用是本批第 ⑨ 步第 9 点；`DO_DEVICE_INITIALIZING` 门闩是对象“建好未就绪”状态在 I/O 侧的对应物。
- 对第 5 批：`PsLocateSystemDll` 的调用点（`IOINIT.C:743`）落在本批第 ④ 步第 6 点；`DriverEntry` 创建的系统线程退出时走 `LpcExitThread`（第 2 批第 ⑳ 步），故 LPC 必须在 Io 之前——本批的波次顺序与第 2 批的 LPC 论证互相印证。
- 对第 6 批：system 波的清单来自 `CmGetSystemDriverList` 对 SYSTEM hive 的直接 cell 遍历（第 ⑦ 步），排序与依赖解算复用第 6 批第 ⑱ 步的 `CmpFindDrivers` / `CmpSortDriverList` / `CmpResolveDriverDependencies`；引导日志的假脱机复用第 6 批的两阶段结构；第 ⑮ 步纠正了第 6 批问答关于“卷第一次被打开”的表述。
- 对第 8 批（下一批）：win32k 作为驱动被装载时走的也是 `IopLoadDriver`（会话空间标志版 `MmLoadSystemImage`），本批第 ⑬ 步的“一函数三用”与术语表第五节的会话空间驱动是第 8 批的前置知识。

## 第九部分：未钉死清单

1. `IopInitializeBootDrivers`、`IopInitializeSystemDrivers`、`IopInitializePlugPlayServices`、`PpInitSystem`、`PpLastGoodDoBootProcessing`、`IopDriverLoadingFailed`、`IopPnpDriverStarted` 本体不在 WRK 树内（`IO/PNPMGR/` 只有头文件）：boot 波如何调 DriverEntry、system 波循环体、PnP 枚举触发细节、LKG 账本处理、ErrorControl 处置实现、PnP 启动通知实现，均未逐行钉死。第 ⑦ 步的“system 波必调 IopLoadDriver”只是 `CmGetSystemDriverList` 头注释的契约反推。
2. `ErrorControl` 各值（0/1/2/3）的处置语义为公开文档口径，树内处置本体缺失，未逐行钉死。
3. `IopPrepareDriverLoading` 的 `Enum` 子键具体写入项未逐行读。
4. `IopResurrectDriver` 的复活条件（驱动对象何种状态可复活、何种状态不可）未逐行读。
5. `MiUseLargeDriverPage` 的触发注册表值未查。
6. 路线术语纠正：`IopInitializeDriverModule` 非树内函数名，实名为 `IopLoadDriver`（本批取材说明已纠）。

## 第十部分：问答补记（本批交付后追问时追加）

- **再初始化：为什么 DriverEntry 之后还要跑第二遍，C 依赖 A、A 也有再初始化时，C 的再初始化是否保证在 A 之后（2026-10-27 答疑补记）**：先给结论——**不保证**。再初始化队列不做依赖排序、不做拓扑排序，它的保证只有三条，下面逐条钉源码。
  - **顺序 = 登记顺序的 FIFO**：登记时插队尾（`IopInterlockedInsertTailList`，`IOSUBS.C:10023`），排空时从队首摘（`IopInterlockedRemoveHeadList`，`INTERNAL.C:10199`），一次摘一个、同步跑完一个再跑下一个，不并发。包的结构是 `REINIT_PACKET`（驱动对象、例程地址、上下文，`IOMGR.H:71`）。
  - **排空以“队空”为条件、每轮重查**：排空期间新登记的包（包括某个再初始化例程自己再登记的——`IoRegisterDriverReinitialization` 的文档原话就是“during their initialization or during their reinitialization”都能登记）追加到队尾、本轮一并跑掉，不会丢。但这也意味着顺序是“先登记先跑”，后登记的永远排在已登记的后面。
  - **两条队**：boot 驱动用 `IoRegisterBootDriverReinitialization`（`IOSUBS.C:9906`）进 `IopBootDriverReinitializeQueueHead`、打 `DRVO_BOOTREINIT_REGISTERED`，排空函数是 `IopCallBootDriverReinitializationRoutines`（`INTERNAL.C:10118`），但调用点不在树内（应在缺失的 boot 波代码里）；system 驱动用 `IoRegisterDriverReinitialization`（`IOSUBS.C:9967`）进 `IopDriverReinitializeQueueHead`、打 `DRVO_REINIT_REGISTERED`，`IoInitSystem` 在 system 波后排它（`IOINIT.C:770`）。动态装载（`NtLoadDriver` 路径）在每次装完后立即排一次 system 队（`INTERNAL.C:4670`），动态驱动的再初始化不等下次启动。
  - **它到底解决了什么**：`DriverEntry` 时的问题是“后加载的驱动根本不存在”——设备对象没建、派遣表没填、按名打开都失败。再初始化时保证的是“所有 system 波驱动的 `DriverEntry` 都跑完了”——设备存在了、派遣表填了。C 的再初始化可以打开 A 的设备、发 IRP；它通常依赖的是 A 的设备与派遣表（`DriverEntry` 的成果），不是 A 的再初始化的成果。机制把“不存在”问题降级为“存在但顺序自理”问题，没有把依赖问题彻底解决。
  - **你的场景在什么条件下成立、怎么解**：若 A、C 都只在 `DriverEntry` 里登记一次，且 A 比 C 先装（`Group` / `Tag` / `DependOnService` 保证），登记顺序 = 装载顺序，FIFO 恰好给出 A 先于 C——但这只是装载顺序正确时的副产品，不是机制保证的依赖排序。若 A 在自己的再初始化里又登记第二包，这包排在 C 的包之后，“A 的全部工作先于 C”即不成立——机制的保证单位是“包”，不是“驱动”。若 C 真要 A 的再初始化成果，正确工具是：(a) A 的再初始化里直接做依赖 C 的事或置位一个事件，C 的再初始化等该事件（再初始化跑在 `PASSIVE_LEVEL`，等待合法）；(b) PnP 的设备接口到达通知（`IoRegisterPlugPlayNotification`）——C 在 A 的设备接口出现时被回调，根本不依赖顺序假设，这是 PnP 世界解“设备出现依赖”的正解。剩下的顺序责任在装载顺序与驱动自己的同步手里，不在队列手里。

## 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，与正文互补。

- **IRP 派遣：函数指针逐层调用，栈位置下移**。`IoCallDriver` 经 `pIofCallDriver` 间接（`IO/IOMGR/IOSUBS.C:2239`，与 `IopSetIoRoutines` 同款的替换点设计）调下一层驱动的派遣例程：取驱动对象派遣表 `MajorFunction[主功能码]` 的函数指针、把 IRP 栈位置下移一格、调过去。转移方式：间接函数调用；每过一层用掉一个栈位置，这就是 IRP 要分大小规格的原因（栈位置数取决于设备栈深度）。派遣例程返回 `STATUS_PENDING` 表示“我接了，稍后完成”，返回其他值表示“本层已处理完，可继续向上传递或完成”。
- **IRP 完成：倒序调完成例程，跑在完成处理器的分发级别**。`IopfCompleteRequest`（`IOSUBS.C:3339`）按栈位置倒序调各层挂的完成例程（`stackPointer->CompletionRoutine`，`:3547` 一带）：每层完成例程可决定放行（继续向上传递）或截停（返回 `STATUS_MORE_PROCESSING_REQUIRED`，IRP 停在该层）。完成例程跑在完成处理器的 DISPATCH_LEVEL、借用完成上下文（可能是中断上下文或 DPC 上下文），不能等 I/O、不能缺页——需要等的事必须像懒刷一样转工作队列（见第 6 批补记）。
- **DriverEntry 调用：直接函数指针调用，跑在 System 进程上下文**。`IopLoadDriver` 内 `driverObject->DriverInit(driverObject, registryPath)`（`INTERNAL.C:4225`）是普通间接函数调用，IRQL 为 PASSIVE_LEVEL（可等待、可缺页、可建设备对象）。失败不是异常转移：返回码换成 `STATUS_FAILED_DRIVER_ENTRY`，驱动对象标临时并解引用，对象删光即卸载——转移方式仍是调用与返回，不是跳转。
- **DPC 排空点：与第 5 批补记同一机制**。驱动波内排队的 DPC（如定时器 DPC、完成 DPC）经 `KiDispatchInterrupt`（软件中断向量 → DPC 栈 → `KiRetireDpcList`）或换线程路径排空，见第 5 批补记，不在此重复。

### 追问：IRP 栈深度、中途加设备、指针谁调、每层填什么（2026-10-28）

三个问题一次讲透，先认错：上轮讲 IRP 时只说了“派遣表函数指针调用、栈位置下移”，没讲深度谁定、中途加设备、指针移动的归属、每层填表原则。

- **问题一：中途加设备，会不会有设备分不到堆栈层？不会，三层保证。** ① 挂接永远落在栈顶：`IopAttachDeviceToDeviceStackSafe`（`IO/IOMGR/IOSUBS.C:1470` 一带）先 `IoGetAttachedDevice(TargetDevice)` 走到链顶再挂，不存在“中间插入”；新设备在旧顶之上。② StackSize 的语义是“从本设备往下数需要的层数”，不是全栈总数：新设备 `StackSize = 旧顶 StackSize + 1`（`:1530` 一带），旧设备的值一个不动（它们往下数的层数没变）。所以新设备只影响发往新顶的新 IRP（分配时取新顶 StackSize，多一层）；发往旧顶的 IRP（绕开新设备）仍按旧尺寸，正确——因为它根本不经过新设备。③ 在途 IRP 的跳数是定死的：每层驱动调 `IoCallDriver` 时传的是自己存着的下层设备指针（挂接时记的），不是“现在我下面是谁”。新设备挂上后，在途 IRP 的调用链里没有它，它自然被 bypass；新 IRP 从新顶进入，尺寸已含新层。④ 挂接全程拿 `IopDatabaseLock`（排队自旋锁），新设备 StackSize 在锁内设好才放锁——分配读到的 StackSize 要么是旧值（IRP 不经过新设备），要么是新值（IRP 含新层），没有中间态。⑤ 真正会出事的情形是反向的：驱动把 IRP 转发到另一个更深的栈而层数不够——这时不是静默分不到，而是蓝屏 `NO_MORE_IRP_STACK_LOCATIONS`（见问题二）。所以分配时必须算够：`IoAllocateIrp` 一律取目标设备的 StackSize；需要穿越额外层时显式加 `extraStack`（`INTERNAL.C:4895` 挂载 IRP：`attachedDevice->StackSize + extraStack`，extraStack 按挂接链现算，注释原话“the number of stack locations that must be allocated in the IRP must include one for the file system itself, and then one for each driver that is attached to it”）。
- **问题二：堆栈指针是不是 I/O 管理器自动调？是，但要分清谁调、何时调。** 初始值：`IoInitializeIrp`（`IOSUBS.C:8127`）把 IRP 全清零，`CurrentLocation = StackSize + 1`，`CurrentStackLocation` = 数组末尾之后一个位置——指针初始指向“栈顶之外”，还没指向任何一层。下发时：驱动先填好下一层的位置（见问题三），再调 `IoCallDriver` → `IopfCallDriver`（`IO/IOMGR/IOMGR.H:1597`，内联函数）：`CurrentLocation--`（≤0 则蓝屏 `NO_MORE_IRP_STACK_LOCATIONS`）→ `CurrentStackLocation` = 下一位置 → 把 DeviceObject 写进该位置（供完成时用）→ 按该位置的 MajorFunction 字段查派遣表调驱动。**所以指针是 IoCallDriver 里自动移的，驱动自己不移它**（只填内容，不移指针——Skip 那个例外见问题三）。完成时 `IopfCompleteRequest` 反向走：指针逐层回升，每层按该位置的 CompletionRoutine/Control 决定调不调完成例程。方向记忆：数组下标 0 是最底层驱动，StackSize-1 是最顶层驱动；指针从数组外一路递减到 0（下发），再一路递增回去（完成）。
- **问题三：本层处理完，复制、现填还是跳过？原则是什么？先立铁律：每层驱动只能读自己的堆栈位置**（CurrentStackLocation），不能读上层的位置——上层的位置是上层驱动给下层的工作单，不是共享内存。下层驱动收到的参数，全来自自己这一层的位置。所以上层驱动在下发前，必须把下一层的位置准备好，三种方式按“下层需要什么”选。① 复制（`IoCopyCurrentIrpStackLocationToNext`，`INC/IO.H:4434`）：下层要和本层一样的参数（直通/过滤驱动）。复制本层位置到下一层，但**不复制** CompletionRoutine/Context/Control（Control 清零）——宏注释原话用法就是“想挂完成例程但不想改参数时，先复制、再 IoSetCompletionRoutine、再 IoCallDriver”。为什么不复制完成例程？因为完成例程是按层挂的：本层的完成例程属于本层，下一层的完成例程要由本层这次现挂，完成时倒序调——复制了就会串层。② 现填（`IoGetNextIrpStackLocation` + 手动写字段）：下层要不一样的参数（翻译：类驱动把读请求翻译成设备专属请求、分包、改 MinorFunction/参数）。驱动自己写 MajorFunction/MinorFunction/Parameters——注意：IoCallDriver 是按下一层位置的 MajorFunction 字段查派遣表的，现填时 MajorFunction 必须写对，否则调错入口。③ 跳过（`IoSkipCurrentIrpStackLocation`，`INC/IO.H:4469`）：本层什么都不需要——不改参数、不挂完成例程。Skip 把指针 +1，随后 IoCallDriver 的 -1 抵消，净效果是下层驱动**共用本层的位置**（同样的参数、同样的完成例程——也就是上层驱动挂的那个）。完成时倒序走过该位置，调的是上层驱动的完成例程，本层直接被跳过（它没有自己的位置，也就没有完成例程可调）。原则一句话：**位置只在两种情况下消费——（i）要进完成路径（挂完成例程）；（ii）要给下层不同的参数。纯直通两者都不要，就不消费。**
- **附带：返回状态的原则（“只返回状态”也得讲清）。** ① `STATUS_PENDING` = “我接了，稍后完成”：IRP 所有权转给异步完成路径（中断/DPC/工作线程里的 `IoCompleteRequest`），调用者此后不得再碰 IRP（可能已被完成释放）。② 非 Pending 状态返回时，必须二选一：要么本层已经 `IoCompleteRequest` 过了（同步完成，状态沿调用链向上传）；要么下层已经同步完成过了（下层的完成链已跑完，本层只是把状态原样向上传）。绝不能既不完成又返回非 Pending（IRP 丢了，没人完成），也绝不能完成两次（第二次完成会重复调完成例程、重复释放）。③ 完成例程返回 `STATUS_MORE_PROCESSING_REQUIRED` = “截停”：完成链到此为止，IRP 不继续向上传、不释放——用于驱动要扣住 IRP（重试、自己稍后完成）的情形；返回其他值 = “放行”，完成链继续向上一层。④ 不处理的主功能码：派遣表默认填 `IopInvalidDeviceRequest`（`IOSUBS.C:14107`，第 7 批已钉），返回 `STATUS_INVALID_DEVICE_REQUEST`——“不处理”也是有明确返回码的，不是静默丢弃。

### 追问：Copy + 改是不是比纯现填快（2026-10-28）

分两层答：写代码更快，执行不更快；快不快不是选哪个的依据。

- **先算账**。`IoCopyCurrentIrpStackLocationToNext`（`INC/IO.H:4434`）是 `RtlCopyMemory(下一层, 本层, FIELD_OFFSET(IO_STACK_LOCATION, CompletionRoutine))` + 下一层 Control 清零。结构体（`INC/IO.H:2303`）是主功能码、次功能码、Flags、Control（4 字节）+ Parameters 联合（约 16–20 字节）+ DeviceObject、FileObject，截断点之前约 32 字节——Copy 就是一次约 32 字节的内存拷贝。`IoGetNextIrpStackLocation` 只是一次减法（当前指针 -1），约等于零成本。手动填是每字段一次写。临界点：保留字段多 → Copy 划算（手动要逐个写保留字段）；改动多 → 手动划算（拷贝了又全覆盖 = 白拷）。
- **真正决定因素第一条：漏字段**。手动填必须记得 FileObject（下层驱动依赖它，如文件上下文、关联文件对象）、Flags 等；Copy 自动带过去。直通驱动手动填漏 FileObject 是经典 bug。Copy + 改在“改得少”时成立，首先是省代码、不漏字段，其次才是写得少。
- **真正决定因素第二条：union 残留**。Parameters 是联合，同一块字节按主功能码有不同解释（如 Read 的 ByteOffset 与 DeviceIoControl 的 IoControlCode/Type3InputBuffer 共用字节）。Copy 后换主功能码、只改几个字段，残留字节会被下层按新解释误读——所以 Copy + 改只适用于主功能码不变（同操作直通）；换操作必须现填整个联合（或先清零再填）。
- **Copy 后再 GetNext 不是多余动作**。Copy 本身不移动指针，它内部就是用 GetNext 取到的指针写；驱动随后 GetNext 取的是同一个指针，用来改字段、挂完成例程。宏注释原话的用法就是“先复制、再 IoSetCompletionRoutine、再 IoCallDriver”——Copy 故意不复制 CompletionRoutine/Context/Control（截断点 + Control 清零），就是为了让调用者用 `IoSetCompletionRoutine`（`INC/IO.H:4323`，同样取下一层位置写完成例程、上下文、控制位）现挂。所以 Copy + GetNext + 改是文档原生的标准序列。
- **结论**。IRP 分发本身（查表调用、自旋锁、真正的 I/O）的开销面前，这点拷贝在噪声里；选哪个看“下层参数和本层有多像”+“主功能码变不变”，不看速度。你的直觉在“改得少”时成立，在“改得多”时不成立。

### 追问：下层堆栈层数据是不是上层初始化的（2026-10-28）

对，就是这个模型，但要补三个精度，否则结论会过头。

- **模型本身是对的**。每层位置是上层驱动给下层的工作单，上层负责填，下层只读自己的。链的起点是创建者：`IoBuildAsynchronousFsdRequest`（`IO/IOMGR/IOSUBS.C:1569` 一带）填顶层位置的 Parameters.Read/Write.Length/ByteOffset，然后 `IoCallDriver` 分发；每层驱动收到后，再按上一条的三种方式（复制、现填、跳过）准备下一层的位置。所谓“上层”包括创建者——创建者就是第 0 层。
- **精度一：未填不是随机垃圾，是确定性的全零**。新 IRP 全清零（`IoInitializeIrp`，`IOSUBS.C:8127`）；复用 IRP 也重新初始化清零（`IoReuseIrp` → `IopInitializeIrp`）。全零位置的主功能码 = 0 = `IRP_MJ_CREATE`（`INC/IO.H:51`），参数全零。所以下层会被按 CREATE 分发、带零参数——确定性的错，不是随机错。后果分三档：走错入口（本想读、结果进了 CREATE 派遣）；空指针（FileObject 为 NULL，下层一用就崩）；最危险的是静默错数据（Length/偏移为零 → 读写错位置，不崩溃但数据错）。
- **精度二：两个例外**。① DeviceObject 字段不是上层填的，是 `IopfCallDriver` 自己写的（`irpSp->DeviceObject = DeviceObject`，`IO/IOMGR/IOMGR.H:1597` 一带）——分发代码填设备对象，上层填参数，分工如此。② Skip 不是“没填”，是“不用填”：下层故意共用本层位置（同样的参数、同样的完成例程），那是设计，不是疏漏。
- **精度三：没有运行时兜底**。正式版不检查位置内容对不对——只检查层数够不够（层数不够才蓝屏 `NO_MORE_IRP_STACK_LOCATIONS`），内容错不会蓝屏，只会错操作/崩溃/静默 corrupt。受检版 + 驱动验证程序（I/O 验证）会抓这类错误，但正式版没有。所以“上层必须填好下层位置”是驱动编写的硬规矩，不是框架保证的——这也是漏填下层位置成为经典驱动 bug 的原因。

### 追问：上层如何得知下层完成了（2026-10-28）

先纠前提：完成例程不是设在“自己这层”，是设在下一层——上一条“每层只能读自己的位置”没把“写下一层”点透，导致了这个误解。

- **设在哪一层**。`IoSetCompletionRoutine`（`INC/IO.H:4323`）写的是 `IoGetNextIrpStackLocation` = 当前位置 -1，即下一层位置。准确说法是：驱动**读**自己这层、**写**下一层。设例程的驱动（上层）在下一层位置上写完成例程地址、上下文、控制位（成功/失败/取消三档），然后 `IoCallDriver` 把指针移到该位置，下层驱动跑起来时那一层就是它的“当前层”。
- **上层得知的机制：不是下层通知，是 I/O 管理器逐层向上走**。下层驱动完成时只调 `IoCompleteRequest`，它不需要读上层的位置；`IopfCompleteRequest`（`IO/IOMGR/IOSUBS.C:3339`）接管：for 循环从当前层开始，指针预先 +1，然后每层按控制位与完成状态（成功调 `SL_INVOKE_ON_SUCCESS`、失败调 `SL_INVOKE_ON_ERROR`、取消调 `SL_INVOKE_ON_CANCEL`）决定调不调存着的例程。例程被调到 = 上层得知。传给例程的设备对象是设例程那层自己的（指针预先 +1 的把戏：例程存在下层位置，但传的是上层位置的 DeviceObject），上下文是设例程时传的。所以上层驱动不需要、也不能去读下层的位置——走访是框架做的，通知就是一次函数调用。
- **完成例程的约束**。跑在完成上下文（常为分发级别、任意线程上下文），不能等；调用前该位置已被清零（`ZeroIrpStackLocation`，只留错误位），例程不能依赖位置里的参数；返回 `STATUS_MORE_PROCESSING_REQUIRED` = 截停（完成链到此为止，IRP 不再向上、不释放，例程自己负责后续——注释原话“the operation will be completed by this routine again sometime in the future”）；返回其他值 = 放行继续向上。另有防呆：重复完成 → 蓝屏 `MULTIPLE_IRP_COMPLETE_REQUESTS`（函数入口检查 `CurrentLocation > StackCount + 1`）；完成时状态不能是 `STATUS_PENDING`（断言）。
- **走到顶之后，请求者如何得知**。走访跑完所有层（或被截停除外），函数头注释第 3–6 步：关联 IRP 计数减一；解锁 MDL；同步请求写状态块、signal 事件；最后向请求线程排特殊内核 APC——请求线程回用户态或可警告等待时投递，最终跑用户完成例程或 signal overlapped 事件。也就是说：层与层之间靠完成例程调用通知，最顶层与请求线程之间靠事件/APC 通知，两段机制不一样。

### 追问：某层返回 STATUS_PENDING 之后（2026-10-28）

分两条线：状态向上传（同步发生在分发返回链上），IRP 本体以后再完成（异步发生在硬件完成之后）。另有一条分支：上层不向上传，改同步等。

- **状态向上传**。下层驱动 L 启动异步操作（把 IRP 挂进自己的队列 / 发给硬件），调 `IoMarkIrpPending`（当前层位置 Control 置 `SL_PENDING_RETURNED`，`INC/IO.H:4001`，注释原话：标记“initial status of STATUS_PENDING was returned to the caller”，供完成时判断“是否要完全完成”）→ 返回 `STATUS_PENDING` 给上层 U。U 从 `IoCallDriver` 拿到 PENDING 后必须原样向上传（返回 PENDING 给它的调用者），每层都这样，直到发起者。发起者分两支：同步请求（如非 overlapped 读、关机路径）拿到 PENDING 后 `KeWaitForSingleObject` 等事件（`IO/IOMGR/IOSUBS.C:11288` / `:11464` 一带就是这个模式：`IoCallDriver` 返回 PENDING → 等事件），线程挂起；异步请求（overlapped）把 PENDING 直接返回给用户态，线程不等待，结果以后通过事件/APC 送达。
- **IRP 本体线**。IRP 仍活着，所有权在 L。上层驱动返回 PENDING 后不得再碰 IRP（可能随时被完成释放）。硬件完成 → 中断 → ISR → DPC（典型）→ DPC 例程填 `IoStatus`（状态 + 实际传输字节数）、做收尾 → 调 `IoCompleteRequest`。`IopfCompleteRequest` 走访向上（上一条已讲）：从当前层起，每层按控制位调完成例程；走访中 PENDING 标记向上传递：每层先把本层 Control 的 `SL_PENDING_RETURNED` 存进 `Irp->PendingReturned`（`IOSUBS.C:3506`），没有例程可调的层、且标记已置、且上面还有层时，调 `IoMarkIrpPending` 把标记写进上一层位置——于是“曾返回过 PENDING”这个事实随走访一路向上传，供上层完成例程与最终收尾判断。完成例程返回 `STATUS_MORE_PROCESSING_REQUIRED` → 走访截停，IRP 停在该层（见分支）；否则继续向上。走访到顶 → 最终收尾：关联 IRP 计数、解锁 MDL、同步请求 signal 事件（唤醒等事件的线程）、异步请求向请求线程排特殊内核 APC → 用户态得知结果；IRP 释放（I/O 管理器分配的）或归还。注意：完成时状态不能还是 PENDING（`IOSUBS.C:3458` 断言）——完成就是要填真实状态，填 PENDING 意味着忘了设状态。
- **分支：上层不向上传，改同步等**。U 想同步等 L 完成：U 挂完成例程（例程里 signal 事件 + 返回 `STATUS_MORE_PROCESSING_REQUIRED` 截停走访，防止 IRP 被释放/继续向上），U 自己 `KeWaitForSingleObject` 等事件；醒来后 U 再次调 `IoCompleteRequest` 恢复走访向上（或 U 自己处理完返回最终状态）。这个模式里，U 的 dispatch 表面上可能返回最终状态而不是 PENDING——因为 U 等完了才返回。截停后 IRP 的指针停在 U 的下一层位置，U 恢复完成时走访从 U 的位置继续向上——这就是 `STATUS_MORE_PROCESSING_REQUIRED` 的用途之一。
- **逃生口与出错情形**。取消逻辑（`IoCancelIrp` + 取消例程）可以把迟迟不完成的 IRP 完成掉（`STATUS_CANCELLED`），防止无限挂起。若 L 返回 PENDING 后再也不完成 → IRP 泄漏；同步等者永久挂起，异步等者永远收不到信号——没有框架兜底，驱动必须保证每条 PENDING 的 IRP 最终都有 `IoCompleteRequest`（或被取消例程完成）。若 U 拿到 PENDING 却不向上传、也不等、直接返回成功 → 上层以为完成了，IRP 还在 L 手里，之后 L 一完成就走访向上、调完成例程、释放——调用方与完成方对 IRP 状态的认知分裂，经典 bug。

### 故事版：IRP 全程——一份批文的旅行（2026-10-28）

把 IRP 想成一份多联批文。I/O 管理器是收发室。设备栈是从上到下排队的一串科室（驱动）。批文有 N 联（堆栈层），N 由收发室按科室数裁好。每联上写：办什么事（主功能码）、怎么办（参数）、办完找谁（完成例程 + 上下文 + 控制位）。

- **第一章：批文诞生**。有人（用户线程）要办事 → 收发室按顶层科室的层数裁好批文（`IoAllocateIrp`），整份清零（白纸），指针停在最后一联之外（还没指向任何一联）。发起人（创建者）填好最上面一联（顶层参数：读多少、从哪读），交给收发室。注意：联数在裁的时候就定了；中途新科室只能挂在最上面（挂接永远落栈顶），在途的批文走的是老路线、见不到新科室；新批文才按新层数裁。
- **第二章：往下送**。收发室递送规矩（`IoCallDriver`）：指针下移一联 → 把收件科室名字写在这一联上（DeviceObject）→ 按这一联上写的“办什么事”查科室的职责表（派遣表）叫人。指针是收发室移的，科室自己不移。每个科室收到批文，只准看自己这一联（不准翻上面科室的联）。看完要往下送时，必须先把下一联准备好，三种准备法：① 照抄（Copy）：下一科室办同样的事 → 把本联抄到下一联，但“办完找谁”那栏故意不抄（留空，等自己现填回执电话），然后填上自己的回执（完成例程），再交收发室。② 现写（填）：下一科室要办不一样的事（翻译、分包）→ 下一联重写，主功能码必须写对（收发室按它查表叫人，写错就叫错人）。③ 不写（Skip）：本层纯过路，什么都不改、也不留回执 → 指针先 +1 再被收发室 -1 抵消，下一科室直接共用本联（同样的事、同样的回执电话），完成时回执直接打给上上层，本层被跳过。原则：联只在两种情况下消耗——要留回执（进完成路径），或要给下层不同的事；纯过路不消耗。如果上层忘了填下一联：下一联是白纸（全零）→ 主功能码 0 = CREATE，参数全零 → 下科室按 CREATE 办、拿零参数——确定性的错，不是随机错。科室职责表里没写的活，默认填着“不办”，返回“办不了”，不是悄悄扔掉。
- **第三章：到底与 PENDING**。最底层科室接了批文，但活要硬件干、得等。于是它干两件事：给批文盖个章“已收，办结另行通知”（`IoMarkIrpPending`，在本联控制位上打标记），然后回上层一句 PENDING。PENDING 像接力棒一样一层层往上传：每层收到 PENDING，原样往上传（除非它选择自己等，见第五章）。传到发起人分两支：同步办事的人（非 overlapped）拿到 PENDING 就坐下等（等事件），线程挂起；异步办事的人（overlapped）拿到 PENDING 就先走了，继续干别的，结果以后再通知。铁律：上层收到 PENDING 后，不许再碰批文——它可能随时被办结、被释放，碰就是 use-after-free。
- **第四章：办完了，往回传**。硬件好了 → 摇铃（中断）→ 值班员（ISR）只干急事：应答、把数据搬到稳妥地方、喊 DPC 来善后 → 铃声的分发中断一到，DPC 在专用栈上跑起来 → DPC 填好回执单（IoStatus：成没成、办了多少）→ 喊一声“办完了”（`IoCompleteRequest`）。收发室接手往回传（`IopfCompleteRequest`）：从当前这联开始，一联一联往上走。每联先看“回执开关”（控制位）：这联留的回执电话是不是在本次结果下要打（成功打成功档、失败打失败档、取消打取消档）→ 要打，就先把这联除了错误位以外擦干净（防止回执里偷看参数），然后打电话（调完成例程，报上留电话那科室的名字和它留的上下文）。“办完叫我”就是这么实现的——不是下层通知上层，是收发室走访时把电话打过去。回执电话那头可以说“知道了，继续往上传”（返回其他值）或“先扣住，别往上传了”（返回 `STATUS_MORE_PROCESSING_REQUIRED`）——扣住后批文停在这一层，收发室不再碰它，扣住的科室自己负责后续（重试、或以后再喊一次完成）。走访时 PENDING 的章也一路向上传：每联把“曾返回过 PENDING”的标记抄到上一联，让上层和最终收尾知道“这份批文当初是先回 PENDING、后办结的”。防呆：同一份批文完成两次 → 蓝屏；完成时状态还填 PENDING → 断言（忘了填真实状态）。
- **第五章：到顶交差，与自己等的分支**。走访到顶：收发室做最后收尾——关联批文计数减一、解锁内存页；同步等的人：敲事件（等事件的线程醒来，拿到最终结果）；异步的人：给请求线程排一个特殊内核 APC（请求线程回用户态时投递，跑用户完成例程或 signal overlapped 事件）。批文随后释放或归还。分支（上层选择自己等）：上层不想把 PENDING 往上传，它留回执电话时说“办完先别往上传，摇我起床”（例程里 signal 事件 + 返回截停），自己坐下等事件；醒来后再喊一次“继续往上传”，走访从它那层继续向上。于是它的派遣表面上返回的是最终结果，而不是 PENDING——因为它等完了才回。逃生口：批文迟迟办不完，取消逻辑可以把它完成掉（`STATUS_CANCELLED`），防止无限等；没人取消、底层又永远不完成 → 批文泄漏、同步等者永挂、异步等者永收不到信——框架不兜底。
- **收束一句话**。批文往下是“每层填下一联、收发室移指针、按联上主功能码叫人”；往回是“收发室逐联往上走、按开关打回执电话”；PENDING 是“已收、办结另行通知”的章，章向上传、批文本体以后再走完。

### 追问：PENDING 期间 IRP 保存在哪里，ISR 怎么再找到它（2026-10-28）

先说结论：批文没有被复制、没有被搬走——它一直在分配时的那块非分页池内存里；被“保存”的是它的指针，存在最底层驱动的待办夹里。

- **批文本体在哪**。`IopAllocateIrpPrivate`（`IO/IOMGR/IOSUBS.C:745` 一带）从 `NonPagedPool` 分配（`:881`，tag `' prI'`）。PENDING 期间批文不释放、不搬家，就在那块内存里——这正是 ISR（设备中断级）、DPC（分发级别）能碰它的前提：这两个级别不允许缺页，批文若在分页池，一碰就蓝屏。所以“保存批文”首先不是找个地方存它，而是它本来就一直在非分页池里活着，直到完成才释放。
- **待办夹三种形态（存的是指针）**。① 设备扩展里的当前批文指针 / 待办链表（最常见）：驱动发硬件命令前，把批文指针写进设备扩展（单批文设备记一个指针，多批文设备挂一条链表，自旋锁保护），然后返回 PENDING。② 系统设备队列：`IoStartPacket`（`IOSUBS.C:11674`，注释原话“If the device is already busy, then the packet is simply queued to the device queue”）——设备忙就把批文排进设备队列，驱动不用自己建链表；`IoStartNextPacket` 取下一个调 StartIo。③ DPC 上下文：`KeInitializeDpc` 的 DeferredContext（初始化时定的任意数据，`KE/DPCOBJ.C:133`）或 `KeInsertQueueDpc` 的 SystemArgument1/2（排队时带的两个无类型参数，`:203`）——ISR 把批文指针塞进 DPC 参数，DPC 例程收到即用。
- **ISR 怎么找到待办夹**。登记中断时，驱动把设备扩展地址当服务上下文交过去（`IoConnectInterrupt` 的 ServiceContext 参数）；中断一到，`KiInterruptDispatch` 把服务上下文当参数递给 ISR（`InServiceContext[edi]`，`KE/I386/INTSUP.ASM:707` 一带）。所以 ISR 的寻物链是：服务上下文 → 设备扩展 → 待办夹（当前批文指针 / 设备队列头）→ 批文指针。一般 ISR 不直接办结（中断级不能久留）：它把指针塞进 DPC 参数、排 DPC；DPC 在分发级别拿到指针 → 调 `IoCompleteRequest` 走访向上。也就是说：ISR 找批文靠服务上下文，DPC 拿批文靠参数传递，两段各有各的递法。
- **第二处记录（不是 ISR 的路）**。线程的 IRP 链表（`IopQueueThreadIrp`，创建时挂）也记着它，供取消 / 线程退出清理用——ISR 不走这条路找批文，不要把“线程链表”当成完成查找的路径。
- **归属收束**。PENDING 期间批文归最底层驱动所有（它握着指针）；上层不许碰（前面已讲）；设备对象在 IRP 未完成前不会被卸载（`IoCallDriver` 调前拿设备对象引用）。所以“批文在哪”完整答案是：本体在非分页池，指针在底层驱动的待办夹（设备扩展 / 设备队列 / DPC 参数），ISR 靠服务上下文找到待办夹。

### 追问：为什么上层和最终收尾必须知道“当初先回了 PENDING”（2026-10-28）

一句话：这个标记回答的是一个问题——**调用方还在分发栈上等结果，还是已经拿了 PENDING 走人了？**完成方必须按答案做不同的事；做错就是重复投递或提前释放。

- **两种结局**。PENDING 没返回过（同步完成 / 同步失败）：调用方还坐在分发调用里，结果顺着返回值往回走，调用方自己处理状态、自己收尾。PENDING 返回过：调用方早就走了（同步调用方停在事件上等，异步调用方干别的去了），结果没人接——完成方必须全套送达：signal 事件、排用户 APC、送完成端口、把状态拷进调用方的 I/O 状态块、释放 IRP。没有标记，完成方分不清该“送”还是该“不送”。
- **树内证据一：用户 APC 那头会丢弃**。`IO/IOMGR/INTERNAL.C:1053`（请求线程的用户 APC 例程，注释原话“If this I/O operation did not complete successfully through the dispatch routine of the driver, then drop everything on the floor now and return to the original call point in the MPW”）：`!PendingReturned && NT_ERROR(status)` 时直接释放 IRP、返回——因为同步失败的结果调用方在分发返回链上已经拿到了，APC 再送一次就是重复投递。只有 PENDING 返回过，才把状态拷给调用方、跑用户 APC 例程。
- **树内证据二：同步完成的优化路径靠它提前返回**。`IO/IOMGR/IOSUBS.C:3760`（完成尾部，注释原话“determine whether or not this is a synchronous I/O operation that is being completed in the context of the original requestor. If so, then an optimal path through I/O completion can be taken”）：`IRP_DEFER_IO_COMPLETION && !PendingReturned` 时直接返回——调用方还在原始请求者上下文里，让原调用路径自己收尾，不必走全套完成。PENDING 返回过时这条优化不适用，必须全套跑完，因为原调用方早就不在分发栈上了。
- **树内证据三：同步出错时靠它决定要不要 signal 事件**。`INTERNAL.C:1510` 一带（注释原话“This is an I/O operation that completed as an error for which a pending status was returned and the I/O operation is synchronous. For this case, the I/O system is waiting on behalf of the caller”）：PENDING 返回过 + 同步 I/O 时，I/O 系统正替调用方等着，所以完成必须把文件对象事件置 signal、把状态还给等着的人。没返回过 PENDING 时没人等，signal 就无意义，状态走返回值。
- **为什么要一路向上传**。标记的消费者（用户 APC、完成端口投递、defer 尾部、事件 signal）全坐在顶层或顶层之上，所以标记必须活着走到顶。走访自动抄的只覆盖“没有完成例程的层”（else 分支）；带完成例程的层要自己重打标记——树内现成例子 `RAW/RAWDISP.C:230`（注释原话“If IoCallDriver returned PENDING, mark our stack location with pending”）：`if (Irp->PendingReturned) IoMarkIrpPending(Irp);`。这就是上层完成例程需要知道的原因之一：它不重打，标记就死在它这一层，顶层的消费者收不到。
- **完成例程视角的另一层意思**。`PendingReturned` 还告诉完成例程这次完成是同步内联还是异步事后：为假时，完成例程跑在分发路径里、原请求线程上下文、自己的分发例程还没返回；为真时，完成跑在任意上下文（常是完成方的 DPC/任意线程），自己的分发例程早已返回，不能假设原线程上下文。
- **精度**。成功时走哪条路差别不大（`INTERNAL.C:1294`：成功一律正常投递）；真正靠这个标记分流的是出错时（送不送、丢不丢）和同步完成优化时（提前返回还是全套跑完）。所以“让上层和最终收尾知道”不是记账，是防重复投递、防提前释放、防该醒的人没醒——三条都是完成时才能定、且只有顶层能执行的操作。

### 追问：PENDING 还在上发路上、设备已完成——会不会两条路径打架（2026-10-28）

先纠正前提：**没有两条路在“上发 IRP”。** PENDING 上发 = 返回值沿分发调用栈往回退（同步、函数返回），IRP 本体不动，还在最底层驱动手里。完成上发 = `IoCompleteRequest` 走访（DPC 发起），沿堆栈位置往上调完成例程、最后收尾。两者是“一句话”和“一件事”的区别：PENDING 是“我先回你一句还没好”，完成是“好了，结果在这”。竞态确实存在，但被三层机制消化了：

- **同一核上先串行化**。分发 unwind 跑在 PASSIVE_LEVEL；中断打断它，ISR 排 DPC，中断返回时分发中断先被投递——DPC 走访先跑完，线程才恢复继续退 PENDING。所以同一核上是“走访先跑完，PENDING 再继续退”，不是并行。
- **不同核上真并行，但释放权不在走访手里——这是根本原因**。走访（DPC）干的事：逐层调完成例程、清位置、signal 事件（`KeSetEvent(Irp->UserEvent)`，`IO/IOMGR/IOSUBS.C:3710` 一带）、向请求线程排特殊内核 APC（`KeInsertQueueApc(&Irp->Tail.Apc, ...)`，`:3728`）。它**不释放顶层 IRP**——释放是特殊内核 APC（`IopCompleteRequest`，`IO/IOMGR/INTERNAL.C:1084` 一带）在请求线程上下文里干的，任务清单原话“If no APC is to be queued, then free the packet now”。特殊内核 APC 什么时候能跑？必须等请求线程离开分发路径、降到可投递 APC 的状态；而分发 unwind 恰恰还没跑完——线程还在分发栈里。所以 APC 必然排在 unwind 之后，不可能在 unwind 中途把 IRP 释放掉。于是：哪怕 DPC 走访已经跑完了（完成例程都调过了、事件 signal 了、APC 排队了），IRP 依然活着；分发 unwind 恢复后，分发例程按规矩做收尾（比如补打标记、原样返回 PENDING）是安全的——它碰的 IRP 还没被释放。unwind 到顶：同步请求方等事件——事件已被走访 signal，等立刻返回，读的是自己的 I/O 状态块（不是 IRP）；异步请求方把 PENDING 交给用户态，结果随后由 APC 送达（APC 里先把状态拷进用户 I/O 状态块，再释放 IRP、再排用户 APC）。没有“两次上发”：PENDING 退的是状态值，走访送的是结果，两者各司其职。
- **完成例程先于分发返回跑，是合法的、被设计好的**。走访调某层完成例程时，该层分发例程可能还没返回（还在退 PENDING 的路上）。这没问题，因为完成例程不靠分发栈——它靠分发时留的 Context 参数（完成例程的上下文是留好的，不是分发栈上的局部变量）。所以“例程跑了、分发还没退”不是乱序，是框架允许的正常时序；完成例程里按 `PendingReturned` 重打标记（RAWDISP 模式）也正是为这个时序准备的。
- **真正的红线只有一条：分发返回 PENDING 后，不许再对 IRP 做“业务操作”**（读结果、改参数、二次完成）。标记传递不靠分发例程在返回后补碰 IRP——走访自己会抄（无例程层 else 分支自动抄、有例程层例程里重打），顶层消费者（APC、完成端口、defer 尾部）拿到标记做分流（上一条已讲）。所以两条路各走各的，交汇点只有一个：事件（同步等的人靠它醒）和 APC（异步结果靠它送），而这两者都被设计成“走访发信号、请求线程收信号”，不存在两个发送者。
- **例外一句**。分页 I/O 与关闭路径是特例——走访尾部自己拷状态、signal、释放（注释原话三步：Copy the I/O status block、Signal the event、deallocate the IRP），不走 APC（因为 APC 可能再缺页、或 APC 重入）。但那两条路的发起方本来就不走“分发返回后补打标记”这套，状态在释放前已拷走，等的人读的是拷走的状态块，同样不打架。
- **收束**。不是“两条路径在上发 IRP”，而是“一条路退状态值（PENDING），一条路送结果（走访 + 事件/APC）”；释放权捏在请求线程的 APC 手里，保证了 unwind 期间 IRP 一直活着；完成例程先于分发返回跑是合法时序。

### 故事版：PENDING 还在上发路上、设备已完成——会不会撞（2026-10-28）

人物不变：批文是 IRP，最底层科室是底层驱动，车间是硬件，值班员是 ISR，夜班善后组是 DPC，发起人是请求线程，发起人自己的文书是特殊内核 APC，铃是事件。

- **第一幕：信使与批文分家**。批文到了最底层科室，科室说“活要车间干，得等”，盖个章“已收，办结另行通知”，派信使往上回话：“还没好”（PENDING）。信使一层层往上走，每到一层，那层科室把话原样往上传。关键：**信使手里没有批文**，批文还躺在最底层科室的待办夹里。信使传的只是一句话，批文本体根本没动。
- **第二幕：半路上车间完工了**。摇铃 → 值班员 → 夜班善后组 → 善后组从待办夹取出批文，开始往回送：逐层打回执电话、清联、最后到顶：敲铃（事件）+ 给发起人留一张“结果条”（APC）。现在两件事同时在进行：信使还在往上走（一句话），善后组也在往上走（批文本体）。会不会撞？
- **第三幕：同一栋楼（同一核）——善后组先行**。信使走在楼道里，铃响了，楼规矩是“善后组先行”：信使靠边站，善后组先走完，信使再继续走。所以同一核上根本没并行，是善后组先跑完、信使再继续传话。
- **第四幕：两栋楼（不同核）——真并行了，为什么不撞**。信使在 A 楼往上走，善后组在 B 楼往上走。三个原因：① 善后组到顶后**不销毁批文**：它只敲铃 + 留结果条。销毁批文的是发起人自己的文书（APC），而文书只有等发起人停下来（等铃、或回工位）才能开工；信使还在走路时，发起人正在分发大楼里走着，文书插不进来。所以信使走路期间，批文一定还活着——信使就算想看一眼也不会扑空；更何况信使根本不碰批文，只传话。② 两句话各司其职：同步等的人坐在等候室，铃一响就醒——铃早响了，他醒得更快，不会死等；异步的人先听到“还没好”走了，稍后收到结果条——一句是“先回你一声”，一句是“结果来了”，不冲突。③ 回执电话先于信使到达是正常的：善后组打回执电话时，某层科室的信使可能还没走到它那层——没关系，回执电话打的是科室预先留的号码（完成例程 + 上下文），不是找信使本人。电话那头该干的活（记账、重盖章）照干，干完说“继续往上传”或“先扣住”。
- **第五幕：章为什么要一路向上传**。善后组每过一层，把“当初先回了还没好”这个章抄到上一联——因为到顶后，发起人的文书要靠这个章判断：发起人是不是还在楼里等？在（没回过 PENDING），结果顺着信使的话已经带回去了，文书就把结果条扔掉、不重复送；不在了（回过 PENDING），文书必须把结果条送到发起人手里、再销毁批文。章不到顶，文书分不清。
- **红线**。信使传完“还没好”后，任何科室不许再动批文做业务（改参数、读结果、再办一次）——批文已经在善后组手里了，动就是两边各改各的。
- **例外**。车间直供的两种活（分页 I/O、关闭）不走“文书销毁”这套，善后组到顶自己拷结果、敲铃、销毁——但那两单的发起人本来就不派信使，结果在销毁前已拷走，等的人读的是拷走的那份，同样不撞。
- **收束**。从来不是两条路在送同一份批文；一条路送的是话（PENDING），一条路送的是批文本体与结果（走访 + 铃 + 结果条）；销毁批文的钥匙捏在发起人自己的文书手里，而文书一定在信使走完之后才开工——所以两条路永远撞不上。

### 故事版：结果先到了、PENDING 后到了——语义怎么自洽（2026-10-28）

先认同：如果你把 PENDING 理解成“你的结果还要等一等”，那结果先到、PENDING 后到，确实不自洽——已经拿到结果的人，又收到“还要等”，荒唐。但 PENDING 从来不是这个意思。**PENDING 的意思是“结果不走我这条路”，而不是“结果还没好”。** 整个自洽都建立在这句话上。

- **两条渠道，说的不是同一件事**。信使的话（PENDING，沿分发返回链往回退）只说一件事：“结果不走我这条路，你去查异步渠道。”它不说结果什么时候好，也不说结果是什么。善后组送的（走访 + 铃 + 结果条）才是结果本身。所以信使后到，传的也不是“还要等”，而是“你去查异步渠道”——发起人听完转头一查，铃已经响了、结果条已经在桌上了，直接取走。两句话不冲突，因为它们说的根本不是同一件事：一句指路，一句给结果。
- **铃是状态，不是消息——这是顺序无关的根源**。消息发过去就没了，错过就错过；状态一直留在那，晚到也能查到。事件一旦 signal 就一直是 signaled（直到有人复位），端口项一旦入队就一直在队里等取，结果条一旦留下就一直在桌上。所以“结果先到”不会造成“错过”：PENDING 先到，发起人坐下等铃；结果先到，发起人起身一看铃已响。两种顺序，同一个结局：发起人拿到结果。Windows 把异步完成做成“状态”（signal 的事件、入队的端口项、排队的 APC），而不是“一次性消息”，正是为了让“结果先到、通知后到”依然自洽。
- **同步一幕（坐下查铃，立刻起身）**。信使后到，传话“结果不走我这条路”；发起人坐到等候室查铃——铃已经响了。更关键的是：他在等的过程中，文书（特殊内核 APC）会先把状态拷到他的结果单（用户 I/O 状态块）上，再让他起身——特殊内核 APC 会打断等待先跑，跑完等待才返回，这是内核的保证，不是巧合。所以他起身时读到的结果单一定是最新的，没有多等一秒，也没有读到旧结果。
- **异步一幕（先走，回头一查就查到）**。发起人拿到 PENDING 先走了；回头查自己的事件或完成端口——信号早就到了（事件已 signal、端口已入队），直接取结果。用户态的 overlapped 契约本来就是这样：拿到 PENDING 就去查事件，事件可能早就 signal 了——这是日常，不是异常。发起人从来不是从信使手里拿结果，所以不存在“拿到结果又被告知要等”。
- **收束**。不存在“已经拿到结果、又收到等一等”——因为发起人拿结果的唯一途径是铃 / 结果条 / 端口，而那些东西只会说“到了”，永远不会说“还要等”。信使的话只指路，不报结果；指路晚到，只说明“你去异步渠道查”，而异步渠道里的结果已经在了。语义自洽的根源就一句：**PENDING 是渠道声明，不是时间承诺。**

### 故事版：结果信号先到、PENDING 后到——发起人会不会已经不在了（2026-10-28）

结论先说：**不可能出现“发起人拿了结果走了、PENDING 到了却没人接”。** 分两层讲：

- **第一层：“拿结果”这个动作，在程序顺序上一定在收到 PENDING 之后。** 先到的只是结果的信号（铃响、端口入队、结果条留下），信号是状态，留着等；拿取是动作，发起人收到 PENDING 后才做。同步：发起人坐在调用里 → 收到 PENDING（函数返回）→ 再等事件 → 事件已响 → 读结果单，顺序锁死，先回话、后拿取。异步：发起人回到用户态拿到 PENDING → 再查事件/端口 → 已到 → 取结果，同样锁死。所以“先把结果拿走了”在单个请求上不存在——拿取动作不可能跑到 PENDING 前面去；能跑到前面的只有信号，而信号不会跑丢。
- **第二层：PENDING 的收件人不可能“不在”。** PENDING 不是寄到家里的信（人走了信就没人收），它是发起人站在柜台前等店员的那句回话——店员没回话之前，发起人就站在那。为什么走不掉：发起人正在分发大楼里走路（执行分发 unwind 的内核代码），线程退出要靠 APC 投递，而 APC 只能在线程停下来（等待、回用户态）时投递——走路时投不进来。所以 unwind 一定先跑完，PENDING 一定先送达，线程然后才能退出。
- **真正的“发起人走了”只发生在 PENDING 送达之后**。同步是等完事件拿结果再走；异步是回用户态后爱查不查——不查也只是用户态的事件/端口项被 abandon，内核侧不乱。如果发起人走了、IRP 还没完成，有专门的善后：`IopDisassociateThreadIrp`（`IO/IOMGR/INTERNAL.C:1689`）：把 IRP 从死去线程的链表上摘下来，线程指针置 NULL；以后驱动完成它时，完成路径发现没有关联线程，就把结果“扔地上”（注释原话“It will essentially drop the I/O on the floor”）：不投递、不唤醒、只释放。驱动照样只完成一次，不会对着虚空送结果，也不会有人读已释放的内存。
- **完整的时序只有三种，都是自洽的**。① PENDING 先到、结果后到：发起人按 PENDING 指路去等铃/查端口，结果到了取走（正常异步）。② 结果信号先到、PENDING 后到：发起人收到 PENDING（指路）→ 一查，铃已响/端口已有项 → 直接取走（今天讨论的情形）。③ 发起人走了、结果后到：IRP 已被摘除关联线程 → 完成时结果扔地上、只释放（线程退出情形）。不存在第四种“发起人拿了结果走了、PENDING 到了没人接”——因为拿取动作在 PENDING 之后，而 PENDING 的收件人（柜台前的人）在回话到达前走不掉。

### 故事版：被扣住的不是信使，是善后组——什么时候放、怎么放（2026-10-28）

先纠正一个前面故事埋下的不精确：**被扣住的不是信使（PENDING），而是善后组（完成走访）。** 信使（PENDING 那句话）扣不住：话要么说了（分发例程返回了），要么没说（分发例程坐下等着，还没返回）。返回值没有“说一半、以后再接着说”这个操作。真正能扣住的，是批文本体往回送的队伍。

- **扣是怎么扣的**。善后组走到某层，打回执电话，电话那头说“先扣住，别往上传了”（返回 `STATUS_MORE_PROCESSING_REQUIRED`）→ 走访当场停下，不再往上、不做最终收尾，批文留在那层科室手里，归它所有。函数头注释原话：例程要“perform work a lot of work or the work needs to be performed in the context of another process”时就返回这个码，意思是“这个操作以后会由这个例程再完成一次”。截停时指针位置：走访循环初始化时指针先 +1（`IO/IOMGR/IOSUBS.C:3485` 一带，`CurrentLocation++` / `CurrentStackLocation++` 发生在第一联检查之前），所以截停返回时，指针正好停在扣住的那一层——批文的“当前位置”就是扣住的科室那一联。
- **什么时候放：扣住的科室说了算**。典型剧本（同步等）：科室在分发里坐下等 → 善后组走到它这层 → 回执电话摇醒它（signal 事件）+ 扣住队伍 → 它醒来处理结果（读回执单、拷数据）→ 处理完，它亲手放行。没有固定时间表，放行条件就是“它处理完了”。
- **怎么放：就一句话——再调一次 `IoCompleteRequest`。** 没有别的仪式。队伍再出发时从被扣那一层继续往上：再调完成时，循环从扣住那一层的位置往上查，查到的是上一层留的回执——扣住那层的回执电话不会再打一次（它的那一联在第一次走访时已被清掉、指针已越过）。一路到顶、敲铃、留条、销毁，和正常走完一模一样。
- **为什么“坐下等”必须配“电话里扣住”**。如果回执电话不说“扣住”，善后组直接走到顶、最终收尾，科室醒来时批文可能已被文书销毁——醒来没东西可读。所以“坐下等”一定配“电话里扣住”，“醒来处理完”一定配“再喊一次出发”，成对出现，缺一不可。
- **还有一种更常见的“扣”：批文本来就没办完**。科室接了批文、排进自己待办夹、回一句 PENDING，什么时候车间办完、什么时候善后组才出发。这不是扣信使，是活还没干完；干完了，科室自己喊 `IoCompleteRequest`，善后组才上路。PENDING 在这里的意思仍是“结果不走我这条路”，而善后组出发的时间，永远由握着批文的科室定。
- **收束**。信使扣不住，也不用扣——话说了就传上去了，没说就是分发例程还在等。能扣、值得扣的只有善后组：电话里一句“扣住”，批文就留在那层；处理完了再调一次完成，队伍就从那层继续往上。放行的钥匙始终在扣住的科室手里，不在信使手里。

### 故事版：驱动为什么要自己新造 IRP 往下发（2026-10-28）

转发成立的前提只有一个：**下面要干的活，和上面要的活，是同一件活**（同操作、同栈、同缓冲区、同完成语义）。只要有一项不一样，就必须新造。分场景讲：

- **一件活变多件活（扇出）**。上面要读 1MB，下面一台盘一次只能吃 64KB——一联变不出多联；RAID 条带：一次逻辑读要拆成 N 次物理盘读；镜像：一次写要写两份。驱动自己当发起人，造 N 份子批文，每份发往一台盘；原批文在上面等着汇总。树里现成的机制就是主从关联：`IoMakeAssociatedIrp`（`IO/IOMGR/IOSUBS.C:8976`，注释原话“allocates an IRP from the system nonpaged pool and makes it an associated IRP to the specified IRP”，调用前主批文先定好计数 `AssociatedIrp.IrpCount`），每份子批文办完计数减一，减到零主批文才完成（走访尾部 `IRP_ASSOCIATED_IRP` 分支）。
- **根本没有原批文（自发活）**。驱动自己要办事：启动时读分区表、挂载时读卷元数据、后台刷脏页、预读、定期巡检。没有上家，就没有批文可转发，只能自己裁一份。连 I/O 管理器自己都这么干：关机时给每台设备发 SHUTDOWN，树里就是现造的（`IOSUBS.C:11274` 一带 `IoBuildSynchronousFsdRequest(IRP_MJ_SHUTDOWN, ...)`）——关机又不是谁递来的请求，哪有原批文。
- **活变了（不同操作）**。上面要“读”，下面要先“问一句话”（IOCTL 查能力、SCSI INQUIRY）——原批文写的是“读”，不能改成“问”，改了原批文就毁了（原请求还等着读的结果）。只能另裁一份“问”的批文（`IoBuildDeviceIoControlRequest`），问完再回来办原批文。
- **路变了（不同设备栈）**。原批文是按这个栈的层数裁的（StackSize 定死），要送到另一个栈（卷管理器送往物理盘、过滤驱动送往另一台设备）——尺寸不对、指针也不对，塞不进去。只能按目标栈的层数新裁一份。
- **缓冲区 / 线程变了**。原批文的缓冲区是请求人的、线程是请求人的；驱动要在自己的缓冲区里读元数据、或在系统工作线程里做后台活——原批文绑着请求人的线程和事件，拿它去干后台活，完成后会往请求人那投递（APC/事件），乱套。只能新裁一份绑自己线程、自己缓冲区的。
- **分发里要同步问话**。驱动在分发里想同步问下层一句话（发 IOCTL 并等）——不能等原批文（原批文就是它正在办的、等它等于等自己，死锁）。只能另裁一份子批文、挂自己的事件、等子批文办完，再继续办原批文。这是“分发里同步等”唯一合法的形态。
- **重试 / 换路**。下层报“此路不通”（多路径 MPIO 故障切换），驱动要换一条物理路径重发——原批文已经走完这条路了（位置指针、状态都已定），换路要么复位重填（`IoReuseIrp`，同栈重发），要么新裁一份发往另一台设备。
- **收束与工具**。转发 = “这件活原样往下送”；新造 = “下面要干的不是这件活、或不只这件活、或根本没人下令”。判断标准就一条：下面要干的活和上面要的活是不是同一件——是，就转发；有任何一项不是，就新造。裁批文的工具：`IoAllocateIrp`（原始裁）、`IoBuildSynchronousFsdRequest` / `IoBuildAsynchronousFsdRequest`（读写专用裁）、`IoBuildDeviceIoControlRequest`（问话专用裁）、`IoMakeAssociatedIrp`（主从关联裁）。驱动新造时，它自己就是子批文的发起人：子批文的完成例程是它设的，子批文办完由它收尾（计数、汇总、释放），原批文的走访完全不知道子批文的存在——两层批文，各走各的路，只在驱动手里汇合。
