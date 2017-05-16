title: 同步函数 dispatch_sync 解析  
description: 简单介绍 GCD 中的同步函数 dispatch_sync  
date: 2017/5/15 22:00    
comments: true  
toc: true  
category: iOS-GCD  

---
# 简述  
在 GCD 中，有一个很常见的问题：同步函数传入串行对列，在串行对列的任务执行同步函数同一串行队列，就会造成死锁，这里，将详细讲述死锁发生的原因和`dispatch_sync`的执行逻辑。  

# dispatch\_sync  

```c
static void
_dispatch_sync_slow(dispatch_queue_t dq, void (^work)(void))
{
	// Blocks submitted to the main queue MUST be run on the main thread,
	// therefore under GC we must Block_copy in order to notify the thread-local
	// garbage collector that the objects are transferring to the main thread
	// rdar://problem/7176237&7181849&7458685
	if (dispatch_begin_thread_4GC) {
		dispatch_block_t block = _dispatch_Block_copy(work);
		return dispatch_sync_f(dq, block, _dispatch_call_block_and_release);
	}
	struct Block_basic *bb = (void *)work;
	dispatch_sync_f(dq, work, (dispatch_function_t)bb->Block_invoke);
}
#endif

void
dispatch_sync(dispatch_queue_t dq, void (^work)(void))
{
#if DISPATCH_COCOA_COMPAT
	if (slowpath(dq == &_dispatch_main_q)) {
		return _dispatch_sync_slow(dq, work);
	}
#endif
	struct Block_basic *bb = (void *)work;
	dispatch_sync_f(dq, work, (dispatch_function_t)bb->Block_invoke);
}
#endif
```   

`dispatch_begin_thread_4GC`在这里是为空的，因此不管是什么对列，都将执行`dispatch_sync_f`函数。  

# dispatch\_sync\_f  

```c
void
dispatch_sync_f(dispatch_queue_t dq, void *ctxt, dispatch_function_t func)
{
	if (fastpath(dq->dq_width == 1)) {
		return dispatch_barrier_sync_f(dq, ctxt, func);
	}
	if (slowpath(!dq->do_targetq)) {
		// the global root queues do not need strict ordering
		(void)dispatch_atomic_add2o(dq, dq_running, 2);
		return _dispatch_sync_f_invoke(dq, ctxt, func);
	}
	_dispatch_sync_f2(dq, ctxt, func);
}
```  

这里有三种情况：  

1. 传入串行队列：执行`dispatch_barrier_sync_f`。  
2. 传入全局队列：执行传入的任务，然后根据`dq_running`检测任务队列有没有激活，没有激活就执行激活函数，这个激活函数将会在异步函数重点介绍，这里不再讲述。  
3. 传入自定义并行队列：执行`_dispatch_sync_f2`。  

## 传入串行队列情况 dispatch\_barrier\_sync\_f  

```c
void
dispatch_barrier_sync_f(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func)
{
	// 1) ensure that this thread hasn't enqueued anything ahead of this call
	// 2) the queue is not suspended
	if (slowpath(dq->dq_items_tail) || slowpath(DISPATCH_OBJECT_SUSPENDED(dq))){
		return _dispatch_barrier_sync_f_slow(dq, ctxt, func);
	}
	if (slowpath(!dispatch_atomic_cmpxchg2o(dq, dq_running, 0, 1))) {
		// global queues and main queue bound to main thread always falls into
		// the slow case
		return _dispatch_barrier_sync_f_slow(dq, ctxt, func);
	}
	if (slowpath(dq->do_targetq->do_targetq)) {
		return _dispatch_barrier_sync_f_recurse(dq, ctxt, func);
	}
	_dispatch_barrier_sync_f_invoke(dq, ctxt, func);
}
```  

`dispatch_barrier_sync_f`解析：  

1. 串行对列前面存在任务，则调用`_dispatch_barrier_sync_f_slow`，`_dispatch_barrier_sync_f_slow`主要是产生一个信号量，将信号量压入串行队列，然后等待这个信号量。现在有个问题，这个信号量是哪里释放的，我看到的一个地方是下面的`_dispatch_barrier_sync_f_invoke`释放。  
2. 检测任务队列的`dq_running`，如果有运行（主队列，因为主对列`dq_running`才不会是 0），进入`_dispatch_barrier_sync_f_slow`，向串行任务压入信号量，等待激活。  
3. 如果任务对列有双重目标对列，调用`_dispatch_barrier_sync_f_recurse`，找到真正的目标队列。  
4. 如果无特殊情况，则执行任务队列里的任务，且执行完后检测到队列有其他任务，则释放前面说过的信号量。  

这里就可以解释一下上面说过的同步串行队列死锁问题，在同步函数中，当第一次执行串行队列任务的时候，跳到 4 ，直接开始执行任务，(这里有个问题的是这里我没看到有东西压到串行队列去，可能是因为这是初期版本，还没成熟，后续代码可能添加，我当初看第一代 GCD 源码的时候，我就看到它的漏洞百出，最新代码 Xcode 跳转不了定义，我就没去看了)，在任务里面通过执行 2 又向这个同步队列中压入信号量，然后等待信号量，进入死锁。  

主队列的死锁和上面唯一的区别是，它会跳转到 2 进入死锁，而不是 1。  

## 传入自定义并行队列  \_dispatch\_sync\_f2

```c
DISPATCH_NOINLINE
static void
_dispatch_sync_f2(dispatch_queue_t dq, void *ctxt, dispatch_function_t func)
{
	// 1) ensure that this thread hasn't enqueued anything ahead of this call
	// 2) the queue is not suspended
	if (slowpath(dq->dq_items_tail) || slowpath(DISPATCH_OBJECT_SUSPENDED(dq))){
		return _dispatch_sync_f_slow(dq, ctxt, func);
	}
	if (slowpath(dispatch_atomic_add2o(dq, dq_running, 2) & 1)) {
		return _dispatch_sync_f_slow2(dq, ctxt, func);
	}
	if (slowpath(dq->do_targetq->do_targetq)) {
		return _dispatch_sync_f_recurse(dq, ctxt, func);
	}
	_dispatch_sync_f_invoke(dq, ctxt, func);
}
```  

4 种情况：  

1. 并行队列内有其他对象：压入信号量，等待其他线程释放这个信号量。  
2. 并行队列没激活：激活后运行任务。  
3. 目标队列多重：重复找到正确目标队列。  
4. 并行队列没其他内容：调用并激活这个队列。  


# 总结  
说实话，我阅读的 GCD 版本源码已经没啥用了，GCD 早就被苹果重构过了，连基本的数据对象都改变了，但阅读这个源码对于练就我的耐心和底层的知识，还有 GCD 的运行逻辑还是有一定帮助的。  