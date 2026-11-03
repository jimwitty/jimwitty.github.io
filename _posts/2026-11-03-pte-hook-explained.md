---
layout: post
title: "PTE Hook 讲透（通俗版）：只给一个进程看的假页表"
categories: essay
---

> 通俗化改写自先知社区《PTE Hook：一种利用页表重映射攻击实现的内核函数 Hook》（https://xz.aliyun.com/news/18999），把原文偏专业的分页与页表知识换成生活类比重新讲一遍，技术结论以原文为准，个别编注有标明。文末附原文全文转载，便于对照阅读。

## 先说这篇在解决什么问题

**Hook（钩子）**，就是在某个函数被调用时"半路截胡"：先跑一段自己的代码，再决定是放行、改参数、还是直接拦下。给 Windows 内核函数挂钩子，最朴素的办法叫 **inline hook（内联钩子）**：把目标函数开头的若干字节改成一条跳转指令，跳到自己的函数里；自己的函数干完活，再跳回原函数继续执行。

朴素办法有个要命的毛病：**它改的是全系统共享的那一份内核代码**。内核函数在内存里只有一份，所有进程共用——你改了它，就等于全系统每一个进程调用这个函数时都走你的钩子，影响面失控。更麻烦的是，Windows 有个巡逻的"保安"叫 **PatchGuard（内核补丁保护，常缩写 PG）**：它定期巡查内核关键代码和结构有没有被篡改，发现被改就直接蓝屏。所以在新版 Windows 上，全局改内核代码这条路基本是死路。

**PTE Hook 的想法一句话**：原函数一个字节都不动，而是**给目标进程换一条"查地址的路线"，让它查到的是一份被改过的副本**。其他进程走的还是原路线、看到的还是原函数；只有被选中的那一个进程，会撞上钩子。PatchGuard 巡查原版代码，看到的是一切正常。这篇剩下的内容，就是把"查地址的路线"——也就是页表——讲清楚，再看这条假路线是怎么搭出来的。

## 前置一：虚拟地址是怎么找到物理内存的

程序里用的地址都是**虚拟地址**——一套"门牌号"，不是内存条上的真实位置。CPU 真正访问内存前，必须把门牌号翻译成真实位置（**物理地址**）。负责翻译的对照表，就是**页表**。

打个比方：页表就像一套四级分级目录。x64 下一个虚拟地址有 48 位有效位，被切成 **9-9-9-9-12** 五段：

- 前四段各 9 位，是四级目录的**序号**：一级目录叫 **PML4**，往下依次是 **PDPT、PD、PT**（名字不用背，理解成 省→市→区→街道 四级通讯录即可）；
- 每级目录有 2⁹=512 条条目，每条条目写着"下一级目录存放在哪个物理页"，最后一级 PT 的条目（就叫 **PTE，页表项**）写着"目标数据所在的物理页"；
- 最后一段 12 位是**页内偏移**：一页有 2¹²=4096 字节（4KB），数据在这一页里的第几个字节，靠这段定位。

走一遍数字例子。虚拟地址 `0x00002A14 63C857B8`，按 9-9-9-9-12 切开，四级序号依次是 **84、81、286、133**，页内偏移是 **0x7B8**。翻译过程就是：拿第 84 号条目找到第二级目录，再拿第 81 号找到第三级，再拿 286 号找到第四级，再拿 133 号找到目标物理页，最后加上 0x7B8 的页内偏移，落点确定。

那 CPU 从哪知道第一级目录（PML4）放在哪？有个专用寄存器 **CR3**，里面存着**当前进程的 PML4 的物理地址**。每切换一个进程，CR3 就换成那个进程的 PML4——相当于换了一整套通讯录。这正是"每个进程看到的地址空间互不相同"的实现方式。

一个关键事实，先埋在这里：每个进程的通讯录只有**低半部分**是自家的（用户空间，各不相同）；**高半部分（内核空间）全系统共用同一套下级目录**。为什么？内核只有一份，所有进程进内核用的都是同一套映射，抄来抄去没意义，直接共享。记住这条——上一节说的"改一个内核函数、全局都遭殃"，根源就在这里：大家共用同一套内核目录，你顺着目录改到最后那页数据，所有人看到的都变了。

还有一个配角要登场：四级查表要访存四次，很慢，所以 CPU 里有个小抄本叫 **TLB（快表，地址翻译缓存）**，把最近翻译过的"虚拟地址→物理地址"记下来，下次直接抄答案。后面它会惹出一个不大不小的麻烦。

## 前置二：页表自映射——通讯录里有一页写着通讯录自己

