---
layout:       post
title:        "Zve32x下部分向量指令的执行问题以及fix方案"
author:       "zhenbaii"
date:         2026-11-17
header-style: text
catalog:      true
lang:         zh-CN
categories:   虚拟化
tags:
    - "QEMU"
    - "RVV"
    - "GVEC"
---
[English version]({% post_url 2026-11-17-riscv-zve32x-gvec-en %})

# 问题复现

## Zve32x

Zve32x是RVV的一个标准扩展子集, 用于那些资源受限的32位嵌入式和边缘设备. 其包含以下特点:

-  支持8, 16, 32位的**整数**向量元素, 以及规定的整数, 访存, 掩码, 规约等指令
-  本身不提供向量浮点运算的支持
-  最小允许的VLEN可以达到32位, 当然也支持64, 128等更大的VLEN
-  不限制于RV32 或者 RV64

因此, 扩展中的`32`应当理解为这个扩展可以使得`VLEN`达到32这个下界, 而不是一定是32.  比如完整的`V`扩展就规定了`VLEN`的最小值是128,  因此在`V`扩展下, 是做不到`VLEN=32`这件事情的. `VLEN`规定了物理向量寄存器的长度, 而`ELEN`则规定了向量元素的最大长度:

| 参数       | 含义            | 直观理解                   |
| -------- | ------------- | ---------------------- |
| **ELEN** | 实现支持的最大向量元素位宽 | 单个元素最大能有多宽             |
| **VLEN** | 单个架构向量寄存器的位数  | 一个 `v0` 或 `v1` 总共能装多少位 |

## issue链路

启用`Zve32x`, 并且设定:
```
VLEN = 32 BIT
ELEN = 32 BIT
```

执行包含如下代码片段的一个程序:
```
vsetivli zero, 1, e32, m1, tu, mu   # SEW = 32, LMUL = 1, vl = 1
vredsum.vs v3, v2, v1               # v1[0] 和 v2的非掩码元素相加, 结果存到v3
```

复现命令:
```
/path/to/qemu-system-riscv64 \
  -M virt \
  -accel tcg \
  -cpu rv64,v=false,zve32x=true,vlen=32,elen=32 \
  -smp 1 \
  -m 128M \
  -bios none \
  -nographic \
  -monitor none \
  -serial none \
  -kernel /path/to/rv64-minimal.elf
```

会出现如下报错:
```
qemu-system-riscv64: ../tcg/tcg-op-gvec.c:56: check_size_align: Assertion `(maxsz & max_align) == 0' failed.
Aborted (core dumped)
```

对于本次测试程序, 其产生错误的链路如下:
```
trans_vredsum_vs
  → opivv_trans
  → tcg_gen_gvec_4_ptr
  → simd_desc(oprsz=4, maxsz=4, data)
  → check_size_align
  → 断言失败，QEMU 进程 SIGABRT
```

这个错误链路涉及到RVV的指令和TCG的GVEC模块, 以及`maxsz`和`oprsz`这两个变量, 特别是`maxsz`这个变量成为了这个issue的关键所在.

# 原因分析

要阅读这一章的内容, 请补充GVEC的背景知识.

### GVEC通用路径 or 指令专用Helper

通过梳理GVEC的RVV前端相关的代码, 我们可以梳理出TCG在处理RVV的指令时的执行链路如下:
```
解码、翻译期
─────────────────────────────────────────────────────────
Guest 机器码（RVV 指令）
  ↓
decode_insn32
  │  匹配指令编码，提取寄存器编号、vm 等字段
  ↓
trans_*                              ← 部分由宏生成
  │  检查指令及操作数是否合法
  │  按 SEW 预选 RVV helper 的调用生成函数 fn
  │  例如 fns[s->sew] = gen_helper_vadd_vv_w
  │  此时还没有调用 fn
  │
  ├─ 本指令提供通用 GVEC 路径，例如 vadd.vv/vx/vi
  │    ↓
  │  do_opivv_gvec / do_opivx_gvec / do_opivi_gvec
  │    │
  │    ├─ 满足进入通用 GVEC 的条件
  │    │    不使用掩码、vstart=0、vl=VLMAX、逻辑组长度≥8字节
  │    │    且 !(s->vta && s->lmul < 0)
  │    │    ↓
  │    │  tcg_gen_gvec_add / adds / addi
  │    │    │
  │    │    ├─ 选择 inline 实现
  │    │    │    → 生成 TCG 向量或标量运算 IR
  │    │    │
  │    │    └─ 选择 .fno 指定的 helper 实现
  │    │         → 准备该 helper 的参数和 descriptor
  │    │         → 调用相应的 gen_helper_*
  │    │           例如 gen_helper_gvec_add32
  │    │
  │    └─ 不满足条件
  │         → 进入下面的 RVV helper 调用生成过程 [H]
  │
  └─ 本指令当前只提供专用 helper，例如 vredsum.vs
       ↓
     opivv_trans
       → 直接进入 [H]，没有通用 GVEC 分流


