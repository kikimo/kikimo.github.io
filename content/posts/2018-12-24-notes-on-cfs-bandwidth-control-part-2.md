---
title:  "CFS bandwidth control 笔记（二）"
date: 2019-01-30T23:36:46+08:00
draft: false
---

在[CFS bandwidth control 笔记（一）](https://coderatwork.cn/kernel/2018/12/23/notes-on-cfs-bandwidth-control_part_1.html)中提到一个问题：

>两个形成父子关系的调度组， 他们的 CPU 带宽限制具体这么运行？

对于这个问题，[CFS Bandwidth Control](https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt)
中的```Hierarchical considerations```一节里对进程组的带宽控制有以下描述：

```txt
The interface enforces that an individual entity's bandwidth is always
attainable, that is: max(c_i) <= C. However, over-subscription in the
aggregate case is explicitly allowed to enable work-conserving semantics
within a hierarchy.
  e.g. \Sum (c_i) may exceed C
[ Where C is the parent's bandwidth, and c_i its children ]


There are two ways in which a group may become throttled:
	a. it fully consumes its own quota within a period
	b. a parent's quota is fully consumed within its period

In case b) above, even though the child may have runtime remaining it will not
be allowed to until the parent's runtime is refreshed.
```

根据如上表述，runq 中子节点的 bandwidth 总是小于父节点。
接下来还提到 CFS 具体带宽限流的方式，有两种情况，分别是
    
1. 调度节点耗尽自己的 CPU 配额
2. 父节点 CPU 配额耗尽，父节点被限流直接导致子节点也被限流

第二点已经可以会带我们在开篇里提出的问题了，
接下来我们通过检查代码来确认这一点。
CFS 调度器主体代码在```kernel/sched/fair.c```文件中，
其中具体的限流函数是 ```throttle_cfs_rq()```，
从行数里我们可以看到，```throttle_cfs_rq()```
通过```walk_tg_tree_from()```函数遍历整颗子树，
然后调用```tg_throttle_down()```函数执行限流，
这一点符合我们上面总结的 CFS 中**父节点被限流子节点也被限流**的特点。

```c
3605 static void throttle_cfs_rq(struct cfs_rq *cfs_rq)
3606 {
3607     struct rq *rq = rq_of(cfs_rq);
3608     struct cfs_bandwidth *cfs_b = tg_cfs_bandwidth(cfs_rq->tg);
3609     struct sched_entity *se;
3610     long task_delta, dequeue = 1;
3611     bool empty;
3612 
3613     se = cfs_rq->tg->se[cpu_of(rq_of(cfs_rq))];
3614 
3615     /* freeze hierarchy runnable averages while throttled */
3616     rcu_read_lock();
3617     walk_tg_tree_from(cfs_rq->tg, tg_throttle_down, tg_nop, (void *)rq);
3618     rcu_read_unlock();
3619 
3620     task_delta = cfs_rq->h_nr_running;
3621     for_each_sched_entity(se) {
3622         struct cfs_rq *qcfs_rq = cfs_rq_of(se);
3623         /* throttled entity or throttle-on-deactivate */
3624         if (!se->on_rq)
3625             break;
3626 
3627         if (dequeue)
3628             dequeue_entity(qcfs_rq, se, DEQUEUE_SLEEP);
3629         qcfs_rq->h_nr_running -= task_delta;
3630 
3631         if (qcfs_rq->load.weight)
3632             dequeue = 0;
3633     }
3634 
3635     if (!se)
3636         sub_nr_running(rq, task_delta);
3637 
3638     cfs_rq->throttled = 1;
3639     cfs_rq->throttled_clock = rq_clock(rq);
3640     raw_spin_lock(&cfs_b->lock);
3641     empty = list_empty(&cfs_b->throttled_cfs_rq);
3642 
3643     /*
3644      * Add to the _head_ of the list, so that an already-started
3645      * distribute_cfs_runtime will not see us
3646      */
3647     list_add_rcu(&cfs_rq->throttled_list, &cfs_b->throttled_cfs_rq);
3648 
3649     /*
3650      * If we're the first throttled task, make sure the bandwidth
3651      * timer is running.
3652      */
3653     if (empty)
3654         start_cfs_bandwidth(cfs_b);
3655 
3656     raw_spin_unlock(&cfs_b->lock);
3657 }
```

接下来要搞清楚，CFS 在什么时候、如何判断 task group 需要被限流这个问题。
为了搞清楚这个问题，从```throttle_cfs_rq()```的调用函数开始分析，
我们找到了```check_cfs_rq_runtime()```这个函数，
这个函数比较简单，它检测 task group 的带宽是否已经耗尽，
如果是则调用```throttle_cfs_rq()```将 task group 限流。

```c
3974 /* conditionally throttle active cfs_rq's from put_prev_entity() */
3975 static bool check_cfs_rq_runtime(struct cfs_rq *cfs_rq)
3976 {
3977     if (!cfs_bandwidth_used())
3978         return false;
3979 
3980     if (likely(!cfs_rq->runtime_enabled || cfs_rq->runtime_remaining > 0))
3981         return false;
3982 
3983     /*
3984      * it's possible for a throttled entity to be forced into a running
3985      * state (e.g. set_curr_task), in this case we're finished.
3986      */
3987     if (cfs_rq_throttled(cfs_rq))
3988         return true;
3989 
3990     throttle_cfs_rq(cfs_rq);
3991     return true;
3992 }
```

继续上溯```check_cfs_rq_runtime()```函数的调用者，
我们找到```pick_next_task_fair()```函数，
这个函数其实是 CFS 调度类```fair_sched_class```中的```pick_next_task```字段。
我们只要关注这个函数中的```do {} while```循环即可，
这个循环所的事情其实是一个自顶向下搜索可执行的 task_struct 的过程。
CFS 中调度队列是一颗红黑树，
红黑树的节点是```struct sched_entity```，
```sched_entity```中既可以指向```struct task_struct```
也可以指向```struct cfs_rq```（```struct cfs_rq```可理解为 task group）。
每搜索到一个 task group 后```pick_next_task_fair()```
都会调用```update_curr()```函数先更新他的 CPU 运行时间等数据，
这些数据会直接影响后面```check_cfs_rq_runtime()```函数的限流检测。

```c
5271 static struct task_struct *
5272 pick_next_task_fair(struct rq *rq, struct task_struct *prev)
5273 {
5274     struct cfs_rq *cfs_rq = &rq->cfs;
5275     struct sched_entity *se;
5276     struct task_struct *p;
5277     int new_tasks;
5278 
5279 again:
5280 #ifdef CONFIG_FAIR_GROUP_SCHED
5281     if (!cfs_rq->nr_running)
5282         goto idle;
5283 
5284     if (prev->sched_class != &fair_sched_class)
5285         goto simple;
5286 
5287     /*
5288      * Because of the set_next_buddy() in dequeue_task_fair() it is rather
5289      * likely that a next task is from the same cgroup as the current.
5290      *
5291      * Therefore attempt to avoid putting and setting the entire cgroup
5292      * hierarchy, only change the part that actually changes.
5293      */
5294 
5295     do {
5296         struct sched_entity *curr = cfs_rq->curr;
5297 
5298         /*
5299          * Since we got here without doing put_prev_entity() we also
5300          * have to consider cfs_rq->curr. If it is still a runnable
5301          * entity, update_curr() will update its vruntime, otherwise
5302          * forget we've ever seen it.
5303          */
5304         if (curr) {
5305             if (curr->on_rq)
5306                 update_curr(cfs_rq);
5307             else
5308                 curr = NULL;
5309 
5310             /*
5311              * This call to check_cfs_rq_runtime() will do the
5312              * throttle and dequeue its entity in the parent(s).
5313              * Therefore the 'simple' nr_running test will indeed
5314              * be correct.
5315              */
5316             if (unlikely(check_cfs_rq_runtime(cfs_rq)))
5317                 goto simple;
5318         }
5319 
5320         se = pick_next_entity(cfs_rq, curr);
5321         cfs_rq = group_cfs_rq(se);
5322     } while (cfs_rq);
5323 
5324     p = task_of(se);
5325 
5326     /*
5327      * Since we haven't yet done put_prev_entity and if the selected task
5328      * is a different task than we started out with, try and touch the
5329      * least amount of cfs_rqs.
5330      */
5331     if (prev != p) {
5332         struct sched_entity *pse = &prev->se;
5333 
5334         while (!(cfs_rq = is_same_group(se, pse))) {
5335             int se_depth = se->depth;
5336             int pse_depth = pse->depth;
5337 
5338             if (se_depth <= pse_depth) {
5339                 put_prev_entity(cfs_rq_of(pse), pse);
5340                 pse = parent_entity(pse);
5341             }
5342             if (se_depth >= pse_depth) {
5343                 set_next_entity(cfs_rq_of(se), se);
5344                 se = parent_entity(se);
5345             }
5346         }
5347 
5348         put_prev_entity(cfs_rq, pse);
5349         set_next_entity(cfs_rq, se);
5350     }
5351 
5352     if (hrtick_enabled(rq))
5353         hrtick_start_fair(rq, p);
5354 
5355     return p;
5356 simple:
5357     cfs_rq = &rq->cfs;
5358 #endif
5359 
5360     if (!cfs_rq->nr_running)
5361         goto idle;
5362 
5363     put_prev_task(rq, prev);
5364 
5365     do {
5366         se = pick_next_entity(cfs_rq, NULL);
5367         set_next_entity(cfs_rq, se);
5368         cfs_rq = group_cfs_rq(se);
5369     } while (cfs_rq);
5370 
5371     p = task_of(se);
5372 
5373     if (hrtick_enabled(rq))
5374         hrtick_start_fair(rq, p);
5375 
5376     return p;
5377 
5378 idle:
5379     /*
5380      * This is OK, because current is on_cpu, which avoids it being picked
5381      * for load-balance and preemption/IRQs are still disabled avoiding
5382      * further scheduler activity on it and we're being very careful to
5383      * re-start the picking loop.
5384      */
5385     lockdep_unpin_lock(&rq->lock);
5386     new_tasks = idle_balance(rq);
5387     lockdep_pin_lock(&rq->lock);
5388     /*
5389      * Because idle_balance() releases (and re-acquires) rq->lock, it is
5390      * possible for any higher priority task to appear. In that case we
5391      * must re-start the pick_next_entity() loop.
5392      */
5393     if (new_tasks < 0)
5394         return RETRY_TASK;
5395 
5396     if (new_tasks > 0)
5397         goto again;
5398 
5399     return NULL;
5400 }
```

