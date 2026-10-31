---
layout: post
title: "PatchGuard Reloaded：PatchGuard 第 3 版简析 中文全译（Uninformed 第 8 卷，Skywing，2007）"
categories: essay
---

> 整理自 Uninformed 第 8 卷（2007 年 9 月）Skywing（Ken Johnson）的《PatchGuard Reloaded: A Brief Analysis of PatchGuard Version 3》全文翻译——第 3 版（随 Windows Server 2008 Beta 3 亮相、经 Windows Update 推向 Vista 与 Server 2003）的保护改进与新一轮绕过、反制讨论（在《PatchGuard 攻防史》中亦作附录四收录）。三部曲顺序：其一《Bypassing PatchGuard on Windows x64》→ 其二《Subverting PatchGuard Version 2》→ 其三（本文，第 3 版）。原文：http://www.uninformed.org/?v=8&a=5&t=txt

---

## PatchGuard Reloaded: A Brief Analysis of PatchGuard Version 3

作者：Skywing（skywing@valhallalegends.com，http://www.nynaeve.net/）
时间：2007 年 9 月

> **译者注**
> - 出处：Uninformed（安全技术刊）Vol 8，原文链接 http://www.uninformed.org/?v=8&a=5&t=txt 。
> - 背景：第 3 版随 Windows Server 2008 Beta 3 亮相，随后经 Windows Update 推向 Windows Vista / Windows Server 2003 x64——也就是说，它不是只随新系统发布，而是被当作一次更新直接送到了已装机用户的系统上。
> - 第 3 版相对第 2 版的变化点（全文主线）：① 随机出现第二个检查上下文（check context，指 PatchGuard 用来核对系统完整性的整套代码＋数据，含自解密检查例程）；② 用汇编写的第一道异常处理先清洗异常记录（过滤异常码、打乱被引用地址），堵住靠“访问违例引用 0xFFFFFFFFFFFFFFFF”识别 PatchGuard 的老办法；③ 新增不经 SEH 的启动路径（定时器 DPC 直接跳进自解密桩），SEH 不再是唯一的单点故障；④ 被借用的 DPC 例程在触发异常前先做随机层数的函数调用，打乱调用栈帧，让“固定上卷一层退出 DPC”失效；⑤ 受保护的内核变量集合扩大，封掉若干“与 PatchGuard 共存式绕过”的办法。文末作者又给了六类新的绕过思路及微软可能的反制。
> - 原文编号照录：原文在 3.4 节之后出现一个编号为“4.5”的“代码补丁支持”小节，之后才进入第 4 章，4.1–4.6 的编号在第 4 章内重新起算；正文里对它的回指也沿用原样，译文不擅自改号。
> - 代码、函数名、数值、反汇编一律保留原文；专有名词首次出现附中文说明。

## 摘要（Abstract）

自从针对内核补丁保护（Kernel Patch Protection，otherwise known as "PatchGuard"，下称 PatchGuard）的绕过或规避技术陆续发表以来，微软一直在持续打磨其补丁保护系统，试图封堵已知的绕过手法。随着 Windows Server 2008 Beta 3 的发布，以及随后经由 Windows Update 把 PatchGuard 全面分发到 Windows Vista / Windows Server 2003，微软把下一代 PatchGuard 推向了公众（“PatchGuard 3”）。与 PatchGuard 此前的历次更新一样，第 3 版代表的是一组渐进式的改动，旨在解决早期版本中被认为存在的弱点和已知的绕过向量。此外，PatchGuard 3 还扩大了受保护、禁止未经授权修改的内核变量集合，消除了若干可被用来在与 PatchGuard 共存（而非禁用它）的前提下规避 PatchGuard 的机制。本文描述 PatchGuard 3 中所做的部分改动。本文还提出若干可用于规避 PatchGuard 防御的新技术，并讨论针对这些技术的反制措施。

## 1) 引言（Introduction）

