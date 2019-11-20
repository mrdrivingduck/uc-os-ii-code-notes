# Chapter 8 - 事件标志组

Created by : Mr Dk.

2019 / 11 / 20 22:07

Nanjing, Jiangsu, China

---

## 8.1 概述

书上的描述给人整晕了

我决定在自己描述一下

与 Linux 的信号位图类似

在 μC/OS-II 中可以分配一个由多个 bit 组成的事件标志组

提供函数可以对其中的某位或某几位标志置位或复位，代表事件发生

每个任务可以等待某个事件标志组的某位或某几位标志置位或复位

需要两个数据结构：

事件标志组数据结构 `OS_FLAG_GRP`

所有的事件标志记录在这个结构体中

```c
typedef struct os_flag_grp {                /* Event Flag Group                                        */
    INT8U         OSFlagType;               /* Should be set to OS_EVENT_TYPE_FLAG                     */
    void         *OSFlagWaitList;           /* Pointer to first NODE of task waiting on event flag     */
    OS_FLAGS      OSFlagFlags;              /* 8, 16 or 32 bit flags                                   */
#if OS_FLAG_NAME_EN > 0u
    INT8U        *OSFlagName;
#endif
} OS_FLAG_GRP;
```

其中的 `OSFlagFlags` 可以是 8、16 或 32 位

`OSFlagWaitList` 指向每个任务对该标志组的等待信息形成的链表

每个任务的等待信息保存在第二个数据结构 `OS_FLAG_NODE` 中

```c
typedef struct os_flag_node {               /* Event Flag Wait List Node                               */
    void         *OSFlagNodeNext;           /* Pointer to next     NODE in wait list                   */
    void         *OSFlagNodePrev;           /* Pointer to previous NODE in wait list                   */
    void         *OSFlagNodeTCB;            /* Pointer to TCB of waiting task                          */
    void         *OSFlagNodeFlagGrp;        /* Pointer to Event Flag Group                             */
    OS_FLAGS      OSFlagNodeFlags;          /* Event flag to wait on                                   */
    INT8U         OSFlagNodeWaitType;       /* Type of wait:                                           */
                                            /*      OS_FLAG_WAIT_AND                                   */
                                            /*      OS_FLAG_WAIT_ALL                                   */
                                            /*      OS_FLAG_WAIT_OR                                    */
                                            /*      OS_FLAG_WAIT_ANY                                   */
} OS_FLAG_NODE;
```

* `OSFlagNodeNext` 和 `OSFlagNodePrev` 维护各个任务的等待信息的双向链表
* `OSFlagNodeTCB` 指向等待事件标志组的 TCB
* `OSFlagNodeFlagGrp` 指向等待的事件标志组
* `OSFlagNodeFlags` 记录了该任务等待时间标志组中的哪几位
* `OSFlagNodeWaitType` 记录了等待方式
    * `OS_FLAG_WAIT_CLR_ALL` / `OS_FLAG_WAIT_CLR_AND` - 所有指定事件标志清 0
    * `OS_FLAG_WAIT_CLR_ANY` / `OS_FLAG_WAIT_CLR_OR` - 任意指定事件标志清 0
    * `OS_FLAG_WAIT_SET_ALL` / `OS_FLAG_WAIT_SET_AND` - 所有指定事件标志置 1
    * `OS_FLAG_WAIT_SET_ANY` / `OS_FLAG_WAIT_SET_OR` - 任意指定事件标志置 1

> 总结一下
>
> OS_FLAG_GRP 维护事件标志组本身
>
> 每个任务可以注册 OS_FLAG_NODE，串接到 OS_FLAG_GRP 上
>
> OS_FLAG_NODE 随着对事件的请求，和事件的发生，而被动态创建或撤销

---

## 8.2 建立事件标志组 - OSFlagCreate() 函数

从空闲事件标志组链表中取一个 `OS_FLAG_GRP` 变量

并用提供的 flag 参数进行初始化

返回指向这个 `OS_FLAG_GRP` 的指针

