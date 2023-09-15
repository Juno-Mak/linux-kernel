# case



核心调度流程中，有些隐蔽的问题存在于注释中，要深入理解这些场景，搞清几个问题：

1. 场景说明。这是什么场景
2. 场景影响。这个场景会带来什么后果
3. 解决方法。社区上代码是如何结局的



### case1

try_to_wake_up和schedule产生并发时，对于on_rq，on_cpu的一致性是如何保护的？

需要注意的有几个变量

p->state：

p->on_rq：由activate_task()和deactivate_task()去写，写的时候持rq锁



p->on_cpu

1. 场景说明

   假设task p在以下的循环中

   ```c
      for (;;) {
          set_current_state(TASK_UNINTERRUPTIBLE);
    
          if (CONDITION)
             break;
    
          schedule();
       }
       __set_current_state(TASK_RUNNING);
   ```

   在set_current_state（）和schedule（）中间的状态，task p也是可以被调度到的，因此

2. 场景影响

3. 解决方法

   schedule流程中smp_cond_load_acquire(&p->on_cpu, !VAL)

   

   smp_cond_load_acquire

   

   ![image-20230917021529476](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20230917021529476.png)

   在ttwu流程中 要确保先读state再读on_rq，不然可能会出现on_rq=1 然后在smp_cond_load_acquire处死锁

   

   当前是如何大胆的去做smp_cond_load_acquire(&p->on_cpu, !VAL)的呢？

   

   首先在ttwu的代码注释中可以看到

   需要依照严格的顺序去读 p->state,p->on_rq,p->on_cpu三个变量

   

   其他的设置流程中，是怎么设置这三个变量的呢

   ​	

   schedule（sched_out)						ttwu

   prev_state = READ_ONCE(prev->__state);		

   if (prev_state)

   p->on_rq = 0;

   

   

   1. 使用适当的内存屏障：在访问p->on_rq之前，可以使用适当的内存屏障指令来确保前面的加载操作完成，并且后续的加载操作不会被重排序。这样可以保证获取到的p->on_rq数据是最新的。
   2. 使用原子操作：为了避免竞争条件和数据不一致性，try_to_wake_up()函数可能会使用原子操作来对p->on_rq进行读取或修改。原子操作可以保证在多线程环境中对共享数据的操作是原子的，从而确保数据的一致性。

   

   

   写p->state的地方，除了ttwu用了p->pi_lock锁之外都没加锁

   写p->on_rq的地方，都加了rq->lock锁

   

   猜测的设计思路：p->on_cpu=1 是on_rq=1的子集，on_rq=1是state = running的子集

   

   所以schedule的设置顺序应该是

   

   scheduler

   

   

   

   在schedule中，如果修改了on_cpu，由于内存屏障，在finish_task的时候才会smp_store_release(&prev->on_cpu, 0)

