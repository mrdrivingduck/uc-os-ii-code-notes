# Chapter 3.2 - 用户任务管理

Created by : Mr Dk.

2019 / 11 / 18 22:54

Nanjing, Jiangsu, China

---

## 3.2.1 任务栈管理

三种堆栈：

* CPU 硬件堆栈 (系统栈或硬件栈)
    * 保存中断或函数调用返回的 PC 以及其它 CPU 寄存器值
* 可重入函数的仿真堆栈
* 任务堆栈 (任务栈或用户栈)

任务栈可以静态或动态分配

任务栈的初始化仅为任务的第一次运行做准备

* 除了 PC 外，其余寄存器的值没有任何意义

--

## 3.2.2 建立任务 - OSTaskCreate() 函数

将任务的 __代码地址__ 和 __参数__ 传递给内核

* 显然，该函数要判断优先级的有效性，是否被占用
* 之后分配并初始化任务的 TCB
* 该函数可以在多任务启动前或启动后调用
    * 如果在多任务启动后调用
    * 还需要判断一下是否高于当前程序的优先级
    * 如果高于当前优先级，则发生任务切换，挂起当前任务

```c
/*
*********************************************************************************************************
*                                            CREATE A TASK
*
* Description: This function is used to have uC/OS-II manage the execution of a task.  Tasks can either
*              be created prior to the start of multitasking or by a running task.  A task cannot be
*              created by an ISR.
*
* Arguments  : task     is a pointer to the task's code
*
*              p_arg    is a pointer to an optional data area which can be used to pass parameters to
*                       the task when the task first executes.  Where the task is concerned it thinks
*                       it was invoked and passed the argument 'p_arg' as follows:
*
*                           void Task (void *p_arg)
*                           {
*                               for (;;) {
*                                   Task code;
*                               }
*                           }
*
*              ptos     is a pointer to the task's top of stack.  If the configuration constant
*                       OS_STK_GROWTH is set to 1, the stack is assumed to grow downward (i.e. from high
*                       memory to low memory).  'pstk' will thus point to the highest (valid) memory
*                       location of the stack.  If OS_STK_GROWTH is set to 0, 'pstk' will point to the
*                       lowest memory location of the stack and the stack will grow with increasing
*                       memory locations.
*
*              prio     is the task's priority.  A unique priority MUST be assigned to each task and the
*                       lower the number, the higher the priority.
*
* Returns    : OS_ERR_NONE             if the function was successful.
*              OS_PRIO_EXIT            if the task priority already exist
*                                      (each task MUST have a unique priority).
*              OS_ERR_PRIO_INVALID     if the priority you specify is higher that the maximum allowed
*                                      (i.e. >= OS_LOWEST_PRIO)
*              OS_ERR_TASK_CREATE_ISR  if you tried to create a task from an ISR.
*********************************************************************************************************
*/

#if OS_TASK_CREATE_EN > 0u
INT8U  OSTaskCreate (void   (*task)(void *p_arg),
                     void    *p_arg,
                     OS_STK  *ptos,
                     INT8U    prio)
{
    OS_STK    *psp;
    INT8U      err;
#if OS_CRITICAL_METHOD == 3u                 /* Allocate storage for CPU status register               */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL_IEC61508
    if (OSSafetyCriticalStartFlag == OS_TRUE) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (prio > OS_LOWEST_PRIO) {             /* Make sure priority is within allowable range           */
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    if (OSIntNesting > 0u) {                 /* Make sure we don't create the task from within an ISR  */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_CREATE_ISR);
    }
    if (OSTCBPrioTbl[prio] == (OS_TCB *)0) { /* Make sure task doesn't already exist at this priority  */
        OSTCBPrioTbl[prio] = OS_TCB_RESERVED;/* Reserve the priority to prevent others from doing ...  */
                                             /* ... the same thing until task is created.              */
        OS_EXIT_CRITICAL();
        psp = OSTaskStkInit(task, p_arg, ptos, 0u);             /* Initialize the task's stack         */
        err = OS_TCBInit(prio, psp, (OS_STK *)0, 0u, 0u, (void *)0, 0u);
        if (err == OS_ERR_NONE) {
            if (OSRunning == OS_TRUE) {      /* Find highest priority task if multitasking has started */
                OS_Sched();
            }
        } else {
            OS_ENTER_CRITICAL();
            OSTCBPrioTbl[prio] = (OS_TCB *)0;/* Make this priority available to others                 */
            OS_EXIT_CRITICAL();
        }
        return (err);
    }
    OS_EXIT_CRITICAL();
    return (OS_ERR_PRIO_EXIST);
}
#endif
```

---

## 3.2.3 建立任务 - OSTaskCreateExt() 函数

前一个函数的扩展版本

