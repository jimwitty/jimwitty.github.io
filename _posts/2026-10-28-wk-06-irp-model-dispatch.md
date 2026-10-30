---
layout: post
title: "Win 内核驱动 第 6 批：IRP（上）——模型与分发"
categories: windows-kernel
series_order: 6
---

- **系列**：Windows 内核驱动循序渐进学习（教材：微软官方《内核模式驱动程序体系结构设计指南》，WDM 中心）
- **本批对应原文档**：「Handling IRPs」章前半——Overview of the Windows I/O Model / End-User I/O Requests and File Objects / Example I/O Request - An Overview / Example I/O Request - The Details / Driver Thread Context / Points to Consider about User I/O Requests / IRP Major Function Codes（15 个主功能码小节）/ I/O Stack Locations / I/O Status Blocks / Passing IRPs down the Driver Stack / Creating IRPs for Lower-Level Drivers / Completing IRPs（When / How / When in Dispatch）/ Using IoCompletion Routines（Registering / Implementing）。
- **取材说明**：正文取自微软官方同名设计指南 PDF（2019 版离线打包）「Handling IRPs」章前半逐节研读。本 PDF 的来源仓库 tpn/pdfs 是第三方文档收集仓库、非微软官方；指南本身是微软官方文档 2019 版离线版。各小节篇幅均为 2–6 分钟阅读量，但 IRP 是全书的枢纽章，密度极高，本笔记按原文范围提炼、不越界展开；队列/StartIo、取消、IOCTL 缓冲方式属下半章，留待第 7 批。
- **学习目标**：学完本批，你应该能说清：① 一个用户态读请求从 Win32 调用到磁盘驱动再回来的完整走查，每一步是谁干的；② IRP 与 I/O 堆栈位置到底是什么关系，为什么“每层只能看自己那一联”；③ 15 个主功能码各管什么，中间层驱动漏掉一个会出什么事；④ 下传、自建、完成三条路的规矩各是什么，IoCompletion 例程在其中扮演什么角色；⑤ CLEANUP 与 CLOSE 为什么是两个不同的请求；⑥ 为什么说“I/O 管理器把堆栈位置清零”这条规则决定了驱动之间只能用状态块通信。

---

## 一、本批地图：一个读请求的全程走查

以“应用读磁盘文件”为例，把本批所有机制串在一条线上。**关键事实：用户态永远看不到驱动，它只看到文件对象句柄；驱动永远看不到应用，它只看到 IRP。**

```
用户态应用
ReadFile(句柄, 缓冲区, 长度)
│
▼ ——责任方：Win32 子系统（受保护子系统）
把 ReadFile 翻译成 I/O 系统服务调用（NtReadFile）
子系统不知道、也不需要知道下面是哪条驱动链
│
▼ ——责任方：I/O 管理器
按句柄找到文件对象 → 找到设备对象 → 分配并初始化 IRP
IRP 的堆栈位置数 = 这条设备堆栈的层数（文件系统层 + 磁盘驱动层……）
IRP 顶层位置填好：主功能码 IRP_MJ_READ、长度、偏移、缓冲区描述
│
▼ ——责任方：文件系统驱动 FSD（最高层驱动，分发例程）
读自己那一联：要读什么、读多少
决定：自己拆包还是原样下传？
· 拆包：调 IoAllocateIrp 裁一个或多个子 IRP（零填充的下层位置），
  给每个子 IRP 挂自己的 IoCompletion 例程（必须挂，否则子 IRP 没人释放），
  再 IoGetNextIrpStackLocation 填下层位置、IoCallDriver 送出
· 原样下传：IoCopyCurrentIrpStackLocationToNext 或 IoSkipCurrentIrpStackLocation
  填下层位置（可选挂 IoCompletion 例程），IoCallDriver 送出
返回：异步请求通常返回 STATUS_PENDING（“我接了，办结另行通知”）
│
▼ ——责任方：最底层磁盘驱动（分发例程）
只读自己那一联：主功能码、目标设备对象、参数
自己先验一遍参数（高层验过也不能省）；验不过 → 当场完成、返回错误
验过 → IoMarkIrpPending → IoStartPacket（设备忙则排队、不忙则进 StartIo）
返回 STATUS_PENDING；分发例程就此结束，线程该干嘛干嘛
│
▼ ——责任方：硬件 + ISR + DPC
设备中断 → ISR 只干急事（止中断、存现场），调 IoRequestDpc 排 DPC
DPC 在较低优先级上办结：填 I/O 状态块（Status + 实际字节数），
调 IoCompleteRequest —— 完成走访从此开始往上走
│
▼ ——责任方：I/O 管理器（完成走访）
把最底层那一联清零 → 按注册情况调 FSD 的 IoCompletion 例程
（例程看状态块：成功/失败/取消；决定重试、收尾、释放子 IRP）
FSD 收齐子 IRP 结果 → 填原始 IRP 的状态块 → IoCompleteRequest 完成原始 IRP
│
▼ ——责任方：I/O 管理器（收尾）
读原始 IRP 的状态块 → 把 NTSTATUS 还给子系统 → 释放自己分配的 IRP
同步请求：唤醒等在事件上的请求线程；异步请求：结果经事件/APC 送达
```

