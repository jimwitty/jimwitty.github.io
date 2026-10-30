---
layout: post
title: "WRK 第 1 批：KiSystemStartup 与处理器点火"
categories: WRK
series_order: 2
---

> **取材说明**：底本为 WRK 源码树 `~/workspace/WRK/WRK/WRK-V1.2`（WS03 SP1 / NT 5.2 代码线）中 `BASE/NTOS` 的 i386 与 AMD64 两套实现；HAL 本体不在 WRK 包内（只有三份预编译 HAL 的 DLL+PDB），凡涉 HAL 内部实现处均按"调用点反读"处理并注明；设计文档对位 `NTDESIGNWORKBOOK` 的 STARTUP.DOC（仅"三阶段初始化 + 执行体拉起 SM（会话管理器，即 SMSS）"骨架存活）。tpn/pdfs 是第三方文档收集仓库、非微软官方；本批不依赖它。
>
> **本批写作规范（2026-10-25 用户定稿，后续各批同此标准）**：
> 1. 术语表前置：凡本批出现的专有名词、缩写，开篇先给定义，并说明它是做什么用的、为什么要有；
> 2. 每个机制、每次判定、每次"降到某级 / 抬到某级"，正文里讲清为什么非如此不可——包括历史因素与兼容考虑（我们无法身处前人所处的历史阶段，当时的限制必须写出来）；
> 3. 凡涉 32 位与 64 位、单核与多核分叉处，必须讲清为什么 64 位 / 多核可以这么干、而 32 位 / 单核只能那么干；
> 4. 正文零错漏标准：每条机制结论以本轮复核过的源码行号为准；树里查不到的，列入文末"未钉死清单"，不编造、不回填旧推断。
>
> **本批一句话地图**：加载器（ntldr）把内核映像、加载器参数块（LoaderBlock）准备好，跳进 `KiSystemStartup`（汇编入口）；它先封场（HIGH_LEVEL）、修好加载器留下的半成品（IDT/GDT/TSS）、立起本核的 PCR/PRCB，再降到 APC_LEVEL 跑 Phase 0 点火总控（`ExpInitializeExecutive`），期间创建 Phase 1 线程；P0 在 Phase 1 里把其余核一个个叫醒（HAL 的 SIPI），各核对账、栅栏放行；最后每核把 Idle 优先级归零，进 `KiIdleLoop` 待命。

---

## 第一部分：术语表（先读这部分，正文不再重复定义）

按"硬件 → 每核结构 → 中断与调度 → 核间通信 → 寄存器与 CPU 特性 → 内核结构"的顺序排。每个词条 = 定义 + 做什么用 + 为什么要有（只写非 obvious 的部分）。

### 一、处理器与核

- **P0 / BSP（引导处理器）**：开机第一个跑起来的核。为什么要有一个"第一个"：上电时硬件只保证一个核执行 BIOS/加载器代码，其余核必须被显式叫醒；没有这个不对称，启动就没有起点。
- **AP（Application Processor）**：之后被 P0 叫醒的其余核。叫"Application"只是历史命名，不是指它跑应用程序。
- **SMP / MP / UP**：SMP（对称多处理）= 多个核地位平等、共享内存；MP 是简称；UP = 单核。WRK 这条是 MP 内核，但跑在单核机器上也能工作（靠 UP HAL 探测 + 运行时改代码，见第五部分）。
- **SMT（同步多线程，Intel 称超线程）**：一个物理核同时冒充两个逻辑核。为什么操作系统要在意：两个逻辑核共用同一套执行单元，调度器若把两个重活同时压在同一物理核的两个逻辑核上，实际吞吐反而下降；所以 `KiAllProcessorsStarted` 要做 SMT 分组，把同物理核的逻辑核认出来记成一组。
- **NUMA 节点（KeNodeBlock / KiNode0）**：按"内存离哪个核近"做的分组。为什么要有：多路服务器上访问远节点内存延迟更高，分配器与调度器要尽量让线程用近处内存；单路机器只有一个节点 0，分组退化但代码统一。

### 二、每核的两张"身份证"：PCR 与 PRCB

- **PCR（Processor Control Region，KPCR）**：每核一份的处理器控制区。做什么用：它是内核在每个核上找东西的总入口——本核 PRCB 指针、GDT/IDT/TSS 地址、当前线程，全从这里出发。i386 用 FS 段寄存器指向它，AMD64 用 GS 基址指向它。`SelfPcr` 是它指向自己的字段：代码拿到 PCR 后能再校验"我是谁"，转储分析也靠这个自指锚点定位。
- **PRCB（Processor Control Block，KPRCB）**：每核一份的调度台账。做什么用：调度器与 DPC 的全部每核状态都在这里——当前线程（CurrentThread）、下一个线程（NextThread）、Idle 线程、就绪队列、DPC 队列（DpcData）、排队锁登记表（LockQueue）、核号、控制寄存器存档（ProcessorState）。为什么每核一份：调度状态天然是每核的，两个核不能共用一张台账，否则每步调度都要跨核加锁。i386 里 PRCB 内嵌在 PCR 内部（PrcbData 成员）；AMD64 里 PCR 与 PRCB 分开，PCR 只存指向 PRCB 的指针。
- **KiProcessorBlock[]**：全局数组，下标 = 核号，存每个核的 PRCB 指针。做什么用：想找某核的台账先查这张总目录；核间操作（发 IPI、偷线程）都从这里定位对方。
- **KiIdleSummary**：位图，记录哪些核当前正闲着（在跑 Idle 线程）。做什么用：P0 要给某核派活时，先看它闲不闲。

### 三、段、表与任务切换的硬件零件

- **GDT（全局描述符表）**：描述段与特殊结构（TSS）的表。CPU 访问某段前先查表拿基址、限长、特权级。为什么每个核都要有自己的 GDT（见正文第十步）：GDT 里有本核的 TSS 描述符与 PCR 选择子，不可能全核共用一张。
- **选择子（Selector）**：段寄存器里装的值 = GDT 索引号 + 特权级。`KGDT_R0_PCR` 就是"指向 PCR 的那个选择子"的常量名。
- **LDT（局部描述符表）**：每个任务可有自己的描述符表。现代 Windows 基本不用，启动时把 LDTR 清掉——留着它只是为了兼容旧的段模型。
- **IDT（中断描述符表）**：256 个表项，中断/异常号就是下标。CPU 收到 8 号中断就查第 8 项找处理函数入口。表项分三种：中断门（进处理函数时自动关中断）、陷阱门（不自动关）、任务门（见下）。
- **TSS（任务状态段）**：CPU 硬件用的"每核一张登记卡"，不是任务本身。卡上记着：从用户态切进内核态时换到哪张栈（Rsp0）、中断走 IST 时用哪张栈、I/O 许可位图（IOPM）在哪。每个核一张，存在 GDT 描述符指向的地方，用 `ltr` 指令告诉 CPU"这张卡归你"。为什么要有硬件登记卡：特权级切换时 CPU 必须在不依赖任何软件状态的前提下找到内核栈——软件还没跑，唯一能信的就是这张卡。
- **任务门（Task Gate）**：IDT 表项的一种，指向一个 TSS。中断发生时 CPU 按任务门自动切换到该 TSS 描述的任务（换栈、换上下文）。为什么 i386 时代要用它处理 NMI/双重故障：出事时当前栈可能已经坏了，普通中断门会继续用坏栈，只有任务门能强行换到一张干净的栈上。
- **IST（中断栈表）**：AMD64 版 TSS 里的一组栈指针（IST1～IST7）。IDT 表项可直接指定"这次中断用第几号 IST 栈"，效果和任务门一样但更轻量——所以 AMD64 不需要 NMI/DF 任务门，改在 TSS 里给 NMI、双重故障各配一张 IST 栈。
- **NMI TSS / DF TSS**：i386 任务门机制里，专门给 NMI（2 号）与双重故障（8 号）准备的两张 TSS，各自指向各自的专用栈。
- **dummy 处理（Dummy Handler，占位处理函数）**：不干正事、只保证"出事也不扩大"的处理函数。本批的两个实物是 `KiDummyNmiHandler` 与 `KiDummyDoubleFaultHandler`（`KE/I386/ALLPROC.C:1124 / 1155`）：只干两件事——清掉对应 TSS 描述符的 busy 位、然后继续 halt。为什么要有：被判"不启动"的核关中断后永久 halt，但 NMI 是关不掉的；万一 NMI 打到这颗核身上，CPU 会走任务门切到 NMI TSS——若 busy 位不清，下一次 NMI 再切同一个 TSS 会触发 #TS（无效 TSS 异常）→ 双重故障 → 三重故障 → 整机复位。占位处理就是"宁可什么都不干，也不能让一颗停机的核因为一次意外打来的 NMI 把整机拖复位"。`KiSetHaltedNmiandDoubleFaultHandler`（`ALLPROC.C:1078`）干的就是改写：查 IDT 里 NMI/DF 向量的任务门选择子、顺 GDT 找到对应的 TSS，把 TSS 的 Eip 改成这两个占位函数。
- **KIDTENTRY**：IDT 表项在源码里的结构名，字段按硬件布局顺序为 `Offset / Selector / Access / ExtendedOffset`（`INC/I386.H:892`）。
- **_KiTrap00…**：0 号中断（除零错）等处理入口的命名惯例。**KxUnexpectedInterrupt**：某个向量根本没登记处理函数、却被打进来时的兜底桩——没有它，未登记的中断会跑飞。

### 四、中断与异常：NMI、双重故障、双重故障栈

- **中断（Interrupt）**：外部事件（键盘、网卡、时钟、别的核）打断当前代码，转去处理函数，完事回来继续。
- **异常（Exception）**：执行当前指令时自己出的错（除零、缺页、非法指令）。分三类：fault（可重试，如缺页——处理完重执行指令）、trap（执行完才报）、abort（现场已毁，无法恢复）。
- **NMI（不可屏蔽中断，2 号）**：`cli` 也关不掉的中断。为什么要有：硬件故障、看门狗超时这类"天塌了也必须报"的事，不能被软件屏蔽。
- **双重故障（Double Fault，8 号）**：处理一次异常的过程中又出了第二次异常（典型：异常处理要用的栈本身坏了、页表坏了），CPU 把它升级为双重故障。再处理不了就是三重故障（Triple Fault），CPU 直接复位重启——连蓝屏都来不及。
- **双重故障栈（DF 栈）**：专门给双重故障留的一张"救命栈"。为什么要有：正常异常处理借的是当前线程的栈，栈要是坏了就无路可走；独立的栈保证哪怕内核栈已损坏，处理函数还能跑进来、把蓝屏信息打出来。本批里它还客串了一把：P0 点火早期内存管理器没上线、分不出正式 DPC 栈，就先借这张静态栈当 DPC 栈。
- **KTRAP_FRAME**：中断/异常进内核时在栈上搭的"现场帧"——寄存器快照，处理函数用它，返回时按它恢复。**ENTER_TRAP** 就是搭这帧的汇编公共代码。

### 五、IRQL、DPC、调度