总体上逻辑类似

```c
/*
*********************************************************************************************************
*                                     CREATE A TASK (Extended Version)
*
* Description: This function is used to have uC/OS-II manage the execution of a task.  Tasks can either
*              be created prior to the start of multitasking or by a running task.  A task cannot be
*              created by an ISR.  This function is similar to OSTaskCreate() except that it allows
*              additional information about a task to be specified.
*
* Arguments  : task      is a pointer to the task's code
*
*              p_arg     is a pointer to an optional data area which can be used to pass parameters to
*                        the task when the task first executes.  Where the task is concerned it thinks
*                        it was invoked and passed the argument 'p_arg' as follows:
*
*                            void Task (void *p_arg)
*                            {
*                                for (;;) {
*                                    Task code;
*                                }
*                            }
*
*              ptos      is a pointer to the task's top of stack.  If the configuration constant
*                        OS_STK_GROWTH is set to 1, the stack is assumed to grow downward (i.e. from high
*                        memory to low memory).  'ptos' will thus point to the highest (valid) memory
*                        location of the stack.  If OS_STK_GROWTH is set to 0, 'ptos' will point to the
*                        lowest memory location of the stack and the stack will grow with increasing
*                        memory locations.  'ptos' MUST point to a valid 'free' data item.
*
*              prio      is the task's priority.  A unique priority MUST be assigned to each task and the
*                        lower the number, the higher the priority.
*
*              id        is the task's ID (0..65535)
*
*              pbos      is a pointer to the task's bottom of stack.  If the configuration constant
*                        OS_STK_GROWTH is set to 1, the stack is assumed to grow downward (i.e. from high
*                        memory to low memory).  'pbos' will thus point to the LOWEST (valid) memory
*                        location of the stack.  If OS_STK_GROWTH is set to 0, 'pbos' will point to the
*                        HIGHEST memory location of the stack and the stack will grow with increasing
*                        memory locations.  'pbos' MUST point to a valid 'free' data item.
*
*              stk_size  is the size of the stack in number of elements.  If OS_STK is set to INT8U,
*                        'stk_size' corresponds to the number of bytes available.  If OS_STK is set to
*                        INT16U, 'stk_size' contains the number of 16-bit entries available.  Finally, if
*                        OS_STK is set to INT32U, 'stk_size' contains the number of 32-bit entries
*                        available on the stack.
*
*              pext      is a pointer to a user supplied memory location which is used as a TCB extension.
*                        For example, this user memory can hold the contents of floating-point registers
*                        during a context switch, the time each task takes to execute, the number of times
*                        the task has been switched-in, etc.
*
*              opt       contains additional information (or options) about the behavior of the task.  The
*                        LOWER 8-bits are reserved by uC/OS-II while the upper 8 bits can be application
*                        specific.  See OS_TASK_OPT_??? in uCOS-II.H.  Current choices are:
*
*                        OS_TASK_OPT_STK_CHK      Stack checking to be allowed for the task
*                        OS_TASK_OPT_STK_CLR      Clear the stack when the task is created
*                        OS_TASK_OPT_SAVE_FP      If the CPU has floating-point registers, save them
*                                                 during a context switch.
*
* Returns    : OS_ERR_NONE             if the function was successful.
*              OS_PRIO_EXIT            if the task priority already exist
*                                      (each task MUST have a unique priority).
*              OS_ERR_PRIO_INVALID     if the priority you specify is higher that the maximum allowed
*                                      (i.e. > OS_LOWEST_PRIO)
*              OS_ERR_TASK_CREATE_ISR  if you tried to create a task from an ISR.
*********************************************************************************************************
*/
/*$PAGE*/
#if OS_TASK_CREATE_EXT_EN > 0u
INT8U  OSTaskCreateExt (void   (*task)(void *p_arg),
                        void    *p_arg,
                        OS_STK  *ptos,
                        INT8U    prio,
                        INT16U   id,
                        OS_STK  *pbos,
                        INT32U   stk_size,
                        void    *pext,
                        INT16U   opt)
{
    OS_STK    *psp;
    INT8U      err;
#if OS_CRITICAL_METHOD == 3u                 /* Allocate storage for CPU status register               */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#ifdef OS_SAFETY_CRITICAL_IEC61508
    if (OSSafetyCriticalStartFlag == OS_TRUE) {
        OS_SAFETY_CRITICAL_EXCEPTION();
    }
#endif

#if OS_ARG_CHK_EN > 0u
    if (prio > OS_LOWEST_PRIO) {             /* Make sure priority is within allowable range           */
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    if (OSIntNesting > 0u) {                 /* Make sure we don't create the task from within an ISR  */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_CREATE_ISR);
    }
    if (OSTCBPrioTbl[prio] == (OS_TCB *)0) { /* Make sure task doesn't already exist at this priority  */
        OSTCBPrioTbl[prio] = OS_TCB_RESERVED;/* Reserve the priority to prevent others from doing ...  */
                                             /* ... the same thing until task is created.              */
        OS_EXIT_CRITICAL();

#if (OS_TASK_STAT_STK_CHK_EN > 0u)
        OS_TaskStkClr(pbos, stk_size, opt);                    /* Clear the task stack (if needed)     */
#endif

        psp = OSTaskStkInit(task, p_arg, ptos, opt);           /* Initialize the task's stack          */
        err = OS_TCBInit(prio, psp, pbos, id, stk_size, pext, opt);
        if (err == OS_ERR_NONE) {
            if (OSRunning == OS_TRUE) {                        /* Find HPT if multitasking has started */
                OS_Sched();
            }
        } else {
            OS_ENTER_CRITICAL();
            OSTCBPrioTbl[prio] = (OS_TCB *)0;                  /* Make this priority avail. to others  */
            OS_EXIT_CRITICAL();
        }
        return (err);
    }
    OS_EXIT_CRITICAL();
    return (OS_ERR_PRIO_EXIST);
}
#endif
```

