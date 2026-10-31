---
layout: post
title: "FUTo 中文全译（Uninformed 第 3 卷，Peter Silberman & C.H.A.O.S.，2006）"
categories: essay
series: uninformed
series_order: 4
---

> 整理自 Uninformed 第 3 卷（2006 年 1 月）Peter Silberman 与 C.H.A.O.S. 的《FUTo》全文翻译——FU 之后 rootkit 转向 DKOM（直接内核对象操纵）藏进程，BlackLight 与 IceSword 改从 PspCidTable（进程 ID 句柄表）交叉比对找被藏的对象，FUTo 则反过来攻击这张句柄表本身。本文为 Uninformed 全译系列第 4 篇（前三篇为 PatchGuard 三部曲）。原文：http://www.uninformed.org/?v=3&a=7&t=txt

---

作者：Peter Silberman & C.H.A.O.S.

> 译者注：原文《FUTo》，作者 Peter Silberman 与 C.H.A.O.S.，发表于安全技术期刊《Uninformed》第 3 卷（Vol 3），日期 2006 年 1 月，栏目 Rootkit Technology（rootkit 技术）。原文链接：http://www.uninformed.org/?v=3&a=7&t=txt 。本文是《Uninformed 全集精彩文章导读》精选清单的第 5 篇。本译文为全文翻译，不删节、不摘要化；代码、函数名、数值、对象名保持原文；专有名词首次出现时给出中文说明。文中 Blacklight/BlackLight 两种拼写原文混用，译文统一作 BlackLight。

---

## 1) Foreword / 前言

### Abstract（摘要）

自 FU 出现以来，rootkit（以隐藏自身及指定对象为目的、通常运行于内核层的工具或恶意软件）界已不再靠实现系统挂钩来隐藏自身。攻方的这一变化迫使守方发展新的防御：BlackLight（F-Secure 开发的 rootkit 检测工具，主要检测隐藏进程）等 rootkit 检测器所用的新算法，试图找出 rootkit 到底藏了什么，而不是单纯检测 rootkit 挂钩的存在。本文将讨论 BlackLight 与 IceSword（冰刃，系统安全检测工具）都在使用的一种隐藏进程检测算法，同时记录当前 rootkit 检测领域的弱点，并介绍在原型 FUTo 中实现的更完整的隐匿技术。

### Thanks（致谢）

Peter 要感谢 bugcheck、skape、thief、pedram、做了出色研究的 F-Secure，以及所有鼓励思想成长的 nologin/research 成员。

C.H.A.O.S. 要感谢 Amy、Santa（这项工作是圣诞节当天用三小时完成的）、lonerancher、Pedram、valerino 与 HBG Unit。

## 2) Introduction / 引言

在过去一两年里，rootkit 界发生了几项重大进展。近期的里程碑包括：FU rootkit 的推出——它使用直接内核对象操纵（Direct Kernel Object Manipulation，DKOM：不挂钩任何函数、直接修改内核中的对象数据结构来达成隐藏的技术）；VICE 的推出——最早的 rootkit 检测程序之一；Sysinternals 的 Rootkit Revealer 与 F-Secure 的 BlackLight 的诞生——第一批主流 Windows rootkit 检测工具；以及最近推出的 Shadow Walker——一款挂钩内存管理器、藏身于众目睽睽之下的 rootkit。

接下来登场的是 BlackLight 与 IceSword。作者选择研究这两种工具所用的算法，是因为它们被业内许多人视为最好的检测工具。BlackLight 由芬兰安全公司 F-Secure 开发，主要关注隐藏进程的检测；它不试图检测系统挂钩，只关心隐藏进程。IceSword 使用与 BlackLight 非常相似的方法。IceSword 与 BlackLight 的区别在于：它是一个功能更完备的工具，能让用户看到哪些系统调用被挂钩、哪些驱动被隐藏，以及哪些程序（如 netstat）看不到的 TCP/UDP 端口正开着。

## 3) Blacklight / BlackLight 的检测算法

本文将主要聚焦 BlackLight，因为它的算法是本文的研究重点。而且在研究 BlackLight 之后可以明显看出，IceSword 使用的算法与之非常相似。因此，BlackLight 中发现的弱点，很可能同样存在于 IceSword 之中。

