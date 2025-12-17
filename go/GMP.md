# GMP调度模型

## 1、基础概念

### 1.1 进程、线程、协程

进程是由一段静态代码经过编译为二进制可执行程序被加载到内存中经 CPU 按顺序执行的一个程序。

线程是进程中的一个执行流程，多个线程之间共享代码段、数据段、打开的文件等资源，但每个线程各自都有一套独立的寄存器和栈，以此来保证其控制流的独立性

协程是用户态线程，是用户程序对对线程概念的二次封装，和线程为多对一关系，在逻辑意义上属于更细粒度的调度单元，其`调度过程由用户态闭环完成`，无需内核介入

go 语言中的协程是 goroutine，能够与 M、P动态结合，体积只有几 KB，栈空间支持动态扩缩。



### 1.2 GMP架构



![GMP架构图](../images/GMP架构图.png)

**G**

goroutine，是 golang 中对协程的抽象；有自己的运行栈、生命周期状态、以及执行的任务函数（用户通过 go func 指定）；需要绑定在 m 上执行。



**M**

machine，是 golang 中对线程的抽象；跟 p 结合，它的运行目标始终在 g0 和 g 之间，当运行 g0 时执行的是 m 的调度流程，负责寻找合适的“任务”，也就是 g；当运行 g 时，执行的是 m 获取到的”任务“，也就是用户通过 go func 启动的 goroutine，不断交替运行任务：寻找任务（执行g0）；执行任务（执行g）

m 维护在一个 m 列表中，操作系统分配给当前 go 程序的线程数，m 的数目可以配置，空闲的 m 会回收或睡眠，有 m 阻塞时，会创建新的 m，因此其数目是动态的。



**P**

processor，是 golang 中的调度器，m 需要与 p 绑定后，才会进入到 gmp 调度模式当中；因此 p 的数量决定了 g 最大并行数量（通过 GOMAXPROCS 进行设定），p 是 g 的存储容器，其自带一个本地 g 队列，承载着等待被调度的 g

p 维护在一个 p 列表中，最大数目由 GOMAXPROCS 决定，可通过环境变量或 runtime 设置



如果将 GMP 看作一个任务管理系统：G 是任务，M 是运行任务的 work，P 是任务系统的中枢，跟 m 绑定开始寻找并运行 g，同时还是存储任务的容器

承载 G 的容器分为两部分：

- `p 的本地队列 lrq（local run queue）`：这是每个 p 私有的 g 队列，通常由 p 自行访问，并发竞争情况较少，因此设计为无锁化结构，通过 CAS（compare-and-swap）操作访问

- `全局队列 grq（global run queue）`：是全局调度模块 schedt 中的全局共享 g 队列，作为当某个 lrq 不满足条件时的备用容器，因为不同的 m 都可能访问 grq，因此并发竞争比较激烈，访问前需要加全局锁

将 g 放入容器和取出容器的流程设计：

- put g：当某个 g 中通过 go func(){...} 操作创建子 g 时，会先尝试将子 g 添加到当前所在 p 的 lrq 中（无锁化）；如果 lrq 满了，则会将 g 追加到 grq 中（全局锁）。

- get g：gmp 调度流程中，m 和 p 结合后，运行的 g0 会不断寻找合适的 g 用于执行，此时会采取`“负载均衡”`的思路，遵循如下实施步骤：

  - 优先从当前 p 的 lrq 中获取 g（无锁化-CAS）,在 g0 每经过 61 次调度循环后，下一次在处理 lrq 前优先处理一次 grq，避免因 lrq 过于忙碌而致使 grq 陷入饥荒状态

  - 从全局的 grq 中获取 g（全局锁）

  - 取 io 就绪的 g（netpoll 机制）

  - 从其他 p 的 lrq 中窃取 g（无锁化-CAS）




全局唯一的一个 m0，就是进程的主线程，拉起 main 函数这个 groutine。

每个 m 都有一个 g0，g0 仅负责调度 g 到 m 上运行，完成一个 g 到另一个 g 的切换。

**M 的自旋状态**：创建新的 G 时，运行的 G 会尝试唤醒其他空闲的 M 绑定 P 去执行，如果 G2 唤醒了M2，M2 绑定了一个 P2，会先运行 M2 的 G0，这时 M2 没有从 P2 的本地队列中找到 G，会进入自旋状态（spinning），自旋状态的 M2 会尝试从全局空闲线程队列里面获取 G，放到 P2 本地队列去执行，获取的数量满足公式：n = min(len(globrunqsize)/GOMAXPROCS + 1, len(localrunsize/2))，含义是每个P应该从全局队列承担的 G 数量，为了提高效率，不能太多，要给其他 P 留点；

**G 发生系统调用时**：如果 G 发生系统调度进入阻塞，其所在的 M 也会阻塞，因为会进入内核状态等待系统资源，和 M 绑定的 P 会寻找空闲的 M 执行，这是为了提高效率，不能让 P 本地队列的 G 因所在 M 进入阻塞状态而无法执行；需要说明的是，如果是 M 上的 G 进入 Channel 阻塞，则该 M 不会一起进入阻塞，因为 Channel 数据传输涉及内存拷贝，不涉及系统资源等待；

