# Chapter 7.1 - 消息邮箱管理

Created by : Mr Dk.

2019 / 11 / 20 11:01

Nanjing, Jiangsu, China

---

## 7.1.1 概述

邮箱是一种通信机制

使任务或中断服务向另一个任务发送一个 __指针型变量__

该指针指向包含消息的数据结构

使用邮箱的目的：

* 通知一个事件的发生 - 邮箱初始化为 NULL
* 作二值信号量用 - 邮箱初始化为一个非 NULL 指针

邮箱只能接受和发送一个消息

* 邮箱为满时，新消息丢失

邮箱相关的函数可以全部被剪裁

---

## 7.1.2 建立消息邮箱 - OSMboxCreate() 函数

建立并初始化一个消息邮箱

可以选择带一个参数，指向消息的指针，作为邮箱中的初始消息

(当然也可以初始化空邮箱)

原理：

* 从空闲 ECB 链表中找到一个空闲 ECB
* 调整 `OSEventFreeList` 指针指向下一个空闲 ECB
* 初始化为消息邮箱 ECB，存入邮箱消息初始值
* 对等待任务列表进行初始化
* 返回 ECB 指针

```c
/*
*********************************************************************************************************
*                                        CREATE A MESSAGE MAILBOX
*
* Description: This function creates a message mailbox if free event control blocks are available.
*
* Arguments  : pmsg          is a pointer to a message that you wish to deposit in the mailbox.  If
*                            you set this value to the NULL pointer (i.e. (void *)0) then the mailbox
*                            will be considered empty.
*
* Returns    : != (OS_EVENT *)0  is a pointer to the event control clock (OS_EVENT) associated with the
*                                created mailbox
*              == (OS_EVENT *)0  if no event control blocks were available
*********************************************************************************************************
*/

OS_EVENT  *OSMboxCreate (void *pmsg)
{
    OS_EVENT  *pevent;
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL_IEC61508
    if (OSSafetyCriticalStartFlag == OS_TRUE) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

    if (OSIntNesting > 0u) {                     /* See if called from ISR ...                         */
        return ((OS_EVENT *)0);                  /* ... can't CREATE from an ISR                       */
    }
    OS_ENTER_CRITICAL();
    pevent = OSEventFreeList;                    /* Get next free event control block                  */
    if (OSEventFreeList != (OS_EVENT *)0) {      /* See if pool of free ECB pool was empty             */
        OSEventFreeList = (OS_EVENT *)OSEventFreeList->OSEventPtr;
    }
    OS_EXIT_CRITICAL();
    if (pevent != (OS_EVENT *)0) {
        pevent->OSEventType    = OS_EVENT_TYPE_MBOX;
        pevent->OSEventCnt     = 0u;
        pevent->OSEventPtr     = pmsg;           /* Deposit message in event control block             */
#if OS_EVENT_NAME_EN > 0u
        pevent->OSEventName    = (INT8U *)(void *)"?";
#endif
        OS_EventWaitListInit(pevent);
    }
    return (pevent);                             /* Return pointer to event control block              */
}
```

---

## 删除消息邮箱 - OSMboxDel() 函数

删除消息邮箱

还是有两个条件选项：

* `OS_DEL_NO_PEND` - 没有任何任务等待该邮箱的消息时，才删除邮箱
* `OS_DEL_ALWAYS` - 不管有没有任务在等待消息，都删除邮箱

如果删除失败，可以通过错误码查询原因

原理：

* 将 ECB 置为未使用状态，并归还给空闲链表

