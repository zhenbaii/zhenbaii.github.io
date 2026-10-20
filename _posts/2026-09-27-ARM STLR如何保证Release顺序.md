---
layout:       post
title:        "ARM STLR如何保证Release顺序"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 操作系统
tags:
    - "内存管理"
---
在AArch64上，C/C++中的release store通常会被编译为`STLR`（Store-Release）。例如：

```cpp
data = 42;
ready.store(1, std::memory_order_release);
```

可能生成：

```asm
str   w2, [data]
stlr  w1, [ready]
```

看到`STLR`以后，最容易形成的错误理解是：处理器给`ready=1`附加一个release标签，另一个核心以后读取`ready`时发现这个标签，再回头等待发布核心清空Store Buffer。

实际情况不是这样。理解`STLR`最重要的一句话是：

> **Release是发布核心单方面、异步履行的排序承诺。另一个核心既看不到release标签，发布核心也不需要等待某次acquire出现。**

### `STLR`保证的不是物理执行顺序

考虑下面三条指令：

```asm
str   x0, [A]
str   x1, [B]
stlr  w2, [flag]
```

`STLR`要求程序顺序中位于它之前的内存访问，在架构可观察的内存顺序上不能跑到这次release store之后。对于这个例子，可以先记成：

```text
写A ──┐
       ├── 必须在release顺序上位于flag之前
写B ──┘
       ↓
STLR flag
```

但这并不表示处理器执行到`STLR`时必须停住流水线，把前面的指令一条一条做完。微架构内部仍然可以：

- 乱序执行指令；
- 提前计算地址和数据；
- 使用Load Queue、Store Queue和Store Buffer；
- 推测执行更年轻的指令；
- 并行推进多个cache miss或一致性事务。

Arm架构约束的是软件能够观察到的内存顺序，而不是要求内部电路严格按照程序顺序活动。只要其他观察者不能看到违反release语义的结果，内部怎样调度是处理器自己的实现选择。

因此，下面这种情况不能成为架构可见结果：

```text
另一个核心已经观察到 flag = 1

但A、B仍处在一种会使该核心
合法观察到release之前旧状态的位置
```

### Release是单向约束

`STLR`更像一条只朝一个方向起作用的边界：

```text
release之前的内存访问
           │
           │ 不允许以违反release语义的方式越过
           ▼
        [ STLR ]
           │
           │ release本身不提供反方向的acquire约束
           ▼
release之后的访问
```

例如：

```asm
str   x1, [A]
stlr  x2, [flag]
ldr   x3, [B]
```

这里必须维持的是`STR A -> STLR flag`的发布顺序。后面的`LDR B`并不会因为前面有一条`STLR`，就自动获得“不能提前执行”的acquire语义；它可以在内部很早开始推测，只要最终没有产生架构不允许的观察结果。

这也是为什么release通常比一个双向的完整屏障限制更少。不要把它理解成“`STLR`不能和前后任何东西交换”，而应理解成“release以前的访问不能从可观察顺序上掉到发布点后面”。

### 最关键的理解：发布与获取是异步的

假设CPU 0是发布者：

```asm
str   x0, [A]
str   x1, [B]
stlr  w2, [flag]
```

CPU 0必须独立完成自己的release约束：

```text
A、B达到release要求的ordering/visibility状态
                    │
                    ▼
      才允许flag以release允许的方式对外可见
```

这个过程与其他核心正在做什么无关。CPU 1可能马上读取`flag`，可能过了一秒才读，也可能永远不读。CPU 0都不会：

```text
等待某个核心执行LDAR
询问是否存在消费者
把release标记留在cache line中等待领取
```

换句话说，发布端与获取端不是一次实时握手：

```text
CPU 0                                      CPU 1

STR A
STR B
STLR flag
  │
  │ 本核及内存系统独立履行release顺序
  ▼

                                           ……可能相隔很久……

                                           LDAR flag
                                           LDR A
```

CPU 0执行`STLR`时不需要知道CPU 1是否存在，也不关心CPU 1以后使用普通`LDR`还是`LDAR`。release约束无条件成立。

只有CPU 1后来通过一个合适的acquire操作，实际读到了这次release store写入的值，两侧的保证才组合起来：

```text
CPU 0                                      CPU 1

STR A
  │
  │ release order
  ▼
STLR flag ───────── reads-from ──────────> LDAR flag
                                                │
                                                │ acquire order
                                                ▼
                                              LDR A
```

在ISA层面，这意味着相应访问之间形成了Arm内存模型要求的顺序；若这些指令来自C/C++原子操作，那么release store与读取到它的acquire load会建立`synchronizes-with`，进而形成`happens-before`。

