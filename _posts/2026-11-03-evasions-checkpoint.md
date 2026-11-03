---
layout: post
title: "Evasions Encyclopedia 网站介绍：Check Point 的反虚拟机与反沙箱逃逸技巧百科"
categories: essay
---

> 2026-11-03 逐页实读该站 Windows 篇全部 15 个分类页及 macOS、Android 分册后整理，与《Anti-Debug Tricks 网站介绍》互为姊妹篇。本文是介绍与导读，不搬运其代码；示例代码请到原站看。

## 一、站点名片

| 项 | 内容 |
|---|---|
| 名称 | Evasions Encyclopedia（逃逸技巧百科） |
| 网址 | https://evasions.checkpoint.com/ |
| 出品 | Check Point Research（CPR，以色列安全厂商 Check Point 的研究部门） |
| 上线 | 2020 年初（2020 年 3 月已有公开报道，早于反调试站）；内容仓库建于 2019-09 |
| 定位 | 系统性收集恶意软件**检测虚拟机/沙箱**的技巧的在线百科：每条技巧给原理、代码、检测签名建议与环境加固对策 |
| 规模 | Windows 篇 15 大类、按编号条目清点约 80 余项；macOS 分册 6 项；Android 分册已上线 1 类 3 项（另 3 类未上线） |
| 开源配套 | 全部技巧实现于 CPR 的开源工具 **InviZzzible**（GitHub: CheckPointSW/InviZzzible）；内容仓库 CheckPointSW/Evasions（MIT 许可）开放 PR 贡献 |
| 姊妹站 | Anti-Debug Tricks（anti-debug.checkpoint.com）：同一团队做的**反调试**技巧百科，配套工具为 ShowStopper；本站内部也嵌有同一套 Anti-Debug 分册 |

**先给术语定义。逃逸（evasion）**：在这个语境里专指恶意软件判断自己是否正运行在**虚拟机**或**沙箱**里，若是就收敛行为（不执行载荷、直接退出、只做无害动作），使自动化分析得不到真实样本行为。**虚拟机（VM）**是由**虚拟机监控器（hypervisor，又称 VMM）**在一台物理机上模拟出的整台"计算机"，恶意软件分析实验室用它批量关押样本；**沙箱（sandbox）**是更广义的受控分析环境——可以建在虚拟机里（如 Cuckoo），也可以是带监控钩子的隔离运行环境（如 Sandboxie）。两者都会留下与真实用户机器不同的**痕迹（artifact）**：特有的驱动文件、注册表项、设备名、进程、硬件描述字符串等等。逃逸技巧的本质就是去枚举这些痕迹。

**它与反调试的分工**，是理解这两站关系的钥匙：反调试防的是"**人**正拿着调试器看我"，逃逸防的是"**机器**把我关在虚拟环境里自动看"。实战中样本通常两套一起上，所以 CPR 把它们做成了一对姊妹百科。还有两个体例术语先说清：**签名建议（signature recommendations）**是给防御方的检测规则写法——哪几个函数调用带哪些参数出现时可判定样本在做逃逸检查；**对策（countermeasures）**是给沙箱/虚拟机维护者的加固办法——怎么改环境才能让对应的检查失效。这站和反调试站一样攻防两面都写，但读者重心更偏防御与环境加固一侧。

**编写与取材**（About 页自述）：主笔 Raman Ladutska（Windows 篇等），macOS 分册 Alexey Bukhteyev，站内 Anti-Debug 分册 Yaraslau Harakhavik。代码示例大量引自社区三大开源检测工具 **al-khaser、pafish、VMDE** 并逐处标注出处；CPR 说明自家 InviZzzible 虽实现了全部检查，但因其模块化代码篇幅太大，不便作示例，故以社区项目代码为主。About 页还提醒：主页上个别分类处于未激活状态，意为内容待补。

## 二、分类逻辑

