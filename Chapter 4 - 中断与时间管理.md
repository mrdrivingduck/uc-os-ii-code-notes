# Chapter 4 - 中断与时间管理

Created by : Mr Dk.

2019 / 11 / 19 21:59

Nanjing, Jiangsu, China

---

## 4.1 中断相关概念

### 4.1.1 中断

中断定义为 CPU 对系统内外发生的异步时间的响应。异步事件：没有时序关系，随机发生的事件。CPU 接到请求后，中止当前工作，保存当前运行环境，转去处理响应的异步事件任务。在中断服务期间，CPU 允许中断嵌套。

### 4.1.2 中断延迟时间

硬件中断发生，到 CPU 开始处理中断（保存 CPU 现场）的时间，是实时内核最重要的指标，通常由关中断的时间决定——关中断的时间越长，中断延迟就越长。

### 4.1.3 中断响应时间

中断响应时间 = 中断延迟 + 保存 CPU 内部寄存器的时间 + 内核进入中断服务函数的时间

### 4.1.4 中断恢复时间

恢复 CPU 内部寄存器 + 执行中断返回指令的时间

### 4.1.6 非屏蔽中断

不能用系统指令关闭的中断

- 优先级高
- 延迟时间短
- 响应快
- 不能被嵌套
- 不能忍受内核的延迟

用于紧急事件处理。

## 4.2 μC/OS-II 的中断处理

### 4.2.1 中断处理程序

μC/OS-II 需要知道当前内核是否正在处理中断，所以在中断服务子程序中，要调用两个函数，通知内核进入或退出中断。这两个函数对 **中断嵌套层数计数器** `OSIntNesting` 进行维护。而 OS 也是通过这个计数器来判断当前内核是否处在中断或嵌套中断中。这个计数器是一个 **单字节** 的整型全局变量，因此嵌套层数最多为 `255`

下面是 `OSIntEnter()` 函数：

- 进入中断处理程序时被调用
- 对计数器 +1

```c
/*
*********************************************************************************************************
*                                              ENTER ISR
*
* Description: This function is used to notify uC/OS-II that you are about to service an interrupt
*              service routine (ISR).  This allows uC/OS-II to keep track of interrupt nesting and thus
*              only perform rescheduling at the last nested ISR.
*
* Arguments  : none
*
* Returns    : none
*
* Notes      : 1) This function should be called ith interrupts already disabled
*              2) Your ISR can directly increment OSIntNesting without calling this function because
*                 OSIntNesting has been declared 'global'.
*              3) You MUST still call OSIntExit() even though you increment OSIntNesting directly.
*              4) You MUST invoke OSIntEnter() and OSIntExit() in pair.  In other words, for every call
*                 to OSIntEnter() at the beginning of the ISR you MUST have a call to OSIntExit() at the
*                 end of the ISR.
*              5) You are allowed to nest interrupts up to 255 levels deep.
*              6) I removed the OS_ENTER_CRITICAL() and OS_EXIT_CRITICAL() around the increment because
*                 OSIntEnter() is always called with interrupts disabled.
*********************************************************************************************************
*/

void  OSIntEnter (void)
{
    if (OSRunning == OS_TRUE) {
        if (OSIntNesting < 255u) {
            OSIntNesting++;                      /* Increment ISR nesting level                        */
        }
    }
}
```

下面是 `OSIntExit()` 函数：

- 退出中断处理程序时被调用
- 计数器 -1
- 找出就绪的最高优先级任务
- 判断最高优先级任务是否是被中断打断的任务，如果不是，则切换 TCB 指针
- 调用 **中断级任务切换函数** `OSIntCtxSw()` 进行任务切换

注意，任务切换的条件是：中断嵌套计数器和调度锁定嵌套计数器同时为 0。

