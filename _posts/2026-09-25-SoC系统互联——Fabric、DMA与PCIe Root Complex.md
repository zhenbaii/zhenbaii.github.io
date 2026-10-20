---
layout:       post
title:        "SoC系统互联——Fabric、DMA与PCIe Root Complex"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 操作系统
tags:
    - "操作系统"
    - "SoC"
    - "interconnect"
    - "DMA"
    - "PCIe"
    - "device-tree"
---
# SoC系统互联——Fabric、DMA与PCIe Root Complex

理解现代 SoC 的关键，不是背诵每一种总线，而是建立一个统一的事务模型：

> **某个 initiator 发起一笔 transaction，transaction 携带地址(物理地址)、数据、访问属性和发起者身份；系统互联根据这些信息完成路由、转换和访问控制，最终把它送到 target。**

CPU 的 MMIO、设备 DMA、PCIe 设备访问内存和 MSI 中断，都可以放进这个模型。它们的差别主要在于谁发起事务、事务经过哪些桥和转换单元、最终地址落到哪里。

## 1. 从“总线”转向“系统互联”

传统的总线模型像一条被多个部件共享的通路：

```text
CPU ───── Bus ───── DRAM
          │
          ├──────── UART
          └──────── GPIO
```

现代 SoC 中同时存在多个 CPU、GPU、NPU、DMA 控制器、PCIe 控制器和内存控制器。它们对带宽、延迟、一致性和服务质量的要求不同，很难再由一条共享总线承担。因此，现代芯片更接近一个片上网络：

```text
                          +-------------+
                          | CPU Cluster |
                          +-------------+
                                 │ coherent traffic
                        +--------v---------+
                        | Coherent Fabric  |
                        +--------+---------+
                                 │
       +-----------+-------------+-------------+-----------+
       │           │                           │           │
   +---v---+   +---v---+                   +---v---+   +---v---+
   |  GPU  |   |  NPU  |                   | PCIe  |   |  DDR  |
   +-------+   +-------+                   |  RC   |   | Ctrl  |
                                           +---+---+   +-------+
                                               │
                                          PCIe Endpoint
```

这里常见的几个词不是同一层概念：

| 名称 | 更准确的含义 |
|---|---|
| AXI、ACE、CHI、APB | 接口或事务协议，规定双方如何交换请求和响应 |
| interconnect / fabric | 把多个 initiator 和 target 连接起来的整体互联基础设施 |
| NoC | 用网络化路由实现的片上互联结构，可能承载 AXI、CHI 等协议语义 |
| PCIe、USB | SoC 与芯片外设备通信的外部互联协议 |

因此，不能简单地说“Fabric 就是 AXI 总线”。一个 Fabric 可能包含多级 NoC、协议桥、地址译码器、防火墙和一致性单元；AXI 只是其中可能采用的一种接口协议。

## 2. 一笔 transaction 包含什么

从软件中执行一次：

```c
writel(value, address);
```

看起来只有地址和数据，但硬件中的事务通常还包含更多信息：

```text
source/requester identity   谁发起
address                     访问哪里
read/write                  访问方向
data + byte enable          数据和有效字节
size/burst                  长度及传输方式
security/privilege          安全域和特权属性
cache/coherency attributes  缓存、一致性属性
transaction ID              用于并发请求和响应配对
```

这里的 `address` 不是一个脱离上下文后仍有唯一含义的“全局地址”，而是**当前发起者在当前互联域中使用的地址**。CPU 执行指令时使用 CPU 虚拟地址；事务穿过 MMU 后，进入 SoC Fabric 的通常是系统物理地址；设备可能使用 DMA 地址或 IOVA；PCIe 链路上的 TLP 又使用 PCIe 地址。事务经过 MMU、IOMMU 或桥时，地址字段可能被转换。

这些地址和其它元数据都不是驱动每次调用 `writel()` 时手工拼装的。驱动建立映射并配置硬件，CPU、设备总线接口和互联硬件则根据当前发起者及其配置自动生成、携带或转换 transaction。

### Initiator 与 target