```c
/*
*********************************************************************************************************
*                                           CREATE AN EVENT FLAG
*
* Description: This function is called to create an event flag group.
*
* Arguments  : flags         Contains the initial value to store in the event flag group.
*
*              perr          is a pointer to an error code which will be returned to your application:
*                               OS_ERR_NONE               if the call was successful.
*                               OS_ERR_CREATE_ISR         if you attempted to create an Event Flag from an
*                                                         ISR.
*                               OS_ERR_FLAG_GRP_DEPLETED  if there are no more event flag groups
*
* Returns    : A pointer to an event flag group or a NULL pointer if no more groups are available.
*
* Called from: Task ONLY
*********************************************************************************************************
*/

OS_FLAG_GRP  *OSFlagCreate (OS_FLAGS  flags,
                            INT8U    *perr)
{
    OS_FLAG_GRP *pgrp;
#if OS_CRITICAL_METHOD == 3u                        /* Allocate storage for CPU status register        */
    OS_CPU_SR    cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#ifdef OS_SAFETY_CRITICAL_IEC61508
    if (OSSafetyCriticalStartFlag == OS_TRUE) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

    if (OSIntNesting > 0u) {                        /* See if called from ISR ...                      */
        *perr = OS_ERR_CREATE_ISR;                  /* ... can't CREATE from an ISR                    */
        return ((OS_FLAG_GRP *)0);
    }
    OS_ENTER_CRITICAL();
    pgrp = OSFlagFreeList;                          /* Get next free event flag                        */
    if (pgrp != (OS_FLAG_GRP *)0) {                 /* See if we have event flag groups available      */
                                                    /* Adjust free list                                */
        OSFlagFreeList       = (OS_FLAG_GRP *)OSFlagFreeList->OSFlagWaitList;
        pgrp->OSFlagType     = OS_EVENT_TYPE_FLAG;  /* Set to event flag group type                    */
        pgrp->OSFlagFlags    = flags;               /* Set to desired initial value                    */
        pgrp->OSFlagWaitList = (void *)0;           /* Clear list of tasks waiting on flags            */
#if OS_FLAG_NAME_EN > 0u
        pgrp->OSFlagName     = (INT8U *)(void *)"?";
#endif
        OS_EXIT_CRITICAL();
        *perr                = OS_ERR_NONE;
    } else {
        OS_EXIT_CRITICAL();
        *perr                = OS_ERR_FLAG_GRP_DEPLETED;
    }
    return (pgrp);                                  /* Return pointer to event flag group              */
}
```

---

## 8.3 等待事件标志 - OSFlagPend() 函数

等待事件标志组中的事件标志位

参数中需要给定一个 bitmap，哪位置 1 说明需要检查哪位，置 0 表示忽略

还需要给定等待事件标志的方式：

* 全都清 0
* 任意清 0
* 全都置 1
* 任意置 1

另外，在等待类型中，还可以在后面 `OS_FLAG_WAIT_SET_ANY + OS_FLAG_CONSUME`

* 使对应的事件标志在使用后被消费

如果事件标志不能被满足，则挂起调用者

* 挂起的过程中，需要用一个 `OS_FLAG_NODE` 记录任务需求和等待方式，链接到 `OS_FLAG_GRP` 上