与反调试站按"副作用来源"分组不同，本站 Windows 篇按**被检查的对象域**分组：样本去翻系统的哪个角落，那个角落就是一类。虚拟化层为了模拟硬件与提供宿主-客户机通道（增强工具、共享文件夹、时间同步等），不得不在客户机里留下驱动、服务、设备与描述字符串；沙箱为了监控行为，不得不注入钩子、留下辅助进程与固定的环境参数——15 类就是这 15 个"角落"。还有一条贯穿全站的判定习惯值得先提：许多检查不是"有/无"的二值，而是**"真实机器不会这么寒酸"**的阈值判定（内存太小、硬盘太小、开机太短、窗口太少），这是沙箱为了省资源批量运行而必然露出的马脚。

### Windows 篇 15 类

#### 1. Filesystem（文件系统）
核心思想：增强工具与沙箱会在磁盘上留下真实机器没有的文件与目录，查存在性即可。五项：查特定文件（VMware 的 vmmouse.sys/vmhgfs.sys、VirtualBox 的 VBoxGuest.sys 一族、Parallels 的 prl*.sys，另有沙箱投放的诱饵文件如 c:\email.doc）；查特定目录（VMware/VirtualBox Guest Additions 安装目录、CWSandbox 的 c:\analysis）；查自身可执行文件全路径是否含 \sample、\virus、sandbox 等字符串（样本被沙箱改名投放的破绽）；查是否从特定目录启动（ Anubis 的 c:\insidetm）；查各盘根目录是否有 sample.exe/malware.exe 这类沙箱惯用文件名。实现都是 `GetFileAttributes` 一把梭，对照表按"检出对象→路径"列全。

#### 2. Registry（注册表）
核心思想同上，只是痕迹在注册表里。三项：查特定键路径是否存在（各家 hypervisor 的服务项、ACPI 表项如 HARDWARE\ACPI\DSDT\VBOX__、Sandboxie 的 SbieDrv、Wine 的 SOFTWARE\Wine，甚至 PCI 厂商号 VEN_80EE=VirtualBox、VEN_15AD=VMware）；查特定值是否含指定字符串（SystemBiosVersion 含 VBOX/VMware/QEMU/BOCHS，各沙箱写死的 Windows ProductID——Anubis、CWSandbox、JoeBox 各有固定值）；查 Office 的 VBAWarnings 是否被设为"启用所有宏"（沙箱为了跑宏样本常这么配）。页面诚实标注了误报来源：**真实机器上装过虚拟机软件，同样会留下这些键**，单看此项会误判。

#### 3. Generic OS queries（通用系统查询）
核心思想：不用任何"虚拟化专属"痕迹，只查一台真实用户机器的"生活常识"参数，沙箱的出厂配置大多对不上。十项：用户名是否是 malware/sandbox/test/tequilaboomboom 等分析环境惯用名；计算机名/主机名是否是沙箱固定名（如 Anubis 的 TU-4NH09SMCG1HC）；总内存低于 1GB；屏幕分辨率非常规；处理器少于 2 核（取法给了四种：GetSystemInfo、直读 PEB、直读 KUSER_SHARED_DATA 等）；显示器数量太少；硬盘容量或可用空间 ≤60GB；系统已开机时间过短（如少于 12 分钟——沙箱都是刚从快照恢复的）；是否从虚拟硬盘原生启动（`IsNativeVhdBoot`，Win8+）。本站对这类坦承：**不是最可靠的方法，但样本里极常见**，且签名建议只能给到"hook 住函数看它被调用"，因为正常程序也会查用户名。

#### 4. Global OS objects（全局对象）
核心思想：增强工具与沙箱的驱动会在 Windows 的对象命名空间里注册全局对象，枚举它们即现形。六项：查特定全局互斥量（如 Sandboxie 的 Sandboxie_SingleInstanceMutex_Control、Virtual PC 与 DeepFreeze 的互斥量）；用 `NtCreateFile` 试开特定虚拟设备（\\.\VBoxGuest、\\.\vmci、\\.\HGFS 等）；查特定命名管道（VBox 的管道是上一项的特例）；枚举对象目录找特定驱动/设备对象（\Driver\SbieDrv、\Device\VBoxGuest、Hyper-V 的 VmGenerationCounter 等）；查 Sandboxie 特有的 \Sandbox 对象目录是否存在；查 Sandboxie 的虚拟注册表——打开 \REGISTRY\USER 后用 `NtQueryObject` 验其真实对象名，对不上即被沙箱重定向。对策是 hook 相关函数在查到黑名单对象时返回不存在。

