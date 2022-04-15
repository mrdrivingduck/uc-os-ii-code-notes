# Chapter 3.1 - 核心任务管理

Created by : Mr Dk.

2019 / 11 / 18 22:14

Nanjing, Jiangsu, China

---

## 3.1.1 临界区的处理

关中断的时间是实时内核的重要指标，影响系统的实时响应特性。在 `os_cpu.h` 中定义了两个宏来进行开关中断：

- `OS_ENTER_CRITICAL()`
- `OS_EXIT_CRITICAL()`

这两个宏一般有三种实现方法：

- `OS_CRITICAL_METHOD = 1`：直接用 CPU 指令开关中断
- `OS_CRITICAL_METHOD = 2`：先利用堆栈保存状态，再关中断；将中断状态从堆栈中弹出，恢复中断原始状态
- `OS_CRITICAL_METHOD = 3`：利用局部变量保存中断开关状态

## 3.1.2 任务的形式

一个 C 语言函数

```c
void MyTask(void *ppdata) reentrant {

}
```

任务结构只有两种：

- 无限循环结构
- 只执行一次就被删除的结构；删除后，代码依然驻留在 RAM 中，但是 OS 不再管理这段代码

任务永不返回。

## 3.1.3 任务的状态

- 休眠
- 就绪
- 挂起
- 被中断
- 运行

## 3.1.4 任务控制块

Task Control Blocks (TCB)，保存任务的各种状态信息，全部驻留在 RAM 中：

```c
typedef struct os_tcb {
    OS_STK          *OSTCBStkPtr;           /* Pointer to current top of stack                         */

#if OS_TASK_CREATE_EXT_EN > 0u
    void            *OSTCBExtPtr;           /* Pointer to user definable data for TCB extension        */
    OS_STK          *OSTCBStkBottom;        /* Pointer to bottom of stack                              */
    INT32U           OSTCBStkSize;          /* Size of task stack (in number of stack elements)        */
    INT16U           OSTCBOpt;              /* Task options as passed by OSTaskCreateExt()             */
    INT16U           OSTCBId;               /* Task ID (0..65535)                                      */
#endif

    struct os_tcb   *OSTCBNext;             /* Pointer to next     TCB in the TCB list                 */
    struct os_tcb   *OSTCBPrev;             /* Pointer to previous TCB in the TCB list                 */

#if (OS_EVENT_EN)
    OS_EVENT        *OSTCBEventPtr;         /* Pointer to          event control block                 */
#endif

#if (OS_EVENT_EN) && (OS_EVENT_MULTI_EN > 0u)
    OS_EVENT       **OSTCBEventMultiPtr;    /* Pointer to multiple event control blocks                */
#endif

#if ((OS_Q_EN > 0u) && (OS_MAX_QS > 0u)) || (OS_MBOX_EN > 0u)
    void            *OSTCBMsg;              /* Message received from OSMboxPost() or OSQPost()         */
#endif

#if (OS_FLAG_EN > 0u) && (OS_MAX_FLAGS > 0u)
#if OS_TASK_DEL_EN > 0u
    OS_FLAG_NODE    *OSTCBFlagNode;         /* Pointer to event flag node                              */
#endif
    OS_FLAGS         OSTCBFlagsRdy;         /* Event flags that made task ready to run                 */
#endif

    INT32U           OSTCBDly;              /* Nbr ticks to delay task or, timeout waiting for event   */
    INT8U            OSTCBStat;             /* Task      status                                        */
    INT8U            OSTCBStatPend;         /* Task PEND status                                        */
    INT8U            OSTCBPrio;             /* Task priority (0 == highest)                            */

    INT8U            OSTCBX;                /* Bit position in group  corresponding to task priority   */
    INT8U            OSTCBY;                /* Index into ready table corresponding to task priority   */
    OS_PRIO          OSTCBBitX;             /* Bit mask to access bit position in ready table          */
    OS_PRIO          OSTCBBitY;             /* Bit mask to access bit position in ready group          */

#if OS_TASK_DEL_EN > 0u
    INT8U            OSTCBDelReq;           /* Indicates whether a task needs to delete itself         */
#endif

#if OS_TASK_PROFILE_EN > 0u
    INT32U           OSTCBCtxSwCtr;         /* Number of time the task was switched in                 */
    INT32U           OSTCBCyclesTot;        /* Total number of clock cycles the task has been running  */
    INT32U           OSTCBCyclesStart;      /* Snapshot of cycle counter at start of task resumption   */
    OS_STK          *OSTCBStkBase;          /* Pointer to the beginning of the task stack              */
    INT32U           OSTCBStkUsed;          /* Number of bytes used from the stack                     */
#endif

#if OS_TASK_NAME_EN > 0u
    INT8U           *OSTCBTaskName;
#endif

#if OS_TASK_REG_TBL_SIZE > 0u
    INT32U           OSTCBRegTbl[OS_TASK_REG_TBL_SIZE];
#endif
} OS_TCB;
```

