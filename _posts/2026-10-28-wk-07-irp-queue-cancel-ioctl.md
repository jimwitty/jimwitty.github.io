---
layout: post
title: "Win 内核驱动 第 7 批：IRP（下）——队列、取消与 IOCTL"
categories: windows-kernel
series_order: 7
---

- **系列**：Windows 内核驱动循序渐进学习（教材：微软官方《内核模式驱动程序体系结构设计指南》，WDM 中心）
- **本批对应原文档**：「Handling IRPs」章后半——Queuing and Dequeuing IRPs / Writing a StartIo Routine（最底层 / 高层 / 要点）/ Driver-Managed IRP Queues（设备队列建与管、互锁队列建与管、驱动自建线程管互锁队列、取消安全队列）/ Canceling IRPs（导论、注册、取消时要点、取消同步、系统取消自旋锁、驱动自带自旋锁、实现、带/不带 StartIo 的取消例程）/ Reusing IRPs / Device Type-Specific I/O Requests / I/O Control Codes（导论、驱动内建请求、定义、缓冲描述、安全问题）/ Using IRP Priority Hints / Processing IRPs in a Lowest-Level Driver / Processing IRPs in an Intermediate-Level Driver / Different ways of handling IRPs - Cheat sheet；以及「I/O Programming Techniques」中与本批直接相关的 Methods for Accessing Data Buffers / Using Buffered I/O / Using Direct I/O（DMA 与 PIO）/ Using Neither Buffered Nor Direct I/O。
- **取材说明**：正文取自微软官方同名设计指南 PDF（2019 版离线打包）「Handling IRPs」章后半与「I/O Programming Techniques」中数据进出四节逐节研读。本 PDF 的来源仓库 tpn/pdfs 是第三方文档收集仓库、非微软官方；指南本身是微软官方文档 2019 版离线版。本批是第 6 批的下半：第 6 批讲“请求怎么流动”，本批讲“请求怎么停留（排队）、怎么打断（取消）、数据怎么进出（IOCTL 与三种缓冲方式）”。DMA 适配器对象、控制器对象的深水区属第 9 批，本批只讲到 StartIo 与它们交接的那一步，不越界。
- **学习目标**：学完本批，你应该能说清：① 请求比设备快时，驱动手里三种排队机制各是什么、怎么选；② StartIo 到底干什么、为什么高层驱动不该有它；③ 取消为什么是驱动竞态 bug 的重灾区，取消自旋锁与 IoSetCancelRoutine 返回值怎么把竞态封死；④ IOCTL 的 32 位编码四字段各是什么意思，METHOD_BUFFERED / DIRECT / NEITHER 下驱动手里拿到的分别是什么；⑤ 审计一个驱动的 IOCTL 处理时，哪几项是必查项。

---

## 一、本批地图：一个请求在队列里的一生

第 6 批的地图讲“流动”，本批补上“停留”与“打断”两段。还是读磁盘的例子，但这次设备正忙：

```
分发例程（PASSIVE_LEVEL，任意线程上下文）
收到 IRP，验参数 → 自己办不了 → 怎么办？
│
├─ 设备一次只做一件事 → StartIo 路
│   IoMarkIrpPending → IoStartPacket
│   · 队空且设备闲 → I/O 管理器当场调 StartIo
│   · 设备忙 → IRP 进系统设备队列，等着
│   StartIo（DISPATCH_LEVEL，任意线程上下文）：读联、备状态、
│   必要时拆部分传输、编程设备（与 ISR 共享的用 SynchCritSection）
│
├─ 设备能并行多件事 → 自建队列路
│   · 设备队列（KeInsertDeviceQueue，队尾或按 SortKey）
│   · 互锁队列（ExInterlockedInsertTailList，双链表 + 执行自旋锁，
│     驱动自建线程在 PASSIVE_LEVEL 取 IRP 干活）
│   · 取消安全队列 CSQ（首选：驱动只给回调，系统包办取消同步）
│
▼ 设备中断 → ISR（只干急事）→ IoRequestDpc → DpcForIsr
DpcForIsr 干两件事，顺序有讲究：
  ① 先 IoStartNextPacket —— 让 StartIo 尽快接下一个 IRP
  ② 再填状态块、IoCompleteRequest —— 完成当前这个
     （先启动下一个再完成当前个，是为了不让设备空闲）
│
└─ 中途有人取消（应用退出、线程终止、驱动主动 IoCancelIrp）
    取消是异步的：可能发生在入队前、入队后处理前、取出后启动前
    封竞态的工具：取消自旋锁 + IoSetCancelRoutine 返回值
    （谁最后碰 IRP，由返回值说了算，不是靠“谁先谁后”猜）
```

---

## 二、核心要点