PatchGuard 是 Windows x64 版本中一项有争议的特性，起于 Windows Server 2003 x64 / Windows XP x64，并延续到 Windows Vista x64 与 Windows Server 2008 x64。PatchGuard 背后的设计目标，是阻止在 x86 版 Windows 上曾经极为普遍的那种对各种内核代码与数据结构的大肆挂钩（hooking，指在别人的执行路径上插入自己的代码）和修改。微软曾表示，内核崩溃的绝大多数是由第三方驱动造成的，作者使用 Windows 的经验也坚定地支持这一判断。由于访问内核内部数据结构、挂钩内核函数通常需要与系统其余部分做错综复杂的同步才能完全安全地完成，尤其是在多处理器机器上，许多执行这类危险任务的第三方驱动历史上都犯过极其严重的错误，往往导致系统稳定性受损或系统安全被攻破。后者在第三方程序挂钩系统调用等函数、却随后未能做充分的参数校验的情形下尤为常见。

微软对这一问题的解决办法，是除劝阻开发者做这类事情之外，还试图用技术手段强行阻止第三方代码对内核内部数据结构和代码做未经授权的修改。然而，由于 Windows 内核（及其配套驱动）的设计方式，内核模式驱动不可能运行在比内核本身更低的有效特权级上。这就给微软“阻止未经授权的内核补丁”这一目标出了难题：内核本身与第三方驱动之间不存在硬件强制的隔离。因此，第三方驱动可以随心所欲地操纵内核代码与数据。

尽管 TPM（可信平台模块）和硬件辅助虚拟化（hypervisor，虚拟机监控器）等新兴技术最终可能提供一种机制，在内核的某些关键部分与和它交互的第三方驱动之间部署一道硬件强制的边界，但就目前所涉技术（硬件与软件能力两方面）的现状而言，这样的做法对今天售出的大多数计算机并不能普遍适用。在缺少完整的、硬件强制的解决方案的情况下，微软转而采用其他办法来劝阻第三方软件做未经授权的内核修改。具体来说，由此产生的内核补丁保护机制（“PatchGuard”）改为建立在高度混淆的代码之上：这段代码虽然运行在与内核本身和第三方驱动相同的有效特权级上，却被设计成能抵御第三方驱动的检测和/或修改。这段代码负责定期检查关键内核代码与数据结构的完整性，一旦检测到这类修改，就把系统打崩。凭借这一事实——以前在 Windows x86 版本上可以肆意给内核打补丁、而试图做同样的操作在 x64 版 Windows 上会导致系统崩溃——第三方驱动实际上已被阻止在客户系统上部署的代码中大规模地做这类修改。

然而，像一切建立在“靠隐晦求安全”（security through obscurity）原则之上的系统一样，PatchGuard 有其固有的弱点。这些弱点可被第三方驱动利用，要么把 PatchGuard 整个禁用，要么在与 PatchGuard 平和共存的同时完全规避它的检查。微软完全清楚 PatchGuard 这一根本路数上的这些缺陷，于是转而周期性地更新 PatchGuard，以封堵已知的公开绕过技术。其净效果是：微软给任何敢违背其意愿、试图规避 PatchGuard 的独立软件供应商（ISV）营造了一种“移动靶”的印象。这有助于表明：任何为阻止或禁用 PatchGuard 而设计的代码，都可能在未来的某个时点失效，比如微软发布一次新的 PatchGuard 更新之时。这导致了一场小规模的军备竞赛：第三方编写规避 PatchGuard 的代码，微软则以开发并部署更新版 PatchGuard 作为回应，使新版不受这些绕过技术的影响。这个循环已经持续了几轮；事实上，PatchGuard 如今已以其第三个迭代版本部署到公众手中。

## 2) 保护方面的改进（Protection Improvements）

与 PatchGuard 2 相比，PatchGuard 3 实现了若干旨在保护 PatchGuard 自身、防止第三方代码将其禁用的渐进改进。对 PatchGuard 自卫逻辑的改动，大多看上去是对此前已发表的、公开已知的绕过技术的直接回应，而不是旨在让 PatchGuard 3 更能抵御分析与攻击的一般性改进。在这个意义上，虽然 PatchGuard 3（相对 PatchGuard 2）的这些改动能有效废掉作者所知的此前发表的大多数绕过机制，但要改动许多此前的攻击机制、使其对 PatchGuard 3 有效，也并非极其困难。PatchGuard 2 中实现的许多保护系统在 PatchGuard 3 中仍以某种形式存在，尽管其中一些已被改动以抵御此前发表的攻击。