这里不能把语言模型和硬件术语完全混为一谈：`happens-before`属于C/C++内存模型，Arm指令级描述使用的是ordered-before、observed-by、reads-from等关系。编译器负责把前者正确映射到后者。

### 其他核心看不到“release标签”

`STLR`写出的仍然只是普通数据：

```text
flag = 1
```

cache line中不会变成：

```text
flag = 1 + RELEASE_TAG
```

因此，获取核心不会执行下面这种流程：

```text
读取flag
   │
   ▼
发现release标签
   │
   ▼
通知CPU 0清空Store Buffer
   │
   ▼
等待CPU 0回复
```

release属性如果在微架构中以某个bit存在，它也是发布核心内部指令或队列entry的控制信息。这个信息用于限制该请求何时可以发向cache、一致性系统或互连，并不会作为数据的一部分传播给读者。

获取端之所以能获得保证，不是因为它识别了一个运行时标签，而是因为：

1. 发布端已经无条件遵守了release顺序；
2. 获取端的`LDAR`无条件遵守acquire顺序；
3. `LDAR`读到`STLR`产生的值后，这两段局部顺序通过reads-from关系接了起来。

### 硬件在哪里“消费”release属性

Arm ISA规定必须得到什么观察结果，但不规定处理器内部一定存在名为`release`的RTL字段。下面是一种典型且便于理解的实现方式，不是架构强制结构。

`STLR`在译码后可能成为一条带release属性的store micro-op：

```text
Store Queue entry

+----------------------------+
| valid                      |
| committed                  |
| address = flag             |
| data = 1                   |
| sequence_number = 12       |
| release = 1                |
+----------------------------+
```

假设队列里是：

```text
             OoO pipeline
                    │
                    ▼
          Load/Store Queue
     +-------------------------+
     | #10  STR  A             |
     | #11  STR  B             |
     | #12  STLR flag  REL=1   |
     | #13  STR  C             |
     +-------------------------+
                    │
             issue/drain logic
                    │
                    ▼
             L1D / coherence
```

普通store允许发送的条件可以粗略表示为：

```text
committed
&& address_ready
&& data_ready
&& 满足普通store的其他约束
```

release store则可能多一个条件：

```text
committed
&& address_ready
&& data_ready
&& all_older_memory_ops_sufficiently_ordered
```

Store Queue或LSQ的调度逻辑读取release属性，把它转换为一条顺序依赖：当程序顺序中更老的内存访问还没有达到要求时，`STLR`暂时不能以会破坏release语义的方式对外可见。

这就是所谓“release标记被消费”的一个具体含义。消费它的不是另一个CPU，而是本核LSU/LSQ的发射逻辑，以及必要时更下游的接口、互连和一致性系统。

### Store Buffer按FIFO排空还不够

如果一个简单处理器的Store Buffer严格按照FIFO顺序向cache排空：

```text
head
  |
  v
+-----------+
| STR A     |
+-----------+
| STR B     |
+-----------+
| STLR flag |
+-----------+
```

那么`A -> B -> flag`这部分store-to-store顺序天然就能维持，`STLR`不一定需要为这些store增加很多额外控制。

但Store-Release约束的不只有旧store，程序顺序中位于它之前的load也属于需要考虑的内存访问：

```text
旧Load Queue entries ──┐
                       ├── 达到release所需状态
旧Store Queue entries ─┘
                       ↓
                  STLR可以推进
```

因此，完整实现通常要看LSQ整体状态，而不能只凭Store Buffer按FIFO排空就断言release已经满足。

### “前面的都排空”应该怎样理解

把`STLR`理解成“等前面的访问排空以后再发布flag”，作为第一层直觉是有用的。但这里的“排空”要小心理解。

它不一定表示：

```text
整个Store Buffer变成empty
```

更不表示：

```text
所有数据已经一路写入DRAM
```

更准确的说法是：

> release以前受约束的访问，已经离开可能造成错误重排的待排序状态，达到架构要求的ordering/visibility point。

例如，`A`的store已经被一致性系统接管，本核取得了相应cache line的写权限，系统能够保证观察者不会以违反release顺序的方式看到`flag`和`A`，这时可能已经足够。数据完全可以仍然停留在处于`M`状态的cache line中，不必写回DRAM。

```text
Store Queue / Store Buffer
             │
             ▼
     coherent cache system
             │
             ▼
达到release要求的可观察顺序

并不要求继续走到：LLC -> memory controller -> DRAM
```

同样，队列中可能还有release之后的年轻store，例如`STR C`。`STLR`没有要求“所有年龄的entry一个不剩”，只要求它负责排序的旧访问满足条件。

### 排序责任也可能继续向下传递

最保守的实现可以让`STLR`一直等在核心里：