1. **排队的原因只有一个：请求来得比设备办得快**。异步 I/O + 多任务 + 多核，IRP 一定会在某个驱动门口排队。唯一不需要排队机制的最底层驱动，是“能在分发例程里当场办结每一种 IRP”的驱动——现实中几乎没有。
2. **三种排队机制，一张选型表**：① 系统设备队列 + StartIo：设备一次只做一件事时的标准答案（I/O 管理器替你排队）；② 互锁队列：设备能并行、或驱动有自建线程时自己管（双链表 + 执行自旋锁）；③ 取消安全队列 CSQ：**新驱动首选**——驱动只实现队列操作回调，系统包办插入/删除的同步与取消逻辑，不用自己写 Cancel 例程。
3. **StartIo 是“第二阶段”**：分发例程是第一阶段（验参数、决定去向），StartIo 是第二阶段（真正启动设备操作）。它跑在 DISPATCH_LEVEL、任意线程上下文——这条约束决定了它能干什么、不能干什么（见详解二）。
4. **高层驱动不该有 StartIo**：会把 IRP 串行化成一次一个（瓶颈），还会把整条链下层分发例程的 IRQL 抬到 DISPATCH_LEVEL（下层驱动默认自己在 DISPATCH_LEVEL 以下跑，假设全破，互操作性完蛋）。系统自带的高层驱动没有一个有 StartIo。
5. **取消的本质是竞态**：取消发生在任意时刻，驱动必须在“入队前、入队后处理前、取出后启动前”三个点都防住“取消与处理同时到达”。封竞态的工具不是“加更多的锁”，而是两样东西：取消自旋锁（保证同一时刻只有一个人在改 IRP 的可取消状态）+ IoSetCancelRoutine 的返回值（告诉你 Cancel 例程到底跑没跑、跑没跑起来）。
6. **取消自旋锁只有一把（系统级）**：I/O 管理器调 Cancel 例程时一定拿着它；Cancel 例程的第一件事就是放掉它（传 Irp->CancelIrql）；任何例程都不许拿着任何自旋锁调 IoCompleteRequest（死锁）；驱动自带锁是性能优化（不用跟全系统抢同一把锁），不是替代品——系统调 Cancel 例程时照样先拿系统锁。
7. **IOCTL 是用户态到内核的正门**：DeviceIoControl → IRP_MJ_DEVICE_CONTROL；驱动之间私下用 IRP_MJ_INTERNAL_DEVICE_CONTROL（用户态发不出来）。32 位编码 = DeviceType + Function + Method + Access 四字段；厂商字段有保留区间（DeviceType ≥ 0x8000、Function ≥ 0x800 才是厂商的）。
8. **数据进出三方式，一句话分清**：Buffered = 系统替你拷贝（小数据，安全但慢）；Direct = 锁页 + MDL（大数据，DMA/PIO 直接读写物理页）；Neither = 只给虚拟地址（驱动自己负责探查、锁页、异常处理，最高层才能用）。栈内各层对同一请求必须用同一种方式（最高层可例外用 Neither）。
9. **IOCTL 安全规则就是审计检查表**：Function 码 ≥ 0x800、必写 Access 位、不开任意内核内存读写、测整个 32 位值、核对长度、不超 SystemBuffer、输出缓冲清零、MmGetSystemAddressForMdlSafe 判 NULL、NEITHER 走专节。违反任何一条都是 BYOVD 的候选。
10. **复用 IRP 有禁区**：只能复用自己建的（IoAllocateIrp + IoReuseIrp / IoInitializeIrp）；I/O 管理器建的、IoMakeAssociatedIrp / IoBuildSynchronousFsdRequest / IoBuildAsynchronousFsdRequest / IoBuildDeviceIoControlRequest 建的，一律不许复用。

---

## 三、关键知识点详解

### 详解一：为什么排队，以及三种机制怎么选

- **触发条件**：异步 I/O 系统里，请求到达速度超过设备完成速度是常态（尤其多核）。最底层驱动必须有地方“暂存”来不及办的 IRP，否则请求无处可去。
- **机制一：系统设备队列 + StartIo**（IoStartPacket / IoStartNextPacket）。I/O 管理器在 IoCreateDevice 时（FSD 除外）就给每个设备对象配了一个设备队列对象。分发例程调 IoStartPacket：队空且设备闲 → 当场调 StartIo；设备忙 → IRP 进队。DpcForIsr 办结后调 IoStartNextPacket 取下一个。适用：一次只做一件事的设备（多数外设）。
- **机制二：互锁队列**（ExInterlockedXxxList）。驱动自己维护双向链表（IRP 自带的 Tail.Overlay.ListEntry 串起来）+ 一把执行自旋锁（KeInitializeSpinLock）。适用：设备能并行多操作（如全双工串口分读/写/ purge / wait 四条队）、驱动有自建线程、PnP/电源状态转换期间须内部暂存 IRP。单链表（SLIST）版本不能用于要重排重试的场景（取下来就回不去了）。
- **机制三：取消安全队列 CSQ**（IoCsqXxx）。驱动实现队列操作回调，系统负责同步与取消。适用：一切新驱动自己管队列的场景——指南原话“首选方法”。
- **例外组合**：SCSI 端口驱动两者兼有——既有 StartIo（走共享 HBA 的系统设备队列），又有一组补充设备队列（按逻辑单元分流），用补充队列抑制“某个忙设备拖住整条总线”，同时让其他设备保持满负荷。

### 详解二：StartIo——第二阶段的规矩

- **谁调、什么时候调**：I/O 管理器在两种情形下调 StartIo——分发例程刚调 IoStartPacket 且队空；或 DpcForIsr 办结上一个后调 IoStartNextPacket 取下一个。调时可假设“设备不忙”。
- **IRQL 与上下文约束（决定一切）**：StartIo 跑在 DISPATCH_LEVEL、任意线程上下文。因此：① 不能等分发器对象；② 不能碰分页内存；③ 要碰的缓冲必须已锁页或来自常驻系统内存；④ 拿驱动自带执行自旋锁要用 KeAcquireSpinLockAtDpcLevel / KeReleaseSpinLockFromDpcLevel（比通用版快）；⑤ 最高层驱动若用直接 I/O 或 Neither，不能把“探查并锁用户缓冲”推迟到 StartIo——那时已不是请求线程上下文，锁不了。
- **职责清单**：读自己那一联（IoGetCurrentIrpStackLocation）；维护当前请求状态（设备扩展或非分页池，如 InterruptExpected 标志、超时计数、CustomTimerDpc）；与 ISR 共享的状态/寄存器必须经 KeSynchronizeExecution 调 SynchCritSection 碰（DIRQL 下编程设备）；给 IRP 编序号（供出错日志用）；把参数翻译成设备值（磁盘驱动算起始扇区、是否跨扇区、是否超单次传输能力）；可移动介质先查介质变没变；DMA 设备拆部分传输（除非高层已预拆）；PIO 设备用 MmGetSystemAddressForMdlSafe 把 MDL 映射成系统空间地址；用控制器对象的调 IoAllocateController；可取消的 IRP 先查是不是已取消（查法见详解六）。
- **缓冲 vs 直接的 StartIo 差别**：缓冲 I/O 的 StartIo 简单——数据在 Irp->AssociatedIrp.SystemBuffer（I/O 管理器分配的系统空间缓冲）；直接 I/O 的 StartIo 复杂——数据在 Irp->MdlAddress 的 MDL 里（DMA 传 MapTransfer，PIO 传双重映射），且大请求必须自己拆部分传输（系统 DMA 与设备单次能力取严者）。
- **设备控制请求在 StartIo 里是子集**：只有“要改设备状态或返回易变当前状态”的有效设备控制请求才进 StartIo；且公开 IOCTL 一律是缓冲方式——哪怕设备对象是直接 I/O，公开 IOCTL 也走 SystemBuffer。
- **高层 StartIo 的三宗罪**：① 串行化（一次一个，瓶颈）；② 抬 IRQL（下层分发例程全被抬到 DISPATCH_LEVEL，可调支撑例程集合缩小，假设全破）；③ 拖慢整条链与系统吞吐。结论：高层驱动用内部队列或直接下传 + 完成例程收尾，不要 StartIo。

