---
layout: post
title: "Win 内核驱动 第 12 批：编程技术工具箱 + WMI 收尾"
categories: windows-kernel
series_order: 12
---

- 本批主题：编程技术工具箱 + WMI 收尾（12 批路线第 12 批，末批）
- 对应原文档：驱动程序编程技术余部 + WMI（概念）
- 研读底本：《Kernel-Mode Driver Architecture Design Guide》2019 版离线 PDF
- 完成日期：2026-10-29

## 取材说明

- 底本是微软官方文档的 2019 版离线打包（《Kernel-Mode Driver Architecture Design Guide》，1279 页）；该 PDF 的来源仓库 tpn/pdfs 是第三方文档收集仓库、非微软官方，取材性质以此为准。
- 本指南不含环境搭建、Hello World 与 KMDF 的内容，本批不承诺它没有的东西；KMDF 驱动的 WMI 支持另有框架文档（指南在 WMI 章首即指路到 "Supporting WMI in Framework-Based Drivers"），不在本批范围。
- 本批是工具箱与收尾性质：一头是"驱动程序编程技术"章的余部（Nt/Zw 例程、NTSTATUS、安全字符串、安全整数、错误日志、注册表例程与筛选、可靠驱动常见错误、64 位移植），另一头是 WMI 章的概念与注册/处理流程。前 11 批已讲过的机制（IRP 分发与完成、队列与取消、IOCTL、中断/DPC/同步、DMA、PnP、电源）只在工具函数与之交接的点提及，不重讲机制本身。
- 同属"编程技术"大章但未列入路线第 12 批定稿范围的内容（CLFS 通用日志、KTM 内核事务、动态硬件分区、驱动/应用通知等独立子系统，以及过时/保留例程与全局变量的清单词表），本批只点名不展开，需要时按 DDI（驱动开发接口）参考集随查。
- 指南中被版面打碎的函数汇总表（安全字符串、安全整数的 Summary 表）不可转录，本批只讲规则与代表函数，函数全表以 WDK（Windows 驱动开发包）头文件（ntstrsafe.h、ntintsafe.h）为准，列入未钉死清单。

## 学习目标

学完本批应能说清：

1. Nt 与 Zw 例程之别：PreviousMode 是什么、验证在哪里发生、为什么在任意线程上下文里必须用 Zw、用 Nt 会怎样丢句柄与崩溃；
2. NTSTATUS 的四类值域与严重性位域，为什么判成功必须用 NT_SUCCESS 等宏、直接比 STATUS_SUCCESS 错在哪，自定义状态码的边界（哪条路径不许用）；
3. 安全字符串与安全整数两套库各自防什么、接口约定是什么（恒 NULL 结尾、唯一成功码、Cb/Cch 计数；溢出返回 STATUS_INTEGER_OVERFLOW、无除法函数的原因）；
4. 错误日志的写入流程（分配条目、%1/%2 插入串、消息源注册）、.mc 自定义消息的编译管线、Bug Check Reason Callback 在 HIGH_LEVEL 的禁令清单；
5. 注册表三条路线（Zw 句柄路线、Rtl 按名路线、PnP 专用开口）各自的代价与禁区（四棵不许直访的 PnP 树）；
6. 注册表筛选的回调模型：altitude 分层、pre 阻断与接管（STATUS_CALLBACK_BYPASS）、post 改写、三层上下文、两类无效键对象指针；
7. 可靠驱动常见错误清单逐条内容，尤其是 Buffered I/O 三错、Direct I/O 的双重映射写保护假象、METHOD_NEITHER 的零验证、CLEANUP 与 CLOSE 的分工、公私 IOCTL 必须分开；
8. 64 位移植要点：LLP64 与三类数据类型、对齐故障为何 32 位能忍 64 位不能、-Wp64 警告背后的截断坑、32 位 I/O 的 thunking（转换）为何只能由驱动做、两种识别 32 位调用方的办法、64 位 DMA 的双重缓冲代价、x64 的禁改清单；
9. WMI 的概念与全流程：MOF 模式（数据块/事件块/GUID/实例名）、发布的编译与资源挂载、注册两阶段、DispatchSystemControl 处理 WMI IRP 的两条路线（WMI 库与自办）、次功能码全表、事件发送的两条路与 1KB 上限。

## 术语表

本表按引用级排序：基础概念在前、派生概念在后，读到某条时它引用的词都已在前面定义。互为前提的一对——GUID 与 WMI 数据块（块靠 GUID 标识、GUID 的用途之一是标识块）——GUID 条给行内解释并在块条目处回指，本表头在此披露该惯例。前批已定义的术语（IRP、分发例程、IOCTL 与四种缓冲方式、MDL、完成例程、取消安全队列 CSQ、自旋锁、设备对象、PDO/FDO）在本批沿用，只在正文首次出现处一行定位，不在本表重写。

- **PreviousMode（前一模式）**：线程的一个属性，记录该线程"本次进入内核之前身在何处"：用户态线程经系统调用进入内核时被置为 UserMode，纯内核线程为 KernelMode。它是系统服务参数验证的总开关：验证逻辑靠它判断"这批参数与句柄要按不可信来源查、还是按可信来源放行"。为什么要有：同一套内核例程既被用户态请求借道调用、也被驱动直接调用，内核必须有个地方记住来源可信度，否则要么对驱动的调用也全量验证（慢且会误伤内核指针），要么对用户参数也放行（安全崩盘）。驱动借用的线程 PreviousMode 可能是 UserMode，这是本批 Nt/Zw 一节全部麻烦的根源。
- **内核句柄（kernel handle）**：以 OBJ_KERNEL_HANDLE 属性打开的句柄，只能在内核态使用、在任何线程上下文里都可查（不受借用线程的句柄表影响）。为什么要有：普通句柄挂在进程句柄表上，驱动在任意线程上下文（借用的是别的进程的线程）里拿普通句柄去用，查的是当前进程的表，查不到或查错；内核句柄绕开进程句柄表，才能在任意上下文里稳定引用对象。
- **NTSTATUS**：内核统一的状态码类型，32 位，最高两位是严重性（见下条）。几乎所有内核例程都返回它，用户态的 Win32 错误码是它在边界上被翻译后的产物。为什么用一套码贯穿全系统：驱动栈是多层协作的，状态要能逐层上传、被 I/O 管理器与上层驱动统一解读，分家就会在层界上丢信息。
- **严重性位域**：NTSTATUS 的高位布局：最高 2 位 Sev 表严重性（0 成功、1 信息、2 警告、3 错误），其下有 C 位（客户定义标记）和 Facility（设施码，标明码出自哪个子系统）。值域按严重性四分：0–0x3FFFFFFF 成功、0x40000000–0x7FFFFFFF 信息、0x80000000–0xBFFFFFFF 警告、0xC0000000–0xFFFFFFFF 错误。"成功"是一个区间而不是一个值，是 NT_SUCCESS 宏必须存在的根源。
- **SEH（结构化异常处理）**：Windows 的 try/except（与 try/finally）异常机制。内核里异常有两类来源：硬件异常（访问违例、对齐故障等）与系统软件异常（例程主动抛出的状态）。为什么驱动必须会用：驱动要碰用户缓冲这类"随时可能翻脸"的地址，探测（Probe）通过不等于访问时页还在，访问本身必须罩在 try/except 里，漏罩一次未处理异常就是蓝屏，且责任记在引发异常的驱动头上。内核不支持 C++ 异常，SEH 是唯一的异常机制。
- **插入串（insertion string）**：错误日志与消息文本里 %1、%2 这类占位符在显示时被替换进去的字符串。%1 由系统自动替换为记录条目的设备名，%2 起是驱动随日志条目附带的字符串。为什么要有：消息文本要本地化、要编译进资源，而设备名、文件名这类运行时值只能在写日志时插入，两者分工即源于此；也因此插入串本身不可本地化。
- **hive（配置单元）**：注册表的一棵子树连同其 backing 文件的整体，如 SYSTEM hive。内核里注册表根是 \Registry，用户态的 HKEY_LOCAL_MACHINE、HKEY_USERS 在内核对应 \Registry\Machine、\Registry\User；HKEY_CLASSES_ROOT 与 HKEY_CURRENT_USER 在内核没有对应物（它们是用户态拼出来的视图）。
- **altitude（高度值）**：筛选类回调注册时领的层号，决定同类回调被调用的顺序：注册表筛选按 altitude 从高到低依次调。为什么要有：多个筛选驱动（杀毒、审计、合规）叠在同一操作上时，必须有全局统一的次序规则，否则谁先谁后靠加载顺序碰运气，阻断与改写的语义会互相打架。它和文件系统 minifilter 的 altitude 是同一套思想。
- **LLP64**：64 位 Windows 的数据模型：指针 64 位，int 与 long 保持 32 位（Long Long Pointer 64）。为什么是它而不是 Unix 的 LP64（long 也 64 位）：保住海量既有代码里"long 是 32 位"的假设不炸，代价是"指针、int、long 三者同宽"的 32 位直觉全部作废——64 位移植一节的大半坑由此而来。
- **固定精度类型 / 指针精度类型**：Basetsd.h 的两类整数类型。固定精度（INT32/INT64/UINT32/UINT64/LONG32/LONG64/DWORD32/DWORD64）宽度写在名字里、两平台同宽；指针精度（ULONG_PTR/DWORD_PTR/INT_PTR/LONG_PTR/UINT_PTR/SIZE_T/SSIZE_T/HALF_PTR 等）随平台变宽（32 位平台 4 字节、64 位平台 8 字节），指针算术与"能装下指针的整数"用它。另有固定精度指针 POINTER_32/POINTER_64：在对侧平台上是截断/符号扩展的指针表示，专为跨位宽结构准备（thunking 一节用）。
- **WOW64**：64 位 Windows 上跑 32 位用户程序的转换层：它拦截 Win32 调用、把指针精度参数转成 64 位形式再送进内核，这个转换动作叫 **thunking（转换）**。关键边界：WOW64 只转它认识的系统 API 参数，DeviceIoControl 的数据缓冲内容是驱动私定义的，它不转、不敢转——这正是 64 位驱动要自己做 32 位 I/O thunking 的原因。内核态没有 WOW64：32 位驱动在 64 位 Windows 上不受支持，驱动必须原生 64 位编译（与第 1 批结论一致）。
- **GUID（全局唯一标识符）**：128 位标识符，用来在不设中央登记处的前提下给事物发永不重复的名字（行内解释：生成算法保证碰撞概率可忽略）。本批在三处用它：WMI 数据块/事件块的标识、注册表筛选二级转储数据的标记、对象与接口标识。它与 WMI 数据块互为前提：块靠 GUID 被引用（见下条），GUID 条目本身不依赖块而成立，故排在块之前。
- **MOF（Managed Object Format）**：DMTF 定的、基于 IDL 的模式定义语言。驱动用它写 WMI 模式：每个数据块/事件块一条类定义，带限定符（qualifier）标注 GUID、是否动态、读写性等。MOF 编译后的二进制挂到驱动镜像上作资源，是 WMI 认识驱动块定义的唯一来源。
- **WMI 数据块 / 事件块**：驱动向 WMI 暴露的信息单元。数据块是可被查询（也可被设置、可执行方法）的仪表数据；事件块是驱动主动上报的通知（必须派生自抽象基类 WmiEvent，注册时带 EVENT_ONLY 标志，只能被启用/禁用、不能被查询）。块靠 GUID 标识（回指 GUID 条），靠类名向用户展示。块内实例靠实例名区分（下条）。
- **实例名（instance name）**：同一数据块有多个实例（如多块同型号网卡）时区分实例的名字。分静态与动态：静态实例名在注册时定死（三种来源：驱动给的名字列表、基名加序号、PDO 的设备实例路径），动态实例名由驱动在每次响应里现给。选哪种决定注册路线（WMI 库只支持静态名的一部分情形），是 WMI 注册一节的主线变量。
- **WNODE_XXX**：WMI 在消费者与驱动之间传数据的一族标准结构（WNODE_SINGLE_INSTANCE、WNODE_ALL_DATA、WNODE_SINGLE_ITEM、WNODE_METHOD_ITEM、WNODE_EVENT_ITEM、WNODE_TOO_SMALL 等），共同点是头部都嵌一个 WNODE_HEADER（缓冲大小、块 GUID、标志位）。自办 WMI IRP 的驱动要亲手读写的就是这一族结构；用 WMI 库的驱动则被库挡在它们之外。
- **DispatchSystemControl**：驱动的分发例程之一，处理主功能码 IRP_MJ_SYSTEM_CONTROL 的 IRP，WMI 请求全走这里（次功能码区分具体请求）。所有驱动都必须给它设分发入口：注册成提供者的自己办，没注册的往下层转（第 6 批分发模型在 WMI 场景的落点）。

