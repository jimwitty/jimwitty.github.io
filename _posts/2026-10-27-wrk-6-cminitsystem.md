---
layout: post
title: "WRK 第 6 批：CmInitSystem 与注册表点火"
categories: WRK
series_order: 7
---

## 本批在点火链中的位置

Phase 1 的 `ExpInitializeExecutive` 走到 `INIT/INITOS.C:1437` 时调 `CmInitSystem1(LoaderBlock)`。
它夹在两个已交付的点之间：前面是 `CcInitializeCacheManager`（`:1426`，缓存管理器刚上线），后面是
`IoInitSystem`（`:1531`，I/O 系统与驱动加载潮，第 7 批才讲）。这个位置本身就是本批的第一条机制结论，
见正文第 ① 步：注册表必须在 I/O 之前可用（驱动加载要读 `Services`），但此时文件系统还没起来，
所以第一阶段的注册表只能是“内存里的注册表”，写操作只留内存、不落盘。

第二阶段在 Phase 1 之后很久：会话管理器 SM 在自动检查（autocheck，即 chkdsk）跑完、页面文件打开之后，
调 `NtInitializeRegistry`（`CONFIG/NTAPI.C:1030`），CM 才把各 hive 绑到 `\SystemRoot\config` 下的
文件上，并把第一阶段攒下的修改刷盘。两阶段的分界线不是“CM 写了两遍代码”，而是“文件系统何时可用”，
见正文第 ⑫ 步。

## 取材说明

底本是 WRK 树（WS03 SP1 代码线）`BASE/NTOS/CONFIG/` 下的注册表实现，核心文件是 `CMSYSINI.C`
（点火）、`CMINIT.C`（hive 初始化）、`HIVEINIT.C`（hive 底层初始化）、`HIVELOAD.C`（文件装载与
日志恢复）、`HIVESYNC.C`（刷盘）、`CMWORKER.C`（懒刷）、`CMPARSE.C`（名字解析与链接节点）、
`CMCONFIG.C`（硬件配置树）、`CMBOOT.C`（驱动清单排序）、`NTAPI.C`（第二阶段入口）。
加载器（ntldr）本体不在 WRK 树内：凡说“加载器做了什么”，依据是内核侧读到的 `LoaderBlock`
字段与 CM 代码里的注释，不是 ntldr 源码；凡涉及 HAL 的，只讲调用点。
本批与第 2 批的关系：第 2 批只给了 `CmInitSystem1` 在点火总控里的时序坐标，本批讲它的全部内部；
与第 4 批的关系：第 4 批讲对象类型机制，本批讲 CM 如何用它注册 `Key` 类型并建名字空间。

## 第一部分：术语表（按引用级排序）

表头惯例（沿用第 3 批）：基础概念在前、派生概念在后；读到某条时它引用的词前面都已定义。
互为前提的一对是“hive 与键控制块”：hive 的根要靠键控制块寻址，键控制块又要指回它所属的 hive。
处理办法与第 3 批的 VA/页表相同：hive 条目里给键控制块行内解释并指到后面的键控制块条目，
本表头在此披露，不藏着。

### 一、注册表的名字世界

**注册表（Registry）**：Windows 集中存放系统与应用配置的层次数据库，逻辑上是一棵以
`\Registry` 为根的树。做什么用：让驱动、服务、系统组件在启动与运行中用统一的名字查配置，
而不是各自读私有文件。为什么要有：启动早期只有内核，没有文件系统之上的应用层服务，
配置必须有一个内核直接能读写的、与文件格式解耦的载体；注册表就是这个载体。

**键（Key）**：注册表树上的一个节点，相当于文件系统里的目录，有名字、子键、值、安全描述符。
做什么用：承载一层配置分组。为什么要有：配置天然分层（机器、系统、服务、驱动），
没有分组节点就只能是一张扁平表，查不到“某服务的所有参数”这种集合。

**值（Value）**：挂在键下面的一个具名数据项，有名字、类型（如 `REG_DWORD`、`REG_SZ`、
`REG_LINK`）、数据。做什么用：存具体的配置项。为什么要有：键只分组不存数，
没有值就存不下“这个服务是自动启动还是手动启动”这类事实。

**根键名**：用户态熟悉的 `HKEY_LOCAL_MACHINE`（HKLM）、`HKEY_USERS` 等只是 Win32 的简写，
内核里的真名是 `\Registry\Machine`、`\Registry\User`。做什么用：把 Win32 的预定义句柄
翻译成内核名字空间路径。为什么要有：内核对象管理器只认 `\Registry` 下的路径，
Win32 简写是应用层的糖，内核层必须用真名。

**名字解析（Parse）**：对象管理器按路径一段一段走树、遇到特殊节点换道的过程。
做什么用：把 `\Registry\Machine\System\...` 这样的路径变成一个键对象。
为什么要有：注册表树由多个独立的存储单位拼成（见下面的 hive 与链接节点），
解析是唯一能把“一段路径”翻译成“某个 hive 里的某个 cell”的机制。

### 二、hive：注册表的存储单位

**hive（配置单元）**：注册表在磁盘与内存里的一个独立存储单位，一个文件（或一段内存映像）
对应一棵子树。做什么用：让不同来源的配置可以独立装载、卸载、刷盘、恢复。
为什么要有：如果整棵注册表树只有一个文件，损坏一个分支就全树遭殃，系统 hive、
软件 hive、用户 hive 也无法按各自的生命周期开关；拆成 hive 才能做到“SYSTEM 先起来，
SOFTWARE 后装载，单个用户 hive 随登录登出”。

**主 hive（Master Hive）**：启动时第一个创建的、只存在于内存的 hive，装 `\Registry`、
`\Registry\Machine`、`\Registry\User` 这几个根节点，以及所有其他 hive 的链接节点。
做什么用：给名字空间一个骨架，让各个 hive 有地方挂。为什么要有：没有骨架，
`\Registry\Machine\System` 这个路径的父目录都不存在，SYSTEM hive 根本挂不上去；
又因为它只装骨架与链接，不装业务数据，落盘没有意义，所以它是易失的（见下面易失 hive）。

**SYSTEM hive**：存机器级系统配置的 hive，文件在 `\SystemRoot\config\system`，
内容以 `ControlSet`、`Services`、`Select` 为核心。做什么用：驱动加载、服务启动、
控制集选择全靠它。为什么它特殊：它是唯一一个由加载器整段读入内存、内核在文件系统
起来之前就必须用的 hive，所以它的点火路径与其他 hive 完全不同（本批第 ⑤、⑭ 步）。

**HARDWARE hive**：存本次启动探测到的硬件描述的 hive，挂在 `\Registry\Machine\Hardware`。
做什么用：给驱动与内核提供“机器上有什么硬件”的只读快照。为什么它必须是易失的：
硬件描述是本次启动探测的结果，下次启动可能不同，落盘保存只会把旧快照误当成新事实。

**机器 hive 表（CmpMachineHiveList）**：一张静态表，列出启动时要装的 6 个机器 hive：
HARDWARE、SECURITY、SOFTWARE、SYSTEM、DEFAULT、SAM（`CONFIG/CMDAT.C:284`）。
做什么用：第二阶段按表并行装载的清单。为什么要有：一张表定死顺序与名字，
并行装载的线程才知道自己负责第几个、该去哪个文件、该挂到哪个路径。

**hive 映像（Hive Image）**：一个 hive 在内存里的连续字节序列，加载器读 SYSTEM 文件
得到的就是一段映像，内核接管时拿到的也是它。做什么用：文件系统未起来时唯一可用的
SYSTEM 数据形态。为什么要有：启动早期没有“打开文件”这个操作，只能由加载器
“整段搬进内存”再交给内核，这是加载器与内核之间的交接物。

**基块（Base Block，HBASE_BLOCK）**：每个 hive 文件与映像开头的 4KB 头，
签名 `regf`（`HBASE_BLOCK_SIGNATURE`，`INC/HIVEDATA.H:326`）、两个序列号、
时间戳、主次版本号、文件类型、格式、根 cell 索引、总长度、簇大小、文件名尾、
校验和、启动类型（BootType）、启动恢复标志（BootRecover）。
做什么用：hive 的身份证与一致性锚点。为什么要有：没有它，内核无法判断“这段字节
是不是一个 hive、版本能不能读、上次写盘有没有写完”；序列号与校验和是崩溃恢复的
全部依据（见第 ⑰ 步）。

**序列号（Sequence1 / Sequence2）**：基块里的两个计数器，写盘开始前升 Sequence1，
写盘完成后升 Sequence2，正常状态下两者相等。做什么用：判断上次写盘是否完整。
为什么要有两个而不是一个标志位：写盘是多步操作，任何一步崩溃都会让两者不等；
一个标志位只能说“脏”，两个计数器还能区分“正在写”与“写完”，且不需要额外的
原子写保证——崩溃后只要两者不等，就知道主文件不可信、必须走日志恢复。

**校验和（CheckSum）**：基块头部字段按固定算法算出的校验值，装载时重算比对。
做什么用：发现基块本身损坏。为什么要有：序列号只能发现“写了一半”，
校验和才能发现“基块被踩坏、磁盘静默出错”这类内容损坏。

**簇（Cluster）**：hive 文件的分配与脏跟踪单位，大小是扇区大小（`HSECTOR_SIZE` = 512 字节）
的整数倍，由卷的扇区大小决定（`HINIT` 时从卷信息算出，`CONFIG/CMINIT.C` 的
`CmpInitializeHive` 里按 `BytesPerSector / HSECTOR_SIZE` 取整，至少为 1）。
做什么用：脏位图以簇为单位记脏，日志以簇为单位扩容。为什么要有：磁盘按扇区读写，
脏跟踪的粒度不能小于物理扇区，否则一次写盘会写出半个扇区这种不存在的操作。

### 三、bin 与 cell：hive 内部的砖块

**bin（HBIN，Hive Bin）**：hive 内部的一段 4KB 对齐的存储块（`HBLOCK_SIZE` = 0x1000，
`INC/HIVEDATA.H:123`），头带签名 `hbin`、偏移与大小。做什么用：hive 文件的
分配单位，cell 住在 bin 里。为什么要有：hive 需要一种“可增长、可回收、可单独写回”
的单位，直接以 cell 为单位管理会在文件里留下大量碎片，bin 把碎片收敛到 4KB 的块里，
写盘与回收都以 bin 为单位进退。

**cell（单元）**：hive 里最小的数据记录，有键节点、键值、子键列表、安全描述符、
大块数据等类型，统一以 cell 索引寻址。做什么用：装一个键、一个值、一张子键表。
为什么要有：注册表是树，树节点大小不一、数量不定，必须有一种变长记录机制；
cell 就是这套机制的记录，bin 是它的容器。

**cell 索引（HCELL_INDEX）**：一个 32 位整数，最高位是类型位（0 = 稳定存储、
1 = 易失存储，`HCELL_TYPE_MASK`，`INC/HIVEDATA.H:112`），其余位是该存储区内的
字节偏移；特殊值 `HCELL_NIL`（-1）表示“无”。做什么用：在 hive 内定位一个 cell，
且一眼能看出它属于哪类存储。为什么索引用“类型位 + 偏移”而不是指针：
hive 数据要在磁盘文件与内存映像之间搬家，指针搬家就失效，偏移搬家仍然有效；
类型位则让稳定与易失两套存储共用一套索引编码。

