[toc]  


# 1_linux 内核简介  
## 操作系统和内核简介  
每个处理器在任何指定时间点上的活动必然概括为下列三者之一：  
- 运行于用户空间，执行用户进程
- 运行于内核空间，处于进程上下文，代表某个特定的进程执行  
- 运行于内核空间，处于中断上下文，与任何进程无关，处理某个特定的中断  

以上几乎包括了所有情况，但也有例外，例如，当CPU空闲时，内核就运行一个空进程，处于进程上下文，但运行于内核空间。  
## 单内核与微内核设计的比较  
**单体内核**：大内核，将OS的全部功能都做进内核中，包括调度、文件系统、网络、设备驱动器、存储管理。比如设备驱动管理、资源分配、进程间通信、进程间切换管理、文件系统、存储管理、网络等。单体内核是指在一大块代码中实际包含了所有操作系统功能，并作为一个单一进程运行，具有唯一地址空间。大部分UNIX（包括Linxu）系统都采用的单体内核。

**微内核**：微内核与单体内核不同，微内核只是将OS中最核心的功能加入内核，包括IPC通信、地址空间分配和基本的调度，这些东西处在内核态运行。如：WINCE系统。而其他功能如设备驱动、文件系统、存储管理、网络等作为一个个处于用户态的进程而向外提供某种服务来实现，而且这些处于用户态的进程可以针对某些特定的应用和环境需求进行定制。有时，也称这些进程为服务器。  
**Linux是一个单内核，也就是说，Linux内核运行在单独的内核地址空间上**。不过，Linux汲取了微内核的精华：其引以为豪的是模块化设计、抢占式内核、支持内核线程以及动态装载内核模块的能力。不仅如此，Linux还避开微内核设计上性能损失的缺陷，让所有事情都运行在内核态，直接调用函数，无须消息传递。  

# 2_从内核出发  

## 内核开发的特点  
内核开发的特点：  
- 内核编程时既不能访问C库也不能访问标准的C头文件  
- 内核编程必须使用GNU C
- 内核编程时缺乏像用户空间那样的内存保护机制  
- 内核编程时难以执行浮点运算  
- 内核给每个进程只有一个很小的定长堆栈  
- 由于内核支持异步中断、抢占、SMP，因此必须时刻注意并发和同步  
- 要考虑可移植性  

# 3_进程管理  

执行线程，简称线程，是在进程中活动的对象。**每个线程都拥有一个独立的程序计数器、进程栈和一组进程寄存器**。内核调度的对象是线程，而不是进程。  

## 进程描述符及任务结构  
内核把进程的列表存放在叫做任务列表(task list)的**双向循环链表中**。链表中的每一项都是类型为task_struct的进程描述符结构。  

Linux通过slab分配器分配task_struct结构，这样能达到对象复用和缓存着色的目的。在2.6以前的内核中，各个进程的task_struct存放在它们内核栈的尾端。这样做是为了让那些像x86那样寄存器较少的硬件体系结构只要通过栈指针就能计算出它的位置，而避免使用额外的寄存器专门记录。**由于现在使用slab分配器动态生成task_struct，所以只需在栈底(对于向下增长的栈来说)或栈顶(对于向上增长的栈来说)创建一个新的结构struct thread_info，该结构中包含task_struct结构的指针**。  

```
struct thread_info {
	struct task_struct *task;	/* main task structure */
	struct exec_domain *exec_domain;/* execution domain */
	unsigned long flags;		/* thread_info flags (see TIF_*) */
	mm_segment_t addr_limit;	/* user-level address space limit */
	__u32 cpu;			/* current CPU */
	int preempt_count;		/* 0=premptable, <0=BUG; will also serve as bh-counter */
	struct restart_block restart_block;
};
```  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/A5871B01087D44539C55AC9550037A2B/13259)

每个任务的thread_info结构在它的内核栈的尾端分配。结构中task域中存放的是指向该任务实际task_struct的指针。  

**获取task_struct的指针：**   
有的硬件体系结构(例如，Power PC )可以拿出一个专门寄存器来存放指向当前进程task_struct的指针，用于加速快速访问；而像X86这样的体系结构，寄存器并不富裕，只能通过在栈尾创建thread_info结构，通过计算偏移间接地查找task_struct结构。  
在ARM中：  

```
static inline struct thread_info *current_thread_info(void)
{
	register unsigned long sp asm ("sp");
	return (struct thread_info *)(sp & ~(THREAD_SIZE - 1));
}
```  
这里栈的大小为8KB，THREAD_SIZE为8192，得到栈底的thread_info结构，通过thread_info->task得到task_struct结构。  

## 进程创建  
Linux fork()使用写时拷贝页实现，资源的复制只有在需要写入的时候才进行，在此之前，只是以只读方式共享。  

### fork  
fork()、vfork()、_clone()库函数都是根据各自需要的参数标志调用clone(),然后clone()去调用do_fork()。do_fork()完成了创建中的大部分工作，该函数调用copy_process()函数完成如下工作：  
1. 调用dup_task_struct()为新进程创建一个内核栈、thread_info结构和task_struct，这些值与当前进程的值相同。此时，子进程与父进程的描述符是完全相同的。  
2. 检查并确保新创建的子进程后，当前用户所拥有的进程数目没有超出给它分配的资源的限制  
3. 子进程着手使自己与父进程区别开来。进程描述符内的许多成员都被清0或着设为初始值。那些不是继承而来的进程描述符成员，主要是统计信息。task_struct中的大多数数据都依然未被修改。  
4. 子进程的状态设置为TASK_UNINTERRUPTIBLE,以保证它不会投入运行  
5. copy_process()更新task_struct的flags成员。
6. 为新进程分配有效的PID。
7. 根据clone()的参数标志，拷贝或共享打开的文件、文件系统信息、信号处理函数、进程地址空间、命名空间等。在一般情况下，这些资源会被给定进程的所有线程共享；否则，这些资源对每个进程是不同的，因此被拷贝到这里。  
8. 返回一个指向子进程的指针。  

在回到do_fork()函数，如果copy_process()函数返回成功，新创建的子进程被唤醒并让其投入运行。内核有意选择子进程首先执行。因为子进程会马上调用exec()函数，这样就避免了写时拷贝的额外开销，如果父进程首先执行的话，有可能向地址空间写入。  

### vfork  
vfork()除了不拷贝父进程的页表外，和fork()的功能相同。子进程作为父进程的一个单独的线程在它的地址空间运行，父进程被阻塞，直到子进程退出或调用exec()。  
vfork()系统调用的实现是通过向clone()系统调用传递一个特殊标志来进行的：  
1. 在调用copy_process()时，task_struct的vfork_done成员被设置为NULL。  
2. 在执行do_fork()时，如果给定特别标志，则vfork_done会指向一个特定的地址  
3. 子进程先运行，父进程等待，直到子进程通过vfork_done指针向它发送信号  
4. 在调用mm_release()时，该函数用于进程退出内存地址空间，并检查vfork_done是否为空，如果不为空，则向父进程发送信号。  
5. 回到do_fork()，父进程醒来并返回。  


# 4_进程调度  

## linux 的进程调度  
从1991年Linux的第一版到后来的2.4内核系列，Linux的调度程序都相当简陋，在众多可运行进程或者多处理器的环境下都难以胜任。  
在Linux 2.5中采用O(1)调度程序的新调度程序，主要感谢**静态时间片算法和针对每一处理器的运行队列**，它们帮助我们摆脱了先前调度程序设计上的限制。O(1)调度算法在交互进程上表现不佳，在服务器上表现良好。  
在Linux 2.6 中，为了提高对交互程序的调度性能引入了新的调度算法。其中最著名的是**反转楼梯最后期限调度算法**(Rotating Staircase Deadline scheduler)(RSDL),该算法吸取了队列理论，将公平调度的概念引入了Linux调度程序。并且最终在2.6.23内核版本中替代了O(1)算法，它此刻被称为**“完全公平调度算法”**，或简称CFS。  

## 策略  

### I/O消耗型和处理器消耗型的进程  
进程可以被分为I/O消耗型和处理器消耗型。前者指进程的大部分时间用来提交I/O请求或是等待I/O请求。处理器消耗型进程把时间大多用在执行代码上。对于处理器消耗型进程，调度策略往往是尽量降低它们的调度频率，而延长其运行时间。对于I/O消耗型往往需要较高的响应。  

### 进程优先级  
Linux 采用两种不同的优先级范围。第一种是用nice值，它的范围是从-20~+19，默认值为0；越大的nice值意味着更低的优先级。相比高nice值(低优先级)的进程，低nice值(高优先级)的进程可以获得更多的处理器时间。有些系统(如Mac OS X)进程的nice值代表分配给进程的时间片的绝对值；**而Linux系统中，nice值则代表时间片的比例**。  
第二种范围是实时优先级，其值是可配置的，默认情况下它的变化范围是从0到99。在系统内部，0是实时优先级的最高级。**任何实时进程的优先级都高于普通进程的优先级**，也就是说实时优先级和nice优先级处于互不相交的两个范畴。  
### 时间片  
时间片是一个数值，它表明进程在被抢占前所能持续运行的时间。调度策略必须规定一个默认的时间片，但是这并不是件简单的事。I/O消耗型不需要长的时间片，而处理器消耗型的进程则希望越长越好。任何长时间片都将导致系统交互表现欠佳。  
分配绝对的时间片引发的固定的切换频率，给公平性造成了很大变数。CFS采用的方法是对时间片分配方式进行根本性的重新设计：完全抛弃时间片而是**分配给进程一个处理器使用比重**。通过这种方式，CFS确保了进程调度中能有恒定的公平性，而将切换频率至于不断变动中。  

## Linux 调度算法  
**linux 调度器是以模块方式提供的，这样做的目的是允许不同类型的进程可以有针对性地选择调度算法**。这种模块化结构被称为调度器类，它允许多种不同的可动态添加的调度算法并存，调度属于自己范畴的进程。每个调度器都有一个优先级，基础的调度器代码定义在sched.c文件中，它会按照优先级顺序遍历调度类，拥有一个可执行进程的最高优先级的调度器类胜出，去选择下面要执行的那个程序。  
**Linux系统的线程是内核线程，所以Linux系统的调度是基于线程的，而不是基于进程的**。  
为了进行调度，Linux系统将线程区分为三类：  
- 实时先进先出 SCHED_FIFO
- 实时轮转 SCHED_RR
- 分时 SCHED_NORMAL  

实时优先级为0~ 99，非实时优先级(普通线程)为100~ 139(默认情况下对应nice的-20~19)。
### 实时调度  
Linux提供了两种实时调度策略: SCHED_FIFO和SCHED_RR。  
SCHED_FIFO实现了一种简单的、先进先出的调度算法：**它不使用时间片**。处于可运行状态的SCHED_FIFO级的进程会比任何SCHED_NORMAL级的进程都先得到调度。一旦一个SCHED_FIFO级进程处于可执行状态，就会一直执行，知道它自己受阻塞或显式地释放处理器为止；它不基于时间片，可以一直执行下去。**只有更高级的SCHED_FIFO或SCHED_RR任务才能抢占SCHED_FIFO任务**。如果有两个或多个同级的SCHED_FIFO级进程，它们会轮流执行，**但是依然只有它们愿意让出处理器时才会退出**。只要有SCHED_FIFO级进程在执行，其他级别较低的进程就只能等待它变为不可运行后才有机会执行。  
SCHED_RR与SCHED_FIFO大体相同，只是SCHED_RR级的进程在耗尽事前分配给它的时间后就不能再继续执行了，也就是说，**SCHED_RR是带时间片的SCHED_FIFO**--这是一种实时轮流调度算法。当SCHED_RR任务耗尽它的时间片时，在同一优先级的其他实时进程被轮流调度。时间片只用来重新调度同一优先级的进程。对于SCHED_FIFO进程，高优先级总是立即抢占低优先级，但低优先级的进程决不能抢占SCHED_RR任务，即使它的时间片耗尽。  
这两种实时算法实现的都是**静态优先级**。内核不为实时进程计算动态优先级，这能保证给定优先级别的实时进程总能抢占优先级比它低的进程。  

### SCHED_NORMAL : CFS  
CFS是对于**普通进程的调度策略**，其出发点基于简单的理念：进程调度的效果应如同系统具备一个理想中的完美多任务处理器。在这种系统中，**每个进程将能获得1/n的处理器时间**--n是进程的数量，同时，我们可以调度给他们**无限小的时间周期**，所以在任何可测量周期内，我们给予n个进程中**每个进程同样多的运行时间**。    

现实的处理器，在一个处理器上无法真的同时运行多个进程。而且如果每个进程运行无限小的时间周期也不是高效的--因为调度时进程抢占时会带来一定的代价：将一个进程换出，另一个进程换入本身有消耗，同时还会影响缓存的效率。**CFS的做法是：允许每个进程运行一段时间、循环轮转、选择运行最少的进程作为下一个运行进程，而不是采用分配给每个进程时间片的做法，CFS在所有可运行进程总数基础上计算出一个进程应该运行多久，而不是依靠nice值来计算时间片**。nice值在CFS中被作为进程获得的处理器运行比的权重：越高的nice(越低的优先级)进程获得更低的处理器使用权重，这是相对默认nice值进程的进程而言的；相反，更低的nice值(越高的优先级)的进程获得更高的处理器使用权重。  
每个进程都按其权重在全部可运行进程中所占比例的“时间片”来运行，为了计算准确的时间片，CFS为完美多任务中的无限小调度周期的近似值设立一个目标。而这个目标称作**“目标延时”**，越小的调度周期带来越好的交互性，同时也接近多任务，但也损失吞吐能力。同时，CFS为每个进程获得的时间片设置底线，这个底线称为**最小粒度**，默认情况下是1ms。即便可运行进程数量趋于无穷，每个进程最少也能获得1ms的运行时间，确保了切换消耗被限制在一定范围内。  
任何进程获得的处理器时间是由它自己和其他所有可运行进程nice值的相对差值决定的。nice值对时间片的作用不再是算数加权，而是**几何加权**。  
CFS使用一颗红黑树作为调度队列的数据结构。  
CFS具体实现：  
- 时间记账
- 进程选择 
- 调度器入口 
- 睡眠和唤醒 

#### 时间记账  
CFS不再有时间片的概念，但是它也必须**维护每个进程运行的时间记账**，因为它需要确保每个进程只在公平分配给它的处理器时间内运行。CFS使用调度器实体结构sched_entity来追踪进程运行记账。每个task_struct都嵌入一个sched_entity结构。  

