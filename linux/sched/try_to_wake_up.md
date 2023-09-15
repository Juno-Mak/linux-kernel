# try_to_wake_up



```c
 * try_to_wake_up - wake up a thread
 * @p: the thread to be awakened
 * @state: the mask of task states that can be woken
 * @wake_flags: wake modifier flags (WF_*)
int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags);
```

## 调用点

try_to_wake_up（core.c）	

​	除了core.c外，没看到其他地方直接调用__try_to_wake_up函数，而是调用被封装的三钟ttwu函数。

​	调用点如下：（TODO:将调用点大致分类，目前看到可以分为进程调度、同步原语、定时器等等）

1. default_wake_function（core.c）

   ```c
   int default_wake_function(wait_queue_entry_t *curr, unsigned mode, int wake_flags, void *key)
   {
     WARN_ON_ONCE(IS_ENABLED(CONFIG_SCHED_DEBUG) && wake_flags & ~(WF_SYNC|WF_CURRENT_CPU));
   
     return try_to_wake_up(curr->private, mode, wake_flags);//todo:explain curr->privite
   }
   ```

   - iocg_wake_fn（blk-iocost.c)
   - ep_autoremove_wake_function（eventpoll.c）    
   - __pollwake（select.c）
   - init_waitqueue_entry（wait.h）
   - child_wait_callback（exit.c）
   - autoremove_wake_function（wait.c）
   - woken_wake_function（wait.c）
   - synchronous_wake_function（shmem.c)

2. wake_up_process（core.c）

   ```c
   int wake_up_process(struct task_struct *p)
   {
   	return try_to_wake_up(p, TASK_NORMAL, 0);
   }
   ```

   - 唤醒进程，调用点茫茫多