**cell 映射（Cell Map）**：把 cell 索引翻译成内存地址的两级表（类似硬件页表的两级结构，
`INC/HIVEDATA.H` 的注释原话）。做什么用：cell 住在分散的 bin 里，映射表按索引
的高位找目录项、低位找 bin 内偏移。为什么要有两级而不是一张大表：hive 可增长到很大，
一张覆盖全部偏移的平表太浪费，两级表只给已分配的区间建目录项，未分配的区间不占内存。

**稳定存储（Stable Storage）与易失存储（Volatile Storage）**：hive 内部的两套 cell
存储区，稳定区的 cell 会写盘，易失区的 cell 只活在内存、关机即丢。
做什么用：同一棵 hive 里既要有落盘的配置，也要有只活本次启动的临时项。
为什么要有两套而不是一个 hive 级开关：一个 hive 里两种生命周期的项是混着长的
（例如 SYSTEM hive 里既有落盘的服务配置，也有启动时现算的临时项），
粒度到 cell 才能让“这一项落盘、那一项不落盘”同时成立。

**易失 hive（HIVE_VOLATILE）**：整个 hive 只用易失存储的标志（`CONFIG/HIVE.H:199`），
整 hive 不落盘。做什么用：主 hive、HARDWARE hive 的整 hive 开关。
为什么要有：与“易失 cell”不同，它是整 hive 级的承诺——这个 hive 的全部内容
都是本次启动的派生物，重建比保存更正确，刷盘逻辑看到它直接跳过。

**键节点（CM_KEY_NODE）**：cell 里表示一个键的结构（`INC/CMDATA.H:423`）：
签名、标志、最后写入时间、父 cell、稳定与易失两套子键计数与子键列表、值列表、
安全描述符 cell、类名 cell、名字。做什么用：树上的一个节点本体。
为什么要有：名字解析、枚举、权限检查、安全描述符查找全靠它，键对象（见第 4 批的
对象）只是它的内核对象外壳，节点才是数据本身。

**键值（CM_KEY_VALUE）**：cell 里表示一个值的结构（`INC/CMDATA.H`）：
签名、名字长度、数据长度、数据 cell 索引、类型、标志、名字。
做什么用：一个配置项本体。为什么要有：值的数据可能很大（超过 4 字节就不能塞进
值结构本身），所以值结构只存“数据在哪（cell 索引）+ 多长 + 什么类型”，
大数据再走大块数据（Big Data）cell 拼。

### 四、hive 之上的内核结构

**HHIVE**：hive 的底层控制结构（hive 引擎层）：签名、六个文件操作回调（分配、释放、
改文件大小、写、读、刷）、日志标志、hive 标志、簇大小、两套存储的映射与空闲表、
脏位图、脏计数、基块指针、取 cell 例程。做什么用：让同一套 hive 引擎能工作在
四种来源上（新建、内存映像、文件、缓存映射文件），引擎只认回调、不认来源。
为什么要有：SYSTEM hive 来自内存、SOFTWARE hive 来自文件、主 hive 凭空新建，
如果引擎为每种来源各写一套，cell 分配、脏跟踪、刷盘逻辑就要复制四份；
回调抽象把“数据从哪来、写到哪去”推给 CM 层，引擎只管 hive 内部结构。

**CMHIVE**：CM 层的 hive 控制结构，第一个字段就是 HHIVE（`FIELD_OFFSET(CMHIVE, Hive) == 0`
的断言在多处出现，意思是 CMHIVE 可以直接当 HHIVE 用）。做什么用：在 HHIVE 之外
挂 CM 才需要的东西：文件句柄数组（主文件、日志、外部文件）、hive 锁、写者锁、
视图锁、安全缓存、视图链表、通知链表、hive 链表节点。为什么要有两层：
HHIVE 是“hive 文件格式引擎”，CMHIVE 是“这个 hive 在内核对象世界里的身份”，
引擎层不该知道文件句柄与对象锁，CM 层不该干预 cell 内部布局。

**键控制块（KCB，Key Control Block）**：内核为每个已打开或已缓存的键路径维护的
缓存结构，存键的 hive 指针、cell 索引、父 KCB、名字哈希、子 KCB 链表。
做什么用：让反复打开同一路径不必每次都从根重新解析整条路径，
也让“键被删除、改名”这类事件能通知到所有打开者。
为什么要有：名字解析是逐段走树的昂贵操作，系统启动与运行中同一路径会被打开
成千上万次，没有 KCB 缓存，注册表访问会慢到不可接受；它与 hive 的互为前提
关系见表头：hive 的根 cell 要靠 KCB 寻址，KCB 又要指回 hive，启动时先有 hive
结构、再由根 cell 建出第一个 KCB，打破循环的是“根 KCB 由链接过程手工建出，
不是解析出来的”（见第 ⑧ 步）。

**视图（View）**：hive 文件在系统缓存或内存里的一段映射窗口，
文件来源的 hive 不一次性全读进内存，访问哪个 bin 才映射哪个 bin 的视图。
做什么用：大 hive（如 SOFTWARE）不必在启动时全量读盘。为什么要有：
启动早期内存紧张，全量读盘既慢又占内存；视图让 hive 的访问按需发生，
且写盘时只需刷脏了的视图。

**链接节点（Link Node）**：主 hive 里的一种特殊键节点，标志带
`KEY_HIVE_ENTRY | KEY_NO_DELETE`（`CONFIG/CMPARSE.C:1832`），它的作用是
“这个名字下面挂着另一个 hive”。做什么用：把一个 hive 的根接到名字空间的
某个路径上。为什么不用普通键：普通键的子键还在本 hive 里，链接节点的子键
在另一个 hive 里；解析走到链接节点必须切换 hive 指针与根 cell，这个切换语义
只有带标志的节点才能触发，普通键触发不了。

**符号链接键（Symbolic Link Key）**：另一种特殊键，带 `REG_OPTION_CREATE_LINK`
选项创建，值 `SymbolicLinkValue` 存目标路径（类型 `REG_LINK`）。
做什么用：`CurrentControlSet` 指向 `ControlSet001` 这种“名字指向名字”的别名。
为什么要有：链接节点挂的是“一个 hive”，符号链接挂的是“一条路径”，
两者解决的问题不同——前者拼存储单位，后者做运行时的名字别名（见第 ⑨ 步）。

**控制集（ControlSet）**：SYSTEM hive 里 `ControlSet001`、`ControlSet002` 这样的
子树，每套是一份完整的系统配置（服务、驱动、硬件配置）。做什么用：
让系统能保存多套配置并在启动时选一套。为什么要有：上次启动成功的配置
（LastKnownGood）必须留着，当前正在试的配置失败时还能退回去；
如果只有一套配置，一次坏的驱动安装就能让机器再也起不来。

**Select 键**：SYSTEM hive 里 `Select` 键下的四个 DWORD 值：Current（本次启动
用的控制集号）、Default（默认控制集号）、LastKnownGood（上次成功启动的控制集号）、
Failed（上次失败的控制集号）。做什么用：启动时选控制集的唯一依据。
为什么要有：控制集本身只是几棵并列的子树，没有 Select，内核不知道该用哪一棵；
Select 是加载器与内核约定的“选票箱”，加载器写 Current，内核读 Current。

### 五、写盘与一致性

**脏位图（Dirty Vector）与脏计数（DirtyCount）**：HHIVE 里的一张位图，
每位对应 hive 文件里的一个扇区（按簇扩到整簇），写了某个 cell 就把对应位置位，
脏计数是置位位的总数。做什么用：刷盘时只写脏了的部分。为什么要有：
hive 文件可能几十 MB，每次改一个值就全量写盘不可接受；位图让刷盘
只写改过的扇区，且日志也只需要记这些扇区。

**日志文件（.LOG）**：hive 主文件同名加 `.LOG` 后缀的文件，内容是：
日志头 + 脏位图 + 所有脏扇区的数据。做什么用：预写日志（Write-Ahead Log），
主文件写坏时靠它恢复。为什么要有：主文件是原地改写，写一半崩溃会得到
半新半旧的文件；日志先写、主文件后写，崩溃后只要日志完整，就能把
主文件恢复到“这次写盘之前或之后”的干净状态，不会停在中间态。

**懒刷（Lazy Flush）**：hive 脏了之后不立即写盘，而是设一个 5 秒定时器，
到时由工作线程把脏 hive 刷盘的机制。做什么用：把高频小修改合并成
低频批量写盘。为什么要有：注册表写操作非常频繁（服务、驱动、应用都在写），
每次写都同步落盘会把磁盘拖垮；5 秒的延迟换来批量合并，且崩溃最多丢
5 秒内的修改——对配置数据这是可接受的代价，对关键数据调用方会显式
调 `FlushKey` 强制同步刷盘。

**显式刷盘（FlushKey）**：调用方主动要求“现在就把这个 hive 写盘”的操作，
绕过懒刷定时器、同步完成。做什么用：关键修改（如安装完成、关机前）的
落盘保证。为什么要有：懒刷是“尽力而为”，关机、安装完成这类时刻
不能赌 5 秒后还在，必须有同步路径。

**自愈（Self Heal）**：加载器发现 SYSTEM hive 损坏时尝试修复的机制，
基块 BootType 里带 `HBOOT_SELFHEAL` 位（`INC/HIVEDATA.H`）时表示
加载器已经做过修复。做什么用：hive 损坏时不直接蓝屏，给一次修复机会。
为什么要有：SYSTEM hive 坏了机器就起不来，加载器在内核起来之前是
唯一能动它的人；但内核不能盲目信任修复结果，所以 `CmpInitializeSystemHive`
里有检查：自愈已禁用而加载器却报告自愈过，就蓝屏（见第 ⑤ 步）。

### 六、沿用术语（前批已定义，本批正文直接用）

对象管理器（第 4 批）、对象类型与解析例程（第 4 批）、句柄（第 4 批）、
安全描述符（第 4 批）、池（分页池/非分页池，第 1、3 批）、IRQL 与
分发级别/APC 级别（第 1、5 批）、自旋锁与排队自旋锁（第 2、5 批）、
ERESOURCE 资源锁与 pushlock（第 2 批）、工作项与延迟工作队列（第 5 批）、
DPC（第 5 批）、系统进程与 System 线程（第 5 批）、配额（第 4 批注记）。
本批新出现的 IRQL 语义只有一处：`CmpInitializeHive` 在传了文件句柄时断言
当前级别是普通级别（`ASSERT_PASSIVE_LEVEL`），因为它要调 `ZwQueryVolumeInformationFile`
查卷扇区大小——文件操作只能在普通级别做。

### 七、其他新词与缩写（正文直接用）

**DMA（直接内存访问）**：设备不经过 CPU、直接读写内存的机制。
本批只出现在硬件资源描述符的枚举里（中断、DMA、端口、内存四类资源），
不展开讲——它是固件枚举结果的一部分，不是 CM 的机制。

**SM（会话管理器）**：即 `smss.exe`，内核起来后第一个用户态进程，
负责跑 autocheck、建页面文件、调 `NtInitializeRegistry` 情况 1、
再启动 CSRSS 与 Winlogon。本批只在第 ⑫ 步出现。

**SC（服务控制器）**：即服务控制管理器，启动被接受（用户登录成功、
启动被确认为好启动）后调 `NtInitializeRegistry` 情况 2，
把本次控制集存为 LastKnownGood。本批只在第 ⑫、⑬ 步出现。

**ACL（访问控制列表）**：安全描述符里“谁能对这个对象做什么”的表。
第 4 批已讲安全描述符，ACL 是它的核心成分；本批第 ③ 步
“键对象的安全检查是必需的”指的就是查 ACL。