本章将描述已做出的若干具体改进。

### 2.1) 多个并发的 PatchGuard 检查上下文（Multiple Concurrent PatchGuard Check Contexts）

在以前的 PatchGuard 版本中，只存在一个 PatchGuard 检查上下文，它会被周期性地用来校验受保护区域的完整性。一些绕过技术依赖“只有一个 PatchGuard 上下文”这一事实：它们在定位到 PatchGuard 之后，禁用掉任何为“当场抓住 PatchGuard”所需要的侵入式内核补丁。PatchGuard 3 对此做了改进：如果 PatchGuard 被启用，它至少创建一个 PatchGuard 上下文，并有一定概率在系统启动时初始化第二个上下文。这一随机化依据处理器时间戳计数器（time stamp counter）做出，正如 PatchGuard 的其他一切随机化所做的那样。两个 PatchGuard 检查上下文——其中包括 PatchGuard 用来检查系统完整性的全部数据（含位于非分页池内存中的自解密检查例程）——在某次具体启动恰好使用两个上下文时，彼此完全独立地运行。

随机创建不止一个检查上下文有几项好处。首先，因为第二个上下文并不总是被创建，绕过技术的测试与开发过程（理论上）被引入了一个不确定因素：一个研究 PatchGuard 3 的人在第一眼时，可能注意不到还有机会创建出不止一个上下文。这可能导致调试过程中浪费时间，因为一些绕过技术会受活动上下文数量的影响。例如，作者为 PatchGuard 2 描述的原始绕过技术 [1]，在第一次出现 PatchGuard 被抓到的积极迹象之后，就有效地把自己关掉了（尽管在那个具体例子中，那些抓 PatchGuard 的钩子本可以事后继续留在原地）。

一个可能受这类方案影响的绕过技术的更好例子，是那些依赖在系统池内存中搜索 PatchGuard 迹象的技术。例如，一种理论上的绕过方案：主动在非分页池中定位 PatchGuard 上下文并以某种方式将其禁用（比如把自解密代码桩改写成展开为一条空操作函数），如果它没有被设计成在一次命中 PatchGuard 之后重新尝试一次池内存扫描，就可能在测试中随机地栽在这一招上。它还消除了这类内存扫描办法所能提供的确信程度：此前，如果一个人有办法在非分页池内存中定位 PatchGuard 上下文，那么他只要得到一次命中，就能确知 PatchGuard 事实上已被禁用（这可以被当作一个信号，表明现在可以安全地执行被 PatchGuard 阻止的动作了）。当多个检查上下文以一定概率运行时，绕过技术再也不可能采用“如果一个 PatchGuard 上下文已被定位并禁用，那么就可以安全地继续”这样的逻辑，因为野外存在的上下文数量可能不是恒定的。

### 2.2) 过滤用于触发 PatchGuard 执行的异常码（Filtering of Exception Codes Used to Trigger PatchGuard Execution）

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

### 2.3) 不用 SEH 执行 PatchGuard（Executing PatchGuard Without SEH）

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

### 2.4) 被借用的 DPC 例程异常路径中随机的调用帧（Randomized Call Frames in Repurposed DPC Routine Exception Paths）

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

## 3) 额外的保护机制（Additional Protection Mechanisms）

PatchGuard 3 与 PatchGuard 2 共有一些此前未曾被描述过的额外保护机制。本章包括对这些保护机制的描述。

### 3.1) 定时器列表混淆（Timer List Obfuscation）

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

### 3.2) PatchGuard 初始化时的反调试代码（Anti-Debugging Code at PatchGuard Initialization Time）

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

### 3.3) KeBugCheckEx 保护（KeBugCheckEx Protection）

