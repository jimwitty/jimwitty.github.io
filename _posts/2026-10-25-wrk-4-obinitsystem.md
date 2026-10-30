---
layout: post
title: "WRK 第 4 批：对象与句柄基础设施点火 — ObInitSystem"
categories: WRK
series_order: 5
---

> 取材说明：本批以 WRK 源码（WS03 SP1 代码线，`BASE/NTOS/`）为底本，主线是 `OB/OBINIT.C` 的 `ObInitSystem`，辅以 `OB/OBTYPE.C`（类型注册）、`OB/OBCREATE.C`（对象分配）、`OB/OBDIR.C`（目录与哈希）、`OB/OBHANDLE.C`（句柄）、`OB/OBSDATA.C`（安全描述符缓存）、`EX/HANDLE.C`（句柄表）、`SE/SEINIT.C` 与 `SE/TOKEN.C`（第四类 Token）。对位资料：Design Notes 合集 #24《Object Management》精读（规范—实现对子）、Probert《Object Manager》讲精读（四层模型与句柄表三级结构）。ntldr、win32k 本体不在 WRK 树内，本批不涉及。

## 本批地图：Ob 在点火链里的位置

`ExpInitializeExecutive`（`INIT/INITOS.C`）两次调 `ObInitSystem`：Phase 0 在 `:827`，Phase 1 在 `:1250`。Phase 0 的前后邻居决定了 Ob 的位置为什么非如此不可：

- 前面已就绪的依赖：`MmInitSystem(0)`（`:582`，池已可用——第 3 批）、`ExInitSystem`（`:568`，前瞻列表、推送锁、工作项）、`Ke`（事件、PRCB）。Ob Phase 0 只用这三样，不碰文件系统、不碰注册表、不碰进程。
- 后面依赖 Ob 的子系统：`SeInitSystem`（Phase 0 在 `:831`，紧跟 Ob 之后——Token 类型要靠 Ob 注册）、`PsInitSystem(0)`（`:835`，进程与线程本身就是对象，还要句柄表）、`PpInitSystem(0)`（`:839`）、`IoInitSystem`（Phase 1 在 `:1531`，设备与驱动全是对象）、`CmInitSystem1`（Phase 1 在 `:1437`，键是对象）。Ob 若不在 Phase 0 最早一批点火，后面所有子系统连"创建一个东西"这个动作都做不了。
- Phase 1 的 Ob（`:1250`）在 `Se`（`:1271`）、`Mm(1)`（`:1286`）之前：根目录、`\ObjectTypes`、`\DosDevices` 必须先立好，`MmInitSystem(1)` 登记 Section 类型时（第 3 批）才能自动插进 `\ObjectTypes`；`Se` Phase 1 的工作（`:1271` 前的注释原话："SE expects directory and executive objects to be available"）也明说它要目录先存在。
- 交接点：第 3 批 Phase 1 的 `MiSectionInitialization`（`MM/SECTSUP.C:958`）调 `ObCreateObjectType` 登记 Section——那是 Phase 1 结束时系统里的第五类（前四类见正文 ⑥）；第 5 批的进程/线程对象、第 7 批的设备/驱动对象，注册时走的都是本批立下的同一条 `ObCreateObjectType` 路。

## 第一部分：术语表

表头说明：按引用级排序，基础概念在前，派生概念在后；读到某条时，它引用的词原则上前面都已定义。少数条目为行文所迫先提后组概念，均在提及处以括号指到定义组；互为前提的两处（对象 ↔ 对象类型、句柄 ↔ 句柄表）先出现者给行内解释并指到后条。沿用第 0、1、2、3 批术语表（IRQL 档、分页池与非分页池、进程与线程、PFN、PTE、VA 等已定义词不再重复）。

### 一、对象与名字基础