派生与专用术语（在上面基础上）：

- **安全字符串函数（RtlStringCb*/RtlStringCch*）**：Ntstrsafe.h 的内核字符串库，Rtl 前缀，Cb 版按字节计数、Cch 版按字符计数，后缀 W/A 分 Unicode/ANSI，另有一组吃 UNICODE_STRING 的。它们的四条硬约定（调用方传目标缓冲大小、输出恒以 NULL 结尾、只返回 NTSTATUS 且唯一成功码 STATUS_SUCCESS、多数有带额外参数的 Ex 版）就是与标准 C 串函数的分界。
- **安全整数函数（ntintsafe 库）**：防整数溢出/下溢/截断的两类函数：转换函数（带范围检查的跨类型转换）与算术函数（加/减/乘）。溢出时返回 STATUS_INTEGER_OVERFLOW 而不是回绕。没有除法函数——除法不存在溢出条件（唯一例外是最小负数除以 −1，指南未展开，本批不引申）。
- **IO_ERROR_LOG_PACKET**：驱动写系统错误日志时填的条目结构：ErrorCode（一个 NTSTATUS，预定义值形如 IO_ERR_XXX）、DumpData（二进制转储数据，附带现场）、StringOffset/NumberOfStrings（管插入串）。IoAllocateErrorLogEntry 分配它、IoWriteErrorLogEntry 写入并自动释放。
- **消息源（message source）**：事件查看器显示日志靠的登记：驱动在注册表 HKLM\System\CurrentControlSet\Services\EventLog\System\<DriverName> 下登记 EventMessageFile（消息文件路径，REG_EXPAND_SZ，分号分隔多个）与 TypesSupported（支持的日志类型位，一般填 7）。用系统预定义错误码的驱动要把 iologmsg.dll 登记为消息文件；用自定义码的要把自家编译出的资源附在驱动镜像上并登记它。
- **.mc 消息文件与 mc.exe**：自定义错误码的源头管线：驱动作者写 .mc 文本（MessageId/Severity/Facility/SymbolicName 与 %n 模板），用消息编译器 mc.exe 编出 .h（符号定义）、.rc 与 .bin，再把资源附加进驱动镜像。运行期事件查看器按 ErrorCode 里的 Facility/Id 去消息文件里查模板、代入插入串显示。
- **Bug Check Reason Callback（蓝屏原因回调）**：驱动用 KeRegisterBugCheckReasonCallback 注册的回调，在系统蓝屏时被以 HIGH_LEVEL 调用，往故障转储里补驱动自己的数据（四种理由：加页 KbCallbackAddPages、转储 I/O、二级转储数据 KbCallbackSecondaryDumpData、三级分诊数据 KbCallbackTriageDumpData，Win10 1809+）。HIGH_LEVEL 意味着它能干的事极少（下条禁令即源于此）。
- **STATUS_CALLBACK_BYPASS**：注册表筛选 pre 通知的特殊返回值，意思是"这个操作我接管了，配置管理器不要再处理"。接管方必须把输出填妥，CM 直接以 STATUS_SUCCESS 回调用线程。它也是 post 通知改写原操作返回值的约定返回值（改走 ReturnStatus 字段）。
- **双重缓冲（double-buffering，DMA 语境）**：64 位系统上，不支持 64 位寻址的 DMA 设备要访问 4GB 以上物理内存时，系统把数据先在低地址缓冲里中转一份再搬的兜底法（第 9 批弹跳缓冲的同一机制）。它保住正确性、吃掉性能，且内存越大吃得越多，所以是"能避则避"的移植问题而非功能问题。
- **WMILIB_CONTEXT 与 DpWmiXxx 回调族**：走 WMI 库路线时驱动要填的上下文结构（块数、WMIGUIDREGINFO 列表、回调入口）与一组回调：DpWmiQueryReginfo（报注册信息，必需）、DpWmiQueryDataBlock（答查询，必需）、DpWmiSetDataBlock/DpWmiSetDataItem/DpWmiExecuteMethod/DpWmiFunctionControl（可选）。库 routine WmiSystemControl 负责拆 IRP、验参数、调对应回调、打包 WNODE，WmiCompleteRequest 负责收尾完成。
- **IRP_MN_REGINFO / IRP_MN_REGINFO_EX**：驱动调 IoWMIRegistrationControl 登记后，WMI 反过来向驱动索取注册详情的两个次功能码（REGINFO 是 98/2000 时代版，XP 起改用 REGINFO_EX）。它们是"先登记意向、再回送索取"两阶段注册的第二段（为什么分两段，正文 WMI 一节解开）。

---

## 一、本批地图

本批是 12 批路线的收官批，没有新的大机制，内容是两大包：

**第一包：编程技术工具箱**（对应"驱动程序编程技术"章的余部）

1. Nt/Zw 例程与 PreviousMode：同一例程的两个版本、验证开关、任意线程上下文的正确姿势；
2. NTSTATUS：值域、位域、判断宏、自定义状态码的可用边界；
3. 链表与 SEH：工具箱的日常件（简版，链表原子版与非原子版不许混用、SEH 罩住一切碰用户缓冲的访问）；
4. 安全字符串与安全整数：两套"防溢出"库（一个防缓冲溢出、一个防算术溢出）；
5. 错误日志：写日志、登记消息源、自定义消息管线、蓝屏原因回调；
6. 注册表例程：Zw 句柄路线、Rtl 按名路线、PnP 专用开口与四棵禁访树；
7. 注册表筛选：CM 回调的过滤模型；
8. 可靠驱动常见错误清单：微软官方点名的九组高频错误，是全批审计价值最高的一节；
9. 64 位移植：数据类型、对齐、警告、thunking、DMA 双重缓冲、x64 禁改清单；
10. 工具箱小件：安全模式检测、驱动内用文件、浮点使用规则、对象引用跟踪标签。

**第二包：WMI（概念与流程）**

11. WMI 概念：提供者/消费者/IRP 通道、MOF 模式、数据块与事件块、实例名；
12. 发布与注册：MOF 编译挂资源、两阶段注册、WMI 库路线与手工路线、注册标志；
13. 处理请求：DispatchSystemControl、次功能码全表、自办时的校验清单、WNODE 结构族；
14. 发送事件：启用/禁用模型、WmiFireEvent 与 IoWMIWriteEvent 两条路、1KB 上限与超限改道。

学完本批，12 批路线全部走完：从 WDM 骨架、设备对象、内存与对象、安全性，到 IRP 上下、同步、DMA、PnP、电源，最后以工具箱与 WMI 收尾。

---

## 二、核心要点

