---
layout: post
title: "A Catalog of Local Windows Kernel-mode Backdoor Techniques 中文全译（Uninformed 第 8 卷，skape & Skywing，2007）"
categories: essay
series: uninformed
series_order: 7
---

> 整理自 Uninformed 第 8 卷（2007 年 8 月）skape 与 Skywing 的《A Catalog of Local Windows Kernel-mode Backdoor Techniques》全文翻译（期刊摘要页题作《A Catalog of Windows Local Kernel-mode Backdoors》）——本地内核后门技术的分类目录：映像补丁、描述符表、MSR、页表项、函数指针、异步读循环等逐项给出类别、出处、能力、注意事项与隐蔽度五维条目，第 5 章正面澄清「PatchGuard 能防内核 rootkit」的常见误解。本文为 Uninformed 全译系列第 7 篇。原文：http://www.uninformed.org/?v=8&a=2&t=txt

---

> 译者注：原文《A Catalog of Local Windows Kernel-mode Backdoor Techniques》，作者 skape（Matt Miller）与 Skywing（Ken Johnson），发表于安全技术期刊《Uninformed》第 8 卷（Vol 8），原文标注日期 2007 年 8 月。期刊摘要页上的题名为《A Catalog of Windows Local Kernel-mode Backdoors》，与原文 txt 的题名略有出入，系同一篇论文的两个题名，本文以原文题名为准。原文链接：http://www.uninformed.org/?v=8&a=2&t=txt 。本译文为全文翻译，不删节、不摘要化；代码、函数名、数值、结构名保持原文；专有名词首次出现时给出中文说明；原文的笔误与空缺照译并加译注，未擅自改动。本文为 Uninformed 全集精选导读第 17 篇，将作为博客「Uninformed 全译系列」第 7 篇发布。

（括注：期刊摘要页上的题名为 “A Catalog of Windows Local Kernel-mode Backdoors”，与本文正文题名 “A Catalog of Local Windows Kernel-mode Backdoor Techniques” 系同一篇论文的两个题名，本片以正文题名为准。）

August, 2007 / 2007 年 8 月

skape & Skywing

`mmiller@hick.org` & `Skywing@valhallalegends.com`

## Abstract / 摘要

本文呈现一份详细的技术目录，这些技术可用于在 Windows 上创建本地内核模式后门（local kernel-mode backdoor）。所谓内核模式后门，是指植入或利用于操作系统内核模式（kernel-mode，即处理器特权级 ring 0，ring 是 CPU 的特权级环，ring 0 权限最高、ring 3 最低，内核运行在 ring 0、普通应用程序运行在 ring 3 的用户模式）之中、绕过内核本应强制执行的安全限制、从而取得本不应开放的资源访问权的方法。这些技术包括函数跳板（function trampolines）、描述符表挂钩（descriptor table hooks）、模型专用寄存器挂钩（model-specific register hooks）、页表修改（page table modifications），以及其他一些此前从未被描述过的技术。这些技术中的大多数，早在本论文出现之前很久就已经为公众所知。然而，在本文撰写之时，对于其中许多技术而言，似乎并不存在一份详细的、单一的参考汇编。本文的意图，是就本地内核模式后门这一主题提供一份扎实的理解。这样一份理解是必需的，其目的在于鼓励人们对潜在的对策以及人们所认为的种种进步展开深思熟虑的讨论。沿着对策这一脉络，本文还就一个常见的误解给出了一些补充思考：即认为 PatchGuard（微软在 64 位 Windows 中引入的内核补丁保护机制，正式名称为 Kernel Patch Protection，它周期性地校验内核关键代码与数据结构的完整性，发现被篡改时可触发蓝屏；它本质上是完整性检测机制，而非访问控制机制）以其当前的设计，可以被用来阻止内核模式 rootkit（kernel-mode rootkits，即运行在内核模式的 rootkit 恶意软件）。

## 1) Introduction / 1）引言

用户模式（user-mode）与内核模式（kernel-mode）之间经典的特权分离，一直是大多数现代操作系统所包含的一项常见特性。这种分离使操作系统能够就进程隔离、内核—用户隔离、内核模式完整性等方面作出安全保证。这些安全保证之所以必需，是为了防止一个特权较低的用户模式进程能够接管系统本身。内核模式后门，正是绕过这些安全限制的一种方法。

可以用来给内核植入后门的技术有很多种。就本文而言，后门（backdoor）将被视为这样一种东西：它提供对那些本来通常会受到内核限制的资源的访问。这些资源可能包括以内核模式特权执行代码、访问内核模式数据、禁用安全检查，等等。为了进一步限定本文的范围，作者将严格聚焦于那些可用于在 Windows 上提供进入内核的本地后门的技术。在此语境下，本地后门（local backdoor）是指这样一种后门：它不依赖于、也不利用网络连接来提供对资源的访问。相反，本地后门可以被看作是削弱内核的一些方法，其目的在于向非特权实体（例如用户模式进程）提供对资源的访问。

本文所讨论的后门技术中的大多数，已经在许多不同的出版物中被长篇、且非常详细地论述过[20, 8, 12, 18, 19, 21, 25, 26]。本论文的首要目标，是充当一个参考汇编点，汇集一些常见的、以及一些不那么常见的本地内核模式后门技术。作者已尝试为每一种技术附上客观的度量指标，并附上对每种技术如何工作的描述。作为界定这些客观度量指标工作的一部分，作者已尝试追溯一些较为知名的后门技术的起源。由于其中许多技术已经被使用了如此之久，其起源已证明在某种程度上难以发掘。

本论文的结构如下。在第  节中，逐一详细讨论了可用于提供本地内核模式后门的每一种技术。第  节简要讨论了可能被用来阻止本文所讨论的某些技术的一般性策略。第  节试图驳斥一些反对阻止内核模式后门本身的常见论点。最后，第  节试图阐明，为什么微软的 PatchGuard 就内核模式后门而言，不应被视为一种安全解决方案。（译注：原文此段各处节号在源文本中即为空缺，未给出具体编号，故译文照原文保留空缺，不擅自补入。）

## 2) Techniques / 2）技术

为了帮助妥善地编目本节所描述的种种技术，作者已尝试纳入对每种技术的客观度量指标。这些度量指标细分为如下几项：

- Category / 类别

  作者已选择采用 Joanna Rutkowska 的恶意软件分类法，以期追求一种标准化的分类[34]。这一模型描述了三种类型的恶意软件。Type 0（0 型）恶意软件归类的是非侵入性的恶意软件；Type I（I 型）包括那些修改本不应被修改之物的恶意软件（代码段、MSRs 等，其中 MSR 是 Model-Specific Register，即模型专用寄存器，是 x86 处理器上用于控制调试、性能监测、系统配置等处理器特有功能的特殊寄存器）；Type II（II 型）包括那些修改本应被修改之物的恶意软件（全局变量、其他数据）；Type III（III 型）不在本文的范围之内[33, 34]。

  除了 Rutkowska 所描述的四种恶意软件类型之外，作者提出 Type IIa（IIa 型），它将归类这样一种可写内存：在给定的上下文中，它实际上应当被视为只写一次（write-once）。例如，当一个全局 DPC 被初始化时，其 `DpcRoutine` 就可以被视为只写一次的。其中 DPC 是 Deferred Procedure Call，即延迟过程调用，是 Windows 内核的一种机制，它把中断处理中不紧急的部分推迟到较低的中断请求级（IRQL）上、由专门的 DPC 例程随后执行；`DpcRoutine` 即该 DPC 被调度时要执行的例程函数指针。作者认为这是 Type II 的一个衍生类型，理由在于：该内存仍保持可写，并且与 Type I 的内存相比，它被检查的可能性更低。

- Origin / 起源

  如果可能的话，给出该技术被使用的第一个已知实例，或者给出关于其起源的一些补充背景。

- Capabilities / 能力

  该后门所提供的能力。这可以是以下一项或多项：内核模式代码执行（kernel-mode code execution）、访问内核模式数据（access to kernel-mode data）、访问受限资源（access to restricted resources）。如果一种技术允许内核模式代码执行，那么它就隐含地拥有此处列出的所有其他能力。

- Considerations / 注意事项

  关于使用某一给定技术时必须指出的任何限制或特殊要点。

- Covertness / 隐蔽性

  对使用某一给定技术可能有多容易被检测到所作的描述。

由于本文所描述的许多技术已经为人所知相当长一段时间，作者已采取尽力而为（best effort）的方法来识别原始想法的来源。在许多情况下，这已证明是困难的、或是不可能的。为此，作者请求：引文中的任何不准确之处，都请报告出来，以便在本文的未来版本中予以更正。

> 原文：skape & Skywing，Uninformed 第 8 卷，2007 年 8 月。本片对应原文 2.1 节全节（Image Patches / 映像补丁），含 2.1.1、2.1.2 两小节。文中方括号数字（如 [21]）为原文参考文献编号，照原文保留。

## 2.1) Image Patches / 映像补丁

给内核植入后门，最显而易见的做法也许就是修改内核自身所用的代码段（code segment，即映像中存放可执行机器指令的区段）。这包括修改 ntoskrnl.exe、ndis.sys、ntfs.sys 等内核态映像的代码段。通过改动这些代码段，只要被钩住的函数（hooked function）被调用，就能劫持内核态的执行流。围绕代码段修改所能玩出的花样，唯一的上限就是内核本身能做的事情。

### 2.1.1) Function Prologue Hooking / 函数序言钩子

函数钩子（function hooking）指的是拦截对某个给定函数的调用，并把这些调用重定向到另一个替代函数。函数钩子这个概念已经存在相当长的时间了，最初是谁提出这个想法的已不可考。现存有不少库和论文专门帮助实现函数钩子[21]。就本地内核态后门而言，函数钩子是一种简单而可靠的造门方法。钩住函数有几种不同的做法。其中最常用的一种技术，是覆写待钩函数的序言（prologue，即函数开头的一小段指令，通常负责建立栈帧、保存寄存器等开场工作），在那里写入一条与具体体系结构相关的跳转指令，把控制权转移到内存中别处的替代函数。这正是微软 Detours 库所采用的方法。序言钩子在概念上虽然简单，但要正确实现它，实际上需要相当多的代码。

要以可移植且可靠的方式实现序言钩子，往往必须借助一个反汇编器（disassembler，能把机器码逐条解析回汇编指令、并给出每条指令字节长度的工具）。原因是：执行序言覆写时，待钩函数开头的头几个字节必须被一条控制转移指令（通常是跳转指令）盖掉。在 Intel 体系结构上，控制转移指令可以有三种操作数之一：寄存器、相对偏移、内存操作数。每种操作数类型决定了所需跳转指令的大小：分别为 2 字节、5 字节和 6 字节。有了反汇编器，就可以在覆写之前，先把函数序言开头的头 n 条指令复制保存下来。n 的取值这样确定：逐条反汇编序言中的指令，直到已反汇编的字节数大于或等于钩子覆写时将要盖掉的字节数为止。

之所以必须把头 n 条指令完整保存下来，是为了让钩子函数仍然能够调用原函数。为了调用函数的原始版本，必须生成一小段桩代码（stub）：它先执行原函数序言的头 n 条原始指令，紧接着跳转到原函数函数体中的第 n + 1 条指令处。这段桩代码的效果，是让原函数可以在不被序言覆写所改道的情况下被调用——业界把这种「保存原序言 + 跳回原函数」的桩称为函数蹦床（function trampoline）：执行流像在蹦床上弹了一下，先落到这段桩上跑完被盖掉的原指令，再弹回原函数未被改动的身体继续执行，从而既完成了拦截，又保留了随时「放行」到原函数的能力。这种实现函数序言钩子的方法，被 Detours 和其他钩子库广泛采用[21]。这类「在函数入口处就地改写跳转」的钩子在业界也通称内联钩子（inline hook）：它不改任何指针表，而是直接在函数代码本体（inline，即「就地」）的前几个字节上动手，与后文靠改函数指针、描述符表条目来改道的钩子形成对照。

较新版本的 Windows，例如 XP SP2 和 Vista，其映像文件里带有一种更优雅的方式来做序言覆写式钩子。事实上，这些映像是用一项编译器增强构建出来的，而这项增强正是专门为提升微软在运行时钩住自家函数的能力而设计的。其做法是：让函数以一条两字节的空操作指令（no-op，例如 `mov edi, edi`，把寄存器 edi 的值原样送回 edi，不产生任何实际效果）作为函数序言的第一条指令。除这条两字节指令外，编译器还在函数本身之前前缀了 5 条空操作指令留出的 5 字节空间。那条两字节空操作指令提供了恰好放下一条两字节相对短跳转指令所需的存储空间；而这条相对短跳转又可以把控制权转移到放在函数之前那 5 字节里的一条相对跳转指令。这套「函数体内两字节 `mov edi, edi` 垫片 + 函数体前 5 字节预留空地」的布局就是后来业界所称的热补丁（hotpatch）垫片：最终效果是一种更具确定性的序言覆写钩子方法，不再依赖反汇编器。一个常见的问题是：为什么要用一条两字节的空操作指令，而不是两条独立的单字节空操作指令？答案分两部分。第一，一条两字节空操作指令可以被原子地（atomic，即整个覆写在其他处理器看来要么全发生、要么全未发生，不会读到写了一半的中间状态）覆写，而其他序言覆写——比如 5 字节或 6 字节的覆写——做不到这一点。第二，两字节空操作指令还能防止竞态条件（race condition，多个线程的执行交错顺序不同导致结果不同乃至出错的情形）：防止某个线程恰好正在被覆写的那组字节内部执行代码时、钩子被装上所引发的问题。这一竞态条件是任何类型的函数序言覆写都共有的毛病。