---

## 3.2.4 优先级变更 - OSTaskChangePrio() 函数

调该函数可以动态改变任务的优先级

调用者只能是任务

实现：

* 根据原任务处于就绪态还是等待态，在就绪表或等待表中用新优先级替换原优先级
* 修改 TCB
* 优先级变更后，会调用调度函数，可能会发生一次任务切换

新优先级必须是没有用过的

```c
/*
*********************************************************************************************************
*                                        CHANGE PRIORITY OF A TASK
*
* Description: This function allows you to change the priority of a task dynamically.  Note that the new
*              priority MUST be available.
*
* Arguments  : oldp     is the old priority
*
*              newp     is the new priority
*
* Returns    : OS_ERR_NONE            is the call was successful
*              OS_ERR_PRIO_INVALID    if the priority you specify is higher that the maximum allowed
*                                     (i.e. >= OS_LOWEST_PRIO)
*              OS_ERR_PRIO_EXIST      if the new priority already exist.
*              OS_ERR_PRIO            there is no task with the specified OLD priority (i.e. the OLD task does
*                                     not exist.
*              OS_ERR_TASK_NOT_EXIST  if the task is assigned to a Mutex PIP.
*********************************************************************************************************
*/

#if OS_TASK_CHANGE_PRIO_EN > 0u
INT8U  OSTaskChangePrio (INT8U  oldprio,
                         INT8U  newprio)
{
#if (OS_EVENT_EN)
    OS_EVENT  *pevent;
#if (OS_EVENT_MULTI_EN > 0u)
    OS_EVENT **pevents;
#endif
#endif
    OS_TCB    *ptcb;
    INT8U      y_new;
    INT8U      x_new;
    INT8U      y_old;
    OS_PRIO    bity_new;
    OS_PRIO    bitx_new;
    OS_PRIO    bity_old;
    OS_PRIO    bitx_old;
#if OS_CRITICAL_METHOD == 3u
    OS_CPU_SR  cpu_sr = 0u;                                 /* Storage for CPU status register         */
#endif


/*$PAGE*/
#if OS_ARG_CHK_EN > 0u
    if (oldprio >= OS_LOWEST_PRIO) {
        if (oldprio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
    if (newprio >= OS_LOWEST_PRIO) {
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    if (OSTCBPrioTbl[newprio] != (OS_TCB *)0) {             /* New priority must not already exist     */
        OS_EXIT_CRITICAL();
        return (OS_ERR_PRIO_EXIST);
    }
    if (oldprio == OS_PRIO_SELF) {                          /* See if changing self                    */
        oldprio = OSTCBCur->OSTCBPrio;                      /* Yes, get priority                       */
    }
    ptcb = OSTCBPrioTbl[oldprio];
    if (ptcb == (OS_TCB *)0) {                              /* Does task to change exist?              */
        OS_EXIT_CRITICAL();                                 /* No, can't change its priority!          */
        return (OS_ERR_PRIO);
    }
    if (ptcb == OS_TCB_RESERVED) {                          /* Is task assigned to Mutex               */
        OS_EXIT_CRITICAL();                                 /* No, can't change its priority!          */
        return (OS_ERR_TASK_NOT_EXIST);
    }
#if OS_LOWEST_PRIO <= 63u
    y_new                 = (INT8U)(newprio >> 3u);         /* Yes, compute new TCB fields             */
    x_new                 = (INT8U)(newprio & 0x07u);
#else
    y_new                 = (INT8U)((INT8U)(newprio >> 4u) & 0x0Fu);
    x_new                 = (INT8U)(newprio & 0x0Fu);
#endif
    bity_new              = (OS_PRIO)(1uL << y_new);
    bitx_new              = (OS_PRIO)(1uL << x_new);

    OSTCBPrioTbl[oldprio] = (OS_TCB *)0;                    /* Remove TCB from old priority            */
    OSTCBPrioTbl[newprio] =  ptcb;                          /* Place pointer to TCB @ new priority     */
    y_old                 =  ptcb->OSTCBY;
    bity_old              =  ptcb->OSTCBBitY;
    bitx_old              =  ptcb->OSTCBBitX;
    if ((OSRdyTbl[y_old] &   bitx_old) != 0u) {             /* If task is ready make it not            */
         OSRdyTbl[y_old] &= (OS_PRIO)~bitx_old;
         if (OSRdyTbl[y_old] == 0u) {
             OSRdyGrp &= (OS_PRIO)~bity_old;
         }
         OSRdyGrp        |= bity_new;                       /* Make new priority ready to run          */
         OSRdyTbl[y_new] |= bitx_new;
    }

#if (OS_EVENT_EN)
    pevent = ptcb->OSTCBEventPtr;
    if (pevent != (OS_EVENT *)0) {
        pevent->OSEventTbl[y_old] &= (OS_PRIO)~bitx_old;    /* Remove old task prio from wait list     */
        if (pevent->OSEventTbl[y_old] == 0u) {
            pevent->OSEventGrp    &= (OS_PRIO)~bity_old;
        }
        pevent->OSEventGrp        |= bity_new;              /* Add    new task prio to   wait list     */
        pevent->OSEventTbl[y_new] |= bitx_new;
    }
#if (OS_EVENT_MULTI_EN > 0u)
    if (ptcb->OSTCBEventMultiPtr != (OS_EVENT **)0) {
        pevents =  ptcb->OSTCBEventMultiPtr;
        pevent  = *pevents;
        while (pevent != (OS_EVENT *)0) {
            pevent->OSEventTbl[y_old] &= (OS_PRIO)~bitx_old;   /* Remove old task prio from wait lists */
            if (pevent->OSEventTbl[y_old] == 0u) {
                pevent->OSEventGrp    &= (OS_PRIO)~bity_old;
            }
            pevent->OSEventGrp        |= bity_new;          /* Add    new task prio to   wait lists    */
            pevent->OSEventTbl[y_new] |= bitx_new;
            pevents++;
            pevent                     = *pevents;
        }
    }
#endif
#endif

    ptcb->OSTCBPrio = newprio;                              /* Set new task priority                   */
    ptcb->OSTCBY    = y_new;
    ptcb->OSTCBX    = x_new;
    ptcb->OSTCBBitY = bity_new;
    ptcb->OSTCBBitX = bitx_new;
    OS_EXIT_CRITICAL();
    if (OSRunning == OS_TRUE) {
        OS_Sched();                                         /* Find new highest priority task          */
    }
    return (OS_ERR_NONE);
}
#endif
```

