---
layout: post
title: "Win 内核驱动 第 9 批：DMA 与控制器对象"
categories: windows-kernel
series_order: 9
---

- **系列**：Windows 内核驱动循序渐进学习（教材：微软官方《内核模式驱动程序体系结构设计指南》，WDM 中心）
- **本批对应原文档**：「Methods for Accessing Data Buffers」章中与 DMA 直接相关的部分（Using Direct I/O with DMA / Using Direct I/O with PIO）+ 「DMA Programming Techniques」章（DMA Programming Techniques / Flushing Cached Data during DMA Operations / Splitting DMA Transfer Requests）+ 「DMA」章全节（Introduction to Adapter Objects / Getting an Adapter Object / Using Scatter/Gather DMA / Writing AdapterControl Routines / Storage Requirements for AdapterControl Routines / Setting Up AdapterControl Routines / AdapterControl Routine Requirements / Using Packet-Based System DMA / Allocating an Adapter Channel for Packet-Based DMA / Setting Up the System DMA Controller for Packet-Based DMA / Using Common-Buffer System DMA / Allocating an Adapter Channel for Common-Buffer System DMA / Setting Up the System DMA Controller for Common-Buffer DMA / Using Bus-Master DMA / Using Packet-Based Bus-Master DMA / Allocating the Bus-Master Adapter Object / Setting Up a Transfer Operation / Using Common-Buffer Bus-Master DMA / Version 3 of the DMA Operations Interface / Basic Calling Pattern for Version-3 DMA Routines / Using the MapTransferEx Routine）+ 「PIO Techniques」章（PIO Techniques / Flushing Cached Data during PIO Operations，仅作 DMA 的对照）+ 「Accessing Device Configuration Space」章全节（Accessing Device Configuration Space / Obtaining Device Configuration Information at IRQL = PASSIVE_LEVEL / Obtaining Device Configuration Information at IRQL = DISPATCH_LEVEL / Obtaining Configuration Information from Other Driver Stacks）+ 「Controller Objects」章全节（Introduction to Controller Objects / Creating Controller Objects and Controller Extensions / Allocating Controller Objects for I/O Operations / Writing ControllerControl Routines / Storage Requirements for ControllerControl Routines / Setting Up ControllerControl Routines / ControllerControl Routine Requirements）。
- **取材说明**：正文取自微软官方同名设计指南 PDF（2019 版离线打包）上述章节逐节研读。本 PDF 的来源仓库 tpn/pdfs 是第三方文档收集仓库、非微软官方；指南本身是微软官方文档 2019 版离线版。本批是第 8 批的下半程：第 8 批讲“设备怎么打断 CPU、打断之后怎么善后”，讲到 DPC 与 DMA/控制器交接的那一步为止；本批从交接点往深处走——适配器对象怎么代表 DMA 硬件、包 DMA 与公共缓冲 DMA 两条路、散点/聚集怎么省事、缓存为什么必须刷、控制器对象怎么串行化多设备。PnP 资源分配全流程属第 10 批，本批只讲 START_DEVICE 里拿适配器对象、以及配置空间访问与 PnP 的交接点，不越界。
- **学习目标**：学完本批，你应该能说清：① 三地址空间（虚拟/物理/逻辑）与映射寄存器到底是什么关系；② 适配器对象代表什么、IoGetDmaAdapter 交回来的两个东西（对象指针、映射寄存器数）各有什么用；③ 系统 DMA 与总线主控 DMA、包 DMA 与公共缓冲 DMA 这两对划分的判据；④ AdapterControl 例程的三返回值各在什么场景用、为什么 AdapterControl 一个 IRP 只调一次；⑤ 传输为什么必须拆（映射寄存器数、页边界、设备限制）以及四种拆法；⑥ 缓存不一致的两处（处理器缓存、控制器内缓冲）各用什么刷；⑦ Version 3 接口相对 v1/v2 简化了什么、为什么 v3 不用 KeFlushIoBuffers；⑧ 控制器对象为什么 WDM 不能用、KeepObject 与 DeallocateObject 的分界；⑨ 配置空间访问为什么 PASSIVE_LEVEL 用 IRP、DISPATCH_LEVEL 必须用总线接口。

---

## 术语表（按引用级排序）

表头说明：基础概念在前、派生概念在后；读到某条时它引用的词前面都已定义。互为前提的一对已按惯例处理：MDL 条给行内解释（它与映射寄存器条目一一对应）并指到后面的映射寄存器条目。第 6、7、8 批已定义的词（IRP、DPC、ISR、DIRQL、DISPATCH_LEVEL、PASSIVE_LEVEL、StartIo、设备扩展、SynchCritSection）与第 1、2、4 批已定义的词（非分页池、WDM、PDO、PnP）本批沿用同一含义，不再全文重写，只在首次出现时给一行定位。

