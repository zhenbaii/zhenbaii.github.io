---
layout:       post
title:        "TCG变量的生存期与存放约束——TCGTempKind"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 虚拟化
tags:
    - "QEMU"
    - "TCG"
---
`TCGType` 说明变量表示多少位、是整数还是向量；`TCGTempKind` 则说明这个值需要保留到哪里，以及有没有特殊的存放约束。读前端时，把这两个问题分开，就不会因为“放进了寄存器”而误以为变量类型发生了变化。

```c
typedef enum TCGTempKind {
    TEMP_EBB,
    TEMP_TB,
    TEMP_GLOBAL,
    TEMP_FIXED,
    TEMP_CONST,
} TCGTempKind;
```

这五种分类并不是简单地按生存期从短到长排列。前三种主要区分值需要保留的范围，后两种还强调固定寄存器和固定常量这两种约束。

| kind          | 通俗理解                           | 典型场景                   |
| ------------- | ------------------------------ | ---------------------- |
| `TEMP_EBB`    | 离开当前 EBB 后，这个旧值可以不要了           | 局部计算的中间值               |
| `TEMP_TB`     | 同一个 TB 内可以跨 EBB 使用，离开 TB 后可以丢弃 | TB 内跨分支使用的中间值          |
| `TEMP_GLOBAL` | 下一个 TB 还需要的状态，要有长期保存的位置        | guest GPR、PC 等 CPU 状态  |
| `TEMP_FIXED`  | 这个变量在生成的代码中固定占用指定 host 寄存器     | `tcg_env`              |
| `TEMP_CONST`  | 值已经确定，不能再被赋成别的值                | `tcg_constant_i64(42)` |

这里说“死亡”，是说旧值不再需要保留，不是要求执行 C 的 `free()`。同样，“在整个 TB 内有效”也不代表从 TB 开头就自动有初值，使用前仍然需要定义它。

## TEMP_GLOBAL：给跨 TB 的 CPU 状态安排一个长期位置

guest 执行完一个 TB 后，GPR 的值当然不能消失。下一块 TB、异常处理代码、某些 helper 都可能需要读取它。

TCG 把这类状态建模成 `TEMP_GLOBAL`。它有约定的内存位置，常见情况是 `CPUArchState` 中的某个字段。例如 RISC-V：

```c
cpu_gpr[i] = tcg_global_mem_new(tcg_env,
    offsetof(CPURISCVState, gpr[i]) + field_offset,
    riscv_int_regnames[i]);
```

这相当于告诉 TCG：“这个变量代表 guest 的第 i 个 GPR；它在 CPU 状态结构中的位置，是 `env + 这个偏移`。”`field_offset` 用于特定 host 字节序与 guest 位宽组合下定位字段中的有效部分，通常的小端场景中为 0。

TCG 可以把这个变量的值临时装到 host 寄存器中，连续做多次运算，再在需要同步的地方写回。`GLOBAL` 并不要求每次使用都重新读内存，也不意味着某个 host 寄存器从此永久属于它。

例如，生成的代码可能表现为下面这样。这里只表达存放位置变化，不对应固定的后端指令序列：

```text
读取 env->gpr[5] 到某个 host 寄存器
在该 host 寄存器中连续更新 x5
在需要同步的边界，把结果写回 env->gpr[5]
```

在整个过程中，表示 x5 的变量仍然是 `TEMP_GLOBAL`。

“global”也不意味着所有 guest CPU 共享同一份 x5。每个 vCPU 有自己的 `CPURISCVState`，运行时的 `env` 指向谁，就访问谁的字段。

在当前 RISC-V 前端中，x1 到 x31 采用这种 global 映射；x0 则特殊处理，读取它时可以直接返回表示零的句柄。不能仅凭 `cpu_gpr[32]` 的数组大小，就认定 x0 也创建了一个普通可写 global。

状态同步也需要看具体边界。正常 TB 退出需要满足 CPU 状态约定；helper 默认可能读写 globals，但声明的调用 flags 可以允许省略部分同步。异常还涉及状态恢复机制，不能概括成“每条可能异常的指令前，把所有 CPU 字段全部写回”。

## TEMP_FIXED：为频繁使用的基址预留一个 host 寄存器

可以把 `TEMP_FIXED` 理解为：这个 TCG IR 变量在生成的代码执行期间，会常驻在某个指定的 host 寄存器中，普通寄存器分配器不能随意把它挪走。

`tcg_env` 是最典型的例子。生成的代码经常要访问 `CPUArchState` 中的 GPR、PC、CSR 等状态。如果每次都重新取得结构体地址，会增加额外工作。TCG 因而预留一个 host 寄存器，让它一直保存这个地址，后面的访问就能直接使用“基址 + 字段偏移”。

常驻寄存器的是 `CPUArchState *` 这个指针值，整个 `CPUArchState` 结构体仍在 host 内存中。

创建代码是：

```c
ts = tcg_global_reg_new_internal(s, TCG_TYPE_PTR, TCG_AREG0, "env");
tcg_env = temp_tcgv_ptr(ts);
```

内部会设置：

```c
ts->kind = TEMP_FIXED;
ts->reg = reg;
tcg_regset_set_reg(s->reserved_regs, reg);
```

`reserved_regs` 表示这个 host 寄存器已经有专门用途，不能再随意分给普通临时变量。当前实现还把 fixed 变量作为只读操作数处理，前端不能像给普通临时变量一样，用普通 IR 输出操作去改写 `tcg_env`。