---

## 3.2.5 删除任务 - OSTaskDel() 函数

不删除任务代码

* 使任务返回并处于休眠
* 任务的代码不再被 OS 管理，除非重新启动

调用后，下一个就绪的最高优先级任务开始运行

* 发生一次任务切换

可以删除自己

但是在删除占用系统资源的任务时要小心

具体实现：

* 将该任务从内核中的所有地方移走
    * 如果就绪，就从就绪表中移走
    * 如果在等待，就从等待列表中移走
* 将 TCB 设置后，从链表中删除
* 重新调度

```c
/*
*********************************************************************************************************
*                                            DELETE A TASK
*
* Description: This function allows you to delete a task.  The calling task can delete itself by
*              its own priority number.  The deleted task is returned to the dormant state and can be
*              re-activated by creating the deleted task again.
*
* Arguments  : prio    is the priority of the task to delete.  Note that you can explicitely delete
*                      the current task without knowing its priority level by setting 'prio' to
*                      OS_PRIO_SELF.
*
* Returns    : OS_ERR_NONE             if the call is successful
*              OS_ERR_TASK_DEL_IDLE    if you attempted to delete uC/OS-II's idle task
*              OS_ERR_PRIO_INVALID     if the priority you specify is higher that the maximum allowed
*                                      (i.e. >= OS_LOWEST_PRIO) or, you have not specified OS_PRIO_SELF.
*              OS_ERR_TASK_DEL         if the task is assigned to a Mutex PIP.
*              OS_ERR_TASK_NOT_EXIST   if the task you want to delete does not exist.
*              OS_ERR_TASK_DEL_ISR     if you tried to delete a task from an ISR
*
* Notes      : 1) To reduce interrupt latency, OSTaskDel() 'disables' the task:
*                    a) by making it not ready
*                    b) by removing it from any wait lists
*                    c) by preventing OSTimeTick() from making the task ready to run.
*                 The task can then be 'unlinked' from the miscellaneous structures in uC/OS-II.
*              2) The function OS_Dummy() is called after OS_EXIT_CRITICAL() because, on most processors,
*                 the next instruction following the enable interrupt instruction is ignored.
*              3) An ISR cannot delete a task.
*              4) The lock nesting counter is incremented because, for a brief instant, if the current
*                 task is being deleted, the current task would not be able to be rescheduled because it
*                 is removed from the ready list.  Incrementing the nesting counter prevents another task
*                 from being schedule.  This means that an ISR would return to the current task which is
*                 being deleted.  The rest of the deletion would thus be able to be completed.
*********************************************************************************************************
*/

#if OS_TASK_DEL_EN > 0u
INT8U  OSTaskDel (INT8U prio)
{
#if (OS_FLAG_EN > 0u) && (OS_MAX_FLAGS > 0u)
    OS_FLAG_NODE *pnode;
#endif
    OS_TCB       *ptcb;
#if OS_CRITICAL_METHOD == 3u                            /* Allocate storage for CPU status register    */
    OS_CPU_SR     cpu_sr = 0u;
#endif



    if (OSIntNesting > 0u) {                            /* See if trying to delete from ISR            */
        return (OS_ERR_TASK_DEL_ISR);
    }
    if (prio == OS_TASK_IDLE_PRIO) {                    /* Not allowed to delete idle task             */
        return (OS_ERR_TASK_DEL_IDLE);
    }
#if OS_ARG_CHK_EN > 0u
    if (prio >= OS_LOWEST_PRIO) {                       /* Task priority valid ?                       */
        if (prio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
#endif

/*$PAGE*/
    OS_ENTER_CRITICAL();
    if (prio == OS_PRIO_SELF) {                         /* See if requesting to delete self            */
        prio = OSTCBCur->OSTCBPrio;                     /* Set priority to delete to current           */
    }
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                          /* Task to delete must exist                   */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }
    if (ptcb == OS_TCB_RESERVED) {                      /* Must not be assigned to Mutex               */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_DEL);
    }

    OSRdyTbl[ptcb->OSTCBY] &= (OS_PRIO)~ptcb->OSTCBBitX;
    if (OSRdyTbl[ptcb->OSTCBY] == 0u) {                 /* Make task not ready                         */
        OSRdyGrp           &= (OS_PRIO)~ptcb->OSTCBBitY;
    }

#if (OS_EVENT_EN)
    if (ptcb->OSTCBEventPtr != (OS_EVENT *)0) {
        OS_EventTaskRemove(ptcb, ptcb->OSTCBEventPtr);  /* Remove this task from any event   wait list */
    }
#if (OS_EVENT_MULTI_EN > 0u)
    if (ptcb->OSTCBEventMultiPtr != (OS_EVENT **)0) {   /* Remove this task from any events' wait lists*/
        OS_EventTaskRemoveMulti(ptcb, ptcb->OSTCBEventMultiPtr);
    }
#endif
#endif

#if (OS_FLAG_EN > 0u) && (OS_MAX_FLAGS > 0u)
    pnode = ptcb->OSTCBFlagNode;
    if (pnode != (OS_FLAG_NODE *)0) {                   /* If task is waiting on event flag            */
        OS_FlagUnlink(pnode);                           /* Remove from wait list                       */
    }
#endif

    ptcb->OSTCBDly      = 0u;                           /* Prevent OSTimeTick() from updating          */
    ptcb->OSTCBStat     = OS_STAT_RDY;                  /* Prevent task from being resumed             */
    ptcb->OSTCBStatPend = OS_STAT_PEND_OK;
    if (OSLockNesting < 255u) {                         /* Make sure we don't context switch           */
        OSLockNesting++;
    }
    OS_EXIT_CRITICAL();                                 /* Enabling INT. ignores next instruc.         */
    OS_Dummy();                                         /* ... Dummy ensures that INTs will be         */
    OS_ENTER_CRITICAL();                                /* ... disabled HERE!                          */
    if (OSLockNesting > 0u) {                           /* Remove context switch lock                  */
        OSLockNesting--;
    }
    OSTaskDelHook(ptcb);                                /* Call user defined hook                      */
    OSTaskCtr--;                                        /* One less task being managed                 */
    OSTCBPrioTbl[prio] = (OS_TCB *)0;                   /* Clear old priority entry                    */
    if (ptcb->OSTCBPrev == (OS_TCB *)0) {               /* Remove from TCB chain                       */
        ptcb->OSTCBNext->OSTCBPrev = (OS_TCB *)0;
        OSTCBList                  = ptcb->OSTCBNext;
    } else {
        ptcb->OSTCBPrev->OSTCBNext = ptcb->OSTCBNext;
        ptcb->OSTCBNext->OSTCBPrev = ptcb->OSTCBPrev;
    }
    ptcb->OSTCBNext     = OSTCBFreeList;                /* Return TCB to free TCB list                 */
    OSTCBFreeList       = ptcb;
#if OS_TASK_NAME_EN > 0u
    ptcb->OSTCBTaskName = (INT8U *)(void *)"?";
#endif
    OS_EXIT_CRITICAL();
    if (OSRunning == OS_TRUE) {
        OS_Sched();                                     /* Find new highest priority task              */
    }
    return (OS_ERR_NONE);
}
#endif
```

