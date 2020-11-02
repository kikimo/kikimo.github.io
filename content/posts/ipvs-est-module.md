---
title: "ipvs 速率统计模块分析"
date: 2020-11-02T13:38:54+08:00
draft: false
---

ip_vs_est.c 是 ipvs 中负责速率统计的模块，我们以内核 4.19.12 版本的为例，分析这个模块的源码。
这个模块的代码量不多总共就 203 行。
结构上也简单清晰，可以分成两部分来看：

1. 一部分是定时器相关代码，这部分代码启动一个内核定时器，每两秒钟统计一遍 ipvs 相关数据
2. 具体的 ipvs 数据统计代码

## 定时器相关代码

定时器相关代码主要有以下几个函数组成

1. 定时器清理函数

```c
void __net_exit ip_vs_estimator_net_cleanup(struct netns_ipvs *ipvs)
{
    del_timer_sync(&ipvs->est_timer);
}
```

2. 定时器初始化函数

```c
int __net_init ip_vs_estimator_net_init(struct netns_ipvs *ipvs)
{
    INIT_LIST_HEAD(&ipvs->est_list);
    spin_lock_init(&ipvs->est_lock);
    timer_setup(&ipvs->est_timer, estimation_timer, 0);
    mod_timer(&ipvs->est_timer, jiffies + 2 * HZ);
    return 0;
}
```

3. ipvs 统计数据读取函数

```c
/* Get decoded rates */
void ip_vs_read_estimator(struct ip_vs_kstats *dst, struct ip_vs_stats *stats)
{
    struct ip_vs_estimator *e = &stats->est;

    dst->cps = (e->cps + 0x1FF) >> 10;
    dst->inpps = (e->inpps + 0x1FF) >> 10;
    dst->outpps = (e->outpps + 0x1FF) >> 10;
    dst->inbps = (e->inbps + 0xF) >> 5;
    dst->outbps = (e->outbps + 0xF) >> 5;
}
```

4. 统计数据清零函数

```c
void ip_vs_zero_estimator(struct ip_vs_stats *stats)
{
    struct ip_vs_estimator *est = &stats->est;
    struct ip_vs_kstats *k = &stats->kstats;

    /* reset counters, caller must hold the stats->lock lock */
    est->last_inbytes = k->inbytes;
    est->last_outbytes = k->outbytes;
    est->last_conns = k->conns;
    est->last_inpkts = k->inpkts;
    est->last_outpkts = k->outpkts;
    est->cps = 0;
    est->inpps = 0;
    est->outpps = 0;
    est->inbps = 0;
    est->outbps = 0;
}
```

5. 把 ipvs 对象加入统计链表

```c

void ip_vs_start_estimator(struct netns_ipvs *ipvs, struct ip_vs_stats *stats)
{
    struct ip_vs_estimator *est = &stats->est;

    INIT_LIST_HEAD(&est->list);

    spin_lock_bh(&ipvs->est_lock);
    list_add(&est->list, &ipvs->est_list);
    spin_unlock_bh(&ipvs->est_lock);
}
```

6. 把 ipvs 对象从统计链表中删除

```c
void ip_vs_stop_estimator(struct netns_ipvs *ipvs, struct ip_vs_stats *stats)
{
    struct ip_vs_estimator *est = &stats->est;

    spin_lock_bh(&ipvs->est_lock);
    list_del(&est->list);
    spin_unlock_bh(&ipvs->est_lock);
}
```

## ipvs 数据统计代码

执行 ipvs 数据统计的主要有两个函数，`estimation_timer()`和`ip_vs_read_cpu_stats()`。
`ip_vs_read_cpu_stats()`分别统计每个 CPU 上的信息，

