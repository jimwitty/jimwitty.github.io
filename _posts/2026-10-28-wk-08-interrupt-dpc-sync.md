---
layout: post
title: "Win 内核驱动 第 8 批：中断、DPC 与同步"
categories: windows-kernel
series_order: 8
---

- **系列**：Windows 内核驱动循序渐进学习（教材：微软官方《内核模式驱动程序体系结构设计指南》，WDM 中心）
- **本批对应原文档**：「Interrupt Service Routines」章（Introduction to ISRs / Registering an ISR / 三种 IoConnectInterruptEx 版本 / Passive-Level ISR / IoConnectInterruptEx Prior to Vista / Removing an ISR / Making an ISR Active or Inactive / Interrupt Affinity / Providing ISR Context Information / Writing an ISR / Synchronizing Access to Device Data / Using Critical Sections）+ 「Message-Signaled Interrupts」章（Introduction / Enabling in the Registry / Using Interrupt Resource Descriptors / Dynamically Configuring MSI-X）+ 「Deferred Procedure Calls」章（Introduction to DPC Objects / Introduction to DPCs / Which Type / Registering and Queuing a DpcForIsr / Registering and Queuing a CustomDpc / Handling Overlapped I/O Operations / Writing DPC Routines / Guidelines / Organization of DPC Queues / Introduction to Threaded DPCs / Synchronization and Threaded DPCs / Converting an Ordinary DPC to a Threaded DPC）+ 「Synchronization」章中与本批直接相关的部分（Introduction to Kernel Dispatcher Objects / Timer Objects / KeXxxTimer / CustomTimerDpc 三节 / ExXxxTimer / Event Objects / Semaphore Objects / Mutex Objects / Alternatives to Mutex Objects / Fast Mutexes and Guarded Mutexes / Spin Locks 全节 / Introduction to Thread Objects 与 System Worker Threads 中工作项部分）+ 「Managing Hardware Priorities」+ 「Dispatch Routines and IRQLs」。
- **取材说明**：正文取自微软官方同名设计指南 PDF（2019 版离线打包）上述章节逐节研读。本 PDF 的来源仓库 tpn/pdfs 是第三方文档收集仓库、非微软官方；指南本身是微软官方文档 2019 版离线版。本批是第 6、7 批的下半程：第 6 批讲“请求怎么流动”，第 7 批讲“请求怎么停留与打断”，本批讲“设备怎么打断 CPU（中断）、打断之后怎么善后（DPC）、以及所有例程在不同 IRQL 下靠什么同步”。DMA 适配器对象、控制器对象的深水区属第 9 批，本批只讲到 DPC 与它们交接的那一步，不越界；PnP 资源分配全流程属第 10 批，本批只讲中断资源描述符与亲和性策略的交接点。
- **学习目标**：学完本批，你应该能说清：① ISR 到底干什么、不干什么，为什么必须越快越好；② 三种 IoConnectInterruptEx 注册方式各在什么场景用，MSI/MSI-X 与线中断的根本区别；③ DPC 单例规则与重叠 I/O 计数为什么是竞态 bug 的模板；④ 线程化 DPC 的“PASSIVE_LEVEL 但仍不能阻塞”为什么反直觉；⑤ IRQL 纪律总表与三条致命线；⑥ 分发器对象与自旋锁的分工——什么场景用等、什么场景用锁；⑦ 自旋锁家族（普通/排队/读写/中断）怎么选，嵌套与递归的死锁红线。

---

## 术语表（按引用级排序）

表头说明：基础概念在前、派生概念在后；读到某条时它引用的词前面都已定义。互为前提的一对已按惯例处理：**分发器对象**条给行内解释（事件、信号量、互斥体、定时器是它的四种具体类型）并指到后面条目。