```c
/*
*********************************************************************************************************
*                                               EXIT ISR
*
* Description: This function is used to notify uC/OS-II that you have completed serviving an ISR.  When
*              the last nested ISR has completed, uC/OS-II will call the scheduler to determine whether
*              a new, high-priority task, is ready to run.
*
* Arguments  : none
*
* Returns    : none
*
* Notes      : 1) You MUST invoke OSIntEnter() and OSIntExit() in pair.  In other words, for every call
*                 to OSIntEnter() at the beginning of the ISR you MUST have a call to OSIntExit() at the
*                 end of the ISR.
*              2) Rescheduling is prevented when the scheduler is locked (see OS_SchedLock())
*********************************************************************************************************
*/

void  OSIntExit (void)
{
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    if (OSRunning == OS_TRUE) {
        OS_ENTER_CRITICAL();
        if (OSIntNesting > 0u) {                           /* Prevent OSIntNesting from wrapping       */
            OSIntNesting--;
        }
        if (OSIntNesting == 0u) {                          /* Reschedule only if all ISRs complete ... */
            if (OSLockNesting == 0u) {                     /* ... and not locked.                      */
                OS_SchedNew();
                OSTCBHighRdy = OSTCBPrioTbl[OSPrioHighRdy];
                if (OSPrioHighRdy != OSPrioCur) {          /* No Ctx Sw if current task is highest rdy */
#if OS_TASK_PROFILE_EN > 0u
                    OSTCBHighRdy->OSTCBCtxSwCtr++;         /* Inc. # of context switches to this task  */
#endif
                    OSCtxSwCtr++;                          /* Keep track of the number of ctx switches */
                    OSIntCtxSw();                          /* Perform interrupt level ctx switch       */
                }
            }
        }
        OS_EXIT_CRITICAL();
    }
}
```

## 4.3 μC/OS-II 的时钟节拍

### 4.3.1 时钟节拍

时钟节拍是特定的周期性中断（时钟中断），是系统心脏的脉动。

### 4.3.2 时钟节拍程序

时钟节拍中断服务子程序 `OSTickISR()`：每个时钟周期执行一次，必须用汇编实现。

节拍服务子函数 `OSTImeTick()` - 由上一个函数调用：

- 给每个 TCB 的延迟项 `OSTCBDly` 减 1 直至为 0
- 执行时间与任务个数成正比

```c
/*
*********************************************************************************************************
*                                         PROCESS SYSTEM TICK
*
* Description: This function is used to signal to uC/OS-II the occurrence of a 'system tick' (also known
*              as a 'clock tick').  This function should be called by the ticker ISR but, can also be
*              called by a high priority task.
*
* Arguments  : none
*
* Returns    : none
*********************************************************************************************************
*/

void  OSTimeTick (void)
{
    OS_TCB    *ptcb;
#if OS_TICK_STEP_EN > 0u
    BOOLEAN    step;
#endif
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register     */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#if OS_TIME_TICK_HOOK_EN > 0u
    OSTimeTickHook();                                      /* Call user definable hook                     */
#endif
#if OS_TIME_GET_SET_EN > 0u
    OS_ENTER_CRITICAL();                                   /* Update the 32-bit tick counter               */
    OSTime++;
    OS_EXIT_CRITICAL();
#endif
    if (OSRunning == OS_TRUE) {
#if OS_TICK_STEP_EN > 0u
        switch (OSTickStepState) {                         /* Determine whether we need to process a tick  */
            case OS_TICK_STEP_DIS:                         /* Yes, stepping is disabled                    */
                 step = OS_TRUE;
                 break;

            case OS_TICK_STEP_WAIT:                        /* No,  waiting for uC/OS-View to set ...       */
                 step = OS_FALSE;                          /*      .. OSTickStepState to OS_TICK_STEP_ONCE */
                 break;

            case OS_TICK_STEP_ONCE:                        /* Yes, process tick once and wait for next ... */
                 step            = OS_TRUE;                /*      ... step command from uC/OS-View        */
                 OSTickStepState = OS_TICK_STEP_WAIT;
                 break;

            default:                                       /* Invalid case, correct situation              */
                 step            = OS_TRUE;
                 OSTickStepState = OS_TICK_STEP_DIS;
                 break;
        }
        if (step == OS_FALSE) {                            /* Return if waiting for step command           */
            return;
        }
#endif
        ptcb = OSTCBList;                                  /* Point at first TCB in TCB list               */
        while (ptcb->OSTCBPrio != OS_TASK_IDLE_PRIO) {     /* Go through all TCBs in TCB list              */
            OS_ENTER_CRITICAL();
            if (ptcb->OSTCBDly != 0u) {                    /* No, Delayed or waiting for event with TO     */
                ptcb->OSTCBDly--;                          /* Decrement nbr of ticks to end of delay       */
                if (ptcb->OSTCBDly == 0u) {                /* Check for timeout                            */

                    if ((ptcb->OSTCBStat & OS_STAT_PEND_ANY) != OS_STAT_RDY) {
                        ptcb->OSTCBStat  &= (INT8U)~(INT8U)OS_STAT_PEND_ANY;          /* Yes, Clear status flag   */
                        ptcb->OSTCBStatPend = OS_STAT_PEND_TO;                 /* Indicate PEND timeout    */
                    } else {
                        ptcb->OSTCBStatPend = OS_STAT_PEND_OK;
                    }

                    if ((ptcb->OSTCBStat & OS_STAT_SUSPEND) == OS_STAT_RDY) {  /* Is task suspended?       */
                        OSRdyGrp               |= ptcb->OSTCBBitY;             /* No,  Make ready          */
                        OSRdyTbl[ptcb->OSTCBY] |= ptcb->OSTCBBitX;
                    }
                }
            }
            ptcb = ptcb->OSTCBNext;                        /* Point at next TCB in TCB list                */
            OS_EXIT_CRITICAL();
        }
    }
}
```