```c
/*
 * Make a summary from each cpu
 */
static void ip_vs_read_cpu_stats(struct ip_vs_kstats *sum,
                 struct ip_vs_cpu_stats __percpu *stats)
{
    int i;
    bool add = false;

    for_each_possible_cpu(i) {
        struct ip_vs_cpu_stats *s = per_cpu_ptr(stats, i);
        unsigned int start;
        u64 conns, inpkts, outpkts, inbytes, outbytes;

        if (add) {
            do {
                start = u64_stats_fetch_begin(&s->syncp);
                conns = s->cnt.conns;
                inpkts = s->cnt.inpkts;
                outpkts = s->cnt.outpkts;
                inbytes = s->cnt.inbytes;
                outbytes = s->cnt.outbytes;
            } while (u64_stats_fetch_retry(&s->syncp, start));
            sum->conns += conns;
            sum->inpkts += inpkts;
            sum->outpkts += outpkts;
            sum->inbytes += inbytes;
            sum->outbytes += outbytes;
        } else {
            add = true;
            do {
                start = u64_stats_fetch_begin(&s->syncp);
                sum->conns = s->cnt.conns;
                sum->inpkts = s->cnt.inpkts;
                sum->outpkts = s->cnt.outpkts;
                sum->inbytes = s->cnt.inbytes;
                sum->outbytes = s->cnt.outbytes;
            } while (u64_stats_fetch_retry(&s->syncp, start));
        }
    }
}
```

ipvs 在`estimation_timer()`是定时器调的回调函数，
这个函数是执行数据统计的主体，
这个函数遍历 ipvs 列表中的每个 ipvs 对象，分别统计他们的信息。
对每个 ipvs 它首先调用`ip_vs_read_cpu_stats()`统计每个 cpu 上的信息，
然后分别统计连接速率、输入包速率、输出包速率、输入流量速率、输出流量速率。
我们在代码中会看到对数据左移九位和左移四位的操作，
这是因为统计单位分别为`2^10`和`2^4`的缘故。
再看具体的速率统计公式：

```txt
avgrate = avgrate * 1/4 + rate * 3/4
```

其中 avgrate 是八秒内的速率统计（参看参考 1，其实我没搞明白这个八秒是怎么来的），
这个公式看起来和代码不太一致，其实把公式延展一下就可以看出他们是等价的，以连接速率 e->cps 的统计为例：

```txt
e->cps += ((s64)rate - (s64)e->cps) >> 2

等价于：

e->cps = e->cps + (rate - e->cps) / 4 = rate * 1/4 + rate * 3/4
```

以上 rate 就是我们前文提到的八秒内的速率 avgrate。

```c
static void estimation_timer(struct timer_list *t)
{
    struct ip_vs_estimator *e;
    struct ip_vs_stats *s;
    u64 rate;
    struct netns_ipvs *ipvs = from_timer(ipvs, t, est_timer);

    spin_lock(&ipvs->est_lock);
    list_for_each_entry(e, &ipvs->est_list, list) {
        s = container_of(e, struct ip_vs_stats, est);

        spin_lock(&s->lock);
        ip_vs_read_cpu_stats(&s->kstats, s->cpustats);

        /* scaled by 2^10, but divided 2 seconds */
        rate = (s->kstats.conns - e->last_conns) << 9;
        e->last_conns = s->kstats.conns;
        e->cps += ((s64)rate - (s64)e->cps) >> 2;

        rate = (s->kstats.inpkts - e->last_inpkts) << 9;
        e->last_inpkts = s->kstats.inpkts;
        e->inpps += ((s64)rate - (s64)e->inpps) >> 2;

        rate = (s->kstats.outpkts - e->last_outpkts) << 9;
        e->last_outpkts = s->kstats.outpkts;
        e->outpps += ((s64)rate - (s64)e->outpps) >> 2;

        /* scaled by 2^5, but divided 2 seconds */
        rate = (s->kstats.inbytes - e->last_inbytes) << 4;
        e->last_inbytes = s->kstats.inbytes;
        e->inbps += ((s64)rate - (s64)e->inbps) >> 2;

        rate = (s->kstats.outbytes - e->last_outbytes) << 4;
        e->last_outbytes = s->kstats.outbytes;
        e->outbps += ((s64)rate - (s64)e->outbps) >> 2;
        spin_unlock(&s->lock);
    }
    spin_unlock(&ipvs->est_lock);
    mod_timer(&ipvs->est_timer, jiffies + 2*HZ);
}
```

---

参考：

1. [Rate Computation of Data Relevant to IPVS Receiving and Sending](https://programmer.group/rate-computation-of-data-relevant-to-ipvs-receiving-and-sending.html)