为了更好地理解这一竞态条件，设想一下：某个函数的序言若是两条单字节空操作指令，会发生什么。在该函数被钩住之前，一个线程执行了第一条空操作指令。就在它执行完这第一条空操作、正要执行第二条空操作之间，该函数在第二个线程的上下文中被钩住，头两个字节被覆写为与一条相对短跳转指令相关的操作码，例如 `0xeb` 和 `0xf9`。序言覆写发生后，第一个线程开始执行原本是第二条空操作的那条指令。然而，此时函数已被钩住，这条空操作指令可能已经从 `0x90` 变成了 `0xf9`。视钩子执行的上下文而定，这可能造成灾难性的后果。这一竞态条件看起来也许不太可能发生，但它确实是可行的，因此会直接影响任何使用序言覆写来钩函数的方案的可靠性。

**类别（Category）：** Type I（第一类）

**起源（Origin）：** 补丁代码这一概念「自数字计算黎明之时起就已存在」[21]。

**能力（Capabilities）：** 内核态代码执行。

**注意事项（Considerations）：** 函数序言钩子的可靠性，直接取决于所用反汇编器的可靠性，以及函数序言中被覆写的字节数。如果那条两字节空操作指令不存在，那么函数序言覆写能做到多处理器安全的可能性就不大了。同样，如果反汇编器没能按照实际处理器准确计算指令长度，函数序言钩子就可能失败，进而导致系统意外崩溃。还有一点值得一提：钩子函数的作者必须小心，不要因为没有对被钩函数的参数做适当的净化与检查，而无意中给操作系统引入不稳定问题。已经有过许多这样的例子：正当软件走了钩函数的路线，却没有把这些注意事项考虑在内[38]。

**隐蔽性（Covertness）：** 在本文写作之时，函数序言覆写被认为是不隐蔽的。像 Joanna Rutkowska 的 System Virginity Verifier（系统纯洁性校验器）[32] 这类工具，可以轻易地把系统映像在内存中的版本与磁盘上的版本进行比对，从而检测内存中的篡改。Windows 调试器（windbg）同样会让分析人员注意到内存代码段与其磁盘对应物之间的差异。

### 2.1.2) Disabling SeAccessCheck / 禁用 SeAccessCheck

在 Phrack 第 55 期中，Greg Hoglund 描述了给 `nt!SeAccessCheck` 打补丁、让它永不返回「访问被拒绝」（access denied）的好处[19]。`SeAccessCheck` 是内核中负责访问检查的核心例程：每当有主体想访问一个可受保护对象（securable object，即带安全描述符、能对其做权限判定的内核对象，如文件、进程、注册表键）时，系统调用它来比对请求方的访问令牌与对象的安全描述符，得出「放行」还是「拒绝」的结论。把它补丁成永不拒绝，效果就是：对可受保护对象的访问检查将永远授予访问权限，无论该访问在正常情况下是否会被授予。结果，非特权用户就能直接访问本来只有特权者才能碰的资源。这一简单的修改并不能直接让人执行特权代码，但它通过允许非特权用户与系统进程交互、并修改系统进程，间接地为执行特权代码铺了路。

**类别（Category）：** Type I（第一类）

**起源（Origin）：** Greg Hoglund 是第一个在 1999 年 9 月公开指出这一技术的人[19]。

**能力（Capabilities）：** 访问受限资源。

**隐蔽性（Covertness）：** 与函数序言覆写一样，`nt!SeAccessCheck` 补丁可以通过 ntoskrnl.exe 被映射的映像与磁盘版本之间的差异被检测出来。

## 2.2) Descriptor Tables / 描述符表

x86 架构有若干种不同的描述符表（descriptor table），处理器用它们来处理诸如内存管理（GDT）、中断分发（IDT）等事项。这里先说明几个专有名词：描述符（descriptor）是描述符表中的一条定长记录，用来描述一段内存、一个代码入口或一种控制转移方式的属性——基址、界限、特权级与其他标志，处理器在执行相关操作时会查这条记录来决定是否允许、以及到哪里去；GDT 即 Global Descriptor Table，全局描述符表，存放段描述符（segment descriptor），供内存管理中的分段机制使用；IDT 即 Interrupt Descriptor Table，中断描述符表，存放中断门、陷阱门等门描述符（gate descriptor），供中断分发使用。除处理器层面的描述符表之外，Windows 操作系统自身还包含若干种不同的软件层面的描述符表，例如 SSDT。SSDT 即 System Service Descriptor Table，系统服务描述符表，是 Windows 内核分发系统调用时查的函数指针表。这些描述符表中的大多数都被操作系统重度依赖，因此对于后门用途而言，它们构成了一个极具诱惑力的目标。和 中所描述的函数挂钩（function hooking）技术一样，本小节所介绍的全部技术都已经为人所知相当长一段时间了。作者已经尽力在可能的情况下，辨明每一种技术的起源。

### 2.2.1) IDT / 中断描述符表

中断描述符表（Interrupt Descriptor Table，IDT）是一种与处理器相关的结构，在分发中断时被使用。中断是处理器用来中断程序执行、以便处理某个事件的一种手段。中断既可以由来自硬件的信号引发，也可以由软件通过 `int` 指令主动发出中断而引发[23]。IDT 包含 256 个描述符，分别与处理器所支持的 256 个中断向量（interrupt vector，即中断的编号，用来索引 IDT 中对应的描述符）相对应。每一个 IDT 描述符可以是三种门描述符类型之一（任务门 task gate、中断门 interrupt gate、陷阱门 trap gate）——门描述符是描述符的一种，专门用来描述当某个特定向量的中断发生时，控制应当转移到何处、以及以何种方式转移。IDT 的基址和界限存放在 `idtr` 寄存器中，该寄存器通过 `lidt` 指令装入。`idtr` 当前的基址和界限可以用 `sidt` 指令读出。

IDT 挂钩（IDT hook）这一概念，很可能自中断处理概念诞生之日起就已经存在了。在大多数情况下，IDT 挂钩的工作方式是：把某个给定 IDT 描述符的过程入口点（procedure entry point，即该中断最终要跳转执行的处理例程地址）重定向到另一个位置。从概念上说，这与挂钩任何函数指针所涉及的过程是相同的（在 中有更详细的描述）。区别在于挂钩一个 IDT 描述符所需要的具体代码有所不同。
在 x86 处理器上，每一个 IDT 描述符都是一个八字节的数据结构。属于中断门或陷阱门描述符的 IDT 描述符，包含当该描述符所关联的中断向量被触发时所要使用的过程入口点和代码段选择子（code segment selector）。除了包含控制转移信息之外，每一个 IDT 描述符还包含一些附加标志，用来进一步控制所采取的动作。Windows 内核用以下结构来描述 IDT 描述符：

```
kd> dt _KIDTENTRY
   +0x000 Offset           : Uint2B
   +0x002 Selector         : Uint2B
   +0x004 Access           : Uint2B
   +0x006 ExtendedOffset   : Uint2B
```

在上面的数据结构中，`Offset` 字段保存过程入口点的低 16 位，`ExtendedOffset` 字段保存其高 16 位。利用这一知识，就可以通过把过程入口点重定向到另一个函数来挂钩一个 IDT 描述符。以下代码说明了如何做到这一点：

```c
typedef struct _IDT
{
  USHORT          Limit;
  PIDT_DESCRIPTOR Descriptors;
} IDT, *PIDT;

static NTSTATUS HookIdtEntry(
  IN UCHAR DescriptorIndex,
  IN ULONG_PTR NewHandler,
  OUT PULONG_PTR OriginalHandler OPTIONAL)
{
  PIDT_DESCRIPTOR Descriptor = NULL;
  IDT             Idt;

  __asm sidt [Idt]

  Descriptor = &Idt.Descriptors[DescriptorIndex];

  *OriginalHandler =
    (ULONG_PTR)(Descriptor->OffsetLow +
                (Descriptor->OffsetHigh << 16));

  Descriptor->OffsetLow  =
    (USHORT)(NewHandler & 0xffff);
  Descriptor->OffsetHigh =
    (USHORT)((NewHandler >> 16) & 0xffff);

  __asm lidt [Idt]

  return STATUS_SUCCESS;
}
```

除了挂钩单个 IDT 描述符之外，还可以通过创建一个新表、然后用 `lidt` 指令设置其信息，来挂钩整个 IDT。

类别（Category）：Type I（第一类）；尽管 IDT 的某些部分可能被合法地挂钩。

起源（Origin）：IDT 挂钩起源于中断向量表（Interrupt Vector Table，IVT）挂钩。1999 年 10 月，Prasad Dabak 等人写过关于 IVT 挂钩的文章[31]。可惜的是，他们似乎也没有注明其来源。可以肯定的是，IVT 挂钩在 1999 年之前就已经存在。作者能找到的最早的病毒引用来自 1994 年，但 DOS 是在 1981 年发布的，第一批 IVT 挂钩很可能在此后不久就出现了。一项于 1985 年 12 月提交的、名为 Dual operating system computer（双操作系统计算机）的专利，以一种暗示存在某种形式的 IVT 挂钩的方式，谈到了 IVT 的“重定位（relocation）”。

能力（Capabilities）：内核模式代码执行。

隐蔽性（Covertness）：检测 IDT 挂钩往往是轻而易举的，而且这是 rootkit 检测工具的一种常见做法[32]。

### 2.2.2) GDT / LDT / 全局描述符表 / 局部描述符表

全局描述符表（Global Descriptor Table，GDT）和局部描述符表（Local Descriptor Table，LDT）用来存放段描述符，段描述符描述的是系统地址空间的一个视图。每一个处理器都有它自己的 GDT。段描述符包括基址、界限、特权信息以及其他标志，处理器在把逻辑地址（logical address，形式为 seg:offset，即段选择子加偏移）翻译成线性地址（linear address）时会用到它们。段选择子（segment selector）是一个整数，用来根据各个段描述符在给定描述符表中的偏移，间接地引用它们。软件通过段寄存器（segment register），例如 CS、DS、ES 等，来使用段选择子。关于分段行为的更多细节，可以在 x86 和 x64 系统编程手册中找到[1]。

在 Phrack 55 中，Greg Hoglund 描述了滥用一致代码段（conforming code segment）的可能性[19]。一致代码段与非一致代码段（non-conforming code segment）相对，它允许 CPL 在数值上大于 DPL 时的控制转移。这里先说明：CPL 即 Current Privilege Level，当前特权级，是当前正在执行的代码所处的特权级；DPL 即 Descriptor Privilege Level，描述符特权级，是段描述符中标明的、访问该段所需的特权级。但是，作为这种控制转移的结果，CPL 并不会被改变。因此，调用者的有效特权并不会发生变化。出于这个原因，尚不清楚这如何能被用来访问内核模式内存，因为在启用分页的情况下，页保护仍然会阻止特权较低的调用者访问内核模式页。

Derek Soeder 在 2003 年发现了一个很棒的缺陷，它允许一个用户模式进程在调用进程的 LDT 中创建一个向下扩展段描述符（expand-down segment descriptor）[40]。向下扩展段描述符会反转与段描述符相关联的界限和基址的含义。以这种方式，界限描述的是下限，基址描述的是上限。这之所以有用，是因为当内核模式例程验证从用户模式传入的地址时，它们假设的是从基址零开始的平坦段（flat segment）。这等同于假设一个逻辑地址等同于一个线性地址。然而，当使用向下扩展段描述符时，线性地址所引用的内存位置，可能与内核模式正在验证的地址形成鲜明的对比。为了利用这一条件来提升特权，所需要做的全部事情，就是找出一个会在提升后的特权下运行的内核模式系统服务，并让它在没有恰当验证的情况下使用由用户模式提供的段选择子。Derek 给出了一个例子：`int 0x2e` 处理程序中的一条 MOVS 指令。这个技巧可以在本地内核模式后门的语境中被滥用，为用户模式代码提供一种能够读写内核模式内存的方法。

除了滥用通过 GDT 和 LDT 引用内存方式中的具体缺陷之外，还可以定义自定义的门描述符，使得从用户模式调用内核模式中的代码成为可能[23]。在后门的语境中，一种特别有用的门描述符类型是调用门描述符（call gate descriptor）。调用门（call gate）是门描述符的一种，其用途是允许特权较低的代码以一种安全的方式调用特权较高的代码[45]——调用者不能随意跳进高特权代码的中间，只能经由调用门中预先登记好的入口点进入，处理器在转移时负责切换特权级和栈。为了滥用这一点，后门可以简单地定义它自己的调用门描述符，然后利用它在内核的语境中运行代码。

类别（Category）：Type IIa（第二类 a）；LDT 除外。考虑到 LDT 向用户模式暴露了一个允许创建自定义 LDT 条目的 API（`NtSetLdtEntries`），把 LDT 归为 Type II（第二类）可能更为合适。

起源（Origin）：目前尚不清楚，是否存在某些情形上的要求，是滥用 Greg Hoglund 所描述的那个问题所需要的。Derek Soeder 在 2003 年发现的那个缺陷，是一个曾在其他操作系统的旧版本（例如 Linux）中发现过的问题重现的一个例子。例如，Morten Welinder 于 1996 年发到 LKML（Linux Kernel Mailing List，Linux 内核邮件列表）上的一封邮件，描述了针对一个看起来与 Derek 所发现的是同一类型问题的修复[44]。为了在后门的语境中使用而创建一个自定义门描述符，这在过去已经被使用过。Greg Hoglund 曾在 1999 年描述过在 rootkit 的语境中使用调用门[19]。

能力（Capabilities）：在向下扩展段描述符的情形下，可以访问内核模式数据。这也可以间接地导致内核模式代码执行，但那将依赖于另一种后门技术。如果门描述符被滥用，则可以直接实现内核模式代码执行。

隐蔽性（Covertness）：完全有可能编写出能够检测 GDT 或每个进程各自的 LDT 中条目的添加或更改的代码。例如，PatchGuard 目前就会检测对 GDT 的更改。

### 2.2.3) SSDT / 系统服务描述符表