- **Nt/Zw 是同一例程的两个版本，分界是"要不要验证"**：内核态调 Zw，等于包装层把当前线程 PreviousMode 临时改成 KernelMode 再调 Nt 本体、返回后恢复，即声明"参数来自可信内核源"；调 Nt 则按线程当下的 PreviousMode 决定验不验。驱动常在任意线程上下文里运行，借来线程的 PreviousMode 可能是 UserMode——此时用 Nt 传内核句柄会 STATUS_INVALID_HANDLE 且句柄泄漏，传内核缓冲会被 Probe 拒为 STATUS_ACCESS_VIOLATION。Zw 前缀本身无含义，只是为避开命名冲突保留的两字母前缀。用户态调两者无区别。
- **判 NTSTATUS 成败必须用宏**：成功是 0–0x3FFFFFFF 一整个区间（含 STATUS_OBJECT_NAME_EXISTS 这类信息性成功），警告（如 STATUS_NO_MORE_ENTRIES、带部分数据的 STATUS_BUFFER_OVERFLOW）也不是失败。用 == STATUS_SUCCESS 直比会把成功误判成失败，是官方点名的常见错误。判断用 NT_SUCCESS/NT_INFORMATION/NT_WARNING/NT_ERROR。
- **自定义状态码有禁区**：成对驱动之间走 IRP_MJ_INTERNAL_DEVICE_CONTROL 可以用私有 STATUS_XXX；但类驱动在结果可能被上层完成例程看到时必须映射回系统定义值；凡用户态可达的 IRP 一律不许用自定义值——它翻不成 Win32 错误码，用户侧只会看到莫名其妙的失败。自定义错误日志码走 FACILITY_IO_ERROR_CODE 的 IO_ERR_XXX 体系。
- **安全字符串的约定就是它的全部价值**：调用方必须传目标缓冲大小、输出恒 NULL 结尾（截断也结尾）、只返回 NTSTATUS 且唯一成功码。标准 C 的 strcpy/strcat/sprintf 不知道目标有多大，是内核里大量安全问题的根源，内核代码禁用裸用。
- **安全整数防的是"算出来的大小本身就是错的"**：加/减/乘与跨类型转换都查溢出，溢出返回 STATUS_INTEGER_OVERFLOW；算术函数只吃无符号数，有符号先用转换函数安全转过去。它的主场是算下标、算缓冲大小、做边界检查——与常见错误清单里"乘法溢出改除法比较"是同一件事的两面。
- **错误日志是给管理员看的正式通道，与 DbgPrint 分工不同**：IoAllocateErrorLogEntry 拿条目、填 ErrorCode/DumpData/插入串、IoWriteErrorLogEntry 写完自动释放；%1 自动是设备名。用预定义 IO_ERR_XXX 必须把 iologmsg.dll 注册为消息源；自定义码要走 .mc → mc.exe → 资源进镜像的管线。蓝屏原因回调在 HIGH_LEVEL 跑：禁分配内存、禁碰分页内存、禁同步、禁调任何要求 IRQL ≤ DISPATCH_LEVEL 的例程，只能读写端口/寄存器一类。
- **注册表三条路线按代价选**：Zw 路线（开句柄→操作→ZwClose）同键多操作效率最高；Rtl 路线按名操作、免管句柄，但每次操作内部开关一次句柄、且没有枚举子键的对应物；PnP 的四棵树（Control\Class、DeviceClasses、Enum、Hardware Profiles）严禁按路径直访——位置未来可变，只能 IoOpenDeviceRegistryKey（驱动软件键/设备硬件键）与 IoOpenDeviceInterfaceRegistryKey 拿句柄再走 Zw 例程。
- **注册表筛选是"任意线程的注册表调用都要过筛"**：Vista+ 用 CmRegisterCallbackEx 带 altitude 分层（高到低依次调，高层阻断或改写后低层不再被调）；pre 通知返回非成功状态即阻断操作（且被阻断的操作没有 post 通知）；返回 STATUS_CALLBACK_BYPASS 是完全接管；post 通知可改输出参数、可改返回值。两类键对象指针无效的坑（见详解）是筛选驱动蓝屏的高发点。
- **常见错误清单的总纲**：设备对象与句柄都要验对象本身（引用成功不等于对象对）；驱动天生多线程多处理器；状态码在 free 版也要查（只写在 ASSERT 里等于没查）；下传 IRP 必须 Skip/Copy 建下一层位置；不支持的 IRP 不许假成功；公私 IOCTL 路径必须分开。Driver Verifier 是最重要的可靠性工具，但不替代设计。
- **64 位移植的头号事实**：用户程序有 WOW64 兜底，驱动没有——32 位驱动在 64 位 Windows 上不受支持，必须原生编译。LLP64 下指针 8 字节、long 仍 4 字节；32 位 Windows 会自动修正内核对齐故障（代价是性能），64 位不修，未对齐的老 bug 移植时必须自己改掉。
- **IOCTL 缓冲的 thunking 是驱动的活**：WOW64 转系统 API 的参数，但不转 DeviceIoControl 缓冲内容（格式是驱动私定的）。缓冲里有指针精度成员时，64 位驱动预期 24 字节、32 位应用只给 12 字节，尺寸校验直接失败——驱动要么改协议消灭指针精度成员，要么自己识别 32 位调用方（控制码 64Bit 位，或 IoIs32bitProcess）并按 32 位版结构（POINTER_32 等固定精度类型）解析。
- **WMI 注册分两段是有意的**：先 IoWMIRegistrationControl 登记意向，WMI 再发 IRP_MN_REGINFO(_EX) 索取详情——因为 WMI 需要的信息（块清单、MOF 资源名、实例名来源）装不进一个登记调用，且索取走标准 IRP 通道能复用既有的分发与完成模型。处理路线二选一：WMI 库（WmiSystemControl + DpWmiXxx 回调，省事但只支持 PDO/基名静态实例名）或自办 DispatchSystemControl（支持动态实例名与静态名列表，代价是亲手读写 WNODE）。
- **WMI 事件是"先启用、后上报"**：没收到 IRP_MN_ENABLE_EVENTS 不许发事件；发送走 WmiFireEvent（受限：静态名、单实例）或 IoWMIWriteEvent（任意，自备 WNODE 结构）；事件加数据以 1KB 为限，超限改发 WNODE_EVENT_REFERENCE 让 WMI 回查。事件不是错误日志的替代品，异常情况该写日志还写日志。

---

## 三、关键知识点详解

### 3.1 Nt/Zw 例程与 PreviousMode：验证的总开关

**是什么。** 指南里成对出现的系统服务例程（NtClose/ZwClose、NtCreateFile/ZwCreateFile 等）不是两个实现，而是同一例程的两个版本。用户态调用时两者无区别（都经 Ntdll.dll 的桩进入内核）。内核态调用时分工明确：

- 调 **Nt 版**：例程按当前线程的 PreviousMode 决定要不要验证参数。PreviousMode 是 UserMode 时全量验证（句柄按调用进程的句柄表解释、缓冲要 Probe、访问权要查），是 KernelMode 时跳过验证、参数按内核来源直接使用。
- 调 **Zw 版**：Zw 是一层薄包装——把当前线程的 PreviousMode 临时改成 KernelMode，调 Nt 本体，返回后恢复原值。语义就是一句声明："这批参数出自内核、可信，不必验证。"

**为什么非如此不可。** 驱动代码大量运行在任意线程上下文：分发例程借用的是发起 I/O 的那个用户线程，它的 PreviousMode 被系统调用入口置为 UserMode。如果驱动在这种上下文里用 Nt 版传一个内核句柄，Nt 版按 UserMode 规则去借用线程所属进程的句柄表里查这个值——查不到（或查到的根本是另一个对象），返回 STATUS_INVALID_HANDLE，而且句柄没有被关掉：本想关闭的内核句柄就此泄漏。反过来，把一块系统内存缓冲传给 Nt 版例程，UserMode 规则要对它 ProbeForRead/ProbeForWrite，用户地址范围之外的系统地址直接判 STATUS_ACCESS_VIOLATION，操作根本做不成。Zw 版把这两类误伤一次性关掉，所以指南的规矩是：驱动打开/关闭自己的句柄、操作自己的内核缓冲，一律用 Zw 版；且句柄要以 OBJ_KERNEL_HANDLE 属性打开（配 InitializeObjectAttributes），这样句柄本身也不挑线程上下文。

**没有会怎样。** 上面两例就是答案：句柄泄漏是慢性病（长时间运行后资源耗尽），Probe 误杀是急性病（功能直接不可用）。还有第三种更隐蔽的：驱动若在 PreviousMode 为 UserMode 时用 Nt 版"成功"操作了一个句柄，那个句柄数值命中的是用户进程表里的对象——验证语义被调用方的身份左右，本身就是安全边界问题。

**附带事实。** ExGetPreviousMode 可以查当前线程的 PreviousMode；IRP 里的 RequestorMode 字段是发起线程 PreviousMode 在 IRP 创建时存下的副本，分发例程判断"这张 IRP 的参数要按谁的规矩验"看的是它（第 6/7 批已用）。Zw 的 "Zw" 前缀没有任何含义，纯粹是当年为避开与既有名字冲突、又保留系统服务两字母前缀惯例而选的。链接上：内核驱动链接 Ntoskrnl 导出的例程，用户态走 Ntdll.dll，两边的例程不能互相调用。访问权一节（ACCESS_MASK、DELETE/READ_CONTROL/SYNCHRONIZE/WRITE_DAC/WRITE_OWNER、GENERIC_* 与 STANDARD_RIGHTS_* 的构成）与第 5 批安全性同源，本批不展开，随查。

### 3.2 NTSTATUS：四类值域、判断宏与自定义码的边界

**值域与位域。** NTSTATUS 是 32 位值：按严重性四分——成功 0–0x3FFFFFFF、信息 0x40000000–0x7FFFFFFF、警告 0x80000000–0xBFFFFFFF、错误 0xC0000000–0xFFFFFFFF。位域拆开看：高 2 位 Sev 是严重性，其下是 C 位（客户定义标记，置位表示这是厂商/驱动自定义的码而非系统码）与 Facility 设施码。判断宏四个：NT_SUCCESS（成功与信息都算过，准确说是 Sev ≤ 1 的都为真——即"没失败"）、NT_INFORMATION、NT_WARNING、NT_ERROR。

**为什么必须用宏、直比錯在哪。** 因为"成功"是一个区间：STATUS_SUCCESS（0）只是最普通的一个，STATUS_OBJECT_NAME_EXISTS（打开已存在对象）是信息性成功，STATUS_NO_MORE_ENTRIES（枚举到头）是警告，STATUS_BUFFER_OVERFLOW 是"警告 + 已给你部分数据"。把返回值直接 == STATUS_SUCCESS 比较，这些全被误判成失败，驱动会把成功的操作当失败处理（丢弃已拿到的数据、向上传错）。反过来，只用 NT_SUCCESS 判断也不够：拿到警告时数据可能是部分的，枚举到头时"没更多了"是正常收尾信号——该细分时要用 NT_WARNING/NT_INFORMATION 区分。这与第 7 批"完成路径上的状态语义"是同一条纪律。

**自定义码的边界。** 驱动可以定义自己的状态码（错误日志码用 FACILITY_IO_ERROR_CODE 定义 IO_ERR_XXX，是另一套并行用途，见 3.6）。但用在 IRP 完成状态上，边界有三条：① 一对自家驱动之间走 IRP_MJ_INTERNAL_DEVICE_CONTROL（用户态够不着的内部通道），可以用私有 STATUS_XXX；② 类驱动的结果可能被上层的完成例程看到，完成例程不认识私有码，所以可能被它看到时必须映射回系统定义值；③ 凡用户态可达的 IRP 绝不许用自定义值——I/O 管理器要把它翻译成 Win32 错误码回给应用，翻不出来，用户侧只会收到无意义的失败。判据一句话：状态码能走多远，取决于最远端那个消费者认识不认识它。

### 3.3 链表与 SEH：工具箱的两个日常件

**链表。** 内核给两套现成侵入式链表：双向 LIST_ENTRY（InitializeListHead/InsertHeadList/InsertTailList/RemoveEntryList/IsListEmpty，靠 CONTAINING_RECORD 从链表节点反推宿主结构）与单向 SINGLE_LIST_ENTRY/SLIST。要注意的是原子版本：ExInterlockedInsertHeadList 等带一把调用方提供的自旋锁（第 8 批自旋锁机制），IRQL 提升由例程内部处理；这把锁只能用于这一组 ExInterlocked 例程，不能拿去保护别的东西；同一条链表，原子版与非原子版不许混用（混用等于一部分访问不拿锁）。SLIST（序列单链表）走 ExInterlockedPushEntrySList 一族，是无锁式的深层队列；64 位上 SLIST_ENTRY 必须 16 字节对齐——这是 LLP64 对齐要求收紧（见 3.10）在工具层的落点，不对齐直接故障。

**SEH。** 异常来源两类：硬件异常（访问了无效/未对齐地址等）与系统软件异常（ExRaiseStatus、ExRaiseAccessViolation、ExRaiseDatatypeMisalignment 等主动抛出）。规矩：凡碰用户缓冲这类不可信地址的操作——不光 ProbeForWrite 本身，它通过之后对那块内存的每一次实际访问——都要罩在 try/except 里。为什么 Probe 通过了还要罩：Probe 只证明"这一瞬间这些页在、权限对"，用户进程的另一条线程随时能把页换走/改权限（即检查与使用时差，TOCTOU——检查通过的那一刻与真正使用的那一刻之间状态已被改，与 3.9 双重映射一节是同一威胁模型），真正访问的那一刻仍可能抛访问违例。没有罩子的结果：未处理异常在内核里没有"崩进程"这个选项，直接蓝屏，且责任判定在引发异常的驱动——下层驱动不能指望上层会替它接。内核不支持 C++ 异常（throw/catch），SEH 是唯一的。

### 3.4 安全字符串：把"目标缓冲大小"变成接口契约