```c
/*
*********************************************************************************************************
*                                        WAIT ON AN EVENT FLAG GROUP
*
* Description: This function is called to wait for a combination of bits to be set in an event flag
*              group.  Your application can wait for ANY bit to be set or ALL bits to be set.
*
* Arguments  : pgrp          is a pointer to the desired event flag group.
*
*              flags         Is a bit pattern indicating which bit(s) (i.e. flags) you wish to wait for.
*                            The bits you want are specified by setting the corresponding bits in
*                            'flags'.  e.g. if your application wants to wait for bits 0 and 1 then
*                            'flags' would contain 0x03.
*
*              wait_type     specifies whether you want ALL bits to be set or ANY of the bits to be set.
*                            You can specify the following argument:
*
*                            OS_FLAG_WAIT_CLR_ALL   You will wait for ALL bits in 'mask' to be clear (0)
*                            OS_FLAG_WAIT_SET_ALL   You will wait for ALL bits in 'mask' to be set   (1)
*                            OS_FLAG_WAIT_CLR_ANY   You will wait for ANY bit  in 'mask' to be clear (0)
*                            OS_FLAG_WAIT_SET_ANY   You will wait for ANY bit  in 'mask' to be set   (1)
*
*                            NOTE: Add OS_FLAG_CONSUME if you want the event flag to be 'consumed' by
*                                  the call.  Example, to wait for any flag in a group AND then clear
*                                  the flags that are present, set 'wait_type' to:
*
*                                  OS_FLAG_WAIT_SET_ANY + OS_FLAG_CONSUME
*
*              timeout       is an optional timeout (in clock ticks) that your task will wait for the
*                            desired bit combination.  If you specify 0, however, your task will wait
*                            forever at the specified event flag group or, until a message arrives.
*
*              perr          is a pointer to an error code and can be:
*                            OS_ERR_NONE               The desired bits have been set within the specified
*                                                      'timeout'.
*                            OS_ERR_PEND_ISR           If you tried to PEND from an ISR
*                            OS_ERR_FLAG_INVALID_PGRP  If 'pgrp' is a NULL pointer.
*                            OS_ERR_EVENT_TYPE         You are not pointing to an event flag group
*                            OS_ERR_TIMEOUT            The bit(s) have not been set in the specified
*                                                      'timeout'.
*                            OS_ERR_PEND_ABORT         The wait on the flag was aborted.
*                            OS_ERR_FLAG_WAIT_TYPE     You didn't specify a proper 'wait_type' argument.
*
* Returns    : The flags in the event flag group that made the task ready or, 0 if a timeout or an error
*              occurred.
*
* Called from: Task ONLY
*
* Note(s)    : 1) IMPORTANT, the behavior of this function has changed from PREVIOUS versions.  The
*                 function NOW returns the flags that were ready INSTEAD of the current state of the
*                 event flags.
*********************************************************************************************************
*/

OS_FLAGS  OSFlagPend (OS_FLAG_GRP  *pgrp,
                      OS_FLAGS      flags,
                      INT8U         wait_type,
                      INT32U        timeout,
                      INT8U        *perr)
{
    OS_FLAG_NODE  node;
    OS_FLAGS      flags_rdy;
    INT8U         result;
    INT8U         pend_stat;
    BOOLEAN       consume;
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register */
    OS_CPU_SR     cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (pgrp == (OS_FLAG_GRP *)0) {                        /* Validate 'pgrp'                          */
        *perr = OS_ERR_FLAG_INVALID_PGRP;
        return ((OS_FLAGS)0);
    }
#endif
    if (OSIntNesting > 0u) {                               /* See if called from ISR ...               */
        *perr = OS_ERR_PEND_ISR;                           /* ... can't PEND from an ISR               */
        return ((OS_FLAGS)0);
    }
    if (OSLockNesting > 0u) {                              /* See if called with scheduler locked ...  */
        *perr = OS_ERR_PEND_LOCKED;                        /* ... can't PEND when locked               */
        return ((OS_FLAGS)0);
    }
    if (pgrp->OSFlagType != OS_EVENT_TYPE_FLAG) {          /* Validate event block type                */
        *perr = OS_ERR_EVENT_TYPE;
        return ((OS_FLAGS)0);
    }
    result = (INT8U)(wait_type & OS_FLAG_CONSUME);
    if (result != (INT8U)0) {                              /* See if we need to consume the flags      */
        wait_type &= (INT8U)~(INT8U)OS_FLAG_CONSUME;
        consume    = OS_TRUE;
    } else {
        consume    = OS_FALSE;
    }
/*$PAGE*/
    OS_ENTER_CRITICAL();
    switch (wait_type) {
        case OS_FLAG_WAIT_SET_ALL:                         /* See if all required flags are set        */
             flags_rdy = (OS_FLAGS)(pgrp->OSFlagFlags & flags);   /* Extract only the bits we want     */
             if (flags_rdy == flags) {                     /* Must match ALL the bits that we want     */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags &= (OS_FLAGS)~flags_rdy;   /* Clear ONLY the flags we wanted    */
                 }
                 OSTCBCur->OSTCBFlagsRdy = flags_rdy;      /* Save flags that were ready               */
                 OS_EXIT_CRITICAL();                       /* Yes, condition met, return to caller     */
                 *perr                   = OS_ERR_NONE;
                 return (flags_rdy);
             } else {                                      /* Block task until events occur or timeout */
                 OS_FlagBlock(pgrp, &node, flags, wait_type, timeout);
                 OS_EXIT_CRITICAL();
             }
             break;

        case OS_FLAG_WAIT_SET_ANY:
             flags_rdy = (OS_FLAGS)(pgrp->OSFlagFlags & flags);    /* Extract only the bits we want    */
             if (flags_rdy != (OS_FLAGS)0) {               /* See if any flag set                      */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags &= (OS_FLAGS)~flags_rdy;    /* Clear ONLY the flags that we got */
                 }
                 OSTCBCur->OSTCBFlagsRdy = flags_rdy;      /* Save flags that were ready               */
                 OS_EXIT_CRITICAL();                       /* Yes, condition met, return to caller     */
                 *perr                   = OS_ERR_NONE;
                 return (flags_rdy);
             } else {                                      /* Block task until events occur or timeout */
                 OS_FlagBlock(pgrp, &node, flags, wait_type, timeout);
                 OS_EXIT_CRITICAL();
             }
             break;

#if OS_FLAG_WAIT_CLR_EN > 0u
        case OS_FLAG_WAIT_CLR_ALL:                         /* See if all required flags are cleared    */
             flags_rdy = (OS_FLAGS)~pgrp->OSFlagFlags & flags;    /* Extract only the bits we want     */
             if (flags_rdy == flags) {                     /* Must match ALL the bits that we want     */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags |= flags_rdy;       /* Set ONLY the flags that we wanted        */
                 }
                 OSTCBCur->OSTCBFlagsRdy = flags_rdy;      /* Save flags that were ready               */
                 OS_EXIT_CRITICAL();                       /* Yes, condition met, return to caller     */
                 *perr                   = OS_ERR_NONE;
                 return (flags_rdy);
             } else {                                      /* Block task until events occur or timeout */
                 OS_FlagBlock(pgrp, &node, flags, wait_type, timeout);
                 OS_EXIT_CRITICAL();
             }
             break;

        case OS_FLAG_WAIT_CLR_ANY:
             flags_rdy = (OS_FLAGS)~pgrp->OSFlagFlags & flags;   /* Extract only the bits we want      */
             if (flags_rdy != (OS_FLAGS)0) {               /* See if any flag cleared                  */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags |= flags_rdy;       /* Set ONLY the flags that we got           */
                 }
                 OSTCBCur->OSTCBFlagsRdy = flags_rdy;      /* Save flags that were ready               */
                 OS_EXIT_CRITICAL();                       /* Yes, condition met, return to caller     */
                 *perr                   = OS_ERR_NONE;
                 return (flags_rdy);
             } else {                                      /* Block task until events occur or timeout */
                 OS_FlagBlock(pgrp, &node, flags, wait_type, timeout);
                 OS_EXIT_CRITICAL();
             }
             break;
#endif

        default:
             OS_EXIT_CRITICAL();
             flags_rdy = (OS_FLAGS)0;
             *perr      = OS_ERR_FLAG_WAIT_TYPE;
             return (flags_rdy);
    }
/*$PAGE*/
    OS_Sched();                                            /* Find next HPT ready to run               */
    OS_ENTER_CRITICAL();
    if (OSTCBCur->OSTCBStatPend != OS_STAT_PEND_OK) {      /* Have we timed-out or aborted?            */
        pend_stat                = OSTCBCur->OSTCBStatPend;
        OSTCBCur->OSTCBStatPend  = OS_STAT_PEND_OK;
        OS_FlagUnlink(&node);
        OSTCBCur->OSTCBStat      = OS_STAT_RDY;            /* Yes, make task ready-to-run              */
        OS_EXIT_CRITICAL();
        flags_rdy                = (OS_FLAGS)0;
        switch (pend_stat) {
            case OS_STAT_PEND_ABORT:
                 *perr = OS_ERR_PEND_ABORT;                /* Indicate that we aborted   waiting       */
                 break;

            case OS_STAT_PEND_TO:
            default:
                 *perr = OS_ERR_TIMEOUT;                   /* Indicate that we timed-out waiting       */
                 break;
        }
        return (flags_rdy);
    }
    flags_rdy = OSTCBCur->OSTCBFlagsRdy;
    if (consume == OS_TRUE) {                              /* See if we need to consume the flags      */
        switch (wait_type) {
            case OS_FLAG_WAIT_SET_ALL:
            case OS_FLAG_WAIT_SET_ANY:                     /* Clear ONLY the flags we got              */
                 pgrp->OSFlagFlags &= (OS_FLAGS)~flags_rdy;
                 break;

#if OS_FLAG_WAIT_CLR_EN > 0u
            case OS_FLAG_WAIT_CLR_ALL:
            case OS_FLAG_WAIT_CLR_ANY:                     /* Set   ONLY the flags we got              */
                 pgrp->OSFlagFlags |=  flags_rdy;
                 break;
#endif
            default:
                 OS_EXIT_CRITICAL();
                 *perr = OS_ERR_FLAG_WAIT_TYPE;
                 return ((OS_FLAGS)0);
        }
    }
    OS_EXIT_CRITICAL();
    *perr = OS_ERR_NONE;                                   /* Event(s) must have occurred              */
    return (flags_rdy);
}
```