系统服务描述符表（System Service Descriptor Table，SSDT）在 Windows 内核分发系统调用时被使用。SSDT 本身在内核模式中通过 `nt!KeServiceDescriptorTable` 全局变量导出。这个变量包含与已经向操作系统注册的系统调用表相关的信息。与其他操作系统形成对比的是，Windows 内核支持在运行时动态注册（`nt!KeAddSystemServiceTable`）新的系统调用表。两个最常见的系统调用表是用于原生（native）系统调用和 GDI 系统调用的表。

在本地内核模式后门的语境中，系统调用是一个明显的目标，因为它们隐含地与存在于用户模式和内核模式之间的特权边界联系在一起。在内核模式中挂钩一个系统调用处理程序，使得利用操作系统定义良好的系统调用接口、向内核中暴露一个有特权的后门成为可能。此外，挂钩系统调用使得后门能够更改用户模式所看到的数据，从而在某种程度上潜在地隐藏其存在。

在实践中，在 Windows 上可以用两种截然不同的策略来挂钩系统调用。第一种策略涉及使用 中所描述的通用函数挂钩技术。第二种策略涉及使用 中所描述的函数指针挂钩技术。使用函数指针挂钩，只需要通过访问包含待挂钩系统调用的系统调用表，更改与某个特定系统调用索引相关联的函数指针即可。

以下代码非常简单地说明了，在 32 位版本的 Windows 上，人们可以如何着手挂钩原生系统调用表中的一个系统调用。在 64 位版本的 Windows 上挂钩系统调用，则需要禁用 PatchGuard：

```c
PVOID HookSystemCall(
  PVOID SystemCallFunction,
  PVOID HookFunction)
{
  ULONG SystemCallIndex =
    *(ULONG *)((PCHAR)SystemCallFunction+1);
  PVOID *NativeSystemCallTable =
    KeServiceDescriptorTable[0];
  PVOID OriginalSystemCall =
    NativeSystemCallTable[SystemCallIndex];

  NativeSystemCallTable[SystemCallIndex] = HookFunction;

  return OriginalSystemCall;
}
```

类别（Category）：如果使用序言挂钩（prologue hook，即改写函数开头指令的挂钩），则为 Type I（第一类）。如果使用函数指针挂钩，则为 Type IIa（第二类 a）。SSDT（无论是原生的还是 GDI 的）实际上都应当被视为只写一次（write-once）的。

起源（Origin）：系统调用挂钩在相当长的一段时间里已经被广泛使用。由于这项技术已经变得如此广为人知，其真正的起源并不清楚。作者能找到的最早描述来自 M. B. Jones 于 1993 年发表的一篇名为 Interposition agents: Transparently interposing user code at the system interface（介入代理：在系统接口处透明地插入用户代码）的论文[27]。Jones 在他关于相关工作的那一节中解释说，在他写作之前，他无法找到任何关于基于代理的介入这一主题的明确研究。然而，显而易见的是，早在这一时间点之前很久，系统调用就已经以一种临时、零散（ad-hoc）的方式被挂钩了。作者无法找到 Jones 所引用的许多论文。Plaguez 似乎是第一批（1998 年 1 月）在 Phrack 52 中公开说明系统调用挂钩在 Linux 中的有用性的人之一，他特别着眼于安全方面[30]。

能力（Capabilities）：内核模式代码执行。

注意事项（Considerations）：在某些版本的 Windows XP 上，SSDT 被标记为只读的。在试图跨多个 Windows 版本写入 SSDT 时，必须把这一点考虑在内。

隐蔽性（Covertness）：Windows 上的系统调用挂钩非常容易被检测。将内存中的 SSDT 与磁盘上的版本进行比较，是所采用的最常见的策略之一。

## 2.3) Model-specific Registers（模型特定寄存器）

Intel 处理器支持一类特殊的、与处理器相关的寄存器，称为模型特定寄存器（Model-specific Registers，MSR）。MSR 使软件能够控制各种硬件和软件特性。与其他寄存器不同，MSR 与特定的处理器型号绑定，并不保证在处理器产品线的未来版本中仍然得到支持。MSR 提供的部分功能包括增强的性能监控和调试等。软件可以使用 `rdmsr` 指令读取 MSR，使用 `wrmsr` 指令写入 MSR[23]。

本小节将介绍一些在本地内核模式后门（local kernel-mode backdoor）场景中可能有用的 MSR。

### 2.3.1) IA32_SYSENTER_EIP / IA32_SYSENTER_EIP（SYSENTER 入口指针寄存器）

Pentium II 引入了对用户模式与内核模式之间转换的增强支持。这一支持是通过引入两条新指令提供的：`sysenter` 和 `sysexit`。AMD 处理器也引入了增强的新指令来提供这一特性。当用户模式应用程序希望转换到内核模式时，它会发出 `sysenter` 指令。当内核准备好返回用户模式时，它会发出 `sysexit` 指令。与 `call` 指令不同，`sysenter` 指令不接受任何操作数。相反，这条指令使用三个由操作系统初始化的特定 MSR 作为控制转移的目标[23]。

这三个与 `SYSENTER`/`SYSEXIT` 快速系统调用机制相关的 MSR 分别是：`IA32_SYSENTER_CS`（`0x174`）MSR，由处理器用来设置内核模式的 `CS`；`IA32_SYSENTER_EIP`（`0x176`）MSR，其中包含转换完成后代码应开始执行的内核模式入口点的虚拟地址；第三个 MSR 是 `IA32_SYSENTER_ESP`（`0x175`），其中包含栈指针应被设置成的虚拟地址。在这三个 MSR 中，`IA32_SYSENTER_EIP` 就其在后门场景中的潜在用途而言是最有意思的。将这个 MSR 设置为后门所控制的某个函数的地址，就能使后门拦截所有在陷入（trap）内核模式之后的系统调用。这提供了一个非常强大的观察点（vantage point）。

关于 `sysenter` 和 `sysexit` 指令行为的更多信息，读者应查阅 Intel 手册以及 John Gulbrandsen 的文章[23, 15]。

类别（Category）：Type I（第一类）

起源（Origin）：这一特性被提供的明确目的，就是允许操作系统控制 `sysenter` 指令的行为。因此，顺理成章地，它也可以被应用到后门的场景中。Kimmo Kasslin 提到，2005 年 12 月的一个病毒曾利用 MSR 挂钩（MSR hooks）[25]。在同年更早的 2 月，rootkit.com 的 fuzenop 发布了一个概念验证（proof of concept）[12]。

能力（Capabilities）：内核模式代码执行（Kernel-mode code execution）

注意事项（Considerations）：这一技术受到一个事实的限制：并非所有处理器都支持这个 MSR。此外，用户模式进程在执行系统调用时，并不必然需要使用它来转换进入内核模式。这些事实限制了这一技术的有效性，因为它并不能保证在所有机器上都能工作。

隐蔽性（Covertness）：更改 `IA32_SYSENTER_EIP` MSR 的值是可以被检测到的。例如，PatchGuard 目前会检查等价的 AMD64 MSR 是否已被修改，这是其轮询检查（polling checks）的一部分[36]。第三方厂商要执行这一检查则更为困难，原因很简单：这个 MSR 的默认值是一个未导出的符号，名为 `nt!KiFastCallEntry`：

```
kd> rdmsr 176
msr[176] = 00000000`804de6f0
kd> u 00000000`804de6f0
nt!KiFastCallEntry:
804de6f0 b923000000      mov     ecx,23h
```

在没有符号（symbols）的情况下，第三方更难区分一个值究竟是合理的，还是不合理的。

## 2.4) Page Table Entries（页表项）

当运行在保护模式（protected mode）下时，x86 处理器通过一种称为分页（paging）的特性，支持对地址空间进行虚拟化。分页特性通过在线性地址（linear addresses）和物理地址（physical addresses）之间增加一个转换层，使得虚拟化地址空间成为可能。当分页未启用时，线性地址等同于物理地址。为了转换地址，处理器会使用被引用地址的若干部分，去索引目录和表；这些目录和表中传达了标志和物理地址信息，描述了转换应如何进行。关于这一转换如何执行的大部分细节都超出了本文的范围。如有必要，读者应查阅 Intel《系统编程手册》（Intel System Programming Manual）第 3.7 节[23]。参考文献中的许多其他论文也讨论了这一话题[41]。

分页系统由于其在后门场景中被滥用的潜力而格外引人关注。当处理器试图转换一个线性地址时，它会遍历（walk）若干页表，以确定与之关联的物理地址。当这一过程发生时，处理器会做一项检查，以确保引用该地址的任务拥有足够的权限这样做。这一访问检查是通过检查与该页相关联的页目录项（Page-Directory Entry，PDE）和页表项（Page-Table Entry，PTE）中的用户/管理者位（User/Supervisor bit）来强制执行的。页表项是页表中记录单个页的物理页框、存在/读写/用户-管理者等权限标志的条目；页目录项则是上一级结构、指向页表的条目。需要说明的是，Windows 还利用页表自映射（page table self-mapping）——即把页目录本身映射进虚拟地址空间、使页目录页同时充当一张页表，从而可以用普通的虚拟地址访问直接读写 PDE/PTE——这一机制与本文此处讨论的修改 PDE/PTE 正是同一访问路径。如果该位被清零，则只有管理者（supervisor，特权级 0）被允许访问该页。如果该位被置位，则管理者和用户都被允许访问该页。不过，情况并不总是如此，这取决于 `CR0` 中的 `WP` 位是否被置位。

这一标志的含义应当是显而易见的。通过切换与某个地址相关联的 PDE 和 PTE 中的这一标志，后门可以获得读取或写入内核模式内存的访问权。这将间接地使利用本文所列的其他技术之一来获得代码执行成为可能。

类别（Category）：Type II（第二类）

起源（Origin）：自硬件分页诞生以来，对 PDE 和 PTE 条目的修改就一直得到支持。作者未能找到这一技术在后门中首次使用的确切来源。近年来，有许多工具以这样或那样的方式滥用管理者位（supervisor bit）的例子[29, 41]。PaX 团队在 2003 年 3 月提供了他们 `PAGEEXEC` 代码的第一份文档。在 1998 年 1 月，Mythrandir 在 phrack 52 中提到了管理者位，但没有明确指出它如何能被滥用[28]。

能力（Capabilities）：访问内核模式数据（Access to kernel-mode data）。

注意事项（Considerations）：试图实现这一方法的代码，需要在 x86 上同时正确支持 PAE（Physical Address Extension，物理地址扩展）和非 PAE 处理器，才能可靠地工作。这一方法也极其危险，并且根据它与内存管理器（memory manager）交互的方式，可能会不可靠。例如，如果页没有被正确地锁定在物理内存中，它们可能会被修剪（pruned），于是任何对 PDE 或 PTE 的修改都会丢失。这将导致用户模式进程失去对某个特定页的访问权。

隐蔽性（Covertness）：在没有某种能够拦截 PDE 或 PTE 修改的工具存在的情况下，这一方法可以被认为相当隐蔽。将页锁定在物理内存中，可能会使它更容易以轮询方式被检测到：遍历已锁定页的集合，并检查它们相关联的 PDE 或 PTE 是否已被设为可供用户模式访问。

## 2.5) Function Pointers / 函数指针

Windows 内核广泛使用函数指针（function pointer，即保存函数地址、通过该地址间接调用函数的指针），把执行控制权从一处间接转移到另一处[18]。与前文所述的覆写函数序言（function prologue，函数开头的一段指令）的做法一样，通过修改函数指针来挂钩（hook，即拦截并改道函数的调用）某个函数，是一种拦截该函数后续调用的简便办法。不过二者有一个区别：通过修改函数指针挂钩函数，只能拦截经由该函数指针、对被挂钩函数发起的间接调用。这一点看似是相当显著的限制，但即便有这样的限制，可被滥用来提供内核模式后门（kernel-mode backdoor，即在内核模式下提供隐蔽控制途径的机制）的函数指针集合，也并没有被大幅缩小。

其概念本身应当足够简单。所要做的全部事情，就是把某个给定函数指针的内容改成指向不受信任的代码。当函数经由该函数指针被调用时，执行的便是那段不受信任的代码，而不是原函数。如果不受信任的代码希望还能调用被挂钩的那个函数，它可以在覆写函数指针之前，先保存其中存着的地址。只要条件允许，通过函数指针挂钩函数就是一种简单而优雅的方案，对系统稳定性的影响应当非常小（替换函数的质量所带来的明显例外除外）。

无论采用哪种方法挂钩函数，都有一个显而易见的问题：与某个给定挂钩函数相关联的后门代码，应当放在哪里。实际上，代码能存放的内存位置只有两大类。它既可以存放在用户模式（user-mode）下——这样它通常就只与某个特定进程相关；也可以存放在内核模式（kernel-mode）下——这样它在整个系统范围内都可见。选用两处中的哪一处，取决于所利用的那个函数指针的上下文限制。例如，如果函数指针是在升高的中断请求级别（IRQL，即 Interrupt Request Level，处理器执行优先级的分级）下被调用的，比如 DISPATCH（分发级）级别，那么挂钩函数的代码就不能存放在可分页内存（pageable memory，可被换出到磁盘的内存）中。另一个限制的例子，是函数指针被使用时的进程上下文（process context，即当时正在运行的那个进程的地址空间与执行环境）。如果一个函数指针可能在任意进程上下文中被调用，那么在用户模式下能放置代码的位置就只有有限的几处。理解代码可以存放的一些具体位置，是很重要的。

可以用来存放将在内核模式下执行的代码的最显而易见的位置，或许就是内核池（kernel pool），例如分页池（PagedPool）和非分页池（NonPagedPool）；它们用于存放动态分配的内存。在某些情况下，也可能把代码存放在包含与设备驱动程序（device driver）相关联的代码或数据的内存区域中。上面这几个例子说明，可存放代码的位置肯定不缺；但有几处位置特别值得点名指出。