- **虚拟地址**：CPU 指令访存用的地址，存在于某个进程或系统地址空间里。为什么要有：CPU 只能按虚拟地址取指、读写；没有它，程序无法以统一方式访问内存。
- **物理地址**：内存条上真实存储单元的地址。为什么要有：虚拟地址最终必须落到真实存储单元；页表就是干这个翻译的。
- **ISA**：老式总线标准（工业标准体系结构），其 DMA 控制器地址线少，是映射寄存器“别名低位地址”的典型来源。
- **PCI**：主流外设总线标准（外设部件互连），其 DMA 设备地址线通常充足，但仍经映射寄存器翻译；PCI 设备的配置空间与总线号是本批详解十一的主题。
- **SoC（片上系统）**：把处理器、内存控制器、DMA 控制器等集成在一块芯片上的系统。为什么本批提它：Version 3 DMA 接口主要是为 SoC 里的系统 DMA 控制器准备的（它们本身就能做散点/聚集）。
- **ARM**：一种处理器架构，常见于手机、嵌入式与部分服务器。本批里它与缓存一致性相关：部分 ARM 平台不由硬件保证缓存一致，驱动的刷缓存纪律在这些平台上才是 correctness 保障（x86 上只是空操作）。
- **逻辑地址（设备地址）**：DMA 设备看内存用的地址，又叫设备地址空间里的地址。为什么要有：设备能发出的地址位数未必覆盖全部物理内存（老式 ISA 设备地址线不够，32 位机的 4GB、PAE 的 64GB 都可能超界）；即使地址线够，设备看到的地址也不一定等于物理地址。没有一层翻译，设备无法可靠寻址内存。
- **页表项（PTE）**：页表里的一项，把一页虚拟地址翻译成一页物理地址。为什么要有：它是虚拟地址到物理地址的翻译载体；没有它，CPU 无法把指令地址落到内存。
- **HAL（硬件抽象层）**：Windows 内核与硬件之间的可移植层，把平台相关的操作（中断控制、DMA 映射、定时器等）包成统一接口。为什么要有：同一套驱动要在不同处理器、不同总线、不同 DMA 控制器上跑，必须有人把平台差异收敛起来；HAL 就是这个人。本批里 HAL 的活是提供映射寄存器翻译与适配器对象。
- **MDL（内存描述符表，第 4、7 批已定义，本批沿用）**：内存管理器建的不透明结构，把一段虚拟地址范围翻译成一组物理页。对驱动而言，MDL 条目与映射寄存器一一对应：每个 MDL 条目与每个映射寄存器都至多映射一整页、最少映射一字节，且都可能只映射页内一段（首尾页可带页内偏移）。
- **映射寄存器（map register）**：HAL 提供的翻译单元，把一个设备可访问的逻辑页翻译成一页 CPU 的物理内存。对设备硬件而言，它干的活与页表对软件、MDL 对驱动干的活相同：翻译地址。为什么要有：设备不能直接用逻辑地址访问物理内存，必须先翻译；没有它，地址线不足的设备够不着高位内存，地址线足的设备也无法保证逻辑地址与物理地址一致。形态上它可以是芯片里的真实硬件寄存器、系统 DMA 控制器里的寄存器、总线主控适配器里的寄存器，也可以是 HAL 在系统内存里建的虚拟寄存器——驱动只把它当不透明句柄（MapRegisterBase）用，不碰本体。
- **DMA（直接内存访问，第 8 批已定义，本批沿用）**：DMA 控制器直接读写物理内存的数据搬运方式，CPU 只编程（源、目的、长度），不参与逐字搬运。
- **PIO（程序控制 I/O，第 7 批已定义，本批沿用）**：CPU 执行指令直接搬运数据的方式，一次一字/一字节。本批里它只作 DMA 的对照出现（PIO 技术与 PIO 读缓存规则见详解九）。
- **从属设备（subordinate device）**：用系统 DMA 控制器的设备，又叫用系统（或从属）DMA 的设备。为什么要有这个区分：这类设备自己没有能力仲裁总线、编程传输，必须借系统 DMA 控制器干活；与此相对的是总线主控设备。
- **总线主控设备（bus-master device）**：自己能与系统仲裁总线使用权、自己编程传输的设备。为什么要有这个区分：这类设备不需要系统 DMA 控制器替它搬运，驱动给它逻辑地址它自己干；两类设备的驱动写法（通道归属、释放例程）因此不同。
- **适配器对象（adapter object）**：代表 DMA 硬件的软件对象——要么代表系统 DMA 控制器的一个通道或端口（从属设备用），要么代表一个总线主控设备。为什么要有：DMA 硬件五花八门（ISA 控制器通道、PCI 总线主控、SoC 集成控制器），驱动不能直接碰硬件；适配器对象把“通道/端口/主控设备”统一成一个对象，并挂上 HAL 提供的一张函数表与映射寄存器集，驱动只跟对象打交道。驱动用 IoGetDmaAdapter 拿到它，指针通常存在设备扩展里。
- **DMA 操作接口（DMA_OPERATIONS）**：适配器对象里挂的一张函数指针表，驱动做 DMA 必须经这张表调函数，不能直接按名字调。为什么要有：DMA 是平台相关操作（不同 HAL、不同总线实现不同），函数表把平台差异收敛到 HAL 一侧；驱动只认接口，不认实现。历史：这张表取代了旧版 Windows NT 的 HalXxx 例程；头文件里留着旧名的宏只为兼容老驱动，新驱动一律走表。版本：v1/v2 是旧接口（本批大部分内容按它讲），v3 自 Windows 8 起可用（见详解八）。
- **DEVICE_DESCRIPTION**：驱动调 IoGetDmaAdapter 时填的设备描述结构，说明自家设备的 DMA 能力。关键字段：是不是总线主控、有没有散点/聚集能力、一次最多传多少字节（MaximumLength）。为什么要有：HAL 靠它决定给驱动分配什么样的适配器对象与多少映射寄存器。填写纪律：先 RtlZeroMemory 全清零，再填相关字段，无关字段留零；BusNumber 字段 WDM 驱动不用。
- **PAGE_SIZE**：内存页的大小，x86/x64 上通常为 4KB。为什么本批提它：映射寄存器按页算，单次传输的字节上限常以它为单位（PAGE_SIZE × 映射寄存器数）。
- **映射寄存器数（NumberOfMapRegisters）**：IoGetDmaAdapter 交回来的第二个东西：每次传输可用的映射寄存器个数。为什么要有：它决定一次 DMA 最多传多少字节（至多 PAGE_SIZE × 寄存器数），是拆传输的硬上限。HAL 的分配策略：条件允许时给“传 MaximumLength 所需数 + 1”，否则给平台上尽可能大的值；没有任何保证驱动一定拿到想要的数，所以驱动必须每次检查返回值，不能假设。
- **AdapterControl 例程**：驱动写的、负责发起 DMA 操作的例程。为什么要有：DMA 通道与映射寄存器是共享资源，驱动不能上来就编程硬件，必须先向 I/O 管理器申请（AllocateAdapterChannel）；资源就绪时 I/O 管理器调入这个例程，驱动在里面编程设备、发起传输。谁不需要它：用散点/聚集例程的驱动、用常驻共享缓冲做总线主控 DMA 的驱动（见详解四、六）。
- **AdapterListControl 例程**：散点/聚集 DMA 专用的、与 AdapterControl 对应的例程。为什么要有：散点/聚集把“申请通道 + 建表”包进了 GetScatterGatherList，驱动不再需要自己调申请通道与建映射的例程，只需要一个例程在表建好后“拿表编程设备”。与 AdapterControl 的区别：无返回值（AdapterControl 返回 IO_ALLOCATION_ACTION）；第三个参数不是 MapRegisterBase，而是指向一张建好的分段表的指针；只干子集（存表指针、设设备、用表传输），不调申请通道与建映射的例程。
- **IO_ALLOCATION_ACTION**：AdapterControl 与 ControllerControl 的返回值类型，回答“我对这个共享资源怎么处置”。三个值：KeepObject（留着，我还要用）、DeallocateObjectKeepRegisters（还通道、留寄存器）、DeallocateObject（全还，下一个可以上了）。为什么要有：通道与寄存器是分开的两样资源，驱动对它们的处置意愿不同，必须用返回值一次性说清，系统才知道何时把资源让给别人。
- **包 DMA（packet-based DMA）**：按 IRP 打包的 DMA：一次传输对应一个 IRP（或它的一个片段），传完即释放资源。为什么要有：大多数传输都是一次性的——读这么多、写那么多，传完就完；包 DMA 的生命周期与 IRP 对齐，资源随 IRP 走。
- **公共缓冲区（common buffer）**：驱动长期持有、反复用于 DMA 的一块内存，又叫连续 DMA（continuous DMA）用的缓冲。为什么要有：有些设备无法让驱动“随时知道传输何时开始、何时结束”（比如用邮箱机制传命令与状态的适配器），或者同一块缓冲要被连续、反复地传；这时按 IRP 打包不合适，需要一块常驻的、驱动与设备共享的缓冲。分配用 AllocateCommonBuffer，释放用 FreeCommonBuffer。
- **自动初始化模式（auto-initialize mode）**：系统 DMA 控制器的一种模式，控制器可以自动重装、不必每次传输都重新编程。为什么要有：公共缓冲系统 DMA 靠它——控制器对着同一块公共缓冲循环传，驱动只管在缓冲与用户缓冲之间倒数据（RtlMoveMemory），不必每次都重设控制器。
- **散点/聚集（scatter/gather）**：把物理上不连续的多段内存当一次传输处理的技术。为什么要有：用户缓冲在物理上常常是不连续的（MDL 里多段）；没有它，驱动必须把每段拆成多次传输；有了它，一次传输描述里可以带多段，设备按表一段段传。注意：设备本身不必有散点/聚集能力——系统可以替它建表（见详解四）。
- **SCATTER_GATHER_LIST**：散点/聚集表结构，含一个 SCATTER_GATHER_ELEMENT 数组与元素个数。每个元素给一段物理连续区的长度与起始物理地址。为什么要有：它是驱动编程设备的直接输入——驱动按元素逐段设设备，不必自己从 MDL 里抠物理地址。
- **MapTransfer**：v1/v2 接口里“建映射并设控制器”的例程。为什么要有：光有 MDL（物理页）还不够，设备要的是逻辑地址；MapTransfer 把 MDL 描述的物理范围映射成设备可用的逻辑地址（系统 DMA：设系统 DMA 控制器；总线主控：返回逻辑地址给驱动编程设备）。v3 的对应物是 MapTransferEx（一次可处理整条 MDL 链，见详解八）。
- **缓存一致性（cache coherency）**：处理器缓存、DMA 控制器内缓冲与内存三处数据必须一致的状态。为什么要有：DMA 直接读写内存、不经过处理器缓存；若处理器缓存里有更新的数据没写回内存，DMA 读到的就是旧数据；若 DMA 写进内存的新数据没使缓存失效，CPU 读到的还是旧数据。没有刷新纪律，DMA 传输的数据完整性无从保证。
- **KeFlushIoBuffers**：刷处理器缓存的例程，传输前调（DMA）或 PIO 读结束时调。为什么要有：它是处理器缓存与内存之间的同步点。在硬件自动保证一致性的平台上它是空操作（几乎无开销），所以驱动可以无条件调它。
- **FlushAdapterBuffers**：刷 DMA 控制器内缓冲的例程，每次传输结束时调。为什么要有：DMA 控制器可以在内部缓冲里攒数据（通常 8 字节以上的定长块，攒满才传）；不刷，驱动无法确定“每个字节到底传过去没有”。它返回布尔值，驱动据此定 IRP 状态块。
- **控制器对象（controller object）**：多设备共享一块物理控制器时的同步对象。为什么要有：一块控制器（如 AT 磁盘控制器）挂多个同类设备，同一时刻只能给一个设备干活；控制器对象把“谁在用控制器”这件事收敛成一个排队机制，驱动不必自己维护 Busy 标志、重排 IRP。注意：它没有名字、不是 I/O 目标，WDM 驱动不能用（见详解十）。
- **控制器扩展（controller extension）**：控制器对象附带的一块驱动定义内存，大小由调 IoCreateController 的驱动定，系统从非分页池分配并清零。为什么要有：控制器相关的状态（当前目标设备、中断对象指针）需要一个常驻的家；控制器扩展就是这个家。
- **ControllerControl 例程**：驱动写的、负责给控制器编程的例程，与 AdapterControl 对应。为什么要有：控制器是共享资源，驱动不能上来就编程，必须先向 I/O 管理器申请（IoAllocateController）；控制器空闲时 I/O 管理器调入这个例程，驱动在里面编程控制器、发起操作。
- **NT4 / Win2K**：Windows NT 4.0 与 Windows 2000 的简称。本批里它们是配置空间访问方式变迁的分界点：NT4 时驱动调 HalGetBusData 查总线，Win2K 起总线归总线驱动管、该办法已废。
- **配置空间（configuration space）**：某些总线为每个设备提供的一段特殊空间，含设备描述与资源需求。为什么要有：驱动有时需要查设备的资源与描述（正常情况下 PnP 的 START_DEVICE 已经给了，写得好的驱动根本不需要查；只有特殊原因才查）。
- **BUS_INTERFACE_STANDARD**：系统提供的、与总线无关的总线接口（GUID_BUS_INTERFACE_STANDARD），含一组总线驱动提供的例程指针（翻译总线地址、取 DMA 适配器、读写配置空间）。为什么要有：在 DISPATCH_LEVEL 不能发 IRP，但驱动仍需访问配置空间；这个接口把“直接调总线驱动例程”变成可能，且接口自带与总线驱动的硬件访问同步，驱动不必自己为此拿自旋锁。
- **ReadDmaCounter**：读 DMA 控制器剩余字节数的例程，公共缓冲系统 DMA 里用。为什么要有：公共缓冲传输是连续的，驱动需要知道“公共缓冲里还剩多少没传完”，才能决定继续往缓冲里填数据还是往用户缓冲里倒数据。
- **链式 MDL（MDL chain）**：多个 MDL 串成的链，每个 MDL 描述一段虚拟连续的缓冲。为什么要有：复杂传输的缓冲可能由多段虚拟连续区组成，一条 MDL 不够；MapTransferEx 一次可处理整条链，驱动不必自己双重循环。
- **IOMMU（I/O 内存管理单元）**：给设备 DMA 加地址翻译与访问控制的硬件单元。本批底本（2019 版指南）不讲它；这里只作边界标记——现代机器上设备 DMA 还要过 IOMMU 这一关，审计现代驱动时“设备能读写多大范围物理内存”的结论须另算 IOMMU 在场与否，本批结论只覆盖映射寄存器给设备什么地址这一层。

---

## 一、本批地图

本批的逻辑是一条线：**先有地址翻译（映射寄存器），再有资源对象（适配器对象），再有两条传输路（包 DMA、公共缓冲 DMA），再有两种设备（系统 DMA、总线主控），最后是两样外围（缓存刷新、配置空间）与一个串行化工具（控制器对象）**。

