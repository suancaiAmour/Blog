title: dispatch_source 的 I/O 模型解析  
description: 解析 dispatch_source 具体使用 I/O 模型  
date: 2017/05/29 23:14  
comments: true  
toc: true  
category: iOS-GCD  

---
# I/O 模型简述  
程序在进行 I/O 操作的时候，都无法立即获取所需要的数据，因此会有一定的延时，且这个延时是无法确定多长的，为了高效的解决这个延时，工程师们提出了几种 I/O 模型，多线程线程池技术也由此引发出来。以我个人的观点（常规的分类还有很多啥信号驱动 I/O 模型，但根据我对硬件上的理解，其实这些都可以分类为一大类，这些可以具体参看[【I/O模型】几种IO模型浅析（一）](http://blog.csdn.net/hejingyuan6/article/details/47679005)），我对 I/O 模型分成三大类，接下来具体分析其中优缺点。  

## 同步阻塞式 I/O 模型  
在处理这个延时，最便捷处理的方法是什么？那就是一直等待这个延时，一直等待，直到数据的来临，例如，当需要读取`scoket`套接字的数据的时候，就调用`read`函数，它会在线程中一直等待，直到有数据的来临，数据补偿完缓冲区后，我们的程序就可以继续执行，然后我们自然可以从缓冲区里面读取我们需要的数据。  

* 优点：一直等待，程序的状态是不变的，在我们读取数据的，我们就可以根据程序的状态进行编码，这样编码带来的逻辑解放，很给力，且程序的逻辑一直是连续的，便于开发者理解。  
* 缺点：无法同时处理其他业务，CPU 处于等待状态，是对 CPU 的巨大的浪费，是不能接受的。在处理高并发的时候，需要创建其他线程，这对操作系统的内核是一种负担，即使用线程池技术也无法弥补，程序的效率及其低下，现在在实际开发中，已经没人使用这种模型，或许说，从来就没有人使用过，也就只能从 Demo 中能看过。  


## 异步回调式 I/O 复用模型  
既然没人使用同步阻塞式，必然就有其他更高效的方式，现在常在多线程 I/O 操作所使用的技术就是异步回调式 I/O 复用模型。这种模型的广泛性，几乎所有的 CPU 都可见。这种模型简单来说，就是在一条线程需要等待延时的时候，写好回调钩子函数，将这个等待扔到另一个专门用来等待这些延时的线程去，等延时结束后，触发回调函数。那么这个线程也就等待着很多的延时，这也就是*复用*的含义。异步回调式 I/O 模型，我又分为两小类。这里统一说一下这种模式的优缺点：  

* 优点：CPU 没有在等待中浪费它的计算能力，匹配线程池技术可以处理高并发的环境，在后台的机器广泛使用，一个优秀的异步回调式 I/O 复用模型高并发库，在一架好一点的服务器上，解决 C10M 已经错错有余。  
* 缺点：对开发者逻辑的巨大的冲突，人类的思维都是同步的，对于这种异步模式，逻辑上是不顺畅的，主要体现在，开发者永远不知道，回调函数是什么时候什么情况下调用的，且回调函数之前的程序状态是早已经消失的了，这对于开发者的代码的书写，比较麻烦，具体描述可以参考[为什么觉得协程是趋势？](https://www.zhihu.com/question/32218874)这篇文章。这种模式的效率极其高，但出于多线程模式，这就避免不了的对数据的加锁，这其实也是效率的一种损害，且锁的机制，让开发者更难以应付。  

### 轮询异步回调式 I/O 复用模型  
轮询主要体现在获取延时结束的信息上，轮询的意思是线程一直在查询操作系统内核：延时有没有结束，数据有没有到来，缓存有没有存完。这样，线程是不会进入休眠的，因为它一直做一件事，就是一直查询，一直查询，通常实现这种方式的常见的两个函数是`select`函数和`poll`函数，在`grpc`的 C 库中使用的就是`poll`函数，`grpc`起了一个线程，一直在轮询查找网络数据的到来，然后分发下到各个`channel`中。  

* 优点：`select`和`poll`函数是多平台通用的，不管在 windows ， Linux ，还是在 FreeBSD 上，都支持很完美，所以 GRPC 使用这一种模式是有道理的，这样它在跨平台的方面做的就很完美了。  
* 缺点：一直在轮询操作系统内核，那线程是无法进入休眠的，这也是对效率和能耗的一种损害。  

### 中断异步回调式 I/O 复用模型  
中断是硬件上的一种概念，像我们点击键盘之类的事件，电流信号会传导到硬件通用 I/O 口，继而触发硬件中断，然后触发编写好的中断服务函数，将信息传递给操作系统的内核了，由操作系统内核做进一步的分发。中断式 I/O 模型在不同的操作系统有不同的函数，Linux 下的`epoll`函数，FreeBSD 下的`kevent`函数，Windows 的 IOCP，这样在不同的平台就会有不同的情况。在 iOS 中，最类似这样的 I/O 复用的就是`runloop`中的`source`，可以很清晰的表明，`source1`是由中断触发的，这是因为触发`source1`的是`mach_port`，而接收或者发送`mach_port`信息的函数是`mach_msg`:  

```c
/*
 *	Routine:	mach_msg
 *	Purpose:
 *		Send and/or receive a message.  If the message operation
 *		is interrupted, and the user did not request an indication
 *		of that fact, then restart the appropriate parts of the
 *		operation silently (trap version does not restart).
 */
__WATCHOS_PROHIBITED __TVOS_PROHIBITED
extern mach_msg_return_t	mach_msg(
					mach_msg_header_t *msg,
					mach_msg_option_t option,
					mach_msg_size_t send_size,
					mach_msg_size_t rcv_size,
					mach_port_name_t rcv_name,
					mach_msg_timeout_t timeout,
					mach_port_name_t notify);
```

由上的函数注释可以看出，`mach_port`是可以由中断触发的，这里可以充分的猜测，`mach_port`的应该实现了对`kevent`的封装（当然，也应该封装了其他信息的传递，比如说线程间的信息传递）。  

说一下这种模式的优缺点吧：  

* 优点：等待结束的信息是由中断触发的，不需要轮询，那线程是可以进入休眠状态的，这样更节省功耗和提高效率。  
* 缺点：不能跨平台实现，每个系统都有自己的实现方式，要想实现一个多平台都能实现的库，复杂度极高，还不如每个平台写一个这样的库。  

## 同步协程式 I/O 复用模型  

同步情况下能不能实现 I/O 复用呢？答案是能，利用协程。协程这几年比较火，因为它在这方面的效率和代码编辑的方便，吸引了一个又一个的软件工程师，甚至会误解协程是这几年才提出来的观念，而实际上，协程早在 1963 年就就有人提出相关的观念并实现，在那个时候，集成电路才刚开始发展，摩尔定律要在 1965 年才能提出。在计算机编程的神书 TAOCP <<计算机程序设计艺术>> 早已有对其的详细概述。谷歌大力推广的 go 语言效率极高，也是因为采取了协程这样类似的概念。协程说白了，就是一种函数跳转模式，允许函数之间的无条件跳转，其实实现上也就是函数的保护现场和恢复现场。那这么久的观念，为什么最近几年才火呢？主要是因为协程的年代遇到了 C 语言最巅峰的时代（事实上，现在也还是 C 语言巅峰时代），C 语言程序编写的理念就是这种过程式，自上而下的编程，不允许随意跳转这种破坏程序结构的东西，在此情况下，像可以函数内随意跳转的`goto`语句遭到大规模的排挤，更过分的协程当然是不允许出现在眼前了，而实际上呢？`goto`语句至今未能在 C 语言的规范中删除，在 C 库中也有为了方便协程实现的`setjump`和`longjump`函数。存在即是合理，虽然 C 强调自上而下，但也无法扼杀掉某些情况随意跳转带来的巨大优势（虽然不随意跳转也能实现），像`goto`语句，是在函数内实现统一资源释放的最好的方式，如果写过 C 代码多的人，都有些能感触到，自己还是要使用`goto`才方便，像 Linux 的内核，现在也有不少于 2 万行的`goto`语句，而 GCD 源码 189 版本中，有 32 行`goto`语句，最新版的 GCD 源码已经上升到 90 多个`goto`语句了。如果有对同步协程式 I/O 复用模型感兴趣的同学，可以看一下腾讯开源的后台协程库`libco`[Libco协程组件](http://code.tencent.com/libco.html)。这里说一下其中的优缺点：  

* 优点：协程会保存程序状态，在代码编程上，开发者更方便，协程在同一条线程中，可以避免锁机制，协程切换是不经过操作系统内核的，在效率上远高于线程，可以实现更高的并发量。  
* 缺点：每一条协程都有自己栈，这样将耗费大量内存，虽然后续有协程共享栈的出现，但还是极大的占内存（幸运的是，三星正在开发的 MRAM 将在 2020 年批量生产，那时候的内存，可以说根本不值得考虑）。开发者必须注意协程中代码的效率，如果一条协程的效率低，将会影响到同一条线程下的所有协程，切不可在协程中运行 CPU 密集型任务。  

# dispatch\_source 的 I/O 模型  
那么`dispatch_source`的 I/O 模型是采取的哪一种模型呢？我们一起来看源码。  

上篇文章讲到了如何激活`dispatch_source`，具体函数为`_dispatch_source_invoke`，如果从此讲此，可能太过烦躁，不再说明，但最后的流程都会流到`_dispatch_update_kq`函数上：  

```c
long
_dispatch_update_kq(const struct kevent *kev)
{
	struct kevent kev_copy = *kev;
	// This ensures we don't get a pending kevent back while registering
	// a new kevent
	kev_copy.flags |= EV_RECEIPT;

	if (_dispatch_select_workaround && (kev_copy.flags & EV_DELETE)) {
		// Only executed on manager queue
		switch (kev_copy.filter) {
		case EVFILT_READ:
			if (kev_copy.ident < FD_SETSIZE &&
					FD_ISSET((int)kev_copy.ident, &_dispatch_rfds)) {
				FD_CLR((int)kev_copy.ident, &_dispatch_rfds);
				_dispatch_rfd_ptrs[kev_copy.ident] = 0;
				(void)dispatch_atomic_dec(&_dispatch_select_workaround);
				return 0;
			}
			break;
		case EVFILT_WRITE:
			if (kev_copy.ident < FD_SETSIZE &&
					FD_ISSET((int)kev_copy.ident, &_dispatch_wfds)) {
				FD_CLR((int)kev_copy.ident, &_dispatch_wfds);
				_dispatch_wfd_ptrs[kev_copy.ident] = 0;
				(void)dispatch_atomic_dec(&_dispatch_select_workaround);
				return 0;
			}
			break;
		default:
			break;
		}
	}

	int rval = kevent(_dispatch_get_kq(), &kev_copy, 1, &kev_copy, 1, NULL);
	if (rval == -1) {
		// If we fail to register with kevents, for other reasons aside from
		// changelist elements.
		(void)dispatch_assume_zero(errno);
		//kev_copy.flags |= EV_ERROR;
		//kev_copy.data = error;
		return errno;
	}

	// The following select workaround only applies to adding kevents
	if ((kev->flags & (EV_DISABLE|EV_DELETE)) ||
			!(kev->flags & (EV_ADD|EV_ENABLE))) {
		return 0;
	}

	// Only executed on manager queue
	switch (kev_copy.data) {
	case 0:
		return 0;
	case EBADF:
		break;
	default:
		// If an error occurred while registering with kevent, and it was
		// because of a kevent changelist processing && the kevent involved
		// either doing a read or write, it would indicate we were trying
		// to register a /dev/* port; fall back to select
		switch (kev_copy.filter) {
		case EVFILT_READ:
			if (dispatch_assume(kev_copy.ident < FD_SETSIZE)) {
				if (!_dispatch_rfd_ptrs) {
					_dispatch_rfd_ptrs = calloc(FD_SETSIZE, sizeof(void*));
				}
				_dispatch_rfd_ptrs[kev_copy.ident] = kev_copy.udata;
				FD_SET((int)kev_copy.ident, &_dispatch_rfds);
				(void)dispatch_atomic_inc(&_dispatch_select_workaround);
				_dispatch_debug("select workaround used to read fd %d: 0x%lx",
						(int)kev_copy.ident, (long)kev_copy.data);
				return 0;
			}
			break;
		case EVFILT_WRITE:
			if (dispatch_assume(kev_copy.ident < FD_SETSIZE)) {
				if (!_dispatch_wfd_ptrs) {
					_dispatch_wfd_ptrs = calloc(FD_SETSIZE, sizeof(void*));
				}
				_dispatch_wfd_ptrs[kev_copy.ident] = kev_copy.udata;
				FD_SET((int)kev_copy.ident, &_dispatch_wfds);
				(void)dispatch_atomic_inc(&_dispatch_select_workaround);
				_dispatch_debug("select workaround used to write fd %d: 0x%lx",
						(int)kev_copy.ident, (long)kev_copy.data);
				return 0;
			}
			break;
		default:
			// kevent error, _dispatch_source_merge_kevent() will handle it
			_dispatch_source_drain_kevent(&kev_copy);
			break;
		}
		break;
	}
	return kev_copy.data;
}
```

上面函数简单逻辑可以描述为，使用`kevent`注册事件，如果事件注册失败，则准备使用`select`函数，上面对`kevent`注册失败的原因在注释中有说明，是因为注册了一个`/dev/*`端口，我本不擅长这些函数，这让我有点迷糊，有明白的同学可以说一下，不甚感激。  

说完了注册事件（注册事件并不是在信息线程完成，而是在其他线程中调用），还没有讲到信息线程真正运行的地方。信息线程具体运行的函数是`_dispatch_mgr_invoke`，看一下源码：  

```c
DISPATCH_NOINLINE DISPATCH_NORETURN
static void
_dispatch_mgr_invoke(void)
{
	static const struct timespec timeout_immediately = { 0, 0 };
	struct timespec timeout;
	const struct timespec *timeoutp;
	struct timeval sel_timeout, *sel_timeoutp;
	fd_set tmp_rfds, tmp_wfds;
	struct kevent kev[1];
	int k_cnt, err, i, r;

	_dispatch_thread_setspecific(dispatch_queue_key, &_dispatch_mgr_q);
#if DISPATCH_COCOA_COMPAT
	// Do not count the manager thread as a worker thread
	(void)dispatch_atomic_dec(&_dispatch_worker_threads);
#endif
	_dispatch_malloc_vm_pressure_setup();

	for (;;) {
		_dispatch_run_timers();

		timeoutp = _dispatch_get_next_timer_fire(&timeout);

		if (_dispatch_select_workaround) {
			FD_COPY(&_dispatch_rfds, &tmp_rfds);
			FD_COPY(&_dispatch_wfds, &tmp_wfds);
			if (timeoutp) {
				sel_timeout.tv_sec = timeoutp->tv_sec;
				sel_timeout.tv_usec = (typeof(sel_timeout.tv_usec))
						(timeoutp->tv_nsec / 1000u);
				sel_timeoutp = &sel_timeout;
			} else {
				sel_timeoutp = NULL;
			}

			r = select(FD_SETSIZE, &tmp_rfds, &tmp_wfds, NULL, sel_timeoutp);
			if (r == -1) {
				err = errno;
				if (err != EBADF) {
					if (err != EINTR) {
						(void)dispatch_assume_zero(err);
					}
					continue;
				}
				for (i = 0; i < FD_SETSIZE; i++) {
					if (i == _dispatch_kq) {
						continue;
					}
					if (!FD_ISSET(i, &_dispatch_rfds) && !FD_ISSET(i,
							&_dispatch_wfds)) {
						continue;
					}
					r = dup(i);
					if (r != -1) {
						close(r);
					} else {
						if (FD_ISSET(i, &_dispatch_rfds)) {
							FD_CLR(i, &_dispatch_rfds);
							_dispatch_rfd_ptrs[i] = 0;
							(void)dispatch_atomic_dec(
									&_dispatch_select_workaround);
						}
						if (FD_ISSET(i, &_dispatch_wfds)) {
							FD_CLR(i, &_dispatch_wfds);
							_dispatch_wfd_ptrs[i] = 0;
							(void)dispatch_atomic_dec(
									&_dispatch_select_workaround);
						}
					}
				}
				continue;
			}

			if (r > 0) {
				for (i = 0; i < FD_SETSIZE; i++) {
					if (i == _dispatch_kq) {
						continue;
					}
					if (FD_ISSET(i, &tmp_rfds)) {
						FD_CLR(i, &_dispatch_rfds); // emulate EV_DISABLE
						EV_SET(&kev[0], i, EVFILT_READ,
								EV_ADD|EV_ENABLE|EV_DISPATCH, 0, 1,
								_dispatch_rfd_ptrs[i]);
						_dispatch_rfd_ptrs[i] = 0;
						(void)dispatch_atomic_dec(&_dispatch_select_workaround);
						_dispatch_mgr_thread2(kev, 1);
					}
					if (FD_ISSET(i, &tmp_wfds)) {
						FD_CLR(i, &_dispatch_wfds); // emulate EV_DISABLE
						EV_SET(&kev[0], i, EVFILT_WRITE,
								EV_ADD|EV_ENABLE|EV_DISPATCH, 0, 1,
								_dispatch_wfd_ptrs[i]);
						_dispatch_wfd_ptrs[i] = 0;
						(void)dispatch_atomic_dec(&_dispatch_select_workaround);
						_dispatch_mgr_thread2(kev, 1);
					}
				}
			}

			timeoutp = &timeout_immediately;
		}

		k_cnt = kevent(_dispatch_kq, NULL, 0, kev, sizeof(kev) / sizeof(kev[0]),
				timeoutp);
		err = errno;

		switch (k_cnt) {
		case -1:
			if (err == EBADF) {
				DISPATCH_CLIENT_CRASH("Do not close random Unix descriptors");
			}
			if (err != EINTR) {
				(void)dispatch_assume_zero(err);
			}
			continue;
		default:
			_dispatch_mgr_thread2(kev, (size_t)k_cnt);
			// fall through
		case 0:
			_dispatch_force_cache_cleanup();
			continue;
		}
	}
}
```  

`_dispatch_mgr_invoke`会进入到一个死循环当中，这样保证线程不会退出去。  

`_dispatch_mgr_invoke`函数逻辑大体如下：  

1. `_dispatch_malloc_vm_pressure_setup`：注册一个检测虚拟内存的`dispatch_source`；
2. `_dispatch_run_timers`：检测 GCD 时间对列，对时间队列到期的任务进行执行；  
3. `_dispatch_get_next_timer_fire`：获取时间队列中下一个执行的任务的时间与现在的时间差；  
4. 利用上面的获取的时间差，在`select`中，最多等待时间差长的时间；  
5. `selelct`结束后，利用`kevent`等待注册的事件，注意等待的时间是 0。  

从上面的解析可以看出，`dispatch_source`使用的 I/O 模型是异步回调式 I/O 模型，且为了弥补`kevent`可能出现的注册失败，还保险式的调用了`select`函数。  