其中一处由单个物理页（physical page）构成，该页在用户模式与内核模式之间共享。这个物理页称为 SharedUserData（共享用户数据页），它映射到用户模式时为只读，映射到内核模式时为可读可写。在所有 Windows NT+ 版本上，这个物理页所映射到的虚拟地址（virtual address）在用户模式（0x7ffe0000）和内核模式（0xffdf0000）下都是静态固定的。自 Windows XP SP2 起，这些虚拟映射不再可执行。不过，后门完全可以修改这些页面的权限。此外，为 SharedUserData 分配的那一页里还有大量未使用的内存。由于映射地址是静态的，它就成了一个有用的位置：可以在此存放少量代码，而无需再从分页池或非分页池额外分配存储空间[24]。

尽管 SharedUserData 映射相当有用，实际上还有另一处可用来存放代码的位置，可以说更加隐蔽。这种方法，是把函数指针覆写为指向本地 DLL（native DLL）即 ntdll.dll 的虚拟映射中某段代码的地址。本地 DLL 的特殊之处在于，它是唯一一个保证会映射进每一个进程上下文的 DLL，包括 System 进程（系统进程）。由于 Windows 内核所作的一些假设，它在每个进程中也都映射在相同的基地址（base address）上。这些都是有用的特性；但使用 ntdll.dll 映射来存放代码的最佳理由是：这样做可以按进程相对（process-relative）的方式存放代码。要理解这在实践中如何运作，还需要一些额外的说明。

本地 DLL 即 ntdll.dll，分别在内核初始化和进程初始化期间，被映射进 System 进程以及后续进程的地址空间。这一映射在内核模式下由 nt!PspMapSystemDll 完成。人们可以在 System 进程的上下文中，通过调试器观察到这一映射的存在，如下所示。也可以采取同样这些基本步骤，确认 ntdll.dll 同样映射进了其他进程（命令 !vad 用于转储某个给定进程的虚拟地址目录（virtual address directory）。该目录包含对给定进程内各内存区域的描述）：

```
kd> !process 0 0 System
PROCESS 81291660  SessionId: none  Cid: 0004
    Peb: 00000000  ParentCid: 0000
    DirBase: 00039000  ObjectTable: e1000a68
    HandleCount: 256.
    Image: System
kd> !process 81291660
PROCESS 81291660  SessionId: none  Cid: 0004
    Peb: 00000000  ParentCid: 0000
    DirBase: 00039000  ObjectTable: e1000a68
    HandleCount: 256.
    Image: System
    VadRoot 8128f288 Vads 4
...
kd> !vad 8128f288
VAD     level start end   commit
...
81207d98 ( 1) 7c900 7c9af 5 Mapped  Exe
kd> dS poi(poi(81207d98+0x18)+0x24)+0x30
e13591a8  "\WINDOWS\system32\ntdll.dll"
```

要把 ntdll.dll 映射用作存放代码的位置，就必须理解修改该映射本身内容所带来的影响。与所有其他映像映射（image mapping，即把可执行映像文件映射进内存）一样，与 ntdll.dll 相关联的代码页被标记为写时复制（Copy-on-Write，COW，即多个进程起初共享同一物理页、任一方写入时才复制出私有页的机制），并且最初在所有进程之间共享。当数据被写入一个标记了 COW 的页面时，内核会分配一个新的物理页，并把共享页的内容复制到新分配的页中。随后，这个新的物理页便与正被写入的那个虚拟页相关联。对新页所作的任何修改，只有在作出修改的那个进程的上下文中才可见。正是这种行为，使得修改与某个映像文件相关联的映射内容，不会导致修改出现在所有进程上下文中。

基于能够对 ntdll.dll 映射作出按进程相对的修改这一能力，人们便可以存放这样的代码：它只在某个函数指针于特定进程的上下文中被调用时才会被用到。当不在特定进程上下文中调用时，执行的将是 ntdll.dll 默认映射中存在的任何代码。为了更好地理解这如何运作，走一遍一个具体的例子是有意义的。

在这个例子中，一个 rootkit（以隐蔽方式维持对系统控制的恶意软件）选择了这样一种后门做法：覆写在针对某个特定设备对象（device object，内核中表示设备的对象）、使用主功能码（major function）IRP_MJ_FLUSH_BUFFERS 分发 I/O 请求包（IRP，即 I/O Request Packet，I/O 请求包）时所使用的那个函数指针。处理 IRP_MJ_FLUSH_BUFFERS 类 IRP 的函数的原型如下所示：

```
NTSTATUS DispatchFlushBuffers(
    IN PDEVICE_OBJECT DeviceObject,
    IN PIRP Irp);
```

为了创建一个与特定上下文相关的后门，该 rootkit 选择把上述函数指针覆写为一个位于 ntdll.dll 之内的地址。默认情况下，rootkit 希望所有进程——除了那些知道该后门存在的进程——在向该设备对象发送 IRP_MJ_FLUSH_BUFFERS 时，都只是发生一次无操作（no-operation，即什么也不做）。而对于知道该后门的进程，rootkit 则希望在内核模式下发生任意代码执行（arbitrary code execution，即执行攻击者任意选定的代码）。为实现这一点，函数指针应当被覆写为 ntdll.dll 内的一个地址，该地址处包含一条 ret 0x8 指令。这样一来，对 IRP_MJ_FLUSH_BUFFERS 的调用就只会直接返回（而不完成该 IRP）。这条 ret 0x8 所在的位置，应当位于一段在用户模式下很少被执行的代码之中。对于希望在内核模式下执行任意代码的进程而言，事情简单得很：只需修改位于该 ret 0x8 指令地址处的代码即可。修改代码之后，该进程只需要在受影响的设备对象上，通过 FlushFileBuffers 函数发出一个 IRP_MJ_FLUSH_BUFFERS 即可。这种与上下文相关的代码执行之所以能够实现，是因为在大多数情况下，IRP 都是在请求进程的上下文中被处理的。

本小节的其余部分将描述一些具体的函数指针，它们可能是用作后门的有用目标。作者已尽量涵盖一些更有意思的、可被挂钩的函数指针例子。尽管如此，不言而喻，还有许多函数指针没有被明确描述。作者很乐意听到更多关于其他函数指针的消息——那些在本地内核模式后门这一语境下具有独特而有用特性的函数指针。

### 2.5.1) Import Address Table / 导入地址表

PE 映像（PE image，即采用可移植可执行文件格式的映像）的导入地址表（Import Address Table，IAT）用于存放从外部 PE 映像导入的函数的绝对虚拟地址[35]。当一个 PE 映像被映射进虚拟内存时，动态加载器（dynamic loader；在内核模式下，它就是 ntoskrnl，即 Windows 内核映像）负责根据依赖函数的实际虚拟地址位置，填充该 PE 映像的 IAT 内容。为简单起见，本说明排除了绑定导入（bound import，即预先把导入地址绑定到预期位置的优化）。编译器则会生成使用间接调用（indirect call）指令来调用导入函数的代码。每个导入函数在 IAT 中都有一个函数指针槽位（slot）。以这种方式，PE 映像无需预先知道依赖的 PE 映像将被映射到虚拟内存中的何处。相反，这一知识可以推迟到运行时再确定。

挂钩一个 IAT 表项（entry）所涉及的根本步骤，其实就归结为改变一个函数指针。IAT 挂钩与其他类型的函数指针挂钩的区别，在于被覆写的函数指针被调用时所处的上下文。由于每个 PE 映像都有自己的 IAT，对某个给定 IAT 所作的任何挂钩，都会隐含地只影响与之相关联的那个 PE 映像。例如，设想这样一种情形：foo.sys 和 bar.sys 都导入了 ExAllocatePoolWithTag。如果 foo.sys 中 ExAllocatePoolWithTag 的 IAT 表项被挂钩，那么只有从 foo.sys 内部发出的对 ExAllocatePoolWithTag 的调用会受影响。从 bar.sys 内部对同一函数发出的调用则不受影响。这种类型的限制，视某个给定后门背后的动机而定，实际上可以是一件好事。

类别（Category）：类型 I（Type I）；可以被合法修改，但应当指向预期的值。

起源（Origin）：第一个 IAT 挂钩的起源尚不清楚。2000 年 1 月，Silvio 描述了通过 ELF PLT（Procedure Linkage Table，过程链接表）进行挂钩的做法；它在某些方面与 PE 映像中的 IAT 在功能上等价。

能力（Capabilities）：内核模式代码执行。

考虑事项（Considerations）：假定 IAT 挂钩的调用限制对于某个给定后门来说是可以接受的，那么就没有其他需要考虑的额外事项了。

隐蔽性（Covertness）：现代工具可以通过分析内核模式下加载的每个 PE 映像的 IAT 内容，来检测 IAT 挂钩。要检测出不一致之处，工具只需检查 IAT 中每个函数所关联的虚拟地址，是否确实与包含某个依赖函数的 PE 映像所导出的虚拟地址相同。

### 2.5.2) KiDebugRoutine / 内核调试例程指针

Windows 内核提供了一套广泛的调试接口（debugging interface），以便内核本身（以及第三方驱动程序）能够在一个实时的、交互式的环境中被调试（与事后的、验尸式的崩溃转储（crash dump）调试相对）。内核调试器程序（kd.exe 或 WinDbg.exe）使用这一调试接口，来执行诸如按需检查内核运行状态（包括内存、寄存器，以及进程、线程等内核状态）之类的任务。该调试接口还为内核提供了一些设施，用于向内核调试器报告各种值得关注的事件，例如异常（exception）、模块加载事件、调试打印输出，以及其他少数几种状态转换。因此，内核调试器接口在内核的各个部分都“内置”了挂钩点，其目的就是把这些事件通知给内核调试器。

内核调试器的能力波及面极广，再加上内核调试器接口（总体上）在所有操作系统构建版本（OS build）中都以兼容的方式存在，这就提供了一种有吸引力的、可用来夺取系统控制权的机制。通过把 KiDebugRoutine 篡改（subvert，即暗中改道其用途）为指向一个自定义回调函数（callback function），便有可能在一些关键时刻悄然夺得控制权（调试打印、异常分发、内核模块加载是首要候选）。

内核调试器事件通知接口的架构，可以概括为内核中的一个全局函数指针（KiDebugRoutine）。若干段不同的代码，例如异常分发器（exception dispatcher）、模块加载器（module loader）等等，都被设计为通过 KiDebugRoutine 进行调用，以便把事件通知给内核调试器。为了把内核调试器未激活的场景下的开销降到最低，KiDebugRoutine 通常被设置为指向一个哑函数（dummy function）KdpStub；它几乎不做任何动作，基本上只是立即返回到调用者处。然而，当系统在启用内核调试器的情况下启动时，KiDebugRoutine 可能被设置为另一个函数 KdpTrap；它把调用者提供的信息传递给远程调试器。

启用或禁用内核调试器，传统上一直只是启动时（boot-time）才能作的决定；但较新的操作系统构建版本，例如 Windows Server 2003 及以后版本，对把系统从“内核调试器未激活”状态转换到“内核调试器已激活”状态有了一些支持。因此，现在哑例程（KdpStub）中被加入了一些额外的逻辑；在某些情况下，它可能导致调试器被按需激活（on-demand）。这种情况下，在按需完成内核调试器初始化之后，控制权会被传递给真正的调试器通信例程（KdpTrap）。所以，在某些情况下，KdpStub 会把控制权传递给 KdpTrap。

此外，在 Windows Server 2003 及更高版本中，可以在运行时即时禁用内核调试器。这可能导致 KiDebugRoutine 被改回指向 KdpStub，而不是启动时分配的 KdpTrap。把这一行为与前面的要点结合起来，是想说明：即便系统是在启用内核调试器的情况下启动的，仅仅强制执行一条“KiDebugRoutine 在系统整个生命周期内不得改变”的策略，也可能是不够的。

除了异常分发通知之外，大多数调试事件都是经由中断 0x2d（interrupt 0x2d）找到 KiDebugRoutine 的；该中断又称为 “DebugService”（调试服务）。这包括用户模式的调试打印事件，以及源自内核模式的事件（例如内核模块加载事件）。中断 0x2d 的陷阱处理程序（trap handler）会把提供给调试服务中断的信息打包成一种特殊异常的格式，随后经由 KiExceptionDispatch（即中断生成异常的正常异常分发器路径）分发。这一过程进而导致 KiDebugRoutine 作为异常分发器运作的正常组成部分被调用。

类别（Category）：类型 IIa（Type IIa），情况不一。尽管在以前的操作系统版本上，KiDebugRoutine 本质上是只写一次（write-once）的，但近来的版本允许在系统启动的情况下，对这个值进行有限的即时修改。

起源（Origin）：在撰写本文时，作者尚不知道有现存恶意软件（malware）使用 KiDebugRoutine。

能力（Capabilities）：把 KiDebugRoutine 重定向到一个由调用者控制的位置，可以在异常分发期间（这是非常常见的事件）以及某些其他情况下（例如模块加载和调试打印输出）夺得控制权。还有一个额外的好处：由于 KiDebugRoutine 对于整个内核调试器设施的运作不可或缺，如果启用了内核调试器，就应当可以通过操纵哪些事件实际上被传递给 KdpTrap，来“过滤”内核调试器收到的事件。然而，应当注意，还需要采取其他步骤，才能阻止内核调试器检测到代码的存在，例如拦截内核调试器的读内存设施（read-memory facilities）。