## 3.1.5 就绪表

- `OSRdyTbl[OS_LOWEST_PRIO/8+1]`：数组长度为 8，8 \* 8bit = 64bit；每一位分别代表一个任务是否就绪
- `OSRdyGrp` 是一个变量 (8-bit)：每一位代表了一组 8 个任务中是否就绪，该组中只要有一个任务就绪，这一 bit 就要被置位
- 优先级也是一个变量 (8-bit)：
  - 5-3bit 为在 `OSRdyTbl` 中的索引，也对应在 `OSRdyGrp` 中的某一 bit；
  - 2-0bit 对应 `OSRdyTbl` 中某索引下的某一位

某任务进入就绪态，就要将 `OSRdyTbl` 中的对应 bit 置位：

```c
OSRdyGrp = OSMapTbl[prio >> 3];
OSRdyTbl[prio >> 3] |= OSMapTbl[prio & 0x07];
```

如果任务需要脱离就绪态，就将对应 bit 清零：

```c
if ((OSRdyTbl[prio >> 3] &= ~OSMapTbl[prio & 0x07]) == 0)
    OSRdyGrp &= ~OSMapTbl[prio >> 3];
```

其中，`OSMapTbl[]` 存放在 ROM 中，是提前计算好的，用于加快计算速度：

```c
OSMapTbl[] = {
    00000001b,
    00000010b,
    00000100b,
    00001000b,
    00010000b,
    00100000b,
    01000000b,
    10000000b
};
```

如何找出进入就绪态的最高优先级代码？

```c
INT8U  const  OSUnMapTbl[256] = {
    0u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x00 to 0x0F                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x10 to 0x1F                   */
    5u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x20 to 0x2F                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x30 to 0x3F                   */
    6u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x40 to 0x4F                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x50 to 0x5F                   */
    5u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x60 to 0x6F                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x70 to 0x7F                   */
    7u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x80 to 0x8F                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0x90 to 0x9F                   */
    5u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0xA0 to 0xAF                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0xB0 to 0xBF                   */
    6u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0xC0 to 0xCF                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0xD0 to 0xDF                   */
    5u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, /* 0xE0 to 0xEF                   */
    4u, 0u, 1u, 0u, 2u, 0u, 1u, 0u, 3u, 0u, 1u, 0u, 2u, 0u, 1u, 0u  /* 0xF0 to 0xFF                   */
};
```

```c
y = OSUnMapTbl[OSRdyGrp];
x = OSUnMapTbl[OSRdyTbl[y]];
prio = (y << 3) + x;
```

## 3.1.6 任务调度

1. 确定进入就绪态的任务中哪个优先级最高
2. 任务切换

任务调度所花的时间是常数，与所建任务数无关。调度过程属于临界区，需要关中断，防止被中断打断：