### 详解三：设备队列——系统自带的那把与补充的那几把

- **建**：KeInitializeDeviceQueue（驱动或设备初始化时，存储须常驻——通常放设备扩展；可在 AddDevice 或 IRP_MN_START_DEVICE 时调）。
- **进**：KeInsertDeviceQueue（队尾）或 KeInsertByKeyDeviceQueue（按驱动定的 SortKey 排序进队）。**经典坑**：返回 FALSE 不代表“插队失败”——它表示队列原来是空的（Not-Busy），IRP **根本没进队**，同时队列状态被置为 Busy。调用者必须把这个 IRP 交给别的例程继续处理；否则 IRP 就丢了。而且因为状态已是 Busy，下一个 IRP 才会真的进队（除非先调 KeRemoveXxx 把状态复位）。
- **出**：KeRemoveDeviceQueue（队首）、KeRemoveByKeyDeviceQueue（按 SortKey）、KeRemoveEntryDeviceQueue（指定某一个，或测某一个在不在队里——返回布尔值）。从“空但 Busy”的队里取一次，状态复位为 Not-Busy。
- **锁与 IRQL**：每个设备队列对象自带一把执行自旋锁，多核安全；但 IRQL 限制 ≤ DISPATCH_LEVEL——**ISR 与 SynchCritSection（DIRQL）里不能调任何 KeXxxDeviceQueue**。
- **补充队列的搬运舞（SCSI 端口驱动模式）**：DpcForIsr 每办结一个 IRP，做四步：① IoStartNextPacket（让 StartIo 接共享控制器的下一个）；② KeRemoveDeviceQueue（从该设备的补充队里取一个）；③ 取到了 → IoStartPacket 把它送进共享控制器队；没取到 → 这次取动作只是把补充队状态复位为 Not-Busy，跳过 IoStartPacket；④ IoCompleteRequest 完成当前 IRP。**要点**：不要等共享控制器空闲了再搬补充队的 IRP——那样会让重负载设备饿死，轻负载设备反而先被服务。

### 详解四：互锁队列与驱动自建线程

- **建**：驱动提供存储（设备扩展 / 控制器扩展 / 非分页池）+ 一把执行自旋锁（KeInitializeSpinLock）+ 链表头（InitializeListHead 双链表；ExInitializeSListHead 单链表）。插：ExInterlockedInsertTailList（队尾）/ ExInterlockedInsertHeadList（队首，只用于必须重试的请求）；取：ExInterlockedRemoveHeadList。**禁**：不能拿着要传给 ExInterlockedXxx 的那把执行自旋锁再调它（死锁）。
- **驱动自建线程模式**（如软盘控制器驱动）：线程在 PASSIVE_LEVEL、自己的（非任意）线程上下文里跑：等分发器对象（分发例程插 IRP 后 signal 它）、取 IRP、干活。线程能等、能碰分页内存——但有两个纪律：① 线程分配的、会被 ISR / DPC / StartIo / 完成例程等高 IRQL 例程碰的内存，必须是非分页的；② DMA 相关支撑例程要求 DISPATCH_LEVEL，线程须在 KeRaiseIrql / KeLowerIrql 之间嵌套调 AllocateAdapterChannel / FreeAdapterChannel（StartIo 里不用，因为 StartIo 本来就在 DISPATCH_LEVEL）。
- **与 StartIo 的取舍**：自建线程的代价是上下文切换；设备准备过程长到影响性能时才值得。多数最底层驱动用 StartIo，不用自建线程。

### 详解五：取消安全队列 CSQ——把取消同步外包给系统

- **分工**：驱动只提供六个回调，系统提供 IoCsqXxx 操作例程并包办全部同步与取消逻辑；**用 CSQ 的驱动不用自己写 Cancel 例程**（系统替它注册一个，取消时调 CsqRemoveIrp 摘 IRP、调 CsqCompleteCanceledIrp 完成它）。
- **六个回调**（指针存在 IO_CSQ 结构里，结构大小固定、可安全嵌进设备扩展；IoCsqInitialize 配 CsqInsertIrp，IoCsqInitializeEx 配 CsqInsertIrpEx）：
  1. CsqInsertIrp 或 CsqInsertIrpEx（二选一）：插 IRP。Ex 版更灵活：有返回值（插失败可返回错误码，如队空时直接处理、队满时拒绝）、多一个 InsertContext 参数（驱动自定义的插入上下文）。
  2. CsqRemoveIrp：摘指定 IRP。
  3. CsqPeekNextIrp：返回指定 IRP 的下一个（系统把 IoCsqRemoveNextIrp 的 PeekContext 原样递过来，驱动自定其含义——如按优先级取）。
  4. CsqAcquireLock / CsqReleaseLock：锁队与放队（自旋锁、互斥体任选；自旋锁版用系统递来的 IRQL 变量存当前 IRQL）。
  5. CsqCompleteCanceledIrp：完成被取消的 IRP（样板就三行：Status = STATUS_CANCELLED、Information = 0、IoCompleteRequest(IO_NO_INCREMENT)）。
- **三个系统操作**：IoCsqInsertIrp / IoCsqInsertIrpEx（插）、IoCsqRemoveNextIrp（取下一个，可带 key）、IoCsqRemoveIrp（摘指定，靠 IO_CSQ_IRP_CONTEXT 定位）。驱动只须在回调里实现“最本质的功能”（如锁回调只管锁），系统在需要时自动调它们。
- **为什么这是首选**：手写取消同步要同时防住“取消与入队/取出/处理”三处竞态（见详解六），CSQ 把这三处收进系统例程，驱动作者不用再算竞态。

### 详解六：取消——异步打断的三处竞态与封法

