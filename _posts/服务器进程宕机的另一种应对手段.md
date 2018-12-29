# 对服务器进程宕机的另一种应对手段的探索


服务器进程宕机*[注1](#Z1)*一直以来都是后台程序员的噩梦，常用的应对手段主要有这么两种：

1. 通过严格的代码规范及review机制，防止写出造成程序崩溃的代码，防患于未然。
2. 引入共享内存或将服务器做成无状态服务器，通过外部监控程序一旦发现进程宕机则马上重新拉起。

而本文要探讨的则是第三种方案：采用异常恢复机制，恢复宕机异常。


### 起因
***tomjywang(王竞原)***同学在《[服务器宕机问题总结][L1]》一文中总结了12种常见的服务器宕机问题，并分析了宕机原因，此处引用其结论：
> Linux上进程的异常退出，一般都是由信号引起的。常见的会引起宕机的信号：
1. SIGSEGV(段错误)
2. SIGBUS(内存访问错误)
3. SIGFPE(算数异常)
4. SIGABRT(中止进程)

既然进程宕机是由信号引起的，那么是否可以通过**自定义这些信号的处理程序，在处理程序中抛出异常，在外层捕获**的方式，让进程从宕机异常中恢复？


### 可行性分析
上文我们提出了一种新的应对宕机的手段，但这种方式是否可实现，还有待证实。其实很多现代高级语言都有异常恢复机制*[注2](#Z2)*，值得我们借鉴。如在go语言中，可以通过recover机制使程序从异常（go语言称为恐慌panic）中恢复。下面是一段go语言从异常中恢复的示例代码

```go
func RecoverTest() {
	defer func() {
		if e := recover(); e != nil {
			fmt.Printf("recover panic from %v\n", e)
		}
	}()

	d := 0
	fmt.Printf("%d\n", 1/d)
}
```

阅读go语言runtime包的源码，可以发现go语言正是通过自定义信号的处理方式，来实现其recover机制的。

信号安装

```go
func initsig() {
	// _NSIG is the number of signals on this operating system.
	// sigtable should describe what to do for all the possible signals.
	if len(sigtable) != _NSIG {
		print("runtime: len(sigtable)=", len(sigtable), " _NSIG=", _NSIG, "\n")
		throw("initsig")
	}

	// First call: basic setup.
	for i := int32(0); i < _NSIG; i++ {
		t := &sigtable[i]

		...

		t.flags |= _SigHandling
		setsig(i, funcPC(sighandler), true) //设置信号的处理方式
	}
}
```

信号处理

```go
// May run during STW, so write barriers are not allowed.
//go:nowritebarrier
func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
	_g_ := getg()
	c := &sigctxt{info, ctxt}

	...

	flags := int32(_SigThrow)
	if sig < uint32(len(sigtable)) {
		flags = sigtable[sig].flags
	}
	if c.sigcode() != _SI_USER && flags&_SigPanic != 0 {
		// Make it look like a call to the signal func.
		// Have to pass arguments out of band since
		// augmenting the stack frame would break
		// the unwinding code.

		...

		c.set_rip(uint64(funcPC(sigpanic))) //设置rip寄存器，令函数返回后执行流转到sigpanic函数
		return
	}

	...
}
```

sigpanic函数则会调用panic函数，当调用panic以后，程序就进入了异常状态，此时程序会沿调用堆栈一路向上返回，同时途径的defer函数都会被调用到，直到某个defer函数中调用了recover函数, 异常状态截止, 程序从异常中恢复。或者到达栈顶退出进程。(类似C++中throw一个异常，然后去catch)


### 实验

通过阅读go语言runtime包源码，我们大致了解了其recover机制的实现原理，现在可以自己动手试验一番了

```c
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <sys/syscall.h>

#include "except.h"

#define SIG_REGS(ctxt) (*((sigcontext *)&(((ucontext_t*)(ctxt))->uc_mcontext)))
#define SIG_RIP(ctxt) (SIG_REGS(ctxt).rip)
#define SIG_RSP(ctxt) (SIG_REGS(ctxt).rsp)

const Except_T signal_except = { "Signal except" };

void setsig(int sig, void (*fn)(int, siginfo_t *, void *))
{
	struct sigaction sa;
	memset(&sa, 0, sizeof(sa));
	sigemptyset(&sa.sa_mask);
	sa.sa_flags = SA_SIGINFO | SA_ONSTACK | SA_RESTART;
	sa.sa_sigaction = fn;
	if(sigaction(sig, &sa, NULL) < 0)
		printf("sigaction error\n");
}

void sigpanic()
{
	printf("sigpainc\n");
	RAISE(signal_except); //抛出异常
}

void signal_handler(int sig, siginfo_t *info, void *ctxt)
{
	uintptr_t rip = SIG_RIP(ctxt);
	uintptr_t *sp = (uintptr_t *)SIG_RSP(ctxt);

	*--sp = rip; //压入retaddr，实际因为sigpanic中会抛出异常，所以永远不会返回该地址
	SIG_RSP(ctxt) = (uintptr_t)sp; //设置RSP寄存器
	SIG_RIP(ctxt) = (uintptr_t)sigpanic; //设置RIP寄存器，令函数返回后执行流转到sigpanic函数入口处

	printf("signal %d, rip:%lu\n", sig, rip);
}

void test1() 
{
	TRY
		abort();
	EXCEPT(signal_except)
		printf("except singal\n");
	END_TRY;

	printf("\n");
}

void test2()
{
	TRY
		int a = 1;
		int b = 0;
		int c = a/b;
		printf("%d\n", c);
	EXCEPT(signal_except)
		printf("except singal\n");
	END_TRY;

	printf("\n");
}

void test3()
{
	TRY
		int *a = NULL;
		*a = 0;
	EXCEPT(signal_except)
		printf("except singal\n");
	END_TRY;

	printf("\n");
}

void test4()
{
	TRY
		raise(SIGFPE);
	EXCEPT(signal_except)
		printf("except singal\n");
	END_TRY;

	printf("\n");
}

int main()
{
	setsig(SIGBUS, signal_handler); //bus error
	setsig(SIGSEGV, signal_handler); //segmentation violation
	setsig(SIGFPE, signal_handler); //floating-point exception
	setsig(SIGABRT, signal_handler); //abort

	test1();
	test1();
	test2();
	test2();
	test2();
	test3();
	test3();
	test4();
	test4();

	return 0;
}
```

此处有几点需要说明一下

1. 为什么要在sigpanic中抛出异常，而不直接在signal_handler中抛出？
2. 为什么不直接使用C++的异常系统，而是采用了一个第三方实现的异常库*[注3](#Z3)*？

 要解释第一点，我们要先了解一下linux下信号的处理流程，下图展示了一个完整的信号处理流程

 ![](http://ww2.sinaimg.cn/mw690/7fd174e1gw1eywqpnpbhij20ku0b40u2.jpg)

 由于一个完整的信号处理流程比较复杂，不是本文讨论的主题，所以这里只讨论我们关注的部分。

 内核在启动信号处理程序之前，做了些什么：
 1. 针对该信号重新计算进程的sigpending标志，屏蔽某些信号，以防止嵌套响应信号
 2. 由于信号处理程序是在用户空间执行，所以内核会在用户空间堆栈中为信号处理程序的执行预先创建一个框架
 3. 由于信号处理程序执行完毕后还要返回到系统空间中，所以内核还会为如何返回系统空间做一些准备

 信号处理程序执行完毕返回到内核中后，内核又做了些什么？
 1. 重新计算sigpending标志，使进程不再挂起屏蔽的信号
 2. 返回到用户空间，返回原先被打断的指令处继续执行

 所以如果直接在信号处理程序中抛出异常，就会打断一次完整的信号处理流程，下次再有该类信号产生时，内核就会检测到信号还在处理中，不允许信号重入而宕掉进程*[注4](#Z4)*。因此我们并不直接抛出异常，而是执行
```c
	SIG_RIP(ctxt) = (uintptr_t)sigpanic; //设置RIP寄存器，令函数返回后执行流转到sigpanic函数入口处
```
 如此一来，当内核要返回原先被打断的指令处继续执行时就会进入sigpanic函数，然后我们就可以在sigpanic函数中抛出异常了。

 对于第二点，则与linux下gcc的异常实现有关。gcc提供了两种异常实现：SJLJ和DWARF2。

 SJLJ实现方式是使用setjmp/longjmp来实现的，由于现在新的芯片都会考虑异常处理的需求，提供专用的寄存器，因此在主流的gcc中，SJLJ方式一般都不会开启。

 DWARF2实现方式则是利用DW2的调试信息和系统提供的异常处理ABI接口来实现的异常。在intel平台，这属于Itanium ABI接口的一部分，Itanium ABI的_Unwind_RaiseException接口Unwinding Stack（栈展开）依赖于ar.pfs寄存器，ar.pfs寄存器所指向的信息记录了调用者的寄存器栈帧状态和收尾计数器，它覆盖了整个调用过程，当函数调用时，会被保存，函数返回时，会被恢复。而sigpanic函数不是通过函数调用的形式进入的，所以在sigpanic中抛出异常时，pfs寄存器指向的还是上上个函数的栈帧状态，而不是预期的上一个函数的栈帧状态。因此此处无法使用C++的异常（或者说要使用C++的异常需要一些额外的工作，我们为了方便使用了一个第三方的异常库）。


### 项目实践

由于KTA的gamesvr采用了协程架构，每一个客户端链接会有一个对应的协程来处理客户端请求，之前示例中所使用的异常库无法适用于这种多协程环境，因此如果我们要为这种多协程环境实现一个完整的异常系统，就要像go语言一样，为每个协程建立一个异常调用链。这虽然是一项很有趣的工作，但是我们的初衷并不是实现一个异常系统，我们的目标是**当服务器进程发生宕机异常时，我们可以从异常中恢复**。因此KTA的解决方案是在协程的入口处调用setjmp
```c
	if(setjmp(r->buf) == 0)
	{   
		kroutine_main(s, id);
	}   
	else
	{   
		fatal_tlog("server panic, id[%"PRIu64"]", id);
	}

	del_kroutine(s, id);
```

当发生宕机异常时，则调用longjmp回到协程入口处
```c
void kroutine_recovery(KSCHEDULE *s)
{
	if(s != NULL)
	{
		KROUTINE *r = kschedule_get_running_routine(s);
		if(r != NULL)
		{
			longjmp(r->buf, 1); //从异常中恢复
		}
	}

	//无法恢复则发送一个硬件错误信号，引发core dump
	raise(SIGTRAP);
}
```

一旦某个协程在运行期发生了宕机异常，我们会：
1. 在信号处理函数中产生输出一个minidump文件，以便今后分析宕机原因
2. 在异常处理函数中调用longjmp，执行流回到setjmp处
3. 将这个协程所服务的客户端以服务器异常为由踢下线（在异常状态下下线，不会将玩家数据落地到DB，以免将脏数据写入DB，由于KTA的玩家数据是10s钟落地一次，因此可能存在最大10s的数据丢失）
4. 销毁协程对象，结束协程运行

由于是使用的longjmp来实现的栈回退，因此不会调用栈上的C++对象的析构函数，像如下所示的代码，在异常发生时就会存在资源泄漏的情况。

```c
	auto_ptr<KTA_ACTOR_DEF> actor_def(new KTA_ACTOR_DEF());
```

但这不是太大的问题，因为宕机异常在正常情况下是不会发生的，**一旦发生则属于非常规操作，在这种情况下我们允许一定程度的资源泄漏**。


### 总结

宕机异常恢复机制，可以使我们的服务器更稳定，但这并不是万能灵药，要使我们的服务器真正稳定，最根本的方法还是要**保证代码质量**，只是没有程序员可以保证自己产出的代码没有bug，即便是有多年经验的资深程序员依旧有可能因为疏忽写出有宕机风险的代码，因此增加一种保障机制，还是很有必要的。而该方案相比于通过外部程序监控拉起的方案，则具有响应时间短，影响范围小的优点。在实际生产环境中可同时使用，两者互补，做双重保障。


### 参考

[服务器宕机问题总结][L1]

Linux内核源代码情景分析6.4节 信号

[C++异常处理1][L2]，[C++异常处理2][L3]

[Itanium™ Software Conventions and Runtime Architecture Guide][L4]


### 注脚
---
<span id="Z1">1. 本文所言宕机问题皆指由代码bug造成的进程奔溃问题，诸如机器掉电、硬件故障之类引起的宕机问题不在本文讨论范围内</span>

<span id="Z2">2. C++虽然有异常机制，但是无法捕获除零、内存越界等异常，所以对于这种导致宕机的异常也就无法恢复</span>

<span id="Z3">3. 此处使用的是《C语言接口与实现》一书中实现的异常库</span>

<span id="Z4">4. 真要在signal_handler中抛出异常也是可以的，只要在安装信号时，将sa_flags设置上SA_NOMASK，就可以把信号设置成为一个可重入信号，从而允许嵌套响应</span>



[L1]: http://km.oa.com/group/1642/articles/show/219031
[L2]: http://www.cnblogs.com/catch/p/3604516.html
[L3]: http://www.cnblogs.com/catch/p/3619379.html
[L4]: http://www.intel.com/content/dam/www/public/us/en/documents/guides/itanium-software-runtime-architecture-guide.pdf