**G 退出系统调用时**：如果刚才进入系统调用的 G2 解除了阻塞，其所在的 M1 会寻找 P 去执行，优先找原来的 P，发现没有找到，则其上的 G2 会进入全局队列，等其他 M 获取执行，M1 进入空闲队列；





## 2、go程序入口

查找程序的执行入口：

https://zhuanlan.zhihu.com/p/654331129



一个 go 程序真正的入口在 runtime包的 src/runtime/rt0_linux_amd64.s (对 AMD64 架构上的 Linux，其他架构在不同的文件下)

```c
TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)
```

跳转到了 src/runtime/asm_amd64.s 包的 _rt0_amd64 函数：

```c
// _rt0_amd64 is common startup code for most amd64 systems when using
// internal linking. This is the entry point for the program from the
// kernel for an ordinary -buildmode=exe program. The stack holds the
// number of arguments and the C-style argv.
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)
```

在 asm_amd64.s 中有 runtime·rt0_go 函数的实现：

```c
TEXT runtime·rt0_go(SB),NOSPLIT|NOFRAME|TOPFRAME,$0
	// copy arguments forward on an even stack
	MOVQ	DI, AX		// argc，保存命令行参数
	MOVQ	SI, BX		// argv
	......

	// create istack out of the given (operating system) stack.
	// _cgo_init may update stackguard.
  // 创建初始 goroutine（g0）的栈结构，设置栈边界
	MOVQ	$runtime·g0(SB), DI
	LEAQ	(-64*1024)(SP), BX
	MOVQ	BX, g_stackguard0(DI)
	MOVQ	BX, g_stackguard1(DI)
	MOVQ	BX, (g_stack+stack_lo)(DI)
	MOVQ	SP, (g_stack+stack_hi)(DI)

......
ok:
	// 建立 goroutine（g0）和 machine（m0）的双向关联，g0 是初始 goroutine，m0 是初始 OS 线程，使用操作系统提	 // 供的初始线程栈（通常是主线程栈）
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)

.......
  // 解析命令行参数，操作系统相关初始化，调度器初始化
	MOVL	24(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	32(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)
	
  // 创建一个新的 goroutine 来运行 runtime.main，runtime.main 最终会调用用户的 main.main()
	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	CALL	runtime·newproc(SB)
	POPQ	AX

	// start this M，启动调度
	CALL	runtime·mstart(SB)

	CALL	runtime·abort(SB)	// mstart should never return
	RET
```

这个汇编函数主要做了四件事：

- 建立 goroutine（g0）和 machine（m0）的双向关联，g0 是初始 goroutine，m0 是初始 OS 线程，它使用操作系统提供的初始线程栈（通常是主线程栈）

- 通过 runtime 中的 osinit、schedinit 等函数对 GMP 初始化。其中 osinit 主要是获取 CPU 数量，页大小等一些操作系统初始化工作

- 创建一个新的 goroutine 来运行 runtime.main，runtime.main 最终会调用用户的 main.main()，操作系统加载的时候只创建好了主线程，协程还是得用户态的 golang 自己管理，在这里 golang 创建出了自己的第一个协程

- 调用 runtime·mstart 开启调度



### 2.1 m0 和 g0 初始化过程

m0 和 g0 的初始化过程：

```shell
// 建立 goroutine（g0）和 machine（m0）的双向关联，g0 是初始 goroutine，m0 是初始 OS 线程，使用操作系统提	 // 供的初始线程栈（通常是主线程栈）
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX   // 将全局变量 g0 的地址加载到 CX 寄存器，此时 CX 指向 g0 结构体
	MOVQ	CX, g(BX)						 //g(BX)宏展开为 0(r)(TLS*1)，即将 CX (g0 的地址)存储到 TLS 的第一个槽位
	LEAQ	runtime·m0(SB), AX   // 将全局变量 m0 的地址加载到 AX 寄存器

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)
```

get_tls(BX):将 TLS 基地址加载到 BX 寄存器，TLS 代表当前线程的线程本地存储区域的基地址。

```shell
// get_tls(r) 在 go_tls.h 中定义

#ifdef GOARCH_amd64
#define	get_tls(r)	MOVQ TLS, r
#define	g(r)	0(r)(TLS*1)
#endif
```

将全局变量 g0 的地址加载到 CX 寄存器，此时 CX 指向 g0 结构体

```shell
LEAQ	runtime·g0(SB), CX
```

g(BX)函数展开为 0(r)(TLS*1)，即将 CX (g0 的地址)存储到 TLS 的第一个槽位,从此刻起，getg() 函数就能工作了

```shell
MOVQ	CX, g(BX)

--------------------
// getg()的定义如下
// getg returns the pointer to the current g.
// The compiler rewrites calls to this function into instructions
// that fetch the g directly (from TLS or from the dedicated register).
func getg() *g
```

将全局变量 m0 的地址加载到 AX 寄存器

```shell
LEAQ	runtime·m0(SB), AX
```

此时的寄存器状态：

