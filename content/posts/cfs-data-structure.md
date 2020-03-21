---
title: "CFS 调度算法中的数据结构"
date: 2020-03-21T16:20:23+08:00
draft: false
---

CFS 是当前 Linux 内核中默认使用的调度算法，它的全称是 Complete Fair Scheduler，中文的意思是完全公平调度算法。
如这个名字所体现出来的意思，CFS 调度算法尽量公平的为每个进程分配 CPU 时间。
CFS 算法的核心并不发复杂，它跟踪每个进程的 CPU 时间，每次调度总是选择当前 CPU 运行时间最小的进程。
在 CFS 出现之前，Linux 使用 O(1) 调度算法，这种算法为每个进程分配一个 nice value，
然后根据 nice value 分配进程的运行时间片。
这种算法存在诸多问题，其中之一是：优先级低的进程分配的时间片较小，
当系统运行的都是优先级低的进程时将会出现频繁的上下文切换，空耗 CPU 资源。
除此之外，O(1) 调度算法在进程优先级的计算上使用了很多难以理解的经验公式，
这些计算公式也许是有效的，但是人们无法解释他们是如何起作用的，这给系统的维护升级都带来麻烦。
CFS 的出现解决了这些问题，同时也为后来 CGroup 的 CPU 资源隔离打下基础。

早先 Linux 内核中只有一个调度队列，后来为了提高多核场景下的并发效率，改成每个 CPU 单独分配一个调度队列。

```C
/*
 * This is the main, per-CPU runqueue data structure.
 *
 * Locking rule: those places that want to lock multiple runqueues
 * (such as the load balancing or the thread migration code), lock
 * acquire operations must be ordered by ascending &runqueue.
 */
struct rq {
    /*
     * nr_running and cpu_load should be in the same cacheline because
     * remote CPUs use both these fields when doing load calculation.
     */
    unsigned int nr_running;

    struct cfs_rq cfs;
    struct rt_rq rt;

#ifdef CONFIG_FAIR_GROUP_SCHED
    /* list of leaf cfs_rq on this cpu: */
    struct list_head leaf_cfs_rq_list;
#ifdef CONFIG_SMP
    unsigned long h_load_throttle;
#endif /* CONFIG_SMP */
#endif /* CONFIG_FAIR_GROUP_SCHED */

#ifdef CONFIG_RT_GROUP_SCHED
    struct list_head leaf_rt_rq_list;
#endif

    struct task_struct *curr, *idle, *stop;

#ifdef CONFIG_SMP
    struct root_domain *rd;
    struct sched_domain *sd;

    /* cpu of this runqueue: */
    int cpu;
    int online;

    struct list_head cfs_tasks;
#endif

#ifdef CONFIG_SMP
    struct llist_head wake_list;
#endif
};
```

调度队列实际上并不保存在`struct rq`中。
Linux 支持 CFS、RT 等不同的调度算法。
每种调度算法可能会使用不同的数据结构来实现调度队列，
因此我们在`struct rq`结构体中看到`struct cfs_rq cfs`和`struct rt_rq rt`，
这两个结构体字段分别为 CFS 和 RT 调度算法的调度队列。
我们主要关心 CFS 的调度算法实现，
所以我们重点关注`struct cfs_rq`这个结构体：

```C
/* CFS-related fields in a runqueue */
struct cfs_rq {
    struct load_weight load;
    unsigned int nr_running, h_nr_running;

    u64 min_vruntime;
#ifndef CONFIG_64BIT
    u64 min_vruntime_copy;
#endif

    struct rb_root tasks_timeline;
    struct rb_node *rb_leftmost;

    /*
     * 'curr' points to currently running entity on this cfs_rq.
     * It is set to NULL otherwise (i.e when none are currently running).
     */
    struct sched_entity *curr, *next, *last, *skip;

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct rq *rq;  /* cpu runqueue to which this cfs_rq is attached */

    /*
     * leaf cfs_rqs are those that hold tasks (lowest schedulable entity in
     * a hierarchy). Non-leaf lrqs hold other higher schedulable entities
     * (like users, containers etc.)
     *
     * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a cpu. This
     * list is used during load balance.
     */
    int on_list;
    struct list_head leaf_cfs_rq_list;
    struct task_group *tg;  /* group that "owns" this runqueue */

#ifdef CONFIG_CFS_BANDWIDTH
    int runtime_enabled;
    u64 runtime_expires;
    s64 runtime_remaining;

    u64 throttled_clock, throttled_clock_task;
    u64 throttled_clock_task_time;
    int throttled, throttle_count;
    struct list_head throttled_list;
#endif /* CONFIG_CFS_BANDWIDTH */
#endif /* CONFIG_FAIR_GROUP_SCHED */
};
```

CFS 的调度队列使用红黑树实现。
上面我们提到过，CFS 选择调度队列中 CPU 运行时间最小的进程来执行，
采用红黑树可以快速找出调度队列中运行时间最小的进程。
`struct rb_root tasks_timeline`就是红黑树调度队列，
`struct rb_node *rb_leftmost`缓存下一次将要被调度运行的进程。
红黑数中存储的调度实体使用`struct sched_entity`来表示，
为什么不是表示进程的结构体`struct task_struct`？
这是因为 CFS 支持进程组调度，CFS 的调度实体指向的既可能是一个进程也可能是一个进程组。
当`struct sched_entity`结构体中的`struct cfs_rq       *my_q`字段不为空时，
它只想的是一个进程组`struct task_group`，`struct cfs_rq *my_q`则是`struct task_group`结构体中嵌套的一个 CFS 调度队列。