- **IRQL（中断请求级别）**：CPU 上屏蔽中断的优先级标尺。数值越大优先级越高；当前 IRQL 为 N 时，本核上级别 ≤ N 的中断只挂起不投递。为什么要有：没有它，任何中断随时打断任何代码，内核无法保证“一段代码在执行时不被同级或低级事件打断”；有了它，驱动可以用“抬级别”换“一段不受打断的时间”。本批四档：PASSIVE_LEVEL（不屏蔽）、APC_LEVEL（屏蔽 APC）、DISPATCH_LEVEL（屏蔽分发与 APC）、DIRQL（屏蔽 ≤ 该设备级别的中断）。
- **APC（异步过程调用）**：内核投递给线程的一段“顺带执行的代码”。内核 APC（如 I/O 完成）与用户 APC 是两类；APC_LEVEL 屏蔽的是 APC 投递。为什么要有：让“某事完成后顺带在目标线程上下文里再干点事”成为可能，而不必专门唤醒一个线程。
- **DIRQL（设备 IRQL）**：某个设备中断向量对应的 IRQL。ISR 与 SynchCritSection 就跑在这个级别。为什么要有：不同设备的中断要有优先级之分，高速设备的中断可以打断低速设备的处理，反之不行。
- **中断**：设备通知 CPU“有事了”的机制。两种：线中断（line-based，设备用一根专用引脚发电信号，Vista 之前只有这一种）；消息中断（message-signaled，设备向特定地址写一个值，无专用引脚）。为什么要有：没有中断，CPU 只能轮询设备状态，白白烧时间；有了中断，CPU 干别的活，设备有事才打断它。
- **PDO（物理设备对象）**：总线驱动为一个物理设备创建的设备对象（第 3 批已定义）。本批里它的作用是注册 ISR 时的锚点：系统用 PDO 自动识别设备的线中断或消息中断；PnP 资源描述符也挂在它上面。
- **INF（安装信息文件）**：驱动安装时执行的安装脚本文件。本批里它的作用是写注册表：MSI 开关（MSISupported）、消息数上限（MessageNumberLimit）、中断亲和性策略（DevicePolicy）。
- **PnP 管理器（即插即用管理器）**：负责枚举设备、分配硬件资源的管理器。本批的交接点只有两个：发 IRP_MN_FILTER_RESOURCE_REQUIREMENTS 让驱动改中断资源需求（两遍：先给意向清单，驱动可改；再发 IRP_MN_START_DEVICE 给正式清单），以及配不齐消息数时的降级。
- **ISR（中断服务例程）**：设备中断时系统调的驱动例程。两种：InterruptService（线中断与消息中断均可，系统传驱动给的上下文值）；InterruptMessageService（只用于消息中断，多传一个消息 ID）。为什么要有：中断来了总得有人第一时间响应——止住设备再中断、把现场搬进稳定内存、排 DPC 善后；这个人就是 ISR。
- **中断对象（KINTERRUPT）**：系统为每次 ISR 注册创建的不透明对象，内含该中断的 DIRQL 与自旋锁。驱动注册时拿到它的指针并保存，注销 ISR 时凭它交还。为什么要有：中断的级别与锁必须有个载体，驱动与系统凭它对上号。
- **中断自旋锁**：保护 ISR 与 SynchCritSection 共享数据的自旋锁。系统在调 ISR 之前拿住它，ISR 返回后放掉。为什么要有：ISR 与临界区例程可能在不同核上同时跑，共享数据必须有同一把锁；且这把锁与 DIRQL 绑定，拿锁即抬级别，缺一不可。
- **SynchronizeIrql（同步 IRQL）**：ISR 执行与临界区执行的级别。注册 ISR 时指定，通常等于设备 DIRQL，驱动可指定更高。为什么要有：临界区必须跑在“ISR 进不来”的级别上，否则同核 ISR 会打断临界区、破坏共享数据；SynchronizeIrql 就是这个“ISR 进不来”的门槛值。
- **SynchCritSection（同步临界区例程）**：驱动写的一段短代码，经 KeSynchronizeExecution 在中断临界区里执行。用途只有两类：编程设备（设寄存器）与访问共享状态。为什么要有：StartIo/DPC 想碰 ISR 也碰的数据时，不能自己拿锁（普通拿锁只抬到 DISPATCH_LEVEL，挡不住 DIRQL 的 ISR），必须请系统把自己抬到 SynchronizeIrql、拿住中断自旋锁后再执行。
- **DPC（延迟过程调用）**：ISR 排后、等 IRQL 降到 DISPATCH_LEVEL 以下时执行的例程。为什么要有：ISR 在 DIRQL 下能调的例程集合极小、且必须极快；真正干活（完成 IRP、启动下一个请求、重试、记日志）只能推迟到较低级别做，DPC 就是这个“推迟执行”的载体。
- **DPC 对象（KDPC）**：系统为每个 DPC 维护的不透明对象，驱动不直接碰其内容。每个设备对象系统自带一个 DPC 对象（给 DpcForIsr 用）；CustomDpc 的对象须驱动自己分配常驻内存。为什么要有：排队、去重（单例规则）、定核、定优先级，都要靠这个对象做账。
- **DpcForIsr**：系统为设备对象配的 DPC 例程。注册用 IoInitializeDpcRequest（AddDevice 或 START_DEVICE 处理里调），排队用 ISR 里的 IoRequestDpc。为什么要有：最常见的“一设备一善后”形态，系统包办对象分配，驱动只写例程。
- **CustomDpc**：驱动自建的 DPC 例程。注册用 KeInitializeDpc（驱动自备 DPC 对象存储），排队用 ISR 里的 KeInsertQueueDpc。为什么要有：一设备多善后（如串口的读/写/ purge / wait 四条活各一善后）时，一个 DpcForIsr 不够用；CustomDpc 不与设备对象绑定，上下文由驱动自己带。
- **IoTimer**：系统每秒调一次的驱动定时器例程。为什么要有：给驱动一个粗粒度的周期性检查点（如超时计数）；粒度是 1 秒，比 CustomTimerDpc 粗得多。
- **CustomTimerDpc**：定时器到期时跑的 DPC 例程。注册分两步：KeInitializeDpc 注册例程 + KeInitializeTimer（或 Ex 版）建定时器对象；KeSetTimer/KeSetTimerEx 设到期时间。为什么要有：驱动需要比 1 秒更细的定时（如 10 毫秒级超时、周期任务），IoTimer 给不了。
- **线程化 DPC**：在 PASSIVE_LEVEL 执行的 DPC（Vista 起，默认开启）。例程叫 CustomThreadedDpc，用 KeInitializeThreadedDpc 注册。为什么要有：普通 DPC 抢占所有线程、且不被任何线程抢占，DPC 一多或一长，全系统延迟飙升（音视频卡顿）；线程化 DPC 可被普通 DPC 抢占（但不被线程抢占），延迟友好。
- **MSI / MSI-X（消息 signaled 中断）**：PCI 2.2 起的线中断替代品。MSI：设备向一个地址写 16 位值，用低 4 位区分消息，最多 16 条；MSI-X（PCI 3.0）：每条消息有自己唯一的地址、32 位值，最多 2048 条，硬件表项可独立屏蔽、可动态改写。为什么要有：线中断要占引脚、要共享认领、边沿/电平语义麻烦；消息中断不用引脚、不共享、可扩展，MSI-X 还能把不同消息分到不同核。
- **亲和性（affinity）**：中断能由哪些处理器服务。KAFFINITY 是掩码类型（32 位系统 32 位、64 位系统 64 位，每组最多处理器数同此）；IRQ_DEVICE_POLICY 是策略枚举（本机默认/就近全部核/就近一核/全部核/指定核/消息分散到全部核）。为什么要有：多核下中断若全挤在一个核上，那个核被打断到死、别的核闲着；亲和性让中断分布可控。
- **分发器对象（dispatcher object）**：内核定义的一组可等待对象：定时器、事件、信号量、互斥体、线程对象（四种驱动常用类型见下面四条）。每个对象有 Signaled / Not-Signaled 两种状态；线程调 KeWaitForSingleObject / KeWaitForMutexObject / KeWaitForMultipleObjects 等它变 Signaled。为什么要有：PASSIVE_LEVEL 的线程需要一种“睡下去、条件到了再醒”的同步方式；自旋锁是“原地空转等”，分发器对象是“让出 CPU 等”。
- **事件（event）**：分发器对象的一种，点名通知用。同步事件（SynchronizationEvent）：放行一个等待线程后自动清零回 Not-Signaled；通知事件（NotificationEvent）：放行全部等待线程，状态保持 Signaled 直到显式清零（KeClearEvent / KeResetEvent）。为什么要有：驱动需要“一事发生、通知等它的人”时，事件是最轻的载体；同步/通知之分对应“只叫一个人”与“叫所有人”。
- **信号量（semaphore）**：分发器对象的一种，计数放行用。初始化给 Count（非零即 Signaled）与 Limit；Limit = 1 叫二值信号量（一次放一个），> 1 叫计数信号量（一次放 Limit 个以内）。为什么要有：事件只能“放行”，信号量还能“数数”——比如设备专用线程靠“Count 非零 = 队里有 IRP”知道何时有活干。
- **互斥体（mutex）**：分发器对象的一种，带所有权的独占锁。内核一次只把所有权给一个线程；可递归（拿几次就要放几次才变 Signaled）；持锁期间禁普通 APC 投递；持锁线程绝不许不放锁就回用户态（否则蓝屏）。为什么要有：二值信号量也能独占，但没有所有权概念——谁放都行，死锁了系统帮不上忙；互斥体的所有权让内核能在 SMP 上给持有线程做防死锁保护。
- **定时器（timer）**：分发器对象的一种，到期变 Signaled。KTIMER（KeXxxTimer，2000 起）：KeInitializeTimer 建通知定时器，KeInitializeTimerEx 可选通知/同步两种；KeSetTimer 只到期一次，KeSetTimerEx 可加 Period 做周期；DueTime 以 100 纳秒为单位，正值绝对时间、负值相对间隔。EX_TIMER（ExXxxTimer，Win8.1 起）：系统分配的不透明对象（ExAllocateTimer / ExDeleteTimer），另支持高精度与不唤醒（no-wake）两种类型。为什么要有：驱动需要“到时叫我”或“到时放行等它的线程”时，定时器是标准载体；到期还可顺带跑一个 CustomTimerDpc。
- **自旋锁（spin lock）**：内核态独占的忙等锁，不透明类型 KSPIN_LOCK。执行自旋锁（executive spin lock）：驱动自己拿放，拿住期间本核 IRQL 在 DISPATCH_LEVEL；中断自旋锁：与中断对象绑定，拿住期间本核 IRQL 在 DIRQL；排队自旋锁：多核高竞争下先到先得的变体；读写自旋锁：读共享、写独占的变体。为什么要有：DISPATCH_LEVEL 以上不能睡、不能等分发器对象，共享数据只能靠“原地空转等锁”保护；自旋锁就是这把锁。
- **快速互斥体 / 守卫互斥体（fast mutex / guarded mutex）**：互斥体对象的低开销替代品。快速互斥体（FAST_MUTEX，ExInitializeFastMutex）：持锁期间跑在 APC_LEVEL；守卫互斥体是直接替换品（Win8 起实现与快速互斥体相同，之前是“APC 全禁但 IRQL 不一定升到 APC_LEVEL”的更快版）。为什么要有：互斥体对象要走分发器等待路径，开销大；只在 APC_LEVEL 以下短临界区里互斥时，快速/守卫互斥体更便宜。代价：不可递归、不可用 KeWait 等它、不可与分发器对象同等。
- **任意线程上下文 / 非任意线程上下文**：驱动例程跑在“当前是哪个线程就是哪个”时叫任意线程上下文（多数例程，身份不可依赖）；跑在“身份可确定的线程”里时叫非任意线程上下文（DriverEntry / AddDevice / Reinitialize / Unload、最高层驱动的分发例程、驱动自建线程）。为什么要有这个区分：只有非任意线程上下文 + PASSIVE_LEVEL 才能等分发器对象——等下去的是一个身份确定的线程，醒来还能接着干；任意线程上下文里等，等于把无辜路过的线程扣住。
- **工作项（work item）**：丢给系统工作线程在 PASSIVE_LEVEL 跑的回调（IO_WORKITEM，IoAllocateWorkItem / IoQueueWorkItem，或 Vista 起的 WorkItemEx 版）。为什么要有：DPC 在 DISPATCH_LEVEL 不能阻塞、不能久跑；需要阻塞或长活的收尾只能委托给工作线程。
- **控制器对象（controller object）**：多设备共享一块硬件控制器时的同步对象（IoCreateController 建，IoAllocateController 取，IoFreeController 还）。本批只在 DPC 职责里出现（完成操作后要还控制器对象），深水区属第 9 批。
- **SMP（对称多处理）**：多个处理器共享内存、地位对等的机器结构。本批里它的意义是“并发是真实的”：ISR 与 DPC、两个例程可同时在不同核上跑，互斥不能只靠抬本核 IRQL，还要有锁本身挡别核。
- **FSD（文件系统驱动）**：实现某种文件系统格式的驱动（第 6、7 批已定义，本批沿用同一含义）。本批里它出现为互斥体的主要用户（FSD 的工作线程、执行线程）与自旋锁的用例（FSD 设备扩展里的互锁工作队列）。
- **DMA（直接内存访问）**：DMA 控制器直接读写物理内存的数据搬运方式，CPU 只编程（源、目的、长度），不参与逐字搬运。为什么要有：大数据传输若由 CPU 逐字搬，CPU 全程被占；DMA 让 CPU 干别的、传完中断通知。本批里它是 Direct I/O 之下与 PIO 并列的搬运手段。
- **PIO（程序控制 I/O）**：CPU 执行指令直接搬运数据的方式，一次一字/一字节（第 7 批已定义，本批沿用同一含义）。本批里它是 Direct I/O 之下与 DMA 并列的搬运手段，CPU 经双重映射的系统地址访存。
- **SCSI（小型计算机系统接口）**：一套存储设备接口标准与协议族。本批里它只出现为一条限制：SCSI miniport 驱动不许用分发器对象，只能调 SCSI Port 库例程。
- **I²C（Inter-Integrated Circuit）**：一种低速串行总线。本批里它只出现为被动级 ISR 的典型例子：I²C 传感器的寄存器只能经总线 I/O 请求访问、串行传输慢，无法在 DIRQL 的 ISR 里操作。
- **IDT（中断描述符表）**：CPU 查中断向量到处理例程入口的表。设备中断时，CPU 按向量号查 IDT 得处理例程地址并切栈压帧——本批“中断向量进 ISR”转移的第一步。
- **WDM（Windows 驱动程序模型）**：本系列教材对应的驱动模型（第 2 批已定义）。本批全部内容都在 WDM 模型内讲；KMDF 是它之上的框架，不在本指南内。
- **KMDF（内核模式驱动程序框架）**：建在 WDM 之上的驱动框架（本指南不含 KMDF，见路线说明）。本批里它只出现为一条限制：KMDF 明确不支持线程化 DPC。
- **DDI（设备驱动程序接口）**：驱动可调的内核支持例程的接口参考集（ddi/_kernel/，API 字典）。本批按指南总表写 IRQL 前置条件，逐例程的精确条件以 DDI 为准，见未钉死清单。
- **BYOVD（Bring Your Own Vulnerable Driver，自带漏洞驱动）**：攻击者利用合法签名但有漏洞的驱动进入内核的攻击手法（第 5 批已定义，本批沿用同一含义）。本批里它只出现为 IOCTL 审计的顺带检查项。
- **WDK（Windows 驱动程序开发工具包）**：微软的驱动开发工具集（含头文件、库与性能分析工具）。本批里它只出现为 Tracelog 测 DPC/ISR 时间的工具来源。
- **WRK（Windows Research Kernel）**：另一条源码学习线的底本（WS03 SP1 内核源码树）。本批里它只在未钉死清单出现一句：WRK 树里的数值口径与本批指南口径不同，不混用。