- **为什么取消难**：取消是异步的，系统或驱动随时可以调 IoCancelIrp。IRP 可能在三个点被取消：① 例程被调后、入队前；② 入队后、处理前（如 StartIo 被调后、还没从队里取出时）；③ 取出后、启动 I/O 前。**关键警告**：驱动把 IRP 入队并放掉队锁后，另一线程就能碰它——原线程恢复执行时（哪怕只是下一行代码），IRP 可能已被取消或改动。任何“入队后还假设 IRP 状态不变”的代码都是错的。
- **IRP 上的三个取消字段**：Irp->Cancel（是不是正在被取消 / 该被取消）；Irp->CancelRoutine（是不是可取消：非 NULL = 可取消；NULL + Cancel 置位 = Cancel 例程正在跑、IRP 正在被取消过程中）。驱动对自己以可取消状态持有的每个 IRP，都有责任设好 Cancel 例程。
- **可取消状态的进出规矩**：① 持有 IRP 进可取消状态前，必须先 IoMarkIrpPending、再 IoSetCancelRoutine 设例程入口，然后才能调 IoStartPacket / IoAllocateController / ExInterlockedInsert 等；② 之后处理它的例程（如 StartIo）动手前，必须先把 Cancel 例程入口清零（IoSetCancelRoutine 传 NULL），再开始 I/O；③ 高层驱动把 IRP 交给下层（IoCallDriver）前，必须先把自己的 Cancel 入口清零——交出去的 IRP 不再归它管。
- **系统取消自旋锁的规矩**：全系统只有一把。I/O 管理器调 Cancel 例程时一定拿着它；Cancel 例程必须放掉它才能返回（放管理器的那把时传 Irp->CancelIrql；自己后来拿的按最近一次 IoAcquireCancelSpinLock 返回的 IRQL 放）；拿锁区间越短越好；**拿着任何自旋锁都不许调 IoCompleteRequest**（死锁）。用 I/O 管理器设备队列的驱动：除 Cancel 例程外，任何改 IRP 可取消状态的例程须先 IoAcquireCancelSpinLock（拿锁 = 别人此刻不能改这个 IRP 的状态，I/O 管理器也不能调你的 Cancel 例程）。
- **驱动自带锁范式（性能优化）**：自己管队列的驱动可用自己的锁保护队列，调 IoSetCancelRoutine 前不必拿系统取消锁——但必须查 IoSetCancelRoutine 的返回值。**返回值是竞态的唯一真相**：返回非 NULL = Cancel 例程没跑（也没正要跑），当前例程可以处置 IRP；返回 NULL = Cancel 例程已跑或正要跑，当前例程必须撒手，让 Cancel 例程完成它。指南给的入队/取出/Cancel 三段样板，核心就是两次 IoSetCancelRoutine + 查 Cancel 标志 + 查返回值：
  - 入队时：拿自己锁 → 入队 → IoMarkIrpPending → IoSetCancelRoutine 设例程 → 查 Irp->Cancel：若置位，再调一次 IoSetCancelRoutine 清零——返回非 NULL 说明 Cancel 例程没跑，自己摘队、放锁、STATUS_CANCELLED 完成；返回 NULL 说明 Cancel 例程已跑/正要跑，IRP 留在队里、撒手、返回 PENDING。
  - 取出时：拿自己锁 → 摘队首 → IoSetCancelRoutine 清零——返回非 NULL 才能用这个 IRP；返回 NULL 说明 Cancel 例程会完成它，把 IRP 的 ListEntry 指回自己（InitializeListHead 自指，防止 Cancel 例程摘队时链表已变、摘错 corrupt 链表），撒手、取下一个。
  - Cancel 例程里：先 IoReleaseCancelSpinLock（放系统锁，且须在不拿其他锁时放，保证退出 IRQL 正确）→ 拿自己锁 → 摘 IRP → 放自己锁 → 填 STATUS_CANCELLED → IoCompleteRequest（全程不拿锁完成）。
- **带 StartIo 的驱动的 Cancel 例程**：I/O 管理器只在有 StartIo 的驱动才维护 DeviceObject->CurrentIrp。Cancel 例程：① 先看输入 IRP 是不是 CurrentIrp——是，说明设备正在办它、很快会完成，放系统锁、直接返回；② 不是 → KeRemoveEntryDeviceQueue 测它在不在设备队里（不能用 KeRemoveDeviceQueue / KeRemoveByKeyDeviceQueue——不能假设它在队里哪个位置）：在队里 → 摘掉、放锁、STATUS_CANCELLED、完成；不在 → 放锁、返回。
- **不带 StartIo 的驱动的 Cancel 例程**：自己维护“当前在办哪个 IRP”的状态，每个队配一个 Cancel 例程；Cancel 例程：放系统锁 → 拿队锁 → 走队找 Cancel 置位的 IRP → 找到了：摘队、放锁、STATUS_CANCELLED、启动下一个排队 IRP、完成；没找到：默认该 IRP 已在办，放锁、返回。
- **高层驱动没有 Cancel 例程时怎么办**：高层驱动交出 IRP 后就不再拥有它、管不了下层怎么取消——但可以在 IoCallDriver 之前用 IoSetCompletionRoutine 挂 InvokeOnCancel = TRUE，这样无论下层取消还是完成，完成例程都会被调，高层至少能“看见”取消。**禁**：高层驱动只能 IoCancelIrp 自己分配的 IRP；没分配过的 IRP 不许取消；且 IoCancelIrp 不保证结果一定是 STATUS_CANCELLED（另一线程可能正在完成它），要确认须靠 InvokeOnCancel 的完成例程。
- **超时兜底**：IRP 若 5 分钟未完成，I/O 管理器视为超时——把它从线程上摘除、给当前持有它的设备记一条错误日志。所以“可能长时间挂着的请求必须可取消”，否则超时也救不了它。
- **大容量存储驱动通常不需要 Cancel 例程**：两条理由——上层文件系统一般替它处理文件 I/O 的取消；大容量存储 IRP 处理得极快，来不及被取消。通常是链上最高层驱动负责取消。

### 详解七：复用 IRP 的禁区

