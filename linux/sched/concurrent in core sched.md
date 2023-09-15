# concurrent in core sched 

### 核心调度流程(task视角)

**p->state**:标记task的状态

​	修改点：

1. set_current_state()
2. __set_current_state()
3. set_special_state() todo: set_special_state
4. try_to_wake_up()

**p->on_rq**：标记task是否在rq上

​	修改点：

1. activate_task()
2. deactivate_task()

**p->on_cpu**：标记task是否在cpu上执行

​	修改点：

1. prepare_task() 
2. cleared by finish_task()

​	

### 是否存在并发

1. try_to_wake_up & __schedule(sched in)
   - 不存在并发问题，因为ttwu完成前task不在rq上无法被选到
2. try_to_wake_up & __schedule(sched out)
   - **存在并发问题，待下文详解**
3. try_to_wake_up & try_to_wake_up
   - 不存在并发问题，由p->pi_lock保护
4. try_to_wake_up & __set_current_state
   - 不存在并发问题，因为ttwu完成前task不会作为curr
7. schedule& schedule
   - 不存在问题，schedule()关中断关抢占
9. schedule & __set_current_state
   - schedule一定发生在本cpu，和__set_current_state冲突
10. set_current_state & set_current_state
    - <u>有可能发生吗？在中断中是否会调用set_current_state?</u>

## try_to_wake_up & __schedule的并发问题

### memory 示意图

![image-20230921234731334](D:\kernel\linux-kernel-dev\linux\sched\image\mb示意图.png)



### schedule和ttwu流程

![image-20230921235014068](D:\kernel\linux-kernel-dev\linux\sched\image\流程图.png)

1. **ttwu中p->state的一致性**

   如果ttwu正在唤醒本cpu上的task，那么和set_current_state不存在并发

   如果其他核上正在跑set_current_state并且将p->state修改为S或D（猜测）

   ​	那么在ttwu判断on_rq时会读到1，这个时候会将state设回running

   ​	这个场景为一个任务将自己设为D，但还没来得及走到schedule被出队就唤醒了，这个时候将其设回running继续跑就行

2. **schedule中p->state的一致性**

   ​	schedule中只会读一次state

3. **ttwu中p->on_rq的一致性**

   <u>读on_rq时的一致性</u>

   目前为止try_to_wake_up()还没有持rq锁，存在和schedule()并发导致on_rq的一致性得不到保证
   	用了以下方法去保证一致性
   	if READ_ONCE(p->on_rq) = 1 :可能有schedule()的并发，尚未完成deactivate_task()
   		ttwu_runnable():持rq锁再去read一次p->on_rq,能够保证schedule()流程完成
   			if ttwu_runnable(p, wake_flags) = 1 说明之前无并发，task在rq上无需再唤醒
   			if ttwu_runnable(p, wake_flags) = 0 说明之前有并发，但当前schedule()已完成	

   ​	if READ_ONCE(p->on_rq) = 0 :可能有schedule()的并发，但是能保证deactivate_task()已完成
   ​            由于schedule()中只有出队流程deactivate_task()，会将p->on_rq  0->1

   <u>读on_rq后的一致性</u>

   当on_rq=0时，能够保证on_rq不会因为被schedule()修改而造成并发问题
           事实上，当p->on_rq=0时，p也不会被调度到

   因此能保证：
           	不需要持锁，p->on_rq能一直保持0 

4. **ttwu中p->on_cpu的一致性**

   <u>读on_cpu时的一致性</u>

   smp_cond_load_acquire(&p->on_cpu, !VAL)这句话的含义是:	**循环读on_cpu，直到读到on_cpu是0**

   ​	因此能保证，读到的on_cpu一定是0

   <u>读on_cpu时死循环的可能性</u>

   ​	由于之前已经能确保on_rq=0的一致性，因此只存在两种可能性：

   ​		1.schedule的流程已走完，那么此时on_cpu=0不会死循环

   ​		2.schedule的流程尚未走完，存在并发，那么只要等，schedule一定会将其设为0		

   <u>读on_cpu后的一致性</u>

   ​	由于将on_cpu置1的操作存在于schedule的sched_in流程，前提是在pick_next_task的时候被选到

   ​	由于on_rq为0，那么不会被选到，也就不存在这个并发问题

   





todo:这段注释看不懂

```c
	/*
	 * Make sure that signal_pending_state()->signal_pending() below
	 * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
	 * done by the caller to avoid the race with signal_wake_up():
	 *
	 * __set_current_state(@state)		signal_wake_up()
	 * schedule()				  set_tsk_thread_flag(p, TIF_SIGPENDING)
	 *					  wake_up_state(p, state)
	 *   LOCK rq->lock			    LOCK p->pi_state
	 *   smp_mb__after_spinlock()		    smp_mb__after_spinlock()
	 *     if (signal_pending_state())	    if (p->state & @state)
	 *
	 * Also, the membarrier system call requires a full memory barrier
	 * after coming from user-space, before storing to rq->curr.
	 */
```

![concurrent in core sched](D:\kernel\linux-kernel-dev\linux\sched\image\concurrent in core sched.png)
