title: 异步函数 dispatch_async 解析  
description: 简单介绍 dispatch_async 函数逻辑  
date: 2017/05/20  
category: iOS-GCD  
comments: true  
toc: true  

---
# 简述  
`dispatch_async`函数是 GCD 内的异步函数，一个函数就可以创建另一个线程，进行多线程编程，让多线程的编程变得异常简单，而且对线程的生命周期进行了处理，不需要外部开发者做太多这方面的事，专注于外部逻辑的处理就好了。  

# dispatch\_async 解析  
`dispatch_async`在 libdispatch 的源码为：  

```c
DISPATCH_NOINLINE
void
dispatch_async_f(dispatch_queue_t dq, void *ctxt, dispatch_function_t func)
{
	dispatch_continuation_t dc;

	// No fastpath/slowpath hint because we simply don't know
	if (dq->dq_width == 1) {
		return dispatch_barrier_async_f(dq, ctxt, func);
	}

	dc = fastpath(_dispatch_continuation_alloc_cacheonly());
	if (!dc) {
		return _dispatch_async_f_slow(dq, ctxt, func);
	}

	dc->do_vtable = (void *)DISPATCH_OBJ_ASYNC_BIT;
	dc->dc_func = func;
	dc->dc_ctxt = ctxt;

	// No fastpath/slowpath hint because we simply don't know
	if (dq->do_targetq) {
		return _dispatch_async_f2(dq, dc);
	}

	_dispatch_queue_push(dq, dc);
}

#ifdef __BLOCKS__
void
dispatch_async(dispatch_queue_t dq, void (^work)(void))
{
	dispatch_async_f(dq, _dispatch_Block_copy(work),
			_dispatch_call_block_and_release);
}
#endif
```

`dispatch_async_f`逻辑主要三种情况：

1. 传入串行队列，执行`dispatch_barrier_async_f`，最后还是执行任务压入队列的操作。  
2. 传入自定义串行队列，执行`_dispatch_async_f2`。  
3. 传入全局队列，任务压入队列中。  

在`dispatch_async_f`具体的逻辑还有，检测当前线程的任务链表，有任务链表，则从当前线程的任务链表中取出一个节点，压入队列中，没有则从堆内存中创建一个任务对象，压入队列中。  

在 2 的情况中，接下来主要做的事情是，检测当前队列的状态，看是否可执行，是否被锁住，然后根据不同情况激活队列，最后将任务压进队列中。  

由上可知，`_dispatch_queue_push`是最关键的宏定义函数，接下来将讲解它。   