```c
/*
*********************************************************************************************************
*                                         DELETE A MAIBOX
*
* Description: This function deletes a mailbox and readies all tasks pending on the mailbox.
*
* Arguments  : pevent        is a pointer to the event control block associated with the desired
*                            mailbox.
*
*              opt           determines delete options as follows:
*                            opt == OS_DEL_NO_PEND   Delete the mailbox ONLY if no task pending
*                            opt == OS_DEL_ALWAYS    Deletes the mailbox even if tasks are waiting.
*                                                    In this case, all the tasks pending will be readied.
*
*              perr          is a pointer to an error code that can contain one of the following values:
*                            OS_ERR_NONE             The call was successful and the mailbox was deleted
*                            OS_ERR_DEL_ISR          If you attempted to delete the mailbox from an ISR
*                            OS_ERR_INVALID_OPT      An invalid option was specified
*                            OS_ERR_TASK_WAITING     One or more tasks were waiting on the mailbox
*                            OS_ERR_EVENT_TYPE       If you didn't pass a pointer to a mailbox
*                            OS_ERR_PEVENT_NULL      If 'pevent' is a NULL pointer.
*
* Returns    : pevent        upon error
*              (OS_EVENT *)0 if the mailbox was successfully deleted.
*
* Note(s)    : 1) This function must be used with care.  Tasks that would normally expect the presence of
*                 the mailbox MUST check the return code of OSMboxPend().
*              2) OSMboxAccept() callers will not know that the intended mailbox has been deleted!
*              3) This call can potentially disable interrupts for a long time.  The interrupt disable
*                 time is directly proportional to the number of tasks waiting on the mailbox.
*              4) Because ALL tasks pending on the mailbox will be readied, you MUST be careful in
*                 applications where the mailbox is used for mutual exclusion because the resource(s)
*                 will no longer be guarded by the mailbox.
*********************************************************************************************************
*/

#if OS_MBOX_DEL_EN > 0u
OS_EVENT  *OSMboxDel (OS_EVENT  *pevent,
                      INT8U      opt,
                      INT8U     *perr)
{
    BOOLEAN    tasks_waiting;
    OS_EVENT  *pevent_return;
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (pevent == (OS_EVENT *)0) {                         /* Validate 'pevent'                        */
        *perr = OS_ERR_PEVENT_NULL;
        return (pevent);
    }
#endif
    if (pevent->OSEventType != OS_EVENT_TYPE_MBOX) {       /* Validate event block type                */
        *perr = OS_ERR_EVENT_TYPE;
        return (pevent);
    }
    if (OSIntNesting > 0u) {                               /* See if called from ISR ...               */
        *perr = OS_ERR_DEL_ISR;                            /* ... can't DELETE from an ISR             */
        return (pevent);
    }
    OS_ENTER_CRITICAL();
    if (pevent->OSEventGrp != 0u) {                        /* See if any tasks waiting on mailbox      */
        tasks_waiting = OS_TRUE;                           /* Yes                                      */
    } else {
        tasks_waiting = OS_FALSE;                          /* No                                       */
    }
    switch (opt) {
        case OS_DEL_NO_PEND:                               /* Delete mailbox only if no task waiting   */
             if (tasks_waiting == OS_FALSE) {
#if OS_EVENT_NAME_EN > 0u
                 pevent->OSEventName = (INT8U *)(void *)"?";
#endif
                 pevent->OSEventType = OS_EVENT_TYPE_UNUSED;
                 pevent->OSEventPtr  = OSEventFreeList;    /* Return Event Control Block to free list  */
                 pevent->OSEventCnt  = 0u;
                 OSEventFreeList     = pevent;             /* Get next free event control block        */
                 OS_EXIT_CRITICAL();
                 *perr               = OS_ERR_NONE;
                 pevent_return       = (OS_EVENT *)0;      /* Mailbox has been deleted                 */
             } else {
                 OS_EXIT_CRITICAL();
                 *perr               = OS_ERR_TASK_WAITING;
                 pevent_return       = pevent;
             }
             break;

        case OS_DEL_ALWAYS:                                /* Always delete the mailbox                */
             while (pevent->OSEventGrp != 0u) {            /* Ready ALL tasks waiting for mailbox      */
                 (void)OS_EventTaskRdy(pevent, (void *)0, OS_STAT_MBOX, OS_STAT_PEND_OK);
             }
#if OS_EVENT_NAME_EN > 0u
             pevent->OSEventName    = (INT8U *)(void *)"?";
#endif
             pevent->OSEventType    = OS_EVENT_TYPE_UNUSED;
             pevent->OSEventPtr     = OSEventFreeList;     /* Return Event Control Block to free list  */
             pevent->OSEventCnt     = 0u;
             OSEventFreeList        = pevent;              /* Get next free event control block        */
             OS_EXIT_CRITICAL();
             if (tasks_waiting == OS_TRUE) {               /* Reschedule only if task(s) were waiting  */
                 OS_Sched();                               /* Find highest priority task ready to run  */
             }
             *perr         = OS_ERR_NONE;
             pevent_return = (OS_EVENT *)0;                /* Mailbox has been deleted                 */
             break;

        default:
             OS_EXIT_CRITICAL();
             *perr         = OS_ERR_INVALID_OPT;
             pevent_return = pevent;
             break;
    }
    return (pevent_return);
}
#endif
```

---

