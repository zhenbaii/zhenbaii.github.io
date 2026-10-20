---
layout:       post
title:        "tcg_env——从RISC-V状态到host寄存器"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 虚拟化
tags:
    - "QEMU"
    - "TCG"
    - "RISC-V"
mermaid:      true
---
在 RISC-V guest 的翻译中，`tcg_env` 表示当前 vCPU 的 `CPURISCVState` 地址。不过这句话分成两个时刻看，会更准确：翻译时，`tcg_env` 是一个 `TCGv_ptr` 句柄；生成的代码运行时，它对应的 host 寄存器才保存真正的 `CPURISCVState *`。

下面沿着源码把这个地址走过的路接起来。

## 先找到它究竟指向哪一层结构

RISC-V 的头文件中有：

```c
typedef struct CPUArchState CPURISCVState;
```

通用头文件又把同一个 `struct CPUArchState` 命名为 `CPUArchState`。所以在 RISC-V 构建中，`CPURISCVState` 与 `CPUArchState` 是同一个结构体类型的两个名字。

状态结构中保存着 guest 的寄存器等信息，节选如下：

```c
struct CPUArchState {
    uint64_t gpr[32];
    uint64_t gprh[32];
    /* 省略向量寄存器等字段。 */
    uint32_t vl;
    uint32_t vstart;
    /* 省略其他字段。 */
    uint64_t pc;
    /* 省略其他字段。 */
    uint64_t fpr[32];
    /* ... */
};
```

它作为成员嵌在完整 CPU 对象中：

```c
struct ArchCPU {
    CPUState parent_obj;
    CPURISCVState env;
    /* 省略其他字段。 */
    RISCVCPUConfig cfg;
    /* ... */
};
```

RISC-V 的 CPU 声明宏把 `RISCVCPU` 对应到这个 `ArchCPU` 类型。对一个 `RISCVCPU *cpu` 而言，所需地址就是 `&cpu->env`。

所以这里有三个不同的东西：完整的 CPU 对象、它开头的通用 `CPUState` 部分，以及其中的体系结构状态 `CPURISCVState`。`tcg_env` 在运行时代表的是第三者的地址。

## 初始化时，TCG 先约定这个变量使用哪只寄存器

`tcg_context_init()` 中有：

```c
ts = tcg_global_reg_new_internal(s, TCG_TYPE_PTR, TCG_AREG0, "env");
tcg_env = temp_tcgv_ptr(ts);
```

这一步建立的是变量描述和句柄：

| 项目 | env 的设置 |
| --- | --- |
| C 句柄类型 | `TCGv_env`，它是 `TCGv_ptr` 的别名 |
| `base_type` | `TCG_TYPE_PTR` |
| `type` | `TCG_TYPE_PTR` |
| `kind` | `TEMP_FIXED` |
| `reg` | `TCG_AREG0` |
| `name` | `"env"` |

这相当于约定：“以后生成的代码里，凡是使用这个 IR 变量，就到 `TCG_AREG0` 指定的 host 寄存器找它。”此时还没有在这个描述符里绑定某一个 vCPU 的实际 `&cpu->env` 地址。

`TCG_TYPE_PTR` 说明采用 host C 指针的宽度；它没有记录“指向 CPURISCVState”这个 C 层面的对象类型。`TEMP_FIXED` 说明它的存放位置固定。两者分别回答“值有多宽”和“放在哪里”。

## 执行入口再把当前 vCPU 的地址传进去

`cpu_tb_exec()` 进入生成代码时调用：

```c
ret = tcg_qemu_tb_exec(cpu_env(cpu), tb_ptr);
```

第一个参数是当前 vCPU 的体系结构状态地址，第二个参数是要执行的 TB 代码地址。

`cpu_env()` 的非 const 分支实质上是：

```c
(CPUArchState *)((cpu) + 1)
```

这里 `cpu` 的类型为 `CPUState *`，加 1 是跨过整个 `CPUState`，不是只增加一个字节。当前对象布局要求体系结构状态紧接在 `CPUState` 后面，于是得到 `env` 的地址。

对 RISC-V 来说，上面这次调用的含义可以近似写为：

```c
/* 语义示意。 */
CPURISCVState *env = &riscv_cpu->env;
tcg_qemu_tb_exec(env, tb_ptr);
```

在原生 JIT 模式下，`tcg_qemu_tb_exec` 是指向生成的 prologue 的函数指针，其签名为：

```c
typedef uintptr_t tcg_prologue_fn(CPUArchState *env, const void *tb_ptr);
extern tcg_prologue_fn *tcg_qemu_tb_exec;
```

因此接下来要看 host 后端怎样生成入口代码。

## prologue 把第一个参数搬到 TCG_AREG0

例如 x86-64 host 后端会生成：

```c
tcg_out_mov(s, TCG_TYPE_PTR, TCG_AREG0, tcg_target_call_iarg_regs[0]);
```

RISC-V 64 host 后端也有同样的调用。它的意思是生成一条搬运操作，把第一个 ABI 参数寄存器中的 `env` 地址放进 TCG 约定的固定寄存器。之后再跳转到 `tb_ptr`。

不同 host 后端选择的固定寄存器不同，例如：

| host 后端 | `TCG_AREG0` |
| --- | --- |
| x86-64 | `TCG_REG_EBP`，此处对应 RBP 的寄存器编号 |
| RISC-V 64 | `TCG_REG_S0`，即 s0 |

