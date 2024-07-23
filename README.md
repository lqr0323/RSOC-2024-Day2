# RSOC-2024-Day2
---
## 总结  
通过第二天的学习，学会了多线程的相关知识，嵌入式开发的相关知识等。第二次发Github，内容涵盖不是很全，见谅！
## 多线程的相关知识
---
  内核是一个操作系统的核心，是操作系统最基础也是最重要的部分。它负责管理系统的线程、线程间通信、系统时钟、中断及内存等。下图为 RT-Thread 内核架构图，可以看到内核处于硬件层之上，内核部分包括内核库、实时内核实现。
### RT-Thread启动流程
---
  一般了解一份代码大多从启动部分开始，同样这里也采用这种方式，先寻找启动的源头。  
  RT-Thread 支持多种平台和多种编译器，而 rtthread_startup() 函数是 RT-Thread 规定的统一启动入口。  
  一般执行顺序是：系统先从启动文件开始运行，然后进入 RT-Thread 的启动函数 rtthread_startup() ，最后进入用户入口函数 main()。  
  这部分启动代码，大致可以分为四个部分：

（1）初始化与系统相关的硬件；

（2）初始化系统内核对象，例如定时器、调度器、信号；

（3）创建 main 线程，在 main 线程中对各类模块依次进行初始化；

（4）初始化定时器线程、空闲线程，并启动调度器。

启动调度器之前，系统所创建的线程在执行 rt_thread_startup() 后并不会立马运行，它们会处于就绪状态等待系统调度；待启动调度器之后，系统才转入第一个线程开始运行，根据调度规则，选择的是就绪队列中优先级最高的线程。

rt_hw_board_init() 中完成系统时钟设置，为系统提供心跳、串口初始化，将系统输入输出终端绑定到这个串口，后续系统运行信息就会从串口打印出来。

main() 函数是 RT-Thread 的用户代码入口，用户可以在 main() 函数里添加自己的应用。

### 线程管理
RT-Thread 线程管理的主要功能是对线程进行管理和调度，系统中总共存在两类线程，分别是系统线程和用户线程，系统线程是由 RT-Thread 内核创建的线程，用户线程是由应用程序创建的线程，这两类线程都会从内核对象容器中分配线程对象，当线程被删除时，也会被从对象容器中删除。      
### 时钟管理  
  
时间是非常重要的概念，和朋友出去游玩需要约定时间，完成任务也需要花费时间，生活离不开时间。操作系统也一样，需要通过时间来规范其任务的执行，操作系统中最小的时间单位是时钟节拍 (OS Tick)。本章主要介绍时钟节拍和基于时钟节拍的定时器，读完本章，我们将了解时钟节拍如何产生，并学会如何使用 RT-Thread 的定时器。  

### 时钟节拍  
  
任何操作系统都需要提供一个时钟节拍，以供系统处理所有和时间有关的事件，如线程的延时、线程的时间片轮转调度以及定时器超时等。时钟节拍是特定的周期性中断，这个中断可以看做是系统心跳，中断之间的时间间隔取决于不同的应用，一般是 1ms–100ms，时钟节拍率越快，系统的实时响应越快，但是系统的额外开销就越大，从系统启动开始计数的时钟节拍数称为系统时间。

RT-Thread 中，时钟节拍的长度可以根据 RT_TICK_PER_SECOND 的定义来调整，等于 1/RT_TICK_PER_SECOND 秒。  
定时器，是指从指定的时刻开始，经过一定的指定时间后触发一个事件，例如定个时间提醒第二天能够按时起床。定时器有硬件定时器和软件定时器之分：

1）硬件定时器是芯片本身提供的定时功能。一般是由外部晶振提供给芯片输入时钟，芯片向软件模块提供一组配置寄存器，接受控制输入，到达设定时间值后芯片中断控制器产生时钟中断。硬件定时器的精度一般很高，可以达到纳秒级别，并且是中断触发方式。

2）软件定时器是由操作系统提供的一类系统接口，它构建在硬件定时器基础之上，使系统能够提供不受数目限制的定时器服务。

RT-Thread 操作系统提供软件实现的定时器，以时钟节拍（OS Tick）的时间长度为单位，即定时数值必须是 OS Tick 的整数倍，例如一个 OS Tick 是 10ms，那么上层软件定时器只能是 10ms，20ms，100ms 等，而不能定时为 15ms。RT-Thread 的定时器也基于系统的节拍，提供了基于节拍整数倍的定时能力。
HARD_TIMER 模式
HARD_TIMER 模式的定时器超时函数在中断上下文环境中执行，可以在初始化 / 创建定时器时使用参数 RT_TIMER_FLAG_HARD_TIMER 来指定。

