---
layout: post
title: "WRK 第 0 批 · 加载链地图与源码树导航"
categories: WRK
---

> 底本：本地 WRK V1.2 源码（WS03 SP1 代码线，NT 5.2），根目录 `~/workspace/WRK/WRK/WRK-V1.2/BASE/NTOS/`。
> 结构定义出处：`WRK-V1.2/PUBLIC/INTERNAL/BASE/INC/ARC.H`（LOADER_PARAMETER_BLOCK）、`WRK-V1.2/PUBLIC/SDK/INC/NTLDR.H`（KLDR_DATA_TABLE_ENTRY），字段名均为原文转录。

## 本批地图（在点火链中的位置）

这是整条 WRK 学习线的"开场白"批，不读任何子系统的实现，只回答三个问题：

1. 开机时控制权怎样一段段交到内核手里，每段在 WRK 里**有没有源码**；
2. osloader 交给内核的那只"行李箱"——LOADER_PARAMETER_BLOCK——里面到底装了什么；
3. 这棵源码树长什么样、几十万行代码是怎样被拼成一个 ntoskrnl.exe 的（理解了构建聚合，才能理解后面每一批里"子系统之间为什么敢直接互访全局变量"）。

后续批次与本批的关系：第 1 批从 KiSystemStartup 进入 KE/KIINIT.C；第 2 批展开 INIT/INITOS.C 的 ExpInitializeExecutive 点火表；第 3–8 批逐个展开点火表里的子系统。

## 加载链全景

| 段 | 发生了什么 | WRK 内有无源码 |
|---|---|---|
| 固件（BIOS/EFI） | 自检、按启动顺序读入第一扇区/启动管理器 | 无（硬件固件，不在包内） |
| ntldr（osloader，WS03） | 读 boot.ini（bootmgr / BCD 是 Vista 起的后继启动器，WS03 代码线不用）、把 ntoskrnl.exe、hal.dll、bootvid.dll 与 boot-start 驱动映像逐个装入内存、建 LOADER_PARAMETER_BLOCK、给每个映像建一条 KLDR_DATA_TABLE_ENTRY，然后跳到内核入口点 | **无**——加载器本体不在 WRK 内；但它留下的**数据结构**在 PUBLIC 头文件里（ARC.H、NTLDR.H），本批读的就是这部分 |
| KiSystemStartup | 内核的 PE 入口点（链接参数 `-entry:KiSystemStartup`，见 NTOS/BUILD/MAKEFILE，i386 为 `KiSystemStartup@4`）。架构相关汇编：i386 在 `KE/I386/NEWSYSBG.ASM`，AMD64 在 `KE/AMD64/START.ASM`。第一件事：把 LoaderBlock 指针存进全局变量 `KeLoaderBlock`（KE/KERNLDAT.C 定义），再调 HalInitializeProcessor、KiInitializeKernel | **有** |
| KiInitializeKernel → ExpInitializeExecutive | KE 做处理器级初始化（PRCB、调度器骨架），随后 INIT/INITOS.C 的 `ExpInitializeExecutive(Number, LoaderBlock)` 接棒，按点火表逐个初始化各子系统（Phase 0 → 创建 Phase 1 线程 → Phase 1） | **有**（细节留第 1、2 批） |
| Phase 1 完成 → SMSS | 执行体创建第一个用户进程 smss.exe，交棒用户世界 | 内核侧有（PS/MM 的进程创建路径，第 8 批）；smss 本体无 |

**要点**：WRK 的覆盖范围从 KiSystemStartup 开始。加载链前半段（固件、加载器）只能通过它交接的数据结构来"反读"——这也是内存取证里还原启动过程的基本方法。

## Loader Block 解剖

### LOADER_PARAMETER_BLOCK（ARC.H 原文）

