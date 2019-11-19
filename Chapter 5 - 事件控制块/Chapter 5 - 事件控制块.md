# Chapter 5 - 事件控制块

Created by : Mr Dk.

2019 / 11 / 19 22:34

Nanjing, Jiangsu, China

---

## 5.1 基本概念

事件通信方法的实现方式 - 事件控制块 (Event Control Blocks, ECB)

任务或中断服务子程序可以通过 ECB 向别的任务发送信号：

* 信号量
* 互斥信号量
* 消息邮箱
* 消息队列
* 事件标志组

ECB 是各种事件的基础性数据结构

是这些信号的 metadata

```c
typedef struct os_event {
    INT8U    OSEventType;                    /* Type of event control block (see OS_EVENT_TYPE_xxxx)    */
    void    *OSEventPtr;                     /* Pointer to message or queue structure                   */
    INT16U   OSEventCnt;                     /* Semaphore Count (not used if other EVENT type)          */
    OS_PRIO  OSEventGrp;                     /* Group corresponding to tasks waiting for event to occur */
    OS_PRIO  OSEventTbl[OS_EVENT_TBL_SIZE];  /* List of tasks waiting for event to occur                */

#if OS_EVENT_NAME_EN > 0u
    INT8U   *OSEventName;
#endif
} OS_EVENT;
#endif
```

* `OSEventType` 指明事件的类型
    * `OS_EVENT_SEM` - 信号量
    * `OS_EVENT_MUTEX` - 互斥信号量
    * `OS_EVENT_TYPE_MBOX` - 邮箱
    * `OS_EVENT_TYPE_Q` - 消息队列
* `OSEventPtr` 指向对应的邮箱或队列
* `OSEventCnt` 指明事件是信号量时的个数
* `OSEventGrp` 和 `OSEventTbl` 指明所有等待该事件的任务
    * 与就绪表的实现方式一致

---

## 5.2 将任务置于等待事件的任务列表中

与就绪表中的操作类似

* `OSEventGrp` 的每一位代表 8 个任务
* `OSEventTbl` 指明了 64 个任务哪个处在等待状态

用于快速获得优先级最高的等待任务

```c
pevent->OSEventGrp |= OSMapTbl[prio >> 3];
pevent->OSEventTbl[prio >> 3] |= OSMapTbl[prio & 0x07];
```

---

## 5.3 从等待事件的任务列表中删除任务

```c
if ((pevent->OSEventTbl[prio >> 3] &= ~OSMapTbl[prio & 0x07]) == 0) {
    pevent->OSEventGrp &= ~OSMapTbl[prio >> 3];
}
```

---

## 5.4 在等待事件的任务列表中查找优先级最高的任务

```c
y = OSUnMapTbl[pevent->OSEventGrp];
x = OSUnMapTbl[pevent->OSEventTbl[y]];
prio = (y << 3) + x;
```

---

## 5.5 空闲事件控制块链表

在调用 `OSInit()` 时，所有的 ECB 被串成一个空闲链表

* `OSEventFreeList` 指针总指向第一个空闲 ECB

建立事件时，就从链表中取下一个空闲 ECB 并进行初始化

删除事件后，ECB 被放回空闲链表中

空闲 ECB 的总数在编译前定义

---

## 5.6 初始化一个 ECB - OS_EventWaitListInit() 函数

当建立任意一种事件时

对应的建立函数都要调该函数，对 ECB 中的等待任务列表进行初始化

```c
/*
*********************************************************************************************************
*                                 INITIALIZE EVENT CONTROL BLOCK'S WAIT LIST
*
* Description: This function is called by other uC/OS-II services to initialize the event wait list.
*
* Arguments  : pevent    is a pointer to the event control block allocated to the event.
*
* Returns    : none
*
* Note       : This function is INTERNAL to uC/OS-II and your application should not call it.
*********************************************************************************************************
*/
#if (OS_EVENT_EN)
void  OS_EventWaitListInit (OS_EVENT *pevent)
{
    INT8U  i;


    pevent->OSEventGrp = 0u;                     /* No task waiting on event                           */
    for (i = 0u; i < OS_EVENT_TBL_SIZE; i++) {
        pevent->OSEventTbl[i] = 0u;
    }
}
#endif
```

