---
layout: post
title: "Anti-Debug Tricks 网站介绍：Check Point 的反调试技巧百科"
categories: essay
---

> 2026-11-03 逐页实读该站全部 8 个分类页后整理。本文是介绍与导读，不搬运其代码；示例代码请到原站看。

## 一、站点名片

| 项 | 内容 |
|---|---|
| 名称 | Anti-Debug Tricks（反调试技巧百科） |
| 网址 | https://anti-debug.checkpoint.com/ |
| 出品 | Check Point Research（CPR，以色列安全厂商 Check Point 的研究部门） |
| 上线 | 2020 年 8 月（CPR 官方博客 2020-08-05 发文宣布） |
| 定位 | 系统性收集 Windows 平台**反调试**技巧的在线百科：每条技巧给原理、C/C++ 与汇编实现、以及对应的绕过方法 |
| 规模 | 8 大类、50 余项技巧（部分含子项） |
| 开源配套 | 全部技巧实现于 CPR 的开源项目 **ShowStopper**（GitHub: CheckPointSW/showstopper）；内容仓库开放 PR 贡献 |
| 姊妹站 | Evasions Encyclopedia（evasions.checkpoint.com）：同一团队做的**反虚拟机/反沙箱**逃逸技巧百科，另有 macOS、Android 分册 |

**先给术语定义。反调试（anti-debugging）**：程序在运行时检测自己是否正被调试器附加/跟踪，一旦判定在被调试，就退出、崩溃或转入无害分支，从而让分析者看不到其真实行为。它是恶意软件逃避分析的标配，也是商业软件壳与授权保护的常用手段。与之相对的**反反调试（anti-anti-debugging）**，是分析方用来隐藏调试器存在、使这些检测全部失效的技术（调试器插件 ScyllaHide 是代表作）。这个站的特色是**攻防两面都写**：每个技巧后面都有 Mitigations（缓解/绕过）一节，既讲手工调试时怎么跳过，也讲写反反调试工具时应该 hook 哪些函数、改哪些返回值。

**收录标准**（CPR 自述）：只收在最新版 Windows + 主流调试器（OllyDbg、WinDbg、x64dbg）上仍然有效的技巧；已过时的（如 SoftICE 时代）一律剔除。取材致谢了这个领域的经典文献：Peter Ferrie 的《The "Ultimate" Anti-Debugging Reference》、Nicolas Falliere 的《Windows Anti-Debug Reference》等——可以理解为把散落在论文和博客里的几十年积累，做了一次官方化的整理与复验。

## 二、分类逻辑与八大类内容

CPR 的分类方式是按**每条技巧所利用的"副作用"来源**分组：调试器在工作时必然在系统里留下某种痕迹（标志位、内核对象、异常处理路径的差异、时间流逝……），技巧就是去测量对应的那一种痕迹。

### 1. Debug Flags（调试标志）——最常用的一类
核心思想：操作系统会在进程的系统结构里留下"正在被调试"的标志位，读这些标志即可判定。分两路：
- **走 API 查**：`IsDebuggerPresent()`（本质只读 PEB 的 BeingDebugged 标志）；`CheckRemoteDebuggerPresent()`；`NtQueryInformationProcess()` 的三个信息类——ProcessDebugPort（被调试时返回 -1）、ProcessDebugFlags（返回 EPROCESS 里 NoDebugInherit 的反值）、ProcessDebugObjectHandle（调试开始后内核会创建"调试对象"，能查到其句柄即在被调试）；查堆标志的 `RtlQueryProcessHeapInformation()` / `RtlQueryProcessDebugInformation()`；查内核调试器的 `NtQuerySystemInformation()` 的 SystemKernelDebuggerInformation 类。
- **手工直读内存**：不调 API，直接读 PEB 的 BeingDebugged（32 位 `fs:[30h]+2`、64 位 `gs:[60h]+2`，页面给了含 WOW64 在内的偏移）；读 PEB 的 NtGlobalFlag——由调试器**创建**的进程会带 0x70 的堆检查标志组合（事后附加不改变此值，这是它与 BeingDebugged 的关键区别）；读堆的 Flags/ForceFlags 字段；查堆尾的保护模式填充（0xABABABAB）；以及读固定地址 0x7FFE0000 处 KUSER_SHARED_DATA 的内核调试器字段。
- **绕过**：把 PEB 相应标志清零（ScyllaHide 正是这么做的），或 hook 上述查询函数篡改返回缓冲。

