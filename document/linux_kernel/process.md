<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

#Process#
Author: Dalink

##一、结构体定义##


```c++
struct task_group {
	struct cgroup_subsys_state css;
	struct sched_entity **se;
	struct cfs_rq **cfs_rq;
	unsigned long shares;
	struct sched_rt_entity **rt_se;
	struct rt_rq **rt_rq;
	struct rt_bandwidth rt_bandwidth;
	struct rcu_head rcu;
	struct list_head list;
	struct task_group *parent;
	struct list_head siblings;
	struct list_head children;
};
```



```c++
struct sched_entity {
	struct rb_node		run_node;
	struct list_head	group_node;
	struct sched_entity	*parent;
	/* rq on which this entity is (to be) queued: */
	struct cfs_rq		*cfs_rq;
	/* rq "owned" by this entity/group: */
	struct cfs_rq		*my_q;
};
```

cfs调度管理器

```c++
struct cfs_rq {
	unsigned int nr_running, h_nr_running;		//可运行的进程数
	struct rb_root tasks_timeline;				//红黑树结构的调度实体
	struct rb_node *rb_leftmost;				//最优先的调度实体
	struct sched_entity *curr, *next, *last, *skip;
	struct rq *rq;
	int on_list;
	struct list_head leaf_cfs_rq_list;
	struct task_group *tg;
};
```



```c++
struct rq {
	unsigned int nr_running;
	u64 nr_switches;

	struct cfs_rq cfs;
	struct rt_rq rt;
	struct dl_rq dl;
	struct task_struct *curr, *idle, *stop;
	
	int cpu;
	int online;
};
```

调度实体

```c++
struct task_struct {
	const struct sched_class *sched_class;
	struct sched_entity se;
	struct sched_rt_entity rt;
	struct task_group *sched_task_group;
	struct sched_dl_entity dl;

	struct sched_info sched_info;

	struct task_struct __rcu *real_parent; /* real parent process */
	struct task_struct __rcu *parent;
	
	struct list_head children;	/* list of my children */
	struct list_head sibling;	/* linkage in my parent's children list */
	struct task_struct *group_leader;	/* threadgroup leader */
};
```

## 二、常量定义##

系统定义全局变量root_task_group、一个链表头task_groups和每个CPU对应的runqueues。

```c++
//kernel/sched/core.c : 7520
struct task_group root_task_group;
LIST_HEAD(task_groups);

//include/linux/list.h : 2
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

//kernel/sched/core.c : 95
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);
```

##三、初始化##

在`setup_arch(&command_line) > sched_init()`函数中，系统使用kzalloc分配了2 * nr_cpu_ids * sizeof(void **) * 2字节内存。这部分内存存储着每个cpu对应的的sched_entity、cfs_rq x和 x指针。所以在接下来的代码7561~7573行代码，就是把这部分内存赋值到root_task_group成员。

```c++
alloc_size += 2 * nr_cpu_ids * sizeof(void **);
ptr = (unsigned long)kzalloc(alloc_size, GFP_NOWAIT);
root_task_group.cfs_rq = (struct cfs_rq **)ptr;
```

7603行，将root_task_group.list插入到task_groups之后。

```c++
//kernel/sched/core.c : 7603
list_add(&root_task_group.list, &task_groups);
```

7609行对cpu进行遍历，然后根据下标i在全局变量runqueues中查找rq。

```c++
//kernel/sched/core.c : 7612
rq = cpu_rq(i);

//kernel/sched/sched.h : 760
#define cpu_rq(cpu)		(&per_cpu(runqueues, (cpu)))
```

```
root_task_group~~~~~~~task_group~~~~~~~task_group~~~~~~~...
  |
  |---->cfs_rq*(CPU0) > cfs_rq*cfs_rq*(CPU1) > cfs_rq*cfs_rq*(CPU2) > cfs_rq*cfs_rq*(CPU3)
```

##四、fork的过程##

fork用以创建新的进程。fork系统调用定义在kernel/fork.c文件。它调用了_do_fork函数。

```c++
//kernel/sched/core.c : 2009
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
```

_do_fork函数首先调用copy_process复制出一个新的task_struct结构体对象，并把当前进程的众多成员赋值给新的进程，譬如说新的进程和父进程共享打开的文件。

```c++
//kernel/sched/core.c : 482
tsk = alloc_task_struct_node(node)
```