- **对象管理器（OB）**：内核中统一管理一切"对象"的子系统，源码在 `BASE/NTOS/OB/`。做什么用：给所有内核数据结构（进程、线程、文件、设备、事件、令牌……）一套统一的命名、引用计数、句柄、安全检查。为什么要有：没有它，每个子系统各造一套名字与权限机制，驱动与系统服务之间无法用统一方式（名字 + 句柄）互相引用。
- **对象（object）**：内核里一个被对象管理器登记的数据结构实例。做什么用：它是内核资源的统一载体。为什么要有统一载体：只有先统一成"对象"，才能统一做命名（路径）、引用（指针计数）、访问（句柄 + 安全描述符）、枚举（按类型列出）。对象分两部分：对象头（见第二组）+ 对象体（见下条）。对象与对象类型互为前提：每个对象都属于一个类型，而类型本身也是对象（见第三组"类型对象"，类型对象的类型就是它自己）。
- **对象体（Body）**：对象中类型专属的那部分数据，如进程的 `EPROCESS` 字段、目录的哈希桶、符号链接的目标串。做什么用：装该类对象真正的状态。为什么对象头与对象体要分开：头（完整定义见第二组）是所有类型共有的（计数、类型指针、安全描述符指针），体是各类型私有的；分开后对象管理器只认头就能做通用操作（引用、删名、查安全），不必认识每种体的布局。
- **对象名**：对象的 `UNICODE_STRING` 名字（如 `Type`、`KernelObjects`）。做什么用：在命名空间里定位对象。为什么名字要单独成段（见第二组"对象头"）：名字可有可无（匿名对象没有名字），且名字本身要进目录（目录见第四组）、要算哈希、要收配额，独立成段才能按需分配。
- **命名空间（namespace）**：以根目录 `\` 为根的对象目录树（目录见第四组），目录与对象混居一树。做什么用：给内核对象一套类文件系统的路径寻址（`\Device\HardDisk0`、`\ObjectTypes`、`\KernelObjects`）。为什么不用纯指针：路径让驱动、系统服务、调试器能用名字互相找到对象，而不必事先持有指针；也让权限能按路径上的目录逐级裁决。
- **对象属性（object attributes）**：创建时 `OBJECT_ATTRIBUTES` 里的标志位，决定对象的行为。本批用到的：`OBJ_CASE_INSENSITIVE`（名字大小写不敏感）、`OBJ_PERMANENT`（永久对象，见下条）、`OBJ_EXCLUSIVE`（独占对象）、`OBJ_INHERIT`（句柄可被子进程继承，见第五组）、`OBJ_OPENLINK`（打开符号链接本身而不跟随，见第四组）、`OBJ_KERNEL_EXCLUSIVE`（内核独占保护）。为什么属性要在创建时一次定死：属性影响对象头的标志位与配额段的有无，创建后改属性等于改对象的不变式，不如创建时定死。
- **永久对象（permanent object）**：`OBJ_PERMANENT` 置位的对象，句柄计数（见本组句柄计数条）归零后也不删除。做什么用：根目录、`\ObjectTypes`、`\KernelObjects` 这类基础设施。为什么要有：它们是命名空间的骨架，若随最后一个句柄关闭而消失，整个命名空间就塌了；永久对象只有显式删除才消失。
- **独占对象（exclusive object）**：`OBJ_EXCLUSIVE` 置位的对象，同一时间只允许一个进程打开它的句柄。做什么用：需要排他访问的资源。为什么要有：有些资源（如某些设备）语义上不允许多路打开，独占标志让对象管理器在打开时直接拒绝第二个打开者，不必每类对象自己再写排他逻辑。独占对象要单独收配额段（见第二组"配额"），因为独占关系要记"哪个进程独占了它"（`ExclusiveProcess`）。
- **内核对象（kernel object）**：`OB_FLAG_KERNEL_OBJECT` 置位的对象，由内核模式创建。做什么用：标记对象的创建者模式。为什么要区分：内核对象与用户对象在配额、安全检查的默认行为上不同（内核创建者不受用户配额（见第二组）与部分访问检查约束）；标志位让检查点一眼区分，不必回查创建者。
- **引用计数（PointerCount）**：对象头里的指针引用数，对象诞生时置 1（给创建者的返回引用），有名字再 +1（目录持有的一份引用）。做什么用：决定对象何时真正释放。为什么要有：指针与句柄是两套引用（见下条），指针计数归零才是"再也没有任何指针指向它"，此时才能释放内存；只看句柄计数会提前释放仍被指针引用的对象。
- **句柄计数（HandleCount）**：对象头里的句柄引用数，插入句柄表 +1，关闭句柄 −1。做什么用：决定对象何时可以删名。为什么要有：句柄是用户态与内核其他部分持有对象的凭证；句柄计数归零表示"再也没有句柄能打开它"，此时若对象非永久且有名字，就可以尝试删名（见正文 ⑮ 的条件删除）。
- **指针计数与句柄计数的铁律**：`PointerCount` 永不允许低于 `HandleCount`。为什么：每个句柄背后都有一份指针引用（句柄表项指向对象体，对象必须活着句柄才有效）；若指针计数先归零，对象被释放，句柄表项就成了悬空指针。维护这条铁律是 `ObpCreateHandle`、`ObpIncrementHandleCount`、`ObpDecrementHandleCount`（`OB/OBHANDLE.C:2475`、`1532`、`2307`）的共同责任；Design Notes #24 也把它写成设计铁律。

### 二、对象头与配额

- **对象头（OBJECT_HEADER，`INC/OB.H:419`）**：每个对象体前面的一段固定 + 可选结构。固定部分：`PointerCount`、`HandleCount`、类型指针 `Type`、三个偏移（`NameInfoOffset`、`HandleInfoOffset`、`QuotaInfoOffset`）、`Flags`、`ObjectCreateInfo`（创建期）或 `QuotaBlockCharged`（创建后）、安全描述符（见第六组）指针 `SecurityDescriptor`、占位 `Body`。可选段（按固定顺序排在固定头之前）：配额段、句柄段、名字段、创建者段。做什么用：装对象管理器通用操作所需的全部元数据。为什么可选段要按固定顺序、且偏移"回指"：分配时按配额 → 句柄 → 名字 → 创建者的顺序依次布局，释放时按逆序回收（`OB/OBCREATE.C:1208` 的注释原话："must be the inverse"）；偏移量记录的是"从固定头往回多少字节是该段"，宏（`OBJECT_HEADER_TO_NAME_INFO` 等）凭偏移就能从体指针反推各段，不必存四个指针（省内存，且体指针本身就能定位一切）。
- **配额（quota）**：对进程收取的池内存账单，记在对象头的配额段（`PAGEDPoolCharge`、`NonPagedPoolCharge`、`SecurityDescriptorCharge`、`ExclusiveProcess`）。做什么用：防止一个进程无限制创建对象耗尽池。为什么按对象收费、且独占与超默认收费才单独立段：普通对象收默认价（类型初始化器里的 `DefaultPagedPoolCharge` / `DefaultNonPagedPoolCharge`，见第三组），不必为每个对象记账；只有收费偏离默认值、或独占（要记独占进程）时才需要配额段。配额系统由 `PsInitializeQuotaSystem` 在 Ob Phase 0 初始化（`OB/OBINIT.C:391`）。
- **前瞻列表（lookaside list）**：预先分配好的一批等长小块，供高频分配快速取用，用完归还而不真正释放。做什么用：对象创建信息（`OBJECT_CREATE_INFORMATION`）与名字缓冲（`OBJECT_NAME_BUFFER_SIZE`）是对象创建路径上每次都要分配的小块。为什么不用通用池分配：通用池分配要拿池锁、可能触发扩容，高频路径上竞争大；前瞻列表按 CPU 分片（见正文 ① 与 ⑦ 的 L/P 两级），无锁或低锁取用，速度与碎片都好得多。深度按系统规模定（见正文 ①）。
- **推送锁（pushlock，`EX_PUSH_LOCK`）**：一种可共享读、独占写的轻量锁，可在 `APC_LEVEL` 及以下获取，可阻塞等待。做什么用：目录锁（第四组，`OBJECT_DIRECTORY.Lock`）、对象管理器的全局锁（`ObpLock`）、句柄表扩表锁（第五组）、SD 缓存每链的锁（第六组）。为什么不用自旋锁：这些锁的持有时间长（要走哈希、比字符串、调内存分配），且持有者可能阻塞；自旋锁只能短持、不能阻塞（第 2 批 QA 12/13 的铁律）。为什么不用 `ERESOURCE`：推送锁比 `ERESOURCE` 轻（一个指针大小，支持共享/独占与"推"式等待），适合高频、短临界区的场合。没有会怎样：若用自旋锁代替推送锁，持有者在需要阻塞时无法阻塞（自旋锁持有期间不能调度），长临界区下其他核空转等待，阻塞需求出现时直接死锁。
- **资源锁（ERESOURCE）**：支持共享读、独占写、可递归、可超时的重量级同步对象。做什么用：每个对象类型（见第三组）的 `Mutex`（类型级互斥）与 `ObjectLocks[4]`（对象级锁数组）。为什么类型用 `ERESOURCE` 而不用推送锁：类型锁的持有时间更长（要遍历类型列表、做配额与安全操作），且需要"共享读"语义（如枚举类型对象时多读并行）；`ERESOURCE` 的共享/独占与拥有者追踪正好匹配。没有会怎样：若用推送锁代替 `ERESOURCE`，类型锁的共享读并行、递归获取与超时等待都无从实现（推送锁无拥有者追踪与递归语义），枚举与删除路径会互相阻塞或误判死锁。`OBJECT_LOCK_COUNT` 为 4（`INC/OB.H:300` 一带），四把锁按哈希分片，降竞争。
- **延迟删除工作项（ObpRemoveObjectWorkItem）**：对象不能在引用归零当场删除时，先挂进删除队列（`ObpRemoveObjectList`，Phase 0 置 `NULL`，`OB/OBINIT.C:369`），由工作线程在合适级别真正删除。做什么用：兜底删除路径。为什么需要延迟：有些删除操作只能在低 IRQL 做（如释放分页池对象），而最后一次引用释放可能发生在高 IRQL；当场删会违反 IRQL 规则，挂队到工作线程（`PASSIVE_LEVEL`）再删就合法。具体哪些路径入队，本批未细读删除全程，列入未钉死清单。

### 三、类型与方法表

- **对象类型（OBJECT_TYPE，`INC/OB.H:302`）**：描述一类对象的元数据结构，字段：`Mutex`（类型互斥体，`ERESOURCE`）、`TypeList`（该类对象的链表头）、`Name`（类型名，从对象头复制来的方便字段）、`DefaultObject`（默认等待对象，见下条）、`Index`（类型编号）、`TotalNumberOfObjects` / `TotalNumberOfHandles`（计数与高水位）、`TypeInfo`（初始化器，见下条）、`Key`（池标签见第七组，`POOL_TAGGING` 开启时）、`ObjectLocks[4]`（对象锁数组）。做什么用：它是"类的类"——所有同类对象共享的行为与记账都挂在这里。为什么要有：对象管理器只认头，体的行为差异（怎么删、怎么解析名字、怎么做安全检查）必须有个地方挂——类型就是这个地方。
- **类型对象（type object）**：类型本身作为对象存在时的实例，如 `ObpTypeObjectType`（"Type" 类型）、`ObpDirectoryObjectType`（"Directory" 类型）、`ObpSymbolicLinkObjectType`（"SymbolicLink" 类型）。做什么用：让类型也能被命名（进 `\ObjectTypes`）、被引用计数、被枚举。为什么类型也要是对象：只有类型也是对象，对象管理器才能用同一套机制管理类型（创建类型 = 创建一个"Type"类型的对象）；这就引出了自举问题（见正文 ④ 的鸡生蛋一）。
- **对象类型初始化器（OBJECT_TYPE_INITIALIZER，`INC/OB.H:277`）**：注册类型时传入的行为契约，字段：`Length`（结构大小校验）、`UseDefaultObject`（是否用默认等待对象）、`CaseInsensitive`（名字是否大小写不敏感）、`InvalidAttributes`（禁用的属性位，超范围直接 `STATUS_INVALID_PARAMETER`）、`GenericMapping`（通用权限到具体权限的映射，见下条）、`ValidAccessMask`（该类合法的访问掩码）、`SecurityRequired`（该类对象是否必须有安全描述符，见第六组）、`MaintainHandleCount`（是否维护句柄计数段——为 `TRUE` 时 `OpenProcedure` 与 `CloseProcedure` 至少要给一个，否则 `STATUS_INVALID_PARAMETER`，Design Notes #24 的校验原话）、`MaintainTypeList`（是否把该类对象挂进类型列表，见下条）、`PoolType`（该类对象体分配自哪种池）、`DefaultPagedPoolCharge` / `DefaultNonPagedPoolCharge`（默认配额价）、以及方法指针（见下条"方法表"）。为什么注册时要一次交齐：类型创建后行为即冻结，后面所有该类对象的创建、打开、删除、安全检查都按这张表走；一次交齐避免运行期到处查配置。
- **方法表（method table）**：初始化器里的一组过程指针：`DumpProcedure`（调试转储）、`OpenProcedure`（打开时回调）、`CloseProcedure`（关闭时回调）、`DeleteProcedure`（删除时回调）、`ParseProcedure`（名字解析时回调，见第四组"重解析"）、`SecurityProcedure`（安全检查回调，缺省为 `SeDefaultObjectMethod`，`OB/OBTYPE.C:367` 一带）、`QueryNameProcedure`（查询名字回调）、`OkayToCloseProcedure`（能否关闭回调）。做什么用：它是类型的行为契约，类似 C++ 的虚函数表。为什么对象管理器不直接写死各类型行为：新类型（如驱动注册的设备类型）只需填一张方法表、调一次 `ObCreateObjectType`，不必改对象管理器的代码——开放扩展的支点。Probert《Object Manager》讲逐系统列出了各主要类型的实际方法表，本批的 Type/Directory/SymbolicLink 三张表见正文 ⑤。
- **通用权限映射（GenericMapping）**：`GENERIC_READ` / `GENERIC_WRITE` / `GENERIC_EXECUTE` / `GENERIC_ALL` 四个通用权限到该类具体权限位的映射表。做什么用：让调用方不必知道每类对象的具体权限位，用通用词就能打开对象。为什么每类要自带映射：不同类型的"读"含义不同（文件的读是 `FILE_READ_DATA`，目录的读是 `DIRECTORY_QUERY`），映射必须由类型自己定义，对象管理器无法统一规定。
- **默认对象（DefaultObject）**：类型可等待时用来挂等待的事件。规则（`OB/OBTYPE.C:390` 一带）：`UseDefaultObject` 为真 → `ValidAccessMask` 并上 `SYNCHRONIZE`，且 `DefaultObject = &ObpDefaultObject`（一个初始即置信号态的通知事件——通知事件是一种可等待的内核对象，置信号态时对它的等待立即成功，`OB/OBINIT.C:377`）；类型名恰为 `File` → `DefaultObject` 是 `FILE_OBJECT.Event` 的偏移；恰为 `WaitablePort` → `LPCP_PORT_OBJECT.WaitEvent` 的偏移；否则 `NULL`（不可等待）。做什么用：给"等待某对象"这个操作一个统一的落点。为什么共用一个常置信号态事件：没有自己事件的类型（如 Type）本就没有可等待的状态，等待它恒立即成功；共用事件省得为每类再建事件。为什么目录与符号链接随后又被清掉 `SYNCHRONIZE`（见正文 ⑤）：这两类对象不允许被等待（名字解析的中转站不是同步对象），清掉是把"可等待"收回。
- **类型列表（TypeList）**：每类对象挂在其类型对象 `TypeList` 上的链表（经对象头的创建者段 `CreatorInfo->TypeList`）。做什么用：`ObEnumerateObjectsByType` 按类型枚举全部对象（如调试器列出所有进程、所有事件）。为什么 `MaintainTypeList` 可关：目录选 `FALSE`——目录数量大、枚举需求低，不挂列表省创建者段与链表维护；但类型对象本身的创建者段恒有（`ObCreateObjectType` 传 `NULL` 创建信息时，创建者段与名字段恒分配，`OB/OBCREATE.C:869` 一带），所以"Type"类型的 `TypeList`（即全部类型对象的链表）恒完整，Phase 1 漫步靠的就是它。
- **类型编号（Index）**：第几个注册的类型，从 1 起。`ObCreateObjectType` 里 `Index = ObpTypeObjectType->TotalNumberOfObjects`（`OB/OBTYPE.C:445`），且 `Index < 48` 时记入全局数组 `ObpObjectTypes[Index-1]`（`OB/OBTYPE.C:447-449`；数组容量 `OBP_MAX_DEFINED_OBJECT_TYPES` 为 48，`OB/OBP.H:263-264`）。做什么用：给类型一个紧凑的数字身份，供调试与快速索引。为什么上限 48：2003 年时点的 NT 对象宇宙约 29 种（Probert 讲义 `\ObjectTypes` 清单），48 是留有余量的静态数组；超限的类型仍可正常工作，只是不进快速索引数组。

### 四、目录与解析

- **对象目录（OBJECT_DIRECTORY，`INC/OB.H:326`）**：字段：`HashBuckets[37]`（37 条哈希桶，见下条）、`Lock`（目录推送锁）、`DeviceMap`（设备映射，见下条）、`SessionId`（所属会话，见第七组会话条）。做什么用：装一层名字到对象的映射。为什么目录本身也是对象：只有目录也是对象，它才能进上级目录、被引用计数、被权限检查——命名空间因此是一棵"目录与对象混居"的树，而不是两套结构。
- **目录项（OBJECT_DIRECTORY_ENTRY，`INC/OB.H:337`）**：字段：`ChainLink`（桶内链的下一项）、`Object`（对象体指针）、`HashValue`（该名字的全哈希值，见下条）。做什么用：目录里的一条"名字 → 对象"记录。为什么项里要存 `HashValue`：查找时先比 `HashValue`（整数比较，快），不等直接跳过，不必比字符串；只有哈希相等才比字符串（防哈希碰撞误判）。
- **目录哈希**：名字到桶号的算法（`OB/OBDIR.C:950` 一带）：64 位下按 4 个宽字符（16 位 Unicode 字符）一组做块哈希（大写折叠：`a`–`z` 减 `a-A`，非 ASCII（7 位英文字符集之外）逐字 `RtlUpcaseUnicodeChar`），块间递推 `ChunkHash += (ChunkHash << 1) + (ChunkHash >> 1) + Chunk`；32 位下逐字同递推；收尾把 64 位哈希折成 32 位（`ChunkHash + (ChunkHash >> 32)`）得 `HashValue`，桶号 = `HashValue % 37`（`NUMBER_HASH_BUCKETS`，`INC/OB.H:323`）。做什么用：把名字均匀散到 37 条桶里，查找时只走一条链。为什么 37：素数，取模分布均匀；为什么大写折叠：目录多为大小写不敏感（`CaseInsensitive`），折叠后 `Foo` 与 `FOO` 落同一桶、比字符串时也按不敏感比对。为什么 64 位与 32 位算出的桶号可能不同：块哈希与逐字哈希的递推粒度不同，同一名字在两架构下可能落不同桶——但桶号只影响本机内存布局，不进任何持久格式，跨架构不一致无害。
- **查找上下文（OBP_LOOKUP_CONTEXT）**：一次目录操作（查找或插入）携带的上下文：目录锁状态、算好的 `HashIndex` 与 `HashValue`、桶内插入点。做什么用：把"查找"与"插入"两步合成一次：先查（无则记下插入点），再插（直接用记下的点，不必重算哈希、不必重拿锁）。为什么需要它：`ObCreateObjectType` 的查重（`OB/OBTYPE.C:200` 一带：目录已存在时先查有无重名，有则 `STATUS_OBJECT_NAME_COLLISION`）与 Phase 1 的漫步插入（见正文 ⑩）都是"查后插"；没有上下文，每次插入都要重新哈希、重新走链、重新拿锁。
- **符号链接（OBJECT_SYMBOLIC_LINK，`INC/OB.H:348`）**：字段：`CreationTime`（创建时间）、`LinkTarget`（目标名字串）、`LinkTargetRemaining`（解析剩余段）、`LinkTargetObject`（已解析的目标对象缓存）、`DosDeviceDriveIndex`（盘符在 `KUSER_SHARED_DATA` 盘符类型表中的 1-based 索引）。做什么用：名字到另一名字的别名（如 `\DosDevices\C:` → `\Device\HarddiskVolume1`）。为什么链接对象要有 `Parse` 方法：解析走到链接时不能停，必须跟随到目标；`Parse` 方法（`ObpParseSymbolicLink`，注册于 `OB/OBINIT.C:476`）就是跟随逻辑。为什么要有 `DosDeviceDriveIndex`：盘符链接还要回答"这个盘是什么类型"（硬盘、光盘、网络），索引让调用方不查目标设备也能取盘符类型。
- **重解析（REPARSE）**：`Parse` 方法的一种返回值（`STATUS_REPARSE`）：告诉调用方"把名字换成我给的新名字，从头再解析一遍"。符号链接的语义（Design Notes #24 §3.4）：剩余名非空 → 目标名 + 分隔符 + 剩余名拼成新全名，返回重解析；剩余名为空 → 调用方本来就是在打开链接本身，返回链接对象。做什么用：让链接、设备名解析这类"中途改道"有统一协议。为什么不直接在 `Parse` 里递归解析到底：递归深度不可控（链接套链接），且调用方持有目录锁时不能任意递归；返回重解析后调用方先放锁、换名、从头再走，锁序与深度都可控。
- **设备映射（DeviceMap，`INC/OB.H:355` 一带）**：字段：`DosDevicesDirectory`（本进程的 `\??` 目录）、`GlobalDosDevicesDirectory`（全局 `\??` 目录）、`ReferenceCount`（引用计数）、`DriveMap`（盘符位图）。做什么用：每个进程解析 `\??\` 开头的名字时，先查自己的设备映射。为什么每个进程一份：不同登录会话（见第七组会话条）的盘符与设备别名必须隔离（A 会话的 `C:` 与 B 会话的 `C:` 可以指向不同设备）；全局 `\??` 是共享的底，进程自己的映射是会话内的覆盖。`LUID`（登录会话的本地唯一标识）设备映射开关（`ObpLUIDDeviceMapsEnabled`，由 `ObpProtectionMode` 与 `ObpLUIDDeviceMapsDisabled` 决定，`OB/OBINIT.C:1731` 一带）控制是否按登录会话隔离设备映射。
- **DosDevices（`\??` 目录）**：Win32 设备名与盘符的根目录。做什么用：`C:`、`COM1`、`PIPE` 这类 Win32 名字经 `\??\` 解析到内核设备对象。为什么 Win32 名字要经一层目录：Win32 的名字空间（盘符、设备名）与内核命名空间（`\Device\…`）是两套词汇，`\??` 目录就是翻译层；盘符链接挂在这里，改链接即改盘符指向，不必动设备对象。
- **命名空间池类型（OB_NAMESPACE_POOL_TYPE）**：目录与符号链接对象本体分配自哪种池。本构建定义了 `OBP_PAGEDPOOL_NAMESPACE`（`OB/OBP.H:47`），故 `OB_NAMESPACE_POOL_TYPE` 为 `PagedPool`（`OB/OBP.H:117`）。做什么用：决定名字空间对象的内存属性。为什么命名空间可以进分页池：名字解析只在 `APC_LEVEL` 及以下发生（见正文 ⑬ 的 IRQL 约束），分页池在该级别可安全访问；进分页池省非分页池（宝贵的常驻内存）。注意不对称：类型对象本体恒在非分页池——因为类型对象的类型是 `Type`，而 `Type` 的 `PoolType` 是 `NonPagedPool`（见正文 ⑤），`ObpAllocateObject` 按对象类型的池类型分配（`OB/OBCREATE.C:960` 一带），与初始化器里的池类型无关。

### 五、句柄与句柄表

- **句柄（HANDLE）**：用户态与内核其他部分引用对象的凭证，本质是句柄表项的索引 × 4（`HANDLE_VALUE_INC` 为 4，`INC/EX.H:4993`）。句柄值结构（`EXHANDLE`，`INC/EX.H:4971`）：低 2 位是 `TagBits`（应用可用标签位，内核句柄恒为 0），高 30 位是表项索引。做什么用：代替指针跨边界传递对象引用。为什么句柄不是指针：指针可伪造、可越界、可直读对象体；句柄只是索引，解引用必经句柄表（查表项、验类型、验权限），伪造的句柄值在表里查无此项即被拒绝。为什么句柄 0 永不发放：0 与 `NULL` 混淆，调用方会把"没有句柄"误当"句柄 0"；首个空闲句柄从 4 起（`FirstFree = HANDLE_VALUE_INC`，`EX/HANDLE.C:2663`）。
- **句柄表（HANDLE_TABLE，`INC/EX.H:5179`）**：每个进程一张（内核共用一张，见下条"内核句柄表"）。字段：`TableCode`（顶层节点指针 + 低 2 位级别，见正文 ⑭）、`QuotaProcess`（收配额的进程）、`UniqueProcessId`（表主的进程号）、`HandleTableLock[4]`（4 把扩表推送锁，条带化）、`HandleTableList`（全局句柄表链表，供快照与调试枚举）、`HandleContentionEvent`（表项锁竞争时的等待事件）、`DebugInfo`（句柄追踪调试信息，按需分配）、`FirstFree`（首个空闲句柄值）、`NextHandleNeedingPool`（下个需要分配池的句柄值，即扩表阈值）。做什么用：进程持有的全部句柄的登记簿。为什么每进程一张：句柄只在所属进程内有效，换进程句柄值即失效——这是进程隔离在对象引用层面的实现；内核句柄是例外（全局有效，见下条）。
- **句柄表项（HANDLE_TABLE_ENTRY，`INC/EX.H:5047`）**：两个字段。第一字段是对象体指针的复用：低 3 位借给 OB 作属性位（继承、保护、审计，见第一组"对象属性"），高位借给表项锁（项被锁定时置位）；有效对象指针恒长字对齐且符号位恒置位（内核地址），故低 3 位与高位可安全复用。第二字段复用：占用时是授予访问掩码（见下条；i386 调试时是 `GrantedAccessIndex` + `CreatorBackTraceIndex`），空闲时是 `NextFreeTableEntry`（下一空闲项的索引）。做什么用：句柄值到对象 + 权限的映射。为什么空闲链用索引而不用指针：表复制（如子进程继承整表）时索引链可原样 `memcpy`，指针链则每项都要重算——FAT 式索引链让整表复制变成内存复制（见正文 ⑮）。
- **授予访问掩码（GrantedAccess）**：打开对象时安全子系统算出的实际权限，存在句柄表项里，不存在句柄值里。做什么用：后续每次用句柄访问对象，凭表项里的掩码放行，不再重算安全描述符（见第六组）。为什么权限要存表项而不是每次重算：安全描述符的解析（走 DACL 的 ACE 链）贵，且对象的安全描述符可能被改；打开时一次算清、存表项，后续访问是常数时间的掩码位检查；且"打开时有权限、后来被收回"的语义正是"权限在打开时刻冻结"（句柄是打开时刻的权限快照）。
- **内核句柄表（ObpKernelHandleTable）**：全局有效的句柄表，与系统进程的 `ObjectTable` 是同一指针（`OB/OBINIT.C:398`：`ObpKernelHandleTable = PsGetCurrentProcess()->ObjectTable = ExCreateHandleTable( NULL )`）。做什么用：内核内部持有的句柄（如驱动打开的设备、系统线程打开的对象）登记在这里。为什么与系统进程共用一表：内核句柄不属于任何用户进程，单独建一张表要多一套配额与生命周期；借系统进程（Phase 0 时是 Idle 占位进程，第 1 批）的 `ObjectTable` 槽位装它，内核句柄天然全局有效。为什么创建时传 `NULL` 进程：`ExCreateHandleTable(NULL)` 不收配额（`EX/HANDLE.C:2536` 一带：`Process` 非空才 `PsChargeProcessPagedPoolQuota`）——内核表不向任何进程收费。
- **句柄继承**：子进程创建时复制父进程句柄表的机制（`ExDupHandleTable` + `ObDupHandleProcedure`，`OB/OBINIT.C:920` 一带）。规则：只有 `OBJ_INHERIT` 置位的项才复制；复制时先给对象指针计数 +1（防复制中途对象被删），再解锁旧项，然后新项的句柄计数 +1。做什么用：让子进程天然持有父进程打开的对象（如标准输入输出、管道）。为什么复制时要先加指针计数再解锁：旧表项的锁放开后，另一线程可能正好关闭该句柄、把对象删掉；先加指针计数，对象在复制完成前不会释放——"先引用再解锁"与目录查找的顺序是同一条防释放后使用（UAF，Use-After-Free）纪律（Probert 讲义目录查找伪代码的注释原话）。
- **句柄严格 FIFO（StrictFIFO）**：调试版（`DBG`）下对句柄复用的约束：`ExSetHandleTableStrictFIFO`（`OB/OBINIT.C:398` 一带）让空闲句柄按严格先进先出复用。做什么用：延迟句柄值重用。为什么：句柄值是索引 × 4，表项复用后旧句柄值可能指向新对象（ABA 问题：调用方拿着已关闭的旧句柄值，误当新对象的句柄）；严格 FIFO 让旧值尽可能晚才重用，调试时 ABA 更容易暴露。零售版不强制（复用越快，表越紧凑）。
- **句柄表锁**：两级。项级：表项指针高位即锁（无额外内存，`ExLockHandleTableEntry` 置位、`ExUnlockHandleTableEntry` 清位；竞争时在 `HandleContentionEvent` 上等）。表级：`HandleTableLock[4]` 四把推送锁，只用于扩表（`ExpAllocateHandleTableEntrySlow` 的调用方必须已持表锁）。为什么分两级：项操作（打开、关闭、引用）极高频，用指针位锁零额外内存、零数组索引；扩表低频但要改 `TableCode` 与整层结构，必须用真正的锁串行化。四把条带化是按句柄值哈希选锁，降竞争。

### 六、安全描述符与缓存

- **安全标识符（SID）**：用户、组、登录会话的变长唯一标识。做什么用：访问控制里"谁"的那一方。为什么变长：SID 由颁发机构 + 域 + 相对标识组成，长度随机构层级变；定长装不下。本批出现的三个众所周知 SID：`SeWorldSid`（所有人）、`SeAliasAdminsSid`（管理员组）、`SeLocalSystemSid`（本地系统）。
- **访问控制项（ACE）**：ACL 里的一条，含：允许还是拒绝、针对哪个 SID、哪组权限、继承标志（`CONTAINER_INHERIT_ACE` 容器继承、`OBJECT_INHERIT_ACE` 对象继承、`INHERIT_ONLY_ACE` 仅继承不作用于自身）。做什么用：访问规则的最小单位。为什么 ACE 要带继承标志：目录的规则要能自动流向子对象（容器继承）与子目录（对象继承），且"仅继承"让父目录自身的规则不作用于自身、只向下传——没有继承标志，每建一层都要重写规则。
- **访问控制列表（ACL）**：ACE 的链表。分两种：`DACL`（自主访问控制列表，管允许与拒绝）与 `SACL`（系统访问控制列表，见下条）。做什么用：把一堆 ACE 打包成"这对象的访问规则"。为什么列表而不是单条：一个对象常要对多方（所有人、管理员、系统、创建者）各订规则，列表是自然的容器。
- **系统访问控制列表（SACL）**：ACL 的一种，装审计 ACE（成功审计、失败审计 + 继承标志）。做什么用：规定"谁的哪种访问要记审计日志"。为什么审计要单列：审计不是放行规则（不影响允许与否），是事后追踪规则；与 DACL 分开，检查路径可以"先算 DACL 放行与否，再看 SACL 是否记账"，互不干扰。Phase 1 的审计 SD（见正文 ⑧）就是在公共默认 SD 上加一条"所有人、全部权限、成功与失败都审计"的 SACL。
- **安全描述符（SECURITY_DESCRIPTOR）**：对象的完整安全元数据：所有者 SID、组 SID、DACL、SACL、控制位。对象头的 `SecurityDescriptor` 指针指向的不是调用方传来的那份，而是缓存副本（见下条）。做什么用：访问检查的唯一依据（Se 按 SD + 调用方令牌算出授予访问掩码）。为什么 SD 不直接存调用方传来的指针：调用方的 SD 在用户池、可能被改、生命周期不受控；对象管理器必须持有自己的一份——而缓存让"自己的一份"不必每对象复制。
- **安全描述符缓存（ObsSecurityDescriptorCache）**：全局 257 条链的数组（`OB/OBSDATA.C:102`；`SECURITY_DESCRIPTOR_CACHE_ENTRIES` 为 257，`OB/OBP.H:382`），每链一把推送锁 + 一条链表头。入缓存入口 `ObLogSecurityDescriptor`（`OB/OBSDATA.C:290` 一带）：算哈希（`ObpHashSecurityDescriptor`，见下条）、槽位 = 哈希 % 257；先以共享锁走链比对（`ObpCompareSecurityDescriptors` 全内容比对），命中则引用计数 +1 并释放传入的 SD（丢弃重复副本）；未命中则升级独占锁建新项（`ObpCreateCacheEntry`）插链。取用入口 `ObpReferenceSecurityDescriptor`（`OB/OBSDATA.C:564`）：按对象头的 SD 指针取缓存项、引用计数 +1；用完 `ObDereferenceSecurityDescriptor` 归还，归零时 `ObpDestroySecurityDescriptorHeader` 摘链释放。做什么用：把内容相同的 SD 去重成一份共享副本。为什么要去重：大量对象共享同一 SD（默认 SD、继承来的父目录 SD），每对象复制一份是池内存的成倍浪费；缓存后对象头只存一个指针。为什么 257：素数，哈希取模分布均匀，减少链长。为什么每链一把锁：257 条链分片，竞争散开；共享锁查、独占锁插，读多写少的典型读写锁用法。为什么缓存的 SD 视为不可变：同一副本被多个对象共享，原地改一处等于改所有共享者的规则；设计上 Se 总是算出新 SD 再入缓存，从不原地改缓存副本。
- **安全描述符哈希（ObpHashSecurityDescriptor）**：`ObpHashBuffer`（`OB/OBSDATA.C:210` 一带）：按 `ULONG` 逐字异或、每步左旋 3 位（`_rotl(Result, 3)`），尾字节同法补入。做什么用：给 SD 内容算 32 位指纹，定槽位。为什么异或 + 旋 3：异或让每字节都影响结果，旋 3 让高位字节的贡献不被低位淹没（纯异或对字节顺序不敏感，旋位引入位置敏感性）；简单、快、无表，适合每次创建对象都算一次的路径。
- **引用偏置（RefBias）**：`ObLogSecurityDescriptor` 的引用计数偏置参数，通常为 1，或 `ExFastRefGetAdditionalReferenceCount() + 1`。做什么用：一次入缓存同时给调用方留好引用。为什么需要偏置：快速引用（`EX_FAST_REF`）的计数不在对象头里单独立字段，而是偷在指针低位；偏置让"入缓存 + 交引用"一次完成，不必先入缓存再单独加一次引用（两次操作之间有竞态窗口）。

### 七、通用小词

- **池标签（pool tag）**：池分配时附带的 4 字节倒序字符（如类型对象的 `'TjbO'`（"ObjT"倒序）、创建信息 `'iCbO'`、名字缓冲 `'mNbO'`、句柄表 `'btbO'`）。做什么用：池泄漏与池统计按标签归因。为什么 4 字节倒序：池头只留 4 字节给标签，倒序是 x86 小端下字符在内存里的自然顺序；调试器按标签能直接定位是哪个子系统漏了池。`POOL_TAGGING` 编译开关控制类型对象的 `Key` 字段是否启用（`OB/OBTYPE.C:289` 一带）。
- **全局标志（NtGlobalFlag）**：内核全局调试开关位图。本批相关的两位：`FLG_KERNEL_STACK_TRACE_DB`（开启内核栈追踪数据库——记录句柄打开/关闭调用栈的调试数据库：i386 下启用缓存授予访问数组 `ObpCachedGrantedAccesses`，`OB/OBINIT.C:490` 一带；且句柄表项的授予访问字段改存索引 + 创建者回溯索引）；`FLG_MAINTAIN_OBJECT_TYPELIST`（强制所有类型维护类型列表，`OB/OBTYPE.C:355` 一带：置位时 `MaintainTypeList` 恒为真，目录的 `FALSE` 也被覆盖）。
- **审计（auditing）**：按 SACL 记录"谁在何时以何种成败访问了何对象"。本批的审计点：Phase 1 根目录的审计 SD（`ObpAuditBaseDirectories` / `ObpAuditBaseObjects` 非零时才建，`OB/OBINIT.C:625` 一带）。为什么审计是可选项：审计日志贵（每次访问都可能记一条），默认关闭，只在打开基础审计时才给根目录挂 SACL；SACL 会被子对象继承（容器继承 + 对象继承），故只需在根上挂一次。
- **会话（session）**：一次用户登录的隔离边界；设备映射按会话隔离（见第四组“设备映射”）；对象目录的 `SessionId` 标识其所属会话。
- **登录会话本地唯一标识（LUID）**：每次登录会话的 64 位本地唯一标识。做什么用：区分不同登录会话的设备映射（见第四组"设备映射"）。为什么不用用户名：同一用户可多次登录，每次会话的设备映射必须独立；LUID 按会话颁发，天然区分"同一用户的两次登录"。
- **同步访问权（SYNCHRONIZE）**：对可等待对象做等待操作（`NtWaitForSingleObject` 一族）的权限位。为什么目录与符号链接被清掉它（`OB/OBINIT.C:460`、`:483`）：这两类对象是名字解析的中转站，不是同步对象；允许等待它们会让等待语义落到"目录"这种没有状态变化的东西上，且等待路径会绕过名字解析的锁序——清掉是把"可等待"从这两类上收回。

## 第二部分：正文

### Phase 0：把对象管理器立起来（`OB/OBINIT.C:224` 起，`InitializationPhase == 0` 分支）

#### ① 前瞻列表与系统规模：先按机器大小定深度

`ObInitSystem` 先问 `MmQuerySystemSize()`（`OB/OBINIT.C:289`）：大系统且为 NTAS（服务器版）时创建信息深度 64、名字缓冲深度 32；大系统非 NTAS 时 32 与 16；小系统两者皆 3（`OB/OBINIT.C:295-306`）。随后 `ExInitializeSystemLookasideList` 建两条系统级前瞻列表：创建信息（非分页池，标签 `'iCbO'`，`OB/OBINIT.C:320`）、名字缓冲（分页池——本构建定义了 `OBP_PAGEDPOOL_NAMESPACE`，故走 `PagedPool` 分支，标签 `'mNbO'`，`OB/OBINIT.C:331`）。当前处理器的两条前瞻指针（`Prcb->PPLookasideList[LookasideCreateInfoList]` 与 `[LookasideNameBufferList]`）的 L（系统级）与 P（处理器级）暂时都指向同一条系统列表（`OB/OBINIT.C:360` 一带；注释原话："Temporarily during the initialization of the system both lookaside list pointers in the processor block point to the same lookaside list structure"）。

为什么深度要按系统规模分三档：前瞻列表是预分配的常驻内存，大系统对象创建并发高、深度浅了会频繁回落到通用池（丢了前瞻的意义），小系统内存紧、深度深了是常驻浪费；三档是"并发需求"与"常驻成本"的折中。为什么 L 与 P 暂时同指一条：Phase 0 只有 P0 在跑，每处理器列表还没法建（建它本身要分配，而分配路径要前瞻列表——先有鸡还是先有蛋）；先让 L 与 P 同指系统列表跑起来，Phase 1 再给每处理器另建 P 列表（见 ⑦）。没有这一步会怎样：对象创建路径一开始就无前瞻可用，每次创建都要走通用池锁，Phase 0 的连续创建（类型、目录、句柄表）会慢且碎片化。

#### ② 安全描述符缓存、默认对象、锁、配额：先把地基的四样立好

`ObpRemoveObjectList = NULL`（`OB/OBINIT.C:369`，延迟删除队列空）。`ObpInitSecurityDescriptorCache()`（`OB/OBINIT.C:375`）初始化 257 条链的推送锁与链表头（`OB/OBSDATA.C:133`，只初始化锁与头，不分配——数组是静态全局量，`OB/OBSDATA.C:102`）。`KeInitializeEvent(&ObpDefaultObject, NotificationEvent, TRUE)`（`OB/OBINIT.C:377`，常置信号态的通知事件，见术语表"默认对象"）。`ExInitializePushLock(&ObpLock)`（`OB/OBINIT.C:378`，对象管理器全局锁）。当前进程与线程的 `GrantedAccess` 置全权（`OB/OBINIT.C:379-380`：进程 `PROCESS_ALL_ACCESS`、线程 `THREAD_ALL_ACCESS`——Phase 0 的 current 进程是 Idle 占位进程，它的操作不受访问检查掣肘）。设备映射锁按构建初始化：本构建定义了 `OBP_PAGEDPOOL_NAMESPACE`，故为 `KeInitializeGuardedMutex(&ObpDeviceMapLock)`（`OB/OBINIT.C:383` 一带；若不定义该宏则为自旋锁——命名空间在分页池时，设备映射操作只能在 `APC_LEVEL` 及以下，而自旋锁要求 `DISPATCH_LEVEL`，互斥体才是正确选择；反之命名空间在非分页池时可用自旋锁）。`PsInitializeQuotaSystem()`（`OB/OBINIT.C:391`）初始化配额系统。

为什么这四样要在类型注册之前：SD 缓存是对象创建时 SD 入缓存的前提（没有缓存，第一个对象的 SD 无处可去）；默认对象是类型注册时 `UseDefaultObject` 的落点（没有它，Type 类型的 `DefaultObject` 悬空）；全局锁是类型列表操作的串行化前提；配额系统是对象分配收费的前提。顺序错了会怎样：类型注册时 `DefaultObject` 指向未初始化的事件，等待语义错乱；对象创建时 SD 缓存未初始化，`ObLogSecurityDescriptor` 走未初始化的链锁，直接崩溃或静默 corrupt。

#### ③ 内核句柄表：系统进程的表就是内核表

`ObpKernelHandleTable = PsGetCurrentProcess()->ObjectTable = ExCreateHandleTable( NULL )`（`OB/OBINIT.C:398`）。`ExCreateHandleTable`（`EX/HANDLE.C:830`）经 `ExpAllocateHandleTable(Process, TRUE)`（`EX/HANDLE.C:2536`）分配句柄表描述符（分页池，标签 `'btbO'`），`Process` 非空才收配额（本调用传 `NULL`，故不收费），随后把新表挂进全局句柄表链表（`HandleTableListHead`，`EX/HANDLE.C:877` 一带，供快照与调试枚举）。调试版下 `ExSetHandleTableStrictFIFO(ObpKernelHandleTable)`（`OB/OBINIT.C:398` 一带的 `#if DBG` 块）。