为 PatchGuard 1 提出的最早的绕过机制之一，是挂钩负责让系统蓝屏的代码 [4]。从那里出发，攻击者只需恢复系统的正常执行即可。

有若干防御机制来防止这一点。在当前版本的 PatchGuard 中，线程栈的全部内容被填满零，使得恢复负责调用进入 PatchGuard 的那个线程的执行变得困难。此外，PatchGuard 似乎在系统初始化时做了一份 KeBugCheckEx 的拷贝，并在运行时、在通过一次 bug check（蓝屏检查）把系统打崩之前，紧接着把这个版本拷贝覆盖到内核中实际驻留的代码之上。这一点可以这样清楚地看到：在进入 PatchGuard 检查上下文的同时，在调试器中对 KeBugCheckEx 做一处修改，然后在 PatchGuard 上下文的内部函数上设置一个断点，以便在清掉栈和所有寄存器之后调用 KeBugCheckEx。如果这时再去检查 KeBugCheckEx，所做过的任何修改都已经消失了。

此外，PatchGuard 似乎在调用 KeBugCheckEx 之前禁用了 DbgPrint（把它用一条 "ret" 操作码补丁掉）。这可能是（一次失败的）企图：防止在不实际给 KeBugCheckEx 本身打补丁的情况下轻易获得在 KeBugCheckEx 内的执行机会，而给 KeBugCheckEx 本身打补丁则会规避前面提到的对蓝屏代码本身所做修改的保护。（KeBugCheckEx 通常利用 DbgPrintEx 在一次蓝屏发生时向调试器显示一条横幅。然而，因为 PatchGuard 只补丁了 DbgPrint，蓝屏最终发生时到底会发生什么，从结果上看几乎没有影响。）

这段代码可以在 PatchGuard 检查例程中、在调用 KeBugCheckEx 包装函数之前看到。指向 DbgPrint 的指针是在启动时 PatchGuard 初始化期间确定的。

```
mov     rax, [rbx+PATCHGUARD_CONTEXT.DbgPrint]
mov     byte ptr [rax], 0C3h ; '+' ; ret
```

### 3.4) 两阶段代码去混淆（Two-Stage Code Deobfuscation）

PatchGuard 2 与 PatchGuard 3 更有趣的一个防御特性，是它混淆 PatchGuard 检查上下文——即校验系统完整性所必需的代码与数据——的机制。PatchGuard 上下文被混淆成这样：它们在不活动时在内存中是完全随机化的，并且每次该上下文被调用来检查系统完整性时，它的位置和混淆密钥（以及因此其内容）都会改变。

PatchGuard 的解密阶段被分成两个阶段。第一阶段本质上是一个小小的桩，它在内存中保持完全混淆状态，直到即将被调用之前。调用者会覆盖这个桩中被调用的第一条指令，把它改写成一条 "lock xor qword ptr [rcx], rdx" 指令。这个桩的参数是桩自身的地址（在 rcx 中）和解密密钥（在 rdx 中）。这样一来，第一条指令现在就修改了它自己（更重要的是，还修改了随后的那条指令，因为每条指令长 4 字节却修改 8 字节的操作码字节），其结果是变成另一条 xor 指令。这一小串 xor 指令就这样接续下去，直到解码桩的第二阶段被完全解码。

到这个点上，解码桩的第二阶段已是明文，现在可以执行了。第二阶段由一个 xor 运算的循环组成，它从 PatchGuard 上下文的末尾开始、向后移动，直到整个检查例程被解码。此外，在第二阶段解码过程中，每一轮 xor 时解密密钥都会被移位。

第二阶段解码循环完成之后，控制权被转移到现已成为明文的完整性检查例程（所有支撑数据，比如指向内核中关键函数的指针，到这个点上也已经被第二阶段解码循环翻译成了明文）。

本文随附了一个解密 PatchGuard 内存上下文的基础程序的源代码。该程序预期被提供一个文件，其中包含来自内核调试器的 "dq" 日志、覆盖整个内存上下文，外加解密密钥（位于 KDPC + 0x40）和 KDPC->DeferredContext 的值。