- **能复用**：驱动自己分配的 IRP——裸分配（ExAllocatePoolWithTag + IoInitializeIrp）后可用 IoInitializeIrp / IoReuseIrp 重置；IoAllocateIrp 建的（Win2000 起）可用 IoReuseIrp 重置；IoAllocateIrp 且 ChargeQuota = FALSE 的可用 IoInitializeIrp 重置（98/Me 兼容打法，2000+ 应优先 IoReuseIrp）。
- **不能复用**：I/O 管理器发下来的 IRP；IoMakeAssociatedIrp / IoBuildSynchronousFsdRequest / IoBuildAsynchronousFsdRequest / IoBuildDeviceIoControlRequest 建的 IRP（包括 IoBuildDeviceIoControlRequest 建的设备控制 IRP——建 IOCTL 请求那节明说“不能复用”）。

### 详解八：IOCTL——32 位编码的四字段

- **两条路**：用户态 DeviceIoControl → IRP_MJ_DEVICE_CONTROL（到栈顶）；驱动之间私下 → IRP_MJ_DEVICE_CONTROL 或 IRP_MJ_INTERNAL_DEVICE_CONTROL（用户态发不出 INTERNAL）。公开 IOCTL 是微软定义并文档化的（如 SCSI 端口 IOCTL、i8042 鼠标内部控制请求）；私有 IOCTL 是厂商自己头文件里定义的、只供自家组件互通——编码方式没区别，区别只在“哪些内部码能用”。
- **CTL_CODE(DeviceType, Function, Method, Access)**：
  - **DeviceType**：须与设备对象的 DeviceType 成员一致。＜ 0x8000 归微软保留；≥ 0x8000 厂商可用（置 Common 位）。
  - **Function**：操作码。＜ 0x800 归微软保留；≥ 0x800 厂商可用（置 Custom 位）。**定义新 IOCTL 时 Function 必须 ≥ 0x800**（安全规则之一）。
  - **Method（TransferType）**：数据怎么在调用者与驱动之间传——METHOD_BUFFERED（缓冲，多数设备与中间驱动的 IOCTL 默认）、METHOD_IN_DIRECT / METHOD_OUT_DIRECT（直接，IN = 调用者给驱动数据、OUT = 驱动给调用者数据）、METHOD_NEITHER（两者皆非）。取字段宏：DEVICE_TYPE_FROM_CTL_CODE、METHOD_FROM_CTL_CODE（方法占低 2 位）。
  - **Access（RequiredAccess）**：调用者打开设备文件对象时必须已请求的权限，I/O 管理器只在权限够时才把该 IOCTL 的 IRP 发给驱动——FILE_ANY_ACCESS（有句柄就行；系统对部分 IOCTL 用它，如独占设备的某些码；**新 IOCTL 用它之前必须绝对确信“无限制访问设备不会给恶意用户留口子”**）、FILE_READ_DATA（须读权限，设备 → 系统内存）、FILE_WRITE_DATA（须写权限，系统内存 → 设备），读写可或起来。驱动可用 IoValidateDeviceIoControlAccess 做比 RequiredAccess 更严的动态检查。
- **驱动内建 IOCTL 请求六步**：① 分配/复用 IRP（IoBuildDeviceIoControlRequest 专建，或 IoAllocateIrp / IoReuseIrp / IoInitializeIrp 通建；IoBuildDeviceIoControlRequest 建的不许复用）；② 填下层联（IOCTL 码 + 参数）；③ 若异步完成，先 KeInitializeEvent 建通知事件；④ IoSetCompletionRoutine（看下层结果、复用或释放自建 IRP）；⑤ IoCallDriver；⑥ 返回 PENDING 则 KeWaitForSingleObject 等事件（完成例程 signal 它；事件若是栈上局部变量，等时 WaitMode 必须用 KernelMode，防栈被换页）。**禁**：IOCTL 参数里几乎不嵌指针（SCSI 例外）——SystemBuffer、MdlAddress、Type3InputBuffer 里不含指向别的缓冲的指针，也不含含指针的结构（系统定义码）；配对的 class/port 驱动私下传驱动分配内存的内嵌指针时，须保证同一时刻只有一个驱动碰它、且 port 驱动在任意线程上下文都能访问。

### 详解九：缓冲描述——三种 Method 下驱动手里拿到的东西

| Method | 输入缓冲在哪 | 输出缓冲在哪 | 长度在哪 |
|---|---|---|---|
| METHOD_BUFFERED | Irp->AssociatedIrp.SystemBuffer（与输出共用同一块） | 同一块 SystemBuffer（驱动先读输入、再覆写输出） | InputBufferLength / OutputBufferLength；系统分配 max(两者) |
| METHOD_IN_DIRECT | Irp->AssociatedIrp.SystemBuffer | Irp->MdlAddress 的 MDL（名义输出，实际可作输入或输出；IN 保证执行线程对它有读权限） | 输入长 InputBufferLength；输出长 OutputBufferLength（MDL 描述的缓冲大小） |
| METHOD_OUT_DIRECT | Irp->AssociatedIrp.SystemBuffer | Irp->MdlAddress 的 MDL（OUT 保证执行线程对它有写权限） | 同上 |
| METHOD_NEITHER | Parameters.DeviceIoControl.Type3InputBuffer（用户态虚拟地址，**未经验证、未经映射**） | Irp->UserBuffer（用户态虚拟地址，同样未经验证映射） | InputBufferLength / OutputBufferLength |

- **METHOD_NEITHER 的使用条件**：驱动必须保证自己跑在发起请求的线程上下文中——只有最高层驱动能保证（下层驱动在请求线程等完成、或上层再叠一层时，上下文就不是请求线程了）。所以 NEITHER 几乎不用于发给低层设备驱动的码；最高层收到 NEITHER 请求时，必须自己决定改走缓冲还是直接（探查、锁页），并把对用户缓冲的访问包在结构化异常处理里——否则调用者可能在驱动用数据前改掉数据、或在驱动访问时被换出。

### 详解十：IOCTL 安全规则（定义时三条 + 处理时六条）