**打开文件是同一条线的变体**（Example I/O Request - An Overview 的 10 步）：子系统调打开服务 → I/O 管理器请对象管理器解析名字、请 SRM 查访问权 → 卷若未挂载，先挂起打开请求、逐个问文件系统谁认这个卷，挂上后再恢复 → 分配 IRP（对驱动来说，“打开”就是一个 IRP_MJ_CREATE）→ 送文件系统 → 各层处理 → 状态块回 I/O 管理器 → 成功则把文件对象句柄还给子系统。

---

## 二、核心要点

1. **I/O 模型四特征**：① 所有 I/O 请求一律是 IRP，不管哪一层驱动；② 分层：I/O 管理器出口 I/O 系统服务，用户态子系统调服务，I/O 管理器把调用翻译成一个或多个 IRP、路由过层层驱动直到物理设备；③ 标准例程：驱动按一组必需/可选的标准例程实现，分工统一；④ 对象化：驱动、设备、硬件都表现为对象，驱动靠 I/O 管理器与其他组件出口的支撑例程操作对象。另外：PnP 与电源请求同样走 IRP（只是换了主功能码）。
2. **隔离是双向的**：子系统不知道驱动链（I/O 管理器替它挡着机器相关的配置与驱动实现）；驱动不知道请求来自哪个子系统（I/O 管理器给所有驱动同一套 I/O 模型、同一套支撑例程、同一个“请求者—驱动”接口）。驱动之间也靠 IRP 隔离：每层只认自己的堆栈位置。
3. **IRP 是“多联批文”**：一个 IRP 带一个堆栈位置数组，数组元素数 = 链上驱动数；每层驱动拥有其中一联。联上写：主功能码（办什么事）、次功能码（PnP/电源/部分 FSD/SCSI 请求才有，办哪种子事）、操作参数（长度、偏移、缓冲区等）、目标设备对象指针、文件对象指针（文件系统驱动看它，其他驱动通常忽略）。
4. **访问规则是铁律**：高层驱动只能碰自己那一联和下一层那一联；最底层驱动只能碰自己那一联。原因是指南明说的：你无法预知将来会不会有新驱动插到你下面（或上面）——镜像驱动、筛选驱动随时可能加进来。所以高层必须假设“新插进来的驱动处理的主功能码和被它顶替的那层一样”，不能越级读写。
5. **完成时清零是通信规则的根**：I/O 管理器在完成走访中，每经过一层就把该层的堆栈位置清零再调上层的完成例程。后果：上层完成例程**只能**从 I/O 状态块判断下层干得怎么样，看不到下层联上的参数。驱动之间想“走后门”传信息（比如读别层的联）既不可移植、也不被允许——状态块是唯一合法的上行通道。
6. **主功能码是分发依据**：驱动对象里每个 MajorFunction 表项对应一个 IRP_MJ_XXX；I/O 管理器按当前联的主功能码查表调分发例程。中间层漏掉下层处理的主功能码 = “断链”：应用往下送的请求到中间层就没人接，I/O 管理器对没有入口点的请求返回 STATUS_INVALID_DEVICE_REQUEST。
7. **下传、自建、完成是三条不同的路**，各有各的规矩（见详解四、五、六），混用就是 bug 之源。最简口诀：传下去的 IRP 归原主，自建的 IRP 归自建者；谁分配谁释放，I/O 管理器只释放它自己分配的。
8. **PENDING 是“已收、办结另行通知”的章**，不是结果：分发例程返回 STATUS_PENDING 后不得再碰 IRP；结果以后由完成走访 + 状态块 + 事件/APC 送达（这与 WRK 第 7 批讲的机制完全咬合：指南讲“必须怎么做”，WRK 讲“底下怎么实现的”）。
9. **线程上下文是“任意”的**：多数驱动在任意线程上下文中跑（当前是哪个线程就是哪个），所以 I/O 状态必须放在设备扩展里，不能依赖“请求线程”的身份。只有 FSD 是两截的：分发部分跑在请求线程里，另有一组系统工作线程（或 FSP）处理内部 IRP 队列；最底层驱动通常不需要自己的线程（除非设备准备过程长到影响性能，代价是上下文切换）。
10. **同步与异步走不同的完成形态**：同步请求（CREATE 之类）分发例程常直接返回 IoCallDriver 的结果；异步请求（READ/WRITE 之类）通常返回 STATUS_PENDING。同步请求一般不挂完成例程（用事件等）；异步请求挂完成例程是常态，自建 IRP 则必须挂。

---

## 三、关键知识点详解

### 详解一：文件对象——用户态与驱动之间的唯一界面

- 用户态代码（包括受保护子系统）只能看到“被 I/O 管理器管着的命名文件对象”，访问它的唯一凭证是句柄。打开一个命名文件时，子系统调 I/O 系统服务，传的可以是子系统自己的别名（符号链接），I/O 管理器负责解析成内核态的文件对象名、找到文件对象、找到对应的驱动。
- 对驱动来说，“打开”不是一个特殊动作，就是一个 IRP_MJ_CREATE。句柄诞生后，后续的读/写/设备控制都沿这个句柄进来，I/O 管理器按句柄找到文件对象、再找到设备堆栈。
- **为什么这样设计**：把“名字解析 + 权限检查 + 设备定位”收进 I/O 管理器，驱动就不用关心请求来自 Win32 还是 POSIX、来自哪个子系统；子系统也不用关心磁盘还是串口、驱动怎么实现。契约只有一个：文件对象 + IRP。