### 4.5) 代码补丁支持（Code Patching Support）

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

## 4) 绕过机制与反制措施（Bypass Mechanisms and Countermeasures）

和 PatchGuard 2 一样，若声称 PatchGuard 3 对蓄意执行被 PatchGuard 阻止的操作的第三方驱动代码的攻击刀枪不入，那将是愚蠢的。针对 PatchGuard 3 的新防御有许多可能的攻击（以及微软可能采取的若干可能的反制措施，以便在未来某个 PatchGuard 迭代中破掉所提出的绕过机制）。本文将描述能够击败 PatchGuard 3 的具体攻击。

### 4.1) 混合异常拦截与内存搜索（Hybrid Exception Interception and Memory Searching）

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

### 4.2) 定时器 DPC 分发器与 DPC 分发（Timer DPC Dispatcher and DPC Dispatching）

尽管 PatchGuard 3 在执行系统完整性检查方面消除了 SEH 这一单点故障，定时器与 DPC 分发器仍然继续是有吸引力的目标。一个简单的绕过机制是定位这两个例程中的调用站点（比如按绕过技术 1 所述记录两个分发器例程的地址，然后做反汇编以定位并给所有调用站点打补丁）。在每一个调用站点上，都可以通过寻找一个非规范的 DeferredContext 参数值、或者一个驻留在非分页池内的 DeferredRoutine，来检测出 PatchGuard 正在被执行。（在 PatchGuard 3 中，只实现前一个检查就证明已经足够了，因为为了 PatchGuard 3 实现上的方便，被借用的 DPC 例程与基于非 SEH 的控制路径都使用兼容的调用约定，该约定规定以一个非规范的混淆指针值作为 DeferredContext 参数。）

这一办法的主要缺点，涉及在 x64 上执行任意代码补丁所固有的困难（具体来说，任何代码补丁的尺寸都很大，以及如今相对常见的指令指针相对寻址指令数量众多）。然而，鉴于这是一个影响 x64 上任何代码补丁的困难，作者认为，对于一个有决心的攻击者而言，它不应被视为一个大问题。事实上，Microsoft Research 自家的 Detours 就实现了一个针对 x64 的代码补丁系统 [5]，这说明总的来说在 x64 上做代码补丁绝不是一项应被视为不可逾越的任务。

因为定时器与 DPC 分发器仍然是相对缺乏保护的目标，而且迄今还没有已发布的公开绕过源代码牵涉它们，作者建议为下一个 PatchGuard 版本加强定时器与 DPC 分发器的防御，因为这两个例程继续代表着一个有吸引力的单点故障。增加第三种根本不涉及传统 DPC 的 PatchGuard 执行机制，就是消除与 DPC 分发器相关的逻辑这一单点故障的一个办法的例子。还可以通过以下组合来增加定位 DPC 分发相关代码内全部调用站点的难度：为定时器 DPC 分发器的三个调用站点中的每一个设置不同的静态调用栈差异（即加入虚设的函数调用），再在这静态调用栈差异之上叠加调用栈随机化。光有随机的调用栈并不够，因为通过检查许多次定时器 DPC 请求迭代的调用栈，在样本量足够大时，以相对高的准确度消除那些随机条目（它们不会是所有记录下来的调用栈共有的）会变得容易。采取这种办法的一个缺点是，它本质上会导致把故意难以维护的“面条代码”加入到操作系统的又一个关键区域（定时器 DPC 分发器逻辑）中。作者猜测，维护定时器 DPC 分发器代码的人很可能不会乐意应付这类事情。

### 4.3) 取消 PatchGuard 定时器（Canceling the PatchGuard Timer(s)）

由于 PatchGuard 继续依赖定时器 DPC 来执行它的检查例程，内核定时器 DPC 列表本身继续是一个相对有吸引力的攻击目标。定时器 DPC 列表是通向 PatchGuard 的所有控制路径共有的，因为定时器总是被用作那个周期性调用检查例程的延迟执行组件。

