# memory barrier

### why need memory barrier?

​	内存屏障解决的也是同步问题

​	cpu对内存访问的指令，会在进行编译的时候被优化，但是优化的同时也就代表着对指令的重新排序，这种优化未必符合编码时 原始逻辑的预期。

**simple case ：**

init state : A=1, B=2

​		CPU0	    |		CPU1

-------------------------------|-------------------------------

​		A=3		|		B=4

​		Y=B		|		X=A



对于CPU0来说，由于内存乱序的存在，A=3和B=4 的执行顺序是未知，同理在CPU1上也是如此

因此将两个cpu的操作综合起来看，四条指令的顺序是完全未知的，因此由24钟可能性

prev>--------------------------------------------------------------->next>------------------->result

STORE A=3，STORE B=4，X = LOAD A ，Y = LOAD B				X = 3，Y = 4

X = LOAD A，STORE A=3，STORE B=4 ，Y = LOAD B				X = 1，Y = 4

Y = LOAD B，X = LOAD A，STORE A=3，STORE B=4 				X = 1，Y = 2

.............

假如加上了memory barrier

init state : A=1, B=2

​		CPU0	    |		CPU1

-------------------------------|-------------------------------

​		A=3		|		B=4

​		mb()	       |		mb()

​		Y=B		|		X=A

Y = LOAD B 一定在 STORE A = 3 之后发生，CPU1上同理，因此将可能性降低成了4种

此时如果加上一些值的判断（随手举个例子，实际不一定这么用）

​		CPU0	    |		CPU1

-------------------------------|-------------------------------

​	                              |	while (Y != 5) 

​		A=3		|		B=4

​		mb()	       |		mb()

​		Y=5		|		X=A

这个时候就能保证一定是

STORE A=3，STORE Y=5，STORE B=4 ，X = LOAD A



### memory barrier function

先搞懂几个memory barrier函数的使用方式：

1. smp_mb__after_spinlock()
   - smp_mb__after_spinlock()函数提供了一个完整的内存屏障，用于保证在程序顺序中早于锁获取操作的位置和晚于锁释放操作的位置之间的内存访问的顺序和同步。
2. smp_rmb()
   - 保证屏障前的LOAD操作永远出现在屏障后的LOAD操作
3. smp_wmb()
   - 保证屏障前的STORE操作永远出现在屏障后的STORE操作
4. smp_mb()
   - smp_wmb() + smp_rmb()
5. smp_acquire__after_ctrl_dep()
   - 怀疑本来设计成one-way memory barrier，但在代码中被宏定义成 smp_rmb
6. smp_store_release()
   - 和smp_load_acquire配合使用，保证了 RELEASE 操作之前的所有内存操作都发生在 RELEASE 之前
7. smp_load_acquire()
   - 和smp_store_release配合使用，保证了 ACQUIRE 操作之后的所有内存操作都发生在ACQUIRE操作之后
8. smp_cond_load_acquire()
   - 在load的基础上加上了cond控制，在符合条件前死循环

大概分为几类：

1. full memory barrier(1-5)
2. one-way permeable barrier(6-8)
   - 仅保证了单向的内存屏障



todo:好文还没读完

​	https://github.com/cncounter/translation/blob/master/tiemao_2020/46_Linux_Kernel_Memory_Barriers/README.md



​	







​	







其他资料：

https://www.kernel.org/doc/Documentation/memory-barriers.txt

https://vh21.github.io/linux/2015/06/22/concurrency-in-the-linux-kernel-2.html

http://www.wowotech.net/kernel_synchronization/memory-barrier.html