## 8.4 设置事件标志 - OSFlagPost() 函数

用于设置事件标志组中的事件标志位

对指定的标志位 __置位__ 或 __复位__

原理：

* 对 `OS_FLAG_GRP` 中的成员变量 `OSFlagFlags` 按照参数的要求进行置位或复位
* 检查是否有任务正在等待事件标志组，如果有，则依次判断条件是否满足
    * 如果满足，则使任务转为就绪
    * (将 `OS_FLAG_NODE` 从等待链表中移出？)

> 令我奇怪的是
>
> 好像没看到对 `OS_FLAG_CONSUME` 的处理啊
>
> emm...好像不必在这里处理？因为在 `OSFlagPend()` 中已经处理过了？

```c
/*
*********************************************************************************************************
*                                         POST EVENT FLAG BIT(S)
*
* Description: This function is called to set or clear some bits in an event flag group.  The bits to
*              set or clear are specified by a 'bit mask'.
*
* Arguments  : pgrp          is a pointer to the desired event flag group.
*
*              flags         If 'opt' (see below) is OS_FLAG_SET, each bit that is set in 'flags' will
*                            set the corresponding bit in the event flag group.  e.g. to set bits 0, 4
*                            and 5 you would set 'flags' to:
*
*                                0x31     (note, bit 0 is least significant bit)
*
*                            If 'opt' (see below) is OS_FLAG_CLR, each bit that is set in 'flags' will
*                            CLEAR the corresponding bit in the event flag group.  e.g. to clear bits 0,
*                            4 and 5 you would specify 'flags' as:
*
*                                0x31     (note, bit 0 is least significant bit)
*
*              opt           indicates whether the flags will be:
*                                set     (OS_FLAG_SET) or
*                                cleared (OS_FLAG_CLR)
*
*              perr          is a pointer to an error code and can be:
*                            OS_ERR_NONE                The call was successfull
*                            OS_ERR_FLAG_INVALID_PGRP   You passed a NULL pointer
*                            OS_ERR_EVENT_TYPE          You are not pointing to an event flag group
*                            OS_ERR_FLAG_INVALID_OPT    You specified an invalid option
*
* Returns    : the new value of the event flags bits that are still set.
*
* Called From: Task or ISR
*
* WARNING(s) : 1) The execution time of this function depends on the number of tasks waiting on the event
*                 flag group.
*              2) The amount of time interrupts are DISABLED depends on the number of tasks waiting on
*                 the event flag group.
*********************************************************************************************************
*/
OS_FLAGS  OSFlagPost (OS_FLAG_GRP  *pgrp,
                      OS_FLAGS      flags,
                      INT8U         opt,
                      INT8U        *perr)
{
    OS_FLAG_NODE *pnode;
    BOOLEAN       sched;
    OS_FLAGS      flags_cur;
    OS_FLAGS      flags_rdy;
    BOOLEAN       rdy;
#if OS_CRITICAL_METHOD == 3u                         /* Allocate storage for CPU status register       */
    OS_CPU_SR     cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (pgrp == (OS_FLAG_GRP *)0) {                  /* Validate 'pgrp'                                */
        *perr = OS_ERR_FLAG_INVALID_PGRP;
        return ((OS_FLAGS)0);
    }
#endif
    if (pgrp->OSFlagType != OS_EVENT_TYPE_FLAG) {    /* Make sure we are pointing to an event flag grp */
        *perr = OS_ERR_EVENT_TYPE;
        return ((OS_FLAGS)0);
    }
/*$PAGE*/
    OS_ENTER_CRITICAL();
    switch (opt) {
        case OS_FLAG_CLR:
             pgrp->OSFlagFlags &= (OS_FLAGS)~flags;  /* Clear the flags specified in the group         */
             break;

        case OS_FLAG_SET:
             pgrp->OSFlagFlags |=  flags;            /* Set   the flags specified in the group         */
             break;

        default:
             OS_EXIT_CRITICAL();                     /* INVALID option                                 */
             *perr = OS_ERR_FLAG_INVALID_OPT;
             return ((OS_FLAGS)0);
    }
    sched = OS_FALSE;                                /* Indicate that we don't need rescheduling       */
    pnode = (OS_FLAG_NODE *)pgrp->OSFlagWaitList;
    while (pnode != (OS_FLAG_NODE *)0) {             /* Go through all tasks waiting on event flag(s)  */
        switch (pnode->OSFlagNodeWaitType) {
            case OS_FLAG_WAIT_SET_ALL:               /* See if all req. flags are set for current node */
                 flags_rdy = (OS_FLAGS)(pgrp->OSFlagFlags & pnode->OSFlagNodeFlags);
                 if (flags_rdy == pnode->OSFlagNodeFlags) {
                     rdy = OS_FlagTaskRdy(pnode, flags_rdy);  /* Make task RTR, event(s) Rx'd          */
                     if (rdy == OS_TRUE) {
                         sched = OS_TRUE;                     /* When done we will reschedule          */
                     }
                 }
                 break;

            case OS_FLAG_WAIT_SET_ANY:               /* See if any flag set                            */
                 flags_rdy = (OS_FLAGS)(pgrp->OSFlagFlags & pnode->OSFlagNodeFlags);
                 if (flags_rdy != (OS_FLAGS)0) {
                     rdy = OS_FlagTaskRdy(pnode, flags_rdy);  /* Make task RTR, event(s) Rx'd          */
                     if (rdy == OS_TRUE) {
                         sched = OS_TRUE;                     /* When done we will reschedule          */
                     }
                 }
                 break;

#if OS_FLAG_WAIT_CLR_EN > 0u
            case OS_FLAG_WAIT_CLR_ALL:               /* See if all req. flags are set for current node */
                 flags_rdy = (OS_FLAGS)~pgrp->OSFlagFlags & pnode->OSFlagNodeFlags;
                 if (flags_rdy == pnode->OSFlagNodeFlags) {
                     rdy = OS_FlagTaskRdy(pnode, flags_rdy);  /* Make task RTR, event(s) Rx'd          */
                     if (rdy == OS_TRUE) {
                         sched = OS_TRUE;                     /* When done we will reschedule          */
                     }
                 }
                 break;

            case OS_FLAG_WAIT_CLR_ANY:               /* See if any flag set                            */
                 flags_rdy = (OS_FLAGS)~pgrp->OSFlagFlags & pnode->OSFlagNodeFlags;
                 if (flags_rdy != (OS_FLAGS)0) {
                     rdy = OS_FlagTaskRdy(pnode, flags_rdy);  /* Make task RTR, event(s) Rx'd          */
                     if (rdy == OS_TRUE) {
                         sched = OS_TRUE;                     /* When done we will reschedule          */
                     }
                 }
                 break;
#endif
            default:
                 OS_EXIT_CRITICAL();
                 *perr = OS_ERR_FLAG_WAIT_TYPE;
                 return ((OS_FLAGS)0);
        }
        pnode = (OS_FLAG_NODE *)pnode->OSFlagNodeNext; /* Point to next task waiting for event flag(s) */
    }
    OS_EXIT_CRITICAL();
    if (sched == OS_TRUE) {
        OS_Sched();
    }
    OS_ENTER_CRITICAL();
    flags_cur = pgrp->OSFlagFlags;
    OS_EXIT_CRITICAL();
    *perr     = OS_ERR_NONE;
    return (flags_cur);
}
```