```shell
BX = TLS 基地址
CX = &g0
AX = &m0
```

m->g0 = g0

```shell
MOVQ	CX, m_g0(AX)
```

g0->m = m0

```shell
MOVQ	AX, g_m(CX)
```

完成 m0 和 g0 的相互引用，通过 TLS，能够快速访问 GMP 的调度链

┌─────────────────────────────────────┐
│  当前线程（M）                        │
├─────────────────────────────────────┤
│  TLS[0] → 当前 G                     │
│           ↓                          │
│           g.m → 当前 M               │
│                 ↓                    │
│                 m.p → 当前 P         │
│                       ↓              │
│                       p.runq → G队列 │
└─────────────────────────────────────┘



![m0&g0](../images/m0&g0.png)



使用 TLS 的好处：

1、通过 TLS，getg() 可以在几个 CPU 周期内完成，无需函数调用开销。

2、支持多线程每个操作系统线程（M）都有自己的 TLS：M1 的 TLS[0] 指向它当前运行的 G；M2 的 TLS[0] 指向它当前运行的 G。互不干扰，无需加锁。

3、架构无关的抽象：不同操作系统和 CPU 架构的 TLS 实现不同，通过 get_tls 宏和 g(r) 宏抽象了这些差异。



### 2.2 schedinit()调度系统初始化



schedinit() 函数对调度系统进行初始化，位于 runtime/proc.go 下，它负责初始化整个 Go 运行时环境

```go
var (
	m0           m	// m0 作为全局变量静态分配
	g0           g  // m0 和 g0 是全局变量，在程序启动时就已经存在于数据段（.data 或 .bss 段）
	mcache0      *mcache
	raceprocctx0 uintptr
	raceFiniLock mutex
)


// The bootstrap sequence is: 启动顺序
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
func schedinit() {
	lockInit(&sched.lock, lockRankSched)
	lockInit(&sched.deferlock, lockRankDefer)
	lockInit(&sched.sudoglock, lockRankSudog)
	lockInit(&deadlock, lockRankDeadlock)
	lockInit(&paniclk, lockRankPanic)
	lockInit(&allglock, lockRankAllg)
	lockInit(&allpLock, lockRankAllp)
	lockInit(&reflectOffs.lock, lockRankReflectOffs)
	traceLockInit()
  ......

	gp := getg()
	
  // 设置最大线程数为 10000，即 M 的最大数目
	sched.maxmcount = 10000
	crashFD.Store(^uintptr(0))

	// The world starts stopped.
	worldStopped()

	ticks.init() // run as early as possible,用于性能分析和调度决策，需尽早运行
	stackinit()		// 栈初始化
	randinit() // must run before mallocinit, alginit, mcommoninit，随机数初始化
	mallocinit() // 内存分配器初始化
	cpuinit(godebug) // must run before alginit
	alginit()        // maps, hash, rand must not be used before this call，算法初始化（map、hash等）
	mcommoninit(gp.m, -1)  // 初始化当前的操作系统线程（M0，即主线程）。
	stkobjinit()    // must run before GC starts，栈对象初始化


  goargs()           // 处理命令行参数
  goenvs()           // 处理环境变量
  gcinit()           // GC初始化

	// Allocate stack space that can be used when crashing due to bad stack
	// conditions, e.g. morestack on g0.
  // 预分配 16KB 栈空间用于崩溃处理，如：当栈溢出或其他严重错误时，使用这个栈来输出错误信息
  // stackguard 栈保护边界，用于检测栈溢出
	gcrash.stack = stackalloc(16384)
	gcrash.stackguard0 = gcrash.stack.lo + 1000
	gcrash.stackguard1 = gcrash.stack.lo + 1000

	lock(&sched.lock)
  // GOMAXPROCS 变量设置，如果未设置，使用 CPU 核心数
	var procs int32
	if n, ok := strconv.Atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
		sched.customGOMAXPROCS = true
	} else {
		// Use numCPUStartup for initial GOMAXPROCS for two reasons:
		//
		// 1. We just computed it in osinit, recomputing is (minorly) wasteful.
		//
		// 2. More importantly, if debug.containermaxprocs == 0 &&
		//    debug.updatemaxprocs == 0, we want to guarantee that
		//    runtime.GOMAXPROCS(0) always equals runtime.NumCPU (which is
		//    just numCPUStartup).
		procs = defaultGOMAXPROCS(numCPUStartup)
	}
  // 调用 procresize 调整 P（处理器）
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
	unlock(&sched.lock)

	// World is effectively started now, as P's can run. 标记运行时已启动，P 可以开始运行
	worldStarted()
	......
}
```

P 的数量取决于当前 cpu 的数量，或者是 runtime.GOMAXPROCS 的配置。

几个重点：

