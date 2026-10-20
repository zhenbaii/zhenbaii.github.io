---
layout:       post
title:        "TCG变量的类型——TCGv句柄与TCGType"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 虚拟化
tags:
    - "TCG"
---
阅读 QEMU TCG 源码时，`TCGv` 很容易让人误以为是“所有 TCG 变量的统一基类”。实际上，它只是一个根据 `TARGET_LONG_BITS` 决定的类型别名。`TCGv_i32`、`TCGv_i64`、`TCGv_ptr` 等类型各自独立表示对应位宽的变量句柄，在 C 语言层面并不存在继承或包含关系。

关于这些句柄最终如何映射到内部的 `TCGTemp` 结构，可以先参考 [TCG变量总览——前端、IR与TCGContext](/虚拟化/2026/10/09/TCG变量总览-前端-IR与TCGContext/)。

## 为什么把句柄定义为结构体指针

在 `include/tcg/tcg.h` 中，各类 TCG 句柄的定义如下：

```c
typedef struct TCGv_i32_d *TCGv_i32;
typedef struct TCGv_i64_d *TCGv_i64;
typedef struct TCGv_i128_d *TCGv_i128;
typedef struct TCGv_ptr_d *TCGv_ptr;
typedef struct TCGv_vec_d *TCGv_vec;
typedef TCGv_ptr TCGv_env;

```

注意这里的 `struct TCGv_xxx_d` 仅作了前置声明，故意没有给出具体的结构体成员定义。C 语言允许声明和传递指向不完整类型（incomplete type）的指针，但由于缺乏内部定义，后续代码无法通过 `v->field` 解引用访问其内容。

这种设计的核心目的是**借助 C 编译器实现静态类型检查**。由于各个结构体名称不同，对应的指针类型在编译器看来互不兼容。一旦前端代码误将 `TCGv_i32` 传给要求 `TCGv_i64` 的接口，编译器就能在编译期直接报错：

```c
TCGv_i32 a = tcg_temp_new_i32();
TCGv_i64 b = tcg_temp_new_i64();

tcg_gen_add_i64(b, a, b);  /* 编译报错：a 的句柄类型不兼容 */

```

如果直接将这些句柄定义为 `typedef uintptr_t ...`，所有句柄在 C 层面都会退化为同一种整数类型，编译器就无法拦截此类不同位宽操作数混用的错误。

因此，定义中的 `*` 只是借用了 C 的类型系统来做类型隔离。`TCGv_i32` 存储的并非真正的 `struct TCGv_i32_d *` 内存地址，更不是指向 guest 寄存器内容的指针。同样，也不能通过 `sizeof(TCGv_i32)` 去推断它所代表的 IR 变量位宽：在常规的 64 位 host 上，该句柄本身占 64 位指针大小，但它表示的 IR 值依然是 32 位。

## 句柄内部实际编码的是偏移量

句柄与内部 `TCGTemp` 指针的核心转换逻辑如下（省略了调试检查）：

```c
static inline TCGv_i32 temp_tcgv_i32(TCGTemp *t)
{
    return (TCGv_i32)((void *)t - (void *)tcg_ctx);
}

static inline TCGTemp *tcgv_i32_temp(TCGv_i32 v)
{
    return (void *)tcg_ctx + (uintptr_t)v;
}

```

对应的换算关系非常直接：

```text
句柄的数值 = TCGTemp 地址 - 当前 TCGContext 地址
TCGTemp 地址 = 当前 TCGContext 地址 + 句柄的数值

```

可以看到，源码利用了 GCC 支持的 `void *` 字节算术运算。`TCGv_xxx` 实际存储的是目标 `TCGTemp` 相对于当前 `TCGContext` 起始地址的**字节偏移量**，而非简单的 `temps[]` 数组下标，也不是 `TCGTemp *` 指针本身。通过上述两个内联函数，即可完成 `TCGTemp *` 与句柄之间的双向转换。

这种偏移量编码还顺带保留了零值作为空句柄：因为第一个 `TCGTemp` 并不位于 `TCGContext` 的零偏移处，合法句柄的数值一定非零。在实际代码中若需访问内部描述符，应始终通过 `tcgv_i32_temp()` 等转换函数获取。

## TCGType：描述 IR 值的数据形态

前端句柄是提供给 C 编译器做类型检查的，而 `TCGTemp` 内部的 `TCGType` 则是 TCG 内部处理变量时使用的类型标签。

