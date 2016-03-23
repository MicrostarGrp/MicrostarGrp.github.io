---
layout: post
title:  "ARM Cortex-M栈的使用概述"
date:   Auguest 13, 2015
keywords: ARM,Cortex,栈,使用,SVC
categories: embedded notes
comments: true
---

概要
------------------------------------
面向Cortex-M核，通过阅读代码，理解栈的操作。

详细内容
============
预备知识
------------
了解复位后的启动时序，了解uC/OS的使用，汇编指令基础。

栈的简介
------------
以Cortex-M3为例，M3处理器使用递减的栈，栈指针指向最后压栈的对象，当处理器执行压栈操作时，SP指针减小，然后写入要压栈的对象值至新的栈内存，M3处理器有2个栈，MSP/PSP。

启动代码对栈的操作
============
由代码可见，假定分配为[0x20000000 - 0x2000031F]的内存范围，可知复位后MSP的值为0x20000320，事实的CPU栈顶指针。
{% highlight ruby %}
Stack_Size      EQU     0x00000320
                AREA    STACK, NOINIT, READWRITE, ALIGN=3
Stack_Mem   SPACE  Stack_Size
__initial_sp

__Vectors       DCD     __initial_sp     ; Top of Stack
{% endhighlight %}

uC/OS-II对栈的操作
============
OSTaskCreate()创建任务时，传入栈地址（准确地说，末尾32bits的低地址）：
{% highlight ruby %}
__align(8) OS_STK RS485TaskStack[200];
OSTaskCreate(&RS485Task, (void*)0, &RS485TaskStack[199], RS485Task_PRIO);
{% endhighlight %}

那么传入OSTaskCreate()的地址为0x2000031C，该值似乎不是预期的值，实际上它内部调用OSTaskStkInit()，进行了以下操作：
{% highlight ruby %}
/* Registers stacked as if auto-saved on exception   */
*(stk)    = (INT32U)0x01000000L; /* xPSR          */
*(--stk)  = (INT32U)task;        /* Entry Point   */
*(--stk)  = (INT32U)0xFFFFFFFEL; /* R14 (LR) (init value will cause fault if ever used)*/
*(--stk)  = (INT32U)0x12121212L; /* R12           */
*(--stk)  = (INT32U)0x03030303L; /* R3            */
*(--stk)  = (INT32U)0x02020202L; /* R2            */
*(--stk)  = (INT32U)0x01010101L; /* R1            */
*(--stk)  = (INT32U)p_arg;       /* R0 : argument */

/* Remaining registers saved on process stack */
*(--stk)  = (INT32U)0x11111111L;    /* R11 */
*(--stk)  = (INT32U)0x10101010L;    /* R10 */
*(--stk)  = (INT32U)0x09090909L;    /* R9  */
*(--stk)  = (INT32U)0x08080808L;    /* R8  */
*(--stk)  = (INT32U)0x07070707L;    /* R7  */
*(--stk)  = (INT32U)0x06060606L;    /* R6  */
*(--stk)  = (INT32U)0x05050505L;    /* R5  */
*(--stk)  = (INT32U)0x04040404L;    /* R4  */
{% endhighlight %}

可以看出来，它心中的“栈顶”也是0x20000320，它需要的是直接可用的内存，而栈顶是不属于分配的内存范围的。希望大家细细揣摩下其中的区别。

使用SVC时对栈的操作
============
SVC全称[supervisor call]，俗称系统调用，常用来间接调用内核函数或者驱动代码。SVC中断优先级是可配置的，它属于系统中断。指令格式：
{% highlight ruby %}
SVC{cond} #imm	; 'cond'为可选的条件码 'imm'为范围为0-255的整数
{% endhighlight %}

调用它所传递的数值是被处理器忽略的，通过被压栈的PC（即二进制码）取得SVC调用时的参数。
{% highlight ruby %}
TST     LR,#4                   ; 栈判定
MRSNE   R12,PSP                 ; PSP
MOVEQ   R12,SP                  ; MSP
LDR     R12,[R12,#24]           ; 读取PC
LDRH    R12,[R12,#-2]           ; 读取半字
BICS    R12,R12,#0xFF00         ; 获得调用参数
{% endhighlight %}
当然，也可以携带参数，通过R0-R3间接传递，也是提取栈帧的数据；返回参数使用R0。