### 4.3.3 时钟节拍器的正确用法

多任务启动后再开放时钟中断，防止时钟中断在 OS 启动第一个任务前发生。

## 4.4 μC/OS-II 的时间管理

### 4.4.1 任务延时函数 - OSTimeDly() 函数

除了永不挂起的空闲任务外，其它任务总该在合适的时候调用系统服务函数，自我挂起。暂时放弃 CPU 使用权，使得低优先级的任务得以运行。该函数向 OS 申请延时，申请以时钟节拍为单位。当：

- 延时时间到
- 其它任务调用 `OSTimeDlyResume()` 取消了延时

任务才会重新进入就绪状态。当然，被不被调度取决于任务的优先级。

原理：

- 将任务从就绪表中移出
- 在 TCB 中记录延时时间，由时钟中断递减该时间

```c
/*
*********************************************************************************************************
*                                       DELAY TASK 'n' TICKS
*
* Description: This function is called to delay execution of the currently running task until the
*              specified number of system ticks expires.  This, of course, directly equates to delaying
*              the current task for some time to expire.  No delay will result If the specified delay is
*              0.  If the specified delay is greater than 0 then, a context switch will result.
*
* Arguments  : ticks     is the time delay that the task will be suspended in number of clock 'ticks'.
*                        Note that by specifying 0, the task will not be delayed.
*
* Returns    : none
*********************************************************************************************************
*/

void  OSTimeDly (INT32U ticks)
{
    INT8U      y;
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    if (OSIntNesting > 0u) {                     /* See if trying to call from an ISR                  */
        return;
    }
    if (OSLockNesting > 0u) {                    /* See if called with scheduler locked                */
        return;
    }
    if (ticks > 0u) {                            /* 0 means no delay!                                  */
        OS_ENTER_CRITICAL();
        y            =  OSTCBCur->OSTCBY;        /* Delay current task                                 */
        OSRdyTbl[y] &= (OS_PRIO)~OSTCBCur->OSTCBBitX;
        if (OSRdyTbl[y] == 0u) {
            OSRdyGrp &= (OS_PRIO)~OSTCBCur->OSTCBBitY;
        }
        OSTCBCur->OSTCBDly = ticks;              /* Load ticks in TCB                                  */
        OS_EXIT_CRITICAL();
        OS_Sched();                              /* Find next task to run!                             */
    }
}
```