---

## 3.2.6 请求删除任务 - OSTaskDelReq() 函数

系统资源可能因为被某个被删除任务持有而丢失

该函数可以使任务在使用完资源后，先释放，再被删除

具体实现：

* 确保要删除的优先级有效
* 若任务存在，则在 TCB 中设置标志

```c
/*
*********************************************************************************************************
*                                    REQUEST THAT A TASK DELETE ITSELF
*
* Description: This function is used to:
*                   a) notify a task to delete itself.
*                   b) to see if a task requested that the current task delete itself.
*              This function is a little tricky to understand.  Basically, you have a task that needs
*              to be deleted however, this task has resources that it has allocated (memory buffers,
*              semaphores, mailboxes, queues etc.).  The task cannot be deleted otherwise these
*              resources would not be freed.  The requesting task calls OSTaskDelReq() to indicate that
*              the task needs to be deleted.  Deleting of the task is however, deferred to the task to
*              be deleted.  For example, suppose that task #10 needs to be deleted.  The requesting task
*              example, task #5, would call OSTaskDelReq(10).  When task #10 gets to execute, it calls
*              this function by specifying OS_PRIO_SELF and monitors the returned value.  If the return
*              value is OS_ERR_TASK_DEL_REQ, another task requested a task delete.  Task #10 would look like
*              this:
*
*                   void Task(void *p_arg)
*                   {
*                       .
*                       .
*                       while (1) {
*                           OSTimeDly(1);
*                           if (OSTaskDelReq(OS_PRIO_SELF) == OS_ERR_TASK_DEL_REQ) {
*                               Release any owned resources;
*                               De-allocate any dynamic memory;
*                               OSTaskDel(OS_PRIO_SELF);
*                           }
*                       }
*                   }
*
* Arguments  : prio    is the priority of the task to request the delete from
*
* Returns    : OS_ERR_NONE            if the task exist and the request has been registered
*              OS_ERR_TASK_NOT_EXIST  if the task has been deleted.  This allows the caller to know whether
*                                     the request has been executed.
*              OS_ERR_TASK_DEL        if the task is assigned to a Mutex.
*              OS_ERR_TASK_DEL_IDLE   if you requested to delete uC/OS-II's idle task
*              OS_ERR_PRIO_INVALID    if the priority you specify is higher that the maximum allowed
*                                     (i.e. >= OS_LOWEST_PRIO) or, you have not specified OS_PRIO_SELF.
*              OS_ERR_TASK_DEL_REQ    if a task (possibly another task) requested that the running task be
*                                     deleted.
*********************************************************************************************************
*/
/*$PAGE*/
#if OS_TASK_DEL_EN > 0u
INT8U  OSTaskDelReq (INT8U prio)
{
    INT8U      stat;
    OS_TCB    *ptcb;
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    if (prio == OS_TASK_IDLE_PRIO) {                            /* Not allowed to delete idle task     */
        return (OS_ERR_TASK_DEL_IDLE);
    }
#if OS_ARG_CHK_EN > 0u
    if (prio >= OS_LOWEST_PRIO) {                               /* Task priority valid ?               */
        if (prio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
#endif
    if (prio == OS_PRIO_SELF) {                                 /* See if a task is requesting to ...  */
        OS_ENTER_CRITICAL();                                    /* ... this task to delete itself      */
        stat = OSTCBCur->OSTCBDelReq;                           /* Return request status to caller     */
        OS_EXIT_CRITICAL();
        return (stat);
    }
    OS_ENTER_CRITICAL();
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                                  /* Task to delete must exist           */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);                         /* Task must already be deleted        */
    }
    if (ptcb == OS_TCB_RESERVED) {                              /* Must NOT be assigned to a Mutex     */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_DEL);
    }
    ptcb->OSTCBDelReq = OS_ERR_TASK_DEL_REQ;                    /* Set flag indicating task to be DEL. */
    OS_EXIT_CRITICAL();
    return (OS_ERR_NONE);
}
#endif
```