这里的“常驻”说的是 TCG 生成代码的执行约定。它并不意味着 QEMU 从进程启动到退出都独占这只硬件寄存器；进入和退出生成代码、调用 helper 时，还要遵循 host ABI 和寄存器保存恢复规则。

`TEMP_FIXED` 也不是 guest GPR 的默认实现。guest GPR 可以在代码生成过程中获得 host 寄存器，但通常仍属于有内存后备位置的 `TEMP_GLOBAL`。

## 为什么函数叫 global_reg_new，结果却是 TEMP_FIXED

`tcg_global_reg_new_internal` 的名字可以读成：“TCG 内部创建一个放在固定寄存器里的全局变量”。

函数先执行：

```c
TCGTemp *ts = tcg_global_alloc(s);
```

这会把它放进 `TCGContext.temps[]` 前面的 globals 区域，并计入 `nb_globals`。随后函数把 `kind` 设为 `TEMP_FIXED`，补充“必须使用指定寄存器”这一约束。

因此，函数名中的 `global` 是广义的全局变量；枚举中的 `TEMP_GLOBAL` 则专指有内存位置的那一类。两者层次不同，不必逐字相等。

| 创建方式 | 最终 kind | 约定的位置 |
| --- | --- | --- |
| `tcg_global_mem_new_*()` | `TEMP_GLOBAL` | `mem_base + mem_offset` |
| `tcg_global_reg_new_internal()` | `TEMP_FIXED` | 指定的 host `reg` |

两者都会进入 globals 区域。`internal` 表示它是 TCG 内部的辅助实现，这里是一个返回原始 `TCGTemp *` 的 `static` 函数。当前源码的 `_frame` 也通过这个内部函数建立，因而不要把 `FIXED` 理解成只能用于 `env` 的特例。

## TEMP_CONST：这个值就是 42

```c
TCGv_i64 c = tcg_constant_i64(42);
```

这里创建的是 `TEMP_CONST`。翻译时就知道它的值是 42，后续也不能把这个变量赋成 43。

后端可以把它直接编码成机器指令的立即数；如果指令不支持该立即数，也可以先把它装进 host 寄存器。它的值固定，不要求存放位置固定。

当前实现按类型和值在当前 TB 的常量表中查找，重复请求同类型、同数值的标量常量会复用描述符；开始翻译下一块 TB 时会清理这份常量表。

下面这段代码虽然也涉及 42，但含义不同：

```c
TCGv_i64 t = tcg_temp_new_i64();
tcg_gen_movi_i64(t, 42);
```

`t` 是一个普通的 `TEMP_TB` 变量，只是这一刻被赋值为 42。之后还可以再次赋值。优化器或寄存器分配器可能记录“当前值已知为常量”，但这不等于变量的 `kind` 变成了 `TEMP_CONST`。

因此下面两个字段可以同时成立：

```text
kind     = TEMP_TB
val_type = TEMP_VAL_CONST
```

前者是变量的保存规则，后者是代码生成当前所知道的值状态。

## EBB 和 TB 只需先把边界分清

一个 TB 的 IR 可以包含多个 EBB。TCG 的 EBB 沿着条件分支“不跳转、继续往下走”的路径延伸；遇到 label、无条件跳转等边界时结束。

```text
    t = a + b          ┐
    if cond goto L1    │ EBB A
    x = t + 1          │ 条件不成立时，仍能使用 t
    goto L2            ┘

L1:                    ┐
    ...                │ EBB B
                       ┘
L2:
    ...
```

如果 `t` 的旧值只在 A 内被使用，`TEMP_EBB` 就足够。若 L1 中也要读取上面算出的 `t`，这个值要跨 EBB 保留，应使用 `TEMP_TB`。

两种临时值在离开 TB 后都没有自动延续的承诺。若把结果写进某个 global，那么跨 TB 保留下来的是那个 CPU 状态的值。

当前源码的普通 `tcg_temp_new_i32/i64()` 创建 `TEMP_TB`；活跃性分析发现它只在单个 EBB 使用时，可以降为 `TEMP_EBB`。显式 EBB 分配和释放接口位于 TCG 内部头文件，普通 guest 前端通常使用公开的 `tcg_temp_new_*()` 即可。

## 对应源码

版本范围见 [TCG变量总览——前端、IR与TCGContext](/虚拟化/2026/10/09/TCG变量总览-前端-IR与TCGContext/)。

- [TCGTempKind 与 TCGTempVal](/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:246)。
- [global 与 fixed 的创建](/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg.c:1977)：`tcg_global_alloc()` 和 `tcg_global_reg_new_internal()`。
- [固定变量的只读判断](/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:443)：`temp_readonly()`。
- [常量查找与创建](/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg.c:2245)：`tcg_constant_internal()`。
- [TB 临时变量降为 EBB](/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg.c:3813)。
- [EBB 内部接口](/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg-temp-internal.h:28)。
- [RISC-V GPR 映射](/home/holo/ospp-qemu-optimisation/qemu/target/riscv/tcg/translate.c:1520)：注意从 x1 开始创建。

把 `env` 与 guest GPR 放在一起看，二者的区别最直观：`env` 是固定寄存器中的基址；GPR 是通过这个基址定位的 CPU 状态。完整运行链路见 [tcg_env——从RISC-V状态到host寄存器](/虚拟化/2026/10/09/tcg_env-从RISC-V状态到host寄存器/)。