- **定义时**：① Function 码 ≥ 0x800；② 必写 RequiredAccess（I/O 管理器靠它挡权限不够的调用者）；③ 不定义“让调用者读写不特定内核内存区域”的码。
- **处理时**：① 分发例程测 IOCTL 码必须测整个 32 位值（不能只测 Function 字段——Method/Access 位也是码的一部分，漏测等于留别名）；② 可用 IoValidateDeviceIoControlAccess 做更严的动态检查；③ 读写 SystemBuffer 不许超过它实际大小——永远先查 InputBufferLength / OutputBufferLength；④ 给应用的输出缓冲须先清零（防把敏感数据顺带拷给应用）；⑤ DIRECT 方式须查 MmGetSystemAddressForMdlSafe 的 NULL 返回（映射失败或零长缓冲）；⑥ NEITHER 方式走专节（探查 + 异常处理，见详解十一）。
- **设备类型专属义务**：替换系统驱动、支持已有类型设备、夹在两系统驱动之间的中间驱动，都必须处理系统驱动处理的那套 IRP_MJ_XXX 与 IOCTL_XXX（哪怕要模拟系统驱动的行为），否则会“打破应用”。

### 详解十一：数据进出三方式——Buffered / Direct / Neither 的完整图

- **谁定方式**：READ/WRITE 由设备对象 Flags 定（DO_BUFFERED_IO / DO_DIRECT_IO，两者皆无 = Neither）；DEVICE_CONTROL / INTERNAL_DEVICE_CONTROL 由 IOCTL 码的 TransferType 定。**栈内一致性**：同一请求，栈内各层必须用同一种方式（最高层可例外用 Neither，不管下层用什么）。
- **Buffered I/O**：适用小数据、交互慢速设备（视频、键盘、鼠标、串口、并口）。流程：I/O 管理器检查用户缓冲可访问性 → ExAllocatePoolWithTag 分配等大非分页系统缓冲 → 写请求先把用户数据拷进系统缓冲再发 IRP；读请求驱动把数据读进系统缓冲、完成时 I/O 管理器再拷回用户缓冲并释放系统缓冲。优点：请求线程可被换出、物理内存可被挪用，系统缓冲地址在完成前一直有效；缺点：两次拷贝 + 非分页池分配——**大数据/多页传输不要用缓冲**（非分页池碎片化后 I/O 管理器分不出大块连续系统缓冲）。直接 I/O 的驱动通常也对部分 IOCTL 用缓冲（公开 IOCTL 一律缓冲，见详解二）。
- **Direct I/O**：适用大数据快速传输（大容量存储，DMA 或 PIO）。流程：I/O 管理器（或 FSD）检查用户缓冲 → 建 MDL（不透明结构，虚拟地址范围 → 一组物理页）→ MmProbeAndLockPages 探查并锁页、填物理地址 → IRP 带 MdlAddress 发下层；完成前页一直锁着（MmUnlockPages 在完成后才调）。**两个警告**：① MDL 里的虚拟地址可能在 IRP 发出前就已不可见/失效——MmGetMdlVirtualAddress 的返回值**不能直接用来访存**，只能传给 MapTransfer 做 DMA；② I/O 管理器不给零长缓冲建 MDL。
  - **DMA 版**：AdapterControl 调 MmGetMdlVirtualAddress 取 MDL 基址 → MapTransfer 设系统 DMA 控制器 → 设备直接读写物理内存。期间要维护缓存一致性（KeFlushIoBuffers 等，第 9 批 DMA 深讲）。
  - **PIO 版**：驱动调 MmGetSystemAddressForMdlSafe 把 MDL 双重映射到系统空间地址（AliasBuff），再用系统空间地址读写。性能忠告：除非必须用虚拟地址，否则别双重映射（浪费系统页表项、可扩展性下降、页表项耗尽会崩溃）。
- **Neither I/O**：I/O 管理器只传用户态虚拟地址与大小，不验证、不映射。驱动收到后必须：① 在请求线程上下文中用 ProbeForRead / ProbeForWrite 探查地址范围与读写权限，访问包在驱动自己的异常处理里（用户线程可能在访问中途改权限或改数据，探查异常 → 返回错误）；② 三选一自处：自己做双重缓冲（仿缓冲）、自己建 MDL 并锁页（仿直接）、或全程在调用线程上下文中直接操作用户缓冲（仍须异常处理）。效果：驱动按每个 IRP 自己决定走缓冲、直接还是线程上下文直访，并自己管全部用户缓冲访问、双重缓冲与映射——等于把 I/O 管理器的活全揽过来。**这就是 NEITHER 危险的根**：探查、锁页、异常处理、上下文四项缺一不可，缺一项就是经典漏洞（TOCTOU、任意地址读写）。
- **IRP 优先级提示（小附）**：Vista 起，IRP 可带 IO_PRIORITY_HINT，驱动排队时可据此定先后——只是提示，不是保证。

### 详解十二：最底层与中间层处理走查（把本批串起来）

- **最底层（以带 StartIo + 系统 DMA 的大容量存储驱动为例）**：分发（DDDispatchReadWrite）→ IoGetCurrentIrpStackLocation → IoMarkIrpPending → IoStartPacket（忙则进队、闲则当场 StartIo；可带 Key 定队内顺序）→ StartIo（拆部分传输、备状态）→ AllocateAdapterChannel（DMA 控制器空闲时调 AdapterControl）→ AdapterControl 调 MapTransfer 设 DMA → 编程设备 → ISR（止中断）→ IoRequestDpc → DpcForIsr（**先 IoStartNextPacket 再 IoCompleteRequest**，顺序不能反）→ 填状态块（STATUS_SUCCESS + 实际字节数）→ 完成。**大容量存储驱动调 IoStartPacket 时通常不带 Cancel 例程**：上层文件系统替它处理取消，且它的 IRP 处理得极快。
- **中间层（以镜像驱动为例）**：分发（DDDispatchWrite）→ 验参数 → IoAllocateIrp 裁多份（份数 = 下层链层数，可多裁一联给自己存上下文——先 IoSetNextIrpStackLocation 再 IoGetCurrentIrpStackLocation）→ IoGetNextIrpStackLocation 填各份的下层联（功能码、缓冲指针、长度照抄）→ 每份挂 IoCompletion（三开关全开）→ IoCallDriver 逐份送出（并行镜像）→ 完成例程里维护计数（存在原始 IRP 自留的那一联里）：每回来一份，计数减一、IoFreeIrp 释掉这份、返回 STATUS_MORE_PROCESSING_REQUIRED；计数到零时，把最后一份的状态块拷进原始 IRP、释掉最后一份、IoCompleteRequest 完成原始 IRP、仍返回 STATUS_MORE_PROCESSING_REQUIRED（让这份子 IRP 的完成处理就此打住——它不关联任何线程，不该再往上走）。任一份失败 → 记错误日志、做镜像数据恢复。
- **速查表（Cheat sheet）的定位**：指南末尾的速查表是 12+ 个带注释的代码场景——前 5 个讲“分发例程里怎么转发”（转发后不管、转发后等、转发后异步完成等），后 7 个讲“怎么自建 IRP 发下去”。概念上没有超出第 6、7 批，价值在代码形态：**完成例程返回值只有两种语义**——STATUS_MORE_PROCESSING_REQUIRED = 截停（栈位置不动、直接返回），其他任何值 = 继续往上完成（所以用 STATUS_SUCCESS，并有个别名 STATUS_CONTINUE_COMPLETION，0 值加载最快）。查代码时先看完成例程返回哪一种，再看它是转发还是自建。

