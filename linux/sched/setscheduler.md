# setscheduler



setscheduler通过syscall走到内核

函数定义：

```c
static int __sched_setscheduler(struct task_struct *p,                                                   const struct sched_attr *attr,
                bool user, bool pi)`
```

p:操作的线程

attr:设置的参数

user:是否由用户态设置

pi:todo:确认下是否为rt_mutex所用

可/以看到主要设置 调度策略 优先级 deadline参数 uclamp这几块

    __u32 sched_policy;
    __u64 sched_flags;
    /* SCHED_NORMAL, SCHED_BATCH */
    __s32 sched_nice;
                                                                                              /* SCHED_FIFO, SCHED_RR */
    __u32 sched_priority;
    
    /* SCHED_DEADLINE */
    __u64 sched_runtime;
    __u64 sched_deadline;
    __u64 sched_period;
    
    /* Utilization hints */
    __u32 sched_util_min;
    __u32 sched_util_max;



1. 判断参数的正确性，不满足则

   - 判断参数是否合法
   - 如果user=1，权限校验是否通过
   - 设置的参数和当前的参数是否不一致

2. 将running状态的任务摘下来

   - 如果on_rq，则dequeue_task

   - 如果on_cpu，则dequeue_task

   - 在合适的时候复原

     note:task的变化需要被感知到

3. 设置参数

   -  __setscheduler_params
   - __setscheduler_prio
   - __setscheduler_uclamp

4. 禁抢占，load_balance