值得注意的是，在复制signal的时候，内核把signal的read_timer.function设置为it_real_fn。定时器到达时，read_timer.function将会被执行，这里read_timer.function的值即为it_real_fn函数。这个小细节在分析singal系统调用的很关键。

```c++
//kernel/sched/core.c : 1355
sig->real_timer.function = it_real_fn;
```

然后系统调用获取一个新的pid赋值给新进程。

```c++
//kernel/sched/core.c : 1955
pid = get_task_pid(p, PIDTYPE_PID);
```

最后，内核把新创建的进程装入调度结构，然后等待调度器选择执行新的进程。装入调度结构的调用栈为`wake_up_new_task(p) > activate_task(rq, p, 0) > enqueue_task(rq, p, flags)`。enqueue_task函数调用sched_class的enqueue_task函数。fair_sched_class对象的enqueue_task函数指针指向enqueue_task_fair，可参见enqueue_task_fair对象的定义。enqueue_task_fair首先调用for_each_sched_entity对p->se进行遍历。

```c++
//kernel/sched/fair.c : 279
#define for_each_sched_entity(se) \
		for (; se; se = se->parent)
```

根据se获取cfs_rq。

```c++
//kernel/sched/fair.c : 4740
cfs_rq = cfs_rq_of(se);
```

然后调用`enqueue_entity(cfs_rq, se, flags`)将se装入cfs_rq中。enqueue_entity调用`__enqueue_entity(cfs_rq, se)`将se->run_node装入cfs_rq->tasks_timeline带标的红黑树中。

```c++
//kernel/sched/fair.c : 578
rb_insert_color(&se->run_node, &cfs_rq->tasks_timeline);

//lib/rbtree.c : 433
void rb_insert_color(struct rb_node *node, struct rb_root *root)
{
	__rb_insert(node, root, dummy_rotate);
}
```

fork过程完成。接下来查看，调度器是如何从进程组织结构用选择目标并进行调度的。

##五、进程调度##

每当进程从内核态返回到用户态时，比如中断的触发，系统调用等，内核都需要检测是否需要进行任务调度。特别的是，x86体系中，linux使用Loca APIC为每个CPU提供一个定时中断。即使内核中长时间没有异常、系统调用、中断等，该定时中断也能保证调度的周期执行。

调度的入口是schedule()函数，该函数通过调用`__schedule(false)`完成调度过程。__schedule函数首先选定一个rq对象。

	//kernel/sched/core.c : 3342
	cpu = smp_processor_id();
	rq = cpu_rq(cpu)

pick_next_task_fair选择cfs最左边的进程作为目标。

```c++
//kernel/sched/fair.c : 6309
do {
	se = pick_next_entity(cfs_rq, NULL);
	set_next_entity(cfs_rq, se);
	cfs_rq = group_cfs_rq(se);
} while (cfs_rq);

p = task_of(se);
```


##六、信号##
####1、信号概述####
当一个进程收到一个信号时，进程会执行指定的函数。每个信号使用对应的数值表达。信号回调函数通过sigaction系统调用完成。
####2、数据结构####

pending表示对应信号的屏蔽位。sighand表达每个信号对应的处理函数。signal表示产生的信号。

```c++
struct task_struct {
	struct signal_struct *signal;	//同一线程组的多个线程共享同一个signal_struct
	struct sighand_struct *sighand;
	struct sigpending pending;
};
```

pending标志位。

```c++
struct sigpending {
	struct list_head list;			//指向sigqueue
	sigset_t signal;				//标志位
};

typedef struct {
	unsigned long sig[_NSIG_WORDS];
} sigset_t;

struct sigqueue {
	struct list_head list;
	int flags;
	siginfo_t info;
	struct user_struct *user;
};
```
信号对应的handler。

```c++
struct sighand_struct {
	atomic_t		count;
	struct k_sigaction	action[_NSIG];
	spinlock_t		siglock;
	wait_queue_head_t	signalfd_wqh;
};
struct k_sigaction {
	struct sigaction sa;

};
struct sigaction {
	__sighandler_t	sa_handler;
	unsigned long	sa_flags;
	sigset_t	sa_mask;	/* mask last for extensibility */
};
```

signal

