## 1. 调度器

Go 语言在并发编程方面有强大的能力，这离不开语言层面对并发编程的支持。

谈到 Go 语言调度器，我们绕不开的是操作系统、进程与线程这些概念，线程是操作系统调度时的最基本单元，而 Linux 在调度器并不区分进程和线程的调度，它们在不同操作系统上也有不同的实现，但是在大多数的实现中线程都属于进程：

<img src="/Users/yuanbli/yuanbli/markdown/Golang/pic/GMP_1.png" alt="4.1" style="zoom:75%;" />

多个线程可以属于同一个进程并共享内存空间。因为多线程不需要创建新的虚拟内存空间，所以它们也不需要内存管理单元处理上下文的切换，线程之间的通信也正是基于共享的内存进行的，与重量级的进程相比，线程显得比较轻量。

虽然线程比较轻量，但是在调度时也有比较大的额外开销。每个线程会都占用 1M 以上的内存空间，在切换线程时不止会消耗较多的内存，恢复寄存器中的内容还需要向操作系统申请或者销毁资源，每一次线程上下文的切换都需要消耗 ~1us 左右的时间，但是 Go 调度器对 Goroutine 的上下文切换约为 ~0.2us，减少了 80% 的额外开销。

<img src="/Users/yuanbli/yuanbli/markdown/Golang/pic/GMP_2.png" alt="4.1" style="zoom:75%;" />

Go 语言的调度器通过使用与 CPU 数量相等的线程减少线程频繁切换的内存开销，同时在每一个线程上执行额外开销更低的 Goroutine 来降低操作系统和硬件的负载。

### 1.1 设计原理

 Go 语言调度器有着优异的性能，调度器经过几个大版本的迭代才有今天的优异性能，历史上几个不同版本的调度器引入了不同的改进，也存在着不同的缺陷:

- 单线程调度器 0.x
  - 只包含 40 多行代码；
  - 程序中只能存在一个活跃线程，由 G-M 模型组成；
- 多线程调度器 1.0
  - 允许运行多线程的程序；
  - 全局锁导致竞争严重；
- 任务窃取调度器 1.1
  - 引入了处理器 P，构成了目前的 **G-M-P** 模型；
  - 在处理器 P 的基础上实现了基于**工作窃取**的调度器；
  - 在某些情况下，Goroutine 不会让出线程，进而造成饥饿问题；
  - 时间过长的垃圾回收（Stop-the-world，STW）会导致程序长时间无法工作；
- 抢占式调度器 1.2~ 至今
  - 基于协作的抢占式调度器 - 1.2 ~ 1.13
    - 通过编译器在函数调用时插入**抢占检查**指令，在函数调用时检查当前 Goroutine 是否发起了抢占请求，实现基于协作的抢占式调度；
    - Goroutine 可能会因为垃圾回收和循环长时间占用资源导致程序暂停；
  - 基于信号的抢占式调度器 - 1.14 ~ 至今
    - 实现**基于信号的真抢占式调度**；
    - 垃圾回收在扫描栈时会触发抢占调度；
    - 抢占的时间点不够多，还不能覆盖全部的边缘情况；
- 非均匀存储访问调度器 · 提案
  - 对运行时的各种资源进行分区；
  - 实现非常复杂，到今天还没有提上日程；

除了多线程、任务窃取和抢占式调度器之外，Go 语言社区目前还有一个非均匀存储访问（Non-uniform memory access，NUMA）调度器的提案。

#### 1.1.1 单线程调度器

0.x 版本调度器只包含表示 Goroutine 的 G 和表示线程的 M 两种结构，全局也只有一个线程。调度函数只包含 40 多行代码：

```go
static void scheduler(void) {
	G* gp;
	lock(&sched);

	if(gosave(&m->sched)){
		lock(&sched);
		gp = m->curg;
		switch(gp->status){
		case Grunnable:
		case Grunning:
			gp->status = Grunnable;
			gput(gp);
			break;
		...
		}
		notewakeup(&gp->stopped);
	}

	gp = nextgandunlock();
	noteclear(&gp->stopped);
	gp->status = Grunning;
	m->curg = gp;
	g = gp;
	gogo(&gp->sched);
}
```

该函数会遵循如下的过程调度 Goroutine：

