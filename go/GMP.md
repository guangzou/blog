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

goroutine，是 golang 中对协程的抽象；有自己的运行栈、生命周期状态、以及执行的任务函数（用户通过 go func 指定）；需要绑定在 m 上执行，



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

- put g：当某个 g 中通过 go func(){...} 操作创建子 g 时，会先尝试将子 g 添加到当前所在 p 的 lrq 中（无锁化）；如果 lrq 满了，则会将 g 追加到 grq 中（全局锁）. 此处采取的思路是`“就近原则”`

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

这个汇编函数主要做了三件事：

- 通过 runtime 中的 osinit、schedinit 等函数对 golang 运行时进行初始化，包括 GMP 的初始化，与调度逻辑。其中 osinit 主要是获取 CPU 数量，页大小等一些操作系统初始化工作。

- 创建一个主协程，并指明 runtime.main 函数是其入口函数，runtime.main 最终会调用用户的 main.main()，操作系统加载的时候只创建好了主线程，协程还是得用户态的 golang 自己管理，在这里 golang 创建出了自己的第一个协程。

- 调用 runtime·mstart 真正开启运行。



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
  // 调用 procresize 创建 P（处理器）
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
2. mcommoninit(gp.m, -1)：完善初始化当前 M（M0，主线程），将 M 加入全局 M 列表，前面 M0 已经存在：作为全局变量静态分配且在 rt0_go 汇编中与 g0 建立了关联，g0 和 m0 不是通过 malloc 动态分配的，而是编译时就分配好的静态内存，解决了“鸡生蛋”问题（在内存分配器初始化之前，我们需要一个 M 和 G 来运行初始化代码）
3. gcinit()：初始化垃圾回收器的数据结构、工作线程、写屏障等
4. GOMAXPROCS 初始化
5. procresize(procs): 调整 P 的数目和 P 处理器的初始化。



重点看下`procresize(procs)`:

```go
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
		pp.init(i)
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
		pp := allp[0]
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
	stealOrder.reset(uint32(nprocs))
	var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))

	return runnablePs
}
```





## 其他



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

为什么需要 TLS？

- Go 运行时需要快速访问当前 goroutine 的 G 结构体
- 每个操作系统线程（M）都需要知道它当前正在运行哪个 G
- TLS 提供了一种高效的机制，无需函数参数传递就能获取当前 G

```shell
// 在 go_tls.h 中定义

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



### 1、GMP 模型的进化思想

在高并发场景下，大量的线程创建、使用、切换、销毁会占用大量的内存，并浪费 CPU 时间工作在非工作任务的上，导致程序并发处理请求的能力降低。

### 2、GMP 生态

go 语言中的并发工具都是以 G 为并发粒度打造的，围绕GMP构造的并发世界

- 锁 mutex、通道 channel 等都是基于 GMP 适配，在执行阻塞操作时，是阻塞在 g 的粒度，而非 m 粒度，因此这里的唤醒与阻塞都在用户态完成，无需内核介入，同时阻塞一个 g 也不会影响 m 下其他 g 的运行。

- 在网络 io 模型方面，采用 epoll 多路复用技术，epoll_wait 阻塞在 m（thread）粒度，golang 专门设计一套 netpoll 机制，使用用户态的 gopark 指令实现阻塞操作，使用非阻塞 epoll_wait 结合用户态的 goready 指令实现唤醒操作，从而将 io 行为也控制在 g 粒度。
- 内存管理，继承 TCMalloc（Thread-Caching-Malloc）思路，为每一个 p 都准备了一份私有的高速缓冲 mcahe，这部分内存分配可以使得 p 无锁化。