这里必须区分两个目录：`target/riscv/` 是在讲“模拟 RISC-V guest”；`tcg/riscv64/` 是在讲“生成供 RISC-V host 执行的机器码”。模拟 RISC-V guest 不代表一定采用后者，在 x86-64 机器上使用的是 x86-64 host 后端。

整条链路可以画成：


<div class="mermaid tex2jax_ignore">
flowchart TD
    A[当前 RISCVCPU 对象] --&gt; B[cpu_env 得到 CPURISCVState 地址]
    B --&gt; C[作为第一个参数进入 tcg_qemu_tb_exec]
    C --&gt; D[prologue 将参数搬到 TCG_AREG0]
    D --&gt; E[TB 中使用 tcg_env 的代码从固定寄存器取得基址]
    E --&gt; F[基址加字段偏移 访问 guest CPU 状态]
</div>


如果 TB 直接链接到下一块 TB，通常会沿用同一个 `env`，不需要每过一个 TB 都重新走一次 C 入口。重新从执行入口进入时，才通过参数设置当前这次执行使用的 CPU 状态地址。

## RISC-V 前端怎样用这个基址

初始化 GPR 映射时，前端执行：

```c
for (i = 1; i < 32; i++) {
    cpu_gpr[i] = tcg_global_mem_new(tcg_env,
        offsetof(CPURISCVState, gpr[i]) + field_offset,
        riscv_int_regnames[i]);
    /* 省略 gprh 的映射。 */
}
```

它建立的是一条位置约定：

```text
cpu_gpr[i] 对应的内存位置
    = 运行时 env 地址
    + offsetof(CPURISCVState, gpr[i])
    + field_offset
```

对于通常的小端 RV64 场景，`field_offset = 0`。生成的代码需要从内存读取这个 global 时，就可以用类似下面的方式定位：

```text
load host_tmp, [固定 env 寄存器 + gpr[i] 的偏移]
```

随后这个 GPR 的值可以暂存在其他 host 寄存器中，不要求每次计算都访问 `env` 内存。

这里两个变量各司其职：

```text
tcg_env
    kind = TEMP_FIXED
    固定 host 寄存器中保存 CPU 状态结构的地址

cpu_gpr[i]
    kind = TEMP_GLOBAL
    对应上述结构中的 gpr[i] 字段
    可由寄存器分配器决定何时放进普通 host 寄存器
```

## 两种 env 不要混用

翻译器中还经常出现普通 C 变量：

```c
CPURISCVState *env = cpu_env(cpu);
```

这个 `env` 是真实的 C 指针，翻译函数可以用它读取当前用于指导翻译的状态，比如执行模式。而 `tcg_env` 是 IR 句柄，传给 `tcg_gen_*()` 或 `gen_helper_*()`，表达生成代码将来如何访问 CPU 状态。

同样一个“env”名字，在两个阶段承担的角色不同：

| 写法 | 所在层次 | 表示什么 |
| --- | --- | --- |
| `CPURISCVState *env` | 翻译器或 helper 的 C 代码 | 真实 host 地址，可以按 C 规则访问字段 |
| `TCGv_ptr tcg_env` | 构建 IR 的前端接口 | 指代“运行时 CPU 状态地址”的 IR 变量句柄 |
| `TCG_AREG0` 中的值 | 生成的 host 代码运行时 | 当前执行的 vCPU 的真实 `CPURISCVState *` |

这些地址属于 QEMU 的 host 进程地址空间。guest 自己的 PC、虚拟地址则是被模拟体系结构中的数值，需要相应的 guest 访存机制解释；它们与 `tcg_env` 的指针用途不同。

## 对应源码

版本范围见 [TCG变量总览——前端、IR与TCGContext](/虚拟化/2026/10/09/TCG变量总览-前端-IR与TCGContext/)。按下面的顺序看，就能把“句柄约定”和“实际地址传入”连起来：

1. [CPURISCVState 的 typedef](/home/holo/ospp-qemu-optimisation/qemu/target/riscv/cpu.h:38)、[通用 CPUArchState typedef](/home/holo/ospp-qemu-optimisation/qemu/include/qemu/typedefs.h:41)。
2. [ArchCPU 中的 env 成员](/home/holo/ospp-qemu-optimisation/qemu/target/riscv/cpu.h:570)。
3. [tcg_env 的初始化](/home/holo/ospp-qemu-optimisation/qemu/tcg/tcg.c:1819)。
4. [cpu_env 宏](/home/holo/ospp-qemu-optimisation/qemu/include/hw/core/cpu.h:603)和 [cpu_tb_exec 的调用](/home/holo/ospp-qemu-optimisation/qemu/accel/tcg/cpu-exec.c:442)。
5. [生成入口的函数类型](/home/holo/ospp-qemu-optimisation/qemu/include/tcg/tcg.h:937)。
6. [x86-64 prologue](/home/holo/ospp-qemu-optimisation/qemu/tcg/x86_64/tcg-target.c.inc:4534)、[RISC-V 64 prologue](/home/holo/ospp-qemu-optimisation/qemu/tcg/riscv64/tcg-target.c.inc:3053)。
7. [RISC-V GPR 的 global 映射](/home/holo/ospp-qemu-optimisation/qemu/target/riscv/tcg/translate.c:1520)。

相关概念：[TCG变量的类型——TCGv句柄与TCGType](/虚拟化/2026/10/15/TCG变量的类型-TCGv句柄与TCGType/)、[TCG变量的生存期与存放约束——TCGTempKind](/虚拟化/2026/10/09/TCG变量的生存期与存放约束-TCGTempKind/)。