在中断上下文环境中执行时，对于超时函数的要求与中断服务例程的要求相同：执行时间应该尽量短，执行时不应导致当前上下文挂起、等待。例如在中断上下文中执行的超时函数它不应该试图去申请动态内存、释放动态内存等。

RT-Thread 定时器默认的方式是 HARD_TIMER 模式，即定时器超时后，超时函数是在系统时钟中断的上下文环境中运行的。在中断上下文中的执行方式决定了定时器的超时函数不应该调用任何会让当前上下文挂起的系统函数；也不能够执行非常长的时间，否则会导致其他中断的响应时间加长或抢占了其他线程执行的时间。

SOFT_TIMER 模式
SOFT_TIMER 模式可配置，通过宏定义 RT_USING_TIMER_SOFT 来决定是否启用该模式。该模式被启用后，系统会在初始化时创建一个 timer 线程，然后 SOFT_TIMER 模式的定时器超时函数在都会在 timer 线程的上下文环境中执行。可以在初始化 / 创建定时器时使用参数 RT_TIMER_FLAG_SOFT_TIMER 来指定设置 SOFT_TIMER 模式。  
定时器工作机制
下面以一个例子来说明 RT-Thread 定时器的工作机制。在 RT-Thread 定时器模块中维护着两个重要的全局变量：

（1）当前系统经过的 tick 时间 rt_tick（当硬件定时器中断来临时，它将加 1）；

（2）定时器链表 rt_timer_list。系统新创建并激活的定时器都会按照以超时时间排序的方式插入到 rt_timer_list 链表中。

如下图所示，系统当前 tick 值为 20，在当前系统中已经创建并启动了三个定时器，分别是定时时间为 50 个 tick 的 Timer1、100 个 tick 的 Timer2 和 500 个 tick 的 Timer3，这三个定时器分别加上系统当前时间 rt_tick=20，从小到大排序链接在 rt_timer_list 链表中，形成如图所示的定时器链表结构。
### 测试代码
 
#include <board.h>
#include <rtthread.h>
#include <drv_gpio.h>
#ifndef RT_USING_NANO
#include <rtdevice.h>
#include <rtdbg.h>
#endif /* RT_USING_NANO */

rt_thread_t th1 = RT_NULL, th2 = RT_NULL, th3 = RT_NULL;

void th1_entry(void *parameter)
{
    int count = 20; // 只打印20次
    while (count--)
    {
        rt_kprintf("Thread 1 is running, count: %d\n", 20 - count);
        rt_thread_mdelay(1000); // 延迟1000毫秒
    }
    rt_kprintf("Thread 1 finished.\n");
}

void th2_entry(void *parameter)
{
    int count = 30; // 只打印30次
    while (count--)
    {
        rt_kprintf("Thread 2 is running, count: %d\n", 30 - count);
        rt_thread_mdelay(2000); // 延迟2000毫秒
    }
    rt_kprintf("Thread 2 finished.\n");
}

void th3_entry(void *parameter)
{
    int count = 40; // 只打印40次
    while (count--)
    {
        rt_kprintf("Thread 3 is running, count: %d\n", 40 - count);
        rt_thread_mdelay(3000); // 延迟3000毫秒
    }
    rt_kprintf("Thread 3 finished.\n");
}

int main(void)
{
    // 创建线程 th1，优先级最高
    th1 = rt_thread_create("th1", th1_entry, RT_NULL, 1024, 10, 10);
    if (th1 == RT_NULL)
    {
        LOG_E("Thread 1 create failed...\n");
    }
    else
    {
        rt_thread_startup(th1);
        LOG_D("Thread 1 create success!\n");
    }

    // 创建线程 th2，优先级第二高
    th2 = rt_thread_create("th2", th2_entry, RT_NULL, 1024, 20, 10);
    if (th2 == RT_NULL)
    {
        LOG_E("Thread 2 create failed...\n");
    }
    else
    {
        rt_thread_startup(th2);
        LOG_D("Thread 2 create success!\n");
    }

    // 创建线程 th3，优先级最低
    th3 = rt_thread_create("th3", th3_entry, RT_NULL, 1024, 30, 10);
    if (th3 == RT_NULL)
    {
        LOG_E("Thread 3 create failed...\n");
    }
    else
    {
        rt_thread_startup(th3);
        LOG_D("Thread 3 create success!\n");
    }

    return 0;
}  
`
