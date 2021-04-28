# 一 dpdk timer简介
dpdk timer的使用参考example/timer/main.c文件
timer库为DPDK执行单元提供timer服务，以便异步执行回调函数。timer库的特点是：
- timer可以是周期（多发）或单发（单发）。
- timer可以从一个核心加载并在另一个核心上执行。必须在调用rte_timer_reset()时指定。
- timer提供高精度（取决于对检查本地核心计时器过期的rte_timer_manage()的调用频率）。
- 如果应用程序中不需要timer，则可以在编译时通过不调用rte_timer_manage()来禁用timer，以提高性能。 

timer库使用rte_get_timer_cycles()函数，该函数使用高精度事件计时器（HPET）或CPU时间戳计数器（TSC）提供可靠的时间参考。

这个库提供了添加、删除和重新启动计时器的接口。该API基于BSD callout()，但有一些不同。请参阅callout手册<http://www.daemon-systems.org/man/callout.9.html>。

重要函数：
- rte_timer_reset(): reset并重新安装timer的handler，然后start。
- rte_timer_manage(): 管理并执行到期的timer的handler，定时器的精度依赖于此函数被调用的频率。
- rte_timer_subsystem_init()：初始化timer库，如果要使用timer库，这个函数必须先调用。
- rte_timer_init()：初始化一个timer。
- rte_timer_stop(): 停用一个timer。


```
/**
 * Reset and start the timer associated with the timer handle.
 *
 * The rte_timer_reset() function resets and starts the timer
 * associated with the timer handle *tim*. When the timer expires after
 * *ticks* HPET cycles, the function specified by *fct* will be called
 * with the argument *arg* on core *tim_lcore*.
 *
 * If the timer associated with the timer handle is already running
 * (in the RUNNING state), the function will fail. The user has to check
 * the return value of the function to see if there is a chance that the
 * timer is in the RUNNING state.
 *
 * If the timer is being configured on another core (the CONFIG state),
 * it will also fail.
 *
 * If the timer is pending or stopped, it will be rescheduled with the
 * new parameters.
 *
 * @param tim
 *   The timer handle.
 * @param ticks
 *   The number of cycles (see rte_get_hpet_hz()) before the callback
 *   function is called.
 * @param type
 *   The type can be either:
 *   - PERIODICAL: The timer is automatically reloaded after execution
 *     (returns to the PENDING state)
 *   - SINGLE: The timer is one-shot, that is, the timer goes to a
 *     STOPPED state after execution.
 * @param tim_lcore
 *   The ID of the lcore where the timer callback function has to be
 *   executed. If tim_lcore is LCORE_ID_ANY, the timer library will
 *   launch it on a different core for each call (round-robin).
 * @param fct
 *   The callback function of the timer.
 * @param arg
 *   The user argument of the callback function.
 * @return
 *   - 0: Success; the timer is scheduled.
 *   - (-1): Timer is in the RUNNING or CONFIG state.
 */
int rte_timer_reset(struct rte_timer *tim, uint64_t ticks,
		    enum rte_timer_type type, unsigned tim_lcore,
		    rte_timer_cb_t fct, void *arg);
```


# 二 dpvs timer
dpvs的timer子系统对dpdk的timer库进行了简单的封装。定义了timer_scheduler结构体，这个结构体封装了一个rte timer，其handler即dpvs的调度函数rte_timer_tick_cb。调度函数会在master和所有的slave core上以1ms的间隔周期执行。

调度器rte_timer_tick_cb()使用了两个重要的结构体：
timer_scheduler和dpvs_timer：

```
struct timer_scheduler {
    /* wheels and cursors */
    rte_spinlock_t      lock;
    uint32_t            cursors[LEVEL_DEPTH];
    struct list_head    *hashs[LEVEL_DEPTH];

    /* leverage dpdk rte_timer to drive us */
    struct rte_timer    rte_tim;
};

/* it's internal struct, user should never modify it directly. */
struct dpvs_timer {
    struct list_head    list;

#ifdef CONFIG_TIMER_DEBUG
    struct list_head    dummy;
#endif

    dpvs_timer_cb_t     handler;
    void                *priv;
    bool                is_period;

    /*
     * 'delay' for one-short timer
     * 'interval' for periodic timer.
     */
    dpvs_tick_t         delay;
};
```

timer_scheduler结构体是调度器的数据部分。hashs表示一个长度为LEVEL_SIZE的数组，这个数组可以想象成时钟的表盘秒针的格子（实际单位不是秒），cursors可以想象成时钟的秒针。cursors始终指向hashs的某一个位置。timer是周期触发的，每触发一次，cursors就往前走一格，就像秒针走了一秒一样。等timer走到最后一格后，下一格指向成0。每一个timer周期触发的时候，先移动一下指针，然后对指针指向的数据进行处理，即链表hashs下的数据。链表hashs下的数据都是dpvs_timer结构体封装的。

下面对关键函数进行说明：
- rte_timer_tick_cb()：timer到期后的回调函数，即dpvs的timer调度器
- dpvs_timer_update()：update dpvs timer，将timer设置的时间戳更新，在连接老化的时候，一般用来重新设置延迟时间为老化时间。
- dpvs_timer_sched(): add dpvs timer，新的timer加入timer调度队列，连接老化的时候，一般来设置新的连接的timer。
- dpvs_timer_cancel()：删除一个timer。

### 2.1 关键代码流程

main函数中首先调用函数rte_timer_subsystem_init()初始化dpdk的timer子系统。然后调用函数dpvs_timer_init()来做dpvs的初始化，其对每一个slave的core调用函数timer_lcore_init()来做初始化，对master core调用函数timer_init_schedler()来做初始化。执行中都调用了函数timer_init_schedler()，其作用是在每一个core上初始化一个1ms的周期timer，并注册timer的调度器回调函数。

