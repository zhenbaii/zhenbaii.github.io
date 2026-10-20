---
layout:       post
title:        "RISC-V的XL与AXL"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 虚拟化
tags:
    - "TCG"
---
在 QEMU 的 RISC-V TCG 代码里，`XL` 和 `AXL` 都会出现在 TB flags 中。它们都在描述“宽度”，但回答的是两个不同的问题：

> `XL` 决定当前指令按多少位的整数 ISA 语义执行；`AXL` 决定显式 load/store 的地址按多少位来解释。

可以先简单记成：**XL 管执行，AXL 管访存地址。**

需要先说明的是，`AXL` 不是 RISC-V 规范里的一只寄存器，也不是 `mstatus` 中的字段。它是 QEMU 对 **address XLEN** 这一派生状态的命名。当前源码的 `DisasContext` 中相应成员叫 `address_xl`。

### XL：当前执行环境的 XLEN

RISC-V 里的 XLEN 首先定义的是整数寄存器宽度，也由此确定基础整数指令的运算宽度。QEMU 要翻译一段指令时，必须知道当前特权级对应的 XLEN：

```text
M-mode → misa.MXL
S-mode → mstatus.SXL
U-mode → mstatus.UXL
```

因此，同样一条 `add`：

```text
XL=32 → 产生 32 位整数结果
XL=64 → 产生 64 位整数结果
```

QEMU 把这个值保存在 `env->xl`，生成 TB flags 时再写入 `TB_FLAGS.XL`。翻译器由此决定寄存器运算、移位宽度、PC 处理以及与 XLEN 有关的指令合法性。

这里说“整数寄存器宽度”，并不等于处理器里的所有寄存器都会一起变宽。浮点寄存器由 FLEN 描述，向量寄存器由 VLEN 描述；CSR 的可见宽度和字段布局则由各自的架构定义决定。比如 RV32D 完全可以是 `XLEN=32, FLEN=64`。

### AXL：访存地址采用的有效 XLEN

大多数时候，处理器以当前特权级的身份访问内存，所以 AXL 与 XL 相同。特殊情况发生在 M-mode 开启 `mstatus.MPRV` 时。

QEMU 源码里的推导过程很直白：

```text
XL = 当前特权级对应的 XLEN

if 当前特权级 == M && MPRV == 1:
    address_mode = MPP
else:
    address_mode = 当前特权级

AXL = address_mode 对应的 XLEN
```

对应到当前 QEMU 源码，`cpu_address_mode()` 先算出访存采用的有效特权级，`cpu_address_xl()` 再取得该特权级的 XLEN。TB 建立时，这个值被写入 `TB_FLAGS.AXL`，翻译开始时再取到 `DisasContext.address_xl` 中。

常见组合如下：

| 当前模式 | MPRV | MPP | XL | AXL |
| --- | --- | --- | --- | --- |
| M | 0 | 任意 | MXLEN | MXLEN |
| S | 任意 | 任意 | SXLEN | SXLEN |
| U | 任意 | 任意 | UXLEN | UXLEN |
| M | 1 | S | MXLEN | SXLEN |
| M | 1 | U | MXLEN | UXLEN |

所以只有在 M-mode 使用 MPRV，且 `MPP` 对应的 XLEN 与 MXLEN 不同时，才会明显看到 `XL != AXL`。

### 一个 XL=64、AXL=32 的例子

假设机器的配置为：

```text
MXLEN = 64
SXLEN = 64
UXLEN = 32
```

当前 CPU 处于 M-mode，并且：

```text
MPRV = 1
MPP  = U
```

那么：

```text
XL  = MXLEN = 64
AXL = UXLEN = 32
```

此时执行：

```asm
ld t0, 0(a0)
```

要分三层理解：

1. `ld` 仍是一条 RV64 指令，因为执行环境的 `XL=64`；
2. 它仍然装载 64 位数据，并把结果写入 64 位整数寄存器；
3. 用来访问内存的地址按 `AXL=32` 处理，并以 U-mode 的地址转换和权限环境发起访问。

因此，AXL 说的是**地址宽度**，不是 load/store 传输的数据宽度。`lb`、`lw`、`ld` 装多少数据，仍由具体指令决定。

这也解释了为什么不能只在 TB flags 里记录 `PRIV=M`。仅知道当前在 M-mode，并不能判断一条 load/store 是使用 M-mode 的 64 位地址环境，还是借助 MPRV 使用 U-mode 的 32 位地址环境。