目前横在定时器 DPC 列表面前的障碍有两道。第一道是：改动它依赖于定位非导出的内核变量。虽然也许可以通过指纹识别来做到这一点，但这确实使这一办法比它起初看上去的要稍微不那么诱人。不过，如果做得仔细，指纹识别是可以奏效的，而且有许多短小的函数以相当可预测的方式引用定时器列表（例如 KeCancelTimer）。找到 DPC 列表的另一种可能办法是，创建并设置一个定时器（从而把它插入定时器列表），然后扫描 ntoskrnl 中一个非分页的未初始化数据节里的每一个 8 字节对齐的值，把每一个有效地址当作一个链表来处理，并在前若干个条目中搜索那个刚刚被链入列表的定时器。虽然这是一个相当难看的、基于蛮力的办法（而且也不完全安全，因为要严重依赖 MmIsAddressValid），但扫描 ntoskrnl 的数据节是就找到定时器列表而言、指纹识别之外的一个替代方案。

这一办法的第二个问题是：从 PatchGuard 2 开始，定时器列表本身被混淆，使得一个 KTIMER 对象与其对应的 KDPC 之间的链接被混淆。这一混淆机制，如前面所描述的（回指 1），围绕另外两个非导出的内核变量（KiWaitAlways、KiWaitNever）打转，它们充当混淆密钥。定位这些变量很可能需要对那些需要把一个定时器插入定时器列表的（可能是导出的）例程——比如 KeSetTimerEx——做代码分析或指纹识别。

另一个完全免于指纹识别和/或蛮力办法的替代方案——代价是需要增加复杂性（一个用户模式组件）——是把任何会触犯 PatchGuard 的驱动代码的激活推迟到用户模式的 Win32 被启动之后。然后可以创建一个用户模式服务，它会下载正在使用的内核二进制的符号，取回 KiTimerTableListHead（定时器列表）、KiWaitNever 与 KiWaitAlways 的地址，并经由任何标准的用户模式到内核模式通信机制（比如 DeviceIoControl）把这些地址传递给驱动。因为内核调试器依赖于能通过 PDB 符号按名字取回这些变量、供 !kdexts.timer 扩展使用，微软将无法通过移除或改名这些混淆密钥变量来封堵这一办法而不损害现有调试器二进制的功能。

一旦定位了 KiTimerTableListHead、KiWaitAlways 与 KiWaitNever，遍历定时器列表、去混淆每个相应定时器对象上的 DPC 链接、并从那里逐个检查每个定时器看它是否具有是一个 PatchGuard 定时器的特征（可能包括诸如一个在未来数分钟之后的定时间隔、一个非规范的 DeferredContext 值，以及可能一个指向非分页池的 DPC 例程指针等属性），就是一件相当简单的事情了（尽管在没有同步的情况下也许不安全，不过人总可以采取“大锤”办法：停掉除一个 CPU 之外的所有 CPU，并把 IRQL 抬到 HIGH_LEVEL）。在定位了所涉定时器之后，它可以被轻易地禁用（要么把它从列表中整个移除，比如经由 KeCancelTimer，要么把它的 DPC 例程改写成指向一个空函数，该函数什么操作都不做、简单地返回）。

因为微软在调试器中有依赖于能使用这些变量来访问定时器列表的功能，就当前的操作系统版本而言，他们不幸把自己逼到了某种墙角，因为微软的一般政策是：现有的调试器二进制在某个已发布的操作系统版本被打了热修复或服务包之后，仍应继续正常工作。抵消这一办法的最佳方式，是让 PatchGuard DPC 在内存中相对于任何给定时刻列表中其他全部定时器 DPC 对象更难被挑出来，并为 PatchGuard 创建不那么严重依赖定时器列表的额外启动向量。还存在许多其他执行代码、又不会引起一个不知道自己在看什么的人注意的办法，其中很多都比一个定时器不那么显眼。

### 4.4) 页表交换（Page-Table Swap）

和 Windows 内核中的一切内存访问一样，PatchGuard 的系统完整性检查例程在启用了分页的保护模式下运作。理论上也许可以利用这一事实来向 PatchGuard 隐藏内核补丁。

