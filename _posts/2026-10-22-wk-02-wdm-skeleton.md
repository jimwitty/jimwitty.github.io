---
layout: post
title: "Win 内核驱动 第 2 批：WDM 模型与驱动骨架"
categories: windows-kernel
series_order: 2
---

- **本批主题**：WDM 驱动的三类分工与协作方式，以及一个 WDM 驱动的"骨架"——标准例程家族（DriverEntry、Reinitialize、AddDevice、Dispatch 家族、Unload）各自的职责、调用时机与默认处理原则。
- **对应原文档章节**：《内核模式驱动程序体系结构设计指南》之「编写 WDM 驱动程序」（Writing WDM Drivers）章，包括 Introduction to WDM、Types of WDM Drivers、Device Configurations and Layered Drivers、Kernel-Mode Driver Components 全部小节。
- **取材说明**：learn.microsoft.com 对抓取工具返回 403，正文取自微软官方同名设计指南 PDF（2019 版）对应章节逐节研读，并与 Learn 搜索索引摘录交叉核对。
- **学习目标**：学完本批，你应当能做到——拿到任意一个 WDM 驱动的源码或反汇编，先看它属于哪类驱动、处在设备堆栈的哪一层；再看它在 DriverEntry 里往 `MajorFunction[]` 表里填了哪些函数指针，就能说出"这个驱动对外提供什么服务、在哪些时机被系统回调、设备的生死由谁决定"。

---

## 一、本批地图

### 1. 一个 WDM 驱动的生命周期线

```
系统加载驱动映像（I/O 管理器创建 DRIVER_OBJECT）
   │
   ▼
DriverEntry ── 驱动级初始化：填 MajorFunction 表、AddDevice/Unload 入口，
   │           初始化驱动级资源；返回失败则驱动不加载，且 Unload 不会被调用
   │ （可选）注册 Reinitialize ── 等其他驱动都初始化完后再补一轮初始化
   ▼
PnP 管理器枚举到设备
   │
   ▼
AddDevice ── 设备级初始化：IoCreateDevice 建设备对象、设置标志、
   │         IoAttachDeviceToDeviceStack 把自己挂进设备堆栈
   ▼
IRP_MN_START_DEVICE ── PnP 管理器把硬件资源（端口/中断/DMA）分配下来，
   │                    此刻起驱动才真正"接管"设备
   ▼
运行期：Dispatch 例程家族反复被调用，处理 Create/Read/Write/
   │    DeviceControl/PnP/Power 等 IRP——这是驱动一生的绝大部分时间
   ▼
IRP_MN_REMOVE_DEVICE ── 设备被移除：每层驱动自上而下清理设备级资源、
   │                    IoDetachDevice 脱栈、IoDeleteDevice 删设备对象
   ▼
Unload ── 当驱动名下再无任何设备对象时由 PnP 管理器调用；
          只需回收 DriverEntry 建的驱动级资源，PnP 驱动的 Unload 往往只有一行 return
```

关键直觉：**DriverEntry 管"驱动这个软件"，AddDevice 管"设备这个实例"，Dispatch 管"请求"，Remove/Unload 管"善后"**。一个驱动可以被加载一次、为多个设备各建一个设备对象、处理无数个 IRP，最后随设备全部消失而卸载。

### 2. 三类驱动在设备堆栈中的位置（自顶向下，请求下行方向）

```
        用户态应用 / 上层内核组件
   ┌─────────────────────────────────┐
   │ 上层筛选驱动 (Upper Filter)      │ ← 增值功能：如键盘安全检查
   ├─────────────────────────────────┤
   │ 功能驱动 (Function Driver)      │ ← 设备的主驱动：处理读写、管电源策略
   ├─────────────────────────────────┤
   │ 下层筛选驱动 (Lower Filter)     │ ← 改硬件行为：如鼠标加速度换算
   ├─────────────────────────────────┤
   │ 总线驱动 (Bus Driver) 的 PDO    │ ← 枚举出子设备、报告资源、真正动电源寄存器
   └─────────────────────────────────┘
        总线（PCI / USB / i8042…）
   旁边还有：总线筛选驱动（bus filter，挂在总线驱动之上，如 ACPI 筛选驱动）
```