为什么内核句柄表要在 Ob Phase 0 建：Phase 1 的根目录创建（见 ⑧）本身就要句柄（`NtCreateDirectoryObject` 返回 `RootDirectoryHandle`，再 `ObReferenceObjectByHandle` 取对象指针、最后 `NtClose` 关句柄）——没有句柄表，连"创建第一个目录"这个动作都做不了。为什么与系统进程共用一表而不是单独立表：内核句柄全局有效，生命周期与任何用户进程无关；单独立表要多一套配额、销毁与枚举逻辑，而系统进程的表槽位现成（Phase 0 的 current 进程就是 Idle 占位进程，其 `ObjectTable` 槽位正好空着），借槽位是零成本的正确选择。没有会怎样：若内核句柄与用户进程表混用，进程退出销毁表时会把内核句柄一并销毁，系统服务持有的内核句柄成悬空。

#### ④ 鸡生蛋一：Type 类型自举——第一个类型没有类型

`ObCreateObjectType` 的头注释（`OB/OBTYPE.C:87`）与 `ObInitSystem` 的头注释（`OB/OBINIT.C:224` 一带）共同点题："The object manager data structures are self describing with the exception of the root directory, the type object type and the directory object type. The initialization code then constructs these objects by hand to get the ball rolling."——对象管理器的数据结构是自描述的，唯独根目录、Type 类型、Directory 类型要手工捏出来滚雪球。