```
用户缓冲（虚拟地址）
  │  MmProbeAndLockPages（第 7 批：请求进程上下文，锁页填 PFN）
  ▼
MDL（一组物理页） ── MapTransfer ──► 逻辑地址（设备看内存的地址）
  │                                    │
  │  映射寄存器：每页一个翻译槽          ▼
  │  （MapRegisterBase 句柄）         设备搬运数据
  │                                    │
  └─ 适配器对象（IoGetDmaAdapter 拿）    ▼
     通道/端口（系统 DMA）或主控设备   传输结束：FlushAdapterBuffers
     AllocateAdapterChannel 申请        释放：FreeAdapterChannel / FreeMapRegisters
     AdapterControl 里编程设备
```

两对划分的判据（本批最重要的一张表）：

| 划分 | 判据 | 两边 |
|---|---|---|
| 系统 DMA vs 总线主控 DMA | 谁搬运 | 系统 DMA：系统 DMA 控制器搬，设备是从属设备；总线主控：设备自己仲裁总线、自己搬 |
| 包 DMA vs 公共缓冲 DMA | 缓冲生命周期 | 包 DMA：一次传输对应一个 IRP（或片段），传完释放；公共缓冲 DMA：一块缓冲长期持有、反复传（如邮箱机制） |

四格组合（系统/总线主控 × 包/公共缓冲）都有对应的指南小节，但**不是四格平权**：包 DMA 是主路，公共缓冲是特例（设备无法让驱动随时知道传输起止、或缓冲需连续复用时才用）；公共缓冲要省着用，因为它会占掉适配器对象的映射寄存器，挤占包 DMA 的资源。

与前批的交接点（只提及，不展开）：第 8 批 DPC 职责里的“包 DMA 每次传输后 FlushAdapterBuffers、全做完后 FreeAdapterChannel / FreeMapRegisters、PIO 读结束时 KeFlushIoBuffers、控制器对象用完 IoFreeController”——本批讲的就是这些动作背后的完整机制。第 7 批 StartIo 里的“DMA 设备拆部分传输、AllocateAdapterChannel”——本批讲拆的四种原因与四种拆法。

## 二、核心要点

1. **三地址空间缺一不可**：虚拟地址（CPU 指令用）、物理地址（内存条用）、逻辑地址（设备用）。页表翻译虚拟→物理，MDL 描述虚拟→物理，映射寄存器翻译逻辑→物理。驱动不能拿虚拟指针当物理地址用，设备不能拿逻辑地址直接访问内存。
2. **映射寄存器是按页算的**：每个映射寄存器至多映射一整页、最少一字节；一次传输需要的寄存器数 = 缓冲跨越的页数（ADDRESS_AND_SIZE_TO_SPAN_PAGES 算）。寄存器数就是拆传输的硬上限。
3. **适配器对象是软件抽象，不是硬件**：它代表系统 DMA 控制器的一个通道/端口，或一个总线主控设备；映射寄存器集可能是真实硬件寄存器，也可能是 HAL 在内存里建的虚拟寄存器。驱动只认对象指针与 MapRegisterBase 句柄。
4. **IoGetDmaAdapter 交回来两样东西**：适配器对象指针（存设备扩展，常驻内存，因为后续例程多在 DISPATCH_LEVEL 调）与 NumberOfMapRegisters（每次传输可用数）。HAL 尽量给“MaximumLength 所需数 + 1”，但不保证——驱动必须检查返回值，不能假设。
5. **系统 DMA 与总线主控 DMA 的三处不同**：① 谁搬运（控制器 vs 设备自己）；② MapTransfer 返回的逻辑地址怎么处置（系统 DMA 必须忽略，总线主控必须用它编程设备）；③ 释放例程（系统 DMA 用 FreeAdapterChannel，总线主控包 DMA 用 FreeMapRegisters）。
6. **AdapterControl 一个 IRP 只调一次**：它只负责第一段传输的编程；后续部分传输由 DpcForIsr / CustomDpc 重编设备。AdapterControl 不能等设备传完——它只保存上下文（特别是 MapRegisterBase）并返回处置值。
7. **三返回值是一张资源处置表**：KeepObject（系统 DMA：留通道 + 留寄存器）；DeallocateObjectKeepRegisters（包总线主控：还通道、留寄存器，DPC 随后 FreeMapRegisters）；DeallocateObject（控制器对象：全还，下一个操作可重叠）。选错的后果见详解四。
8. **拆传输的四个原因**：映射寄存器不够、Length 太大、缓冲跨页太多、设备自身限制（如 AT 磁盘控制器 256 扇区）。拆的四种手法：部分 MDL、部分 IRP、类/端口驱动的分片 IRP、单体驱动自己分段；部分传输的 CurrentVa / Length 更新有固定公式。
9. **散点/聚集是系统服务，不是设备能力**：GetScatterGatherList / PutScatterGatherList 包了“申请通道 + 建表”，设备没有 SG 能力也能用（此时表只有 1 个元素）；一次可排多个 GetScatterGatherList（AllocateAdapterChannel 不行）；用完必须先 PutScatterGatherList 才能碰缓冲里的数据。
10. **缓存要刷两处**：处理器缓存（KeFlushIoBuffers，传输前；PIO 读在结束时）与控制器内缓冲（FlushAdapterBuffers，每次传输结束时）。v1/v2 接口必须手动刷；v3 接口把刷内建进了 MapTransferEx / FlushAdapterBuffersEx，KeFlushIoBuffers 不再需要（ARM 上多调反而降性能）。
11. **Version 3 是 SoC 时代的接口**：Win8 起可用；允许系统 DMA 控制器也做散点/聚集（v1/v2 只允许总线主控做 SG）；一次 MapTransferEx 可处理整条 MDL 链；部分映射时 Length 出参告诉驱动实际映射了多少，驱动按 Offset += X、Length -= X 续传。
12. **控制器对象是 WDM 之外的遗产工具**：无名、不是 I/O 目标、WDM 驱动不能用；只服务单体驱动 + 同类设备 + 控制器不做长操作不中断的场景。它的价值是“可重叠”（一个盘 seek 的同时另一个盘传数据），代价是驱动必须自己管 KeepObject / DeallocateObject 的分界与 IoFreeController 的及时归还。
13. **配置空间访问的 IRQL 决定手段**：PASSIVE_LEVEL 走 IRP（IRP_MN_READ_CONFIG / WRITE_CONFIG）；DISPATCH_LEVEL 走总线接口（先在 PASSIVE_LEVEL 用 IRP_MN_QUERY_INTERFACE 拿到 BUS_INTERFACE_STANDARD 存非分页池，再调 GetBusData / SetBusData，用完解引用）。NT4 的 HalGetBusData 在 Win2K 起已废——总线归总线驱动管，不归 HAL。
14. **不要依赖总线号**：PCI 总线号可动态变化；直接按总线号访问 PCI 端口可能导致系统失败。只要总线/设备/功能号，用 IoGetDeviceProperty 问 PDO 就够了。

---

## 三、关键知识点详解

### 详解一：三地址空间与映射寄存器——DMA 的翻译层

**为什么非要三层不可**。CPU 指令只能用虚拟地址；内存条只认物理地址；DMA 设备只认逻辑地址。三者没有一对是天然相等的：虚拟地址经页表才到物理地址；逻辑地址经映射寄存器才到物理地址。驱动手里拿虚拟指针不能直接当物理地址用（第 7 批已讲过：MmGetMdlVirtualAddress 的返回值只是 MDL 索引，不是可访存地址）；设备手里拿逻辑地址也不能直接访问内存。没有翻译层，DMA 根本无从寻址。

**为什么设备需要自己的地址空间**。历史原因：老式 ISA DMA 控制器、从属设备、总线主控设备的地址线常常不够——32 位机的 4GB、x86 PAE 的 64GB 都可能超出它们的寻址能力。PCI DMA 设备地址线通常够，但即便如此，映射后的逻辑地址也不一定等于物理地址。所以 HAL 的策略是统一的：不管设备地址线够不够，一律给映射——逻辑地址永远是“经映射寄存器翻译后的地址”，驱动不许假设逻辑地址等于物理地址。64 位下地址线充足的设备，逻辑地址常常恰好等于物理地址，但这只是恰好，不是契约；驱动绕过 MapTransfer 直接拿物理地址编程设备，在另一台机器上就可能错。

**映射寄存器的粒度为什么是页**。每个映射寄存器与每个 MDL 条目一一对应：至多一整页、最少一字节，首尾页可带页内偏移。为什么按页：因为翻译的硬件单位就是页（页表、MDL、映射寄存器三者同构）。后果：一次传输需要的寄存器数 = 缓冲跨越的页数，与字节数没有直接关系——一段跨 12 页的小缓冲（首尾带偏移）比一段整齐的 5 页大缓冲更费寄存器。ADDRESS_AND_SIZE_TO_SPAN_PAGES 就是算这个数的宏。

**映射寄存器到底是什么东西**。驱动视角里它只是一个句柄（MapRegisterBase）：可能是芯片里的真实硬件寄存器、系统 DMA 控制器里的寄存器、总线主控适配器里的寄存器，也可能是 HAL 在系统内存里建的虚拟寄存器。为什么要抽象成句柄：HAL 要在各种 DMA 设备与总线、各种机器上提供同一套适配器对象；驱动若直接碰硬件寄存器，换一台机器就得重写。句柄制的代价是驱动永远不知道寄存器到底在哪——也不需要知道。

**32/64 位分叉**。32 位机：虚拟地址空间 4GB，设备地址线不足是常态，映射寄存器是刚需（ISA 设备只能访问低位逻辑地址，HAL 把高位物理页别名到低位逻辑地址）。64 位机：地址线充足的设备逻辑地址常等于物理地址，但映射抽象仍在——驱动仍须走 MapTransfer，不能图省事直传物理地址。单/多核分叉：映射寄存器集是适配器对象的资源，多设备、多核并发申请时，通道与寄存器集的分配必须排队（AllocateAdapterChannel 排队机制），单核上只是“别的驱动先占着”，多核上是“别的核上的驱动同时在抢”——排队纪律相同，竞争烈度不同。

### 详解二：适配器对象与 IoGetDmaAdapter——DMA 的资源对象