### 详解二：I/O 堆栈位置与 I/O 状态块

- **堆栈位置（IO_STACK_LOCATION）**：每联内容如核心要点 3。层数由设备堆栈决定：I/O 管理器给大容量存储驱动的 IRP 至少带两个位置（上面一定有文件系统）；给没有上层驱动的驱动只带一个位置。新驱动挂进链时，I/O 管理器会调整发给链上所有驱动的 IRP 的位置数——所以“层数”是动态的，驱动不能假设自己是第几层。
- **自建 IRP 的层数由自建者定**：按下一层驱动设备对象的 StackSize 值裁。注意：自建 IRP 里没有自建者自己的位置（除非自建者特意给自己留一联存完成例程用的上下文——见详解五）。
- **状态块（IO_STATUS_BLOCK）**：每个 IRP 自带一块，分 Status 与 Information 两栏。Status 说成没成（NTSTATUS）；Information 说细情：读写成功时是实际传输字节数，警告状态（如 STATUS_BUFFER_OVERFLOW）时也是传输字节数，其他成功请求按请求而异，出错时通常为零。完成例程判断下层干得如何，只看这一块。

### 详解三：15 个主功能码一览

| 主功能码 | 谁发 / 干什么 | 备注 |
|---|---|---|
| IRP_MJ_CREATE | 打开文件对象或设备对象的句柄（ZwCreateFile 的实际执行者） | 每个驱动都必须处理；参数：SecurityContext（DesiredAccess）、Options（高 8 位 CreateDisposition、低 24 位 CreateOptions）、ShareAccess；FileAttributes/EaLength 留给文件系统 |
| IRP_MJ_CLOSE | 最后一个句柄被关闭**且释放**，全部未完成 I/O 已完成或取消 | 每个驱动都必须处理（除非设备卸不掉，如扛着系统页面文件的磁盘）；恒在 PASSIVE_LEVEL；不一定在关闭进程的上下文中；一般是“ undo CREATE 时做的事”，独占设备可顺手复位硬件 |
| IRP_MJ_CLEANUP | 最后一个句柄被关闭（但因还有未完成 I/O，文件对象可能尚未释放） | 只在关闭进程的上下文中送达；驱动在此释放进程专属资源（锁住/映射的用户内存）；独占设备须完成全部排队 IRP（STATUS_CANCELLED），非独占则只取消与该文件对象相关的排队 IRP，再以 STATUS_SUCCESS 完成本 IRP |
| IRP_MJ_READ | 从设备传数据出来 | 参数含义随设备类型而异，详见 IFS 文档 |
| IRP_MJ_WRITE | 往设备传数据进去 | 同上 |
| IRP_MJ_DEVICE_CONTROL | 按 IOCTL 让设备执行设备类型专属操作（设置/复位等） | 用户态 DeviceIoControl 的入口；IOCTL 定义与缓冲方式是第 7 批 |
| IRP_MJ_INTERNAL_DEVICE_CONTROL | 驱动之间私下约定的 IOCTL | 只有配对的高层驱动发给低层驱动（IoBuildDeviceIoControlRequest）；不经过用户态 |
| IRP_MJ_FLUSH_BUFFERS | 把设备/驱动内部缓冲的数据刷出去 | 主要是大容量存储 |
| IRP_MJ_FILE_SYSTEM_CONTROL | 文件系统专属控制 | 只有文件系统驱动处理 |
| IRP_MJ_PNP | 即插即用操作 | PnP 管理器经 I/O 管理器发出；次功能码 IRP_MN_XXX 决定子事；第 10 批细讲 |
| IRP_MJ_POWER | 电源操作 | 电源管理器经 I/O 管理器发出；次功能码决定子事；**下传必须用 PoCallDriver，不能用 IoCallDriver**；第 11 批细讲 |
| IRP_MJ_QUERY_INFORMATION | 取文件/文件句柄的元数据 | 驱动可选择处理 |
| IRP_MJ_SET_INFORMATION | 改文件/文件句柄的元数据 | 驱动可选择处理 |
| IRP_MJ_SHUTDOWN | 文件系统通知“系统正在关机” | 驱动须先向系统登记（IoRegisterShutdownNotification 或 IoRegisterLastChanceShutdownNotification）才收得到；收到后须先把设备/内部缓冲的数据传完再完成；PnP 管理器在 IRQL ≤ APC_LEVEL、任意线程上下文中发出；一个关机过程里同一驱动可能收到多次 |
| IRP_MJ_SYSTEM_CONTROL | WMI 操作 | 每个驱动都必须提供 DispatchSystemControl；次功能码标识 WMI 动作；第 12 批讲 WMI 概念 |

- **“必须处理”与“可选处理”的分界**：CREATE、CLOSE、PNP、POWER、SYSTEM_CONTROL 是每个驱动的必答题；QUERY/SET_INFORMATION 是选答题；READ/WRITE/DEVICE_CONTROL 看设备类型。中间层驱动的额外义务：下层处理的主功能码，中间层必须有入口点（哪怕只是原样下传），否则断链。