所提出的绕过技术将涉及：给定时器与 DPC 分发器中的第一条指令打补丁，使其分支到第三方代码。当 DPC 与定时器 DPC 即将被考虑执行时——这由对两个分发器例程之一的一次调用发出信号——一份页表的影子副本被创建出来。这份影子副本被配置成与当前进程的正常页表完全相同，只是任何已被打补丁的内核代码页的页表项被改动成指向代表原始状态的物理页。栈上 DPC 或定时器 DPC 分发器的返回地址被替换成一个指向驱动提供的代码的指针，cr3 被重新配置成指向影子页表。然后，执行被转移回定时器或 DPC 分发器的入口点（由于页表交换，它不再显现任何被打补丁的迹象），DPC 被分发。当分发器完成它的工作——如果 PatchGuard 要在任何一批定时器 DPC 中被执行，这工作就包括调用 PatchGuard——之后，控制权被交回驱动提供的代码，后者随后把自影子副本被做出来之后发生的任何页表修改镜像回进程的实际页表，并把 cr3 恢复成它的原始值。然后控制权被转移到分发器的正常返回点。

这一办法根本不涉及禁用 PatchGuard。相反，它描述了一种可能与它“和平共存”的方式，前提是只做内核代码补丁。（数据页则预计会被一个 DPC 修改，作者认为以这种方式保护它们免于 PatchGuard 要不切实际得多。）因为 DPC 与定时器 DPC 分发器逻辑在 IRQL DISPATCH_LEVEL 上执行，当前线程的线程上下文切换被禁用，这使得 cr3 交换办法相对可行。

因为这一办法不涉及直接攻击 PatchGuard，它自动地规避了当前版本 PatchGuard 中内置的种种防御机制，这使它成为一条相当有吸引力的潜在攻击路线。然而，它也有一些缺点。其中就包括，在多处理器环境中完成一次页表交换所需的同步，很可能复杂且难以安全地复制——如果一个人允许 DPC 例程执行改变 PTE 的操作的话。此外，这一办法还会带来性能上的影响，因为它需要在系统整个存续期间持续运行在一条影响相对较大的路径（DPC 分发）上。在每一批 DPC 上都使 TLB 失效所带来的性能含义在某些情况下可能成问题（交换 cr3 会自动清空 TLB）。

这一办法的另一个缺点是：凭借所有 DPC（以及潜在地所有设备硬件中断）都可能运行在页表影子副本之下的这一事实，大多数与硬件相关的事件将不受到被这一机制隐藏的内核代码补丁的影响。这是否成问题，取决于期望的内核补丁目标是什么。

微软可以通过在 PatchGuard 初始化时做一份描述内核的全部 PTE 的拷贝、然后从 PatchGuard 检查例程内部校验全部内核代码 PTE，来抵消这一办法。此外，如果微软可以假设 PatchGuard 总是在系统进程中执行，另一个办法是要求 cr3 取一个已知的值。

### 4.5) DPC 异常处理程序补丁（DPC Exception Handler Patching）

PatchGuard 3 相对 PatchGuard 2 引入的改动之一，是调用解密过程第一阶段所用协议的一处小小改变。具体来说，一个加密的 PatchGuard 上下文的全部调用者，现在都包含一个静态的 8 字节字符串（由指令操作码组成），它与 PatchGuard 上下文开头的一个值做 xor，构成初始的解密密钥。

做出这一改变（相对原有行为）的原因，在作者看来并不清楚，但它不幸成了禁用 PatchGuard 的一个容易的目标，因为这个字符串本身（0x8513148113148F0）在内核代码中相当独特，不太可能出现在 PatchGuard 之外。此外，所有 PatchGuard 调用者——包括全部十个被借用的 DPC 例程异常处理程序，以及非分页池内存中的 DPC 适配器（如果被使用的话）——都毫无混淆地引用这个字符串。这就呈现出一种极其容易的、基于指纹的禁用 PatchGuard 的办法。通过扫描非分页池空间以及内核代码区域来找这个字符串，定位每一条负责调用 PatchGuard 检查上下文的代码路径中间的一条指令，是微不足道地容易。