**为什么需要适配器对象**。DMA 硬件形态太多：ISA 控制器通道、PCI 总线主控、SoC 集成控制器——驱动不能为每种硬件写一套。适配器对象把它们统一成“一个对象 + 一张函数表 + 一组映射寄存器”，驱动只跟对象打交道，HAL 在对象后面处理平台差异。没有它，驱动要么直接碰硬件（换机器就坏），要么为每种控制器写分支（不可维护）。

**IoGetDmaAdapter 在什么时候调**。设备启动时，通常是处理 IRP_MN_START_DEVICE 的分发例程里（PASSIVE_LEVEL 的 PnP 分发上下文，以函数调用方式调 I/O 管理器，I/O 管理器再调 HAL 建对象）。为什么在启动时：适配器对象是设备整个生命周期都要用的资源，不是每次传输现申请的；启动时一次拿好，存在设备扩展里，后续传输只管用。

**DEVICE_DESCRIPTION 为什么要先清零再填**。结构里有些字段对某些设备无意义（如 BusNumber 字段 WDM 驱动不用），不清零就会把栈上垃圾当成设备能力交给 HAL——HAL 可能据此分配错误的适配器对象或寄存器数。先 RtlZeroMemory 全清零、再填相关字段，是“无关字段留零”的纪律。关键字段的含义：Master（是不是总线主控，决定对象代表通道还是主控设备）、ScatterGather（有没有 SG 能力——注意：从属设备只有在“传输被拆成多次时能等系统 DMA 控制器重编”才可传 TRUE，否则传 TRUE 会让 HAL 假设设备能等重编，而设备其实不能等）、MaximumLength（一次最多传多少字节，HAL 据此算寄存器数）。

**交回来的两样东西各有什么用**。适配器对象指针：后续所有 DMA 例程（AllocateAdapterChannel、MapTransfer、FlushAdapterBuffers、FreeAdapterChannel / FreeMapRegisters、GetScatterGatherList 等）的必填参数；存储必须常驻（设备扩展、控制器扩展或驱动分配的非分页池），因为后续例程多在 DISPATCH_LEVEL 调，分页内存此时不可碰。NumberOfMapRegisters：每次传输可用数，是拆传输的硬上限；HAL 的策略是“条件允许时给 MaximumLength 所需数 + 1，否则给平台上尽可能大的值”——没有任何保证，所以驱动必须检查返回值。为什么给“+1”：多一个寄存器，首尾带偏移的缓冲也能一次传完，少一次拆分；但这只是“尽量”，不是承诺。

**什么时候还**。全部 DMA 做完时调 PutDmaAdapter 释放适配器对象。为什么必须有“还”：适配器对象占着 HAL 的通道/寄存器资源描述，不还是泄漏；且 PutDmaAdapter 之前必须先还清通道与寄存器（FreeAdapterChannel / FreeMapRegisters），否则资源账对不上。

### 详解三：缓存一致性——DMA 数据正确的两道刷

**为什么 DMA 有缓存问题而 CPU 访存没有**。CPU 访存经过自己的缓存体系，硬件保证缓存与内存一致；DMA 控制器读写内存时不经过处理器缓存。于是出现两处不一致：① 处理器缓存里有比内存更新的数据（写操作前没刷，DMA 从内存读到的是旧数据；读操作前没刷，DMA 写进内存的新数据随后被处理器缓存里的旧数据覆盖）；② DMA 控制器自己有内部缓冲（定长块，通常 8 字节以上，攒满才传），驱动以为传完了，其实还有字节躺在控制器缓冲里。

**v1/v2 的两条纪律（指南明言只适用于 v1/v2，v3 另有规矩）**。第一，传输前调 KeFlushIoBuffers：同步处理器缓存与内存。若驱动调 AllocateCommonBuffer 时 CacheEnabled = TRUE，也必须在每次传输前调它。为什么传输前：DMA 读的是内存、写的是内存，传输开始那一刻内存必须是准的；传输开始后再刷就晚了。第二，每次传输结束时调 FlushAdapterBuffers：把控制器内缓冲的余字节真正写进内存或推出到设备。为什么每次传输结束都要刷：控制器攒满才传的习性意味着“设备中断说传完了”不等于“每个字节都到了”；不刷，驱动无法确定每个字节的去向。FlushAdapterBuffers 返回布尔值，驱动据此定 IRP 状态块——刷失败，IRP 就不能报成功。

**为什么在 x86 上这些调用几乎无开销**。KeFlushIoBuffers 在硬件自动保证一致性的平台上是空操作。为什么指南仍要求调：驱动要跑在所有平台上（包括无硬件一致性的架构），调用本身在一致性平台上近零成本，在非一致性平台上是 correctness 的唯一保障。不调，在 x86 上测不出来，在 ARM/SoC 上就是数据损坏——这是典型的“在你机器上能跑、在别人机器上错”的 bug 来源。

**PIO 读的单独一条**。PIO 读操作结束时调 KeFlushIoBuffers（指令与数据缓存 在 PIO 读时也可能不一致）。为什么是读结束、不是读开始：PIO 读是 CPU 把设备寄存器读进内存，读完那一刻处理器缓存里可能还留着旧数据，驱动随后访问内存会读到缓存旧值；读结束刷一次，缓存与内存重新对齐。DMA 的 KeFlushIoBuffers 在传输前、PIO 的在读结束——方向不同，时序不同，不要记混。

### 详解四：散点/聚集 DMA 与 AdapterControl——两条省事路与一条老路

**散点/聚集为什么能省事**。包 DMA 的老路是：AllocateAdapterChannel 申请通道 → AdapterControl 里 MapTransfer 建映射 → 编程设备 → 传完 FlushAdapterBuffers → FreeAdapterChannel / FreeMapRegisters。散点/聚集把前两步包进 GetScatterGatherList：它内部调 AllocateAdapterChannel 与 MapTransfer，建好 SCATTER_GATHER_LIST，再调入驱动的 AdapterListControl。驱动在 AdapterListControl 里只干三件事：存表指针、设设备、用表传输。为什么能包：建表需要的输入（MDL、长度、方向）驱动本来就有，系统替驱动跑申请与映射的流程，驱动只剩“编程设备”这一件本职。

**设备没有 SG 能力也能用，为什么**。GetScatterGatherList 建的表描述的是缓冲的物理分段，不是设备的能力。设备有 SG 能力：表有多个元素，设备按表逐段传。设备没有 SG 能力：表只有 1 个元素，驱动按单段传——与老路等价，但省了自己调 AllocateAdapterChannel 与 MapTransfer。为什么指南强调这一点：很多驱动以为“我的设备没 SG 就不能用这组例程”，其实系统建表与设备能力无关；用它还能换来一个老路没有的好处：一次可排多个 GetScatterGatherList（AllocateAdapterChannel 同一时刻只能排一个申请）。

**AdapterListControl 与 AdapterControl 的三处不同**。① 返回值：AdapterListControl 无返回值，AdapterControl 返回 IO_ALLOCATION_ACTION——因为散点/聚集路里资源处置由 Get/PutScatterGatherList 包办，驱动不必在例程里表态。② 第三个参数：AdapterListControl 拿到的是 SCATTER_GATHER_LIST 结构指针，AdapterControl 拿到的是 MapRegisterBase 句柄——前者是“建好的表”，后者是“建映射的原材料”。③ 职责子集：AdapterListControl 不调 AllocateAdapterChannel 与 MapTransfer，只存表、设设备、传数据。运行上下文：AdapterListControl 在任意线程上下文、DISPATCH_LEVEL 运行（指南明言）；这意味着它不能等、不能碰分页、上下文不能是常驻假设之外的东西。

**用完必须先 PutScatterGatherList 才能碰数据，为什么**。PutScatterGatherList 干三件事：刷适配器缓冲、还映射寄存器、释放散点/聚集表。没调它之前，数据可能还在控制器缓冲里、映射还没拆除——驱动此时读缓冲，读到的是旧数据或不完整数据。顺序是硬性的：传完 → PutScatterGatherList → 才能访问缓冲数据 → 才能完成 IRP。

**AdapterControl 的老路为什么还值得讲**。因为三类驱动仍走它：包系统 DMA、包总线主控 DMA（不用 SG 例程时）、以及所有需要精细控制映射的驱动。它的最低职责只有两条：保存 MapRegisterBase（后续 FlushAdapterBuffers 与 MapTransfer 都要它）与返回处置值。为什么最低职责只有这两条：AdapterControl 不能等设备传完（它在 DISPATCH_LEVEL，等待会死锁或蓝屏），所以它只能“记下关键句柄、表态、返回”，把“等传完、刷、释放”留给 DpcForIsr / CustomDpc。AdapterControl 一个 IRP 只调一次——后续部分传输由 DPC 重编设备；若 AdapterControl 被调多次，说明驱动把“首次编程”与“后续重编”混在一起了，上下文会被覆盖。

**三返回值的选择逻辑**。KeepObject：系统 DMA（从属设备）——通道与寄存器都要留着，因为同一 IRP 的后续部分传输还要用同一个控制器；SG 能力的从属设备也返回 KeepObject，但前提是设备能等控制器重编（拆传输时控制器要重设，设备必须等得起）。DeallocateObjectKeepRegisters：包总线主控 DMA——通道还掉（别的设备可以用总线了），寄存器留着（本 IRP 后续部分传输还要映射）；DPC 随后调 FreeMapRegisters 还寄存器。例外：设备不支持命令排队时，AdapterControl 可在当前 IRP 传完时返回 KeepObject，DPC 改调 FreeAdapterChannel——因为此时“通道”与“寄存器”的区分对该设备没有意义，整套一起还更干净。选错的后果：该留的还了，后续部分传输无映射可用；该还的留了，通道被占着，别的设备饿死。

### 详解五：拆传输——四个原因与四种手法

**为什么必须拆**。四个原因，任一成立就得拆：① 映射寄存器不够（ADDRESS_AND_SIZE_TO_SPAN_PAGES 算出的页数 > NumberOfMapRegisters）；② Length 太大（单次长度上限是 PAGE_SIZE × 寄存器数）；③ 缓冲跨页太多（与 ① 同源，但跨页数由缓冲的物理布局决定，不是字节数决定）；④ 设备自身限制（指南举例：系统 AT 磁盘驱动的控制器一次最多 256 扇区）。为什么不能“将就一次传完”：映射寄存器是硬上限，超了就没有翻译槽；设备限制是硬约束，超了设备会错传或拒传。拆不是优化，是正确性要求。