---

## 8.5 删除事件标志组 - OSFlagDel() 函数

删除事件标志组

* 与之前一样，可以指明删除条件

原理：

* 将 `OS_FLAG_GRP` 置为未使用，并归还给空闲链表
* 遍历所有等待标志组的任务，使所有任务就绪

```c
/*
*********************************************************************************************************
*                                     DELETE AN EVENT FLAG GROUP
*
* Description: This function deletes an event flag group and readies all tasks pending on the event flag
*              group.
*
* Arguments  : pgrp          is a pointer to the desired event flag group.
*
*              opt           determines delete options as follows:
*                            opt == OS_DEL_NO_PEND   Deletes the event flag group ONLY if no task pending
*                            opt == OS_DEL_ALWAYS    Deletes the event flag group even if tasks are
*                                                    waiting.  In this case, all the tasks pending will be
*                                                    readied.
*
*              perr          is a pointer to an error code that can contain one of the following values:
*                            OS_ERR_NONE               The call was successful and the event flag group was
*                                                      deleted
*                            OS_ERR_DEL_ISR            If you attempted to delete the event flag group from
*                                                      an ISR
*                            OS_ERR_FLAG_INVALID_PGRP  If 'pgrp' is a NULL pointer.
*                            OS_ERR_EVENT_TYPE         If you didn't pass a pointer to an event flag group
*                            OS_ERR_INVALID_OPT        An invalid option was specified
*                            OS_ERR_TASK_WAITING       One or more tasks were waiting on the event flag
*                                                      group.
*
* Returns    : pgrp          upon error
*              (OS_EVENT *)0 if the event flag group was successfully deleted.
*
* Note(s)    : 1) This function must be used with care.  Tasks that would normally expect the presence of
*                 the event flag group MUST check the return code of OSFlagAccept() and OSFlagPend().
*              2) This call can potentially disable interrupts for a long time.  The interrupt disable
*                 time is directly proportional to the number of tasks waiting on the event flag group.
*********************************************************************************************************
*/

#if OS_FLAG_DEL_EN > 0u
OS_FLAG_GRP  *OSFlagDel (OS_FLAG_GRP  *pgrp,
                         INT8U         opt,
                         INT8U        *perr)
{
    BOOLEAN       tasks_waiting;
    OS_FLAG_NODE *pnode;
    OS_FLAG_GRP  *pgrp_return;
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register */
    OS_CPU_SR     cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (pgrp == (OS_FLAG_GRP *)0) {                        /* Validate 'pgrp'                          */
        *perr = OS_ERR_FLAG_INVALID_PGRP;
        return (pgrp);
    }
#endif
    if (OSIntNesting > 0u) {                               /* See if called from ISR ...               */
        *perr = OS_ERR_DEL_ISR;                            /* ... can't DELETE from an ISR             */
        return (pgrp);
    }
    if (pgrp->OSFlagType != OS_EVENT_TYPE_FLAG) {          /* Validate event group type                */
        *perr = OS_ERR_EVENT_TYPE;
        return (pgrp);
    }
    OS_ENTER_CRITICAL();
    if (pgrp->OSFlagWaitList != (void *)0) {               /* See if any tasks waiting on event flags  */
        tasks_waiting = OS_TRUE;                           /* Yes                                      */
    } else {
        tasks_waiting = OS_FALSE;                          /* No                                       */
    }
    switch (opt) {
        case OS_DEL_NO_PEND:                               /* Delete group if no task waiting          */
             if (tasks_waiting == OS_FALSE) {
#if OS_FLAG_NAME_EN > 0u
                 pgrp->OSFlagName     = (INT8U *)(void *)"?";
#endif
                 pgrp->OSFlagType     = OS_EVENT_TYPE_UNUSED;
                 pgrp->OSFlagWaitList = (void *)OSFlagFreeList; /* Return group to free list           */
                 pgrp->OSFlagFlags    = (OS_FLAGS)0;
                 OSFlagFreeList       = pgrp;
                 OS_EXIT_CRITICAL();
                 *perr                = OS_ERR_NONE;
                 pgrp_return          = (OS_FLAG_GRP *)0;  /* Event Flag Group has been deleted        */
             } else {
                 OS_EXIT_CRITICAL();
                 *perr                = OS_ERR_TASK_WAITING;
                 pgrp_return          = pgrp;
             }
             break;

        case OS_DEL_ALWAYS:                                /* Always delete the event flag group       */
             pnode = (OS_FLAG_NODE *)pgrp->OSFlagWaitList;
             while (pnode != (OS_FLAG_NODE *)0) {          /* Ready ALL tasks waiting for flags        */
                 (void)OS_FlagTaskRdy(pnode, (OS_FLAGS)0);
                 pnode = (OS_FLAG_NODE *)pnode->OSFlagNodeNext;
             }
#if OS_FLAG_NAME_EN > 0u
             pgrp->OSFlagName     = (INT8U *)(void *)"?";
#endif
             pgrp->OSFlagType     = OS_EVENT_TYPE_UNUSED;
             pgrp->OSFlagWaitList = (void *)OSFlagFreeList;/* Return group to free list                */
             pgrp->OSFlagFlags    = (OS_FLAGS)0;
             OSFlagFreeList       = pgrp;
             OS_EXIT_CRITICAL();
             if (tasks_waiting == OS_TRUE) {               /* Reschedule only if task(s) were waiting  */
                 OS_Sched();                               /* Find highest priority task ready to run  */
             }
             *perr = OS_ERR_NONE;
             pgrp_return          = (OS_FLAG_GRP *)0;      /* Event Flag Group has been deleted        */
             break;

        default:
             OS_EXIT_CRITICAL();
             *perr                = OS_ERR_INVALID_OPT;
             pgrp_return          = pgrp;
             break;
    }
    return (pgrp_return);
}
#endif
```