```c
typedef struct _LOADER_PARAMETER_BLOCK {
    LIST_ENTRY LoadOrderListHead;          // 已加载映像链表（KLDR 条目串）
    LIST_ENTRY MemoryDescriptorListHead;   // 固件内存图（MEMORY_DESCRIPTOR 串）
    LIST_ENTRY BootDriverListHead;         // boot 驱动条目（IoInitSystem 按它加载启动驱动，第 7 批）
    ULONG_PTR KernelStack;
    ULONG_PTR Prcb;
    ULONG_PTR Process;
    ULONG_PTR Thread;
    ULONG RegistryLength;
    PVOID RegistryBase;                    // 已装入内存的 SYSTEM hive 映像
    PCONFIGURATION_COMPONENT_DATA ConfigurationRoot;  // ARC 配置树
    PCHAR ArcBootDeviceName;
    PCHAR ArcHalDeviceName;
    PCHAR NtBootPathName;
    PCHAR NtHalPathName;
    PCHAR LoadOptions;                     // /DEBUG、/MAXMEM 等启动开关原文
    PNLS_DATA_BLOCK NlsData;               // ANSI/OEM/Unicode NLS 表
    PARC_DISK_INFORMATION ArcDiskInformation;
    PVOID OemFontFile;                     // 蓝屏/启动画面用的 OEM 字体
    struct _SETUP_LOADER_BLOCK *SetupLoaderBlock;
    PLOADER_PARAMETER_EXTENSION Extension; // 扩展块（见下）
    union { I386_LOADER_BLOCK I386; /* Alpha/IA64 已注释掉 */ } u;
} LOADER_PARAMETER_BLOCK, *PLOADER_PARAMETER_BLOCK;
```

按用途分五组看：

1. **映像清单**：`LoadOrderListHead`——加载顺序链表，每个已装入映像（内核、HAL、每个 boot 驱动）一条 KLDR_DATA_TABLE_ENTRY。这是后面 PsLoadedModuleList 的"胚胎"。
2. **内存图**：`MemoryDescriptorListHead`——固件视角的内存描述符链，每条 `MEMORY_DESCRIPTOR { MEMORY_TYPE MemoryType; ULONG BasePage; ULONG PageCount; }`，MemoryType 是枚举（MemoryFree、MemoryLoadedProgram、MemoryFirmwarePermanent、MemoryBad、MemorySpecialMemory……）。第 3 批 MM 点火时，就是按这张图建 PFN 数据库：哪些页能用、哪些已被占用，全靠它。
3. **"半成品"内核现场**：`KernelStack / Prcb / Process / Thread`——加载器在跳进内核之前，已经替内核把初始栈、boot 处理器的 PRCB、初始进程与线程对象的地址都备好放在这里。AMD64 的 START.ASM 里能看到它先读 `LpbPrcb[rcx]`（为空则用内建 KiInitialPCR 的 PRCB）。也就是说内核不是"从零"启动的，它的执行上下文是加载器搭好的。
4. **配置与文件原料**：`RegistryBase + RegistryLength`（SYSTEM hive 已被加载器整段读入内存，CM 第 6 批点火时接管它——这解释了为什么注册表在文件系统就绪之前就能用（加载器读 hive 靠它自带的极简只读文件系统代码，见第 2 批 QA 第 19 条，不是正式文件系统驱动））；`ConfigurationRoot` 是 ARC 固件配置树（硬件拓扑的树形描述，PnP 的祖先数据）；`ArcDiskInformation` 给磁盘子系统起步用。
5. **字符串与杂项**：四个 ARC/NT 路径名（ArcBootDeviceName、NtBootPathName 等——ARC 命名就是 `\Device\Harddisk0\Partition1` 这套的祖先）、`LoadOptions`（启动开关原文，后面 INITOS.C 里逐字解析 /BURNMEMORY 等选项的源头）、`NlsData`（字符集表，蓝屏打印中文/DBCS 的前提）、`OemFontFile`。

**Extension 块（LOADER_PARAMETER_EXTENSION）** 是后加的"行李箱夹层"，带 `Size + MajorVersion/MinorVersion` 自描述版本头——ExpInitializeExecutive 进门第一件事就是 `ExpIsLoaderValid(LoaderBlock)` 校验它，对不上直接 `KeBugCheckEx(MISMATCHED_HAL, 3, ...)`（INITOS.C 实测）。夹层里装的是新时代的东西：Profile 参数（内核剖析参数块）、TriageDumpBlock（故障转储分诊）、InfFileImage（标识问题机器的 Inf 映像）、LoaderPagesSpanned（加载器占用的虚拟内存跨度，MM 不能回收其 VA 的部分）、DrvDBImage（"已知坏驱动"数据库）、HeadlessLoaderBlock（无头服务器）、SMBiosEPSHeader（SMBIOS 入口点结构头）、AcpiTable（内存中的覆盖版 ACPI 表）、NetworkLoaderBlock（PXE 网络启动的 DHCP/BINL 包原文），x86 下还有指向 HAL 内 IRQL 翻译表的两个指针。为什么带版本头：加载器与内核若对交接单格式理解不一致，后面按格式读字段全会错位，不如进门就校验、对不上立刻蓝屏（第 2 批第 2 步）。**设计信号很明确：主结构冻结、演进走扩展块**——这是 NT 兼容性哲学的又一例。