## 7.1.4 等待邮箱中的信息 - OSMboxPend() 函数

用于等待邮箱中的信息

* 如果邮箱中已有信息，那么将消息返回调用者，并清除消息
* 如果邮箱中没有信息，则挂起当前任务，直到有信息来或超时
* 如果多个任务等待同一个消息，则交给优先级最高的任务并转为就绪
* 由 `OSTaskSuspend()` 函数挂起的任务也可以接受消息，但维持挂起状态不变

原理：

* 从 ECB 中读取 `OSEventPtr`
    * 若不为空，则直接返回消息指针
    * 若为空，则任务挂起等待消息

```c
/*
*********************************************************************************************************
*                                      PEND ON MAILBOX FOR A MESSAGE
*
* Description: This function waits for a message to be sent to a mailbox
*
* Arguments  : pevent        is a pointer to the event control block associated with the desired mailbox
*
*              timeout       is an optional timeout period (in clock ticks).  If non-zero, your task will
*                            wait for a message to arrive at the mailbox up to the amount of time
*                            specified by this argument.  If you specify 0, however, your task will wait
*                            forever at the specified mailbox or, until a message arrives.
*
*              perr          is a pointer to where an error message will be deposited.  Possible error
*                            messages are:
*
*                            OS_ERR_NONE         The call was successful and your task received a
*                                                message.
*                            OS_ERR_TIMEOUT      A message was not received within the specified 'timeout'.
*                            OS_ERR_PEND_ABORT   The wait on the mailbox was aborted.
*                            OS_ERR_EVENT_TYPE   Invalid event type
*                            OS_ERR_PEND_ISR     If you called this function from an ISR and the result
*                                                would lead to a suspension.
*                            OS_ERR_PEVENT_NULL  If 'pevent' is a NULL pointer
*                            OS_ERR_PEND_LOCKED  If you called this function when the scheduler is locked
*
* Returns    : != (void *)0  is a pointer to the message received
*              == (void *)0  if no message was received or,
*                            if 'pevent' is a NULL pointer or,
*                            if you didn't pass the proper pointer to the event control block.
*********************************************************************************************************
*/
/*$PAGE*/
void  *OSMboxPend (OS_EVENT  *pevent,
                   INT32U     timeout,
                   INT8U     *perr)
{
    void      *pmsg;
#if OS_CRITICAL_METHOD == 3u                          /* Allocate storage for CPU status register      */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (pevent == (OS_EVENT *)0) {                    /* Validate 'pevent'                             */
        *perr = OS_ERR_PEVENT_NULL;
        return ((void *)0);
    }
#endif
    if (pevent->OSEventType != OS_EVENT_TYPE_MBOX) {  /* Validate event block type                     */
        *perr = OS_ERR_EVENT_TYPE;
        return ((void *)0);
    }
    if (OSIntNesting > 0u) {                          /* See if called from ISR ...                    */
        *perr = OS_ERR_PEND_ISR;                      /* ... can't PEND from an ISR                    */
        return ((void *)0);
    }
    if (OSLockNesting > 0u) {                         /* See if called with scheduler locked ...       */
        *perr = OS_ERR_PEND_LOCKED;                   /* ... can't PEND when locked                    */
        return ((void *)0);
    }
    OS_ENTER_CRITICAL();
    pmsg = pevent->OSEventPtr;
    if (pmsg != (void *)0) {                          /* See if there is already a message             */
        pevent->OSEventPtr = (void *)0;               /* Clear the mailbox                             */
        OS_EXIT_CRITICAL();
        *perr = OS_ERR_NONE;
        return (pmsg);                                /* Return the message received (or NULL)         */
    }
    OSTCBCur->OSTCBStat     |= OS_STAT_MBOX;          /* Message not available, task will pend         */
    OSTCBCur->OSTCBStatPend  = OS_STAT_PEND_OK;
    OSTCBCur->OSTCBDly       = timeout;               /* Load timeout in TCB                           */
    OS_EventTaskWait(pevent);                         /* Suspend task until event or timeout occurs    */
    OS_EXIT_CRITICAL();
    OS_Sched();                                       /* Find next highest priority task ready to run  */
    OS_ENTER_CRITICAL();
    switch (OSTCBCur->OSTCBStatPend) {                /* See if we timed-out or aborted                */
        case OS_STAT_PEND_OK:
             pmsg =  OSTCBCur->OSTCBMsg;
            *perr =  OS_ERR_NONE;
             break;

        case OS_STAT_PEND_ABORT:
             pmsg = (void *)0;
            *perr =  OS_ERR_PEND_ABORT;               /* Indicate that we aborted                      */
             break;

        case OS_STAT_PEND_TO:
        default:
             OS_EventTaskRemove(OSTCBCur, pevent);
             pmsg = (void *)0;
            *perr =  OS_ERR_TIMEOUT;                  /* Indicate that we didn't get event within TO   */
             break;
    }
    OSTCBCur->OSTCBStat          =  OS_STAT_RDY;      /* Set   task  status to ready                   */
    OSTCBCur->OSTCBStatPend      =  OS_STAT_PEND_OK;  /* Clear pend  status                            */
    OSTCBCur->OSTCBEventPtr      = (OS_EVENT  *)0;    /* Clear event pointers                          */
#if (OS_EVENT_MULTI_EN > 0u)
    OSTCBCur->OSTCBEventMultiPtr = (OS_EVENT **)0;
#endif
    OSTCBCur->OSTCBMsg           = (void      *)0;    /* Clear  received message                       */
    OS_EXIT_CRITICAL();
    return (pmsg);                                    /* Return received message                       */
}
```