能够主动发起访问的一方是 initiator，也常被称作 requester 或 bus master，例如：

- CPU；
- GPU、NPU；
- 设备自带的 DMA engine；
- 独立 DMA controller；
- PCIe Root Complex 的入站事务接口。

接收访问的一方是 target，例如：

- DDR controller；
- UART、GPIO 等 MMIO 寄存器；
- GIC、ITS；
- PCIe Root Complex 的 DBI/APB/配置窗口。

同一个硬件可以同时具有两种身份。例如网卡的寄存器是 CPU MMIO 的 target，而网卡的 DMA engine 又是内存系统的 initiator。

## 3. 地址决定事务去向，但地址必须带上所属空间

“地址是什么”必须沿着一次访问的路径回答。最常见的几种地址如下：

| 地址 | 谁使用 | 含义 |
|---|---|---|
| CPU virtual address（VA） | CPU 指令和操作系统指针 | 先由 CPU MMU 翻译，通常不会原样出现在 SoC Fabric 上 |
| system physical address（PA） | CPU MMU 输出、SoC Fabric | SoC 全局物理地址空间中的地址，可以指向 DRAM，也可以指向 MMIO |
| DMA/bus address | 设备 DMA engine | 驱动写入设备描述符或寄存器、供设备发起访问的地址 |
| IOVA | 启用 IOMMU 的设备 | 一种需要由 IOMMU 翻译成 PA 的 DMA 地址 |
| PCIe address | PCIe TLP | PCIe 地址域中的 Memory/IO/Configuration 地址，由 Root Complex 与系统地址空间建立窗口映射 |

### CPU 访问时，Fabric 看见的通常是物理地址

假设驱动执行：

```c
writel(value, mmio_ptr);
```

`mmio_ptr` 是 CPU 要解引用的内核虚拟地址，不是直接放到 SoC 引脚上的地址。一次典型访问是：

```text
内核虚拟地址 VA
      ↓ CPU MMU 页表翻译
系统物理地址 PA
      ↓ 进入 SoC Fabric
地址译码和路由
      ↓
某个 MMIO target
```

所以，当讨论“Fabric 根据地址把 CPU MMIO 送到 UART/NPU/GIC”时，这里的地址通常是 **CPU 地址翻译之后的系统物理地址**。物理地址不等于“DRAM 地址”：DRAM、ROM 和设备 MMIO 窗口共同占据系统物理地址空间。

SoC 会为这份系统物理地址空间定义一张 address map。例如：

```text
0x0000_0000 ...  → DRAM/ROM
0xFDAB_0000 ...  → NPU MMIO
0xFE60_0000 ...  → GIC
0xFE66_0000 ...  → ITS
0xFEB5_0000 ...  → UART
```

Fabric 对 PA 做 address decode，选择对应的 target。因此下面两条 CPU 路径只是目的地址不同：

```text
CPU VA → MMU → DRAM PA → Fabric → DDR Controller
CPU VA → MMU → MMIO PA → Fabric → UART/NPU/GIC
```

### 设备 DMA 时，设备最初发出的不一定是 PA

驱动通过 DMA API 得到一个 DMA address，再把它写进设备的描述符或寄存器。这个地址只保证“设备可以用它完成 DMA”，不保证它与 CPU PA 数值相同。

无 IOMMU 且采用恒等映射时：

```text
设备发出的 DMA address
      = system PA
      ↓
Fabric → DDR
```

存在固定总线偏移或桥窗口时：

```text
设备 bus address
      ↓ bus/bridge translation
system PA
      ↓
Fabric → DDR
```

启用 IOMMU 时：

```text
设备发出的 DMA address（此时是 IOVA）
      ↓ IOMMU：依据 Stream ID 选择页表
system PA
      ↓
Fabric → DDR
```

因此，说“设备向地址 `0x4000_0000` DMA”仍然不够完整：这个数可能是 IOVA、设备总线地址，也可能恰好就是 PA，必须结合设备的 DMA/IOMMU 拓扑判断。

### PCIe 链路上使用的是 PCIe 地址