---

## 一、本批地图：中断从设备到完成的一生

```
设备中断
  │
  ├─ 转移形式：中断向量（IDT → 内核中断陷阱处理例程）
  │   系统先拿中断自旋锁 → 调 ISR（DIRQL，中断上下文，借用被中断代码的执行流）
  │
  ▼ ISR：只干急事，越快越好
  │   · 不是自己的中断 → 返回 FALSE（共享线中断的礼仪：让下一个 ISR 认领）
  │   · 是自己的 → 止住设备再中断 → 把现场搬进设备扩展（常驻内存）
  │     → 排 DPC（IoRequestDpc / KeInsertQueueDpc）→ 返回 TRUE
  │
  ├─ 转移形式：软件中断（IRQL 降到 DISPATCH_LEVEL 以下时，分发中断出 DPC）
  │   DPC（DISPATCH_LEVEL，任意处理器，DPC 上下文，非线程上下文）
  │     · 先 IoStartNextPacket —— 设备别闲着（第 7 批的顺序讲究）
  │     · 再填状态块、IoCompleteRequest —— 完成当前 IRP
  │     · 需要阻塞 / 长活 → 委托工作项（PASSIVE_LEVEL，系统工作线程上下文）
  │
  └─ 特殊分支：被动级 ISR（Win8+）
      陷阱处理例程先在中断控制器上止住/屏蔽中断
      → 调度 ISR 到 PASSIVE_LEVEL 跑（可同步发 I/O 请求、可阻塞等完成）

同步两条线（本批的骨架）：
  高 IRQL 线（ISR / SynchCritSection / StartIo / DPC）：自旋锁 + 临界区
    —— 不许等分发器对象、不许碰分页内存、不许久占
  PASSIVE_LEVEL 线（分发例程 / 自建线程 / 工作项）：分发器对象
    —— 事件、信号量、互斥体、定时器，可以等、可以睡
  两条线交接的唯一正规口子：KeSynchronizeExecution（抬 IRQL + 拿中断自旋锁，
    在同处理器上调 SynchCritSection）与 KeSetEvent（把等待线程变就绪，调度器换栈 + ret 唤醒）
```

---

## 二、核心要点

1. **ISR 三句话：只干急事、越快越好、可调例程集合最小**。止住设备再中断、把现场搬进设备扩展、排 DPC、返回——这就是 ISR 的全部。DIRQL 下同核上 ≤ 该级别的中断全被屏蔽，ISR 拖一秒，全机同级以下的设备都停一秒。ISR 里不做真正的 I/O 处理，那是 DPC 的活。
2. **TRUE / FALSE 是共享线中断的礼仪**。线中断可共享：多个设备的 ISR 挂同一向量，系统按序调，ISR 先判断“是不是我的设备”——不是立刻返回 FALSE，让下一个 ISR 认领；是则处理并返回 TRUE。MSI 是边沿语义、每条消息独立，不存在共享认领问题。
3. **注册三版本，按场景选**：CONNECT_LINE_BASED（Vista+，单 ISR 管全部线中断，系统自动识别）；CONNECT_MESSAGE_BASED（Vista+，单 InterruptMessageService 管全部消息，可带 FallBack 的 InterruptService 兜底纯线中断设备，注册后 Version 可能被改回 LINE_BASED）；CONNECT_FULLY_SPECIFIED（逐个中断指定，信息来自 START_DEVICE 的资源描述符，是 Vista 之前唯一能用的版本，老系统靠 Iointex.lib 链接）。
4. **中断临界区 = 抬 IRQL 到 SynchronizeIrql + 拿中断自旋锁**。普通拿锁（KeAcquireSpinLock）只抬到 DISPATCH_LEVEL，挡不住同核上 DIRQL 的 ISR；所以 ISR 共享的数据必须走 KeSynchronizeExecution + SynchCritSection，系统替你抬级别、拿锁、同处理器执行、返回后降级放锁。只拿锁不抬级别，等于门没关严。
5. **被动级 ISR 是 Win8+ 的例外口**。寄存器只能经 I/O 请求访问的设备（如 I²C 传感器，寄存器不在内存映射里、串行传输慢）无法在 DIRQL 的 ISR 里操作：注册时 Irql = PASSIVE_LEVEL、SynchronizeIrql = PASSIVE_LEVEL、SpinLock = NULL；陷阱处理例程先在中断控制器上止住/屏蔽中断，再把 ISR 调度到 PASSIVE_LEVEL（可同步发 I/O 请求、可阻塞等完成）；KeSynchronizeExecution 对此改用事件同步而非自旋锁；传 SpinLock 必败（STATUS_INVALID_PARAMETER）；KeAcquireInterruptSpinLock / KeReleaseInterruptSpinLock 对被动级 ISR 会蓝屏。
6. **MSI 与线中断的根本区别：从“占引脚发电信号”到“写地址一个值”**。MSI 是边沿语义、无硬件确认；MSI 最多 16 条（低 4 位区分），MSI-X 最多 2048 条、每条独立地址、表项可独立屏蔽、可动态改写（GUID_MSIX_TABLE_CONFIG_INTERFACE 的 SetTableEntry / MaskTableEntry / UnmaskTableEntry，Vista SP1+）；一设备同时支持 MSI 与 MSI-X 时系统自动用 MSI-X。
7. **PnP 配不齐时的降级是驱动必须忍的**。驱动想多要消息，PnP 配不出就只给 1 条；最坏只给 1 条线中断。驱动必须在注册后核对实际配到的消息数（资源描述符的 MessageCount，或 IoConnectInterruptEx 返回的 MessageBased.ConnectionContext 里的 MessageCount），按实际数工作，不能按想要的数写死。
8. **DPC 单例规则是竞态 bug 的模板**。同一 DPC 对象同一时刻只能排一次：ISR 多次排队只跑一次；但 ISR 在 DPC 已跑在别核时再排队，DPC 可在两核上并发。重叠 I/O 的驱动因此必须：DPC 按“未完成计数”办（一次跑完计数个请求）、ISR 绝不覆盖 DPC 未用完的上下文、共享上下文经 SynchCritSection 访问。单核上也不会有 1:1 对应：ISR 在 DPC 跑之前又触发，DPC 还是只跑一次。
9. **DPC 100 微秒纪律**。DPC 跑在 DISPATCH_LEVEL 时同核所有线程停摆、已排队的其他 DPC 也被堵住；典型 DPC 每次不超 100 微秒；超了就拆 CustomTimerDpc 或工作项；KeStallExecutionProcessor 也不许超 100 微秒。拿 DPC 当长活跑，是全系统卡顿的第一嫌疑。
10. **DPC 队列组织：每核一条队，驱动可控三件事**。KeSetTargetProcessorDpc 定核（默认当前核；Vista 起线程化 DPC 也有每核一条队）；KeSetImportanceDpc 定队内位置（High 放队首）与是否立即处理：本核且 Importance ≠ Low 立即处理；本核 Low 则队深超系统上限或请求率跌破下限才处理；跨核则 MediumHigh/High 立即触发目标核，队深超限也触发；其余 deferred。
11. **线程化 DPC：PASSIVE_LEVEL 跑，但仍受 DPC 环境约束**。Vista+ 默认开启（注册表 ThreadDpcEnable 置零可关，关了就按普通 DPC 跑）；可被普通 DPC 抢占、不被线程抢占；但代码仍须遵守 DISPATCH_LEVEL 约束——不许阻塞等 KEVENT 之类被动级同步对象。多数现有协议栈（网络、存储、USB）与 KMDF 明确不支持线程化 DPC，原因是它们检测到 PASSIVE_LEVEL 就可能阻塞。转换只换两处：KeInitializeDpc → KeInitializeThreadedDpc；AtDpcLevel 版拿锁函数 → ForDpc 版（线程化 DPC 里禁用 AtDpcLevel 版，因为例程可能在 PASSIVE_LEVEL 跑）。
12. **IRQL 纪律总表与三条致命线**。PASSIVE_LEVEL：DriverEntry / AddDevice / Reinitialize / Unload、多数分发、自建线程、工作项回调；APC_LEVEL：部分分发（主要是页面路径的低层/中间层读写与设备控制）；DISPATCH_LEVEL：StartIo / AdapterControl / ControllerControl / IoTimer / Cancel（持取消自旋锁时）/ DpcForIsr / CustomDpc / CustomTimerDpc；DIRQL：InterruptService / SynchCritSection。三条致命线：① ≥ DISPATCH_LEVEL 等分发器对象（KeWaitForSingleObject 非零等待）必蓝屏；② > APC_LEVEL 碰分页内存（分配分页池、访问分页数据）必蓝屏；③ KeRaiseIrql 只能往高调（传更低值必错）、KeLowerIrql 只能还原（非配对还原必错）。
13. **分发器对象等的前提：非任意线程上下文 + PASSIVE_LEVEL**。最高层分发可等；低层分发只在同步操作可等（CREATE / FLUSH / SHUTDOWN / CLOSE、部分设备控制与 PnP/电源操作）；低层分发绝不许为异步 I/O 完成而等；IOCTL 分发等只限 METHOD_BUFFERED；分页设备传输完成不许等；SCSI miniport 不用分发器对象。预期常 stall 超 50 微秒的驱动，指南建议上自建线程（最高层则考虑系统工作线程）。
14. **自旋锁家族选型与红线**。普通自旋锁（低竞争）/ 排队自旋锁（XP+，多核高竞争先到先得，KLOCK_QUEUE_HANDLE 栈上分配，同锁上两套例程不可混用）/ 读写自旋锁（Vista SP1+，读共享写独占，写等待时新读者也得等，共享可转独占但须无读者且无等待写者）/ 中断自旋锁（ISR 共享数据专用）。红线：持有不超 25 微秒；持有期间不许异常、不许碰分页、不许递归（递归必死锁）、嵌套必须固定顺序、不许调破坏前述规则的外部例程；AtDpcLevel 版只许在已是 DISPATCH_LEVEL 时用，且拿放必须配对（KeAcquireSpinLock 拿的只能 KeReleaseSpinLock 放）。
15. **事件 / 信号量 / 互斥体分工**。事件 = 点名通知（同步放一个自动清零、通知放全部手动清零）；信号量 = 计数放行（Limit 1 二值、> 1 计数）；互斥体 = 带所有权与防死锁保护的独占（可递归、持锁禁普通 APC、禁带锁回用户态）；快速/守卫互斥体 = 更快但不可递归、不可 KeWait、不可与分发器对象同等。二值信号量 ≠ 互斥体：没有所有权，死锁了系统帮不上忙。
16. **定时器两套 API 与 CustomTimerDpc 的坑**。KTIMER（KeXxx）与 EX_TIMER（ExXxx，Win8.1+，系统分配不透明对象，另有高精度与不唤醒类型）；通知定时器放全部、同步定时器只放一个且自动清零；KeSetTimer 一次性、KeSetTimerEx 可周期；DueTime 100 纳秒单位，正值绝对、负值相对。CustomTimerDpc 的坑：同一 DPC 指针绝不许同时给 KeSetTimer 与 KeInsertQueueDpc（先到先跑，另一边的活直接被丢）；同一定时器对象绝不许既做 CustomTimerDpc 又做等待（重设会取消另一边）；周期定时器不许在 DPC 里释放；KeCancelTimer 有竞态（到期抢先时仍会 signal 并排 DPC）。