---

## 8.6 无等待地获得事件标志 - OSFlagAccept() 函数

检查事件标志组中的标志位是置位还是复位

可以检查某一位，也可以检查所有的位

* 传入一个参数，检查所有置 1 的位即可

不挂起调用者，所以任务和中断都可以调用

```c
/*
*********************************************************************************************************
*                              CHECK THE STATUS OF FLAGS IN AN EVENT FLAG GROUP
*
* Description: This function is called to check the status of a combination of bits to be set or cleared
*              in an event flag group.  Your application can check for ANY bit to be set/cleared or ALL
*              bits to be set/cleared.
*
*              This call does not block if the desired flags are not present.
*
* Arguments  : pgrp          is a pointer to the desired event flag group.
*
*              flags         Is a bit pattern indicating which bit(s) (i.e. flags) you wish to check.
*                            The bits you want are specified by setting the corresponding bits in
*                            'flags'.  e.g. if your application wants to wait for bits 0 and 1 then
*                            'flags' would contain 0x03.
*
*              wait_type     specifies whether you want ALL bits to be set/cleared or ANY of the bits
*                            to be set/cleared.
*                            You can specify the following argument:
*
*                            OS_FLAG_WAIT_CLR_ALL   You will check ALL bits in 'flags' to be clear (0)
*                            OS_FLAG_WAIT_CLR_ANY   You will check ANY bit  in 'flags' to be clear (0)
*                            OS_FLAG_WAIT_SET_ALL   You will check ALL bits in 'flags' to be set   (1)
*                            OS_FLAG_WAIT_SET_ANY   You will check ANY bit  in 'flags' to be set   (1)
*
*                            NOTE: Add OS_FLAG_CONSUME if you want the event flag to be 'consumed' by
*                                  the call.  Example, to wait for any flag in a group AND then clear
*                                  the flags that are present, set 'wait_type' to:
*
*                                  OS_FLAG_WAIT_SET_ANY + OS_FLAG_CONSUME
*
*              perr          is a pointer to an error code and can be:
*                            OS_ERR_NONE               No error
*                            OS_ERR_EVENT_TYPE         You are not pointing to an event flag group
*                            OS_ERR_FLAG_WAIT_TYPE     You didn't specify a proper 'wait_type' argument.
*                            OS_ERR_FLAG_INVALID_PGRP  You passed a NULL pointer instead of the event flag
*                                                      group handle.
*                            OS_ERR_FLAG_NOT_RDY       The desired flags you are waiting for are not
*                                                      available.
*
* Returns    : The flags in the event flag group that made the task ready or, 0 if a timeout or an error
*              occurred.
*
* Called from: Task or ISR
*
* Note(s)    : 1) IMPORTANT, the behavior of this function has changed from PREVIOUS versions.  The
*                 function NOW returns the flags that were ready INSTEAD of the current state of the
*                 event flags.
*********************************************************************************************************
*/

#if OS_FLAG_ACCEPT_EN > 0u
OS_FLAGS  OSFlagAccept (OS_FLAG_GRP  *pgrp,
                        OS_FLAGS      flags,
                        INT8U         wait_type,
                        INT8U        *perr)
{
    OS_FLAGS      flags_rdy;
    INT8U         result;
    BOOLEAN       consume;
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register */
    OS_CPU_SR     cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (pgrp == (OS_FLAG_GRP *)0) {                        /* Validate 'pgrp'                          */
        *perr = OS_ERR_FLAG_INVALID_PGRP;
        return ((OS_FLAGS)0);
    }
#endif
    if (pgrp->OSFlagType != OS_EVENT_TYPE_FLAG) {          /* Validate event block type                */
        *perr = OS_ERR_EVENT_TYPE;
        return ((OS_FLAGS)0);
    }
    result = (INT8U)(wait_type & OS_FLAG_CONSUME);
    if (result != (INT8U)0) {                              /* See if we need to consume the flags      */
        wait_type &= ~OS_FLAG_CONSUME;
        consume    = OS_TRUE;
    } else {
        consume    = OS_FALSE;
    }
/*$PAGE*/
    *perr = OS_ERR_NONE;                                   /* Assume NO error until proven otherwise.  */
    OS_ENTER_CRITICAL();
    switch (wait_type) {
        case OS_FLAG_WAIT_SET_ALL:                         /* See if all required flags are set        */
             flags_rdy = (OS_FLAGS)(pgrp->OSFlagFlags & flags);     /* Extract only the bits we want   */
             if (flags_rdy == flags) {                     /* Must match ALL the bits that we want     */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags &= (OS_FLAGS)~flags_rdy;     /* Clear ONLY the flags we wanted  */
                 }
             } else {
                 *perr = OS_ERR_FLAG_NOT_RDY;
             }
             OS_EXIT_CRITICAL();
             break;

        case OS_FLAG_WAIT_SET_ANY:
             flags_rdy = (OS_FLAGS)(pgrp->OSFlagFlags & flags);     /* Extract only the bits we want   */
             if (flags_rdy != (OS_FLAGS)0) {               /* See if any flag set                      */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags &= (OS_FLAGS)~flags_rdy;     /* Clear ONLY the flags we got     */
                 }
             } else {
                 *perr = OS_ERR_FLAG_NOT_RDY;
             }
             OS_EXIT_CRITICAL();
             break;

#if OS_FLAG_WAIT_CLR_EN > 0u
        case OS_FLAG_WAIT_CLR_ALL:                         /* See if all required flags are cleared    */
             flags_rdy = (OS_FLAGS)~pgrp->OSFlagFlags & flags;    /* Extract only the bits we want     */
             if (flags_rdy == flags) {                     /* Must match ALL the bits that we want     */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags |= flags_rdy;       /* Set ONLY the flags that we wanted        */
                 }
             } else {
                 *perr = OS_ERR_FLAG_NOT_RDY;
             }
             OS_EXIT_CRITICAL();
             break;

        case OS_FLAG_WAIT_CLR_ANY:
             flags_rdy = (OS_FLAGS)~pgrp->OSFlagFlags & flags;   /* Extract only the bits we want      */
             if (flags_rdy != (OS_FLAGS)0) {               /* See if any flag cleared                  */
                 if (consume == OS_TRUE) {                 /* See if we need to consume the flags      */
                     pgrp->OSFlagFlags |= flags_rdy;       /* Set ONLY the flags that we got           */
                 }
             } else {
                 *perr = OS_ERR_FLAG_NOT_RDY;
             }
             OS_EXIT_CRITICAL();
             break;
#endif

        default:
             OS_EXIT_CRITICAL();
             flags_rdy = (OS_FLAGS)0;
             *perr     = OS_ERR_FLAG_WAIT_TYPE;
             break;
    }
    return (flags_rdy);
}
#endif
```