**指南的算例**。12 页需要、5 个寄存器可用、无设备限制 → 必须做 3 次 DMA 操作，MapTransfer 调 3 次。为什么是 3 次不是 2 次：5 + 5 = 10 < 12，第三次传剩下 2 页。部分传输的更新公式是固定的：CurrentVa = CurrentVa +（上次 MapTransfer 请求的长度）；Length = min（剩余长度，PAGE_SIZE × NumberOfMapRegisters）。为什么 CurrentVa 要累加：每次 MapTransfer 的 CurrentVa 是“这次从缓冲哪里开始”，必须接着上次结束的地方；Length 要取 min：剩余长度与单次上限的较小者，超了上限的那部分留给下一次。

**四种拆法，各在什么场景**。① 部分 MDL：IoAllocateMdl 为一段用户缓冲建 MDL → MmProbeAndLockPages 锁这一段 → 传这一段 → MmUnlockPages；MDL 够大则 MmPrepareMdlForReuse 复用，否则 IoFreeMdl 重建。适用：驱动自己控制缓冲分段。② 部分 IRP：高层驱动调 IoBuildSynchronousFsdRequest 建部分传输 IRP、锁一段用户缓冲 → IoCallDriver 下发 → 下层返回 STATUS_PENDING 时 KeWaitForSingleObject 等事件 → 醒来重复，直到传完再完成原始 IRP。适用：高层驱动在内存受限机器上锁不下整段缓冲。③ 类/端口分片：存储类驱动为每片另建一个 IRP、挂 IoCompletion 例程跟踪整单状态并释放分片 IRP，再 IoCallDriver 发给端口驱动。适用：类/端口对；前提是类驱动必须知道端口驱动的映射寄存器数——端口驱动须把该数存注册表，或两驱动用内部 IOCTL 私有接口传。④ 单体驱动自己分段：非类/端口对的单体驱动自己把大请求切成一段段 DMA 操作。适用：单体驱动。为什么分法这么多：拆的活可以在栈的任何一层做，做拆的那层必须同时满足两个条件——知道映射寄存器数（或设备限制），且有办法跟踪“整单传完没有”；不同驱动模型里这两个条件落在不同层。

**谁负责判断“要不要拆”**。调 AllocateAdapterChannel 的例程负责判断（通常是 StartIo 或 ControllerControl 在调之前算好）。为什么是它：它手里有 Length、MDL、NumberOfMapRegisters 三样输入，是唯一能算出“一次够不够”的位置；AdapterControl 只负责执行分段，不负责决策——决策与执行分离，AdapterControl 一个 IRP 只调一次才成立。

**64 位与单/多核分叉**。64 位：地址线充足的设备逻辑地址常等于物理地址，但映射寄存器上限仍在（寄存器数是 HAL 按设备与平台给的，不是按地址线给的），拆传输的四个原因一个不少。单核：拆的段是串行的，前一段的 DPC 重编设备、再传下一段；多核：多个设备的 DMA 申请可能同时在不同核上排队，通道与寄存器集的竞争更激烈，但拆的公式与释放纪律不变——FreeAdapterChannel / FreeMapRegisters 必须紧跟最后一次 FlushAdapterBuffers，否则别的核上的驱动饿死。

### 详解六：系统 DMA 的两条路——包 DMA 与公共缓冲 DMA

**包系统 DMA 的完整序列（v1/v2）**。KeFlushIoBuffers（传输前，刷处理器缓存）→ AllocateAdapterChannel（调用方须在 DISPATCH_LEVEL；资源就绪时 I/O 管理器以函数调用方式调入 AdapterControl，传 DeviceObject、Context、MapRegisterBase；有 StartIo 的驱动还会传 DeviceObject->CurrentIrp，没有 StartIo 的驱动须把当前 IRP 放进 Context）→ AdapterControl 里 MmGetMdlVirtualAddress（取 MDL 索引——再次强调：返回值不是可访存地址，只能传给 MapTransfer）→ MapTransfer（设系统 DMA 控制器；返回的逻辑地址系统 DMA 驱动必须忽略——控制器用它，驱动不用）→ 设备传 → DpcForIsr 里 FlushAdapterBuffers（每次传输结束一次；MapTransfer 调几次，FlushAdapterBuffers 就调几次）→ 全部传完或出错时 FreeAdapterChannel（紧跟最后一次 FlushAdapterBuffers，越快越好，让别的驱动能用控制器）。

**为什么 MapTransfer 的逻辑地址系统 DMA 要忽略**。系统 DMA 里，真正编程控制器的是 MapTransfer 本身——它已经把映射写进控制器了；返回的逻辑地址只是“控制器看到的地址”，驱动拿它没有用（驱动不直接编程控制器寄存器）。总线主控则相反：MapTransfer 只建映射，编程设备的是驱动自己，所以驱动必须拿返回的逻辑地址去设设备。同一例程、同一返回值，两边处置相反——这是本批最容易记混的点之一。

**公共缓冲系统 DMA 什么时候用**。设备用系统 DMA 控制器的自动初始化模式时。自动初始化模式下，控制器对着同一块缓冲循环传，不必每次重编；驱动的活变成“在公共缓冲与用户缓冲之间倒数据”。为什么需要一块驱动自己的缓冲：自动初始化模式要求传输目标是一块固定地址的缓冲，不能直接对着每次都不同的用户缓冲——用户缓冲每 IRP 都变，控制器没法“自动”对着它循环。

**公共缓冲的建立四步（START_DEVICE 里做）**。① AllocateCommonBuffer（传适配器对象指针与长度；长度最好 ≤ PAGE_SIZE 或为 PAGE_SIZE 的整倍数，省内存也省寄存器）；② 若返回 NULL，释放已占资源、START_DEVICE 回 STATUS_INSUFFICIENT_RESOURCES；③ 成功则交回两个指针：LogicalAddress（存起来、此后忽略——系统 DMA 驱动不用它）与 VirtualAddress（存起来，用它建 MDL）；④ IoAllocateMdl 为缓冲建 MDL → MmBuildMdlForNonPagedPool 把缓冲虚拟地址范围映射到系统物理内存。为什么建 MDL：后续 MapTransfer 要的是 MDL，不是裸虚拟地址；公共缓冲也必须先变成 MDL 才能进 DMA 流程。

**公共缓冲的传输序列**。① 酌情 RtlMoveMemory 把用户缓冲数据拷进公共缓冲（往设备方向）；② AllocateAdapterChannel；③ MapTransfer（用公共缓冲的 MDL，只调一次——控制器已设好对着公共缓冲传）；④ 传输中用 ReadDmaCounter 查还剩多少字节没传完，酌情继续往公共缓冲填数据或往用户缓冲倒数据；⑤ FlushAdapterBuffers；⑥ FreeAdapterChannel。为什么 MapTransfer 只调一次：控制器在自动初始化模式下对着同一块缓冲循环，映射一次就够；驱动的循环在“倒数据”上，不在“重映射”上。AdapterControl 仍返回 KeepObject——通道与寄存器要留到整单传完。

### 详解七：总线主控 DMA 的两条路——包 DMA 与公共缓冲 DMA

**包总线主控 DMA 的判据**。设备能让驱动知道“传输何时结束、何时可以开始下一次”时用包 DMA。为什么这是判据：包 DMA 的生命周期与 IRP 对齐——驱动必须在传输结束时刷、释放、完成 IRP；若设备不告诉驱动何时结束，驱动就无法在正确时刻做这些事，包 DMA 的流程转不起来。

**包总线主控 DMA 的完整序列（v1/v2）**。KeFlushIoBuffers（分配映射寄存器之前）→ AllocateAdapterChannel（调用方 DISPATCH_LEVEL；AdapterControl 里 MmGetMdlVirtualAddress + MapTransfer）→ MapTransfer 返回逻辑地址（驱动必须用它编程总线主控适配器——与系统 DMA 的“忽略”相反）→ 设备传 → 每次传输结束 FlushAdapterBuffers → 全部传完或出错时 FreeMapRegisters（紧跟最后一次 FlushAdapterBuffers）。为什么释放例程不同：包总线主控的 AdapterControl 返回 DeallocateObjectKeepRegisters——通道已还，只剩寄存器；所以 DPC 还的是寄存器（FreeMapRegisters），不是通道（FreeAdapterChannel）。Win2K 起，MapTransfer 与 FlushAdapterBuffers 必须传非空适配器对象指针（NT4 之前允许 NULL，已废）。

**SG 能力的总线主控设备怎么传**。两种：① AdapterControl 里反复调 MapTransfer，用光所有可用映射寄存器，攒出一张散点/聚集表给设备；② 直接用 GetScatterGatherList / PutScatterGatherList（无论设备有没有 SG 能力都可）。为什么有两种：前者是老路（驱动自己攒表），后者是系统服务（系统替驱动攒表）；后者省事且可多排队，前者给驱动完全控制。SG 设备上 MapTransfer 的 Length 参数是输入输出双向的：入参是想传的字节数，出参是实际映射的字节数——与逻辑地址一起，界定本次操作设备可用的逻辑地址范围。红线：SG 设备上绝不许把 IRP 堆栈位置里 Length 的指针直接传给 MapTransfer——MapTransfer 会覆盖它，IRP 里的原始长度就毁了，驱动再也算不出“整单传完没有”。

**公共缓冲总线主控 DMA 的判据**。设备不能让驱动随时知道传输起止，或同一块缓冲要被连续、反复地传时用。典型例子：用邮箱机制传命令与状态的总线主控适配器——邮箱是驱动与适配器共享的固定缓冲，命令与状态一直在里面流转；此时驱动常混用：邮箱走公共缓冲，数据传输走包 DMA。为什么混用合理：邮箱需要“固定地址、随时可读写”的共享区，包 DMA 需要“按 IRP 打包、传完释放”的临时映射，两者生命周期不同，必须分开。