|**TCGType**|**含义**|
|---|---|
|`TCG_TYPE_I32`|32 位标量整数|
|`TCG_TYPE_I64`|64 位标量整数|
|`TCG_TYPE_I128`|128 位标量整数|
|`TCG_TYPE_V64`|总宽度 64 位的向量|
|`TCG_TYPE_V128`|总宽度 128 位的向量|
|`TCG_TYPE_V256`|总宽度 256 位的向量|
|`TCG_TYPE_REG`|host 通用寄存器宽度的别名，当前等于 `I64`|
|`TCG_TYPE_PTR`|host 原生指针宽度的别名，按 ABI 对应 `I32` 或 `I64`|

围绕 `TCGType` 的定义，有几个设计细节需要注意：

* **整数类型不区分符号**：`I32`、`I64` 等类型仅表示数据位宽，不携带符号属性。具体按有符号还是无符号解释，完全由作用在其上的 IR 操作决定（例如有符号比较、无符号比较、算术右移等）。
* **向量类型与 `I128` 的区别**：向量名称中的数字代表总位宽。例如一个 `V128` 既可以按 16 个 8 位元素处理，也可以按 4 个 32 位元素处理，具体元素宽度由 vector op 中的 `vece` 等信息指定。而 `I128` 则表示一个整体的 128 位标量整数；两者虽然同占 128 位，但运算语义和后端处理方式并不相同。此外，具体能使用哪些向量类型和操作，还取决于 host 后端的硬件支持能力。
* **别名与枚举总数**：`REG` 和 `PTR` 都是已有枚举值的别名，因此独立的底层数据类型依然是 6 种：

```c
#define TCG_TYPE_COUNT (TCG_TYPE_V256 + 1)  /* 值为 6 */
```

这里的 `TCG_TYPE_COUNT` 仅用于定义数组大小等边界场景，本身并不是第七种合法数据类型。当前源码中 `TCG_TARGET_REG_BITS` 固定为 64，因此 `REG` 恒等于 `I64`；而 `PTR` 则单独依据 `UINTPTR_MAX` 判定，用于表达 C 语言层面的原生指针宽度。

## TCG_TYPE_PTR 相等，不代表 TCGv_ptr 相同

在常见的 64 位指针 ABI 下，以下两个枚举常量是相等的：

```c
TCG_TYPE_PTR == TCG_TYPE_I64
```

但这只是内部枚举值的重合，C 语言层面的两个句柄类型依然彼此独立：

```c
TCGv_ptr  /* struct TCGv_ptr_d * */
TCGv_i64  /* struct TCGv_i64_d * */
```

因此，不能将 `TCGv_ptr` 视为 `TCGv_i64` 的别名。虽然内部枚举将它们归入同一种位宽，但前端句柄依然保留了“该变量为 host 指针”的语义区分。

此外，`PTR` 标签仅保证变量采用 host 指针宽度，并不会记录它具体指向哪种结构体，也不提供指针合法性或目标对象类型的检查。例如 `tcg_env` 之所以能表示 `CPUArchState *`，是因为入口代码确实将该结构体的地址加载到了对应的 host 寄存器中。

至于 `TCGv_env`，它的定义 `typedef TCGv_ptr TCGv_env` 确实只是同一个 C 类型的别名，作用是通过类型名称强调其固定用途。

## 为什么还要设计 TCGv

除了显式标明位宽的句柄外，TCG 还提供了 `TCGv`，用于对应 guest 架构的 `target_ulong` 宽度：

```c
#if TARGET_LONG_BITS == 32
typedef TCGv_i32 TCGv;
#define tcg_temp_new() tcg_temp_new_i32()
#elif TARGET_LONG_BITS == 64
typedef TCGv_i64 TCGv;
#define tcg_temp_new() tcg_temp_new_i64()
#endif
```

当前端代码的语义是“按 guest 自然字长进行运算”时，可以直接使用 `TCGv` 编写：

```c
/* a、b 已经是同样的 target-long 句柄 */
TCGv result = tcg_temp_new();
tcg_gen_add_tl(result, a, b);
```

在 32 位构建下会自动选择 `i32` 版本，64 位构建下则选择 `i64` 版本（函数后缀 `_tl` 即代表 target long）。类似地，`tcg_global_mem_new` 等宏也在同一处完成了位宽分发。如果不引入 `TCGv`，而是全部使用显式的 `TCGv_i32` 和 `TCGv_i64`，那么在 32 位与 64 位共享的前端代码中，每个翻译函数都会充斥着重复的 `#if` 条件编译分支。`TCGv` 的作用正是把“变量位宽随 guest target-long 变化”这一属性直接体现在类型名称上。