**问题。** 标准 C 串函数（strcpy/strcat/sprintf 一族）的接口里没有目标缓冲大小这一项，写溢出与否全凭调用方心算长度——指南直说这是大量安全问题的根源。内核里再叠一层：溢出的目标若是池内存，砸坏的是相邻的内核对象。

**解法。** 内核版字符串库在 Ntstrsafe.h 里，函数 Rtl 前缀、内联实现，分工：Cb 后缀按字节计数，Cch 后缀按字符计数；W/A 后缀分 Unicode/ANSI；再有一组专吃 UNICODE_STRING 的（RtlStringCbCopyUnicodeString、RtlUnicodeStringCopy 等）。四条硬约定构成它与 C 库的分界：① 调用方必须把目标缓冲大小传进来，库绝不写越界；② 输出恒以 NULL 结尾，哪怕发生截断也先保证结尾再谈内容；③ 只返回 NTSTATUS，且唯一成功码就是 STATUS_SUCCESS（截断返回 STATUS_BUFFER_OVERFLOW 之类的明确状态，不像 C 库那样截了也不吭声）；④ 多数函数有 Ex 扩展版（多收标志参数，如控制截断行为）。裁剪宏：NTSTRSAFE_NO_CCH_FUNCTIONS 与 NTSTRSAFE_NO_CB_FUNCTIONS 二选一（不能同时定义，那等于把整个库裁空）、NTSTRSAFE_NO_UNICODE_STRING_FUNCTIONS 关掉 UNICODE_STRING 组；NTSTRSAFE_MAX_CCH 一类上限宏只许调小不许调大。

**没有会怎样。** 不用它而用 C 库，等于每个拷贝点都重新发明一次长度判断，审漏一处就是内核池溢出——审计视角下，驱动源码里出现 strcpy/sprintf 裸调用本身就是高危信号（解读一节再提）。

### 3.5 安全整数：防"算出来的长度"本身溢出

**问题。** 缓冲区大小、元素个数×元素大小、下标——这些整数算式一旦溢出回绕，校验看到的是"小数字、合法"，随后的拷贝按真实的大数字写，校验形同虚设。第 7 批讲 IOCTL 校验时已点过"数目×大小乘法溢出要改除法比较"，这一节是它的库化版本。

**解法。** ntintsafe.h（内联或链接库形式）给两类函数：转换函数（跨整数类型转换时查范围，装不下就失败）与算术函数（加/减/乘，查溢出/下溢）。失败统一返回 STATUS_INTEGER_OVERFLOW，不给回绕值。算术函数只吃无符号数：有符号数先经转换函数安全地转成无符号再算——把"符号问题"与"溢出问题"拆开处理。没有除法函数，因为除法不产生溢出（商只会更小），这是库的完备性边界而不是遗漏。典型落点：算下标、算缓冲总大小、做边界检查前的长度合成。

**与 3.4 的分工。** 安全字符串管"往缓冲里写字时别越界"，安全整数管"算该写多长、该分多大时别回绕"，两者合起来才是一条完整的缓冲安全链：先用安全整数算大小，再用安全字符串做拷贝。

### 3.6 错误日志：写给管理员的正式通道

**写入流程。** IoAllocateErrorLogEntry（按设备对象）分配一个 IO_ERROR_LOG_PACKET，驱动填：ErrorCode（NTSTATUS；系统预定义集形如 IO_ERR_XXX，定义在 WDK 的 ntiologc.h）、DumpDataSize/DumpData（一段二进制原始数据，把出事现场的关键字节留给事后分析）、StringOffset/NumberOfStrings（插入串区）。填好调 IoWriteErrorLogEntry，写入后条目由系统自动释放；分配了却不用，要 IoFreeErrorLogEntry 归还。显示时事件查看器拿 ErrorCode 去消息文件查模板，%1 自动替换成记录条目的设备名，%2、%3……依次替换成驱动附带的插入串。

**消息源注册。** 光写条目不够，查看器得知道去哪查模板：在 HKLM\System\CurrentControlSet\Services\EventLog\System\<DriverName> 下登记 EventMessageFile（REG_EXPAND_SZ，分号分隔可多个）与 TypesSupported（REG_DWORD，一般填 7＝错误/警告/信息三类）。用预定义 IO_ERR_XXX 的驱动，EventMessageFile 必须包含系统的 iologmsg.dll（预定义码的模板住在那里）；一个驱动可以同时用预定义码与自定义码，两个文件都登记。

**自定义码管线。** 写 .mc 消息文本文件：文件头声明 MessageIdTypedef=NTSTATUS、SeverityNames/FacilityNames 的名字与取值，每条消息给 MessageId/Severity/Facility/SymbolicName 与带 %n 占位符的正文；mc.exe 编译出 .h（SymbolicName 成宏）、.rc 与 .bin，资源附加进驱动镜像，EventMessageFile 登记自家镜像即可。插入串不可本地化、模板才可本地化，所以非必要不用插入串（要进模板正文的值才用）。

**蓝屏原因回调。** KeRegisterBugCheckReasonCallback 注册、KeDeregisterBugCheckReasonCallback 注销。系统蓝屏时回调以 HIGH_LEVEL 被调用（转移方式：蓝屏路径上的直接函数调用；上下文是正在崩溃的系统、不可阻塞）——这个 IRQL 决定了禁令清单：禁分配内存、禁碰分页内存、禁任何同步（等锁等于死等）、禁调一切要求 IRQL ≤ DISPATCH_LEVEL 的例程；能做的只剩读写端口/寄存器与已备好的非分页数据。四种理由对应四种往转储里塞数据的姿势：KbCallbackAddPages（把自家非分页页加进主转储区）、KbCallbackDumpIo（转储 I/O 过程回调）、KbCallbackSecondaryDumpData（往二级区写带 GUID 标记的数据块；同一 GUID 写多块只有第一块可达，多写是坏习惯）、KbCallbackTriageDumpData（Win10 1809+ 的分诊转储，体积小、给初步分析用）。

**没有会怎样。** 不写错误日志的驱动，出故障时管理员在事件查看器里什么都看不到，只有蓝屏转储与调试输出——对要交付运维的驱动，错误日志是唯一不需要调试器在场的事后通道。

---

### 3.7 注册表例程：三条路线与四棵禁访树

**地基。** 键在内核是执行体对象，注册表根是 \Registry。用户态的 HKEY_LOCAL_MACHINE 对应 \Registry\Machine，HKEY_USERS 对应 \Registry\User；HKEY_CLASSES_ROOT 与 HKEY_CURRENT_USER 是用户态拼装的视图，内核里没有对应物——内核代码要找当前用户配置，得自己按 \Registry\User\<SID>（SID 为安全标识符，第 5 批已定义）拼路径，且"当前用户"在任意线程上下文里到底是谁需要想清楚再拼。

**Zw 句柄路线（三步）。** ① 开：InitializeObjectAttributes 填路径与属性（驱动自开的句柄带 OBJ_KERNEL_HANDLE），ZwCreateKey（不存在则建）或 ZwOpenKey（不存在返回 STATUS_OBJECT_NAME_NOT_FOUND）拿句柄，DesiredAccess 按要做的操作选（KEY_QUERY_VALUE/KEY_SET_VALUE/KEY_ENUMERATE_SUB_KEYS/KEY_CREATE_SUB_KEY/DELETE 等；GENERIC_READ/WRITE 与合成的 KEY_READ/KEY_WRITE 等价）。② 用：ZwQueryKey/ZwEnumerateKey（键信息与子键）、ZwQueryValueKey/ZwEnumerateValueKey（值）、ZwSetValueKey、ZwDeleteValueKey、ZwDeleteKey。读值的标准姿势是两步法：先以小缓冲探大小（得 STATUS_BUFFER_TOO_SMALL/STATUS_BUFFER_OVERFLOW 与所需长度），分配够大的缓冲再查一次——因为值可能在两查之间变化，一步到位假设固定大小是竞态。③ 关：ZwClose。注意已 ZwDeleteKey 的键，其句柄不许再 ZwClose，删除后指向该键的全部句柄一并失效。改动先被缓存、数秒后落盘，要立刻落盘用 ZwFlushKey（贵，只在真需要时用，如写完关键配置马上要断电重启的路径）。

**Rtl 按名路线。** RtlXxxRegistryXxx 一族按名字直接操作、免管句柄：RelativeTo 参数选根（RTL_REGISTRY_ABSOLUTE 全路径、RTL_REGISTRY_HANDLE 已打开的句柄、RTL_REGISTRY_USER 当前用户根等），RtlCreateRegistryKey/RtlCheckRegistryKey 建与查，RtlQueryRegistryValues（可批量查、带查询表与回调）、RtlWriteRegistryValue、RtlDeleteRegistryValue。代价写在明处：每次操作内部都开关一次句柄，同一个键要做多件事时效率低于 Zw 路线；且这一族没有 ZwEnumerateKey 的对应物，枚举子键只能回 Zw 路线。选型判据：一两次零星读写用 Rtl 图省事，反复操作同一键用 Zw。

**PnP 专用开口（禁区）。** HKLM\SYSTEM\CurrentControlSet\Control 下的 Class、DeviceClasses、Enum、Hardware Profiles 四棵树严禁按路径直访——它们是 PnP 管理器自家的账本，内部位置未来版本可以变，直访等于把驱动绑死在未承诺的布局上。正口是 IoOpenDeviceRegistryKey：PLUGPLAY_REGKEY_DRIVER 开驱动软件键、PLUGPLAY_REGKEY_DEVICE 开设备硬件键，可与 PLUGPLAY_REGKEY_CURRENT_HWPROFILE 组合定位到当前硬件配置文件；接口类配置走 IoOpenDeviceInterfaceRegistryKey。两者都只负责"拿到正确键的句柄"，拿到后读写仍走 Zw 例程。这是第 10 批"不许自找硬件资源"在注册表上的对应纪律：PnP 的数据只能经 PnP 的门进出。

### 3.8 注册表筛选：配置管理器回调

**模型。** 注册表筛选驱动（如杀毒的注册表防护组件）不拦 IRP——注册表操作根本不走设备栈。它向配置管理器（CM）注册回调，此后**任意线程**执行的每次注册表操作，CM 都会在该线程上下文里以函数调用方式先调 pre 通知、操作执行后再调 post 通知（转移方式：CM 在处理注册表调用的路径上同步直调回调，无 IRQL 升降的花活，回调就嵌在操作里）。XP 用 CmRegisterCallback（只有操作前通知与建/开键完成通知），Vista 起改用 CmRegisterCallbackEx，多领一个 altitude；注销用 CmUnRegisterCallback。