```
struct sched_entity {
	struct load_weight	load;		/* for load-balancing */ //当前负载
	struct rb_node		run_node; //红黑树连接
	struct list_head	group_node;
	unsigned int		on_rq; //是否在队列上

	u64			exec_start; // 开始执行时间
	u64			sum_exec_runtime; //总执行时间
	u64			vruntime; //虚拟运行时间
	u64			prev_sum_exec_runtime;

	u64			nr_migrations;
	....
	}
```
**虚拟实时：**   
**vruntime变量存放进程的虚拟运行时间，该运行时间(花在运行上的时间和)的计算是经过了所有可运行进程总数的标准化(或者说被加权的)**。虚拟时间是以ns为单位的，所以vruntime和定时器节拍不再相关。CFS使用vruntime变量来记录一个程序到底运行了多长时间以及它还应该运行多久。update_curr()函数实现了记账功能，主要步骤：  
- 首先计算进程当前时间与上次启动时间的差值(真实的时间)  
- 通过负荷权重和当前时间模拟出进程的虚拟运行时间  
- 重新设置cfs的min_vruntime保持其单调性  

**时间差计算:**  
如果在CFS就绪队列上无进程，则返回；否则，内核计算当前和上一次更新负荷权重时两次的时间差值delta，然后重新更新启动时间exec_start为now  
**虚拟运行时间计算：**  
代码如下，其中calc_delta_fair()函数是计算的关键。  
忽略舍入和溢出检查，calc_delta_fair函数所做的就是根据下列公式计算：  

```math
delta = delta * \frac{NICE0LOAD}{curr \to se \to load.weight}
```
每个进程拥有一个vruntime，每次需要调度的时候就先运行红黑树中拥有最小的vruntime的那个进程来运行。vruntime增长如下：  
```math
curr \to vruntime + = delta * \frac{NICE0LOAD}{curr \to se \to load.weight}  
```
在该计算中，可知越重要的进程会有越高的优先级(越低的nice值)，会得到更大的权重，因此累加的虚拟运行时间会小一点。  
nice 与 weight的转换通过查找转换数组，prio_to_weight[40]数组，nice对应的值-20最大，19最小。  

```
static inline void
__update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,
	      unsigned long delta_exec)
{
	unsigned long delta_exec_weighted;

	schedstat_set(curr->statistics.exec_max,
		      max((u64)delta_exec, curr->statistics.exec_max));
//将时间差加到先前统计的时间
	curr->sum_exec_runtime += delta_exec;
	schedstat_add(cfs_rq, exec_clock, delta_exec);
	delta_exec_weighted = calc_delta_fair(delta_exec, curr); //根据nice计算权重,delta/=w

	curr->vruntime += delta_exec_weighted;//更新vruntime
	update_min_vruntime(cfs_rq);//更新最小vruntime

#if defined CONFIG_SMP && defined CONFIG_FAIR_GROUP_SCHED
	cfs_rq->load_unacc_exec_time += delta_exec;
#endif
}
static inline unsigned long
calc_delta_fair(unsigned long delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = calc_delta_mine(delta, NICE_0_LOAD, &se->load);

	return delta;
}
/*
 * delta *= weight / lw
 */
static unsigned long
calc_delta_mine(unsigned long delta_exec, unsigned long weight,
		struct load_weight *lw)
{
	u64 tmp;

	if (!lw->inv_weight) {
		if (BITS_PER_LONG > 32 && unlikely(lw->weight >= WMULT_CONST))
			lw->inv_weight = 1;
		else
			lw->inv_weight = 1 + (WMULT_CONST-lw->weight/2)
				/ (lw->weight+1);
	}

	tmp = (u64)delta_exec * weight;
	/*
	 * Check whether we'd overflow the 64-bit multiplication:
	 */
	if (unlikely(tmp > WMULT_CONST))
		tmp = SRR(SRR(tmp, WMULT_SHIFT/2) * lw->inv_weight,
			WMULT_SHIFT/2);
	else
		tmp = SRR(tmp * lw->inv_weight, WMULT_SHIFT);

	return (unsigned long)min(tmp, (u64)(unsigned long)LONG_MAX);
}
```

#### 进程选择  
**CFS选择下一个运行进程时，它会挑选一个具有最小vruntime的进程。**  
CFS使用红黑树来组织可运行进程队列，并利用其迅速找到最小的vruntime值的进程。  

#### 调度器入口  
进程调度器主要入口是函数schedule()，它会找到一个最高优先级的调度类--后者需要有自己的可运行队列，然后问后者谁才是下一个要运行的进程。  

#### 睡眠和唤醒  
当进程阻塞或睡眠时，进程把自己标志为休眠状态，从可执行红黑树中移出，放入等待队列，然后调用schedule()选择和执行一个其他进程。唤醒的过程刚好相反：进程被设置为可执行状态，然后再从等待队列中移到可执行红黑树中。  

## 抢占和上下文切换  
进程切换有自愿(Voluntary)和强制(Involuntary)之分，在前文中详细解释了两者的不同，简单来说，自愿切换意味着进程需要等待某种资源，强制切换则与抢占(Preemption)有关。  

抢占(Preemption)是指内核强行切换正在CPU上运行的进程，在抢占的过程中并不需要得到进程的配合，在随后的某个时刻被抢占的进程还可以恢复运行。发生抢占的原因主要有：进程的时间片用完了，或者优先级更高的进程来争夺CPU了。  

抢占的过程分两步，第一步触发抢占，第二步执行抢占，这两步中间不一定是连续的，有些特殊情况下甚至会间隔相当长的时间：  

- 触发抢占：给正在CPU上运行的当前进程设置一个请求重新调度的标志(TIF_NEED_RESCHED)，仅此而已，此时进程并没有切换。
- 执行抢占：在随后的某个时刻，内核会检查TIF_NEED_RESCHED标志并调用schedule()执行抢占。  

抢占只在某些特定的时机发生，这是内核的代码决定的。  
触发抢占的时机：  
- 周期的时钟中断
- 唤醒进程的时候
- 新进程创建的时候
- 进程修改nice值的时候
- 进行负载均衡的时候

### 用户抢占  
内核即将返回用户空间的时候，如果need_resched标志被设置，会导致schedule()被调用，此时就会发生用户抢占。  
用户抢占在一下情况下发生：  
- 从系统调用返回用户空间时
- 从中断处理程序返回用户空间时  

### 内核抢占  
内核抢占简而言之就是内核代码也可以被抢占。只要重新调度是安全的，内核就可以在任何时间抢占正在执行的任务。  
内核抢占会发生在：  
- 中断处理程序正在执行，且返回内核空间之前  
- 内核代码再一次具有可抢占性的时候  
- 如果内核中的任务显式地调用schedule()
- 如果内核中的任务阻塞(这同样也会导致调用schedule())。  



```math
 \sum_{i=1}^{n} {\sqrt[4]{2} } = \prod_{i=1}^{n} \frac{1}{i^2}
```

# 5_系统调用  

```
asmlinkage long sys_getpid();
```
函数声明中的asmlinkage限定词，这是一个**编译指令**，通知编译器仅从栈中提取函数的参数。所有的系统调用都需要这个限定词。  


内核在执行系统调用的时候处于进程上下文。current指针指向当前任务，即引发系统调用的那个进程。在进程上下文中，内核可以休眠并且可以抢占。在进程上下文中能够被抢占其实表明，像用户空间内的进程一样，当前进程同样可以被抢占。因为新的进程可以使用相同的系统调用，所有必须小心，必须保证该系统调用是可重入的。当系统调用返回的时候，控制权任然在system_call()中，它最终会负责切换到用户空间，并让用户进程继续执行下去。  


# 6_内核数据结构  

## 双向链表  
struct list_head  

## 队列  
struct kfifo  

## 红黑树  


# 7_中断和中断处理
所谓的中断机制，就是在硬件需要的时候向内核发出的信号，从而相应的硬件获得CPU的使用。  
中断是硬件发出的，是随机的；异常是CPU执行指令时发生的，必须考虑到与处理器时钟同步。  

## 中断处理程序  
中断处理程序是被内核调用来响应中断的，运行在中断上下文中。中断处理程序要负责通知硬件设备中断已被接收。  

## 上半部与下半部的对比  
中断处理程序运行的要快，又要完成工作量多，这就把终端处理切为两部分。中断处理程序是上半部，即接收到一个中断，立即开始执行，但只做严格时限的工作，例如对硬件应答等，这些工作都是在所有中断被禁止的情况下完成的。能够被稍后完成的工作会推迟到下半部去。  

## 注册中断处理程序  
```
int request_irq(unsigned int irq,
		irq_handler_t handler,
		unsigned long flags, const char *devname, void *dev_id);
		
typedef irqreturn_t (*irq_handler_t)(int, void *);	

void free_irq(unsigned int irq, void *dev_id)；
```
参数：  
- irq: 中断号
- handler:中断处理程序
- flags: 中断处理标志，可以为0，IRQF_DISABLED、IRQF_SAMPLE_RANDOM、IRQF_TIMER、IRQF_SHARED  
- devname: 与中断相关的设备名字，会在/proc/irq 和/proc/interrupts文件中使用
- dev_id : 用于共享中断线，唯一  


## 中断上下文  
**中断处理程序栈的设置是一个配置选项**。曾经，中断处理程序并不具有自己的栈。相反，它们共享所中断进程的内核栈。内核栈的大小是两页，也就是说32位体系架构上是8KB，64位体系架构上是16KB。中断处理程序运行在别人的栈上，所有它们应该非常节约空间。  
在2.6版早期的内核中，增加了一个选项，把栈的大小从两页减到一页。这就减轻了内存的压力，因为系统中每个进程原先都要两页连续，且不可换出的内核内存。为了应对栈大小的减少，中断处理程序拥有了自己的栈，每个处理器一个，大小为一页。这个栈就是中断栈，尽管中断栈的大小只有原先共享栈的一半，但平均可用栈空间大得多，因为中断处理程序把这一整页占为己有。  


## 中断处理机制的实现  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/319F46CD652E4540A6F5A53467CCB043/14337)  

## 中断控制
**一般来说，控制中断系统的原因归根到底是需要提供同步。通过禁止中断，可以确保某个中断处理程序不会抢占当前的代码。此外，禁止中断还可以禁止内核抢占。然而，不管是禁止中断还是禁止内核抢占，都没有提供任何保护机制来防范来自其他处理器的并发访问**。Linux支持多处理器，因此，内核代码一般都需要获取某种锁，防止来自其他处理器的并发访问。获取这些锁的同时也伴随着禁止本地中断。  


函数 | 说明
---|---
local_irq_disable() | 禁止本地中断传递
local_irq_enable() | 激活本地中断传递
local_irq_save() | 保存本地中断传递的当前状态，然后禁止本地中断传递
local_irq_restore() | 恢复本地中断传递到给定的状态
disable_irq() | 禁止给定中断线，并确保该函数返回之前在该中断线上没有处理程序在运行
disable_irq_nosync() | 禁止给定中断
enable_irq() | 激活给定中断线
irqs_disabled() | 如果本地中断传递被禁止，则返回非0；否则返回0
in_interrupt() | 如果在中断上下文中，则返回非0；若果在进程上下文中，则返回0
in_irq() | 若当前正在执行中断处理程序，则返回非0；否则返回0

# 8_下半部和退后执行的工作  
下半部的任务就是执行与中断处理密切相关但中断处理程序本身不执行的工作。在理想情况下，最好是中断处理
程序将所有工作都交给下半部分执行。  

Linux 2.6版本中有三种机制可以实现将工作推后执行：软中断、tasklet和工作队列。  
**软中断和tasklet**  
软中断是一组**静态**定义的下半部接口，有32个，可以在所有处理器上同时执行--即使两个类型相同也可以。tasklet是一种基于软中断实现的**灵活性强、动态创建**的下半部实现机制。两个不同类型的tasklet可用在不同的处理器上同时执行，但是类型相同的tasklet不能同时执行。两个相同的软中断有可能同时被执行，此外，软中断还必须在编译期间就进行静态注册。与此相反，taskLet可以通过代码进行动态注册。  
## 软中断
软中断保留给系统中对时间要求最严格以及最重要的下半部使用。一个软中断不会抢占另外一个软中断，实际上，唯一可以抢占软中断的是中断处理程序。不过，其他的软中断(甚至是相同类型的软中断)可以在其他处理器上同时执行。  
一个注册的软中断必须在被标记后才会执行，这被称作触发软中断。通常，中断处理程序会在返回前标记它的软中断，使其在稍后被执行。于是，在合适的时刻，该软中断就会运行，待处理的软中断会被检查和执行：  
- 从一个硬件中断代码处返回时
- 在ksoftirqd内核线程中  
- 在那些显示检查和执行待处理的软中断的代码中，如网络子系统中  

使用软中断：  
1. 分配索引  
```
enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,
	BLOCK_SOFTIRQ,
	BLOCK_IOPOLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,	/* Preferable RCU should always be the last softirq */

	NR_SOFTIRQS
};
```
索引号越小优先级越高。建立一个新的软中断必须在此枚举类型中加入新的项。   
2. 注册处理程序  
在运行时通过调用open_softirq注册软中断处理程序，该函数有三个参数：软中断的索引号、处理函数和dada域存放的数值。  
eg: open_softirq(NET_TX_SOFTIRQ,next_tx_action,NULL)  
**软中断处理程序执行时，允许影响中断，但它自己不能休眠**。在一个处理程序运行的时候，当前处理器上的软中断被禁止。但是其他的处理器仍能够执行别的软中断。实际上，如果同一个软中断在它被执行的同时再次被触发，那么另一个处理器可以同时运行其处理程序。这意味着任何共享数据，甚至在软中断处理程序内部使用的全局变量，都需要严格的锁保护。如果仅仅通过互斥的加锁方式来防止它自身的并发执行，那么使用软中断就没有任何意义了。因此，大部分软中断处理程序，都采用单处理器数据(仅属于某一个处理器的数据，因此根本不用加锁)或其他一些技巧来避免显示地加锁。    
3. 触发软中断  
raise_softirq()函数可以将一个软中断设置为挂起状态，让他在下次调用do_softirq()函数时投入运行。  
eg: raise_softirq(NET_TX_SOFTIRQ);  
如果中断本来本来就已经禁止了，那么可以调用另一个函数raise_softirq_irqoff():  
eg: raise_softirq_irqoff(NET_TX_SOFTIRQ);  



## tasklet