BlackLight 采用用户态方法检测进程。其算法虽然简单，却出奇地有效。BlackLight 使用了一些很强的反调试特性，其起点是创建一个线程局部存储（Thread Local Storage，TLS：每线程独立的存储机制；TLS 回调指在程序入口点执行之前由系统调用的一组函数）回调表。BlackLight 的 TLS 回调试图在进程对象完全创建之前分叉（fork）主进程，以此迷惑调试器；之所以能做到，是因为 TLS 回调例程在进程完全初始化之前就会被调用。BlackLight 还有检测调试器是否附加到它身上的反调试措施。作者没有去硬碰这些反调试——即不去绕过 TLS 回调并对程序做其他修改——而是决定干脆禁用 TLS 例程。做法是使用一款名为 LordPE 的工具，LordPE 允许用户编辑 PE（Portable Executable，可移植可执行）文件。作者用这款工具把 TLS 回调表清零。这禁用了分叉例程，使作者能够使用 API 监视器。需要说明的是，禁用回调例程后确实可以附加调试器，但当用户在 BlackLight 的图形界面中点击"scan"（扫描）时，BlackLight 会检测到调试器并退出。作者没有再想第二套办法去绕过反调试例程，而是决定分析 BlackLight 内部发生的调用。为此，作者使用了 Rohitab 的 API Monitor（监视并记录程序 API 调用的工具）。

在测试中，可以看到对 API 函数 OpenProcess 的失败调用（tls zero 指清除了 TLS 表的 BlackLight）。BlackLight 尝试打开进程 ID（PID）为 0x1CC、0x1D0、0x1D4、0x1D8 等等的进程。作者把 BlackLight 使用的这种方法称为 PID 暴力枚举（PID Bruteforce，PIDB）。BlackLight 遍历 0x0 到 0x4E1C 范围内的所有可能 PID，逐个调用 OpenProcess。BlackLight 保留一份它用 PIDB 方法能够打开的全部进程的名单。然后 BlackLight 调用 CreateToolhelp32Snapshot（创建进程快照的 Win32 API），得到第二份进程名单。接着 BlackLight 比较这两份名单，看 PIDB 名单中是否有不在 CreateToolhelp32Snapshot 函数返回的名单中的进程。如有任何差异，这些进程即被视为隐藏进程，并报告给用户。

### 3.1) Windows OpenProcess / Windows 的 OpenProcess

在 Windows 中，OpenProcess 函数是 NtOpenProcess 例程的包装。NtOpenProcess 在内核中由 NTOSKRNL.EXE 实现。NtOpenProcess 的函数原型如下：

```
NTSTATUS NtOpenProcess (
    OUT PHANDLE ProcessHandle,
    IN ACCESS_MASK DesiredAccess,
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    IN PCLIENT_ID ClientId OPTIONAL);
```

ClientId 参数是由 OpenProcess 传入的实际 PID。该参数是可选的，但在我们的观察中，OpenProcess 在调用 NtOpenProcess 时总是指定了 ClientId。

NtOpenProcess 执行三项主要功能：

1. 它通过调用 PsLookupProcessByProcessId 验证进程存在。
2. 它通过调用 ObOpenObjectByPointer 尝试打开一个指向该进程的句柄。
3. 如果成功打开了进程句柄，它把句柄回传给调用者。

PsLookupProcessByProcessId 是下一个显而易见的研究对象。一个突出的问题是：PsLookupProcessByProcessId 如何知道某个给定的 PID 属于一个有效进程？答案在反汇编的头几行就变得清楚了：

```
PsLookupProcessByProcessId:
  mov edi, edi
  push ebp
  mov ebp, esp
  push ebx
  push esi
  mov eax, large fs:124h
  push [ebp+arg_4]
  mov esi, eax
  dec dword ptr [esi+0D4h]
  push PspCidTable
  call ExMapHandleToPointer
```

从上面的反汇编可以清楚地看出，ExMapHandleToPointer 向 PspCidTable（内核中登记进程与线程客户 ID 的句柄表，详见 3.2 节）查询进程 ID。

至此，我们对 BlackLight 如何检测隐藏进程已经有了完整的图景：

1. BlackLight 开始遍历有效进程 ID 的范围，0 到 0x41DC。（译注：此上限与第 3 节正文所写的 0x4E1C 不一致，疑为原文笔误，照译。）
2. BlackLight 对每一个可能的 PID 调用 OpenProcess。
3. OpenProcess 调用 NtOpenProcess。
4. NtOpenProcess 调用 PsLookupProcessByProcessId 验证进程存在。
5. PsLookupProcessByProcessId 使用 PspCidTable 验证进程存在。
6. NtOpenProcess 调用 ObOpenObjectByPointer 取得指向进程的句柄。
7. 如果 OpenProcess 成功，BlackLight 保存该进程的信息并继续循环。
8. 在耗尽所有可能的 PID、创建出进程名单之后，BlackLight 把 PIDB 名单与它通过调用 CreateToolhelp32Snapshot 创建的名单作比较。CreateToolhelp32Snapshot 是一个对系统上所有正在运行的进程拍快照的 Win32 API。两份名单之间的差异意味着存在一个隐藏进程。这种情况会被 BlackLight 报告。

### 3.2) The PspCidTable / PspCidTable