考虑事项（Considerations）：视系统全局标志（NtGlobalFlag）的配置方式，以及系统是否以抑制向内核调试器通知用户模式异常的方式启动而定，异常事件可能并不总是会被投递到 KiDebugRoutine。此外，由于 KiDebugRoutine 并未导出，要拦截它，就有必要先定位它。再者，许多调试器事件都发生在任意上下文（arbitrary context）中，因此把 KiDebugRoutine 指向用户模式（ntdll 空间之内除外）可能被视为危险之举。即便把 KiDebugRoutine 指向 ntdll，也存在把系统搞崩的风险，因为某些调试器事件可能在系统无法容忍分页（paging）的情况下被报告（例如调试打印）。从线程安全（thread-safety）的角度看，对 KiDebugRoutine 进行原子交换（interlocked exchange）应当是一种相对而言同步安全的操作（不过，新的回调例程可能永远不能从地址空间中取消映射，除非有某种办法能确保没有回调正在活动）。

隐蔽性（Covertness）：由于 KiDebugRoutine 是一个未导出的、可写的内核全局变量，它对简单的检测技术具有一些固有的防御能力。然而，在合法的系统运行中，KiDebugRoutine 只有两个合法值：KdpStub 和 KdpTrap。尽管这两个例程都没有导出，但多种检测技术组合起来（例如验证只读内核代码的完整性，以及验证 KiDebugRoutine 指向内核内存映像中预期代码区域内的某个位置），可能会让针对 KiDebugRoutine 的公然攻击更容易被定位。例如，只需把 KiDebugRoutine 设置为指向内核之外的某个位置，用这种方法就可能被检测出来；把它指向内核内别处然后向那里写入，同样可能被检测出来（因为目标位置要么必须位于正常代码区域之外、容易被检测，要么就必须覆写通常只读的代码，同样相对容易被检测）。此外，所有版本的 PatchGuard（补丁卫士，微软的内核补丁保护机制）都会在 x64 版本的 Windows 上保护 KiDebugRoutine。这意味着，在这类系统上长期有效地利用 KiDebugRoutine，需要攻击者去对付 PatchGuard。作者认为，这只是一个相对较小的困难。

### 2.5.3) KTHREAD's SuspendApc / KTHREAD 的 SuspendApc

为了支持线程挂起（thread suspension），Windows 内核在与系统中每个正在运行的线程相关联的 KTHREAD 结构（线程的内核表示结构）中，包含了一个名为 SuspendApc 的 KAPC 字段（KAPC 即内核异步过程调用对象）。当有人请求挂起线程时，内核会采取步骤，把 SuspendApc 结构排入该线程的 APC 队列（APC queue，即异步过程调用队列）。当 APC 队列被处理时，内核会从 SuspendApc 结构中调用该 APC 的 NormalRoutine，其通常被初始化为 nt!KiSuspendThread；调用发生在正被挂起的那个线程的上下文中。一旦 nt!KiSuspendThread 完成，该线程便被挂起。下图显示了 SuspendApc 通常被初始化为什么样的值：

```
kd> dt -r1 _KTHREAD 80558c20
...
 +0x16c SuspendApc      : _KAPC
  +0x000 Type           : 18
  +0x002 Size           : 48
  +0x004 Spare0         : 0
  +0x008 Thread         : 0x80558c20 _KTHREAD
  +0x00c ApcListEntry   : _LIST_ENTRY [ 0x0 - 0x0 ]
  +0x014 KernelRoutine  : 0x804fa8a1 nt!KiSuspendNop
  +0x018 RundownRoutine : 0x805139ed nt!PopAttribNop
  +0x01c NormalRoutine  : 0x804fa881 nt!KiSuspendThread
  +0x020 NormalContext  : (null)
  +0x024 SystemArgument1: (null)
  +0x028 SystemArgument2: (null)
  +0x02c ApcStateIndex  : 0 ''
  +0x02d ApcMode        : 0 ''
  +0x02e Inserted       : 0 ''
```

由于 SuspendApc 结构是针对某个给定 KTHREAD 的，对某个线程的 SuspendApc.NormalRoutine 所作的任何修改，都只影响那个特定的线程。通过修改与某个给定线程相关联的 SuspendApc 的 NormalRoutine，后门只需简单地尝试挂起该线程，便可以在内核模式下获得任意代码执行。对于用户模式应用程序来说，触发该后门是轻而易举的。下面的示例代码说明了：如果一个线程的 SuspendApc 已被修改，它如何可能在内核模式下执行任意代码：

```c
SuspendThread(GetCurrentThread());
```

下面的代码给出了一个汇编示例，它实现了上文所述的技术，并考虑了下文“注意事项”中所述的关于 `InitialStack`（初始栈，`KTHREAD` 结构中记录线程初始内核栈指针的字段）的洞见：

```asm
public _RkSetSuspendApcNormalRoutine@4
_RkSetSuspendApcNormalRoutine@4 proc
  assume fs:nothing
  push  edi
  push  esi
  ; Grab the current thread pointer
  xor   ecx, ecx
  inc   ch
  mov   esi, fs:[ecx+24h]
  ; Grab KTHREAD.InitialStack
  lea   esi, [esi+18h]
  lodsd
  xchg  esi, edi
  ; Find StackBase
  repne scasd
  ; Set KTHREAD->SuspendApc.NormalRoutine
  mov   eax, [esp+0ch]
  xchg  eax, [edi+1ch]
  pop   esi
  pop   edi
  ret
_RkSetSuspendApcNormalRoutine@4 endp
```

**Category（类别）：** Type IIa

**Origin（起源）：** 作者相信，这是对该技术的首次公开描述。这一想法应归功于 Skywing。Greg Hoglund 曾提到滥用 APC（Asynchronous Procedure Call，异步过程调用，内核将工作排入指定线程上下文中异步执行的机制）队列来执行代码，但他并未明确指出 `SuspendApc`（挂起 APC，线程挂起机制内部使用的那个特殊内核 APC）[18]。

**Capabilities（能力）：** 内核模式代码执行。

**Considerations（注意事项）：** 这一技术极其有效。它提供了一种简单的方法，只需劫持用于挂起某个特定线程的机制，就能在内核模式下执行任意代码。此外，还有一些值得一提的有趣副作用。覆写 `SuspendApc` 的 `NormalRoutine`（普通例程，APC 在满足投递条件时被调用的回调函数指针）会使该线程再也无法被挂起。更妙的是，如果替换 `NormalRoutine` 的钩子函数永不返回，那么该线程——进而其所属进程——将变得不可能被杀死，因为 `NormalRoutine` 是在 APC 级别（APC level，一种处理器中断请求级别 IRQL）被调用的。这两个副作用在 rootkit（隐藏自身并维持特权访问的恶意软件）的语境下都很有价值。

从后门的角度出发，必须考虑的一点是：需要设计出一种技术，能够跨多个 Windows 版本定位 `KTHREAD`（内核线程结构，内核用来描述和调度一个线程的核心数据结构）中的 `SuspendApc` 字段。幸运的是，有一些启发式方法可以做到这一点。在迄今分析过的所有 Windows 版本中，`SuspendApc` 字段前面紧挨着的都是 `StackBase`（栈基址）字段。已在多个操作系统上确认，`StackBase` 字段与 `InitialStack` 字段相等。而 `InitialStack` 字段在作者检查过的所有 Windows 版本上都位于一个可靠的偏移处（`0x18`）。利用这一知识，编写一段代码就变得轻而易举：按指针大小对齐的偏移扫描 `KTHREAD` 结构，直到遇到一个等于 `InitialStack` 的值为止。一旦找到匹配，就可以假定紧随其后的就是 `SuspendApc`。

**Covertness（隐蔽性）：** 这一技术涉及覆写一块动态分配内存区域中的一个函数指针，该区域与某个特定线程相关联。这使得该技术相当隐蔽，但并非不可能被检测。检测该技术的一种方法是：枚举每个进程中的线程，查看 `SuspendApc` 的 `NormalRoutine` 是否被设置为预期值 `nt!KiSuspendThread`。对于微软以外的人来说，安全地实现这一点颇具挑战。作者目前不知道有任何工具能做到这一点。

### 2.5.4) Create Thread Notify Routine（创建线程通知例程）

Windows 内核为驱动程序提供了一种能力：注册一个回调函数，当线程被创建和终止时获得通知。这一能力通过 Windows Driver Model（WDM，Windows 驱动程序模型）导出函数 `nt!PsSetCreateThreadNotifyRoutine` 提供。当一个线程被创建或终止时，内核会枚举已注册回调的列表，并将该事件通知给它们。

**Category（类别）：** Type II

**Origin（起源）：** 注册一个在线程创建和终止时获得通知的回调这一能力，自 WDM 首次发布以来就已包含在内。

**Capabilities（能力）：** 内核模式代码执行。

**Considerations（注意事项）：** 这一技术之所以有用，是因为用户模式进程只需创建或终止一个线程，就能控制该回调的调用。此外，该回调将在创建或终止线程的那个进程的上下文中获得通知。这使得可以将回调例程设置为一个驻留在 `ntdll.dll` 内的地址。

**Covertness（隐蔽性）：** 这一技术的隐蔽性在于，后门可以混迹于任何其他已注册的回调之中。在没有一个已知良好状态可供比对的情况下，要断定某个已注册回调与后门相关，将颇具挑战。不过，也有一些迹象可以表明事有蹊跷，例如回调例程驻留在 `ntdll.dll` 中，或者它驻留在分页池（paged pool，可被换出到磁盘的内核内存池）或非分页池（non-paged pool，始终驻留在物理内存中的内核内存池）中。

### 2.5.5) Object Type Initializers（对象类型初始化器）

Windows NT 内核使用一种面向对象的方法来表示文件、驱动程序、设备、进程、线程等资源。每个对象都按一种对象类型进行分类。这种对象类型分类为内核提供了一种途径，可以对同一类型的对象应用通用动作，此外还有其他用途。在这一设计下，每个对象只与一种对象类型相关联。例如，进程对象与 `nt!PsProcessType` 对象类型相关联。用来表示对象类型的结构是 `OBJECT_TYPE`（对象类型结构），其中包含一个名为 `OBJECT_TYPEIN_ITIALIZER` 的嵌套结构。正是这第二个结构提供了一些可被用于后门的、特别有趣的字段。

正如人们可能预料的那样，最值得关注的字段是函数指针。如果这些函数指针非空，内核会在与某个特定对象类型相关联的一个对象的生命周期中的某些时刻调用它们。下面的调试器输出展示了这些函数指针字段：

```
kd> dt nt!_OBJECT_TYPE_INITIALIZER
...
   +0x02c DumpProcedure    : Ptr32
   +0x030 OpenProcedure    : Ptr32
   +0x034 CloseProcedure   : Ptr32
   +0x038 DeleteProcedure  : Ptr32
   +0x03c ParseProcedure   : Ptr32
   +0x040 SecurityProcedure : Ptr32
   +0x044 QueryNameProcedure : Ptr32
   +0x048 OkayToCloseProcedure : Ptr32
```

有两个相当容易理解的过程是 `OpenProcedure`（打开过程）和 `CloseProcedure`（关闭过程）。当给定类型的对象被打开和关闭时，分别会调用这些函数指针。这给了对象类型初始化器一个机会，对某个对象类型的一个实例执行某种通用操作。就后门而言，这暴露了一种机制：每当给定类型的对象被打开或关闭时，任意代码就能在内核模式下被执行。

**Category（类别）：** Type IIa

**Origin（起源）：** Matt Conover 在 XCon 2005 上做了一次精彩的演讲，讲的是对象类型初始化器如何被用来检测 rootkit[8]。反过来，它们也可以被用来给系统植入后门。作者不知道在 Conover 的演讲之前有任何公开的例子。Greg Hoglund 也在 2006 年 6 月提到了这类方法[18]。

**Capabilities（能力）：** 内核模式代码执行。

**Considerations（注意事项）：** 使用这一技术没有涉及任何独特的注意事项。

**Covertness（隐蔽性）：** 这一技术可以被那些旨在对照已知良好状态验证对象类型初始化器状态的工具检测到。目前，作者不知道有任何工具会执行这类检查。

### 2.5.6) PsInvertedFunctionTable（Ps 反向函数表）

随着 Windows for x64（面向 x64 的 Windows）的引入，异常处理方式相较于 x86 版 Windows 中的异常运行方式发生了重大变化。在 x86 版 Windows 上，异常处理程序本质上是由带有异常处理程序的例程在运行时按需注册的（更像是一种基于代码的异常注册机制）。在 x64 版 Windows 上，异常注册路径则使用一种更数据驱动的模型来完成。具体而言，异常处理（尤其是展开处理，unwind handling，即在异常传播时逐帧回退并恢复例程机器状态的处理）现在由附加在每个 PE（Portable Executable，可移植可执行文件）映像上的元数据驱动（称为“exception directory”，异常目录），该元数据描述了例程与其异常处理程序之间的关系、例程每个区域的异常处理函数指针是什么，以及如何以完全数据驱动的方式展开每个例程的机器状态。

尽管用数据驱动模型来完成异常与展开分发有显著优势，但相较于 x86 方法（它由一个异常与展开处理程序的链表组成，这些处理程序注册在一个已知位置、且按线程划分），它也存在潜在的性能代价。一个具体的例子是：在 Windows for x86 上，操作系统定位并调用异常处理程序以进行异常或展开处理所需的全部信息都集中在一处（`NTTIB` 中的链表）；而在 Windows for x64 上，这些信息如今散布在所有已加载模块之中。为了给某个特定例程定位异常处理程序，必须搜索已加载模块列表，找到包含与所涉函数相对应的指令指针的那个模块。定位到模块之后，还必须处理该模块的 PE 头，以定位该模块的异常目录。最后，还必须在该模块的异常目录中搜索与涵盖所请求指令指针的位置相对应的元数据。对于异常可能穿越的每一个函数，这一过程都必须重复一遍。