结为什么会形成：创建任何对象都要指定它的类型（`ObpAllocateObject` 的 `ObjectType` 参数决定池类型、配额价、是否挂类型列表）；创建"Type"类型对象时，它的"类型"应该是"Type"本身——而"Type"还不存在。循环依赖：要 Type 先有 Type。

靠什么打破循环：分三步。第一步，`ObInitSystem` 调 `ObCreateObjectType(&TypeTypeName, &ObjectTypeInitializer, NULL, &ObpTypeObjectType)`（`OB/OBINIT.C:431`）时，全局量 `ObpTypeObjectType` 还是 `NULL`，`ObCreateObjectType` 照常把 `NULL` 传给 `ObpAllocateObject`（`OB/OBTYPE.C:243`）。第二步，`ObpAllocateObject` 的 `ObjectType` 参数标为 `OPTIONAL`，`NULL` 时走特殊路径（`OB/OBCREATE.C:869` 一带：`ObjectCreateInfo` 为 `NULL` 时只分配名字段与创建者段，不分配配额段与句柄段；`OB/OBCREATE.C:960` 一带：`ObjectType == NULL` 时池类型恒为 `NonPagedPool`，池标签恒为 `'TjbO'`）。第三步，分配回来后，`ObCreateObjectType` 检测到 `ObpTypeObjectType` 仍为 `NULL`，判定"这就是 Type 类型本身"，做自指赋值：`ObpTypeObjectType = NewObjectType; NewObjectTypeHeader->Type = ObpTypeObjectType; NewObjectType->TotalNumberOfObjects = 1;`（`OB/OBTYPE.C:289` 一带）——Type 对象的头里类型指针指向它自己，对象计数置 1（它自己就是第一个 Type 对象）。

打破后是什么局面：`ObpTypeObjectType` 非空，此后一切 `ObCreateObjectType` 调用都以 Type 为类型正常走（`ObpAllocateObject` 按 Type 的 `NonPagedPool` 分配类型对象本体）；Type 的 `TypeList`（全部类型对象的链表）从此开始累积（见 ⑩ 的漫步）；Type 的 `Index` 为 1，`ObpObjectTypes[0]` 就是 Type。若自指赋值漏了会怎样：Type 对象的头类型指针悬空（`NULL`），后续任何"取对象类型"的操作（访问检查、删除、枚举）都解引用空指针；且 `TotalNumberOfObjects` 不计数，`Index` 分配错位，`\ObjectTypes` 的漫步会漏 Type。

#### ⑤ 三类注册：Type、Directory、SymbolicLink 的初始化器差异

三类共用一套 `ObjectTypeInitializer` 骨架（`OB/OBINIT.C:425` 一带：`Length`、`InvalidAttributes = OBJ_OPENLINK`、`PoolType` 初值 `NonPagedPool`），逐类改差异字段：