PCIe Endpoint 发出的 Memory Read/Write TLP 携带 PCIe 地址。Root Complex 再通过 inbound window 和 IOMMU 等机制，把它转换为 SoC 内部使用的地址：

```text
PCIe TLP address
      ↓ Root Complex inbound translation
DMA address / IOVA
      ↓ IOMMU（若启用）
system PA
      ↓ Fabric address decode
DRAM 或 MMIO target
```

反方向上，CPU 访问 Endpoint BAR 时也会发生转换：

```text
CPU VA
      ↓ MMU
分配给 PCIe outbound window 的 system PA
      ↓ Root Complex outbound translation
PCIe address
      ↓
Endpoint BAR
```

在简单平台中，各层地址可能采用恒等映射，数值看起来完全相同；但这只表示映射关系碰巧是 identity mapping，不表示这些地址空间在概念上是同一个东西。

有了这些区分，下面几件看似不同的事情才能被准确地放进统一模型：

```text
CPU store：VA → PA                         → UART
NPU DMA：DMA address/IOVA → PA             → DDR
PCIe DMA：PCIe address → IOVA/PA           → DDR
PCIe MSI：PCIe MSI address → ITS 所在 PA    → ITS
```

## 4. DMA 不是一种总线协议

DMA 描述的是：**CPU 不亲自搬运数据，由另一个 initiator 直接访问内存。** 它没有规定必须使用 AXI、PCIe 或某一种具体总线。

CPU 访问内存：

```text
CPU load/store 使用 VA
      ↓ CPU MMU
system PA
      ↓
SoC Fabric
      ↓
DDR Controller
      ↓
DRAM
```

SoC 内置设备 DMA：

```text
NPU / Ethernet DMA engine
      ↓ DMA address / IOVA
IOMMU 或 bus translation（可选）
      ↓ system PA
SoC Fabric
      ↓
DDR Controller
      ↓
DRAM
```

外部 PCIe 设备 DMA：

```text
NVMe Endpoint
      ↓ PCIe address + Requester ID
PCIe Root Complex
      ↓ DMA address / IOVA
IOMMU（可选）
      ↓ system PA
SoC Fabric / DDR Controller
      ↓
DRAM
```

### 两种 DMA 发起方式

1. **设备自带 DMA engine**：网卡、NVMe、GPU、NPU 等设备自己成为 initiator。
2. **独立 DMA controller 代为搬运**：UART、SPI 等外设通过 DMA request 信号请求 DMA controller 在 FIFO 与内存之间搬运数据。此时真正向内存发起事务的是 DMA controller，而不是 UART 本身。

DMA 地址与 CPU 物理地址不必相同。启用 IOMMU 时，设备发出的是 IOVA，IOMMU 根据 requester/Stream ID 选择地址空间，再完成 `IOVA → PA` 翻译。驱动同时持有的 CPU 虚拟地址只用于 CPU 自己访问缓冲区，不能直接写进设备。详细内容见 [DMA——直接内存访问](/操作系统/2026/09/13/DMA-直接内存访问/)。

## 5. Requester identity 的地位

地址回答“访问哪里”，requester identity 回答“谁在访问”。IOMMU、MSI、访问控制和虚拟化都依赖这个身份。

不同互联域使用的身份形式可能不同：

```text
SoC 内置设备        → Stream ID / master ID 等
PCIe Function       → Requester ID（通常由 BDF 形成）
GIC ITS             → DeviceID
```

这些 ID 在穿过桥和控制器时可能被转换，而不是原样贯穿整个系统：

```text
PCIe BDF / Requester ID
          ↓ Root Complex 映射
IOMMU Stream ID 或 ITS DeviceID
```

设备树中的 `iommu-map` 和 `msi-map` 就可以描述这类“PCI requester ID 到另一个编号空间”的映射。

这里还要区分三个名字相近但意义不同的概念：

