---
layout:       post
title:        "TCG变量总览——前端、IR与TCGContext"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 虚拟化
tags:
    - "QEMU"
    - "TCG"
mermaid:      true
---
读 TCG 前端时，经常会同时遇到 `TCGv`、`TCGTemp`、`TCGType`、`TCGTempKind` 和 `tcg_env`。这些名字都在谈“变量”，但各自负责的事情不同。

可以先把它们放在同一张图里：前端拿着一个带类型的句柄，告诉 TCG 要生成什么操作；TCG 在自己的工作区里记录变量信息和 IR；后端再决定这些值在生成的代码中怎样计算、怎样存放。


<div class="mermaid tex2jax_ignore">
flowchart LR
    F[guest 前端 trans 函数] --&gt; H[TCGv_xxx 变量句柄]
    H --&gt; G[tcg_gen_xxx 生成操作]
    H --&gt; T[TCGTemp 变量描述]
    G --&gt; O[TCGOp 指令与操作数]
    O --&gt; T
    subgraph C[TCGContext 翻译工作区]
        T
        O
    end
    C --&gt; B[优化、寄存器分配、生成 host 代码]
</div>


建议按下面的顺序读这一组笔记：

1. 本篇：先知道前端、IR 和翻译工作区怎样接起来。
2. [TCG变量的类型——TCGv句柄与TCGType](/虚拟化/2026/10/15/TCG变量的类型-TCGv句柄与TCGType/)：分清句柄类型、IR 类型，以及为什么还要有 `TCGv`。
3. [TCG变量的生存期与存放约束——TCGTempKind](/虚拟化/2026/10/09/TCG变量的生存期与存放约束-TCGTempKind/)：理解 `GLOBAL`、`FIXED`、`CONST` 各自服务什么场景。
4. [tcg_env——从RISC-V状态到host寄存器](/虚拟化/2026/10/09/tcg_env-从RISC-V状态到host寄存器/)：沿源码找到 `CPURISCVState *` 怎样进入生成的代码。

## 先分清“正在翻译”与“正在执行”

假设前端要表达“把 x6 的值加 1，写入 x5”。在 `TARGET_LONG_BITS == 64` 的构建中，可以用下面这段简化示意理解：

```c
TCGv t = tcg_temp_new();
tcg_gen_addi_tl(t, cpu_gpr[6], 1);
tcg_gen_mov_tl(cpu_gpr[5], t);
```

这段 C 代码在翻译时运行。它是在描述将来执行的操作，此时并没有通过这两个 `tcg_gen_*()` 调用算出 guest 的 x5。

`t` 是新建 IR 变量的句柄，`cpu_gpr[6]` 是表示 guest x6 的 IR 变量句柄。立即数 `1` 则是翻译时已经知道的 C 整数，生成器可以把它包装成 TCG 常量操作数。

忽略后续优化，得到的 IR 大致是：

```text
add_i64 t, x6, $1
mov_i64 x5, t
```

后端可能合并掉这个临时变量和 `mov`，也可能因为寄存器分配而产生 load/store。前端表达的是 guest 指令的语义，最终生成几条 host 指令，要由后续处理决定。

等 TB 的 host 机器码真正执行时，才会读取当时的 x6，完成加法，并按 TCG 的同步规则维护 x5 的状态。编译阶段的 `TCGTemp` 并不是一个装着 guest 当前寄存器值的运行时盒子。

## TCGTemp 是变量的说明卡

`TCGTemp` 记录 TCG 处理这个变量时需要知道的信息。只保留几个核心字段，可以写成：

```c
/* 示意：省略位域宽度和其他字段。 */
typedef struct TCGTemp {
    TCGType base_type;
    TCGType type;
    TCGTempKind kind;
    TCGTempVal val_type;
    TCGReg reg;
    int64_t val;
    struct TCGTemp *mem_base;
    intptr_t mem_offset;
    const char *name;
} TCGTemp;
```

这些字段分别回答不同的问题：

| 字段 | 可以怎样理解 |
| --- | --- |
| `base_type` | 这个变量整体表示什么数据，比如一个 I128 整数 |
| `type` | 后端处理当前这一部分时使用什么类型；I128 拆分后可以是 I64 |
| `kind` | 值需要保留到哪里，有没有固定寄存器或常量约束 |
| `val_type` | 在代码生成的当前处理位置，这个值被描述为寄存器值、内存值、已知常量还是已死 |
| `reg` | 当前分配的，或者事先固定的 host 寄存器 |
| `val` | 编译器已知的常量值等信息，不是所有变量的运行时当前值 |
| `mem_base`、`mem_offset` | 有内存位置时，用哪个基址和偏移定位 |
| `name` | 调试输出中的名字，例如 `env`、`pc` |