1. mallocinit()：内存分配器，初始化 mcache（每个 P 的本地缓存），分三级mcache、mcentral（中心缓存）、mheap（全局堆），在此之前不能进行堆内存分配
2. mcommoninit(gp.m, -1)：完善初始化当前 M（M0，主线程），将 M 加入全局 M 列表。前面 M0 已经存在：作为全局变量静态分配且在 rt0_go 汇编中与 g0 建立了关联，g0 和 m0 不是通过 malloc 动态分配的，而是编译时就分配好的静态内存，解决了“鸡生蛋”问题（在内存分配器初始化之前，我们需要一个 M 和 G 来运行初始化代码）
3. gcinit()：初始化垃圾回收器的数据结构、工作线程、写屏障等
4. GOMAXPROCS 初始化
5. procresize(procs): 调整 P 的数目和 P 处理器的初始化。



重点看下`procresize(procs)`:

```go
var allp []*p

// Change number of processors.
//
// sched.lock must be held, and the world must be stopped.
//
// gcworkbufs must not be being modified by either the GC or the write barrier
// code, so the GC must not be running if the number of Ps actually changes.
//
// Returns list of Ps with local work, they need to be scheduled by the caller.
func procresize(nprocs int32) *p {
	assertLockHeld(&sched.lock)
	assertWorldStopped()

	......
  old := gomaxprocs
	// Grow allp if necessary.
	if nprocs > int32(len(allp)) {
		lock(&allpLock)
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
			nallp := make([]*p, nprocs)
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}
		unlock(&allpLock)
	}

	// initialize new P's
	for i := old; i < nprocs; i++ {
		pp := allp[i]
		if pp == nil {
			pp = new(p)
		}
		pp.init(i)  // 设置id，初始化mcahe，设置状态为_Pgcstop
		atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
	}

	gp := getg()
	if gp.m.p != 0 && gp.m.p.ptr().id < nprocs {
		// continue to use the current P
		gp.m.p.ptr().status = _Prunning
		gp.m.p.ptr().mcache.prepareForSweep()
	} else {
		// release the current P and acquire allp[0].
		//
		// We must do this before destroying our current P
		// because p.destroy itself has write barriers, so we
		// need to do that from a valid P.
		if gp.m.p != 0 {
			gp.m.p.ptr().m = 0
		}
		gp.m.p = 0
		pp := allp[0]		// 取第一个P
		pp.m = 0
		pp.status = _Pidle
		acquirep(pp)
	}

	// g.m.p is now set, so we no longer need mcache0 for bootstrapping.
	mcache0 = nil

	// release resources from unused P's
	for i := nprocs; i < old; i++ {
		pp := allp[i]
		pp.destroy()
		// can't free P itself because it can be referenced by an M in syscall
	}

	// Trim allp.
	if int32(len(allp)) != nprocs {
		lock(&allpLock)
		allp = allp[:nprocs]
		idlepMask = idlepMask[:maskWords]
		timerpMask = timerpMask[:maskWords]
		unlock(&allpLock)
	}
	
  // id 大于nprocs的 P 都不返回了
	var runnablePs *p
	for i := nprocs - 1; i >= 0; i-- {
		pp := allp[i]
		if gp.m.p.ptr() == pp {
			continue
		}
		pp.status = _Pidle
		if runqempty(pp) {
			pidleput(pp, now)
		} else {
			pp.m.set(mget())
			pp.link.set(runnablePs)
			runnablePs = pp
		}
	}
	stealOrder.reset(uint32(nprocs))  // 更新窃取其他 P 本地队列的 order
	var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))

	return runnablePs  // 返回可用的 P 链表头指针
}

func wirep(pp *p) {
	gp := getg()

	if gp.m.p != 0 {
		systemstack(func() {
			throw("wirep: already in go")
		})
	}
	if pp.m != 0 || pp.status != _Pidle {
		systemstack(func() {
			id := int64(0)
			if pp.m != 0 {
				id = pp.m.ptr().id
			}
			print("wirep: p->m=", pp.m, "(", id, ") p->status=", pp.status, "\n")
			throw("wirep: invalid p state")
		})
	}
	gp.m.p.set(pp)  // 当前 m 关联 p
	pp.m.set(gp.m)  // p 关联当前 m
	pp.status = _Prunning
}

// 乐观无锁+双重检查判队列是否为空，for循环不会一直进行下去:1、时间窗口极窄，三条原子操作只有几纳秒；2、runqtail有限次变化（p的本地队列长度有限，256），单个 P 每秒最多处理 数百万次 队列操作，但在 5 纳秒的窗口内，最多只能发生 1-2 次 修改
// runqempty reports whether pp has no Gs on its local run queue.
// It never returns true spuriously.
func runqempty(pp *p) bool {
	// Defend against a race where 1) pp has G1 in runqnext but runqhead == runqtail,
	// 2) runqput on pp kicks G1 to the runq, 3) runqget on pp empties runqnext.
	// Simply observing that runqhead == runqtail and then observing that runqnext == nil
	// does not mean the queue is empty.
	for {
		head := atomic.Load(&pp.runqhead)
		tail := atomic.Load(&pp.runqtail)
		runnext := atomic.Loaduintptr((*uintptr)(unsafe.Pointer(&pp.runnext)))
		if tail == atomic.Load(&pp.runqtail) {
			return head == tail && runnext == 0
		}
	}
}

func pidleput(pp *p, now int64) int64 {
	assertLockHeld(&sched.lock)

	if !runqempty(pp) {
		throw("pidleput: P has non-empty run queue")
	}
	if now == 0 {
		now = nanotime()
	}
	if pp.timers.len.Load() == 0 {
		timerpMask.clear(pp.id)
	}
	idlepMask.set(pp.id)
	pp.link = sched.pidle
	sched.pidle.set(pp)
	sched.npidle.Add(1)
	if !pp.limiterEvent.start(limiterEventIdle, now) {
		throw("must be able to track idle limiter event")
	}
	return now
}
```