#### 5. UI artifacts（界面痕迹）
核心思想：利用图形界面的贫富差异——虚拟环境窗口稀少，且托盘工具有专属窗口类。仅两项：用 `FindWindow` 查 VirtualBox 托盘窗口类（VBoxTrayToolWndClass）；用 `EnumWindows` 数顶层窗口总数，少于 10 个即疑似虚拟环境（真实桌面通常远多于此）。本站明说这一类**不给签名建议**：无法区分枚举窗口是逃逸检查还是正常行为。对策也很朴素：在沙箱里造些假窗口充数。

#### 6. OS features（系统机制特性）
核心思想：利用操作系统与监控架构本身的工作方式，而非某个痕迹物。三项，质量很高：其一，**调试特权检查**——在 Cuckoo 这类监控下启动的进程会继承已启用的 SeDebugPrivilege，能 `OpenProcess` 打开 csrss.exe 等关键进程即说明身处监控环境（样本甚至可进一步结束它使系统蓝屏、中断分析）；其二，**非平衡栈（unbalanced stack）检测**——CPR 自家团队在 Virus Bulletin 2016 发表：监控钩子比原函数多用栈空间，样本精确预留栈空间并在栈底埋下金丝雀数据，调用后检查数据是否被钩子踩坏，从而判定函数被 hook，对策是 Cuckoo 后来采用的两阶段钩子（先挪栈再执行钩子）；其三，**检测 Wine**——利用 `MulDiv(1, 0x80000000, 0x80000000)` 在真 Windows 上因古老实现返回 2、逻辑上应为 1 的怪癖，或查 Wine 独有导出函数如 wine_get_unix_file_name。

#### 7. Processes（进程与模块）
核心思想：虚拟环境与沙箱会运行真实机器没有的辅助进程、并往样本进程里注入监控模块。五项：枚举进程查黑名单（vmtoolsd.exe、VBoxService.exe、prl_tools.exe、joeboxserver.exe、qemu-ga.exe 等，连抓包工具 WPE Pro、内核转储工具 KsDumper 也在名单里，页面逐一注明它们并非虚拟机、只是常与 VM 检测搭配出现）；查自身地址空间是否加载了黑名单 DLL（Sandboxie 的 sbiedll.dll、CWSandbox 的 api_log.dll/dir_watch.dll 等）；查特定库是否导出特定函数（Wine 的 wine_get_version 等，与第 6 类呼应）；试加载一批库——真实机器应有的冷门系统库加载失败、或根本不存在的假库名竟"加载成功"，都说明身处模拟环境（模拟器对 LoadLibrary 的模拟不完整）；专查 Sandboxie 在进程内存里留下的特征字节（'kuzt'/'xobs' 标记，用 `NtQueryVirtualMemory` 遍历自有地址空间搜）。

#### 8. Network（网络）
核心思想：虚拟网卡与沙箱网络配置同样有出厂指纹。六项：查网卡 **MAC 地址**（介质访问控制地址，网卡的硬件标识，其前 3 字节是厂商前缀 OUI）是否命中厂商段——VirtualBox 08:00:27、VMware 00:0C:29/00:50:56 等、Parallels 00:1C:42、Xen 00:16:E3；查网卡适配器名称（Vmware）；查网络共享提供者名是否为 "VirtualBox Shared Folders"；**查出口 IP 是否属于"安全厂商势力范围"**——样本向 MaxMind 的接口查自己出口 IP 的归属组织，命中 Amazon、FireEye、ProofPoint、数据中心等字符串就认定身处安全公司的分析网络（这是检测"网络环境"而非"本机环境"，视角独特）；**NetValidateName 返回值滥用**——沙箱常关 NetBIOS over TCP/IP，该函数对非法参数的返回码随之不同，样本据此算跳转地址，算错即自崩，达到"只在真实网络配置下才正常运行"；查是否存在到 TCP 2042 端口的连接——那是 Cuckoo 的 ResultServer（结果回传服务）端口，监控通道本身成了指纹。