- **IRQL（中断请求级）**：内核的"软件中断屏蔽等级"，不是硬件优先级。数字越大，能打断你的中断越少。常用档位从低到高：PASSIVE_LEVEL（0，正常代码）→ APC_LEVEL（1）→ DISPATCH_LEVEL（2）→ 各种设备级（DIRQL）→ HIGH_LEVEL（31，i386）/ 15（AMD64）。为什么要有软件级别：硬件只能开关全部中断，内核需要"只挡某类以下的中断、放行以上的"这种精细控制——IRQL 就是这套软件标尺。
- **APC（异步过程调用）**：排队到某线程去执行的小任务。APC_LEVEL 是它的级别。为什么 APC 单独占一级：APC 是"在目标线程上下文里跑"的机制，必须保证目标线程的 APC 状态已建好才能投递（见正文第五步）。
- **DPC（延迟过程调用）**：中断处理里"先记下来、稍后批量做"的工作项。为什么要有：中断处理函数本身要尽快返回（高 IRQL 上不能做重活），于是把重活打包成 DPC，降到 DISPATCH_LEVEL 再执行。
- **DPC 栈**：DPC 例程运行时借用的独立栈。为什么不借被中断线程的栈：防止 DPC 把中断现场的栈搞乱；防止 DPC 自己的栈用量把本就紧张的中断栈顶爆。
- **调度器（Dispatcher）**：决定"哪个线程在哪个核上跑"的内核部分。抢占规则只有一条：**只有优先级高于当前线程的就绪线程，才能把 CPU 抢走**（同级不抢，靠时间片轮转——每线程分得固定 CPU 时间片，用完让出、同级排队）。本批尾巴的 HIGH_PRIORITY 盾牌就是这条规则的直接应用。
- **就绪队列（Ready List）**：每核 32 个优先级档次的链表，等着跑的线程按优先级挂在对应档上；调度器从最高非空档取线程。
- **自旋锁（Spinlock）**：拿不到锁就原地空转等待的锁。为什么要有：高 IRQL 上不能睡眠（睡眠意味着让出 CPU，而高 IRQL 的上下文不许被调度走），只能空转等。
- **排队自旋锁（Queued Spinlock）**：自旋锁的改进：等锁的核排成队轮流拿，避免所有核同时抢总线。锁变量在全局，排队节点存在各核 PRCB 的 LockQueue 数组里。`KiInitSpinLocks` 干的"地面工程"就是把全局锁逐个登记进每核的这张表。
- **线程 / 进程（ETHREAD / EPROCESS）**：内核里线程与进程的对象。**Idle 线程**：每核一个"没活干时跑"的空转线程，跑 `KiIdleLoop`，名义优先级 0。**Idle 栈**：Idle 线程的栈。**亲和性（Affinity）**：线程允许跑在哪些核上的位图。**页目录基址 / CR3**：进程地址空间的根页表物理地址。
- **HIGH_PRIORITY / LOW_PRIORITY**：NT 优先级标尺的两端。HIGH_PRIORITY = 31（i386 档位的最大值），LOW_PRIORITY = 0（只有 Idle 线程的名义优先级）。注意：31 在 i386 上既是优先级最大值、又是 HIGH_LEVEL 的 IRQL 值——这是两个标尺在顶端重合，不是同一个概念。
- **WaitIrql**：线程等待时的 IRQL，醒来恢复到这个级别。**NextThread**：调度器已选中、下一棒要跑的线程。

### 六、核间通信：APIC、IPI、ICR、INIT/SIPI

- **APIC（高级可编程中断控制器）**：每核一个本地 APIC，系统再加一个 I/O APIC。做什么用：收外部设备中断、分发给某个核、核与核之间互相发消息。xAPIC 与 x2APIC 是它的两代（后者用 MSR 访问，更快）。
- **MMIO（内存映射 I/O）**：把设备寄存器映射到内存地址空间的访问方式——读写某个内存地址，实际上就是读写设备寄存器。本地 APIC 的 TPR 就是这样访问的（地址 0xFFFE0080），而不是用专门的 I/O 指令。
- **IPI（核间中断）**：一个核发给另一个核的中断。用途：叫对方重新调度、叫对方停核、通知对方某段地址的 TLB 缓存失效了，等等。本批注释"先让 IPI 能寻址到本核"就是这个意思——PRCB 的 Number 与 SetMember 登记好，别人才找得到你。
- **ICR（中断命令寄存器）**：本地 APIC 里"发 IPI 用的寄存器"。往 ICR 写"目标核 + 中断向量号"，就等于发出一次 IPI。发之前要看投递状态位，确认上一条已经发出去。
- **INIT / SIPI（启动核间中断序列）**：叫醒 AP 的硬件协议，见正文第十一步。SIPI 与普通 IPI 的关键区别：普通 IPI 的向量是中断号、走 IDT 分发；SIPI 只对"等启动"的核有效，它的向量不是中断号、而是启动页号，不走 IDT。
- **KiProcessorStart / KiProcessorStartControl**：AP 刚醒来时先进入的"命令回路"：P0 通过全局变量给它下命令——`KcStartGetId`（回填 CPUID 信息）、`KcStartContinue`（放行）、`KcStartDoNotStart`（不合群的核先由 `KiSetHaltedNmiandDoubleFaultHandler` 把 NMI/DF 的 TSS 入口改写成占位处理函数——见第四节 dummy 处理——再关中断、执行 `hlt` 永久停机）、`KcStartWait`（原地等）。`hlt` 是 CPU 停机指令：停取指、等中断，等不到唤醒就一直停着。
- **栅栏（KiBarrierWait）**：全局开关变量。为什么要有：AP 初始化完也不许立刻进 Idle 循环乱跑——P0 还没拉完所有核、系统还没到"所有核就绪"的状态；等 P0 把所有核拉起、清零放行，才一起开跑。

### 七、寄存器与 CPU 特性

- **CR0 / CR1 / CR2 / CR3 / CR4 / CR8（控制寄存器）**：CR0 管总开关（其中 WP 位 = 内核写只读页也要触发故障——没有 WP，内核写坏只读页不会报，错误会静默扩散）；CR1 是架构保留禁区（规定不得访问）；CR2 记最近一次缺页的线性地址（缺页处理函数靠它知道是哪一页 fault）；CR3 是页表根（换 CR3 = 换地址空间）；CR4 管各种扩展开关（PAE/PSE/PGE/OSFXSR 等）；CR8 是任务优先级寄存器（TPR），AMD64 直接用它表达 IRQL（见第四部分）。
- **MSR（型号专用寄存器）**：CPU 的扩展寄存器，用 `rdmsr` / `wrmsr` 读写，如 GS_BASE、EFER。**swapgs**：AMD64 切换用户态/内核态 GS 基址的指令（进内核换成内核 PCR，出内核换回用户态 GS）——没有它，内核与用户态会共用同一个 GS，互相踩。
- **CPUID**：x86 查 CPU 型号与特性的指令。**FeatureBits**：内核汇总的特性位图。**步进（Stepping）**：同一型号 CPU 的小版本修订号（同一型号不同步进可能有不同勘误（芯片设计缺陷的官方记录，厂商按步进发布），内核要按最弱的对齐）。**FXSR / XMMI / cmpxchg8b / PAT / MTRR**：浮点状态保存、SSE 指令、8 字节原子比较交换、页属性表、内存类型范围寄存器。AP 对账就是逐项核对这些，全场按最弱的核对齐。
- **NX（不可执行位）**：数据页标上"不许当代码跑"，防缓冲区溢出后跳去执行注入的数据。**NOEXECUTE=**：启动选项里的开关（ALWAYSON / OPTIN / OPTOUT / ALWAYSOFF）。**EFER.NXE**：打开 NX 功能的 CPU 开关。**PAE**：物理地址扩展（36 位寻址），打开 NX 的前提条件（没有 PAE 就没有页表项里放 NX 位的位置）。
- **IOPM（I/O 许可位图）**：每个任务一张位图，规定用户态程序能直接碰哪些 I/O 端口；本批里 P0 分了 8KB 保存区存它。
- **NPX**：老式浮点协处理器（x87），现在集成在 CPU 里，名字沿用。
- **安全 cookie（__security_cookie）**：栈保护哨兵值，见正文第十七步。为什么叫"cookie"：编译器在栈上埋一个只有它知道的值，函数返回前校验——像饼干屑一样，动过就露馅。

### 八、内核结构与启动杂项

- **HAL（硬件抽象层）**：把主板、芯片组差异挡在内核之下的那层。内核调 HAL 的固定接口（如 `HalInitializeProcessor`、`HalInitSystem`、`HalStartNextProcessor`），HAL 针对不同主板各自实现。本批 HAL 源码不在包里，只能从调用点反读它的分工。
- **SSDT（系统服务描述符表）**：系统调用号 → 内核函数地址的表。**影子表（Shadow SSDT）**：复制出来的一张空表，留给 win32k（内核态窗口/图形子系统，不在 WRK 包内）以后登记第二张系统调用表的位置——为什么先占位：系统调用分发表在启动早期就要定形，win32k 晚于内核加载，只能先留空位。
- **win32k**：内核态的窗口与图形子系统，第 8 批交棒时见。
- **定时器表**：按到期时间散列的定时器链表；**KiTimerExpireDpc** 是 0 号核上定期扫描到期定时器的 DPC。**KiTimeIncrementReciprocal**：时钟滴答换算用的倒数，定时器表散列靠它。
- **LoadOptions**：启动选项字符串（如 `/NOEXECUTE=OPTIN`），加载器原样传给内核，内核用字符串匹配解析——启动选项没有结构化配置，只能靠字符串约定。
- **INIT 段**：内核映像里"只在启动时用、启动完就回收"的段。点火代码放这里，跑完就把内存还回去——为什么要回收：启动代码占的内存不小，留在常驻映像里是浪费。
- **热补丁（Hotpatch）**：函数入口预留的补丁桩，运行时可替换函数实现而不重启。本批"MP 内核撞上 UP HAL 时把自旋锁例程首字节改成 RET"就是同一类"运行时改自己代码"的早期实物（见第五部分）。
- **KiFreezeExecutionLock**：点火期最原始的自旋锁，用 `lock bts`（带总线锁的位测试并置位）指令手写。为什么不敢用自旋锁宏：调试器设施还没就绪，宏里带的调试逻辑会踩到未建好的结构——只能用最原始的指令。
- **蓝屏（BugCheck）**：内核遇到无法恢复的错误时主动停机并记录错误码（如 UNSUPPORTED_PROCESSOR、MULTIPROCESSOR_CONFIGURATION_NOT_SUPPORTED、PHASE0_EXCEPTION）。**转储（Dump）**：蓝屏时把内存写盘的文件，事后用调试器分析；**kd** 是内核调试器，`!pcr` / `!prcb` 是它的扩展命令。**DKOM**：直接内核对象操纵——Rootkit 摘链隐藏进程/驱动的手法；**PsLoadedModuleList** 是内核的已加载模块链表，取证时拿它和 KeLoaderBlock 的原始映像链对照。
- **静态对象 vs 动态分配**：静态对象是编译进内核映像、地址固定的（如 P0 的 KiInitialProcess/KiInitialThread）；动态分配是运行时从内存池或栈分配器拿的（如 AP 的 ETHREAD、正式 DPC 栈由 `MmCreateKernelStack` 分配）。点火早期只能用静态对象，因为分配器还没上线——这是本批反复出现的约束。
- **KiInitializePcr / KiInitializeBootStructures**：填 PCR/PRCB 表的函数（i386 / AMD64 各一份）。**KiInitProcessorState**：P0 替 AP 预制整套每核数据的函数。**KiInitializeKernel / KiSystemStartup**：C 点火主函数 / 汇编入口。**KeAdjustInterruptTime**：同步各核中断时间计数器。**KiInitSpinLocks / KiInitSystem / KeInitSystem / KeNumaInitialize**：KIINIT.C 里的四个收尾函数（KeNumaInitialize 做 NUMA 节点初始化）。
- **LoaderBlock / KeLoaderBlock**：加载器参数块。做什么用：加载器与内核之间的唯一交接单——已加载模块链表、内存布局、启动选项、本核的 KernelStack/Thread/Prcb 三字段（AP 拉起时 P0 逐个改写它，见第十步）。**Extension**：LoaderBlock 里的一块"夹层"：HAL 通过它把向量↔IRQL 翻译表等平台相关数据递给内核（`NEWSYSBG.ASM:465` 取 `LpbExtension`）。
- **KPROCESSOR_STATE / ContextFrame**：AP 启动用的"上下文帧"。做什么用：P0 把 AP 起跑所需的全部寄存器初值（Eip、Esp、段选择子、EFlags、GDT/IDT 基址）写在这里，跳板代码照单全收、原样装入 AP——AP 的第一条指令就是从这里来的。