执行完 acquirep(pp) 后此时 m0 与 g0，p0 的关系就建立起来了。

![Clipboard_Screenshot_1765446771](../images/g0m0p0.png)

此后，构建 p 的空闲单向链表，更新 sched 结构体的空闲 p 链表头节点指针和空闲的 p 数目。其中 sched.pidle：全局唯一的链表头指针，存储在调度器结构体中；p.link：每个 P 结构体内部的字段，用于串联成链表，使用头插法（LIFO - 后进先出），这样头插法和头取出都是 O(1) 时间复杂度，且不需要尾指针。



![sched_pidle](../images/sched_pidle.png)



### 2.3 创建 main goroutine

```c
	// 创建一个新的 goroutine 来运行 runtime.main，runtime.main 最终会调用用户的 main.main()
	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// runtime·mainPC 是一个全局变量，存储了 runtime.main 函数的地址
	PUSHQ	AX												// 将 mainPC 压栈作为参数
	CALL	runtime·newproc(SB)
	POPQ	AX

    
// mainPC is a function value for runtime.main, to be passed to newproc.
// The reference to runtime.main is made via ABIInternal, since the
// actual function (not the ABI0 wrapper) is needed by newproc.
DATA	runtime·mainPC+0(SB)/8,$runtime·main<ABIInternal>(SB)
GLOBL	runtime·mainPC(SB),RODATA,$8
    
等价于：
// 只读全局变量，存储 runtime.main 函数的地址
var mainPC = (*funcval)(unsafe.Pointer(&runtime.main))
```



调用 newproc()，当写 go func() 时，编译器会将其转换为对 newproc 的调用。

```go
// Create a new g running fn.
// Put it on the queue of g's waiting to run.
// The compiler turns a go statement into a call to this.
func newproc(fn *funcval) {
	gp := getg()
	pc := sys.GetCallerPC()
	systemstack(func() {
		newg := newproc1(fn, gp, pc, false, waitReasonZero)

		pp := getg().m.p.ptr()
		runqput(pp, newg, true)

		if mainStarted {
			wakep()
		}
	})
}
```

#### 2.3.1 systemstack切换g0栈

systemstack()，切换系统栈，保存当前 goroutine 的上下文，如果已经在 g0 或 gsignal 上，则直接执行，否则切换到 g0 栈，执行传入的函数，执行完毕后恢复原来的 goroutine 上下文。

```go
// systemstack runs fn on a system stack.
// If systemstack is called from the per-OS-thread (g0) stack, or
// if systemstack is called from the signal handling (gsignal) stack,
// systemstack calls fn directly and returns.
// Otherwise, systemstack is being called from the limited stack
// of an ordinary goroutine. In this case, systemstack switches
// to the per-OS-thread stack, calls fn, and switches back.
// It is common to use a func literal as the argument, in order
// to share inputs and outputs with the code around the call
// to system stack:
//
//	... set up y ...
//	systemstack(func() {
//		x = bigcall(y)
//	})
//	... use x ...
//
//go:noescape
func systemstack(fn func())
```

系统栈（g0 栈）上执行，空间更大，不会栈增长，避免在用户 goroutine 栈上执行可能因为栈增长的操作，导致的递归问题；不会触发 GC 扫描（g0 的栈不需要扫描），安全地操作 schd 调度器数据结构，不会被抢占，为 runtime 提供了一个稳定、安全、高效的执行环境。



#### 2.3.2 新建一个 goroutine

```go
// Create a new g in state _Grunnable (or _Gwaiting if parked is true), starting at fn.
// callerpc is the address of the go statement that created this. The caller is responsible
// for adding the new g to the scheduler. If parked is true, waitreason must be non-zero.
func newproc1(fn *funcval, callergp *g, callerpc uintptr, parked bool, waitreason waitReason) *g {
  // 防止创建一个没有执行函数的 goroutine，var f func() 和 go f() 等会触发 fatal error
	if fn == nil {
		fatal("go of nil func value")
	}

	mp := acquirem() // disable preemption because we hold M and P in local vars.
	pp := mp.p.ptr()
	newg := gfget(pp)
	if newg == nil {
		newg = malg(stackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
	if newg.stack.hi == 0 {
		throw("newproc1: newg missing stack")
	}

	if readgstatus(newg) != _Gdead {
		throw("newproc1: new g is not Gdead")
	}
	// g结构体的各个字段值的赋值
	totalSize := uintptr(4*goarch.PtrSize + sys.MinFrameSize)
	totalSize = alignUp(totalSize, sys.StackAlign)
	sp := newg.stack.hi - totalSize
	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.stktopsp = sp
	newg.sched.pc = abi.FuncPCABI0(goexit) + sys.PCQuantum
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	gostartcallfn(&newg.sched, fn)
	newg.parentGoid = callergp.goid
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn

	var status uint32 = _Grunnable
	if parked {
		status = _Gwaiting
		newg.waitreason = waitreason
	}
	newg.goid = pp.goidcache
	casgstatus(newg, _Gdead, status)
	
	
	releasem(mp)

	return newg
}
```