### 2. Object Handles（对象句柄）
核心思想：某些内核对象与句柄操作在有无调试器时行为不同。
- `OpenProcess(csrss.exe)`：能成功打开说明当前用户带调试特权，间接提示调试环境；
- `CreateFile` / `LoadLibrary` 独占打开自身文件：调试器在 CREATE_PROCESS_DEBUG_EVENT 时握着被调试文件的句柄且常忘记关闭，独占打开失败即说明有调试器；
- `CloseHandle` 喂无效句柄：有调试器时 NtClose 会抛出 EXCEPTION_INVALID_HANDLE，能被自己的异常处理捕获即判定被调试——这一条本是调试器的规范行为被反用；
- `NtQueryObject` 枚举全系统对象类型：数 DebugObject 的数量，只能说明"系统自开机以来有过调试会话"，定不了当前进程，页面如实标注了这条局限。

### 3. Exceptions（异常）
核心思想：故意触发异常，看异常**落到谁手里**——调试器会先截获特定异常，这是无调试器时不会发生的。
- 注册自定义未处理异常过滤器后执行 INT 3：无调试器时控制流会进过滤器，有调试器时被调试器吞掉；
- `RaiseException` 抛出 DBG_CONTROL_C、DBG_RIPEVENT 这类调试专用异常：它们不会被进程的异常处理链处理、只会被调试器消费，处理函数没被调用即判定被调试；
- 延伸用法：把真正的控制流藏进一串异常处理链里（SEH/VEH 接力），这不检测调试器，而是让静态与动态跟踪都难以还原路径。

### 4. Timing（计时）
核心思想：被单步/断点调试时，指令间的时间会被拉长到远超正常值；测一段代码的执行耗时、与"原生耗时"比较即可判定。测法给了一排：`RDTSC`/`RDPMC` 指令、`GetTickCount`、`GetLocalTime`/`GetSystemTime`、`QueryPerformanceCounter`、`timeGetTime`，以及内核侧的 `ZwGetTickCount`/`KiGetTickCount`（读 KUSER_SHARED_DATA 页）。页面对这类技巧的评价很诚实：**本身不可靠**，绕过方"没太大必要专门处理"，手工调试时 NOP 掉即可。

### 5. Process Memory（进程内存）
核心思想：进程检查**自己的内存**——找调试器留下的断点与补丁。
- 扫软件断点：遍历关键函数机器码找 0xCC（INT 3 的操作码）；并给出"反单步步过"：调试器 step over 时会在返回地址处下 0xCC，被调试进程可以检查并改写这个字节反将一军（直接改内存、用 ReadFile/WriteProcessMemory/Toolhelp32ReadProcessMemory 覆盖，四种变体）；
- 内存断点的反制：利用 OllyDbg 类调试器以保护页（guard page）实现内存断点的机制，构造一页只有一个 RET 的可执行缓冲并标为 guard page，看控制流落点判定；
- 硬件断点：`GetThreadContext` 读 DR0–DR3 调试寄存器，非零即有人下了硬件断点；
- 其他：`NtQueryVirtualMemory` 查工作集——代码页被下过软件断点就不再是共享页（Shared/ShareCount 变化）；检测 `IsDebuggerPresent` 等函数本体是否被 patch（与干净进程比对首字节）；patch 掉 ntdll 的 `DbgBreakPoint`/`DbgUiRemoteBreakin` 使调试器附加即崩溃；对自身代码段做校验和（checksums），发现任何改动即判定。

### 6. Assembly Instructions（汇编指令）
核心思想：利用 CPU 指令在有无调试器（尤其单步跟踪）时**语义层面的细微差异**。
- INT 3（含长格式 CD 03 时 EIP 回退一字节的坑）、INT 2D（执行后内核对 EIP 的特殊处理会"吃掉"下一字节）、`DebugBreak()`：均为"异常进不进自己处理函数"的变体；
- ICE 指令（未公开指令，操作码 0xF1）：正常执行产生单步异常，被跟踪时该异常被调试器当作单步事件吞掉；
- 堆栈段寄存器花招（push ss/pop ss/pushf）：MOV SS 会抑制中断一拍，使单步留下的陷阱标志（Trap Flag）在 EFLAGS 里可被读到；
- 指令计数：给一串指令逐条下硬件断点、用 VEH 计数，数目对不上即被调试器截胡；
- POPF 与陷阱标志（TF）：自己置 TF 位，单步异常没来说明被调试器清掉了；
- 指令前缀花招：利用个别调试器处理 F3 前缀的方式差（页面注明只对某些调试器有效）。