| 名称 | 用途 |
|---|---|
| PCI Vendor ID / Device ID | 标识设备厂商和型号，用于驱动匹配 |
| PCI BDF / Requester ID | 标识 PCI hierarchy 中具体的 Function，也是事务发起者身份 |
| ITS DeviceID | ITS 翻译 MSI 时使用的设备索引，可能由 Requester ID 映射得到 |

## 6. PCIe Root Complex 是两个世界之间的桥

Root Complex（RC）是主机侧硬件，通常集成在 SoC 中，不属于 NVMe、网卡等 Endpoint。它一侧连接 SoC 内部 Fabric，另一侧连接 PCIe hierarchy：

```text
CPU / DRAM / IOMMU / ITS
           │
       SoC Fabric
           │
   +-------v--------+
   | Root Complex   |
   +-------+--------+
           │ PCIe Link
      +----v-----+
      | Root Port|
      +----+-----+
           │
      PCIe Switch（可选）
       ├── NVMe Endpoint
       ├── NIC Endpoint
       └── GPU Endpoint
```

几个相关概念应当分开：

- **Root Complex**：主机侧 PCIe 根实体，连接处理器/内存系统和 PCIe hierarchy。
- **Root Port**：RC 提供的一条下游 PCIe 端口。
- **Endpoint**：NVMe、网卡、显卡等最终设备。
- **PCIe Switch**：扩展上下游端口，形成更深的 PCIe 拓扑。

Root Complex 常负责：

- 链路训练和 Root Port 状态管理；
- PCI 配置空间访问；
- CPU 地址窗口到 PCIe 地址空间的 outbound translation；
- PCIe DMA/MSI 到 SoC 地址空间的 inbound translation；
- PCIe TLP 与 SoC 内部 transaction 之间的协议转换；
- 传递或转换 Requester ID；
- INTx、MSI/MSI-X 相关路由；
- PHY、时钟、复位和电源域控制。

### CPU 访问 Endpoint BAR

```text
CPU 用内核 VA 执行 MMIO load/store
       ↓ CPU MMU
PCIe outbound window 对应的 system PA
       ↓ SoC Fabric
Root Complex：system PA → PCIe address
       ↓ 形成 PCIe Memory Read/Write TLP
PCIe Endpoint BAR
```

### Endpoint 发起 DMA

```text
Endpoint DMA engine
       ↓ PCIe address + Requester ID
Root Complex inbound translation
       ↓ IOVA 或 system PA
IOMMU（可选）
       ↓ system PA
SoC Fabric
       ↓ address decode
DRAM
```

TLP（Transaction Layer Packet）只是 PCIe 链路上传输事务的报文形式。驱动通常不会手工构造普通 Memory TLP；驱动配置 BAR、队列、DMA 地址和控制寄存器，Endpoint 与 Root Complex 的 PCIe 硬件负责生成和解析 TLP。

## 7. DMA write 与 MSI write 的统一性

对于 PCIe Endpoint，DMA 写和 MSI 都可以表现为 PCIe Memory Write。区别主要在目标地址和数据语义：

```text
DMA write:
    requester = 03:00.0
    address   = 设备可见的 PCIe/DMA 地址
    data      = 设备数据
    translation = RC window → IOVA → PA（依平台而定）
    target    = DRAM

MSI write:
    requester = 03:00.0
    address   = 设备可见的 MSI target address
    data      = EventID
    translation = RC/MSI window → GITS_TRANSLATER 所在 PA
    target    = GIC ITS
```

这里说“MSI 写入 `GITS_TRANSLATER`”描述的是**最终 target**。操作系统实际写入 PCIe 设备 MSI/MSI-X 表的是设备所在 PCIe 地址域能够使用的 Message Address。若 Root Complex 采用恒等映射，这个数可能与 `GITS_TRANSLATER` 的系统物理地址相同；若存在地址窗口转换，两者可以不同。

在 GICv3 ITS 场景中，一条典型路径是：

```text
设备内部某个 queue 完成工作
       ↓
设备的 MSI/MSI-X logic 读取 Message Address/Data
       ↓
PCIe Memory Write
       │ requester = BDF/RID
       │ data = EventID
       ▼
Root Complex：RID → ITS DeviceID
       ↓
ITS：(DeviceID, EventID) → LPI + Collection
       ↓
目标 CPU Redistributor
       ↓
CPU 接收到 LPI
```

