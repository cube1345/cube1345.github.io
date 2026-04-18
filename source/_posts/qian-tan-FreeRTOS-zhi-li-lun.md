---
title: 浅谈 FreeRTOS 之理论（一）：队列、TCB 与互斥量
date: 2026-04-18 22:10:00
updated: 2026-04-18 22:10:00
categories:
  - 嵌入式系统
tags:
  - FreeRTOS
  - RTOS
  - 嵌入式
  - 队列
  - 互斥量
description: 从 FreeRTOS 的队列机制讲起，梳理任务阻塞、ISR 级 API、TCB、PendSV 上下文切换、优先级反转和互斥量的基本原理。
top_img: false
toc: true
---


笔者是很久之前学习[RTOS](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=RTOS&zhida_source=entity)了，当时学了很多概念一知半解，然而实际开发的时候由于爱好屎山的缘故，基本上也不会用到那些概念。

最近对底层有学习的需求，我想将[FreeRTOS](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=FreeRTOS&zhida_source=entity)的一些概念拾起来。

因此，本系列文章的主要特点是：

* 本系列并不会具体到某一个实际的项目中（因为笔者的FreeRTOS项目无一例外的屎山哈哈哈）
* 本系列侧重于基于[RTOS官方文档](https://link.zhihu.com/?target=https%3A//www.freertos.org/zh-cn-cmn-s/Documentation/00-Overvie)以及个人理解对理论展开解释（可能有误或不准确的地方，还请多多指点）
* 本系列文档可能对一些概念（任务、栈、队列）并不会解释，总之笔者懂的概念就懒得写（逃
* 事实上，本文档并不适合入门，如果您有对部分概念不了解的地方，欢迎一起交流讨论
* 之后有机会也会分享一些实际开发遇到的问题

## 谈谈RTOS以及FreeRTOS

RTOS是指能够在确定的时间约束内完成特定任务的操作系统，而FreeRTOS 是一款开源的、适合资源紧张的MCU的实时操作系统内核（之后会对FreeRTOS的源码展开分析）。我们常常在一些[ARM Cortex-M](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=ARM+Cortex-M&zhida_source=entity)系列的MCU里使用它。

## 队列

在不同任务间，我们可能会需要将一个任务的参数传递给另一个任务作为其一个输入量，我们需要借助队列做到。队列是任务间通信的主要形式。在大多数情况下，队列用作线程安全的 FIFO（先进先出）缓冲区， 新数据被发送到队列的后面，但也可以发送到前面。

e.g. [xQueueSendToFront](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=xQueueSendToFront&zhida_source=entity)()这个函数允许你将一些数据发送到队列前面，使一些紧急的数据可以被队列优先取出。

![](https://pica.zhimg.com/v2-cbc1994cdcb2e4466ec32a20d197d4d0_1440w.jpg)

### Queue 结构

在FreeRTOS里面，一个队列都可以分成两个部分，我们叫他 Queue Control 以及 Queue Item。

```text
Queue Control里面存放这一个队列的数据信息，belike：

uxLength          = 3        ← 总共有 3 个格子                
uxItemSize        = 8        ← 每个格子 8 字节                
uxMessagesWaiting = 0        ← 队列里有0个数据等待被取走                      
pcWriteTo         = Item 0   ← 下次写到这儿                   
pcReadFrom        = Item 2   ← 下次从这儿读（初始化特殊位置）    
xTasksWaitingToSend    = {空}  ← 没有任务在等"不满"            
xTasksWaitingToReceive = {空}  ← 没有任务在等"不空"
```

而Queue Item存放的就是我们任务间需要传递的具体数据了，值得注意的是环形服用。

举一个例子：

![](https://pica.zhimg.com/v2-fbc29542fac258171262881201998ba2_1440w.jpg)

假设说，Item 0 原来存了一个ID为1的数据（它先进去，也先被接收），此后将会轮到ID为2、3的数据被接收，但于此同时，Id为4的数据会被传入Item 1（其实是Item 0 我画错了...）总之，这个队列是一个环形的缓冲区，Id为1的数据被接收后，这个空出来的位置变成了这个队列的尾部，Id为4的数据按照顺序会被插入到原来Id为1的地方，形成上图。

这里Id为4的数据是正常插入，并不是上文提到的插队。

### 阻塞机制

```c
xQueueSend(queue, &value, xTicksToWait);
```

如果这个队列未满，则会立即拷贝数据；如果这个队列已满，则发送任务会进入到xTasksWaitingToSend 列表，并阻塞xTicksToWait 个Tick，如果在超时前有空间，则唤醒继续发送。

```c
xQueueReceive(queue, &buffer, xTicksToWait);
```

同理，如果队列非空，则立即取出数据；如果队列空了，则进入 `xTasksWaitingToReceive`列表阻塞等待。如果超时，则会报错。

**假设现在多个任务被阻塞在同一个队列...**

* **`xTasksWaitingToSend`** ：当数据被取出、队列有空位时，**优先级最高**的任务被唤醒；若优先级相同，**等待最久**的任务被唤醒
* **`xTasksWaitingToReceive`** ：当数据被放入时，同样按**优先级 + 等待时间**唤醒

其目的是保证高优先级任务优先获得资源。

### 队列之创建

**动态创建**

```c
QueueHandle_t q = xQueueCreate(10, sizeof(int32_t));
```

// 创建容纳10个int32_t 数据的队列，从FreeRTOS堆自动分配空间给控制块和数据区。最后会使用 vQueueDelete() 销毁。

**静态创建**

```c
 QueueHandle_t xQueueCreateStatic(
    UBaseType_t uxQueueLength,
    UBaseType_t uxItemSize,
    uint8_t *pucQueueStorageBuffer,      // 用户提供数据缓冲区
    StaticQueue_t *pxQueueBuffer         // 用户提供控制块
);
```

简单来说，静态创建就是由用户指定数据区域与控制区域的空间。

### 关于ISR

FreeRTOS里，另有一套ISR级API，即在中断中调用的API，其函数命名一般是在任务级API后面加一个FromISR。

在队列相关的API中，ISR级API一般上没有超时参数，即绝不阻塞。如果队列满/空，直接返回错误，如果中断需要唤醒一个阻塞状态的任务则需要通过pxHigherPriorityTaskWoken参数唤醒，它作为[portYIELD_FROM_ISR](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=portYIELD_FROM_ISR&zhida_source=entity)()函数的参数，该函数可以输出pdFALSE：没有更高优先级任务，pdTRUE:有一或多个优先级更高的任务，ISR退出后需要优先处理，此时CPU会发生 **上下文切换** 。

**底层原理**

之所以会有以上这个现象，是因为执行中断时，中断优先级最高，但其本身并不是一个任务，它没有[TCB](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=TCB&zhida_source=entity)。

TCB的本质是一个C结构体，其包括：

* volatile StackType_t *pxTopOfStack：一个指向任务的私有栈顶，上下文切换的时候，CPU把寄存器值保存在此，再把它放进TCB中；
* ListItem_t xStateListItem： 状态列表：本任务是就绪/阻塞/挂起？
* ListItem_t xEventListItem： 时间列表：本任务等待什么队列？
* UBaseType_t uxPriority： 描述任务优先级
* StackType_t *pxStack & uint32_t ulStackDepth：描述栈的起始地址与大小
* char pcTaskName[configMAX_TASK_NAME_LEN]： 只是一个调试用的名字
* UBaseType_t uxBasePriority & UBaseType_t uxMutexesHeld：任务原始优先级与持有的互斥量数量

每次创建一个任务，就会创建一个TCB控制块记录上述信息，作为这个任务的档案袋。

TCB是如何做到上下文切换的呢？

当 `portYIELD_FROM_ISR(pdTRUE)`，此时，实际对应了这个宏：

```c
 #define portYIELD_FROM_ISR(xSwitchRequired) \
    do { \
        if(xSwitchRequired != pdFALSE) { \
            portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT; /* 触发 PendSV */ \
        } \
    } while(0)
```

事实上，这个函数就做了一件事，就是向ICSR寄存器的PENDSVSET位写1。

这位置1，代表PendSV异常（本质上也是中断）被触发，这个异常是优先级最低位，所以当前状态下正在执行某一个中断时，该异常不会被触发。

当中断正常退出后，CPU发现PendSV被挂起，于是执行这个异常中断，即执行[PendSV_Handler](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=PendSV_Handler&zhida_source=entity)函数。这样，我们确保了切换的过程时无其他中断打扰的：

而这个函数，在FreeRTOS里面是一段被写好的汇编代码：

```text
 1. 保存当前任务的现场                   
    - 把 R4~R11压入当前栈（Cortex-M内核在硬件层面上，只要进入异常，CPU硬件压入R0, R1, R2, R3, R12, LR, PC, xPSR）  
    - 保存当前 SP 到当前任务的 TCB        
                                       
 2. 查找下一个要运行的任务                
    - pxCurrentTCB = pxNextTCB          
                                       
 3. 恢复新任务的现场                     
    - 从新任务的 TCB 读取 SP             
    - 从栈弹出 R4~R11, LR, PC, xPSR     
                                       
 4. BX LR 返回                          
    - PC 被恢复为 Task B 上次打断的位置   
    - CPU 开始执行 Task B 的代码
```

1. Task A被打断：
   *// 1. CPU 寄存器 → 压入任务 A 的栈*
   *// 2. 保存栈顶地址到任务 A 的 TCB*
   pxCurrentTCB->pxTopOfStack = 当前SP值;
2. 切换中（选任务）：
   // 调度器查就绪列表，找到优先级最高的任务 B
   pxCurrentTCB = pxNextTCB; // 指向任务 B 的 TCB
3. 切换后（恢复）
   // 从任务 B 的 TCB 取出栈顶
   SP = pxCurrentTCB->pxTopOfStack;
   // 从栈中弹出寄存器，任务 B 继续运行

简单来说，PendSV_Handler()函数在保存好原任务现场后将sp指针切到新的任务开始执行。

这就是我们所谓的上下文切换

**TCB与队列的关系**

前面讲过队列的 `xTasksWaitingToSend` 和 `xTasksWaitingToReceive` 列表，里面挂的其实不是"任务"，而是 **指向任务 TCB 的引用** ：

xTasksWaitingToReceive ──→ TCB_B ──→ TCB_C ──→ NULL

当 `xQueueSendFromISR` 被调用时：

1. 数据放入队列
2. 内核查看 `xTasksWaitingToReceive` 链表
3. 取出第一个 TCB（比如 TCB_B）
4. 把 TCB_B 的状态改为"就绪"
5. 把 TCB_B 从事件列表移到就绪列表

## TCB的两个字段

**队列的知识到这里基本结束了，现在我要最后讲一下TCB两个字段的作用，确保知识的框架时完整的**

```text
UBaseType_t uxBasePriority;     // 任务原始的优先级（户口本上的优先级）
UBaseType_t uxMutexesHeld;      // 当前持有多少个互斥量（用于嵌套情况）
```

理解这两个字段的作用，首先我们要理解优先级反转

### 优先级反转

现在我们有三个任务，根据优先级由高到低分别叫作Task H/Task M/Task L。

现在请记住，Task L目前拿到互斥量，Task H需要获取互斥量，而Task M无需互斥量。

这是会存在一个问题，由于互斥量在Task L手中，Task M会被该互斥量导致阻塞。又因为Task M不需要互斥量，纯粹优先级比较的情况下，Task M比Task L优先执行。我们的Task H作为最高优先级的任务，竟然在此处最后一个执行？！

### 优先级继承

为了避免上述问题，我们的互斥量（Mutex）遵循[优先级继承协议](https://zhida.zhihu.com/search?content_id=273316053&content_type=Article&match_order=1&q=%E4%BC%98%E5%85%88%E7%BA%A7%E7%BB%A7%E6%89%BF%E5%8D%8F%E8%AE%AE&zhida_source=entity)。

一句话解释：**如果高优先级任务因为等互斥量而被阻塞，那么持有互斥量的低优先级任务，临时" borrow "高优先级任务的优先级，直到它释放互斥量。**

有了该规定，Task H 只会等待Task L执行完毕才会执行，尽管Task L优先级低，但是既然互斥量在它手中，它先于Task H执行是完全符合常理的，只是不会再让Task M浑水摸鱼先于Task H执行。

### 两个字段

现在回过头看这两个字段：

```text
UBaseType_t uxBasePriority;     // 任务原始的优先级（户口本上的优先级）
UBaseType_t uxMutexesHeld;      // 当前持有多少个互斥量（用于嵌套情况）
```

TCB中有一个字段叫uxPriority，根据上文所述，它可以临时变高；

但是 uxBasePriority就是告诉你这个任务创建时规定的原始优先级。

一旦发生上述优先级继承的情况，uxPriority就会临时变高。

而uxPriority临时变高时，uxBasePriority就在告诉内核，在这之后应该降回多少。

为什么我们要uxMutexesHeld记录一个任务的互斥量的个数？

在刚才的例子上，我们分别设优先级为1、2、3、4的四个任务，现在Task L（优先级1）有两个互斥量，分别和Task H（优先级3）以及Task I（优先级4）竞争。

根据优先级继承协议，Task L会先升高优先级到3，再立马到4，直到Task L释放互斥量2后，Task L的优先级会下降。降到哪里？是降到3还是1呢？此时我们一看uxMutexesHeld = 1，也就是说我们手上还存在互斥量1，我们才会明白，此时Task L的优先级降到3，等到Task I执行完毕后，再轮到Task L执行直至释放互斥量1后，uxMutexesHeld = 0，才会把优先级恢复到uxBasePriority。

换句话说，**uxMutexesHeld 确保只有最后一个互斥量释放时，才恢复原始优先级。**

### **互斥量**

我之前一直说的互斥量，本质上就是一个特殊的信号值，和我们常见的flag并无二样。然而没普通信号量任意一个任务都有资格修改；而互斥量只有已经获取到这个量的人才可以修改，并且只有获取到的人才可以释放。

由于切换任务是汇编层面的操作，所以理论上存在当我一个任务的某一个C语言代码执行到一般（其对应的一长段汇编代码只执行一部分完整的汇编代码），便突然切换到了另一个任务，不用想这一定会带来一些冲突。所以多任务并发访问共享变量时，可能出现"读-改-写"操作被其他任务插足，导致数据竞争。互斥量的作用是把访问共享资源的**一段代码区间**保护起来，确保同一时刻只有一个任务能执行这段代码。

**互斥量在内核中的结构**

```c
typedef struct {
    int isLocked;           // 0=空闲，1=被占用
    TaskHandle_t owner;  
    List_t waitList;        // 因为拿不到锁而排队的任务
} Mutex_t;
```

通过这个结构体，使owner对应的任务才可以修改isLocked，而owner对应的任务释放后，又会从waitList（一个按照优先级排序的链表）中找到优先级最高的任务，这个任务成为新的owner，并修改isLocked为1。