#### 9. CPU（处理器）
核心思想：直接问 CPU 或利用指令在虚拟化下行为的差异，是最"底层"的一类。六项：**CPUID 厂商字符串**——EAX 置 0x40000000 执行 CPUID，hypervisor 会在 EBX/ECX/EDX 里交回签名（KVMKVMKVM、VMwareVMware、VBoxVBoxVBox、Microsoft Hv、XenVMMXenVMM、bhyve bhyve），这是架构规范留给 hypervisor 的自我介绍位，藏无可藏；**CPUID hypervisor 存在位**——EAX=1 时 ECX 第 31 位被置 1 即表示跑在 hypervisor 之上；**描述符表基址（Red Pill）**——用 sidt/sgdt/sldt 读 IDT/GDT/LDT 基址，客户机的这些表会被挪到高地址（如 IDT 基址高字节为 0xFF），源自 Joanna Rutkowska 的经典研究，页面注明在最新 VMware 上已失效、为完整性收录；用模拟器不支持的冷门指令集（如 MMX）使模拟环境抛异常；执行在真 CPU 上必触发异常、在 Virtual PC 里却被"正常执行"的非法指令序列；**VMware 后门端口**——用 IN 指令读 I/O 端口 'VX'（0x5658），EAX 放魔数 'VMXh'，这是 VMware 宿主与客户机的通信后门，真机上该端口读会触发异常、VMware 里返回签名。对策只能改 hypervisor 或改虚拟机配置（如关掉后门、伪装 CPUID 结果），本站对这类直言签名建议无从给起。

#### 10. Hardware（硬件信息）
核心思想：虚拟硬件的设备描述里写着厂商名，查询设备属性即可。五项：查硬盘设备名（SetupDi 系列函数枚举磁盘，FriendlyName 含 QEMU/VBOX/VIRTUAL HD/VMware）；查硬盘厂商 ID（`DeviceIoControl` 的 IOCTL_STORAGE_QUERY_PROPERTY）；查音频设备是否缺失（源自 TeslaCrypt 样本：许多虚拟机不配声卡，样本用 DirectShow 图构建试探音频渲染器）；查 CPU 温度是否可得（源自 GravityRAT：经 WMI 查 MSAcpi_ThermalZoneTemperature，虚拟机通常无此数据）；查物理显示适配器（Direct3DCreate9 后 GetAdapterIdentifier，VMware 的 VendorId 0x15AD、描述 "VMware SVGA 3D" 一望而知）。对策多为环境整容：改硬盘名、给虚拟机补声卡、hook 显卡信息查询改返回值。

#### 11. Firmware tables（固件表）
核心思想：固件留给操作系统的 ACPI/SMBIOS 表是硬件身份的"户口本"，虚拟机的户口本上写着厂商名。只有两项，但很硬：导出原始固件表（'FIRM'）与原始 SMBIOS 表（'RSMB'）后全文搜字符串——Parallels、Innotek/Oracle/VirtualBox、VMware 字样直接出现。取表方法按系统版本分两路：Vista 起用 `NtQuerySystemInformation` 的 SystemFirmwareTableInformation 类（76 号）；XP 时代则去读 csrss.exe 进程里 0xC0000/0xE0000 处的固件映像（固件表在启动时被映射保留）。对策：新系统上 hook 该查询并清洗表内容。