3. wake_up_state（core.c)

   ```c
   int wake_up_state(struct task_struct *p, unsigned int state)
   {
     return try_to_wake_up(p, state, 0);
   }
   ```

   - dma_fence_default_wait_cb（dma-fence.c）
   - userfaultfd_wake_function(userfaultfd.c)
   - __set_notify_signal(signal.h)
   - io_req_local_work_add(io_uring.c)
   - freeze_task(freezer.c)
   - __thaw_task(freezer.c)
   - kthread_unpark(thread.c)
   - ptrace_unfreeze_traced(ptrace.c)
   - ptrace_resume(ptrace.c)
   - signal_wake_up_state(signal.c)
   - prepare_signal(signal.c)
   - requeue_pi_wake_futex(requeue.c)
   - klp_send_signals(transition.c)
   - rt_mutex_wake_up_q(rt_mutex.c)
   - rt_mutex_adjust_prio_chain(rt_mutex.c)
   - swake_up_all(swait.c)
   - wake_page_function(filemap.c)

   

   ## code note

   ```cc
   int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
   {
   	guard(preempt)();
   	int cpu, success = 0;
   
   	if (p == current) {
           /*
   		 * We're waking current, this means 'p->on_rq' and 'task_cpu(p)
   		 * == smp_processor_id()'. Together this means we can special
   		 * case the whole 'p->on_rq && ttwu_runnable()' case below
   		 * without taking any locks.
   		 *
   		 * In particular:
   		 *  - we rely on Program-Order guarantees for all the ordering,
   		 *  - we're serialized against set_special_state() by virtue of
   		 *    it disabling IRQs (this allows not taking ->pi_lock).
   		 */
   		//todo:什么时候会触发对current的wakeup？如果wakeup的是其他cpu的current是怎么处理的？
   		if (!ttwu_state_match(p, state, &success))
   			goto out;
   
   		trace_sched_waking(p);
   		ttwu_do_wakeup(p);
   		//todo：为什么仍然需要去set p的state
   		goto out;
   	}
   
   	/*
   	 * If we are going to wake up a thread waiting for CONDITION we
   	 * need to ensure that CONDITION=1 done by the caller can not be
   	 * reordered with p->state check below. This pairs with smp_store_mb()
   	 * in set_current_state() that the waiting thread does.
   	 */
   	scoped_guard (raw_spinlock_irqsave, &p->pi_lock) {
   		smp_mb__after_spinlock();
   		if (!ttwu_state_match(p, state, &success))
               //test task的state是否符合
   			break;
   
   		trace_sched_waking(p);
   
   		/*
   		 * Ensure we load p->on_rq _after_ p->state, otherwise it would
   		 * be possible to, falsely, observe p->on_rq == 0 and get stuck
   		 * in smp_cond_load_acquire() below.
   		 *
   		 * sched_ttwu_pending()			try_to_wake_up()
   		 *   STORE p->on_rq = 1			  LOAD p->state
   		 *   UNLOCK rq->lock
   		 *
   		 * __schedule() (switch to task 'p')
   		 *   LOCK rq->lock			  smp_rmb();
   		 *   smp_mb__after_spinlock();
   		 *   UNLOCK rq->lock
   		 *
   		 * [task p]
   		 *   STORE p->state = UNINTERRUPTIBLE	  LOAD p->on_rq
   		 *
   		 * Pairs with the LOCK+smp_mb__after_spinlock() on rq->lock in
   		 * __schedule().  See the comment for smp_mb__after_spinlock().
   		 *
   		 * A similar smb_rmb() lives in try_invoke_on_locked_down_task().
   		 */
   		smp_rmb();
           //如果task已在队列上，不需要再次去走wakeup流程
           /*	目前为止try_to_wake_up()还没有持rq锁，存在和schedule()并发导致on_rq的一致性得不到保证
           	用了以下方法去保证一致性
           	if READ_ONCE(p->on_rq) = 1 :可能有schedule()的并发，尚未完成deactivate_task()
           			ttwu_runnable():持rq锁再去read一次p->on_rq,能够保证schedule()流程完成
   	    	 		if ttwu_runnable(p, wake_flags) = 1 说明之前无并发，task在rq上无需再唤醒
                       if ttwu_runnable(p, wake_flags) = 0 说明之前有并发，但当前schedule()已完成
               if READ_ONCE(p->on_rq) = 0 :可能有schedule()的并发，但是能保证deactivate_task()已完成
               
               由于schedule()中只有出队流程deactivate_task()，会将p->on_rq  0->1
               当on_rq=0时，能够保证on_rq不会因为被schedule()修改而造成并发问题
               事实上，当p->on_rq=0时，p也不会被调度到
               因此能保证：
               	不需要持锁，p->on_rq能一直保持0 
           */
   		if (READ_ONCE(p->on_rq) && ttwu_runnable(p, wake_flags))
   			break;
   
   #ifdef CONFIG_SMP
   		/*
   		 * Ensure we load p->on_cpu _after_ p->on_rq, otherwise it would be
   		 * possible to, falsely, observe p->on_cpu == 0.
   		 *
   		 * One must be running (->on_cpu == 1) in order to remove oneself
   		 * from the runqueue.
   		 *
   		 * __schedule() (switch to task 'p')	try_to_wake_up()
   		 *   STORE p->on_cpu = 1		  LOAD p->on_rq
   		 *   UNLOCK rq->lock
   		 *
   		 * __schedule() (put 'p' to sleep)
   		 *   LOCK rq->lock			  smp_rmb();
   		 *   smp_mb__after_spinlock();
   		 *   STORE p->on_rq = 0			  LOAD p->on_cpu
   		 *
   		 * Pairs with the LOCK+smp_mb__after_spinlock() on rq->lock in
   		 * __schedule().  See the comment for smp_mb__after_spinlock().
   		 *
   		 * Form a control-dep-acquire with p->on_rq == 0 above, to ensure
   		 * schedule()'s deactivate_task() has 'happened' and p will no longer
   		 * care about it's own p->state. See the comment in __schedule().
   		 */
   		smp_acquire__after_ctrl_dep();
   
   		/*
   		 * We're doing the wakeup (@success == 1), they did a dequeue (p->on_rq
   		 * == 0), which means we need to do an enqueue, change p->state to
   		 * TASK_WAKING such that we can unlock p->pi_lock before doing the
   		 * enqueue, such as ttwu_queue_wakelist().
   		 */
           /*
           	从
           */
   		WRITE_ONCE(p->__state, TASK_WAKING);
   
           /*
   		 * If the owning (remote) CPU is still in the middle of schedule() with
   		 * this task as prev, considering queueing p on the remote CPUs wake_list
   		 * which potentially sends an IPI instead of spinning on p->on_cpu to
   		 * let the waker make forward progress. This is safe because IRQs are
   		 * disabled and the IPI will deliver after on_cpu is cleared.
   		 *
   		 * Ensure we load task_cpu(p) after p->on_cpu:
   		 *
   		 * set_task_cpu(p, cpu);
   		 *   STORE p->cpu = @cpu
   		 * __schedule() (switch to task 'p')
   		 *   LOCK rq->lock
   		 *   smp_mb__after_spin_lock()		smp_cond_load_acquire(&p->on_cpu)
   		 *   STORE p->on_cpu = 1		LOAD p->cpu
   		 *
   		 * to ensure we observe the correct CPU on which the task is currently
   		 * scheduling.
   		 */
           /*如果当前正在唤醒的是这样一个任务：在其他cpu上正在处于schedule并发流程中，并且尚未走完schedOUt流程
           	那么尝试将这个唤醒的动作交给其cpu去做
           	方法是将这个任务加入cpu的wake_entry队列，在sched_ttwu_pending函数中会将队列中的task入队
           */
   		if (smp_load_acquire(&p->on_cpu) &&
   		    ttwu_queue_wakelist(p, task_cpu(p), wake_flags))
   			break;
   
           /*
   		 * If the owning (remote) CPU is still in the middle of schedule() with
   		 * this task as prev, wait until it's done referencing the task.
   		 *
   		 * Pairs with the smp_store_release() in finish_task().
   		 *
   		 * This ensures that tasks getting woken will be fully ordered against
   		 * their previous state and preserve Program Order.
   		 */
           /*
           	如果和schedule()存在并发问题，on_cpu=1的情况
           	p作为prev，等着被sched_out，并在最后finish_task的地方store p->on_cpu = 0
           	因此这里在mb的保护下，循环等到p->on_cpu=0，保证schedule()走完
           */
   		smp_cond_load_acquire(&p->on_cpu, !VAL);
   
           //为唤醒的task选择一个cpu
   		cpu = select_task_rq(p, p->wake_cpu, wake_flags | WF_TTWU);
   		if (task_cpu(p) != cpu) {
               //todo：选到了task非当前的cpu，需要做哪些操作？
   			if (p->in_iowait) {
   				delayacct_blkio_end(p);
   				atomic_dec(&task_rq(p)->nr_iowait);
   			}
   
   			wake_flags |= WF_MIGRATED;
   			psi_ttwu_dequeue(p);
   			set_task_cpu(p, cpu);
   		}
   #else
   		cpu = task_cpu(p);
   #endif /* CONFIG_SMP */
   
           //将task入队
   		ttwu_queue(p, cpu, wake_flags);
   	}
   out:
   	if (success)
   		ttwu_stat(p, task_cpu(p), wake_flags);
   
   	return success;
   }
   ```
   
   
   
   
   
   
   
   ```c
   和ttwu中的smp_cond_load_acquire(&p->on_cpu, !VAL);成对出现，保证了，在finish_task之后，ttwu才会走下去
   static inline void finish_task(struct task_struct *prev)
   {
   #ifdef CONFIG_SMP
   	/*
   	 * This must be the very last reference to @prev from this CPU. After
   	 * p->on_cpu is cleared, the task can be moved to a different CPU. We
   	 * must ensure this doesn't happen until the switch is completely
   	 * finished.
   	 *
   	 * In particular, the load of prev->state in finish_task_switch() must
   	 * happen before this.
   	 *
   	 * Pairs with the smp_cond_load_acquire() in try_to_wake_up().
   	 */
   	smp_store_release(&prev->on_cpu, 0);
   #endif
   }
   ```
   
   
   
   ## Q&A
   
   ​	