```c++
struct signal_struct {
	atomic_t		sigcnt;
	atomic_t		live;
	int			nr_threads;
	struct list_head	thread_head;
	wait_queue_head_t	wait_chldexit;	/* for wait4() */
	struct task_struct	*curr_target;
	struct sigpending	shared_pending; 	//分组的pending
	int			group_exit_code;
	int			notify_count;
	struct task_struct	*group_exit_task;
	/* thread group stop support, overloads group_exit_code too */
	int			group_stop_count;
	unsigned int		flags; /* see SIGNAL_* flags below */
	ktime_t it_real_incr;	//定时器
};
```

//仅仅用于数据传递

```c++
struct ksignal {
	struct k_sigaction ka;
	siginfo_t info;
	int sig;
};
typedef struct siginfo {
	int si_signo;			//信号编号
	int si_errno;
	int si_code;
} __ARCH_SI_ATTRIBUTES siginfo_t;
```
####3、设置信号处理函数####

该系统调用为signal。

```c++
//kernel/signal.c : 3501
SYSCALL_DEFINE2(signal, int, sig, __sighandler_t, handler)
{
	struct k_sigaction new_sa, old_sa;
	int ret;

	new_sa.sa.sa_handler = handler;
	new_sa.sa.sa_flags = SA_ONESHOT | SA_NOMASK;
	sigemptyset(&new_sa.sa.sa_mask);

	ret = do_sigaction(sig, &new_sa, &old_sa);

	return ret ? ret : (unsigned long)old_sa.sa.sa_handler;
}
```

do_sigaction把目标handler装入到sig对应的结构体中。

```c++
//kernel/signal.c : 3074
k = &p->sighand->action[sig-1];

spin_lock_irq(&p->sighand->siglock);
if (oact)
	*oact = *k;
```

####4、发送信号####
__send_signal函数调用sigaddset把对应的pending位置位。

```c++
//kernel/signal.c : 1075
sigaddset(&pending->signal, sig);

//linux/signal.h : 55
static inline void sigaddset(sigset_t *set, int _sig)
{
	unsigned long sig = _sig - 1;
	if (_NSIG_WORDS == 1)
		set->sig[0] |= 1UL << sig;
	else
		set->sig[sig / _NSIG_BPW] |= 1UL << (sig % _NSIG_BPW);
}
```

####5、处理信号####
exit_to_usermode_loop调用的`do_signal(regs)`处理信号。该函数通过get_signal获取信号，然后执行handle_signal处理信号。dequeue_signal从pending中选择一个sig，装入ksignal。然后再从sighand中选择action，装入ksignal的ka字段。

```c++
//kernel/signal.c : 2206
dequeue_signal(current, &current->blocked, &ksig->info)

//kernel/signal.c : 568
__dequeue_signal(&tsk->pending, mask, info);

//kernel/signal.c : 548
int sig = next_signal(pending, mask)

//kernel/signal.c : 2217
ka = &sighand->action[signr-1]；
ksig->ka = *ka；
```

执行handle_signal处理信号。handle_signal的参数为ksignal。由于信号的回调函数的执行是在用户空间执行，所以并不能简单的进行函数调用。

##七、Timer##
####1、概述####
alarm和nonasleep系统调用都是通过设置定时器，触发函数的定时运行。
####2、数据结构定义####

```c++
struct hrtimer_cpu_base {
	struct hrtimer			*running;
	struct hrtimer_clock_base	clock_base[HRTIMER_MAX_CLOCK_BASES];
} ____cacheline_aligned;

struct hrtimer_clock_base {
	struct hrtimer_cpu_base	*cpu_base;
	int			index;
	clockid_t		clockid;
	struct timerqueue_head	active;
} __attribute__((__aligned__(HRTIMER_CLOCK_BASE_ALIGN)));

struct hrtimer {
	struct timerqueue_node		node;
	ktime_t				_softexpires;
	enum hrtimer_restart		(*function)(struct hrtimer *);
	struct hrtimer_clock_base	*base;
};
```

定义全局变量

```c++
DEFINE_PER_CPU(struct hrtimer_cpu_base, hrtimer_bases) =
{
...
```

####3、设置定时器####

设置定时器的函数为do_setitimer。

它的简化代码如下