### 详解四：下传——四步（Passing IRPs down the Driver Stack）

高层驱动分发例程收到 IRP，若自己不能当场办结，按以下四步下传：

1. **读自己那一联**：IoGetCurrentIrpStackLocation，验参数。验不过 → 当场完成、返回错误（越早上层拦截，整条链的吞吐越好）。
2. **填下一联**：二选一——IoSkipCurrentIrpStackLocation（下层共用本层位置：参数相同、完成例程相同，纯过路时用）或 IoCopyCurrentIrpStackLocationToNext（把本层参数复制到下一联：下层要相同参数时用；**完成例程不复制**，因为完成例程是按层现挂的）。也可以改下一联的部分参数（比如设备有传输上限时把大请求拆成多次部分传输、复用同一个 IRP）。
3. **挂完成例程（可选但有用）**：IoSetCompletionRoutine。原样下传时可选（用途：看下层办得如何、复用 IRP 做部分传输、维护自己追踪的 IRP 状态、对出错请求重试）；自建 IRP 时必选（见详解五）。
4. **送出并返回**：IoCallDriver 送出；返回 STATUS_PENDING（异步请求，如 READ/WRITE 的常态）或直接返回 IoCallDriver 的结果（同步请求，如 CREATE 的常态）。

最底层驱动的分发例程走另一条：IoMarkIrpPending → IoStartPacket（设备忙则进设备队列、不忙则进 StartIo；自己管队列的驱动走 Driver-Managed Queues，第 7 批）→ 返回 STATUS_PENDING。没有 StartIo 又要处理可取消 IRP 的驱动，必须注册 Cancel 例程或用 cancel-safe 队列（第 7 批）。

### 详解五：自建——四工具与六条规矩（Creating IRPs for Lower-Level Drivers）

**什么时候必须自建**：原 IRP 不能直接往下送时——要拆包（一变多）、要换操作（上面要读、下面要先问）、要换栈/换设备、要换缓冲区或线程（原 IRP 绑着请求线程，拿它干后台活会往请求人处投递事件/APC，乱套）。这正是 WRK 第 7 批“驱动自造 IRP 的七场景”的指南版，结论一致。

**四工具**：

- **IoAllocateIrp**：裁一张白纸——分配 IRP + 指定数量的零填充堆栈位置。自建者须自己填下一联（通常从原始 IRP 的本层位置复制、按需修改）。若自建者想在新 IRP 里给自己留一联存完成例程用的上下文，须先调 IoSetNextIrpStackLocation 再调 IoGetCurrentIrpStackLocation（顺序不能反）。
- **IoBuildAsynchronousFsdRequest**：按调用者给的参数直接把下层位置填好，专用于 READ / WRITE / FLUSH_BUFFERS / SHUTDOWN 的异步请求。代价：完成例程在新 IRP 里没有本地上下文可存，上下文须由驱动自己在常驻内存里另行维护。
- **IoMakeAssociatedIrp**：裁一张“从属 IRP”并挂到主 IRP 上。**只有最高层驱动能用**（中间层不能用——它收到的 IRP 可能本身已是从属 IRP，不能再挂一层）；最高层发完从属 IRP、给原始主 IRP 打上 IoMarkIrpPending 后即可返回，I/O 管理器在全部从属 IRP 办结后自动完成主 IRP。但若最高层给从属 IRP 挂了完成例程且例程返回 STATUS_MORE_PROCESSING_REQUIRED，I/O 管理器就不再自动完成主 IRP，例程须自己调 IoCompleteRequest 完成主 IRP。另外：从属 IRP 不能用于缓冲 I/O 请求。
- **IoBuildDeviceIoControlRequest / IoBuildSynchronousFsdRequest**：同步路。IoBuildSynchronousFsdRequest 只能在三种情形下调：驱动自建线程（能在自己的、非任意的线程上下文里等事件）、初始化或卸载时的系统线程上下文、建本来就是同步的操作（create、flush、shutdown、close、device control）的 IRP。实践中设备控制 IRP 更常用 IoBuildDeviceIoControlRequest。

**六条规矩**：

1. **PENDING 打在原始 IRP 上，不打在自建 IRP 上**——因为自建 IRP 的完成例程会释放它，打标记没有意义；分发例程发完所有自建 IRP 后返回 STATUS_PENDING。
2. **完成例程必须释放自建 IRP**：调 IoFreeIrp，且最好在完成原始 IRP 之前；完成原始 IRP 之前，须先释放全部自建 IRP（以及为它们分配的 MDL 等逐 IRP 资源）。
3. **自建 IRP 的完成例程若不能让原始 IRP 成功**，须把原始 IRP 的状态块改成导致失败的那个自建 IRP 返回的值。
4. **可移动介质要传线程指针**：做部分传输且下层可能是可移动介质驱动时，须把原始 IRP 的 Tail.Overlay.Thread 值抄进新 IRP——下层可能调 IoSetHardErrorOrVerifyDevice，靠这个指针找到用户线程弹“取消/重试/失败”对话框。
5. **下层看不出 IRP 的来历**：高层把自建（或复用）IRP 布置得与“来自文件系统或用户态应用”无异，下层驱动不需要、也不应该区分。
6. **中间层自建不用 IoMakeAssociatedIrp**：中间层自建用 IoAllocateIrp / IoBuildDeviceIoControlRequest / IoBuildSynchronousFsdRequest / IoBuildAsynchronousFsdRequest。