- **Type**（`OB/OBINIT.C:431`）：名字 `L"Type"`，`ValidAccessMask = OBJECT_TYPE_ALL_ACCESS`，`GenericMapping = ObpTypeMapping`，`DefaultNonPagedPoolCharge = sizeof(OBJECT_TYPE)`，`MaintainTypeList = TRUE`，`UseDefaultObject = TRUE`，`DeleteProcedure = ObpDeleteObjectType`。`MaintainTypeList` 为真：Type 对象要挂进 Type 类型的列表（枚举全部类型靠它）。`DeleteProcedure` 删类型对象时只清 `ObjectLocks` 与 `Mutex`（`OB/OBTYPE.C:499` 一带）——类型对象没有体外资源。
- **Directory**（`OB/OBINIT.C:450`）：`PoolType` 改为 `OB_NAMESPACE_POOL_TYPE`（即 `PagedPool`，见术语表"命名空间池类型"），`DefaultNonPagedPoolCharge = sizeof(OBJECT_DIRECTORY)`。注意收费的不对称：初始化器只写了 `DefaultNonPagedPoolCharge`（体大小），`DefaultPagedPoolCharge` 未赋值（`RtlZeroMemory` 清零后保持 0）；`ObCreateObjectType` 按 `PoolType` 把标准头收费加到对应池的默认值上（`OB/OBTYPE.C:340` 一带：`PoolType` 为 `NonPagedPool` 时加到 `DefaultNonPagedPoolCharge`，否则加到 `DefaultPagedPoolCharge`）；创建时未指定收费即取类型默认值（`OB/OBCREATE.C:188-197`）——于是目录对象的收费是分页池记头收费、非分页池记体大小（SymbolicLink 同理）。，`CaseInsensitive = TRUE`，`GenericMapping = ObpDirectoryMapping`，`UseDefaultObject = TRUE`，`MaintainTypeList = FALSE`，`DeleteProcedure = NULL`。随后 `ObpDirectoryObjectType->TypeInfo.ValidAccessMask &= ~SYNCHRONIZE`（`OB/OBINIT.C:460`）清掉同步权。`MaintainTypeList` 为假：目录不挂类型列表（目录量大、枚举需求低，省创建者段与链表维护——但类型对象本身的创建者段恒有，见术语表"类型列表"）。`DeleteProcedure` 为空：目录对象删除时无体外资源要清（目录项的释放走目录删除路径本身）。
- **SymbolicLink**（`OB/OBINIT.C:473`）：`DefaultNonPagedPoolCharge = sizeof(OBJECT_SYMBOLIC_LINK)`，`ValidAccessMask = SYMBOLIC_LINK_ALL_ACCESS`，`CaseInsensitive = TRUE`，`GenericMapping = ObpSymbolicLinkMapping`，`DeleteProcedure = ObpDeleteSymbolicLink`，`ParseProcedure = ObpParseSymbolicLink`；随后同样清掉 `SYNCHRONIZE`（`OB/OBINIT.C:483`）。`ParseProcedure` 是符号链接的灵魂（跟随逻辑，见术语表"重解析"）；`DeleteProcedure` 清链接目标串等体外资源。

为什么三类要按 Type → Directory → SymbolicLink 的顺序：Type 必须第一（自举，见 ④）；Directory 第二（它是目录的类，后面建根目录、`\ObjectTypes`、`\DosDevices` 都要它）；SymbolicLink 第三（它的 `Parse` 要在目录机制存在后才有意义，且链接对象常挂在目录里）。顺序反了会怎样：先建 Directory 而无 Type，`ObpAllocateObject` 的类型参数悬空，分配路径无池类型可依；先建 SymbolicLink 而无 Directory，链接对象无处可挂（目录不存在时 `ObpTypeDirectoryObject` 为空，类型插入跳过——见 ⑥ 的目录插入规则，链接类型本身虽能注册，但链接对象无法进目录，等于不可用）。

`ObCreateObjectType` 的通用收尾（`OB/OBTYPE.C:367` 一带）：`SecurityProcedure` 为空时默认 `SeDefaultObjectMethod`；初始化 `Mutex` 与 4 把 `ObjectLocks`（`ERESOURCE`）；初始化 `TypeList`；`UseDefaultObject` 时并上 `SYNCHRONIZE` 并置 `DefaultObject`（File 与 WaitablePort 的偏移特判见术语表"默认对象"）；上类型互斥锁，把新类型的创建者段挂进 `ObpTypeObjectType->TypeList`（`OB/OBTYPE.C:436`）；编号 `Index` 并记入 `ObpObjectTypes`（`OB/OBTYPE.C:445-449`）；最后是目录插入规则（`OB/OBTYPE.C:470` 一带）：`ObpTypeDirectoryObject` 为空（Phase 0 尚无 `\ObjectTypes`）则直接成功返回；否则尝试 `ObpInsertDirectoryEntry`，成功才返回，失败返回 `STATUS_INSUFFICIENT_RESOURCES`。Phase 0 的三类走的都是"目录为空、直接成功"支——它们的名字当时还不在任何目录里，等 Phase 1 漫步补插（见 ⑩）。

#### ⑥ 第四类 Token：Se 在 Phase 0 紧跟注册

`SeInitSystem` 按 `InitializationPhase` 分派（`SE/SEINIT.C:63`：Phase 0 → `SepInitializationPhase0`，Phase 1 → `SepInitializationPhase1`）；Phase 0 分支里 `SepTokenInitialization()`（`SE/SEINIT.C:144`）在 `SE/TOKEN.C:1864` 注册 `L"Token"`（`SE/TOKEN.C:1901` 一带）：`InvalidAttributes = OBJ_OPENLINK`，`GenericMapping = SepTokenMapping`，`SecurityRequired = TRUE`，`UseDefaultObject = TRUE`，`PoolType = PagedPool`，`ValidAccessMask = TOKEN_ALL_ACCESS`。

为什么 Token 是第四类、且必须在 Phase 0：Phase 0 结束时系统里的类型恰为四类（OB 三类 + Se 一类），与路线"前四类"的说法一致（OB 一家只注册三类，第四类是 Se 的 Token——路线"前四类"若理解为"ObInitSystem 注册四类"就错了，正确理解是"Phase 0 结束时已有的四类"）。`SecurityRequired = TRUE`：令牌对象必须有安全描述符（没有 SD 的令牌无从做访问裁决，等于没锁的门）。`PoolType = PagedPool`：令牌对象可换出（令牌只在 `PASSIVE_LEVEL` 访问，与命名空间同理）。为什么 Token 要在 Phase 0 而不在 Phase 1：`PsInitSystem(0)`（`:835`）紧接着要创建系统进程与线程，每个进程与线程都要挂令牌（`EPROCESS.Token`）；Token 类型若 Phase 1 才有，Phase 0 的进程创建无类型可用。顺序的另一面：Token 注册在 Ob Phase 0（`:827`）之后（Se Phase 0 在 `:831`），因为注册本身要 Ob 的 `ObCreateObjectType` 与 Type 类型先存在。

### Phase 1：把命名空间立起来（`OB/OBINIT.C:527` 起，`InitializationPhase == 1` 分支）

#### ⑦ 每处理器前瞻列表：L 与 P 终于分开

`for (Index = 0; Index < KeNumberProcessors; Index++)`（`OB/OBINIT.C:533` 一带）：每处理器的 `Prcb->PPLookasideList[LookasideCreateInfoList].L` 仍指向系统列表，但 P（处理器级）改指新分配的每处理器列表（`ExAllocatePoolWithTag(NonPagedPool, sizeof(GENERAL_LOOKASIDE), 'ICbO')` 建创建信息列表；名字缓冲同理，标签 `'MNbO'`，分配失败则 P 回落到系统列表）。Phase 0 的"暂时同指"到此结束。

为什么 L 与 P 要分开：P（每处理器）是无锁快路径——本处理器创建对象时先从自己的 P 列表取，不碰全局锁；P 空了才回落到 L（系统级，带锁）。多处理器下各 CPU 的创建路径互不干扰，竞争只发生在回落时。为什么 Phase 0 不能一步到位：建 P 列表本身要池分配，而 Phase 0 的分配路径尚未完全就绪（前瞻列表就是分配路径的一部分）；先同指系统列表跑起来，等 Phase 1（调度器已跑、内存已稳）再分。单处理器下这步只跑 P0 一次，L 与 P 分开后仍是同一处理器的两条列表（P 快、L 兜底），逻辑不变。

#### ⑧ 根目录 `\`：审计 SD 与公共默认 SD 的选择

`EffectiveSd` 初值为 `SePublicDefaultUnrestrictedSd`（Se 的公共默认无限制安全描述符，Se Phase 0 的产物）。若 `ObpAuditBaseDirectories` 或 `ObpAuditBaseObjects` 非零（打开了基础审计），则现场造一份审计 SD（`OB/OBINIT.C:625` 一带）：复制公共默认的 DACL，另建 SACL——一条审计 ACE，`GENERIC_ALL`（全部通用权限）、`SeWorldSid`（所有人）、成功与失败都审计；按开关给 ACE 加继承标志（目录审计加 `CONTAINER_INHERIT_ACE | INHERIT_ONLY_ACE`，对象审计再加 `OBJECT_INHERIT_ACE`）。随后 `RtlInitUnicodeString(&RootDirectoryName, L"\\")`（`OB/OBINIT.C:676`），`InitializeObjectAttributes`（`OBJ_CASE_INSENSITIVE | OBJ_PERMANENT`，父目录 `NULL`，SD 为 `EffectiveSd`），`NtCreateDirectoryObject(&RootDirectoryHandle, DIRECTORY_ALL_ACCESS, &ObjectAttributes)` 建根目录；`ObReferenceObjectByHandle(RootDirectoryHandle, 0, ObpDirectoryObjectType, KernelMode, &ObpRootDirectoryObject, NULL)`（`OB/OBINIT.C:694`）取对象指针；`NtClose(RootDirectoryHandle)` 关句柄。

为什么根目录要 `OBJ_PERMANENT`：根是整棵树的锚，句柄归零也不能删（见术语表"永久对象"）。为什么先引用后关句柄：`NtClose` 只减句柄计数；若不先 `ObReferenceObjectByHandle` 加指针引用，关句柄后句柄计数归零、根目录可能被当场删除（非永久对象且有名字时句柄归零即尝试删名删对象——根虽永久不删，但"先引用再关闭"是通用正确顺序，Probert 讲义目录查找的"先引用再解锁"同理）。为什么审计 SD 只在根上挂一次：SACL 的继承标志让子对象自动继承审计规则（容器继承进子目录、对象继承进子对象），根上挂一次即覆盖全树；每对象单独挂是重复与不一致的来源。没有审计 SD 会怎样：默认公共 SD 无 SACL，基础审计开着也记不到根目录树的访问——审计出现"根下对象无日志"的空洞。

#### ⑨ `\KernelObjects`：给所有人"看"但不给"改"

先建 SD（`OB/OBINIT.C:720` 一带）：ACL 长度 = `sizeof(ACL) + 3 × sizeof(ACCESS_ALLOWED_ACE) + 三个 SID 长度`，三条允许 ACE——`SeWorldSid` 仅 `DIRECTORY_QUERY | DIRECTORY_TRAVERSE | READ_CONTROL`（查询目录、遍历目录、读控制信息），`SeAliasAdminsSid` 与 `SeLocalSystemSid` 各 `DIRECTORY_ALL_ACCESS`（全部目录权限）。随后 `RtlInitUnicodeString(&TypeDirectoryName, L"\\KernelObjects")`（`OB/OBINIT.C:787`），`OBJ_CASE_INSENSITIVE | OBJ_PERMANENT`，以该 SD 建目录，`NtClose` 关句柄（不取对象指针——`\KernelObjects` 不需要全局量持有，名字本身就是入口）。

为什么 World 能查不能改：内核对象目录（如 `\KernelObjects\…` 下的驱动、设备对象）是名字解析的必经之路——用户态打开设备要先遍历它；若 World 连遍历都没有，普通用户打不开任何设备；若 World 能建对象，低权限用户可往内核对象目录里塞名字（命名空间投毒）。"查询 + 遍历 + 读控制"是最小可用集：能列出名字、能走过它、能读它的 SD（读控制信息是"看规则"本身，不改规则）。为什么管理员与系统给全部：驱动安装、系统服务要在该目录下建对象，必须有创建权限。安全含义（见第五部分）：World 可查意味着低权限用户能枚举 `\KernelObjects` 下的名字——驱动与设备名对所有人可见，是信息披露面（名字本身不是秘密，但给攻击者省了侦察步骤）。

#### ⑩ `\ObjectTypes` 与漫步插入：把 Phase 0 的四类补进目录

`RtlInitUnicodeString(&TypeDirectoryName, L"\\ObjectTypes")`（`OB/OBINIT.C:816`），`OBJ_CASE_INSENSITIVE | OBJ_PERMANENT`，SD 传 `NULL`（走默认 SD——内核对象默认规则），建目录；`ObReferenceObjectByHandle(TypeDirectoryHandle, 0, ObpDirectoryObjectType, KernelMode, &ObpTypeDirectoryObject, NULL)`（`OB/OBINIT.C:834`）取对象指针存全局量；`NtClose` 关句柄。随后漫步（`OB/OBINIT.C:865` 一带）：`Head = &ObpTypeObjectType->TypeList`，从 `Flink` 起逐项取创建者段（`CONTAINING_RECORD(Next, OBJECT_HEADER_CREATOR_INFO, TypeList)`），得对象头（创建者段 +1 即头），取名字段；条件是有名字且 `NameInfo->Directory == NULL`（还不在任何目录里），先 `ObpLookupDirectoryEntry` 查无，才 `ObpInsertDirectoryEntry` 插进 `\ObjectTypes`。最后 `ObpReleaseLookupContext(&LookupContext)`（`OB/OBINIT.C:905`）放锁。