多层设备会嵌套成树：USB 控制器是 PCI 总线上的一个设备（有自己的功能驱动），同时又充当 USB 总线的总线驱动，枚举出 Hub，Hub 再枚举出游戏杆……每一层"设备"都有自己的一摞堆栈。文档的例子：PCI 驱动（总线）→ USB 主控制器 class/miniclass 对（既是功能驱动、又是下层总线驱动）→ USB Hub 驱动（总线）→ HID class/miniclass 对（游戏杆的功能驱动）+ 宏按键上层筛选 + 模拟鼠标下层筛选。

---

## 二、核心要点

1. **WDM 的立约方式是"角色制"**：任何 WDM 驱动必须先声明自己是总线、功能、筛选三类中的哪一类，并遵守该类的契约（如必须支持 PnP 和电源管理、包含 `Wdm.h` 而非 `Ntddk.h`）。它是哪类，直接决定它对每个 PnP 次功能码该"处理、下传还是完成"——先定角色，再谈代码。

2. **总线驱动 = 会枚举孩子的功能驱动**：它服务总线控制器本身（这是它的"功能驱动"面），同时探测挂在总线上的设备、为每个孩子创建 PDO（物理设备对象）、报告资源需求、在启停时真正去拨弄电源寄存器。但它**不处理子设备的读写请求**——那是孩子自己的功能驱动的事。常见总线（PCI、USB、SCSI、PnpISA）微软都自带了总线驱动。

3. **功能驱动是设备唯一的"正主"**：PnP 管理器对一个设备最多加载一个功能驱动（通常由设备厂商写），它提供设备的操作界面——处理读写、管设备电源策略。设备若以 raw mode 工作则可以没有功能驱动（也没有上下层筛选），一切 I/O 由总线驱动兜底。

4. **筛选驱动是可选的"中间人"，靠在堆栈中的位置吃饭**：上层筛选在功能驱动之上，典型用途是加值（如给键盘加额外安全检查、类筛选给一类设备统一加功能）；下层筛选在功能驱动之下、贴近硬件，典型用途是修正硬件行为（如鼠标移动的非线性换算）；总线筛选挂在总线驱动上方，如 ACPI 筛选负责执行设备电源策略。筛选驱动可以有任意多个，但文档明确警告：筛选层堆太多会拖垮性能。

5. **分层可以复用，于是有了 class/port、miniport 这一票"驱动对"**：系统把硬件无关的通用逻辑做成可复用的 class/port 驱动（键盘/鼠标 class 驱动、SCSI port、USB port），厂商只写设备相关的 miniclass/miniport 部分与其配对，两者合起来才是一个完整的功能驱动。逆向时看到一个很"瘦"的 .sys，多半是这个原因——主体逻辑在它的搭档里。

6. **DRIVER_OBJECT 是驱动的"登记表"，MajorFunction 表是它的服务菜单**：I/O 管理器加载驱动时创建 `DRIVER_OBJECT`，DriverEntry 的首要职责就是往里填入口点：`MajorFunction[IRP_MJ_XXX]` 数组填各 Dispatch 例程，`DriverExtension->AddDevice` 填 AddDevice，`DriverUnload`、`DriverStartIo`（如有）各就各位。填了什么，系统就按什么回调；没填的槽位，I/O 管理器有默认处理（多为直接失败）。ISR、IoCompletion 这类例程不在登记表里，而是通过调用系统例程在运行时动态登记。

7. **DriverEntry 只做"驱动级"初始化，设备级的事一律推迟**：它在系统线程上下文、IRQL = PASSIVE_LEVEL 被调用（所以可以等、可以碰分页内存）；它应当放在 INIT 段、执行完即被丢弃。它要导出入口点、建驱动级对象、拷贝保存 RegistryPath（注意：必须拷字符串内容，指针在返回后失效）、返回 NTSTATUS。**返回失败 = 驱动不加载，并且 Unload 永远不会被调用**——失败路径上自己分配的东西必须自己就地释放干净。

8. **AddDevice 是"每设备一次"的仪式，且它只建壳、不点火**：PnP 管理器为每个设备调一次。功能/筛选驱动的 AddDevice 做十件事，核心是：`IoCreateDevice` 建 FDO/筛选 DO（**不要指定设备名**，命名会绕过 PnP 的安全检查；用户态要访问就注册设备接口给符号链接）、把 PDO 指针存进设备扩展、设好缓冲方式标志（DO_BUFFERED_IO / DO_DIRECT_IO，要与下层保持一致）、初始化 remove lock、`IoAttachDeviceToDeviceStack` 挂栈并保存返回的"下一层设备对象"指针（下传 IRP 时 `IoCallDriver` 就靠它）、清掉 DO_DEVICE_INITIALIZING 标志。但真正的硬件资源要等 `IRP_MN_START_DEVICE` 才发下来——**在那之前驱动不许碰硬件**。