### 详解六：完成——三时机、五步、完成例程（Completing IRPs / IoCompletion Routines）

**发起完成的三时机**：① 参数无效或别的条件使处理无法推进；② 驱动不经下层、自己就能办完且办完了；③ IRP 被取消（第 7 批）。三者皆无 → 分发例程必须下传或自己接手处理，不能悬在半空。

**分发例程里当场完成的五步**：

1. 填状态块：Status 置 STATUS_SUCCESS 或适当的错误码；Information 按请求而异（读写成功 = 实际字节数；警告状态如 BUFFER_OVERFLOW = 传输字节数；出错通常为零）。
2. 调 IoCompleteRequest，PriorityBoost 传 IO_NO_INCREMENT（分发例程里当场完成时，请求者没等过，不需要优先级补偿；真正等过的异步完成由最底层驱动按设备类型给补偿值，高层沿用下层的值）。
3. 返回刚才填进状态块的那个 STATUS 值。**注意**：IoCompleteRequest 一调，IRP 就不再属于调用者——不能再从已完成 IRP 的状态块里取返回值（那是取已释放内存）。
4. **调 IoCompleteRequest 之前，先放掉手里拿着的自旋锁**。完成要走多久不确定（层层完成例程），且高层完成例程可能把 IRP 再往下送——若下层还拿着锁，就是死锁。
5. **下层返回了 STATUS_PENDING，就不要完成**，只有一个例外：高层用事件在完成例程与分发例程之间同步——完成例程 signal 事件并返回 STATUS_MORE_PROCESSING_REQUIRED 截停走访，分发例程等到事件后再调 IoCompleteRequest 恢复走访。

**分发例程里完成 vs 不完成**：高层分发例程通常只在“参数无效”或“CLEANUP”时当场完成；最底层分发例程在“参数无效 / 不支持 / CLEANUP / 不需要设备 I/O（如查设备当前模式，模式存在设备扩展里）”时当场完成，其他一律 IoMarkIrpPending + 排队 + 返回 STATUS_PENDING。真正要碰设备的完成，一般发生在 DpcForIsr 里（中断 → DPC → 完成，见本批地图）。

**IoCompletion 例程的注册（IoSetCompletionRoutine）**：

- **三开关**：InvokeOnSuccess（下层成功完成时调）、InvokeOnError（出错时调）、InvokeOnCancel（取消时调），可任意组合。自建 IRP 与复用 IRP 须三开关全开（InvokeOnCancel 通常不管会不会取消都开，为的是确保完成例程一定有机会释放自建 IRP 或核对复用状态）。
- **三大用途**：① 释放自建 IRP（IoAllocateIrp / IoBuildAsynchronousFsdRequest 建的必须有完成例程来释放）；② 复用原 IRP 做多次操作（部分传输），直到原始请求满足再完成；③ 对下层出错的请求重试（文件系统这类最高层更常见；重试次数须由分发例程定上限，不能无限重试）。
- **谁不能挂**：最底层驱动不能注册 IoCompletion 例程（它下面没人可监听）；同步 I/O 的 IRP 一般不挂（用事件等，如 IoBuildDeviceIoControlRequest / IoBuildSynchronousFsdRequest 的常态）。

**IoCompletion 例程的实现规则**：

- 入口给一个 Context 指针（注册时由分发例程提供），**上下文区不能分页**——因为完成例程可能在 DISPATCH_LEVEL 被调。
- 只能从状态块判断下层结果（堆栈位置已被清零，见核心要点 5）。
- 释放自建 IRP 前，先释放为它分配的逐 IRP 资源（如 IoAllocateMdl + IoBuildPartialMdl 建的 MDL 用 IoFreeMdl 释放）；释放与完成原始 IRP 的顺序：先释资源、再完成原始 IRP，且一定在返回前做完。
- **返回 STATUS_MORE_PROCESSING_REQUIRED = 截停**：I/O 管理器的完成处理当场停下，IRP 不继续往上、不释放；以后谁再调一次 IoCompleteRequest，走访从截停点上一层的完成例程继续。驱动自建 IRP 的完成例程释完子 IRP 后、复用/重试的完成例程把 IRP 再往下送后，都返回这个值。
- **PendingReturned 检查**：完成例程里查 Irp->PendingReturned，置了就须对本 IRP 调 IoMarkIrpPending（把“已收、办结另行通知”的章往上传），只有一个例外：分发例程在等事件的同步模式——完成例程 signal 事件 + 返回 STATUS_MORE_PROCESSING_REQUIRED，不打标记。
- **重试时的状态块复位**：重试前先把状态块复位为 STATUS_SUCCESS + Information 零（先存好上次的错误信息），重试计数由分发例程定、完成例程递减，耗尽则以错误完成原始 IRP；复用/重试时不要重复打 IoMarkIrpPending（分发例程打过一次就够，原始 IRP 在整条链完成前一直是 pending）。
- **高层完成例程的调用顺序**：下层全部完成例程跑完，才轮到上层——完成走访是自底向上的。

---

## 四、解读：站在审计与逆向的一侧看这一章