参数说明：

- fn *funcval：要执行的函数（包含函数指针和闭包变量）
- gp *g：父 goroutine（当前 goroutine）
- pc uintptr：调用者的 PC（创建位置），即 go 语句的位置
- false：parked 参数，表示新 goroutine 不是处于等待状态
- waitReasonZero：等待原因（这里为0，因为不等待）



Part 1:

禁止当前 m 被抢占，绑定 m 和 p，从 p 的本地队列中获取空闲的 g，若没有则从全局队列中批量获取空闲的 g，仍然没有则创建一个 g，并分配栈空间，标记状态为 `_Gdead`，并添加到 allgs 中（全局变量，维持着所有的 g 数组，主要用来统计）

```go
mp := acquirem() // disable preemption because we hold M and P in local vars.
pp := mp.p.ptr()
newg := gfget(pp)
if newg == nil {
  newg = malg(stackMin)
  casgstatus(newg, _Gidle, _Gdead) // 切换状态
  allgadd(newg)	// 添加到全局数组 allgs    []*g
}
if newg.stack.hi == 0 {
  throw("newproc1: newg missing stack")
}

if readgstatus(newg) != _Gdead {
  throw("newproc1: new g is not Gdead")
}

//go:nosplit
func acquirem() *m {
	gp := getg()
	gp.m.locks++  // 绑定 m 与 p，禁止抢占
	return gp.m
}
```



gfget 获取 g：

- 优先从 pp.gFree.pop() 获取空闲的 g

- retry 循环中不断从获取 sched.gFree 全局队列中获取空闲的 g，获取时优先获取那些已经分配了栈空间的 g
- 为 g 分配栈空间，若已分配则判断其栈空间大小是否与起始栈空间大小一致，不一致则重新分配。

```go
// Get from gfree list.
// If local list is empty, grab a batch from global list.
func gfget(pp *p) *g {
retry:
	if pp.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
		lock(&sched.gFree.lock)
		// Move a batch of free Gs to the P.
		for pp.gFree.size < 32 {
			// 优先获取那些已经分配了栈空间的 g
			gp := sched.gFree.stack.pop()
			if gp == nil {
				gp = sched.gFree.noStack.pop()
				if gp == nil {
					break
				}
			}
			pp.gFree.push(gp)
		}
		unlock(&sched.gFree.lock)
		goto retry
	}
	gp := pp.gFree.pop()
	if gp == nil {
		return nil
	}
  // 如果 g 已经分配了栈空间，并且栈空间大小不等于起始栈空间大小，需要重新分配，确保每个新使用的 g的栈空间都是从 startingStackSize（2KB，该值可设置调整） 开始
	if gp.stack.lo != 0 && gp.stack.hi-gp.stack.lo != uintptr(startingStackSize) {
		systemstack(func() {
			stackfree(gp.stack)
			gp.stack.lo = 0
			gp.stack.hi = 0
			gp.stackguard0 = 0
		})
	}
	if gp.stack.lo == 0 {
		// Stack was deallocated in gfput or just above. Allocate a new one.
		systemstack(func() {
			gp.stack = stackalloc(startingStackSize)
		})
		gp.stackguard0 = gp.stack.lo + stackGuard
	}
	return gp
}
```

malg(stackMin)，p 本地队列和全局队列都没有空闲的 g 时，则新建 g

```go
// Allocate a new g, with a stack big enough for stacksize bytes.
func malg(stacksize int32) *g {
	newg := new(g)
	if stacksize >= 0 {
		stacksize = round2(stackSystem + stacksize)  // 按2的指数向上取整
		systemstack(func() {
			newg.stack = stackalloc(uint32(stacksize))
		})
		newg.stackguard0 = newg.stack.lo + stackGuard
		newg.stackguard1 = ^uintptr(0)
		// Clear the bottom word of the stack. We record g
		// there on gsignal stack during VDSO on ARM and ARM64.
		*(*uintptr)(unsafe.Pointer(newg.stack.lo)) = 0
	}
	return newg
}
```

Part2: g结构体的各个字段值的赋值



#### 2.3.3 runqput

runqput 函数负责将一个 goroutine (gp) 放入 P (pp) 的本地可运行队列中。有两种放置模式：
普通模式：将 goroutine 放到队列尾部
next 模式：将 goroutine 放到 pp.runnext 槽位（优先执行）