### 4.4.2 按时分秒毫秒延时函数 - OSTimeDlyHMSM() 函数

按时间单位为单位进行延时，显然需要将延时参数换算成时钟节拍数量，记录在 TCB 中。所以需要利用全局常量 `OS_TICKS_PER_SEC` 将时间转换为节拍数。这个常量的含义：每秒产生多少个节拍。

```c
/*
*********************************************************************************************************
*                                     DELAY TASK FOR SPECIFIED TIME
*
* Description: This function is called to delay execution of the currently running task until some time
*              expires.  This call allows you to specify the delay time in HOURS, MINUTES, SECONDS and
*              MILLISECONDS instead of ticks.
*
* Arguments  : hours     specifies the number of hours that the task will be delayed (max. is 255)
*              minutes   specifies the number of minutes (max. 59)
*              seconds   specifies the number of seconds (max. 59)
*              ms        specifies the number of milliseconds (max. 999)
*
* Returns    : OS_ERR_NONE
*              OS_ERR_TIME_INVALID_MINUTES
*              OS_ERR_TIME_INVALID_SECONDS
*              OS_ERR_TIME_INVALID_MS
*              OS_ERR_TIME_ZERO_DLY
*              OS_ERR_TIME_DLY_ISR
*
* Note(s)    : The resolution on the milliseconds depends on the tick rate.  For example, you can't do
*              a 10 mS delay if the ticker interrupts every 100 mS.  In this case, the delay would be
*              set to 0.  The actual delay is rounded to the nearest tick.
*********************************************************************************************************
*/

#if OS_TIME_DLY_HMSM_EN > 0u
INT8U  OSTimeDlyHMSM (INT8U   hours,
                      INT8U   minutes,
                      INT8U   seconds,
                      INT16U  ms)
{
    INT32U ticks;


    if (OSIntNesting > 0u) {                     /* See if trying to call from an ISR                  */
        return (OS_ERR_TIME_DLY_ISR);
    }
    if (OSLockNesting > 0u) {                    /* See if called with scheduler locked                */
        return (OS_ERR_SCHED_LOCKED);
    }
#if OS_ARG_CHK_EN > 0u
    if (hours == 0u) {
        if (minutes == 0u) {
            if (seconds == 0u) {
                if (ms == 0u) {
                    return (OS_ERR_TIME_ZERO_DLY);
                }
            }
        }
    }
    if (minutes > 59u) {
        return (OS_ERR_TIME_INVALID_MINUTES);    /* Validate arguments to be within range              */
    }
    if (seconds > 59u) {
        return (OS_ERR_TIME_INVALID_SECONDS);
    }
    if (ms > 999u) {
        return (OS_ERR_TIME_INVALID_MS);
    }
#endif
                                                 /* Compute the total number of clock ticks required.. */
                                                 /* .. (rounded to the nearest tick)                   */
    ticks = ((INT32U)hours * 3600uL + (INT32U)minutes * 60uL + (INT32U)seconds) * OS_TICKS_PER_SEC
          + OS_TICKS_PER_SEC * ((INT32U)ms + 500uL / OS_TICKS_PER_SEC) / 1000uL;
    OSTimeDly(ticks);
    return (OS_ERR_NONE);
}
#endif
```

### 4.4.3 结束任务延时 - OSTimeDlyResume() 函数

唤醒一个用上面两个延时函数挂起的任务，取消延时，使其进入就绪状态

原理：

- 将 TCB 中的 `OSTCBDly` 直接清 0
- 将任务插入到就绪表中，如果有需要的话，需要重新调度
- 不使用延时挂起函数挂起的任务不能用这个函数唤醒

