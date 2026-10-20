---
layout:       post
title:        "kmalloc与vmalloc的区别——从线性映射到DMA"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 操作系统
tags:
    - "内存管理"
    - "Linux"
    - "memory-management"
    - "arm64"
    - "DMA"
    - "cache-coherence"
---
`kmalloc()` 和 `vmalloc()` 都返回**内核虚拟地址**，但它们解决的是不同的分配问题：`kmalloc()` 给出物理上连续的一段内存，通常直接使用内核已有的 RAM 线性映射；`vmalloc()` 优先保证内核虚拟地址连续，底层物理页可以分散，并在 vmalloc 区域为这些页建立映射。

讨论 DMA 时，还需要把第三件事单独拿出来：**CPU 如何缓存这段内存**。在 arm64 上，某些不支持硬件 DMA 一致性的设备使用 `dma_alloc_coherent()` 时，内核可能为分配到的页建立 Normal Non-Cacheable（Normal NC）CPU 映射。这说明“选哪些物理页”“返回哪个 CPU 虚拟地址”“该虚拟地址带什么内存属性”是三个不同的问题。

## 1. 先分清三个对象

```text
CPU 虚拟地址（VA） ── CPU 页表和内存属性 ──> 物理页（PA）
                                                 ▲
                                                 │ 总线转换 / IOMMU（若有）
                                             DMA 地址
```

- **物理页**是实际分配的 RAM。分配器负责避免一组正在使用的物理页又被分配给别人。
- **CPU 虚拟地址**是内核代码解引用的指针。页表除了决定它指向哪个物理页，还携带可缓存性、权限等属性。
- **DMA 地址**是驱动交给设备的数值。设备使用它发起访问；设备既不使用 `kmalloc()` 返回的 CPU VA，也不读取 CPU 页表。

因此，“两段 VA 指向同一组物理页”属于 CPU 地址映射问题；“设备能否访问这些页”属于 DMA 地址和设备能力问题。二者相关，但不能混写成同一条 VA→PA 翻译。

## 2. 内核已有一段 RAM 线性映射