# \_dispatch\_queue\_push
`_dispatch_queue_push`最后会演变为`_dispatch_trace_queue_push_list`:

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_trace_queue_push_list(dispatch_queue_t dq, dispatch_object_t _head,
		dispatch_object_t _tail)
{
	if (slowpath(DISPATCH_QUEUE_PUSH_ENABLED())) {
		struct dispatch_object_s *dou = _head._do;
		do {
			_dispatch_trace_continuation(dq, dou, DISPATCH_QUEUE_PUSH);
		} while (dou != _tail._do && (dou = dou->do_next));
	}
	_dispatch_queue_push_list(dq, _head, _tail);
}

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_queue_push_list(dispatch_queue_t dq, dispatch_object_t _head,
		dispatch_object_t _tail)
{
	struct dispatch_object_s *prev, *head = _head._do, *tail = _tail._do;

	tail->do_next = NULL;
	dispatch_atomic_store_barrier();
	prev = fastpath(dispatch_atomic_xchg2o(dq, dq_items_tail, tail));
	if (prev) {
		// if we crash here with a value less than 0x1000, then we are at a
		// known bug in client code for example, see _dispatch_queue_dispose
		// or _dispatch_atfork_child
		prev->do_next = head;
	} else {
		_dispatch_queue_push_list_slow(dq, head);
	}
}
```

在`_dispatch_trace_queue_push_list`中，主要是压进的对象进行类型处理，不同的压进对象，对对象内部的值进行赋值，以确保后续的使用。  

在`_dispatch_queue_push_list`中，判断队列中是否有其他对象，有就将压入的对象接在队列后面，没有就执行`_dispatch_queue_push_list_slow`以触发`_dispatch_wakeup`来激活队列。  

# \_dispatch\_wakeup

```c
dispatch_queue_t
_dispatch_wakeup(dispatch_object_t dou)
{
	dispatch_queue_t tq;

	if (slowpath(DISPATCH_OBJECT_SUSPENDED(dou._do))) {
		return NULL;
	}
	if (!dx_probe(dou._do) && !dou._dq->dq_items_tail) {
		return NULL;
	}

	// _dispatch_source_invoke() relies on this testing the whole suspend count
	// word, not just the lock bit. In other words, no point taking the lock
	// if the source is suspended or canceled.
	if (!dispatch_atomic_cmpxchg2o(dou._do, do_suspend_cnt, 0,
			DISPATCH_OBJECT_SUSPEND_LOCK)) {
#if DISPATCH_COCOA_COMPAT
		if (dou._dq == &_dispatch_main_q) {
			_dispatch_queue_wakeup_main();
		}
#endif
		return NULL;
	}
	_dispatch_retain(dou._do);
	tq = dou._do->do_targetq;
	_dispatch_queue_push(tq, dou._do);
	return tq;	// libdispatch does not need this, but the Instrument DTrace
				// probe does
}
```

`_dispatch_wakeup`会找到队列的目标队列，并继续向目标队列压入这个队列（注意这里是队列压入队列当中哦，当然在其他情况下会压入其他 GCD 数据类型，如果传入的自定义队列，都是队列压入全局队列中），当找到没有目标队列的队列，也就是 GCD 初始化的那几个队列(主队列或全局队列)时，全局队列会执行队列中的`dx_probe`函数，也就是`_dispatch_queue_wakeup_global`，激活全局队列，主队列会调用函数`_dispatch_queue_wakeup_main`，主队列的激活不再说明，主要靠`mach_port`和在`runloop`中注册相对应的`source1`，具体可看`runloop`源码。  


# \_dispatch\_queue\_wakeup\_global

```c
static bool
_dispatch_queue_wakeup_global(dispatch_queue_t dq)
{
	static dispatch_once_t pred;
	struct dispatch_root_queue_context_s *qc = dq->do_ctxt;
	int r;

	if (!dq->dq_items_tail) {
		return false;
	}

	_dispatch_safe_fork = false;

	dispatch_debug_queue(dq, __PRETTY_FUNCTION__);

	dispatch_once_f(&pred, NULL, _dispatch_root_queues_init);

#if HAVE_PTHREAD_WORKQUEUES
#if DISPATCH_ENABLE_THREAD_POOL
	if (qc->dgq_kworkqueue)
#endif
	{
		if (dispatch_atomic_cmpxchg2o(qc, dgq_pending, 0, 1)) {
			pthread_workitem_handle_t wh;
			unsigned int gen_cnt;
			_dispatch_debug("requesting new worker thread");

			r = pthread_workqueue_additem_np(qc->dgq_kworkqueue,
					_dispatch_worker_thread2, dq, &wh, &gen_cnt);
			(void)dispatch_assume_zero(r);
		} else {
			_dispatch_debug("work thread request still pending on global "
					"queue: %p", dq);
		}
		goto out;
	}
#endif // HAVE_PTHREAD_WORKQUEUES
#if DISPATCH_ENABLE_THREAD_POOL
	if (dispatch_semaphore_signal(qc->dgq_thread_mediator)) {
		goto out;
	}

	pthread_t pthr;
	int t_count;
	do {
		t_count = qc->dgq_thread_pool_size;
		if (!t_count) {
			_dispatch_debug("The thread pool is full: %p", dq);
			goto out;
		}
	} while (!dispatch_atomic_cmpxchg2o(qc, dgq_thread_pool_size, t_count,
			t_count - 1));

	while ((r = pthread_create(&pthr, NULL, _dispatch_worker_thread, dq))) {
		if (r != EAGAIN) {
			(void)dispatch_assume_zero(r);
		}
		sleep(1);
	}
	r = pthread_detach(pthr);
	(void)dispatch_assume_zero(r);
#endif // DISPATCH_ENABLE_THREAD_POOL

out:
	return false;
}
```

`_dispatch_queue_wakeup_global`主要是利用了 pThread 创建了新线程，新线程的入口是`_dispatch_worker_thread`。这里还使用了一种线程保活技术，值得关注，这种线程保活技术是由`dispatch_semaphore`实现的，`dispatch_semaphore`其中两种函数的返回值的意义：

1. `dispatch_semaphore_signal`返回值意义：当返回值为0时表示当前并没有线程等待其处理的信号量，其处理的信号量的值加1即可。当返回值不为0时，表示其当前有（一个或多个）线程等待其处理的信号量(多个线程有使用`dispatch_semaphore_wait`在等待)，并且该函数唤醒了一个等待的线程（当线程有优先级时，唤醒优先级最高的线程；否则随机唤醒）。  

2. `dispatch_semaphore_wait`返回值意义：返回0表示接收到了`dispatch_semaphore_signal`，返回其他表示超时。  

在`_dispatch_worker_thread`会有一个 do...while 循环检测`dispatch_semaphore_wait`设置的 65 秒的超时状态，如果在 任务结束后 65 秒内接收到`dispatch_semaphore_signal`发出的信号量，就重新触发 do...while，重新执行队列中的任务，所以 GCD 创建的队列在任务结束后还有 65 秒的生命时间。`_dispatch_worker_thread`源码：

```c
static void *
_dispatch_worker_thread(void *context)
{
	dispatch_queue_t dq = context;
	struct dispatch_root_queue_context_s *qc = dq->do_ctxt;
	sigset_t mask;
	int r;

	// workaround tweaks the kernel workqueue does for us
	r = sigfillset(&mask);
	(void)dispatch_assume_zero(r);
	r = _dispatch_pthread_sigmask(SIG_BLOCK, &mask, NULL);
	(void)dispatch_assume_zero(r);

	do {
		_dispatch_worker_thread2(context);
		// we use 65 seconds in case there are any timers that run once a minute
	} while (dispatch_semaphore_wait(qc->dgq_thread_mediator,
			dispatch_time(0, 65ull * NSEC_PER_SEC)) == 0);

	(void)dispatch_atomic_inc2o(qc, dgq_thread_pool_size);
	if (dq->dq_items_tail) {
		_dispatch_queue_wakeup_global(dq);
	}

	return NULL;
}
```

这个函数主要是为了实现 65 秒的线程保活，在 `_dispatch_worker_thread2`实现了真正的任务，`_dispatch_worker_thread2`的形参是触发这个函数的全局队列。  


# \_dispatch\_worker\_thread2

```c
static void
_dispatch_worker_thread2(void *context)
{
	struct dispatch_object_s *item;
	dispatch_queue_t dq = context;
	struct dispatch_root_queue_context_s *qc = dq->do_ctxt;


	if (_dispatch_thread_getspecific(dispatch_queue_key)) {
		DISPATCH_CRASH("Premature thread recycling");
	}

	_dispatch_thread_setspecific(dispatch_queue_key, dq);
	qc->dgq_pending = 0;

#if DISPATCH_COCOA_COMPAT
	(void)dispatch_atomic_inc(&_dispatch_worker_threads);
	// ensure that high-level memory management techniques do not leak/crash
	if (dispatch_begin_thread_4GC) {
		dispatch_begin_thread_4GC();
	}
	void *pool = _dispatch_begin_NSAutoReleasePool();
#endif

#if DISPATCH_PERF_MON
	uint64_t start = _dispatch_absolute_time();
#endif
	while ((item = fastpath(_dispatch_queue_concurrent_drain_one(dq)))) {
		_dispatch_continuation_pop(item);
	}
#if DISPATCH_PERF_MON
	_dispatch_queue_merge_stats(start);
#endif

#if DISPATCH_COCOA_COMPAT
	_dispatch_end_NSAutoReleasePool(pool);
	dispatch_end_thread_4GC();
	if (!dispatch_atomic_dec(&_dispatch_worker_threads) &&
			dispatch_no_worker_threads_4GC) {
		dispatch_no_worker_threads_4GC();
	}
#endif

	_dispatch_thread_setspecific(dispatch_queue_key, NULL);

	_dispatch_force_cache_cleanup();

}
```

在`_dispatch_worker_thread2`执行真正的处理队列函数。  

`_dispatch_worker_thread2`中会创建一个自动释放池，利用 TSD 将线程绑定在这个队列，函数结束后释放队列和任务缓存对线程的绑定，这里面的关键函数是：

`_dispatch_queue_concurrent_drain_one`：这个函数主要取出队列的内容

1. 队列`dq_item_head`没内容，返回 NULL。
2. 队列内容有冲突，没从`do_next`赋值上，重新调用唤醒函数，重新起一个线程，返回 NULL。  
3. 判断`do_next`是否还有内容，有内容就重新调用唤醒函数，重新起一个线程，并将 head 返回回去，如果没有内容，根据`dq_items_tail`来判断，如果不跟 head 一致，就说明即将有内容压进，等待`do_next`内容压进，有内容压进后，然后重复 3 开始的判断。
 
`_dispatch_continuation_pop`:判断取出的是不是队列，如果是队列执行`_dispatch_queue_invoke`，如果不是队列，就以任务对象来执行任务（这里不说明对 group 之类的处理）。在`_dispatch_queue_invoke`中，还要更深一层的调用，这里只简单说明，主要是等待内容的压进，然后判断是不是串行队列，如果是串行队列，就取出串行队列中的任务，进行执行；如果是并行队列（这个并行队列只能是自定义的），里面还有更深的调用，但初步可以看出是一个一个取出任务，再一个一个压入全局队列中，触发新的线程，这样便实现了多任务并发执行。  