队列只是事件来源，真正发出 Memory Write 的是设备内部的中断/MSI-X 逻辑和 PCIe 接口。MSI Address/Data 也不是设备随意决定的，而是操作系统的 PCI、IRQ 和 ITS 驱动先建立映射，再写入设备的 MSI capability 或 MSI-X table。GICv3、ITS 和 LPI 的细节见 GICv3。

## 8. 操作系统中的 PCI 分层

操作系统编写 PCI 支持时，面对的不只是 Root Complex。合理的软件分层是：

```text
平台 PCIe Host 驱动
    ├── 操作 SoC/DesignWare 等控制器寄存器
    ├── 配置 PHY、clock、reset、link
    ├── 建立配置空间和地址转换窗口
    └── 向上提供 config-space access
                │
                ▼
通用 PCI Core
    ├── 扫描 Bus/Device/Function
    ├── 识别 bridge 并递归枚举
    ├── 分配 BAR 和 bus number
    ├── 遍历 capability
    ├── 管理 INTx/MSI/MSI-X
    └── 按 Vendor/Device/Class 匹配驱动
                │
                ▼
Endpoint 驱动
    ├── NVMe
    ├── 网卡
    ├── GPU
    └── 其他设备
```

所以：

> **Host Controller 驱动负责让操作系统能够进入 PCIe 世界；PCI Core 负责管理整个 PCI hierarchy；Endpoint 驱动只关心自己的 BAR、队列、中断和 DMA 语义。**

Endpoint 通常不需要作为普通平台设备逐个写入设备树，因为 PCI Core 会从配置空间动态枚举它们。设备树主要描述不可枚举的 Root Complex 及其平台资源。

## 9. 设备树描述的是多张相互正交的图

设备树的节点嵌套关系不能单独代表所有硬件连接。至少需要区分以下几张图：

### 设备/总线拓扑

```text
SoC
 ├── I2C controller
 │    └── sensor
 └── PCIe Root Complex
      └── 动态枚举的 PCIe hierarchy
```

相关字段包括 `compatible`、`reg`、`ranges`、`bus-range` 和 `dma-ranges`。

设备树中的 `reg` 也不能无条件读成 CPU 物理地址。`reg` 的地址由**父总线的地址空间**解释：

```text
child reg 中的地址
      ↓ 父节点 ranges
父总线地址
      ↓ 继续逐级翻译
root/system physical address
```

对于直接位于根节点或 identity-mapped `simple-bus` 下的 SoC 外设，`reg` 最终通常就是 CPU 可见的 MMIO PA；对于 PCI 子节点，地址 cell 则包含 PCI 地址空间类型和 BDF 等 PCI 语义，需要通过 PCI host 的 `ranges` 映射到 CPU system PA。`dma-ranges` 描述的是设备发起 DMA 时，子总线 DMA 地址如何映射到父总线地址。换言之，`reg/ranges` 主要解释 CPU 如何访问设备，`dma-ranges` 主要解释设备如何访问内存。

### 中断拓扑

```text
sensor → GPIO interrupt controller → GIC → CPU
PCIe Endpoint → ITS → LPI → GIC Redistributor → CPU
```

相关字段包括 `interrupt-parent`、`interrupts`、`interrupt-map`、`msi-parent` 和 `msi-map`。详见 [中断管理和设备树](/操作系统/2026/09/24/中断管理和设备树/)。

### DMA/IOMMU 拓扑

```text
device/requester → IOMMU domain → memory
```

相关字段包括 `iommus`、`iommu-map`、`dma-ranges` 和 coherent 属性。

### 时钟、复位和电源拓扑

相关字段包括 `clocks`、`resets` 和 `power-domains`。

因此，一个设备树节点的语法父节点、数据访问路径、中断父控制器和 IOMMU 上游可以分别属于不同的拓扑。设备树通过 phandle 把这些图连接起来。

