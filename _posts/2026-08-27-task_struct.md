---
layout:       post
title:        "task_struct"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: 操作系统
tags:
    - "进程"
---
这个结构体就是Linux系统下的任务控制块，进程和线程都使用这个结构体来描述。他包含了一个process的所有可能的信息，和多个子系统产生联系。原始的结构体定义十分长，这里不可能把他们一一贴出来。我们仅展示一些比较关键的字段，这些字段都是各个子系统比较重要的字段。
```c
// ========== include/linux/sched.h (节选) ==========

struct task_struct {
struct task_struct {
    // ==================== 基础标识与状态 ====================
    // 这些字段通常由更高级的机制保护（如 RCU、原子操作），
    // 但有些也间接与 alloc_lock 相关（如 exit_state 的修改）。
    pid_t               pid;
    pid_t               tgid;
    unsigned int        __state;        // 进程状态，由调度器和信号系统共同管理
    int                 exit_state;
    int                 exit_code;
    int                 exit_signal;
    unsigned int        flags;          // 进程标志，部分由 alloc_lock 保护（如 PF_EXITING）

    // ==================== 资源指针与凭证 ====================
    // 以下指针字段由 alloc_lock 保护。
    // 修改这些指针（例如在 execve、exit 或 fork 中）时必须持有 alloc_lock。
    spinlock_t          alloc_lock;     // 保护下面列出的资源指针和某些状态字段

    struct mm_struct    *mm;            // 用户地址空间
    struct mm_struct    *active_mm;     // 内核线程借用的地址空间
    struct fs_struct    *fs;            // 文件系统信息（根目录、cwd 等）
    struct files_struct *files;         // 打开文件表
    struct nsproxy      *nsproxy;       // 命名空间代理
    struct signal_struct *signal;       // 线程组信号信息
    struct sighand_struct *sighand;     // 信号处理函数表
    sigset_t            blocked;        // 阻塞信号掩码（与信号子系统交互）
    sigset_t            real_blocked;   // 临时保存的阻塞掩码
    const struct cred   *real_cred;     // 真实凭证
    const struct cred   *cred;          // 有效凭证
    void                *security;      // LSM 安全模块私有数据
    unsigned int        no_new_privs;   // 禁止获取新特权标志
    struct seccomp      seccomp;        // seccomp 过滤状态
    int                 pdeath_signal;  // 父进程死亡时发送的信号

    // ==================== 调度与优先级 ====================
    // 以下字段由 pi_lock 保护。
    // 修改进程的调度参数、优先级、调度类或 CPU 亲和性时必须持有 pi_lock。
    raw_spinlock_t      pi_lock;        // 保护下面列出的调度相关字段

    int                 prio;           // 动态优先级
    int                 static_prio;    // 静态优先级
    int                 normal_prio;    // 归一化优先级
    unsigned int        rt_priority;    // 实时优先级
    unsigned int        policy;         // 调度策略
    const struct sched_class *sched_class;  // 调度类指针
    struct sched_entity se;             // CFS 调度实体
    struct sched_rt_entity rt;          // RT 调度实体
    struct sched_dl_entity dl;          // Deadline 调度实体
    cpumask_t           cpus_mask;      // 允许运行的 CPU 掩码
    int                 on_rq;          // 是否在运行队列
    int                 on_cpu;         // 是否在 CPU 上

    // 优先级继承相关（也由 pi_lock 保护）
    struct rb_root_cached pi_waiters;   // PI 等待队列
    // 注意：pi_blocked_on 和 pi_top_task 也由 pi_lock 保护，此处省略

    // ==================== 其他字段 ====================
    // 时间统计（通常使用原子操作或专门的记账锁）
    u64                 utime;
    u64                 stime;
    u64                 start_time;
    u64                 real_start_time;

    // 缺页统计（使用原子操作）
    unsigned long       min_flt;
    unsigned long       maj_flt;

    // 链表节点（由全局进程链表锁保护）
    struct list_head    tasks;

    // 进程关系（由 tasklist_lock 保护）
    struct task_struct  *real_parent;
    struct task_struct  *parent;
    struct list_head    children;
    struct list_head    sibling;
    struct task_struct  *group_leader;

    // ...... 其他很多字段
};
};
```