为什么 Phase 0 的类型要漫步补插：Phase 0 建三类时 `\ObjectTypes` 还不存在，`ObCreateObjectType` 的目录插入规则（见 ⑤）是"目录为空则直接成功"——名字当时无处可插；Phase 1 目录一立，必须把已有的类型补插，否则 `\ObjectTypes` 是空的，调试器与 `ObEnumerateObjectsByType` 的类型清单不完整。为什么漫步条件是"有名字且不在目录里"：无名字（匿名类型，理论上不存在但防御性判断）无法进目录（目录项以名字为键）；已在目录里的跳过（防重复插入导致同一名字两项）。注释说的"跳过第一项"辨析：注释原话是"Be sure to skip the first entry in the type object types lists"，但循环实际从 `Flink` 起处理每一项——"第一项"指的是链表头本身（头不是对象，从 `Flink` 起即是跳过头），而不是跳过 Type 类型；Phase 0 的四类（Type、Directory、SymbolicLink、Token——Token 由 Se Phase 0 注册，同样满足"有名字且不在目录里"）全部被插进 `\ObjectTypes`，无一跳过。漫步后是什么局面：`ObpTypeDirectoryObject` 非空，此后任何新类型注册（Section、Process、Device……）都由 `ObCreateObjectType` 自动查重 + 插目录（见 ⑤ 的目录插入规则），不再需要漫步。

#### ⑪ `\DosDevices`：全局 `\??` 与系统进程的设备映射

`ObpCreateDosDevicesDirectory()`（`OB/OBINIT.C:911`，定义在 `OB/OBINIT.C:1693`）：先定 LUID 设备映射开关（`ObpProtectionMode == 0` 或 `ObpLUIDDeviceMapsDisabled != 0` 时关，否则开，`OB/OBINIT.C:1731` 一带）；经 `ObpGetDosDevicesProtection` 取 `\??` 的 SD；以 `ObpGlobalDosDevicesShortName`（全局 `\??` 的短名）建全局 `\??` 目录（`OBJ_PERMANENT`）；`ObSetDeviceMap(NULL, DirectoryHandle)` 把该目录设为系统进程的设备映射（注释原话："It will be stored in the each EPROCESS for use by ObpLookupObjectName when translating names that begin with \\??\\"）。

为什么 `\??` 要在 Ob Phase 1 建：Win32 名字解析（盘符、设备名）从 Phase 1 后半段就开始（Io、SMSS 都要经 `\??` 找设备）；Ob Phase 1 是最后一个能在驱动加载潮之前把翻译层立好的时机。为什么设备映射存在每个 `EPROCESS` 里：`\??\` 开头的名字解析先查本进程的设备映射（会话内覆盖），再查全局 `\??`（共享底）——进程级映射是会话隔离的实现（不同会话的同盘符指向不同设备，靠的正是各进程映射不同）。系统进程的映射先设为全局 `\??`：系统进程没有登录会话，它的 `\??` 就是全局的；用户进程的映射在令牌设置时（登录时）换成会话的。没有会怎样：若 `\??` 未立，Win32 名字（`C:\…`、`\??\C:`）解析无翻译层，全部失败——内核对象存在但 Win32 找不到，系统停在"有设备无盘符"。

### 机制深挖

#### ⑫ 对象头布局：可选段顺序与回指偏移是释放的依据

`ObpAllocateObject`（`OB/OBCREATE.C:805`）的布局算法：先按创建信息算四段大小——`ObjectCreateInfo` 为 `NULL`（类型注册路径）时只分配名字段与创建者段（配额段与句柄段为 0，`OB/OBCREATE.C:869` 一带）；否则按条件算：收费偏离默认值或独占 → 配额段；`MaintainHandleCount` → 句柄段；有名字 → 名字段；`MaintainTypeList` → 创建者段。随后按配额 → 句柄 → 名字 → 创建者的顺序依次布局（`OB/OBCREATE.C:985` 一带），再接固定头，最后接对象体；总大小 = 四段 + 固定头 + 体大小。偏移按"从固定头往回多少字节是该段"算：`QuotaInfoOffset` = 四段总和，`HandleInfoOffset` = 句柄 + 名字 + 创建者段，`NameInfoOffset` = 名字 + 创建者段（`OB/OBCREATE.C:1046` 一带）。固定头初始态：`Flags = OB_FLAG_NEW_OBJECT`（有创建者段时并上 `OB_FLAG_CREATOR_INFO`，有句柄段时并上 `OB_FLAG_SINGLE_HANDLE_ENTRY`），`PointerCount = 1`、`HandleCount = 0`、`Type = ObjectType`（`OB/OBCREATE.C:1091`），`SecurityDescriptor = NULL`（`OB/OBCREATE.C:1124`，SD 在创建后由安全方法赋值并入缓存），`ObjectType` 非空时 `TotalNumberOfObjects` 原子 +1（`OB/OBCREATE.C:1128`）。

为什么顺序固定且释放按逆序：释放（`ObpFreeObject`，`OB/OBCREATE.C:1208` 的注释原话："must be the inverse of that used by ObpAllocateObject"）要按创建的逆序拆段——先拆创建者段（链表摘除）、再拆名字段（目录删名）、再拆句柄段、最后拆配额段（退配额）；顺序乱了，拆名字段时创建者段的链表指针可能已失效，拆配额段时对象可能已被释放。为什么偏移回指而不是存四个指针：四个指针占 16/32 字节（64 位下更多），每个对象都付这笔账；偏移是 3 个字节字段，体指针 + 偏移即定位，省内存且体指针本身就是全部定位信息。为什么 `PointerCount` 初始 1 而 `HandleCount` 初始 0：创建者拿到的是指针引用（返回的对象指针），不是句柄；句柄要另经 `ObpCreateHandle` 插入表项才 +1——指针与句柄从诞生起就是两套账（见术语表铁律）。

#### ⑬ 目录哈希与查找：先哈希、再比值、后比串，锁内先引用

查找（`ObpLookupDirectoryEntry`，`OB/OBDIR.C:950` 一带）：算哈希（见术语表"目录哈希"：`HashValue` 全值，桶号 `% 37`，记入 `LookupContext` 的 `HashIndex` 与 `HashValue`）；取桶头（`Directory->HashBuckets[HashIndex]`），若上下文未持锁则以共享锁锁目录（`ObpLockDirectoryShared`）；沿 `ChainLink` 走链：先比 `DirectoryEntry->HashValue == HashValue`（`OB/OBDIR.C:1047`），不等跳过；相等才取对象头、取名字段，按大小写标志 `RtlEqualUnicodeString` 比串；命中即返回该项（调用方在锁内先引用对象再放锁——Probert 讲义目录查找伪代码的注释原话："命中即在目录共享锁内先引用对象再放锁"）。插入（`ObpInsertDirectoryEntry`，`OB/OBDIR.C:1270` 一带）：新项的 `HashValue` 取自 `LookupContext`，链入桶（头插或按查找记下的插入点）。

为什么三级比对（桶 → 哈希值 → 字符串）：桶定位把比较范围从全目录缩到一条链；哈希值比较是整数比较，比字符串比较便宜一个数量级，且能过滤绝大多数哈希碰撞前的非命中；字符串比较是最终裁决（哈希碰撞时两名字同桶同哈希值，唯有串比能区分）。为什么锁内先引用再放锁：查找返回后若不先加指针引用，放锁的瞬间另一线程可能正好删除该对象（删名 + 指针归零 + 释放），返回的指针成悬空；锁内先引用，对象在引用归零前不会释放，调用方拿到的是活对象。为什么目录锁是推送锁而非自旋锁：查找要走链、比字符串、可能触发引用计数的原子操作，持有时间远超自旋锁的"短持"上限，且查找发生在 `APC_LEVEL` 及以下（分页池的名字空间不允许高 IRQL 访问，见术语表"命名空间池类型"）——推送锁的共享/独占与可阻塞正好匹配；自旋锁在该场景下会要么超时要么违反 IRQL 规则。没有哈希会怎样：目录查找退化为全目录线性比串，`\ObjectTypes`、`\Device` 这类大目录每次打开设备都要扫全表，系统调用延迟随目录规模线性增长。

#### ⑭ 句柄表三级几何：从单页长到 16M 句柄

三级由 `TableCode` 低 2 位编码（`LEVEL_CODE_MASK` 为 3，`EX/HANDLE.C:149`）：0 级 = `TableCode` 直接指低页（`LOWLEVEL_COUNT` 个表项）；1 级 = 指中页（`MIDLEVEL_COUNT` 个低页指针）；2 级 = 指顶页（`HIGHLEVEL_COUNT` 个中页指针）。容量（`EX/HANDLE.C:139-147`）：`LOWLEVEL_COUNT = TABLE_PAGE_SIZE / sizeof(HANDLE_TABLE_ENTRY)`（`TABLE_PAGE_SIZE` 即 `PAGE_SIZE`，`EX/HANDLE.C:96`），`MIDLEVEL_COUNT = PAGE_SIZE / sizeof(PHANDLE_TABLE_ENTRY)`，`HIGHLEVEL_COUNT = MAX_HANDLES / (LOWLEVEL_COUNT × MIDLEVEL_COUNT)`（`MAX_HANDLES` 为 `1<<24` 即 16M，`EX/HANDLE.C:101`）。32 位下表项 8 字节 → 低页 512 项、中页 1024 指针、顶页 32 指针（16M 句柄）；64 位下表项 16 字节 → 低页 256 项、中页 512 指针、顶页 128 指针（仍是 16M 上限，上限与架构无关，几何随指针宽度变）。

初始态（`ExpAllocateHandleTable`，`EX/HANDLE.C:2536`）：表描述符分页池分配（标签 `'btbO'`），一页 `TABLE_PAGE_SIZE` 作首个低页，`TableCode` 直接指该页（低 2 位为 0，即 0 级，`EX/HANDLE.C:2626`）；0 号项留作签名（`EX_ADDITIONAL_INFO_SIGNATURE`），1 号起链空闲表（`NextFreeTableEntry = (i+1) × 4`，FAT 式索引链，末项为 0），`FirstFree = 4`（`EX/HANDLE.C:2663`），`NextHandleNeedingPool = LOWLEVEL_COUNT × 4`（`EX/HANDLE.C:2668`，首个低页用满的阈值）。

增长（`ExpAllocateHandleTableEntrySlow`，`EX/HANDLE.C:3099`；调用方必须已持表锁）：0 级用满时，分配中页（`ExpAllocateMidLevelTable`，附带一个新低页），新低页放中页 [1]，旧低页放中页 [0]，`TableCode = 中页 | 1`（`EX/HANDLE.C:3177`）；1 级下若下个索引仍在中页容量内（`NextHandleNeedingPool / (LOWLEVEL_COUNT × 4) < MIDLEVEL_COUNT`），只分配新低页填中页对应槽；否则 2 级域已满，分配顶页（`HIGHLEVEL_SIZE`）与新中页（附带新低页），顶页 [0] = 旧中页、[1] = 新中页，`TableCode = 顶页 | 2`（`EX/HANDLE.C:3255`）；2 级下按需分配中页与低页。`TableCode` 的更换用 `InterlockedExchangePointer` 原子写——读路径（查表项）无锁读 `TableCode`，写路径原子换指针，旧层在引用归零前不释放（读侧的无锁读与写侧的原子换是句柄表高频读、低频扩的并发设计）。

为什么三级而不是定长大表：每进程一张定长 16M 表项的表要 128MB（32 位）/ 256MB（64 位）虚拟空间，绝大多数进程只用几十个句柄；三级按需生长，小进程一页足矣，大进程（如句柄上万的服务）才长到 2 级，内存与虚拟空间都省。为什么级别编进 `TableCode` 低 2 位：页对齐的指针低 2 位恒为 0，借来编码级别零额外内存；查表项时一次掩码即知级别，不必另存级别字段。为什么空闲链用索引而不用指针：整表复制（继承）时索引链可原样复制，指针链要逐项重算——见术语表"句柄表项"。

#### ⑮ 句柄生命周期：创建、引用、关闭、继承、条件删除

创建（`ObpCreateHandle`，`OB/OBHANDLE.C:2475`）：先验对象类型与期望类型是否一致（不一致直接失败）；经 `ExCreateHandle` 在表里取空闲项（快路径取 `FirstFree`，空则走 `ExpAllocateHandleTableEntrySlow` 扩表）；表项的 `GrantedAccess` 落访问状态里算好的掩码（见术语表"授予访问掩码"），`Object` 指针落对象体（低 3 位并上属性位）；`ObpIncrementHandleCount` 给对象句柄计数 +1（并做配额：Design Notes #24 的配额原话——收费触发点是每创建一个句柄，不是每个对象；自建、继承、再开第二个句柄各收一份；关句柄时退还；收费额 = 对象头 + 对象体 + 句柄表项 + 对象名 + SD，没有 SD 的也要固定收 256 字节，预留将来挂 SD 的空间）。

引用（`ObReferenceObjectByHandle`）：凭句柄值查表项（索引 = 句柄值 / 4），验表项对象类型与期望类型、验 `GrantedAccess` 是否覆盖期望访问（`DesiredAccess`），通过则指针计数 +1 并返回对象指针。为什么引用时要验类型与权限：句柄值可猜（索引 × 4，见第五部分），不验类型，文件句柄可冒充进程句柄；不验权限，读句柄可冒充写访问——表项里的 `GrantedAccess` 就是引用时的验权依据。

关闭（`NtClose` → `ObCloseHandle` → `ExDestroyHandle`）：表项从空闲链摘除（或挂回空闲链），句柄计数 −1；若句柄计数归零且对象非永久且有名字，尝试删名——条件删除（Design Notes #24 的"放锁—重验"：先放开已持有的互斥体、拿目录互斥体、定位目录项、再查一次 `HandleCount` 仍为零才真正删；重验防关闭与新建句柄的竞态）；删名后指针计数 −1；指针计数归零则删对象（走 `DeleteProcedure`，再释放头与体；分页池对象在高 IRQL 下不能当场释放时挂延迟删除队列，见术语表"延迟删除工作项"）。

继承（见术语表"句柄继承"）：`OBJ_INHERIT` 项在子进程创建时由 `ExDupHandleTable` 整表复制（索引链原样复制是 FAT 链的收益），`ObDupHandleProcedure` 逐项过滤（非继承项跳过）、先加指针计数再解锁旧项、新项句柄计数 +1。为什么继承是整表复制而非逐项重开：重开要重做安全检查（贵）且语义不同（继承是"父进程已打开的，我直接有"，不是"我重新申请打开"）；整表复制 + 过滤是既快又保语义的选择。

#### ⑯ 安全描述符缓存：内容相同的 SD 只存一份

机制见术语表"安全描述符缓存"与"安全描述符哈希"，此处补三点为什么。第一，为什么缓存不按对象指针而按 SD 内容哈希：同一 SD 内容（如默认 SD、从父目录继承的 SD）被成千上万对象共享，按指针缓存等于每对象一份，毫无去重；按内容哈希，内容相同的 SD 落同一槽、比对命中即共享。第二，为什么比对要全内容比对（`ObpCompareSecurityDescriptors`）而非只比哈希：哈希 32 位必有碰撞，两 SD 同哈希不同内容时若只比哈希就错共享——错共享等于把 A 对象的访问规则安到 B 对象头上，是权限错乱；全内容比对是去重的正确性底线。第三，为什么取用与归还是引用计数而非"缓存永不释放"：SD 会被改（`NtSetSecurityObject` 换新 SD）与被删（对象删除时 SD 引用归还）；永不释放则缓存只增不减，长期运行池泄漏；引用计数让"最后一个共享者离开时释放"精确成立。没有缓存会怎样：每对象复制一份 SD（典型 SD 数百字节），对象上万时池浪费数 MB，且 SD 比较（访问检查时比 SD 是否相同）退化为逐字节比对；缓存后"是否同一 SD"可先比指针（同一缓存项即同一内容），访问检查的快路径受益。

#### ⑰ 方法表：类型行为契约的收口

`ObCreateObjectType` 的收尾（`OB/OBTYPE.C:367` 一带）把方法表钉死：`SecurityProcedure` 为空默认 `SeDefaultObjectMethod`（Se 的默认安全方法：按 SD 算授予访问、做审计）；`OpenProcedure` / `CloseProcedure` 在句柄打开与关闭时回调（`MaintainHandleCount` 为真时至少要给一个，否则注册失败——Design Notes #24 的校验原话）；`DeleteProcedure` 在对象删除时回调（Type 清锁、SymbolicLink 清目标串）；`ParseProcedure` 在名字解析中途回调（SymbolicLink 跟随、设备解析）；`QueryNameProcedure` 与 `OkayToCloseProcedure` 分别在查询名字与关闭前回调。为什么方法表是"契约"而非"建议"：对象管理器在打开、删除、解析、安全检查这些通用路径上只认方法表，不认具体类型；方法表缺失的方法走默认（安全方法有默认，删除方法可空——空即无体外资源可清），方法表给错的（如 `Parse` 返回非法重解析名）直接导致解析错误或死循环。Probert 讲义的价值正在于此：它逐系统列出了各主要类型的实际方法表（本批三类 + 后续 Process、File、Device 等），读源码时可对照"这类的行为契约是哪张表"。

#### ⑱ 鸡生蛋结总述：三段讲透（对应 ④、⑩、③ 的解开机制，此处补来龙去脉）

**结一：Type 自举。** 结为什么会形成：NT 的对象模型（1989 年设计时即定）要求"一切皆对象"——对象由类型定义行为，类型由对象承载存在；这是一条自指定义，不是实现疏忽，是模型本身的代价。任何"先有类型还是先有对象"的问题，在这个模型里都是同义反复。靠什么打破：第一次调用不传类型（`NULL`），分配路径对 `NULL` 类型有专用的无类型分支（非分页池 + `'TjbO'` 标签 + 只建名字段与创建者段），分配回来后事后自指（头类型指针指自己、计数置 1）。打破后是什么局面：Type 的 Type 是 Type，此后一切类型创建都有类型可依；`ObpObjectTypes[0]` 是 Type，`TypeList` 从 Type 开始累积。注意：自指不是"循环引用导致泄漏"——Type 是永久对象（`OB_FLAG_PERMANENT_OBJECT`，`ObCreateObjectType` 统一置位，`OB/OBTYPE.C:266` 一带），本就不删除，自指不产生释放问题。

**结二：目录与类型互赖。** 结为什么会形成：命名空间要求类型"已发布"（进 `\ObjectTypes` 目录）才算存在（调试器、枚举、查重都以目录为准），而目录本身是 Directory 类型的对象——类型要目录背书，目录要类型先存在。靠什么打破：分时——类型先裸创（目录不存在时跳过插入，不报错），目录后建（用已存在的 Directory 类型），再漫步把裸创的类型补插进目录（见 ⑩）。打破后是什么局面：`\ObjectTypes` 完整（Phase 0 四类全在），此后新类型注册自动查重 + 插目录，漫步不再需要。历史注脚：Design Notes #24 的规范写于 1989 年，当时即定"类型名存放在 `\ObjectTypes` 目录下"（精读 §1），而 WRK 的实现（2003 年代码线）用"裸创 + 漫步"兑现了这条规范——规范没写漫步，实现补了漫步。

**结三：句柄表与进程。** 结为什么会形成：句柄表挂在进程对象上（`EPROCESS.ObjectTable`），而进程对象本身要靠对象管理器创建（`PsInitSystem` 在 Ob 之后）——表要进程，进程要表。靠什么打破：借位——Phase 0 的 current 进程是 Idle 占位进程（第 1 批的 `KiInitialProcess` 静态 `EPROCESS`），它的 `ObjectTable` 槽位空着，内核句柄表直接装进该槽位；内核句柄全局有效（不归属任何用户进程），借槽位不产生归属矛盾。打破后是什么局面：`ObpKernelHandleTable` 独立存在（全局量持有），用户进程由 `Ps` 在创建时另建自己的表（`ObInitProcess` → `ExCreateHandleTable(Process)`，收配额）；内核句柄与用户句柄从此分表而治，进程退出销毁自己的表时不碰内核表。

### 第三部分：32 位与 64 位对照

- **句柄表几何随指针宽度变，上限不变**：表项 32 位 8 字节、64 位 16 字节（指针 + 掩码/索引各一指针宽），故低页 512 → 256 项、中页 1024 → 512 指针、顶页 32 → 128 指针；`MAX_HANDLES`（16M）是架构无关的常量，上限不变，只是 64 位下顶页更大（128 指针 × 8 字节 = 1KB，仍在一页内）。为什么上限不随架构变：句柄值是 30 位索引（`EXHANDLE.Index : 30`，`INC/EX.H:4971`），30 位索引 × 4 即 32 位句柄值空间；16M（`1<<24`）是留有余量的实用上限（30 位理论上限是 1G 句柄，表结构撑不到——顶页 128 指针 × 中页 512 × 低页 256 = 16M 是三级几何的实际顶）。
- **i386 的缓存授予访问是 32 位调试专属**：`ObpCachedGrantedAccesses` 数组（2 页，索引代替掩码存表项）只在 `i386` 且 `FLG_KERNEL_STACK_TRACE_DB` 时启用（`OB/OBINIT.C:490` 一带）；`ObDupHandleProcedure` 里 `#if i386` 分支（`OB/OBINIT.C:995` 一带）才做 `ObpTranslateGrantedAccessIndex` 翻译，AMD64 直用 `GrantedAccess`。为什么 64 位不要这套：缓存授予访问是为"句柄追踪数据库"（按授予访问反查句柄）省内存——32 位下掩码 4 字节、索引 2 字节，省一半；64 位下表项本就 16 字节，索引化省的比例小，且 AMD64 的句柄追踪路径直接读掩码更简单——架构差异导致同一功能在 64 位下被简化掉。
- **配额段的 64 位对齐**：`OBJECT_HEADER_QUOTA_INFO` 在 `_WIN64` 下多一个 `Reserved` 字段（`INC/OB.H:450` 一带），只为把结构撑到 16 字节对齐。为什么：64 位下对象头的可选段布局要求 16 字节对齐（`QUAD Body` 占位与原子操作对齐），32 位下 8 字节足矣——对齐差异不影响语义，只影响头大小（64 位头略大）。
- **目录哈希的块路径是 64 位性能优化**：64 位下 4 宽字符一组做 128 位块运算（`OB/OBDIR.C:950` 一带的 `#if` 块路径），32 位下逐字递推。为什么：64 位寄存器一次能吞 4 个宽字符（8 字节）并行大写折叠，哈希算得快；32 位寄存器吞不下，只能逐字。桶号跨架构可能不同（见术语表"目录哈希"），但桶号不进持久格式，无 ABI 影响。