9. **总线驱动的 AddDevice 是个"兼职"**：它只在总线驱动充当自家控制器/适配器的功能驱动时被调用（如 USB Hub 驱动为 Hub 本身建 FDO）；为子设备建 PDO 是枚举逻辑干的，不走 AddDevice。

10. **Dispatch 例程的默认决策只有四种**：拿到 IRP 后——(a) 自己能办就办完并 `IoCompleteRequest` 完成它；(b) 不是自己的事就设置好下一层的 I/O 堆栈位置、用 `IoCallDriver` 下传；(c) 需要异步处理就入队、`IoMarkIrpPending` 并返回 STATUS_PENDING；(d) 参数非法就立刻以错误状态完成。**参数校验的责任在链条的最高层**（它最贴近请求来源），中间层和最低层一般信任上层传下来的参数合法，但涉及自家设备特有限制时仍须自查。

11. **每类 IRP 的"脾气"由它的语义决定默认动作**（详见第三节逐个拆解）：Create/Close 多为"登记存在感"；Read/Write 除最高层外一律异步；DeviceControl 在最低层用 switch 按 IoControlCode 逐个校验长度并处理，在上层基本原样下传；PnP/Power 必须让整条堆栈每一层都有机会过手；Cleanup 负责取消还挂在队列里、属于某个文件对象的未决 IRP；SystemControl（WMI）哪怕不支持也必须下传。

12. **Unload 与 PnP 移除是两回事**：设备消失走的是 `IRP_MN_REMOVE_DEVICE`（每层驱动在此删自己的设备对象和设备级资源）；等一个驱动名下再无设备对象、且无未决引用时，PnP 管理器才调它的 Unload。PnP 驱动的 Unload 因此通常极简（只回收 DriverEntry 建的驱动级资源，没有就直接 return）。而非 PnP（legacy）驱动没有 Remove 这条路，Unload 必须亲自干完全部善后：先关设备中断、再分阶段释放资源、删设备对象、脱栈。系统关机时 Unload 不会被调用——关机善后是 DispatchShutdown 的活。

---

## 三、关键知识点详解

### 3.1 WDM 三类驱动的分工与协作

**总线驱动（Bus Driver）**
- 服务对象：总线控制器、适配器或桥。"总线"的外延很宽：PCI、SCSI、USB 是总线，并口、串口、i8042 控制器也算总线（凡是能挂其他物理/逻辑/虚拟设备者皆是）。
- 四项职责：枚举总线上的设备（识别方式因总线而异，如 PCI 读配置空间、USB 走描述符）；响应 PnP 与电源 IRP；某些总线要复用仲裁总线访问；对子设备做通用管理（如以编程方式拨寄存器改变子设备电源状态）。
- 边界：它创建子设备的 PDO 供上层驱动"落户"，但不碰子设备的读写流量。

**功能驱动（Function Driver）**
- 设备的操作界面提供者：处理读写、管理设备电源策略（"策略"指决定何时让设备睡/醒，机制由总线驱动执行）。
- 常见实现是"驱动对"：class/miniclass、port/miniport——通用半边由微软提供（HID class、存储 port 等），设备相关半边由厂商提供，两个映像逻辑上是一个功能驱动。

**筛选驱动（Filter Driver）**
- 观察并（可选地）修改流经的 IRP，不拥有设备。按位置分四种：总线筛选（给总线增值，如 ACPI 电源策略筛选）、上层设备筛选（给单个设备增值）、上层类筛选（给一整类设备增值）、下层筛选（设备级/类级，贴近硬件修正行为）。
- 一个特殊纪律：如果筛选驱动的 AddDevice 发现这台设备不需要自己服务，**必须返回 STATUS_SUCCESS 且不建设备对象、不挂栈**，放行其余驱动正常组堆栈——返回失败会把整台设备的加载搞黄。