### KiSystemStartup 从箱子里取了什么（签名层速览，细节留第 1 批）

- i386（NEWSYSBG.ASM）：`KiSystemStartup` 是 stdcall、1 个参数，`LoaderBlock` 在 `[ebp+8]`；汇编里 `mov _KeLoaderBlock, ebx` 先存全局，随后调 `_HalInitializeProcessor`（参数含 LoaderBlock）与 `_KiInitializeKernel`（6 参数，含 LoaderBlock）。为什么签名不同：x86 用 stdcall（参数走栈），AMD64 用统一调用约定（前四个参数走寄存器）——这是架构调用约定的差异，不是内核设计的差异。
- AMD64（START.ASM）：`NESTED_ENTRY KiSystemStartup`，LoaderBlock 走第一个寄存器参数 RCX，注释写明 "Save the address of the loader block. N.B. This is the same address for all processors."——多核启动时每个 AP 都拿同一只箱子。为什么同一只箱子能给所有核：LoaderBlock 是启动期只读的交接数据，各核只读不写，不用每核一份。
- 全局句柄：`PLOADER_PARAMETER_BLOCK KeLoaderBlock`（KE/KERNLDAT.C 定义、INC/KE.H 声明），整个初始化期各子系统都通过它回查箱子；甚至 KE/BUGCHECK.C 在蓝屏时还判 `if (KeLoaderBlock != NULL)` 回退用它的 LoadOrderListHead 枚举模块——箱子在启动后并不会立刻"消失"，取证价值见后文。

## 构建体系：几十万行怎样拼成一个内核

先纠正一个容易踩的直觉：`INIT/NTOSKRNL.C` 与 `INIT/NTKRNLMP.C` **不是**把各子系统 `#include` 进来的聚合编译文件——打开看，NTOSKRNL.C 自我描述为 "Null program for the INIT subcomponent"（一个空桩），NTKRNLMP.C 全文只有一行 `#include "ntoskrnl.c"`。真正的聚合发生在**链接期**，分三层：

1. **按目录编译成库**：NTOS/MAKEFILE 列出模块清单 `rtl config ex ob se mm ke ps io\iomgr io cache lpc dbgk raw fstub fsrtl wmi perf init`；每个目录有自己的 BUILD/MAKEFILE 把本目录 .C/.ASM 编成一个 .lib（如 INIT 的 BUILD/MAKEFILE：`ccobjs = initos.obj`，外加把 ntkrnlmp.obj 这个空桩拷到顶层 obj 目录当"链接锚"）。
2. **生成导出表**：NTOS/BUILD/MAKEFILE 的 `kernelexp` 目标把 `INIT/NTOSKRNL.SRC` 与架构文件 `I386DEF.SRC`/`AMD64DEF.SRC` 拼接、预处理成 .def——所以 .SRC 文件是**链接器导出定义**（几千行 `CcCopyRead`、`ExAllocatePoolWithTag`……），不是源码；架构 .SRC 追加该架构特有的导出（如 i386 的 Exi386Interlocked* 系列）。
3. **一次链接成映像**：`kernelexe` 目标：`ntkrnlmp.obj + *.lib（全部子系统）+ hal.lib + bootlibs`，参数含 `-entry:KiSystemStartup`、`-driver`、`-version:5.2`，并用 `-merge:` 把 PAGECONST/INITCONST 等段合并、把 **INIT 段标为可丢弃**（`-SECTION:INIT,d`）——这就是"INIT 段代码启动完就被回收"在构建层的出处：ExpInitializeExecutive 等函数头上的 `#pragma alloc_text(INIT, ...)` 把它们放进这个用完即扔的段。

由此得到两个对读源码至关重要的结论：