---

## 3.2.7 堆栈检验 - OSTaskStkChk() 函数

检查任务所需任务栈的空间大小

---

## 3.2.8 任务挂起 - OSTaskSuspend() 函数

无条件挂起一个任务

* 必须和任务恢复函数 `OSTaskResume()` 成对使用
* 任务可以挂起自己，或其它任务
* 本任务被挂起后，只能由其它任务恢复

原理：

* 检查任务是否就绪，如果就绪，则移出就绪表
* 在 TCB 中设置标志，表示任务被挂起
* 重新调度

```c
/*
*********************************************************************************************************
*                                            SUSPEND A TASK
*
* Description: This function is called to suspend a task.  The task can be the calling task if the
*              priority passed to OSTaskSuspend() is the priority of the calling task or OS_PRIO_SELF.
*
* Arguments  : prio     is the priority of the task to suspend.  If you specify OS_PRIO_SELF, the
*                       calling task will suspend itself and rescheduling will occur.
*
* Returns    : OS_ERR_NONE               if the requested task is suspended
*              OS_ERR_TASK_SUSPEND_IDLE  if you attempted to suspend the idle task which is not allowed.
*              OS_ERR_PRIO_INVALID       if the priority you specify is higher that the maximum allowed
*                                        (i.e. >= OS_LOWEST_PRIO) or, you have not specified OS_PRIO_SELF.
*              OS_ERR_TASK_SUSPEND_PRIO  if the task to suspend does not exist
*              OS_ERR_TASK_NOT_EXITS     if the task is assigned to a Mutex PIP
*
* Note       : You should use this function with great care.  If you suspend a task that is waiting for
*              an event (i.e. a message, a semaphore, a queue ...) you will prevent this task from
*              running when the event arrives.
*********************************************************************************************************
*/

#if OS_TASK_SUSPEND_EN > 0u
INT8U  OSTaskSuspend (INT8U prio)
{
    BOOLEAN    self;
    OS_TCB    *ptcb;
    INT8U      y;
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#if OS_ARG_CHK_EN > 0u
    if (prio == OS_TASK_IDLE_PRIO) {                            /* Not allowed to suspend idle task    */
        return (OS_ERR_TASK_SUSPEND_IDLE);
    }
    if (prio >= OS_LOWEST_PRIO) {                               /* Task priority valid ?               */
        if (prio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
#endif
    OS_ENTER_CRITICAL();
    if (prio == OS_PRIO_SELF) {                                 /* See if suspend SELF                 */
        prio = OSTCBCur->OSTCBPrio;
        self = OS_TRUE;
    } else if (prio == OSTCBCur->OSTCBPrio) {                   /* See if suspending self              */
        self = OS_TRUE;
    } else {
        self = OS_FALSE;                                        /* No suspending another task          */
    }
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                                  /* Task to suspend must exist          */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_SUSPEND_PRIO);
    }
    if (ptcb == OS_TCB_RESERVED) {                              /* See if assigned to Mutex            */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }
    y            = ptcb->OSTCBY;
    OSRdyTbl[y] &= (OS_PRIO)~ptcb->OSTCBBitX;                   /* Make task not ready                 */
    if (OSRdyTbl[y] == 0u) {
        OSRdyGrp &= (OS_PRIO)~ptcb->OSTCBBitY;
    }
    ptcb->OSTCBStat |= OS_STAT_SUSPEND;                         /* Status of task is 'SUSPENDED'       */
    OS_EXIT_CRITICAL();
    if (self == OS_TRUE) {                                      /* Context switch only if SELF         */
        OS_Sched();                                             /* Find new highest priority task      */
    }
    return (OS_ERR_NONE);
}
#endif
```

