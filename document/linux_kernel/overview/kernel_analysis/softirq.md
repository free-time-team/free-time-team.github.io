<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

#Softirq#
Author: Dalink

##一、概述##
软中断是内核提供的一种延迟机制，tasklet总是工作当前CPU上。内核设置一个pending标志位，当其中标志位打开时，会执行指定下表的数组。这个pending标志位相当于中断使能控制寄存器。当系统从中断返回时，检查对应的标志位，执行相应的“中断函数”。在中断这些中断函数加入一些任务链表时，就构成了tasklet工作机制。

##二、结构体定义##

系统为每个内核定义了一个tasklet_vec和tasklet_hi_vec双向链表。

```c++
// include/linux/interrupt.h : 526
struct tasklet_struct
{
	struct tasklet_struct *next;
	unsigned long state;
	atomic_t count;
	void (*func)(unsigned long);
	unsigned long data;
};
```


```c++
//kernel/softirq.c : 455
struct tasklet_head {
	struct tasklet_struct *head;
	struct tasklet_struct **tail;
};
// include/linux/interrupt.h : 447
enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,
	BLOCK_SOFTIRQ,
	IRQ_POLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,
	NR_SOFTIRQS
};
```

```c++
// include/linux/interrupt.h : 475
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};
```

## 三、全局变量##

内核同时定义一个类型为softirq_action的softirq_vec[NR_SOFTIRQS]数组。

```c++
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);
//kernel/softirq.c : 56
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```



## 四、初始化##

初始化过程的调用栈为`setup_arch(&command_line) > softirq_init()`。softirq_init首先对tasklet_vec和tasklet_hi_vec双向链表做初始化操作。初始化之后，两个链表均为空链表。然后调用open_softirq函数分别向softirq_vec装入tasklet_action函数和tasklet_hi_action函数。

softirq_vec的类型虽然为softirq_action类型，但是参数action却为函数指针类型，这是因为，softirq_action只包含一个函数指针成员。可以参见第二小节softirq_action结构体的定义。

```c++
//kernel/softirq.c : 648
void __init softirq_init(void)
{
	int cpu;
	//初始化tasklet_vec和tasklet_hi_vec
	for_each_possible_cpu(cpu) {
		per_cpu(tasklet_vec, cpu).tail =
			&per_cpu(tasklet_vec, cpu).head;
		per_cpu(tasklet_hi_vec, cpu).tail =
			&per_cpu(tasklet_hi_vec, cpu).head;
	}
	//将tasklet_action和tasklet_hi_action放入softirq_vec数组
	open_softirq(TASKLET_SOFTIRQ, tasklet_action);
	open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}

//kernel/softirq.c : 447
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```

##三、pending状态分析##

定义irq_cpustat_t类型全局变量irq_stat。

```C++
DECLARE_PER_CPU_SHARED_ALIGNED(irq_cpustat_t, irq_stat);　
```

系统代码调用local_softirq_pending函数检测是否有软中断发生。

```c++
//include/linux/irq_cpustat.h
#define __IRQ_STAT(cpu, member)	(irq_stat[cpu].member)

#define local_softirq_pending() \
	__IRQ_STAT(smp_processor_id(), __softirq_pending)
```

local_softirq_pending()预编译之后为

```c++
(irq_stat[smp_processor_id()].__softirq_pending)
```

##四、工作过程##

tasklet_action函数会对tasklet_vec链表进行遍历，并执行tasklet_struct中的func函数，并将该tasklet_struct从链表中移除。

同样，tasklet_hi_action函数会对tasklet_hi_vec链表进行遍历，并执行tasklet_struct中的func函数，并将该tasklet_struct从链表中移除。

那么tasklet_action和tasklet_hi_action函数是什么时候被执行的呢？前面分析到，这两个函数被装入softirq_vec数组。内核显然会遍历这个数组，从而执行对应的action函数。中断返回时执行`exiting_irq()`从而触发软中断。调用栈为`exiting_irq()　> irq_exit() > invoke_softirq() > __do_softirq()`

在__do_softirq()函数中，内核遍历softirq_vec数组，分别执行对应的`h->action(h)`函数。

```c++
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
	struct softirq_action *h;
  	//当前CPU的标志位
	__u32 pending = local_softirq_pending();
restart:
	set_softirq_pending(0);
  	//读取tasklet_action数组变量softirq_vec
	h = softirq_vec;
  	//ffs用以获取pending位图的置位索引
	while ((softirq_bit = ffs(pending))) {
		h += softirq_bit - 1;
	    //执行tasklet_action对应的函数
		h->action(h);
		h++;
		pending >>= softirq_bit;
	}
  	//如果有软中断，继续执行
	pending = local_softirq_pending();
	if (pending) {
      	//软中断执行超时，则开启softirq线程继续处理处理
		if (time_before(jiffies, end) && !need_resched() &&
		    --max_restart)
			goto restart;
		wakeup_softirqd();
	}
}
```

ffs函数执行bsfs汇编指令，在变量中寻找第一个非0bit位，并把返回该位索引。

```
bsfl op1 op2
含义:
	扫描操作数op1,找到到第1个非0bit位, 把非0bit位的索引下标(从0计算)存入op2. 扫描从低位到高位扫描(也就是从右->左)(摘抄自：http://blog.csdn.net/zdl1016/article/details/8763803)。
```

前面已经分析，softirq_vec已经装入tasklet_action函数和tasklet_hi_action函数。以tasklet_action函数为例，他会对tasklet_vec进行遍历，并执行相应的软中断。每执行一个软中断，讲该软中断从链表中移除。所以软中断是单次触发而不是循环触发。

```c++
static __latent_entropy void tasklet_action(struct softirq_action *a)
{
	struct tasklet_struct *list = __this_cpu_read(tasklet_vec.head);
  	//首先置空链表
	__this_cpu_write(tasklet_vec.head, NULL);
	__this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
	while (list) {
      	//遍历tasklet_struct
		struct tasklet_struct *t = list;
		list = list->next;
		if (tasklet_trylock(t)) {
			if (!atomic_read(&t->count)) {
				if (!test_and_clear_bit(TASKLET_STATE_SCHED,
							&t->state))
					BUG();
              	//执行相应的函数
				t->func(t->data);
				tasklet_unlock(t);
				continue;
			}
			tasklet_unlock(t);
		}

		local_irq_disable();
		t->next = NULL;
      	//tasklet_vec指向未执行的tasklet_struct链表
		*__this_cpu_read(tasklet_vec.tail) = t;
		__this_cpu_write(tasklet_vec.tail, &(t->next));
		__raise_softirq_irqoff(TASKLET_SOFTIRQ);
	}
}
```

