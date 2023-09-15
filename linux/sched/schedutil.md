# schedutil

### 初始化

在调用sugov_start将sugov_policy中的参数进行初始化，其中包括：

freq_update_delay_ns：	调频间隔（可动态配置）

last_freq_update_time：	记录上一次的调频时间

next_freq：				用于保存计算出的下一个频点

work_in_progress：		标记上一次调频是否已经完成（慢速调频路径）	

limits_changed：			用于标记限频值的变动

cached_raw_freq：

同时将调频的主函数注册到sugov_cpu中，调用cpufreq_update_util()函数时会走到这个调频函数中

```c
if (policy_is_shared(policy))
		uu = sugov_update_shared;//多个cpu共享一个cpufreq_policy
	else if (policy->fast_switch_enabled && cpufreq_driver_has_adjust_perf())
		uu = sugov_update_single_perf;
	else
		uu = sugov_update_single_freq;

	for_each_cpu(cpu, policy->cpus) {
		struct sugov_cpu *sg_cpu = &per_cpu(sugov_cpu, cpu);

		cpufreq_add_update_util_hook(cpu, &sg_cpu->update_util, uu);
	}
```

### sugov_update_shared流程

```c
static void
sugov_update_shared(struct update_util_data *hook, u64 time, unsigned int flags)
{
	struct sugov_cpu *sg_cpu = container_of(hook, struct sugov_cpu, update_util);
	struct sugov_policy *sg_policy = sg_cpu->sg_policy;
	unsigned int next_f;

	raw_spin_lock(&sg_policy->update_lock);

	sugov_iowait_boost(sg_cpu, time, flags);//处理cpu的iowait状态
	sg_cpu->last_update = time;//更新此次调频的时间

	ignore_dl_rate_limit(sg_cpu);

    /*
    	cpufreq_this_cpu_can_update 为false时 不调频
    	由limits_changed触发的调频不考虑调频时延
    	调频间隔不满足freq_update_delay_ns时不调频
    */
	if (sugov_should_update_freq(sg_policy, time)) {
		next_f = sugov_next_freq_shared(sg_cpu, time);//根据负载计算出下一个频点

		if (!sugov_update_next_freq(sg_policy, time, next_f))//如果计算出的频点未改变，不调频
			goto unlock;

		if (sg_policy->policy->fast_switch_enabled)
			cpufreq_driver_fast_switch(sg_policy->policy, next_f);//调频快速路径
		else
			sugov_deferred_update(sg_policy);//调频慢速路径
	}
unlock:
	raw_spin_unlock(&sg_policy->update_lock);
}
```

sugov_update_next_freq的流程类似，区别是只对一个核进行调频

todo:搞懂sugov_update_single_perf是干啥的

### 频点计算方式

从policy的每个cpu中获取负载，取其中的最大值

其中每个cpu负载通过 还被iowait_boost和uclamp影响

todo：主线的负载计算是从哪里获取的

next_freq = curr_freq * C * cap / util

当前C写死为0.8，因此目的是将cpu的负载维持在80%左右



### 实际调频

如果支持快速路径，走驱动的调频接口cpufreq_driver_fast_switch()

否则走慢速调频路径sugov_deferred_update()

将调频的irq_work入队，在irq_work中实现真正的调频

### iowait_boost

如果一个cpu上的任务正在等待io，那么认为在io完成时，需要给予较大的供给，因此追踪iowait状态。每次iowait的任务入队时，会调用sugov_iowait_boost去更新iowait_boost的值，每次乘以二



### 调频点

deadline.c

1. __add_running_bw
2. __sub_running_bw

fair.c

1. cfs_rq_util_change
2. enqueue_task_fair(SCHED_CPUFREQ_IOWAIT)
3. update_blocked_averages

rt.c

1. sched_rt_rq_dequeue
2. sched_rt_rq_dequeue