---

## 3.2.9 任务恢复 - OSTaskResume() 函数

唯一能唤醒被挂起任务的函数

原理：

* 清除 TCB 中的挂起标志
* 若任务就绪，则将任务加入就绪表
* 重新调度

```c
/*
*********************************************************************************************************
*                                        RESUME A SUSPENDED TASK
*
* Description: This function is called to resume a previously suspended task.  This is the only call that
*              will remove an explicit task suspension.
*
* Arguments  : prio     is the priority of the task to resume.
*
* Returns    : OS_ERR_NONE                if the requested task is resumed
*              OS_ERR_PRIO_INVALID        if the priority you specify is higher that the maximum allowed
*                                         (i.e. >= OS_LOWEST_PRIO)
*              OS_ERR_TASK_RESUME_PRIO    if the task to resume does not exist
*              OS_ERR_TASK_NOT_EXIST      if the task is assigned to a Mutex PIP
*              OS_ERR_TASK_NOT_SUSPENDED  if the task to resume has not been suspended
*********************************************************************************************************
*/

#if OS_TASK_SUSPEND_EN > 0u
INT8U  OSTaskResume (INT8U prio)
{
    OS_TCB    *ptcb;
#if OS_CRITICAL_METHOD == 3u                                  /* Storage for CPU status register       */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#if OS_ARG_CHK_EN > 0u
    if (prio >= OS_LOWEST_PRIO) {                             /* Make sure task priority is valid      */
        return (OS_ERR_PRIO_INVALID);
    }
#endif
    OS_ENTER_CRITICAL();
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                                /* Task to suspend must exist            */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_RESUME_PRIO);
    }
    if (ptcb == OS_TCB_RESERVED) {                            /* See if assigned to Mutex              */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }
    if ((ptcb->OSTCBStat & OS_STAT_SUSPEND) != OS_STAT_RDY) { /* Task must be suspended                */
        ptcb->OSTCBStat &= (INT8U)~(INT8U)OS_STAT_SUSPEND;    /* Remove suspension                     */
        if (ptcb->OSTCBStat == OS_STAT_RDY) {                 /* See if task is now ready              */
            if (ptcb->OSTCBDly == 0u) {
                OSRdyGrp               |= ptcb->OSTCBBitY;    /* Yes, Make task ready to run           */
                OSRdyTbl[ptcb->OSTCBY] |= ptcb->OSTCBBitX;
                OS_EXIT_CRITICAL();
                if (OSRunning == OS_TRUE) {
                    OS_Sched();                               /* Find new highest priority task        */
                }
            } else {
                OS_EXIT_CRITICAL();
            }
        } else {                                              /* Must be pending on event              */
            OS_EXIT_CRITICAL();
        }
        return (OS_ERR_NONE);
    }
    OS_EXIT_CRITICAL();
    return (OS_ERR_TASK_NOT_SUSPENDED);
}
#endif
```