### XL、AXL、PRIV 和 MEM_IDX 各管什么

这几个字段放在一起看会更清楚：

| 状态 | 回答的问题 |
| --- | --- |
| `PRIV` | 当前代码实际在哪个特权级执行？ |
| `XL` | 当前指令按 RV32 还是 RV64 语义执行？ |
| `AXL` | 显式 load/store 的地址按 32 位还是 64 位处理？ |
| `MEM_IDX` | 这次访存使用哪套 MMU、虚拟化和权限上下文？ |

AXL 只描述地址的宽度维度，并不能单独代表完整的访存环境。以哪个特权级检查权限、是否处于虚拟化状态等信息，仍需要 `MEM_IDX` 等状态共同表达。

### 为什么它们必须进入 TB flags

TB 是 QEMU 针对某一组 CPU 状态翻译出来的代码缓存。同一地址上的相同机器码，在不同 XL 下可能具有不同的运算宽度；即使 XL 不变，不同 AXL 也可能让 load/store 生成不同的地址处理代码。

可以把 TB 的适用条件理解为：

```text
相同 PC
+ 相同 PRIV
+ 相同 XL
+ 相同 AXL
+ 其他会影响翻译结果的状态
```

只有这些条件相符，已有 TB 才能安全复用。若 M-mode 修改了 `MPRV`、`MPP` 或影响各级 XLEN 的字段，QEMU 需要结束当前 TB，下一次再按新的 flags 查找或生成 TB。否则就可能拿着一份“64 位地址版本”的翻译结果，去执行本应使用 32 位地址语义的访存。

### 在 QEMU 源码中的位置

当前 QEMU 源码的关系可以归纳为：

```text
cpu_get_xl(env, env->priv)
        │
        └─→ env->xl ─→ TB_FLAGS.XL ─→ DisasContext.xl

cpu_address_mode(env)
        │
        └─→ cpu_address_xl(env)
                  │
                  └─→ TB_FLAGS.AXL ─→ DisasContext.address_xl
```

主要定义分布在：

- `target/riscv/cpu.h`：`cpu_get_xl()`、`cpu_address_mode()`、`cpu_address_xl()` 以及 TB flags 定义；
- `target/riscv/tcg/tcg-cpu.c`：把 `XL` 和 `AXL` 写入 TB flags；
- `target/riscv/tcg/translate.c`：从 TB flags 初始化翻译上下文。

QEMU 9.2 的 `get_address()` 会直接根据 address XLEN 对 RV32 地址做零扩展。较新的 master 分支经历过 pointer masking 相关重构，地址宽度在翻译器里的具体消费位置有所变化，但 `XL` 与 `AXL` 分开记录这一设计仍保留着。本文核对的 master 版本是提交 `c1c18d1e640b64292859ce9f30f3c344edfb0294`（2026-09-19）；读源码时最好先确认所看的 QEMU 版本，不能只凭某一版的函数位置推断整个机制。

### 最后再压缩成一句话

```text
XL  = “我正在按多少位的 ISA 执行这条指令？”
AXL = “这条指令访问内存时，地址按多少位来解释？”
```

通常二者相等；M-mode 通过 MPRV 借用低特权级访存环境时，二者才可能分开。MPRV 为什么存在、具体修改哪些访存属性，可以接着看 MPRV机制。

### 参考资料

- [RISC-V Privileged Architecture：mstatus.MPRV](https://docs.riscv.org/reference/isa/v20260120/priv/machine.html#machine-status-mstatus-and-mstatush-registers)
- [QEMU `target/riscv/cpu.h`](https://gitlab.com/qemu-project/qemu/-/blob/master/target/riscv/cpu.h)
- [QEMU `target/riscv/tcg/tcg-cpu.c`](https://gitlab.com/qemu-project/qemu/-/blob/master/target/riscv/tcg/tcg-cpu.c)
- [QEMU `target/riscv/tcg/translate.c`](https://gitlab.com/qemu-project/qemu/-/blob/master/target/riscv/tcg/translate.c)
- MPRV机制
- [Qemu的TCG源码梳理](/虚拟化/2026/09/17/Qemu的TCG源码梳理/)
- [TB的查找、链接与失效](/虚拟化/2026/09/20/TB的查找-链接与失效/)