---

## 四、解读：站在审计与逆向的一侧看这一章

**1. 取消是驱动竞态 bug 的重灾区，本批给了封竞态的完整工具箱。** WRK 第 7 批讲了 IRP 完成语义（PENDING、PendingReturned、走访、APC 收尾），本批补上另一半：取消语义（Cancel 标志、CancelRoutine 指针、取消自旋锁、IoSetCancelRoutine 返回值）。审计时看到这几处，红灯直接亮：① IoSetCancelRoutine 返回值不查（等于没防竞态）；② 拿着自旋锁调 IoCompleteRequest（死锁）；③ Cancel 例程返回前没放系统取消锁（IRQL 失衡）；④ 入队后还假设 IRP 状态不变（下一行代码时 IRP 可能已被取消）；⑤ 高层驱动取消了没分配过的 IRP（越权取消）。

**2. IOCTL 是用户态到内核的正门，第 7 批的检查表就是 BYOVD 审计的检查表。** 一个 DEVICE_CONTROL 分发例程，审计时按顺序问：Function 码是不是 ≥ 0x800（厂商码）？Access 位是不是 FILE_ANY_ACCESS（无限制访问）？Method 是不是 NEITHER（且有没有 Probe + 异常处理）？长度有没有核对 InputBufferLength / OutputBufferLength？输出缓冲有没有清零？DIRECT 有没有判 MmGetSystemAddressForMdlSafe 的 NULL？IOCTL 码测的是整个 32 位还是只测 Function 字段？这七问答完，一个 IOCTL 入口的风险等级基本就定了——第 5 批讲的“默认 SD 极宽是 BYOVD 入口”是设备打开层面的入口，本批是请求处理层面的入口，两处合起来才是完整的 BYOVD 审计面。

**3. 队列选型看设备，不看喜好。** 一次一件事 → StartIo；能并行 → 互锁队列或 CSQ；有自建线程 → 互锁队列；新驱动自己管队列 → CSQ（框架替你管取消，别手写）。逆向时看到驱动调 IoStartPacket，就知道它是单操作设备、走系统设备队列；看到 IoCsqInsertIrp，就知道它用取消安全框架、取消逻辑在系统例程里；看到 ExInterlockedInsertTailList + 自建线程，就知道它在手写队列——手写队列的取消同步是重点审计区。

**4. 与第 6 批的咬合**：第 6 批讲流动（下传、自建、完成），本批讲停留（排队）与打断（取消）与翻译（IOCTL 缓冲）。“PENDING 后不许碰 IRP”与“取消三竞态点”合起来，才是 IRP 生命周期的完整约束：IRP 一旦交出去（返回 PENDING 或入队），它的状态就不再归你——完成方与取消方都可能在任意时刻动它，唯一的同步工具是锁 + 返回值，不是时序假设。

**5. 与第 8、9 批的预告**：StartIo 里“与 ISR 共享的状态必须经 SynchCritSection 碰”是第 8 批（中断、DPC 与同步）的主战场；AdapterControl / MapTransfer / 缓存一致性是第 9 批（DMA）的主战场。本批只讲到交接点，不展开。

---

## 五、易混点辨析

1. **KeInsertDeviceQueue 返回 FALSE 不是失败**。它表示队列原来是空的，IRP 根本没进队、队列状态被置为 Busy——调用者必须把这个 IRP 交给别的例程处理，丢了就等于请求丢失。
2. **StartIo 不等于分发例程**。分发是第一阶段（验参数、定去向），StartIo 是第二阶段（启动设备操作），且 StartIo 在 DISPATCH_LEVEL、任意线程上下文——不能等、不能碰分页内存、不能推迟锁用户缓冲。
3. **高层驱动有 StartIo 是反模式**。串行化瓶颈 + 抬整条链下层分发例程的 IRQL，系统自带高层驱动没有一个这样干。
4. **取消自旋锁只有一把，但驱动自带锁不是替代品**。系统调 Cancel 例程时照样先拿系统锁；自带锁只是让驱动少跟全系统抢同一把锁，Cancel 例程里仍须先放系统锁。
5. **IoSetCancelRoutine 的返回值是竞态的唯一真相**。返回非 NULL = Cancel 例程没跑，当前例程可处置；返回 NULL = 已跑或正要跑，当前例程必须撒手。查 Cancel 标志不等于查返回值——标志置位时例程可能还没跑。
6. **Cancel 例程的第一件事是放锁，不是干活**。I/O 管理器调它时拿着系统取消锁；不放锁就调 IoCompleteRequest 是死锁，不放锁就返回是 IRQL 失衡。
7. **CurrentIrp 只在有 StartIo 的驱动里有效**。带 StartIo 的 Cancel 例程先比 CurrentIrp——是，说明正在办、很快完成，直接返回；不是，再测队里有没有。不带 StartIo 的驱动没有 CurrentIrp 可比，只能走队找 Cancel 置位的。
8. **METHOD_NEITHER 不是“省事”，是“全揽”**。I/O 管理器什么都不替你干：探查、锁页、异常处理、上下文，四项全由驱动自己负责，缺一项就是漏洞；且只有最高层驱动能保证请求线程上下文。
9. **DIRECT 方式下 MDL 的虚拟地址不能直接访存**。MmGetMdlVirtualAddress 的返回值只能传给 MapTransfer；PIO 要用 MmGetSystemAddressForMdlSafe 双重映射；且零长缓冲根本没有 MDL。
10. **复用禁区与 IOCTL 码测试禁区**。IoBuild* 建的 IRP 一律不许复用；IOCTL 码必须测整个 32 位值，只测 Function 字段等于留别名。