Windows 在 PML4 里留了**一条指向 PML4 自己的条目**，叫**页表自映射（递归映射）**：在第 S 号条目里，填上 PML4 自己的物理地址——相当于通讯录里有一页，内容就是这本通讯录本身。

这看似浪费一条条目，妙处在于：**把第几级目录的序号重复填进地址的不同段位，就能"翻到目录本身"**。举例：虚拟地址的四段序号全填 S，CPU 一路查下去，每一级都回到 PML4 自己，最后落点就是 PML4 所在的物理页——于是这个构造出来的虚拟地址，就能当普通内存一样直接读写 PML4 的条目。同理变通段位组合，就能摸到任意一级目录、乃至**任意一个虚拟地址对应的那个 PTE**——有个简洁公式：先定位所有 PTE 排布的起始位置（PTE 基址），任意地址的 PTE 位置 = 基址 +（该地址的页号 × 8），因为每页恰好对应一个 8 字节的 PTE。

这件事的意义：**不需要知道任何物理地址，光靠拼虚拟地址，就能直接改写页表条目**。改 PTE，就等于改"这个门牌号最终指向哪块物理内存"——这是 PTE Hook 的扳手。

剩下一个小问题：出于安全，Windows 每次开机都会把自映射的位置 S 和页表基址**随机化**（基址随机化，防止攻击者按固定地址下手）。原文给出一个定位办法（文中称来自腾讯的方法）：CR3 里存着 PML4 的物理地址，而自映射条目的内容恰好就是这个值——把 CR3 对应的那一页映射出来，逐个比对 512 个条目，**哪个条目的值等于 CR3，它就是自映射条目**，位置和基址一并解出。

## 主菜：给一个进程搭一条假路线

材料齐了。PTE Hook 对某个目标进程、某个想挂钩的内核函数，分五步：

**第一步，复制函数所在页。** 把目标函数所在的那一页（4KB）内存原样复制一份，得到"副本页"。原页不动。

**第二步，伪造整条目录链。** 把目标进程的 PDPT、PD、PT 各**整页复制一份**（复制整页是为了只改一条、其余 511 条保持原样，不影响这个区间里的其他函数）。然后像接力一样把副本链起来：假 PDPT 里的那一条指向假 PD，假 PD 里的那一条指向假 PT，假 PT 里对应目标函数的那一条 PTE，**不指向原物理页，改指向第一步的副本页**。

**第三步，只换目标进程的门牌号总目录。** 把目标进程 PML4 里对应的那一条（PML4E），改成指向假 PDPT。这一步是全篇的题眼：PML4 是**每进程一份**的，改它只影响这一个进程。从此这个进程访问目标函数时，一路走假链，落点是副本页；其他进程 PML4 没变，走真链，落点是原页。同一个虚拟地址，两种落点——**分流完成**。

**第四步，在副本页里做普通的 inline hook。** 副本页随便改，不再有全局顾虑。原文用的跳转写法值得一提：x64 下一条 `FF 25` 开头的 14 字节指令可以跳到任意 64 位地址（绝对跳转）；被覆盖掉的原函数开头指令（要凑够至少 14 字节，用反汇编器逐条数长度）搬到一块"跳板"内存里保存，跳板末尾再用一个小花招跳回原函数续执行——把 64 位返回地址拆成两个 32 位、先后写进栈顶，再用一条 `ret` 指令"返回"过去。自己的钩子函数处理完，调用跳板，就等于调用了原函数。写副本页时还有个小麻烦：代码页通常只读，原文用 **MDL**（内存描述符表，一种把物理页重新映射成可写窗口的内核机制）绕开只读限制写入。

**第五步，收尾防串味——关掉 G 位、刷 TLB。** 这是原文专门用一段讲的坑。前面说过 TLB 是 CPU 的小抄；小抄里的条目有个 **G 位（Global，全局位）**：标了 G 的翻译"全进程通用、切换进程时不清空"，这是给内核这种全系统共享的映射省事的。而危险恰恰在此：如果假链的翻译带着 G 位进了 TLB，别的进程切换进来时可能直接抄到这条小抄，**撞上本应只给目标进程的钩子**，隔离当场失效。所以要把涉及的 PTE/PDE 的 G 位清零，强制它们随进程切换被刷掉；换完链还要用 `invlpg` 指令把旧翻译从 TLB 里作废。原文换表时还关了中断，防止换到一半被打断、链处于半真半假状态。

## 一个绕不开的麻烦：大页与页表分割

上面默认函数住在 4KB 小页里，但内核为了省查表开销，常把整片代码用 **2MB 大页**映射——大页在 PD 这一级就直接指向 2MB 的物理块，**跳过了 PT 这一级**，没有 PTE 可改，假链无处下手。