**公共缓冲总线主控 DMA 的建立与经济账**。START_DEVICE 里调 AllocateCommonBuffer（传适配器对象指针）；成功交回 VirtualAddress（存设备扩展/控制器扩展/非分页池）与 LogicalAddress（设备用）；失败（NULL）则要么只用包 DMA，要么 START_DEVICE 回 STATUS_INSUFFICIENT_RESOURCES。经济账：公共缓冲占映射寄存器——缓冲长度决定的页数（BYTES_TO_PAGES 算）不得超过 NumberOfMapRegisters；且公共缓冲会挤占包 DMA 的寄存器。所以指南要求省着用：PAGE_SIZE 块或单次分配，留更多寄存器给包 DMA、留更多内存给系统。CacheEnabled 参数注意：传 TRUE/FALSE 都行，操作系统说了算——x86/x64/Itanium 一律开缓存，ARM/ARM64 按每设备的 ACPI_CCA 方法决定该设备是否缓存一致。驱动假设“缓存一定开”或“一定关”都会错。设备停止或移除时（PnP 停/删 IRP），驱动必须调 FreeCommonBuffer 还每一块公共缓冲。

### 详解八：Version 3 DMA 接口——SoC 时代的简化

**v3 为什么存在**。v1/v2 把散点/聚集限制在总线主控设备，且驱动要手动跑“申请通道 → 建映射 → 刷缓存 → 释放”的全套。SoC 集成电路里的系统 DMA 控制器能力更强（本身就能做散点/聚集），v3 接口就是为这些控制器准备的：让系统 DMA 也能做 SG，简化复杂传输的管理，减少传输中的驱动干预。适用范围：Win8 起可用，覆盖全部 Windows 硬件平台，但能力收益主要在 SoC 系统 DMA 控制器上。

**v3 的十步调用模式**。① IoGetDmaAdapter 拿适配器对象；② GetDmaTransferInfo 拿资源需求描述（输入：缓冲与方向；输出：映射寄存器数、SG 表大小——下一步 AllocateAdapterChannelEx 要把寄存器数当入参）；③ AllocateAdapterChannelEx 分配通道与寄存器（可异步：不设 DMA_SYNCHRONOUS_CALLBACK，ExecutionRoutine 在资源就绪时被调，调用本身立即回 STATUS_SUCCESS；可同步：设标志，资源能立即分配则调 ExecutionRoutine（若有）后回成功，不能立即分配则回 STATUS_INSUFFICIENT_RESOURCES；ExecutionRoutine 为空时 MapRegisterBase 入参必须非空）；④ 必要时 CancelAdapterChannel 取消挂起的资源请求（回 TRUE 则取消成功、ExecutionRoutine 不跑；回 FALSE 则请求已批准、ExecutionRoutine 会跑）；⑤ MapTransferEx 建映射并启动传输（可在调 AllocateAdapterChannelEx 的线程里，也可在 ExecutionRoutine 里；一次映射不完时，下一次 MapTransferEx 可在上一次的完成例程里调）；⑥ 必要时做硬件相关操作（某些平台传输启动有延迟，但驱动不许依赖延迟做资源决策）；⑦ 等传输完成通知（总线主控：设备中断；系统 DMA：MapTransferEx 的 DmaCompletionRoutine 入参——系统 DMA 控制器中断后，该例程被调度在 DISPATCH_LEVEL 运行）；⑧ 每次 MapTransferEx 后调 FlushAdapterBuffersEx；⑨ 全部传完调 FreeAdapterChannel；⑩ 调 PutDmaAdapter 还适配器对象。转移方式说明：步骤 ⑤→⑦ 的通知，系统 DMA 是“中断向量 → 调度完成例程（DISPATCH_LEVEL）”，总线主控是“设备中断 → ISR → DPC”那条第 8 批已讲的路；步骤 ③→⑤ 的 ExecutionRoutine 是资源就绪时 I/O 管理器以函数调用方式调入（同步模式下就在调用线程里）。

**MapTransferEx 相对 MapTransfer 的改进**。MapTransfer 一次只能映射 MDL 里一连续物理块；复杂传输的缓冲若是 MDL 链（多段虚拟连续区，每段又含多段物理连续块），用 MapTransfer 得双重循环（内层：每 MDL 内物理块；外层：MDL 链）。MapTransferEx 一次可处理整条链：入参 Mdl（链首）、Offset（缓冲起始字节偏移）、Length（指针，入参为待映射字节数）；它从 Offset 处起步，沿链建 SG 表（每片段是一物理连续块，顺序与 MDL 链一致），直到表描述的字节数等于 Length 入参。映射不完时：Length 出参 = 实际映射字节数（小于入参），驱动按 Offset += X、Length -= X 续传（同一条 MDL 链不改）。若指定了 DmaCompletionRoutine，MapTransferEx 先写 Length 出参、再调度完成例程——保证完成例程跑时一定能读到更新后的 Length（Length 指针通常指在传给完成例程的 CompletionContext 里）。驱动必须按“一次映射不完”来设计：映射不完的原因包括寄存器不够、驱动给 SG 表留的存储不够、系统 DMA 控制器的硬件 SG 表有片段上限，未来平台还可能加新限制。

**v3 的缓存规矩为什么变了**。v3 里 KeFlushIoBuffers 传输前后都不需要：MapTransferEx 在写（内存→设备）传输前刷处理器缓存，FlushAdapterBuffersEx 在读（设备→内存）传输后使缓存失效。为什么能内建：v3 例程自己知道传输方向与缓存操作点，不必驱动在正确时刻手动调。后果：x86/x64 上 KeFlushIoBuffers 本来就是空操作，多调无害但无用；ARM 上多调会做不必要的缓存操作、降性能——v3 驱动不该再调它。v1/v2 驱动升级到 v3 时，必须把手动刷缓存的调用拿掉，不是“留着也无妨”。

### 详解九：PIO 技术对照——DMA 不在时 CPU 怎么搬

**PIO 为什么被 DMA 取代**。PIO 是 CPU 等数据传完，传输期间 CPU 干不了别的；DMA 把传输交给控制器，CPU 干别的活。为什么 PIO 还在指南里：低速、小数据、无 DMA 能力的设备仍用 PIO；且 Direct I/O 的 PIO 路（第 7 批：双重映射）与 DMA 路共享 MDL 基础设施，理解 PIO 是理解“DMA 解决什么问题”的对照。

**PIO 唯一的缓存纪律**。PIO 读操作结束时调 KeFlushIoBuffers。为什么只讲读、不讲写：PIO 读是 CPU 把设备数据读进内存，读完时处理器缓存里可能还留着旧数据，驱动随后访问内存会读到缓存旧值；PIO 写是 CPU 把内存数据写进设备寄存器，写路径不经过“DMA 读内存”那个不一致点。为什么在一致性平台上仍要求调：同详解三——空操作、无开销，非一致性平台上是 correctness 保障。

**PIO 与 DMA 在本批里的位置**。PIO 技术一节只有两页，且 PIO 双重映射的完整机制已在第 7 批讲过（MmGetSystemAddressForMdlSafe、AliasBuff、系统页表项代价）。本批不重讲双重映射，只钉住两点交接：① PIO 是 Direct I/O 之下与 DMA 并列的搬运手段，不是与 Direct 并列的“方式”；② PIO 读的缓存刷在结束时，DMA 的处理器缓存刷在传输前——时序相反，不要记混。

### 详解十：控制器对象——多设备共享一块控制器的串行化工具

**控制器对象到底是什么**。一块物理控制器（如 AT 磁盘控制器）挂多个同类设备时，驱动为控制器建的对象；它没有名字、不是 I/O 目标，用户态子系统与高层驱动都看不见它、不能给它发 IRP；I/O 请求永远发给设备对象，控制器对象只供驱动自己用来串行化。为什么要有：同一时刻控制器只能给一个设备干活，“谁在用控制器”必须有个排队机制；没有它，驱动得自己维护 Busy 标志、重排 IRP。

**为什么 WDM 驱动不能用**。指南明言两处：控制器对象不被 WDM 驱动支持；WDM 驱动不能用控制器对象与 ControllerControl 例程。为什么：WDM 的模型是分层（总线/功能/筛选）+ PnP 状态机，控制器与设备的串行化由端口/类驱动分工与 PnP 锁承担；控制器对象是单体（非 WDM）驱动时代的工具。现代驱动里出现控制器对象，要么是老式单体驱动，要么是误用——审计时这是个年代标记。

**适用条件（三条，缺一不可）**。① 控制器不做长操作不中断——否则驱动得另起设备专属线程或系统工作线程，控制器对象排队会拖垮吞吐；② 所挂设备是同类的——键盘与鼠标挂同一键盘/辅助设备控制器那种“物理属性与操作功能完全不同”的组合不适用；③ 驱动是单体的——相对控制器与所挂设备是单层，不是“控制器端口驱动 + 所挂设备类驱动”的分层。另有一类也可用：带 I/O 通道与一组逻辑设备对象的设备，用控制器对象串行化各通道间的 I/O。为什么条件这么严：控制器对象只解决“串行化”，不解决“长操作调度”与“异类设备抽象”；条件之外的场景用它，等于用错工具。

**不用控制器对象的土办法为什么差**。土办法：给每个目标设备建命名设备对象 → 维护 Busy 标志（各设备扩展或统一扩展）→ 只给当前忙设备做 I/O、其他设备的 IRP 重排，直到当前 IRP 完成。为什么差：它强制“当前 IRP 完成前 StartIo 不能取下一个 IRP”，串行化过了头——即使某些操作可以重叠（一个盘 seek 的同时另一个盘读写），土办法也不允许。控制器对象的价值正在于可重叠：驱动在 ControllerControl 里判断“这次能不能与上次重叠”，能重叠就返回 DeallocateObject、立即接下一个操作。反过来，全双工串口控制器、总线主控适配器这类天生重叠的设备，驱动通常直接用内部 IRP 队列，不用控制器对象——因为重叠是常态，不是例外，控制器对象的“一次一操作”模型反而碍事。