**1. 指南讲“规矩”，WRK 讲“实现”，两边咬合才算真懂。** 本批是指南的契约版：每层只能看自己那一联、完成时清零、谁分配谁释放、PENDING 后不许碰——这些在 WRK 第 7 批里都有对应的实现（IopfCallDriver 移指针、完成走访逐层清位置、PendingReturned 抄标记、特殊内核 APC 收尾）。逆向时看到驱动调 IoCallDriver + IoSetCompletionRoutine，就知道它在分层链里；看到它跳过某层联直接读别层的联，就知道它在走后门——指南明说这叫“损害可移植性与互操作性”，审计时这就是可疑点。

**2. 异步是整套机制存在的理由。** 设备慢、线程不能在高 IRQL 上等，所以才需要 PENDING（先接下、以后再说）+ 完成走访（以后再送）+ DPC（中断上下文里办结）。分发例程“返回 STATUS_PENDING 后不得再碰 IRP”不是礼貌，是内存安全：IRP 随时可能被完成方释放，再碰就是 use-after-free——这是驱动蓝屏的经典来源之一。

**3. 审计一张检查表（本批规矩直接导出）**：① 最底层有没有自己验参数（不能只信高层验过）；② 有没有漏主功能码入口点（断链）；③ 自建 IRP 有没有挂完成例程、有没有释放（泄漏）；④ 完成例程的上下文是不是分页内存（DISPATCH_LEVEL 下一碰就蓝屏）；⑤ 拿着自旋锁调没调 IoCompleteRequest（死锁）；⑥ 完成之后还碰不碰 IRP（use-after-free）；⑦ CLEANUP 里有没有取消与该文件对象相关的排队 IRP（句柄关了请求还跑）；⑧ 复用/重试有没有上限（无限重试 = 挂起）；⑨ POWER 下传有没有错用 IoCallDriver（必须用 PoCallDriver）。

**4. 与第 5 批的咬合**：第 5 批讲“谁能打开设备”（CREATE 路上的裁决），本批讲“打开之后请求怎么走”。CREATE 的 SecurityContext（DesiredAccess）是第 5 批裁决的结果，到了本批，它只是 CREATE 那一联上的一个参数——安全裁决在打开时做完，后续读写默认不再逐条查 ACL（这就是为什么“打开”是安全边界，而“读写”不是）。

**5. 与第 7 批的预告**：本批只讲“请求怎么流动”，没讲“请求怎么排队、怎么取消、IOCTL 的数据怎么进出”。DEVICE_CONTROL 在本批只是个名字，它的 IOCTL 定义、缓冲 I/O / 直接 I/O / Neither 三种数据进出方式、以及“驱动怎么从用户态缓冲区安全取数据”，全在第 7 批——那是 BYOVD 审计的主战场（IOCTL 分级、缓冲区校验）。

---

## 五、易混点辨析

1. **PENDING 不是完成，是“已收”**。返回 STATUS_PENDING 的分发例程已经把 IRP 的所有权交出去了，再读结果、改参数、二次完成全是错。结果只从状态块 + 事件/APC 来。
2. **Skip 与 Copy 不是一回事**。Skip：下层共用本层位置（参数相同、完成例程相同，纯过路）；Copy：把参数复制到下一联（下层要相同参数），但完成例程不复制——完成例程是按层现挂的，复制会串层。
3. **CLEANUP 与 CLOSE 是两次不同的“关”**。CLEANUP：最后一个句柄关了（但可能还有未完成 I/O），在关闭进程上下文中，干“释放进程专属资源 + 取消该文件对象的排队 IRP”的活；CLOSE：最后一个句柄关了且释放了、全部 I/O 已了结，不一定在关闭进程上下文中，恒在 PASSIVE_LEVEL，干“undo CREATE”的活。进程专属资源（锁住/映射的用户内存）必须在 CLEANUP 里放，CLOSE 里放就晚了（上下文已不是关闭进程）。
4. **STATUS_MORE_PROCESSING_REQUIRED 不是错误，是“截停”**。完成例程返回它，走访当场停下、IRP 不释放、以后再调一次 IoCompleteRequest 就从截停点继续往上。同步等模式（事件 + 截停）与自建 IRP 释放后都靠它。
5. **最底层驱动没有完成例程**。它下面没人可监听；它的“完成”是 ISR → DPC → 直接 IoCompleteRequest。反过来，自建 IRP 的驱动必须有完成例程——没人替它释放。
6. **同步请求不挂完成例程，异步请求挂**。同步用事件等（分发例程等事件、完成例程 signal 事件 + 截停）；异步靠完成例程收尾。混用（同步请求挂了完成例程又等事件）是死锁与重复完成的温床。
7. **“I/O 管理器释放它分配的，驱动释放它分配的”**。原始 IRP（I/O 管理器分配）由 I/O 管理器释放；自建 IRP 由自建者的完成例程调 IoFreeIrp 释放。反过来（驱动释放原始 IRP、或不释放自建 IRP）分别是 use-after-free 与泄漏。
8. **完成例程里能看到的只有状态块**。下层联已被清零，别想从联上读参数；想在完成时用的上下文，注册时用 Context 指针传（且不能分页）。
9. **POWER 下传必须用 PoCallDriver**。IoCallDriver 送 POWER 是错——电源 IRP 有自己的完成与同步规则（第 11 批），PoCallDriver 是唯一入口。
10. **“断链”与“无效请求”是两回事**。断链：中间层漏了下层处理的主功能码入口点，请求到中间层就没人接；无效请求：驱动收到了主功能码但参数不对，当场完成、返回错误。断链是设计缺失，无效请求是运行期裁决。