**分层示例走查（键盘/鼠标与 USB 游戏杆）**
- 键盘/鼠标：无论键鼠是直连还是共用 i8042 控制器，上层永远可以复用系统键盘/鼠标 class 驱动处理硬件无关逻辑，厂商只写 port 部分（i8042prt）对接硬件。若有人想在 class 驱动之上加一层筛选（如按平台过滤键盘输入），这层筛选必须"认得"和 class 驱动完全相同的 IRP 与 IOCTL 集合——**插入任何一层的前提是向下兼容两侧的接口契约，不得削弱上下层原有功能**。另外文档点名：系统 SCSI、视频 port 驱动不可替换；替换最低层驱动必须实现与原驱动完全相同的对上接口。

**WDM 版本差异（要点）**
- 跨系统驱动用 `IoIsWdmVersionAvailable` 运行时探测当前系统支持的 WDM 版本；每个 WDK 例程参考页都会标注其版本限制。差异主要集中在：个别 PnP 次功能码的引入版本（如 IRP_MN_SURPRISE_REMOVAL 需 WDM 1.10 / Windows 2000+）、Win98/Me 与 NT 系在 PnP、电源语义上的历史分歧（现代目标只有 NT 系，这部分多为考古）。保证兼容的最懒办法：只用最低目标版本支持的特性集。

### 3.2 DriverEntry 与初始化

**执行环境**：I/O 管理器在加载驱动时调用；上下文是系统线程；IRQL = PASSIVE_LEVEL；代码可分页、应置于 INIT 段（`#pragma alloc_text(INIT, ...)`），初始化结束即被回收。

**必需职责（按顺序）**：
1. 在 DRIVER_OBJECT / DriverExtension 中填好全部入口点（AddDevice、每个要处理的 MajorFunction、StartIo、Unload）。
2. 创建/初始化驱动级对象与资源；如用到设备专用线程或内核分发器对象（事件、互斥体等），在此或 AddDevice 里初始化。**按设备一份的对象不在此处建**——那是 AddDevice / START_DEVICE 的活。
3. 释放仅初始化期间用过的临时内存（可用分页池，因为在 PASSIVE_LEVEL 且此时还没接管放页面文件的设备）。
4. 返回 NTSTATUS；失败前须：释放已建的一切、（支持的话）把 FLUSH_BUFFERS/SHUTDOWN 的分发表项清 NULL、写一条错误日志。记住失败后 Unload 不兜底。

**可选职责**：
- `IoAllocateDriverObjectExtension`：给驱动对象挂一块驱动级上下文（如保存 RegistryPath 副本、全局配置）。
- 最高层驱动（如文件系统）可 `PsCreateSystemThread` 建自己的工作线程。
- 注册 Reinitialize 例程（见下）。
- 读注册表取初始化参数：RegistryPath 指向 `\Registry\Machine\System\CurrentControlSet\Services\<驱动名>`；**拷字符串，不存指针**。

**Reinitialize 例程**：为"分阶段初始化"而设。DriverEntry 里调 `IoRegisterDriverReinitialization` 注册（且只能在 DriverEntry 将返回 STATUS_SUCCESS 的前提下、在返回前最后调用），它会在 DriverEntry 返回、且其他驱动也初始化之后被调用，还可自我再注册排下一次（参数里带调用次数）。典型场景：kbdclass 要同时服务 PnP 键盘和 PnP 枚举不到的 legacy 端口——legacy 设备的设备对象创建、挂栈只能等别的驱动就位后在 Reinitialize 里补做。它拿不到 RegistryPath 参数，需要的话靠注册时传入的上下文指针把路径捎过去。

### 3.3 AddDevice 三种写法

| | 功能/筛选驱动 | 总线驱动 |
|---|---|---|
| 触发 | PnP 管理器为其服务的每个设备调用 | 仅当总线驱动作为自家控制器/适配器的**功能驱动**时调用 |
| 产物 | FDO 或筛选 DO，挂到 PDO 之上 | 控制器的 FDO |
| 为子设备建 PDO | 不涉及 | 走枚举逻辑，**不走** AddDevice |