tasklet是利用软中断实现的一种下半部机制。tasklet由两类软中断代表：HI_SOFTIRQ和TASKLET_SOFTIRQ，这两者的区别在于优先级不同而已。  
```
struct tasklet_struct
{
	struct tasklet_struct *next; //链表中下一个tasklet
	unsigned long state; //tasklet的状态
	atomic_t count;  //引用计数
	void (*func)(unsigned long); //tasklet处理函数
	unsigned long data; //给tasklet处理函数的参数
};
```
### tasklet的调度
tasklet由tasklet_schedule()和tasklet_hi_schedule()函数进行调度，它们会吧需要调度的tasklet加入到每个处理器的一个tasklet_vec链表或tasklet_hi_vec链表的表头中,然后唤醒软中断，这样在下一次调用do_softirq()时就会执行该tasklet。
### 使用tasklet
1. 声明一个tasklet  
静态创建：  
   DECLARE_TASKLET(name, func, data);  
   DECLARE_TASKLET_DISABLED(name, func, data);  
前者把tasklet的引用计数设置为0，该tasklet处于激活状态。另一个把引用计数器设置为1，该tasklet处于禁止状态。  

动态创建：  
  tasklet_init(struct tasklet_struct *t,
		  void (*func)(unsigned long), unsigned long data)；  
2. tasklet处理函数  
    void func(unsigined long )  

tasklet实现是靠软中断，所以tasklet不能睡眠，这意味tasklet实现中不能使用信号量或其他阻塞式的函数。  
3. 调度tasklet  
void tasklet_schedule(struct tasklet_struct *t);  

在tasklet被调度后，只要有机会他就会尽可能早地运行。在它运行之前，如果一个相同的tasklet又被调度，那么它仍然只运行一次。可以调用tasklet_disable()函数来禁止某个指定的tasklet，如果该tasklet正在执行，这个函数会等待它执行完毕再返回。  
   tasklet_disable_nosync():禁止指定的tasklet，无需等待。  
   tasklet_enable():激活一个tasklet  
   tasklet_kill():从挂起的队列中去掉一个tasklet  
### ksoftirqd
每个处理器都有一组辅助处理软中的内核线程。当内核中出现大量软中断的时候，这些内核进程就会辅助处理它们。这些内核线程在最低的优先级上运行(nice值为19),这能避免它们根其他重要的任务抢夺资源。  

## 工作队列
工作队列(work queue)可以把工作退后，交由一个内核线程去执行--这个下半部分总是会在进程上下文执行。通过工作队列执行的代码能占尽进程上下文的所有优势。最重要的就是工作队列允许重新调度甚至睡眠。  
```
typedef void (*work_func_t)(struct work_struct *work);
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
struct workqueue_struct {
	unsigned int		flags;		/* I: WQ_* flags */
	union {
		struct cpu_workqueue_struct __percpu	*pcpu;
		struct cpu_workqueue_struct		*single;
		unsigned long				v;
	} cpu_wq;				/* I: cwq's */
	struct list_head	list;		/* W: list of all workqueues */

	struct mutex		flush_mutex;	/* protects wq flushing */
	int			work_color;	/* F: current work color */
	int			flush_color;	/* F: current flush color */
	atomic_t		nr_cwqs_to_flush; /* flush in progress */
	struct wq_flusher	*first_flusher;	/* F: first flusher */
	struct list_head	flusher_queue;	/* F: flush waiters */
	struct list_head	flusher_overflow; /* F: flush overflow list */

	mayday_mask_t		mayday_mask;	/* cpus requesting rescue */
	struct worker		*rescuer;	/* I: rescue worker */

	int			saved_max_active; /* W: saved cwq max_active */
	const char		*name;		/* I: workqueue name */
#ifdef CONFIG_LOCKDEP
	struct lockdep_map	lockdep_map;
#endif
};
```
使用工作队列：  
1. 创建退后的工作  
    DECLARE_WORK(name, void (*func)(void*),void *data)  
    INIT_WORK(struct work_struct *work, void (*func)(void*),void *data);  

2. 工作队列处理函数  
    void func(void * data)  
    这个函数会由一个工作者线程执行，因此，函数会运行在进程上下文中。  

3. 对工作进行调度  
    把指定的工作处理函数提交给默认的events工作线程，只需调用：  
    schedule_work(&work)  
    延迟指定的时间执行：  
    schedule_delayed_work(&work,delay)  

4. 刷新操作  
    void flush_scheduled_work(struct work_struct *work)  
    函数会一直等待，直到队列中所有对象都被执行以后才返回。  
取消延迟执行的工作：  
    int cancel_delayed_work(struct work_struct *work);  

5. 创建新的工作队列  
如果默认的工作队列补鞥呢满足你的需要，你应该创建一个新的工作队列和与之相应的工作者线程。由于这么做会在每个处理器上都创建一个工作者线程，所以只有你明确了必须要自己的一套线程来提高性能的情况下，再创建自己的工作队列。
创建一个新的任务队列和与之相关的工作者线程，只需调用一个简单的函数：  
    struct workqueue_struct *create_workqueue(const char *name);  
    name 参数用于该内核线程的命名。  
创建之后，可以调用下面列举的函数，这些函数与schedule_work()以及schedule_delayed_work()相近，唯一的区别是它们针对给定的工作队列而不是默认的event队列进行操作。  
    初始化一个work_struct ：  
    INIT_WORK(struct work_struct *work, void (*func)(void*),void *data);  
    进行调度：  
    int queue_work(struct workqueue_struct *wq, struct work_struct *work);  
    int queue_delayed_work(struct workqueue_struct *wq,
			struct delayed_work *dwork, unsigned long delay);  
最后调用下面函数刷新指定的工作队列：  
    void flush_workqueue(struct workqueue_struct *wq);  
当你用完一个工作队列, 你可以去掉它, 使用:   
    void destroy_workqueue(struct workqueue_struct *queue);   


# 9_内核同步介绍 

## 到底是什么造成了并发执行？

用户空间之所以需要同步，是因为用户程序会被调度程序抢占和重新调度。由于用户进程可能在任何时刻被抢占，而调度程序完全可能选择另一个高优先级进程到处理器上执行，所以有可能在一个进程正在处理临界区时，就被非自愿地抢占了，如果新调度的进程随后也进入同一个临界区，这样前后两个进程就会产生竞争。在同一个处理器上称为伪并发执行。  
如果在一个支持对称多处理器的机器，那么两个进程就可以真正的在临界区中同时执行，这种称为真并发。  
内核中有类似可能造成并发执行的原因：  
1. 中断 --中断几乎可以在任何时候异步发生，也就可能随时打断当前正在执行的代码  
2. 软中断和tasklet--内核能在任何时刻唤醒或调度软中断和tasklet，打断当前正在执行的代码  
3. 内核抢占--因为内核具有抢占性，所以内核中的任务可能会被另一个任务抢占  
4. 睡眠及与用户空间的同步--在内核执行的进程可能会睡眠，这就会唤醒调度程序，从而导致
调度一个新的用户进程执行  
5. 对称多处理器--两个或多个处理器可以同时执行代码  

## 争用和扩展性  
锁的争用是指当锁正在被占用时，有其他线程试图获得该锁。说一个锁处于高度争用状态，就是指有多个其他线程在等待获得该锁。由于锁的作用是使程序以串行方式对资源进行访问，所有使用锁无疑会降低系统的性能。  
扩展性是对系统可扩展程度的一个度量。  

## 锁机制

解决并发访问数据引起的竞争问题，Linux内核引入锁机制，但是加锁要注意  
为什么要加锁，对什么数据加锁，对多大的数据块加锁，即加锁要保护什么。  
加锁时要注意死锁的问题，加锁顺序要一致，加锁时对争用锁和扩展性要进  
行思索，加锁过粗与过细可能带来的问题。  
加锁机制的精髓：力求简单

# 10_内核同步方法

## 原子操作：

原子操作可以保证指令以原子的方式执行，即执行的过程不被打断。内核提  
供了两组原子操作接口：针对整数进行操作与针对单独的位进行操作。


## 自旋锁(spin lock)
自旋锁最多只能被一个可执行线程持有。如果一个执行线程试图获得一个  
被争用(已经被持有)的自旋锁，那么该线程会一直忙等下去直到锁可用；  
如果锁未被争用，请求锁的执行线程就会立刻得到它，继续执行。  
自旋锁的实现和体系结构密切相关，代码通常通过汇编实现。这些与体系  
结构相关的代码定义在<asm/spinlock.h>中，实际需要用到结构定义在  
文件<linux/spinlock.h>中。  
自旋锁的基本使用形式如下：  
```
    spinlock_t mr_lock = SPIN_LOCK_UNLOCKED;
    spin_lock(&mr_lock);
    /*临界区*/
    spin_unlock(&mr_lock);
```    
注意：Linux内核实现的自旋锁不可递归  

自旋锁可以使用在中断处理程序中(此处不能使用信号量，因为他们会导致睡眠)。  
在中断处理程序中使用自旋锁时，一定要在获取锁之前，首先禁止本地中断  
(在当前处理器上的中断请求)，否则，中断处理程序就会打断正持有锁的内核代码，  
有可能会试图去争用这个已经被持有的自旋锁。这样一来，中断处理程序就会自旋，  
等待该锁重新可用，但是锁的持有者在这个中断处理程序执行完毕前不可能运行，  
这就导致**双重请求死锁**。  

内核提供的禁止中断同时请求锁的接口：
```
    spinlock_t mr_lock = SPIN_LOCK_UNLOCKED;
    unsigned long flags;
    spin_lock_irqsave(&mr_lock,flags);
    /*临界区*/
    spin_unlock_irqrestore(&mr_lock,flags);
```
自旋锁方法列表：
- spin_lock() : 获取指定的自旋锁
- spin_lock_irq() : 禁止本地中断并获取指定的自旋锁
- spin_lock_irqsave() : 保存本地中断的当前状态，禁止本地中断，并获取指定的锁
- spin_unlock() : 释放指定的锁
- spin_unlock_irq() : 释放指定的锁，并激活本地中断
- spin_unlock_irqrestore() : 释放指定的锁，并让本地中断恢复到以前状态
- spin_lock_init() : 动态初始化指定的spinlock_t
- spin_trylock() : 试图获取指定的锁，如果未获取，则返回非0
- spin_is_locked() : 如果指定的锁当前正在被获取，则返回非0，否则，返回0


## 读-写自旋锁
有时，锁的用途可以明确地分为读取和写入。例如任务链表的存取模式。  
Linux内核提供了专门的读-写自旋锁，这种自旋锁为读和写分别提供了不同的锁。  
一个或多个读任务可以并发的持有读者锁；相反，用于写的锁最多只能被一个写  
任务持有，而且此时不能有并发的读操作。  
```
读写锁初始化：
    rwlock_t mr_rwlock = RW_LOCK_UNLOCKED;
读者的代码分支使用函数：
    read_lock(&mr_rwlock);
    /*临界区(只读)*/
    read_unlock(&mr_rwlock);
    
写者的代码分支使用函数：
    write_lock(&mr_rwlcok);
    /*临界区(读写)*/
    write_unlock(&mr_rwlock);

```

## 信号量
linux中的信号量是一种**睡眠锁**。如果有一个任务试图获得一个已经被占用的  
信号量时，信号量会将其推进一个等待队列，然后让其睡眠，这是处理器能重获自  
由，从而执行其他代码；当持有信号量的进程将信号量释放后，处于等待队列中的  
那个任务将被唤醒，并获得该信号量。  
信号量可以**同时允许任意数量**的锁持有者，而自旋锁在一个时刻**最多允许一个**任务  
持有它。信号量持有者的数量可以在声明信号量时指定，如果指定为1，则称为**二值信号量**  
**或互斥信号量**。获取一个信号量就对信号量计数减一，释放信号量就对计数加一。  
如果在该信号量上的等待队列不为空，那么释放信号量时，处于队列中等待的任务在被**唤醒**  
的同时会获得该信号量。  
信号量的创建，信号量的实现与体系架构相关，定义在<asm/semaphore.h>中。  
```
struct semaphore {
	spinlock_t		lock;
	unsigned int		count;
	struct list_head	wait_list;
};
```
```
静态声明信号量：
    __SEMAPHORE_INITIALIZER(name, n);
name为信号量变量名，n为指定的信号量使用者数量
    DEFINE_SEMAPHORE(name);
name为信号量变量名,使用者数量为1.
    static inline void sema_init(struct semaphore *sem, int val)；
设置计数值为val。
```    
使用信号量：  

函数down_interruptible()试图获取指定的信号量，如果获取失败，它将以TASK_INTER  
RUPTIBLE状态进入睡眠。如果进程在等待获取信号量的时候接收到了信号，那么该进程   
就会被唤醒，而函数down_interruptibla()会返回-EINTR.另外一个函数down()会让进  
程在TASK_UNINTERRUPTIBLE状态下睡眠，进程在等待信号量的时候就不再相应信号了。  
使用down_trylock()函数，可以尝试以阻塞方式来获取指定的信号量，在信号量已经被  
占用时，它立刻返回非0值；否则，它返回0，而且成功持有信号量锁。  
要释放指定的信号量，需要调用up()函数。
例子：
```
    static __SEMAPHORE_INITIALIZER(mr_sem, 1);
    //试图获取信号量
    if(down_interruptible(&mr_sem)
    {
        /* 信号被接收，信号量还未获取*/
    }
    /*临界区  */
    //释放指定的信号量
    up(&mr_sem);
    
```

## 读-写信号量
读-写信号量在内核中是由rw_semaphore结构表示，定义在<linux/rwsem.h>中。  
```
/*
 * the rw-semaphore definition
 * - if activity is 0 then there are no active readers or writers
 * - if activity is +ve then that is the number of active readers
 * - if activity is -1 then there is one active writer
 * - if wait_list is not empty, then there are processes waiting for the semaphore
 */
struct rw_semaphore {
	__s32			activity;
	spinlock_t		wait_lock;
	struct list_head	wait_list;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
};
```
静态声明读-写信号量：  

    static DECLARE_RWSEM(name);
动态创建读-写信号量：  

    init_rwsem(struct rw_semaphore *sem);
所有的读写信号量都是互斥信号量(也就是说，它们的引用计数等于1)。只要没有写者，并  
发持有读锁的读者数量不限。相反，只有唯一的写者(在没有读者时)可以获得写锁。  
所有读写锁的睡眠都不会被信号打断，所以它只有一个版本的down()操作。  

例如：
```
    static DECLARE_RWSEM(mr_rwsem);
    //试图获取信号量用于读
    down_read(&mr_rwsem);
    /*临界区*/
    up_read(&mr_rwsem);
    ....
    //试图获取信号量用于写
    down_write(&mr_rwsem);
    //临界区
    up_write(&mr_rwsem);
    
```

**自旋锁与信号量：**  
在中断上下文中只能使用自旋锁，而在任务睡眠时只能使用信号量。