**QA**：本批笔记“问答补记区”的简称，沿用第 2 批起的惯例：
正文讲不完、答疑中补清的结论记在那里。

**WRK**：Windows Research Kernel，即本学习线的底本（WS03 SP1 代码线）。

**NT**：Windows NT 的简称；本批代码线是 NT 5.2（Windows Server 2003 SP1）。

**XP**：Windows XP 的简称；`HSYS_MINOR_SUPPORTED` 的值 5 对应
Whistler（XP 的开发代号）时期的 hive 格式。

**IO / MM / OB / PS**：四种子系统的缩写，分别指 I/O 系统、
内存管理器（Memory Manager）、对象管理器（Object Manager）、
进程管理器（Process Manager，含线程与调度）。正文用缩写时均指
第 2、3、4、5 批已讲过的对应子系统。

## 第二部分：正文

### ① 位置即机制：为什么 CmInitSystem1 卡在缓存管理器之后、I/O 之前

`CmInitSystem1` 的调用点在 `INITOS.C:1437`，前一步是 `CcInitializeCacheManager`（`:1426`），
后一步是 `IoInitSystem`（`:1531`）。这个顺序不是随意排的，三条理由钉死它：

第一，I/O 系统点火要读注册表。`IoInitSystem` 里要按 `Services` 子树装 boot 驱动
（第 7 批的主题），读不到 `Services` 就不知道装什么驱动；所以注册表必须在 I/O 之前可用，
`NtCreateKey`、`NtOpenKey`、`NtQueryValueKey` 在 `CmInitSystem1` 返回后就能用。

第二，文件系统还没起来。`IoInitSystem` 之后才有磁盘驱动与文件系统，
`CmInitSystem1` 期间根本打不开 `\SystemRoot\config\system` 这个文件；
所以第一阶段的 SYSTEM hive 只能来自加载器内存映像，写操作也只能留内存。
`CmInitSystem1` 的函数头注释原话就是：“After this call, Nt*Key calls work, but only
part of the name space is available and any changes written must be held in memory.”
（调用返回后 Nt*Key 可用，但名字空间只有一部分，写下去的修改必须留在内存里。）

第三，缓存管理器必须在前。hive 的文件来源版本（第二阶段）要走系统缓存映射视图，
缓存管理器是视图机制的前提；虽然第一阶段的 SYSTEM hive 还没用到缓存视图，
但 CM 的初始化代码（视图链表、缓存初始化 `CmpInitializeCache`）已经按“缓存管理器
已存在”来写。顺序反过来，CM 初始化就会踩到未初始化的缓存结构。

### ② CmInitSystem1 的前置工作：先搭 CM 自己的基础设施，再碰任何 hive

`CmInitSystem1`（`CONFIG/CMSYSINI.C:311`）在拿注册表锁之前，先做一串不依赖
任何 hive 的初始化，顺序是：

- `CmpInitializeRegistryNames`：把 `\Registry\Machine`、`\Registry\User` 等路径
  字符串初始化成全局 Unicode 串。为什么先做：后面建节点、链接 hive 时处处要用到
  这些名字，集中初始化一次比每次现拼更安全，也避免在拿锁之后做可能失败的分配。
- `CmpComputeGlobalQuotaAllowed`：算注册表全局配额上限。为什么在此时算：
  配额是“注册表总共能占多少池”的天花板，后面建 hive、建 KCB、写 cell 都要扣配额，
  天花板必须在第一次分配之前立好，否则早期的分配会绕过配额。
- `InitializeListHead(&CmpHiveListHead)` 与两个 pushlock（`CmpHiveListHeadLock`、
  `CmpLoadHiveLock`）：hive 链表是所有已装载 hive 的登记簿，第二阶段的并行装载线程
  与懒刷工作线程都要查它，链表头必须在第一个 hive 登记之前存在。
- `ExInitializeResourceLite(&CmpRegistryLock)`：注册表全局资源锁（ERESOURCE）。
  为什么用 ERESOURCE 而不用自旋锁：注册表操作经常要等待（等文件 I/O、等视图映射），
  自旋锁的持有者不能等待，ERESOURCE 可以共享/独占、可以等待，正好匹配
  “读多写少、操作耗时”的注册表访问模式。锁的粒度见第三部分单核 / 多核对照。
- 缓存（`CmpInitializeCache`）、私有分配器（`CmpInitCmPrivateAlloc` 与
  `CmpInitCmPrivateDelayAlloc`）、延迟解引用 KCB 引擎（`CmpInitDelayDerefKCBEngine`）、
  回调模块（`CmpInitCallback`）、自愈工作项队列与保护互斥。
  为什么 CM 要有私有分配器：hive 的 cell 分配有特殊的对齐与记账要求
  （簇对齐、脏跟踪、配额），直接调通用池分配会丢这些语义，所以 CM 在池之上
  再包一层自己的分配器；延迟解引用 KCB 引擎则是性能机制——KCB 引用计数归零时
  不立即释放，攒一批再释放，避免高频打开关闭同一路径时的反复分配。
- `CmpSystemProcess = &PsGetCurrentProcess()->Pcb`：把当前进程（此时是 System 进程，
  见第 5 批）记下来。为什么：CM 的工作线程、懒刷线程、文件操作都要在某个进程
  上下文里跑，System 进程是内核启动后唯一长期存在的进程锚点，后面凡要
  “挂靠到系统进程”都用它；如果不记，工作线程起来时连自己属于哪个进程都不知道。
- `CmpLockRegistryExclusive()`：拿注册表全局锁的独占模式，整个 `CmInitSystem1`
  的后半段都在这把锁里。为什么是独占而不是共享：点火期间是在“建”名字空间，
  不是“读”名字空间，任何并发的读都会读到半建成的结构，独占是唯一安全的模式。

### ③ 注册 Key 对象类型：CM 向对象管理器登记“键”这种对象

`CmpCreateObjectTypes`（`CONFIG/CMSYSINI.C:915`）在 `CmInitSystem1` 里
第一个可能蓝屏的步骤（失败则 `CONFIG_INITIALIZATION_FAILED`，参数 1）。
它调 `ObCreateObjectType` 登记类型名 `Key`，登记内容是：

- 通用权限映射：`KEY_READ`、`KEY_WRITE`、`KEY_EXECUTE`、`KEY_ALL_ACCESS`
  四档；键对象的默认池开销是 `CM_KEY_BODY` 的大小；安全检查是必需的
  （`SecurityRequired = TRUE`），因为注册表项有 ACL。
- 关闭例程 `CmpCloseKeyObject`、删除例程 `CmpDeleteKeyObject`、
  解析例程 `CmpParseKey`、安全例程 `CmpSecurityMethod`、查名例程 `CmpQueryKeyName`。

这里有一个第 4 批留下的结要当场解开：对象管理器在 Phase 0 就建好了类型机制，
为什么 `Key` 类型拖到 Phase 1 的 CM 里才注册？结的形成原因是双向依赖——
对象管理器需要类型才能解析名字，CM 需要对象管理器才能把键挂进名字空间；
打破循环靠的是“机制与语义分离”：第 4 批 OB Phase 0 注册的是通用的类型机制
（类型对象本身、类型链表、解析回调的槽位），不依赖任何具体类型的语义；
CM 在 Phase 1 再把“键的解析语义”（`CmpParseKey`，含链接节点切换、符号链接
跟随、KCB 查找）填进这个槽位。打破后的局面是：OB 只管“按路径找对象”，
CM 才管“路径在注册表里是什么意思”，两层各司其职，`Key` 类型之后就可以
像其他对象类型一样被打开、关闭、做安全检查。

### ④ 主 hive：先建一个只活在内存里的骨架

`CmpInitializeHive(&CmpMasterHive, HINIT_CREATE, HIVE_VOLATILE, HFILE_TYPE_PRIMARY, ...)`
（`CmInitSystem1` 内 `:476`，紧接对象类型注册之后）建主 hive。三个参数各有为什么：

- `HINIT_CREATE`（从零新建）：主 hive 没有磁盘文件、没有内存映像可抄，
  它的全部内容就是后面要建的几个根节点与链接节点，只能从零新建。
  新建时基块签名写 `regf`、两个序列号都置 1、根 cell 置 `HCELL_NIL`
  （`CONFIG/HIVEINIT.C:348` 一带），表示“空 hive，等后面填”。
- `HIVE_VOLATILE`（整 hive 易失）：主 hive 只装骨架与链接节点，
  这些节点的唯一价值是“指路”，下次启动时 CM 会重新建一遍一模一样的骨架；
  落盘保存它们既无意义（重建更便宜），又有害（盘上的旧链接可能指向
  已经不存在的 hive）。所以刷盘逻辑看到易失 hive 直接跳过（`HIVESYNC.C`
  的 `HvSyncHive` 里 `HIVE_VOLATILE` 直接返回成功）。
- `HFILE_TYPE_PRIMARY` 且无日志：主 hive 不写盘，日志也就无从谈起；
  `CmpInitializeHive` 的参数校验里明文拒绝“易失 hive 又传文件句柄”
  的组合（除 WinPE 共享模式外），就是怕调用方在“不落盘”与“给了文件”
  之间自相矛盾。

主 hive 建好后，`CmpCreateRegistryRoot` 在它里面建 `\Registry` 根节点，
再用两次 `NtCreateKey` 建 `\Registry\Machine` 与 `\Registry\User`。
注意这里已经在用 `NtCreateKey`——CM 在点火中途就开始用自己刚注册的
对象类型与刚建的解析路径给自己建节点，这是自举的第二环：
对象类型（第 ③ 步）→ 主 hive（本步）→ 根节点 → 后面所有 hive 的链接点。
`CmpNoMasterCreates` 在 `CmInitSystem1` 末尾置 `TRUE`，之后再想往主 hive 里
随意建节点会被挡住：骨架一旦封顶，就不允许业务代码往里面乱挂节点，
否则名字空间的骨架会被越建越乱。

### ⑤ SYSTEM 接管之一：从加载器手里接过映像

`CmpInitializeSystemHive`（`CONFIG/CMSYSINI.C:1574`）是本批的核心，
分四段：

第一段，捕获启动选项。`LoaderBlock->LoadOptions` 是加载器命令行尾巴
（如 `/SAFEBOOT`、`/NOGUIBOOT` 这类），先转成 Unicode 存进全局
`CmpLoadOptions`，后面 `CmpSetSystemValues` 会把它写进注册表。
为什么在这里捕获：`LoaderBlock` 在 Phase 1 尾会被释放（`INITOS.C:1615-1621`
的 `MmFreeLoaderBlock`，第 3 批已讲），不现在拷走就再也拿不到；
而且启动选项是“本次启动”的事实，必须在注册表可用之前就钉死，
不能等第二阶段再从别处找。

第二段，拿映像指针：`HiveImageBase = LoaderBlock->RegistryBase`。
这是加载器与内核约定的交接字段：加载器把 `system` 文件整段读进内存后，
把基地址写在这里。内核不问文件在哪、怎么读的，只认这个指针。
如果指针为空（加载器没给 hive，比如某些修复、安装场景），就走
`HINIT_CREATE` 新建一个空 SYSTEM hive（`HFILE_TYPE_LOG`，即带日志），
后面再由第二阶段从文件补——空 hive 不是错误，是“加载器没交货”时的
合法退路，失败才返回 `FALSE` 让调用方蓝屏。