### 7. Interactive Checks（与调试器直接互动）
核心思想：不再被动查痕迹，而是**主动去碰**调试器或界面环境。
- 自调试（Self-Debugging）：派生第二个实例试图用 `DebugActiveProcess` 附加父进程——一个进程同时只能挂一个调试器，附加失败即说明已有调试器在位；
- `NtSetInformationThread` 的 ThreadHideFromDebugger（0x11）：把线程从调试器视野里隐藏，让它暗中执行校验逻辑（页面警告：用了它之后再在该线程里下断点会让进程崩、调试器卡死）；
- `GenerateConsoleCtrlEvent` 自发 Ctrl+C：被调试时会产生 DBG_CONTROL_C 异常被调试器拦截，落点不同即可判定；
- `BlockInput` 封锁键鼠（顺带还能反测 hook：此 API 正常只能成功一次，第二次还返回 TRUE 说明被 hook 了）；
- 枚举窗口找父进程（调试器）的窗口标题、直接 `SuspendThread` 把调试器线程挂起；`SwitchDesktop` 新建桌面把调试器甩在旧桌面；`OutputDebugString` 的错误码检查（已注明仅对 Vista 以前有效，属"太有名不能不收"的历史条目）。

### 8. Misc（杂项）
归不进前七类的零散招：`FindWindow` 按窗口类名找 OllyDbg/WinDbg 等已知调试器窗口；父进程检查（正常双击启动的进程父进程应为 explorer.exe，用 NtQueryInformationProcess 或 Toolhelp32 快照比对）；段选择子花招（利用异常/单步事件后选择子被恢复默认值的 32/64 位差异判定跟踪，源自 Ferrie 的研究）；`DbgPrint` 的 DBG_PRINTEXCEPTION_C 异常落点检查；`DbgSetDebugFilterState` 在内核调试器在场时才成功；`NtYieldExecution`/`SwitchToThread` 在单步时无法切换线程的反跟踪用法；以及借 `GetWriteWatch` 追踪谁动过内存页的思路（源头是 al-khaser 项目的讨论）。

## 三、每条技巧的固定体例（这站最好用的地方）

每条技巧都按同一模板写，这使它能当手册查：
1. **原理**：一段话说清利用的是什么机制、在什么条件下成立；
2. **实现**：可直接编译的 C/C++ 代码，多数再加 x86 与 x86-64 两份汇编；
3. **Mitigations**：分两种读者——手工调试的人（NOP 掉检查、改标志位、把断点下在检查之后）；和写反反调试工具的人（该 hook 哪个函数、把返回值改成什么，都给了具体目标值）；
4. 对不可靠或有局限的技巧会**如实标注**（如计时类"不可靠"、NtQueryObject"只能说明系统有过调试会话"），不夸大。

全站总纲式的建议是：绕过反调试优先用 **ScyllaHide**（支持 OllyDbg、x64dbg、IDA 的开源插件），本站的逐条 Mitigations 更多是给要自己造轮子的人看的原理级说明。

## 四、怎么用这个站（结合逆向工作流）

- **动态分析时样本"行为不对"**：样本在调试器里直接退出、崩溃、或只走无害分支，先别怀疑样本坏了——来这个站按现象反查：启动即退多半是 Debug Flags 类；单步就崩查 Assembly 与 Process Memory 类；附加不上查 DbgUiRemoteBreakin 被 patch 与 Self-Debugging。
- **配调试环境时当清单**：用 ScyllaHide 时逐项对照它勾选的隐藏项与本站的检测项，看还漏了哪一类（它默认处理 PEB 标志，但 ThreadHideFromDebugger、计时类等要另行留意）。
- **验证与练手**：配套的 ShowStopper 把全站技巧做成了一个可运行的检测程序，可以拿它直接测自己的调试环境"隐身"程度，比逐条手写验证快得多。
- **加固与检测侧**：写检测规则（沙箱/EDR 识别样本的反分析行为）时，这 50 余项就是一份现成的行为清单：哪些 API 调用组合、哪些 PEB/堆偏移访问，是反调试的指纹。

## 五、边界

- 平台只覆盖 **Windows**（以用户态为主，含少量内核调试器检测项，如 SystemKernelDebuggerInformation）；macOS、Android 的对应内容在 Evasions 家族的其他分册里，不在本站。
- 只谈**检测与对抗技术本身**，不涉及具体恶意软件家族分析；代码均为检测逻辑片段，不含载荷。
- 个别条目是历史名篇性质的收录（如 OutputDebugString 仅适用于 Vista 以前），页面内均有注明，使用时以其标注的适用范围为准。

## 六、链接

- 本站：https://anti-debug.checkpoint.com/
- 发布公告（CPR 博客，2020-08-05）：https://research.checkpoint.com/2020/cpr-anti-debug-encyclopedia-the-check-point-anti-debug-techniques-repository/
- 配套实现 ShowStopper：https://github.com/CheckPointSW/showstopper
- 姊妹站 Evasions：https://evasions.checkpoint.com/