### 九、实查补漏术语（2026-10-25 全文实查补）

- **实模式 / 保护模式**：实模式是 CPU 上电的 16 位寻址模式，只能寻 1MB；保护模式是 32 位分段分页模式，内核跑的模式。SIPI 叫醒 AP 时 AP 在实模式，跳板代码置 CR0.PE 进保护模式，再跳进 `KiSystemStartup`（正文第 11 步）。
- **ACPI / MP 表**：ACPI（高级配置与电源接口）是固件提供的硬件与电源描述表；MP 表（多处理器规范表）是 BIOS 提供的核拓扑表。HAL 靠它们知道每个核的 APIC ID（正文第 11 步）。
- **页目录 / 页表 / PDE**：页目录是地址空间的根表，页表是第二级表，PDE（页目录项，Page Directory Entry）是页目录里的一项、指向一张页表。CR3 装的就是页目录基址（正文第 9 步）。
- **SwapContext**：上下文切换函数——保存当前线程寄存器、恢复下一线程寄存器、换 CR3（正文第 16 步）。
- **KiIdleSchedule**：Idle 线程里从别核就绪队列偷线程的函数（IdleSchedule 置位时调它，正文第 16 步）。
- **KeYieldProcessor**：处理器空转提示（x86 上是 pause 指令的封装），自旋等待时调它、让核别把流水线打满（机理见第 2 批 QA 第 15 条）。
- **KiProcessorStartData**：P0 与 AP 共享的启动数据区，AP 执行 CPUID 后把型号、特性位回填到这里、P0 读回对账（正文第 12 步）。
- **SetMember**：PRCB 里的 KAFFINITY 位图字段（`INC/I386.H:1090`、`INC/AMD64.H:1322`），登记该核所属的处理器集合；与核号 Number 配合，别的核靠它们寻址到本核（发 IPI 时按集合定位）。
- **SList**：单向链表的无锁实现，表头 = 指针 + 序列号（防 ABA），32 位下靠 cmpxchg8b 原子更新表头（正文第 13 步）。
- **SEH / SafeSEH**：SEH（结构化异常处理）是 Windows 的异常处理机制；SafeSEH 是编译器登记合法异常处理函数的保护（GS 不管 SEH，第六部分）。
- **Rootkit**：隐藏自身存在的恶意软件（root + kit），手法包括挂钩 IDT、DKOM 摘链（第六部分）。
- **PatchGuard / 驱动签名**：PatchGuard 是内核补丁保护（防改内核关键结构）；驱动签名是微软对驱动的数字签名认证。两者随 x64 而来（第四部分第 5 项）。
- **CRT（C 运行时库）**：C 语言运行时库，用户态安全 cookie 的熵源与初始化由它做；i386 内核 cookie 初始化点疑在链接进内核的 CRT 库（未钉死清单第 4 条）。
- **rdtsc**：读 CPU 时间戳计数器的指令，值随时间单调增，适合做熵（AMD64 内核 cookie 生成用它取熵，正文第 17 步）。
- **寄存器小词**：Eip（指令指针）、Esp（栈指针）、EFLAGS（标志寄存器，含进位、零、中断使能等标志位）、CR0.PE（CR0 的保护使能位，置 1 进保护模式）、帧指针（EBP / RSP，指向当前栈帧基址）、PID / TID（进程号 / 线程号）。
- **epilogue（函数尾声）**：函数结尾拆除栈帧的代码，定义见第 2 批术语表第六节；GS 校验就在各 return 路径的 epilogue 里（正文第 17 步）。
- **SM（会话管理器）**：取材说明里的"SM"即会话管理器（SMSS），"执行体拉起 SM"指执行体创建第一个用户进程 SMSS。
- **自旋锁机理**：本批术语表自旋锁条目只讲了"为什么空转"，完整机理（硬件/软件分工、抢-等-放、TTAS、pause、UP 退化）见第 2 批术语表自旋锁条目。

---

## 第二部分：本批地图（两条线：P0 立自己，P0 拉 AP）

```
ntldr（加载器）
  │ 准备好：内核映像、LoaderBlock、系统 hive、已加载模块链
  ▼
KiSystemStartup（汇编入口，NEWSYSBG.ASM / START.ASM）
  │ P0 支线                                  │ AP 支线（被 SIPI 叫醒后）
  │ ① HIGH_LEVEL 封场                        │ 跳板 → KiSystemStartup
  │ ② 修 IDT/GDT/TSS（加载器的半成品）        │ ①' 同样封场、修表
  │ ③ 立 PCR/PRCB、Idle 线程对象              │ ②' KiProcessorStart 命令循环
  │ ④ 降 APC_LEVEL → KiInitSystem            │     （等 P0 的 GetId / Continue）
  │ ⑤ ExpInitializeExecutive Phase 0         │ ③' KiInitializeKernel 的 AP 分支
  │    （建 Phase 1 线程、MM/OB/PS 上线）      │ ④' 栅栏 KiBarrierWait
  │ ⑥ Phase 1：KeStartAllProcessors 拉 AP ────┘
  │ ⑦ 收尾：定时器倒数、DPC 栈换正
  │ ⑧ KeRaiseIrql(DISPATCH_LEVEL) + KeSetPriorityThread(0)
  ▼
KiIdleLoop（每核）：查 DPC → 查 NextThread → 查 IdleSchedule → HAL 省电
  │ 第一次进循环就切到 Phase 1 线程（P0）或纯待命（AP）
  ▼
第 2 批：ExpInitializeExecutive Phase 0/1 点火总控（本批只讲到"门打开"为止）
```

关键分界：本批讲的是"处理器点火"——让每个核能跑起来、能被调度；第 2 批讲的是"执行体点火"——MM、对象管理器、进程管理器这些子系统怎么上线。两批的交接点就是 Phase 0/Phase 1 线程。

---

## 第三部分：正文（每步：做了什么 → 为什么非如此不可）

### 第 1 步：入口先抬到 HIGH_LEVEL——封场

`KiSystemStartup` 的第一件事是把 IRQL 抬到 HIGH_LEVEL（i386 是 31，AMD64 是 15），并且关中断。为什么一上来就要封场：此时本核的现场还没搭好——IDT 是加载器留下的半成品、PCR 还没立起来、当前线程对象还没初始化。任何中断（哪怕是时钟滴答）打进来，处理函数都会去查一张没修好的 IDT、摸一个不存在的 PCR，结果不是跑飞就是三重故障复位。封场的意思是："在我把现场搭好之前，谁也别打断我。"这是点火全程的头号原则：**先立足，再开门**。

为什么封场用 HIGH_LEVEL 而不是"关中断就够了"：`cli` 只能挡可屏蔽中断，挡不住 NMI；HIGH_LEVEL 是软件级别，连 DPC、APC 这类软件中断源也一并挡住。两者叠加才是完整的封场。

### 第 2 步：修 IDT——"反"的不是字节序，是字段位置

**做了什么**：内核有一张静态 IDT 表 `_IDT`（`KE/I386/TRAP.ASM` 的 `IDTEntry` 宏逐项发出，`TRAP.ASM:356`）。宏的写法是：

```asm
IDTEntry macro name, access
        dd      offset FLAT:name    ; 一次写完 32 位处理函数地址
        dw      access
        dw      KGDT_R0_CODE
        endm
```

对照硬件布局（`INC/I386.H:892` 的 KIDTENTRY：`Offset / Selector / Access / ExtendedOffset`，各 2 字节），这张静态表是"反"的：`dd offset` 一次写完 32 位地址，占了前 4 个字节——低 2 字节正好是 Offset，但高 2 字节（地址的高 16 位）落在了本该放 Selector 的位置；而 `dw KGDT_R0_CODE`（代码段选择子）落在了本该放 ExtendedOffset 的位置。**Selector 与 ExtendedOffset 互换了位置**。`KiSwapIDT`（`KE/I386/KERNLINI.C:1230`）的工作就是把这两处换回来，换好后 `NEWSYSBG` 再用 `rep movsd` 把它覆盖到生效的 IDT 上。注意：**只由 P0 执行一次**；AP 克隆的是已换好的表，不用再换。

**为什么宏作者要故意写反**：图省事。`dd offset` 一次写完 32 位地址，比拆成"低 16 位写 Offset、高 16 位写 ExtendedOffset"两条指令省事；反正后面有个 `KiSwapIDT` 统一修正，静态初始化阶段的"反"不影响运行。注释自承这是为了"easy static init"（静态初始化图方便）。这是一个典型的"启动期将就、运行期修正"的折中：静态表只是草稿，生效前必须经过一次正规化。

**为什么硬件布局本身是分裂的（历史根源）**：286 时代的门描述符本来是 `[Offset16][Selector][Access]` 连续自然布局；386 把偏移扩到 32 位时，必须与 16 位门（类型 0110）/ 32 位门（1110）共用前 6 字节——同一解码路径、兼容 286 生态——所以多出来的高 16 位只能贴在字节 6–7。段描述符同理（基址从 24 位扩到 32 位，多的 8 位贴字节 7）。AMD64 的 KIDTENTRY64（`INC/AMD64.H:1064`）仍把偏移劈成 OffsetLow / OffsetMiddle / OffsetHigh——前 8 字节与 32 位门一致，历史重演。**这不是设计失误，是兼容性把布局钉死后、后代只能往空隙里贴**。理解这一点，就不会再问"为什么不一开始就设计成连续的"——因为一开始（286）就是连续的，是扩位时被兼容性逼成了分裂。

### 第 3 步：修 GDT 与 TSS——加载器给的 TSS 是 16 位的

