---
title: "Linux 内核读写锁 rwsem 分析"
date: 2020-02-16T14:56:06+08:00
draft: true
---

`rwsem` 是 Linux 内核中的读写锁，
它应该是读写信号量的缩写（read-write semaphore）。
`rwsem`的特点是读者之间不互斥，写者既和读者互斥也和其他写者互斥。
`rwsem` 的操作接口定义在 `include/linux/rwsem.h` 文件中，
`rwsem`的结构体定义为`struct rw_semaphore`：
```C
/* All arch specific implementations share the same struct */
struct rw_semaphore {
    RH_KABI_REPLACE(long        count,
            atomic_long_t   count)
    raw_spinlock_t  wait_lock;
#if defined(CONFIG_RWSEM_SPIN_ON_OWNER) && !defined(__GENKSYMS__)
    struct optimistic_spin_queue osq; /* spinner MCS lock */
    struct slist_head   wait_list;
    /*
     * Write owner. Used as a speculative check to see
     * if the owner is running on the cpu.
     */
    struct task_struct  *owner;
#else
    struct list_head    wait_list;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```

核心操作接口为：

```C
/*
 * lock for reading
 */
extern void down_read(struct rw_semaphore *sem);

/*
 * trylock for reading -- returns 1 if successful, 0 if contention
 */
extern int down_read_trylock(struct rw_semaphore *sem);

/*
 * lock for writing
 */
extern void down_write(struct rw_semaphore *sem);

/*
 * trylock for writing -- returns 1 if successful, 0 if contention
 */
extern int down_write_trylock(struct rw_semaphore *sem);

/*
 * release a read lock
 */
extern void up_read(struct rw_semaphore *sem);

/*
 * release a write lock
 */
extern void up_write(struct rw_semaphore *sem);

/*
 * downgrade write lock to read lock
 */
extern void downgrade_write(struct rw_semaphore *sem);
```

接口的实现代码`kernel/rwsem.c`文件中，
分析`down_read()`的实现：

```C
/*
 * lock for reading
 */
void __sched down_read(struct rw_semaphore *sem)
{
    might_sleep();
    rwsem_acquire_read(&sem->dep_map, 0, 0, _RET_IP_);

    LOCK_CONTENDED(sem, __down_read_trylock, __down_read);
    rwsem_set_reader_owned(sem);
}
```

`down_read()`核心在于`LOCK_CONTENDED(sem, __down_read_trylock, __down_read);`，
其他跟语句可以先忽略。

```C
#define LOCK_CONTENDED(_lock, try, lock)            \
do {                                \
    if (!try(_lock)) {                  \
        lock_contended(&(_lock)->dep_map, _RET_IP_);    \
        lock(_lock);                    \
    }                           \
    lock_acquired(&(_lock)->dep_map, _RET_IP_);         \
} while (0)
```

`LOCK_CONTENDED`宏先调用`__down_read_trylock()`尝试获取读者锁，
失败的话继续调用`__down_read()`。
`__down_read_trylock()`的实现跟平台相关，
我们看 x86 平台下的代码，代码路径在`arch/x86/include/asm/rwsem.h`：

```C
/*
 * trylock for reading -- returns 1 if successful, 0 if contention
 */
static inline int __down_read_trylock(struct rw_semaphore *sem)
{
    long result, tmp;
    asm volatile("# beginning __down_read_trylock\n\t"
             "  mov          %0,%1\n\t"
             "1:\n\t"
             "  mov          %1,%2\n\t"
             "  add          %3,%2\n\t"
             "  jle      2f\n\t"
             LOCK_PREFIX "  cmpxchg  %2,%0\n\t"
             "  jnz      1b\n\t"
             "2:\n\t"
             "# ending __down_read_trylock\n\t"
             : "+m" (sem->count), "=&a" (result), "=&r" (tmp)
             : "i" (RWSEM_ACTIVE_READ_BIAS)
             : "memory", "cc");
    return result >= 0 ? 1 : 0;
}
```

`__down_read_trylock()`用内联汇编实现，
核心是 x86 的`cmpxchg`指令。