1. 获取调度器的全局锁；
2. 调用 `runtime.gosave:9682400`保存栈寄存器和程序计数器；
3. 调用 `runtime.nextgandunlock:9682400`获取下一个需要运行的 Goroutine 并解锁调度器；
4. 修改全局线程 `m` 上要执行的 Goroutine；
5. 调用 `runtime.gogo:9682400` 函数运行最新的 Goroutine；

#### 1.1.2 多线程调度器

Go 语言在 1.0 版本正式发布时就支持了多线程的调度器。核心代码：

```go
static void schedule(G *gp) {
	schedlock();
	if(gp != nil) {
		gp->m = nil;
		uint32 v = runtime·xadd(&runtime·sched.atomic, -1<<mcpuShift);
		if(atomic_mcpu(v) > maxgomaxprocs)
			runtime·throw("negative mcpu in scheduler");

		switch(gp->status){
		case Grunning:
			gp->status = Grunnable;
			gput(gp);
			break;
		case ...:
		}
	} else {
		...
	}
	gp = nextgandunlock();
	gp->status = Grunning;
	m->curg = gp;
	gp->m = m;
	runtime·gogo(&gp->sched, 0);
}
```

整体的逻辑与单线程调度器没有太多区别，因为我们的程序中可能同时存在多个活跃线程，所以多线程调度器引入了 `GOMAXPROCS` 变量帮助我们灵活控制程序中的最大处理器数，即活跃线程数。

多线程调度器的主要问题是调度时的锁竞争会严重浪费资源，该调度器有以下问题需要解决：

1. 调度器和锁是全局资源，所有的调度状态都是中心化存储的，锁竞争问题严重；
2. 线程需要经常互相传递可运行的 Goroutine，引入了大量的延迟；
3. 每个线程都需要处理内存缓存，导致大量的内存占用并影响数据局部性；
4. 系统调用频繁阻塞和解除阻塞正在运行的线程，增加了额外开销；

#### 1.1.3 任务窃取调度器