## 完成变量
如果在内核中一个任务需要发出信号通知另一个任务发生了某个特定事件，利用完成变  
量(completion variable)是使两个任务得以同步的简单方法。  
完成变量结构completion定义在 <linux/completion.h>中。
- DECLARE_COMPLETION(mr_comp) : 静态创建并初始化完成变量
- init_completion(struct completion*) : 初始化指定的动态创建的完成变量
- wait_for_completion(struct completion*) : 等待指定的完成变量接收信号
- complete(struct completion*) : 发信号唤醒任何等待任务




# 11_定时器和时间管理  

时间管理在内核中占有非常重要的地位。相对于事件驱动而言，内核中有大量的函数都是基于时间驱动的。  
## 内核中的时间概念  
硬件为内核提供了一个系统定时器用以计算流逝的时间，该时钟在内核中可看成是一个电子时间资源，比如数字时钟或处理器频率等。系统定时器以某种频率自行触发时钟中断，该频率可以通过编程预定，称作节拍率。内核就是依靠这种已知的时钟中断间隔来计算墙上时间和系统运行时间。  
利用时钟中断执行的工作：  
- 更新系统运行时间
- 更新实际时间
- 在smp系统上，均衡调度程序中各处理器上的运行队列
- 检查当前进程是否用尽了时间片，判断是否调度
- 运行超时的动态定时器
- 更新资源消耗和处理器时间的统计值  

## 节拍率：HZ
系统定时器频率是通过**静态预处理**定义的，也就是HZ，在系统启动时按照HZ值对硬件进行设置。体系结构不同，HZ的值也不同。大多数体系结构HZ值都是可调的。
高HZ值的优势：  
- 内核定时器能够以更高的频率和更高的准确度运行
- 依赖定时值执行的系统调用，比如poll()和select()，能够以更高的精度运行
- 对诸如资源消耗和系统运行时间等的测量会有更精细的解析度
- 提高进程抢占的准确度

高HZ值也会带来副作用，节拍越高，意味着时钟中断频率越高，也就意味着系统负担越重。  


## jiffies
全局变量jiffies用来记录自系统启动以来产生的节拍数。启动时，内核将该变量初始化为0，此后，每次时钟中断处理程序都会增加该变量的值，因为一秒内时钟中断的次数等于Hz，所以jiffies一秒内增加的值就是Hz。系统运行时间以秒为单位，就等于jiffies/Hz。  

jiffies是unsigned long 类型，定义在<linux/jiffies.h>，在32为机上最大为2^32-1，所以jiffies溢出时会回绕，即jiffies又为0.所以使用jiffies时有时需要判断回绕是否发生。  
```
    #include <linux/jiffies.h> 
    int time_after(unsigned long a, unsigned long b); 
    int time_before(unsigned long a, unsigned long b); 
    int time_after_eq(unsigned long a, unsigned long b); 
    int time_before_eq(unsigned long a, unsigned long b); 
```    
第一个当 a, 作为一个 jiffies 的快照, 代表 b 之后的一个时间时, 取值为真,第二个当 时间 a 在时间 b 之前时取值为真,   以及最后2个比较"之后或相同"和"之前或相同". 这个代码工作通过转换这个值为 signed long, 减它们, 并且比较结果.  
内核输出 4 个帮助函数来转换以jiffies 表达的时间值：  
```
    #include <linux/time.h>  
    unsigned long timespec_to_jiffies(struct timespec *value); 
    void jiffies_to_timespec(unsigned long jiffies, struct timespec *value); 
    unsigned long timeval_to_jiffies(struct timeval *value); 
    void jiffies_to_timeval(unsigned long jiffies, struct timeval *value);
struct timespec {
        long       ts_sec;
        long       ts_nsec;
};
struct timeval {
	__kernel_time_t		tv_sec;		/* seconds */
	__kernel_suseconds_t	tv_usec;	/* microseconds */
};
```
存取这个 64-位 jiffy 计数值：  
    u64 get_jiffies_64(void);   


## 时钟中断处理程序
时钟中断处理程序可以划分为两个部分：体系结构相关部分和体系结构无关部分。  
与体系结构相关的例程作为系统定时器的中断处理程序而注册到内核中，以便在产生中断时，它能够相应地运行。虽然处理程序的具体工作依赖于特定的体系结构，但是绝大多数处理程序最低限度也都要执行如下工作：  
- 获得xtime_lock锁，以便对访问jiffies_64和墙上时间xtime进行保护
- 获取时应答或重新设置系统时钟
- 周期性地使用墙上时间更新实时时钟
- 调用体系结构无关的时钟例程: tick_periodic()。

中断服务程序主要通过调用与体系结构无关的例程，tick_periodic()执行更多的工作：  
- 给jiffies_64变量增加1
- 更新资源消耗的统计值，比如当前进程所消耗的系统时间和用户时间
- 执行已经到期的启动定时器
- 执行sheduler_tick()函数
- 更新墙上时间，改时间存放在xtime变量中
- 计算平均负载值



墙上时间：  
墙上时间，在系统启动过程中根据实时钟（RTC）芯片保存数据进行初始化，在系统运行期间由系统时钟维护并在合适的时刻和RTC芯片进行同步。墙上时间存储于系统核心变量xtime中，该变量记录了现实世界中的年月日格式的时间，以便内核对某些对象和事件作时间标记，如记录文件的创建时间、修改时间、上次访问时间，或者供用户进程通过系统调用来使用。  
内核中使用struct timespec类型的变量xtime来记录墙上时间，该变量在文件src/kernel/time/timekeeping.c中的第160行声明如下：  

    static struct timespec xtime __attribute__ ((aligned (16)));
该结构用来表示当前时刻距UNIX时间基准1970/01/01/00:00:00的相对时间。其中成员变量tv_sec用来记录距标准时间1970/01/01/00:00:00的秒数，成员变量tv_nsec用来记录不足一秒的微秒值，其取值范围为0～999 999。 变量xtime的初值在系统初始化过程由函数time_init()进行设置，该函数通过读取系统实时钟
芯片RTC的值来为变量xtime赋初值；该变量的值在系统运行过程中由系统时钟中断处理程序负责
在每次时钟中断时进行更新。该变量的初始化语句如下:

    xtime.tv_sec = get_cmos_time();
    xtime.tv_nsec = (INITIAL_JIFFIES % HZ) * (NSEC_PER_SEC / HZ); 

获知当前时间

有一个内核函数转变一个墙上时钟时间到一个 jiffies 值：
    #include <linux/time.h>  
    unsigned long mktime (unsigned int year, unsigned int mon, 
     unsigned int day, unsigned int hour, 
     unsigned int min, unsigned int sec);   
     
在内核空间中处理绝对时间：
     #include <linux/time.h> 
     void do_gettimeofday(struct timeval *tv); 
当前时间也可用( 尽管使用 jiffy 的粒度 )来自 xtime 变量：
    #include <linux/time.h> 
    struct timespec current_kernel_time(void); 

延后执行：

1. 忙等待
    j1 =jiffies + n;
    while (time_before(jiffies, j1)) 
        cpu_relax(); 
2. 让出CPU
     j1 =jiffies + n;
    while (time_before(jiffies, j1)) 
        schedule(); 
3. 超时
    #include <linux/wait.h> 
    long wait_event_timeout(wait_queue_head_t q, condition, long timeout); 
    long wait_event_interruptible_timeout(wait_queue_head_t q, condition, long 
    timeout); 

    #include <linux/sched.h> 
    signed long schedule_timeout(signed long timeout);
schedule_timeout 请求调用者首先设置当前的进程状态, 
因此一个典型调用看来如此:  
    set_current_state(TASK_INTERRUPTIBLE); 
    schedule_timeout (delay);
4. 短延时
    #include <linux/delay.h> 
    void ndelay(unsigned long nsecs); 
    void udelay(unsigned long usecs); 
    void mdelay(unsigned long msecs); 
有另一个方法获得毫秒(和更长)延时而不用涉及到忙等待：
    void msleep(unsigned int millisecs); 
    unsigned long msleep_interruptible(unsigned int millisecs); 
    void ssleep(unsigned int seconds) 

## 定时器
定时器是管理内核时间的基础。定时器只需初始化工作，设置一个超时时间，指定超时发生后执行的函数，然后激活定时器就可以啦。指定的函数将在定时器到期时自动执行。**注意定时器并不周期运行，它在超时后会自行销毁；定时器不断的创建和销毁，而且它的运行次数也不受限制**。
为能够被执行, 多个动作需要进程上下文. 当你在进程上下文之外(即, 在中断上下文),你必须遵守下列规则(中断上下文):   
-  没有允许存取用户空间. 因为没有进程上下文, 没有和任何特定进程相关联的到用户空间的途径. 
-  这个 current 指针在原子态没有意义, 并且不能使用因为相关的代码没有和已被中断的进程的联系. 
-  不能进行睡眠或者调度. 原子代码不能调用 schedule 或者某种 wait_event, 也不能调用任何其他可能睡眠的函数.  
  
例如, 调用 kmalloc(..., GFP_KERNEL) 是违犯规则的. 旗标也必须不能使用因为它们可能睡眠.  
```
struct timer_list {
	/*
	 * All fields that change during normal runtime grouped to the
	 * same cacheline
	 */
	struct list_head entry;//定时器链表入口
	unsigned long expires; //以jiffies为单位的定时值
	struct tvec_base *base; //定时器内部值，用户不要使用

	void (*function)(unsigned long); //定时器处理函数
	unsigned long data; //定时器处理函数的参数

	int slack;

#ifdef CONFIG_TIMER_STATS
	int start_pid;
	void *start_site;
	char start_comm[16];
#endif
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
```
使用定时器：  
创建定时器时需定义它：  
    struct timer_list my_timer;  
初始化定时器数据结构的内部值，初始化必须在使用其他定时器管理函数前完成：  
    init_timer(&my_tiemr);  
填充结构中需要的值：  
    my_timer.expires = jiffies + delay; //设置超时时的节拍数  
    my_timer.data = 0; //定时器处理函数出入值  
    my_timer.function = my_function; //定时器超时调用函数  
激活定时器：  
    add_timer(&my_timer);  
更改定时器超时时间：  
    mod_timer(&my_timer,jiffies + new_delay);  
    如果定时未被激活，mod_timer()会激活它  
如果在定时器超时前停止定时器，使用：  
    del_timer(&my_timer);  
当删除定时器时，在多处理器机器上定时器处理函数可能已经运行，所以删除定时器时要等到其他处理器上运行的定时器处理程序都退出，这时就要使用del_timer_sync()函数执行  
删除工作： 
    del_timer_sync(&my_timer);

请注意定时值的重要性，当前节拍计数等于或大于指定的超时时，内核就开始执行定时器处理函数。虽然内核可以保证不会超时时间前运行定时器处理函数，但是有可能延误定时器的执行。**一般来说，定时器都在超时后马上就会执行，但是也有可能推迟到下一次时钟节拍时才能运行，所以不能用定时器来实现任何硬实时任务**。  

## 实现定时器  
内核在时钟中断发生后执行定时器，定时器作为软中断在下半部上下文中执行。具体来说，时钟中断处理程序会执行update_process_times()函数，该函数即调用run_local_timers()函数：  
```
/*
 * Called by the local, per-CPU timer interrupt on SMP.
 */
void run_local_timers(void)
{
	hrtimer_run_queues();
	raise_softirq(TIMER_SOFTIRQ); //执行定时器软中断
}
```
raise_softirq(TIMER_SOFTIRQ)函数处理软中断TIMER_SOFTIRQ，从而在当前处理器上运行所有的(如果有个话)超时定时器。  
虽然所有定时器都以链表形式存放在一起，但是让内核经常为了寻找超时定时器而遍历整个链表是不明智的。同样，将链表以超时时间进行排序也是很不明智的做法，因为这样一来在链表中插入和删除定时器都会很费时。**为了提高搜索效率，内核将定时器按它们的超时时间划分为五组，当定时器超时时间接近时，定时器将随组一起下移**。采用分组定时器的方法在执行软中断的多数情况下，确保内核尽可能减少搜索超时定时器所带来的负担。因此定时器管理代码是非常高效的。  

# 12_内存管理

## 基本概念
每个Linux进程都有一个地址空间，逻辑上有三段：代码段、数据段、堆栈段。  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/WEBRESOURCE6cc73d055ccb2dfc82a60432341e184a/14729)  
数据段包括初始化的数据和未初始化的BSS。BSS并不存放在可执行文件中，可执行文件仅16KB(代码+初始化数据)，加上一个很短的头部来告诉系统在初始化数据后另外再分配4KB，同时在程序启动之前把他们初始化为0.这个技巧避免了在可执行文件中存储4KB的0.**为了避免分配一个全是0的物理页框，在初始化时，Linux就分配了一个静态零页面，即一个全0的写保护页面**。当加载程序的时候，未初始化数据区域被设置为指向该零页面。当一个进程真要写这个区域的时候，写时复制的机制开始起作用，一个实际的页框被分配给了该进程。  
Linux允许数据段随着内存的分配和回收而增长和缩减，通过这种机制来解决动态分配的问题。有一个系统调用brk，允许程序设置其数据段的大小。那么，为了分配更多的内存，一个程序可以增加数据段的大小。C库函数malloc就是大量使用这个系统调用。  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/WEBRESOURCEf6379fe86e1b9e234227d83ab346d755/14848)  
注意：get_free_page是在内核中分配内存，不同于malloc在用户空间中分配，malloc利用堆动态分配，实际上是调用brk()系统调用，该调用的作用是扩大或缩小进程堆空间（它会修改进程的brk域）。如果现有的内存区域不够容纳堆空间，则会以页面大小的倍数为单位，扩张或收缩对应的内存区域，但brk值并非以页面大小为倍数修改，而是按实际请求修改。因此Malloc在用户空间分配内存可以以字节为单位分配,但内核在内部仍然会是以页为单位分配的。    

栈段，在大多数机器里，它从虚拟地址空间的顶部或者附近开始，向下生长。在32位系统上，栈的起始地址0XC0000000，这是用户态下对进程可见的3GB虚拟地址限制。**如果栈生长到了栈段的底部一下，就会产生一个硬件错误同时操作系统把栈段的底部降低一个页面**，程序并不现实地控制栈段的大小。当一个程序启动的时候，它的栈并不是空的。相反，它包含所有的环境变量以及为了调用它而向shell输入的命令行。这样，一个程序就可以发现它的参数了。  
大多数Linux系统都支持**共享代码段**。多个进程通过映射到相同的代码段页面上。  
处理动态分配更多的内存，Linux中的进程可以通过**内存映射文件**来访问文件数据。这个特性使我们可以把一个文件映射到进程空间的一部分而该文件就可以像位于内存中的字节数组一样被读写。共享库的访问就是通过这种机制映射进来后进行的。  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/WEBRESOURCEf07e3d3ecc4d44729bac0a5ad3251b75/14806)  