#### 12. Hooks（钩子）
核心思想分两个方向。其一，**查钩子**：沙箱靠 hook（钩子，在函数入口改写跳转、把调用引到监控代码）记录行为，样本读回系统函数开头的字节即可发现——正常的 kernel32 函数序言是 `mov edi, edi`（8B FF），出现 E9 跳转、push/ret 组合、CC 断点字节都是被动的证据。其二，**用钩子**：装一个低层级鼠标钩子（WH_MOUSE_LL）等真实用户点击，等不到点击就认定无人值守。其三最阴：**查钩子装得对不对**——ntdll 有 400 多个 Nt 函数常被沙箱 hook，量大必出错；CPR 实测发现 Cuckoo 与 CAPE 的监控器把 NtLoadKeyEx 按旧版 4 参数原型 hook，而 Vista 起它实际是 8 参数，样本只要正常调一次 RegLoadAppKeyW 就会栈错位抛异常——"合法调用能把沙箱打崩"本身就是检测信号。对策要求沙箱方逐个核对 hook 原型的参数个数。

#### 13. Timing（时间）
核心思想：沙箱的分析时长以分钟计且可被操纵，时间因此成为整类信号源，子项最丰富。**拖延**：用 Sleep/等待对象/定时器、乃至 winsock 的 select、ICMP 这种冷门等待函数把恶意行为推到分析超时之后；用计划任务延迟启动；只在重启后才发作（沙箱分析期内从不重启）；只在特定日期发作（Sazoora 样本查每月 16–18 日）。**反拖延的拖延检测**：沙箱会"跳过睡眠"（sleep skipping，把长 Sleep 缩短并伪造时钟），样本反手检测它——并行用两种机制计时互相对账、用未被 hook 的计时源（QueryPerformanceCounter、直读 KUSER_SHARED_DATA 里的 TickCount）测同一段 Sleep、跨进程比对时间流速（跳过睡眠不是全系统的）、查 NtDelayExecution 返回后延时参数是否被改过、用绝对超时（相对超时是沙箱常处理的，绝对超时常被算错）。**外部时间源**：向 NTP 服务器或 HTTP 响应的 Date 头取真实时间，戳破沙箱内部改日期的把戏。**执行时差**：RDTSC 测 CPUID 的耗时——在真机上 CPUID 很快，在虚拟机上它触发 VM Exit（客户机陷入 hypervisor 处理）而显著变慢；Locky 的变体比较 GetProcessHeap 与 CloseHandle 两个调用的耗时比。另有两项交叉项：多方法比对系统最后启动时间（与 WMI 类联动）、给延时函数喂非法参数看沙箱的 hook 是否做了参数校验（未对齐指针应返回 STATUS_DATATYPE_MISALIGNMENT，沙箱常错误地执行了延时甚至自己崩掉）。本站对这类同样坦承不给定式签名：调用延时函数本身太常见，无法与正常代码区分。

#### 14. WMI（Windows 管理接口）
先给定义：**WMI（Windows Management Instrumentation）**是 Windows 内建的系统信息与管理查询框架，用类 SQL 的查询（WQL）经 COM 接口取硬件与系统数据，如 Win32_Processor、Win32_Bios。本类四项：用 WMI 查询重做其他类的检查（核数、硬盘、MAC、BIOS 序列号与版本里的 VBOX/QEMU/VMware、主板厂商 innotek GmbH 等，对照表按"查询→字段→可疑值→检出对象"整表列出）——意义在于沙箱若只防了 API 层检查，WMI 这条平行通道常被漏掉；**用 WMI 逃逸监控**——经 Win32_Process 的 Create 方法建进程，不经过父进程里的 CreateProcessInternalW，绕开常见的用户态进程创建 hook（WMI 还能在 Win7 上设计划任务）；查最后启动时间（快照恢复的虚拟机，WMI 库里存的可能是快照拍摄时的旧值，算出的 uptime 大到以年计）；查网卡最后重置时间（同理，快照机的网卡 uptime 会异常地长）。对策强调内核级监控（PsSetCreateProcessNotifyRoutineEx 回调）与改完 KeBootTime 后必须重启 winmgmt 服务或重置 WMI 仓库——细节很见功力。