**做了什么**：i386 的 `NEWSYSBG.ASM:235` 注释写明：加载器给的 TSS 是 16 位的，内核要把它改成 32 位（改描述符字节为 089h）。**为什么**：16 位 TSS 与 32 位 TSS 的格式不同（32 位 TSS 才有 Esp0、SS0、IOPM 这些字段）；内核跑在 32 位保护模式下，特权级切换要靠 TSS 里的 Esp0 找内核栈——16 位 TSS 没有这个字段，硬用会错。

**为什么 AMD64 省了这一步**：AMD64 的 `START.ASM` 头注释明言 GDT/TSS 已"completely initialized"、LoaderBlock 已重排 64 位——加载器契约换代了。AMD64 的 IDT 由 C 代码 `KiInitializeBootStructures` 按 `KiInterruptInitTable` 现填，没有静态宏表可修；long mode 下也没有 16 位 TSS 这一物种（任务门被 IST 取代）。**这不是 AMD64 更"高级"，是新平台的加载器契约把旧债一次性还清了**——32 位要修，是因为它的加载器契约定型于 16/32 位过渡期，历史包袱只能由内核自己扛。

### 第 4 步：立 PCR/PRCB——每核的"身份证"为什么必须先立

**做了什么**：`KiInitializePcr`（i386）/ `KiInitializeBootStructures`（AMD64）填 PCR/PRCB：核号、自指、GDT/IDT/TSS 地址、当前线程指向 Idle 线程、DPC 栈指针、就绪队列、锁登记表。DPC 栈在此换正：P0 在 `KiInitializeKernel` 的 P0 分支里（`KERNLINI.C:918`）用 `MmCreateKernelStack` 分正式 DPC 栈、`:923` 写入 `Prcb->DpcStack`（`KiInitializePcr` 在 `:1081` 再写入 `Pcr->PrcbData.DpcStack`）——此前借用的 DF 栈至此退役；AP 不走这段，它的 DPC 栈由 P0 在 `KiInitProcessorState` 里预制（第 10 步）。**为什么必须在点火早期立**：内核代码寻址本核状态全靠 PCR（i386 的 FS、AMD64 的 GS）；没有 PCR，`KeGetCurrentPrcb`、`KeGetCurrentThread`、IRQL 读写全部无从谈起。PCR 是"内核在每个核上找东西的总入口"，不先立它，后面每一步都没有落脚点。

**为什么每核一份、不能共用**：调度状态（当前线程、就绪队列、DPC 队列）天然是每核的；两个核共用一张台账，每步调度都要跨核加锁，等于把多核的并行性又锁回单核。GDT 也必须每核一份：GDT 里有本核的 TSS 描述符与 PCR 选择子，共用一张就指不明白"我的 TSS 在哪"。IDT 每核一份是注释原话（`KERNLINI.C:987`："Note that each processor has its own IDT, GDT, and TSS as well as PCR!"，`KERNLINI.C:1078` 的 `Pcr->IDT = Idt` 落实）——为什么每核一份，解读是"顺带统一 + 独立打补丁"：各核从同一张已修正的表克隆（起点统一），此后各核的 IDT 独立、某核打补丁不影响别核（此为解读，注释只确立"每核一份"的事实，未给理由）。

### 第 5 步：降到 APC_LEVEL——为什么非得是这一级

**做了什么**：P0 在 `KE/I386/KERNLINI.C:573` 执行 `KeLowerIrql(APC_LEVEL)`，然后跑 `KiInitSystem` 与 `ExpInitializeExecutive` 的 Phase 0。

**为什么是 APC_LEVEL，而不是 PASSIVE_LEVEL**：降 IRQL 只摘屏蔽、不召唤任何东西（见易混点第 3 条）。PASSIVE_LEVEL 与 APC_LEVEL 的唯一差别，是 APC_LEVEL 多挡一道：**APC 软件中断**。此时 Idle 线程的 ApcState 还没建好（ApcState 是 Phase 0 里 `KeInitializeThread` 一系才建的），如果停在 PASSIVE_LEVEL，任何 APC 例程都可以在此线程上执行——而这个线程的 APC 接应结构还没建好。停在 APC_LEVEL 买来的是结构保证：**在 Idle 线程的 APC 状态建好之前，无任何 APC 例程能在此线程执行**。这就是"降到刚好够用"的原则：PASSIVE 是"全开"，APC_LEVEL 是"只挡 APC"——点火期要的就是"只挡 APC"这一档。

**为什么不是停在更高（比如 DISPATCH_LEVEL）**：Phase 0 的入口契约就是 APC_LEVEL。两个硬约束都卡在这一级：① 紧随其后的 UP-HAL 探测要 `KfAcquireSpinLock`（`KERNLINI.C:617`），自旋锁的铁规矩是获取时 IRQL ≤ DISPATCH_LEVEL——APC_LEVEL（1）满足，DISPATCH_LEVEL（2）也满足，但这只是"不高于 DISPATCH"，不是"必须是 APC"；② 真正钉死 APC_LEVEL 的是池分配红线（第 8 步）：Phase 0 里要分配分页池，而分页池要求 IRQL ≤ APC_LEVEL。所以 APC_LEVEL 是"Phase 0 能用的最高一级"与"分页池要求的最低一级"的交点——**能高则高、降到刚好够用，P0 的"刚好"就是 APC_LEVEL**。

**KiInitSystem 本身不挑 IRQL（辨析）**：`KIINIT.C` 的 `KiInitSystem` 本体只有五类填字段动作（InitializeListHead / KeInitializeSpinLock / KeInitializeDpc + KeSetTargetProcessorDpc / KeInitializeEvent / ExInitializeFastMutex / SSDT 指针 + RtlCopyMemory），不分配、不等待、不拿锁，本身不需要宽松 IRQL。`KERNLINI.C:573` 的下降是为整个 P0 段降的，不是为它一个函数降的。把"段的要求"安到"函数头上"，是旧版笔记的错（见第七部分第 2 条）。

### 第 6 步：KiInitSystem 与三个收尾函数——"填字段"为什么值得单列

`KIINIT.C` 有四个函数：`KiInitSystem`（填 PCR/PRCB 的收尾字段）、`KiInitSpinLocks`（把全局自旋锁逐个登记进本核 PRCB 的 LockQueue——"地面工程"）、`KeInitSystem`（收尾）、`KeNumaInitialize`（NUMA 节点初始化，定义在 `KE/KIINIT.C:559`，调用点在 Phase 0 的 `INITOS.C:576`）。为什么单列它们：因为 `KiInitializeKernel` 的主体在架构文件（i386 的 `KERNLINI.C`、AMD64 的 `INITKR.C`），而这三个是架构无关的收尾——分层的意思是：**架构相关的点火（查 CPU、修表）归架构文件，架构无关的台账收尾归 KIINIT.C**。读代码时按文件名分工找，比按调用顺序找快。

### 第 7 步：Phase 0 点火总控与 Phase 1 线程——"线程先建、门后开"

**做了什么**：`ExpInitializeExecutive` 的 Phase 0（`INIT/INITOS.C`）按顺序拉起各执行体子系统的 Phase 0：对象管理器、内存管理器（`MmInitSystem(0, …)`）、安全引用监视器等；期间 `PsInitSystem` 创建 Phase 1 系统线程（`PS/PSINIT.C:507`，`PsCreateSystemThread(…, Phase1Initialization, LoaderBlock)`）。

**为什么 Phase 1 要做成一个"线程"而不是直接调用**：Phase 0 与 Phase 1 的分界，本质是"能不能分页、能不能等待"的分界。Phase 0 跑在启动早期：分页机制刚上线、很多结构还是静态的；Phase 1 要做的是"正常内核代码"（可以等待、可以分页、可以创建系统线程）。把 Phase 1 包成一个线程，调度器就能用统一的机制（就绪队列、优先级、等待）来运行它，而不用为启动期另写一套驱动逻辑。**线程是内核的通用执行载体，启动后期的一切工作都复用它**。

**为什么线程已经建好、却还不能跑（接第 15 步）**：Phase 1 线程在 Phase 0 里一创建就进了就绪队列——"就绪"只意味着对象存在、调度器能选中它，不意味着系统已就绪。它能不能真的跑起来，取决于 Idle 线程的优先级盾牌（第 15 步）。顺序是刻意排好的：先建好线程（对象就绪），再护住启动路径（Idle = 31），收尾后开闸（Idle = 0），Idle 循环里第一次调度就切到 Phase 1 线程。

### 第 8 步：池分配红线——为什么分页池卡在 APC_LEVEL

**做了什么**：`EX/POOL.C:551` 的 `ASSERT_ALLOCATE_IRQL` 是犯规检测，不是要求——分页池分配时 IRQL > APC_LEVEL → `BAD_POOL_CALLER` 蓝屏；非分页池分配时 IRQL > DISPATCH_LEVEL → 同样蓝屏。所以规则是：**分页池 ≤ APC_LEVEL、非分页池 ≤ DISPATCH_LEVEL**（`>` 严格大于，档位本身永远合法）。

**为什么分页池的红线比非分页池低两档**：分页池的内存在内存不足时可以被换出到磁盘；分配分页池可能触发缺页——缺页处理要等 I/O 完成（等磁盘把页读回来），而等待只能在 IRQL < DISPATCH_LEVEL 完成（DISPATCH_LEVEL 以上不许等待、不许调度）。所以分页池的红线钉在 APC_LEVEL：再高一级（DISPATCH_LEVEL），缺页时就等不了 I/O，只能蓝屏。非分页池永不换出，分配不触发缺页等待，所以红线可以放宽到 DISPATCH_LEVEL。**红线的高度 = 该操作最坏情况下需要"等"的东西所允许的最高级别**——这是全书通用的推导法，后几批凡见红线都可照此推。

**为什么断言是"犯规检测"不是"要求"**：`ASSERT_ALLOCATE_IRQL` 只在调试版里检查、犯规则蓝屏；它不阻止你调用，它只在你犯规后把机器停下来告诉你犯规了。把"犯规会蓝屏"读成"调用要求某某级别"，是方向性误读（见第七部分第 5 条）。

### 第 9 步：KiInitialProcess 与"借系统页表"——Idle 进程为什么没有自己的地址空间

**做了什么**：`KiInitialProcess` 不是函数，是 `DECLSPEC_CACHEALIGN EPROCESS KiInitialProcess;`（`KE/KERNLDAT.C:346`；`KiInitialThread` 是 ETHREAD，`:353`）。命名规律：`Ki` + 动词 = 函数，`KiInitial` + 名词 = 初始对象。`NEWSYSBG.ASM:541` 传 `offset _KiInitialProcess`。

**为什么是静态对象**：此时对象管理器、池分配器都不存在，没人能"分配"一个 EPROCESS；只能链接时在内核映像数据段里预留。静态 = 编译时定好地址，启动时直接用。**这是点火早期的通用约束：分配器没上线之前，一切只能用静态的**。