**建立三步（START_DEVICE 里做）**。① IoCreateController（传驱动定的控制器扩展大小；系统从非分页池分配并清零；返回 NULL 则启动失败、回 STATUS_INSUFFICIENT_RESOURCES）；② 保存控制器对象指针（通常存每个设备对象的设备扩展里——因为每个目标设备的例程都要凭它取控制器）；③ 初始化控制器扩展内容（常放：当前目标设备对象/扩展指针、中断对象指针——若控制器产生中断）。为什么指针要存每个设备扩展：控制器对象是共享的，但发起 I/O 的是各个设备对象；每个设备对象的例程都要能从自己扩展里找到控制器对象，再经 ControllerObject->ControllerExtension 访问控制器状态。

**申请与归还（IoAllocateController / IoFreeController）**。驱动在 IRP 需要编程物理控制器时调 IoAllocateController（调用方须在 DISPATCH_LEVEL；StartIo 里调天然满足），传：控制器对象指针、目标设备对象指针、ControllerControl 例程入口、Context。控制器忙时 ControllerControl 被排队，空闲时立即以函数调用方式调入（传 Context；有 StartIo 的驱动，I/O 管理器还会传 DeviceObject->CurrentIrp；无 StartIo 的驱动须把当前 IRP 放进 Context）。Context 存放纪律：控制器扩展里不能放“每 IRP”的上下文——除非驱动每 IRP 做完才开始下一个控制器操作，否则新 IRP 的上下文会覆盖旧的；目标设备对象的设备扩展里可以放（即使重叠，另一设备的 IRP 也不会覆盖本设备的扩展；有 StartIo 时，下一 IRP 在 IoStartPacket 里排着、不到 IoStartNextPacket 不会进 StartIo，所以同一设备的扩展也不会被覆盖）。ControllerControl 返回 DeallocateObject（能再起一个操作，可重叠——如给一个盘编好 seek 就能立即给另一个盘编传输）或 KeepObject（还需后续处理——如传输没完，需 ISR + DPC 完成）。KeepObject 时，完成 IRP 的例程必须调 IoFreeController，且越快越好（否则下一个控制器操作迟迟排不上）。默认倾向：除重叠操作外的大多数中断驱动 I/O，ControllerControl 应返回 KeepObject（DpcForIsr / CustomDpc 完成操作与 IRP）。

**ControllerControl 的职责清单**。① 更新设备扩展与控制器扩展里的上下文；② 若用 DMA：判断本次传输是否需拆（系统或设备限制），并调 AllocateAdapterChannel（若有 AdapterControl）；若用 PIO：按硬件需要拆部分传输范围，并调 MmGetSystemAddressForMdlSafe（用 Irp->MdlAddress 的 MDL）；③ 编程硬件（若设备/控制器扩展可被 ISR 访问，必须经 KeSynchronizeExecution 调 SynchCritSection——第 8 批的临界区纪律）；④ 若驱动有 Cancel 例程，先查 Irp->Cancel：已置位则状态块填 STATUS_CANCELLED / Information 0 → IoFreeController → IoStartNextPacket（或自管队列时取下一个）→ IoCompleteRequest 完成；未置位则 IoSetCancelRoutine 置 NULL（用 I/O 管理器设备队列时须拿取消自旋锁）→ 再编程硬件。完成 IRP 的例程随后调 IoFreeController + IoStartNextPacket，越快越好。

### 详解十一：设备配置空间访问——IRQL 决定手段

**为什么正常驱动不需要查配置空间**。配置空间含设备描述与资源需求，但 Win2K 起总线归总线驱动管，PnP 管理器在 IRP_MN_START_DEVICE 里已经把资源交给驱动了。写得好的驱动根本不需要查配置空间——查它通常是“驱动想知道 PnP 没给的东西”，属于例外。NT4 的 HalGetBusData / HalGetBusDataByOffset（驱动自己扫总线、调 HAL 查）在 Win2K 起已废：硬件总线由各自的总线驱动控制，不再由 HAL 控制。

**PASSIVE_LEVEL：走 IRP**。用 IRP_MN_READ_CONFIG / IRP_MN_WRITE_CONFIG：IoBuildSynchronousFsdRequest 建同步 PnP IRP（传事件与状态块），下一层位置填 MinorFunction（读/写）、Parameters.ReadWriteConfig（WhichSpace = PCI_WHICHSPACE_CONFIG、Buffer、Offset、Length），IoCallDriver 下发；返回 STATUS_PENDING 时 KeWaitForSingleObject 等事件，醒来取状态块。为什么必须 PASSIVE_LEVEL：发 IRP、等事件都要求 PASSIVE_LEVEL；DISPATCH_LEVEL 下既不能等，也不能保证 IRP 能走完。示例代码里先把 Irp->IoStatus.Status 预置为 STATUS_NOT_SUPPORTED——防止总线驱动没填状态时驱动误读成功。为什么必须在同一驱动栈：读配置空间要经目标设备的 PDO 发 PnP 请求，只有同栈的驱动（功能驱动或筛选驱动）才有 PDO 可用。

**DISPATCH_LEVEL：走总线接口**。DISPATCH_LEVEL 下不能发 IRP，必须用直接调总线驱动例程的办法：① 在 PASSIVE_LEVEL 先发 IRP_MN_QUERY_INTERFACE（InterfaceType = GUID_BUS_INTERFACE_STANDARD）拿到 BUS_INTERFACE_STANDARD 结构，存非分页池（通常设备扩展）；② 在 DISPATCH_LEVEL 调接口里的 GetBusData / SetBusData 读写配置空间；③ 用完解引用（总线驱动返回接口时已计引用，访问方必须解一次；解引用后不许再调接口例程）。为什么接口比 IRP 好：接口自带与总线驱动的硬件访问同步——驱动不必自己为争总线硬件拿自旋锁；且用 BUS_INTERFACE_STANDARD 不需要总线号（总线专属接口常要总线号，而 PCI 总线号可动态变化）。红线：不要按总线号直接访问 PCI 端口——总线号会变，依赖它可能导致系统失败。

**只要总线/设备/功能号时的捷径**。不必拿总线接口：把目标设备的 PDO 传给 IoGetDeviceProperty，问 DevicePropertyBusNumber（总线号，警告：动态、可变）与 DevicePropertyAddress（PCI：高字是设备号、低字是功能号）。为什么这是捷径：它不碰配置空间、不拿接口，只问 PnP 管理器已经知道的枚举信息。

**别的驱动栈的设备怎么办**。Win2K 起不允许驱动访问别的驱动栈的硬件（如 PCI-to-PCI 桥：系统为每座桥建 PDO，但不给桥注册设备接口，所以设备接口机制够不着它）。怎么办：写一个筛选驱动挂到桥的驱动栈上（INF 里写桥的硬件 ID，让 PnP 管理器在检测到桥时加载筛选驱动；或用 SetupDiXxx 在协同安装程序里编程安装），筛选驱动再用 BUS_INTERFACE_STANDARD 访问桥。为什么必须挂筛选驱动：访问权限跟着驱动栈走，没有同栈身份就没有访问路径；NT4 的 HalGetBusData 旁路在 Win2K 起已被堵死。

---

## 四、解读（审计与逆向视角）

1. **映射寄存器数是设备的指纹**。IoGetDmaAdapter 交回的 NumberOfMapRegisters 暴露了 HAL 与设备的硬约束。逆向时，驱动若不检查返回值、直接假设“MaximumLength 一次能传完”，就是拆传输 bug 的候选——大缓冲 IRP 一来就超界。审计时先找驱动保存 NumberOfMapRegisters 的位置（设备扩展），再看每次 MapTransfer 前有没有按 PAGE_SIZE × 寄存器数算上限。
2. **DMA 是物理内存的直通口，IOCTL 是它的守门人**。设备拿到映射后可以读写映射范围内的任何物理内存；若驱动的 IOCTL 路径允许用户态指定“物理地址 + 长度”去编程 DMA（或指定任意 MDL），等于把任意物理内存读写原语交给用户态——这是 BYOVD 的经典命门之一（与第 5 批“默认 SD 极宽”、第 7 批“IOCTL 分级检查表”合起来，是同一审计面的第三块）。审计时沿着 IOCTL → 传输编程的路径查：地址与长度有没有校验、有没有限制在驱动自己的缓冲内。
3. **释放配对是资源泄漏的检查表**。每次传输结束必须成对出现：FlushAdapterBuffers（每次 MapTransfer 后）→ FreeAdapterChannel（系统 DMA）或 FreeMapRegisters（包总线主控）；散点/聚集路是 PutScatterGatherList；公共缓冲路是 FreeCommonBuffer（停/删设备时）。出错路径（设备 I/O 错、IRP 失败）最容易漏还——漏还的后果不是立刻蓝屏，而是通道/寄存器被占着，别的设备饿死、系统吞吐慢慢垮。逆向时看 DPC 的收尾段：有没有“出错也还”的分支。
4. **公共缓冲是驱动与设备的共享内存区**。邮箱机制的公共缓冲里流转的是命令与状态——逆向时它是理解驱动与设备协议的捷径（看谁往邮箱里写什么、谁读什么）；审计时它是竞争面（驱动与设备同时读写同一块缓冲，同步靠什么？缓存开了还是关了？ACPI_CCA 在 ARM 上说了算，驱动假设错了就是时序 bug）。
5. **缓存刷漏了在 x86 上测不出来**。KeFlushIoBuffers / FlushAdapterBuffers 在硬件一致性平台上是空操作——驱动漏刷，在 x86 测试机上跑得好好的，到 ARM/SoC 上就数据损坏。审计时不要以“测试通过”论，要以“调用序列有没有”论；v1/v2 驱动升级到 v3 时反过来查：手动刷有没有拿掉（ARM 上多调降性能）。
6. **控制器对象是年代标记**。现代 WDM/KMDF 驱动里出现 IoCreateController / IoAllocateController，先问一句“为什么不用端口/类分工与 PnP 锁”——要么是老式单体驱动的合理遗产，要么是误用。KeepObject / DeallocateObject 选错的症状很典型：该还的没还，后续 I/O 全卡在控制器排队里（症状像死锁，其实是资源没归还）。
7. **配置空间访问方式暴露驱动年代与 IRQL 意识**。还在调 HalGetBusData 的是 NT4 写法，Win2K 起已废；DISPATCH_LEVEL 下发 IRP 读配置空间的是 IRQL 违规；按总线号直接访问 PCI 端口的是脆弱写法（总线号动态变化）。三者都是“驱动没跟上模型”的信号。
8. **本批指南的边界（不要外推）**。2019 版指南讲的是 v1/v2/v3 接口与 HAL 映射寄存器抽象，不讲 IOMMU / DMA 重映射 / 基于虚拟化的 DMA 保护（如 Windows 的 DMA 保护、内核 DMA 保护）。现代机器上设备 DMA 还要过 IOMMU 这一关，但那是本批底本之外的机制——审计现代驱动时，IOMMU 在场与否会改变“设备能读写多大范围物理内存”的结论，本批的结论只覆盖“映射寄存器给设备什么地址”这一层。