```go
// runqput tries to put g on the local runnable queue.
// If next is false, runqput adds g to the tail of the runnable queue.
// If next is true, runqput puts g in the pp.runnext slot.
// If the run queue is full, runnext puts g on the global queue.
// Executed only by the owner P.
func runqput(pp *p, gp *g, next bool) {
	if !haveSysmon && next {
		// A runnext goroutine shares the same time slice as the
		// current goroutine (inheritTime from runqget). To prevent a
		// ping-pong pair of goroutines from starving all others, we
		// depend on sysmon to preempt "long-running goroutines". That
		// is, any set of goroutines sharing the same time slice.
		//
		// If there is no sysmon, we must avoid runnext entirely or
		// risk starvation.
		next = false
	}

	if next {
	retryNext:
		oldnext := pp.runnext
		if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry:
	h := atomic.LoadAcq(&pp.runqhead) // load-acquire, synchronize with consumers
	t := pp.runqtail
	if t-h < uint32(len(pp.runq)) {
		pp.runq[t%uint32(len(pp.runq))].set(gp)
		atomic.StoreRel(&pp.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	if runqputslow(pp, gp, h, t) {
		return
	}
	goto retry
}
```

如果本地队列已经放满了，那么就往全局队列中放。即调用 `runqputslow`，这里会将 p 一半的本地队列和当前 goroutine 批量转移到全局队列，批量操作，操作全局队列需要加锁。全局队列按照链表串联起来，无限容量，有锁，起一个兜底作用。



### 2.4 启动 GMP 调度循环



汇编：

```c
// start this M，启动调度
	CALL	runtime·mstart(SB)
    
TEXT runtime·mstart(SB),NOSPLIT|TOPFRAME|NOFRAME,$0
	CALL	runtime·mstart0(SB)
	RET // not reached
```

在 runtime 中调用 mstart0，mstart0 中调用 mstart1，它的实现在 proc.go 中：

```go
func mstart1() {
	gp := getg()

	if gp != gp.m.g0 {
		throw("bad runtime·mstart")
	}
  // 保存返回点，执行完后需要回到调用方的下一条语句上继续执行
	gp.sched.g = guintptr(unsafe.Pointer(gp))
	gp.sched.pc = sys.GetCallerPC()
	gp.sched.sp = sys.GetCallerSP()	
	
  // 如果当前 m 不等于 m0，则需要绑定一个 p
	if gp.m != &m0 {
		acquirep(gp.m.nextp.ptr())
		gp.m.nextp = 0
	}
	schedule()
}
```

schedule()：开启GMP调度，不在返回，循环执行下去

```go
// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
func schedule() {
	mp := getg().m

	// 省略其他代码......

top:
	pp := mp.p.ptr()
	pp.preempt = false

	// Safety check: if we are spinning, the run queue should be empty.
	// Check this before calling checkTimers, as that might call
	// goready to put a ready goroutine on the local run queue.
	if mp.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
		throw("schedule: spinning with local work")
	}
	// 找到一个可执行的goroutine
	gp, inheritTime, tryWakeP := findRunnable() // blocks until work is available

	// 解除M自旋
	if mp.spinning {
		resetspinning()
	}

	// If about to schedule a not-normal goroutine (a GCworker or tracereader),
	// wake a P if there is one.
	if tryWakeP {
		wakep()
	}
	// 执行 goroutine
	execute(gp, inheritTime)
}
```

#### 2.4.1 findRunnable()

函数返回值，分别代表：

- gp：找到的 goroutine
- inheritTime：是否继承时间片
- tryWakeP：是否需要唤醒其他 P

findRunnable() 查找顺序：

1. 清理和初始化
   ├─ 清除 allp 快照
   ├─ 检查 GC 等待 → gcstopm() → goto top
   ├─ 检查安全点函数 → runSafePointFn()
   └─ 检查定时器

2. 特殊 goroutine（优先级最高）
   ├─ Trace reader (tryWakeP=true)
   └─ GC worker (tryWakeP=true)

3. 全局队列公平性检查（每 61 次）
   └─ schedtick % 61 == 0 → globrunqget()

4. 唤醒特殊 goroutine
   ├─ Finalizer goroutine
   └─ GC cleanup goroutine

5. 本地队列（快速路径）
   └─ runqget(pp) → 优先检查 runnext

6. 全局队列（批量获取）
   └─ globrunqgetbatch() → 获取一半到本地队列

7. 网络轮询（非阻塞）
   └─ netpoll(0) → 检查就绪的网络 I/O

8. 工作窃取
   └─ stealWork() → 从其他 P 窃取

9. 空闲 GC 标记
   └─ 如果有 GC 工作，运行空闲标记

10. 准备休眠
    ├─ 释放 P
    ├─ 再次检查所有队列
    ├─ 网络轮询（阻塞）
    └─ stopm() → 休眠 M





## 其他

### 1 补充

#### 1.1 g 栈空间分布：

![sched_pidle](../images/goroutine栈分布.png)

栈大小计算：
`stack.hi - stack.lo = 2048 字节` (初始栈大小，即 stackMin)

如果 `SP < stackguard0`:
    触发栈增长（调用 runtime.morestack）
    分配更大的栈
    复制旧栈内容到新栈
    继续执行

除了栈溢出检测，stackguard0 还用于实现抢占。