**"借系统页表"两步**：① `KERNLINI.C:668-669` 先写 `DirectoryTableBase[0] = 0` / `[1] = 0` 占位；② MM 上线后，`MM/I386/INIT386.C:1013` 把系统页目录物理帧（DirBase，取自当前 CR3 对应目录）填入 `[0]`（`CurrentProcess->Pcb.DirectoryTableBase[0] = DirBase`），并随即把低 2GB 的 PDE 清零——**Idle 进程的页目录就是系统页目录本身**（只映射系统空间；系统空间在所有进程页目录中映射一致）。（[1] 的用途与后续填充点未在树里定位：`INC/KE.H:958` 只声明了 `DirectoryTableBase[2]` 数组、无注释，`INIT386.C` 只回填 [0]——见未钉死清单新增第 8 条。）`PS/PSINIT.C:304` 的 `PsIdleProcess = PsGetCurrentProcess()` 佐证：启动上下文一直活在 Idle 进程里；启动路径本身即 Idle 线程胚胎（CurrentThread 已由 `KiInitializePcr` 指向 `KiInitialThread`，`KiInitializeKernel` 返回后汇编 `jmp KiIdleLoop`），第一个真正被创建的线程是 Phase 1 系统线程。

**为什么 Idle 进程不需要自己的地址空间**：Idle 线程只跑内核代码（`KiIdleLoop`），内核代码只访问系统空间；用户地址空间对它毫无意义。给它一套独立页表纯属浪费——直接借系统页表是最省的方案。低 2GB 清零的意思是：Idle 进程看不到任何用户空间（2GB 以下全空），误访问用户地址会立刻 fault，而不是静默读到别人的数据。

**未钉死项（诚实标注）**：真实线程切到 Idle 时 CR3 如何避免装入占位值——两架构 `SwapContext` 的进程切换段（`CTXSWAP.ASM:551` 的 `mov cr3`）表面上无条件装 CR3，与 0 的衔接点未在树里定位到确切行号。但 MM 回填 DirBase 后 `DirectoryTableBase[0]` 已是真实系统目录帧，占位 0 在运行期不复存在——回填事实本身已足够解释运行期不崩。行号问题留待后续有机会回填，不编造。

### 第 10 步：拉 AP——P0 为什么要替 AP 预制整套每核数据

**做了什么**：P0 在 Phase 1 里调 `KeStartAllProcessors`（`KE/I386/ALLPROC.C`），逐个拉 AP。每拉一个，先由 `KiInitProcessorState`（`ALLPROC.C:189`）预制该 AP 的全套数据，再由 HAL 叫醒它（第 11 步）。预制清单：一块连续内存里的 PCR（清零的 KPCR）、TSS、NMI TSS、双重故障 TSS 与 DF 栈、Idle 线程对象（ETHREAD）空位；克隆的 GDT/IDT；`MmCreateKernelStack` 分好的 Idle 栈与 DPC 栈；上下文帧（Eip = `KiSystemStartup`、Esp = 新栈、栈顶预压 `KeLoaderBlock` 指针、段选择子）；最后把 `LoaderBlock` 的 KernelStack / Thread / Prcb 三字段改写成这个 AP 的三件套。

**为什么 AP 不能自己填——鸡生蛋问题**：AP 要填这些结构，得先有地方站着、有栈可跑、有 PCR 可寻址，而这些正是它要填的东西。AP 被 SIPI 叫醒的那一刻几乎一无所有：一个被 HAL 指好的程序计数器，除此之外没有栈、没有 PCR、没有自己的 GDT/IDT/TSS、没有分配器。它连一行 C 代码都跑不起来——C 代码要栈，要用 FS/GS 找 PCR，要段寄存器有合法的 GDT。`KiSystemStartup` 的全部工作就是"建立这些"，但建立的前提是内存已经留好、指针已经交到它手里。唯一能打破循环的，是一个已经完全跑起来的核：P0。

**分工表（P0 负责分配与布局，AP 负责初始化与激活）**：

| P0 预制（容器 + 入口） | AP 自己填（内容 + 激活） |
|---|---|
| 连续内存块：PCR、TSS、NMI/DF TSS、DF 栈、ETHREAD 空位 | `KiInitializePcr` 填 PCR/PRCB 字段（核号、自指、DPC 栈指针……） |
| 克隆的 GDT/IDT | `KiInitSpinLocks` 建自己的就绪队列、DPC 数据、锁登记表 |
| MmCreateKernelStack 分的 Idle 栈、DPC 栈 | `KiSetProcessorType` 等查自己的 CPU 型号、特性位，与 P0 对账 |
| 上下文帧：Eip/Esp/段选择子/栈顶 LoaderBlock 指针 | `KeInitializeThread` 把预留的 ETHREAD 空位填成真正的 Idle 线程对象 |
| LoaderBlock 三字段改写成该 AP 的三件套 | 走完 AP 分支后把 `LoaderBlock->Prcb` 清零——给 P0"签收" |

**为什么 LoaderBlock 是逐个改写的（串行握手的根源）**：LoaderBlock 是全系统唯一的一块"投递单"，P0 一次只能给一个 AP 改写三个字段、等它签收（Prcb 清零）再改下一个。这就是 `KeStartAllProcessors` 里 `while (KeLoaderBlock->Prcb != 0) KeYieldProcessor();` 的由来——AP 必须一个一个拉，不是因为 AP 脆弱，是因为投递单只有一张。

**一个同构的比喻**：P0 对 AP 干的事，和 ntldr 对 P0 干的事是同一件事——ntldr 给 P0 搭好 GDT/IDT/页表/栈，再跳进 `KiSystemStartup`；P0 给 AP 搭好 GDT/IDT/PCR/TSS/栈，再（经 HAL 的 SIPI）把它送进同一个 `KiSystemStartup`。**P0 就是 AP 的 bootloader**。

### 第 11 步：HAL 的 SIPI——叫醒 AP 的硬件协议

**做了什么**：内核调 HAL 接口 `HalStartNextProcessor(LoaderBlock, ProcessorState)`（`INC/HAL.H:645`；AMD64 的 `KE/AMD64/ALLPROC.C:486` 在树里能直接看到这个调用；i386 的 `KiInitProcessor` 本体不在 WRK 树里，只剩声明，HAL 本体也不在树里，只有预编译 DLL——HAL 内部怎么写 ICR，树里没法核实，此处按 Intel MP 规范与接口语义陈述）。HAL 往 P0 的本地 APIC 的 ICR（中断命令寄存器）里写：目标 = 该 AP 的 APIC ID（HAL 从 ACPI/MP 表知道每个核的编号）、投递模式、向量；并事先在低端内存对应页放好跳板代码。

**完整序列 INIT-SIPI-SIPI（为什么是三发）**：
1. **INIT IPI**（投递模式 101）：把 AP 复位到已知状态、清掉内部状态，进入"等 SIPI"状态。相当于先按复位键，但不让它跑起来。
2. **SIPI**（投递模式 110，带 8 位向量）：AP 收到后以实模式启动，入口地址 = 向量 × 0x1000。向量是页号，只能指向头 1MB 内的某一页（实模式只能寻址 1MB）——比如向量 0x08，就是从物理地址 0x8000 开始执行。
3. **再发一次 SIPI**：规范要求的防御性重复。AP 处理 INIT 需要一点时间窗口，第一发 SIPI 可能在这个窗口里丢掉；第二发保证它一定能收到。不是"启动两次"——AP 只会响应一次有效的 SIPI，第二发到时它已经醒了，不会重复执行。

**跳板代码干什么**：16 位实模式代码，关中断 → 装最小 GDT → 置 CR0.PE 进保护模式 → 远跳到 32 位代码 → 按 P0 预制的 ProcessorState 装好寄存器（Esp 指向新栈、Eip = `KiSystemStartup`、栈顶已压好 LoaderBlock 指针）→ 跳进 `KiSystemStartup`。跳板是"实模式与保护模式之间的摆渡"，它的全部使命就是把 AP 从"刚醒的实模式"送到"能跑内核 C 代码的保护模式"。

**SIPI 与普通 IPI 的区别（易混）**：普通 IPI（比如 `HalRequestIpi`）是给已运行的核发中断，向量是中断号、走 IDT 分发；SIPI 只对"等启动"的核有效，它的向量不是中断号、而是启动页号，不走 IDT。把两者混为一谈，会误以为 SIPI 也要查 IDT——它查的是低端内存的跳板页。

### 第 12 步：AP 的命令循环与 CPU 对账——为什么拉起来还要先"审"一遍

**做了什么**：AP 的 `KiSystemStartup` 在 `NEWSYSBG.ASM` 的 `kiss_notp0` 处调 `KiProcessorStart`（`KE/I386/ALLPROC.C:989`），进命令循环等 P0 的指令。P0 先下 `KcStartGetId`：让 AP 自己执行 CPUID，把型号、特性位回填到 `KiProcessorStartData`，P0 读回后与自己的特性逐项对账；对账通过才下 `KcStartContinue` 放行（AP 继续进 `KiInitializeKernel` 的 AP 分支）；对账不过、或 P0 判定该核不该启动，则下 `KcStartDoNotStart`：AP 先由 `KiSetHaltedNmiandDoubleFaultHandler`（`KE/I386/ALLPROC.C:1078`）把 NMI/DF 任务门指向的 TSS 入口改写成两个占位处理函数（`KiDummyNmiHandler` / `KiDummyDoubleFaultHandler`），再关中断、`hlt` 永久停机。占位处理只干两件事：清掉对应 TSS 描述符的 busy 位、继续 halt——因为 NMI 是关不掉的，万一打到这颗已停机的核身上，CPU 会走任务门切进 NMI TSS；busy 位不清的话，下一次 NMI 再切同一个 TSS 就会 #TS → 双重故障 → 三重故障 → 整机复位。**停机不是"不管了"，是"连出事的路径都给堵死"**。

**为什么要对账、全场按最弱的核对齐**：SMP 的前提是"所有核看到的指令集与行为一致"。如果某个核缺一条指令（比如 cmpxchg8b，见第 13 步）、或浮点行为不同，内核代码在不同核上跑出不同结果——自旋锁、原子操作、浮点保存全会悄悄错。对账不是"登记一下"，是"不一致就拒绝入场"：要么全场降到最弱核的水平（特性位图取交集），要么有硬线要求的特性缺失就直接蓝屏、不让系统带病跑。**对齐的代价是性能（迁就最弱的核），不对齐的代价是正确性**——内核选了正确性。

**为什么对账要用"命令循环"而不是 AP 自己跑完再汇报**：AP 刚醒时连自己的 PCR 都还没完全立起来（它的 `KiInitializeKernel` 还没跑），不能直接调内核服务汇报；P0 与 AP 之间唯一的通信手段就是共享内存里的两个全局变量（`KiProcessorStartControl` + `KiProcessorStartData`）+ AP 原地 `KeYieldProcessor` 空转等待。这是最原始的核间通信——没有锁、没有队列、没有中断，只有"一个写、一个轮询"。原始，但在这个阶段是唯一可用的。

### 第 13 步：cmpxchg8b——为什么缺一条指令就蓝屏

**做了什么**：`cmpxchg8b` 是 8 字节原子比较交换指令。32 位下，它是唯一的 64 位原子操作。