第三段，`HINIT_MEMORY` 接管。指针非空时调
`CmpInitializeHive(&SystemHive, HINIT_MEMORY, HIVE_NOLAZYFLUSH, HFILE_TYPE_LOG,
HiveImageBase, ..., &CmpSystemFileName, CM_CHECK_REGISTRY_SYSTEM_CLEAN)`。
`HIVE_NOLAZYFLUSH` 是临时的，源码注释把理由写得非常直白：
此时还没有打开 SYSTEM hive 的主文件，如果懒刷工作线程现在就跑，
`CmpFileWrite` 里“文件句柄为空就直接返回成功”的分支会让懒刷
误以为“已经保存”，而实际上一个字节都没写盘；之后若崩溃，
这段时间内对 SYSTEM hive 的修改就丢了。注释还点名这是 5.0 就可能有的问题。
所以旗标的意思是“在文件句柄绑好之前，禁止懒刷报告成功”，
等第二阶段绑好句柄后再清掉（第 ⑭ 步的断言 `ASSERT(HiveFlags & HIVE_NOLAZYFLUSH)`
就是在检查“这个旗标确实还立着，现在由我来清”）。
`HFILE_TYPE_LOG` 表示这个 hive 带日志文件，刷盘时走“先日志后主文件”的顺序
（第 ⑰ 步）。

第四段，启动类型与自愈检查。`CmpBootType = SystemHive->Hive.BaseBlock->BootType`，
把加载器写在基块里的启动类型（正常、修复、备份、自愈）抄到全局。
接着：如果自愈被禁用（`CmSelfHeal` 为假）而加载器却报告了自愈
（`BootType & HBOOT_SELFHEAL`），直接蓝屏 `BAD_SYSTEM_CONFIG_INFO`。
为什么不能“将就着用”：自愈意味着加载器已经动过 hive 内容，
内核若不信任这次修复却继续用，等于在可能已损坏的配置上启动驱动，
后果比蓝屏更糟（驱动按错误配置加载可能毁数据）；蓝屏是“停下来、
让人修 hive”的诚实选择。

### ⑥ SYSTEM 接管之二：HINIT_MEMORY 在底层到底做了什么

`CmpInitializeHive`（`CONFIG/CMINIT.C:511`）先做 CM 层的活：
分配 `CMHIVE`、初始化 hive 锁（pushlock）、写者锁、视图锁（保护互斥）、
安全缓存、视图链表、文件句柄数组（主文件、日志、外部文件三槽），
然后调 `HvInitializeHive`（`CONFIG/HIVEINIT.C:115`）做 hive 引擎层的活。
`HvInitializeHive` 对 `HINIT_MEMORY` 的分支（`HIVEINIT.C` 的
“HINIT_MEMORY / memory copy case”）按这个顺序执行，缺一不可：

1. 验基块。签名必须是 `regf`、类型必须是主文件、格式必须是内存格式、
   主版本必须等于 1（`HSYS_MAJOR`）、次版本不得高于支持的上限
   （`HSYS_MINOR_SUPPORTED`，即 Whistler/XP 的 5）、基块校验和必须重算一致。
   任一不符返回 `STATUS_REGISTRY_CORRUPT`。为什么主版本必须“等于”
   而次版本只要求“不高于”：主版本变了意味着文件格式换代，旧引擎读新格式
   会误读字段，只能拒绝；次版本是向后兼容的增量（新字段只追加在后），
   旧引擎读新次版本顶多看不懂新字段，不会误读旧字段，所以放行。
   注意一个与 `HINIT_MEMORY_INPLACE` 的区别：原地模式还要求
   `Sequence1 == Sequence2`，拷贝模式不查这一项——树内没有注释解释
   这个差别，原因列入未钉死清单，不编造。
2. 新建基块并拷头。按簇对齐分配一块新基块（不对齐就改分配一页），
   只拷第一个扇区（`HSECTOR_SIZE` = 512 字节），再把原基块的
   `BootRecover` 与 `BootType` 单独拷过来。为什么只拷 512 字节：
   基块逻辑上就是 4KB 块里的第一个扇区，后面是保留区，拷 512 字节
   就够了；为什么 `BootRecover`/`BootType` 要单独拷：它们是加载器
   与内核之间的“带外消息”，不在“拷头”的常规语义里，必须显式带过来，
   否则第 ⑤ 步的自愈检查与第 ⑭ 步的“加载器已恢复则全量刷盘”都会丢信号。
3. 建空闲表与两级映射。`HvpAdjustHiveFreeDisplay` 按基块的 `Length`
   建稳定存储的空闲显示表，再 `HvpBuildMapAndCopy` 从映像基块之后
   （`HiveData + HBLOCK_SIZE`）开始，把每个 bin 拷进 CM 分配的池内存，
   并建好 cell 索引到内存地址的两级映射。
   为什么是“拷”而不是“原地用”（这正是 `HINIT_MEMORY` 与
   `HINIT_MEMORY_INPLACE` 的分水岭）：加载器内存是借来的，
   Phase 1 尾 `MmFreeLoaderBlock` 会把它还给内存管理器；CM 若原地用，
   释放后就成了野指针。拷进自己的池存储后，加载器内存爱什么时候
   释放都行。原地模式只给加载器自己用（`HIVEINIT.C` 注释原话
   “[Used by osloader.]”），内核不用。
4. 回到 `CmpInitializeHive`，对拷贝、文件、映射三种来源统一做
   `CmCheckRegistry`（`CMINIT.C:760` 一带）。校验失败则拆掉刚建的
   视图链表、安全缓存、hive 视图，返回 `STATUS_REGISTRY_CORRUPT`。
   校验内容见第 ⑦ 步。
5. 登记进 hive 链表。`CmpInitializeHive` 末尾把新 hive 插进
   `CmpHiveListHead`（`CONFIG/CMINIT.C:794`），源码注释写明插在末尾
   是为了让 SYSTEM hive 在懒刷迭代里排在前面：主 hive 虽然先登记，
   但它是易失的、刷盘时跳过，所以懒刷扫链表时第一个真正要刷的
   就是 SYSTEM——最关键的 hive 最先落盘。

### ⑦ 校验：CmCheckRegistry 为什么要清易失信息

`CmCheckRegistry`（`CONFIG/CMCHEK.C:111`）从根 cell 开始整树走一遍：
每个键的签名、子键计数与子键列表是否对得上、值列表是否合法、
子键是否按字典序排列（`CmpCheckLexicographicalOrder`），
不合法就返回非零，调用方按损坏处理。
它的第二个职责更重要：按传入的 `CheckFlags` 清除易失信息。
`CMCHEK.C:849` 一带的逻辑是：如果要求强制清除，或者要求检查清除
且该键确实存有上一启动残留的易失子键，就把这些易失信息清掉。
为什么上一启动的易失信息必须清：易失 cell 存的是“本次启动现算”的东西
（例如某些驱动临时写的状态），上一启动的易失项指向的是上一启动的
设备、进程、状态，本次启动重用它们等于把旧世界的快照当成新世界的事实；
而稳定 cell 不受影响——落盘的配置本来就是要跨启动复用的。
所以校验不只是“查错”，还是“把不属于新世界的残留清掉”，
SYSTEM hive 用 `CM_CHECK_REGISTRY_SYSTEM_CLEAN` 正是要这个语义。

### ⑧ 链接入主 hive：为什么 SYSTEM 不是“拷进”主 hive，而是“挂”上去

`CmpLinkHiveToMaster`（`CONFIG/CMSYSINI.C:1267`，`CmpInitializeSystemHive` 内调它，
路径名 `\Registry\Machine\System`，`Allocate = FALSE`）做三件事：

1. 填解析上下文。`ParseContext.CreateLink = TRUE`，
   `ChildHive.KeyHive = &SystemHive->Hive`，
   `Allocate` 为假时 `ChildHive.KeyCell = BaseBlock->RootCell`
   （为真时置 `HCELL_NIL`，表示“根节点还没建，链接时现建”；
   SYSTEM 走为假分支，因为它的根 cell 在映像里已经存在）。
2. 调 `ObOpenObjectByName`，对象类型是 `CmpKeyObjectType`，名字是链接名，
   要求“除末段外都存在、末段不存在”。末段不存在是硬性要求：
   链接点必须是新名字，挂到已存在的名字上会歧义（到底是打开旧键还是挂新 hive）。
3. 解析走到 `CmpCreateLinkNode`（`CONFIG/CMPARSE.C:1661`）：
   先断言链接节点只能建在主 hive 里（在非主 hive 里建链接节点直接
   `STATUS_ACCESS_DENIED`），再在主 hive 里分配一个键节点，
   标志置 `KEY_HIVE_ENTRY | KEY_NO_DELETE`，节点里藏子 hive 的指针与根 cell。
   `KEY_NO_DELETE` 的意思是这个节点不能被删——删了它，整棵子 hive
   就从名字空间里丢了，但 hive 文件还在，成了孤儿；删 hive 必须走
   卸载路径（先解链接、再释放 hive），不能靠删节点。

为什么用链接而不是把 SYSTEM 树拷进主 hive，三个理由：
第一，存储单位独立。SYSTEM hive 有自己的脏位图、日志、文件句柄，
拷进主 hive 就丢了“按 hive 刷盘、按 hive 恢复”的全部能力。
第二，生命周期独立。SYSTEM hive 可以在第二阶段换轨（绑文件、清旗标），
用户 hive 可以随登录登出，拷进主 hive 后这些操作都没有着落点。
第三，名字解析的切换语义。解析走到链接节点时，解析器把当前 hive 指针
换成子 hive、把当前 cell 换成子 hive 的根 cell，后续路径段都在子 hive 里走；
这个切换只有链接节点能触发，普通键做不到——这就是第 ③ 步注册 `Key` 类型时
`CmpParseKey` 里要处理的语义，也是“主 hive 只装骨架”的全部含义。

链接成功后，`CmpLinkHiveToMaster` 把 hive 的 `DirtyFlag` 清掉
（刚接管的 hive 在内存里的状态就是干净起点），再发一次名字变更通知，
最后把 `SystemHive` 登记进 `CmpMachineHiveList[SYSTEM_HIVE_INDEX].CmHive`，
供第二阶段的线程认领。

### ⑨ CurrentControlSet：符号链接为什么是易失的

`CmpCreateControlSet`（`CONFIG/CMSYSINI.C:2435`，`CmInitSystem1` 内在
SYSTEM 链接之后调）只做一件事：按 `Select` 的值建两个符号链接。

第一步，读 `Select`。打开 `\Registry\Machine\System\Select`，
查 `Current` 值（DWORD），得到控制集号 N。
第二步，建链接键。`NtCreateKey` 建 `\Registry\Machine\System\CurrentControlSet`，
选项是 `REG_OPTION_VOLATILE | REG_OPTION_CREATE_LINK`（易失 + 创建链接），
并断言返回的处置必须是 `REG_CREATED_NEW_KEY`（新建，不是打开）——
断言的理由写在注释里：这个键永远是易失的，hive 文件里不可能有它，
如果打开到了旧的，说明启动状态已经错了。
第三步，写目标。往这个键写 `SymbolicLinkValue` 值，类型 `REG_LINK`，
内容是 `\Registry\Machine\System\ControlSetNNN`（N 格式化成三位数）。
第四步，同理建 `CurrentControlSet\Hardware Profiles\Current` 指向
`ControlSetNNN\Hardware Profiles\NNNN`，N 由 `Control\IDConfigDB` 的
`CurrentConfig` 值决定。