- **单映像、单链接**：所有子系统最终是同一个 PE 里的对象文件集合，没有 DLL 边界、没有导出检查（导出表只服务于驱动在运行时按名解析）。所以源码里 KE 可以直接读写 MM 的全局、EX 的 POOL.C 直接被所有人调用——**内核内部的"接口"只是头文件约定，不是链接强制**。逆向时看到 ntoskrnl 内部跨子系统直接互访，不要用用户态 DLL 的思维去找"边界"。
- **UP/MP 之别不在这两个 .C**：WRK 里 NTOSKRNL.C/NTKRNLMP.C 都是同一个空桩；单核/多核版本的差异来自编译期宏定义对各目录源码的条件编译，而非聚合文件不同。（WRK 交付的构建脚本实际只链接出一个 fullkernel 映像。）

## 源码树地图（NTOS 目录实测）

统计口径：find/wc 实测。NTOS 全树 find 全文件 675 个；按 .C/.H/.ASM/.INC 计 627 个（下表即此口径，表内文件数与行数已逐目录复核、与实测一致）；.C/.H 行数合计 699,729（约 70 万行）。原"675 个文件"的口径表述有误（675 是全文件数，不是 .C/.H/.ASM/.INC 数），已纠。

| 目录 | 职责 | 文件 | 行数 | 备注 |
|---|---|---|---|---|
| MM | 内存管理 | 59 | 164,438 | 全树最大，占近 1/4；第 3 批主场 |
| CONFIG | 配置管理器（注册表，Cm） | 55 | 71,706 | 第 6 批主场 |
| KE | 内核层（调度、同步、陷阱） | 122 | 57,202 | 文件数第一（含 I386/AMD64 架构子目录）；第 1、5 批 |
| RTL | 运行时库 | 74 | 54,399 | 字符串/位图/堆等基础件 |
| IO | I/O 管理器与 PnP | 34 | 52,074 | 含 IOMGR 子目录；第 7 批主场 |
| INC | 全局内部头文件 | 44 | 49,471 | KE.H/MM.H/PS.H 等"子系统宪法"在此 |
| SE | 安全引用监视器 | 34 | 41,642 | 令牌、ACL、审计 |
| EX | 执行体支持层 | 38 | 38,165 | 池、ERESOURCE、回调、工作线程 |
| WMI | WMI/ETW 内核侧 | 26 | 37,977 | |
| FSRTL | 文件系统运行时 | 17 | 29,265 | |
| PS | 进程/线程管理 | 23 | 26,277 | 第 5、8 批主场 |
| OB | 对象管理器 | 17 | 26,148 | 第 4 批主场 |
| CACHE | 缓存管理器（Cc） | 11 | 23,427 | |
| LPC | 本地过程调用 | 14 | 9,322 | ALPC 的前身 |
| FSTUB/RAW/PERF/DBGK | 杂项支撑 | 4/13/5/4 | 4,585/2,871/1,415/3,425 | FSTUB：文件系统桩；RAW：裸设备文件操作；PERF：性能数据；DBGK：用户态调试内核侧 |
| INIT | 初始化代码 | 3 | 2,133 | 第 1、2 批主场（INITOS.C 点火总控、NTOSKRNL.C 空桩） |
| VERIFIER | 驱动验证器内核侧 | 28 | 3,711 | 检查驱动非法操作（死锁、错误 IRQL 调用等） |
| VDM | 虚拟 DOS 机（x86） | 1 | 0 | 仅 I386 子目录，无 C/H 代码；VDM 定义见第 2 批术语表 |
| BUILD | 构建支撑 | 1 | 76 | 顶层构建文件（MAKEFILE、WARNING.H） |

---

## LoaderBlock 的取证价值（补 2026-10-25 实查，接"取证价值见后文"）

启动期（Phase 1 的 LoaderBlock 释放之前），LoaderBlock 是"加载器到底装了什么"的唯一完整清单：LoadOrderListHead（映像清单，可与事后 PsLoadedModuleList 对照找摘链）、MemoryDescriptorListHead（固件内存图）、RegistryBase（SYSTEM hive 映像）、LoadOptions（启动开关原文）。蓝屏时 BUGCHECK.C 回退用它枚举模块（`if (KeLoaderBlock != NULL)`）。