在使用 `TCGv` 时，有两点需要注意：

1. **编译期绑定与运行时状态的区别**：`TARGET_LONG_BITS` 是在编译 QEMU 时确定的静态配置。以 RISC-V 为例，不能将其与某个 TB 运行时的 `ctx->xl` 完全等同——在支持较窄执行模式的构建中，前端仍可能使用 `i64` 句柄，再通过截断或扩展操作来表达 32 位运算（详见 [RISC-V的XL与AXL](/虚拟化/2026/09/20/RISC-V的XL与AXL/)）。
2. **多种位宽句柄的并存**：guest 为 64 位架构并不要求所有 IR 变量都采用 `i64`。例如 RISC-V 前端就同时声明了以下几类变量：
```c
static TCGv cpu_gpr[32], cpu_gprh[32], cpu_pc;
static TCGv_i64 cpu_fpr[32];
static TCGv_i32 cpu_vl, cpu_vstart;
```

其中，通用寄存器（GPR）使用 target-long 句柄，浮点寄存器固定使用 64 位存储句柄，而 CPU 状态中的 `vl`、`vstart` 字段本身是 32 位，因此直接使用 `TCGv_i32`。不同位宽的句柄可以根据实际硬件语义在同一前端中共存。

## 句柄类型与 TCGTemp 内部类型的一致性

通过标准 API 创建变量时，外层句柄类型与内部 `TCGTemp` 类型会自动保持配套。例如创建 64 位临时变量：

```c
TCGv_i64 tcg_temp_new_i64(void)
{
    return temp_tcgv_i64(
        tcg_temp_new_internal(TCG_TYPE_I64, TEMP_TB));
}
```

对于常规的 `i64` 标量变量，其对应关系十分直接：

```text
句柄类型：TCGv_i64
base_type：TCG_TYPE_I64
type：TCG_TYPE_I64
```

但在处理特定类型时，内部表示会有所区分：

* **`I128` 的拆分表示**：在当前的 64 位 host 实现中，一个 128 位整数由两个相邻的 `TCGTemp` 组合表示。此时其整体逻辑类型 `base_type` 为 `TCG_TYPE_I128`，但每个子部分的 `type` 则是 `TCG_TYPE_REG`（即 `I64`）。因此，在核对类型一致性时应首先检查逻辑类型 `base_type`，而不能机械地要求句柄名称与每个组成部分的 `type` 完全相同。


* **`TCGv_vec` 的宽度隐藏**：向量句柄有意不在 C 语言类型层面区分 `V64`、`V128` 和 `V256`，统一使用 `TCGv_vec`，具体的向量总宽度则保存在内部描述符中。



对于普通标量而言，如果一个 `TCGv_i64` 句柄指向了 `base_type = I32` 的底层变量，就构成了错误的类型搭配。在 C 语言中进行强制类型转换，仅仅改变了编译器视角下的表面类型，并不会改变底层的 IR 变量属性：

```c
TCGv_i32 src = tcg_temp_new_i32();
/* 假设此前的 IR 已给 src 赋值 */

TCGv_i64 wrong = (TCGv_i64)src;  /* 错误：不会把 32 位值变成 64 位 */
```

句柄本质上只是“某个 IR 变量的带类型编号”。在 C 代码里强转编号的类型，并不会替底层变量完成位宽扩展、截断或计算。真正的数值位宽转换必须通过生成对应的 IR 指令来完成：

```c
TCGv_i64 dst = tcg_temp_new_i64();
tcg_gen_extu_i32_i64(dst, src);  /* 生成零扩展 IR 操作 */
```

## 对应源码

本文涉及的源码版本范围见 [TCG变量总览——前端、IR与TCGContext](/虚拟化/2026/10/09/TCG变量总览-前端-IR与TCGContext/)。

* [TCGType 与句柄设计注释](https://www.google.com/search?q=/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:128)

* [句柄的偏移编码](https://www.google.com/search?q=/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:489)

* [TCGv 与 target-long API](https://www.google.com/search?q=/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg-op.h:34)

* [I128 拆分与临时变量创建](https://www.google.com/search?q=/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg.c:2063)

* [RISC-V 前端的变量声明](https://www.google.com/search?q=/home/holo/ospp-qemu-optimisation/qemu/target/riscv/tcg/translate.c:41)