```C
struct sched_entity {
    struct load_weight  load;       /* for load-balancing */
    struct rb_node      run_node;
    struct list_head    group_node;
    unsigned int        on_rq;

    u64         exec_start;
    u64         sum_exec_runtime;
    u64         vruntime;
    u64         prev_sum_exec_runtime;

    u64         nr_migrations;

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct sched_entity *parent;
    /* rq on which this entity is (to be) queued: */
    struct cfs_rq       *cfs_rq;
    /* rq "owned" by this entity/group: */
    struct cfs_rq       *my_q;
#endif

/*
 * Load-tracking only depends on SMP, FAIR_GROUP_SCHED dependency below may be
 * removed when useful for applications beyond shares distribution (e.g.
 * load-balance).
 */
#if defined(CONFIG_SMP) && defined(CONFIG_FAIR_GROUP_SCHED)
    /* Per-entity load-tracking */
    struct sched_avg    avg;
#endif

    RH_KABI_USE(1, struct sched_statistics *statistics)

    /* reserved for Red Hat */
    RH_KABI_RESERVE(2)
    RH_KABI_RESERVE(3)
    RH_KABI_RESERVE(4)
};
```
`struct sched_entity`中的`vruntime`字段既用来表示 CPU 运行时间，他是红黑树中用来排序的 key。
再来看进程结构体`strut task_struct`中和 CFS 有关的字段：
```C
struct task_struct {
    int on_rq;
    struct sched_entity se;
    struct sched_rt_entity rt;

    struct task_group *sched_task_group;
}
```
可以看到`task_struct`结构体中包含了`schec_entity`，
当我们知道一个`sched_entity`指向的是`task_struct`之后就可以查找它关联的到`task_struct`结构体了。
最后我们看`struct task_group`结构体，我们已经知道这个结构体表示一个进程组，它也有自己的调度队列。

```C
/* task group related information */
struct task_group {
#ifdef CONFIG_FAIR_GROUP_SCHED
    /* schedulable entities of this group on each cpu */
    struct sched_entity **se;
    /* runqueue "owned" by this group on each cpu */
    struct cfs_rq **cfs_rq;
    unsigned long shares;
#endif
    struct task_group *parent;
    struct list_head siblings;
    struct list_head children;

    struct cfs_bandwidth cfs_bandwidth;
};
```

当 CFS 选择的是一个进程组时，它会递归的从这个进程组的调度队列中选择一个进程来运行。

```C
static struct task_struct *pick_next_task_fair(struct rq *rq)
{
    struct task_struct *p;
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;

    if (!cfs_rq->nr_running)
        return NULL;

    do {
        se = pick_next_entity(cfs_rq);
        set_next_entity(cfs_rq, se);
        cfs_rq = group_cfs_rq(se);
    } while (cfs_rq);

    p = task_of(se);
    if (hrtick_enabled(rq))
        hrtick_start_fair(rq, p);

    return p;
}
```

一个进程组包含多个进程，这些进程可能同时运行于不同的 CPU 上。
为了适应多核的场景，`task_group`为每个核都分配了`cfs_rq`和`sched_entity`。
如此，便可以让不同 CPU 调度自己队列中的进程互不影响。
一个`task_group`拥有多个 CPU 上的调度队列，这一点可能不是很好理解。
我们可以这样理解，不管是`task_struct`还是`task_group`，
每个 CPU 上调度队列的调度实体都是`task_entity`，
一个`task_group`为每个 CPU 都分配了`sched_entity`，
实际上就是，不同的`sched_entity`可能只想相同的`task_group`，他们都是`task_group`中`se`数组中的元素；
同时`sched_entity`中的`my_q`字段指向对应的 CPU 调度对垒；所有这些操作都是在`init_tg_cfs_entry()`函数中完成的：

```C
int alloc_fair_sched_group(struct task_group *tg, struct task_group *parent)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se;
    int i;

    tg->cfs_rq = kzalloc(sizeof(cfs_rq) * nr_cpu_ids, GFP_KERNEL);
    if (!tg->cfs_rq)
        goto err;
    tg->se = kzalloc(sizeof(se) * nr_cpu_ids, GFP_KERNEL);
    if (!tg->se)
        goto err;

    tg->shares = NICE_0_LOAD;

    init_cfs_bandwidth(tg_cfs_bandwidth(tg));

    for_each_possible_cpu(i) {
        cfs_rq = kzalloc_node(sizeof(struct cfs_rq),
                      GFP_KERNEL, cpu_to_node(i));
        if (!cfs_rq)
            goto err;

        se = kzalloc_node(sizeof(struct sched_entity),
                  GFP_KERNEL, cpu_to_node(i));
        if (!se)
            goto err_free_rq;

        se->statistics = kzalloc_node(sizeof(struct sched_statistics),
                          GFP_KERNEL, cpu_to_node(i));
        if (!se->statistics)
            goto err_free_se;

        init_cfs_rq(cfs_rq);
        init_tg_cfs_entry(tg, cfs_rq, se, i, parent->se[i]);
    }
    ...
}

void init_tg_cfs_entry(struct task_group *tg, struct cfs_rq *cfs_rq,
            struct sched_entity *se, int cpu,
            struct sched_entity *parent)
{
    struct rq *rq = cpu_rq(cpu);

    cfs_rq->tg = tg;
    cfs_rq->rq = rq;
    init_cfs_rq_runtime(cfs_rq);

    tg->cfs_rq[cpu] = cfs_rq;
    tg->se[cpu] = se;

    /* se could be NULL for root_task_group */
    if (!se)
        return;

    if (!parent)
        se->cfs_rq = &rq->cfs;
    else
        se->cfs_rq = parent->my_q;

    se->my_q = cfs_rq;
    /* guarantee group entities always have weight */
    update_load_set(&se->load, NICE_0_LOAD);
    se->parent = parent;
}
```