为什么 CurrentControlSet 必须是符号链接、且必须是易失的，两个为什么：
符号链接而不是复制整棵 ControlSet，是因为控制集可能有好几套并存
（Current、LastKnownGood、Default、Failed），驱动与服务读到的
“当前配置”必须永远是同一份数据；复制会在“改了 ControlSet001 但
CurrentControlSet 里的副本没改”的那一刻分裂成两份事实，而符号链接
让所有读 CurrentControlSet 的访问都落到同一棵 ControlSet 上，
改一处即改全部。易失而不是落盘，是因为“当前”是启动时选出来的，
落盘会把上次的“当前”钉死成下次的“当前”；每启动按 Select 重建，
选票箱（Select）一改，下次启动的当前就跟着改，这正是
LastKnownGood 机制能工作的前提：启动失败时把 Select 的 Current
指回 LastKnownGood，下次启动就自动用旧的好配置。

### ⑩ HARDWARE hive：为什么它从零建、为什么它易失、数据从哪来

`CmInitSystem1` 在 SYSTEM 之后建 HARDWARE hive：
`CmpInitializeHive(&HardwareHive, HINIT_CREATE, HIVE_VOLATILE, HFILE_TYPE_PRIMARY, ...)`，
再 `CmpLinkHiveToMaster(\Registry\Machine\Hardware, ...)`，`Allocate = TRUE`
（根节点现建，因为新 hive 没有根 cell）。
`HINIT_CREATE` 与 `HIVE_VOLATILE` 的理由与主 hive 相同但更彻底：
HARDWARE 的内容不是从文件来的，是本次启动现算的，所以既无映像可抄，
也无落盘价值。

数据来源是加载器的配置树。`CmpInitializeHardwareConfiguration`
（`CONFIG/CMCONFIG.C:79`）先建 `\Registry\Machine\Hardware\DeviceMap`
与 `\Registry\Machine\Hardware\Description` 两个节点，
再以 `LoaderBlock->ConfigurationRoot` 为根，调 `CmpSetupConfigurationTree`
递归走整棵配置树：每个配置组件（系统、总线、控制器、外设）按
固件枚举时给出的设备路径名与类型写成 Description 下的子键与值，资源描述符
（中断、DMA、端口、内存）拼成 `CM_FULL_RESOURCE_DESCRIPTOR` 写进去。
为什么用加载器树而不是等即插即用枚举：boot 驱动在 PnP 管理器起来之前
就要查“机器上有什么硬件”，等 PnP 枚举完再建 HARDWARE 就晚了；
加载器在进内核之前已经枚举过固件给出的硬件清单，这棵树就是
内核启动早期唯一的硬件事实来源。
之后 `CmpInitializeMachineDependentConfiguration` 再往
`CurrentControlSet\Control` 里写机器相关的配置值，
`CmpSetSystemValues` 把启动选项（第 ⑤ 步捕获的 `CmpLoadOptions`）
与系统版本等值写进注册表，网络启动时 `CmpSetNetworkValue` 再补
网络加载块的值。为什么这些值要在第一阶段就写进注册表：
驱动与系统组件在 I/O 点火后立刻就要读它们，写晚了就读不到；
而写操作此时只留内存（第 ① 步），等第二阶段绑好文件后再由懒刷落盘，
顺序上是“先让读方看到、再让盘方持久化”。

### ⑪ 鸡生蛋总解：注册表点火的三重依赖是怎么解开的

本批涉及三对“看似循环”的依赖，按规范在正文当场解开，不留到问答区：

第一对：注册表依赖对象管理器，对象管理器依赖注册表吗？不依赖。
对象管理器在 Phase 0 只建了类型机制（第 4 批），它解析名字时
不需要注册表；注册表在 Phase 1 借对象管理器的类型机制、名字空间、
句柄表把键挂上去。结形成的原因是“两者都叫系统基础服务”，
容易误以为互相需要；打破循环的是分层：OB 提供“对象”这个通用概念，
CM 提供“键”这个具体语义，通用层不依赖具体层。

第二对：注册表依赖文件系统，文件系统依赖注册表吗？启动阶段不依赖。
文件系统驱动的加载参数确实存在注册表里，但那是 I/O 点火之后的事；
第一阶段的注册表根本不碰文件系统（内存映像 + 暂缓写），
文件系统起来之后，第二阶段才把 hive 绑到文件上。
结形成的原因是“注册表最终要落盘”这个终态印象；
打破循环的是分阶段：先内存态、后文件态，终态不等于初态。

第三对：注册表依赖内存管理器，内存管理器依赖注册表吗？不依赖。
MM 在 Phase 1 之前已经上线（第 3 批），池、视图、页表都已可用，
CM 的池分配、bin 拷贝、视图映射都建立在 MM 已上线的前提上；
MM 的启动参数来自加载器内存描述符，不来自注册表。
三对解开后，点火链的真实顺序就是：MM（第 3 批）→ OB（第 4 批）
→ PS（第 5 批）→ CM 本批 → IO（第 7 批），每一步只依赖前面已上线的服务。

### ⑫ 第二阶段之一：NtInitializeRegistry 为什么要等 SM

`NtInitializeRegistry`（`CONFIG/NTAPI.C:1030`）的函数头注释写明它只在
两种情况下被调：SM 在 autocheck 跑完、页面文件打开之后调它做情况 1
（把内存 hive 绑到文件、打开其余 hive 文件）；SC 在当前启动被接受之后
调它做情况 2（把本次启动用的控制集存为 LastKnownGood）。
两种情况各有“只做一次”的守卫：情况 1 靠 `CmFirstTime`，
第二次调直接 `STATUS_ACCESS_DENIED`；情况 2 靠 `CmBootAcceptFirstTime`，
接受过一次就不再接受。

为什么情况 1 必须等 autocheck 与页面文件，三个理由：
第一，文件系统与磁盘驱动在 I/O 点火后才就绪，`CmInitSystem1` 时
根本打不开 `\SystemRoot\config\system`，硬要在那时绑文件只会失败。
第二，autocheck（chkdsk）可能要修盘，修盘期间写注册表文件
会与修盘冲突，修完之前写进去的数据可能被修盘操作覆盖或作废；
等修完再绑，写盘的起点就是修好的干净文件系统。
第三，页面文件打开之后，分页池的后备才完整，hive 的缓存映射视图
（分页池 + 系统缓存）才能正常工作；在此之前绑文件，视图机制的
后备存储是不完整的。
所以 `NtInitializeRegistry` 的入口先做两件事：若调用方是用户模式，
转 `ZwInitializeRegistry` 强制以内核模式重进（注册表点火是内核特权操作，
不能由用户模式直接驱动）；校验 `BootCondition` 在合法范围内，
再按情况 1/情况 2 分流。情况 1 的核心是 `CmpLockRegistryExclusive` 后调
`CmpCmdInit(SetupBoot)`（见第 ⑬ 步），再 `CmpSetVersionData` 写版本数据、
`PoInitHiberServices` 通知电源管理器卷已可用。

### ⑬ 第二阶段之二：CmpCmdInit 与并行装载 6 个机器 hive

`CmpCmdInit`（`CONFIG/CMWORKER.C:224`，由 `NtInitializeRegistry` 情况 1 调用）做四件事：

1. 初始化懒刷定时器、DPC、工作项（`KeInitializeDpc(&CmpLazyFlushDpc,
   CmpLazyFlushDpcRoutine)`、`KeInitializeTimer`、`ExInitializeWorkItem`），
   以及一个 600 秒（`ENABLE_LAZY_FLUSH_INTERVAL_IN_SECONDS`，
   `CONFIG/CMWORKER.C:54`）后强制放行懒刷的定时器与 DPC。
   为什么 600 秒强制放行：启动被接受之前，懒刷是被 `CmpHoldLazyFlush`
   压住的（怕启动失败时把半成品配置写盘）；但如果启动一直不被接受
   （例如没人登录确认），压住懒刷会让所有修改永远不落盘，
   600 秒是“再不放行就永远不写盘”的兜底。
2. `CmpNoWrite = CmpMiniNTBoot`：WinPE（MiniNT）模式下继续禁写，
   正常模式下清掉禁写——从这一刻起，写盘路径才真正打开
   （第 ① 步的“只留内存”到此结束）。
3. 非安装启动时调 `CmpInitializeHiveList`（`CONFIG/CMSYSINI.C:687`）。
   安装启动（SetupBoot）跳过它，因为安装时的注册表布局与正常启动不同。
4. MiniNT 且共享系统 hive 时清掉共享标志——共享是 WinPE 远程启动
   时的临时措施，正常初始化完成后不再需要。

`CmpInitializeHiveList` 的并行装载是本批最值得细看的一段：
它对机器 hive 表里的 6 个 hive 各起一个系统线程（`PsCreateSystemThread`
调 `CmpLoadHiveThread`，参数就是表下标 i），然后主线程在
`CmpLoadWorkerEvent` 上等全部线程完成（最后一个完成的线程
`InterlockedIncrement` 计数到 6 时置事件）。为什么并行：
6 个 hive 文件互相独立，串行读盘要等 6 次磁盘往返，
并行读盘只要等最慢的那一次；又因为每个线程只碰自己的 hive，
并行不需要跨 hive 的锁，只有最后的“链接”步骤由主线程串行做
（链接要改主 hive 的名字空间，不能并行）。
主线程等齐后逐个处理：对线程里还没链接的 hive（`CmHive` 为空、
`CmHive2` 为新 hive 的情况），按表里的 `BaseName` 与 `Name` 拼出
注册表路径与文件路径，调 `CmpLinkHiveToMaster` 链接；
若该 hive 是新建的（`Allocate == TRUE`，即文件不存在、新建的空 hive），
立即调 `HvSyncHive` 把它写盘——新建的 hive 必须立刻有文件，
否则下次启动又要新建一次。最后补两个链接：
`CmpLinkKeyToHive(L"\\Registry\\Machine\\Security\\SAM",
L"\\Registry\\Machine\\SAM\\SAM")`（SECURITY hive 里的 SAM 键
指向 SAM hive 里的 SAM 键，因为安全子系统读 SAM 要走 SECURITY 的路径），
以及 `S-1-5-18`（本地系统账户的安全标识符）指向 `.Default`
（系统账户的用户配置就是默认用户配置，两者是同一份数据，
链接比复制更能保证一致）。最后 `CmpCreatePerfKeys` 建性能计数器键，
`CmpNoMasterCreates` 置 `TRUE` 封顶主 hive。

### ⑭ 第二阶段之三：SYSTEM hive 的换轨（从内存像到文件绑定）

6 个并行线程里，SYSTEM 线程走的分支与其他 5 个不同，因为它的
`CmHive` 在第一阶段已经存在（`CmpMachineHiveList[SYSTEM_HIVE_INDEX].CmHive`
在第 ⑧ 步就登记了）。`CmpLoadHiveThread`（`CONFIG/CMSYSINI.C:3792`）
对已存在且非易失的 hive 做“换轨”，顺序是：

1. 开文件。`CmpOpenHiveFiles` 打开主文件与 `.LOG` 文件，
   两个句柄缺一不可：日志句柄为空就报硬错
   `STATUS_CANNOT_LOAD_REGISTRY_FILE`，再蓝屏
   `BAD_SYSTEM_CONFIG_INFO`（参数 `BAD_HIVE_LIST`）。
   为什么日志句柄缺一不可：SYSTEM hive 是带日志的 hive，
   没有日志就无法保证“先日志后主文件”的写盘顺序，
   换轨后写盘会失去崩溃恢复能力；源码注释的原话是
   “我们刚刚告诉用户，像 SYSTEM hive 这样的关键东西坏了，
   不要继续跑，我们只会冒毁掉用户数据的风险。”