#### 15. Human-like behavior（真人行为）
核心思想：前面查的都是机器，最后一类查"**有没有人**"——沙箱里没有真人留下的生活痕迹与实时操作。分两组。**生活痕迹组**：最近打开的文档数少于 3 个（样本取自真实 VB 宏代码）；浏览器历史少于 10 条 URL（示例直接读 Chrome 的 History 数据库）；已安装软件数量与种类不像工作机（装机清单太干净、或全是分析工具）。**实时在场组**：两次取鼠标坐标间隔数秒不变即无人（进阶版取自 LummaC2 窃密木马的研究：采集鼠标轨迹向量并算夹角，超过 45 度阈值判定为"不像人"——沙箱的随机鼠标恰恰转角过大）；等用户点击才行动，并反制沙箱的自动点击模块（把真正的"下一步"做成 Static 控件的假按钮、或放一个 1×1 像素的诱饵 Button 按钮，谁点了诱饵谁就是机器人）；Office 文档要滚动到第二页才触发载荷（源自 FireEye 报告：漏洞利用代码被大量段落标记顶到第二页）；用 `GetLastInputInfo` 查距上次键鼠输入的时长（页面还顺手指出一个反面教材：Agent Tesla v3 这条检查写错了量纲、永远检测不到沙箱，并给了正确实现）。本站对全类不给签名建议，理由与第 5 类相同：这些动作与正常软件无法区分。

### macOS 分册（一页，6 项）

与 Windows 篇同体例，但因平台特性不给代码块、只给 shell 命令——检测主要靠 `sysctl`（读内核状态变量）与 `ioreg`（转储 I/O Kit 注册表，macOS 的设备树）。六项：硬件型号（`sysctl -n hw.model`，真机回 Macmini8,1 一类、虚拟机回 VMware7,0，不含 "Mac" 字样即判定；见于 MacRansom 样本）；超线程是否开启（逻辑核数÷物理核数，真机旧款为 2，页面注明 2018 年后新硬件已不成立、此法过时）；内存大小（真机通常大于 4GB）；I/O Kit 注册表（平台序列号为 "0"、board-id/manufacturer 含 VirtualBox/innotek 等）；Boot ROM 版本（真机是 MM/MBP 等产品字母码，虚拟机直接写 VirtualBox，见于 OceanLotus 样本）；SIP（系统完整性保护）是否开启（`csrutil status`——沙箱若需加载未签名内核扩展做监控，往往被迫关 SIP）。分册末尾给了一串命令行的签名清单与一条特殊的对策：苹果许可不允许在非苹果硬件上虚拟化 macOS、单机最多 2 台虚拟机，所以 CPR 建议干脆不用虚拟化、改用 DeepFreeze 类还原方案加签名内核扩展。

### Android 分册（已上线 1 类 3 项，另 3 类未上线）

About 页交代：Android 平台技巧数量远少于 Windows，故逃逸与反调试合在一个分册里。实读核查结果：索引页上只有 **APK anti-analysis（APK 反分析）** 一类是可点的，其余三类——Environment detection 标 "coming soon"、Emulator & debug flags 与 Human behavior 标 "planned"——链接未上线（仓库里有同名文件，网站未发布）。已上线的这一类讲的不是运行时检测，而是**把 APK 文件本身做坏到"真机能装、工具打不开"**：其一，篡改 ZIP 尾部 EOCD 记录的磁盘号与目录计数字段（APK 本质是 ZIP，真机启动器不深究、JEB/apktool 等严格解析的工具会报错或算错条目数，文中给了逐字段的修复办法）；其二，做坏二进制 AndroidManifest 的结构——文件头魔数写成 0x00080000（正确应为 0x00080003）、字符串偏移数组末元素越界、样式池偏移在样式数为 0 时乱填，启动器宽容、apktool 严格校验而崩；其三，往 assets 里塞路径总长超过 300 字符的深层嵌套文件，噎死不能重映射路径的反编译工具（且这些文件根本不被代码引用，删了不影响运行）。修复思路是全类共性：**分析工具比系统严格，样本就专攻这个严格差**。

### 站内 Anti-Debug 分册