---

## 三、关键知识点详解

### 详解一：ISR——只干急事的四步与六步

- **为什么 ISR 必须短**：ISR 跑在 DIRQL，同核上 ≤ 该级别的中断全被屏蔽。ISR 每多拖 1 微秒，同核上所有低级别设备的中断就晚 1 微秒响应；ISR 里调一个分页例程或等一个对象，不是慢，是直接蓝屏（> APC_LEVEL 不许碰分页、≥ DISPATCH_LEVEL 不许等）。所以指南把 ISR 的职责压到四件事：判断是不是自己的中断、止住设备再中断、搬现场、排 DPC。
- **四步（通用）**：① 设备不是 ISR 支持的 → 立刻返回 FALSE；② 是自己的 → 必要时清中断（止住设备再中断）；③ 把 DPC 需要的上下文搬进设备扩展（常驻内存，DPC 在 DISPATCH_LEVEL 也能碰）；④ 排 DPC 后返回 TRUE。
- **六步（不重叠 I/O 的驱动）**：① 判伪中断（spurious），是则立刻 FALSE，让真正的中断源 ISR 尽快跑；② 必要时止住设备再中断；③ 收集 DPC 需要的上下文；④ 存进设备扩展；⑤ 有 DpcForIsr 调 IoRequestDpc（传设备对象、当前 IRP、上下文），有 CustomDpc 调 KeInsertQueueDpc（传 DPC 对象与上下文）；⑥ 返回 TRUE。
- **重叠 I/O 的附加规矩**：上下文必须含“未完成请求计数”，DPC 按计数办；ISR 在 DPC 用完上下文之前绝不覆盖它。为什么：DPC 单例规则下，ISR 第二次触发时 DPC 可能还没跑，旧上下文还在；覆盖等于把未办的请求现场擦掉。
- **执行流转**：设备中断 → 中断向量（IDT → 内核中断陷阱处理例程）→ 系统拿中断自旋锁 → ISR（DIRQL，中断上下文，借用被中断代码的执行流）。ISR 返回后系统放锁；DPC 的执行是另一段转移（见详解四）。
- **上下文存放点**：注册时传的 ServiceContext 通常就是设备对象或设备扩展指针；中断对象指针也存设备扩展（注销时凭它交还），或放驱动分配的非分页池。为什么放设备扩展：ISR 与 DPC 都要碰，且都要在高 IRQL 下碰，只能是常驻内存。
- **单核 / 多核分叉**：单核上，ISR 与 DPC 不会真正并行，但 ISR 可在 DPC 跑之前再次触发——DPC 还是只跑一次（单例规则在单核也成立）。多核上，ISR 可在 CPU1 跑、DPC 在 CPU2 跑，ISR 与 DPC 真正并发，共享上下文必须经 SynchCritSection（见详解五）；且系统保证“ISR 不在另一核上同时跑”（调 ISR 前拿中断自旋锁），但不保证“DPC 只跑一份”（见详解四的并发情形）。

### 详解二：注册 ISR——三个版本与软连接

- **CONNECT_LINE_BASED**：单 ISR 管全部线中断。参数：PDO（系统自动识别线中断）、ServiceRoutine、ServiceContext、InterruptObject（出参，注销凭据）、SpinLock（通常 NULL 让系统分配）、SynchronizeIrql（0 让系统定）。限制：只能注册一个 ISR 管全部线中断；多中断的驱动须用 FULLY_SPECIFIED。
- **CONNECT_MESSAGE_BASED**：单 InterruptMessageService 管全部消息中断。参数多出 MessageServiceRoutine、FallBackServiceRoutine（设备只有线中断时系统改注册 FallBack 的 InterruptService，并把 Version 改回 CONNECT_LINE_BASED，驱动须记下实际注册类型）、ConnectionContext（出参：InterruptMessageService 时是 IO_INTERRUPT_MESSAGE_INFO，InterruptService 时是 KINTERRUPT，注销凭据）。SpinLock 同样通常 NULL。
- **CONNECT_FULLY_SPECIFIED**：逐个中断指定。信息来自 START_DEVICE 的 CM_PARTIAL_RESOURCE_DESCRIPTOR：线中断取 u.Interrupt（Vector / Level / Affinity），消息中断取 u.MessageInterrupt.Translated（同三个字段）；InterruptMode 由 Flags 的 CM_RESOURCE_INTERRUPT_LATCHED 位定（锁存 Latched / 电平 LevelSensitive）；ShareVector 由 ShareDisposition 定。消息中断的资源描述符只有 Vista+ 才给。Vista 之前唯一能用的版本，老系统靠链接 Iointex.lib 实现，指定其他版本会返回错误并把 Version 置回 FULLY_SPECIFIED——驱动可用这个行为探测系统版本后重试。
- **注销与软连接**：注销用 IoDisconnectInterruptEx（LINE_BASED / FULLY_SPECIFIED 凭 InterruptObject，MESSAGE_BASED 凭 Version + ConnectionContext），只能在 PASSIVE_LEVEL 调。Win8+ 另有软连接：IoReportInterruptActive / IoReportInterruptInactive（≤ DISPATCH_LEVEL，几乎不失败），只开关 ISR 调用、不动注册——典型用法是设备出 D0 前软断、回 D0 后软连。为什么要有软连接：完整注销/注册可能因资源不足失败，电源状态转换路径上失败不起；且软连接更快。红线：软断前必须先让设备停发中断，否则共享电平触发线上的其他设备 ISR 永远认领不到、中断一直 firing；软连后才让设备恢复发中断。
- **亲和性策略**：管理员或 INF 在 Interrupt Management\Affinity Policy 下写 DevicePolicy（REG_DWORD，对应 IRQ_DEVICE_POLICY：0x00 本机默认、0x01 就近全部核、0x02 就近一核、0x03 全部核、0x04 指定核、0x05 消息分散到全部核）与 AssignmentSetOverride（REG_BINARY 的 KAFFINITY 掩码，仅 DevicePolicy = 0x04 时有效）。系统在 IRP_MN_FILTER_RESOURCE_REQUIREMENTS 时把策略交给驱动（AffinityPolicy / TargetedProcessors 字段）。32/64 分叉：KAFFINITY 在 32 位系统是 32 位、64 位系统是 64 位，每组最多处理器数同此（32 / 64），超了用 KeQueryMaximumProcessorCountEx 查实际数。

### 详解三：MSI / MSI-X——从引脚到写地址