**为什么 8 字节原子操作不可替代**：无锁 SList 表头是"指针 + 序列号"拼成的 8 字节（`RTL/I386/SLIST.ASM` 满屏 `lock cmpxchg8b`）。为什么表头要带序列号：防 ABA 问题——指针 A 被弹出、又被压回，看起来没变，实际上中间换过一轮；没有序列号，无锁算法会把"换过一轮"误判成"没变过"，悄悄丢数据或死循环。8 字节一次原子更新（指针 4 字节 + 序列号 4 字节）是 SList 正确性的根基；没有 `cmpxchg8b`，32 位下没有任何单条指令能原子地换 8 字节。`InterlockedCompareExchange64`、无锁队列（`EX/NBQUEUE.C`）同理。

**为什么 NT4 能忍、XP 要划硬线（历史）**：NT4 时代有自旋锁 fallback（`Ki486CompatibilityLock`，`KERNLINI.C:267` 附近，至今留作 UP-HAL 探测的道具）——缺指令的核走加锁慢路。但 fallback 意味着内核所有无锁路径都要留一条加锁分支，维护成本与正确性风险都高。XP 起划硬线（`KERNLINI.C:544` 注释原话："As of Windows XP, cmpxchg8b is a required instruction"）：P0 缺失 → `UNSUPPORTED_PROCESSOR`；AP 缺失（且 P0 有）→ `MULTIPROCESSOR_CONFIGURATION_NOT_SUPPORTED`（`KERNLINI.C:729-735`）。**划硬线的逻辑是：当缺指令的核已经绝迹（奔腾以上全有），保留 fallback 的成本高于收益，不如直接拒绝**——这是"兼容性债"还清后的正常收口，不是傲慢。

**为什么兼容 CPU 要"藏位"、内核要"正名"（NT4 的错）**：`KERNLINI.C:1612` 注释原话："CMPXCHG8B MUST be done in a generic manner or clone processors will not be able to boot if they set this feature bit. This was incorrect in NT4 and resulted processor vendors claiming not to support cmpxchg8b even if they did." 注释只确立了因果（NT4 的处理方式非通用 → 厂商藏位自保），NT4 那个"不正确"的具体机制在 WRK 树里没有 NT4 代码、无法核实；最合措辞的推断（NT4 掺了厂商/型号假设，克隆 CPU 亮位即出问题）只能标注为推断，不能写成事实。厂商的应对是出厂屏蔽 CPUID 里的 cmpxchg8b 位（硬件有、但对外说没有）；内核的应对是按厂商逐家"正名"：Transmeta 写 MSR 80860004、Centaur 写 MSR 107/1107 的位 1、Rise 无机制直接当有。**这是一场"检测方式错了 → 厂商撒谎 → 内核逐家拆谎"的三方拉锯，根子在 NT4 的检测不够通用**。

### 第 14 步：栅栏放行——为什么 AP 初始化完还不许跑

**做了什么**：`KeStartAllProcessors` 拉 AP 之前先立 `KiBarrierWait = 1`（`KE/I386/ALLPROC.C:567`）；每个 AP 初始化完进 Idle 循环前要过栅栏；P0 拉完所有核后清零放行（`ALLPROC.C:617`），所有核一起开跑。

**为什么要有栅栏**：AP 初始化完时，P0 可能还没拉完剩下的核、系统还没到"所有核就绪"的状态。如果 AP 提前进 Idle 循环、开始接 DPC、偷线程，它会在系统还没准备好的时候参与调度——比如 P0 还在改写 LoaderBlock（投递单），AP 却已经开始跑 Phase 1 之后的东西。栅栏的意思是："都到齐了再开饭。"这是一个启动期的全局同步点，用一个变量实现，因为此时用正式同步原语（事件、信号量）还太早。

### 第 15 步：收尾——先升 DISPATCH，再把 Idle 优先级归零（开闸）

**做了什么**：每核的 `KiInitializeKernel` 尾巴（`KE/I386/KERNLINI.C:943-944`；AMD64 的 `KE/AMD64/INITKR.C:719`）：

```c
KeRaiseIrql(DISPATCH_LEVEL, &OldIrql);
KeSetPriorityThread(Thread, (KPRIORITY)0);
```

此前，Idle 线程顶着 HIGH_PRIORITY（`KERNLINI.C:868` 的 `Thread->Priority = HIGH_PRIORITY`）。

**为什么先升到 DISPATCH_LEVEL 再改优先级**：改优先级是调度敏感操作——它动的是调度器用来选线程的字段。调度 DPC 本身跑在 DISPATCH_LEVEL；先把 IRQL 升到 DISPATCH，就是把调度 DPC（以及一切 DPC）挡在门外，保证"改优先级"这个动作不被调度器插一脚——改完回到汇编，`KiIdleLoop` 本身就是在 DISPATCH_LEVEL 上跑的，衔接正好。**升 IRQL 是给"改调度状态"加的原子性**：调度器的状态，只能在调度器跑不到的时候改。

**为什么 Idle 要顶 HIGH_PRIORITY、又为什么尾巴才归零（"线程先建、门后开"的收束）**：Phase 1 线程在 Phase 0 里一创建就进了就绪队列（第 7 步）。如果 Idle 的优先级是 0，调度器下次一跑（DPC 形式；Phase 0 时 IRQL 是 APC_LEVEL，低于 DISPATCH，所以 DPC 能跑）就会发现"有个就绪线程优先级高于当前线程 0"，于是切走——切到 Phase 1 线程，而此时 P0 的尾巴还没干完：定时器倒数没算、DPC 栈还是借来的双重故障栈、AP 的 `LoaderBlock->Prcb` 还没清零签收。Phase 1 线程的设计前提就是"点火总控的 P0 调用栈已经退干净、CPU 已进 Idle 循环"，提前跑等于提前开工，会撞上这些半成品。

盾牌的机制就是调度器的抢占规则：只有优先级高于当前线程的就绪线程才能抢走 CPU。Idle 顶着 HIGH_PRIORITY（31，最大值），任何就绪线程都抢不动它——DPC 里调度器跑一圈，发现没人比 31 高，只能让当前线程（也就是启动路径）继续跑。注意 IRQL 挡不住这个：DPC 在 DISPATCH_LEVEL 照样跑，调度 DPC 照样会看就绪队列；能挡住切换的只有优先级比较这一关。**所以"不被任何就绪线程抢走本核"的准确含义是：不是"没有线程"，是"有线程也抢不动"**。

降到 0 就是开闸：点火收尾全部干完之后，才把 Idle 恢复成名义上的 0，让调度器下次一跑就选中 Phase 1 线程。汇编返回后进 `KiIdleLoop`，第一件事就是找就绪线程切过去——Phase 1 线程这才正式开跑。AP 同理：AP 的 Idle 在自己的 `KiInitializeKernel` 里同样顶着 31，防止握手签收前被抢走；尾巴同样降 0，只不过 AP 没有 Phase 1 线程可切，降完就纯待命，等 P0 派活（IPI/DPC）。

**为什么 AP 停在 DISPATCH_LEVEL、比 P0 的 APC_LEVEL 高一级（旧版写反了，已纠正）**：APC_LEVEL = 1，DISPATCH_LEVEL = 2；P0 是 31 → 1，AP 是 31 → 2——**AP 少降一级、停得更高**。两个原因：① AP 的剩余工作（`KeInitializeThread`、AP 分支、栅栏自旋、进 Idle 循环）只需 ≤ DISPATCH，降下去还要升回，多此一举；② 更关键的是，APC_LEVEL 会放行 DPC（DPC 在当前 IRQL < DISPATCH 时投递、升至 DISPATCH 执行）——栅栏自旋中的 AP 若在 APC_LEVEL，P0 的跨核 DPC、定时器 DPC 会在 AP 半就绪时打断自旋、执行 DPC 代码；停在 DISPATCH 则 DPC 与 APC 全挡，AP 是单纯等待者。**停多高，取决于"剩余工作需要多低"与"不想放进来什么"——AP 的答案是 DISPATCH_LEVEL**。

### 第 16 步：KiIdleLoop 不是傻转——"空转"与"待命"的区别

**做了什么**：`KE/I386/CTXSWAP.ASM:1303-1537` 的 `KiIdleLoop` 每轮四查：① DPC 队列（`KiRetireDpcList`，调度/定时器 DPC 在此跑，是切到真线程的出口）→ ② `Prcb->NextThread` 非空则 `SwapContext` 切走 → ③ IdleSchedule 置位则 `KiIdleSchedule` 偷别核线程 → ④ 皆无则调 HAL 的 `PpIdleFunction` 省电；DBG 版还轮询调试器。

**为什么 Idle 线程必须"循环查"而不是"睡死等中断"**：调度器与 DPC 是软件机制，没有"有活干时自动叫醒我"的硬件信号；Idle 线程只能主动轮询——查 DPC 队列、查 NextThread、查 IdleSchedule。这不是低效，是"待命"的唯一实现：CPU 在跑 Idle 线程（调度视角：CurrentThread = Idle 对象、IP 在 `KiIdleLoop`）与"空转"（代码视角：循环里没干活）是同一事；空转 = 待命，不是卡死。

**HAL 的省电函数与 sti;hlt 的原子性**："停机"指核心级 HLT（停取指、等中断，机器其他部分照常），不是关机、不是 ACPI 睡眠状态（S3 挂起到内存、S4 挂起到硬盘那种整机睡眠）。睡多深是平台策略（ACPI C1 = HLT、C2 停时钟、C3 处理缓存/停总线主控、芯片组握手）→ HAL 的地盘。接口是 `PRCB.PowerState.IdleFunction` 函数指针（`INC/PROCPOWR.H:43`，即 `_PROCESSOR_POWER_STATE` 的 `IdleFunction` 字段），HAL 初始化时登记，`KiIdleLoop` 调用（`CTXSWAP.ASM:1328` 的 `PpIdleFunction` 就是这个指针的汇编层名字，两者是同一个指针，不是两个函数）。防丢活握手（HAL 注释原文精神）：HAL 被调用时中断关闭、返回时中断开启，靠 `sti;hlt` 的 x86 保证（sti 后一条指令内不响应中断）原子地"开中断 + 睡"——没有这个原子性，"开中断"与"睡"之间插进来一个中断，醒来后回环重查没问题，但反过来（先睡、后开中断）会在中断已到、CPU 却已睡的窗口里丢活。HAL 的 Idle 实现本身不在 WRK 树里（只有接口与调用点可见）。

### 第 17 步：__security_cookie——把"线性溢出改返回地址"变成崩溃

**定位**：检测，不是预防。GS 的目标是把"线性栈溢出 → 改写返回地址 → 控制流劫持"变成"校验失败 → 立刻崩溃"。它不管堆溢出、不管只改数据不改控制流的攻击、不管信息泄露 + 帧指针伪造后的绕过——这些是边界，不是疏忽（见文末边界小节）。

**布局（低地址 → 高地址）**：[局部变量（数组离哨兵最近）][cookie][保存帧指针][返回地址]。为什么数组离哨兵最近：线性溢出（strcpy 这类从低往高写）必先踩哨兵、再踩帧指针、最后踩返回地址——哨兵是第一道、也是必经的一道。