## Linux中内存管理的实现  
在32位机器上，Linux内核内存通常驻留在低端物理内存中，但是被映射到每个进程虚拟地址空间顶部的1GB中。在目前的64位X86机器上，最多只有48位用于寻址，意味着寻址存储器的大小理论极限值为256TB。  
### 物理内存管

在许多系统中由于异构硬件限制，并不是所有的物理内存都能被相同地对待，尤其是对于I/O和虚拟内存。Linux区分以下内存区域(zone):  
- ZONE_DMA和ZONE_DMA32：可用于DMA操作的页
- ZONE_NORMAL:正常的，规则映射的页
- ZONE_HIGHMEM: 高内存地址的页，并不永久性映射

**内存区域的确切边界和布局是硬件体系结构相关的**。在X86硬件上，一些设备只能在最低的16MB地址空间进行DMA操作，因此ZONE_DMA就在0~16MB的范围，ZONE_HIGHMEM用于高于896M的任何地址，ZONE_NORMAL是介于其中的任何地址。  

![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/WEBRESOURCE1e1f6d6f4dc8fcae96004e67e5b2704e/14850)  

Linux的内存由三部分组成。前两部分是**内核和内存映射，被固定**在内存中(页面从来不换出)。内存的其他部分被划分成页框，每个页框都可以包含一个代码、数据或栈页面、一个页表页面或者在空闲列表中。  
内核维护内存的一个映射，该映射包含了所有系统物理内存使用情况的信息，比如区域、空闲页框等。信息组成如下图：  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/WEBRESOURCE1fc9274b3eab4dcfde9875ea908ee630/14878)  
首先，Linux维护一个**页描述符**数组，称为mem_map，其中页描述符是page类型的，而且系统当中的每个物理页框都有一个页描述符。每个页描述符都有个指针，在页面非空闲时指向它所属的地址空间，另有一对指针可以使得它根其他描述符形成双向链表，来记录所有的空闲页框和一些其他的域。因为物理内存被分成区域，所有Linux为每个区域维护一个**区域描述符**。区域描述符包含了每个区域中内存利用情况的信息，例如活动和非活动页的数目，页面置换算法所使用的高低水印位，还有许多其他相关信息等。  
此外，区域描述符包含一个空闲区数组。该数组中的第i个元素标记了2^i个空闲也的第一个块的第一个页描述符。既然可能有多块2^i个空闲页，Linux使用页描述符的指针对把这些页面连接起来。这个信息在Linux的内存分配操作中使用。  
为了使分页机制在32位和64位体系结构下都能高效工作，Linux采用了一个**四级分页策略**。每个虚拟地址划分为五个域，目录域是页目录索引，**每个进程都有一个私有的页目录**。找到的值是指向其中一个下一级目录的指针，该目录页可以从虚拟地址进行索引。中级页目录表中的表项指向最终的页表，它是由虚拟地址的页表域索引的。页表的表项指向所需要的页面。**在需要的时候可以使用三级分页，此时把上级目录域大小设置为0就可以了。  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/WEBRESOURCEb5c9b8d3b79b5b46d89a6df68c67d0aa/14940)  
物理内存可以用于多种目的。内核自身是完全“硬连线”的，它的任何一部分都不会换出。内存的其余部分可作为用户页面、分页缓存和其他目的。分页缓存保存最近已读的或者由于未来有可能使用而预读的文件块，或则需要写回磁盘的文件块页面。分页缓存并不是一个独立的缓存，而是那些不需要的或者等待换出的用户页面集合。如果分页缓存当中的一个页面在被换出内存前复用，它可以被快速收回。  
此外，Linux支持动态加载模块，最常见的是设备驱动。它们可以是任意大小的并且必须被分配一个连续的内核内存。这些需求的一个直接结果是，Linux用这样一种方式来管理物理内存使得他可以随意分配任意大小的内存片。它使用的算法就是**伙伴算法**。  

### 内存分配机制  
分配物理内存页框的主要机制是**页面分配器**，它使用了**伙伴算法**。

#### 伙伴系统算法
在实际应用中，经常需要分配一组连续的页框，而频繁地申请和释放不同大小的连续页框，必然导致在已分配页框的内存块中分散了许多小块的 空闲页框。这样，即使这些页框是空闲的，其他需要分配连续页框的应用也很难得到满足。为了避免出现这种情况，Linux内核中引入了伙伴系统算法(buddy system)。**把所有的空闲页框分组为11个块链表，每个块链表分别包含大小为1，2，4，8，16，32，64，128，256，512和1024个连续页框的页框块。最大可以申请1024个连 续页框，对应4MB大小的连续内存。每个页框块的第一个页框的物理地址是该块大小的整数倍**。  
假设要申请一个256个页框的块，先从256个页框的链表中查找空闲块，如果没有，就去512个 页框的链表中找，找到了则将页框块分为2个256个 页框的块，一个分配给应用，另外一个移到256个页框的链表中。如果512个页框的链表中仍没有空闲块，继续向1024个页 框的链表查找，如果仍然没有，则返回错误。  
**页框块在释放时，会主动将两个连续的页框块合并为一个较大的页框块**。  

#### slab分配器
slab分配器源于 Solaris 2.4 的 分配算法，**工作于物理内存页框分配器之上，管理特定大小对象的缓存，进行快速而高效的内存分配**。  
slab分配器为每种使用的内核对象建立单独的缓冲区。Linux 内核已经采用了伙伴系统管理物理内存页框，因此 slab分配器直接工作于伙伴系 统之上。每种缓冲区由多个 slab 组成，每个 slab就是一组连续的物理内存页框，被划分成了固定数目的对象。根据对象大小的不同，缺省情况下一个 slab 最多可以由 1024个页框构成。出于对齐 等其它方面的要求，slab 中分配给对象的内存可能大于用户要求的对象实际大小，这会造成一定的 内存浪费。  
例如：当内核需要分配一个新的进程描述符的时候，它在task结构的对象缓存中寻找，首先试图找一个部分满的slab并且在那里分配一个新的task_struct对象。如果没有这样的slab可用，就在空闲slab列表中查找。最后，如果有必要，它会分配一个新的slab，把新的task结构放在那里，同时把该slab连接到task结构对象缓存中。**在内核地址空间分配连续的内存区域的kmalloc内核服务，实际上就是建立在slab和对象缓存接口之上的**。  

#### vmalloc
第三个内存分配器vmalloc也是可用的，并且用于那些仅仅要求**虚拟地址连续**的请求，而**物理地址并不一定连续**。  
### 虚拟地址空间表示  
虚拟地址空间被分割成同构连续页面对齐的区域。也就是说，每个区域由一系列连续的具有相同**保护和分页属性**的页面组成。  
在内核中，每个区都是vm_area_struct项来描述的。一个进程的所有vm_area_struct用一个链表连接在一起，并且按照虚拟地址排序以便可以找到所有的页面。当这个链表太长时(多于32项)，就创建一个**树来加速搜索**。vm_area_struct项列出了该区的属性，这些属性包括：保护模式(如，只读或可读可写)、是否固定在内存中(不可换出)、朝向哪个方向生长(数据段向上生长，栈段向下生长)。  

vm_area_struct也记录该区是私有的还是跟一个或多个进程共享的。fork之后，Linux为子进程复制一份区链表，但是让父进程指向相同的页表。**区被标记为可读可写，但是页面自己却被标记为只读**。如果任何一个进程试图写页面，就会产生一个保护故障，此时内核发现该内存区逻辑上是可写的，但是页面却不是可写的，因此它把该页面的一个副本给当前进程同时标记为可读可写。这个机制就是写时复制。  
vm_area_struct也记录该区**是否在磁盘上有备份存储**，如果有，在什么地方。  
**一个顶层内存描述符mm_struct收集属于一个地址空间的所有虚拟内存区相关的信息**，还有关于不同段(代码、数据、栈)和用户共享地址空间的信息等。**一个地址空间的所有vm_area_struct元素可以通过内存描述符用两种方式访问**。首先，它们是**按照虚拟地址顺序组织在链表中**的。这种方式的有用指出是：当所有的虚拟地址区需要被访问时，或者当内核查找分配一个指定大小的虚拟内存区域时。此外，vm_area_struct项目被组织成**二叉红黑树**。这种方法用于访问一个指定的虚拟内存地址。为了能够用这两种方法访问进程地址空间的元素，Linux为每个进程使用了更多的状态，但是却允许不同的内核操作来使用这些访问方法，对进程更加高效。  

## Linux中的分页  
Linux分页基本思想是简单的：为了运行，一个进程并不需要完全在内存中。**实际上只需要用户结构和页表**。  
分页是一部分由内核实现而一部分由一个新的进程--**页面守护进程**实现的。页面守护进程是进程2。跟所有守护进程一样，页面守护进程周期性地运行，一旦唤醒，它主动查找是否有工作要干；如果它发现空闲页面数量太少，就开始释放更多的页面。  
Linux是一个请求换页系统，没有预分页和工作集的概念。**代码段和映射文件被换页到它们各自在磁盘中的文件。所有其他的都被换页到分页分区或者一个固定长度的分页文件，叫做交换区**。换页到交换区的优点：首先，原始设备访问的方式要比换页到一个文件更加高效；其次，物理写可以任意大小，并不仅仅是文件块大小；第三，一个页总是被连续写到磁盘，用一个分页文件，就并不一定这样。  
当页面换出时，**页表被及时更新以反映页面已经不在内存了同时磁盘位置被写入到页表项**。  
### 页面置换算法
页面置换算法是这样工作的，Linux试图保留一些空闲页面，这样可以在需要的时候分配它们。当然，这个页面池必须不断地加以补充。PFRA(页框回收算法)算法展示了它是如何发生的。  
Linux区分四中不同的页面：**不可回收的、可交换的、可同步的、可丢弃的**。不可回收页面包括保留或者锁定页面、内核态栈等，不会被换出。可交换页必须在回收之前写回到交换区或者分页磁盘分区。可同步的页面如果被标记为dirty就必须要写回到磁盘。最后，可丢弃的页面可以被立即收回。  
在启动的时候，init开启一个页面守护进程kswapd(每个内存节点都有一个)，并且配置它们能周期性运行。每次kswapd被唤醒，他通过比较每个内存区域的高低水位和当前内存的使用来检查是否有足够的空闲页面可用。如果有足够的空闲页面，它就继续睡眠。当然它也可以在需要更多页面时被提前唤醒。如果任何内存区域的可用空间低于一个阈值，kswapd初始化页框回收算法。在每次运行过程中，仅有一个确定数目的页面被回收，典型值是最大值32.这个值是受限的，以控制I/O压力(由PFRA操作导致的磁盘写的次数)。回收页面的数量和扫描页面的总数量都是可配置的参数。  
#### PFRA
每次PFRA执行时，它首先收回容易的页面，然后处理更难的。
1. 可丢弃页面和未被引用的页面都可以把他们添加到区域的空闲链表中从而立即回收。
2. 接着它查找**有备份存储同时近期未使用**的页面，使用一个类似时钟的算法。  
3. 再后来就是用户使用不多的共享页面。共享页面带来的挑战是，如果一个页面被回收，那么所有共享了该页面的所有地址空间的页表都要同步更新。Linux维护高效的类树数据结构来方便地找到一个共享页面的所有使用者。
4. 接下来是普通用户页面，如果被选中换出，它们必须被调度写入交换区。系统的swappiness，即有备份存储的页面和在FPRA中被换出的页面的比率，是该算法的一个可调参数。 
5. 最后，如果有一个页是无效的、不在内存、共享、锁定在内存或拥有DMA，那么它被跳过。  

![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/WEBRESOURCEfaf1649f5bff347593ba42940ff82230/15237)  
PFRA用一个类似时钟的算法来选择旧页换出。这个算法的核心是一个循环，它扫描每个区域的活动和非活动列表，试图按照不同的急迫程度回收不同类型的页面。紧迫性数值作为一个参数传递给该过程，说明花费多大的代价来回收一些页面。  
在PFRA期间，页面按照上图描述符的方式在活动和非活动列表之间来回移动。为了维护一些**启发方法**并且尽量找出没有被引用的和近期不可能被使用的页面，PFRA为每个页面维护两个标记：活动/非活动和是否被引用。这两个标记组成四种状态，在对一个页面集合的第一遍扫描中，PFRA首先清除它们的引用位。如果在第二次运行期间确定它被引用，则把它提升到另一个状态，这样就不太可能回收它了。否则，将该页面移动到另一个更可能回收的状态。  
处于非活动列表上的页面，自从上次检查未被引用过，故而是移出的最佳候选。然后，如果有需要，处于其他状态的页面也可能被回收。  
PFRA维护一些页面，尽管这些页面可能已经被引用但在非活动列表中，其原因是为了避免如下情形。考虑到一个进程周期性访问不同页面，比如一个小时，从最后一次循环开始被访问的页面会设置其引用标志为。然而，接下来的一个小时都不在使用它，没有理由不考虑把它作为一个回收的候选。  


## 页
内核把物理页作为内存管理的基本单位。MMU以页大小为单位来管理系统中的页表。
内核用struct page结构表示系统中的每个物理页，该结构位于<linux/mm.h>中：  
```
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	atomic_t _count;		/* Usage count, see below. */
	union {
		atomic_t _mapcount;	/* Count of ptes mapped in mms,
					 * to show when page is mapped
					 * & limit reverse map searches.
					 */
		struct {		/* SLUB */
			u16 inuse;
			u16 objects;
		};
	};
	union {
	    struct {
		unsigned long private;		/* Mapping-private opaque data:
					 	 * usually used for buffer_heads
						 * if PagePrivate set; used for
						 * swp_entry_t if PageSwapCache;
						 * indicates order in the buddy
						 * system if PG_buddy is set.
						 */
		struct address_space *mapping;	/* If low bit clear, points to
						 * inode address_space, or NULL.
						 * If page mapped as anonymous
						 * memory, low bit is set, and
						 * it points to anon_vma object:
						 * see PAGE_MAPPING_ANON below.
						 */
	    };
#if USE_SPLIT_PTLOCKS
	    spinlock_t ptl;
#endif
	    struct kmem_cache *slab;	/* SLUB: Pointer to slab */
	    struct page *first_page;	/* Compound tail pages */
	};
	union {
		pgoff_t index;		/* Our offset within mapping. */
		void *freelist;		/* SLUB: freelist req. slab lock */
	};
	struct list_head lru;		/* Pageout list, eg. active_list
					 * protected by zone->lru_lock !
					 */
	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */
#ifdef CONFIG_WANT_PAGE_DEBUG_FLAGS
	unsigned long debug_flags;	/* Use atomic bitops on this */
#endif

#ifdef CONFIG_KMEMCHECK
	/*
	 * kmemcheck wants to track the status of each byte in a page; this
	 * is a pointer to such a status block. NULL if not tracked.
	 */
	void *shadow;
#endif
};
```
page结构与物理页相关，并非与虚拟页相关！每个物理页都分配一个page结构体。

