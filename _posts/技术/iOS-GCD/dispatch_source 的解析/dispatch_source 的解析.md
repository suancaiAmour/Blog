title: dispatch_source 的解析  
description: 解析 dispatch_source 的数据结构及其创建过程，为后续的 GCD 信号线程的 I/O 模型铺垫基础  
date: 2017/05/28 14:36  
comments: true  
toc: true  
category: iOS-GCD  

---
# 简述  
`dispatch_source`是 GCD 中为信号线程（GCD 中会默认创建一条线程，且这条线程会一直执行，在程序崩溃的时候，提供崩溃信息，我在日常开发中，也看到`NSURLSession`的网络数据也来源这条线程）提供的一个回调接口，类似于`Runloop`中的`source`。我们在`dispatch_source`注册相对应的事件，当事件来临之时，就会触发我们的回调，这具体是线程 I/O 模型的内容，这里不说明。  

# dispatch\_source 数据结构  
```c
struct dispatch_source_s {
	DISPATCH_STRUCT_HEADER(dispatch_source_s, dispatch_source_vtable_s);
	DISPATCH_QUEUE_HEADER;
	// Instruments always copies DISPATCH_QUEUE_MIN_LABEL_SIZE, which is 64,
	// so the remainder of the structure must be big enough
	union {
		char _ds_pad[DISPATCH_QUEUE_MIN_LABEL_SIZE];
		struct {
			char dq_label[8];
			dispatch_kevent_t ds_dkev;
			dispatch_source_refs_t ds_refs;
			unsigned int ds_atomic_flags;
			unsigned int
				ds_is_level:1,
				ds_is_adder:1,
				ds_is_installed:1,
				ds_needs_rearm:1,
				ds_is_timer:1,
				ds_cancel_is_block:1,
				ds_handler_is_block:1,
				ds_registration_is_block:1;
			unsigned long ds_data;
			unsigned long ds_pending_data;
			unsigned long ds_pending_data_mask;
			unsigned long ds_ident_hack;
		};
	};
};
```

如上可以看见，前面两个是`dispatch_object`的常规数据结构，后面是一个联合体，包含有大量的标志位，FreeBSD 的 I/O 复用模型 kqueue 的事件（`ds_dkev`）和`dispatch_source`的引用。  

# dispatch\_source\_create 函数  
```c
dispatch_source_t
dispatch_source_create(dispatch_source_type_t type,
	uintptr_t handle,
	unsigned long mask,
	dispatch_queue_t q)
{
	const struct kevent *proto_kev = &type->ke;
	dispatch_source_t ds = NULL;
	dispatch_kevent_t dk = NULL;

	// input validation
	if (type == NULL || (mask & ~type->mask)) {
		goto out_bad;
	}

	switch (type->ke.filter) {
	case EVFILT_SIGNAL:
		if (handle >= NSIG) {
			goto out_bad;
		}
		break;
	case EVFILT_FS:
#if DISPATCH_USE_VM_PRESSURE
	case EVFILT_VM:
#endif
	case DISPATCH_EVFILT_CUSTOM_ADD:
	case DISPATCH_EVFILT_CUSTOM_OR:
	case DISPATCH_EVFILT_TIMER:
		if (handle) {
			goto out_bad;
		}
		break;
	default:
		break;
	}

	ds = calloc(1ul, sizeof(struct dispatch_source_s));
	if (slowpath(!ds)) {
		goto out_bad;
	}
	dk = calloc(1ul, sizeof(struct dispatch_kevent_s));
	if (slowpath(!dk)) {
		goto out_bad;
	}

	dk->dk_kevent = *proto_kev;
	dk->dk_kevent.ident = handle;
	dk->dk_kevent.flags |= EV_ADD|EV_ENABLE;
	dk->dk_kevent.fflags |= (uint32_t)mask;
	dk->dk_kevent.udata = dk;
	TAILQ_INIT(&dk->dk_sources);

	// Initialize as a queue first, then override some settings below.
	_dispatch_queue_init((dispatch_queue_t)ds);
	strlcpy(ds->dq_label, "source", sizeof(ds->dq_label));

	// Dispatch Object
	ds->do_vtable = &_dispatch_source_kevent_vtable;
	ds->do_ref_cnt++; // the reference the manger queue holds
	ds->do_ref_cnt++; // since source is created suspended
	ds->do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_INTERVAL;
	// The initial target queue is the manager queue, in order to get
	// the source installed. <rdar://problem/8928171>
	ds->do_targetq = &_dispatch_mgr_q;

	// Dispatch Source
	ds->ds_ident_hack = dk->dk_kevent.ident;
	ds->ds_dkev = dk;
	ds->ds_pending_data_mask = dk->dk_kevent.fflags;
	if ((EV_DISPATCH|EV_ONESHOT) & proto_kev->flags) {
		ds->ds_is_level = true;
		ds->ds_needs_rearm = true;
	} else if (!(EV_CLEAR & proto_kev->flags)) {
		// we cheat and use EV_CLEAR to mean a "flag thingy"
		ds->ds_is_adder = true;
	}

	// Some sources require special processing
	if (type->init != NULL) {
		type->init(ds, type, handle, mask, q);
	}
	if (fastpath(!ds->ds_refs)) {
		ds->ds_refs = calloc(1ul, sizeof(struct dispatch_source_refs_s));
		if (slowpath(!ds->ds_refs)) {
			goto out_bad;
		}
	}
	ds->ds_refs->dr_source_wref = _dispatch_ptr2wref(ds);
	dispatch_assert(!(ds->ds_is_level && ds->ds_is_adder));

	// First item on the queue sets the user-specified target queue
	dispatch_set_target_queue(ds, q);
#if DISPATCH_DEBUG
	dispatch_debug(ds, "%s", __FUNCTION__);
#endif
	return ds;

out_bad:
	free(ds);
	free(dk);
	return NULL;
}
```
上面这个函数没啥好说的，主要就是对`dispatch_source`数据结构进行内部变量的赋值，值得注意的是，`dispatch_source`数据结构在这里`ds->do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_INTERVAL`是处于挂起状态的，传入的形参简单说明一下：

