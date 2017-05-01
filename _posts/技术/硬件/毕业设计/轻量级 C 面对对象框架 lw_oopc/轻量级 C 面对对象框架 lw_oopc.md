title: 轻量级 C 面对对象框架 lw_oopc  
description: 简单介绍轻量级 c 面对对象框架 lw_oopc  
date: 2017/5/1 18:00  
category: 毕业设计  
comments: true  
toc: true  

---
# c 为什么要面对对象
最近公司忙的要死，都没写啥东西，趁要做毕业设计，请了一个月假，好好的弄点东西。  
我的毕业设计的初衷是弄一个在 stm32 运行的协程式嵌入式操作系统（非抢占式）。如果一味的应用面对过程的思想去弄，那必然是崩溃的，是不可能完成的，所以必须在 C 中应用面对对象思想，而 C 中最好体现这个思想的便是结构体了，合理的运用结构体可以比较完美的写出类和对象，封装和继承。我也认为这是嵌入式程序员必须掌握的技能，不然，程序的状态多的让你处理不过来，代码的耦合性强，难以维护。可能有人会疑惑，为什么不用 C++ ，C++ 完美的达到了这些要求，而且在面对对象的实现是碾压 C 语言的，C++ 同样可以在很多嵌入式平台应用。我之所以不选择 C++ 的原因有三： 
 
1.  C 语言更具有广泛性，如果有一门语言是全栈语言，我会给 C 语言投一票，而不是那些脚本语言。有哪家芯片厂商敢不支持 GCC ？但 C 的难度确保了它无法成为一门全栈语言，所以才有 JavaScript 之类的脚本语言异军突起，但讲到性能，C 对脚本语言来说是高山仰止。  
2.  很多嵌入式程序员都是以 C 语言起家的，在嵌入式领域，C 的统治力是无可撼动的，虽然有些单片机厂商支持了 C++ ，但对于嵌入式程序员更多的喜欢用 C 语言。
3.  我对 C++ 的熟练度远比不上 C 。 