2. 绑句柄。把主文件句柄与日志句柄写进 `CmHive->FileHandles` 数组。
   从这一刻起，`CmpFileWrite` 的“句柄为空返回成功”分支不再触发，
   第 ⑤ 步立的 `HIVE_NOLAZYFLUSH` 旗标才可以清。
3. 缓存转换准备。`CmpSetupPrivateWrite` 把 hive 的缓存流标成
   私有写（PRIVATE_WRITE），注释原话是“下一次刷盘会做真正的转换”：
   内存像（池存储）到缓存映射视图（文件存储）的转换不在换轨时
   一次做完，而是在下一次刷盘时完成。为什么分两步：
   换轨时文件刚打开，立即全量转换要把整个 hive 重新映射一遍，
   既耗时又要在注册表锁里做重活；先绑句柄、让懒刷能写盘，
   转换留给刷盘时按需做，启动路径更短。
4. 清旗标。断言旗标确实还立着（`ASSERT(HiveFlags & HIVE_NOLAZYFLUSH)`），
   再清掉它。从这一刻起，SYSTEM hive 的修改才真正能落盘。
5. 簇大小修正。内存像时期簇大小假设为 1（没有卷信息），
   现在从卷拿到了真实簇大小，若不同，就把脏位图里“含脏扇区的簇”
   整个置脏、重算脏计数。为什么必须扩到整簇：脏位图的单位是簇，
   一个簇只能是“全脏”或“全干净”，不能是“半脏”；内存像时期
   按扇区记的脏，换到簇粒度后，含脏扇区的簇必须整体算脏，
   否则刷盘会漏写同簇里其他扇区的修改。
6. 文件大小对齐。把主文件大小设成内存像长度 + 一个块
   （`Storage[Stable].Length + HBLOCK_SIZE`，块是基块本身占的 4KB）。
   为什么加一个块：基块不算在 `Length` 里（基块的 `Length` 字段
   注释原话是“Includes all but header”），文件总长 = 基块 + 数据区。
7. 启动恢复处理。若基块的 `BootRecover` 非零（加载器做过 hive 恢复），
   把脏位图全部置位、脏计数设成位图大小，立即调 `HvSyncHive`
   全量刷盘。为什么：加载器恢复的结果只在内存里，磁盘上的主文件
   还是坏的；不立即全量刷盘，下次启动又要恢复一次，且恢复结果
   可能因后续修改而丢失，全量刷盘是把“恢复结果”钉死成磁盘事实。
8. SYSTEM 专属收尾。`CmpMarkCurrentValueDirty` 把 `Select!Current`
   值标脏——加载器可能改过 Current（例如上次启动失败后改指
   LastKnownGood），这个改动必须保留到刷盘，不能被“换轨后
   状态干净”的假设吃掉。

### ⑮ 文件装载与日志恢复：其余 hive 怎么从文件起来

其他 5 个 hive（SECURITY、SOFTWARE、DEFAULT、SAM，以及
HARDWARE 不走这条路——它是易失的，第一阶段已建好）由
`CmpInitHiveFromFile`（`CONFIG/CMSYSINI.C:2011`）从文件装载：

1. 开文件。`CmpOpenHiveFiles` 打开主文件与 `.LOG`，
   有日志则文件类型为 `HFILE_TYPE_LOG`，无日志则为 `HFILE_TYPE_PRIMARY`。
2. 定操作。若主文件是新建的（`FILE_CREATED`），操作为 `HINIT_CREATE`、
   `Allocate` 置真；否则操作为 `HINIT_MAPFILE`（缓存映射视图），
   若映射方式失败（`STATUS_RETRY`），重试一次 `HINIT_FILE`
   （直接读文件，不走缓存映射）。为什么有两档：缓存映射视图
   是快路径（按需映射、不全量读盘），但某些卷或状态下映射会失败，
   直接读文件是慢但稳的退路；重试逻辑保证“映射失败 ≠ 启动失败”。
3. 调 `CmpInitializeHive`，引擎层对 `HINIT_FILE` 走 `HvLoadHive`：
   读基块、校验签名与版本、检查序列号；若 `Sequence1 != Sequence2`，
   说明上次写盘没写完，就用日志恢复：日志里有头、脏位图、脏扇区数据，
   按脏位图把脏扇区写回内存像，恢复完成后立即把恢复好的像写回主文件、
   清脏位图、把日志文件截断成 0（`HIVELOAD.C` 的恢复分支与
   `HIVEINIT.C` 的 `HINIT_FILE` 分支注释原话）。
   恢复后若状态是 `STATUS_REGISTRY_RECOVERED`，`HvInitializeHive`
   还会再做一次主文件写盘，确保“恢复好的像”已经落盘，
   日志才能安全清零——否则清了日志、主文件又没写好，
   恢复成果就丢了。
4. 回到 `CmpInitializeHiveList` 的主线程：新 hive 链接进主 hive，
   新建的立即 `HvSyncHive`，已存在的按正常路径登记。

### ⑯ 懒刷链：脏了之后，修改是怎么在 5 秒后落盘的

懒刷不是一个函数，是一条四段链，任何一段断了都会让修改“只留内存”：

第一段，标脏。任何写 cell 的操作在改完数据后，把对应扇区在
脏位图里置位、脏计数加一、hive 的 `DirtyFlag` 置真。
为什么标脏与写数据是同一操作的两面：不标脏，刷盘时就不知道
这个扇区改过；标脏而不写数据，刷盘会把旧数据当成新数据写回去。
两者必须在同一个临界区里完成，否则崩溃窗口里会出现“数据已改、
位图未标”的丢修改状态。

第二段，设定时器。`CmpLazyFlush`（`CONFIG/CMWORKER.C:446`）
在两个门都打开时才设 5 秒定时器（`CmpLazyFlushIntervalInSeconds = 5`，
`CMWORKER.C:37`）：`CmpNoWrite` 为假（写盘路径已打开，
第 ⑬ 步）且 `CmpHoldLazyFlush` 为假（启动已被接受，
或 600 秒强制放行已到）。为什么定时器是 5 秒而不是立即：
注册表写操作高频，立即写盘会把每次小修改都变成一次磁盘 I/O；
5 秒窗口把多次修改合并成一次刷盘，磁盘 I/O 次数从“修改次数”
降到“5 秒窗口数”。为什么门有两个：`CmpNoWrite` 管“能不能写”
（文件系统就绪与否），`CmpHoldLazyFlush` 管“该不该写”
（启动是否已被接受、配置是否已定稿），两者缺一不可。

第三段，DPC 只排工作项。定时器到时，`CmpLazyFlushDpcRoutine`
（`CMWORKER.C:508`）在分发级别只做一件事：如果还没有待处理
的懒刷工作项，就把 `CmpLazyWorkItem` 排进延迟工作队列。
为什么 DPC 不直接写盘：第 5 批已讲，DPC 在分发级别借当前线程
上下文跑，不能等待、不能缺页；写盘要等磁盘 I/O、要走文件系统、
要拿注册表锁，全是 DPC 不能做的事。DPC 在这里的角色和第 5 批
讲的一样：中断/定时器的后半段，只负责“把活儿递给能干活的线程”。

第四段，工作线程刷盘。`CmpLazyFlushWorker`（`CMWORKER.C:557`）
在系统进程的工作线程里（普通级别，可以等待、可以缺页）拿
注册表锁、锁住 hive 装载（刷盘期间不许装载/卸载 hive，
否则链表会变），调 `CmpDoFlushNextHive` 按序扫 hive 链表：
每轮最多同步 `CmpLazyFlushHiveCount = 7` 个 hive（`CMWORKER.C:42`），
已同步的打上本轮刷盘计数标记；扫完一轮（返回“没有更多”）
则全局刷盘计数加一；若还有未刷完的或仍脏的，
工作线程不直接再排工作项，而是重新调 `CmpLazyFlush`
设 5 秒定时器，下一轮再刷。为什么刷盘期间要锁住装载：hive 链表是刷盘的
遍历对象，遍历中途插一个新 hive 或摘掉一个旧 hive，遍历指针
就会错乱；锁住装载是“遍历期间结构不变”的保证。

### ⑰ HvSyncHive：先日志、后主文件，顺序就是一致性

`HvSyncHive`（`CONFIG/HIVESYNC.C:1558`）是“把内存像写回磁盘”的
唯一出口，显式 `FlushKey` 与懒刷最后都走到它。它的顺序是：

1. 脏数为零直接返回成功。为什么：没改过就不用写，写盘是
   最贵的注册表操作，能省则省。
2. 关机后直接返回失败。为什么：关机流程有自己的刷盘顺序，
   关机后再刷会与关机写盘冲突。
3. MiniNT 模式下系统 hive 跳过写盘（`CmpMiniNTBoot` 时对机器 hive 表
   里的 hive 直接返回成功）。为什么：WinPE 远程启动时系统 hive
   是共享的，写盘会污染共享源，WinPE 的正确语义就是“只读”。
4. 易失 hive 直接返回成功。为什么：第 ④ 步已讲，易失 hive 的
   内容不落盘，刷盘逻辑看到它就当“已写好”。
5. 截断尾部空闲 bin（`HvpTruncateBins`）：把 hive 末尾已全空闲的
   bin 从文件长度里截掉。为什么：hive 缩小后文件尾巴会留一截
   全空闲的 bin，不截掉文件只会越长越大；截掉是“文件长度
   跟得上实际数据量”的保证。
6. 写日志（`HvpWriteLog`，仅当 `Hive->Log` 为真）。
   日志内容是：日志头 + 脏位图 + 全部脏扇区的数据，
   写之前先把日志文件扩到够大（头一簇 + 脏位图 + 脏扇区数 × 扇区大小，
   按 `HLOG_GROW` 即一个块取整）。写日志时序列号的玩法是：
   若 `Sequence1 != Sequence2`，先把 Sequence1 加一再写，
   保证日志头里的序列号与本次写盘对应。
   为什么日志要先写：日志是“这次写盘的意图记录”，主文件是
   “这次写盘的结果”；意图先落盘，结果后落盘，崩溃后才能
   根据意图判断结果是否完整——这就是预写日志的全部理由，
   顺序反过来（先主文件后日志）会在主文件写一半崩溃时
   留下一个“半新半旧、且无日志可救”的文件。
7. 写主文件（`HvpDoWriteHive(HFILE_TYPE_PRIMARY)`）。
   只写脏扇区（按脏位图），写完后把 Sequence2 加一，
   使 Sequence1 与 Sequence2 重新相等。
8. 清脏位图、脏计数归零。为什么清位图放在最后：
   清早了，写盘中途崩溃会丢“我还没写完”的记录；
   清晚了，下次刷盘会重复写一遍（只是浪费，不是错误）。
   所以“写完主文件 → 清位图”是唯一正确的顺序，
   位图是“还有什么没写”的账本，账本只能在货全部出库后销账。

序列号对的语义至此完整：相等 = 上次写盘完整结束；
不等 = 上次写盘在日志与主文件之间崩溃，下次装载时走日志恢复
（第 ⑮ 步）。校验和是另一道防线：序列号相等但基块内容被踩坏时，
校验和重算不符，装载时照样报损坏——两道防线防的是两种不同的坏：
序列号防“写一半”，校验和防“内容错”。