### 第四部分：单核与多核对照

- **每处理器前瞻列表**：Phase 1 给每处理器另建 P 列表（见 ⑦）。单核下循环只跑 P0 一次，L 与 P 分开后仍是 P0 的两条列表（P 快、L 兜底），逻辑与多核一致；多核下各 CPU 的 P 列表互不干扰，创建路径的竞争只发生在 P 空回落到 L 时。为什么单核不省掉 P：省掉要多一套"单核走 L、多核走 P"的分支，分支本身是 bug 来源；统一走 P/L 两级，单核下 P 恒命中、L 恒空闲，代价只是一条空列表的内存。
- **锁：多核要，单核也走同一套**：目录推送锁、类型 `ERESOURCE`、句柄表 4 把推送锁条带化、SD 缓存 257 链分片——单核下这些锁恒无竞争（获取即成功），但代码不为单核特判跳过。为什么：锁的正确性不依赖核数（单核下中断与 APC 仍能制造"并发"——如 APC 插队、DPC 抢占），跳过锁等于在单核下埋竞态；且无竞争锁的获取代价是几条原子指令，可忽略。唯一的单核特判在别处：`#ifndef NT_UP` 的块（如第 2 批的拉 AP、重读 RTC）——Ob 的锁不在特判之列。
- **SD 缓存分片**：257 链在多核下把哈希竞争散开（不同 SD 落不同链、拿不同锁）；单核下等价于单链（一次只一条链被访问），分片无额外代价（257 个链表头与推送锁是静态内存，不随核数变）。
- **句柄表扩表锁**：`HandleTableLock[4]` 条带化在多核下按句柄值哈希选锁，扩表并发时四把锁分流；单核下扩表恒串行（一次只一个扩表者），四把锁退化为"按哈希挑一把用"，逻辑不变。

### 第五部分：逆向与安全解读