<font color="red">dispatch\_source\_type\_t:</font> 要回调的事件的类型，具体类型由 GCD 确定，开发者只能使用 GCD 定义好的事件类型，具体类型在`source.h`文件中详细给出且说明。  

<font color="red">handle:</font> 事件句柄，当你调用创建事件函数后，操作系统会返回一个整型数据给你，用来表明你创建的事件，就像一个指针一样，这个句柄指向了你创建的事件，为你后续操作这个事件提供支持。  

<font color="red">mask:</font> 通常传进去 0 ，在使用上，我也不太清楚干嘛，在`dispatch_source_create`函数中，只有一开始判断和初始化`dispatch_source`的引用使用过。  

<font color="red">dispatch\_queue\_t:</font> 回调函数触发被调用时所使用的队列，将会触发到此条队列所在的线程调用回调函数。

# dispatch\_source\_set\_event\_handler 函数  

```c
void
dispatch_source_set_event_handler(dispatch_source_t ds,
		dispatch_block_t handler)
{
	handler = _dispatch_Block_copy(handler);
	dispatch_barrier_async_f((dispatch_queue_t)ds, handler,
			_dispatch_source_set_event_handler2);
}
```  

`dispatch_barrier_async_f`是异步调用函数，但这里不会调用到`_dispatch_source_set_event_handler2`，因为`ds`是挂起的，是无法调用的，但这里可以实现将`_dispatch_source_set_event_handler2`压进去`ds`中·，具体代码的解析，在我的另外一篇[异步函数 dispatch_async 解析](http://chenmaomao.com/2017/05/20/%E6%8A%80%E6%9C%AF/iOS-GCD/%E5%BC%82%E6%AD%A5%E5%87%BD%E6%95%B0%20dispatch_async%20%E8%A7%A3%E6%9E%90/%E5%BC%82%E6%AD%A5%E5%87%BD%E6%95%B0%20dispatch_async%20%E8%A7%A3%E6%9E%90/)已经详细说明，这里不必再次讲述。那什么时候会触发`_dispatch_source_set_event_handler2`函数呢？可以继续往下看，会有答案。  

# dispatch\_resume  函数  
```c
DISPATCH_NOINLINE
static void
_dispatch_resume_slow(dispatch_object_t dou)
{
	_dispatch_wakeup(dou._do);
	// Balancing the retain() done in suspend() for rdar://8181908
	_dispatch_release(dou._do);
}

void
dispatch_resume(dispatch_object_t dou)
{
	// Global objects cannot be suspended or resumed. This also has the
	// side effect of saturating the suspend count of an object and
	// guarding against resuming due to overflow.
	if (slowpath(dou._do->do_ref_cnt == DISPATCH_OBJECT_GLOBAL_REFCNT)) {
		return;
	}
	// Check the previous value of the suspend count. If the previous
	// value was a single suspend interval, the object should be resumed.
	// If the previous value was less than the suspend interval, the object
	// has been over-resumed.
	unsigned int suspend_cnt = dispatch_atomic_sub2o(dou._do, do_suspend_cnt,
			DISPATCH_OBJECT_SUSPEND_INTERVAL) +
			DISPATCH_OBJECT_SUSPEND_INTERVAL;
	if (fastpath(suspend_cnt > DISPATCH_OBJECT_SUSPEND_INTERVAL)) {
		// Balancing the retain() done in suspend() for rdar://8181908
		return _dispatch_release(dou._do);
	}
	if (fastpath(suspend_cnt == DISPATCH_OBJECT_SUSPEND_INTERVAL)) {
		return _dispatch_resume_slow(dou);
	}
	DISPATCH_CLIENT_CRASH("Over-resume of an object");
}
```  
在这里，会将`ds`的挂起状态取消掉，这个时候，就会触发到`_dispatch_source_set_event_handler2`函数的调用，其实`_dispatch_source_set_event_handler2`函数也只是一些内部变量的赋值，真正注册`dispatch_source`的地方在`ds`的`do_invoke`函数中，也就是`_dispatch_source_invoke`函数，下篇文章会说明这些。这样，就详细了说明了`dispatch_source`的构成和它外部开发者生成操作的具体函数源码。  

# 参考文献  
[变态的libDispatch源码分析-全局队列异步延时任务处理过程-原理与创建ds](http://www.aiuxian.com/article/p-1827544.html)