---

## 7.1.5 发出邮箱消息 - OSMboxPost() 函数

* 如果邮箱中已有消息，则立刻返回调用者，并丢弃新消息
* 如果邮箱中没有消息
    * 如果有任务在等待消息，则最高优先级任务得到这个消息
    * 如果没有任务等待消息，则消息的指针被保存在邮箱中

调用者可以是任务或中断

原理：

* 检查 ECB 的 `OSEventPtr` 是否为空咯
* 检查 ECB 的等待任务列表检查是否有任务等待消息咯
* 投递消息或丢弃消息咯

```c
/*
*********************************************************************************************************
*                                       POST MESSAGE TO A MAILBOX
*
* Description: This function sends a message to a mailbox
*
* Arguments  : pevent        is a pointer to the event control block associated with the desired mailbox
*
*              pmsg          is a pointer to the message to send.  You MUST NOT send a NULL pointer.
*
* Returns    : OS_ERR_NONE          The call was successful and the message was sent
*              OS_ERR_MBOX_FULL     If the mailbox already contains a message.  You can can only send one
*                                   message at a time and thus, the message MUST be consumed before you
*                                   are allowed to send another one.
*              OS_ERR_EVENT_TYPE    If you are attempting to post to a non mailbox.
*              OS_ERR_PEVENT_NULL   If 'pevent' is a NULL pointer
*              OS_ERR_POST_NULL_PTR If you are attempting to post a NULL pointer
*
* Note(s)    : 1) HPT means Highest Priority Task
*********************************************************************************************************
*/

#if OS_MBOX_POST_EN > 0u
INT8U  OSMboxPost (OS_EVENT  *pevent,
                   void      *pmsg)
{
#if OS_CRITICAL_METHOD == 3u                          /* Allocate storage for CPU status register      */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#if OS_ARG_CHK_EN > 0u
    if (pevent == (OS_EVENT *)0) {                    /* Validate 'pevent'                             */
        return (OS_ERR_PEVENT_NULL);
    }
    if (pmsg == (void *)0) {                          /* Make sure we are not posting a NULL pointer   */
        return (OS_ERR_POST_NULL_PTR);
    }
#endif
    if (pevent->OSEventType != OS_EVENT_TYPE_MBOX) {  /* Validate event block type                     */
        return (OS_ERR_EVENT_TYPE);
    }
    OS_ENTER_CRITICAL();
    if (pevent->OSEventGrp != 0u) {                   /* See if any task pending on mailbox            */
                                                      /* Ready HPT waiting on event                    */
        (void)OS_EventTaskRdy(pevent, pmsg, OS_STAT_MBOX, OS_STAT_PEND_OK);
        OS_EXIT_CRITICAL();
        OS_Sched();                                   /* Find highest priority task ready to run       */
        return (OS_ERR_NONE);
    }
    if (pevent->OSEventPtr != (void *)0) {            /* Make sure mailbox doesn't already have a msg  */
        OS_EXIT_CRITICAL();
        return (OS_ERR_MBOX_FULL);
    }
    pevent->OSEventPtr = pmsg;                        /* Place message in mailbox                      */
    OS_EXIT_CRITICAL();
    return (OS_ERR_NONE);
}
#endif
```

---

## 7.1.6 发出邮箱消息 - OSMboxPostOpt() 函数