原文的解法叫**页表分割**：新建一张 PT 表，里面 512 条 PTE 依次指向这块 2MB 内存的 512 个 4KB 切片（物理上严丝合缝地接续），再把 PD 里那条大页条目换成指向这张 PT 的普通条目——大页就被"化整为零"成了 512 个小页，内容和落点分毫不差，但从此可以在小页粒度上单独替换其中一页。为什么不干脆伪造一整块新的 2MB？原文给了现实理由：伪造大页要申请 2MB **连续**物理内存，而 Windows 的物理内存越用越碎，开机越久越难凑出这么大一整块；申请一页 4KB 则随时都有。这一步做完，再接回主菜的第二步即可。

## 效果：PatchGuard 为什么抓不到

原文的实测（作者自述）：在 Win10 1903 上对 explorer.exe 进程上下文里的 NtCreateFile 挂钩，钩子检查文件名、拦截对 MyHook.txt 的创建；另在 Win10 21H2、Win11 21H2 上挂了约 6 小时，未出现 PatchGuard 蓝屏。

原理上说得通：PatchGuard 巡查的是**全局共享的那份原版内核代码和关键结构**，而原版一字节都没被改；被改的只有单个进程页表里的一条分流路线，不在它的巡查单上。其他进程、以及任何直接检查原函数字节的手段，看到的都是干净的原函数。

泼一盆冷水（本文编注）：这不等于"无敌"。一来页表本身被改这件事，在开了 **HVCI/VBS**（基于虚拟化的安全， hypervisor 在更底层盯着内存）的环境里，另有眼睛能看见；二来这项技术本身年头已久（作者结语也坦言"已经是比较老的方法了"），主流 EDR 对这类页表篡改早有针对性检测。它的价值——作者说得实在——在于**把 x64 分页机制、自映射、TLB 这些内核基本功一次练透**，是极好的入门实战题。

## 一表收尾

| | 普通 inline hook | PTE Hook |
|---|---|---|
| 改的是什么 | 原函数的代码字节 | 目标进程的页表分流路线 + 一份副本页里的代码 |
| 影响范围 | 全系统所有进程 | 只有选定的那一个进程 |
| 原函数字节 | 被改写 | 一字节不动 |
| PatchGuard | 巡查必中，蓝屏 | 原版完好，其巡查单上没有这一项（实测 6 小时未触发） |
| 代价 | 简单直接 | 要搭整条假页表链、处理大页分割、G 位与 TLB，复杂度高得多 |
| 本质 | 改内容 | 改"查号路线"，让同名地址在不同进程落到不同内容 |

最后再用一句话收束全篇：**页表就是地址世界的路由表；谁掌握了路由，谁就能决定一个进程眼中的"事实"。** PTE Hook 不过是把这句用到了极致——不改事实本身，只改某一个人看事实的路。


---

## 附：原文全文

> 以下为先知社区原文《PTE Hook：一种利用页表重映射攻击实现的内核函数 Hook》全文转载，便于对照阅读；版权归原作者所有。原文地址：https://xz.aliyun.com/news/18999

### 什么是PTE Hook

常规的inlineHook思路是直接修改目标函数的代码，使其先执行我们自己的函数，再跳转回来执行原函数。这种Hook是全局的，即Windows中每一个进程一旦调用被Hook的函数，就会受到我们的影响，也很容易被PatchGuard检查到。

因此提出一种新的Hook思路，**隔离具体进程的四级页表，使我们的Hook不影响全局**。当然具体的Hook方法依然是inlineHook

### 理论知识

#### CPU访问物理地址

首先解释一下x64架构下，CPU访问一个虚拟地址（也称线性地址）时的流程。Windows使用了四级页表结构，分别是PML4 (Page Map Level 4)、PDPT (Page Directory Pointer Table)、PD (Page Directory)、PT (Page Table)。我们知道一个虚拟地址共64位，其中高16位是不使用的，因此总共只有48位，这48位按照9-9-9-9-12的结构拆分，前四个9表示四级页表的索引，最后一个12表示一个物理页（通常是4K）的页内偏移，这样CPU就找到了一个虚拟地址对应的物理地址。下面演示一个虚拟地址是如何拆分的。

```
比如这样一个虚拟地址 : 0xfffff8037888e000
    高16位不使用，可以看到全部被置为了1，因此有效地址为 ： 0xf8037888e000
    转为二进制： 1111 1000 0000 0011 0111 1000 1000 1000 1110 0000 0000 0000
    第一个9位: 1 1111 0000 -> 0x1f0  PML4 Index
    第二个9位: 0 0000 1101 -> 0xd   PDPT Index
    第三个9位: 1 1100 0100 -> 0x1c4 PT Index
    第四个9位: 0 1000 1110 -> 0x8e  PD Index
    最后12位页内偏移: 00 0000 0000 -> 0    Offset
```