**版本演进三级。** ① XP：pre 通知返回非成功状态即可阻断操作——CM 立即以该状态返回调用线程，被阻断的操作不再有 post 通知（"阻断即终局"，筛选方不需要在 post 里再处理它）。② WS2003：补齐完成（post）通知。③ Vista+：分层与接管——多个回调按 altitude 从高到低依次调（此前按注册顺序、且新注册的插在栈顶附近，次序不可控是分层要解决的问题）；高层阻断或改写后，低层不再被调；某回调在通知里自己调了别的注册表函数，CM 不会从栈顶重新走一遍回调链（防递归爆栈）；pre 返回 STATUS_CALLBACK_BYPASS 表示完全接管：CM 直接回 STATUS_SUCCESS、不再执行原操作，接管方必须把全部输出填妥；post 里可以改输出结构后回 STATUS_SUCCESS，也可以填 REG_POST_OPERATION_INFORMATION.ReturnStatus 再回 STATUS_CALLBACK_BYPASS 改掉原操作的返回值——成功改失败时要代收 CM 已分配的对象（不收就泄漏），失败改成功时输出得自己备齐。

**三层上下文。** 注册时给回调的驱动上下文（一份，XP 起）；CallContext 操作上下文（Vista+，在 pre 通知结构里存下，post 通知里以同一 CallContext 配对捞回——一次操作的前后两半靠它缝合）；键对象上下文（CmSetCallbackObjectContext 给某个键对象挂一份，此后该对象的全部通知都带 ObjectContext；键对象销毁时收到 RegNtCallbackObjectContextCleanup 通知，此时释放上下文资源）。补信息例程：CmCallbackGetKeyObjectIDEx（从键对象拿 ID 与名字）、CmGetBoundTransaction（拿绑定在操作上的事务对象）、CmGetCallbackVersion（查回调机制版本）。

**两类无效键对象指针（高发蓝屏点）。** 其一，RegNtPostCreateKeyEx/RegNtPostOpenKeyEx 通知里的 Object 字段，只有原操作 Status == STATUS_SUCCESS 时才有效——注意是严格等于，其他 NT_SUCCESS 状态下该字段未定义，拿去用就是踩空。其二，RegNtPreKeyHandleClose/PostKeyHandleClose 与 CallbackObjectContextCleanup 通知里的 Object 指向引用计数已归零、正在销毁的键对象：不许 ObReferenceObjectByPointer 试图续命，但可以把它传给 Cm 例程（如 CmGetBoundTransaction）查信息。两条的共同教训：通知里的对象指针先查本通知的前提条件再碰。

**应用 hive 的坑。** 应用 hive 装在 \REGISTRY\A 下，该路径不可遍历、直接打开必得 STATUS_ACCESS_DENIED，只有 RegLoadAppKey 返回的根句柄能进；整个 hive 共用其文件的那个安全描述符，在单个键上设安全描述符必失败。筛选驱动会收到应用 hive 上的操作通知且 CM 不作区分；若看到某次开键的 CompleteName 是 \REGISTRY\A\ 开头的绝对路径，那是注册表管理器自己在干活，筛选驱动仿效这种开法必失败——别把系统内部动作当成自己也能用的姿势。另有一批未公开调用（NtRestoreKey、NtSaveKey(Ex)、NtLoadKeyEx、NtUnloadKey2、NtUnloadKeyEx、NtReplaceKey、NtRenameKey、NtSetInformationKey）指南明确不建议筛选驱动去改它们的行为。

### 3.9 可靠驱动常见错误清单：官方点名的九组错误

这一节是"可靠驱动程序"章的总纲加逐项展开，审计价值最高。总纲先行：设备对象要有恰当的安全设置；多类型设备对象逐个验；用安全字符串；验句柄；按多处理器设计；查驱动状态；IRP 的全部输入值都要验；下传 IRP 必须用 Skip/Copy 建下一层堆栈位置（不许手抄堆栈位置，那是把 I/O 管理器的账本自己另记一份，必对不上）；不支持的 IRP 不许以 STATUS_SUCCESS 完成（假成功比失败更坏：上层会以为事办了）；取消要对（CSQ 可避开一整类同步坑，第 7 批）；CLEANUP 与 CLOSE 分工分清；Driver Verifier 是最重要的可靠性工具，但它查的是运行时行为，不替代设计。

**① 设备对象不验。** 同一驱动里 DriverEntry 建的控制设备对象与 AddDevice 建的 FDO，设备扩展内容完全不同。分发例程不先判断"这张 IRP 落在哪个设备对象上"就伸手拿扩展字段，拿到的是另一种布局的内存，轻则数据错乱重则崩溃。控制路径与数据路径要么分开分发、要么进门先验对象。

**② 句柄不验。** ObReferenceObjectByHandle 成功只证明"这是个活对象"，不证明"这是我要的那种对象"：还要查对象类型、这次访问申请的权限、访问模式，以及对象里自家的标记字段（如 FsContext 指的私有结构里的 Type 字段是不是自家魔数）。为什么 I/O 管理器不代验：句柄是调用方递进来的，它只属于递交那一刻的语境，驱动的 IOCTL 若拿一个调用方句柄直接引用对象，调用方可以递一个指向别种对象的句柄进来——类型混淆在内核里就是提权原语。

**③ 多处理器错误。** 驱动天生多线程：同一分发例程可以在多个处理器上同时跑。改全局数据、改文件对象上的私有数据，不用锁或 ExInterlocked 系列就是裸奔。指南给的两个典型样本：全局指针在重入时被第二次调用覆写，第一次分配的内存从此泄漏；文件对象上挂的缓冲无锁做 realloc（释放旧的换新的），并发下演成重复释放或用已释放内存。第 8 批同步机制在这里的落点是：先默认一切共享皆并发，再逐个论证哪里可以省锁，而不是反过来。

**④ 状态不查。** 只在 checked 版 ASSERT 里查的状态，到了 free 版等于没查——指南样本：IOCTL_WAIT_FOR_EVENT 的处理只在 ASSERT 里防重复调用，free 版里第二次调用直接把存着的第一张 IRP 弄丢。另一条配套纪律：把 IRP 指针存起来备用之前，必须先 IoMarkIrpPending；且"标记 pending + 存指针"两步要放进互锁序列里，否则多核下另一处理器能在两步之间插进来读到未标记先行值的指针，完成路径的 SL_PENDING_RETURNED 传播（第 6 批）就断了。

**⑤ Buffered I/O 三错。** 其一，不查输入/输出缓冲大小：带 FILE_ANY_ACCESS 的 IOCTL，任何拿到句柄的人都能调（见 3.9⑨）；驱动按假设的最小结构去读输入、或把 Information 写得比输出缓冲还大，后者直接损坏池内存/向用户多报数据。其二，输出缓冲不初始化：Buffered 方式下输入与输出共用一块缓冲，I/O 管理器只把用户输入那一段拷进来，其余部分是池里的旧数据；驱动没写满就按全长返回，用户收到的是内核内存残渣——这是经典的内核信息泄漏形态。其三，变长缓冲校验写错：校验"偏移+长度 ≤ 总长"时偏移与长度都是攻击者可控的 32 位数，相加可以整数溢出绕过；正确写法是减法：先查总长 ≥ 定长头大小（防下一步减法下溢），再查 InputBufferLength − FIELD_OFFSET(结构, 变长成员) ≥ NameLength。同族还有"条数×条大小"的乘法溢出，改用除法比较（总长/条大小 ≥ 条数），与 3.5 的安全整数是同一防线的手算版与库版。

**⑥ Direct I/O 两坑。** 其一，零长缓冲时 I/O 管理器不建 MDL：MdlAddress 是 NULL，MmGetSystemAddressForMdlSafe 对 NULL 返回 NULL，驱动不查就解引用即崩——"长度为零"在校验里要单列。其二，双重映射的写保护是假象：Direct I/O 把用户页双重映射进系统空间供内核访问，驱动常见自保手法是"往缓冲末尾强写一个 NULL 结尾"，但映射的是同一物理页，用户进程的另一条线程随时能把刚写好的 NULL 再覆写掉；推论是两次读同一地址可以读到不同值（TOCTOU），凡"先校验一遍、再放心使用"的代码都中招——校验与使用之间值已变。配套纪律：MmProbeAndLockPages 时声明的访问方式（IoReadAccess 等）必须与后续实际访问一致，按"读"探来的 MDL 不许往里写。

**⑦ 用户地址不验（METHOD_NEITHER 与内嵌指针）。** Buffered/Direct 至少还有 I/O 管理器代管缓冲；METHOD_NEITHER 的 IOCTL/FSCTL，I/O 管理器完全不验用户地址，驱动必须自己：查 IRP 的 RequestorMode、ProbeForRead/ProbeForWrite、全部引用罩进 try/except（3.3）。容易漏的是第二层：Buffered 请求的缓冲本体被 I/O 管理器拷进内核了，但缓冲里内嵌的指针仍指向用户空间，那个指针同样不验——内嵌指针要么逐个 Probe，要么把协议改成相对偏移/变长尾随数据，从根上消灭内嵌指针。

**⑧ CLEANUP 与 CLOSE 混淆。** 两次到达的语义不同：IRP_MJ_CLEANUP 在最后一个句柄关闭时到，IRP_MJ_CLOSE 在最后一个引用释放时到。关键时差：I/O 管理器在普通 I/O 调用进行期间持有文件对象的引用，所以 CLEANUP 之后、CLOSE 之前，该文件对象上仍可能进来新的请求。在 CLEANUP 里把文件对象的资源全释放掉，后到的请求就会摸到悬空资源。分工应该是：CLEANUP 只做"作废挂起请求、停收新活"这类与句柄绑定的事，资源本体留到 CLOSE 才释放（第 7 批已立此规矩，本批是官方清单的复核确认）。

**⑨ 附加 IRP 错误与公私 IOCTL 混流。** IRP 丢失、重复完成、漏调 IoStartNextPacket，多半发生在错误处理路径（正常路径写得再对，错误路径没走通就是雷）。IOCTL 的结构性错误是公私混流：从控制码本身看不出请求来自内核还是用户态，若特权私有 IOCTL 与公开 IOCTL 汇进同一条处理路径且校验按公开标准放水，非特权用户就能摸到特权操作。官方规矩：公私路径必须分开，私有路径的入口校验按内核调用方的身份单独设防。

---

### 3.10 64 位移植：数据类型、对齐、thunking 与 DMA

**头号事实与分工。** 64 位 Windows 给用户程序留了 WOW64 转换层，给驱动什么都没留：32 位驱动不受支持，驱动必须为 64 位原生编译。原因在边界形状：用户程序与内核之间隔着系统调用这道窄口，参数转换可以在口上做；驱动与内核同处一个地址空间、共享结构定义，没有可供转换的"口"，只能同宽。

**LLP64 与类型三类。** 指针变 8 字节，int/long 仍 4 字节。对策类型都在 Basetsd.h：固定精度（INT32/INT64 等，精度写进名字、跨平台同宽，协议与落盘结构用它）、指针精度（ULONG_PTR/SIZE_T 等，随平台变宽，指针算术与计数用它）、固定精度指针（POINTER_32 是 64 位环境里截断表示的 32 位指针、POINTER_64 是 32 位环境里符号扩展的 64 位指针，为跨位宽结构备的，thunking 用）。句柄与整数互转走 HandleToUlong/PtToUlong/IntToPtr/UIntToPtr 这组辅助函数（符号扩展与零扩展的语义由函数名钉死），不许裸强转。平台宏：_WIN64 标 64 位；_WIN32 在 64 位编译器下为兼容也定义着，拿它判位宽必错；_M_IA64/_M_IX86 一类架构宏非架构代码不要用。移植第一遍的标准动作是开 -Wp64 警告组扫一遍（C4311 指针截断、C4312 反向、C4313 格式串、C4305、C4319、C4244），把警告逐条消干净再谈运行。打印指针用 %p——用 %x 打指针会截掉高 32 位。