在定位了引用这个 8 字节字符串的指令之后，给它们打补丁、使之执行一次从异常处理逻辑中展开出去（或者在非分页池内存代码的情形下，直接返回），是微不足道的。这样一种攻击阻止 PatchGuard 根本启动，而且还具有一个优势：与本文所概述的许多其他绕过技术相比，它所需要的额外支撑逻辑最少。

微软废掉这一技术将是微不足道的。作者的建议是去掉每个 PatchGuard 调用者中引用的那个静态 8 字节字符串。具有讽刺意味的是，PatchGuard 2 必然有一个类似的 4 字节字符串（它在 PatchGuard 3 中也仍然被使用），它代表第一阶段解密桩的初始指令。然而，与 PatchGuard 3 不同，PatchGuard 2 小心地混淆了把操作码字符串写出到 PatchGuard 上下文的过程，这样一来，人们就不能简单地用一个单一的、全覆盖的指纹来覆盖所有情形。PatchGuard 3 中所做的这一改变，可以说是把这项工作完全付诸东流了，而且它还多了一个好处：作为一个可供指纹识别的值，它是前者的两倍大。

### 4.6) 系统调用 MSR 交换（System Call MSR Swap）

作为前面描述的技术（回指 4）的一个变体，理论上应该有可能在 DPC 或定时器 DPC 分发的持续期间，在线地交换系统调用 MSR（或者事实上，还有其他若干个受 PatchGuard 保护的处理器控制寄存器），并在分发器返回之后恢复那些“被污染的”值。系统调用 MSR 负责指定系统调用分发器的地址，因此对那些想要做系统调用挂钩的第三方来说，是一个有吸引力的目标。

这一技术将应用与前面在 cr3 交换技术中描述的相同的基本概念。如果系统调用是唯一打算挂钩的目标，那么对单处理器系统来说，cr3 交换可以被消除，因为它是不必要的（因为可以安全地在一次 DPC 分发器调用之前和之后做并恢复对实际底层物理页的改动，利用栈上的返回地址作为一种办法，在不让操作码在分发器调用之间一直留在内核中被打补丁的情况下，返回到那个被改动的位置）。对多处理器系统而言，需要开发某种机制，允许 MSR 交换在 DPC 分发器之间进行，同时防止代码补丁对第二个处理器变得可见。这一点之所以必要，是因为随着 PatchGuard 3 增加了在系统启动时有一定概率初始化第二个检查上下文，可能会有不止一个 PatchGuard 上下文同时执行。

为了封堵这样一种技术，微软最可能受到良好服务的做法是：让人难以定位为了在各次 PatchGuard 检查之间维持一个未打补丁系统的假象所必需补丁的全部区域。做到这一点的主要方式，是为 PatchGuard 创建其他与 DPC 无关的替代启动向量，并且最好不涉及从第三方视角看容易拦截的导出 API。

## 5) 结论（Conclusion）

尽管 PatchGuard 3 确实对许多此前已披露的绕过技术带来了若干有针对性的反击，但第 3 版和它的前任们一样，远非免疫于被完全禁用或被简单地共存。未来 PatchGuard 的修订版本很可能继续易受多种绕过技术的影响，不过，封堵许多公开披露的绕过向量无疑是在微软力所能及的范围之内的。作者预计，直到 PatchGuard 能以硬件支持来实现——比如经由可信启动（TPM）与一个常驻 hypervisor 的组合——未来的修订版本仍将继续易受有决心的个人攻击。

另一方面，就阻止 Windows x64 上违反 PatchGuard 的驱动被大规模采用而言，微软在 PatchGuard 上的努力迄今看上去已经得到了回报。换句话说，可以提出这样一个论点：微软不需要在 PatchGuard 上做到完美，只需要做到“足够好”，好到让供应商对试图出货绕过它的产品心生怯意。至于这一点在未来是否仍将成立，只有时间能告诉我们。

## 参考文献（References）

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
