title: ZHOS 中的任务调度器 
description: 简单介绍 ZHOS 的协程任务调度理念  
date: 2017/5/6 16:00  
category: 毕业设计  
comments: true  
toc: true  

---

# 什么是协程  
协程在最近几年的编程概念中变得很火，比较新的语言都支持协程的编程方式，比如说 go，lua，python 等。但实际上，协程是一个很老的概念，早在 20 世纪 60 年代末期，协程这个概念就已经提出来了，编程大师 D. E. Knuth 把协程概念提出者归功于 Melvin Conwa。早期的协程是为了程序的编译器，而今协程更多的使用于阻塞式的线程当中，特别是那些网络协议线程。近几年的网络服务框架开始大规模使用协程，微信后台就使用腾讯自己的携程库 libco，libco 在腾讯的服务器已经广泛使用，对于它的效率是得到验证的。我个人认为协程是线程的更小一级 CPU 运作单位，协程基于线程，线程和协程的模式是 1 ： N 的，这表示一条线程可以有几条协程。协程其实是一种函数跳跃方法，在执行一个函数的时候，可以跳跃到另一个函数，跟线程不同的是，这两个函数同属于一条线程上，这样也就不存在线程安全的问题，这对开发者来说是一种莫大的解放。协程在函数跳跃中，保存了跳跃前的函数的所有数据，这也就保存了函数编程中的一个重要概念--状态，将状态保存了，为后续的编程带来的方便是很大的。这也带来了一个问题，为了保存状态，一条协程必须拥有自己的栈，这对于内存来说是一种挑战，幸好而后的开发者用一种共享栈的方式，也减轻了内存的压力，但损失了效率。在现实中，相对于 CPU 的计算，内存是很廉价的，共享栈并没有在协程中大量使用。协程分为两种方式：

1. 非对称协程，协程跳转函数，必须跳转回它的调用者函数，由调用者函数再跳转到其他函数，这样，协程是无法跳转到其他线程的协程的。  
2. 对称协程，协程不需要跳转到它的调用者函数，有些对称协程库甚至可以允许协程跳转到其他线程的协程。  

协程在编程史上，在这几年才得到较多的关注，在 20 世纪时，它是被开发者所唾弃的，这也是因为协程跟当时的编程理念所冲突导致的。  

## 从 goto 语句说起  
goto 语言时至今日还是不能被很多开发者所接受的。在我们的教科书中，goto 语句是不建议使用的，这也就造成了很多开发者认为使用 goto 语句是不可接受，使用 goto 语句是一名开发者水平低下的表现。这是一个很大很大的错误，goto 语句遭到大规模排挤是因为它是一种无条件跳转语句，开发者怕在跳转中破坏了自己的程序结构，在数学上已经证明，程序的三种基本结构，顺序，条件和循环可以编译出所有的程序，那这种无条件跳转明显是没必要的。在 80 年代，C 语言是所有语言中最多人使用的语言，C 语言自上而下的理念也深入人心，像这种无条件跳转是不能被 C 的理念所接受，但实际上，C 还是保留了这种无条件跳转的特性。因为，很多时候，无条件跳转是我们必须要使用的，不然造成代码的累赘是很大的。至今，linux 的内核代码还有 2 万多的 goto 语句，那是什么情况下使用语句呢？大部分情况都是 _统一资源释放接口_ 是使用 goto 语句，不然，在一个函数中，大量的 retuen 语句会给代码逻辑带来冲击且代码很多是一模一样的，D. Knuth 也曾专门写过文章给 goto 语句辩护过。正如这种无条件跳转不能被开发者所接受，那协程的更加过分的函数间无条件跳转更是不能让人接受的，所以协程一度黯淡无声，而今协程也开始伟大起来了，因为它带来的效率和逻辑上的优化的诱惑是开发者所不能阻挡的。  

# ZHOS 中的任务  
ZHOS 每一个协程都起源于`ZHOSTask`类型的函数。  

```c
#define ZHOSTask void (*)(void *)
```  

创建任务是由`CreateTaskWithTackDeep`实现的：

```c
void CreateTaskWithTackDeep(void (*task)(void *), void *env, uint32_t tackDeep)
{
	OS_STK *ptos; // ÈÎÎñ¶ÑÕ»
	TCB *newTask; // ÐÂ´´½¨µÄÈÎÎñÀà
	if(NULL == task)
		return;
		
	ptos = (OS_STK *)mymalloc(sizeof(OS_STK) * tackDeep);
	if(NULL == ptos)
	{
		ZHLog("ÈÎÎñÕ»´´½¨Ê§°Ü¡£\r\n");
		return;
	}
	newTask = (TCB *)mymalloc(sizeof(TCB));
	newTask->freeAddr = ptos;
    ptos += (TASK_TACK_DEEP - 1);                                          
    *(ptos)    = (INT32U)0x01000000L;             
    *(--ptos)  = (INT32U)task;
	*(ptos - 6) = (INT32U)env;
	newTask->pTaskStack = ptos - 14;
	newTask->Delay = 0;
	// Ñ¹ÈëÈÎÎñ¶ÓÁÐ
	TCBQueue->enQueue(TCBQueue, (void *)newTask);
	taskCount = TCBQueue->count;
}
```  