**存与验**：栈上存的值 = 全局 cookie ^ 帧指针（x86 ^ EBP、x64 ^ RSP）；校验在所有 return 路径的 epilogue 里，由 `__security_check_cookie` 做（栈值 ^ 帧指针 vs 全局值），失败 → `__report_gsfailure`。为什么要 ^ 帧指针：纵深防御，不是秘密（帧指针就在栈上可见）——它让攻击者不能只泄露一个全局 cookie 就全函数通用，每帧的栈上值都不同。

**失败后果（用户态 vs 内核态）**：用户态失败 = `STATUS_STACK_BUFFER_OVERRUN`（0xC0000409）、进程终止，业务代码 catch 不住（故意：让溢出无法被业务异常处理吞掉、继续跑）。内核态失败 = 蓝屏。**WS03 时代内核版 `__report_gsfailure` 的实现不在 WRK 树、具体 bugcheck 码未钉出**——Win8 后的 0x139（KERNEL_SECURITY_CHECK_FAILURE）只能作版本注脚，不能回套到 WS03。

**AMD64 内核 cookie 生成（已实证，`KE/AMD64/START.ASM:250-264`，仅 P0，`KiInitializeKernel` 返回后）**：`rdtsc` 取熵 → `shl/or` 合并 → `ror 49 / xor` 掺混 → `rol 16 / mov ax,0 / ror 16`。验算净效果：高 16 位清零、留 48 位熵。为什么顶部两字节清零：小端下 cookie 顶部字节为 0x00，`strcpy` 类 payload 不能含 0 字节——字符串溢出穿不过这两字节去伪造 cookie（`memcpy` 类溢出不受此限，这是 GS 的已知边界）。补数（`not` 后存 `__security_cookie_complement`）的用意：全局值自校验（cookie ^ complement 应恒全 1，只改一处即露馅）。用户态 CRT 的熵源是时间 / PID / TID / 性能计数器混合，算出 0 则用默认值 0xBB40E64E。

**编译器插桩启发式**：有栈数组、或取地址的局部变量才插，纯算术函数通常没有——GS 不是"每个函数都有哨兵"，是"编译器判断可能溢出的函数才有"。/GS 自 VS2003 默认开。

**i386 内核 cookie 初始化点（未钉死，诚实标注）**：在 WRK 树里没翻到（KE/I386 无引用，疑在链接进内核的 CRT 库）。AMD64 的点在 START.ASM 能实证，i386 的点不能——两架构不对称，照实写，不把 AMD64 的结论平移到 i386。

---

## 第四部分：32 位 vs 64 位对照（每项讲清为什么只能 / 可以这样）

### 1. IRQL 档数：32 档 vs 16 档

- i386：HIGH_LEVEL = 31（`INC/I386.H:170`），共 32 档，其中 24 档是 DIRQL（设备级）。
- AMD64：HIGH_LEVEL = 15（`INC/AMD64.H:354`），共 16 档。

**为什么 32 位当年要搞 32 档**：历史顺序是反的——32 档是 1989–1993 年 NT 的跨架构软件抽象（x86 / MIPS / Alpha / PowerPC 通用），早于 APIC 的 4 位优先级类（Pentium 时代才普及）。当时的设计意图：留 24 个 DIRQL 做设备优先级精细化（不同设备可以互相抢占、排优先级），顶档 PROFILE / CLOCK / IPI / POWER / HIGH 是内核关键段级别。**32 档是"软件先行、硬件后到"的产物**，不是为 APIC 设计的。

**为什么 32 位后来不改编号**：IRQL 常量是全系统 ABI 契约——内核、HAL、全体驱动都用 `DISPATCH_LEVEL`、`DIRQL` 这些具名常量，但常量背后的数值编进了已发布的驱动二进制。改编号的收益只是"去掉一张能 work 的翻译表"，代价是破坏全体现存驱动的二进制兼容。**收益 < 代价，所以不改**。

**为什么 64 位能改成 16 档**：新平台零兼容负担（没有已发布的 x64 驱动要迁就），且 CR8 使"IRQL 值 = 硬件值"成为可能（下一项）。改编号在 64 位是免费的——因为驱动本来就要重写（见第 5 项），编号只是顺带统一。

### 2. IRQL 的实现：翻译表 vs CR8

- i386：没有可用的控制寄存器存 IRQL（CR0–CR4 各有其主，CR1 是禁区）。实际实现是 APIC 的 TPR（任务优先级寄存器，MMIO 地址 0xFFFE0080）：`KfRaiseIrql` / `KfLowerIrql` 查 `HalpIRQLToTPR[IRQL]` 再 MMIO 写（带屏障 + 读回），`KeGetCurrentIrql` 读 TPR 后查 `HalpVectorToIRQL[tprValue/16]`（`INC/KE.H:3465-3483`）。两张表由 HAL 经 LoaderBlock 的 Extension 递给内核（`NEWSYSBG.ASM:465` 取 `LpbExtension`）——**因为向量 ↔ IRQL 的分配是 HAL 的事**（哪个设备用哪个向量、哪个向量对应哪级，HAL 定）。
- AMD64：CR8 一条 MOV 指令读写（`KE/AMD64/PROCSTAT.ASM`），不需要表。

**为什么 i386 只能用表**：硬件规则是"向量 V 仅当 (V >> 4) > TPR 类时投递"——APIC 的 TPR 只有 4 位有效（类 0–15），而 IRQL 有 32 档。32 档往 16 类里塞，只能是有损多对一单调映射：低三档精确（0 → 0x00、1 → 0x10、2 → 0x20），HIGH_LEVEL（31）→ 0xF0（类 15 挡全部）。多挡无害（只延迟、不破坏互斥——同类设备不能互相抢占只是延迟 trade-off，投递顺序仍按向量号）、少挡不可能（单调性：一旦某类被挡，低于它的全被挡）、往返一致（IRQL 数值可在同类内漂移 N → N'，硬件屏蔽状态不变，内核依赖的是屏蔽状态不是数值）。**DIRQL 具体哪几档共用哪类，取决于 HAL 运行时的向量分配，WRK 树里无精确表值——已如实说明，不编表**。

**为什么 AMD64 能省表**：两个条件同时齐备——① CR8 是架构规定的 TPR 镜像（CR8 与 APIC TPR 硬件同步，写 CR8 就等于写 TPR）；② IRQL 档数压到 16，与 4 位字段对齐。缺任何一个都省不了：没有 CR8，纵有 16 档也得走 MMIO；没有 16 档，纵有 CR8 也得换算。**CR8 是 AMD 给的架构礼物，16 档是 NT 为接这份礼物付的代价**。

**更深一层：就算有空 CR 也没用**：真正挡中断的是 APIC 的 TPR；除非架构规定某个 CR 是 TPR 的镜像，否则往 CR 里写值不会影响中断投递。AMD64 的 CR8 之所以行，是因为架构白纸黑字规定了它与 TPR 同步——不是"找个空寄存器凑合"。

### 3. IDT 表项：8 字节 vs 16 字节，任务门 vs IST

- i386：IDT 表项 8 字节，NMI/双重故障靠任务门 + NMI/DF TSS 换干净栈。
- AMD64：IDT 表项 16 字节（偏移 64 位），NMI/双重故障靠 TSS 里的 IST 栈指针。

**为什么 64 位能用 IST**：64 位地址放不进 8 字节表项（偏移要 64 位），表项必须扩到 16 字节；扩表项的同时，AMD 在 TSS 里加了 IST——IDT 表项直接指定 IST 栈号，效果与任务门相同但更轻量（任务门要换整个任务状态，IST 只换栈）。**IST 是"表项扩位"顺带给的更优解**，i386 没有 IST，只能用任务门这个重武器。

### 4. 加载器契约：修半成品 vs 现填

- i386：加载器给的 IDT 是静态宏表（字段反了要换）、TSS 是 16 位的（要改 32 位）——内核花一整段代码修加载器的半成品。
- AMD64：加载器契约换代（GDT/TSS 已完全初始化、LoaderBlock 已重排 64 位），内核的 IDT 由 C 代码按表现填，无静态宏表可修。

**为什么 32 位只能修、64 位可以现填**：32 位的加载器契约定型于 16/32 位过渡期，历史包袱只能由内核扛；64 位是新契约，加载器与内核可以重新约定"谁把活干干净"。**契约的版本，决定了内核要修多少前任的活**。

### 5. 驱动 ABI：为什么 64 位跑不了 32 位驱动

加载器查 PE 机器类型，直接拒 I386 映像。根因是驱动二进制 ABI 全变：指针 4 → 8 字节（DRIVER_OBJECT / DEVICE_OBJECT / IRP / MDL 布局全变）、调用约定（x64 是 RCX/RDX/R8/R9 + 影子空间——调用者要在栈上给被调用者预留 32 字节、存放前四个寄存器参数的"回家位置"）、IDT 表项 16 字节 + IST（32 位 ISR 无法被分发）、HAL 接口、IRQL 编号。

**为什么兼容分界划在用户态 / 内核态**：用户态有 WOW64（CPU 兼容模式 + syscall 翻译，有干净边界——用户代码只通过系统调用号与内核打交道，翻译层可以逐个翻译）；内核态无翻译边界（驱动直摸内核结构、直接中断，翻译层无处下手）。所以用户态可以兼容、内核态只能干净分界。**IRQL 改编号"免费"，是因为驱动用具名常量、源码兼容、只破坏本已被其他因素破坏的二进制兼容**——编号从来不是分界的真正原因，指针与调用约定才是。

**历史注脚**：2005 年 XP x64 驱动荒；驱动签名与 PatchGuard 随 x64 而来——分界干净了，管控才立得起来。

### 6. 安全 cookie 初始化：AMD64 实证 vs i386 未找到

AMD64 内核 cookie 初始化在 `KE/AMD64/START.ASM:250-264` 能实证（仅 P0，`KiInitializeKernel` 返回后）；i386 内核 cookie 初始化点在 WRK 树里没翻到（KE/I386 无引用，疑在链接进内核的 CRT 库）。**两架构不对称，照实写，不把 AMD64 的结论平移到 i386**。

---

## 第五部分：单核 vs 多核对照（为什么单核可以省、多核必须有）

### 1. UP 机器上，哪些步骤直接消失

没有 AP 可拉：`KeStartAllProcessors` 循环零次；没有 `KiProcessorStart` 命令循环、没有 CPU 对账、没有栅栏、没有 `KiBarrierWait`。P0 的点火路径在 Phase 1 里直接收尾。

### 2. MP 内核跑在 UP 机器上：UP HAL 探测与运行时改代码

**做了什么**：点火早期要探测 HAL 是 UP 还是 MP（`KERNLINI.C:617` 附近的 UP-HAL 探测要 `KfAcquireSpinLock`）。如果 MP 内核撞上 UP HAL（单核机器、但装的是 MP 内核），内核把自旋锁例程的首字节改成 RET——自旋锁在单核上退化为空操作。

**为什么能这么干**：单核上没有第二个核来抢锁，自旋锁的互斥语义天然满足（本核不被抢占时，临界区就是原子的）。把首字节改成 RET，等于"运行时把锁函数变成空函数"——这是热补丁思想的早期实物：**函数入口预留的补丁桩，运行时替换实现而不重启**。为什么不在编译期决定：同一个内核二进制要同时跑 UP 与 MP 机器，编译期不知道目标机器是几核，只能运行时探测、运行时改。