```text
旧访问达到要求
       │
       ▼
再把STLR发给cache/coherence
```

高性能实现也可以让多个事务都离开LSQ，但把顺序信息继续传给更下游的结构：

```text
CPU master interface
        │
        ├── A transaction
        ├── B transaction
        `── flag transaction [ordered]
                    │
                    ▼
             interconnect/coherence
             保证观察顺序不被破坏
```

这时release语义由核心接口、互连和一致性协议共同完成。事务离开CPU接口的先后，不一定就是最终被其他观察者看到的先后。

所以，下面两种方案都可能正确：

```text
方案一：核心内部等旧访问足够完成，再发送STLR

方案二：允许请求继续下行，由内存系统携带并维护ordering约束
```

Arm只规定最终允许哪些观察结果，不要求所有实现必须选择同一方案。

### 指令退休、发出请求和对外可见不是一回事

讨论微架构时还要区分三个时间点：

```text
STLR从ROB退休
        │
        ▼
STLR等待或离开Store Queue
        │
        ▼
STLR在架构定义的范围内对其他观察者可见
```

为了维持精确异常，一条store可以在确认自身不会再被回滚后从ROB退休，但它仍然留在Store Queue或Store Buffer中等待cache权限。`STLR`的release约束主要限制后两层如何推进，不代表前端必须一直停在这条指令上。

以gem5 O3的概念路径为例，store的提交和向cache发送本来就是不同阶段：

```text
executeStore()
      ↓
Store Queue
      ↓
commitStores()
      ↓
标记canWB
      ↓
writebackStores()
      ↓
sendPacketToCache()
```

具体函数和release检查位置会随gem5版本变化，但这条路径很好地展示了：指令已经提交，不代表store已经进入一致性系统，更不代表其他核心已经可以观察到它。

### 用发布—订阅例子收束

```text
Producer                               Consumer

data = 42
    │
    ▼
STLR ready = 1
    │
    │ Producer独立履行release承诺
    │ Consumer此时可以尚未运行
    ▼

                                       LDAR ready
                                           │
                                           │ 若读到1
                                           ▼
                                       read data
```

整个过程可以拆成三个互不混淆的事实：

1. `STLR`保证发布端以前的内存访问不会以违反release语义的方式落到发布点以后；
2. `LDAR`保证获取端后面的内存访问不会以违反acquire语义的方式跑到获取点以前；
3. 当`LDAR`读到`STLR`写出的值，两条本地单向顺序才通过这次通信组成跨核同步。

所以，release/acquire不是“发布核心与获取核心同时到场完成握手”，而是两个核心各自遵守本地规则，随后由一次真实的读取关系把规则拼接起来。

### 几个容易混淆的说法

|容易产生误解的说法|更准确的理解|
|---|---|
|`STLR`会禁止处理器乱序执行|内部仍可乱序，禁止的是违反release语义的可观察结果|
|release标签会随数据传到其他核心|release属性属于发布端的排序控制，读者只看到普通数据|
|等其他核心acquire后，发布端才开始清队列|发布端无条件、异步履行release约束|
|`STLR`必须把Store Buffer全部清空|只需让release以前受约束的访问达到规定的顺序点|
|排空表示写入DRAM|数据进入能维持相应顺序的一致性域就可能足够|
|Store Buffer FIFO就能实现全部release语义|旧load也受release排序约束，需要考虑LSQ和更下游内存系统|
|`STLR`是一条完整双向屏障|它是面向release方向的单向约束|

说到底，release保证的事情是：在其他核心看来，如果对release对应的flags的修改已经生效了，那么其之前的读写也必须已经生效了，除此之外没有做任何其他的保证。换句话说，flags只有在其之前的读写都完成后，自己才会被修改，才会变得被其他核心可见。因此，请求端才需要使用while循环去不断轮询这个flags是否变化：
```c
while (atomic_load_explicit(&ready, memory_order_acquire) == 0);
```

因为这个保证的落地完全是异步的。


### 参考资料

- [Armv8-A memory model guide](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/Armv8-A%20memory%20model%20guide.pdf?revision=58b1dd0a-3800-4218-b21a-f95a0332034c)
- [Arm：Memory access ordering in the Arm Architecture](https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/memory-access-ordering-part-3---memory-access-ordering-in-the-arm-architecture)
- [Arm：Implementing sequential consistency with acquire/release](https://developer.arm.com/community/arm-community-blogs/b/tools-software-ides-blog/posts/armv8-sequential-consistency)
- [内存一致性模型](/操作系统/2026/09/22/内存一致性模型/)
- [原子变量和锁的关系](/操作系统/2026/09/02/原子变量和锁的关系/)