```c++
int do_setitimer(int which, struct itimerval *value, struct itimerval *ovalue)
{
	struct task_struct *tsk = current;
	struct hrtimer *timer;
	ktime_t expires;
again:
  		//获取当前进程的timer
		timer = &tsk->signal->real_timer;
		expires = timeval_to_ktime(value->it_value);
		if (expires != 0) {
         	//构建一个定时器对象，装入signal->it_real_incr
			tsk->signal->it_real_incr =
				timeval_to_ktime(value->it_interval);
			hrtimer_start(timer, expires, HRTIMER_MODE_REL);
		} else
			tsk->signal->it_real_incr = 0;
	return 0;
}

```


####5、调度定时器####

__hrtimer_run_queues负责定时器检测任务。当选取到一个到期hrtimer之后，通过

```c++
//kernel/timer/hrtimer.c : 1221
fn = timer->function;
//kernel/timer/hrtimer.c : 1238
restart = fn(timer);
```

执行回调。经过前面的分析，hrtimer的function被设置为it_real_fn。下面继续分析it_real_fn函数，追踪它是如何发生SIGALRM信号的。

it_real_fn执行kill_pid_info函数。

```c++
//kernel/timer/itimer.c : 127
kill_pid_info(SIGALRM, SEND_SIG_PRIV, sig->leader_pid);
```

kill_pid_info调用：

```c++
//kernel/signal.c : 1303
group_send_sig_info(sig, info, p)
```

最终调用\_\_send_signal发送定时器到达信号。__send_signal我们前面已经分析。

```c++
//kernel/signal.c : 1092
__send_signal(sig, info, t, group, from_ancestor_ns);
```

##八、系统调用##
为什么分析这三个系统调用？一个进程在调度结构中，每隔一定时间都会被调度。那么当它执行pause和sleep时，将会停止调度。这两个系统做了什么什么操作让它停止调度。探索这两个系统调用对摸清linux的进程调度方式（结构）有很大帮助。信号绝对是一个“神奇”工作机制，信号从哪里来，如何执行，结果是什么?	当我们在应用层写这些系统调用的时候，它们具体如何工作的？探索alarm，解决这些疑问。当然alarm使用了timer机制，这也是我把timer机制的分析放到前面的原因。
####1、pause####

查看源码，发现pause系统调用十分简单，核心语句就一句：

```c++
//kernel/signal.c : 3532
__set_current_state(TASK_INTERRUPTIBLE)；
```

它把当前进程的状态设置为TASK_INTERRUPTIBLE。也就是说，当前进程还处于调度数据结构里，这里显然有个疑问，它难道不会被调度吗？可以推测，schedule函数显然会有对应判断的机制。返回schedule继续分析调度机制吧。

在__schedule函数中，pick_next_task之前，首先对prev进程做一次检测，如果该进程为不可执行状态。那么就执行deactivate_task函数将其移除调度结构。这就是pause的实现de进程“暂停”实现方式。

```c++
//kernel/sched/core.c : 3367
if (unlikely(signal_pending_state(prev->state, prev))) {
	prev->state = TASK_RUNNING;
} else {
	deactivate_task(rq, prev, DEQUEUE_SLEEP);
```

####2、nanosleep###

nanosleep会让进程睡眠指定时间，时间到达后继续执行。睡眠指定时间有hrtimer设置定时器完成，这一步通过调用hrtimer_set_expires_range_ns完成。睡眠的阻塞效果通过do_nanosleep函数完成。

```c++
//kernel/timer/hrtimer.c : 1471
static int __sched do_nanosleep(struct hrtimer_sleeper *t, enum hrtimer_mode mode)
{
	hrtimer_init_sleeper(t, current);
	do {
		set_current_state(TASK_INTERRUPTIBLE);
		hrtimer_start_expires(&t->timer, mode);
		if (likely(t->task))
			freezable_schedule();
		hrtimer_cancel(&t->timer);
		mode = HRTIMER_MODE_ABS;
	} while (t->task && !signal_pending(current));
	__set_current_state(TASK_RUNNING);
	return t->task == NULL;
}
```

do_nanosleep反复检测当前是否存在信号，用以确定原地阻塞。上一步设置的计时器到达时，将会发生信号，阻塞效果取消，进程得以继续运行。

####3、alarm###
alarm用以设定计时器，它通过设置hrtimer完成定时功能。关于定时器的用法参见第八小结。

```c++
//kernel/time/itimer.c
do_setitimer(ITIMER_REAL, &it_new, &it_old);
```