CR3寄存器是x64架构下一个重要的系统寄存器，其中存储的值是当前进程PML4表的物理基地址，之前已经计算出Index，因此通过CR3拿到基地址后，可以逐层的索引到物理地址。

#### 页表自映射

根据上面讲的，我们明白CPU是这样找到物理地址的：PML4->PDPT->PD->PT->Physical Page，共有4次访问内存。由于CPU寻址是一个高频操作，为了简化这一流程，Windows引入了一个很精妙的机制：**页表自映射**。

页表自映射，即在PML4这张表的其中一项中，存储PML4表基地址。比如假设在索引0x100处存储：

![](https://xzfile.aliyuncs.com/media/upload/picture/20260326215842-e63884b5-291b-1.png)

因此，在访问PML4索引0x100处，其实就是在访问PML4自身，所以被称为页表自映射。

你可能会想，这么做白白浪费了8字节空间，为什么要设置一个这样的页表项，他的精妙之处是这样的：

我们先从PML4入手，我们知道一个48位有效地址的高9位是PML4的索引，这个索引对应的页表项存储的本应该是下一级页表，即PDPT的物理基地址，但是由于引入了页表自映射，因此就必然有一个索引对应的页表项存储的是PML4自身的基地址，不妨将这一索引设为S（在上图中这个索引是0x100）。

我们先采用这样的格式表示一个48位的虚拟地址 (A)(B)(C)(D)(Offset) ，其中A是PML4的索引

当 A == B == C == D == S && Offset == 0 时，CPU访问这个虚拟地址时会出现这样的寻址情况：

PML4 Index == S，指向PML4表物理基地址，PDPT Index == S ，又指向PML4表物理基地址，以此类推，PD和PT也都指向PML4表物理基地址。因此 (S)(S)(S)(S)(000000000000) 对应的物理地址就是PML4表的**物理基地址**，不妨把这个虚拟地址称作**PML4的表虚拟基地址**。用数学公式将之表达：

```
PML4_VirtualBase = (S << 39) | (S << 30) | (S << 21) | (S << 12)
    //S << 39 就是将一串48位的二进制数的高9位设为S，或运算就是将每次设置的值连起来放在一个数上
    //假设 S << 39 == 0xF6E0000000000000 S << 30 == 0x0000074000000000
    //那么 (S << 39) | (S << 30) == 0xF6E0074000000000
```

根据上面的理论，显然当 A == B == C == S && D == PDPT_Index && Offset == 0 时，也就是说其中一位指向了PDPT的基地址，可以得到PDPT的表虚拟基地址

PDPT_VirtualBase = (S << 39) | (S << 30) | (S << 21) | (PDPT_Index << 12)

以此类推，我们就可以得到计算最低一级的页表PT的表虚拟基地址的计算公式，这里顺便展示PD的计算公式：

```
PT_VirtualBase = (S << 39) | (PDPT_Index << 30) | (PD_Index << 21) | (PT_Index << 12)
    PD_VirtualBase = (S << 39) | (S << 30) | (PDPT_Index) | (PD_Index << 12)
```

以上是利用页表自映射机制定位到PT表基地址，实际上也可以利用这个原理定位到任意一个VA（虚拟地址）对应的PTE（Page Table Entry，即PT条目，也即PT表中的一项）。

显然以下公式成立：

```
PTE_VirtualAddress = (S << 39) | (PML4_Index << 30) | (PDPT_Index << 21) | (PD_Index << 12) | (PT_Index << 3)
    /*
    理解最后的(PT_Index << 3)的这一项，因为每个PTE占8字节，左移3位即乘8，也理解为i*8作为偏移来找到具体的
    的PTE，实际上就是通过自映射机制将PT表当作了原来的物理页面进行查找
    */
```

通过我们构造出的这个虚拟地址，可以直接读写PTE而不需要知道其物理地址。

最后再将一个很简洁的通过PML4的表虚拟基地址得到其他三级页表虚拟基地址的公式，我们知道：

PML4_VirtualBase = (S << 39) | (S << 30) | (S << 21) | (S << 12)

如果将后两个S置为0，即不考虑偏移且往后找一级页表，就可以得到PDPT的表虚拟基地址

PDPT_VirtualBase = (S << 39) | (S << 30) | (0 << 21) | (0 << 12)

数学上，上面这个式子实际上等价于PML4的虚拟基地址的低21位置0，即

PDPT_VirtualBase = (PML4_VirtualBase >> 21) << 21

同理：

```
PD_VirtualBase = (PML4_VirtualBase >> 30) << 30
    PT_VirtualBase = (PML4_VirtualBase >> 39) << 39
```

另外，由于上面计算VA对应的PTE公式过于复杂，是主动构造了9-9-9-9-12的虚拟地址结构，还有一个通过基地址+偏移的方法来计算：

```
PTE_VirtualAddress = PTE_Base + (VA >> 12) << 3
    /*
    VA >> 12 其实就是 VA / 4096，我们知道一个页就是4096K，所以VA >> 12 就是虚拟页号，每一页对应一个PTE，
    一个PTE是8字节，那么上式就变成了：
    */
    PTE_VirtualAddress = PTE_Base + Offset * 8
```

由于Windows开启了基址随机化，页表的虚拟基地址每次开机都不一样，因此需要一个巧妙的方法定位页表基地址，这里给出鹅厂的方法：利用页表自映射定位。由于存在页表自映射这一机制，因此，在PML4表的512个地址中，必然有一个存放着PML4的表物理地址，即CR3的值。所以可以通过映射CR3物理地址的虚拟地址，遍历这个地址页面的512个地址，哪个地址等于CR3的值，哪个地址就是PML4的表虚拟基地址。考虑以下算法：

```c
ULONG64 GetPml4Base()
{
    PHYSICAL_ADDRESS pCr3 = { 0 };
    pCr3.QuadPart = __readcr3();
    PULONG64 pCmpArr = (PULONG64)MmGetVirtualForPhysical(pCr3);
    int count = 0;
    /*
    *pCmpArr（当前条目，即PML4E的值）表示指向下一级页表的物理地址
    &0xFFFFFFFFF000 获取Page_Frame_Number 页帧号
    */
    while ((*pCmpArr & 0xFFFFFFFFF000) != pCr3.QuadPart)
    {
        if (++count >= 512)
        {
            return -1;
        }
        pCmpArr++;
    }
    return (ULONG64)pCmpArr & 0xFFFFFFFFFFFFF000;//忽略后12位标志位
}
```

理论知识就到这里了。

### PTE Hook原理

首先思考一个问题，为什么常规Hook只是修改了一个进程的内核函数，却导致全局的内核函数被修改而被PG检查到。这是因为，用户态下的进程对应的PML4表的高256项都是相同的，指向了共享的内核PDPT，一旦我们修改任意一项PTE，就会产生连锁的影响PT->PD->PDPT->PML4，又由于PML4是共享的，所以全局的函数都被修改了。

因此我们可以先替换掉一项PML4E，这一项PML4E被我们替换后，指向一个伪造的PDPT表，再指向一个伪造的PD表，一个伪造的PT表，最后指向我们Hook的函数。这样被修改的函数只局限于这一个进程，在一定程度上可以规避PG。

此外，还需要考虑大小页的问题。小页指的就是一般的4K大小的页，大页指的是2M的页，是操作系统为了提高访问内存性能而开发的。根据上面的伪造替换规则，一般来说会想到：**大页换大页，小页换小页**。但是由于页表的物理内存需要是连续的，而Windows的物理内存机制是碎片化的，开机越久越难申请到2M的连续内存，所以考虑使用页表分割的方法，即将一个2M的大页分成512个小页。在如下的代码中，我们只申请了一页连续的内存，用来存放指向512个小页的PTE，每个都用来指向原大页的不同4K部分。

### 实战

页表分割：

```c
bool splitLargePages(pde_64* in_pde, pde_64* out_pde)
{
    PHYSICAL_ADDRESS MaxAddrPa{ 0 }, LowAddrPa{ 0 };
    MaxAddrPa.QuadPart = MAXULONG64;
    LowAddrPa.QuadPart = 0;
    pt_entry_64* Pt;
    auto start_pfn = in_pde->page_frame_number;
    Pt = (pt_entry_64*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);//默认对齐
    if (!Pt) {
        DbgPrintEx(77, 0, "failed to alloc contiguous for new pt.\r\n");
        return false;
    }
    for (int i = 0; i < 512; i++) {
        //分割成小页，构建Pt
        Pt[i].flags = in_pde->flags;
        Pt[i].large_page = 0;
        Pt[i].global = 0;
        Pt[i].page_frame_number = start_pfn + i;
    }
    out_pde->flags = in_pde->flags;
    out_pde->large_page = 0;
    out_pde->page_frame_number = va_to_pa(Pt) / PAGE_SIZE;
    return true;
}
```

下面是页表伪造部分的代码：

```c
typedef struct _PTE_TABLE {
    void* LineAddress;
    pte_64* PteAddress;
    pde_64* PdeAddress;
    pdpte_64* PdpteAddress;
    pml4e_64* Pml4eAddress;
}PTE_TABLE, * PPTE_TABLE;
```

```c
bool isolationPageTable(cr3 cr3_reg, void* replaceAlignAddr, pde_64* splitPDE)
{
    //均指向4kb内存
    uint64_t* VaPt, * Va4kb, * VaPdt, * VaPdpt, * VaPml4t;
    PTE_TABLE Table{ 0 };
    PHYSICAL_ADDRESS MaxAddrPa{ 0 }, LowAddrPa{ 0 };
    MaxAddrPa.QuadPart = MAXULONG64;
    LowAddrPa.QuadPart = 0;
    //这里申请伪造页表的内存
    VaPt = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
    Va4kb = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
    VaPdt = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
    VaPdpt = (uint64_t*)MmAllocateContiguousMemorySpecifyCache(PAGE_SIZE, LowAddrPa, MaxAddrPa, LowAddrPa, MmCached);
    VaPml4t = (uint64_t*)pa_to_va(cr3_reg.address_of_page_directory * PAGE_SIZE);
    if (!VaPt || !Va4kb || !VaPdt || !VaPdpt) {
        DbgPrintEx(77, 0, "failed to alloc page table entry.\r\n");
        return false;
    }
    Table.LineAddress = replaceAlignAddr;
    getPagesTable(Table);//这个函数是利用理论知识讲的公式获取任意VA对应的PTE、PDPTE、PDE、PXE
    //获取索引
    UINT64 pml4eindex = ((uint64_t)replaceAlignAddr & 0x0000FF8000000000) >> 39;
    UINT64 pdpteindex = ((uint64_t)replaceAlignAddr & 0x0000007FC0000000) >> 30;
    UINT64 pdeindex = ((uint64_t)replaceAlignAddr & 0x000000003FE00000) >> 21;
    UINT64 pteindex = ((uint64_t)replaceAlignAddr & 0x00000000001FF000) >> 12;
    //判断是否为大页，因为大页的PT是没有值的，所以要将VaPt指向分割成小页后的PT表，就是上面代码展示的
    if (Table.PdeAddress->large_page) {
        MmFreeContiguousMemorySpecifyCache(VaPt, PAGE_SIZE, MmCached);
        VaPt = (uint64_t*)pa_to_va(splitPDE->page_frame_number * PAGE_SIZE);
    }
    else {
        //小页，Pt数组是有值的，先复制
        memcpy(VaPt, Table.PteAddress - pteindex, PAGE_SIZE);
    }
    //这里我的Table结构中的页表均为指针，因此这里做的减法是指针减法，不需要乘以8
    //另外这里做减法的意思是获取页表的起始地址，即基地址
    memcpy(Va4kb, replaceAlignAddr, PAGE_SIZE);
    memcpy(VaPdt, Table.PdeAddress - pdeindex, PAGE_SIZE);//指针减法
    memcpy(VaPdpt, Table.PdpteAddress - pdpteindex, PAGE_SIZE);
    //替换页表的页框号，从Pte开始一直到Pml4e
    _disable();//关中断防止替换被打断
    auto pReplacePte = (pte_64*)&VaPt[pteindex];
    pReplacePte->page_frame_number = va_to_pa(Va4kb) / PAGE_SIZE;
    auto pReplacePde = (pde_64*)&VaPdt[pdeindex];
    pReplacePde->page_frame_number = va_to_pa(VaPt) / PAGE_SIZE;
    pReplacePde->large_page = 0;
    pReplacePde->ignored_1 = 0;
    pReplacePde->page_level_cache_disable = 1;
    auto pReplacePdpte = (pdpte_64*)&VaPdpt[pdpteindex];
    pReplacePdpte->page_frame_number = va_to_pa(VaPdt) / PAGE_SIZE;
    auto pReplacePml4e = (pml4e_64*)&VaPml4t[pml4eindex];
    pReplacePml4e->page_frame_number = va_to_pa(VaPdpt) / PAGE_SIZE;
    //刷新TLB
    __invlpg(pReplacePml4e);
    _enable();
    return true;
}
```

最后写一段代码调用上面两个函数：

```c
bool isolationPages(HANDLE pid, void* iso_address)
{
    if (!MmIsAddressValid(iso_address)) {
        DbgPrintEx(77, 0, "Invalid address: %p\r\n", iso_address);
        return false;
    }
    PEPROCESS Process;
    KAPC_STATE Apc{ 0 };
    NTSTATUS status;
    void* AliginIsoAddr;
    PTE_TABLE Table{ 0 };
    status = PsLookupProcessByProcessId(pid, &Process);
    //附加要隔离的进程，每个进程的空间是独立的，这一步一定要做
    KeStackAttachProcess(Process, &Apc);
    AliginIsoAddr = PAGE_ALIGN(iso_address);
    Table.LineAddress = AliginIsoAddr;
    getPagesTable(Table);
    bool bSuc = false;
    while (1) {
        //大页分割
        pde_64 splitPDE{ 0 };
        if (Table.PdeAddress->large_page) {
            bSuc = splitLargePages(Table.PdeAddress, &splitPDE);
            if (!bSuc)break;
            if (Table.PdeAddress->flags & 0x100) {
                Table.PdeAddress->flags &= ~0x100;
                /*
                这里以及下面的的Table.PteAddress->global = 0;是在关闭G位
                */
            }
        }
        else {
            if (Table.PteAddress->global) {
                Table.PteAddress->global = 0;
            }
        }
        cr3 Cr3;
        Cr3.flags = __readcr3();
        bSuc = isolationPageTable(Cr3, AliginIsoAddr, &splitPDE);
        if (bSuc) {
            DbgPrintEx(77, 0, "isolation succeed.\r\n");
            break;
        }
        else {
            DbgPrintEx(77, 0, "failed to isolation pages.\r\n");
            break;
        }
    }
    KeUnstackDetachProcess(&Apc);
    ObDereferenceObject(Process);
    return bSuc;
}
```

这里有一步之前没讲的操作，设置了G位。在CPU的内部有一个表叫做TLB，学过计组的都知道，这玩意叫做快表，其实是一个Cache，内部记录了很多东西，比如直接记录一个虚拟地址对应的物理地址，而不需要通过四级页表机制来查找。每次切换进程时，CR3都会改变，而TLB是跟着CR3变的。但是由于操作系统的高位映射是基本不变的，如果每次切换CR3都重新维护一个TLB，就会造成很大开销。

因此诞生了Global，即全局位。这个位一旦被置1，那么切换进程CR3的时候，就不会刷新PDE或PTE的G位为1的页。这将导致：进程A切换到进程B，而进程A的某个虚拟地址还是保存在进程B的TLB中，下次进程B查找这个虚拟地址的值后，就会在TLB中找到进程A虚拟地址中对应的物理地址。

这对我们的隔离操作是有害的， 因为若进程A恰好是我们隔离的进程，我们Hook了某个函数后，其他进程又从TLB中找到了被Hook的这个函数，那么隔离就失效了。因此，我们关闭伪造页对应的PTE或PDE的G位，强制切换进程时刷新TLB，其他进程就找不到我们的Hook了。

讲完上面的代码，PTE Hook的核心进程隔离部分已经实现了，接下来只需要使用inlineHook框架就好了。下面是一个例子：

```c
bool PTEHookManager::PTEHook(HANDLE pid, void** oFuncAddr, void* targetFuncAddr)
{
    static bool bFirst = true;
    if (bFirst) {
        m_PTEBase = nullptr;
        m_trampLine = (char*)ExAllocatePoolWithTag(NonPagedPool, PAGE_SIZE * 5, 'Line');
        if (!m_trampLine) {
            DbgPrintEx(77, 0, "failed to create trampline.\r\n");
            return false;
        }
        memset(&m_info, 0, sizeof(m_info));
        memset(&m_globalBit, 0, sizeof(m_globalBit));
        m_trampLineUsed = 0;
        bFirst = false;
    }
    PEPROCESS Process{ 0 };
    KAPC_STATE Apc{ 0 };
    NTSTATUS status;
    const uint32_t breakBytesLeast = 14;//ff 25
    const uint32_t trampLineBreakBytes = 20;
    uint32_t uBreakBytes = 0;
    char* TrampLine = m_trampLine + m_trampLineUsed;
    hde64s hde_info{ 0 };
    char* JmpAddrStart = (char*)*oFuncAddr;
    if (m_curHookCount == MAX_HOOK_COUNT) {
        DbgPrintEx(77, 0, "Hook too many.\r\n");
        return false;
    }
    status = PsLookupProcessByProcessId(pid, &Process);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(77, 0, "failed to get pid.\r\n");
        return false;
    }
    auto ret = isolationPages(pid, *oFuncAddr);
    if (!ret)return false;
    DbgPrintEx(77, 0, "ready to diasm.\r\n");
    while (uBreakBytes < breakBytesLeast) {
        if (!hde64_disasm(JmpAddrStart + uBreakBytes, &hde_info)) {
            DbgPrintEx(77, 0, "failed to diasm addr.\r\n");
            ObDereferenceObject(Process);
            return false;
        }
        uBreakBytes += hde_info.len;
    }
    DbgPrintEx(77, 0, "finish disasm.\r\n");
    unsigned char trampLineCode[trampLineBreakBytes] = {
        0x6A, 0x00,                                                // push 0
        0x3E, 0xC7, 0x04, 0x24, 0x00, 0x00, 0x00, 0x00,            // mov dword ptr ss : [rsp] , 0x00
        0x3E, 0xC7, 0x44, 0x24, 0x04, 0x00, 0x00, 0x00, 0x00,      // mov dword ptr ss : [rsp + 4] , 0x00
        0xC3                                                       // ret
        };
    char absolutejmpCode[14] = { 0xFF,0x25,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00 };
    *((PUINT32)&trampLineCode[6]) = (UINT32)(((uint64_t)JmpAddrStart + uBreakBytes) & 0XFFFFFFFF);
    *((PUINT32)&trampLineCode[15]) = (UINT32)((((uint64_t)JmpAddrStart + uBreakBytes) >> 32) & 0XFFFFFFFF);
    memcpy(TrampLine, JmpAddrStart, uBreakBytes);
    memcpy(TrampLine + uBreakBytes, trampLineCode, trampLineBreakBytes);
    //添加Hook信息
    for (int i = 0; i < MAX_HOOK_COUNT; i++) {
        if (m_info[i].pid == 0) {
            m_info[i].oriAddr = JmpAddrStart;
            memcpy(m_info[i].oriBytes, JmpAddrStart, 14);
            m_info[i].pid = pid;
            m_curHookCount++;
            break;
        }
    }
    DbgPrintEx(77, 0, "ready to create trampline.\r\n");
    *((ULONG64*)(&absolutejmpCode[6])) = (ULONG64)targetFuncAddr;
    KeStackAttachProcess(Process, &Apc);
    //auto oIrpl = WPOFF();
    //memcpy(JmpAddrStart, absolutejmpCode, 14);
    //DbgPrintEx(77, 0, "[JmpAddrStart]%p\r\n",JmpAddrStart);
    DbgPrintEx(77, 0, "[absolutejmpCode]");
    for (int i = 0; i < 14; i++) {
        DbgPrintEx(77, 0, "%02X", (unsigned char)absolutejmpCode[i]);
    }
    DbgPrintEx(77, 0, "\r\n");
    BOOLEAN success = MDLWriteMemory(JmpAddrStart, absolutejmpCode, 14);
    if (!success) {
        DbgPrintEx(77, 0, "failed to MDL write jmpcode.\r\n");
        return false;
    }
    KeUnstackDetachProcess(&Apc);
    *oFuncAddr = TrampLine;
    m_trampLineUsed += uBreakBytes + trampLineBreakBytes;
    ObDereferenceObject(Process);
    return true;
}
```

下面我尝试Hook explore.exe进程的NtCreateFile函数看看效果，系统版本：Win10 1903，另外我还测试了Win10 21H2、Win11 21H2，挂了6小时左右未出现PG

```c
NTSTATUS HookNtCreateFile(
    OUT PHANDLE FileHandle,
    IN ACCESS_MASK DesiredAccess,
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    OUT PIO_STATUS_BLOCK IoStatusBlock,
    IN PLARGE_INTEGER AllocationSize OPTIONAL,
    IN ULONG FileAttributes,
    IN ULONG ShareAccess,
    IN ULONG CreateDisposition,
    IN ULONG CreateOptions,
    IN PVOID EaBuffer OPTIONAL,
    IN ULONG EaLength)
{
    DbgPrintEx(77, 0, "[+]Create Files.\r\n");
    if (ObjectAttributes && ObjectAttributes->ObjectName && ObjectAttributes->ObjectName->Buffer) {
        wchar_t* name = (wchar_t*)ExAllocatePoolWithTag(NonPagedPool, ObjectAttributes->ObjectName->Length + sizeof(wchar_t), 'name');
        RtlZeroMemory(name, ObjectAttributes->ObjectName->Length + sizeof(wchar_t));
        RtlCopyMemory(name, ObjectAttributes->ObjectName->Buffer, ObjectAttributes->ObjectName->Length);
        if (wcsstr(name, L"MyHook.txt")) {
            ExFreePool(name);
            return STATUS_ACCESS_DENIED;
        }
        ExFreePool(name);
    }
    return g_OriginNtCreateFile(FileHandle, DesiredAccess, ObjectAttributes, IoStatusBlock, AllocationSize, FileAttributes, CreateOptions, EaBuffer, EaLength);
}
```

效果图：

![](https://xzfile.aliyuncs.com/media/upload/picture/20260326215843-e68b27ad-291b-1.png)

### 结语

PTE Hook其实已经是一种比较老的方法了，不过研究完之后，我觉得对于x64架构的认识以及Windows内核的入门还挺有帮助的。

> （附录完。以上原文内容转载自先知社区，版权归原作者所有。）