与上个函数的逻辑基本相同

不同的是，如果有任务在等待邮箱里的信息

提供了两种选项对信息进行处理：

* `OS_POST_OPT_NONE` - 让最高优先级的任务得到这则消息
* `OS_POST_OPT_BROADCAST` - 让所有等待消息的任务得到这则消息

原理：

* 检查 ECB 的等待任务列表
* 将所有或最高优先级任务置为就绪
* 如果没有任务正在等待消息，则保存至 ECB 中

```c
/*
*********************************************************************************************************
*                                       POST MESSAGE TO A MAILBOX
*
* Description: This function sends a message to a mailbox
*
* Arguments  : pevent        is a pointer to the event control block associated with the desired mailbox
*
*              pmsg          is a pointer to the message to send.  You MUST NOT send a NULL pointer.
*
*              opt           determines the type of POST performed:
*                            OS_POST_OPT_NONE         POST to a single waiting task
*                                                     (Identical to OSMboxPost())
*                            OS_POST_OPT_BROADCAST    POST to ALL tasks that are waiting on the mailbox
*
*                            OS_POST_OPT_NO_SCHED     Indicates that the scheduler will NOT be invoked
*
* Returns    : OS_ERR_NONE          The call was successful and the message was sent
*              OS_ERR_MBOX_FULL     If the mailbox already contains a message.  You can can only send one
*                                   message at a time and thus, the message MUST be consumed before you
*                                   are allowed to send another one.
*              OS_ERR_EVENT_TYPE    If you are attempting to post to a non mailbox.
*              OS_ERR_PEVENT_NULL   If 'pevent' is a NULL pointer
*              OS_ERR_POST_NULL_PTR If you are attempting to post a NULL pointer
*
* Note(s)    : 1) HPT means Highest Priority Task
*
* Warning    : Interrupts can be disabled for a long time if you do a 'broadcast'.  In fact, the
*              interrupt disable time is proportional to the number of tasks waiting on the mailbox.
*********************************************************************************************************
*/

#if OS_MBOX_POST_OPT_EN > 0u
INT8U  OSMboxPostOpt (OS_EVENT  *pevent,
                      void      *pmsg,
                      INT8U      opt)
{
#if OS_CRITICAL_METHOD == 3u                          /* Allocate storage for CPU status register      */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#if OS_ARG_CHK_EN > 0u
    if (pevent == (OS_EVENT *)0) {                    /* Validate 'pevent'                             */
        return (OS_ERR_PEVENT_NULL);
    }
    if (pmsg == (void *)0) {                          /* Make sure we are not posting a NULL pointer   */
        return (OS_ERR_POST_NULL_PTR);
    }
#endif
    if (pevent->OSEventType != OS_EVENT_TYPE_MBOX) {  /* Validate event block type                     */
        return (OS_ERR_EVENT_TYPE);
    }
    OS_ENTER_CRITICAL();
    if (pevent->OSEventGrp != 0u) {                   /* See if any task pending on mailbox            */
        if ((opt & OS_POST_OPT_BROADCAST) != 0x00u) { /* Do we need to post msg to ALL waiting tasks ? */
            while (pevent->OSEventGrp != 0u) {        /* Yes, Post to ALL tasks waiting on mailbox     */
                (void)OS_EventTaskRdy(pevent, pmsg, OS_STAT_MBOX, OS_STAT_PEND_OK);
            }
        } else {                                      /* No,  Post to HPT waiting on mbox              */
            (void)OS_EventTaskRdy(pevent, pmsg, OS_STAT_MBOX, OS_STAT_PEND_OK);
        }
        OS_EXIT_CRITICAL();
        if ((opt & OS_POST_OPT_NO_SCHED) == 0u) {     /* See if scheduler needs to be invoked          */
            OS_Sched();                               /* Find HPT ready to run                         */
        }
        return (OS_ERR_NONE);
    }
    if (pevent->OSEventPtr != (void *)0) {            /* Make sure mailbox doesn't already have a msg  */
        OS_EXIT_CRITICAL();
        return (OS_ERR_MBOX_FULL);
    }
    pevent->OSEventPtr = pmsg;                        /* Place message in mailbox                      */
    OS_EXIT_CRITICAL();
    return (OS_ERR_NONE);
}
#endif
```

---

## 7.1.7 无等待地从邮箱中获取消息 - OSMboxAccept() 函数

查看指定的邮箱中是否有需要的消息，且不挂起任务