- **为什么要有消息中断**：线中断要占物理引脚、要共享向量认领、要处理电平/边沿语义；设备一多，引脚与向量都不够分。消息中断把“中断”变成一次普通的内存写：设备向特定地址写一个值，系统按地址与值识别中断源。没有引脚、没有共享认领、可扩展。
- **MSI（PCI 2.2）**：一地址 + 16 位值；一设备一个地址，用值低 4 位区分消息，最多 16 条；全部消息共享同一亲和性。
- **MSI-X（PCI 3.0）**：每条消息自己唯一的地址 + 32 位值，最多 2048 条；硬件表每表项可独立屏蔽、可动态改写（SetTableEntry 指派消息 ID、MaskTableEntry 屏蔽、UnmaskTableEntry 解屏，接口 GUID_MSIX_TABLE_CONFIG_INTERFACE，经 IRP_MN_QUERY_INTERFACE 向总线驱动要）；每条消息可有不同亲和性——多核下才能把中断分散到各核。默认表配置是第 n 项配消息 ID n，表项多于消息数时多余项配消息 ID 0。
- **系统选型**：一设备同时支持 MSI 与 MSI-X 时系统自动用 MSI-X。驱动注册仍是同一套 IoConnectInterruptEx（MESSAGE_BASED 管全部消息，或 FULLY_SPECIFIED 逐条指定）。
- **开启步骤**：① 安装时 INF 在 Interrupt Management\MessageSignaledInterruptProperties 下写 MSISupported = 1，并可用 MessageNumberLimit 定上限（MSI 只能 1/2/4/8/16，MSI-X 可到 2048）；② 可选：在 IRP_MN_FILTER_RESOURCE_REQUIREMENTS 里改消息数与每消息设置（MSI 改 MinimumVector 算条数，MSI-X 增删描述符条目；MSI-X 加的条目在 START_DEVICE 时不许再删）；③ START_DEVICE 处理里调 IoConnectInterruptEx 注册；④ 不再服务时先禁设备中断、再 IoDisconnectInterruptEx 注销。
- **降级现实**：PnP 配不出想要的条数时只给 1 条；最坏只给 1 条线中断。驱动必须在注册后核对实际数（资源描述符的 MessageCount，或返回的 MessageBased.ConnectionContext 里 InterruptMessageTable 的 MessageCount），按实际数工作。系统上限：Win8 起每设备功能不超 2048 条，Vista/Win7 不超 910 条；超了设备可能起不来；指南建议每处理器不多于 1 条中断。为什么：中断数超过核数并无收益，反而浪费向量与表项。
- **边沿语义的含义**：消息中断是边沿触发——设备发消息后收不到硬件确认。后果：驱动不能靠“中断还 assert 着”判断设备状态；丢消息就是丢事件，驱动须用计数/状态位自查，不能假设每次设备事件都有一次 ISR。

### 详解四：DPC——善后的全部规矩

- **两种 DPC 与注册排队对**：DpcForIsr（系统为设备对象配）：IoInitializeDpcRequest 注册（AddDevice 或 START_DEVICE 处理里），ISR 里 IoRequestDpc 排队（传设备对象、当前 IRP、上下文；I/O 管理器代调 KeInsertQueueDpc）。CustomDpc（驱动自备）：KeInitializeDpc 注册（驱动自备 DPC 对象存储——设备扩展、控制器扩展或非分页池，必须是常驻内存，因为 ISR 要传它的指针），ISR 里 KeInsertQueueDpc 排队（传 DPC 对象与两个驱动定义的参数）。两者跑的时机相同：某核 IRQL 降到 DISPATCH_LEVEL 以下时出队，在该核上以 DISPATCH_LEVEL 跑。
- **DPC 到底干什么（职责清单）**：① 尽快 IoStartNextPacket（或通知驱动可取下一个 IRP）——设备别闲着；② 完成当前 IRP（填状态块、IoCompleteRequest），或重编设备做下一次部分传输/重试；③ 缓冲 I/O 时把读到的数据拷进 Irp->AssociatedIrp.SystemBuffer 再完成；④ 直接 I/O 大传输拆部分传输（记已完成部分、算下一段、经 SynchCritSection 重编设备）；⑤ 包 DMA 时每次传输后 FlushAdapterBuffers，全做完后 FreeAdapterChannel / FreeMapRegisters；⑥ PIO 读传输结束时 KeFlushIoBuffers；⑦ 控制器对象用完 IoFreeController；⑧ 记错误日志（IoAllocateErrorLogEntry / IoWriteErrorLogEntry）、该重试的重试；⑨ 只完成“改设备状态或返回易变当前状态”的那部分设备控制请求。
- **单例规则与并发真相**：同一 DPC 对象同一时刻只能排一次。ISR 多次排队 → 只跑一次（合并）。但 ISR 在 DPC 已跑在别核时再排队 → DPC 可在两核并发。为什么：排队操作只保证“队里至多一份”，不保证“执行中不再入队”。所以重叠 I/O 驱动必须按计数办、ISR 不许覆盖未用完的上下文（见详解一）；且 DPC 跑在哪个核不确定——不一定是 IoRequestDpc 所在的核（指南自旋锁例子里 DpcForIsr 就跑在第三核上）。
- **DPC 的约束（DISPATCH_LEVEL 的全部后果）**：不许访问/分配分页内存；不许等分发器对象；拿驱动执行自旋锁用 AtDpcLevel 版（更快，因为不用重设 IRQL）；需要阻塞或长活时委托工作项（DpcForIsr 调 IoQueueWorkItem，CustomDpc 调 ExQueueWorkItem，工作线程在 PASSIVE_LEVEL 跑）；每次不超 100 微秒；KeStallExecutionProcessor 不超 100 微秒；适配器/控制器对象须在完成当前 IRP 之前释放。
- **执行流转**：ISR 调 IoRequestDpc / KeInsertQueueDpc 是函数调用（把 DPC 对象挂进某核 DPC 队）；IRQL 降到 DISPATCH_LEVEL 以下时，分发中断（软件中断）出 KiDispatchInterrupt 跑 DPC——DISPATCH_LEVEL，任意处理器，DPC 上下文（不是任何线程的上下文，所以不能依赖“请求线程”身份，也不能碰请求线程的私有东西）。
- **IoTimer 与 CustomTimerDpc 的位置**：IoTimer 是 1 秒一次的粗粒度周期例程；CustomTimerDpc 是定时器到期跑的 DPC（最小约 10 毫秒）。两者都是 DPC 环境，约束相同。

### 详解五：临界区——ISR 共享数据唯一正规口子

- **为什么普通拿锁不够**：KeAcquireSpinLock 只把本核 IRQL 抬到 DISPATCH_LEVEL；ISR 跑在 DIRQL（高于 DISPATCH_LEVEL），同核 ISR 照样能打断拿锁的代码、碰同一份数据。所以“ISR 也碰的数据”不能用普通执行自旋锁保护，必须走中断临界区。
- **中断临界区的构成**：抬本核 IRQL 到 SynchronizeIrql（ISR 进不来）+ 拿中断自旋锁（别核的临界区代码进不来）。系统在调 ISR 之前自动进临界区；驱动想进临界区，调 KeSynchronizeExecution，传中断对象指针 + 上下文 + SynchCritSection 例程，系统抬级别、拿锁、同处理器调用、返回后降级放锁。
- **SynchCritSection 的两类用途与写法**：① 编程设备（设寄存器）；② 访问共享状态。写法要求：越快越好（跑它时 ISR 被堵住）；职责离散（一个例程管一组状态变量，别写一个大而全的 switch 例程，也别写一堆只改一个字节的例程）；IRP 预处理、取硬件资源等准备工作在调它之前做完。指南的计时器例子：StartIo 初始化计时器计数值；ISR 把计数值置 -1；IoTimer 每秒经 SynchCritSection_1 读/减计数值，到零经 SynchCritSection_2 编程设备复位；DpcForIsr 重编部分传输时也要经 SynchCritSection 重置计数值——一个计数值只由一个 SynchCritSection 例程与 ISR 维护，避免多例程争同一状态。
- **被动级 ISR 的变体**：被动级 ISR 的中断对象没有自旋锁，KeSynchronizeExecution 改用内核同步事件对象同步 SynchCritSection（IoConnectInterruptEx 注册时由系统分配，驱动不许传 SpinLock）。
- **多中断向量共享临界区**：多个中断向量须共享同一临界区时，驱动须自己提供自旋锁与 SynchronizeIrql（FULLY_SPECIFIED 的 SpinLock 与 SynchronizeIrql 字段）；LINE_BASED / MESSAGE_BASED 传 NULL 时系统按线/按消息各配一把锁与级别。
- **执行流转**：KeSynchronizeExecution 是函数调用（不是中断、不是换栈）：系统在本核抬 IRQL、拿锁后，在同处理器上调用 SynchCritSection，返回后降 IRQL、放锁。上下文仍是调用者的线程上下文（只是 IRQL 被抬高），所以 SynchCritSection 里仍不许等、不许碰分页。

### 详解六：IRQL 纪律——总表与致命线

- **总表（级别 → 屏蔽 → 例程）**：PASSIVE_LEVEL（不屏蔽；DriverEntry / AddDevice / Reinitialize / Unload、多数分发、自建线程、工作项回调）；APC_LEVEL（屏蔽 APC；部分分发——主要是页面路径的低层/中间层读写与设备控制，且这些例程必须常驻、不可分页、不可依赖阻塞调用）；DISPATCH_LEVEL（屏蔽分发与 APC，设备/时钟/掉电中断仍可进；StartIo / AdapterControl / AdapterListControl / ControllerControl / IoTimer / Cancel（持取消自旋锁时）/ DpcForIsr / CustomDpc / CustomTimerDpc）；DIRQL（屏蔽 ≤ 该中断对象级别的中断，更高 DIRQL 与时钟/掉电中断仍可进；InterruptService / SynchCritSection）。
- **低层与高层驱动的 IRQL 画像**：低层驱动在三档间走：分发 PASSIVE_LEVEL → StartIo/DPC DISPATCH_LEVEL → ISR/SynchCritSection DIRQL。高层驱动只在两档间走：分发 PASSIVE_LEVEL → 完成例程 DISPATCH_LEVEL。例外：页面路径的低层/中间层读写分发可到 APC_LEVEL；休眠/页面路径的 DispatchPower 与需启动涌流的 DispatchPower 可到 DISPATCH_LEVEL。
- **为什么 IRQL 决定可调例程集合**：每个支持例程都有“调用者最高 IRQL”前提——前提来自两点：例程会不会等（等只能在 PASSIVE_LEVEL 非任意线程上下文）、例程会不会碰分页（分页只能在 < DISPATCH_LEVEL，且 > APC_LEVEL 碰分页必蓝屏）。所以“某例程在某 IRQL 能不能调”不是风格问题，是会不会蓝屏的问题。
- **三条致命线（指南原话级）**：① ≥ DISPATCH_LEVEL 调 KeWaitForSingleObject / KeWaitForMultipleObjects 做非零等待 → 致命错误；② > APC_LEVEL 分配/访问分页池 → 致命错误（页故障发生在不能缺页的级别）；③ KeRaiseIrql 传低于当前 IRQL 的值 → 致命错误，KeLowerIrql 不用于还原配对的 KeRaiseIrql → 致命错误。另两条配对规则：AtDpcLevel 版拿锁例程只能在已是 DISPATCH_LEVEL 时调；KeAcquireSpinLock 拿的锁只能 KeReleaseSpinLock 放（混用 FromDpcLevel 版放，调用者原 IRQL 得不到还原）。
- **IRQL 升降的为什么**：抬 IRQL 是“用屏蔽换不受打断的时间”，代价是同核上被屏蔽的事件全部延迟；所以指南要求“抬了尽快降”（KeRaiseIrql 后尽快 KeLowerIrql），高 IRQL 代码越短越好。降 IRQL 不是“放松”，是“把挂起的中断放进来”——DPC 队、APC 投递都在降级别的那一刻获得执行机会。
- **32/64 分叉**：IRQL 档位本身不分字长；分叉在亲和性掩码（KAFFINITY 32 位 vs 64 位）与每组处理器上限（32 vs 64）。DIRQL 具体数值由系统按中断向量分配，指南不给数值表——驱动不许硬编码 DIRQL，只能用系统给的 SynchronizeIrql。