```c
void  OS_Sched (void)
{
    /* 为关中断预留用于保存状态的局部变量 8?
#if OS_CRITICAL_METHOD == 3u                           /* Allocate storage for CPU status register     */
    OS_CPU_SR  cpu_sr = 0u;
#endif


    /* 关中断，进入临界区 */
    OS_ENTER_CRITICAL();
    if (OSIntNesting == 0u) {                          /* Schedule only if all ISRs done and ...       */
        /* 只有所有嵌套中断都退出，才允许调度 */
        if (OSLockNesting == 0u) {                     /* ... scheduler is not locked                  */
            /* 调度器没有被锁定 */
            OS_SchedNew();
            OSTCBHighRdy = OSTCBPrioTbl[OSPrioHighRdy]; /* 找到最高优先级的任务 */
            if (OSPrioHighRdy != OSPrioCur) {          /* No Ctx Sw if current task is highest rdy     */
                /* 找到最高优先级任务不是当前任务 */
#if OS_TASK_PROFILE_EN > 0u
                OSTCBHighRdy->OSTCBCtxSwCtr++;         /* Inc. # of context switches to this task      */
#endif
                /* 任务切换计数器累加 */
                OSCtxSwCtr++;                          /* Increment context switch counter             */
                /* 任务切换 */
                OS_TASK_SW();                          /* Perform a context switch                     */
            }
        }
    }
    /* 开中断，退出临界区 */
    OS_EXIT_CRITICAL();
}
```

## 3.1.8 调度器上锁和解锁

禁止任务调度，使任务保持对 CPU 的控制权。实现原理：对全局变量锁定嵌套计数器 `OSLockNesting` 进行操作。该变量记录了被锁定的次数，允许嵌套深度 255 层。所谓加锁和解锁就是对该变量 +1 或 -1。

```c
/*
*********************************************************************************************************
*                                          PREVENT SCHEDULING
*
* Description: This function is used to prevent rescheduling to take place.  This allows your application
*              to prevent context switches until you are ready to permit context switching.
*
* Arguments  : none
*
* Returns    : none
*
* Notes      : 1) You MUST invoke OSSchedLock() and OSSchedUnlock() in pair.  In other words, for every
*                 call to OSSchedLock() you MUST have a call to OSSchedUnlock().
*********************************************************************************************************
*/

#if OS_SCHED_LOCK_EN > 0u
void  OSSchedLock (void)
{
#if OS_CRITICAL_METHOD == 3u                     /* Allocate storage for CPU status register           */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    if (OSRunning == OS_TRUE) {                  /* Make sure multitasking is running                  */
        /* 多任务已经运行 */
        OS_ENTER_CRITICAL(); /* 临界区 */
        if (OSIntNesting == 0u) {                /* Can't call from an ISR                             */
            /* 不允许在中断上下文中调用 */
            if (OSLockNesting < 255u) {          /* Prevent OSLockNesting from wrapping back to 0      */
                OSLockNesting++;                 /* Increment lock nesting level                       */
            }
        }
        OS_EXIT_CRITICAL(); /* 临界区 */
    }
}
#endif
```

```c
/*
*********************************************************************************************************
*                                          ENABLE SCHEDULING
*
* Description: This function is used to re-allow rescheduling.
*
* Arguments  : none
*
* Returns    : none
*
* Notes      : 1) You MUST invoke OSSchedLock() and OSSchedUnlock() in pair.  In other words, for every
*                 call to OSSchedLock() you MUST have a call to OSSchedUnlock().
*********************************************************************************************************
*/

#if OS_SCHED_LOCK_EN > 0u
void  OSSchedUnlock (void)
{
#if OS_CRITICAL_METHOD == 3u                               /* Allocate storage for CPU status register */
    OS_CPU_SR  cpu_sr = 0u;
#endif



    if (OSRunning == OS_TRUE) {                            /* Make sure multitasking is running        */
        OS_ENTER_CRITICAL();
        if (OSLockNesting > 0u) {                          /* Do not decrement if already 0            */
            OSLockNesting--;                               /* Decrement lock nesting level             */
            if (OSLockNesting == 0u) {                     /* See if scheduler is enabled and ...      */
                if (OSIntNesting == 0u) {                  /* ... not in an ISR                        */
                    OS_EXIT_CRITICAL();
                    OS_Sched();                            /* See if a HPT is ready                    */
                } else {
                    OS_EXIT_CRITICAL();
                }
            } else {
                OS_EXIT_CRITICAL();
            }
        } else {
            OS_EXIT_CRITICAL();
        }
    }
}
#endif
```

当调度锁定计数器减为 0 时，还要重新调度，前提是调用者不是中断服务子程序。

## 3.1.9 空闲任务