为了提高 Windows for x64 上异常分发的性能，微软开发了一个多级缓存系统，它能加速解析异常分发信息——这项工作由负责查找与函数相关联的元数据的例程使用。负责此事的例程名为 `RtlLookupFunctionTable`。在搜索展开信息（一个指向 `RUNTIME_FUNCTION` 条目结构的指针）时，根据搜索请求的原因不同，可能会先搜索一个内部的一级缓存（`RtlpUnwindHistoryTable`），其中保存着常见函数的展开信息。在本文写作时，该表由 `RtlUnwindex`、`_C_specific_handler`、`RtlpExecuteHandlerForException`、`RtlDispatchException`、`RtlRaiseStatus`、`KiDispatchException` 和 `KiExceptionDispatch` 组成。由于异常分发在 x64 上的运行方式[39]，这些函数中有许多会经常出现在任何异常调用栈中。正因如此，为它们准备一个一级的快速查阅表对性能有益。

在搜索完 `RtlpUnwindHistoryTable` 之后，会扫描第二个缓存，称为 `PsInvertedFunctionTable`（在内核模式下）或 `LdrpInvertedFunctionTable`（在用户模式下）。这一二级缓存包含最先加载的 `0x200` 个（Windows Server 2008、Windows Vista）或 `0xA0` 个（Windows Server 2003）模块的列表。`PsInvertedFunctionTable` / `LdrpInvertedFunctionTable` 中包含的已加载模块列表，以一个可快速搜索的、未排序的线性数组形式呈现，它将一个完整已加载映像所占用的内存映射到给定模块的异常目录。因此，通过反向函数表（inverted function table，将映像地址范围反向映射到其异常目录的查找表）进行查找，省去了为定位某模块的异常目录所必需的、代价高昂的链表（已加载模块列表）遍历和可执行文件头解析步骤。对于被 `PsInvertedFunctionTable` / `LdrpInvertedFunctionTable` 引用的模块，在模块加载完成且反向函数表被填充之后，所涉模块 PE 头中的异常目录指针和大小信息就不再被使用。由于反向函数表的大小是固定的，如果同时加载的模块足够多，就可能出现这样的情况：在某一点之后，当某个模块加载时反向函数表的所有条目都已被占用，该模块就只能通过已加载模块列表查找来扫描。不过，这种情况很少发生，而且大多数有趣的系统模块（如 HAL，即 Hardware Abstraction Layer，硬件抽象层，以及内核内存映像本身）在 `PsInvertedFunctionTable` 中都处于开机时固定的位置[37]。

通过将 `PsInvertedFunctionTable` 中的异常目录指针重定向，使其指向调用者提供的内存中（位于实际模块 PE 头之外）的一个“影子”（``shadow''）异常目录，就有可能改变一个模块内所有代码点的异常（或展开）处理行为。例如，通过操纵异常目录信息，可以创建一个横跨一个模块内每一个代码字节的异常处理程序。通过更改一个模块的反向函数表缓存，就这一目标而言可以实现多重好处。第一，可以为展开元数据划出任意大的空间，因为打过补丁的展开元数据不必塞进某个特定映像异常目录的范围之内（如果有人希望“赠予”一个模块内的所有函数一个异常处理程序，这一点尤其重要）。第二，所涉模块的内存映像无需被修改，这提高了该技术对抗朴素检测系统的韧性。

**Category（类别）：** Type IIa，不一而足。尽管像 HAL 和内核内存映像本身这类常驻加载模块的条目本质上被视为只写一次（write-once），但当内核模块被加载或卸载时，该数组整体上在系统运行时是可以被修改的。因此，虽然 `PsInvertedFunctionTable` 的最初几个条目相对容易验证，但与按需加载（且可能按需卸载）的内核模块相对应的那些“动态”条目，在系统的合法运行期间可能会频繁变化；正因如此，拦截单个驱动程序的异常目录指针，可能远比拦截内核的异常目录更难被检测出来。

**Origin（起源）：** 在本文写作时，作者不知道有现存恶意软件在使用 `PsInvertedFunctionTable`。Skywing 曾提出劫持 `PsInvertedFunctionTable` 是 PatchGuard（内核补丁保护，微软用于校验关键内核结构的保护机制）第 2 版的一种可能绕过途径[37]。同一篇文章也简要描述了它作为一种与隐藏内核模式代码相关的可能攻击向量的适用性。

**Capabilities（能力）：** 这一技术提供的主要能力是：在目标模块内的任意位置建立一个异常处理程序（如果愿意，甚至可以是模块内的每一个代码字节处）。凭借创建这样的异常处理程序，就有可能在一个模块内任何可能被异常穿越的位置获得控制权，即便该异常本来会被该模块或该模块的调用者以安全的方式处理掉。

**Considerations（注意事项）：** 由于 `PsInvertedFunctionTable` 未被导出，要给它打补丁，就必须先定位它（这被认为是可能的，因为许多已导出的例程都以一种明显、有规律的方式引用它，例如 `RtlLookupFunctionEntry`）。另外，尽管该结构由一个未导出的同步机制守护（Windows Server 2008 中为 `PsLoadedModuleSpinLock`，已加载模块自旋锁），但对应于 HAL 和内核内存映像本身的最初几个条目应该是静态的，并且可以在不同步的情况下安全访问（毕竟，系统启动之后，HAL 和内核内存映像都不能被卸载）。应当可以执行一次互锁交换（interlocked exchange，原子地交换一个值的操作）来换掉异常目录指针，前提是交换完成后，异常目录不会以一种需要同步的方式被修改（例如，只能向其追加）。异常目录的大小在反向函数表条目数组中是作为一个单独的值提供的，需要被单独修改；如果对异常目录的更改没有被仔细规划，以确保在做出更改时、就并发访问而言在所有可能的突发情况下都是安全的，这就可能构成一个同步问题。此外，由于展开元数据采用基于 32 位 RVA（Relative Virtual Address，相对虚拟地址）的格式，一个模块的所有异常处理程序都必须位于距该模块加载基址 4GB 以内的范围内。这意味着自定义异常处理程序需要位于一片与某个模块相对接近的内存“窗口”之内。在特定基址分配内存涉及额外工作，因为内存不能位于地址空间中的任意一点，而必须在距目标 4GB 以内。不过，如果调用者能够查询地址空间并请求基于某个特定区域的分配，这并不被视为一个特别难以逾越的问题。

**Covertness（隐蔽性）：** 这种方法的主要优势在于，它允许调用者在模块执行过程中任何产生异常的点获得控制权，而无需修改所涉模块内的任何代码或数据（前提是该模块被缓存在 `PsInvertedFunctionTable` 内）。由于缓存被填充之后，模块的异常目录信息就不再被使用，针对 PE 头的完整性检查对于检测一个已缓存模块的异常处理行为更改是无用的。此外，`PsInvertedFunctionTable` 是一个未导出的、可写的内核模式全局变量，这使它对简单的检测技术具有某种内在的保护。只要已加载模块列表保持完整，扫描已加载模块列表并将其中的异常目录指针与 `PsInvertedFunctionTable` 中包含的指针进行比较，就能揭示这类攻击中的大多数。另外，PatchGuard 第 3 版似乎守护了 `PsInvertedFunctionTable` 的关键部分（例如，阻止重定向内核的异常目录），这导致在基于 Windows x64 的系统上进行长期利用时需要绕过 PatchGuard。作者认为这是一个相对次要的困难。

### 2.5.7) Delayed Procedures（延迟过程）

Windows 内核提供的许多特性都允许设备驱动程序异步地执行代码。这些特性的一些例子包括异步过程调用（APCs）、延迟过程调用（DPCs，Deferred Procedure Calls，被推迟到较低 IRQL 执行的内核例程）、工作项（work items，排队交由系统工作线程执行的工作单元）、线程等等。一个后门只需利用内核所暴露的 API，就可以利用其中任意多个特性来调度一个将在内核模式下运行任意代码的任务。例如，一个后门可能会使用本节开头描述的 `ntdll.dll` 技巧来排队一个内核模式 APC。当该 APC 执行时，它会在内核模式上下文中运行已在 `ntdll.dll` 中被更改过的代码。同样的基本概念对其余所有延迟过程都适用。

**Category（类别）：** Type II

**Origin（起源）：** 这一技术隐含地利用了操作系统所暴露的特性，因此属于显而易见的那一类。Greg Hoglund 尤其在 2006 年 6 月提到了这些[18]。

**Capabilities（能力）：** 内核模式代码执行。

**Considerations（注意事项）：** 这里重要的注意事项是，一些支持运行延迟过程的方法对代码页所在的位置有限制。例如，一个 DPC 是在分发级别（dispatch level，即 `DISPATCH_LEVEL` 这一 IRQL）被调用的，因此必须执行驻留在非分页内存中的代码。

**Covertness（隐蔽性）：** 这一技术的隐蔽性在于，后门始终处于一种瞬态的执行状态，因此可以被认为在很大程度上处于休眠状态。由于后门状态与其他瞬态状态一起存储在操作系统中，与本文所述的其他一些方法相比，这一技术应该更难被检测出来。

**作者：** skape & Skywing（2007 年 8 月）  
**来源：** Uninformed 第 8 卷  
**本片：** 第 7 片，共 8 片（2.6 异步读循环 与 2.7 泄漏 CS）

---

## 2.6) Asynchronous Read Loop（异步读循环）

在试图实现一个本地内核模式后门时，并不总是需要钩住（hook）内核的某个部分。在某些情况下，最简单的做法就是利用目标操作系统本身提供的功能，使后门与正常行为融为一体。有一个特别好的候选方案，就涉及滥用 Windows 的 I/O（input/output，输入/输出）管理器所提供的一些功能。

> **译注——异步读循环所借用的系统机制：** 这里借用的不是某个漏洞，而是 Windows I/O 管理器的正常异步完成机制。内核发起一次读操作时，会构造一个 IRP（见下文），把 IRP 交给目标设备对象去执行；读操作完成后，内核会调用 IRP 上预先登记的完成例程（completion routine）。本节的手法是把后门的内核例程本身登记为完成例程，并让它在每次被调用后重新投递下一个读 IRP，由此形成一个自我续接的循环：有数据到达就执行数据、执行完再挂起下一次读。它不需要常驻钩子，平时只是在等待 I/O 完成。

Windows 所使用的 I/O 模型有许多侧面。就本文的目的而言，只需要理解它在从文件读取数据时是如何运作的。为了支持这一过程，内核会构造一个 I/O 请求包（I/O Request Packet，IRP），并将其 `MajorFunction` 设置为 `IRP_MJ_READ`。随后，内核将填充好的 IRP 向下传递给与正在被读取的文件相关的设备对象。目标设备对象采取必要的步骤从底层设备读取数据，然后将获取到的数据存入与该 IRP 相关联的一个缓冲区中。一旦读操作完成，如果设置了 IRP 的完成例程，内核就会调用它。这给了原始调用者一个机会，可以利用已经读到的数据继续向前推进。

这种非常基本的行为，在后门的语境中可以被以一种相当隐蔽的方式有效利用。一个有趣的思路是：由一个用户模式进程托管一个命名管道（named pipe）服务器，再由一团（a blob of）内核模式代码从该服务器读取数据，然后在内核模式上下文中执行这些数据。通过简单地把附加代码经由一条命名管道穿梭运送过来，这种通用行为就使得在内核模式上下文中运行附加代码成为可能。使其能够运作的具体细节，几乎与上一段所描述的步骤一样简单。

用户模式部分很简单：使用 `CreateNamedPipe` 创建一个命名管道服务器，然后等待连接。内核模式部分则更有意思。一个基本的想法可能涉及这样一个内核模式例程：它构建一个异步读 IRP，并把该 IRP 的完成例程定义为该内核模式例程本身。这样一来，当数据从用户模式进程到达时，该例程就会得到通知，并获得一个机会来执行所提供过来的代码。在代码被执行之后，它可以简单地复用当初把 IRP 传递给与它正在交互的命名管道相关联的底层设备所需要的那些代码。下面的伪代码说明了如何做到这一点：

```
KernelRoutine(DeviceObject, ReadIrp, Context)
{
  // First time called, ReadIrp == NULL
  if (ReadIrp == NULL)
  {
    FileObject = OpenNamedPipe(...)
  }
  // Otherwise, called during IRP completion
  else
  {
    FileObject = GetFileObjectFromIrp(ReadIrp)

    RunCodeFromIrpBuffer(ReadIrp)
  }
  DeviceObject = IoGetRelatedDeviceObject(FileObject)
  ReadIrp = IoBuildAsynchronousFsdRequest(...)
  IoSetCompletionRoutine(ReadIrp, KernelRoutine)
  IoCallDriver(DeviceObject, ReadIrp)
}
```

**类别（Category）：** Type II（第二类）

**起源（Origin）：** 作者相信，这是对该技术的首次公开描述。

**能力（Capabilities）：** 内核模式代码执行。

**隐蔽性（Covertness）：** 作者相信该技术相当隐蔽，原因是内核模式代码的轮廓（profile）极其小。必须始终存在的代码只有这些：执行读缓冲区中的内容所需要的代码，以及随后向目标设备对象投递下一个读 IRP 所需要的代码。可能被用来检测该技术的策略主要有两种。第一种可能包括识别目标设备的恶意实例，例如一个恶意的命名管道服务器。第二种可能涉及尝试对完成例程代码做内存指纹（in-memory fingerprint）比对，不过这远非万无一失，尤其是在内核模式代码在被调用之前一直处于编码状态的情况下。

---

## 2.7) Leaking CS（泄漏 CS）

> **译注——CS 代码段选择子：** CS 是代码段寄存器（Code Segment Register），其中装载的是代码段选择子（segment selector）。在 x86 保护模式下，选择子的最低两位是请求特权级（Requested Privilege Level，RPL）；CS 选择子的最低两位被处理器用来推导当前特权级（Current Privilege Level，CPL）——即处理器此刻实际运行在哪个特权环上。把 CS 的低位清零，就等于把 CPL 伪装成 ring 0。本节所谓“泄漏 CS（Leaking CS）”，就是设法让一份原本属于内核的、ring 0 的 CS 值在返回用户模式时被带出去、被恢复进处理器，从而让原本的用户模式代码在 ring 0 下继续执行。