调用者可以是任务，也可以是中断

```c
/*
*********************************************************************************************************
*                                     ACCEPT MESSAGE FROM MAILBOX
*
* Description: This function checks the mailbox to see if a message is available.  Unlike OSMboxPend(),
*              OSMboxAccept() does not suspend the calling task if a message is not available.
*
* Arguments  : pevent        is a pointer to the event control block
*
* Returns    : != (void *)0  is the message in the mailbox if one is available.  The mailbox is cleared
*                            so the next time OSMboxAccept() is called, the mailbox will be empty.
*              == (void *)0  if the mailbox is empty or,
*                            if 'pevent' is a NULL pointer or,
*                            if you didn't pass the proper event pointer.
*********************************************************************************************************
*/

#if OS_MBOX_ACCEPT_EN > 0u
void  *OSMboxAccept (OS_EVENT *pevent)
{
    void      *pmsg;
#if OS_CRITICAL_METHOD == 3u                              /* Allocate storage for CPU status register  */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#if OS_ARG_CHK_EN > 0u
    if (pevent == (OS_EVENT *)0) {                        /* Validate 'pevent'                         */
        return ((void *)0);
    }
#endif
    if (pevent->OSEventType != OS_EVENT_TYPE_MBOX) {      /* Validate event block type                 */
        return ((void *)0);
    }
    OS_ENTER_CRITICAL();
    pmsg               = pevent->OSEventPtr;
    pevent->OSEventPtr = (void *)0;                       /* Clear the mailbox                         */
    OS_EXIT_CRITICAL();
    return (pmsg);                                        /* Return the message received (or NULL)     */
}
#endif
```

---

## 7.1.8 查询邮箱的状态 - OSMboxQuery() 函数

将 ECB 中的邮箱状态复制到一个简单的数据结构 `OS_MBOX_DATA` 中并返回

无需等待，所以调用者可以是任务也可以是中断

```c
/*
*********************************************************************************************************
*                                        QUERY A MESSAGE MAILBOX
*
* Description: This function obtains information about a message mailbox.
*
* Arguments  : pevent        is a pointer to the event control block associated with the desired mailbox
*
*              p_mbox_data   is a pointer to a structure that will contain information about the message
*                            mailbox.
*
* Returns    : OS_ERR_NONE         The call was successful and the message was sent
*              OS_ERR_EVENT_TYPE   If you are attempting to obtain data from a non mailbox.
*              OS_ERR_PEVENT_NULL  If 'pevent'      is a NULL pointer
*              OS_ERR_PDATA_NULL   If 'p_mbox_data' is a NULL pointer
*********************************************************************************************************
*/

#if OS_MBOX_QUERY_EN > 0u
INT8U  OSMboxQuery (OS_EVENT      *pevent,
                    OS_MBOX_DATA  *p_mbox_data)
{
    INT8U       i;
    OS_PRIO    *psrc;
    OS_PRIO    *pdest;
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register */
    OS_CPU_SR   cpu_sr = 0u;
#endif



#if OS_ARG_CHK_EN > 0u
    if (pevent == (OS_EVENT *)0) {                         /* Validate 'pevent'                        */
        return (OS_ERR_PEVENT_NULL);
    }
    if (p_mbox_data == (OS_MBOX_DATA *)0) {                /* Validate 'p_mbox_data'                   */
        return (OS_ERR_PDATA_NULL);
    }
#endif
    if (pevent->OSEventType != OS_EVENT_TYPE_MBOX) {       /* Validate event block type                */
        return (OS_ERR_EVENT_TYPE);
    }
    OS_ENTER_CRITICAL();
    p_mbox_data->OSEventGrp = pevent->OSEventGrp;          /* Copy message mailbox wait list           */
    psrc                    = &pevent->OSEventTbl[0];
    pdest                   = &p_mbox_data->OSEventTbl[0];
    for (i = 0u; i < OS_EVENT_TBL_SIZE; i++) {
        *pdest++ = *psrc++;
    }
    p_mbox_data->OSMsg = pevent->OSEventPtr;               /* Get message from mailbox                 */
    OS_EXIT_CRITICAL();
    return (OS_ERR_NONE);
}
#endif                                                     /* OS_MBOX_QUERY_EN                         */
```

---

## Summary

仔细一想，邮箱和信号量并没有什么卵区别

一个是数，一个是指针罢了

不过邮箱中比较特殊的一点就是

新消息不能覆盖旧消息，而是被丢弃了

---