## 五、易混点辨析

1. **适配器对象 ≠ DMA 控制器硬件**。适配器对象是 HAL 提供的软件抽象，代表通道、端口或总线主控设备；映射寄存器集可能是真实硬件寄存器，也可能是 HAL 虚拟寄存器。驱动只认对象指针与句柄，不碰本体。
2. **逻辑地址 ≠ 物理地址 ≠ 虚拟地址**。三者各有翻译层（映射寄存器、页表/MDL）；系统 DMA 的 MapTransfer 返回逻辑地址必须忽略（控制器用它），总线主控的 MapTransfer 返回逻辑地址必须用（驱动编程设备）。同一返回值、两边处置相反。
3. **KeepObject / DeallocateObjectKeepRegisters / DeallocateObject 不是同义词**。KeepObject：留通道 + 留寄存器（系统 DMA）；DeallocateObjectKeepRegisters：还通道、留寄存器（包总线主控，DPC 随后 FreeMapRegisters）；DeallocateObject：全还、可重叠（控制器对象）。前两个是 AdapterControl 的，第三个是 ControllerControl 的，不要串。
4. **FreeAdapterChannel ≠ FreeMapRegisters**。前者还通道 + 寄存器（系统 DMA 的收尾）；后者只还寄存器（包总线主控的收尾，通道已在 AdapterControl 返回时还掉）。用错等于账对不上。
5. **包 DMA ≠ 公共缓冲 DMA**。包 DMA 按 IRP 打包、传完释放；公共缓冲 DMA 一块缓冲长期持有、反复传。公共缓冲要省着用——它占映射寄存器，挤占包 DMA。
6. **散点/聚集表 ≠ 设备 SG 能力**。表描述的是缓冲的物理分段，系统可以替无 SG 能力的设备建表（此时表只有 1 个元素）。“我的设备没 SG 所以不能用 GetScatterGatherList”是误解。
7. **MapTransfer ≠ MapTransferEx**。前者一次一连续物理块，MDL 链要双重循环；后者一次整条 MDL 链，v3 才有，且缓存处理内建。v3 驱动不应再手动 KeFlushIoBuffers。
8. **MmGetMdlVirtualAddress 的返回值 ≠ 可访存地址**。它只是 MDL 索引，只能传给 MapTransfer；想用 CPU 访存，走 MmGetSystemAddressForMdlSafe 双重映射（第 7 批）。
9. **配置空间访问的手段由 IRQL 定**。PASSIVE_LEVEL 走 IRP（READ_CONFIG / WRITE_CONFIG）；DISPATCH_LEVEL 走总线接口（QUERY_INTERFACE 拿 BUS_INTERFACE_STANDARD，再调 GetBusData / SetBusData）。HalGetBusData 是 NT4 遗物，Win2K 起已废。
10. **控制器对象 ≠ 设备对象**。控制器对象无名、不是 I/O 目标、WDM 不能用；它只是驱动内部串行化多设备的工具。IRP 永远发给设备对象。
11. **“一次传完”不是默认**。映射寄存器数、Length、跨页数、设备限制四者任一超标就得拆；驱动假设“一次能传完”而不算 ADDRESS_AND_SIZE_TO_SPAN_PAGES，是拆传输 bug 的标准起手式。
12. **SG 设备上 Length 是出参**。MapTransfer 会覆盖 Length 入参指向的值；把 IRP 堆栈位置里 Length 的指针直接传进去，会毁掉 IRP 的原始长度，导致驱动算不出整单传完没有。

## 六、自查问题

1. 一段缓冲跨 12 页、适配器对象只有 5 个映射寄存器、无设备限制时，驱动必须做几次 DMA 操作？每次 MapTransfer 的 CurrentVa 与 Length 按什么公式更新？
2. 系统 DMA 与包总线主控 DMA 的 AdapterControl 返回值各是什么？DPC 收尾时前者调什么释放、后者调什么释放？为什么不同？
3. 为什么散点/聚集 DMA 的设备可以没有 SG 能力？GetScatterGatherList 相对 AllocateAdapterChannel 多出哪两个好处？
4. v1/v2 接口里，处理器缓存与控制器内缓冲各用什么刷、分别在什么时序？v3 接口为什么不需要 KeFlushIoBuffers？
5. 驱动在 DISPATCH_LEVEL 需要读 PCI 配置空间时，正确的三步是什么？为什么不能直接发 IRP，也不能按总线号直接访问 PCI 端口？

## 未钉死清单

1. **AdapterControl 例程自身的 IRQL**：本批底本明言调用 AllocateAdapterChannel 须在 DISPATCH_LEVEL、AdapterListControl 在 DISPATCH_LEVEL 任意线程上下文运行，但 AdapterControl 自身的 IRQL 在底本正文里未明言（DDI 口径为 DISPATCH_LEVEL，交付时未再查 DDI，列入待查，不写成定论）。
2. **散点/聚集节的一句歧义**：底本在 AdapterListControl 返回后的资源处置上有一句表述含糊（涉“map registers 保留”与另一资源的释放），本批未采用该句，只采用无歧义的部分（PutScatterGatherList 刷缓冲、还寄存器、释表）；该句以 DDI 为准，待查。
3. **DMA_OPERATIONS 各版本例程全表**：底本正文只列出本批用到的例程（v1/v2 的 AllocateAdapterChannel / MapTransfer / FlushAdapterBuffers / FreeAdapterChannel / FreeMapRegisters / GetScatterGatherList / PutScatterGatherList / AllocateCommonBuffer / FreeCommonBuffer / ReadDmaCounter，v3 的 GetDmaTransferInfo / AllocateAdapterChannelEx / CancelAdapterChannel / MapTransferEx / FlushAdapterBuffersEx），各版本函数表的完整成员以 DDI 参考集为准。
4. **HAL 映射寄存器分配算法的平台细节**：底本只给策略（尽量给 MaximumLength 所需数 + 1，否则给平台尽可能大的值），各平台具体数与分配算法不透明，驱动只能以返回值为准。
5. **ARM/ARM64 的 ACPI_CCA 判定细节**：底本只说明系统按每设备的 ACPI_CCA 方法决定公共缓冲是否开缓存，ACPI_CCA 的具体语义与设备端配合不在底本内。
6. **DEVICE_DESCRIPTION 全部字段语义**：底本只讲清与本批相关的字段（Master、ScatterGather、MaximumLength、BusNumber 不用），其余字段以 DDI 为准。

---

## 附：追问补记——逻辑地址与映射寄存器怎么翻译（2026-10-29）

- **对位**：逻辑地址 = 设备在总线上发出的地址，是“设备版的虚拟地址”。三地址各有使用者与翻译者：虚拟地址（CPU 用，页表翻译，进程页表描述）、物理地址（内存条用，不翻译）、逻辑地址（设备用，映射寄存器翻译，MDL 描述源页）。类比：虚拟地址 : 页表 = 逻辑地址 : 映射寄存器，同一思想换使用者。
- **为什么不让设备直接用物理地址**：三个问题——① 位宽：老设备地址位宽不够（32 位/24 位 DMA），够不到高地址物理页；② 连续性：驱动缓冲区虚拟连续、物理散开，不支持散点/聚集的设备想看到一段连续地址；③ 安全：设备拿到物理地址 = 全内存通行证，故障或恶意设备能读写任意内存。逻辑地址 + 映射寄存器把三件事收口：够不到的高页用弹跳缓冲落到低地址，散页拼成逻辑连续，未映射的页设备碰不到。
- **映射寄存器怎么翻译（槽位模型）**：映射寄存器是 HAL 管理的一组按页的翻译槽位，每槽记一对（逻辑页号 → 物理页框号 PFN），每槽至多一整页、最少 1 字节。MapTransfer 时系统从池里拿 N 个连续逻辑槽位（N = 缓冲跨越的页数），把 MDL 的 N 个 PFN 依次填入，返回这段槽位的起始逻辑地址。设备访问逻辑地址 L：以 L 的页号查槽位得 PFN，页内偏移（低 12 位）原样透传，拼成物理地址。三种翻译共同的骨架都是“页号查表换、偏移原样过”。
- **寄存器不够就拆**：只分到 2 个槽位就只映射 2 页，MapTransfer 的 Length 出参告诉实际映射量，驱动传完这段后在 DPC 里推进 CurrentVa 续映射——这是拆传输与 CurrentVa/Length 更新公式的由来。
- **三种平台落地**：① 恒等映射：设备地址能力覆盖全部物理内存时，逻辑地址 = 物理地址，映射寄存器退化为记账（供 FlushAdapterBuffers 收尾）；② 弹跳缓冲：设备够不到高页时，HAL 分配设备够得到的低端连续内存中转，写方向先拷入、设备从中转处 DMA，读方向设备先 DMA 进中转、FlushAdapterBuffers 时拷回真缓冲——这是 FlushAdapterBuffers 不能省的原因之一；③ IOMMU（Intel VT-d / AMD-Vi / ARM SMMU）：映射寄存器 = IOMMU 页表项，设备发逻辑地址（IOVA），IOMMU 查 I/O 页表得物理地址，未映射的 IOVA 直接报错，设备碰不到未授权内存（DMA 隔离）。