随着保护模式被引入 x86 架构，不同特权级——或者说环（rings）——的概念诞生了。特权较低的环（例如 ring 3）被设计为不能访问与特权较高的环（例如 ring 0）相关联的资源。为了支持这一概念，段描述符能够基于“哪些环应当被允许访问某一给定的内存区域”来定义访问限制。当 CS 段选择子被装载时，处理器查看它的低阶两位（low order two bits），由此推导出当前特权级（Current Privilege Level，CPL）。如果所有位都被清零，处理器就运行在 ring 0，即特权最高的环。如果所有位都被置位，那么处理器就运行在 ring 3，即特权最低的环。

当某些需要操作系统内核接管控制权的事件发生时，例如一次中断，处理器会自动从它当前正在执行的那个环转换到 ring 0，以便该请求可以由内核来提供服务。作为这一转换的一部分，处理器会把许多不同寄存器的值——包括 CS 先前的值——保存到栈上，以便在请求得到服务之后，能够从先前中断的地方恢复执行。下面的结构体描述了这些寄存器被保存到栈上的顺序：

```
typedef struct _SAVED_STATE
{
    ULONG_PTR Eip;
    ULONG_PTR CodeSelector;
    ULONG     Eflags;
    ULONG_PTR Esp;
    ULONG_PTR StackSelector;
} SAVED_STATE, *PSAVED_STATE
```

如果存在这样一种情形：某段代码能够以某种方式改动被保存的执行状态，使得被保存的 CS 被从一个特权较低的 CS 修改为一个特权较高的 CS——通过清零其低阶位来实现——那么就可能产生潜在的安全影响。当被保存的执行状态被用来恢复处理器的活动状态时，例如通过一条 `iret` 指令，原来的调用者便会立即获得 ring 0 特权。

**类别（Category）：** 未定义（Undefined）；这种方法不属于任何已定义的类别，因为它只是利用了硬件在“如何使用 CS 来判定一台处理器的 CPL”这件事上的行为。如果为了能够修改被保存的 CS 而使用了代码补丁（code patching），那么该实现就是 Type I（第一类）。

**起源（Origin）：** 自从保护模式（以及由此而来的环）随 80286 于 1982 年被引入 x86 架构以来 [22]，把 CS 泄漏给用户模式是危险的，这一点就已经为人所知。因此，由于在返回用户模式时泄漏一个内核模式 CS 所具有的、有文档记录的硬件影响，这一方法属于显而易见（obvious）的范畴。

**能力（Capabilities）：** 内核模式代码执行。

**注意事项（Considerations）：** 把内核模式 CS 泄漏给用户模式可能会带来不希望出现的后果。任何将要在用户模式下被调用的代码，都必须考虑到它将会在一个内核模式上下文中运行。此外，内核会尽可能严格地进行检查，以确保一个正在用户模式下执行的线程不被允许拥有一份内核模式 CS。

**隐蔽性（Covertness）：** 取决于用来截获并改动被保存的执行状态的方法，这种方法有可能是相当隐蔽的。如果该方法为了修改状态而涉及二次钩子（secondary hooking），那么它就可能通过一些与“映像补丁（image patching）”一节中所描述的相同的技术被检测出来。

**作者：** skape & Skywing　**出处：** Uninformed 第 8 卷，2007 年 8 月

> 本片为全文第 8 片（末片）：3) Prevention & Mitigation、4) Running Code in Kernel-Mode、5) PatchGuard versus Rootkits、6) Acknowledgements、7) Conclusion 与 References（参考文献，共 45 条）。

---

## 3) Prevention & Mitigation（预防与缓解）

本文的主要目的，并不是明确指出可以采取哪些办法来预防或缓解本文所述的各类攻击。不过，花一点时间来谈谈某些思路的优点，还是值得的——如果真有人想着手做预防或缓解，这些思路可能会带来极大的好处。至于如何防止后门被安装并持久化（persisted，即让后门在重启等情况下依然存留下来）这一主题，在第　节中有更详细的讨论〔译注：原文此处写作 “in section  and therefore”，章节编号缺失，疑为原文笔误／排版遗漏，照译并保留空缺〕，因此本节不再考虑这一问题。

有一个颇为有趣、可以用来阻止多种不同类型后门的想法，是不可变内存（immutable memory）。所谓内存是不可变的，指的是它不允许被修改。Windows 内核（kernel，操作系统核心）所使用的几块关键内存区域，会从不可变内存中获益良多，例如可执行代码段，以及那些实际上只写一次（write-once，写入一次之后便不应再改）的区域，例如 SSDT（System Service Descriptor Table，系统服务描述符表，即内核系统服务分发表）。不可变内存在原理上也许行得通〔译注：原文作 “immutable memory way work in principle”，其中 “way” 疑为 “may” 之笔误，照此理解译出〕，但据作者所知，目前还没有哪种 x86 或 x64 硬件能提供这种级别的控制。

尽管看起来没有硬件支持这一特性，在虚拟化（virtualized）环境中仍然可以实现不可变内存。在采用某种形式的 hypervisor（虚拟机监视器，位于客户操作系统之下、掌控硬件虚拟化的底层软件）的硬件辅助虚拟化实现中，尤其如此。在这种模型下，hypervisor 可以很容易地对外提供一个 hypercall（超级调用，类似于系统调用 system call，但陷入的是 hypervisor 而非内核），让一个「开明」的客户系统（enlightened guest，即知道自己运行在虚拟化环境中、并能主动与 hypervisor 协作的客户操作系统）把一组页（page，内存分页单位）标记为不可变。从那一刻起，hypervisor 就会限制对与该不可变区域相关联的页面的全部写入。

如前所述，不可变内存特别好的候选对象，是像 SSDT、Windows 的 ALMOSTRO 只写一次段（ALMOSTRO segment）〔译注：原文作 “Window's ALMOSTRO”，照录〕，以及内核中存在的其他单次修改（single-modification）数据元素。对这些区域强制实行不可变内存，将能有效地阻止后门建立某些类型的钩子（hook，即拦截并改道正常执行流程的挂钩点）。其代价是，内核将失去对自身进行热补丁（hot-patch，即不重启而在运行时修改正在执行的代码）的能力。当前确实有一些场合需要内核模式热补丁，尤其是在 x64 上〔译注：原文作 “kernel-mode hot-patching is currently require”，“require” 疑为 “required” 之笔误；且前一句 “…hot-patch itself” 之后缺句号，原文作 “…itself There are some instances…”，均照此理解译出〕。尽管如此，其安全上的收益看起来还是超过了潜在的代价。在 x64 上，使用不可变内存将提高 PatchGuard（内核补丁保护，微软用于检测内核关键结构是否被修改的机制）的韧性（resilience）：它可以主动阻止热补丁，而不是依赖轮询周期（polling cycle，定时反复检查）来检测热补丁。

---

## 4) Running Code in Kernel-Mode（在内核模式下运行代码）

有许多人可能会主张，根本没有必要去编写阻止或检测特定类型内核模式后门的代码。这一主张可以建立在两个非常具体的论点之上。第一个论点是：要想给内核植入后门，就必须有某种办法在内核模式下执行代码。按照这一推理思路，有人可能会主张，注意力应当转而放在阻止不受信任的代码在内核模式下运行上。这一主张的第二个论点是：要想真正攻陷（compromise）一台主机，就必须以某种形式持久化某些数据。如果假定情况确实如此，那么一个显而易见的解决办法，就是找出阻止或检测这些持久化数据的方法。除此之外或许还有其他论点，但上述两点代表了作者所观察到的共同主题。不幸的是，事实是：这两个论点在本文写作之时，都是站不住脚的。

以当今的操作系统和 x86/x64 硬件，要保证只有特定的代码才能在操作系统内核的上下文（context）中运行，目前是不可能的。尽管微软希望这是可能的——这一点从他们在 Code Integrity（代码完整性）和 Trusted Boot（可信启动）方面所做的努力中可以清楚地看出来——但实际上没有任何办法能保证内核模式代码不会以某种可能导致代码执行的方式被利用（exploit）[2]。Windows 内核模式漏洞并不缺乏，足以说明这类攻击向量（vector，攻击路径／入口）的可行性 [6, 10]。这一问题还因另一个事实而更加严重：Windows 内核目前几乎没有什么漏洞利用缓解措施（exploit mitigations，旨在提高漏洞利用难度的防护机制）。与 Windows XP SP2 上用户模式中、以及最近 Windows Vista 上用户模式中的某些缓解措施相比，这使得内核漏洞的利用变得轻而易举。

除了漏洞利用这一向量之外，还必须考虑在内核模式下执行代码的其他办法——这些办法在很大程度上对内核本身是不可见的。John Heasman 就「利用 BIOS（Basic Input/Output System，基本输入输出系统）、扩展 ROM（expansion ROM，外设卡上的固件只读存储器）和 EFI（Extensible Firmware Interface，可扩展固件接口）作为在内核上下文中运行任意代码的手段、而不必依赖任何内核本身直接可见的钩子」这一主题，提供了一些出色的研究成果 [16, 17]。Loïc Duflot 描述了如何利用 Intel 处理器的 SMM（System Management Mode，系统管理模式，处理器的一种高特权特殊运行模式）作为一种颠覆操作系统、绕过 BSD 的 securelevel 限制的方法 [9]〔译注：原文此处 “the  System” 中有双空格，属排版瑕疵，不影响文意〕。围绕利用 DMA（Direct Memory Access，直接内存访问，外设绕过 CPU 直接读写物理内存的机制）来直接与物理内存交互并修改物理内存、而不牵涉操作系统，也已经有过大量讨论。不过，这类攻击相对不那么令人担忧，因为它需要物理接触（physical access）才能实施。

检测或阻止 rootkit（rootkit，隐藏自身并维持特权访问的恶意软件）持久化数据这一想法，是值得深思的。确实，要想让恶意软件在重启之后依然存活，它必须以这样或那样的某种形式把自身持久化下来。通过阻止或检测这些被持久化的数据，就有可能有效地阻止任何形式的持续性感染。表面上看，这个想法似乎既简单又优雅，但魔鬼藏在细节里。这一想法从根本上站不住脚这一事实，用当前杀毒（Anti-Virus）技术的现状就可以明白地说明。

为了便于论证，先暂且假定：真的存在一种办法，可以确定性地（deterministically，即必然、可靠地）阻止恶意软件以任何形式持久化自身。现在，设想这样一个场景：一家金融机构的 Web 服务器被攻陷，并被植入了一个内存驻留（memory resident，只存在于内存中、不落盘）rootkit。这里的要点应该是显而易见的：与该 rootkit 相关联的任何数据都没有触及物理硬件。在这个例子中，人们可能会理所当然地认为，这台 Web 服务器在很长一段时间内都不会重启。在这种情况下，持久型 rootkit 和非持久型 rootkit 之间实际上没有任何区别。诚然，内存驻留 rootkit 在某些情形下可能并不理想，但理解其含义是很重要的。

以目前的技术水平（state-of-the-art）而言，要确定性地阻止恶意软件持久化自身，是不可能的。持久化数据的方法实在太多了。John Heasman 在其 ACPI（Advanced Configuration and Power Interface，高级配置与电源接口）和扩展 ROM 方面的研究，进一步说明了这一点。据作者所知，现代工具的取证分析（forensic analysis）聚焦于操作系统和文件系统。然而这并不充分，因为 rootkit 数据可以存放在对操作系统而言在很大程度上不可见的位置。尽管如此，近年来有一股强大的推动力，致力于提供实现可信系统启动（trusted system boot）所需的硬件。这一倡议由 Trusted Computing Group（可信计算组织）推动，微软和 Intel 等公司参与其中 [42]。该组织的主要成果之一是 TPM（Trusted Platform Module，可信平台模块，一种旨在协助实现可信系统启动等功能的安全芯片）[43]。在本文写作之时，TPM 的有效性在很大程度上尚属未知，但预计随着它日趋成熟，它将成为一个强大而有用的安全特性。

「确实没有任何办法能阻止不受信任的代码在内核模式下运行」这一事实，与「确实没有任何办法能普遍地阻止不受信任的代码持久化自身」这一事实结合起来，恰好说明了为什么需要认真思考如何既预防又检测内核模式后门。

---

## 5) PatchGuard versus Rootkits（PatchGuard 对决 Rootkits）

围绕 PatchGuard 能否被视为对 rootkit 的一种威慑（deterrent），一直存在一些困惑。表面上看，PatchGuard 似乎确实是 rootkit 开发者一个强大的对手，因为它会检查许多不同类型的钩子。然而，在表面之下，很清楚，PatchGuard 在作为 rootkit 威慑手段这一点上有着根本性的缺陷。这一缺陷的核心在于这样一个事实：在其当前实现中，PatchGuard 与其他驱动程序代码运行在相同的特权级别（privilege level）上。这就使 PatchGuard 暴露在那些旨在阻止它完成检查的攻击之下。作者此前已经概述了许多可以用来禁用 PatchGuard 的不同方法 [36, 37]。微软当然有可能针对这些攻击实现修复，而且事实上他们在较新的版本中已经实现了其中一些修复，但问题仍然是一场猫鼠游戏（cat-and-mouse game）。在这一场特定的猫鼠游戏中，rootkit 作者无论在时间上还是在制高点（vantage point，有利观察／攻击位置）上，都将始终占据优势。

在未来，PatchGuard 可以得到改进，去利用虚拟化环境中 hypervisor 的某些特性，从而有可能保护它免受在客户系统上下文中运行的恶意代码的侵害。例如，当前版本的 PatchGuard 目前大量使用混淆（obfuscation，故意使代码和数据难以辨认的技术），据推测是为了阻止恶意软件在内存中找到它的代码和上下文结构。hypervisor 的存在，可能允许 PatchGuard 更加广泛地使用不可变内存，或者转而在一个比正在执行的客户系统更高的特权级别上运行，例如在 hypervisor 本身之内运行（不过，如果做得不当，这可能会带来严重的安全影响）。