在本文讨论的通常 arm64 Linux 配置中，内核为可直接映射的 RAM 建立线性映射（linear map，也叫 direct map）。对这段映射中的内存，CPU VA 与 PA 有固定的对应关系；具体偏移和布局由内核配置及启动时的地址布局决定，不必是某个固定常数。普通内核 RAM 通常使用 Normal、Write-Back cacheable 属性。[Linux arm64 内存布局](https://docs.kernel.org/arch/arm64/memory.html)、[Arm 内存属性说明](https://documentation-service.arm.com/static/63a43e333f28e5456434e18b)

用虚构地址表示：

```text
已建立的内核线性映射

VA 0xffff....1000 ──> PA 0x8100_1000  [Normal, cacheable]
VA 0xffff....2000 ──> PA 0x8100_2000  [Normal, cacheable]
VA 0xffff....3000 ──> PA 0x8100_3000  [Normal, cacheable]
```

它是内核访问普通 RAM 的基础设施。`kmalloc()` 从中分配一个对象时，通常是在这些已有页里选出可用的物理内存，再返回它在线性映射中的 VA；**一般不需要为每个小对象单独选一段新 VA 并建立新页表映射**。

这并不意味着每次 `kmalloc()` 返回相同地址。返回值随实际分配到的对象和物理页而变；固定的是“这类 RAM 的 PA 如何在线性映射中找到对应 VA”。

## 3. `kmalloc()`：物理连续，通常沿用线性映射

典型调用：

```c
void *p = kmalloc(size, GFP_KERNEL);
/* 使用 p */
kfree(p);
```

小对象通常来自 slab/SLUB 管理的页：内核先取得物理页，再把页切成多个对象交给不同的调用者。较大的 `kmalloc()` 请求需要取得足够大的连续物理区域，因此分配大小和成功概率受物理内存碎片等因素影响。`kmalloc()` 返回的是 CPU VA；在这里讨论的普通 arm64 RAM 情况下，它通常落在线性映射中。[Linux 内存分配指南](https://docs.kernel.org/core-api/memory-allocation.html)、[Linux DMA API 对 `kmalloc` 物理连续性的要求](https://docs.kernel.org/core-api/dma-api.html)

```text
kmalloc 返回的 CPU VA：      [对象 A ───────────────]
                                │ 线性映射
物理内存：                 [同一段连续的 PA ───────]
```

**物理连续是分配结果的性质**，不意味着分配器创建了一个新的地址空间。在kmalloc中，返回的就是系统初始化时建立的那一段RAM线性映射中的对应映射段，而不是重新动态地在内核地址空间中寻找一段va。

需要注意两个边界：

- `kmalloc()` 的物理连续性不保证设备一定能寻址这段 PA。DMA mask、`dma-ranges`、IOMMU、SWIOTLB 等仍可能参与。
- 物理连续也不自动处理非一致性设备的 CPU cache。驱动仍须使用 DMA API 建立映射并遵守所有权和同步规则。

## 4. `vmalloc()`：新建连续 VA，物理页可以分散

典型调用：

```c
void *p = vmalloc(size);
/* 使用 p */
vfree(p);
```

`vmalloc()` 取得若干物理页，在内核 **vmalloc 区域** 找到一段连续 VA，再建立 VA→各物理页的页表映射。它保证调用者看到的虚拟区间连续，**不保证物理页连续**；这些页偶然相邻也不能作为接口承诺。[Linux 内存分配指南](https://docs.kernel.org/core-api/memory-allocation.html)、[Linux `vmalloc()` API](https://docs.kernel.org/core-api/mm-api.html)

```text
vmalloc 返回的连续 CPU VA
VA V+0x0000 ──> PA 0x8100_1000
VA V+0x1000 ──> PA 0x9234_5000
VA V+0x2000 ──> PA 0x8400_7000
VA V+0x3000 ──> PA 0xa100_2000
```

CPU 逐字节访问 `p` 时，页表让这些分散的物理页表现为连续数组。这适合较大的内核缓冲区，因为不必取得一整块连续物理页；代价是要管理独立的 VA 区间和页表，通常也有更多 TLB/映射开销。

**这里的“重新映射”并不表示把物理页搬走。** 它只是新增一组 CPU 页表项，使新 VA 指向已经取得的物理页。底层普通 RAM 可能仍有其线性映射别名。普通 `vmalloc()` 映射通常仍是可缓存的 Normal RAM；`vmalloc()` 本身不意味着 NC。

## 5. 两种分配方式的对照

| 问题 | `kmalloc()` | `vmalloc()` |
|---|---|---|
| 返回值 | 内核 CPU VA | 内核 CPU VA |
| 对调用者的 VA | 连续 | 连续 |
| 底层物理页 | 分配范围内连续 | 不保证连续 |
| 常见 CPU 映射 | 使用已有 RAM 线性映射 | 在 vmalloc 区域建立新映射 |
| 常见用途 | 小对象、需要物理连续的缓冲区 | 较大的纯 CPU 缓冲区、无需连续 PA 的数据 |
| 释放 | `kfree()` | `vfree()` |
| 直接交给 `dma_map_single()` | 合适的候选缓冲区，仍须满足 DMA API 条件 | **不能直接传入 vmalloc 指针** |

`kvmalloc()` 会先尝试 `kmalloc()`，再可能退到 `vmalloc()`。所以调用者不能假设它返回的内存物理连续，应使用 `kvfree()` 释放。[Linux 内存分配指南](https://docs.kernel.org/core-api/memory-allocation.html)

### 与用户态 `malloc()` 的差别

用户态 `malloc()` 可以通过 `brk`、`mmap` 或分配器自身的内存池取得地址，不应把其行为套到内核 `kmalloc()` 上。`kmalloc()` 通常返回已有内核线性映射中的对象地址；`vmalloc()` 才明确在内核的另一个 VA 区域组织连续虚拟空间。

另外，用户进程的 **VMA** 通常指 `mm_struct` 中描述用户地址区间的 `vm_area_struct`。内核 `vmalloc` 使用 vmap/vmalloc 区域的管理结构；不能仅因为名字相似，就说 `kmalloc` 和 `vmalloc` 分别创建了一个“用户态 VMA”。

## 6. 为什么 DMA 会让这件事看起来复杂

一个设备的 DMA 是否与 CPU cache 一致，由设备接口、互联和平台实现共同决定。设备能发起 DMA，和它是否能侦听/参与 CPU cache 一致性协议是两个独立能力。IOMMU 解决 DMA 地址翻译与访问控制，也不会单凭自己的存在让非一致性设备自动看到 CPU cache 中的脏数据。[Linux DMA API 指南](https://docs.kernel.org/core-api/dma-api-howto.html)

假设非一致性设备要读取一段普通的 `kmalloc` 缓冲区：

```text
CPU 在 cacheable 线性映射中写入 buffer
               ↓
新数据可能仍在 CPU cache，DRAM 尚未更新
               ↓
设备绕过 CPU cache 读取 DRAM
               ↓
若未做 cache 维护，可能读到旧数据
```

反方向上，设备写入 DRAM 后，CPU cache 里也可能保留旧副本。驱动不能直接把 CPU VA 写给设备；它应调用 DMA API 取得 `dma_addr_t`，并让 DMA API 根据设备的一致性属性完成所需的 cache 维护和 DMA 地址映射。

### 6.1 一次传输：streaming DMA

对已有的普通内核缓冲区，典型流程是：

```c
void *buf = kmalloc(size, GFP_KERNEL);
dma_addr_t dma = dma_map_single(dev, buf, size, DMA_TO_DEVICE);

if (dma_mapping_error(dev, dma)) {
    kfree(buf);
    return -EIO;
}

/* 把 dma 写给设备，等待设备完成；映射期间遵守 DMA 所有权规则 */

dma_unmap_single(dev, dma, size, DMA_TO_DEVICE);
kfree(buf);
```

这是示意流程；实际驱动还要处理设备启动、完成通知和错误路径。对重复使用的映射，CPU 与设备每次交接所有权时使用 `dma_sync_single_for_device()` / `dma_sync_single_for_cpu()`。非一致性 arm64 平台的同步实现会进行相应的 cache clean/invalidate。**普通的 cacheable 内核线性映射继续存在；每次交接时显式同步。**[Linux DMA API 指南](https://docs.kernel.org/core-api/dma-api-howto.html)、[arm64 DMA cache 同步实现](https://github.com/torvalds/linux/blob/master/arch/arm64/mm/dma-mapping.c)

`vmalloc()` 返回的指针不能直接传给 `dma_map_single()`。它背后可能是分散物理页，而该接口要求适合单段映射的内核缓冲区。需要分散页参与 DMA 时，应按具体 API 使用页或 scatter-gather 等机制；即使有 IOMMU，也不能仅因 IOMMU 能拼接 IOVA，就把 `vmalloc` 指针当作合法的 `dma_map_single()` 输入。[Linux DMA API：哪些内存可用于 DMA](https://docs.kernel.org/core-api/dma-api-howto.html#what-memory-is-dma-able)

### 6.2 长期共享：`dma_alloc_coherent()` 与 Normal NC

驱动也可以要求一块“CPU 与设备持续共享、无需每次 手动cache clean/invalidate”的 DMA 内存：

```c
dma_addr_t dma_handle;
void *cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

/* CPU 通过 cpu_addr 访问；设备通过 dma_handle 访问 */

dma_free_coherent(dev, size, cpu_addr, dma_handle);
```

这里的 **coherent 是 DMA API 对返回这块内存的使用承诺**，不等于设备硬件原本就具备 cache snooping。对于不支持硬件一致性的设备，arm64 上的常见实现会给 CPU 创建全新的 Normal NC 映射，使 CPU 通过返回的 `cpu_addr` 访问时不在通常的 Write-Back cache 中留下该缓冲区数据；设备则通过 `dma_handle` 访问同一批物理页。[Linux DMA API](https://docs.kernel.org/core-api/dma-api.html)、[arm64 `pgprot_dmacoherent`](https://github.com/torvalds/linux/blob/master/arch/arm64/include/asm/pgtable.h)

```text
同一批物理页 P

CPU 常规线性映射 VA1 ──> P  [Normal, cacheable]
                                 ▲
CPU DMA 返回指针 VA2 ───> P  [Normal NC]
                                 ▲
设备 DMA 地址 D ── bus/IOMMU ────┘
```

这个图里：

- `VA1`、`VA2` 都是 **CPU VA**，页表项可以指向相同的 PA；`D` 是设备使用的 DMA 地址。
- 两个 CPU VA 不意味着存在“两套彼此独立的内核页表”或“两组物理页”；关键是不同虚拟映射可以携带不同的内存属性。
- 这个例子里的 `VA2` 是 DMA 分配器借助 `vmap` 类机制建立的特殊映射，**不是说调用者先 `vmalloc()` 了一块 DMA buffer**。即使 VA2 位于 vmalloc/vmap 区域，底层物理页在这条实现路径中仍可连续。
- `dma_alloc_coherent()` 的 CPU 指针由 DMA API 选择，**不能通过 `kmalloc()` 加一次普通 `vmap()` 自行替代**。

在一条 arm64 常见的 DMA direct 路径中，内核先取得物理页，调用 `arch_dma_prep_coherent()` 清理线性映射别名中的脏 cache line，再用 `dma_common_contiguous_remap()` 创建带 `dma_pgprot()` 属性的 CPU 映射。arm64 的 `pgprot_dmacoherent()` 把它设为 `MT_NORMAL_NC`。这些是具体实现路径，不是保证所有平台、配置和 DMA 分配都走同一分支。[DMA direct 分配实现](https://github.com/torvalds/linux/blob/master/kernel/dma/direct.c)、[DMA remap 实现](https://github.com/torvalds/linux/blob/master/kernel/dma/remap.c)、[arm64 页表属性](https://github.com/torvalds/linux/blob/master/arch/arm64/include/asm/pgtable.h)

## 7. 关于 VA1、VA2 和“访问同步”的精确结论

**VA1 仍可能有有效的 cacheable 映射，但驱动应使用 DMA API 返回的 VA2。** 内核不需要为了这块 DMA 内存把整个 RAM 线性映射改成 NC。把 VA1 当作另一个入口随意读写，会混用 cacheable 与 NC 属性，可能破坏这块缓冲区的可见性；Arm 文档也强烈建议同一物理位置的多个 VA 使用兼容的内存属性。[Arm 内存别名说明](https://documentation-service.arm.com/static/63a43e333f28e5456434e18b)

Normal NC 的作用是控制 **CPU 通过 VA2 的访问是否进入通常的缓存路径**。它本身不分配物理页、不生成 DMA 地址，也不是“自动通知设备”的同步命令。CPU 和设备之间仍需按驱动协议确定何时生产/消费数据，必要时使用内存屏障；DMA API 明确指出，coherent 内存也不能省略有关写入顺序的屏障。[Linux DMA API 指南：coherent 映射与屏障](https://docs.kernel.org/core-api/dma-api-howto.html#types-of-dma-mappings)

**`VA1` 不应因某个 DMA 请求被调用者擅自改写。** 内核线性映射服务整个系统；特殊 DMA 分配通过 DMA API 管理物理页、CPU 映射属性和设备 DMA 地址。分配器负责防止物理页重复分配；DMA API 负责设备可达性与 cache 管理；驱动负责按顺序交接缓冲区所有权。

## 8. 选用时的快速判断

| 需求 | 常见选择 | 原因 |
|---|---|---|
| 普通小对象 | `kmalloc()` / `kzalloc()` | 分配和访问直接，物理连续 |
| 较大、主要由 CPU 使用的连续数组 | `vmalloc()` / `vzalloc()` | 无需取得大块连续 PA |
| 已有普通缓冲区做一次或多次 DMA | 合适的 buffer + `dma_map_*()` / `dma_sync_*()` | DMA API 处理地址和所有权切换 |
| CPU 与非一致性设备持续共享描述符等 | `dma_alloc_coherent()` / DMA pool | API 提供可持续共享的 CPU 映射与 DMA 地址 |
| 物理页分散但设备要 DMA | 页、scatter-gather 或专门的 DMA 非连续分配接口 | 按页建立 DMA 映射，不能直接传 `vmalloc` VA |

最后用一句话收束：**`kmalloc` 与 `vmalloc` 比的是物理连续性和 CPU VA 的组织方式；Normal NC 说明同一批物理页还可以有专门的 CPU 映射属性；设备能否安全 DMA，最终要按 DMA API 的契约处理。**

## 延伸阅读

- [DMA——直接内存访问](/操作系统/2026/09/13/DMA-直接内存访问/)
- [SoC系统互联——Fabric、DMA与PCIe Root Complex](/操作系统/2026/09/25/SoC系统互联-Fabric-DMA与PCIe-Root-Complex/)
- [设备树关键概念解析](/操作系统/2026/09/30/设备树关键概念解析/)