## 区
Linux使用了三种区：  
- ZONE_DMA ： 这个区包含的页能用来执行DMA操作
- ZONE_NORMAL : 这个区包含的都是能正常映射的页
- ZONE_HIGHMEM : 这个区包含的“高端内存"，其中的页并不能永久地映射到内核地址空间  
每个区都用struct zone表示，定义在<linux/mmzone.h>中。  
Linux把系统的页划分为区，形成不同的内存池，这样就可以根据用途进行分配了。区只是内核为了管理页而采用的一种逻辑上的分组。  
## 获取页

所有以页为单位分配内存，定义于<linux/gfp.h>中：
    struct page* alloc_pages(unsigned int gfp_mask, unsigned int order);
该函数分配2^order个**连续**的物理页，并返回一个指针，该指针指向第一个页的page结构体
；如果出错，返回NULL。  
    void *page_address(struct page *page);  
该函数返回一个指针，指向给定的物理页当前所在的逻辑地址。  
    unsigned long_get_free_pages(unsigned int gfp_mask, unsigned int order);  
这个函数与alloc_pages()作用相同，不过它直接返回所请求的第一个页的逻辑地址。  
    如果只需要一页，可以使用下面封装好的函数：  
    struct page *alloc_page(unsigned int gfp_mask);  
    unsigned long __get_free_page(unsigned int gfp_mask);  
返回的页的内容全是0，：  
    unsigned long get_zeroed_page(unsigned int gfp_mask);  
释放页：  
    void __free_pages(struct page *page,unsigned int order);  
    void free_pages(unsigned long addr,unsigned int order);  
    void free_page(unsigned long addr);  

## kmalloc
kmalloc可以获取以**字节为单位**的一块内核内存，kmalloc()函数是建立在slab内存分配器上的。kmalloc()在<linux/slab.h>中：  
void *kmalloc(size_t size, int flags);  
这个函数返回一个纸箱内存块的指针，其内存块至少有size个字节。所分配的内存在物理上是连续的。出错时返回NULL。使用kmalloc()函数时一定要检查返回值！  

gfp_mask标志：  
标志分为三类：行为修饰符、区修饰符、类型。行为修饰符表明内核应当如何分配内存；区修饰符表明使用哪个区的内存。类型标志组合了行为修饰符与区修饰，使用时只需指明类型修饰符即可。  
类型标志：  
- GFP_ATOMIC:分配是高优先级的，且不会睡眠。这个标志用在中断处理程序、下半部、持有自旋锁，以及其他不能睡眠的地方
- GPF_NOIO:这种分配可以阻塞，但是不会启动磁盘I/O。这个标志在不能引发更多磁盘I/O时能够阻塞I/O代码，这可能导致令人不愉快的递归
- GFP_NOFS:这种分配在必要时能阻塞，也可能启动磁盘I/O，但是不会启动文件系统操作。这个标志在你不能再启动另一个文件系统的操作时用在文件系统部分的代码中
- GFP_KERNEL:这是一种常规分配方式，可能会阻塞。这个标志在睡眠安全时用在进程上下文代码中。为了获得调用者所需的内存，内核会尽力而为。这个标志应道是首选标志。GFP_KERNEL, 意思是这个分配((内部最终通过调用 __get_free_pages 来进行, 它是 GFP_ 前缀的来源)代表运行在内核空间的进程而进行的
- GFP_USER:这是一种常规分配方式，可能会阻塞。这个标志用于用户空间进程分配内存
- GFP_HIGHUSER:这是从ZONE_HIGHMEM进行分配，可能会阻塞。这个标志用于为用户空间进程分配内存
- GFP_DMA:这是从ZONE_DMA进行分配。需要获取能够获取DMA使用的内存的设备驱动程序使用这个标志，通常与以上的某个标志组合在一起使用
    
kmalloc()的另一个端就是kfree(),声明于<linux/slab.h>中：  
void kfree(const void *ptr)  

## vmalloc()
vmalloc()函数的工作方式类似于kmalloc(),只不过前者分配的内存虚拟地址是连续，而物理地址则无需连续。vmalloc()函数只确保页在虚拟地址空间是连续的，它通过分配非连续的物理内存块，在“修正”页表，把内存映射到逻辑地址空间的连续区域中，就能做到这点。很多内核代码都用kmalloc()来获得内存，而不是vmalloc()，这主要是出于性能的考虑。vmalloc()函数为了把物理上不连续的页转换为虚拟地址空间上连续的页，必须专门建立页表项。槽糕的是，通过vmalloc()获得的页必须一个一个进行映射(因为物理上是不连续的)，这就会导致比直接内存映射大得多的TLB抖动。  
    vmalloc()函数在<linux/vmalloc.h>中声明：
    void *vmalloc(unsigned long size);
该函数返回一个指针，指向逻辑上连续的一块内存去，其中大小至少为size.发生错误是返
回NULL。
 void vfree(void *addr)；释放内存
 
## slab层

slab分配器试图在几个基本原则之间寻求一种平衡：  
- 频繁使用的数据结构也会频繁的分配和释放，因此应当缓存它们。
- 频繁分配和回收必然会导致内存碎片(难以找到大块连续的可用内存)，为了避免这种现象，空闲链表的缓存会连续地存放。因为已释放的数据结构又会放回空闲链表，因此不会导致碎片。
- 回收的对象可用立即投入下一次分配
- 如果分配器知道对象的大小、页大小和总的高速缓存的大小这样的概念，它会做出更明智的决策。
- 如果让部分缓存专属于单个处理器(对系统上的每个处理器独立而唯一)，那么，分配和释放就可以在不加SMP锁的情况下进行。
- 如果分配器是与NUMA相关的，它就可以从相同的内存节点为请求者进行分配
- 对存放的对象进行着色，以防止多个对象映射到相同的高速缓存行。  

### slab层的设计
**slab层把不同的对象划分为所谓高速缓存组，其中每个高速缓存组都存放不同类型的对象**。每种对象类型对应一个告诉缓存。kmalloc()接口建立在slab层之上，使用了一组通用高速缓存。  
这些高速缓存又被划分为slab,slab由一个或多个物理上连续的页组成。一般情况下，slab就仅仅由一个页组成，每个高速缓存可以由多个slab组成。每个slab都包含一些对象成员，这里的对象指的是被缓存的数据结构。每个slab处于三种状态之一：**满、部分满或空**。当内核需要一个新的对象时，先从部分满的slab进行分配，如果没有部分满的slab，就会从空的slab中进行分配；如果没有空的slab，就要创建一个slab了。这种策略能够减少碎片。  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/1B97761ED22842718030B82088164E64/15424)
每个高速缓存都使用kmem_cache结构表示。这个结构包含三个链表：slabs_full、slabs_partial和slabs_empty,均存放在kmem_list3结构内。这些链表包含高速缓存中的所有slab。  
```
struct slab {
	struct list_head list; //满、部分满、空链表
	unsigned long colouroff; //slab着色的偏移量
	void *s_mem;		/* including colour offset */ //在slab中的第一个对象
	unsigned int inuse;	/* num of objs active in slab */ //slab中已分配的对象数
	kmem_bufctl_t free; //第一个空闲对象
	unsigned short nodeid;
};
```
slab描述符要么在slab之外另行分配，要么就放在slab自身开始的地方。  
slab分配器可以创建新的slab，这是通过__get_free_pages()低级内核页分配器进行的。 
在下列情况下才会调用释放函数：当可用内存变得紧缺时，系统试图释放出更多内存以供使用；或者当高速缓存显示地被撤销时。  
当你创建了一个高速缓存后，slab层所起的作用就像一个专用的分配器，可以为具体的对象类型进行分配。  
### slab分配器的接口
slab 分配器实现有一个 kmem_cache 类型的缓存; 使用一个对 kmem_cache_create的调用来创建它们:   
struct kmem_cache *
    kmem_cache_create (const char *name, size_t size, size_t align,
	    unsigned long flags, void (*ctor)(void *))  
这个函数创建一个新的可以驻留任意数目全部同样大小的内存区的缓存对象, 大小由size 参数指定. name 参数和这个缓存关联并且作为一个在追踪问题时有用的管理信息;align 是页内的第一个对象的偏移; flags 控制如何进行分配并且是下列标志的一个位掩码:   
- SLAB_NO_REAP：设置这个标志保护缓存在系统查找内存时被削减。设置这个标志通常是个坏主意；重要的是避免不必要地限制内存分配器的行动自由. 
- SLAB_HWCACHE_ALIGN：这个标志需要每个数据对象被对齐到一个缓存行;实际对齐依赖主机平台的缓存分布。这个选项可以是一个好的选择, 如果在 SMP机器上你的缓存包含频繁存取的项。但是，用来获得缓存行对齐的填充可以浪费可观的内存量. 
- SLAB_CACHE_DMA：这个标志要求每个数据对象在 DMA 内存区分配. 
- ctor是可选函数，用来初始化新分配的对象。  

一旦一个对象的缓存被创建，可以通过调用kmem_cache_alloc从它分配对象：  
    void *kmem_cache_alloc(kmem_cache *cache, int flags);   
    cache 参数是你之前已经创建的缓存; flags 是你会传递给 kmalloc 的相同。  
释放一个对象：  
    void kmem_cache_free(kmem_cache_t *cache, const void *obj);  
当驱动代码用完这个缓存, 典型地当模块被卸载, 它应当如下释放它的缓存:  
    int kmem_cache_destroy(kmem_cache_t *cache);  
    
例子：  
```
struct kmem_cache *task_struct_cachep;
task_struct_cachep = //创建高速缓存
		kmem_cache_create("task_struct", sizeof(struct task_struct),
			ARCH_MIN_TASKALIGN, SLAB_PANIC | SLAB_NOTRACK, NULL);
//利用高速缓存创建task_struct结构  
struct task_struct *tsk = kmem_cache_alloc(task_struct_cachep, GFP_KERNEL);

```
## 高端内存的映射
根据定义，**高端内存中的页不能永久地映射到内核地址空间上**。因此，通过alloc_pages()函数以__GFP_HIGHMEM标志获得的页不可能有逻辑地址。  
在X86体系结构上，高于896MB的所有物理内存的范围大都是高端内存，它并不会永久地或自动地映射到内核地址空间，尽管X86处理器能够寻址物理RAM的范围达到4GB(开启PAE可以寻址64GB)。一旦这些页被分配，就必须映射到内核的逻辑地址空间上。在X86上，高端内存中的页被映射到3GB~4GB。  
!image[](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/CADB744E2B124D5D81A7B4D322B01F5D/15493)  
### 永久映射
要映射一个给定的page结构到内核地址空间，可以使用：  
    void *kmap(struct page *page);  
这个函数在高端内存或低端内存上都能使用。如果page结构对应的是低端内存中的一，函数只会单纯返回该页的虚拟地址。如果页位于高端内存，则会建立一个永久映射，再返回地址。这个函数**可以睡眠**，因此kmap()只能使用在进程上下文中。  
因为允许永久映射的数量有限，当不再需要使用高端内存时，要解除映射：  
    void kunmap(struct page* page);  
    
### 临时映射
当必须创建一个映射而当前的上下文**不能睡眠时**，内核提供了**临时映射(原子映射)**。有一组**保留**的映射，它们可以存放创建的临时映射。  
建立一个临时映射：  
    void *kmap_atomic(struct page *page, enum km_type type);  
```
 enum km_type {
	KM_BOUNCE_READ,
	KM_SKB_SUNRPC_DATA,
	KM_SKB_DATA_SOFTIRQ,
	KM_USER0,
	KM_USER1,
	KM_BIO_SRC_IRQ,
	KM_BIO_DST_IRQ,
	KM_PTE0,
	KM_PTE1,
	KM_IRQ0,
	KM_IRQ1,
	KM_SOFTIRQ0,
	KM_SOFTIRQ1,
	KM_L1_CACHE,
	KM_L2_CACHE,
	KM_KDB,
	KM_TYPE_NR
};
```
这个函数不会阻塞，因此可以用在中断上下文和其他不能重新调度的地方。它也禁止内核抢占，这是有必要的，因为映射对每个处理器都是唯一的。  
取消映射：  
    void kunmap_atomic(void *kvaddr, enum km_type type);  
    



后备缓存：


内存池：

```
    在内核中有不少地方内存分配不允许失败. 作为一个在这些情况下确保分配的方式, 内核
开发者创建了一个已知为内存池(或者是 "mempool" )的抽象. 一个内存池真实地只是一
类后备缓存, 它尽力一直保持一个空闲内存列表给紧急时使用.

一个内存池有一个类型 mempool_t ( 在 <linux/mempool.h> 中定义); 你可以使用 
mempool_create 创建一个:
    mempool_t *mempool_create(int min_nr, mempool_alloc_t *alloc_fn,
				mempool_free_t *free_fn, void *pool_data)
min_nr 参数是内存池应当一直保留的最小数量的分配的对象. 实际的分配和释放对象由 
alloc_fn 和 free_fn 处理, 它们有这些原型:
    typedef void *(mempool_alloc_t)(int gfp_mask, void *pool_data); 
    typedef void (mempool_free_t)(void *element, void *pool_data);
给 mempool_create 最后的参数 ( pool_data ) 被传递给 alloc_fn 和 free_fn.

    如果需要, 你可编写特殊用途的函数来处理 mempool 的内存分配. 常常, 但是, 你只需
要使内核 slab 分配器为你处理这个任务. 有 2 个函数 ( mempool_alloc_slab 和 
mempool_free_slab) 来进行在内存池分配原型和 kmem_cache_alloc 和 
kmem_cache_free 之间的感应淬火. 因此, 设置内存池的代码常常看来如此:
    cache = kmem_cache_create(. . .);  
    pool = mempool_create(MY_POOL_MINIMUM,mempool_alloc_slab, mempool_free_slab, 
    cache);  
一旦已创建了内存池, 可以分配和释放对象,使用:
    void *mempool_alloc(mempool_t *pool, int gfp_mask); 
    void mempool_free(void *element, mempool_t *pool); 
一个 mempool 可被重新定大小, 使用: 
    int mempool_resize(mempool_t *pool, int new_min_nr, int gfp_mask);   
这个调用, 如果成功, 调整内存池的大小至少有 new_min_nr 个对象. 如果你不再需要一
个内存池, 返回给系统使用: 
    void mempool_destroy(mempool_t *pool);     


```
---
get_free_page 和其友：