---

## 六、自查问题（只给问题，自行回原文核对）

1. 为什么高层驱动只能碰自己和下一层的位置、最底层只能碰自己的位置？这条规则防的是什么未来变化？
2. I/O 管理器在完成走访中把每层位置清零，这条规则对驱动之间的通信方式施加了什么限制？“走后门”为什么不可移植？
3. 一个读请求：FSD 拆成两个子 IRP 下传，两个子 IRP 各自的完成例程、原始 IRP 的完成、以及子 IRP 的释放，顺序是怎样的？谁释放谁？
4. 中间层驱动漏掉了 IRP_MJ_DEVICE_CONTROL 的入口点，下层磁盘驱动处理它，应用发 DeviceIoControl 会发生什么？I/O 管理器返回什么？
5. 驱动在分发例程里拿着自旋锁调 IoCompleteRequest，为什么可能死锁？正确的顺序是什么？
6. CLEANUP 与 CLOSE：哪一个在关闭进程的上下文中送达？进程专属资源（锁住的用户内存）为什么必须在 CLEANUP 里释放？
7. 自建 IRP 的完成例程返回 STATUS_MORE_PROCESSING_REQUIRED 后，IRP 处于什么状态？谁、什么时候让走访继续？
8. 为什么 IoMakeAssociatedIrp 只有最高层驱动能用，中间层不能用？从属 IRP 不能用于哪种 I/O？

---

## 附：追问补记——自建 IRP 给自己留一联（2026-10-28）

- **结论**：自建者想在新 IRP 里给自己留一联存完成例程用的上下文，分配时必须多分一层——IoAllocateIrp 的层数 = 下层链需要的 StackSize + 1。指南中间层驱动那节原话就是：调用 IoAllocateIrp 时可以把层数加一，“to get a stack location of its own”。
- **顺序的道理**：IRP 里有两个字段管位置——CurrentLocation（当前层号）与 CurrentStackLocation（当前联指针）。IoAllocateIrp 出来的初始状态是指针停在栈顶之外（CurrentLocation = StackSize + 1）；IoSetNextIrpStackLocation 把指针往下挪一格（CurrentLocation 减一），最顶上一联变成“当前联”；IoGetCurrentIrpStackLocation 拿到它，往里写上下文；IoGetNextIrpStackLocation 再往下一格，给下层填。顺序反了，IoGetCurrentIrpStackLocation 拿到的是栈顶之外的越界位置。
- **完成时怎么拿回来**：完成走访自底向上逐层加一，走到这一层时当前指针正好指着自己这一联，所以完成例程里调 IoGetCurrentIrpStackLocation 就能读出当初存的上下文。精度：完成时被清零的是下层刚办完的那一联，不是自己这一联——自己这一联在例程跑的时候是完整的，不然它就没法当仓库用；例程不能依赖的是下层联的参数，只能看状态块。
- **对比**：IoBuildAsynchronousFsdRequest 建的 IRP 没有这一联，所以指南说它的完成例程“在 IRP 里没有本地上下文可存”，上下文须驱动自己在常驻内存另行维护。Context 参数（注册时直接传指针）是另一条路；留联的办法是让上下文跟着 IRP 本身走。
- **原始 IRP 不需要这招**：I/O 管理器建原始 IRP 时已经给了链上每一层一联，驱动自己的那一联现成；只有自建 IRP（自建者本来不在链里）才需要多分一层。

---

## 附：追问补记——发完自建 IRP 就返回 PENDING，结果从哪来（2026-10-28）

- **是**：分发例程把自建的 IRP 全送出去（IoCallDriver），立刻返回 STATUS_PENDING，不等下层或别的设备办完。
- **为什么不能等**：① 阻塞请求线程，异步 I/O 的设计目的就是不阻塞；② 死锁：分发例程等事件，下层完成可能要进同一把锁、或要在请求线程上下文跑；③ IRQL：分发例程可能在 DISPATCH_LEVEL 被调，根本不能等分发器对象；④ 吞吐：一个请求等住，整条链被拖住。
- **结果从完成例程来**：送出每个自建 IRP 之前必须挂 IoSetCompletionRoutine（三开关全开）。下层办结子 IRP 时，完成走访调这个例程；例程看状态块得知子结果、汇总（计数减一、失败值拷回原始 IRP、IoFreeIrp 释子 IRP）；子 IRP 全回齐（计数到零）→ 填原始 IRP 状态块 → IoCompleteRequest 完成原始 IRP。分发管发、完成管收，是两个例程、两个时刻、甚至两个线程上下文。
- **原始 IRP 的状态**：分发例程返回 PENDING 的同时，原始 IRP 已打 IoMarkIrpPending，躺着等；最后一个子 IRP 的完成例程触发它完成。同步请求者等事件、异步请求者先走，结果经原始 IRP 的完成走访送达。
- **例外（同步等模式）**：驱动内建 IOCTL 可建事件、挂完成例程（signal 事件 + 返回 MORE_PROCESSING_REQUIRED 截停），分发例程 KeWaitForSingleObject 等事件，醒来后再处理。但条件苛刻（≤ APC_LEVEL、非任意线程上下文；栈上事件须 KernelMode 等），且即使这里，结果递送者仍是完成例程——分发只是换个地方读结果，不是向 IoCallDriver 的返回值要结果。IoBuildSynchronousFsdRequest 同理，只允许驱动自建线程等。
- **IoCallDriver 的返回值不能当结果**：它只是下层分发例程的返回值，异步时就是 STATUS_PENDING（“接了”），不是办结结果。同步请求（CREATE 之类，下层当场办结）才能直接返回它。