**对齐：32 位能忍、64 位不忍。** 32 位 Windows 会自动修正内核里发生的对齐故障，只是慢得难看；64 位 Windows 直接不修正，未对齐访问就是故障/异常。为什么分叉：x86 硬件本身能做未对齐访问，32 位时代系统选择了"替你擦屁股保兼容"；到了 64 位新平台，没有存量包袱要保，且自动修正的性能代价与静默掩盖 bug 的代价都不可接受，于是改成立规矩。落地动作三条：结构体偏移一律 FIELD_OFFSET 算、不许手算 malloc(sizeof(头)+n*sizeof(指针))——编译器会在指针成员前塞填充，手算必错（指南例：DWORD 后跟指针数组，64 位上 FIELD_OFFSET 是 8 不是 4）；对齐要求用 TYPE_ALIGNMENT(类型) 取（如 ProbeForRead 的对齐参数）；#pragma pack(1) 压过的结构里装指针，RtlCopyMemory/memcpy 这类例程对未对齐成员不是慢慢做而是抛异常，补救用 UNALIGNED 宏但代价高，正道是定义结构时把 64 位成员与指针放前面。

**算术与强转的静默坑（编译器不报的那些）。** ① 无符号污染：表达式里混一个 ULONG，整个式子按无符号 32 位算，负的 LONG 被转成大正数再缩放加到指针上，32 位上"碰巧对"、64 位上错得离谱。② 十六进制常量与取反顺序：先 ~(PAGE_SIZE−1) 再扩到 64 位，高 32 位是 0 不是全 1，掩码类计算整片错位；必须先转 UINT_PTR 再运算（a & ~((UINT_PTR)b − 1)）。③ 索引回绕失效：DWORD index 为 0 时 index−1 在 32 位回绕成 −1 的位形"碰巧可用"，64 位上是 0x00000000FFFFFFFF，老老实实的越界地址——下标类型改 DWORD_PTR。④ 多态接口：参数能装指针又能装整数的，必须声明成 UINT_PTR/PVOID 不许 DWORD；反例是拿 DWORD 数组装异常参数，64 位上每个指针被截半。⑤ 指针 OUT 参数的覆写：把 (PULONG*)&bufAddress（bufAddress 是 ULONG）传给写 PVOID 的函数，函数写 8 字节，后面 4 字节的邻居被无声砸坏，且强转把编译器的报错也堵死了。⑥ 指针转整数只许转 UINT_PTR/INT_PTR；真要截断用 PtrToLong/PtrToUlong（它们顺手压掉截断警告），但截断值永远不许再转回指针用。顺带：IO_STATUS_BLOCK.Information 在 64 位是 ULONG_PTR，公共结构里这类"悄悄变宽"的成员要逐个核对。

**32 位 I/O 的 thunking。** 场景：64 位驱动对用户态暴露 IOCTL，缓冲里含指针精度成员。WOW64 把缓冲指针本身转成 64 位指针，但内容一字节不动。尺寸就此对不上：含 HANDLE 与 UNICODE_STRING 的 DRIVER_DATA，32 位布局 12 字节、64 位布局 24 字节（多出的含给 Buffer 成员做 8 字节对齐的 4 字节填充），64 位驱动按自家结构验长度必然失败。解法是驱动自己转：定义只含固定精度类型的 32 位版结构（HANDLE→VOID* POINTER_32、UNICODE_STRING→UNICODE_STRING32 等，按指南对照表），识别出 32 位调用方后按 32 位版解析再转成原生。识别两法：一法在控制码里划一个 "64Bit" 位（常用 Function 字段最高位），64 位应用用置位的另一组控制码——查一位最快，代价是要应用配合改；二法驱动调 IoIs32bitProcess(IRP) 自查（传 NULL 是给快速 I/O 路径的）——应用零改动，代价是每次查一下。选哪个看驱动与应用的配合条件。注意固定精度类型也不全免疫：本身是结构体的固定精度类型（如 __int64/LARGE_INTEGER/KFLOATING_SAVE）在不同平台的对齐要求可能不同（x86 上 4 字节、Itanium 上 8 字节），这类成员所在的缓冲同样可能要转，对齐要求一律 TYPE_ALIGNMENT 现查。

**64 位 DMA 的双重缓冲。** 设备不支持 64 位寻址时，系统靠双重缓冲兜底（低地址中转），正确性无碍、性能受损且随物理内存变大而变重（指南量级：8GB 机器上约个位数百分比，I/O 密集负载最疼）。正解四条：物理地址一律 PHYSICAL_ADDRESS 类型承接，绝不截断 MmGetPhysicalAddress 的高 32 位（截断等于把 DMA 指向错误内存：数据损坏、I/O 丢失、系统故障一条龙）；用 GetScatterGatherList/PutScatterGatherList 一族；查 Mm64BitPhysicalAddress 全局变量确认平台情形；DEVICE_DESCRIPTION.Dma64BitAddresses 能置 TRUE 就置。反过来说，32 位时代的 DMA 例程本身就是 64 位就绪的，用对了无需改——问题只出在地址被中途截断的代码上。

**x64 的禁改清单（结构保护）。** x64 上内核代码与若干关键结构受保护，驱动动手改就 CRITICAL_STRUCTURE_CORRUPTION 蓝屏：禁运行时改内核代码、禁用自备栈、禁改 IDT/GDT 等硬件分发表、禁改未公开的内核结构。x86/Itanium 上这些动作不触发这个蓝屏，但指南明说在任何平台都不许做、未来版本不保证能用——这与驱动签名一线的 PatchGuard 是同一政策（另有专文，不在本批展开）。

### 3.11 工具箱小件四则

**安全模式检测。** 内核导出变量 InitSafeBootMode（指针）：0 非安全模式、1 MINIMAL、2 NETWORK、3 DSREPAIR（仅域控）。功能驱动在 AddDevice 里查它决定行为——注意不许从 DriverEntry 返回失败来"拒绝在安全模式加载"（加载时序上那时还轮不到这种表态，且失败会污染启动）；筛选驱动在安全模式下应不挂接设备栈但 AddDevice 返回成功（挂了会干扰安全模式的最小化诊断环境，不挂又报错会让系统以为驱动坏了，"成功但不干活"是正解）。

**驱动内用文件。** 文件在内核按对象名引用：\DosDevices\C:\...（\?? 与 \DosDevices 等价）。流程同注册表三步：InitializeObjectAttributes 填对象名 → ZwCreateFile/ZwOpenFile 开句柄（Create 不存在则建、Open 不存在返回 STATUS_OBJECT_NAME_NOT_FOUND；驱动几乎不用 IoCreateFile）→ ZwReadFile/ZwWriteFile（ByteOffset 指定位置，FILE_APPEND_DATA 打开的句柄忽略偏移恒追加）与 ZwQueryInformationFile/ZwSetInformationFile → ZwClose。默认没有"当前文件位置"（I/O 管理器维护它要为每次读写加同步，默认不付这钱）；要流式读写需开句柄时带 SYNCHRONIZE 访问权加 FILE_SYNCHRONOUS_IO_ALERT/NONALERT 选项，位置用 FilePositionInformation 查改。这里开的句柄同样走 Zw 路线、OBJ_KERNEL_HANDLE——3.1 的规矩在文件与注册表上一以贯之。

**浮点与扩展寄存器。** x86：编译器会拿 x87/MMX 寄存器算浮点，而这些寄存器同时被用户应用用着，驱动用之前必须 KeSaveExtendedProcessorState 保存、完事 KeRestoreExtendedProcessorState 恢复，且浮点代码要放进 noinline 的独立子程序——否则编译器重排可能把浮点指令挪到"检查保存返回值"之前。调用时机 IRQL ≤ DISPATCH_LEVEL，x86 的 ISR 里禁用浮点。x64 分叉：编译器改用 SSE 寄存器且自理保存恢复，那对保存/恢复调用不必要（带 XSTATE_MASK_LEGACY 时调用也无害），ISR 里可以用浮点；但 AVX 这类扩展特性在 x86/x64 上都要显式保存恢复、ISR 里禁用。分叉的根源是两代编译器/ABI（应用二进制接口）对浮点寄存器的所有权约定不同，不是硬件能不能算的问题。

**对象引用跟踪标签。** Win7 起 ObXxxWithTag 一族（ObReferenceObjectWithTag 等）给每次引用/解引用打一个 4 字节标签，无标签版默认 'Dflt'。Gflags 打开对象引用跟踪后，用 !obtrace 按标签对账：某标签 References 与 Dereferences 数不等，就是那类引用在泄漏或过度解引用——把"引用计数对不平"从一锅粥拆成按来源归账。查过度解引用要开 Gflags 的 Permanent 选项，让跟踪记录在对象释放后仍保留，否则对象一释放账本就没了、无从对起。

### 3.12 WMI：概念、发布、注册、处理与事件

**概念与通道。** WMI 在这里是给驱动装的"仪表接口"：驱动成为 WMI 提供者后，用户态消费者（管理应用、脚本、系统自带工具）经 WMI 的用户态 API 提出查询/设置/方法调用/事件订阅，WMI 内核组件把请求翻译成 IRP_MJ_SYSTEM_CONTROL 的 IRP 发给驱动；驱动反过来也能主动发事件，不需要消费者轮询或自发 IRP。好处是指南列的：自定义数据走标准接口暴露、设备配置不必再配专用控制面板、只收被请求的数据开销低、块带类名与描述可被客户端枚举展示。适用前提：处理 IRP 的驱动（类驱动、端口驱动等）自己注册；不处理 IRP 的驱动（SCSI/NDIS 小端口）把 WMI 请求顺栈下转，由下层类/端口驱动代办，小端口只需支持类驱动定的专用接口（如 SCSI 的 SRB_FUNCTION_WMI、设置 WmiDataProvider=TRUE）。

**MOF 模式。** 每个块一条 MOF 类定义。类限定符里必需三件：Dynamic（块的实例由提供者运行时给出；驱动注册的块必须 Dynamic，标 Static 的块数据住在 WMI 数据库里、驱动不要注册）、Provider("WMIProv")、Guid("...")。可选 Description（给客户端展示的说明，配 Locale）、WmiExpense（采集成本提示）。类名在 WMI 命名空间内必须唯一，防撞的通行做法是定义驱动专属的抽象基类、所有块类从它派生（MSSerial 基类之于串口驱动）；事件块必须派生自 WmiEvent，所以事件块的基类要从 WmiEvent 派生一个非抽象中间层再往下派生。每个类（嵌入类除外）必须原样包含两个规定项：[key, read] string InstanceName 与 [read] boolean Active——它们归 WMI 内部使用，驱动响应查询或发事件时不要去填（不属于驱动的数据块内容）。属性限定符管每个数据项：key（只有 InstanceName 能标）、read/write、BitMap/BitValues、DisplayInHex 等。标准块优先：设备类型若有系统已定义的标准块（在 Wmicore.mof 里），应支持标准块，再按需加自定义块。