### ⑱ CMBOOT.C 的交棒面：注册表怎么给第 7 批准备驱动清单

`CONFIG/CMBOOT.C` 不是“启动时期的注册表”，而是“注册表在启动时
给 I/O 系统准备的驱动清单排序器”。它的入口是 `CmpFindDrivers`
（`CMBOOT.C:505`），被 `CmGetSystemDriverList`（`CONFIG/CMSYSINI.C` 的
驱动清单函数，第 7 批细讲）调用，逻辑是：

- 读 `CurrentControlSet\Services` 子树，筛出 `Start` 值为
  Boot（0）与 System（1）的服务——即“内核启动时就要装的驱动”。
- 按 `LoadOrderGroup` 分组、按 `Tag` 在组内排序，
  `CmpOrderGroup`（`:1323`）处理组顺序，`CmpDoSort`（`:1162`）
  处理组内排序，`CmpResolveDriverDependencies`（`:1246`）按
  `DependOnGroup` 与 `DependOnService` 解依赖（A 依赖 B 则 B 排前面），
  `CmpValidateSelect`（`:1406`）校验 Select 值合法。
- 为什么排序逻辑放在 CM 而不是 IO：排序规则是注册表数据的语义
  （组名、Tag 值、依赖关系都在注册表值里），CM 的解析层离数据最近，
  顺手做排序比 IO 再读一遍注册表更直接；IO 只管按清单顺序
  调 `MmLoadSystemImage` 装映像、调 `DriverEntry`（第 7 批）。
- 为什么这部分算第 6 批而不算第 7 批：清单的“算”法属于注册表
  （读什么、怎么排），清单的“用”法属于 I/O（按清单装载）；
  本批讲到“清单怎么算出来”为止，装载链留给第 7 批。

## 第三部分：32 / 64 位对照与单核 / 多核对照

### 32 / 64 位对照

1. hive 文件格式与字长无关。基块用 `#pragma pack(4)` 定死布局，
   字段全是 `ULONG` 与定长数组，没有指针；cell 索引是 32 位
   “类型位 + 偏移”，不是指针；bin 大小 4KB、扇区 512 字节
   都是定值。为什么格式必须与字长无关：hive 文件要在不同字长的
   内核之间可读（同一块盘、同一个 SYSTEM 文件，32 位内核与
   64 位内核都要能启动），格式里一旦出现指针或字长相关字段，
   跨字长启动就断了。内存里的 `CMHIVE`、`KCB` 则随字长变，
   因为它们含指针与句柄——文件格式与内存结构分离，正是
   `HHIVE` 回调抽象（术语表第四节）存在的理由。
2. 版本号是格式版本，不是字长版本。`HSYS_MAJOR = 1` 必须相等、
   `HSYS_MINOR` 只要求不高于 5（`HSYS_MINOR_SUPPORTED`），
   与运行在 32 位还是 64 位无关；64 位内核读 32 位时代留下的
   hive 文件，走同一套校验逻辑。
3. 簇大小与字长无关，只与卷扇区大小有关。`Cluster` 从卷信息算出，
   32 位与 64 位在同一卷上算出的簇大小相同，脏位图的粒度也相同；
   第 ⑭ 步的簇修正逻辑在两种字长下走同一条路。

### 单核 / 多核对照

1. 锁分三层，粒度从粗到细：注册表全局锁（ERESOURCE，
   `CmpRegistryLock`）管名字空间与 hive 链表；每 hive 的 hive 锁
   （pushlock，`CMHIVE.HiveLock`）管该 hive 的 cell 分配、脏位图、
   视图链表；视图锁（保护互斥，`CMHIVE.ViewLock`）管 bin 视图的
   映射与解映射。为什么分层：多核下不同 hive 的读写可以并行，
   只在跨 hive 操作（链接、装载、全局刷盘）时才升到全局锁；
   一把全局锁管一切会让所有注册表访问串行化，多核的并行能力
   全浪费在锁等待上。单核下分层依然存在（代码同一套），
   但竞争消失：pushlock 与自旋部分退化为仅抬 IRQL，
   层与层之间的嵌套顺序（全局 → hive → 视图）仍是死锁预防的依据，
   顺序反了在单核上也可能自锁（拿着细锁再拿粗锁，
   粗锁的持有者又在等细锁）。
2. 并行装载只在多核上才有收益，单核上只是“多线程轮流跑”。
   `CmpInitializeHiveList` 的 6 个线程在单核上仍然并行创建，
   但同一时间只有一个在跑，收益只剩“一个线程等磁盘时另一个能跑”，
   不像多核那样真正同时读盘；正确性不依赖核数，因为链接步骤
   由主线程串行做，线程之间只通过事件与计数同步。
3. 懒刷工作线程与 DPC 的核归属：定时器 DPC 在哪个核到期就在
   哪个核排工作项，工作线程是系统进程的线程、可以在任意核跑；
   刷盘时拿的注册表锁是跨核的（ERESOURCE），所以无论 DPC 在哪核、
   工作线程在哪核，刷盘的互斥都成立，不存在“某核的修改
   只有某核能刷”的归属问题。

## 第四部分：易混点

1. 链接节点 vs 符号链接键：前者挂一个 hive（存储单位切换），
   后者指一条路径（名字别名）。`System` 是链接节点，
   `CurrentControlSet` 是符号链接键；把前者当成后者会以为
   “System 只是个别名”，把后者当成前者会以为
   “CurrentControlSet 背后有个独立的 hive 文件”。
2. 主 hive vs SYSTEM hive：主 hive 是内存骨架（`\Registry`、
   `\Registry\Machine`、`\Registry\User` 与链接点），
   SYSTEM hive 是机器配置本体（`ControlSet`、`Services`）。
   主 hive 不落盘，SYSTEM hive 落盘；混淆两者会以为
   “注册表根不落盘所以注册表都不落盘”。
3. `HINIT_MEMORY` vs `HINIT_MEMORY_INPLACE` vs `HINIT_MAPFILE` vs
   `HINIT_FILE`：MEMORY 是“拷进池存储”（内核用，SYSTEM 走它），
   INPLACE 是“原地只读映射”（加载器用，内核不用），
   MAPFILE 是“缓存映射视图、按需读盘”（第二阶段的文件 hive 走它），
   FILE 是“直接读文件”（MAPFILE 失败时的退路）。
   四者的区别不在“数据从哪来”而在“数据住哪、谁负责拷、能不能写”。
4. 易失 cell vs 易失 hive：前者是 hive 内部一套存储区
   （同一 hive 里部分项不落盘），后者是整 hive 开关
   （整个 hive 不落盘）。SYSTEM hive 里既有稳定 cell 也有易失 cell，
   HARDWARE hive 整个是易失 hive。
5. 脏位图 vs 日志：位图是“哪些扇区改过”的账本（内存里），
   日志是“改过扇区的数据长什么样”的记录（磁盘上）。
   位图丢了可以重算（全量刷盘），日志丢了就无法恢复
   “主文件写一半”的状态；两者一个管“写什么”，一个管“写坏了怎么救”。
6. 懒刷 vs 显式刷盘：懒刷是 5 秒合并、尽力而为；
   显式 `FlushKey` 是同步、立即、调用方等它返回。
   关机、安装完成这类时刻必须走显式刷盘，赌懒刷的 5 秒窗口
   等于赌机器不会在窗口内断电。
7. `CurrentControlSet` 链接 vs ControlSet 副本：链接是别名，
   改 ControlSet001 就是改 CurrentControlSet；副本是两份数据，
   改一份另一份不动。本批第 ⑨ 步的全部意义就在于
   “不要副本、只要别名”。

## 第五部分：自查题

1. 为什么 `CmInitSystem1` 必须在 `IoInitSystem` 之前，而
   `NtInitializeRegistry` 的情况 1 必须在 autocheck 之后？
   （答案要点：前者是因为驱动加载要读 `Services`；后者是因为
   绑文件需要文件系统就绪、且不能与修盘冲突。）
2. `HIVE_NOLAZYFLUSH` 旗标为什么是临时的？如果第二阶段
   忘了清它，会发生什么？（答案要点：旗标防的是“懒刷误报已保存”；
   忘了清，SYSTEM hive 的修改永远只留内存、永不落盘，
   下次启动全部丢失。）
3. `Sequence1 != Sequence2` 时，下次装载走什么路径？
   日志里有什么？恢复完成后为什么还要再写一次主文件？
   （答案要点：走日志恢复；日志里有头、脏位图、脏扇区数据；
   再写一次主文件是为了让“恢复好的像”先落盘、再清日志，
   否则清了日志、主文件又没写好，恢复成果就丢了。）
4. 为什么 `CurrentControlSet` 必须是易失的符号链接，
   而不能是落盘的普通键或 ControlSet 的副本？
   （答案要点：符号链接保证“改一处即改全部”；易失保证
   每启动按 Select 重建；落盘或副本都会让“当前”钉死或分裂。）
5. 6 个机器 hive 并行装载时，为什么“链接”步骤必须由主线程
   串行做，而“读文件”步骤可以并行？（答案要点：
   读文件各碰各的 hive，无共享结构；链接改的是主 hive 的
   名字空间这个共享结构，并行链接会互相踩。）

## 第六部分：未钉死清单

1. `HINIT_MEMORY` 不查 `Sequence1 == Sequence2`、而
   `HINIT_MEMORY_INPLACE` 查的原因：树内无注释，两种分支的
   校验清单差异已实查（`HIVEINIT.C` 两分支的 if 条件），
   但“为什么拷贝模式放宽”没有源码依据，不编造。
2. `CmpInitializeMachineDependentConfiguration` 的具体写入项：
   本批只实查到它“往 `CurrentControlSet\Control` 写机器相关值”
   这一层，未逐项列出它写的全部值名。
3. `HvpBuildMapAndCopy` 的两级映射细节（小目录与大目录的分界、
   大 bin 的映射方式）：本批只实查到“建两级映射并拷 bin”
   这一层，未钉死目录项的切换阈值。
4. 32 位与 64 位内核读同一 hive 文件的实测证据：格式与字长无关
   是从 `#pragma pack(4)`、定长字段、无指针字段推断的，
   未在真机上实测跨字长启动。
5. `CMBOOT.C` 排序中 `Tag` 与 `LoadOrderGroup` 冲突时的确切裁决顺序：
   本批只实查到分组、组内排序、依赖解析三层，未逐行读完
   `CmpDoSort` 的全部比较分支。

## 第七部分：对位小结

- 点火链位置：`INITOS.C:1437`（CmInitSystem1）→ `IoInitSystem`
  （`:1531`，第 7 批）→ SM 调 `NtInitializeRegistry`（情况 1）
  → 驱动按 CMBOOT 清单装载（第 7 批）。
- 与第 2 批的关系：第 2 批给了时序坐标，本批给了全部内部；
  第 2 批 QA 里“CM Part 1 在文件系统之前、只开 SYSTEM + HARDWARE”
  的结论在本批得到完整展开（Part 1 就是 `CmInitSystem1`，
  Part 2 就是 `NtInitializeRegistry` 情况 1）。
- 与第 4 批的关系：`Key` 对象类型是第 4 批对象机制的第一个
  大规模使用者；链接节点与符号链接键是对象解析语义的两种扩展，
  理解了它们，第 4 批的“解析例程决定名字是什么意思”才算落了地。