## 10. 用 RK3588 串起完整路径

以 RK3588 上的 NPU、PCIe 和 GICv3 为例，可以建立下面的整体图：

```text
                                     +----------------+
                                     |      CPU       |
                                     +---+---------+--+
                                         │         ▲
                              MMIO/DMA    │         │ IRQ
                                         ▼         │
                     +-----------------------------------------+
                     |           SoC Fabric / NoC              |
                     +--+------------+-------------+-----------+
                        │            │             │
                 +------v----+  +----v-----+  +----v-----------+
                 | DDR Ctrl  |  | NPU/IOMMU|  | GICv3 + ITS    |
                 +-----------+  +----------+  +----------------+
                        ▲                              ▲
                        │ DMA                         │ MSI write
                        │                              │
                  +-----+------------------------------+---+
                  |         PCIe Root Complex              |
                  +------------------+----------------------+
                                     │ PCIe
                                  Endpoint
```

几条典型访问路径如下：

1. CPU 配置 NPU：`CPU VA → MMU → NPU MMIO PA → Fabric → NPU`。
2. NPU 访问任务缓冲区：`NPU IOVA → IOMMU → DRAM PA → Fabric → DDR`。
3. CPU 访问 PCIe BAR：`CPU VA → MMU → RC window PA → RC → PCIe address → Endpoint`。
4. PCIe 设备 DMA：`PCIe address + RID → RC → IOVA/PA → IOMMU/Fabric → DDR`。
5. PCIe 设备 MSI：`PCIe MSI address + RID → RC → ITS MMIO PA → ITS → LPI → CPU`。

它们共享同一个总体模型，但每条路径经过的协议桥、地址转换和身份转换不同。

## 11. 阅读硬件路径时的固定方法

面对一个新设备，可以依次回答：

1. **谁是 initiator？** CPU、设备本身还是独立 DMA controller？
2. **谁是 target？** DRAM、某段 MMIO、ITS，还是 PCIe BAR？
3. **事务经过哪些互联和协议桥？** AXI/NoC、APB bridge、Root Complex？
4. **当前这一跳使用哪个地址空间？** CPU VA、system PA、bus address、IOVA，还是 PCIe address？
5. **哪里发生地址翻译？** MMU、IOMMU、RC outbound/inbound window？
6. **发起者身份是什么？** Stream ID、BDF/RID、ITS DeviceID？
7. **身份在哪里转换？** `iommu-map`、`msi-map` 或控制器内部映射？
8. **完成事件怎样回到 CPU？** SPI、GPIO 级联、INTx，还是 MSI→ITS→LPI？
9. **设备树描述的是哪张图？** 总线、中断、DMA/IOMMU，还是时钟/电源拓扑？

## 12. 常见误区

- **Fabric 不是一根总线。** 它是由协议、交换节点、桥和控制单元组成的互联网络。
- **DMA 不是一种总线。** 它描述非 CPU initiator 直接访问内存的行为。
- **“地址”不带所属空间就不完整。** CPU VA、system PA、DMA address、IOVA 和 PCIe address 可能数值相同，但语义和负责解释它们的硬件不同。
- **地址不等于完整事务。** Requester identity、权限、一致性属性同样重要。
- **PCIe Requester ID 不在 MSI Data 中。** MSI Data 可以承载 EventID，Requester ID 由 PCIe transaction header/sideband 身份提供。
- **Root Complex 不是 Endpoint。** 它属于主机/SoC，是内部 Fabric 和外部 PCIe hierarchy 之间的桥。
- **PCI 子系统不等于 Root Complex 驱动。** 后者只是平台相关的底层入口。
- **设备树父子关系不等于全部硬件拓扑。** 中断、DMA/IOMMU 和时钟关系由各自的 phandle 属性单独描述。

## 相关笔记

- [DMA——直接内存访问](/操作系统/2026/09/13/DMA-直接内存访问/)
- GICv3
- [中断管理和设备树](/操作系统/2026/09/24/中断管理和设备树/)
- 引入初步的平台驱动框架