[H] RVV helper 调用生成过程
─────────────────────────────────────────────────────────
按当前指令的参数形式准备调用：

  vv 形式：
    do_opivv_gvec 的 else 分支，或 opivv_trans
      │  构造 data：VM、LMUL、VTA、VMA 等
      ↓
    tcg_gen_gvec_4_ptr
      │  simd_desc(oprsz, maxsz, data) 构造 descriptor
      │  准备 vd、v0、vs1、vs2 的指针，以及额外参数 env
      └─ 调用 fn(...)

  vx / vi 形式：
    opivx_trans / opivi_trans
      │  自己构造 data 和 descriptor
      │  准备 vd、v0、vs2 的指针
      │  准备标量值或立即数，以及 env
      └─ 调用 fn(...)
           fn 的类型是 gen_helper_opivx *
           gen_helper_opivx 本身只是类型，不是流程中的函数

这些例子中的 fn 调用：
  fn(...) = gen_helper_vadd_vv_w(...) 等具体函数  ← 宏生成
    ↓
  tcg_gen_call6
    ↓
  tcg_gen_callN
    ↓
  生成 INDEX_op_call

完成当前指令的 IR 生成
```

上面的流程图比较详细, 整个流程中比较关键的链路其实可以总结如下:
```
trans_*
  │
  │
  ├─ 本指令提供通用 GVEC 路径，例如 vadd.vv/vx/vi
  │    ↓
  │  满足进入通用 GVEC 的条件
  │    │
  │    ├─ inline TCG IR(.fni4 / .fni8) or gvec helper(.fno)
  │    │
  │    │
  │	不满足条件
  │	   │
  │    ├─ 回退到发射指令专用的helper
  │
  │
  ├─ 不提供通用GVEC路径
       │
       │
	   ├─ 发射指令专用的helper
```

核心流程: **判断这个指令是否具备GVEC通用路径. 如果具备, 尝试进入通用GVEC, 否则回退到指令专用helper. 而如果不具备, 则直接产生指令专用helper.**

对于**绝大多数RVV指令**而言, 满足进入通用GVEC的条件都是大差不差的, 并且都包含了这个条件:
```
vl_eq_vlmax == TRUE
```

比如我们查看`do_opivv_gvec`的源码:
```c
static inline bool
do_opivv_gvec(DisasContext *s, arg_rmrr *a, GVecGen3Fn *gvec_fn,
              gen_helper_gvec_4_ptr *fn)
{
    if (a->vm && s->vl_eq_vlmax && !(s->vta && s->lmul < 0)) {
	    /* 进入了通用的GVEC路径 */
        gvec_fn(s->sew, vreg_ofs(s, a->rd),
                vreg_ofs(s, a->rs2), vreg_ofs(s, a->rs1),
                MAXSZ(s), MAXSZ(s));
    } else {
	    /* 进入了指令专用Helper路径 */
        uint32_t data = 0;

        data = FIELD_DP32(data, VDATA, VM, a->vm);
        data = FIELD_DP32(data, VDATA, LMUL, s->lmul);
        data = FIELD_DP32(data, VDATA, VTA, s->vta);
        data = FIELD_DP32(data, VDATA, VMA, s->vma);
        tcg_gen_gvec_4_ptr(vreg_ofs(s, a->rd), vreg_ofs(s, 0),
                           vreg_ofs(s, a->rs1), vreg_ofs(s, a->rs2),
                           tcg_env, s->cfg_ptr->vlenb,
                           s->cfg_ptr->vlenb, data, fn);
    }
    finalize_rvv_inst(s);
    return true;
}
```

而通过查阅QEMU源码, 我们可以找到这个条件成立所需要满足的条件:
```
vl_eq_vlmax =
    (env->vstart == 0) &&
    (env->vl == vlmax) &&
    (maxsz >= 8);
```

其中, `maxsz`是本次问题的关键所在. 在讲解关键问题之前, 我们先看看`maxsz`这个变量在整个翻译过程的哪个阶段就被设置, 发生**进入GVEC通用路径还是指令专用helper**时, `maxsz`的值又是多少.

`maxsz`的值在TB诞生之初就被设置, 具体发生在`riscv_get_tb_cpu_state`:
```c
int8_t lmul = /* 从 env->vtype 提取 LMUL 编码 */;
uint32_t vsew = /* 从 env->vtype 提取 SEW 编码 */;