这里特别容易把 `kind` 和 `val_type` 混起来。guest x5 可以一直是 `TEMP_GLOBAL`，但生成代码时，它的值可能先从内存读入寄存器，再被写回。位置状态发生变化，不会让它因此变成 `TEMP_FIXED`。

## TCGContext 把这些说明卡和 IR 放在一起

`TCGContext` 中直接包含一个数组：

```c
int nb_globals;
int nb_temps;

TCGTemp temps[TCG_MAX_TEMPS]; /* globals first, temps after */
QTAILQ_HEAD(, TCGOp) ops, free_ops;
```

也就是说，`TCGTemp` 本体就在 `TCGContext.temps[]` 里。`ops` 是当前翻译过程中的 IR 操作队列；每条操作又引用自己使用的变量。

数组的有效部分按下面的方式组织：

```text
temps[0 ... nb_globals - 1]
    预先建立的全局描述，包括 TEMP_GLOBAL 和 TEMP_FIXED

temps[nb_globals ... nb_temps - 1]
    当前 TB 翻译过程中创建的临时变量、常量等

temps[nb_temps ... TCG_MAX_TEMPS - 1]
    尚未使用的槽位
```

所以 `nb_temps` 包含前面的 globals；不能把它理解成“仅普通临时变量数量”。一个 I128 还可能占用多个相邻 `TCGTemp` 槽位，槽位数也不必等于前端逻辑变量数。

`TCGv_xxx` 句柄让前端引用这些描述符，而不用直接操作 `TCGTemp` 的字段。当前实现把句柄编码为 `TCGTemp` 相对 `tcg_ctx` 的字节偏移，细节见 [TCG变量的类型——TCGv句柄与TCGType](/虚拟化/2026/10/15/TCG变量的类型-TCGv句柄与TCGType/)。

到了 `TCGOp.args[]` 这一层，对于变量操作数，当前实现又把对应的 `TCGTemp *` 编码成 `TCGArg`。因此，前端句柄的偏移编码与 IR 参数中的编码需要区分；`args[]` 还可能容纳立即数等其他参数，要按具体 opcode 来解释。

## 工作区可以重用，生成的 TB 代码另有存放位置

开始翻译下一块 TB 时，`tcg_func_start()` 会执行：

```c
tcg_pool_reset(s);
s->nb_temps = s->nb_globals;
/* 清理常量表、重置操作队列等。 */
```

前面的全局描述继续保留，当前 TB 的临时变量和 IR 工作区则可以重新使用。

这不会抹掉 guest 的寄存器值，因为运行时状态保存在 `CPUArchState` 等位置；也不会抹掉已经生成的 TB 机器码，因为机器码存放在代码缓存中，由 TB 的 `tc.ptr` 等信息定位。

可以把 `TCGContext` 想成翻译时使用的工作台。译完一块 TB，就把成品放到代码缓存中，清理工作台上的临时记录，再翻译下一块。

`tcg_ctx` 是线程局部的上下文指针。当前 system-mode 为注册的执行/翻译线程创建上下文，并从初始化上下文复制全局描述；user-mode 的 `tcg_register_thread()` 则让线程使用共享的初始化上下文。因而不宜简单说成“每个 vCPU 永远独占一个 TCGContext”。更完整的上下文说明见 [TCG翻译上下文——TCGContext](/虚拟化/2026/10/07/TCG翻译上下文-TCGContext/)。

## 源码核对范围

这一组笔记核对于 2026-10-09，源码目录为 `/home/holo/ospp-qemu-optimisation/qemu`，当时 HEAD 为 `f9587d4045c6`。代码片段中的省略与简化会明确标出，函数名可用于源码更新后重新定位。

- [TCGTemp 定义](/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:270)：变量描述符。
- [TCGContext 的 temps 和 ops](/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:419)：变量与 IR 的存放位置。
- [句柄和 IR 参数转换](/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:470)：`temp_arg()`、`tcgv_i32_temp()`、`temp_tcgv_i32()`。
- [tcg_gen_op3](/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg-op.c:57)：怎样填写一条操作的类型和参数。
- [tcg_func_start](/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg.c:1936)：开始下一块 TB 时重置什么。
- [tcg_register_thread](/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg.c:1261)：线程如何取得上下文。

读完后可以回到 QEMU decodetree：insn32，沿着 `trans_* → tcg_gen_* → TCGOp` 再看一次。此时就能分清：解码器给出指令参数，前端描述操作语义，而变量类型、保存约束与实际存放位置由 TCG 逐层处理。