- **`\ObjectTypes` 是对象宇宙的清单**：Phase 1 后 `\ObjectTypes` 下是当时已注册的全部类型；Probert 讲义 p.4–5 列出 2003 年时点的 29 种（Type、Directory、SymbolicLink、Token、Section、Process、File、Device 等）。逆向价值：调试器（WinObj、WinDbg `!object`）枚举 `\ObjectTypes` 即得内核对象类型全集；新驱动注册的新类型会立即出现在这里——"系统里多了什么类型"是驱动加载的直接信号。防御价值：监控 `\ObjectTypes` 的新增类型是检测未签名驱动 / rootkit 注册新对象类型的廉价探针（正常驱动的类型集合相对稳定）。
- **`\KernelObjects` 的 World 可查是信息披露面**：World 仅 `QUERY | TRAVERSE | READ_CONTROL`（见 ⑨），低权限用户能枚举 `\KernelObjects` 下的名字（驱动名、设备名、回调名）。名字本身不是秘密，但攻击者借此省了侦察步骤（不必猜驱动名，直接列出）。缓解不在 Ob 层（World 可查是"能打开设备"的必要条件，收紧会断正常用户设备访问），而在驱动层（设备对象自身的 DACL 决定谁能真正打开——目录可查不等于设备可用）。
- **`\??` 与设备映射是会话隔离的边界**：进程的设备映射（会话内 `\??`）与全局 `\??` 分离，意味着 A 会话的盘符劫持（如把 `C:` 指到攻击者设备）不影响 B 会话；全局 `\??` 的写入需要特权（系统进程的映射是全局的，普通用户进程的映射是会话的，改全局映射要 `SeTcbPrivilege` 一类特权——具体特权名未在本批源码段内钉死，见未钉死清单）。攻击面在"会话内"：同一会话内的低权限进程若能写本会话的 `\??`（如经某个有写权限的目录项），可劫持同会话高权限进程的盘符解析——这是"会话内提权"而非"跨会话提权"，边界比想象中窄，但同会话内仍需警惕。
- **符号链接 `Parse` / `REPARSE` 是链接竞态的温床**：链接目标在解析时才跟随，目标在检查时与使用时之间被换（TOCTOU，Time-Of-Check-Time-Of-Use）是经典问题；且链接可指向 `\Device\…` 之外的敏感路径（如 `\KernelObjects` 下的对象），`Parse` 返回的重解析名若未经权限重验即放行，等于绕过原路径的目录权限。缓解在 Ob 层：重解析后从头再走（见术语表"重解析"），重走时每级目录的权限重验——"从头再走"本身就是防"中途改道绕权限"的设计。
- **句柄值可预测，权限在表项**：句柄值 = 索引 × 4（见术语表"句柄"），低权限进程可猜高权限进程的句柄值（如 4、8、12……）；但猜中值不等于拿到对象——`ObReferenceObjectByHandle` 按句柄所属进程的表查项（跨进程句柄值无效），且验 `GrantedAccess`；跨进程拿句柄的唯一合法路是 `NtDuplicateObject`（需源进程 `PROCESS_DUP_HANDLE` 权限）。严格 FIFO（调试版）防 ABA（见术语表"句柄严格 FIFO"）：零售版复用快，旧句柄值重指向新对象是已知语义（调用方必须在关闭后不再用旧值——"关闭后用旧句柄"是调用方 bug，不是内核 bug）。
- **SD 缓存共享的不可变约定**：同一缓存 SD 被多个对象共享，原地改一处等于改所有共享者的规则；设计上 Se 从不原地改缓存副本（总是算新 SD 再入缓存）。攻击面在引用计数：`ObLog`（入缓存 +1）与 `ObDereferenceSecurityDescriptor`（归还 −1）若失衡，缓存项提前释放（悬空 SD 指针）或永不释放（池泄漏）；前者是权限错乱（悬空指针被重用为其他数据，访问检查读错规则），后者是拒绝服务（DoS）。缓存每链一把推送锁 + 全内容比对是该计数的并发防线。
- **方法表是新类型的攻击面入口**：驱动注册新类型时挂的 `Parse` / `Delete` / `Security` 方法在对象管理器的上下文运行（解析、删除、安全检查路径），方法里的 bug（空指针、锁序错、IRQL 违规）即内核 bug；且方法表一旦注册即全局生效（所有进程解析该类型都走它）。防御上，类型注册本身需要内核权限（驱动加载），方法表的风险由驱动签名与代码审计兜底，不由 Ob 层再验。

### 第六部分：易混点辨析

1. **"前四类"不是 Ob 一家注册四类**：Ob Phase 0 注册三类（Type、Directory、SymbolicLink），第四类 Token 由 Se Phase 0（`SepTokenInitialization`，`SE/SEINIT.C:144`）注册；Phase 0 结束时系统里恰为四类。路线"前四类"的正确读法是"Phase 0 结束时已有的四类"，不是"ObInitSystem 注册四类"——后者与源码（`OB/OBINIT.C` 里仅三处 `ObCreateObjectType` 调用，`:431`、`:450`、`:473`）矛盾。
2. **内核句柄表 ≠ 系统进程的私有表**：`ObpKernelHandleTable` 与系统进程 `ObjectTable` 是同一指针（`OB/OBINIT.C:398`），但语义是全局内核表，不是"系统进程私有的表"。系统进程退出（理论上不退出）不会销毁它；用户进程的表由 `Ps` 另建。把"借槽位"理解成"归属系统进程"是错的——归属是"内核"，槽位只是存放处。
3. **Type 自引不是循环错误**：Type 对象的头类型指针指向它自己（`OB/OBTYPE.C:289`），是自举的正确终态，不是"循环引用导致泄漏"。Type 是永久对象，本就不删除；自指不产生释放问题，也不影响 `TotalNumberOfObjects`（置 1 计数自身）。
4. **`\ObjectTypes` 漫步只插当时已有的四类，之后类型自动插**：漫步（`OB/OBINIT.C:865`）是一次性的补插（Phase 0 已有、当时不在目录里的类型）；此后 `ObCreateObjectType` 在目录已存在时自动查重 + 插目录（`OB/OBTYPE.C:470`），不再漫步。把"漫步"理解成"每次注册都漫步"是错的——漫步是补历史欠账，自动插是常态。
5. **句柄值不是指针**：句柄值 = 表项索引 × 4（`HANDLE_VALUE_INC`），低 2 位是标签位（内核句柄恒 0）；权限在表项（`GrantedAccess`），不在句柄值里。把句柄值当指针解引用是错的（用户态地址空间里该值无意义）；把句柄值当权限凭证也是错的（值本身不带权限，表项才带）。
6. **目录 `HashValue` ≠ 桶号**：`HashValue` 是名字的全哈希（32 位），桶号是 `HashValue % 37`；查找先比 `HashValue`（整数快筛），再比字符串（最终裁决）。把"同桶"当成"同名"是错的（同桶不同哈希值的名字共存一链，靠串比区分）；把"同哈希值"当成"同名"也是错的（哈希碰撞时两名字同哈希值，唯有串比能区分）。
7. **SD 缓存的"共享"不是"共用一份任改"**：缓存 SD 被多个对象共享指针，但设计约定是不可变（Se 总是算新 SD 再入缓存，从不原地改缓存副本）。"共享指针"若被理解成"改一处即改全部规则"是对的（这正是不可变约定的反面警示），但"可以原地改共享 SD"是错的——原地改是 bug，不是功能。
8. **命名空间在分页池，名字解析有 IRQL 上限**：目录与符号链接对象本体在分页池（`OB_NAMESPACE_POOL_TYPE = PagedPool`），故名字解析只能在 `APC_LEVEL` 及以下（分页池在 `DISPATCH_LEVEL` 及以上不可访问，见第 3 批池红线）。"任何 IRQL 都能解析名字"是错的——`DISPATCH_LEVEL` 及以上的路径（如 DPC、ISR（中断服务例程））不能走名字解析，必须事先持有对象指针或句柄。

### 第七部分：自查题

1. `ObCreateObjectType` 第一次被调用时，`ObpTypeObjectType` 是 `NULL`，`ObpAllocateObject` 走了哪条特殊路径？事后哪三行代码完成自指？若自指漏了，`Index` 与 `TotalNumberOfObjects` 会错成什么样？
2. Phase 0 的三类注册为什么走"目录为空、直接成功"支？Phase 1 的漫步为什么以"有名字且 `Directory == NULL`"为条件？若漫步漏了 Token，`\ObjectTypes` 会缺什么、谁会受影响？
3. 根目录的 SD 为什么是"审计开则现场造、不开则用公共默认"？审计 SD 的 SACL 为什么只需在根上挂一次？`CONTAINER_INHERIT_ACE | INHERIT_ONLY_ACE` 与 `OBJECT_INHERIT_ACE` 各管什么？
4. 句柄表从 0 级长到 2 级的过程中，`TableCode` 的低 2 位如何变？旧低页在 0→1 时被放到中页的哪一槽？为什么空闲链用索引而不用指针（从子进程继承整表的角度回答）？
5. SD 缓存的槽位为什么是哈希 % 257 而不是 % 256？`ObLogSecurityDescriptor` 命中时为什么"引用已有并释放传入的 SD"，而不是"直接返回传入的 SD"？

### 附录：未钉死清单

1. 延迟删除队列的具体入队条件：哪些删除路径把对象挂进 `ObpRemoveObjectList`（`OBCLOSE.C` / `OBREF.C` 的删除全程未细读）；本批只钉死了队列的初始化（`OB/OBINIT.C:369`）与工作项（`:410`）。
2. `ObpParseSymbolicLink` 在 WRK 里的完整实现细节：`LinkTargetRemaining` 与 `LinkTargetObject` 的维护（缓存目标对象何时失效、何时重解析）未细读；本批只钉死了注册（`OB/OBINIT.C:476`）与 Design Notes #24 的 `Parse` 语义。
3. `\DosDevices` 的 SD 具体条款：`ObpGetDosDevicesProtection` 的 SD 内容（哪些 SID、哪些权限）未细读；本批只钉死了"经该函数取 SD、建全局 `\??`、设系统进程设备映射"（`OB/OBINIT.C:1693` 一带）。
4. 改全局设备映射所需的特权名：普通用户进程改全局 `\??` 映射要哪项特权（`SeTcbPrivilege` 一类），未在本批源码段内钉死；本批只钉死了"系统进程的映射是全局的、用户进程的映射是会话的"这一结构。
5. `SePublicDefaultUnrestrictedSd` 的具体 ACE：Se Phase 0 造的公共默认无限制 SD 的条款（哪些 SID、哪些权限）未细读；本批只用了它的名字（Phase 1 根目录的默认 SD）。

### 对位小结

- **Design Notes #24《Object Management》是规范，WRK `OB/` 是实现**：规范定的方法表模板（`OBJECT_TYPE_INITIALIZER` 的过程指针）、`MaintainHandleCount` 的校验原话（Open/Close 至少给一个）、配额"按句柄收费、无 SD 收 256 字节预留"、指针计数铁律（`PointerCount` 永不低于 `HandleCount`）、条件删除的"放锁—重验"、锁序三级（对象表互斥体 < 目录互斥体 < 类型互斥体）、`Parse` / `REPARSE` 语义与符号链接的 `Parse` 范例——在 WRK 里逐一落到了 `OBTYPE.C`、`OBCREATE.C`、`OBHANDLE.C`、`OBDIR.C` 的对应函数上。读规范知"为什么这样设计"，读源码知"这样设计长什么样"。
- **Probert《Object Manager》讲是实现的导游**：四层模型（命名空间、类型与方法表、打开链、句柄表三级结构）与本批正文的 ⑫–⑰ 一一对应；命名空间树（`\`、`\Device`、`\BaseNamedObjects`、`\ObjectTypes`、`\KernelObjects` 等）与本批 ⑧–⑪ 的目录一一对应；`\ObjectTypes` 2003 年 29 种清单是"Phase 0 四类 → Phase 1 漫步 → 后续类型陆续注册"这条时间线的终态快照；目录查找伪代码的"先引用再解锁"与本批 ⑬ 的锁内引用是同一条防悬空指针纪律；句柄表一节是源码级伪代码，本批 ⑭ 的三级几何与增长路径是它的 WRK 落点。

### 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，与正文互补。

- **类型过程调用：函数指针间接调用**。对象操作（解析、打开、关闭、删除、安全检查）不是 switch 分发，是经对象类型对象里的过程指针调用：`ObCreateObjectType` 时填进类型对象的 `ParseProcedure` / `OpenProcedure` / `CloseProcedure` / `DeleteProcedure` / `SecurityProcedure`，用时直接调指针。转移方式：间接函数调用；新增对象类型即新增一组指针，对象管理器代码不用改。
- **符号链接重解析：循环内 goto 重启，有次数上限**。路径解析走到符号链接对象时，类型过程 `ObpParseSymbolicLink`（`OB/OBINIT.C:472` 注册）返回 `STATUS_REPARSE`（或 `STATUS_REPARSE_OBJECT`）→ `ObpLookupObjectName` 的循环内 `MaxReparse--`（防无限链接环）→ 若剩余名以 `\` 开头或带重解析对象，丢掉原根目录、从根目录对象 `ObpRootDirectoryObject` 经 `goto ParseFromRoot` 重走（`OB/OBDIR.C:2573-2660` 一带）；带已解析对象时经 `goto ReparseObject` 继续。转移方式：循环内 goto 重启，不是递归——重解析不压新栈帧，同一循环体内换个起点再走一遍。
- **句柄 → 对象：查表翻译，不是执行转移**。句柄值是句柄表项索引（`HANDLE_VALUE_INC` 为 4 的缩放），`ObReferenceObjectByHandle` 按索引取表项、取对象头、做访问检查——全程函数调用，没有中断、没有换栈。特意写明：本批最容易误读成“转移”的其实是翻译，不要把查表当成跳转。
- **对象删除：引用计数归零 → 类型删除过程**。最后一个引用解开时，对象管理器调类型对象的删除过程（函数指针），随后释放对象体——转移方式仍是间接函数调用，触发点是计数归零，不是显式调用删除。