窗口到 Phase 1 的 `INITOS.C:1615-1621` 为止：内核先判 `LoaderBlock == KeLoaderBlock` 成立则把全局 `KeLoaderBlock` 置 NULL（注释原话："Force KeBugCheck to look at PsLoadedModuleList now that it is setup"——PsLoadedModuleList 已建好，蓝屏回退枚举从此改查它，不再需要 LoaderBlock），再调 `MmFreeLoaderBlock` 释放整块加载器内存。此后 LoaderBlock 指针已空、页面归还——事后转储里不能再拿它当清单读；若转储里还残留 LoaderBlock 内容，只能当作启动期快照的残留对待，且须先验证页面未被改写。

## 补漏术语（2026-10-25 实查补，凡正文出现而未定义的）

- **PE（Portable Executable）**：Windows 可执行文件格式（.exe、.dll、.sys 都是它）。KiSystemStartup 是 ntoskrnl.exe 的 PE 入口点。
- **KLDR_DATA_TABLE_ENTRY**：已加载映像的链表节点（映像基址 DllBase、大小、名字），LoadOrderListHead 串起它们；结构见 NTLDR.H，字段见第 2 批术语表。
- **KiInitialPCR**：内核映像里内建的初始 PCR（P0 在自己的 PCR 立好之前先用它）。
- **boot.ini**：WS03 的启动菜单配置文件，写着 ARC 路径，告诉加载器去哪个分区的哪个目录找 Windows。
- **BCD（Boot Configuration Data）**：Vista 起的启动配置数据库，bootmgr 的配置来源；WS03 代码线不用。
- **ARC（Advanced RISC Computing）**：固件与加载器之间的启动约定（设备命名、配置树、加载器参数块格式），不是某个硬件品牌。
- **SetupLoaderBlock**：安装加载器专用块（Setup 时用）；本体定义不在 ARC.H 内，此处只有前向声明（ARC.H:326）。
- **I386_LOADER_BLOCK**：LoaderBlock 联合体里的 x86 专属部分：CommonDataArea（公共数据区）、MachineType（临时机型字段，注释原话 "Temporary only"）、VirtualBias（加载器虚拟地址偏移）。NT 历史上支持过 Alpha、IA64 等架构，WS03 代码线里这些分支已注释停用，只剩 x86/AMD64。
- **SMBIOS（System Management BIOS）**：系统管理 BIOS，固件提供的硬件信息表；SMBiosEPSHeader 是它的入口点结构头。
- **ACPI（Advanced Configuration and Power Interface）**：高级配置与电源接口，固件提供的硬件与电源描述表；AcpiTable 是内存中的覆盖版 ACPI 表（覆盖固件原表用）。
- **PXE（Preboot Execution Environment）**：网卡预启动执行环境，从网络启动；**DHCP（动态主机配置协议）**：自动分配 IP；**BINL（启动信息协商层）**：PXE 网络启动时向服务器查询启动信息的协议层。NetworkLoaderBlock 里存的就是这套网络启动参数原文。
- **OEM（Original Equipment Manufacturer）**：原始设备制造商；OemFontFile 是厂商定制的蓝屏/启动画面字体。
- **DBCS（Double-Byte Character Set）**：双字节字符集（如中文）；NlsData 里的字符集表是蓝屏打印中文/DBCS 的前提。
- **PAE（Physical Address Extension）**：物理地址扩展，32 位 CPU 访问 4GB 以上内存的机制；ntkrnlpa / ntkrpamp 的 "pa" 即它。
- **ETW（Event Tracing for Windows）**：Windows 事件追踪机制；WMI 行的 "WMI/ETW 内核侧"指内核里的追踪支撑。
- **ACL（Access Control List）**：访问控制表，挂在安全描述符里规定谁能对对象做什么（DACL 自主、SACL 审计）。
- **Profile 参数块（PROFILE_PARAMETER_BLOCK）**：内核剖析（profiling）参数块，Extension 里给性能分析用的。
- **sdbapint.lib**：应用程序兼容性（垫片数据库）的接口库，链接进内核供垫片机制调用。
- **LINKEDIT**：链接后编辑工具，改 PE 段属性；这里用它去掉 .rsrc 段的可丢弃标志（蓝屏字符串必须常驻）。
- **RVA（Relative Virtual Address）**：相对虚拟地址，即相对映像基址的偏移；链接器定段 RVA 就是定每段在映像里的相对位置。
- **PAGECONST / INITCONST**：分页常量段 / 初始化期常量段，链接时用 -merge 并入 PAGE / INIT 段。
- **-version:5.2**：PE 版本号 5.2，即 Windows Server 2003（NT 5.2）。
- **-subsystem:native**：原生子系统——不依赖 Win32、直接跑在内核上的程序（驱动、SMSS 早期等）的子系统类型。
- **热补丁（Hotpatching）**：函数入口预留补丁桩，运行时替换函数实现而不重启；-stub:stub512.com 就是映像头预置的 512 字节热补丁桩区。
- **VDM（虚拟 DOS 机）**：跑 16 位 DOS 程序的兼容机制，定义见第 2 批术语表；VERIFIER 目录是驱动验证器内核侧。
- **HAL（硬件抽象层）**：定义见第 1、2 批术语表；本批的 hal.dll / hal.lib 即它。