```c
/*
*********************************************************************************************************
*                                         RESUME A DELAYED TASK
*
* Description: This function is used resume a task that has been delayed through a call to either
*              OSTimeDly() or OSTimeDlyHMSM().  Note that you can call this function to resume a
*              task that is waiting for an event with timeout.  This would make the task look
*              like a timeout occurred.
*
* Arguments  : prio                      specifies the priority of the task to resume
*
* Returns    : OS_ERR_NONE               Task has been resumed
*              OS_ERR_PRIO_INVALID       if the priority you specify is higher that the maximum allowed
*                                        (i.e. >= OS_LOWEST_PRIO)
*              OS_ERR_TIME_NOT_DLY       Task is not waiting for time to expire
*              OS_ERR_TASK_NOT_EXIST     The desired task has not been created or has been assigned to a Mutex.
*********************************************************************************************************
*/

#if OS_TIME_DLY_RESUME_EN > 0u
INT8U  OSTimeDlyResume (INT8U prio)
{
    OS_TCB    *ptcb;
#if OS_CRITICAL_METHOD == 3u                                   /* Storage for CPU status register      */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    if (prio >= OS_LOWEST_PRIO) {
        return (OS_ERR_PRIO_INVALID);
    }
    OS_ENTER_CRITICAL();
    ptcb = OSTCBPrioTbl[prio];                                 /* Make sure that task exist            */
    if (ptcb == (OS_TCB *)0) {
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);                        /* The task does not exist              */
    }
    if (ptcb == OS_TCB_RESERVED) {
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);                        /* The task does not exist              */
    }
    if (ptcb->OSTCBDly == 0u) {                                /* See if task is delayed               */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TIME_NOT_DLY);                          /* Indicate that task was not delayed   */
    }

    ptcb->OSTCBDly = 0u;                                       /* Clear the time delay                 */
    if ((ptcb->OSTCBStat & OS_STAT_PEND_ANY) != OS_STAT_RDY) {
        ptcb->OSTCBStat     &= ~OS_STAT_PEND_ANY;              /* Yes, Clear status flag               */
        ptcb->OSTCBStatPend  =  OS_STAT_PEND_TO;               /* Indicate PEND timeout                */
    } else {
        ptcb->OSTCBStatPend  =  OS_STAT_PEND_OK;
    }
    if ((ptcb->OSTCBStat & OS_STAT_SUSPEND) == OS_STAT_RDY) {  /* Is task suspended?                   */
        OSRdyGrp               |= ptcb->OSTCBBitY;             /* No,  Make ready                      */
        OSRdyTbl[ptcb->OSTCBY] |= ptcb->OSTCBBitX;
        OS_EXIT_CRITICAL();
        OS_Sched();                                            /* See if this is new highest priority  */
    } else {
        OS_EXIT_CRITICAL();                                    /* Task may be suspended                */
    }
    return (OS_ERR_NONE);
}
#endif
```

### 4.4.4 系统时间函数 - OSTimeGet() 和 OSTimeSet()

时钟节拍每发生一次，OS 就会给一个 32-bit 计数器 `OSTime` + 1。这两个函数分别 get 或 set 这个计数器。这个全局变量是 32-bit 的，因此在 8-bit 的 CPU 上操作需要多条指令，这些指令需要一次执行完毕——因此需要在临界区中！

```c
/*
*********************************************************************************************************
*                                         GET CURRENT SYSTEM TIME
*
* Description: This function is used by your application to obtain the current value of the 32-bit
*              counter which keeps track of the number of clock ticks.
*
* Arguments  : none
*
* Returns    : The current value of OSTime
*********************************************************************************************************
*/

#if OS_TIME_GET_SET_EN > 0u
INT32U  OSTimeGet (void)
{
    INT32U     ticks;
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    OS_ENTER_CRITICAL();
    ticks = OSTime;
    OS_EXIT_CRITICAL();
    return (ticks);
}
#endif
```

```c
/*
*********************************************************************************************************
*                                            SET SYSTEM CLOCK
*
* Description: This function sets the 32-bit counter which keeps track of the number of clock ticks.
*
* Arguments  : ticks      specifies the new value that OSTime needs to take.
*
* Returns    : none
*********************************************************************************************************
*/

#if OS_TIME_GET_SET_EN > 0u
void  OSTimeSet (INT32U ticks)
{
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    OS_ENTER_CRITICAL();
    OSTime = ticks;
    OS_EXIT_CRITICAL();
}
#endif
```