**发布。** 流程级（指南 Publishing 一节的骨架）：MOF 文件编译成二进制形式，作为资源挂进驱动镜像；注册时把 MOF 资源名报给 WMI，WMI 按块的 GUID 去资源里查类定义，把模式装进 WMI 数据库，客户端才能按名字枚举与显示。也就是说模式（MOF）与数据（驱动运行时给）是两条供应链：模式靠发布一次性进库，数据靠每次 IRP 现查现给——这正是 Dynamic 限定符的含义所在。指南还提到动态 MOF 与本地化 MOF 的变体、以及登记用的注册表值（MofImagePath），细节随查。

**注册两阶段（为什么分段的结，当场解开）。** 表面看注册绕了个圈：驱动先调 IoWMIRegistrationControl(WMIREG_ACTION_REGISTER, 设备对象)，WMI 收到后反手发一张 IRP_MN_REGINFO_EX（98/2000 是 REGINFO）到驱动的 DispatchSystemControl，Parameters.WMI.DataPath 置 WMIREGISTER、ProviderId 置该设备对象指针，驱动在这张 IRP 里才交出块清单等全部详情。结为什么形成：登记调用的参数只能装下"哪个设备要注册"这点信息，而 WMI 真正需要的是整份块清单、注册表路径、MOF 资源名、实例名来源，塞不进一个调用；把"索取详情"做成一张 IRP，又能让它复用既有的分发/排队/完成模型（包括在设备完全初始化好、能处理 WMI IRP 之后才走到这一步）。打破循环的就是这个两段式：第一段只登记意向，第二段走标准 IRP 把详情拉过去。注册时机在设备启动、初始化到能处理 WMI IRP 的程度之后。

**两条注册/处理路线。** 路线甲，WMI 库：驱动填 WMILIB_CONTEXT（GuidCount、WMIGUIDREGINFO 列表——每个块的 GUID/实例数/注册标志——加 DpWmiXxx 回调入口），收到 WMI IRP 时调 WmiSystemControl 交库处理；库回调 DpWmiQueryReginfo 时驱动报注册表路径、MOF 资源名、全块注册标志与实例命名依据（PDO 指针或基名字符串）。限制：库没有传动态实例名与静态名列表的机制，只支持 PDO/基名两种静态命名——块用动态名或名字列表的，这条路走不通。路线乙，自办 REGINFO：驱动在 IRP 的 Parameters.WMI.Buffer 里亲手填 WMIREGINFO（总字节数、自家注册表路径、MOF 资源名、块数、WMIREGGUID 数组），每个块一条 WMIREGGUID：GUID、标志、以及静态命名的落点（INSTANCE_LIST 时 InstanceNameList 是名字列表的偏移；INSTANCE_BASENAME 时 BaseNameOffset 是基名偏移；INSTANCE_PDO 时 Pdo 放 PDO 指针，处理 REGINFO_EX 时必须先对该 PDO 调 ObReferenceObject 加引用，系统事后自会解引用、驱动不许自己解）。替别的驱动代注册（类驱动代小类驱动）时再续一份 WMIREGINFO、用 NextWmiRegInfo 偏移串起来。

**注册标志。** 三个 INSTANCE_* 标志互斥地声明静态命名来源，一个都不设即动态命名。WMIREG_FLAG_EXPENSIVE：块采集昂贵，WMI 会在第一个客户端打开该块时发 IRP_MN_ENABLE_COLLECTION、最后一个关闭时发 IRP_MN_DISABLE_COLLECTION，驱动在收到 ENABLE 之前可以不采集。WMIREG_FLAG_EVENT_ONLY_GUID：这是事件块，只能启用/禁用。WMIREG_FLAG_REMOVE_GUID：只在更新注册信息时有效，声明注销某个块。

**更新与注销。** 初次注册之后再调 IoWMIRegistrationControl 可选三种动作：WMIREG_ACTION_REREGISTER（全量替换，WMI 回发 REGINFO(_EX) 索取新详情）；WMIREG_ACTION_UPDATE_GUIDS（增删块或改静态名，WMI 回发 DataPath 为 WMIUPDATE 的 REGINFO(_EX)，块上 REMOVE_GUID 置位即删、清零即增改）；WMIREG_ACTION_DEREGISTER（注销，WMI 不再回发 IRP；驱动通常在处理 IRP_MN_REMOVE_DEVICE 时注销）。注意注销调用会阻塞到该设备的全部 WMI IRP 完成为止——驱动若把 WMI IRP 排了队，必须先把队列里的取消干净再调注销，否则自锁。

**处理请求。** 所有驱动必须给 DispatchSystemControl 设入口；注册了的自己办，没注册的原样下传（未知次功能码也下传）。WMI 次功能码全表（均为 IRP_MJ_SYSTEM_CONTROL 的次码）：REGINFO、REGINFO_EX（注册/更新详情）、QUERY_ALL_DATA（查某块全部实例）、QUERY_SINGLE_INSTANCE（查单个实例）、CHANGE_SINGLE_INSTANCE（改一个实例的全部项）、CHANGE_SINGLE_ITEM（改单个项）、ENABLE_COLLECTION/DISABLE_COLLECTION（昂贵块的采集开关）、ENABLE_EVENTS/DISABLE_EVENTS（事件开关）、EXECUTE_METHOD（执行块的方法）。走库路线时，WMI 验完 IRP 参数后调对应 DpWmiXxx（QueryReginfo 与 QueryDataBlock 必需，SetDataBlock/SetDataItem/ExecuteMethod/FunctionControl 可选；没设入口的可选回调，库以默认值与状态替驱动完成 IRP），回调里除 QueryReginfo 外用 WmiCompleteRequest 收尾（或返 STATUS_PENDING 延后完成，与普通 IRP 同规矩），库再做后处理、把输出打包成 WNODE 交消费者。

**自办的校验清单（DispatchSystemControl 内）。** ① 先认领：Parameters.WMI.ProviderId 必须等于自己注册时传的设备对象指针，不是自己的立刻下传。② 认块：查 Parameters.WMI.DataPath 的 GUID，不支持就以 STATUS_WMI_GUID_NOT_FOUND 失败。③ 认实例（QUERY_SINGLE_INSTANCE/CHANGE_SINGLE_INSTANCE/CHANGE_SINGLE_ITEM/EXECUTE_METHOD 要查）：输入 WNODE 的 WnodeHeader.Flags 置了 WNODE_FLAG_STATIC_INSTANCE_NAMES 就用 InstanceIndex 索引静态名列表，没置就用 OffsetInstanceName 偏移去读名字串（偏移处先是 USHORT 字节长度、后跟 Unicode 串）；找不到实例以 STATUS_WMI_INSTANCE_NOT_FOUND 失败。EXECUTE_METHOD 还要验 WNODE_METHOD_ITEM 里的 MethodID，无效回 STATUS_WMI_ITEMID_NOT_FOUND。④ 验缓冲（QUERY_ALL_DATA/QUERY_SINGLE_INSTANCE/EXECUTE_METHOD 出数据时）：缓冲小于所需但至少装得下 WNODE_TOO_SMALL，就成功这张 IRP 并写入 WNODE_TOO_SMALL 结构告知所需大小；连 WNODE_TOO_SMALL 都装不下，回 STATUS_BUFFER_TOO_SMALL 失败。⑤ 交数据：QUERY_ALL_DATA 写 WNODE_ALL_DATA，QUERY_SINGLE_INSTANCE 写 WNODE_SINGLE_INSTANCE，方法输出按驱动自定格式覆写在输入 WNODE_METHOD_ITEM 之后；设 IoStatus.Information 为写入字节数、Status 为 STATUS_SUCCESS，IoCompleteRequest 完成。

**WNODE 结构族要点。** 走库路线不用碰它们；自办必须亲手读写。共性：头部嵌 WNODE_HEADER（BufferSize、Guid、Flags——标明这是哪种 WNODE、用静态还是动态名等）；固定成员之后跟可变数据（实例名、方法入出参、实例数据），所以缓冲总要大于 sizeof(结构本身)。硬纪律两条：WMI 不对驱动给的可变数据做类型检查，数据错位消费者直接解错；对齐要驱动自己保证——每个实例从 8 字节边界开始、每个数据项按其模式定义的自然边界对齐、动态实例名 2 字节对齐即可。WNODE_ALL_DATA 与单实例版的差别在于它装多实例：InstanceCount 计数，定长实例置 WNODE_FLAG_FIXED_INSTANCE_SIZE 并给 FixedInstanceSize，变长实例改给偏移/长度数组（OffsetInstanceDataAndLength）。

**发送事件。** 前提：事件块已以 EVENT_ONLY 注册、且 WMI 已发 IRP_MN_ENABLE_EVENTS 启用——没启用就发是白发（且违反模型）。触发后两条发送路线：WmiFireEvent（库路线）只适用于静态命名（基名或 PDO 实例路径）、单实例、无数据的事件可免缓冲；要带数据就从非分页池分配缓冲装数据，调完由 WMI 释放，参数内存一律非分页池——因为发送动作可能发生在不能碰分页内存的上下文（与第 8 批"上下文决定能碰什么内存"的通则一致）。IoWMIWriteEvent（自办路线）什么事件都能发（单项、单实例、全实例、动态名），代价是 WNODE_XXX 要驱动全员自备自初始化：置 WNODE_FLAG_EVENT_ITEM、类型标志（ALL_DATA/SINGLE_INSTANCE/SINGLE_ITEM 之一）、命名标志（STATIC_INSTANCE_NAMES/PDO_INSTANCE_NAMES 按实情），转成 PWNODE_EVENT_ITEM 后调用，成功后 WMI 释放内存。体积红线：WNODE 加数据不得超过注册表定的 1KB 上限（事件越小到得越快）；确需超限（不推荐）改发 WNODE_EVENT_REFERENCE（只给 GUID、大小、实例索引或名字），WMI 凭它回查事件全量。这条"引用代全量"的兜底与错误日志里 DumpData 只留关键字节是同种克制。

---

## 四、解读（审计与逆向视角）

**PreviousMode 是审计的显微镜。** 逆向一个驱动的系统服务调用点时，先看它调的是 Nt 版还是 Zw 版：调 Zw 版且参数来自 IRP 携带的用户缓冲，是把用户数据按"可信"放行的经典漏洞形态——验证被整段跳过。反过来，看到驱动对自家句柄用 Nt 版，先标记"任意线程上下文下可能句柄泄漏/误伤"，再找它的调用上下文核对。ExGetPreviousMode 的出现位置也值得记：它附近往往就是驱动在做"按来源分流"的判断。