没有其它任务进入就绪态时，该任务转入运行态。任务优先级永远为最低：`OS_LOWEST_PRIO`，永远不被挂起或删除。不停给一个 32-bit 计数器 +1：由于 `+1` 操作需要多条指令，因此需要开关中断。

## 3.1.10 统计任务

计算当前 CPU 利用率。

## 3.1.11 μC/OS-II 的初始化

调系统初始化函数 `OSInit()` 初始化所有变量和数据结构：

- 建立空闲任务
- 建立统计任务
- 初始化空闲数据结构链表
- 初始化所有的变量和数据结构

## 3.1.12 μC/OS-II 的启动

调用系统启动函数 `OSStart()` 实现。在启动前，应用程序需要至少建立一个任务：

```c
void main(void) {
    OSInit();

    /* OSTaskCreate() / OSTaskCreateExt() */

    OSStart();
}
```

```c
/*
*********************************************************************************************************
*                                          START MULTITASKING
*
* Description: This function is used to start the multitasking process which lets uC/OS-II manages the
*              task that you have created.  Before you can call OSStart(), you MUST have called OSInit()
*              and you MUST have created at least one task.
*
* Arguments  : none
*
* Returns    : none
*
* Note       : OSStartHighRdy() MUST:
*                 a) Call OSTaskSwHook() then,
*                 b) Set OSRunning to OS_TRUE.
*                 c) Load the context of the task pointed to by OSTCBHighRdy.
*                 d_ Execute the task.
*********************************************************************************************************
*/

void  OSStart (void)
{
    if (OSRunning == OS_FALSE) {
        OS_SchedNew();                               /* Find highest priority's task priority number   */
        OSPrioCur     = OSPrioHighRdy;
        OSTCBHighRdy  = OSTCBPrioTbl[OSPrioHighRdy]; /* Point to highest priority task ready to run    */
        OSTCBCur      = OSTCBHighRdy;
        OSStartHighRdy();                            /* Execute target specific code to start task     */
    }
}
```

首先找到优先级最高的就绪任务：

```c
/*
*********************************************************************************************************
*                              FIND HIGHEST PRIORITY TASK READY TO RUN
*
* Description: This function is called by other uC/OS-II services to determine the highest priority task
*              that is ready to run.  The global variable 'OSPrioHighRdy' is changed accordingly.
*
* Arguments  : none
*
* Returns    : none
*
* Notes      : 1) This function is INTERNAL to uC/OS-II and your application should not call it.
*              2) Interrupts are assumed to be disabled when this function is called.
*********************************************************************************************************
*/

static  void  OS_SchedNew (void)
{
#if OS_LOWEST_PRIO <= 63u                        /* See if we support up to 64 tasks                   */
    INT8U   y;


    y             = OSUnMapTbl[OSRdyGrp];
    OSPrioHighRdy = (INT8U)((y << 3u) + OSUnMapTbl[OSRdyTbl[y]]);
#else                                            /* We support up to 256 tasks                         */
    INT8U     y;
    OS_PRIO  *ptbl;


    if ((OSRdyGrp & 0xFFu) != 0u) {
        y = OSUnMapTbl[OSRdyGrp & 0xFFu];
    } else {
        y = OSUnMapTbl[(OS_PRIO)(OSRdyGrp >> 8u) & 0xFFu] + 8u;
    }
    ptbl = &OSRdyTbl[y];
    if ((*ptbl & 0xFFu) != 0u) {
        OSPrioHighRdy = (INT8U)((y << 4u) + OSUnMapTbl[(*ptbl & 0xFFu)]);
    } else {
        OSPrioHighRdy = (INT8U)((y << 4u) + OSUnMapTbl[(OS_PRIO)(*ptbl >> 8u) & 0xFFu] + 8u);
    }
#endif
}
```

然后调 `OSStartHighRdy()` 运行就绪的最高优先级任务（由汇编实现，移植需要改写）：

- 将任务栈保存的 CPU 现场 pop 进 CPU 中
- 执行一条中断返回指令，将 PC 弹出到 CPU 中继续执行
- 永不返回 `OSStart()`

接下来多任务调度就开始了。