# lw_oopc
lw_oopc 是台湾高焕堂先生和金永华先生写的一套 C 关于面对对象的宏定义，是一种 C 语言编程框架，支持了 C 语言中进行面对对象的编程，是一套极其强大的宏，我将在我的毕业设计，以它为一个基础搭建我的毕业设计，当然其中我会加入我的一些想法。如果涉及到侵权，请告知我，我会删掉侵权代码。接下来，我将尽可能剖析 lw_oopc 库，可以看出两位作者对宏的高深功力。相关代码在 [嵌入式操作系统 ZHOS](https://github.com/suancaiAmour/ZHOS.git) ZHOS 文件夹内的`lw_oopc.h`和`lw_oopc.c`文件中。   

## lw_oopc 抽象类  
在面对对象中，还有更升华的一种思想是面对协议，我个人是很推崇面对协议这种概念的。面对协议在解决代码复用，代码耦合性上，是一种很有效的方法，而且对于程序员读懂代码是很有用的一种方法。为此，在 lw_oopc 中，设计了类似 Java 类中的 Interface 和 抽象类，但我个人是把 Interface 去除掉了。抽象类本来跟 Interface 是不同的两个东西，但很多情况可以理解复用，为了简便，我就把 Interface 给去除掉，只剩下抽象类。抽象类是不能创建对象的，只能以类的形似存在，这样便也可以把它当做协议，让抽象类的子孙必须实现这些协议。下面是抽象类的实现方式:  

```c
#define ABS_CLASS(type)             \
typedef struct type type;           \
void type##_ctor(type* t);          \
void type##_dtor(type* t);          \
void type##_release(type* t);       \
void type##_retain(type* t);        \
struct type                         \
{   								\
     int referenceCount;            \

#define END_ABS_CLASS };

#define ABS_CTOR(type)              \
void type##_ctor(type* cthis) {

#define END_ABS_CTOR }

// 根类
ABS_CLASS(rootClass)
END_ABS_CLASS
```  

`referenceCount`属性是标志引用计数，在 lw_oopc 中生成的对象，我采取了引用计数的方法，这样对象拥有中进行内存管理比较方便。在引用计数中，每一个对象负责维护对象所有引用的计数值。当一个新的引用指向对象时，引用计数器就递增，当去掉一个引用时，引用计数就递减。当引用计数到零时，该对象就将释放占有的资源。在这样的管理中，能尽量的防止野指针和内存泄露的出现，但这并不能完全处理掉内存泄露和野指针，即使使用了引用计数，也需要程序员高水平的使用。在 lw_oopc 中，创建一个引用计数默认是1(在下面非抽象类的 `type##new`中可以看到)，调用一次 release 函数，引用计数减一，调用一次 retain 函数，引用计数加1。在 lw_oopc 中，并没有实现一个对象拥有另一个对象时候，引用计数自动加1，当一个对象引用一个对象的时候，还是需要开发者调用 retain 去引用计数加1，这也是 lw_oopc 中的一个很大的不足。这也就要求开发者有很强的内存管理意识，不过开发者可以放心，在 lw_oopc 中，有检测内存泄露的方法，开发者可以检测出哪里内存泄露了，进行处理，这个稍后再说。  

`rootClass`是一个根类，如果不需要实现某些特定的协议，我建议可以继承这个`rootClass`，因为非抽象类的实现都需要继承一个父类，这跟 OC 的 NSObject 很像，因为这两种其实都是 C 语言，无法躲过 C 语言的约束。但这也很好的保证了子类必须采取引用计数的方案，统一内存管理的方案，这也是一种好处。  

## lw_oopc 类
类是面对对象的核心所在，类可以表示万物，用不同的类可以表示不同的东西，也可以用类集成一些比较羞涩抽象的概念。面对对象的理念就是万物都可以用类表示，而对象就是万物的具体表现。类的形成可以从人类文化成长可以看出一点端倪。我们或许听过一个故事，画一个圈，问小学生，小学生会答是盘子，是碟子，是足球等等，而高中生只会答是圆，最后得出教育毁灭了孩子的创造力，这其实是一种混淆概念。当人未学的文化时，人类通常都是 _口头文化_ 的表现，而当人类学到知识，就会有一种 _书本文化_ 的表现，所以小学生会答很多种具体的对象，而高中生只会答圆。高中生已经具有一定的 _书本文化_ 在他们潜意识中，会将看到的东西抽象起来，看做一个类，然后回答出来，而 _口头文化_ 的小学生则是把它看做各种对象回答，这并不是创造力的问题，而是两者文化不同的表现，人类学的知识，更喜欢统计起来，抽象起来，这样文化才能更好的传承。这也是面对对象中类的意义所在。下面是 lw_oopc 中类是如何实现的：  

```c
#define CLASS(type, Father)                 \
typedef struct type type;           \
type* type##_new(lw_oopc_file_line_params); \
void type##_ctor(type* t);          \
void type##_dtor(type* t);          \
void type##_release(type* t);       \
void type##_retain(type *t);        \
struct type                         \
{                                   \
    void (*dealloc)(type* cthis);   \
    struct Father father;

#define END_CLASS  };

#define CTOR(type, Father)                              \
type* type##_new(const char* describe)                  \
{            											\
    struct type *cthis = (struct type*)lw_oopc_malloc(sizeof(struct type), #type, describe);   \
    cthis->father.referenceCount = 1;                   \
    if(!cthis)                                          \
    {                                                   \
        return 0;                                       \
    }                                                   \
    type##_ctor(cthis);                                 \
    return cthis;                                       \
}                                                       \
void type##_release(type* cthis)                    \
{                                                   \
     if(--cthis->father.referenceCount == 0)         \
     {                                               \
         cthis->dealloc(cthis);                      \
         lw_oopc_free(cthis);                        \
     }                                               \
}                                                   \
void type##_retain(type* cthis)                         \
{                                                       \
     cthis->father.referenceCount++;                    \
}                                                       \
void type##dealloc11111(type* cthis){}                  \
void type##_ctor(type* cthis) {                         \
    cthis->dealloc = type##dealloc11111;                \
    Father##_ctor(&(cthis->father));

#define END_CTOR	} \
```  

要想用类生成具体的对象，必须调用 `type##_new` 函数，当调用此函数的时候，会调用 `lw_oopc_malloc`分配具体的空间，然后将引用计数默认为1，最后调用 `type##_ctor` 初始化对象内部的函数。  

函数 `type##_ctore`是由宏定义 `CTOR(type, Father)` 生成的，在这个宏定义内调用另一宏定义 `FUNCTION_SETTING(f1, f2)` 初始化对象内部的函数。这个调用操作是由开发者调用的，因为只有开发者才知道对象内部的函数需要什么函数。  

```c
#define FUNCTION_SETTING(f1, f2)	cthis->f1 = f2
```  
`FUNCTION_SETTING(f1, f2)` 很简单，其实就是向结构体内的 f1 赋值 f2。这样也就能向对象内部的函数指针赋值。  

其他几个宏定义不是那么重要，这里就不在提起，开发者可以自行看看，很简单，主要是实现多态之类的需求。  

值得一提的是，每个对象都有一个 `type##_dealloc` 函数，在对象释放之前都会调用这个函数，默认的这个函数是没什么作用的，但如果开发者对这个函数指针进行了赋值，就可以实现在释放对象前调用指定的函数。  


# lw_oopc 使用举例
在 `ZHOS` 中，有对 lw_oopc 具体的使用，也就是队列的类的创建，具体在 ZHOS 文件夹内的 `queue.h` 和 `queue.c` 文件中。  

```c
#ifndef _QUEUE_H_
#define _QUEUE_H_

#include "lw_oopc.h"

CLASS(ZHQueue, rootClass)
void **cache;
int front;
int count;
int maxCount;
void (*init)(ZHQueue *cthis, int maxCount);
void (*enQueue)(ZHQueue *cthis, void *con);
void *(*deQueue)(ZHQueue *cthis);
void *(*getConWithIndex)(ZHQueue *cthis, unsigned int index);
END_CLASS

#endif
```  

如上，是 `queue.h` 文件中的内容，声明一个 `ZHQueue` 类，继承于 `rootClass` 。 `ZHQueue` 类里面属性有 `cache`(用于存储对列的具体内容)，`front`(队列的头指针)，`count`(队列内容的数量)，`maxCount`(队列所能拥有内容的最大数量)，剩下的是队列的一些操作，比如说初始化，出列入列等。  

```c
#include "queue.h"
#include "log.h"
#include "memory.h"
#include <stddef.h>

void queueInit(ZHQueue *cthis, int maxCount)
{
	cthis->cache = (void **)mymalloc(sizeof(void *) * maxCount);
	if(cthis->cache == NULL)
	{
		ZHLog("´´½¨¶ÓÁÐÊ§°Ü¡£\r\n");
		cthis->maxCount = 0;
	}
	else
	{
		cthis->maxCount = maxCount;
	}
}

void enQueue(ZHQueue *cthis, void *con)
{
	int rear;
	if (cthis->count == cthis->maxCount)
	{
		ZHLog("¶ÓÁÐÒç³ö¡£\r\n");
	}
	else
	{
		rear = (cthis->front + cthis->count) % cthis->maxCount;
		rear = (rear + 1) % cthis->maxCount;
		cthis->cache[rear] = con;
		cthis->count++;
	}
}

void *deQueue(ZHQueue *cthis)
{
	if(cthis->count == 0)
	{
		ZHLog("¶ÓÁÐÎª¿Õ,³öÕ»Îª¿Õ¡£\r\n");
		return NULL;
	}
	else
	{
		cthis->front = (cthis->front + 1) % cthis->maxCount;
		cthis->count--;
		return cthis->cache[cthis->front];
	}
}

void *getConWithIndex(ZHQueue *cthis, unsigned int index)
{
	unsigned int conIndex;
	if(cthis->count == 0)
	{
		ZHLog("¶ÓÁÐÎª¿Õ,»ñÈ¡ÄÚÈÝÊ§°Ü¡£\r\n");
		return NULL;
	}
	else
	{
		conIndex = (cthis->front + index + 1) % cthis->maxCount;
		return cthis->cache[conIndex];
	}
}

void queueDealloc(ZHQueue *cthiss)
{
	myfree(cthiss->cache);
}

CTOR(ZHQueue, rootClass)
FUNCTION_SETTING(init, queueInit);
FUNCTION_SETTING(enQueue, enQueue);
FUNCTION_SETTING(deQueue, deQueue);
FUNCTION_SETTING(getConWithIndex, getConWithIndex);
FUNCTION_SETTING(dealloc, queueDealloc);
END_CTOR

```  

如上是 `queue.c` 文件的内容，主要是对 queue 对象的函数指针赋值，注意这里只是生成了 `type##_ctor`函数，并没有调用它，必须在使用中调用 `type##_new` 函数去触发它，如 `ZHQueue` 的创建是这样的：

```c
	TCBQueue = ZHQueue_new("ÏµÍ³ÈÎÎñ¶ÓÁÐ");
	TCBQueue->init(TCBQueue, MAX_TASK); // ÈÎÎñ¶ÓÁÐÖ»ÄÜ´æ MAX_TASK ¸öÈÎÎñ 
```  

`ZHQueue_new`的形参是对 `TCBQueue` 的解释，这样做的目的是方便调试，根据解释我们就可以精确到哪个对象内存泄露。  

# 如何实现对内存泄露的检测  
实现对内存泄露的检测主要是在 `lw_oopc_malloc`，`lw_oopc_free` 和 `lw_oopc_report` 三个函数实现，这三个函数实现在 `le_oopc.c` 文件中。  

实现原理简单说明：每生成一个对象，都会将对象的信息放在 `LW_OOPC_MemAllocUnit` 的链表里面，释放对象的时候，都会从 `LW_OOPC_MemAllocUnit` 链表中去除这个对象的信息，当调用 `lw_oopc_report` 就会遍历 `LW_OOPC_MemAllocUnit ` 链表，如果有内存泄露，链表并不为空，将链表保存的信息打印出来，我们便知道哪个对象是内存泄露了，由此可以精确解决内存泄露。注意，如果使用 `mymalloc` 函数将会跳过这个内存泄露的检测方式。  