**常见错误清单就是一份现成的漏洞模式库。** ⑤ 的输出缓冲不初始化、定长头校验缺失、偏移+长度加法校验、计数×大小乘法溢出，是 Windows 内核提权漏洞通报里反复出现的四种形态，审驱动 IOCTL 时按这四式逐个对表最省力。⑦ 提醒审 METHOD_NEITHER 时要追到第二层：缓冲本体没问题时，专查内嵌指针。② 的"引用成功≠对象对"对应类型混淆类漏洞的入口形态。⑨ 的公私混流则是逻辑提权：不靠内存破坏，靠路径设计——审 IOCTL 分发表时把"这个控制码谁能调、走哪条路径"列成表，混流一眼可见。

**注册表筛选是双刃位置。** 它坐在一切注册表操作的必经之路上，安全软件靠它做自保护与监控，恶意软件（历史上带签名的恶意驱动）也垂涎这个位置：pre 阻断可以藏键，接管（BYPASS）可以给查询方喂假数据。逆向分析带 CM 回调的驱动时，重点看三处：altitude（站位多高）、pre 里对哪些键名返回非成功/BYPASS、post 里改了哪些输出——这三处就是它的行为边界。另注意 \REGISTRY\A 开头的开键动作是系统自身的，别误判成样本在直捣应用 hive。

**错误日志与蓝屏回调是取证面。** 排查一台蓝过的机器时，驱动装没装消息源决定了事件查看器里的记录能不能读成人话；写了 KbCallbackSecondaryDumpData 的驱动，其数据块在转储的二级区里按 GUID 检索——找驱动自述现场先查这里。反过来，一个声称企业级的驱动若完全不写错误日志，本身就是成熟度信号。

**64 位 thunking 是兼容层的裂缝带。** 32/64 两套结构定义并存，意味着同一 IOCTL 有两种解析路径；审计时按 32 位布局把每条边界再走一遍，历史上不乏"64 位验过、32 位布局下溢出"的案例。IoIs32bitProcess 分支之后的解析代码是重点：它处理的是 WOW64 应用递来的、与原生布局不同的数据，校验常按原生布局写就。另注意 POINTER_32 的截断语义：thunk 后的指针只有低 32 位，驱动若把它当完整指针回写给 64 位消费者，信息丢失点就在那里。

**WMI 是被低估的情报面。** 驱动注册的 MOF 模式是自带文档的数据字典：块名、项名、读写性都写明了——分析一个带 WMI 的驱动，先把它的 MOF 资源提出来读，等于拿到作者自备的接口说明书。EXPENSIVE 标志与 ENABLE_COLLECTION 的存在还透露了哪些数据"平时不采、被打开才采"。从防守看，WMI 事件通道与 ETW（Windows 事件追踪）并列是系统自带遥测面；从分析看，IoWMIWriteEvent 的调用点就是驱动"觉得值得上报"的事件清单。

**x64 禁改清单与引用标签是平台态度的注脚。** 前者说明 x64 起微软把"内核完整性"从建议升格为强制执行（改了就蓝屏，不与驱动商量）；后者（ObXxxWithTag + !obtrace）说明引用计数的可观测性被做进系统——调试自己驱动的引用泄漏时，这是比逐行人肉对账高效得多的官方工具。

## 五、易混点辨析

- **Nt 版 vs Zw 版**：不是"一个给用户态、一个给内核态"的分工（用户态调两者无区别），分界是验证语义：Zw＝强制按 KernelMode 不验，Nt＝按当前线程 PreviousMode 决定。驱动用错 Nt 的后果不是"多验一遍无所谓"，而是验错了对象（借别人句柄表查）与验错了地址（系统地址被当用户地址 Probe）。
- **STATUS_SUCCESS vs "成功"**：前者是成功区间里的一个点。== 直比漏掉信息性成功与警告；只用 NT_SUCCESS 又会在警告带部分数据时当全量数据用。两个错误方向相反，审代码时都要查。
- **安全字符串 vs 安全整数**：一个防写越界（拷贝时不知道目标多大），一个防算越界（算大小时回绕）。只用前者，大小算错了它照样"安全地"把错大小的数据写进缓冲；只用后者，裸 strcpy 仍能溢出。成对出现才完整。
- **CLEANUP vs CLOSE 再确认**：CLEANUP 跟最后一个句柄走、CLOSE 跟最后一个引用走；I/O 调用期间系统持引用，所以"CLEANUP 后无新请求"是错觉。与之相邻的另一个混淆：ZwDeleteKey 后句柄不用再关（也不许再关），而普通 ZwOpenKey 的句柄必须 ZwClose——"删键"与"关句柄"是两个动作，别把前者当后者的替代。
- **Zw 注册表路线 vs Rtl 路线**：不是新旧关系，是句柄成本与能力的关系：Rtl 省事但每次开关句柄且不能枚举子键；Zw 繁琐但同键多操作只开一次。还有第三条（PnP 开口）不是风格选择而是强制：四棵 PnP 树只有 IoOpenDeviceRegistryKey/IoOpenDeviceInterfaceRegistryKey 这一扇门。
- **注册表筛选的"阻断" vs "接管"**：pre 返回非成功状态是阻断（操作不执行、调用方收到该状态、且没有 post）；返回 STATUS_CALLBACK_BYPASS 是接管（操作不执行、但调用方收到 SUCCESS，输出由接管方伪造填妥）。两者对调用方的可见效果南辕北辙，写筛选时选错一个字，语义整个反过来。post 改返回值同样要走 BYPASS 约定并自理对象收尾。
- **双重映射 vs 双重缓冲**：同名近亲、完全两事。双重映射（第 7 批 Direct I/O）是同一物理页挂两个虚拟地址，威胁是 TOCTOU（内容可被用户线程改）；双重缓冲（本批 64 位 DMA）是数据在低地址中转一份，代价是性能。审材料时看到 double 先定位语境再下结论。
- **WOW64 的 thunking vs 驱动的 thunking**：WOW64 只转系统 API 的参数与 IOCTL 缓冲的指针本身，不转缓冲内容；64 位驱动支持 32 位 I/O 要转的正是内容。把"系统会替我转"当默认假设，是 32 位应用在 64 位驱动上全灭的典型原因。
- **WMI 库路线 vs 自办路线**：库不是"低级版"，是拿能力换省事：省掉 WNODE 读写与参数校验，代价是动态实例名与静态名列表用不了。反过来块全是 PDO 静态名时还自办，等于白白接手一整套易错的打包活。
- **WMI 事件 vs 错误日志**：事件是给在线消费者的实时通知（没启用的事件不该发、没人订阅就没人看），错误日志是给事后管理员的持久记录（写了就进系统日志）。指南明说事件不该当错误日志的替代品——异常情况只发事件不写日志，等于把现场只喊给当时正好在听的人。
- **REGINFO vs REGINFO_EX**：同一个"索取注册详情"动作的两个版本，98/2000 用前者、XP 起用后者；两者的数据结构大体同构，实打实的差别在 INSTANCE_PDO：走 EX 时驱动必须对报上去的 PDO 先 ObReferenceObject。对每个 GUID "登记"动作本身则是 IoWMIRegistrationControl，与这两张 IRP 不是一回事（前者是驱动主动调的例程，后者是 WMI 回发的 IRP）。

## 六、自查问题

1. 驱动在某用户线程借来的上下文里要关闭自己早先打开的内核句柄，应调 NtClose 还是 ZwClose？若错用 NtClose，会发生哪两个后果？
2. 为什么说"返回值 == STATUS_SUCCESS 就当成功"是双向错误？各举一个指南点名的状态码说明两个方向。
3. 自定义 STATUS_XXX 哪条路径可以用、哪条必须映射回系统值、哪条绝不许用？判据是什么？
4. 安全字符串函数的四条硬约定是什么？其中"恒 NULL 结尾"为什么对截断情形尤其重要？
5. 安全整数库为什么没有除法函数？算术函数为什么只吃无符号数，有符号数要先做什么？
6. 写错误日志时 %1 替换成什么？用预定义 IO_ERR_XXX 的驱动在消息源注册里必须包含哪个文件？TypesSupported 一般填几？
7. Bug Check Reason Callback 在什么 IRQL 跑？四条禁令是什么？KbCallbackSecondaryDumpData 的 GUID 纪律是什么？
8. Zw 注册表路线读一个值为什么要"两步法"？ZwFlushKey 为什么不能随手用？
9. Rtl 注册表路线的两个硬伤是什么？四棵严禁直访的 PnP 树是哪四棵、正口是哪两个例程？
10. 注册表筛选里，pre 返回非成功状态与返回 STATUS_CALLBACK_BYPASS 分别发生什么？被阻断的操作还有 post 通知吗？
11. 哪两类通知里的键对象指针是无效的？各自允许与禁止的动作是什么？
12. 常见错误清单里 Buffered I/O 三错各是什么？"偏移+长度"校验为什么必须改成减法式？
13. "往双重映射缓冲末尾写 NULL 就安全了"为什么错？由它推出的一般性结论（关于校验与使用的关系）是什么？
14. CLEANUP 里为什么不能释放文件对象的全部资源？I/O 管理器在其中扮演什么角色？
15. LLP64 下 DRIVER_DATA（含 HANDLE 与 UNICODE_STRING）在 32/64 位下各多少字节？差额由什么构成？
16. 识别 32 位 IOCTL 调用方的两种办法各有什么代价？为什么这件事不能指望 WOW64？
17. 32 位 Windows 与 64 位 Windows 对内核对齐故障的处置差在哪？移植时对应的三条落地动作是什么？
18. WMI 注册为什么要分两阶段？两阶段各由什么动作构成？
19. 哪些块走不了 WMI 库路线？REGINFO_EX 里报 INSTANCE_PDO 时驱动多出一个什么义务？
20. 自办 WMI 查询时缓冲"小但装得下 WNODE_TOO_SMALL"与"连它都装不下"分别怎么处理？事件发送的 1KB 红线超了走什么改道？

## 未钉死 / 待查清单

- 安全字符串（Ntstrsafe.h）与安全整数（ntintsafe.h）的函数全表及其参数细节：指南的 Summary 表在 PDF 版面里被逐字打碎、不可转录，本批只钉死了规则与代表函数，全表以 WDK 头文件为准随查。
- MOF 语法细节（属性限定符全表、嵌入类、数组与变长数据项定义）与块设计准则（Designing WMI Data and Event Blocks 一节的展开建议）：本批按路线要求取概念级，写真实块时需回指南该节与 SDK 的 MOF 文档逐项核对。
- WMI 发布一节的登记细节（MofImagePath 注册表值的确切位置与写法、动态 MOF 与本地化 MOF 的完整流程）：本批按流程骨架钉死，落地时回指南 Publishing a WMI Schema 一节核对。
- 标准 WMI 块清单（Wmicore.mof 里为各设备类定义的标准块）与 WMI 属性页、WmiMofCk 校验工具、WMI 事件追踪（与 ETW 的关系）、测试排障一节：路线范围外沿，只点名，未逐节研读。
- "编程技术"章内未入路线第 12 批范围的子系统（CLFS、KTM、动态硬件分区、各类通知例程）与过时/保留例程、全局变量、宏的清单词表：只点名存在，需要时按 DDI 参考集随查。
- 访问权一节（ACCESS_MASK 构成、GENERIC_* 到具体权限的映射规则）与第 5 批安全性重叠，本批未重读原文，引用时以第 5 批口径为准、细节随查 DDI。