### 详解七：分发器对象——能等的同步

- **状态机**：每个分发器对象只有 Signaled / Not-Signaled 两种状态。线程调 KeWaitForSingleObject（等一个）、KeWaitForMutexObject（等互斥体）、KeWaitForMultipleObjects（等任一或等全部）睡下去；对象变 Signaled 时等它的线程变就绪（同步事件与同步定时器例外：只放一个等待线程）。就绪线程按优先级与核可用性被调度——“被放行”不等于“立刻跑”。
- **能等的两个前提（缺一不可）**：① 非任意线程上下文（身份可确定的线程：DriverEntry / AddDevice / Reinitialize / Unload、最高层分发、自建线程；或完全同步的 I/O 请求——没有任何驱动排队、没有任何驱动提前返回）；② PASSIVE_LEVEL。为什么：等下去的是一个线程，醒来还要接着干；任意线程上下文里等，等于把无辜路过的线程扣住；≥ DISPATCH_LEVEL 时调度器根本不切线程，等就是死等。
- **分层规则**：最高层分发可等；低层分发只在同步操作可等（CREATE / FLUSH / SHUTDOWN / CLOSE、部分设备控制与 PnP/电源操作）；低层分发绝不许为异步 I/O 完成而等；IOCTL 分发等只限 METHOD_BUFFERED（因为只有缓冲方式下数据载体与等待线程的上下文无关）；分页设备传输完成不许等（等会把页面路径扣住，系统缺页风暴）；SCSI miniport 不用分发器对象（只调 SCSI Port 库例程）。
- **等不起时的出路**：自建线程（PsCreateSystemThread）或系统工作线程（IoQueueWorkItem）：线程在 PASSIVE_LEVEL 等事件/信号量，DPC 在 DISPATCH_LEVEL 设事件/放信号量。指南的经验值：预期常 stall 超 50 微秒的驱动就该考虑自建线程——50 微秒是“空转等”与“睡下去等”的盈亏线。
- **执行流转**：KeSetEvent / KeReleaseSemaphore / KeReleaseMutex 是函数调用（把对象变 Signaled、把等待线程变就绪）；等待线程的恢复是调度器换栈 + ret（PASSIVE_LEVEL，新线程上下文——被唤醒的线程在自己的栈上继续跑）。

### 详解八：事件、信号量、互斥体——分工与坑

- **事件**：初始化 KeInitializeEvent（或 IoCreateNotificationEvent / IoCreateSynchronizationEvent），存储须常驻（设备扩展、控制器扩展或非分页池）。同步事件放一个且自动清零——“叫一个人”；通知事件放全部且须手动清零——“叫所有人”，清零用 KeClearEvent（除非需要 KeResetEvent 返回的旧状态，否则优先用 Clear，性能更好）。驱动自建同步 IRP（IoBuildSynchronousFsdRequest / IoBuildDeviceIoControlRequest）配的事件几乎总是通知事件（调用者等“一个或多个低层驱动办完”的通知）。标准事件（\KernelObjects\High/LowMemoryCondition、High/LowPagedPoolCondition、High/LowNonPagedPoolCondition、Low/High/MaximumCommitCondition）全是通知事件，用 IoCreateNotificationEvent 开句柄、须自建线程去等。
- **Wait 参数的坑**：KeSetEvent / KeReleaseSemaphore / KeReleaseMutex 的 Wait = TRUE 表示“调用者马上就要等”——系统可借此优化（不等调度器空转）。但可分页线程/例程在 < DISPATCH_LEVEL 时绝不许 Wait = TRUE：调用者在 KeSetEvent 与 KeWait 之间被换出分页，回来时页不在，致命页故障。DISPATCH_LEVEL 的例程本来就不能等，但可以在 ≤ DISPATCH_LEVEL 时调 KeSetEvent（只设状态、不等）。
- **信号量**：KeInitializeSemaphore（存储常驻，给 Count 与 Limit）。Count 非零即 Signaled；每次 KeReleaseSemaphore 的 Adjustment 加进 Count。典型用法：设备专用线程等信号量（队空时睡），分发每入队一个 IRP 就 KeReleaseSemaphore（Adjustment = 1）放它起来；线程取 IRP 干活、干完再等；Count 仍非零说明队里还有活，线程立刻再被放行。为什么不用事件：事件只能“有活”，信号量还能“有几个活”。
- **互斥体**：KeInitializeMutex（初始即 Signaled，存储常驻）。所有权语义：内核一次只给一个线程；等它的线程（KeWaitForSingleObject / KeWaitForMutexObject）拿到后变所有者、对象变 Not-Signaled；放（KeReleaseMutex）后变 Signaled，内核挑一个等待线程当新所有者（可能提优先级到最低实时级）。递归：拿几次就要放几次才变 Signaled。持锁禁普通 APC（特殊内核 APC 如 I/O 完成仍可进）；持锁线程绝不许不放锁就回用户态（内核直接蓝屏）。主要用户是最高层驱动（FSD 的工作线程、执行线程）。
- **二值信号量 ≠ 互斥体**：表面都是“一次一个”，但信号量没有所有权——谁都可以放，死锁了内核帮不上忙；互斥体的所有权让内核能在 SMP 上做防死锁保护。指南原话：二值信号量没有互斥体那种针对 SMP 系统线程的内置防死锁保护。

### 详解九：定时器与 CustomTimerDpc

- **两套 API**：KTIMER（KeXxx，2000 起）：驱动分配 KTIMER 存储，KeInitializeTimer 建通知定时器，KeInitializeTimerEx 可选通知/同步；KeSetTimer 到期一次，KeSetTimerEx 加 Period 做周期。EX_TIMER（ExXxx，Win8.1 起）：系统分配不透明对象（ExAllocateTimer / ExDeleteTimer），另支持高精度与不唤醒（no-wake，不为定时无故唤醒低功耗核）两种类型；ExSetTimer / ExCancelTimer 对应 KeSetTimer / KeCancelTimer。Win8.1+ 仍支持 KeXxx。
- **通知 vs 同步定时器**：通知定时器到期放全部等待线程，状态保持 Signaled 直到显式复位；同步定时器到期只放一个等待线程，随即自动回 Not-Signaled。选哪个取决于“到期时叫一个人还是叫所有人”。
- **DueTime 单位与符号**：100 纳秒单位；正值绝对时间（自 1601-01-01），不受系统时间改动影响等待时长；负值相对间隔（从现在起过多久），不受系统时间改动影响（相对计时只数流逝）。等定时器对象的线程传 NULL 的 DPC 指针（等对象本身，不跑 DPC 例程）。
- **CustomTimerDpc 注册与运行**：AddDevice 里 KeInitializeDpc 注册例程 + KeInitializeTimer 建对象；KeSetTimer/KeSetTimerEx 设 DueTime 入队。到期时对象出队变 Signaled；若当时所有核都在 ≥ DISPATCH_LEVEL，DPC 对象进 DPC 队，否则直接调例程（DISPATCH_LEVEL）。最小间隔约 10 毫秒——比 IoTimer 的 1 秒细得多，适合小超时。
- **CustomTimerDpc 的四个坑**：① 同一 DPC 指针绝不许同时给 KeSetTimer 与 KeInsertQueueDpc：先到先跑，另一边的活直接被丢（指南的例子：StartIo 设了定时器、ISR 同时用同一指针排 DPC，哪个先到哪个跑，另一边的活就没了）；② 同一定时器对象绝不许既做 CustomTimerDpc 又做等待：重设/取消会打断另一边；③ 周期定时器不许在 DPC 里释放，只能释放非周期定时器；④ KeCancelTimer 有竞态：取消时若已濒临到期，到期仍可能抢先发生（signal + 排 DPC 照样走）。正确姿势：每 CustomTimerDpc 一个定时器对象；每组等待线程一套“可等待”定时器对象；两者绝不混用。
- **上下文保护**：CustomTimerDpc 的 DeferredContext 须常驻（设备扩展或非分页池）；与 ISR 共享的上下文须经 KeSynchronizeExecution；与其他非 ISR 例程共享的须用执行自旋锁。CustomTimerDpc 与设备对象无必然关联（不像 IoTimer），想关联就把设备对象指针放进上下文。

### 详解十：自旋锁家族——选型与红线