```go
gp.stackguard0 = stackPreempt  // 0xfffffade（一个很大的值）
```

下次函数调用时：
`SP (正常值) < stackPreempt (超大值)` → 触发 morestack
morestack 检测到 `stackguard0 == stackPreempt`
执行抢占逻辑而非栈增长



#### 1.2 goroutine 状态转换

![goroutine状态转换图](../images/goroutine状态转换图.png)

#### 1.3 抢占禁用p的next槽位

```go
// Goroutine A
for {
    ch <- 1  // 发送后立即阻塞，没有函数调用
}

// Goroutine B  
for {
    <-ch     // 接收后立即阻塞，没有函数调用
}

```

上述例子中会出现 A B 互相唤醒，互相执行，继承 p 的时间片一直执行下去，导致 p 上的其他 g 处于饥饿状态，时间片一直会被刷新，是因为从 runqget 中获取 g 时，如果 next 有值，则优先返回，并且 `inheritTime = true`,标志继承时间片。

```go
func runqget(pp *p) (gp *g, inheritTime bool) {
    // 如果 runnext 有值，优先返回它
    if next != 0 && pp.runnext.cas(next, 0) {
        return next.ptr(), true  // ← inheritTime = true
    }
    
    // 否则从普通队列取
    // ...
    return gp, false  // ← inheritTime = false
}
```

在调度器执行时，会判断 `inheritTime` 标识是否为 true，为 true 则调度器计数器不增加。

```GO
func execute(gp *g, inheritTime bool) {
    mp := getg().m
    // ... 省略其他代码 ...
    if !inheritTime {
        mp.p.ptr().schedtick++  // ← 只有 inheritTime=false 时才增加 schedtick
    }
    // ... 省略其他代码 ...
    gogo(&gp.sched)  // 开始执行 goroutine
}
```

因此，在 runqput 中如果没有系统监控主动抢占时，需要禁止 next。

系统监控抢占是通过 sysmon 检查 pp.schedtick 是否变化来判断，如果 schedtick 长时间不变，说明时间片用完，触发抢占。

```go
// sysmon 每 10ms 检查一次
func retake(now int64) uint32 {
    // ...
    if s == _Prunning || s == _Psyscall {
        // Preempt G if it's running on the same schedtick for
        // too long. This could be from a single long-running
        // goroutine or a sequence of goroutines run via
        // runnext, which share a single schedtick time slice.
        t := int64(pp.schedtick)
        if int64(pd.schedtick) != t {
            pd.schedtick = uint32(t)
            pd.schedwhen = now
        } else if pd.schedwhen+forcePreemptNS <= now {
            preemptone(pp)  // ← 抢占
        }
    }
    // ...
}
```



#### 1.4 完整启动流程

时间线：
T0: 程序启动
    ↓
T1: 汇编入口（asm_amd64.s）
    ├─ mainStarted = false（默认值）
    ├─ newproc(runtime.main)  <-创建 main goroutine
    │   ├─ runqput(P0, main_g, true)  <-放入 runnext
    │   └─ if mainStarted { wakep() }  <-不执行！
    └─ mstart()  <-启动 M0
    
T2: M0 进入调度循环
    ├─ mstart0()
    ├─ mstart1()
    └─ schedule()  <-永不返回
    
T3: schedule() 查找 goroutine
    ├─ findRunnable()
    ├─ runqget(P0)  <-从 runnext 取出 main goroutine
    └─ execute(main_g, inheritTime=true)  <-继承时间片
    
T4: 执行 runtime.main
    ├─ mainStarted = true
    ├─ 启动 sysmon
    ├─ 执行 init 函数
    └─ 执行 main.main()  ← 执行用户代码
    
T5: 用户代码中创建 goroutine
    ├─ go func() { ... }
    ├─ newproc(fn)
    ├─ runqput(P0, new_g, true)
    └─ if mainStarted { wakep() }  <-唤醒 m 执行 g



### 2 GMP 模型的进化思想

在高并发场景下，大量的线程创建、使用、切换、销毁会占用大量的内存，并浪费 CPU 时间工作在非工作任务的上，导致程序并发处理请求的能力降低。

### 3 GMP 生态

go 语言中的并发工具都是以 G 为并发粒度打造的，围绕GMP构造的并发世界

- 锁 mutex、通道 channel 等都是基于 GMP 适配，在执行阻塞操作时，是阻塞在 g 的粒度，而非 m 粒度，因此这里的唤醒与阻塞都在用户态完成，无需内核介入，同时阻塞一个 g 也不会影响 m 下其他 g 的运行。

- 在网络 io 模型方面，采用 epoll 多路复用技术，epoll_wait 阻塞在 m（thread）粒度，golang 专门设计一套 netpoll 机制，使用用户态的 gopark 指令实现阻塞操作，使用非阻塞 epoll_wait 结合用户态的 goready 指令实现唤醒操作，从而将 io 行为也控制在 g 粒度。
- 内存管理，继承 TCMalloc（Thread-Caching-Malloc）思路，为每一个 p 都准备了一份私有的高速缓冲 mcahe，这部分内存分配可以使得 p 无锁化。