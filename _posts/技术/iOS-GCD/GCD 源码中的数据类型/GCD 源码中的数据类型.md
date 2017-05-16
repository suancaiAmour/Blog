title: GCD 源码中的数据类型  
description: 简单介绍 GCD 源码中的几种数据类型  
date: 2017/5/13 15:48  
comments: true  
toc: true  
category: iOS-GCD

---
# 简述
我曾在我的简书博客中写过 GCD 相关源码的文章，在此，有些东西都不太记得，重新写一遍，看看自己成长多少。 GCD 的源码可以从[GCD 源码](https://opensource.apple.com/tarballs/libdispatch/)下载。这里只讲解 187.9 的 GCD，有些功能可能已经失效，有些功能和现在的差距很大，但主要函数的内部逻辑变化是不会很大的，在此也只是为了探究它的内部逻辑，并没有精益求精的去找最新的代码。    

# GCD 的数据类型  
GCD 在苹果中的框架名是 libdispatch，下面可能会涉及到这个词，它指的就是 GCD。  

在 libdispatch 的`bash.h`文件中，统计了 GCD 的数据类型:  

```c
typedef union {
	struct dispatch_object_s *_do;
	struct dispatch_continuation_s *_dc;
	struct dispatch_queue_s *_dq;
	struct dispatch_queue_attr_s *_dqa;
	struct dispatch_group_s *_dg;
	struct dispatch_source_s *_ds;
	struct dispatch_source_attr_s *_dsa;
	struct dispatch_semaphore_s *_dsema;
	struct dispatch_data_s *_ddata;
	struct dispatch_io_s *_dchannel;
	struct dispatch_operation_s *_doperation;
	struct dispatch_disk_s *_ddisk;
} dispatch_object_t __attribute__((transparent_union));
```  

`dispatch_object_t`是一种类多态的形式，创建这样的联合体，使用`dispatch_object_t `便可以细分到具体的对象。  

<font color="#0099ff"> dispatch\_object\_s:</font> GCD 的根类，类似于 OC 中的`NSObject`，它更多的是订好 GCD 数据的类型协议，所有的数据类型必须遵守这个协议，这样做可以方便撰写内存处理函数，且可以更方便调试。在数据传递中也有一种多态的形式。  

<font color="#0099ff"> dispatch\_continuation\_s:</font> GCD 的任务环境，在 OC 中可以使用闭包 block ，而在 C 的环境中是不可能使用这个的，为此，在 block 转换为函数时，`dispatch_continuation_s`是它们两的中转工厂。  

<font color="#0099ff"> dispatch\_queue\_s:</font> GCD 任务队列。  

<font color="#0099ff"> dispatch\_queue\_attr\_s:</font> GCD 任务队列的属性。   

<font color="#0099ff"> dispatch\_source\_s</font>  GCD 信号源。  

<font color="#0099ff"> dispatch\_source\_attr\_s</font> GCD 信号源属性。  

上面还有 GCD 的其他数据类型，比如信号量等，这些不再详细讲解。  

## 创建 GCD 的数据类型的两个关键宏定义  
所有 GCD 的数据类型都遵循`dispatch_object_s`数据类型协议，而`dispatch_object_s`搭建是由两个宏定义所搭建的。  

```c
#define DISPATCH_STRUCT_HEADER(x, y)	\
	const struct y *do_vtable;	\
	struct x *volatile do_next;	\
	unsigned int do_ref_cnt;	\
	unsigned int do_xref_cnt;	\
	unsigned int do_suspend_cnt;	\
	struct dispatch_queue_s *do_targetq;	\
	void *do_ctxt; \
	void *do_finalizer
	
	
#define DISPATCH_VTABLE_HEADER(x)	\
	unsigned long const do_type;	\
	const char *const do_kind; \
	size_t (*const do_debug)(struct x *, char *, size_t);	\
	struct dispatch_queue_s *(*const do_invoke)(struct x *);	\
	bool (*const do_probe)(struct x *); \
	void (*const do_dispose)(struct x *)
```  

`DISPATCH_STRUCT_HEADER` 定义了一个结构体，结构体内的属性和属性功能有：

<font color="red"> do_vtable:</font> 由`DISPATCH_VTABLE_HEADER`创建的结构体对象，主要包含了对对象的处理函数。  

<font color="red"> do_next:</font> 链表引向下一个数据。   

<font color="red"> do\_ref\_cnt:</font> 引用计数，没多看到它的使用，接下来看到源码的解析后，我会更正它的内容。  

<font color="red"> do\_xref\_cnt:</font> 外部引用计数，当外部引用计数成为某一个值得时候，会调用这个对象的销毁函数，也就是说会调用`do_vtable`中的`do_dispose`函数(由此也会触发`do_finalizer`)。  

<font color="red"> do\_suspend\_cnt</font> 挂起计数，用于是否挂起对象的计数量。  

<font color="red"> dispatch\_queue\_s</font> 对象将要压入的目标队列。  

<font color="red"> do_ctxt</font> 上下文环境，通常我们需要传入的函数形参参数。  

<font color="red"> do_finalizer</font> 类似 OC 的 `dealloc`函数。  


`DISPATCH_VTABLE_HEADER`在宏定义中并没有所体现他是一个结构体，但在外部使用中，它都是被`struct`所包围，且形成的结构都会赋值到`DISPATCH_STRUCT_HEADER`中的`vtable`中。`vtable`主要定义对象的处理函数：  

<font color="blue"> do_type:</font> 数据类型。数据类型的定义有下:  

```c
enum {
	_DISPATCH_CONTINUATION_TYPE		=    0x00000, // meta-type for continuations
	_DISPATCH_QUEUE_TYPE			=    0x10000, // meta-type for queues
	_DISPATCH_SOURCE_TYPE			=    0x20000, // meta-type for sources
	_DISPATCH_SEMAPHORE_TYPE		=    0x30000, // meta-type for semaphores
	_DISPATCH_NODE_TYPE				=    0x40000, // meta-type for data node
	_DISPATCH_IO_TYPE				=    0x50000, // meta-type for io channels
	_DISPATCH_OPERATION_TYPE		=    0x60000, // meta-type for io operations
	_DISPATCH_DISK_TYPE				=    0x70000, // meta-type for io disks
	_DISPATCH_META_TYPE_MASK		=  0xfff0000, // mask for object meta-types
	_DISPATCH_ATTR_TYPE				= 0x10000000, // meta-type for attributes

	DISPATCH_CONTINUATION_TYPE		= _DISPATCH_CONTINUATION_TYPE,

	DISPATCH_DATA_TYPE				= _DISPATCH_NODE_TYPE,

	DISPATCH_IO_TYPE				= _DISPATCH_IO_TYPE,
	DISPATCH_OPERATION_TYPE			= _DISPATCH_OPERATION_TYPE,
	DISPATCH_DISK_TYPE				= _DISPATCH_DISK_TYPE,

	DISPATCH_QUEUE_ATTR_TYPE		= _DISPATCH_QUEUE_TYPE |_DISPATCH_ATTR_TYPE,

	DISPATCH_QUEUE_TYPE				= 1 | _DISPATCH_QUEUE_TYPE,
	DISPATCH_QUEUE_GLOBAL_TYPE		= 2 | _DISPATCH_QUEUE_TYPE,
	DISPATCH_QUEUE_MGR_TYPE			= 3 | _DISPATCH_QUEUE_TYPE,
	DISPATCH_QUEUE_SPECIFIC_TYPE	= 4 | _DISPATCH_QUEUE_TYPE,

	DISPATCH_SEMAPHORE_TYPE			= _DISPATCH_SEMAPHORE_TYPE,

	DISPATCH_SOURCE_KEVENT_TYPE		= 1 | _DISPATCH_SOURCE_TYPE,
};
```  

<font color="blue"> do_kind:</font> 对这个对象的描述。比如说信号量对象会赋值为`.do_kind = "semaphore"`。  

<font color="blue"> do_debug:</font> debug 调试方法。  

<font color="blue"> do_invoke:</font> 唤醒对象方法。  

<font color="blue"> do_dispose:</font> 销毁对象方法，通常会调用对象的`finalizer`函数。  

<font color="blue"> do_probe:</font> 对象中的关键函数，在 GCD 的 `rootqueue`会被赋值为`_dispatch_queue_wakeup_global`函数。  

## dispatch\_object\_s 是如何创建的  

`dispatch_object_s`比较简单，主要使用上面的两个宏定义被可以完成:  

```c
struct dispatch_object_vtable_s {
	DISPATCH_VTABLE_HEADER(dispatch_object_s);
};

struct dispatch_object_s {
	DISPATCH_STRUCT_HEADER(dispatch_object_s, dispatch_object_vtable_s);
};
```  

如上，便可以定义一个`dispatch_object_s`的类，由此便创建一个`dispatch_object_s`对象。  

## dispatch\_queue\_s 是如何创建的  

```c
#define DISPATCH_QUEUE_MIN_LABEL_SIZE 64

#ifdef __LP64__
#define DISPATCH_QUEUE_CACHELINE_PAD 32
#else
#define DISPATCH_QUEUE_CACHELINE_PAD 8
#endif

#define DISPATCH_QUEUE_HEADER \
	uint32_t volatile dq_running; \
	uint32_t dq_width; \
	struct dispatch_object_s *volatile dq_items_tail; \
	struct dispatch_object_s *volatile dq_items_head; \
	unsigned long dq_serialnum; \
	dispatch_queue_t dq_specific_q;

struct dispatch_queue_s {
	DISPATCH_STRUCT_HEADER(dispatch_queue_s, dispatch_queue_vtable_s);
	DISPATCH_QUEUE_HEADER;
	char dq_label[DISPATCH_QUEUE_MIN_LABEL_SIZE]; // must be last
	char _dq_pad[DISPATCH_QUEUE_CACHELINE_PAD]; // for static queues only
};
```  

相对于`dispatch_object_s`，`dispatch_queue_s`多了一个`DISPATCH_QUEUE_HEADER`的宏定义和描述这个任务队列的字符串，从上可以看出，我们在外部传入的任务队列描述字符串不能大于 64，否则可能会造成内存问题。  

`DISPATCH_QUEUE_HEADER` 中的参数的作用：  

<font color="green"> dq\_running:</font> 任务队列中，执行的任务数量（还是可同时执行的任务数量？）。  

<font color="green"> dq\_width:</font> 任务队列的宽度，串行队列为 1，并行队列大于 1。  

<font color="green"> dq\_item\_tail:</font> 指向任务队列的尾节点。  

<font color="green"> dq\_item\_head:</font> 指向任务队列的头节点。  

<font color="green"> dq\_serialnum:</font>  任务队列的编号，任务队列的编号可看下面:  

```c
// skip zero
// 1 - main_q
// 2 - mgr_q
// 3 - _unused_
// 4,5,6,7,8,9,10,11 - global queues
// we use 'xadd' on Intel, so the initial value == next assigned
unsigned long _dispatch_queue_serial_numbers = 12;
```  

由此可见，GCD 会默认创建 10 个任务队列，默认编号为 1 到 11。  

<font color="green"> dq\_finalizer\_ctxt:</font> 任务队列释放时调用的函数的参数。  

<font color="green"> dq\_finalizer\_func:</font> 任务对列的释放函数。  

### dispatch\_queue\_create 函数  

```c
// Note to later developers: ensure that any initialization changes are
// made for statically allocated queues (i.e. _dispatch_main_q).
static inline void
_dispatch_queue_init(dispatch_queue_t dq)
{
	dq->do_vtable = &_dispatch_queue_vtable;
	dq->do_next = DISPATCH_OBJECT_LISTLESS;
	dq->do_ref_cnt = 1;
	dq->do_xref_cnt = 1;
	// Default target queue is overcommit!
	dq->do_targetq = _dispatch_get_root_queue(0, true);
	dq->dq_running = 0;
	dq->dq_width = 1;
	dq->dq_serialnum = dispatch_atomic_inc(&_dispatch_queue_serial_numbers) - 1;
}

dispatch_queue_t
dispatch_queue_create(const char *label, dispatch_queue_attr_t attr)
{
	dispatch_queue_t dq;
	size_t label_len;

	if (!label) {
		label = "";
	}

	label_len = strlen(label);
	if (label_len < (DISPATCH_QUEUE_MIN_LABEL_SIZE - 1)) {
		label_len = (DISPATCH_QUEUE_MIN_LABEL_SIZE - 1);
	}

	// XXX switch to malloc()
	dq = calloc(1ul, sizeof(struct dispatch_queue_s) -
			DISPATCH_QUEUE_MIN_LABEL_SIZE - DISPATCH_QUEUE_CACHELINE_PAD +
			label_len + 1);
	if (slowpath(!dq)) {
		return dq;
	}

	_dispatch_queue_init(dq);
	strcpy(dq->dq_label, label);

	if (fastpath(!attr)) {
		return dq;
	}
	if (fastpath(attr == DISPATCH_QUEUE_CONCURRENT)) {
		dq->dq_width = UINT32_MAX;
		dq->do_targetq = _dispatch_get_root_queue(0, false);
	} else {
		dispatch_debug_assert(!attr, "Invalid attribute");
	}
	return dq;
}
```  

如上是`dispatch_queue_create`的源代码，具体逻辑可分为以下：  

1. 分配`dispatch_queue_t`对象`dq`的内存空间，将形参`label`的内容赋值到`dq->dq_label`中，调用`_dispatch_queue_init`初始化`dq`。  
2. 判断是否存在形参`attr`，如果存在，改变`dq`的目标队列，释放函数和释放参数。  

（PS: `slowpath`的解释可参考[深入理解 GCD](https://bestswifter.com/deep-gcd/)）  
当存在`attr`时，会调用到`_dispatch_get_root_queue`，我们可以从中观测到 GCD 的初始化的任务队列是怎么样的：  

```c
static inline dispatch_queue_t
_dispatch_get_root_queue(long priority, bool overcommit)
{
	if (overcommit) switch (priority) {
	case DISPATCH_QUEUE_PRIORITY_LOW:
		return &_dispatch_root_queues[
				DISPATCH_ROOT_QUEUE_IDX_LOW_OVERCOMMIT_PRIORITY];
	case DISPATCH_QUEUE_PRIORITY_DEFAULT:
		return &_dispatch_root_queues[
				DISPATCH_ROOT_QUEUE_IDX_DEFAULT_OVERCOMMIT_PRIORITY];
	case DISPATCH_QUEUE_PRIORITY_HIGH:
		return &_dispatch_root_queues[
				DISPATCH_ROOT_QUEUE_IDX_HIGH_OVERCOMMIT_PRIORITY];
	case DISPATCH_QUEUE_PRIORITY_BACKGROUND:
		return &_dispatch_root_queues[
				DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_OVERCOMMIT_PRIORITY];
	}
	switch (priority) {
	case DISPATCH_QUEUE_PRIORITY_LOW:
		return &_dispatch_root_queues[DISPATCH_ROOT_QUEUE_IDX_LOW_PRIORITY];
	case DISPATCH_QUEUE_PRIORITY_DEFAULT:
		return &_dispatch_root_queues[DISPATCH_ROOT_QUEUE_IDX_DEFAULT_PRIORITY];
	case DISPATCH_QUEUE_PRIORITY_HIGH:
		return &_dispatch_root_queues[DISPATCH_ROOT_QUEUE_IDX_HIGH_PRIORITY];
	case DISPATCH_QUEUE_PRIORITY_BACKGROUND:
		return &_dispatch_root_queues[
				DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_PRIORITY];
	default:
		return NULL;
	}
}

// 默认任务队列
DISPATCH_CACHELINE_ALIGN
struct dispatch_queue_s _dispatch_root_queues[] = {
	[DISPATCH_ROOT_QUEUE_IDX_LOW_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_LOW_PRIORITY],

		.dq_label = "com.apple.root.low-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 4,
	},
	[DISPATCH_ROOT_QUEUE_IDX_LOW_OVERCOMMIT_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_LOW_OVERCOMMIT_PRIORITY],

		.dq_label = "com.apple.root.low-overcommit-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 5,
	},
	[DISPATCH_ROOT_QUEUE_IDX_DEFAULT_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_DEFAULT_PRIORITY],

		.dq_label = "com.apple.root.default-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 6,
	},
	[DISPATCH_ROOT_QUEUE_IDX_DEFAULT_OVERCOMMIT_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_DEFAULT_OVERCOMMIT_PRIORITY],

		.dq_label = "com.apple.root.default-overcommit-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 7,
	},
	[DISPATCH_ROOT_QUEUE_IDX_HIGH_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_HIGH_PRIORITY],

		.dq_label = "com.apple.root.high-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 8,
	},
	[DISPATCH_ROOT_QUEUE_IDX_HIGH_OVERCOMMIT_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_HIGH_OVERCOMMIT_PRIORITY],

		.dq_label = "com.apple.root.high-overcommit-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 9,
	},
	[DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_PRIORITY],

		.dq_label = "com.apple.root.background-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 10,
	},
	[DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_OVERCOMMIT_PRIORITY] = {
		.do_vtable = &_dispatch_queue_root_vtable,
		.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
		.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
		.do_ctxt = &_dispatch_root_queue_contexts[
				DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_OVERCOMMIT_PRIORITY],

		.dq_label = "com.apple.root.background-overcommit-priority",
		.dq_running = 2,
		.dq_width = UINT32_MAX,
		.dq_serialnum = 11,
	},
};
```  

从上可以看出，这 8 个全局队列的宽度都是为无限大的，则为并行对列，而它的引用计数和外部引用计数都输无限大，那它是不会被销毁的。挂起系数都为 1，不同的只是`label`和`ctxt`，`ctxt`主要是不同的信号量，由此来实现对列任务的优先级。值得一提的是，它们的`vtable`都是`_dispatch_queue_root_vtable`，由此它们的`probe`函数都是`_dispatch_queue_wakeup_global`，这个函数比较重要，是产生多线程的关键函数，后续文章会有所说明。  

由此也可以看出，如果`dispatch_queue_create`第二个形参传入不空且符合一定要求，创建的队列的目标队列都是并行队列。  

这里再举出主任务队列的源码：

```c
DISPATCH_CACHELINE_ALIGN
struct dispatch_queue_s _dispatch_main_q = {
#if !DISPATCH_USE_RESOLVERS
	.do_vtable = &_dispatch_queue_vtable,
	.do_targetq = &_dispatch_root_queues[
			DISPATCH_ROOT_QUEUE_IDX_DEFAULT_OVERCOMMIT_PRIORITY],
#endif
	.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
	.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
	.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
	.dq_label = "com.apple.main-thread",
	.dq_running = 1,
	.dq_width = 1,
	.dq_serialnum = 1,
};
```  

主任务队列的目标队列是优先级为默认的全局队列。  

还有一个`mgr-q`：

```c
DISPATCH_CACHELINE_ALIGN
struct dispatch_queue_s _dispatch_mgr_q = {
	.do_vtable = &_dispatch_queue_mgr_vtable,
	.do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
	.do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
	.do_suspend_cnt = DISPATCH_OBJECT_SUSPEND_LOCK,
	.do_targetq = &_dispatch_root_queues[
			DISPATCH_ROOT_QUEUE_IDX_HIGH_OVERCOMMIT_PRIORITY],

	.dq_label = "com.apple.libdispatch-manager",
	.dq_width = 1,
	.dq_serialnum = 2,
};
```  

它的目标对列是任务优先级最高的全局队列，这个主要是用来处理 GCD 默认线程的任务处理（PS: GCD 会在初始化的时候，默认创建一个消息线程，用来检测软件崩溃，接收内核消息的线程）。  

`DISPATCH_CACHELINE_ALIGN`是设置结构体 64 字节对齐  

本篇文章先不讲述其他 GCD 的数据类型了，以后我更新了，会在这里接上去。  

# GCD 中的一些宏定义  
在上面的一个链接已经很清楚的讲述了 GCD 的`fastpath`和`lowpath`的使用，这里不再讲述。  

GCD 中有一些原子操作的宏定义，是基于 GCC 的原子操作，具体内容可看[变态的libDispatch结构分析-原子操作方法](http://www.aiuxian.com/article/p-1827537.html)，这里只讲述这些原子操作的具体含义：

<font color="orange"> dispatch\_atomic\_cmpxchg(A, B, C):</font> 将 A 和 B对比，相等，则将 C 赋值给 A，返回 YES，否则返回 NO。（PS，著名的 CAS 操作）  

<font color="orange"> dispatch\_atomic\_xchg(A, C):</font> 将 C 赋值给 A , 返回赋值前的 A 。  

<font color="orange"> dispatch\_atomic\_inc(A):</font> A 自增1。  

<font color="orange"> dispatch\_atomic\_dec(A):</font>  A 自减1。  

<font color="orange"> dispatch\_atomic\_add(A, B):</font> A = A + B。  

<font color="orange"> dispatch\_atomic\_sub(A, B):</font> A = A - B。  

<font color="orange"> dispatch\_atomic\_or:</font> A = A | B 。  

<font color="orange"> dispatch\_atomic\_and(A, B):</font> A = A & B 。

<font color="orange"> \_dispatch\_hardware\_pause():</font> 这个宏定义是`asm("pause")`，我查了一下这方面的资料[__asm__(＂pause＂)用法](http://www.ithao123.cn/content-629618.html)，这个主要是在 x86 架构的 CPU 在循环中使用这个汇编指令，降低功耗，确保循环不会退出。苹果手机和现在市场上的几乎百分百安卓机都是 ARM 架构的，作用应该不大，我以前在 ARM 单片机编程中更喜欢使用的是`nop`，延迟一下，省点功耗。  

# TSD  
在 pThread 中，每个线程都有自己专有的存储空间，是以 Key-Value 的形式存储，每条线程总共可以存储 128 个 Key-Value，这就是 TSD。在 GCD 中也采用了这一个技术，主要是为了存储每个线程 GCD 的环境，在 187.9 的 GCD 中，有六个 Key:  

```c
#if DISPATCH_USE_DIRECT_TSD
static const unsigned long dispatch_queue_key		= __PTK_LIBDISPATCH_KEY0;
static const unsigned long dispatch_sema4_key		= __PTK_LIBDISPATCH_KEY1;
static const unsigned long dispatch_cache_key		= __PTK_LIBDISPATCH_KEY2;
static const unsigned long dispatch_io_key			= __PTK_LIBDISPATCH_KEY3;
static const unsigned long dispatch_apply_key		= __PTK_LIBDISPATCH_KEY4;
static const unsigned long dispatch_bcounter_key	= __PTK_LIBDISPATCH_KEY5;
//__PTK_LIBDISPATCH_KEY5
```  

<font color="#FF44FF"> dispatch\_queue\_key</font> 存储当前线程的任务对列。  

<font color="#FF44FF"> dispatch\_sema4\_key</font> 存储信号量，保证串行任务队列执行顺序，串行对列死锁的原因也在于此。  

其他具体作用，在我看了源码会添加进去。  