uint32_t vlmax = vext_get_vlmax(cpu->cfg.vlenb, vsew, lmul);
uint32_t maxsz = vlmax << vsew;   /* 语义: 逻辑寄存器的长度 */

/* maxsz会影响vl_eq_vlmax的设置 */
bool vl_eq_vlmax = (env->vstart == 0) &&
                   (vlmax == env->vl) &&
                   (maxsz >= 8);
```

因此, 在发生**进入GVEC通用路径还是指令专用helper**这个判断时, 我们可以认为, 对于大多数指令而言(条件判断中包含了vl_eq_vlmax), 这个判断本身隐含了**逻辑寄存器的长度是否大于等于8字节**这个判断(通过vl_eq_vlmax间接产生). 换句话说, 如果想要进入GVEC通用路径, 首先必须满足**逻辑寄存器长度>=8字节**这一个条件. 请记住这一事实.

实际上, 还存在在做判断时没有包括vl_eq_vlmax的指令, 比如`vmv1r`. 下表给出了完整的统计:

| 入口                                | 是否检查 `vl_eq_vlmax`             |
| --------------------------------- | ------------------------------ |
| 普通运算、移动/广播、gather，共 44 个入口        | **检查**，间接要求逻辑组至少 8 字节          |
| 整寄存器移动 `vmv1r/2r/4r/8r.v`，共 4 个入口 | **不检查**，分流只检查 `vstart_eq_zero` |

### check_size_align(oprsz, maxsz, ofs)

回顾前面的错误链路, 这个函数就是发生断言的关键所在.  这个函数实际上是在构造GVEC descriptor时被调用的, 在`simd_desc`内部被调用.  以及, 他也会在进入通用GVEC路径后, 选择`fni4, fni8, fno`之前被调用. 我们看一下这个函数做了什么事情:
```c
static void check_size_align(uint32_t oprsz, uint32_t maxsz, uint32_t ofs)
{
    uint32_t max_align;

    switch (oprsz) {
    case 8:
    case 16:
    case 32:
        tcg_debug_assert(oprsz <= maxsz);
        break;
    default:
        tcg_debug_assert(oprsz == maxsz);
        break;
    }
    tcg_debug_assert(maxsz <= (8 << SIMD_MAXSZ_BITS));

    max_align = maxsz >= 16 ? 15 : 7;
    tcg_debug_assert((maxsz & max_align) == 0);
    tcg_debug_assert((ofs & max_align) == 0);
}
```

通过梳理这个函数的逻辑, 我们可以发现QEMU主线中, 对于调用到了这个函数的上下文, 这个上下文对于oprsz和maxsz的关系, 以及这两个值本身的限制的约定:

| `maxsz` 的范围 | 对齐要求 |
| --- | --- |
| 小于 16 字节 | 必须是 8 的倍数 |
| 至少 16 字节 | 必须是 16 的倍数 |

```
oprsz ∈ {8, 16, 32}：
    要求 oprsz ≤ maxsz

oprsz 是其他值：
    要求 oprsz = maxsz
```

因此, 对于调用到了这个check函数的上下文, `maxsz`的合法取值为:
```
8、16、32、48、64、……、2048
```

即`maxsz`的最小值必须是8. 我们通过梳理源码, 可以知道调用到了这个函数的上下文包括:
```
RISC-V 层分流
    │
    ├─ 通用 GVEC 路径
    │    ↓
    │  check_size_align(...)
    │    ★ 收到 4 字节尺寸，此处就会断言
    │    ↓
    │  选择实现
    │    │
    │    ├─ .fniv / .fni8 / .fni4
    │    │    → 展开 IR，不调用 simd_desc()
    │    │
    │    └─ .fno
    │         → simd_desc(oprsz, maxsz, data)
    │              → 内部再次调用 check_size_align(...)
    │         → 生成通用 helper 调用
    │
    └─ RVV 专用 helper 路径
         → simd_desc(oprsz, maxsz, data) (构造descriptor)
              → check_size_align(...)
                   ★ 原始复现的 4/4 在此断言
         → 生成 RVV helper 调用