```
如果一个模块需要分配大块的内存, 它常常最好是使用一个面向页的技术.
1.
为分配页, 下列函数可用: 
get_zeroed_page(unsigned int flags);  
    返回一个指向新页的指针并且用零填充了该页. 
__get_free_page(unsigned int flags);  
    类似于 get_zeroed_page, 但是没有清零该页. 
__get_free_pages(unsigned int flags, unsigned int order);  
    分配并返回一个指向一个内存区第一个字节的指针, 内存区可能是几个(物理上连
    续)页长但是没有清零. 
flags 参数同 kmalloc 的用法相同;order 是你在请求的或释放的页数的以 2 为底的对数，即2^order页。
释放页：
    void free_page(unsigned long addr); 
    void free_pages(unsigned long addr, unsigned long order);
    页级别分配的主要优势实际上不是速度, 而是更有效的内存使用. 按页分配不浪费内存,
而使用 kmalloc 由于分配的粒度会浪费无法预测数量的内存  

2. alloc_pages 接口
struct page 是一个描述一个内存页的内部内核结构。
Linux 页分配器的真正核心是一个称为 alloc_pages_node 的函数: 
    struct page *alloc_pages_node(int nid, unsigned int flags, 
     unsigned int order); 
这个函数页有 2 个变体(是简单的宏); 它们是你最可能用到的版本: 
    struct page *alloc_pages(unsigned int flags, unsigned int order); 
    struct page *alloc_page(unsigned int flags); 
为释放这种方式分配的页, 你应当使用下列一个: 
    void __free_page(struct page *page); 
    void __free_pages(struct page *page, unsigned int order); 
    void free_hot_page(struct page *page); 
    void free_cold_page(struct page *page);     
如果你对一个单个页的内容是否可能驻留在处理器缓存中有特殊的认识, 你应当使用 
free_hot_page (对于缓存驻留的页) 或者 free_cold_page 通知内核. 这个信息帮助内
存分配器在系统中优化它的内存使用. 


```
---
在启动时获得专用的缓冲：

```
如果你真的需要一个大的物理上连续的缓冲, 最好的方法是在启动时请求内存来分配它.
在启动时分配内存是一个"脏"技术, 因为它绕开了所有的内存管理策略通过保留一个私有的内存池。

当内核被启动, 它赢得对系统种所有可用物理内存的存取. 它接着初始化每个子系统通过
调用子系统的初始化函数, 允许初始化代码通过减少留给正常系统操作使用的 RAM 数量, 
来分配一个内存缓冲给自己用.

启动时内存分配通过调用下面一个函数进行: 
#include <linux/bootmem.h> 
void *alloc_bootmem(unsigned long size); 
void *alloc_bootmem_low(unsigned long size); 
void *alloc_bootmem_pages(unsigned long size); 
void *alloc_bootmem_low_pages(unsigned long size); 

这些函数分配或者整个页(如果它们以 _pages 结尾)或者非页对齐的内存区. 分配的内存
可能是高端内存除非使用一个 _low 版本.
一个接口释放这个内存:
void free_bootmem(unsigned long addr, unsigned long size);

```


# 13_虚拟文件系统  
VFS抽象层之所以能衔接各种各样的文件系统，是因为它定义了所有文件系统都支持的、基本的、概念上的接口和数据结构。实际文件系统通过编程提供VFS所期望的抽象接口和数据结构，这样，内核就可以毫不费劲地和任何文件系统协同工作，并且这样提供给用户空间的接口，也可以和任何文件系统无缝地连接在一起，完成实际工作。  
## VFS对象及其数据结构  
VFS中有四个主要的对象类型，它们分别是：  
- 超级块对象，它代表一个具体的已安装文件系统  
- 索引节点对象，它代表一个具体的文件  
- 目录项对象，它代表一个目录项，是路径组成的一部分
- 文件对象，它代表由进程打开的文件  

## 超级块对象  
超级块对象super_block，各种文件系统都必须实现超级块对象，**该对象用于存储特定文件系统的信息**，通过对应于存放在磁盘特定扇区中的文件系统超级块或文件系统控制块(所以称为超级块对象)。对于非基于磁盘的文件系统(如基于ram的文件系统，sysfs)，它们会在使用现场创建超级块并将其保存到内存中。  
## 索引节点对象  
索引节点对象包含了内核在**操作文件或目录时需要的全部信息**。对于Unix风格的文件系统来说，这些信息可以从磁盘索引节点直接读入。如果一个文件系统没有索引节点，那么，不管这些相关信息在磁盘上怎么存放的，文件系统都必须从中提取这些信息。  
## 目录项对象  
VFS把目录当做文件对待。为了方便查找操作，VFS引入了目录项的概念。每个dentry代表路径中的一个特定部分。目录项对象**没有对应的磁盘数据结构，VFS根据字符串形式的路径现场创建它**。  
如果VFS层遍历路径中所有的元素并将它们逐个地解析成目录项对象，还有达到最深层目录，将是一件非常费力的工作，会浪费大量的时间。所以内核将目录项对象缓存在目录项缓存(简称dcache)中。  
## 文件对象  
文件对象file表示进程已打开的文件。  

# 块I/O层  
系统中能够**随机**(不需要按顺序)访问**固定大小**数据片的硬件设备称作块设备，这些固定大小的数据片就称作块。常见的块设备是硬盘，注意，它们都是以**安装文件系统**的方式使用的--这是块设备一般的访问方式。  

## 剖析一个块设备  
**块设备中最小的可寻址单元是扇区。扇区大小一般是2的整数倍，而最常见的是512字节**。扇区的大小是设备设备的物理属性，扇区是所有块设备的基本单元--**块设备无法对比它还小的单元进行寻址和操作**。  
因为各种软件的用途不同，所以它们都会用到自己的最小逻辑可寻址单元--**块**。块是文件系统的一种抽象--只能基于块来访问文件系统。虽然物理磁盘寻址是按照扇区级进行的，但是内核执行的所有磁盘操作都是按照块来进行。**块的大小必须是扇区大小2的整数被，并且要小于页面大小。  

## 缓冲区和缓冲头  
每个缓冲区与一个块对应，它相当于是磁盘块在内存中的表示。每个缓冲区都有一个对应的描述符，该描述符用buffer_head结构表示，称为缓冲头，它包含了内核操作缓冲区所需的全部信息。该结构只扮演了一个描述符的角色，说明**缓冲区到块的映射关系**。对于内核来说，更倾向于操作页面结构，在2.6版中，许多IO操作都是直接对页面或地址空间进行操作来完成，不在使用缓冲区头。  
## bio结构体  
内核块I/O操作的基本容器是由bio结构体表示，该结构体代表了**正在现场的以片段链表形式组织的块I/O操作**。一个片段是一小块连续的内存缓冲区。这样的话，就不需要保证单个缓冲区一定要连续。  

```
struct bio {
	sector_t		bi_sector;	/* device address in 512 byte
						   sectors */
	struct bio		*bi_next;	/* request queue link */
	struct block_device	*bi_bdev;
	unsigned long		bi_flags;	/* status, command, etc */
	unsigned long		bi_rw;		/* bottom bits READ/WRITE,
						 * top bits priority
						 */

	unsigned short		bi_vcnt;	/* how many bio_vec's */
	unsigned short		bi_idx;		/* current index into bvl_vec */

	/* Number of segments in this BIO after
	 * physical address coalescing is performed.
	 */
	unsigned int		bi_phys_segments;

	unsigned int		bi_size;	/* residual I/O count */

	/*
	 * To keep track of the max segment size, we account for the
	 * sizes of the first and last mergeable segments in this bio.
	 */
	unsigned int		bi_seg_front_size;
	unsigned int		bi_seg_back_size;

	unsigned int		bi_max_vecs;	/* max bvl_vecs we can hold */

	unsigned int		bi_comp_cpu;	/* completion CPU */

	atomic_t		bi_cnt;		/* pin count */

	struct bio_vec		*bi_io_vec;	/* the actual vec list */

	bio_end_io_t		*bi_end_io;

	void			*bi_private;
#if defined(CONFIG_BLK_DEV_INTEGRITY)
	struct bio_integrity_payload *bi_integrity;  /* data integrity */
#endif

	bio_destructor_t	*bi_destructor;	/* destructor */

	/*
	 * We can inline a number of vecs at the end of the bio, to avoid
	 * double allocations for a small number of bio_vecs. This member
	 * MUST obviously be kept at the very end of the bio.
	 */
	struct bio_vec		bi_inline_vecs[0];
};
```
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/185CD6F40C8C4E01BEF90E898C81D1E1/16067)  

## 请求队列
块设备将它们挂起的块I/O请求保存在请求队列中，该请求队列由reques_queue结构体表示。通过内核中像文件系统这样高层的代码将请求加入到队列中。请求队列只要不为空，队列对应的块设备驱动程序就会从队列头获取请求，然后将其送入到对应的块设备上去。  
## I/O调度程序  
每一次寻址(移动硬盘磁头到特定块上的某个位置)需要花费不少时间，所以尽量缩短寻址时间无疑是提高系统性能的关键。对磁盘请求进行**合并和排序**操作，可以极大地提高系统的整体性能。在内核中负责提交I/O请求的子系统称为I/O调度程序。  
合并指将两个或多个请求结合成一个请求。排序是整个请求队列按照扇区增长方向有序排列。  
### Linus 电梯  
linus 电梯执行合并和排序预处理。当一个请求加入到队列中时，有可能发生四中操作：  
- 如果队列中已经存在一个对相邻磁盘扇区操作的请求，那么请求将和这个已经存在的请求合并成一个请求。  
- 如果队列中存在一个**驻留时间过长的请求**，那么新请求将插入到队列尾部，以防止其他旧的请求饥饿发生。  
- 如果队列中已扇区方向为序存在合适的插入位置，那么新的请求将被插入到该位置，保证队列中的请求是以被访问磁盘物理位置为序进行排序。  
- 如果队列中不存在合适的请求插入位置，请求将被插入到队列尾部。  

linus电梯中的“年龄”检测方法并不很有效，因为并非是给了等待的一段时间的请求提供实质性服务，它仅仅是经过了一定时间后停止插入排序请求，这改善了等待时间但最终还是会导致**请求饥饿**现象的发生。  
### 最终期限I/O调度  
最终期限I/O调度程序是为了解决linus电梯所带来的饥饿问题而提出的。读操作就有同步性，并且彼此之间往往相互依靠，所以读请求响应时间直接影响系统性能，最后期限I/O调度程序来**减少请求饥饿现象，特别是读请求饥饿现象**。  
减少请求饥饿现象必须以降低全局吞吐量为代价。  
在最后期限I/O调度程序中，每个请求都有一个超时时间。默认情况下，读请求的超时时间是500ms，写请求的超时时间为5s。最后期限I/O调度请求类似于linus地电梯，以磁盘物理位置为次序维护请求队列，这个队列称为**排序队列**。当一个新的请求提交给排序队列时，执行合并与插入请求时类似于Linus电梯，但是最后期限I/O调度程序同时也会以请求类型为依据将它们插入到额外队列中。读请求按次序被插入**读FIFO**队列中，写请求按次序被插入**写FIFO**队列中。普通队列以磁盘扇区为序进行排列，但是FIFO以时间为基准进行排序组织的。对于普通操作来说，调度程序将请求从排序队列的头部取下，在推入到**派发队列**中，派发队列然后将请求提交给磁盘驱动，从而保证了最小化的请求寻址。  
如果在写FIFO队列头，或者在读FIFO队列头的请求超时，那么最后期限I/O调度程序便从FIFO队列中提取请求进行服务。依靠这种方法，最后期限I/O调度程序试图保证不会发生有请求在明显超期的情况下仍不能得到服务的现象。  
![iamge](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/F6C40669141242AC8F78E73FD7B39318/16190)  

### 预测I/O调度程序  
预测I/O调度程序的目标是在保持良好的读良好相应的同时也能提供良好的全局吞吐量。  
预测I/O调度的基础任然是最后期限I/O调度程序，所以它们有很多相同之处。预测I/O调度程序也实现三个队列(加上一个派发队列)，并为每个请求设置了超时时间，这点与最后期限I/O调度程序一样。预测I/O调度程序最主要的改进是它增加了**预测启发**能力。  
预测I/O调度试图减少在进行I/O操作期间，**处理新到的读请求所带来的寻址数量**。请求提交后并不直接返回处理其他请求，而是刻意**空闲片刻**(实际空闲时间可以设置，默认6ms)。这几ms，对应用程序来说是提交其他请求的好机会--任何对相邻磁盘位置操作的请求都会立刻得到处理。在等待时间结束后，预测I/O调度程序重新返回原来的位置，继续执行以前剩下的请求。   
预测I/O调度程序所能带来的优势取决于**能否正确预测应用程序和文件系统的行为**。预测I/O调度程序跟踪并且统计每个应用程序块I/O操作的习惯行为，以便正确预测应用程序的未来行为。  

### 完全公平的排队I/O调度程序  
完全公平的排队I/O调度程序(complete fair queuing, CFQ)是为了专有的负荷设计的，不过，在实际中，也为多种工作负荷提供了良好的性能。  
CFQ I/O调度程序把进入的I/O请求放入特定的队列中，**这种队列是根据引起I/O请求的进程组织的**。CFQ I/O 调度程序的差异在于每一个提交I/O的进程都有自己的队列。 CFQ I/Q 调度程序以时间片轮转调度队列，从每个队列中选取请求数(默认值为4，可以进行配置)，然后进行下一轮调度。**这就在进程级提供了公平，确保每个进程接收公平的磁盘宽带片段**。  
### 空操作的I/O调度程序  
空操作I/O调度程序不进行排序，或者也不进行什么其他形式的预寻址操作。不过，**空操作I/O调度程序忘不了执行合并**，当一个新的请求提交到队列时，就把它与任一相邻的请求合并。  