即便 PatchGuard 被改进到再也无法禁用其安全检查的程度，它仍然会存在另一个根本性的缺陷。这第二个缺陷的核心在于这样一个事实：PatchGuard 和其他任何被设计来执行显式检查（explicit checks）的代码一样，就像一匹戴了眼罩（blinders，遮挡侧面视线的马眼罩）的马。它只能检测它所知道的那些特定结构所遭受的修改。诚然，这些结构可能确实是最有可能被挂钩的候选对象，但同样真实的是，还存在许多其他结构也会是合适的候选对象，例如某个特定线程的 `SuspendApc`。列举这些替代候选对象，是为了说明 PatchGuard 在不断演进其检查手段、以跟上 rootkit 作者步伐方面所面临的挑战。以这种方式，PatchGuard 将继续被迫处于被动反应（reactive）模式，而不是主动预防（proactive）模式。如果说 IDS（Intrusion Detection System，入侵检测系统）产品说明了什么的话，那就是：被动反应型的安全解决方案，在熟练的攻击者面前，在很大程度上是不够的。

PatchGuard 最好还是被看作一个走廊监督员（hall monitor，负责在学校走廊维持秩序、监督学生守规的学生监督员）。它的职责是确保学生们按规矩做事。好学生，例如 ISV（Independent Software Vendor，独立软件开发商），天生就会服从 PatchGuard 的意志，以免发现自己身处不受支持的境地（unsupported waters）。坏学生，例如 rootkit，并不畏惧 PatchGuard 的怒火，即使将来用来绕过它的技术可能不再奏效，他们也不会对绕过它有多少顾虑。

---

## 6) Acknowledgements（致谢）

作者谨向所有人——无论具名还是未具名——表示感谢，他们先前的研究为本文所包含的内容作出了贡献。

---

## 7) Conclusion（结论）

到此应当已经清楚：可以用来在 Windows 上开设（expose，建立并暴露）一个本地内核模式后门的技术，并不缺乏。这些技术提供了一种微妙的（subtle，不易察觉的）办法，通过向用户模式进程暴露受限制的资源，来削弱 Windows 内核的安全保证（security guarantees）。这些资源可能包括对内核模式数据的访问、安全检查的禁用，或者在内核模式下执行任意代码。在 rootkit 的语境下，这些类型的后门之所以有用，原因有很多。

这些技术在 rootkit 中有用，最显而易见的原因，正是它们提供了对受限制资源的访问。一个不那么显而易见的有用原因，是它们可以用作一种减小 rootkit 内核模式代码特征（code profile，即 rootkit 留在内核模式中的代码体量／足迹）的方法。由于许多工具被设计来扫描内核模式内存、寻找后门的存在 [32, 14]，任何对 rootkit 内核模式代码特征的减小都可能是有用的。本文描述了一些技术，与其把代码放在内核模式中，不如把代码执行重定向（redirect）到以进程特定的（process-specific）方式存储在用户模式中的代码上。实现这一点的方法，是把代码执行重定向到 `ntdll` 映射（mapping）的一部分中——这份映射存在于每一个进程中，包括 System 进程（系统进程）。

理解不同的后门技术是如何工作的，对于考虑可以采取哪些办法来预防或检测使用这些技术的 rootkit，是必要的。例如，不可变内存的存在，可能会消除许多不同类型 rootkit 所使用的一些常见技术。同样，当这些技术被消除之后，新的技术将被开发出来，从而延续这个渗透于大多数对抗性系统（adversarial systems）之中的循环。

---

## References（参考文献）

[1] AMD. AMD64 Architecture Programmer's Manual Volume 2: System Programming（《AMD64 架构程序员手册第 2 卷：系统编程》）. Dec, 2005.

[2] Anonymous Hacker. Xbox 360 Hypervisor Privilege Escalation Vulnerability（《Xbox 360 Hypervisor 特权提升漏洞》）. Bugtraq. Feb, 2007. http://www.securityfocus.com/archive/1/461489

[3] Blanset, David et al. Dual operating system computer（《双操作系统计算机》）. Oct, 1985. http://www.freepatentsonline.com/4747040.html

[4] Brown, Ralf. Pentium Model-Specific Registers and What They Reveal（《Pentium 型号专用寄存器及其所揭示的信息》）. Oct, 1995. http://www.rcollins.org/articles/p5msr/PentiumMSRs.html

[5] Butler, James and Sherri Sparks. Windows Rootkits of 2005（《2005 年的 Windows Rootkits》）. Nov, 2005. http://www.securityfocus.com/infocus/1850

[6] Cerrudo, Cesar. Microsoft Windows Kernel GDI Local Privilege Escalation（《Microsoft Windows 内核 GDI 本地特权提升》）. Oct, 2004. http://projects.info-pull.com/mokb/MOKB-06-11-2006.html

[7] CIAC. E-34: Onehalf Virus (MS-DOS)（《E-34：Onehalf 病毒（MS-DOS）》）. Sep, 1994. http://www.ciac.org/ciac/bulletins/e-34.shtml

[8] Conover, Matt. Malware Profiling and Rootkit Detection on Windows（《Windows 上的恶意软件画像与 Rootkit 检测》）. 2005. http://xcon.xfocus.org/xcon2005/archives/2005/Xcon2005_Shok.pdf

[9] Duflot, Loïc. Security Issues Related to Pentium System Management Mode（《与 Pentium 系统管理模式相关的安全问题》）. CanSecWest, 2006. http://www.cansecwest.com/slides06/csw06-duflot.ppt

[10] Ellch, John et al. Exploiting 802.11 Wireless Driver Vulnerabilities on Windows（《利用 Windows 上的 802.11 无线驱动漏洞》）. Jan, 2007. http://www.uninformed.org/?v=6&a=2&t=sumry

[11] Firew0rker, the nobodies. Kernel-mode backdoors for Windows NT（《Windows NT 的内核模式后门》）. Phrack 62. Jan, 2005. http://www.phrack.org/issues.html?issue=62&id=6#article

[12] fuzenop. SysEnterHook（《SysEnterHook》）. Feb, 2005. http://www.rootkit.com/vault/fuzen_op/SysEnterHook.zip

[13] Garfinkel, Tal. Traps and Pitfalls: Practical Problems in System Call Interposition Based Security Tools（《陷阱与隐患：基于系统调用拦截的安全工具中的实际问题》）. http://www.stanford.edu/ talg/papers/traps/traps-ndss03.pdf
〔译注：URL 中 “stanford.edu/” 之后原文缺失 “~” 符号、含空格，照录原文。〕

[14] Gassoway, Paul. Discovery of kernel rootkits with memory scan（《用内存扫描发现内核 Rootkits》）. Oct, 2005. http://www.freepatentsonline.com/20070078915.html

[15] Gulbrandsen, John. System Call Optimization with the SYSENTER Instruction（《用 SYSENTER 指令优化系统调用》）. Oct, 2004. http://www.codeguru.com/Cpp/W-P/system/devicedriverdevelopment/article.php/c8223/

[16] Heasman, John. Implementing and Detecting an ACPI BIOS Rootkit（《实现与检测 ACPI BIOS Rootkit》）. BlackHat Federal, 2006. https://www.blackhat.com/presentations/bh-federal-06/BH-Fed-06-Heasman.pdf

[17] Heasman, John. Implementing and Detecting a PCI Rootkit（《实现与检测 PCI Rootkit》）. Nov, 2006. http://www.ngssoftware.com/research/papers/Implementing_And_Detecting_A_PCI_Rootkit.pdf

[18] Hoglund, Greg. Kernel Object Hooking Rootkits (KOH Rootkits)（《内核对象挂钩 Rootkits（KOH Rootkits）》）. Jun, 2006. http://www.rootkit.com/newsread.php?newsid=501

[19] Hoglund, Greg. A *REAL* NT Rootkit, patching the NT Kernel（《一个真正的 NT Rootkit：给 NT 内核打补丁》）. Phrack 55. Sep, 1999. http://phrack.org/issues.html?issue=55&id=5

[20] Hoglund, Greg and James Butler. Rootkits: Subverting the Windows Kernel（《Rootkits：颠覆 Windows 内核》）. 2006. Addison-Wesley.

[21] Hunt, Galen and Doug Brubacher. Detours: Binary Interception of Win32 Functions（《Detours：Win32 函数的二进制拦截》）. Proceedings of the 3rd USENIX Windows NT Symposium, pp. 135-143. Seattle, WA, July 1999. USENIX.

[22] Intel. 2.1.2 The Intel 286 Processor (1982)（《2.1.2 Intel 286 处理器（1982）》）. Intel 64 and IA-32 Architectures Software Developer's Manual. Denver, Colorado: Intel, 34. http://www.intel.com/products/processor/manuals/index.htm.

[23] Intel. IA-32 Intel Architecture Software Developer's Manual Volume 3: System Programming Guide（《IA-32 Intel 架构软件开发者手册第 3 卷：系统编程指南》）. Sep, 2005.

[24] Jack, Barnaby. Remote Windows Kernel Exploitation: Step into the Ring 0（《远程 Windows 内核利用：踏入 Ring 0》）. Aug, 2005. http://www.blackhat.com/presentations/bh-usa-05/BH_US_05-Jack_White_Paper.pdf

[25] Kasslin, Kimmo. Kernel Malware: The Attack from Within（《内核恶意软件：来自内部的攻击》）. 2006. http://www.f-secure.com/weblog/archives/kasslin_AVAR2006_KernelMalware_paper.pdf

[26] Kdm. NTIllusion: A portable Win32 userland rootkit [incomplete]（《NTIllusion：一个可移植的 Win32 用户态 Rootkit［未完成］》）. Phrack 62. Jan, 2005. http://www.phrack.org/issues.html?issue=62&id=12&mode=txt

[27] M. B. Jones. Interposition agents: Transparently interposing user code at the system interface（《拦截代理：在系统接口处透明地插入用户代码》）. In Symposium on Operating System Principles, pages 80-93, 1993. http://www.scs.stanford.edu/nyu/04fa/sched/readings/interposition-agents.pdf

[28] Mythrandir. Protected mode programming and O/S development（《保护模式编程与操作系统开发》）. Phrack 52. Jan, 1998. http://www.phrack.org/issues.html?issue=52&id=17#article

[29] PaX team. PAGEEXEC（《PAGEEXEC》）. Mar, 2003. http://pax.grsecurity.net/docs/pageexec.txt

[30] Plaguez. Weakening the Linux Kernel（《削弱 Linux 内核》）. Phrack 52. Jan, 1998. http://www.phrack.org/issues.html?issue=52&id=18#article

[31] Prasad Dabak, Milind Borate, and Sandeep Phadke. Hooking Software Interrupts（《挂钩软件中断》）. Oct, 1999. http://www.windowsitlibrary.com/Content/356/09/1.html

[32] Rutkowska, Joanna. System Virginity Verifier（《系统纯洁性验证器》）. http://invisiblethings.org/tools/svv/svv-2.3-src.zip

[33] Rutkowska, Joanna. Rookit Hunting vs. Compromise Detection（《Rootkit 狩猎与入侵检测》）. BlackHat Europe, 2006. http://invisiblethings.org/papers/rutkowska_bheurope2006.ppt
〔译注：题名原文作 “Rookit”，疑为 “Rootkit” 之笔误，照录原文，中文译名按 “Rootkit” 译出。〕

[34] Rutkowska, Joanna. Introducing Stealth Malware Taxonomy（《隐蔽恶意软件分类法导论》）. Nov, 2006. http://invisiblethings.org/papers/malware-taxonomy.pdf

[35] Silvio. Shared Library Call Redirection Via ELF PLT Infection（《通过 ELF PLT 感染重定向共享库调用》）. Phrack 56. Jan, 2000. http://www.phrack.org/issues.html?issue=56&id=7#article

[36] skape and Skywing. Bypassing PatchGuard on Windows x64（《绕过 Windows x64 上的 PatchGuard》）. Uninformed Journal. Jan, 2006. http://www.uninformed.org/?v=3&a=3&t=sumry

[37] Skywing. Subverting PatchGuard version 2（《颠覆 PatchGuard 第 2 版》）. Uninformed Journal. Jan, 2007. http://www.uninformed.org/?v=6&a=1&t=sumry

[38] Skywing. Anti-Virus Software Gone Wrong（《杀毒软件出错》）. Uninformed Journal. Jun, 2006. http://www.uninformed.org/?v=4&a=4&t=sumry

[39] Skywing. Programming against the x64 exception handling support（《针对 x64 异常处理支持编程》）. Feb, 2007. http://www.nynaeve.net/?p=113

[40] Soeder, Derek. Windows Expand-down Data Segment Local Privilege Escalation（《Windows 向下扩展数据段本地特权提升》）. Apr, 2004. http://research.eeye.com/html/advisories/published/AD20040413D.html

[41] Sparks, Sherri and James Butler. Raising the Bar for Windows Rootkit Detection（《提高 Windows Rootkit 检测的门槛》）. Phrack 63. Jan, 2005. http://www.phrack.org/issues.html?issue=63&id=8

[42] Trusted Computing Group. Trusted Computing Group: Home（《可信计算组织：主页》）. https://www.trustedcomputinggroup.org/home

[43] Trusted Computing Group. TPM Specification（《TPM 规范》）. https://www.trustedcomputinggroup.org/specs/TPM/

[44] Welinder, Morten. modifyldt security holes（《modifyldt 安全漏洞》）. Mar, 1996. http://lkml.org/lkml/1996/3/6/13

[45] Wikipedia. Call gate（《调用门》）. http://en.wikipedia.org/wiki/Call_gate