功能/筛选驱动 AddDevice 的十步（已在核心要点 8 里概括），再强调三处易错点：
- **缓冲标志要"随下层"**：DO_BUFFERED_IO / DO_DIRECT_IO 描述的是 I/O 管理器为发往本堆栈的请求准备数据的方式，高层驱动应把该标志设成与紧邻下层相同的值，否则同一 IRP 在不同层看到的数据通道不一致。
- **电源标志**：可分页的驱动必须设 DO_POWER_PAGABLE；这类标志通常由总线驱动建 PDO 时定下，上层一般在 AddDevice 里原样继承、偶尔才改。
- **安全**：建设备对象时带上 `FILE_DEVICE_SECURE_OPEN` 特性，让 I/O 管理器对所有打开（包括相对打开、带尾随文件名的打开，如 `\Device\foo\abc`）都按设备对象的安全描述符检查——否则会出现"设备本身管理员才能开、其伪文件路径却人人能开"的命名空间绕过；一个驱动设了，该特性会被 PnP 管理器传播到该设备的全部设备对象。

### 3.4 Dispatch 例程家族：逐个职责与默认处理原则

**通则**（先于逐个看）：
- 分发例程拿到的是 (DeviceObject, IRP)；第一动作通常是 `IoGetCurrentIrpStackLocation` 看自己的那份 IO_STACK_LOCATION：主功能码已由分发表决定，但带次功能码（PnP、Power、部分 SCSI/文件系统请求）或控制码（DeviceControl）的，必须再看次码/控制码才知道具体要干什么。
- 运行环境：多数在**任意线程上下文**、PASSIVE_LEVEL 被调用；两个例外要刻在脑子里——最高层驱动的分发例程在**请求发起线程**的上下文里跑（文件系统即如此）；最低层及分页路径上的中间层驱动的 Read/Write/DeviceControl 可能在 **APC_LEVEL、任意上下文**被调用，因此这些例程及它们碰的数据**不可分页、不可阻塞等待**；冬眠/分页路径上驱动的 DispatchPower 可在 DISPATCH_LEVEL 被调用。
- 完成 IRP 的标准姿势：填 `Irp->IoStatus.Status`、填 `IoStatus.Information`（字节数或 0）、`IoCompleteRequest(Irp, IO_NO_INCREMENT)`、返回值与所填 Status 一致。
- 挂起的标准姿势：先 `IoMarkIrpPending` 再入队/下传，返回 STATUS_PENDING——**标了 pending 就必须返回 pending，返回了 pending 就必须先标过**，这对一一对应是 IRP 完成协议的生命线（第 6、7 批会深挖）。

**逐个拆解**：

- **DispatchCreate（IRP_MJ_CREATE）**——语义是"有人要拿这个设备的一个文件对象句柄"（用户态 CreateFile，或上层驱动 IoGetDeviceObjectPointer/挂栈）。与 Close 天生同步。多数驱动（尤其最低层）只需最小动作：Status 置成功、Information 置 0、完成并返回——即"登记存在"。复杂者在此初始化内部队列等状态。最高层驱动须校验 FileObject 的 FileName：长度为零才算合法打开设备本身，非空（如打开 `\device\parallel0\temp.dat` 这种伪文件）直接 STATUS_INVALID_PARAMETER 完成——这一步挡掉的是"用路径后缀在设备命名空间里乱探"的请求。可以与 Close 共用一个 DispatchCreateClose。

- **DispatchClose（IRP_MJ_CLOSE）**——某个文件对象的**最后一个句柄**已关闭且无未决引用时的通知。动作通常与 Create 对称的最小化完成。注意语义差：Close 时该文件对象的所有 I/O 都已了结，而下面 Cleanup 不是。

- **DispatchCleanup（IRP_MJ_CLEANUP，可选）**——某文件对象的最后一个句柄正在关闭、但针对它的未决 I/O 可能还在队列里。职责：遍历本设备队列中属于该文件对象的 IRP，逐一摘掉其 Cancel 例程指针（`IoSetCancelRoutine(Irp, NULL)`）后取消并完成它们。它在"关句柄的那个进程"的上下文里执行，该进程可能与当初打开句柄的进程不是同一个（句柄被 DuplicateHandle 复制过）——在此做与原始进程绑定的清理是个经典坑。它与 Read/Write 等例程并发执行，清理与队列操作必须做好同步。

