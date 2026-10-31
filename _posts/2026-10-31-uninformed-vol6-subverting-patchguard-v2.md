---
layout: post
title: "Subverting PatchGuard Version 2 中文全译（Uninformed 第 6 卷，Skywing，2006）"
categories: essay
series: uninformed
series_order: 2
---

> 整理自 Uninformed 第 6 卷（2006 年 12 月）Skywing（Ken Johnson）的《Subverting PatchGuard Version 2》全文翻译——第 2 版的防拆强化、九条绕过路线，以及把完整性检查例程本身策反的完整演示，亦是《PatchGuard 攻防史》的事实底座之一（在该文中亦作附录三收录）。三部曲顺序：其一《Bypassing PatchGuard on Windows x64》→ 其二（本文，第 2 版）→ 其三《PatchGuard Reloaded》（第 3 版）。原文：http://www.uninformed.org/?v=6&a=1&t=txt

---

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

## 1) 前言（Foreword）

摘要（Abstract）：Windows Vista x64 以及最近打过热修复补丁的 Windows Server 2003 x64 内核版本，包含微软的内核模式补丁防护技术——即众所周知的 PatchGuard（内核补丁防护，一种阻止第三方未经授权修改内核代码与关键数据结构的技术）——的一个更新版本。这个新版本的 PatchGuard 在若干方面改进了先前的版本，主要涉及的是：从一个部署会给内核打补丁的驱动程序的独立软件供应商（ISV，Independent Software Vendor，独立软件供应商）的视角出发，设法提高绕过 PatchGuard 的难度。PatchGuard 第二版的功能集在其他方面与 PatchGuard 第一版相当相似；SSDT（System Service Descriptor Table，系统服务描述符表，内核中存放系统服务函数指针的表）、IDT/GDT（Interrupt Descriptor Table / Global Descriptor Table，中断描述符表／全局描述符表）、各种 MSR（Model Specific Register，型号专用寄存器，处理器特有的控制寄存器），以及若干内核全局函数指针变量（还有内核代码本身），都被保护起来，防止未经授权的修改。本文提出了若干可以用来彻底绕过 PatchGuard 第二版的方法。本文还提出了针对这些绕过技术的潜在解决方案。此外，本文描述了一种机制，借助它可以颠覆 PatchGuard 第二版，让它运行自定义代码来代替 PatchGuard 的系统完整性检查代码，而且在 PatchGuard 被颠覆之后，系统中不留下任何内核补丁或已加载的自定义内核驱动程序的痕迹。这一点从利用 PatchGuard 自身的防御来隐藏内核模式代码这一视角来看尤其有意思，而这一目标（在许多方面）与 PatchGuard 的设计初衷完全背道而驰。

致谢（Thanks）：作者想要感谢 skape、bugcheck 和 Alex Ionescu。

免责声明（Disclaimer）：本文的发表是出于教育以及增进公众一般知识的目的。对于本文所披露信息的任何潜在使用（或误用），作者不承担责任。尽管作者已经尽可能地警惕，以确保本文准确无误，但仍有可能残留一处或多处错误。如果发现了这样一处不准确或错误，作者将非常感谢能得到通知，以便做出适当的更正。

## 2) 引言（Introduction）

在 Windows 内核的 x64 版本上，微软试图采取一种激进的立场[1]，反对使用某一类技术——这类技术在先前版本的 Windows 上曾被频繁用来以潜在不安全的方式「扩展」内核。这包括给内核本身打补丁、钩住（hook，挂钩，即拦截并改道函数调用）内核的系统服务表、重定向中断处理程序，以及其他几种不太常见、在内核被触及之前拦截执行控制权的技术，例如篡改系统调用目标 MSR（即决定系统调用入口的型号专用寄存器）。

微软为阻止历史上在 x86 上猖獗的、未经授权的内核补丁而部署的技术，就称为 PatchGuard。这项技术最初随 Windows Server 2003 x64 Edition 和 Windows XP x64 Edition 一同发布（即众所周知的 PatchGuard 第一版）。Windows Vista 的 x64 版本，以及最近打过热修复补丁的 Windows Server 2003 x64 内核版本，包含一个较新版本的 PatchGuard 技术，即众所周知的 PatchGuard 第二版。新版本的设计目的是，让独立软件供应商（ISV）在禁用 PatchGuard 所提供的内核补丁防护机制之后，在实际部署环境中部署涉及给内核打补丁的解决方案，变得显著更加困难。PatchGuard 本身的内部细节与 PatchGuard 第一版时大体相同，因此本文不会详细讨论它们（第二版改进的反调试与反补丁技术除外）。足够有兴趣、希望了解该主题更多背景信息的读者，可以在 Uninformed 先前关于该主题的文章[2]《在 Windows x64 上绕过 PatchGuard》（"Bypassing PatchGuard on Windows x64"）中找到关于 PatchGuard 第一版如何运作的更多内容。

PatchGuard 第二版以最初的 PatchGuard 发布版为基础，试图堵住其基于混淆（obfuscation，即故意使代码与数据难以理解和分析）的反补丁系统在实现上的各种漏洞。在这一方面，它取得了一些成败参半的结果。尽管新的 PatchGuard 版本在表面上看起来确实禁用了先前所提出的[2]、用作禁用最初 PatchGuard 发布版的手段的大多数绕过技术，但这些技术中至少有好几种，只需做一些微小的改动或增加一些新的代码，就可以相当轻易地重新启用。此外，绕过 PatchGuard 第二版仍然是可能的，且无需依赖危险的（与具体版本绑定的）构造，例如硬编码的偏移量或对频繁变化的代码做代码指纹识别（code fingerprinting，即通过特征字节模式识别代码）。另外，除了基于禁用 PatchGuard 本身的技术之外，仍然存在若干潜在的绕过机制，它们很有潜力做到与未来的 PatchGuard 新版本「向前兼容」——这是凭借阻止 PatchGuard 甚至察觉到内核已被做了未经授权的篡改来实现的（从而使它们自身与 PatchGuard 系统完整性检查被调用方式上的、任何基于混淆的变化隔离开来）。不过，应当肯定微软的是，PatchGuard 抵抗被调试和分析的能力已经得到了显著提高（至少就某些关键步骤而言是如此，例如开机时的初始化）。

## 3) 值得注意的保护机制（Notable Protection Mechanisms）

PatchGuard 第二版实现了一系列值得介绍的反调试、反分析和混淆机制。PatchGuard 的防御并非全部都在本文中详细涵盖；而那些（例如 PatchGuard 内部数据结构的混淆）在原理上至少与先前的 PatchGuard 发布版相同的机制（且已由 Uninformed 先前关于 PatchGuard 的文章[2]披露过），本文同样不予涵盖。

### 3.1) 初始化期间的反调试代码（Anti-Debug Code During Initialization）

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

## 3.2) Expanded Set of DPC Routines / 扩充的 DPC 例程集合

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

## 3.3) Self-Decrypting and Mutating System Integrity Check Routine / 自解密且可变异的系统完整性检查例程

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

## 3.4) Obfuscation of System Integrity Check Calls via Structured Exception Handling / 通过结构化异常处理混淆系统完整性检查调用

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
