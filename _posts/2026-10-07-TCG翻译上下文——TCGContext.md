---
layout:       post
title:        "TCG翻译上下文——TCGContext"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 虚拟化
tags:
    - "TCG"
---
关于 `temps[]`、`TCGTemp` 和前端 `TCGv_xxx` 句柄的关系，可以先读 [TCG变量总览——前端、IR与TCGContext](/虚拟化/2026/10/09/TCG变量总览-前端-IR与TCGContext/)；句柄的偏移编码和类型约束见 [TCG变量的类型——TCGv句柄与TCGType](/虚拟化/2026/10/15/TCG变量的类型-TCGv句柄与TCGType/)。

这个结构体定义在`include/tcg/tcg.h`中，作为TCG的公用模块。TCG的运行模式一共有两种，根据宿主机线程数量区分：

| 模式                                             | 宿主机线程                            | 怎么跑 vCPU           |
| ---------------------------------------------- | -------------------------------- | ------------------ |
| single（round-robin，默认在 icount 或不支持 MTTCG 的目标上） | 一条，`rr_cpu_thread_fn`            | 所有 vCPU 轮流在这一条线程上跑 |
| multi（MTTCG）                                   | 每个 vCPU 一条，`mttcg_cpu_thread_fn` | 同时执行               |

single模式中，虽然有多个vCPU，但是他们都是在一条线程上轮流跑的; 而多线程模式下，每个vCPU都有自己单独的线程，同时执行。`TCGContext`这个结构体是跟随线程，而不是跟随vCPU的。也就是说，在单线程模式下，即使有多个vCPU，全局也仅有一个`TCGContext`。 而多线程模式下，则是每个线程有自己独立的`TCGContext`。 换句话说，这个结构体是翻译线程的上下文，因此必然跟随线程。
在进入讲解之前，我们首先需要认识到，这个结构体并没有去持久维护每个TB的翻译结果，而是维护了翻译过程中的上下文，因此当中的许多字段只对翻译某一块TB时有效。当翻译下一块TB时，里面的某些字段是会被重置的。

## 结构体定义

下面展示了简化的`TCGContext`, 删去了一些不关键的字段:
```c
struct TCGContext {
    uintptr_t pool_cur, pool_end;
    TCGPool *pool_first, *pool_current, *pool_first_large; 
	/* ... */
    TCGType addr_type;            /* TCG_TYPE_I32 or TCG_TYPE_I64 */
    TCGBar guest_mo;

    TCGRegSet reserved_regs;
    intptr_t current_frame_offset;
    intptr_t frame_start;
    intptr_t frame_end;
    TCGTemp *frame_temp;

    TranslationBlock *gen_tb;     /* tb for which code is being generated */
    tcg_insn_unit *code_buf;      /* pointer for start of tb */
    tcg_insn_unit *code_ptr;      /* pointer for running end of tb */

    /* Code generation.  Note that we specifically do not use tcg_insn_unit
       here, because there's too much arithmetic throughout that relies
       on addition and subtraction working on bytes.  Rely on the GCC
       extension that allows arithmetic on void*.  */
    void *code_gen_buffer;
    size_t code_gen_buffer_size;
    void *code_gen_ptr;
    void *data_gen_ptr;

    /* Threshold to flush the translated code buffer.  */
    void *code_gen_highwater;

    /* Track which vCPU triggers events */
    CPUState *cpu;                      /* *_trans */

    /* These structures are private to tcg-target.c.inc.  */
    QSIMPLEQ_HEAD(, TCGLabelQemuLdst) ldst_labels;
    struct TCGLabelPoolData *pool_labels;

    TCGLabel *exitreq_label;

    GHashTable *const_table[TCG_TYPE_COUNT];
    TCGTempSet free_temps[TCG_TYPE_COUNT];
    TCGTemp temps[TCG_MAX_TEMPS]; /* globals first, temps after */

    QTAILQ_HEAD(, TCGOp) ops, free_ops; 
    QSIMPLEQ_HEAD(, TCGLabel) labels;

    /*
     * When clear, new ops are added to the tail of @ops.
     * When set, new ops are added in front of @emit_before_op.
     */
    TCGOp *emit_before_op;
    /* Exit to translator on overflow. */
    sigjmp_buf jmp_trans;
};
```

下面，我们就来解读一下这个结构体中一些十分关键的字段。理解了这些字段，那么就可以大致理解TCG是如何做翻译，如何维护翻译上下文的。

### pool_*

在结构体的最上方，定义了如下的字段：
```c
struct TCGContext {
    uintptr_t pool_cur, pool_end;
    TCGPool *pool_first, *pool_current, *pool_first_large; 
    
    /* .... */
}
```

这是翻译线程用于满足翻译过程中所产生的临时内存需求，用`TCGPool`来表示这个缓存池。我们去看一下`TCGPool`的定义，其同样定义在`include/tcg/tcg.h`中：
```c
typedef struct TCGPool {
    struct TCGPool *next;
    int size;
    uint8_t data[] __attribute__ ((aligned));
} TCGPool;  

#define TCG_POOL_CHUNK_SIZE 32768         // 32KB
```

从next字段就可以看到，整个大缓存池由许多的`TCGPool`小缓存池组成，小缓存池之间用链表结构连接起来。data字段的最大长度是`TCG_POOL_CHUNK_SIZE`。 这种定义方式就是所谓的柔性数组，data的数据空间会紧接在`size`字段后面(可能还有对齐需求产生的空洞)，在分配`TCGPool`的空间时，顺便分配这个data的数据长度：
```c
TCGPool *pool = malloc(sizeof(TCGPool) + n)  // sizeof(data) = 0
```

`TCGContext`利用`pool_first`字段拥有整个链表。因此，整个缓存池的结构如下：
```
TCGContext
┌──────────────────┐
│       ...        │
│   pool_first     │
│       .│.        │
└────────┬─────────┘
         │
         ▼
   ┌───────────┐      ┌───────────┐      ┌───────────┐
   │ TCGPool   │      │ TCGPool   │      │ TCGPool   │
   │    ...    │      │    ...    │      │    ...    │
   │   next    │─────►│   next    │─────►│   next    │─────► NULL
   │    ...    │      │    ...    │      │    ...    │
   └───────────┘      └───────────┘      └───────────┘
```

每个小缓存池的最大大小是4KB，按需分配。并且，分配完后就不会释放，而会被后续的翻译流程反复使用，从而防止反复malloc. `pool_current`就指向了当前正在使用的那一块小缓存池。而`pool_cur`和`pool_end`则是`pool_current`这块小缓存池内部的指针，我们可以画出来如下：
```

```