## 疑问与解答（QA 区）

> 本区收录用户在读本批时提出的问题与逐项核查后的解答（2026-10-24 起持续追加）。

### Q1：NTOSKRNL.C 与 NTKRNLMP.C 只是空桩、聚合在链接期，那这两个文件为什么要弄成这样？（2026-10-24）

**证据**：NTOSKRNL.C 去掉版权头只剩自述一句——"Null program for the INIT subcomponent of the NTOS project"；NTKRNLMP.C 全文一行 `#include "ntoskrnl.c"`。INIT 目录的 makefile 把后者编成名为 `nullobj = ntkrnlmp.obj` 的对象，xcopy 到顶层 obj 目录；顶层链接行（BUILD/MAKEFILE kernelexe 规则）以 `$(LINK) … ntkrnlmp.obj *.lib …` 链接，ntkrnlmp.obj 是第一个代码对象。

**解答（三层）**：
1. **链接需要一个"锚"，但内核没有主源文件。** ntoskrnl 的全部代码住在 19 个模块目录各自编成的 .lib 里（入口 KiSystemStartup 在 KE 库、点火总控在 INIT 库），不存在一个含 main 的 .c。而链接命令与按目录构建的 BUILD.EXE 体系要求：映像目标必须挂在某个目录产出的对象上。挑任何真实对象当锚都会扰乱它与库的符号关系，于是专门造一个零符号的空对象当锚——只占链接行首位、给映像目标一个构建图上的归属。变量名 nullobj 表明这是自觉的占位设计。
2. **名与实分离，所以有两个文件、且用 #include 而非复制。** 对象文件名由源文件名决定：顶层点名 `ntkrnlmp.obj`，就必须有 NTKRNLMP.C。正文两处毫无区别（皆空），故 NTKRNLMP.C 一行 include 把 NTOSKRNL.C 的正文引为单一出处，文件名单独成立。
3. **"mp"是映像变体名。** 零售版同出 ntoskrnl（UP）/ntkrnlmp（MP）/ntkrnlpa（PAE）/ntkrpamp（PAE+MP）四个变体，同一套库链接而成，各有命名的锚对象；WRK 只接线 MP 变体，故只剩 ntkrnlmp 一个。NTOSKRNL.C 是变体家族的原型空程序。

**一句话**：这两文件不是代码、是构建系统适配器——真实结构是"模块库链接成单映像"，工具心智是"映像须从目录对象长出来"，空桩让前者迁就后者而不迫使真实代码搬家。其推论在逆向中可验证：ntoskrnl 里找不到"入口附近的主干代码"，KiSystemStartup 直接跳进各子系统初始化。

### Q2：链接行最前面的 ntoskrnl.res 是什么？是链接器吗？（2026-10-24）

**不是链接器。** 链接器是该 makefile 规则行首的 `$(LINK)`（即 link.exe），连同 `-entry:KiSystemStartup`、`-version:5.2`、`-subsystem:native`、`-driver`、`-SECTION:INIT,d` 等参数出现；链接行里跟在参数后面的全是**输入材料**，按序为：