---

## 六、自查问题（只给问题，自行回原文核对）

1. 设备一次只做一件事、能并行多件事、有自建线程三种情形，排队机制各应选哪一种？为什么新驱动自己管队列时指南首选 CSQ？
2. KeInsertDeviceQueue 返回 FALSE 时，IRP 在哪里？调用者必须做什么？为什么“从空但 Busy 的队里取一次”会把状态复位为 Not-Busy？
3. StartIo 跑在什么 IRQL、什么线程上下文？这两条约束分别禁了它干哪几件事？为什么最高层驱动不能把锁用户缓冲推迟到 StartIo？
4. 高层驱动有 StartIo 时，为什么会“抬整条链下层分发例程的 IRQL”？这为什么破坏互操作性？
5. 取消的三处竞态点各是什么？IoSetCancelRoutine 的返回值为什么比 Irp->Cancel 标志更能说明问题？
6. 驱动自带锁范式里，入队时为什么要调两次 IoSetCancelRoutine？取出时 ListEntry 为什么要自指（InitializeListHead）？
7. 一个 DEVICE_CONTROL 分发例程：Function 码、Access 位、Method、长度校验、输出清零、DIRECT 的 NULL 检查、NEITHER 的探查，七项中哪几项缺失时风险最高？为什么？
8. METHOD_BUFFERED、METHOD_IN_DIRECT、METHOD_OUT_DIRECT、METHOD_NEITHER 四种方式下，驱动手里拿到的输入缓冲与输出缓冲分别是什么？栈内各层对同一请求的方式必须满足什么一致性？

---

## 附：追问补记——探查锁页与双重映射不在同一个上下文（2026-10-28）

- **关键**：探查锁页（第 1 步）与双重映射（第 2 步）不在同一个线程上下文里。双重映射那一步根本不需要确定“当前用户缓冲区是不是当前发起进程的”——到那一步时，缓冲区已经不再以“用户地址”的形式存在，只以 MDL 里的 PFN 列表存在；双重映射只认 PFN，不认进程。
- **第 1 步在请求进程上下文**：Direct I/O 下，I/O 管理器在系统调用路径里、请求线程还没离开时建 IRP、探查并锁页；FSD 自建 IRP 时，FSD 自己在用户地址还有效的时候干。此时用户地址有意义：翻译成 PFN + 钉住。翻译一旦完成，就和进程脱钩。
- **第 2 步在任意线程上下文**：下层驱动 PIO 时，拿 MDL 里的 PFN 做双重映射，不碰用户虚拟地址，不需要知道请求进程是谁。系统地址空间统一，解决的是“CPU 在哪都能访存”；MDL 解决的是“知道要访存的是哪批物理页”。两个脱钩各管一摊。
- **为什么不能跳过第 1 步直接做第 2 步**：双重映射映射的是 PFN，不是用户地址；没有第 1 步就没有 PFN 列表，MmGetSystemAddressForMdlSafe 没有输入。且不锁页的话，PFN 可能在传输中途被回收给别的用途，系统地址会指到别人的数据——所以页必须钉住直到传输完成。

---

## 附：追问补记——定死的是缓冲方式，不是 PIO（2026-10-28）

- **定死的是缓冲方式（Buffered / Direct / Neither）**：IRP 的数据载体在建 IRP 时就按方式准备好了（Buffered 分 SystemBuffer 并拷贝；Direct 建 MDL 并锁页；Neither 只传虚拟地址），栈内各层对同一请求必须用同一种方式。中途改 = 下层按 MDL 读、上层给 SystemBuffer，对不上号。所以方式是 IRP 的“出身”，中途不能改。
- **PIO vs DMA 不是按请求定的，是按设备定的**：DMA 设备永远 DMA，PIO 设备永远 PIO——驱动针对自己的硬件写死。严格说决定点比请求发起还早：设备对象创建时的 DO_BUFFERED_IO / DO_DIRECT_IO 标志（或 IOCTL 码里的 Method 位）定了方式；驱动开发时就定了这块硬件用 DMA 还是 PIO。请求发起时（建 IRP）只是执行既定方式。
- **精度**：双能力的硬件，驱动可以在 StartIo 里按传输大小临时选手段（小数据 PIO 省 DMA 编程开销、大数据 DMA）——但这只是搬运手段的临时选择，不改 IRP 的方式；前提永远是 IRP 是 Direct（有 MDL）。Buffered 的 IRP 没有 MDL，PIO 双重映射无从谈起。
- **别和上一条混淆**：探查锁页必须在请求进程上下文做，是因为“翻译依赖进程地址空间”；方式定死，是因为“数据载体已按方式建好”。两个“必须早”的原因不同。

---

## 附：追问补记——PIO/DMA 是 Direct 之下的两种搬运手段（2026-10-28）

- **定位**：Direct 方式下，MDL 是共同输入，两种搬运手段：DMA（MapTransfer 拿 PFN，控制器搬）、PIO（MmGetSystemAddressForMdlSafe 双重映射拿系统地址，CPU 搬）。
- **术语精度**：叫它们“手段”比“方式”准——Windows 里“方式”（METHOD_*）指 Buffered / Direct / Neither 这层；PIO/DMA 是 Direct 之下的搬运手段，和 METHOD_* 不并列。
- **适用范围精度**：DO_DIRECT_IO 管的是读写请求；IOCTL 的 Direct 靠 CTL_CODE 里的 METHOD_IN_DIRECT / METHOD_OUT_DIRECT，两者进的是同一个 Direct 体系（建 MDL、锁页）。
- **边注**：DMA 手段本身不限于 Direct（Buffered 的 SystemBuffer 是非分页内存，硬件上也能 DMA）；指南把这对概念放在 Direct 一节，是因为 Direct 下 MDL 同时服务于两种手段，对比最干净。
