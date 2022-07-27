---
title: golang中的系统调用
date: 2022-07-27 10:53:13
index_img: /images/golang.webp
banner_img: /images/banner.webp
excerpt: 聊一聊golang如何实现系统调用的
tags:
  - golang
  - syscall
categories: 技术文章
---

## 什么是系统调用

{% note info %}
In computing, a system call (commonly abbreviated to syscall) is the programmatic way in which a computer program requests a service from the kernel of the operating system on which it is executed. This may include hardware-related services (for example, accessing a hard disk drive or accessing the device's camera), creation and execution of new processes, and communication with integral kernel services such as process scheduling. System calls provide an essential interface between a process and the operating system.
{% endnote %} 

以上是维基百科上关于[系统调用](https://en.wikipedia.org/wiki/System_call)的介绍，实际概括一句话就是: 运行在用户空间的程序向内核请求调用需要更高权限的服务，诸如一些：

1. 进程控制
2. 文件管理
3. 硬件设备管理
4. 一些系统信息的管理
5. 通信管理
6. 权限管理

## Golang中是如何做系统调用的

{% note primary %}
golang 调试环境
version： 1.16.15
os: linux
arch: amd64
{% endnote %}
 

golang中系统调用主要分成两部分实现，分别在syscall包中和runtime包里。syscall中暴露的一些系统调用的接口，都是直接提供给用户程序使用的。runtime包中则是供内部使用的，对用户程序不可见的。

实际上golang中，系统调用随处可见，就拿我们常用的fmt.println来说，最下层就是调用的syscall.Write。

### syscall包分析

syscall包里主要分为两种文件：

- 不同操作系统对应的操作文件, 例如：[syscall_linux.go](https://cs.opensource.google/go/go/+/refs/tags/go1.16.15:src/syscall/syscall_linux.go)
- 自动生成的接口文件, 例如: [zsyscall_linux.go](https://cs.opensource.google/go/go/+/master:src/cmd/vendor/golang.org/x/sys/unix/zsyscall_linux.go;l=1?q=zsyscall_linux&sq=&ss=go%2Fgo)

syscall_linux.go文件中会有很多注释：

```
//sys	Acct(path string) (err error)
//sys	Adjtimex(buf *Timex) (state int, err error)
//sys	Chdir(path string) (err error)
//sys	Chroot(path string) (err error)
//sys	Close(fd int) (err error)
//sys	Dup(oldfd int) (fd int, err error)
//sys	Dup3(oldfd int, newfd int, flags int) (err error)
//sysnb	EpollCreate1(flag int) (fd int, err error)
//sysnb	EpollCtl(epfd int, op int, fd int, event *EpollEvent) (err error)
```

根据这些注释，mksyscall.pl 脚本会生成对应的平台的具体实现。(mksyscall.pl 是一段 perl 脚本)。

我们可以发现，注释中包含两种前缀：sys 和 sysnb (nb其实就是non-blocking的意思)。我们可以从上文所述的perl脚本中窥见两种的大体上区别。

```
# A line beginning with //sysnb is like //sys, except that the
# goroutine will not be suspended during the execution of the system
# call.  This must only be used for system calls which can never
# block, as otherwise the system call could cause all goroutines to
# hang.
```

大致意思就是，非阻塞的调用是在执行过程中对应的goroutine是不会挂起的。通常使用非阻塞系统调用，是不能有阻塞情况的，否则可能导致所有挂载在当前P的本地协程队列全部挂起。

由此，我们可以知道，实际上系统调用是分为了两种:

1. 阻塞调用
2. 非阻塞调用

还有的说有一种wrapper系统调用，实际上是对上面两种调用进行一点包裹，减少点参数传递或者换个调用名字，实际还是囊括在上面两种范围内。

### 系统调用实现分析

实际上，我们发现，syscall_linux.go中所有的系统调用实现最终都是套用以下几个接口实现的

{% asset_img syscall_interface.png syscall_interface %}

其中，Syscall和Syscall6的区别只在于传入参数的多少(RawSyscall和RawSyscall6区别也在于此)。上文我们有说过有两种调用，实际代码可以看出来，阻塞调用实际最后调用的是Syscall，非阻塞调用使用的是RawSyscall。

这部分方法实现是直接用汇编处理的，我们可以在类似[asm_linux_amd64.s](https://cs.opensource.google/go/go/+/refs/tags/go1.16.15:src/syscall/asm_linux_amd64.s)文件中找到具体实现。

其中，Syscall实现如下:

```go
// func Syscall(trap int64, a1, a2, a3 uintptr) (r1, r2, err uintptr);
// Trap # in AX, args in DI SI DX R10 R8 R9, return in AX DX
// Note that this differs from "standard" ABI convention, which
// would pass 4th arg in CX, not R10.

TEXT ·Syscall(SB),NOSPLIT,$0-56
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
ok:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
```

RawSyscall调用如下:

```
// func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2, err uintptr)
TEXT ·RawSyscall(SB),NOSPLIT,$0-56
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok1
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	RET
ok1:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	RET
```

我们只要在汇编中把参数依次传入寄存器，并调用 SYSCALL 指令即可进入内核处理逻辑，系统调用执行完毕之后，返回值存储在AX和DX中。大致如下：

| DI | SI | DX | R10 | R9 | R8 | AX |
| --- | --- | --- | --- | --- | --- | --- |
| 参数1 | 参数2 | 参数3/返回值 | 参数4 | 参数5 | 参数6 | 调用指令/返回值 |

{% note info %}
golang使用的是plan9汇编，和IA64名字上存在映射关系，大致映射规则是少了一个R前缀。例如DI代表中RDI。
同时，plan9引入了四个伪寄存器：FP、PC、SB、SP，可以参见[plan9 assembly解析](https://segmentfault.com/a/1190000039978109)大致了解一下
{% endnote %}

上面代码也可以看出来，RawSyscall相较于Syscall只是少了开始的runtime·entersyscall以及执行完调用之后的runtime·exitsyscall。

接下来，我们来看看这两处方法到底执行了什么

### runtime.syscallenter

```go
func reentersyscall(pc, sp uintptr) {
	_g_ := getg()
	_g_.m.locks++    // 防止M被抢占式调度使用

	// 捕获可能发生的调用，将堆栈保护替换为使任何堆栈检查失败的内容，留一个标志通知newstack终止
	_g_.stackguard0 = stackPreempt
	_g_.throwsplit = true

	// 保留当前执行现场，方便后续复原
	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
        // 设置当前状态为_Gsyscall,当前G被挂起，直到系统调用结束，才会重新让G进入Grunning状态
	casgstatus(_g_, _Grunning, _Gsyscall)
	if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
		systemstack(func() {
			print("entersyscall inconsistent ", hex(_g_.syscallsp), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
			throw("entersyscall")
		})
	}

        // 捕获堆栈进行跟踪
	if trace.enabled {
		systemstack(traceGoSysCall)
		save(pc, sp)
	}

        // 唤醒sysmon线程，对进行系统调用的P进行抢占
	if atomic.Load(&sched.sysmonwait) != 0 {
		systemstack(entersyscall_sysmon)
		save(pc, sp)
	}

	if _g_.m.p.ptr().runSafePointFn != 0 {
		// runSafePointFn may stack split if run on this stack
		systemstack(runSafePointFn)
		save(pc, sp)
	}

	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
	_g_.sysblocktraced = true

        // P中去除M的引用，但是M还持有P的引用，方便系统调用返回之后优先选取原来的P
	pp := _g_.m.p.ptr()
	pp.m = 0
	_g_.m.oldp.set(pp)
	_g_.m.p = 0

        // 变更当前P的状态
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting != 0 {
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}

	_g_.m.locks--

```

上述方法主要是为系统调用前做准备工作：

1. 修改G状态为_Gsyscall
2. 唤醒sysmon线程，让其执行retake()抢占式调度阻塞的P
3. 修改P的状态

{% note info %}
当 P 处于 _Psyscall 状态时，表明对应的 goroutine 正在进行系统调用。如果抢占 p，需要满足几个条件：
1. p 的本地运行队列里面有等待运行的 goroutine。这时 p 绑定的 g 正在进行系统调用，无法去执行其他的 g，因此需要接管 p 来执行其他的 g。
2. 没有“无所事事”的 p。`sched.nmspinning` 和 `sched.npidle` 都为 0，这就意味着没有“找工作”的 m，也没有空闲的 p，大家都在“忙”，可能有很多工作要做。因此要抢占当前的 p，让它来承担一部分工作。
3. 从上一次监控线程观察到 p 对应的 m 处于系统调用之中到现在已经超过 10 毫秒。这说明系统调用所花费的时间较长，需要对其进行抢占，以此来使得 `retake` 函数返回值不为 0，这样，会保持 sysmon 线程 20 us 的检查周期，提高 sysmon 监控的实时性。
{% endnote %}

### runtime.syscallexit

当系统调用返回的时候，会调用该方法恢复调度

```go
func exitsyscall() {
    _g_ := getg()
    _g_.m.locks++
    if getcallersp() > _g_.syscallsp {
        throw("exitsyscall: syscall frame is no longer valid")
    }

    _g_.waitsince = 0
    oldp := _g_.m.oldp.ptr()
    _g_.m.oldp = 0
     // 重新获取p
    if exitsyscallfast(oldp) {
        if trace.enabled {
            if oldp != _g_.m.p.ptr() || _g_.m.syscalltick != _g_.m.p.ptr().syscalltick {
                systemstack(traceGoStart)
            }
        }
        // There's a cpu for us, so we can run.
        _g_.m.p.ptr().syscalltick++
        // We need to cas the status and scan before resuming...
        casgstatus(_g_, _Gsyscall, _Grunning)

        // Garbage collector isn't running (since we are),
        // so okay to clear syscallsp.
        _g_.syscallsp = 0
        _g_.m.locks--
        if _g_.preempt {
            // restore the preemption request in case we've cleared it in newstack
            _g_.stackguard0 = stackPreempt
        } else {
            // otherwise restore the real _StackGuard, we've spoiled it in entersyscall/entersyscallblock
            _g_.stackguard0 = _g_.stack.lo + _StackGuard
        }
        _g_.throwsplit = false

        if sched.disable.user && !schedEnabled(_g_) {
            // Scheduling of this goroutine is disabled.
            Gosched()
        }

        return
    }

    _g_.sysexitticks = 0
    if trace.enabled {

        for oldp != nil && oldp.syscalltick == _g_.m.syscalltick {
            osyield()
        }

        _g_.sysexitticks = cputicks()
    }

    _g_.m.locks--

    // 没有获取到p，只能解绑当前g，重新调度该m了
    mcall(exitsyscall0)

    _g_.syscallsp = 0
    _g_.m.p.ptr().syscalltick++
    _g_.throwsplit = false
}

```

其实这个过程就两个步骤：

1. exitsyscallfast M尝试重新绑定P，如果之前的P被占用了，看下全局调度中有没有空闲的P，进行绑定，如果没有则进行下面一步
2. exitsyscall0 M解绑关联的G，进入休眠状态，等待下次唤醒，G进入全局队列，等待P窃取

### 如何使用

{% asset_img x86_syscall.png x86_syscall %}

这是从[Chromium OS Docs](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#x86_64-64_bit) 截取的部分系统调用指令表

实际上这部分指令可以在[zsysnum_linux_amd64.go](https://cs.opensource.google/go/x/sys/+/master:unix/zsysnum_linux_amd64.go;l=1?q=sysnum_linux_amd&sq=)中看到：

{% asset_img golang_syscall_trap.png golang_syscall_trap %}

当然，部分指令已经暴露接口给用户程序了，部分未实现的则可以直接使用Syscall或者RawSyscall调用

## 一些可以使用的点

### 调用自己编译的系统调用

这个需要实现系统调用，通过编译内核的方式或者使用插入模块的方式使之生效，然后使用API调用。可以参考一下[如何添加新的系统调用](https://www.kernel.org/doc/html/v4.10/process/adding-syscalls.html#compatibility-system-calls-x86)以及[实现自己的系统调用](https://developer.aliyun.com/article/364575)

### 禁用一些非法的系统调用或者添加一些系统调用的日志

微服务场景下常见的一种保护措施，例如docker就提供了[seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)来限制一些容器的系统调用。当然，我们自己也可以使用类似的第三方包[libseccomp-golang](https://github.com/seccomp/libseccomp-golang)来做一些自定义操作。

### 调用一些未暴露的系统调用

例如共享内存的操作就没有暴露给用户，可以通过自己调用Syscall做实现，参考实现：[shm.go](https://github.com/gen2brain/shm/blob/master/shm.go)

go:linkname的使用

在runtime里面发现了很多go:linkname的使用，例如 syscall.Exit 实现就如下：

在文件：syscall/syscall.go中，只有方法签名

```go
func Exit(code int)
```

而在文件: runtime/runtime.go中，就通过linkname关联了实现：

```go
//go:linkname syscall_Exit syscall.Exit
//go:nosplit
func syscall_Exit(code int) {
	exit(int32(code))
}
```

查阅[官方文档](https://pkg.go.dev/cmd/compile)有如下发现：

> //go:linkname localname [importpath.name]
> 
> 
> This special directive does not apply to the Go code that follows it. Instead, the //go:linkname directive instructs the compiler to use “[importpath.name](http://importpath.name/)” as the object file symbol name for the variable or function declared as “localname” in the source code. If the “[importpath.name](http://importpath.name/)” argument is omitted, the directive uses the symbol's default object file symbol name and only has the effect of making the symbol accessible to other packages. Because this directive can subvert the type system and package modularity, it is only enabled in files that have imported "unsafe".
> 

通过这种方式，我们可以突破go包的一些访问限制，将一些私有的变量或者函数导出到本地来。具体使用方法可以参考：[How to call private functions (bind to hidden symbols) in GoLang](https://sitano.github.io/2016/04/28/golang-private/)

## 参考资料

[1] [曹春晖：谈一谈 Go 和 Syscall](https://segmentfault.com/a/1190000019375999)

[2] [golang调度学习-调度流程 (五) Syscall](https://segmentfault.com/a/1190000039855617)

[3] [go协作与抢占](https://golang.design/under-the-hood/zh-cn/part2runtime/ch06sched/preemption/)

[4] [Plan9汇编解析](https://go.xargin.com/docs/assembly/assembly/)