PspCidTable 是"进程和线程客户 ID 的句柄表"（handle table：内核中以句柄值为索引、登记对象指针的表结构）。每个进程的 PID 都对应它在 PspCidTable 中的位置。PspCidTable 是一个指向 HANDLE_TABLE 结构的指针。

```
typedef struct _HANDLE_TABLE {
   PVOID        p_hTable;
   PEPROCESS    QuotaProcess;
   PVOID        UniqueProcessId;
   EX_PUSH_LOCK HandleTableLock [4];
   LIST_ENTRY   HandleTableList;
   EX_PUSH_LOCK HandleContentionEvent;
   PHANDLE_TRACE_DEBUG_INFO DebugInfo;
   DWORD        ExtraInfoPages;
   DWORD        FirstFree;
   DWORD        LastFree;
   DWORD        NextHandleNeedingPool;
   DWORD        HandleCount;
   DWORD        Flags;
};
```

Windows 提供了一组未导出的函数，用于操纵 PspCidTable 并从中检索信息。其中包括：

- [ExCreateHandleTable] 创建非进程句柄表。除 PspCidTable 之外，所有句柄表内的对象都是指向对象头的指针，而不是对象本身的地址。
- [ExDupHandleTable] 在派生（spawning）进程时调用。
- [ExSweepHandleTable] 用于进程 rundown（收尾清理）。
- [ExDestroyHandleTable] 在进程退出时调用。
- [ExCreateHandle] 创建新的句柄表条目。
- [ExChangeHandle] 用于更改句柄上的访问掩码。
- [ExDestroyHandle] 实现 CloseHandle 的功能。
- [ExMapHandleToPointer] 返回与句柄对应的对象的地址。
- [ExReferenceHandleDebugIn] 跟踪句柄。
- [ExSnapShotHandleTables] 供句柄搜索工具使用（例如 oh.exe 中）。

下面是使用未导出函数把一个进程对象从 PspCidTable 中移除的代码。它对所需的未导出函数使用了硬编码地址；不过，rootkit 也可以动态查找这些函数地址。

```
typedef PHANDLE_TABLE_ENTRY (*ExMapHandleToPointerFUNC)
                     ( IN PHANDLE_TABLE HandleTable,
                       IN HANDLE ProcessId);

void HideFromBlacklight(DWORD eproc)
{
               PHANDLE_TABLE_ENTRY CidEntry;
               ExMapHandleToPointerFUNC map;
               ExUnlockHandleTableEntryFUNC umap;
               PEPROCESS p;
               CLIENT_ID ClientId;

               map = (ExMapHandleToPointerFUNC)0x80493285;

               CidEntry = map((PHANDLE_TABLE)0x8188d7c8,
                         LongToHandle( *((DWORD*)(eproc+PIDOFFSET)) ) );
               if(CidEntry != NULL)
               {
                  CidEntry->Object = 0;
               }
               return;
}
```

由于 PspCidTable 的工作是跟踪全部进程和线程，rootkit 检测器利用 PspCidTable 来寻找隐藏进程是合乎逻辑的。然而，依赖单一数据结构并不是一个很健壮的算法。如果一个 rootkit 篡改了这一个数据结构，操作系统和其他程序将完全不知道这个隐藏进程的存在。应当设计新的 rootkit 检测算法，使其依赖关系彼此重叠，这样单一处改动就不会不被察觉。

## 4) FUTo / FUTo 的实现

为了演示 BlackLight、IceSword 等当前 rootkit 检测软件所用算法的弱点，作者创建了 FUTo。FUTo 是 FU rootkit 的新版本。FUTo 新增了不使用任何函数调用即可操纵 PspCidTable 的能力。它使用 DKOM 技术来隐藏 PspCidTable 中的特定对象。

实现 FUTo 的新功能时有几个设计上的考虑。第一个是：与 ExMapHandleXXX 系列函数一样，PspCidTable 并未被内核导出。为了克服这一点，FUTo 通过找到 PsLookupProcessByProcessId 函数、反汇编它并寻找其中的第一个函数调用，来自动检测 PspCidTable。在本文写作时，第一个函数调用总是对 ExMapHandleToPointer 的调用。ExMapHandleToPointer 把 PspCidTable 作为它的第一个参数。利用这一点，找到 PspCidTable 就相当直接了。

```
PsLookupProcessByProcessId:
  mov edi, edi
  push ebp
  mov ebp, esp
  push ebx
  push esi
  mov eax, large fs:124h
  push [ebp+arg_4]
  mov esi, eax
  dec dword ptr [esi+0D4h]
  push PspCidTable
  call ExMapHandleToPointer
```