# 15_进程地址空间  
每个进程都有一个32位或64位的平坦(flat)地址空间，空间的大小取决于体系结构。尽管一个进程可以寻址4GB虚拟地址空间(在32位地址空间中)，但是**并不代表它就有权访问所有的虚拟地址**。在地址空间中我们更关心的是一些虚拟内存的地址空间，它们可以被进程访问。这些可被访问的合法地址空间被称为**内存区域**。通过内核，进程可以给自己的地址空间动态地添加或减少内存区域。  
进程只能访问有效内存区域内的内存地址。每个内存区域也**具有相关权限**如对相关进程有可读、可写、可执行属性。  
## 内存描述符  
内核使用内存描述符结构体表示进程的地址空间，该结构包含了和**进程地址空间有关的全部信息**。内存描述符由mm_struct结构体表示。  
```
struct mm_struct {
	struct vm_area_struct * mmap;		/* list of VMAs */ //内存区域链表
	struct rb_root mm_rb; //VMA形成的红黑树
	struct vm_area_struct * mmap_cache;	/* last find_vma result */ //最近使用的内存区域
#ifdef CONFIG_MMU
	unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
	void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
#endif
	unsigned long mmap_base;		/* base of mmap area */
	unsigned long task_size;		/* size of task vm space */ //栈的虚拟地址大小
	unsigned long cached_hole_size; 	/* if non-zero, the largest hole below free_area_cache */
	unsigned long free_area_cache;		/* first hole of size cached_hole_size or larger */ //地址空间第一个空洞
	pgd_t * pgd;         //页全局目录
	atomic_t mm_users;			/* How many users with user space? */ //使用地址空间的用户数
	atomic_t mm_count;			/* How many references to "struct mm_struct" (users count as 1) */ //主使用计数器
	int map_count;				/* number of VMAs */ //内存区域的个数
	struct rw_semaphore mmap_sem; //内存区域的信号量
	spinlock_t page_table_lock;		/* Protects page tables and some counters */

	struct list_head mmlist;		/* List of maybe swapped mm's.	These are globally strung
						 * together off init_mm.mmlist, and are protected
						 * by mmlist_lock
						 */ //所有mmlist形成的链表


	unsigned long hiwater_rss;	/* High-watermark of RSS usage */ //高水位
	unsigned long hiwater_vm;	/* High-water virtual memory usage */
    //数据段、堆、环境变量、命令行参数等的首地址
	unsigned long total_vm, locked_vm, shared_vm, exec_vm;
	unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
	unsigned long start_code, end_code, start_data, end_data;
	unsigned long start_brk, brk, start_stack;
	unsigned long arg_start, arg_end, env_start, env_end;

	unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

	/*
	 * Special counters, in some configurations protected by the
	 * page_table_lock, in other configurations by being atomic.
	 */
	struct mm_rss_stat rss_stat;

	struct linux_binfmt *binfmt;

	cpumask_t cpu_vm_mask;

	/* Architecture-specific MM context */
	mm_context_t context;

	/* Swap token stuff */
	/*
	 * Last value of global fault stamp as seen by this process.
	 * In other words, this value gives an indication of how long
	 * it has been since this task got the token.
	 * Look at mm/thrash.c
	 */
	unsigned int faultstamp;
	unsigned int token_priority;
	unsigned int last_interval;

	unsigned long flags; /* Must use atomic bitops to access the bits */

	struct core_state *core_state; /* coredumping support */
#ifdef CONFIG_AIO
	spinlock_t		ioctx_lock;
	struct hlist_head	ioctx_list; // AIO I/O链表
#endif
#ifdef CONFIG_MM_OWNER
	/*
	 * "owner" points to a task that is regarded as the canonical
	 * user/owner of this mm. All of the following must be true in
	 * order for it to be changed:
	 *
	 * current == mm->owner
	 * current->mm != mm
	 * new_owner->mm == mm
	 * new_owner->alloc_lock is held
	 */
	struct task_struct __rcu *owner;
#endif

#ifdef CONFIG_PROC_FS
	/* store ref to file /proc/<pid>/exe symlink points to */
	struct file *exe_file;
	unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
	struct mmu_notifier_mm *mmu_notifier_mm;
#endif
#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	pgtable_t pmd_huge_pte; /* protected by page_table_lock */
#endif
	/* How many tasks sharing this mm are OOM_DISABLE */
	atomic_t oom_disable_count;
};
```
mm_users域记录正在使用该地址空间的进程数目。mmap和mm_rb这两个不同数据结构体描述符的对象是相同的，即该地址空间中的全部内存区域；mmap结构体作为链表，利于简单、高效地遍历所有元素；而mm_rb结构体作为红黑树，适合搜索指定的元素。  
**所有的mm_struct结构体都通过自身的mmlist域连接在一个双向链表中**，该链表的首元素是init_mm内存描述符，它代表init进程的地址空间。  
### 分配内存描述符  
fork()函数利用copy_mm()函数复制父进程的内存描述符，子进程中的mm_struct结构体通过allocate_mm()宏从mm_cachep slab缓存中分配得到的。  
如果父进程希望子进程**共享地址空间**，可以调用clone()时，设置CLONE_VM标志，我们把这样的进程称作线程。共享地址空间时，子进程并没有分配mm_struct结构体，只是使用指针指向了父进程的mm_struct结构。  
当进程退出时，内核调用exit_mm()函数，该函数执行一些常规的撤销工作，同时更新一个统计量。  
### mm_struct与内核线程  
**内核线程没有进程地址空间，也没有相关的内存描述符**。所以内核线程对应的进程描述符中mm域为空，内核线程没有用户上下文。  
内核线程不需要访问用户空间的内存，但是要访问内核内存，内核线程也还是需要使用一些数据的，比如页表。**为了避免内核线程为内存描述符和页表浪费内存，也为了当新内核线程运行时，避免浪费处理器周期向新地址空间进程转换，内核线程将直接使用前一个进程的内存描述符**。  
当一个内核线程被调度时，内核发现它的mm域为NULL，就会保留前一个进程的地址空间，随后内核更新内核线程**对应的**进程描述符中的active_mm域，使其指向前一个进程的内存描述符。所以在需要时，内核线程边可以使用前一个进程的页表。因为内核线程不访问用户空间，所以它们仅仅**用地址空间中和内核内存相关**的信息，这些信息的含义和普通进程完全相同。(所以进程共享内核1GB的内核空间)   

## 虚拟内存区域  
内存区域由vm_area_struct结构体描述，被称为虚拟内存区域。vm_area_struct结构体描述了指定地址空间连续区间上的一个独立内存范围。内核将每个内存区域作为一个单独的内存对象管理，每个内存区域**都拥有一致的属性，比如访问权限等，另外，相应的操作也都一致。  
```
struct vm_area_struct {
	struct mm_struct * vm_mm;	/* The address space we belong to. *///相关的mm_struct结构
	unsigned long vm_start;		/* Our start address within vm_mm. 该内存区域的起始处和结束处的虚拟地址 */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */ //区间的尾地址

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */ //访问控制权限
	unsigned long vm_flags;		/* Flags, see mm.h. */ //标志位

	struct rb_node vm_rb; //树上该VMA节点

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap prio tree, or
	 * linkage to the list of like vmas hanging off its node, or
	 * linkage of vma in the address_space->i_mmap_nonlinear list.
	 */
	union {
		struct {
			struct list_head list;
			void *parent;	/* aligns with prio_tree_node parent */
			struct vm_area_struct *head;
		} vm_set;

		struct raw_prio_tree_node prio_tree_node;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_sem &
					  * page_table_lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock *///匿名VMA对象

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops; //VMA 操作

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE //文件中的偏移量
					   units, *not* PAGE_CACHE_SIZE */
	struct file * vm_file;		/* File we map to (can be NULL). 指向该区域相关联的file结构指针 */
	void * vm_private_data;		/* was vm_pte (shared mem) */
	unsigned long vm_truncate_count;/* truncate_count or restart_addr */

#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
};

```  
vm_flags域内是VMA标志，标志了内存区域**所包含的页面的行为和信息**。和物理页的访问权限不同，**VMA标志反应了内核处理页面所需要遵守的行为准则，而不是硬件要求**。  

## mmap()和do_mmap():创建地址空间  
内核使用do_mmap()函数创建一个新的线性地址区间。但是说该函数创建了一个新的VMA并不非常准确，因为如果创建的地址区间和一个已经存在的地址区间相邻，并且他们有相同的访问权限的话，两个区间将合并为一个。如果不能合并，就确实要创建一个新的VMA了。  
```
unsigned long do_mmap(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long offset)
```  
如果file参数为NULL并且offset参数也是0，那么代表这次映射没有和文件相关，该情况称作**匿名映射**。如果指定了文件名和偏移量，那么该映射称为**文件映射**。  
用户空间使用mmap()系统调用获取内核函数do_mmap()的功能。  
mummap()和do_mummap()：删除地址区间   

## 页表  
虚拟地址到物理地址的映射需要通过页表转换，即地址转换需要将虚拟地址分段，使每段虚拟地址都作为一个索引指向页表，则页表项指向下一级别的页表或最终的物理页面。  
每个进程都有自己的页表(线程会共享页表)。内存描述符的pgd域指向的就是进程的**页全局目录**。每个进程的在页全局目录中是唯一的。  
![image](https://note.youdao.com/yws/public/resource/d59db60d3f7778a1029a388f302c78b0/xmlnote/BEC232CCB8794FCF9CC620044DC6CA39/16446)  
多数体系结构都是用TLB作为虚拟地址映射物理地址的硬件缓存，加快地址访问。  

# 16_页高速缓存和页回写  
**页高速缓存(cache)是linux内核实现磁盘缓存**。它主要用来减少对磁盘的I/O操作。具体地讲，是通过把磁盘中的数据缓存到**物理内存**中，把对磁盘的访问变为对物理内存的访问。  
磁盘高速缓存之所以在任何现代操作系统中尤为重要源自两个因素：第一，**访问磁盘的速度要远远低于访问内存的速度--ms和ns的差距**；第二，**数据一旦被访问，就很有可能在短期内再次被访问到**。  
## 缓存手段  
页面高速缓存是由内存中的物理页面组成的，其内容对应磁盘上的物理块。页高速缓存大小能动态调整--它可以通过占用空闲内存以扩张大小，也可以自我收缩以缓解内存使用压力。当内核开始一个读操作(进程发起一个read()系统调用)，它首先会检查需要的数据是否在页高速缓存中。如果在，则放弃访问磁盘，直接从内存中读取，这个行为称作缓存命中。如果数据没有在缓存中，称为缓存未命中，那么内核必须调度块I/O操作从磁盘去读取数据，然后内核将读来的数据放入到页缓存中，于是任何后续相同的数据读取都可命中缓存。  
### 写缓存  
缓存一般实现三种策略之一：  
1. 不缓存，也就是说高速缓存不缓存任何写操作。当对一个缓存中的数据片进行写时，将直接跳过缓存，写到磁盘，同时也使缓存中的数据失效。  
2. 自动更新内存缓存，同时也更新磁盘文件，通常也称为**写透缓存**。这种策略对缓存一致性有好处。  
3. **回写**，这种策略下，程序执行写操作直接写到缓存中，后端存储不会立刻直接更新，而是将页高速缓存中被写入的页面标记为**脏**，并且被加入到脏页链表中。然后由一个进程(回写进程)周期性将脏页链表中的页写回到磁盘，从而让磁盘中数据和内存中最终一致。  

### 缓存回收  
Linux的缓存回收是通过选择干净页(不脏)进行简单替换。  
#### 最近最少使用  
最近最少使用，简称LRU，回收策略需要跟踪每个页面的访问踪迹(或者至少按照访问时间为序的页链表)，以便能回收最老时间戳的页面(或着回收排序链表头所指的页面)。  
#### 双链策略  
Linux实现的是一个修改过的LRU，也称为双链策略。维护两个链表：活跃链表和非活跃链表。处于活跃链表上的页面被认为是“热”的且不会被换出，而处于非活跃的页面则是可以被换出的。两个链表都被伪LRU规则维护：页面从尾部加入，从头部移除，如同队列。更普通的是n个链表，故称为LRU/n。  

## Linux 页高速缓存  
Linux页高速缓存的目标是缓存任何基于页的对象，这包含各种类型的文件和各种类型的内存映射。  
Linux页高速缓存使用了一个新对象管理缓存项和页I/O操作，这个对象是address_space结构体。该结构体是虚拟地址vm_zrea_struct的物理地址对等体。当一个文件可以被多个vm_area_struct结构体标识，那么这个文件**只能由一个**address_space数据结构--也就是文件可以有多个虚拟地址，但是只能在内存有一份。  
```
struct address_space {
	struct inode		*host;		/* owner: inode, block_device */ //拥有节点
	struct radix_tree_root	page_tree;	/* radix tree of all pages */ //包含全部页面的radix树
	spinlock_t		tree_lock;	/* and lock protecting it */
	unsigned int		i_mmap_writable;/* count VM_SHARED mappings */ //VM_SHARE计数
	struct prio_tree_root	i_mmap;		/* tree of private and shared mappings */ //私有映射链表
	struct list_head	i_mmap_nonlinear;/*list VM_NONLINEAR mappings */
	spinlock_t		i_mmap_lock;	/* protect tree, count, list */
	unsigned int		truncate_count;	/* Cover race condition with truncate */ //截断计数
	unsigned long		nrpages;	/* number of total pages */ //页总数
	pgoff_t			writeback_index;/* writeback starts here */ //回写的起始偏移地址
	const struct address_space_operations *a_ops;	/* methods */
	unsigned long		flags;		/* error bits/gfp mask */
	struct backing_dev_info *backing_dev_info; /* device readahead, etc */
	spinlock_t		private_lock;	/* for use by the address_space */
	struct list_head	private_list;	/* ditto */
	struct address_space	*assoc_mapping;	/* ditto */
	struct mutex		unmap_mutex;    /* to protect unmapping */
} __attribute__((aligned(sizeof(long))));
```
## flusher线程  
页面被写回磁盘发生的条件：  
- 当空闲内存低于一个特定的阈值时，内核必须将脏页写回以便释放内存，因为只有干净内存才可以被回收。当内存干净后，内核就可以从缓存清理数据，然后收缩缓存，最终释放出更多的内存。  
- 当脏页在内存中驻留时间超过一个特定的阈值时，内核必须将超时的脏页写回到磁盘，以确保脏页不会无限地驻留在内存中。
- 当用户进程调用sync()和fsync()系统调用时，内核会按要求执行回写动作  

在Linux 2.6内核中，由一组内核线程(flusher线程)执行这三种工作。  
首先，flusher线程在系统中的空闲内存低于一个特定的阈值时，将脏页刷新写回到磁盘。当空闲内存特别少时，内核便会唤醒一个或多个flusher线程。其次，flusher线程后台例程被周期性唤醒，将那些在内存中驻留时间过长的脏页写出，确保内存中不会有长期存在的脏页。  
























