| 输入 | 是什么 |
|---|---|
| `$(hotpatch)` | i386 为 `-stub:PREBUILT\i386\stub512.com`（映像头预置 512 字节热补丁桩区）；AMD64 为 `hotpatch.obj` |
| `PREBUILT\$(targ)\ntoskrnl.res` | **编译好的资源文件**（见下） |
| `$(OBJ)\ntkrnlmp.obj` | Q1 的空锚对象 |
| `$(OBJ)\*.lib` + `$(ntoswrklib)` | 19 个模块库 + WRK 支撑库——真实代码 |
| `hal.lib` | PREBUILT 里的 HAL 导入/静态库 |
| `$(fullkernel).exp` | 由 .def 经 LIB.EXE 生成的导出文件：使最终映像的导出表列全内核导出函数（其他驱动要按名/序链接进 ntoskrnl） |
| `$(bootlibs)` | bootvid.lib（启动画面/蓝屏的 Inbv 显示输出）、kdcom.lib（串口 KD 传输）、sdbapint.lib |

**ntoskrnl.res 专答**：.res 是资源编译器（rc.exe）把 .rc 资源脚本编译成的二进制资源块，链接时整块塞进映像的 `.rsrc` 段。对 ntoskrnl 而言它装的不是图标，而是（对 PREBUILT/I386/NTOSKRNL.RES 实测 `strings` 可见）**蓝屏 bugcheck 的消息表**——APC_INDEX_MISMATCH、IRQL_NOT_LESS_OR_EQUAL、SPIN_LOCK_ALREADY_OWNED 等全部 bugcheck 符号名/文本，外加版本信息资源。两个佐证：①该 .res 在 WRK 里是按架构预置的成品（PREBUILT 目录），WRK 不再现场编译它；②链接后紧跟一条 `$(LINKEDIT) -section:.rsrc,!d`，专门把 .rsrc 段的"可丢弃"标志去掉——道理很实在：蓝屏是最不能缺页失败的时刻，bugcheck 字符串必须常驻、不可被判为可丢弃。


### Q3：obj 文件是什么、为什么要有？lib 和 DLL 是一回事吗？obj+lib 生成 EXE 的内部机制如何？"映像必须挂在某目录产出的对象上"是 NT 独有要求还是普遍必须？（2026-10-24）

**obj 是编译的半成品，不是可有可无的中间垃圾。** 编译器一次只吃一个源文件（一个"翻译单元"）：把 INITOS.C 编成 initos.obj 时，它只看得见这个文件——里面调用的 Ke/Mm/Ob 函数在别的文件里，它不知道地址。于是 obj 里装四样东西：①已译好的机器码（按段摆放：.text、INIT、PAGE 等）；②**符号表**：本文件定义了哪些符号（ExpInitializeExecutive 等，给出）、引用了哪些外部符号（KeLoaderBlock 等，欠着）；③**重定位表**：凡用到外部符号地址的地方先留空位，记一条"此处待链接器按符号 X 的最终地址回填"；④段属性（可分页、可丢弃等，`#pragma alloc_text(INIT,…)` 就落在这里）。

**为什么必须有这一层**：分开编译带来四件事——增量（改一行只重编一个文件）、并行（19 个目录同时编）、混编（C 与汇编各自成 obj 再合流）、复用（同一批 obj 既能进这个映像、也能打包给别人）。没有 obj 中间层，70 万行就只能整锅重编。

**lib 不是 DLL，两者差在"何时合体"。** lib 是**一捆 obj 加一份符号索引**（LIB.EXE 打的包）：代码还没进你的映像，链接时按需抽取。DLL 是**已经链接完成的 PE 映像**：运行时才被加载器装进来、按导出表对接，独立存在、独立版本。还有第三种易混的 **导入库（import lib）**：里面没有真代码，只有某 DLL 的导出清单，链接器靠它在你的映像里建好导入表，运行时由加载器去找那个 DLL 对接。WRK 这条链接行上两种 lib 都有：19 个模块的 `*.lib` 是真代码包（静态库，代码最终长进内核映像——因为根本没有另一个映像装它们）；`hal.lib`、`bootvid.lib`、`kdcom.lib` 则是导入库——HAL、BOOTVID、KDCOM 本身是独立映像，由 osloader 与内核并排装入（第 0 批的 LoadOrderList 里各有一条 KLDR 记录为证），内核只是通过导入表调用它们。