可以写出更健壮的方法来寻找 PspCidTable，因为只要内核做了哪怕简单的编译器优化，上述算法就会失效。Opc0de 写了一种更健壮的方法，用于检测 PspCidTable、PspActiveProcessHead、PspLoadedModuleList 等未导出变量。Opc0de 的方法不像 FUTo 当前所用的方法那样需要扫描内存。相反，Opc0de 发现，处理器控制区（Process Control Region，即 KPCR，处理器级的内核控制结构）结构中的 KdVersionBlock 字段指向一个 KDDEBUGGER_DATA32 结构。该结构如下：

```
typedef struct _KDDEBUGGER_DATA32 {

    DBGKD_DEBUG_DATA_HEADER32 Header;
    ULONG   KernBase;
    ULONG   BreakpointWithStatus;       // address of breakpoint
    ULONG   SavedContext;
    USHORT  ThCallbackStack;            // offset in thread data
    USHORT  NextCallback;               // saved pointer to next callback frame
    USHORT  FramePointer;               // saved frame pointer
    USHORT  PaeEnabled:1;
    ULONG   KiCallUserMode;             // kernel routine
    ULONG   KeUserCallbackDispatcher;   // address in ntdll

    ULONG   PsLoadedModuleList;
    ULONG   PsActiveProcessHead;
    ULONG   PspCidTable;

    ULONG   ExpSystemResourcesList;
    ULONG   ExpPagedPoolDescriptor;
    ULONG   ExpNumberOfPagedPools;

    [...]

    ULONG   KdPrintCircularBuffer;
    ULONG   KdPrintCircularBufferEnd;
    ULONG   KdPrintWritePointer;
    ULONG   KdPrintRolloverCount;

    ULONG   MmLoadedUserImageList;

} KDDEBUGGER_DATA32, *PKDDEBUGGER_DATA32;
```

正如读者所见，该结构包含许多常用/必需的未导出变量的指针。这是寻找 PspCidTable 及同类变量的一种更健壮的方法。

第二个设计考虑稍微麻烦一些。当 FUTo 从 PspCidTable 中移除一个对象时，HANDLE_ENTRY 被替换为 NULL，表示该进程"不存在"。问题出现在被隐藏的进程（且在 PspCidTable 中没有条目）被关闭之时。当系统试图关闭该进程时，它会索引进 PspCidTable 并解引用一个空对象，从而导致蓝屏（blue screen）。解决这个问题的办法简单但不优雅。首先，FUTo 通过调用 PsSetCreateProcessNotifyRoutine（注册进程创建/删除通知回调的内核函数）设置一个进程通知例程。回调函数会在每次进程创建时被调用，但更重要的是，它会在每次进程删除时被调用。该回调在被隐藏进程终止之前执行；因此，它在系统崩溃之前被调用。当 FUTo 删除那些包含指向 rogue process（原文用词，指被 FUTo 隐藏的那个进程）的对象的索引时，FUTo 会保存 HANDLE_ENTRY 的值和索引，以备后用。当进程被关闭时，FUTo 会在进程关闭之前恢复这些对象，使系统能够解引用到有效对象。

## 5) Conclusion / 结论

2005 年的流行语是："我们（再次）抬高了 rootkit 检测的门槛"。希望读者已经对顶级 rootkit 检测程序如何检测隐藏进程、以及它们如何能得到改进有了更好的理解。有些读者可能会问："我能做些什么？"嗯，简单的解决办法是不连接互联网，但同时使用 BlackLight、IceSword 和 Rootkit Revealer 的组合，将大大提高你保持无 rootkit 状态的机会。一款名为 RAIDE（Rootkit Analysis Identification Elimination，rootkit 分析、识别与清除）的新工具将在未来几个月于 Black Hat 阿姆斯特丹大会（Blackhat Amsterdam）上公布。这款新工具不受本文所揭示问题的困扰。

## Bibliography / 参考文献

- Blacklight Homepage. F-Secure Blacklight
  http://www.f-secure.com/blacklight/
- FU Project Page. FU
  http://www.rootkit.com/project.php?id=12
- IceSword Homepage. IceSword
  http://www.xfocus.net/tools/200505/1032.html
- LordPE Homepage. LordPE Info
  http://mitglied.lycos.de/yoda2k/LordPE/info.htm
- Opc0de. 2005. How to get some hidden kernel variables without scanning
  http://www.rootkit.com/newsread.php?newsid=101
- Rohitabs API Monitor. API Monitor - Spy on API calls
  http://www.rohitab.com/apimonitor/
- Russinovich, Solomon. Microsoft Windows Internals Fourth Edition.
- Silberman. RAIDE:Rootkit Analysis Identification Elimination
  http://www.blackhat.com/html/bh-europe-06/bh-eu-06-speakers.htmlSilberman
  （译注：该链接末尾与作者名 Silberman 粘连，系原文如此，照录。）

（全文完）