4. ```c
   /*
    * Notes on Program-Order guarantees on SMP systems.
    *
    *  MIGRATION
    *
    * The basic program-order guarantee on SMP systems is that when a task [t]
    * migrates, all its activity on its old CPU [c0] happens-before any subsequent
    * execution on its new CPU [c1].
    *
    * For migration (of runnable tasks) this is provided by the following means:
    *
    *  A) UNLOCK of the rq(c0)->lock scheduling out task t
    *  B) migration for t is required to synchronize *both* rq(c0)->lock and
    *     rq(c1)->lock (if not at the same time, then in that order).
    *  C) LOCK of the rq(c1)->lock scheduling in task
    *
    * Release/acquire chaining guarantees that B happens after A and C after B.
    * Note: the CPU doing B need not be c0 or c1
    *
    * Example:
    *
    *   CPU0            CPU1            CPU2
    *
    *   LOCK rq(0)->lock
    *   sched-out X
    *   sched-in Y
    *   UNLOCK rq(0)->lock
    *
    *                                   LOCK rq(0)->lock // orders against CPU0
    *                                   dequeue X
    *                                   UNLOCK rq(0)->lock
    *
    *                                   LOCK rq(1)->lock
    *                                   enqueue X
    *                                   UNLOCK rq(1)->lock
    *
    *                   LOCK rq(1)->lock // orders against CPU2
    *                   sched-out Z
    *                   sched-in X
    *                   UNLOCK rq(1)->lock
    *
    *
    *  BLOCKING -- aka. SLEEP + WAKEUP
    *
    * For blocking we (obviously) need to provide the same guarantee as for
    * migration. However the means are completely different as there is no lock
    * chain to provide order. Instead we do:
    *
    *   1) smp_store_release(X->on_cpu, 0)   -- finish_task()
    *   2) smp_cond_load_acquire(!X->on_cpu) -- try_to_wake_up()
    *
    * Example:
    *
    *   CPU0 (schedule)  CPU1 (try_to_wake_up) CPU2 (schedule)
    *
    *   LOCK rq(0)->lock LOCK X->pi_lock
    *   dequeue X
    *   sched-out X
    *   smp_store_release(X->on_cpu, 0);
    *
    *                    smp_cond_load_acquire(&X->on_cpu, !VAL);
    *                    X->state = WAKING
    *                    set_task_cpu(X,2)
    *
    *                    LOCK rq(2)->lock
    *                    enqueue X
    *                    X->state = RUNNING
    *                    UNLOCK rq(2)->lock
    *
    *                                          LOCK rq(2)->lock // orders against CPU1
    *                                          sched-out Z
    *                                          sched-in X
    *                                          UNLOCK rq(2)->lock
    *
    *                    UNLOCK X->pi_lock
    *   UNLOCK rq(0)->lock
    *
    *
    * However, for wakeups there is a second guarantee we must provide, namely we
    * must ensure that CONDITION=1 done by the caller can not be reordered with
    * accesses to the task state; see try_to_wake_up() and set_current_state().
    */
   
   ```

   ```c
   /*
    * Serialization rules:
    *
    * Lock order:
    *
    *   p->pi_lock
    *     rq->lock
    *       hrtimer_cpu_base->lock (hrtimer_start() for bandwidth controls)
    *
    *  rq1->lock
    *    rq2->lock  where: rq1 < rq2
    *
    * Regular state:
    *
    * Normal scheduling state is serialized by rq->lock. __schedule() takes the
    * local CPU's rq->lock, it optionally removes the task from the runqueue and
    * always looks at the local rq data structures to find the most eligible task
    * to run next.
    *
    * Task enqueue is also under rq->lock, possibly taken from another CPU.
    * Wakeups from another LLC domain might use an IPI to transfer the enqueue to
    * the local CPU to avoid bouncing the runqueue state around [ see
    * ttwu_queue_wakelist() ]
    *
    * Task wakeup, specifically wakeups that involve migration, are horribly
    * complicated to avoid having to take two rq->locks.
    *
    * Special state:
    *
    * System-calls and anything external will use task_rq_lock() which acquires
    * both p->pi_lock and rq->lock. As a consequence the state they change is
    * stable while holding either lock:
    *
    *  - sched_setaffinity()/
    *    set_cpus_allowed_ptr():	p->cpus_ptr, p->nr_cpus_allowed
    *  - set_user_nice():		p->se.load, p->*prio
    *  - __sched_setscheduler():	p->sched_class, p->policy, p->*prio,
    *				p->se.load, p->rt_priority,
    *				p->dl.dl_{runtime, deadline, period, flags, bw, density}
    *  - sched_setnuma():		p->numa_preferred_nid
    *  - sched_move_task():	p->sched_task_group
    *  - uclamp_update_active()	p->uclamp*
    *
    * p->state <- TASK_*:
    *
    *   is changed locklessly using set_current_state(), __set_current_state() or
    *   set_special_state(), see their respective comments, or by
    *   try_to_wake_up(). This latter uses p->pi_lock to serialize against
    *   concurrent self.
    *
    * p->on_rq <- { 0, 1 = TASK_ON_RQ_QUEUED, 2 = TASK_ON_RQ_MIGRATING }:
    *
    *   is set by activate_task() and cleared by deactivate_task(), under
    *   rq->lock. Non-zero indicates the task is runnable, the special
    *   ON_RQ_MIGRATING state is used for migration without holding both
    *   rq->locks. It indicates task_cpu() is not stable, see task_rq_lock().
    *
    * p->on_cpu <- { 0, 1 }:
    *
    *   is set by prepare_task() and cleared by finish_task() such that it will be
    *   set before p is scheduled-in and cleared after p is scheduled-out, both
    *   under rq->lock. Non-zero indicates the task is running on its CPU.
    *
    *   [ The astute reader will observe that it is possible for two tasks on one
    *     CPU to have ->on_cpu = 1 at the same time. ]
    *
    * task_cpu(p): is changed by set_task_cpu(), the rules are:
    *
    *  - Don't call set_task_cpu() on a blocked task:
    *
    *    We don't care what CPU we're not running on, this simplifies hotplug,
    *    the CPU assignment of blocked tasks isn't required to be valid.
    *
    *  - for try_to_wake_up(), called under p->pi_lock:
    *
    *    This allows try_to_wake_up() to only take one rq->lock, see its comment.
    *
    *  - for migration called under rq->lock:
    *    [ see task_on_rq_migrating() in task_rq_lock() ]
    *
    *    o move_queued_task()
    *    o detach_task()
    *
    *  - for migration called under double_rq_lock():
    *
    *    o __migrate_swap_task()
    *    o push_rt_task() / pull_rt_task()
    *    o push_dl_task() / pull_dl_task()
    *    o dl_task_offline_migration()
    *
    */
   ```

   ```c
   /*
    * set_current_state() includes a barrier so that the write of current->__state
    * is correctly serialised wrt the caller's subsequent test of whether to
    * actually sleep:
    *
    *   for (;;) {
    *	set_current_state(TASK_UNINTERRUPTIBLE);
    *	if (CONDITION)
    *	   break;
    *
    *	schedule();
    *   }
    *   __set_current_state(TASK_RUNNING);
    *
    * If the caller does not need such serialisation (because, for instance, the
    * CONDITION test and condition change and wakeup are under the same lock) then
    * use __set_current_state().
    *
    * The above is typically ordered against the wakeup, which does:
    *
    *   CONDITION = 1;
    *   wake_up_state(p, TASK_UNINTERRUPTIBLE);
    *
    * where wake_up_state()/try_to_wake_up() executes a full memory barrier before
    * accessing p->__state.
    *
    * Wakeup will do: if (@state & p->__state) p->__state = TASK_RUNNING, that is,
    * once it observes the TASK_UNINTERRUPTIBLE store the waking CPU can issue a
    * TASK_RUNNING store which can collide with __set_current_state(TASK_RUNNING).
    *
    * However, with slightly different timing the wakeup TASK_RUNNING store can
    * also collide with the TASK_UNINTERRUPTIBLE store. Losing that store is not
    * a problem either because that will result in one extra go around the loop
    * and our @cond test will save the day.
    *
    * Also see the comments of try_to_wake_up().
    */
   #define __set_current_state(state_value)				\
   	do {								\
   		debug_normal_state_change((state_value));		\
   		WRITE_ONCE(current->__state, (state_value));		\
   	} while (0)
   
   #define set_current_state(state_value)					\
   	do {								\
   		debug_normal_state_change((state_value));		\
   		smp_store_mb(current->__state, (state_value));		\
   	} while (0)
   ```

   

![image-20230917031252287](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20230917031252287.png)

这里两者确保了 schedule的 deactive_task 走完了才会





```c
			/*
			 * __schedule()			ttwu()
			 *   prev_state = prev->state;    if (p->on_rq && ...)
			 *   if (prev_state)		    goto out;
			 *     p->on_rq = 0;		  smp_acquire__after_ctrl_dep();
			 *				  p->state = TASK_WAKING
			 *
			 * Where __schedule() and ttwu() have matching control dependencies.
			 *
			 * After this, schedule() must not care about p->state any more.
			 */
```

在schedule里，只有p->state的部分的条件中才会去 走到p->on_rq = 0

在ttwu中，只有p->on_rq=0才会去修改p->state=wake



set_current_state								try_to_wake_up

smp_store_mb(current->__state, (state_value))	  scoped_guard (raw_spinlock_irqsave, &p->pi_lock) {

​    											smp_mb__after_spinlock();



成对的内存屏障保证了try_to_wake_up读到的state是最新的

然后用这个state去判断是否需要break