- 与 Probert《NT Registry Implementation》讲的对位：
  该讲的 11 步 Flush 算法在本批第 ⑰ 步有实现落点
  （标脏 → 懒刷定时 → DPC → 工作线程 → 先日志后主文件 →
  清位图），但 WRK 的 `HvSyncHive` 是简化版（无交替文件写），
  讲义里的“交替文件”步骤在 WRK 树里对应的是日志文件，不是
  另一个主文件副本——这一点不要按讲义硬套。
- 下一批预告：第 7 批 I/O 点火，按本批第 ⑱ 步的清单，
  `MmLoadSystemImage` 装映像 → `IopInitializeDriverModule` →
  `DriverEntry` 的全链，以及 system-start 与自动启动的后续波次。

## 第八部分：问答补记

- **注册表点火阶段文件系统初始化了吗（2026-10-27 答疑补记）**：分两段说。第一阶段 `CmInitSystem1`（`INITOS.C:1437`）时没有：文件系统驱动还没装、卷还没挂载、打不开任何文件，所以 SYSTEM hive 只能用加载器内存映像、写操作只留内存（`CmpNoWrite` 为真）。第二阶段 `NtInitializeRegistry`（SM 在 autocheck 跑完、页面文件打开之后调）时有了：卷已可访问，hive 才能绑到 `\SystemRoot\config` 下的文件上。容易混淆的两点不算文件系统已初始化：其一，`CmInitSystem1` 前一步的缓存管理器（`:1426`）只是缓存映射基础设施，不是文件系统；其二，加载器自带的极简只读文件系统代码只在进内核之前跑，属于加载器，不属于内核文件系统。
- **加载器预读与第二阶段绑定（2026-10-27 答疑补记）**：主干是对的，但三处要收窄。① 只有 SYSTEM 一个 hive 文件是加载器预读进内存的，其他机器 hive（SOFTWARE、SAM、SECURITY、DEFAULT）是第二阶段现从文件读的，HARDWARE 根本没有文件。② 绑定的动作不是文件系统做的，是 CM 做的：CM 在 `CmpLoadHiveThread` 里打开主文件与 `.LOG`、把句柄塞进 `CMHIVE`、清 `HIVE_NOLAZYFLUSH`；文件系统只负责“文件能打开”这个前提。③ 时间上不是文件系统一好就绑：I/O 初始化（Phase 1）→ 建 SMSS → autocheck → 页面文件 → SM 调 `NtInitializeRegistry` 才绑；且绑完也不是立即全量转换，内存像到缓存映射视图的转换在下一次刷盘时完成。
- **文件系统在哪一阶段完成初始化（2026-10-27 答疑补记）**：没有单一时刻，分三层。① I/O 管理器就绪：`IoInitSystem`（`IO/IOMGR/IOINIT.C:141`）开头建 I/O 数据库与磁盘/CD-ROM/网络文件系统队列头，这是“文件系统能注册”的前提。② 驱动注册：`IoInitSystem` 内两波——boot 波（`:686`，`IopInitializeBootDrivers`，调加载器已载入内存的 boot 驱动的 DriverEntry）与 system 波（`:760`，`IopInitializeSystemDrivers`，从注册表 Services 装 system 驱动）；文件系统驱动在 DriverEntry 里调 `IoRegisterFileSystem` 把自己挂到对应队列上，至此 I/O 管理器才知道有这种文件系统；末尾 `IopReassignSystemRoot` 重指 `\SystemRoot` 并分配盘符，名字空间层面才完整。③ 卷挂载：驱动注册不等于卷可用，卷是第一次被打开时才挂载的（I/O 管理器匹配已注册的文件系统、调其挂载例程、建卷参数块 VPB）。**（2026-10-27 第 7 批纠正：原句“系统卷第一次被打开就是 SM 阶段”有误，现修正——系统卷的第一次打开发生在 `IoInitSystem` 期间，至迟在 system 波读第一个驱动文件时（`PsLocateSystemDll` 读 ntdll 更早），远早于 SM 阶段；SM 阶段只是 hive 文件第一次被打开，卷那时早已挂载。详见第 7 批第 ⑮ 步。）**诚实边界：WRK 树内没有 PnP 管理器的 .C 文件，`IopInitializeBootDrivers` / `IopInitializeSystemDrivers` 的本体不在树内，具体哪个文件系统驱动在哪一波，在树内钉不死；Start 值筛选（0=Boot、1=System）已由第 7 批经 `CmpFindDrivers` 的 LoadType 参数钉死。
- **文件系统基础知识与 NTFS 优劣（2026-10-27 答疑补记）**：磁盘只认按扇区号读写，文件系统是盖在其上的翻译与组织层：把文件、目录、名字、权限翻译成扇区归属。分层为应用程序 → 文件系统驱动 → 卷（分区格式化后的样子）→ 磁盘驱动 → 物理介质。核心概念：簇（分配最小单位，NTFS 默认 4KB，管理开销与内部碎片的权衡）；元数据（文件名、大小、时间、权限、数据位置，坏了则数据找不到）；目录（名字 → 文件索引的映射表，路径逐级查表）；索引（NTFS 为 MFT 记录，小文件数据可常驻记录内，大文件用 run list 描述簇分布）；打开的文件（路径解析后建文件对象/句柄，后续按文件内偏移读写）；缓存（系统缓存/Cc，写先脏缓存后刷盘，故断电可丢最近修改）；链接（硬链接为同一索引多名字、引用计数归零才释放，软链接为存目标路径的特殊文件）。崩溃一致性三路：事后检查（chkdsk/fsck，全盘扫、盘大则慢）、日志（先记打算怎么改，崩溃后重放/回滚，NTFS、ext4 路线，通常只保元数据不保用户数据）、写时复制（先写新副本再原子切换索引，ZFS、btrfs 路线，可快照与校验，代价是写放大与管理复杂）。衡量标准：正确性与可恢复性、性能（顺序大文件、小文件批量、随机读写、碎片化后衰减、并发锁粒度）、空间效率、可扩展性、功能（ACL、压缩、加密、配额、快照、稀疏、长文件名）、静默损坏防护（端到端校验和）、可移植与生态。NTFS 优点：日志恢复快（$LogFile 重放）、MFT 统一（一切皆记录，小文件常驻、大文件 run list）、权限完整（细粒度 ACL、EFS、配额、USN 日志）、容量与功能（大卷大文件、稀疏、压缩、硬/软链接、ADS、$BadClus 坏簇重映射、$MFTMirr 镜像）、碎片可治理（官方整理接口、目录 B 树）。缺点：只保元数据不保数据、没有端到端校验和与原生快照/去重（快照靠卷层 VSS）、实现复杂且跨平台弱（Linux 写支持后进、macOS 默认只读）、元数据开销与碎片（MFT 膨胀、日志写对闪存不友好）、历史包袱（ADS 藏恶意代码、权限配复杂、260 字符长路径遗留）。落回本线：第 7 批的 ntfs.sys 在 DriverEntry 里向 I/O 管理器注册（IoRegisterFileSystem），卷首次打开时挂载（读引导扇区、定位 $MFT、建 VPB），之后 hive 写盘与缓存刷盘都走它。
- **为什么要有文件系统、没有会怎样、DOS 是不是文件系统（2026-10-27 答疑补记）**：磁盘只给按扇区读写，不给名字、可变长文件、目录与空间归属；文件系统补四样：名字（扇区范围变成可按名打开的东西）、分配与回收（空闲表记录簇归属，写取删还）、可变长与增长（追加新簇、释放旧簇）、隔离与权限。没有它：对用户，无文件/文件夹/文件名，只能记扇区起止，改名移动扩容要手动搬扇区，删了也不自动腾空间；对程序，各自重造互不兼容的迷你文件系统且无隔离，一个 bug 可踩掉别家数据，启动文件位置只能固定死；对系统，没法共享、没法按名启动，连按路径打开 hive 文件的操作都不存在。裸用未绝迹，只退到不需要名字与目录的场景：交换分区（swap，内核按页槽直接管理）、数据库裸分区（在内部重做一套组织，代价是别家读不懂）。DOS 不是文件系统：DOS（Disk Operating System，MS-DOS/PC DOS）是一整套操作系统（IO.SYS、MSDOS.SYS、COMMAND.COM 等），它用的文件系统叫 FAT（文件分配表）：卷里一张表、表项与簇一一对应，值表空闲/下一簇/坏簇/结束，目录项记起始簇、顺表跟簇链；按表项位宽分 FAT12（软盘小盘）、FAT16（DOS 时代硬盘，簇数上限 65525）、FAT32（Win95 OSR2 起，约 2.68 亿簇）。混称原因：DOS 时代几乎只用 FAT，且 Linux 挂载类型名 msdos/vfat 带 msdos。FAT 优点是极简、处处可读；缺点是无日志（断电须全盘扫描）、无真正权限（仅只读/隐藏/系统/存档属性位）、FAT16 根目录项数固定 512、8.3 短名（长文件名靠 VFAT 补丁拼额外目录项）、碎片严重、大簇浪费，适合 U 盘/存储卡，不适合系统盘——这也是 Windows 系统盘从 FAT32 迁到 NTFS 的原因。

## 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，与正文互补。

- **懒刷：定时器 → DPC → 工作队列 → 工作线程，四次换上下文**。5 秒定时器到期 → 分发级别的 DPC（DPC 在分发级别不能等 I/O、不能缺页，所以只干一件事）→ DPC 例程（`CONFIG/CMWORKER.C:508`）调 `ExQueueWorkItem(&CmpLazyWorkItem, DelayedWorkQueue)`（`:549`）把工作项挂进延迟工作队列 → 系统工作线程在 PASSIVE_LEVEL 跑工作例程（`:557`）：拿注册表锁、锁 hive、`CmpDoFlushNextHive` 同步。转移方式：定时器中断 → 软件中断（DPC 向量）→ 工作队列（线程间移交）→ 工作线程执行；级别一路下降（分发级别 → 被动级别），能等 I/O 的事只在最后一段做。
- **hive 文件读写：逐层 IRP 派遣与逐层完成返回**。`NtCreateKey` / 键值读写进 CM 后，读 hive 文件走：CM → 缓存管理器（`Cc`）→ 文件系统驱动 → 卷驱动 → 磁盘驱动，每层之间是 IRP 派遣（函数指针调用，栈位置下移一格，见第 7 批补记）；读回后逐层完成返回（完成例程倒序调，见第 7 批补记）。转移方式：函数指针逐层调用 + 完成链倒序返回，没有中断（除非磁盘完成时走中断 + DPC，那是另一条链：磁盘中断 → ISR → DPC → 完成 IRP）。
- **第二阶段并行装载：线程 + 事件等待**。6 个机器 hive 各起一个系统线程（`CmpLoadHiveThread`，`CONFIG/CMSYSINI.C:3792`）并行读文件，主线程在事件上等齐（`KeWaitForSingleObject`，等待期间主线程被换出，CPU 去跑别的线程）→ 事件触发 → 主线程被唤醒、串行链接。转移方式：线程创建 + 事件等待/唤醒 + 换栈；链接必须串行，因为链接改主 hive 名字空间，并行会互相踩。
- **注册表解析：函数指针，不是 switch**。键路径解析走到 hive 边界时，经链接节点切换 hive 指针与根 cell（`CmpLinkHiveToMaster` 的链接过程）；符号链接键解析走对象管理器的重解析循环（见第 4 批补记）。转移方式：函数指针调用 + 循环内 goto 重启。