---

## 附：追问补记——自建 IRP 不必须关联原始 IRP（2026-10-28）

- **结论**：关联（IoMakeAssociatedIrp）只是自建 IRP 的一种形态（扇出专用），不是必选项；完全新建、无关联的 IRP 是常规操作。
- **三种关系**：
  1. 关联子 IRP（IoMakeAssociatedIrp）：一份原始请求扇出成多份、且要等全部回齐才算完成。主 IRP 先定 AssociatedIrp.IrpCount；子 IRP 办完计数减一，到零 I/O 管理器自动完成原始 IRP；驱动不必也不应自己再完成原始 IRP。
  2. 无关联、但结果要汇总回原始 IRP（上一条讲的形态）：驱动用 IoAllocateIrp 自裁，挂完成例程，自己计数、自己在完成例程里调 IoCompleteRequest 完成原始 IRP。这是“手动关联”，比关联更灵活（能合并状态、处理部分失败、改写结果）。
  3. 无关联、结果也不回原始 IRP（自发活）：驱动自己发、自己收、自己处理，原始 IRP 不知道也不关心。
- **自发活的例子**：读分区表、挂载时读卷元数据、刷脏页、关机 SHUTDOWN（驱动登记后收到的是现造的 IRP，不是用户请求的）；活变了（上面要 A，驱动先问下层 B，如 IOCTL 查能力）；路变了（原 IRP 按本栈层数裁死，换链得新裁）；缓冲区/线程变了；分发里同步问话（等原 IRP = 等自己，死锁，故新裁同步 IRP 自己等）；重试/换路（MPIO）。
- **上一条规矩的适用范围**：“发完返回 PENDING”针对的是第 1、2 种——自建 IRP 是为了完成原始请求的情况。分发例程的返回值回答的是“原始 IRP 办完了没有”，不是“自建 IRP 办完了没有”。第 3 类自发活通常根本不在分发例程里发起（在驱动自己的线程、工作项、DPC 里），和分发例程的返回值无关。
- **判据**：关联与否只决定“子 IRP 回齐后谁来完成原始 IRP”（I/O 管理器自动 vs 驱动手动），不决定“能不能自建”；转发成立的前提是“下面要干的活和上面要的活是同一件活”，任一不同就新造。

---

## 附：追问补记——无关联自建 IRP 的完成例程如何拿到原始 IRP（2026-10-28）

- **结论**：是的，原始 IRP 的指针必须以某种形式跟着走；I/O 管理器不会自动告诉完成例程原始 IRP 是谁。无关联 = I/O 管理器层面两者独立，关系只存在于驱动自己的数据里。关联（IoMakeAssociatedIrp）则是 I/O 管理器也知道关系（MasterIrp 指针 + IrpCount 在 IRP 结构里），所以它能自动完成原始 IRP。
- **载体一：Context 参数（最常用）**：IoSetCompletionRoutine 的第三个参数 Context 就是干这个的——注册时传原始 IRP 指针，完成例程签名 CompletionRoutine(DeviceObject, Irp, Context) 的第三个参数原样还你。注意第二个参数 Irp 是刚办完的自建 IRP，不是原始 IRP；原始 IRP 只能从 Context 来。
- **载体二：上下文结构（最工程化）**：一个指针不够用时（还要计数、汇总状态、锁），驱动分配一个非分页的上下文结构，里面放原始 IRP 指针 + 待回计数 + 汇总状态，把结构指针当 Context 传。生产驱动最常见的做法。
- **载体三：自留联（上一条的多分一层）**：多分的那一层里存原始 IRP 指针（联里的通用参数区，如 Parameters.Others.Argument1，自留联不下发，字段随便用）；完成例程里调 IoGetCurrentIrpStackLocation 读出来。
- **载体四：设备扩展表（较重，少用）**：驱动在设备扩展里维护未完成 IRP 的配对表，自建 IRP 发出时登记（子 IRP 指针 ↔ 原始 IRP 指针），完成例程查表找到原始 IRP。需要取消、超时清理时本来就要一张表，才顺带用它。
- **配套要点**：① 复制的是指针不是本体——原始 IRP 只有一份，躺着 pending；完成例程拿到指针后先把子结果汇总进原始 IRP 的状态块，再调 IoCompleteRequest；完成之后不许再碰原始 IRP。② 计数必须原子（InterlockedDecrement）：多核下两个完成例程可能同时回，普通减一会丢完成（原始 IRP 永远没人完成）或重复完成（蓝屏）。