---

## 8.7 查询事件标志组的状态 - OSFlagQuery() 函数

查询事件标志组当前的状态

只返回事件标志组中的 flag

```c
/*
*********************************************************************************************************
*                                           QUERY EVENT FLAG
*
* Description: This function is used to check the value of the event flag group.
*
* Arguments  : pgrp         is a pointer to the desired event flag group.
*
*              perr          is a pointer to an error code returned to the called:
*                            OS_ERR_NONE                The call was successfull
*                            OS_ERR_FLAG_INVALID_PGRP   You passed a NULL pointer
*                            OS_ERR_EVENT_TYPE          You are not pointing to an event flag group
*
* Returns    : The current value of the event flag group.
*
* Called From: Task or ISR
*********************************************************************************************************
*/

#if OS_FLAG_QUERY_EN > 0u
OS_FLAGS  OSFlagQuery (OS_FLAG_GRP  *pgrp,
                       INT8U        *perr)
{
    OS_FLAGS   flags;
#if OS_CRITICAL_METHOD == 3u                      /* Allocate storage for CPU status register          */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL
    if (perr == (INT8U *)0) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (pgrp == (OS_FLAG_GRP *)0) {               /* Validate 'pgrp'                                   */
        *perr = OS_ERR_FLAG_INVALID_PGRP;
        return ((OS_FLAGS)0);
    }
#endif
    if (pgrp->OSFlagType != OS_EVENT_TYPE_FLAG) { /* Validate event block type                         */
        *perr = OS_ERR_EVENT_TYPE;
        return ((OS_FLAGS)0);
    }
    OS_ENTER_CRITICAL();
    flags = pgrp->OSFlagFlags;
    OS_EXIT_CRITICAL();
    *perr = OS_ERR_NONE;
    return (flags);                               /* Return the current value of the event flags       */
}
#endif
```

---

## Summary

这个通信机制的实现方式和之前几个有些不一样

主要是数据结构上的变化吧 理清关系就行

---