- **DispatchRead / DispatchWrite（IRP_MJ_READ / IRP_MJ_WRITE）**——数据搬运。除最高层外一律**异步**：Dispatch 内只做参数校验，然后或入自家队列（经 IoStartPacket 交给 StartIo）或下传，最后返回 STATUS_PENDING，绝不在 Dispatch 里等传输完成。最高层负责校验长度等参数；最低层按缓冲方式干活：缓冲 I/O 操作 `Irp->AssociatedIrp.SystemBuffer`（系统已备好的内核缓冲）；直接 I/O 操作 `Irp->MdlAddress` 描述的用户页（已被上层或 I/O 管理器探测并锁定，不要重复 MmProbeAndLockPages；取系统地址用 `MmGetSystemAddressForMdlSafe`）。传输量超出设备单次能力时要**拆分**为多次部分传输（SCSI class 驱动是典型），但拆分决策尽量推迟到真正编程设备前（StartIo 里）做。大数据量驱动内部缓冲数据的，通常还要同时支持 FlushBuffers/Shutdown。

- **DispatchDeviceControl（IRP_MJ_DEVICE_CONTROL）+ DispatchInternalDeviceControl（IRP_MJ_INTERNAL_DEVICE_CONTROL）**——前者承载设备类型相关的 IOCTL：来源既有用户态的 `DeviceIoControl`，也有上层驱动 `IoBuildDeviceIoControlRequest` 自建的 IRP；控制码、输入/输出长度在 `Parameters.DeviceIoControl` 里。最低层的标准形态是一个大 switch：按 IoControlCode 分组、先逐个校验 InputBufferLength/OutputBufferLength 是否够装下对应结构体（不够立刻 STATUS_BUFFER_TOO_SMALL 完成）、能当场办的当场完成（性能最好），要排队的标 pending。**对不认识的控制码必须以错误状态完成 IRP，绝不能下传或装没看见**。高层驱动对 DeviceControl 几乎不校验参数（假定最懂设备的是底层），只复制 I/O 堆栈位置后 `IoCallDriver` 下传并原样返回其状态；若它自己分配了 IRP 或资源则须挂 IoCompletion。Internal 变体是驱动间的"内部协议"（如键盘/鼠标 class 向 port 发的启用/禁用中断控制码、并口的 IOCTL_PARALLEL_PORT_XXX），只在紧密耦合的驱动对之间使用。注意：绝大多数 IOCTL 数据走缓冲 I/O（SystemBuffer），哪怕设备对象标的是直接 I/O。

- **DispatchPnP（IRP_MJ_PNP）**——PnP 管理器用次功能码指挥设备的全生命周期：START_DEVICE、QUERY_REMOVE、REMOVE_DEVICE、QUERY_CAPABILITIES、QUERY_DEVICE_RELATIONS 等。铁律：**栈上每个驱动都有权过手**，IRP 从栈顶下发，功能/筛选驱动除非自己处理时出错，否则必须下传，最终由父总线驱动完成；**不认识的次码不许失败，必须原样下传且不动其状态**；处理成功要自己把 IoStatus.Status 设为 STATUS_SUCCESS（指望别人兜底是脆弱写法）；处理失败则以错误状态就地完成、不再下传；绝不能用 STATUS_NOT_SUPPORTED 去答一个自己"处理"了的 PnP IRP（那是 PnP 管理器预填的"无人处理"标记）。方向性：REMOVE 类从栈顶先处理，START 类要等下层（总线）先完成再轮到上层做自己的启动（靠 IoCompletion 在"回程"上处理）——下行加信息、回程改/删信息是 PnP 查询类 IRP 的约定姿势。细节是第 10 批的主菜，这里先立框架。

- **DispatchPower（IRP_MJ_POWER）**——电源管理器用次功能码指挥系统级与设备级电源状态变迁与唤醒。姿势与 PnP 类似：逐层过手，用 `PoCallDriver`（不是 IoCallDriver）下传，最终总线驱动实际改电源并完成。**方向取决于升降**：上电 IRP 先由最低层处理、逐层向上；下电 IRP 先由栈顶处理、逐层向下——因为上电要先有下层供能、下电要先让上层停工。可移动设备驱动在此还要查设备是否已被拔走，在则下传，不在则以 STATUS_DELETE_PENDING 就地完成。

- **DispatchQueryInformation / DispatchSetInformation（可选）**——主要出现在最高层/文件系统驱动：响应 `GetFileInformationByHandle`、`ZwQueryInformationFile`（查文件对象所代表设备的长度等信息）与 `SetEndOfFile`、`ZwSetInformationFile`（设 EOF）。并口 class、串口驱动是少数处理它们的设备驱动例子。

- **DispatchFlushBuffers（可选）**——响应 `FlushFileBuffers`：把内部缓冲的数据/元数据刷到设备。凡是自己在内存里攒数据的文
...[truncated 11085 chars]