`CreateTaskWithTackDeep`主要实现任务堆栈的实现，任务上下文的搭建，且将`ZHOSTask`的形参赋值到寄存器 R0 上，也就是`CreateTaskWithTackDeep`的形参`env`。由此被创建了一个任务上下文，并且将任务上下文压进调度器的队列中。  

# ZHOS 实现非阻塞式延时  
在 ZHOS 实现延时主要是由`SwitchDelay`和 STM32 滴答定时器实现的：

```c
void  SysTick_Handler (void)
{   
	uint8_t i;
	TCB *task;
	sysTickTime++;
	for(i = 0; i < taskCount; i++)
	{
		task = TCBQueue->getConWithIndex(TCBQueue, i);
		if(task->Delay != 0)
		{
			task->Delay--;
		}
    }
}

uint64_t getOSTickTime(void)
{
	return sysTickTime;
}

void SwitchDelay(uint16_t nTick)
{
     uint8_t i;
	 TCB *task;
	 TaskRuning->Delay = nTick;	 
	 for(i = 0; i < taskCount; i++)
	 {
		task = TCBQueue->getConWithIndex(TCBQueue, i);
		if(task->Delay == 0)
		{
			if(task == TaskRuning || task == IdleTask)
			{
				continue;
			}
			else
			{
				TaskNew = task;
				break;
			}
		}
	 }
	 
	 if(i == taskCount)
	 {
		 if (task->Delay != 0)
		 {
			 TaskNew = IdleTask;
		 }
		 else return;
	 }
	 TaskSwitch();
}
```  

在滴答定时器的异常服务函数中会将任务队列所有任务延时系数不为 0 的任务的延时系数自减 1。  

在`SwitchDelay`中会查询任务队列中的任务，检测到任务的延时系数不为 0 的任务，就切换到这个任务，这样就没有阻塞住 CPU 的执行了。当没有任务执行的时候，就切换到空闲任务：  

```c
void TaskIdle(void *env) 
{
	while(1)
	{
		__ASM("WFE");
		SwitchDelay(0);
	}
}
```  

空闲任务主要进入睡眠，当有异常触发的时候，重新切换任务。 ZHOS 中，滴答定时器会一毫米触发一次，如此空闲任务也会一毫米触发一次。  

# ZHOS 中的协程信息传递  
ZHOS 协程的信息可以从创建协程的时候需要传递的形参进行信息传递。  

ZHOS 中还有条件变量进行协程间交互。协程是同一条线程中的，是不存在线程安全这个概念的，也就不存在需要锁的机制之类的。这里条件变量是为了方便协程之间的信息传递，这样实现经典的生产者消费者的问题显得异常简单。 ZHOS 条件变量由一下几个函数实现：

```c
ZHOSCond *createCond(int condNum)
{
	ZHOSCond *cond = ZHOSCond_new("Ð­³ÌÌõ¼þ±äÁ¿");
	cond->CondNum = condNum;
	return cond;
}

void releaseCond(ZHOSCond *cond)
{
	ZHOSCond_release(cond);
}

void condSignal(ZHOSCond *cond)
{
	cond->CondNum--;
}

// ·µ»Ø 0 ±íÊ¾ ½ÓÊÕµ½Ìõ¼þ±äÁ¿
// ·µ»Ø 1 ±íÊ¾ ³¬Ê±
// waitTime Îª -1 ±íÊ¾ÓÀ¾ÃµÈ´ý
int condWait(ZHOSCond *cond, int64_t waitTime)
{
	uint64_t cuTime = getOSTickTime();
	while(1)
	{
		if(cond->CondNum <= 0)
		{
			cond->CondNum++;
			return 0;
		}
		else if(getOSTickTime() - cuTime >= waitTime)
		{
			return 1;
		}
		else
		{
			SwitchDelay(0);
		}	
	}
}
```  

主要是检测 ZHOSCond 中的 CondNum 是否为 0，不为 0 就切换任务。还实现了条件变量的等待超时，这样可以在更多的情景得到应用。  

# ZHOS 协程实例应用  
在 ZHOS 的`APP.c`中，有 ZHOS 的应用举例：  

```c
void producer(void *args)
{
	ZHOSCond *cond = (ZHOSCond *)args;
	while (true)
	{
		ZHOS.condSignal(cond);
		ZHOS.log("Éú²úÕßÉú²úÒ»´Î¡£\r\n");
		ZHOS.delay(10000);
	}
}

void consumer(void *args)
{
	int result;
	ZHOSCond *cond = (ZHOSCond *)args;
	while (true)
	{
		ZHOS.log("Ïû·ÑÕßµÈ´ýÉú²ú¡£\r\n");
		result = ZHOS.condWait(cond, 50000);
		if(result)
		{
			ZHOS.log("³¬Ê±¡£\r\n");
		}
		else
		{
			ZHOS.log("Ïû·ÑÕßÏû·ÑÒ»´Î¡£\r\n");
		}
	}
}


int main(void)
{
	ZHOSCond *cond;
	ZHOSInit();
	cond = ZHOS.createCond(1);
	ZHOS.createTask(consumer, cond);
	ZHOS.createTask(producer, cond);
	ZHOS.switchTask();
	while(1);
}
```  
实现了经典的生产者消费者问题，在上面，生产者和消费者相互打印生产信息。  

# ZHOS 的其他内容  
ZHOS 还实现一些不是核心的内容，比如说，探针功能，利用 STM32 的串口 1 打印消息，这是很有必要的，在调试中，我们需要这些信息进行调试。  