```

因此, 可能发生这个断言错误的地方就集中在了**RVV专用helper路径**和**通用GVEC路径的check_size_align**这两个地方.

对于RVV专用helper路径, 其原因是因为构造了descriptor. 我们通过对源码树的统计, 可以发现**绝绝绝大多数RVV指令**在发射helper时, 都会去构造descriptor, 并且这些指令中的大多数, 都会把`maxsz`设置为物理寄存器而非逻辑寄存器的长度:

| RVV 专用指令helper的发射方式                           |     入口数 | 尺寸                                          |
| --------------------------------------------- | ------: | ------------------------------------------- |
| 经 `tcg_gen_gvec_N_ptr()` 包装                   | **182** | 178 个使用 `vlenb/vlenb`；4 个整寄存器移动使用 `vlenb×N` |
| translator 自己构造 descriptor，再调用 `gen_helper_*` | **220** | 全部使用 `vlenb/vlenb`                          |
| 不构造 descriptor                                |   **5** | 三条 `vset*` 和两条向量到标量读取                       |

因此, 可以预想, 这个错误也会影响到RVV中的大多数指令.

而对于**通用GVEC路径的check_size_align**则比较简单. 前面我们提到过, 对于绝大多数提供了GVEC通用路径的指令, 他们进入GVEC通用路径时, 都会判断`vl_eq_vlmax`是否为true, 而这个判断为true, 必须满足`maxsz >= 8`, 从而也就不会触发断言.

整寄存器移动 `vmv1r/2r/4r/8r.v` 的通用路径不检查 `vl_eq_vlmax`。但在本次 VLEN=32 的配置下，只有 `vmv1r.v` 的总尺寸是 4 字节，会在路径内部的 `check_size_align` 处触发尺寸断言；`vmv2r/4r/8r.v` 的总尺寸分别是 8、16、32 字节。

### Combine all facts together

把前面的事实总结一下, 我们就可以得到如下的边界条件:

- **RVV 专用 helper 调用生成路径**：多数入口使用 `oprsz=maxsz=vlenb=4`，在构造 `simd_desc()` 时，由内部的 `check_size_align()` 触发断言。原始 `vredsum.vs` 复现属于这一类。
-  **通用 GVEC 展开路径**：大多数入口通过 `vl_eq_vlmax` 排除了小于 8 字节的逻辑组；但整寄存器移动没有这项门控。`vmv1r.v` 在 `vstart=0` 时将 `4/4` 直接传入通用 GVEC，在展开器入口的 `check_size_align()` 处触发断言，尚未选择 inline 或 `.fno`。

RVV 专用 helper 的尺寸也并非全部等于物理寄存器长度：整寄存器移动使用 `vlenb×N`。因此判断是否触发问题，应看**实际传给检查函数的 `oprsz/maxsz`**。

不构造 descriptor 的路径，不会在 descriptor 构造处触发该断言；通用 `.fno` 路径则已经通过前面的尺寸检查，不是此次 4 字节 descriptor 故障的入口。

# 修复方案讨论：当前补丁覆盖 descriptor 路径

本轮讨论的本地实验补丁为 `7609a8e46d8b`，基于 `4437f0c7ef33`。它解决的是 **GVEC descriptor 无法表达 `oprsz=maxsz=4`** 的问题，而不是一次性完成所有 VLEN=32 执行路径的适配。

## 编码与解码的修改

原始 `MAXSZ` 字段按 `(maxsz / 8) - 1` 编码，按 `(field + 1) * 8` 解码，因此不能表达 4 字节。补丁复用字段值 `2` 表示 4 字节：它原来对应 24 字节，而 24 不满足原有 `maxsz` 必须按 16 字节对齐的要求，并不是一个原本支持的合法最大尺寸。

`OPRSZ` 已经有字段值 `2` 表示“等于 MAXSZ”的约定，因此无需新增位数。新增情况的编码与解码过程为：

```text
simd_desc(4, 4, data)
    MAXSZ 字段 = 2，新增语义为 4 字节
    OPRSZ 字段 = 2，沿用“等于 MAXSZ”的语义
    ↓
simd_maxsz(desc) = 4
simd_oprsz(desc) = simd_maxsz(desc) = 4
```

实现上，`simd_desc()` 对 `maxsz=4` 单独处理，并用调试断言要求 `oprsz=4`；其他尺寸沿用原有检查和编码。`simd_maxsz()` 增加字段值 `2` 的解码特例，`simd_oprsz()` 现有逻辑即可取得 4。原有合法尺寸的编码、字段位置和 22 位 DATA 保持兼容。

这一轮只增加 `4/4`，没有增加 `4/8` 等混合尺寸，也没有把原来的尺寸约束普遍放宽为“四字节对齐”。

## 未覆盖的若干种情况

- `vmv1r.v` 在 `VLEN=32`、`vstart=0` 时仍会触发通用 GVEC 展开器中的 `check_size_align()` 断言。
- 整寄存器加载和存储的直接展开路径仍按 8 字节访问，超出了单个 4 字节寄存器的范围。
- `vmvr_v` 的大端分支按 8 字节边界拷贝，4 字节寄存器的处理仍需修复和验证。