为避免 for 循环而增加开销

这部分代码也可以由条件编译实现 😂

(之后还会有很多这样的方法)

---

## 5.7 使一个任务脱离等待进入就绪 - OS_EventTaskRdy() 函数

事件发生后，等待任务列表中，优先级最高的任务得到这个事件

从而脱离等待进入就绪状态

原理：

* 从等待任务列表中查找优先级最高的任务，从中清除
* 将 TCB 中的延时变量清零 (等待事件可以设定超时时间)
* 将 TCB 中指向 ECB 的指针置 `NULL` (任务不再等待该事件发生)
* 可能还要将事件中的消息放到 TCB 中 (比如邮箱)
* 用位掩码将 TCB 中的 `OSTCBStat` 位清零
* 判断最高优先级任务的挂起条件
    * 如果是因为等待该事件而挂起，则插入就绪表
    * (任务可能还因为其它条件挂起，因此也不一定进入就绪状态)

```c
/*
*********************************************************************************************************
*                             MAKE TASK READY TO RUN BASED ON EVENT OCCURING
*
* Description: This function is called by other uC/OS-II services and is used to ready a task that was
*              waiting for an event to occur.
*
* Arguments  : pevent      is a pointer to the event control block corresponding to the event.
*
*              pmsg        is a pointer to a message.  This pointer is used by message oriented services
*                          such as MAILBOXEs and QUEUEs.  The pointer is not used when called by other
*                          service functions.
*
*              msk         is a mask that is used to clear the status byte of the TCB.  For example,
*                          OSSemPost() will pass OS_STAT_SEM, OSMboxPost() will pass OS_STAT_MBOX etc.
*
*              pend_stat   is used to indicate the readied task's pending status:
*
*                          OS_STAT_PEND_OK      Task ready due to a post (or delete), not a timeout or
*                                               an abort.
*                          OS_STAT_PEND_ABORT   Task ready due to an abort.
*
* Returns    : none
*
* Note       : This function is INTERNAL to uC/OS-II and your application should not call it.
*********************************************************************************************************
*/
#if (OS_EVENT_EN)
INT8U  OS_EventTaskRdy (OS_EVENT  *pevent,
                        void      *pmsg,
                        INT8U      msk,
                        INT8U      pend_stat)
{
    OS_TCB   *ptcb;
    INT8U     y;
    INT8U     x;
    INT8U     prio;
#if OS_LOWEST_PRIO > 63u
    OS_PRIO  *ptbl;
#endif


#if OS_LOWEST_PRIO <= 63u
    y    = OSUnMapTbl[pevent->OSEventGrp];              /* Find HPT waiting for message                */
    x    = OSUnMapTbl[pevent->OSEventTbl[y]];
    prio = (INT8U)((y << 3u) + x);                      /* Find priority of task getting the msg       */
#else
    if ((pevent->OSEventGrp & 0xFFu) != 0u) {           /* Find HPT waiting for message                */
        y = OSUnMapTbl[ pevent->OSEventGrp & 0xFFu];
    } else {
        y = OSUnMapTbl[(OS_PRIO)(pevent->OSEventGrp >> 8u) & 0xFFu] + 8u;
    }
    ptbl = &pevent->OSEventTbl[y];
    if ((*ptbl & 0xFFu) != 0u) {
        x = OSUnMapTbl[*ptbl & 0xFFu];
    } else {
        x = OSUnMapTbl[(OS_PRIO)(*ptbl >> 8u) & 0xFFu] + 8u;
    }
    prio = (INT8U)((y << 4u) + x);                      /* Find priority of task getting the msg       */
#endif

    ptcb                  =  OSTCBPrioTbl[prio];        /* Point to this task's OS_TCB                 */
    ptcb->OSTCBDly        =  0u;                        /* Prevent OSTimeTick() from readying task     */
#if ((OS_Q_EN > 0u) && (OS_MAX_QS > 0u)) || (OS_MBOX_EN > 0u)
    ptcb->OSTCBMsg        =  pmsg;                      /* Send message directly to waiting task       */
#else
    pmsg                  =  pmsg;                      /* Prevent compiler warning if not used        */
#endif
    ptcb->OSTCBStat      &= (INT8U)~msk;                /* Clear bit associated with event type        */
    ptcb->OSTCBStatPend   =  pend_stat;                 /* Set pend status of post or abort            */
                                                        /* See if task is ready (could be susp'd)      */
    if ((ptcb->OSTCBStat &   OS_STAT_SUSPEND) == OS_STAT_RDY) {
        OSRdyGrp         |=  ptcb->OSTCBBitY;           /* Put task in the ready to run list           */
        OSRdyTbl[y]      |=  ptcb->OSTCBBitX;
    }

    OS_EventTaskRemove(ptcb, pevent);                   /* Remove this task from event   wait list     */
#if (OS_EVENT_MULTI_EN > 0u)
    if (ptcb->OSTCBEventMultiPtr != (OS_EVENT **)0) {   /* Remove this task from events' wait lists    */
        OS_EventTaskRemoveMulti(ptcb, ptcb->OSTCBEventMultiPtr);
        ptcb->OSTCBEventPtr       = (OS_EVENT  *)pevent;/* Return event as first multi-pend event ready*/
    }
#endif

    return (prio);
}
#endif
```