**obj+lib → EXE 的内部机制（链接器五步）**：①符号解析：从命令行上的 obj 开始，凡欠着的符号就去各 lib 的索引里查，找到就把那个成员 obj 抽进来（它可能又欠新符号，继续抽，直到闭合）——这就是"按需抽取"，也是 lib 顺序有时要紧、循环依赖难缠的原因；WRK 的解法是 kernelexp 规则先用 LIB.EXE 把全部 *.lib 合并成一个 fullkernel.lib，循环依赖在一个大包内部自解。②段合并：所有 obj 的同名段拼到一起（ntosmerge 参数还在拼之上再合并：INITCONST、INITDATA 并入 INIT 等），按 -SECTION:INIT,d 之类规则定属性。③地址布局：给每个段定最终 RVA，PE 头成形。④重定位回填：按各 obj 的重定位表，把留空的地址坑逐个填上符号的最终地址。⑤建表收尾：写导入表（对 HAL 等 DLL）、导出表——这里正是 `$(fullkernel).exp` 的作用：它由 .def 生成，**把全部导出函数"钉"为根**，否则 -OPT:REF 会把没人引用的函数当死代码删掉，而内核导出函数恰恰大多没人（映像内部）引用、是给外部驱动用的。最后 -entry 指定 KiSystemStartup 为入口，PE 落盘。

**"必须挂在目录产出的 obj 上"是谁的要求？** 分两半答：
- **链接器物理上并不需要这个空 obj**：证据就是它空无一物、零符号——链接器从它那里什么也得不到，去掉它、链接照样能成（link.exe 甚至有 /NOENTRY 只链 .res 出资源 DLL 的玩法）。所以这不是链接的自然法则。
- **它是构建系统（BUILD.EXE 的目录模型）的要求**：该体系里每个目录声明自己的产物，顶层靠"谁产出了什么文件"建依赖图、定构建顺序、判增量重编。映像这个目标要在这张图里有户口，就得有某个目录产出一个文件代表它；INIT 作为内核"主场"目录（点火代码在此），就用 nullobj 充当这个代表，并被 xcopy 送到顶层 obj 目录成为链接的具名输入。不这么做的代价：内核要在构建图里处处开特例——顶层规则没有具名先决文件可挂、INIT 目录对映像"无贡献"的假象会让增量与并行调度出错，且全 Windows 数百个组件共用同一套"目录产出 obj/lib → 顶层链接"的统一约定会被打破。**好处即由此而来**：增量（改一个子系统只重编它+一次链接）、并行（19 目录同时编译）、变体矩阵（换个锚对象名+参数即出 UP/MP/PAE 变体）、约定统一（内核与普通驱动同构，工具链零特例）。空 obj 的本质是：**用一个零成本文件，把"链接成单映像的模块集合"适配进"按目录产出文件"的构建世界观**。

## 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，主文讲“做了什么”，这里讲“怎么过去的”。

- **固件 → ntldr：跳转到加载地址执行**。固件只会按扇区号读盘：读 MBR → 活动分区引导扇区 → 引导扇区里的代码把 ntldr 读进内存、跳转到它的入口点。转移方式是跳转（执行权不再回来），不是函数调用；固件代码本身不在 WRK 树内，此为架构级陈述。
- **实模式 → 保护模式 → 长模式：远跳是转移本身**。跳板链（关中断 → 开 A20 → 装 GDTR → 置 CR0.PE → 远跳转 → 重装数据段与栈 → 装 CR3 → 置 CR0.PG → 远跳 64 位代码段）里，两次远跳不是装饰：远跳重装 CS 的隐藏描述符缓存、清空流水线，CPU 从这条指令起按新模式解码。没有远跳，模式位改了、解码方式也不会换。
- **ntldr → KiSystemStartup：跳转 + 参数传递**。ntldr 跳进内核 PE 入口点，LoaderBlock 指针作为参数交过去，执行权不再回 ntldr。LoaderBlock 的消费不是执行转移，是指针传递：`ExpInitializeExecutive` 读它，直到 Phase 1 尾 `MmFreeLoaderBlock` 释放，指针失效。
- **SIPI 叫醒 AP：向量即入口地址**。SIPI 只做两件事：叫醒核、给启动页号（入口 = 向量 × 0x1000）。AP 从该地址的实模式跳板代码起跑，重走保护模式跳板——SIPI 本身不换模式，换模式的是跳板代码（第 1 批问答已详述，这里只点转移方式：核间中断 + 跳板跳转）。
