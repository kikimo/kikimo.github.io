---
title: "Linux 中的 wake_q_add() 函数"
date: 2020-02-04T11:14:56+08:00
draft: false
---

`wake_q_add()`是 [Linux 内代码中的一个函数](https://elixir.bootlin.com/linux/latest/source/kernel/sched/core.c#L450)，
它尝试将一个系统进程放置到等待唤醒的队列中：

```C
static bool __wake_q_add(struct wake_q_head *head, struct task_struct *task)
{
	struct wake_q_node *node = &task->wake_q;

	/*
	 * Atomically grab the task, if ->wake_q is !nil already it means
	 * its already queued (either by us or someone else) and will get the
	 * wakeup due to that.
	 *
	 * In order to ensure that a pending wakeup will observe our pending
	 * state, even in the failed case, an explicit smp_mb() must be used.
	 */
	smp_mb__before_atomic();
	if (unlikely(cmpxchg_relaxed(&node->next, NULL, WAKE_Q_TAIL)))
		return false;

	/*
	 * The head is context local, there can be no concurrency.
	 */
	*head->lastp = node;
	head->lastp = &node->next;
	return true;
}

/**
 * wake_q_add() - queue a wakeup for 'later' waking.
 * @head: the wake_q_head to add @task to
 * @task: the task to queue for 'later' wakeup
 *
 * Queue a task for later wakeup, most likely by the wake_up_q() call in the
 * same context, _HOWEVER_ this is not guaranteed, the wakeup can come
 * instantly.
 *
 * This function must be used as-if it were wake_up_process(); IOW the task
 * must be ready to be woken at this location.
 */
void wake_q_add(struct wake_q_head *head, struct task_struct *task)
{
	if (__wake_q_add(head, task))
		get_task_struct(task);
}

/**
 * wake_q_add_safe() - safely queue a wakeup for 'later' waking.
 * @head: the wake_q_head to add @task to
 * @task: the task to queue for 'later' wakeup
 *
 * Queue a task for later wakeup, most likely by the wake_up_q() call in the
 * same context, _HOWEVER_ this is not guaranteed, the wakeup can come
 * instantly.
 *
 * This function must be used as-if it were wake_up_process(); IOW the task
 * must be ready to be woken at this location.
 *
 * This function is essentially a task-safe equivalent to wake_q_add(). Callers
 * that already hold reference to @task can call the 'safe' version and trust
 * wake_q to do the right thing depending whether or not the @task is already
 * queued for wakeup.
 */
void wake_q_add_safe(struct wake_q_head *head, struct task_struct *task)
{
	if (!__wake_q_add(head, task))
		put_task_struct(task);
}

void wake_up_q(struct wake_q_head *head)
{
	struct wake_q_node *node = head->first;

	while (node != WAKE_Q_TAIL) {
		struct task_struct *task;

		task = container_of(node, struct task_struct, wake_q);
		BUG_ON(!task);
		/* Task can safely be re-inserted now: */
		node = node->next;
		task->wake_q.next = NULL;

		/*
		 * wake_up_process() executes a full barrier, which pairs with
		 * the queueing in wake_q_add() so as not to miss wakeups.
		 */
		wake_up_process(task);
		put_task_struct(task);
	}
}
```

它的原形定义在 [wake_q.h](https://elixir.bootlin.com/linux/latest/source/include/linux/sched/wake_q.h) 中：

```C
/* SPDX-License-Identifier: GPL-2.0 */
#ifndef _LINUX_SCHED_WAKE_Q_H
#define _LINUX_SCHED_WAKE_Q_H

/*
 * Wake-queues are lists of tasks with a pending wakeup, whose
 * callers have already marked the task as woken internally,
 * and can thus carry on. A common use case is being able to
 * do the wakeups once the corresponding user lock as been
 * released.
 *
 * We hold reference to each task in the list across the wakeup,
 * thus guaranteeing that the memory is still valid by the time
 * the actual wakeups are performed in wake_up_q().
 *
 * One per task suffices, because there's never a need for a task to be
 * in two wake queues simultaneously; it is forbidden to abandon a task
 * in a wake queue (a call to wake_up_q() _must_ follow), so if a task is
 * already in a wake queue, the wakeup will happen soon and the second
 * waker can just skip it.
 *
 * The DEFINE_WAKE_Q macro declares and initializes the list head.
 * wake_up_q() does NOT reinitialize the list; it's expected to be
 * called near the end of a function. Otherwise, the list can be
 * re-initialized for later re-use by wake_q_init().
 *
 * NOTE that this can cause spurious wakeups. schedule() callers
 * must ensure the call is done inside a loop, confirming that the
 * wakeup condition has in fact occurred.
 *
 * NOTE that there is no guarantee the wakeup will happen any later than the
 * wake_q_add() location. Therefore task must be ready to be woken at the
 * location of the wake_q_add().
 */

#include <linux/sched.h>

struct wake_q_head {
	struct wake_q_node *first;
	struct wake_q_node **lastp;
};

#define WAKE_Q_TAIL ((struct wake_q_node *) 0x01)

#define DEFINE_WAKE_Q(name)				\
	struct wake_q_head name = { WAKE_Q_TAIL, &name.first }

static inline void wake_q_init(struct wake_q_head *head)
{
	head->first = WAKE_Q_TAIL;
	head->lastp = &head->first;
}

static inline bool wake_q_empty(struct wake_q_head *head)
{
	return head->first == WAKE_Q_TAIL;
}

extern void wake_q_add(struct wake_q_head *head, struct task_struct *task);
extern void wake_q_add_safe(struct wake_q_head *head, struct task_struct *task);
extern void wake_up_q(struct wake_q_head *head);

#endif /* _LINUX_SCHED_WAKE_Q_H */
```

从代码上看,`wake_q_add()` 函数的主要是：
1. 判断`task`进程是否已经在某个`wake_q` 队列中（`task->wake_q->next` 是否等于`NULL`）,
如果是，则先将`task->wake_q->next`设置为`WAKE_Q_TAIL`并返回`NULL`，
（`WAKE_Q_TAIL`应该是用于表示队列尾，作用类似于哨兵节点）,
否则函数返回`WAKE_Q_TAIL`，
这步操作使用`cmpxchg`宏保证操作的原子性。

2. 如果`task`不在某个`wake_q`中，进一步将`task`添加到`head`指向的队列`wait_q`中，注意到这里的指针操作，它**通过二级指针操作把一个元素添加到队列尾部**，这个技巧 Linus 曾经在某个帖子里专门提到过。

`wake_q_add()`如果成功吧`task`添加到唤醒队列后会增加`task`的引用，
所以在调用`wake_q_add()`之前调用方应该未持有`task`的引用(`get_task_struct(task);`);
`wake_q_add_safe()`函数作用类似，
只不过如果它添加对队列失败，
它会释放`task`的引用(`put_task_struct(task);`)，
所以调用方在调用前应该已经持有`task`的引用。

和`wake_q`相关的还有一个`wake_up_q()`函数，
这个函数将所有`head`队列中的进程唤醒。