---

## 5.8 使一个任务进入等待事件发生状态 - OS_EventTaskWait() 函数

某个任务要等待某个事件发生时被调用

* 将任务从就绪表中删除
* 放到对应 ECB 的等待任务列表中

```c
/*
*********************************************************************************************************
*                                   MAKE TASK WAIT FOR EVENT TO OCCUR
*
* Description: This function is called by other uC/OS-II services to suspend a task because an event has
*              not occurred.
*
* Arguments  : pevent   is a pointer to the event control block for which the task will be waiting for.
*
* Returns    : none
*
* Note       : This function is INTERNAL to uC/OS-II and your application should not call it.
*********************************************************************************************************
*/
#if (OS_EVENT_EN)
void  OS_EventTaskWait (OS_EVENT *pevent)
{
    INT8U  y;


    OSTCBCur->OSTCBEventPtr               = pevent;                 /* Store ptr to ECB in TCB         */

    pevent->OSEventTbl[OSTCBCur->OSTCBY] |= OSTCBCur->OSTCBBitX;    /* Put task in waiting list        */
    pevent->OSEventGrp                   |= OSTCBCur->OSTCBBitY;

    y             =  OSTCBCur->OSTCBY;            /* Task no longer ready                              */
    OSRdyTbl[y]  &= (OS_PRIO)~OSTCBCur->OSTCBBitX;
    if (OSRdyTbl[y] == 0u) {                      /* Clear event grp bit if this was only task pending */
        OSRdyGrp &= (OS_PRIO)~OSTCBCur->OSTCBBitY;
    }
}
#endif
```

---

## 5.9 由于等待超时而将任务置为就绪态 - OS_EventTO() 函数

在预先设定的时间内，事件没有发生

任务的状态会被重新设置为就绪

* 将任务从等待列表中删除
* 将任务设置为就绪状态
* 从 TCB 中删除指向 ECB 的指针

```c
/* no source code ??? */
#if OS_EVENT_EN > 0
void  OS_EventTO (OS_EVENT *pevent) reentrant
{
    if((pevent->OSEventTbl[OSTCBCur->OSTCBY] &= ~OSTCBCur->OSTCBBitX) == 0) {
        pevent->OSEventGrp &= ~OSTCBCur->OSTCBBitY;
    }
    OSTCBCur->OSTCBStat = OS_STAT_RDY;
    OSTCBCur->OSTCBEventPtr = (OS_EVENT *) 0;
}
#endif
```

---

## Summary

这一章中的函数会被接下来几章中不同事件的具体实现所调用

---