---

## 3.2.10 任务信息的获取 - OSTaskQuery() 函数

获取自身或其它任务的信息

其实是返回任务 TCB 的一个拷贝

所以任务需要分配一个 `OS_TCB` 类型的空间

```c
/*
*********************************************************************************************************
*                                            QUERY A TASK
*
* Description: This function is called to obtain a copy of the desired task's TCB.
*
* Arguments  : prio         is the priority of the task to obtain information from.
*
*              p_task_data  is a pointer to where the desired task's OS_TCB will be stored.
*
* Returns    : OS_ERR_NONE            if the requested task is suspended
*              OS_ERR_PRIO_INVALID    if the priority you specify is higher that the maximum allowed
*                                     (i.e. > OS_LOWEST_PRIO) or, you have not specified OS_PRIO_SELF.
*              OS_ERR_PRIO            if the desired task has not been created
*              OS_ERR_TASK_NOT_EXIST  if the task is assigned to a Mutex PIP
*              OS_ERR_PDATA_NULL      if 'p_task_data' is a NULL pointer
*********************************************************************************************************
*/

#if OS_TASK_QUERY_EN > 0u
INT8U  OSTaskQuery (INT8U    prio,
                    OS_TCB  *p_task_data)
{
    OS_TCB    *ptcb;
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



#if OS_ARG_CHK_EN > 0u
    if (prio > OS_LOWEST_PRIO) {                 /* Task priority valid ?                              */
        if (prio != OS_PRIO_SELF) {
            return (OS_ERR_PRIO_INVALID);
        }
    }
    if (p_task_data == (OS_TCB *)0) {            /* Validate 'p_task_data'                             */
        return (OS_ERR_PDATA_NULL);
    }
#endif
    OS_ENTER_CRITICAL();
    if (prio == OS_PRIO_SELF) {                  /* See if suspend SELF                                */
        prio = OSTCBCur->OSTCBPrio;
    }
    ptcb = OSTCBPrioTbl[prio];
    if (ptcb == (OS_TCB *)0) {                   /* Task to query must exist                           */
        OS_EXIT_CRITICAL();
        return (OS_ERR_PRIO);
    }
    if (ptcb == OS_TCB_RESERVED) {               /* Task to query must not be assigned to a Mutex      */
        OS_EXIT_CRITICAL();
        return (OS_ERR_TASK_NOT_EXIST);
    }
                                                 /* Copy TCB into user storage area                    */
    OS_MemCopy((INT8U *)p_task_data, (INT8U *)ptcb, sizeof(OS_TCB));
    OS_EXIT_CRITICAL();
    return (OS_ERR_NONE);
}
#endif
```

---

## Summary

从总体上来说

μC/OS-II 的逻辑比 Linux 简单多了

主要的差别在于没有那些复杂的内存管理机制

也没有复杂的调度算法

TCB 也比 Linux 简单很多

优美源自于简单吧 🥱

---