**为什么单核不能反过来省掉"锁"这个概念**：UP 机器上跑 MP 内核可以省锁，是因为"没有竞争者"；但 MP 内核的代码处处按"有竞争者"写，省锁只是运行时优化，不是设计省略。反过来，UP 内核跑在 MP 机器上会直接错——因为 UP 内核的代码根本没写锁，竞争一来就数据错乱。**省，是运行时的退化；有，是设计的底线**。

### 3. AP 的 DISPATCH_LEVEL 与 P0 的 APC_LEVEL（再强调一次方向）

P0 停 APC_LEVEL（1），AP 停 DISPATCH_LEVEL（2）——AP 少降一级、停得更高。原因见第三部分第 15 步：AP 的剩余工作只需 ≤ DISPATCH，且 APC_LEVEL 会在栅栏自旋中放行 DPC。**单核机器没有 AP，这一级的差别根本不存在；多核机器上，这一级别的差别决定了 AP 是"单纯等待者"还是"半就绪时被 DPC 打断的倒霉蛋"**。

---

## 第六部分：逆向 / 安全解读（本批机制的攻防面）

- **IDT 是篡改面**：IDT 表项 = 中断号 → 处理函数。改 IDT 表项就是改"中断来了去哪"——Rootkit 挂钩中断处理的老路。本批的教训：IDT 在启动期是静态宏表 + `KiSwapIDT` 修正，生效后的 IDT 在内存里是可写的（除非后续有写保护）；取证时对照"静态表 + 修正规则"能算出 IDT 应有的样子，与实际 IDT 比对即见篡改。
- **GS cookie 的攻防边界**：GS 只把"线性溢出改返回地址"变成崩溃。绕过面：信息泄露 + 帧指针伪造（栈上值 = cookie ^ 帧指针，帧指针就在栈上可见，泄露全局 cookie 即可按帧伪造）；任意写原语可直接改 cookie 或返回地址而不触发校验（校验只在 return 路径，不在每次写时）；只改数据不改控制流的攻击不报警；堆溢出不管。SEH 归 SafeSEH 管，不归 GS 管。**把 GS 当成"防溢出"全解，是误读；它是"把一类溢出从控制流劫持降级为崩溃"的单点检测**。
- **PsLoadedModuleList 与 DKOM**：`PsLoadedModuleList` 是内核的已加载模块链表；DKOM（直接内核对象操纵）的经典手法是把进程/驱动对象从链表里摘链——`ps` 命令看不见，但对象还在、线程还在跑。取证对照：拿 `PsLoadedModuleList` 和 `KeLoaderBlock` 的原始映像链（加载器留下的、Rootkit 往往顾不上改）比对，差集就是摘链的痕迹。本批的 LoaderBlock 链（第 0 批已讲）就是这条对照链的源头。
- **AP 拉起是信任边界**：SIPI + 跳板 + ProcessorState 是一条"无认证"的启动链——P0 全权决定 AP 从哪执行。任何能往低端内存跳板页、或往 LoaderBlock / ProcessorState 写数据的攻击者，都能在 AP 上执行任意代码。这条链的安全性完全依赖"启动早期还没有攻击者"这个前提；启动完成后的防护（PatchGuard 等）管不到这一段。
- **CR3 与 Idle 进程页表**：Idle 进程借系统页表、低 2GB 清零，意味着 Idle 上下文里访问用户地址必 fault。逆向时看到"某线程的 CR3 指向系统页目录"，先判断它是不是 Idle/系统线程，不要误当成"页表被破坏"。

---

## 第七部分：易混点辨析（旧版笔记与旧回答中已纠正的 5 处，逐条对照）

| # | 旧说法（错） | 正确说法 | 证据 |
|---|---|---|---|
| 1 | "加载器 IDT 字节序反了" | 反的不是字节序、不是加载器的表，是内核静态 `_IDT` 表的字段位置：`dd offset` 使 Selector 与 ExtendedOffset 互换了位置 | `TRAP.ASM:356` 宏体 vs `I386.H:892` 硬件布局；`KiSwapIDT`（`KERNLINI.C:1230`）换回 |
| 2 | "KiInitSystem 需要宽松 IRQL" | 是整个 P0 段需要 APC_LEVEL（Phase 0 契约 + 池红线），KiInitSystem 本身只填字段、不挑 IRQL | `KIINIT.C` 本体五类填字段动作；`KERNLINI.C:573` 下降点在 P0 段 |
| 3 | "降到 PASSIVE 会放 APC 投递和线程调度进来" | 机制错误：降 IRQL 只摘屏蔽、不召唤任何东西；此时无就绪线程、无排队 APC、时钟未启动，调度器与 APC 投递的触发器都不存在。APC_LEVEL 相对 PASSIVE 唯一多挡的是 APC 软件中断 | IRQL 语义：屏蔽级别，不是事件源 |
| 4 | "AP 比 P0 多降一级" | 方向反了：AP 少降一级、停得更高——P0 停 APC_LEVEL（1），AP 停 DISPATCH_LEVEL（2） | `KERNLINI.C:943` 尾巴统一升 DISPATCH；AP 分支无 APC_LEVEL 下降；栅栏自旋怕 DPC 打断 |
| 5 | 池分配检查表述颠倒 | 规则是分页池 ≤ APC_LEVEL、非分页池 ≤ DISPATCH_LEVEL；`ASSERT_ALLOCATE_IRQL` 是犯规检测（> 红线 → BAD_POOL_CALLER），不是"调用要求" | `EX/POOL.C:551` 断言体 |

---

## 第八部分：自查题（只给题，不给答案）

1. `IDTEntry` 宏为什么故意把 Selector 与 ExtendedOffset 写反？`KiSwapIDT` 在哪个核、哪个阶段执行一次？AP 为什么不用再换？
2. P0 为什么停在 APC_LEVEL 而不是 PASSIVE_LEVEL？APC_LEVEL 相对 PASSIVE_LEVEL 唯一多挡的是什么？这个"多挡"在点火期买来的是什么结构保证？
3. 分页池的红线为什么是 APC_LEVEL 而不是 DISPATCH_LEVEL？用"最坏情况下需要等的东西"推导一遍。
4. `KeSetPriorityThread(Thread, 0)` 之前为什么要先 `KeRaiseIrql(DISPATCH_LEVEL)`？如果 Idle 一开始就是 0，Phase 1 线程会在哪个时刻、因为什么机制抢走 CPU？
5. 为什么 64 位能把 IRQL 压到 16 档、省掉翻译表，而 32 位只能保留 32 档 + 翻译表？列出两个缺一不可的条件，并说明 32 位缺的是哪一个。

---

## 附录：未钉死清单（树里查不到的，不编造）

1. 真实线程切到 Idle 时 CR3 如何避免装入占位值：`SwapContext` 的 `mov cr3`（`CTXSWAP.ASM:551`）表面上无条件装 CR3，与 0 的衔接点未在树里定位到确切行号；MM 回填事实已足够解释运行期不崩。
2. NT4 的 cmpxchg8b"不正确"的具体机制：`KERNLINI.C:1612` 注释只确立"NT4 处理方式非通用 → 厂商藏位自保"的因果；树里没有 NT4 代码，无法核实具体机制；"掺了厂商/型号假设"仅为最合措辞的推断。
3. WS03 时代内核版 `__report_gsfailure` 的具体 bugcheck 码：实现不在 WRK 树；Win8 后的 0x139 仅作版本注脚，不回套 WS03。
4. i386 内核 cookie 初始化点：KE/I386 无引用，疑在链接进内核的 CRT 库；AMD64 的点（`START.ASM:250-264`）已实证，不平移。
5. DIRQL 具体哪几档共用哪类：取决于 HAL 运行时的向量分配，WRK 树里无精确表值；只给映射规则（有损多对一单调映射、低三档精确、HIGH_LEVEL → 0xF0），不编表。
6. HAL 的 SIPI 内部实现（怎么写 ICR、跳板代码原文）：HAL 本体不在 WRK 树（只有预编译 DLL）；此处按 Intel MP 规范与 `HalStartNextProcessor` 接口语义陈述。
7. i386 的 `KiInitProcessor` 本体：WRK 树里只剩声明（`ALLPROC.C:73`），实现在预编译库里；AP 拉起的内核侧细节以 AMD64 的 `ALLPROC.C:486` 调用与 i386 的 `KiInitProcessorState` 为准。
8. DirectoryTableBase[1] 的用途与后续填充点：`INC/KE.H:958` 只声明了 `DirectoryTableBase[2]` 数组、无注释；`MM/I386/INIT386.C:1013` 只回填 [0]；[1] 由谁填、作何用，树里未定位到。

---

*下一批预告：第 2 批 ExpInitializeExecutive 点火总控（Phase 0/1）——MM 上线、OB 点火、PS/调度的顺序与依赖。本批问题清完、用户喊「wrk 下一批」时推进。*

## 执行流转补记（2026-10-27 按新标准补）

本节只补“CPU 怎么从一条路径进入另一条”，与正文互补：正文讲各部件做了什么，这里讲执行权怎么过去的。

- **AP 被叫醒：SIPI → 跳板 → IdleLoop**。SIPI 向量是启动页号，AP 从该页的实模式代码起跑，走保护模式跳板（64 位再走长模式跳板），克隆 P0 已修正的 IDT，进 `KiIdleLoop`。转移方式：核间中断（叫醒）+ 跳板跳转（换模式）+ 函数调用（进循环）。AP 停在 DISPATCH_LEVEL（2 级），P0 停在 APC_LEVEL（1 级）——级别不同，因为 AP 一进循环就可能接管调度（分发级别），P0 还要跑 Phase 0 的初始化代码（APC 级别即可）。
- **中断/异常返回时的检查点：iret 之前先看挂起项**。中断或系统调用返回用户态之前，出口宏先升到 APC_LEVEL、调 `KiDeliverApc`（先前模式 = 用户，`KE/I386/TRAP.ASM:906` 一带），投递挂起的用户 APC（改陷阱帧 → iret 落进分发器，见第 5 批问答与第 8 批第十部分）；DPC 则走分发中断（见第 5 批补记）。转移方式：陷阱帧改写 + iret，不是函数调用——iret 恢复的上下文就是被改写过的那份。
- **线程切换：换栈 + ret**。`KiSelectNextThread`（`KE/THREDSUP.C:985`）选出下一个线程后，`SwapContext`（`KE/I386/CTXSWAP.ASM:374`）把旧线程的栈指针存进旧线程对象、新线程的栈指针从新线程对象取、换页目录，随后 ret——ret 从新线程的栈顶弹出返回地址：新线程第一次是 `KiThreadStartup`（建线程时栈就是按它搭的），老线程是上次被换出时停下的下一条指令。所以“切到另一个线程”不是函数调用，是换栈 + ret，目的地由新线程的栈决定。
- **IdleLoop 到首个线程：不是跳转，是调度**。`KiIdleLoop` 每轮查 DPC 队列与 NextThread：NextThread 非空时，`KiDispatchInterrupt` 内抬同步级别、拿核锁、把旧线程（Idle 线程）挂回就绪队列、调 `SwapContext`——Idle 线程被换出，新线程被换入。转移方式与上一条相同。