- **执行自旋锁（普通）**：驱动自己拿放。KeAcquireSpinLock（须 ≤ DISPATCH_LEVEL，拿住后本核在 DISPATCH_LEVEL）→ KeReleaseSpinLock（还原原 IRQL）。AtDpcLevel 版（KeAcquireSpinLockAtDpcLevel / KeReleaseSpinLockFromDpcLevel）只许在已是 DISPATCH_LEVEL 时用（不碰 IRQL，所以更快）；混用配对必错（通用版拿的用 FromDpcLevel 版放，原 IRQL 得不到还原）。
- **中断自旋锁**：与中断对象绑定，拿住期间本核在 DIRQL。系统在调 ISR 前自动拿、返回后放；驱动不直接拿它，而是经 KeSynchronizeExecution 进临界区。驱动须自己提供存储的情形：多向量 ISR 或多个 ISR（FULLY_SPECIFIED 的 SpinLock 字段）。ISR / SynchCritSection 里绝不许调 KeAcquireSpinLock 之类的执行自旋锁例程（会死锁：ISR 在 DIRQL 拿 DISPATCH_LEVEL 的锁，锁的持有者可能正在等 ISR 用的东西）——但 ISR / SynchCritSection 可以调 ExInterlockedXxxList，前提是传的锁只被 ISR 与 SynchCritSection 用，且这把锁不许再混用于 KeXxx 例程。
- **排队自旋锁（XP+）**：多核高竞争下的变体，保证先到先得。KeInitializeSpinLock 初始化（与普通锁同一初始化），KeAcquireInStackQueuedSpinLock / KeReleaseInStackQueuedSpinLock 拿放，KLOCK_QUEUE_HANDLE 栈上分配、每次拿锁现配；DISPATCH_LEVEL 时可用 AtDpcLevel 版。红线：同锁上两套例程（排队版与普通版）不可混用。为什么先到先得重要：普通自旋锁在高竞争下，刚放锁的核可能立刻又抢到，远核长期饿死；排队锁按到达顺序放行，公平性换来整体吞吐。
- **读写自旋锁（Vista SP1+）**：读共享、写独占。ExAcquireSpinLockShared / ExAcquireSpinLockExclusive（抬到 DISPATCH_LEVEL），AtDpcLevel 版对应已是 DISPATCH_LEVEL 的调用者；ExReleaseSpinLockShared / ExReleaseSpinLockExclusive（FromDpcLevel 版不降 IRQL）；ExTryConvertSharedSpinLockExclusive 试把已持有的共享转独占（成功条件：无其他读者、且无等待的写者）。写等待时新读者也得等（防写者饿死）。存储是 EX_SPIN_LOCK 不透明结构，须非分页内存、初始化为零。递归必死锁。
- **持有纪律（全家族通用）**：存储与被保护数据必须常驻（设备扩展、控制器扩展或非分页池；锁本身被换出等于锁没了）；初始化 KeInitializeSpinLock 先于一切使用；持有不超 25 微秒（指南原话级）；持有期间不许异常（硬件/软件异常都致命）、不许碰分页、不许递归（递归必死锁：内层实例等外层放锁，外层等内层返回）、嵌套必须固定顺序（两例程各持一把、互相想拿对方那把，SMP 上间歇死锁）；不许调破坏上述规则的外部例程。
- **单核 / 多核分叉**：单核上，自旋锁的互斥主要靠抬 IRQL——同核上低级别代码进不来，锁本身几乎不会“转起来”；多核上，抬 IRQL 只挡本核，别核照样跑，所以必须有锁本身挡别核。指南的自旋锁例子就是多核场景：ISR 在 CPU1 跑（拿中断自旋锁），StartIo 在 CPU2 上 KeSynchronizeExecution 自旋等锁、同时把 CPU2 抬到 SynchronizeIrql（防本核再进 ISR），DpcForIsr 在 CPU3 上同样等锁——三核各就其位，锁是唯一仲裁者。反过来，单核上“ISR 覆盖 DPC 上下文”的竞态依然存在（ISR 在 DPC 跑之前又触发），所以计数与不覆盖规则与核数无关。

### 详解十一：快速互斥体与守卫互斥体

- **定位**：互斥体对象的低开销替代品，只用于 IRQL ≤ APC_LEVEL 的短临界区。比互斥体对象快，但限制多三条：不可用 KeWaitForSingleObject / KeWaitForMultipleObjects 等它（因而不能与分发器对象同等）；不可递归（已持有的再拿必死锁）；Win8 之前守卫互斥体与快速互斥体实现不同（快速跑在 APC_LEVEL，守卫是“APC 全禁但 IRQL 不一定升到 APC_LEVEL”的更快版），Win8 起守卫互斥体按快速互斥体实现，Driver Verifier 一律按 APC_LEVEL 检查。
- **快速互斥体**：FAST_MUTEX 结构，驱动自备存储，ExInitializeFastMutex 初始化；ExAcquireFastMutex（已被拿则挂起等）、ExTryToAcquireFastMutex（立刻返回 TRUE/FALSE，不挂起）、ExReleaseFastMutex 放；持锁期间跑在 APC_LEVEL（拿放时升降 IRQL）；已保证在 APC_LEVEL 的代码可用 Unsafe 版（ExAcquireFastMutexUnsafe / ExReleaseFastMutexUnsafe，不碰 IRQL，只许在已是 APC_LEVEL 时用）。
- **为什么比互斥体对象快**：互斥体对象要走分发器等待路径（对象状态、等待队列、调度）；快速互斥体在 APC_LEVEL 以下用“禁 APC + 短临界区”的轻量路径。代价就是上面三条限制——需要等、需要递归、需要与别的分发器对象同等时，只能用互斥体对象。

### 详解十二：执行流转小结（本批出现的四种转移）

- **中断向量进 ISR**：设备中断 → IDT → 内核中断陷阱处理例程 → 系统拿中断自旋锁 → ISR。级别 DIRQL，上下文是中断上下文（借用被中断代码的执行流），ISR 返回后系统放锁。
- **软件中断出 DPC**：ISR 的 IoRequestDpc / KeInsertQueueDpc 是函数调用（把 DPC 对象挂进某核 DPC 队）；IRQL 降到 DISPATCH_LEVEL 以下时，分发中断出 DPC。级别 DISPATCH_LEVEL，任意处理器，DPC 上下文（非线程上下文）。
- **函数调用进临界区**：KeSynchronizeExecution 是函数调用：本核抬 IRQL 到 SynchronizeIrql、拿中断自旋锁，同处理器调用 SynchCritSection，返回后降级放锁。被动级 ISR 的变体：改用事件同步，级别 PASSIVE_LEVEL。
- **调度器唤醒等事件的线程**：KeSetEvent / KeReleaseSemaphore / KeReleaseMutex 是函数调用（对象变 Signaled、等待线程变就绪）；等待线程恢复是换栈 + ret，级别 PASSIVE_LEVEL，新线程上下文（被唤醒线程在自己的栈上继续）。
- **工作项委托**：DPC 的 IoQueueWorkItem / ExQueueWorkItem 是函数调用（工作项挂进系统工作队列）；系统工作线程在 PASSIVE_LEVEL 取出执行，线程上下文。DPC 不能阻塞的活，只有经这条路才能阻塞。

---

## 四、解读（审计与逆向视角）

1. **先对 ISR 与 DPC 标 IRQL，再审其他**。DIRQL 下可调例程集合最小，违规调用（分页例程、等待、执行自旋锁例程）是蓝屏根源；审计一个驱动时，先把 ISR、SynchCritSection、DpcForIsr、CustomDpc 的 IRQL 标出来，再逐个核对它们调的例程是否在该级别允许。IRQL 标注不清的驱动，稳定性审计无从下手。
2. **DPC 单例 + 重叠 I/O 计数是竞态 bug 的模板**。逆向时重点看两处：ISR 是否在 DPC 未用完上下文前覆盖设备扩展里的现场；DPC 是否按“未完成计数”办、计数是否由中断自旋锁保护（计数本身是 ISR 与 SynchCritSection 共同维护的状态）。单核上“ISR 又触发、DPC 只跑一次”同样会丢请求——不要以为单核就没有这个竞态。
3. **线程化 DPC 的“PASSIVE_LEVEL 但仍不能阻塞”是反直觉点**。审计时不要看到 PASSIVE_LEVEL 就放行阻塞调用：线程化 DPC 里等 KEVENT 之类被动级同步对象仍违规；且多数现有协议栈（网络、存储、USB）与 KMDF 明确不支持——一个驱动若在这些栈里用线程化 DPC，本身就是可疑点。逆向遇到 KeInitializeThreadedDpc 时，先确认该栈是否支持。
4. **MSI-X 表动态改写是攻击面**。SetTableEntry / MaskTableEntry / UnmaskTableEntry 能改中断路由（消息 ID 指派与屏蔽）；审计 GUID_MSIX_TABLE_CONFIG_INTERFACE 的使用时，问一句：调用方身份是否受控？一张能改中断路由的表若可被低权限路径触及，等于可改“哪个核响应哪个设备事件”。
5. **自旋锁持有超 25 微秒、DPC 超 100 微秒是性能审计红线**。长持锁 + 多核高竞争 = 全系统延迟；长 DPC = 同核所有线程停摆。性能分析工具（WDK 的 Tracelog 测 DPC/ISR 时间）是量化这两条红线的标准手段，不是凭感觉。
6. **嵌套顺序不一致是间歇死锁模板**。两把锁、两个例程、拿锁顺序不同，SMP 上间歇死锁——复现难、定位难。审计时把所有拿锁点的顺序画出来，顺序不一致即缺陷，不必等复现。
7. **互斥体递归与“禁带锁回用户态”是 FSD 审计点**。FSD 的工作线程拿互斥体后回用户态路径前必须放干净；递归拿几次就要放几次。漏放一次，互斥体永不 Signaled，后续等待线程全饿死。二值信号量当互斥体用的驱动，死锁时系统无所有权信息可查，定位难度高一档。
8. **被动级 ISR 是 Win8+ 的新形态，别误判**。逆向遇到注册参数 Irql = PASSIVE_LEVEL、SynchronizeIrql = PASSIVE_LEVEL、SpinLock = NULL 时，这是被动级 ISR（寄存器须经 I/O 请求访问的设备），不是“驱动把 ISR 错写成了分发”。且它的 KeSynchronizeExecution 走事件同步，拿中断自旋锁的旧经验不适用。
9. **软连接 vs 注册是电源路径审计点**。设备出 D0 前软断、回 D0 后软连；若软断前没让设备停发中断，共享电平触发线上的其他设备 ISR 永远认领不到，中断一直 firing——电源转换路径上的中断风暴常由此起。审计 D0 进出路径时，核对“停发中断 → 软断”与“软连 → 恢复发中断”的顺序。
10. **IOCTL 等只限 METHOD_BUFFERED 是 BYOVD 审计的顺带检查项**。低层分发为异步 I/O 完成而等、为分页设备传输完成而等，都是指南明文禁止的写法；审计低层驱动的分发例程时，KeWaitForSingleObject 出现即问一句：这是同步操作吗？是缓冲方式吗？