首页四入口中的 "Windows: Anti-debug" 与独立站 anti-debug.checkpoint.com 同源同内容（同一仓库、同一批作者），本站 About 页对它的介绍与姊妹篇所记一致。此处不重复，见《Anti-Debug Tricks 网站介绍》。

## 三、每条技巧的固定体例

Windows 篇每条技巧按统一模板写，这是它能当手册与清单用的根本：
1. **原理**：先说清利用的是哪种痕迹、在什么条件下成立；
2. **实现**：可编译的 C/C++ 代码（偶有 VB/PowerShell），逐处标注来自 al-khaser、pafish、VMDE 还是 InviZzzible；
3. **签名建议**：防御方怎么检——盯哪个函数、哪个参数位置、比对什么值；
4. **检测对照表**：这一站最有辨识度的部分——把每个特征常量（文件路径、注册表项、字符串、MAC 前缀、CPUID 签名……）逐行映射到它能检出的环境（VMware/VirtualBox/Parallels/Hyper-V/Xen/QEMU，以及 Cuckoo、Sandboxie、JoeBox、Anubis 等沙箱与 Wine），等于随书附赠一份可直接落地的指纹字典；
5. **对策**：环境方怎么防——hook 改返回值、改配置、补设备、造假数据。
体例也有诚实的缺口：凡是行为与正常软件无法区分的类（界面痕迹、CPU、钩子、时间、真人行为），明确写"不提供签名建议"及其原因，不硬凑。

## 四、怎么用这个站（结合分析与加固工作）

- **搭/加固自己的分析环境时当验收清单**：把 15 类逐项过一遍——改掉默认用户名与计算机名、内存硬盘别给太小、装齐增强工具的"整容"（改 MAC 前缀、删托盘进程、清注册表键）、给虚拟机补声卡与显示器、别用快照一恢复就开跑（先让它"活"十几分钟并产生浏览历史、最近文档）。配套的 **InviZzzible** 把全站检查做成了一个可运行程序，在样本机里跑一遍，它逐项报"会被哪招看穿"，相当于这本书的标准答卷。
- **样本在沙箱里"装死"时反查**：样本一进沙箱就退出或只睡不干活，按类排查它可能在查什么：只睡不醒先看 Timing 与 Human-like behavior；连 C2 都不回连、疑似先验了出口 IP，看 Network 的安全势力范围一项；只在某些 Cuckoo/CAPE 版本崩，看 Hooks 的 NtLoadKeyEx 那条。
- **写检测规则时当字典**：检测对照表就是现成的 IoC（入侵指标）库——哪些文件/键值/设备名/进程名是"环境指纹"、哪些 API+参数组合是"逃逸检查行为"，可直接转为沙箱埋点或 EDR 规则。
- **与姊妹篇合用**：动态调试前先用本站清单把环境整容、再用反调试百科处理样本的调试器检测，两站正好覆盖"环境"与"工具"两个暴露面。

## 五、边界

- **指纹会老化**：对照表里的常量是 2019–2020 年代收集的，各家 hypervisor 与沙箱版本迭代后数值会变（页面自己就注明 Red Pill 在最新 VMware 上失效、macOS 超线程法已过时）；当原理手册用没问题，当现行指纹库用需自行复验。
- **误报是结构性存在的**（站点自述）：装过虚拟机软件的真实机器会带着一身 VM 痕迹，单个痕迹命中不能定罪，这类检查在实践中都是多项投票。
- **覆盖不完整且明示**：About 页明说个别分类待补；Android 分册三类未上线、macOS 分册因苹果许可拿不到多家 hypervisor 的对照输出——站点的态度是"缺就标缺"，引用时以其标注为准。
- 只谈检测与对抗技术本身，代码均为环境检查片段，不含载荷实现。

## 六、链接

- 本站：https://evasions.checkpoint.com/
- About 页（作者与编写说明）：https://evasions.checkpoint.com/about/
- 配套工具 InviZzzible：https://github.com/CheckPointSW/InviZzzible
- 内容仓库：https://github.com/CheckPointSW/Evasions
- 姊妹站 Anti-Debug Tricks：https://anti-debug.checkpoint.com/