2012 年 Google 的工程师 Dmitry Vyukov 在 [Scalable Go Scheduler Design Doc](http://golang.org/s/go11sched) 中指出了现有多线程调度器的问题并在多线程调度器上提出了两个改进的手段：

1. 在当前的 G-M 模型中引入了处理器 P，增加中间层；
2. 在处理器 P 的基础上实现基于工作窃取的调度器；

基于任务窃取的 Go 语言调度器使用了沿用至今的 G-M-P 模型，我们能在 runtime: improved scheduler提交中找到任务窃取调度器刚被实现时的源代码：

```go
static void schedule(void) {
    G *gp;
 top:
    if(runtime·gcwaiting) {
        gcstopm();
        goto top;
    }

    gp = runqget(m->p);//从本地运行队列获取待执行Gorotine
    if(gp == nil)
        gp = findrunnable(); //运行队列不包含gorotine 触发工作窃取

    ...

    execute(gp); // 在当前线程M上运行Gorotine
}
```

1. 如果当前运行时在等待垃圾回收，调用 `runtime.gcstopm:779c45a`函数；
2. 调用 `runtime.runqget:779c45a`和 `runtime.findrunnable:779c45a`从本地或者全局的运行队列中获取待执行的 Goroutine；
3. 调用 `runtime.execute:779c45a` 在当前线程 M 上运行 Goroutine；

当前处理器本地的运行队列中不包含 Goroutine 时，调用 `runtime.findrunnable:779c45a`会触发工作窃取，从其它的处理器的队列中随机获取一些 Goroutine。

运行时 G-M-P 模型中引入的处理器 P 是线程和 Goroutine 的中间层，我们从它的结构体中就能看到处理器与 M 和 G 的关系：

```go
struct P {
	Lock;

	uint32	status;
	P*	link;
	uint32	tick;
	M*	m;		// 线程M
	MCache*	mcache;

	G**	runq;  // 环形的运行队列
	int32	runqhead;
	int32	runqtail;
	int32	runqsize;

	G*	gfree;
	int32	gfreecnt;
};
```

处理器持有一个由可运行的 Goroutine 组成的环形的运行队列 `runq`，还反向持有一个线程。调度器在调度时会从处理器的队列中选择队列头的 Goroutine 放到线程 M 上执行。如下所示的图片展示了 Go 语言中的线程 M、处理器 P 和 Goroutine 的关系。

<img src="/Users/yuanbli/yuanbli/markdown/Golang/pic/GMP_3.png" alt="4.1" style="zoom:75%;" />

基于工作窃取的多线程调度器将每一个线程绑定到了独立的 CPU 上，这些线程会被不同处理器管理，不同的处理器通过工作窃取对任务进行再分配实现任务的平衡，也能提升调度器和 Go 语言程序的整体性能，今天所有的 Go 语言服务都受益于这一改动。

#### 1.1.4 抢占式调度器

对 Go 语言并发模型的修改提升了调度器的性能，但是 1.1 版本中的调度器仍然不支持抢占式调度，程序只能依靠 Goroutine 主动让出 CPU 资源才能触发调度。Go 语言的调度器在 1.2 版本4中引入基于协作的抢占式调度解决下面的问题5：

- 某些 Goroutine 可以长时间占用线程，造成其它 Goroutine 的饥饿
- 垃圾回收需要暂停整个程序(Stop-the-world，STW)，最长可能需要几分钟的时间，导致整个程序无法工作

1.2 版本的抢占式调度虽然能够缓解这个问题，但是它实现的抢占式调度是基于协作的，在之后很长的一段时间里 Go 语言的调度器都有一些无法被抢占的边缘情况，例如：for 循环或者垃圾回收长时间占用线程，这些问题中的一部分直到 1.14 才被基于信号的抢占式调度解决。

**基于协作的抢占式调度**

Go 语言会在分段栈的机制上实现抢占调度，利用编译器在分段栈上插入的函数，所有 Goroutine 在函数调用时都有机会进入运行时检查是否需要执行抢占。

基于协作的抢占式调度的工作原理如下：

1. 编译器会在调用函数前插入 `runtime.morestack`
2. Go 语言运行时会在垃圾回收暂停程序、系统监控发现 Goroutine 运行超过 10ms 时发出抢占请求 `StackPreempt`
3. 当发生函数调用时，可能会执行编译器插入的 `runtime.morestack`，它调用的 `runtime.newstack`会检查 Goroutine 的 `stackguard0` 字段是否为 `StackPreempt`；
4. 如果 `stackguard0` 是 `StackPreempt`，就会触发抢占让出当前线程；

这里的抢占是通过编译器插入函数实现的，还是需要函数调用作为入口才能触发抢占，所以这是一种**协作式的抢占式调度**。

**基于信号的抢占式调度**

Go 语言在 1.14 版本中实现了非协作的抢占式调度，在实现的过程中重构已有的逻辑并为 Goroutine 增加新的状态和字段来支持抢占。

1. 程序启动时，在 `runtime.sighandler` 中注册 `SIGURG` 信号的处理函数 `runtime.doSigPreempt`

2. 在触发垃圾回收的栈扫描时会调用`runtime.suspendG`挂起 Goroutine，该函数会执行下面的逻辑：

   - 将 `_Grunning` 状态的 Goroutine 标记成可以被抢占，即将 `preemptStop` 设置成 `true`；

   - 调用 `runtime.preemptM`触发抢占；

3. `runtime.preemptM` 会调用 `runtime.signalM` 向线程发送信号 `SIGURG`

4. 操作系统会中断正在运行的线程并执行预先注册的信号处理函数 `runtime.doSigPreempt`

5. `runtime.doSigPreempt` 函数会处理抢占信号，获取当前的 SP 和 PC 寄存器并调用 `runtime.sigctxt.pushCall`

6. `runtime.sigctxt.pushCall`会修改寄存器并在程序回到用户态时执行 `runtime.asyncPreempt`

7. 汇编指令 `runtime.asyncPreempt`会调用运行时函数 `runtime.asyncPreempt2`

8. `runtime.asyncPreempt2`会调用 `runtime.preemptPark`

9. `runtime.preemptPark`会修改当前 Goroutine 的状态到 `_Gpreempted` 并调用 `runtime.schedule`让当前函数陷入休眠并让出线程，调度器会选择其它的 Goroutine 继续执行

### 1.2 数据结构

运行时调度器的三个重要组成部分：

- 线程 M：操作系统的线程，它由操作系统的调度器调度和管理
- Goroutine G：是一个待执行的任务
- 处理器 P：处理器，它可以被看做运行在线程上的本地调度器

<img src="/Users/yuanbli/yuanbli/markdown/Golang/pic/GMP_4.png" style="zoom:50%;" />

#### 1.2.1 goroutine G

Goroutine 是 Go 语言调度器中待执行的任务，它在运行时调度器中的地位与线程在操作系统中差不多，但是它占用了更小的内存空间，也降低了上下文切换的开销。

Goroutine 只存在于 Go 语言的运行时，它是 Go 语言在用户态提供的线程，作为一种粒度更细的资源调度单元，如果使用得当能够在高并发的场景下更高效地利用机器的 CPU。

Goroutine 在 Go 语言运行时使用私有结构体 [`runtime.g`](https://draveness.me/golang/tree/runtime.g) 表示。这个私有结构体非常复杂，总共包含 40 多个用于表示各种状态的成员变量。

```go
type g struct {
	stack       stack		// 栈内存范围[stack.lo, stack.hi]
	stackguard0 uintptr	// 用于调度器抢占式调度
  
  preempt       bool // 抢占信号
	preemptStop   bool // 抢占时将状态修改成 `_Gpreempted`
	preemptShrink bool // 在同步安全点收缩栈
  
  _panic       *_panic // 最内侧的 panic 结构体
	_defer       *_defer // 最内侧的延迟函数结构体

	m              *m		 	// 当前 Goroutine 占用的线程，可能为空
	sched          gobuf 	// Goroutine 的状态
	atomicstatus   uint32	// 存储 Goroutine 的调度相关的数据
	goid           int64 	// Goroutine 的 ID，该字段对开发者不可见
}
```

需要展开介绍 `sched` 字段：

```go
type gobuf struct {
	sp   uintptr			// 栈指针
	pc   uintptr			// 程序计数器
	g    guintptr			// 持有gobuf的goroutine
	ret  sys.Uintreg	// 系统调用的返回值
	...
}
```

这些内容会在调度器保存或者恢复上下文的时候用到，其中的栈指针和程序计数器会用来存储或者恢复寄存器中的值，改变程序即将执行的代码。

结构体 `runtime.g`的 `atomicstatus` 字段存储了当前 Goroutine 的状态。除了几个已经不被使用的以及与 GC 相关的状态之外，Goroutine 可能处于以下 9 种状态：

| 状态          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `_Gidle`      | 刚刚被分配并且还没有被初始化                                 |
| `_Grunnable`  | 没有执行代码，没有栈的所有权，存储在运行队列中               |
| `_Grunning`   | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P  |
| `_Gsyscall`   | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上 |
| `_Gwaiting`   | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
| `_Gdead`      | 没有被使用，没有执行代码，可能有分配的栈                     |
| `_Gcopystack` | 栈正在被拷贝，没有执行代码，不在运行队列上                   |
| `_Gpreempted` | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒 |
| `_Gscan`      | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |

上述状态中比较常见是 `_Grunnable`、`_Grunning`、`_Gsyscall`、`_Gwaiting` 和 `_Gpreempted` 五个状态。Goroutine 的状态迁移是个复杂的过程，触发 Goroutine 状态迁移的方法也很多，没有办法介绍全部的迁移路线，从中选择一些介绍。

虽然 Goroutine 在运行时中定义的状态非常多而且复杂，但是我们可以将这些不同的状态聚合成三种：等待中、可运行、运行中，运行期间会在这三种状态来回切换：

- 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态
- 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 `_Grunnable`
- 运行中：Goroutine 正在某个线程上运行，即 `_Grunning`

<img src="/Users/yuanbli/yuanbli/markdown/Golang/pic/GMP_5.png" style="zoom:50%;" />

上图展示了 Goroutine 状态迁移的常见路径，其中包括创建 Goroutine 到 Goroutine 被执行、触发系统调用或者抢占式调度器的状态迁移过程。

#### 1.2.2 线程 M

Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行。

在默认情况下，运行时会将 `GOMAXPROCS` 设置成当前机器的核数，我们也可以在程序中使用 `runtime.GOMAXPROCS`来改变最大的活跃线程数。

<img src="/Users/yuanbli/yuanbli/markdown/Golang/pic/GMP_6.png" style="zoom:50%;" />

在默认情况下，一个四核机器会创建四个活跃的操作系统线程，每一个线程都对应一个运行时中的 `runtime.m`结构体。

在大多数情况下，我们都会使用 Go 的默认设置，也就是线程数等于 CPU 数，默认的设置不会频繁触发操作系统的线程调度和上下文切换，所有的调度都会发生在用户态，由 Go 语言调度器触发，能够减少很多额外开销。

Go 语言会使用私有结构体 `runtime.m`表示操作系统线程，这个结构体也包含了几十个字段，这里先来了解几个与 Goroutine 相关的字段：

```go
type m struct {
	g0   *g  // 持有调度栈的Goroutine
	curg *g	 // 当前线程上运行的用户Goroutine
  
  p             puintptr	// 正在运行代码的处理器
	nextp         puintptr	// 暂存的处理器
	oldp          puintptr	// 执行系统调用之前使用线程的处理器
	...
}
```

其中 g0 是持有调度栈的 Goroutine，`curg` 是在当前线程上运行的用户 Goroutine，这也是操作系统线程唯一关心的两个 Goroutine。

g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。

#### 1.2.3 处理器 P

调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。

因为调度器在启动时就会创建 `GOMAXPROCS` 个处理器，所以 Go 语言程序的处理器数量一定会等于 `GOMAXPROCS`，这些处理器会绑定到不同的内核线程上

`runtime.p`是处理器的运行时表示，作为调度器的内部实现，它包含的字段也非常多，其中包括与性能追踪、垃圾回收和计时器相关的字段，主要关注处理器中的线程和运行队列：

```go
type p struct {
	m           muintptr

	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	runnext guintptr				// 线程中下一个需要执行的Goroutine
	...
}
```

反向存储的线程维护着线程与处理器之间的关系，而 `runqhead`、`runqtail` 和 `runq` 三个字段表示处理器持有的运行队列，其中存储着待执行的 Goroutine 列表，`runnext` 中是线程下一个需要执行的 Goroutine。

`runtime.p`结构体中的状态 `status` 字段会是以下五种中的一种：

| 状态        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `_Pidle`    | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                |
| `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                       |
| `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                  |
| `_Pdead`    | 当前处理器已经不被使用                                       |

通过分析处理器 P 的状态，我们能够对处理器的工作过程有一些简单理解，例如处理器在执行用户代码时会处于 `_Prunning` 状态，在当前线程执行 I/O 操作时会陷入 `_Psyscall` 状态。

### 1.3 调度器启动

运行时通过 `runtime.schedinit`初始化调度器：

```go
func schedinit() {
	_g_ := getg()
	...
	sched.maxmcount = 10000
	...
	sched.lastpoll = uint64(nanotime())
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
```

在调度器初始函数执行的过程中会将 `maxmcount` 设置成 10000，这也就是一个 Go 语言程序能够创建的最大线程数，虽然最多可以创建 10000 个线程，但是可以同时运行的线程还是由 `GOMAXPROCS` 变量控制。

我们从环境变量 `GOMAXPROCS` 获取了程序能够同时运行的最大处理器数之后就会调用 `runtime.procresize`更新程序中处理器的数量，在这时整个程序不会执行任何用户 Goroutine，调度器也会进入锁定状态，`runtime.procresize`的执行过程如下：

1. 如果全局变量 `allp` 切片中的处理器数量少于期望数量，会对切片进行扩容；
2. 使用 `new` 创建新的处理器结构体并调用 `runtime.p.init`初始化刚刚扩容的处理器；
3. 通过指针将线程 m0 和处理器 `allp[0]` 绑定到一起；
4. 调用 `runtime.p.destroy` 释放不再使用的处理器结构；
5. 通过截断改变全局变量 `allp` 的长度保证与期望处理器数量相等；
6. 将除 `allp[0]` 之外的处理器 P 全部设置成 `_Pidle` 并加入到全局的空闲队列中；

 调用 `runtime.procresize`是调度器启动的最后一步，在这一步过后调度器会完成相应数量处理器的启动，等待用户创建运行新的 Goroutine 并为 Goroutine 调度处理器资源。