可以看到，这个结构体涉及到了调度子系统，内存管理子系统，信号子系统等，还包含了大量的生命周期字段和状态字段。

涉及到了如此多的字段和子系统，Linux是怎么做好这些量的并发保护呢？总体而言，Linux采用的方案是子系统内聚+粗锁获取子系统token的形式来保护的。
比较重要的粗锁是其中的`alloc_lock`和`pi_lock`. `alloc_lock`主要用于保护信号、内存管理等模块，而`pi_lock`则用于保护进程优先级和调度器相关的模块。

所谓的子系统内聚，就是把子系统相关的字段单独封装为一个结构体，而不是在`task_struct`中分散开。并且，在这些子系统自身的结构体内，也会有锁变量的存在。这个锁是用来保护子系统结构体内部字段的并发访问的。也就是说，各个子系统内部的字段保护责任和权利被下放到了各个子系统自身来完成，而不是通过`task_struct`一把大锁来统一保护。

所谓的粗锁获取子系统token, 就是在获取某一些子结构体时，短暂持有父结构体的锁，然后在锁上下文快速获取子结构体的访问权限，比如说可以是指针形式。获取完之后，就快速释放父结构体的锁。比如说，Linux中的`get_task_mm()`:
```c
// include/linux/sched.h
#define task_lock(task)		spin_lock(&(task)->alloc_lock)
#define task_unlock(task)	spin_unlock(&(task)->alloc_lock)

struct mm_struct *get_task_mm(struct task_struct *task)
{
	struct mm_struct *mm;

	task_lock(task);
	mm = task->mm;
	if (mm) {
		if (task->flags & PF_KTHREAD)
			mm = NULL;
		else
			atomic_inc(&mm->mm_users);
	}
	task_unlock(task);
	return mm;
}
```

在`task_lock(task)`和`task_unlock(task)`期间， 获取`mm`结构体的指针，然后原子性地递增它的引用计数。在引用计数未归0前，这个结构体不会被释放。至于为什么task_lock等于操作`alloc_lock`， 这应该是历史遗留问题，这里不展开讨论。其命名并不影响我们所讨论的策略。

简单来说，父结构体的短锁获取子结构体的token就是**获取指针+原子性递增引用计数**的结合。

这种短锁长锁相结合的方案有诸多优点：
#### 显著降低锁竞争
大锁使得访问每一个需要保护的字段都要去获取锁，在字段高达上百个的`task_struct`下，锁的竞争就会十分严重，并发度严重受限。 如果有进程持锁执行长时间的I/O, 甚至休眠，就会带来更大的安全风险。 通过粗锁和细琐的结合，在粗锁上下文快速获取子资源的引用后，就释放粗锁; 针对子资源的并发控制由嵌在子资源内部的细琐完成，这样子就实现了高并发度。

#### 允许使用不同的锁适应不同场景
一些场景需要带锁睡眠，比如I/O。 一些则不允许，比如原子上下文，中断上下文。通过粗锁和细琐的结合，大家只要认清楚自己的上下文属于哪种类型的，使用对应类型的锁即可，比较灵活。

#### 降低死锁风险
如果只持有一把大锁，那么大锁的平均上下文大概率是比较长的，这样子，不同的执行流对于锁的获取顺序可能刚好符合死锁条件，并且他们的锁上下文极大概率会重叠，大大增加死锁风险。而使用粗锁加细琐的组合，粗锁的上下文极短，重叠的概率极低; 细琐嵌入在子资源中，所有对子资源的访问都需要经过细琐，而在子资源下，影响范围受限，层次比较清晰，可以比较好地设计来避免死锁。

#### 提高缓存局部性，降低缓存行争用
在多核系统中，锁本身是存放于内存某个位置的一个变量。频繁并发地读写(更重要的是写，读本身不会触发缓存一致性协议)内存的同一个位置，那么该锁所在的缓存行会成为热点，反复触发缓存一致性协议，总线中会占据大量的同步流量，严重影响了系统的性能。
https://cloud.tencent.com/developer/article/2416612