## 五、易混点辨析

1. **DIRQL vs SynchronizeIrql**：DIRQL 是中断向量本身的级别；SynchronizeIrql 是 ISR 与临界区执行的级别，通常相等，驱动可指定更高（LINE_BASED / MESSAGE_BASED 传 0 让系统定，FULLY_SPECIFIED 可自填）。临界区的门槛看 SynchronizeIrql，不是 DIRQL。
2. **DpcForIsr vs CustomDpc**：前者系统为设备对象配（IoInitializeDpcRequest / IoRequestDpc，对象系统分配）；后者驱动自备存储（KeInitializeDpc / KeInsertQueueDpc，对象驱动分配）；CustomDpc 不与设备对象绑定，上下文全靠驱动自己带。选哪个看“一设备几善后”，不是看喜好。
3. **普通 DPC vs 线程化 DPC**：执行级别不同（DISPATCH_LEVEL vs PASSIVE_LEVEL）、抢占性不同（普通不被任何线程抢占，线程化可被普通 DPC 抢占、不被线程抢占）；约束相同（仍不能阻塞等被动级同步对象）。“线程化 = 可以阻塞”是错的。
4. **同步事件 vs 通知事件**：同步放一个且自动清零（叫一个人）；通知放全部且须手动清零（叫所有人）。驱动自建同步 IRP 配的事件几乎总是通知事件。
5. **互斥体 vs 二值信号量**：表面一次一个，实则所有权、递归、防死锁保护、禁 APC、禁带锁回用户态五项全不同。需要其中任一项时，二值信号量都不合格。
6. **快速互斥体 vs 互斥体对象**：前者更快但不可递归、不可 KeWait、不可与分发器对象同等；后者走分发器等待路径，可递归、可同等。只在 APC_LEVEL 以下短临界区用前者。
7. **执行自旋锁 vs 中断自旋锁**：前者 DISPATCH_LEVEL，驱动自己拿放；后者 DIRQL，与中断对象绑定，系统调 ISR 前自动拿、驱动经 KeSynchronizeExecution 间接用。ISR 共享数据只能用后者。
8. **KeAcquireSpinLock vs AtDpcLevel 版**：前者抬 IRQL 到 DISPATCH_LEVEL、释放时还原，可在 ≤ DISPATCH_LEVEL 调；后者要求调用者已在 DISPATCH_LEVEL、不碰 IRQL、更快。拿放必须配对，混用必错（原 IRQL 得不到还原）。
9. **排队自旋锁 vs 普通自旋锁**：同锁初始化（KeInitializeSpinLock），不同拿放例程；排队版多核高竞争先到先得，防远核饿死；同锁上两套例程不可混用。
10. **IoTimer vs CustomTimerDpc**：前者 1 秒粗粒度、与设备对象关联；后者最小约 10 毫秒、无必然设备关联、须自备定时器对象。需要细粒度定时时 IoTimer 给不了。
11. **注册/注销 vs 软连接/断开**：IoConnectInterruptEx / IoDisconnectInterruptEx 是注册/注销（PASSIVE_LEVEL，可能因资源不足失败）；IoReportInterruptActive / IoReportInterruptInactive 是软连接/断开（≤ DISPATCH_LEVEL，几乎不失败），只开关 ISR 调用、不动注册。电源路径用后者。
12. **MSI vs MSI-X**：前者一地址 + 16 位值、最多 16 条、全部消息同亲和性；后者每条独立地址 + 32 位值、最多 2048 条、每条可不同亲和性、表项可独立屏蔽动态改写。系统有 MSI-X 就用 MSI-X。
13. **KeSetTimer vs KeSetTimerEx**：前者到期一次；后者加 Period 做周期。重设同一对象会取消旧的到期安排——CustomTimerDpc 与等待混用同一对象时，这就是互相取消的竞态源。

## 六、自查问题

1. ISR 返回 FALSE 与 TRUE 各在什么情形？共享线中断下，ISR 不判断“是不是我的设备”直接处理会怎样？
2. CONNECT_LINE_BASED、CONNECT_MESSAGE_BASED、CONNECT_FULLY_SPECIFIED 三个版本各在什么场景用？Vista 之前为什么只能用第三个？
3. 为什么 ISR 共享的数据不能用 KeAcquireSpinLock 保护？KeSynchronizeExecution 到底替调用者做了哪两件事？
4. 被动级 ISR 适用于什么设备？注册参数与普通 ISR 有哪三处不同？它的 KeSynchronizeExecution 为什么改用事件同步？
5. DPC 单例规则下，ISR 在 DPC 跑在别核时又排队会怎样？重叠 I/O 的驱动必须满足哪三条才能不丢请求？
6. 线程化 DPC 在 PASSIVE_LEVEL 跑，为什么仍不能阻塞等 KEVENT？从普通 DPC 转换到线程化 DPC 要换哪两处调用？
7. 一个低层驱动的分发例程想等一个事件，指南允许吗？分哪几种情形？IOCTL 分发等为什么只限 METHOD_BUFFERED？
8. 排队自旋锁比普通自旋锁多解决了什么问题？同锁上混用两套例程为什么不行？
9. 同步事件与通知事件在“放几个、谁清零”上各是什么？驱动自建同步 IRP 配的事件为什么几乎总是通知事件？
10. CustomTimerDpc 的四个坑各是什么？为什么“同一 DPC 指针绝不许同时给 KeSetTimer 与 KeInsertQueueDpc”？

## 未钉死 / 待查清单

- DDI 参考集细节：各支持例程的精确 IRQL 前置条件以 DDI 为准，指南只给总表；本批按总表写，未逐例程核 DDI。
- IoTimer 例程的注册/注销例程名：指南本节只讲 IoTimer 概念（1 秒、DISPATCH_LEVEL），未展开注册 API，不写成定论。
- MSI-X 表接口（GUID_MSIX_TABLE_CONFIG_INTERFACE）的调用方权限检查：指南只讲接口三例程，未讲调用方身份核验，审计时须另查。
- 线程化 DPC 在各协议栈中“检测到 PASSIVE_LEVEL 就阻塞”的具体表现：指南只说可能阻塞，未列举，须按栈另查。
- DIRQL 具体数值表：指南不给数值（系统按向量分配），驱动不许硬编码；WRK 树里的 HIGH_LEVEL 等数值是另一套口径，不混入本批。
- 快慢路径之外的 DPC 队列“系统上限 / 请求率下限”具体数值：指南只说“系统定义”，未给数字，不编造。

---

## 附：追问补记——中断暂存是位图不是队列，不会满但会合并（2026-10-29）

- **暂存设备是中断控制器**：现代 x86 是每核一个 LAPIC（本地 APIC）+ 芯片组里的 IOAPIC；老机器是 8259A PIC。关中断/抬 IRQL 时，中断就挂在它这里。此前讲的“TPR 挡住的中断留在 LAPIC 的 IRR 里”，IRR 就是这张位图。
- **容量模型是位图不是队列**：LAPIC 的 IRR（Interrupt Request Register）是 256 位，1 个向量 1 个位；8259A 的 IRR 是 8 位。所谓“存”就是把对应位点亮。所以没有“第 N+1 个中断没地方放”的溢出概念。
- **真正的失败模式是合并**：同一向量在待处理位还亮着时重复触发——边沿触发下多次事件并成一次记录（至多“在服务 1 个 + 待处理 1 个”：在服务指 APIC 的 In-Service Register 位图，与中断服务例程同缩写、不同物）。这不是控制器丢东西，是设计语义：中断只当门铃，ISR/DPC 必须查设备状态把活干干净，不能假设一次中断对应一件事——这是第 8 批 ISR 纪律的硬件根源。
- **电平触发不丢**：设备把线一直拉着，CPU 迟早看到；EOI 后线还拉着就立即再挂起。会“看似丢”的只有边沿触发的重复脉冲。
- **不保顺序**：多个向量同时挂起，按优先级（向量号高 4 位）投递，不是先到先得。
- **MSI/MSI-X 同理**：设备侧 MSI-X 的 PBA（Pending Bit Array）也是每向量 1 位，合并语义相同。
- **深队列在内存里**：高性能设备（网卡、NVMe）的事件积压靠内存中的描述符环（DMA 写入，成千上万项），中断只敲门；环满了是驱动流控/轮询的问题，与中断控制器无关。设计哲学：控制器只负责“谁喊过”，不负责“喊了几次”；次数与积压由设备状态 + 内存队列承载。
