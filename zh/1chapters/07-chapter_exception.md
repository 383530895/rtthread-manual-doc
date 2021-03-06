# 异常与中断 #

异常是导致处理器脱离正常运行转向执行特殊代码的任何事件，如果不及时进行处理，轻则系统出错，重则会导致系统毁灭性地瘫痪。所以正确地处理异常，避免错误的发生是提高软件鲁棒性（稳定性）非常重要的一环，对于实时系统更是如此。

异常通常可以分成两类：同步异常和异步异常。同步异常主要是指由于内部事件产生的异常，例如除零错误。异步异常主要是指由于外部异常源产生的异常，例如按下设备某个按钮产生的事件。同步异常与异步异常的区别还在于，同步异常触发后，系统必须立刻进行处理而不能够依然执行原有的程序指令步骤；而异步异常则可以延缓处理甚至是忽略，例如按键中断异常，虽然中断异常触发了，但是系统可以忽略它继续运行（同样也忽略了相应的按键事件）。

中断，通常也叫做外部中断，中断属于异步异常。当中断源产生中断时，处理器也将同样陷入到一个固定位置去执行指令。

## 中断处理过程 ##

中断处理的一般过程如下图所示：
 
中断处理过程

当中断产生时，处理机将按如下的顺序执行：

* 保存当前处理机状态信息
* 载入异常或中断处理函数到PC寄存器
* 把控制权转交给处理函数并开始执行
* 当处理函数执行完成时，恢复处理器状态信息
* 从异常或中断中返回到前一个程序执行点

中断使得CPU可以在事件发生时才予以处理，而不必让CPU连续不断地查询是否有相应的事件发生。通过两条特殊指令：关中断和开中断可以让处理器不响应或响应中断（在关闭中断期间，通常处理器会把新产生的中断挂起，当中断打开时立刻进行响应）。在执行中断服务例程的过程中，如果有更高优先级别的中断源触发中断，由于当前处于中断处理上下文环境中，根据不同的处理器构架可能有不同的处理方式，比如新的中断等待挂起直到当前中断处理离开后再行响应；或新的高优先级中断打断当前中断处理过程，而去直接响应这个更高优先级的新中断源。后面这种情况，称之为中断嵌套。在硬实时环境中，前一种情况是不允许发生的，不能响应中断的时间应尽量的短。而在软件处理（软实时环境）上，RT-Thread允许中断嵌套，即在一个中断服务例程期间，处理器可以响应另外一个优先级更高的中断，过程如下图所示：

当正在执行一个中断服务例程（中断1）时，如果有更高的中断（中断2、中断3）触发，那么操作系统将先保存当前中断服务例程的上下文环境，然后转向中断2的中断服务例程，依此类推，直至中断3。当中断3的中断服务例程运行完成后，系统才恢复中断2的上下文环境，然后转回到中断2的中断服务例程去接着执行，依此类推，直至中断1。

即使如此，对于中断的处理仍然存在着（中断）时间响应的问题，先来看看中断处理过程中的一些特定时间量（图8-3）：
 
中断延迟TB定义为，从中断开始的时刻到中断服务例程开始执行的时刻之间的时间段。而中断服务例程的处理时间TC主要取决于中断服务例程处理的方法，对于不同的系统因其对中断处理的方法不同，其相应服务例程的时间需求也不一样。中断响应时间TD = TB + TC。

## 中断栈 ##

从上节的中断处理过程中，我们看到，在系统响应中断前，软件代码（或处理器）需要把当前任务的上下文保存下来（通常保存在当前任务的任务栈中），再调用中断服务例程进行中断响应、处理。在进行中断处理时（实质是调用用户的中断服务例程函数），中断处理函数中很可能会有自己的局部变量，这些都需要相应的栈空间来保存，所以中断响应依然需要一个栈空间来做为上下文运行中断处理函数。
中断栈可以保存在打断任务的栈中，当从中断中退出时，返回相应的任务继续执行。

中断栈也可以与打断任务栈完全分离开来，即每次进入中断时，在保存完打断任务上下文后，切换到新的中断栈中独立运行。在中断退出时，再做相应的上下文恢复。使用独立中断栈相对来说更容易实现，并且对于任务栈使用情况也比较容易了解掌握（否则必须要为中断栈预留空间，如果系统支持中断嵌套，还需要考虑应该为嵌套中断预留多大的空间）。

RT-Thread采用的方式是提供独立的中断栈，即中断发生时，中断的前期处理程序会将用户的栈指针更换到系统事先留出的中断栈空间中，等中断退出时再恢复用户的栈指针。这样中断就不会占用任务的栈空间，从而提高了内存空间的利用率，且随着任务的增加，这种减少内存占用的的效果也越明显。

## 中断的底半处理 ##

RT-Thread不对中断服务例程所需要的处理时间做任何假设、限制，但如同其它实时操作系统或非实时操作系统一样，用户需要保证所有的中断服务例程在尽可能短的时间内完成（相当于中断服务例程在系统中拥有最高的优先级，会抢占所有线程优先执行）。这样在发生中断嵌套，或屏蔽了相应中断源的过程中，不会耽误了嵌套的其它中断处理过程，或自身中断源的下一次中断信号。

当一个中断发生时，中断服务例程需要取得相应的硬件状态或者数据。如果中断服务例程接下来要对状态或者数据进行简单处理，比如CPU时钟中断，中断服务例程只需对一个系统时钟tick变量进行加一操作，然后就结束中断服务例程。这类中断需要的运行时间往往都比较短。对于另外一些中断，中断服务例程在取得硬件状态或数据以后，还需要进行一系列更耗时的处理过程，通常需要将该中断分割为两部分，即上半部分（Top Half）和下半部分、底半部分（Bottom Half）。在Top Half中，取得硬件状态和数据后，打开被屏蔽的中断，给相关线程发送一条通知（可以是RT-Thread所提供的semaphore，event，mailbox或message queue等方式），然后结束中断服务例程；而接下来，相关的线程在接收到通知后，接着对状态或数据进行进一步的处理，这一过程称之为Bottom Half（底半处理）。

### 底半处理实现范例 ###

在这一节中，为了详细描述Bottom Half在RT-Thread中的实现，我们以一个虚拟的网络设备接收网络数据包作为范例（例8-1a），并假设接收到数据报文后，系统对报文的分析、处理是一个相对耗时的，比外部中断源信号重要性小许多的，而且在不屏蔽中断源信号情况下也能处理的过程。

~~~{.c}
/*
 * 程序清单：中断底半处理例子
 */

/* 用于唤醒线程的信号量 */
rt_sem_t demo_nw_isr;

/* 数据读取、分析的线程 */
void demo_nw_thread(void *param)
{
    /* 首先对设备进行必要的初始化工作 */
    device_init_setting();

    /* 装载中断服务例程 */
    rt_hw_interrupt_install(NW_IRQ_NUMBER, demo_nw_isr, RT_NULL);
    rt_hw_interrupt_umask(NW_IRQ_NUMBER);

    /*..其他的一些操作..*/

    /* 创建一个semaphore来响应Bottom Half的事件 */
    nw_bh_sem = rt_sem_create("bh_sem", 1, RT_IPC_FLAG_FIFO);

    while(1)
    {
        /* 最后，让demo_nw_thread等待在nw_bh_sem上 */
        rt_sem_take(nw_bh_sem, RT_WAITING_FOREVER);

        /* 接收到semaphore信号后，开始真正的Bottom Half处理过程 */
        nw_packet_parser (packet_buffer);
        nw_packet_process(packet_buffer);
    }
}

int rt_application_init()
{
    rt_thread_t thread;

    /* 创建处理线程 */
    thread = rt_thread_create("nwt",
		demo_nw_thread, RT_NULL, 1024, 20, 5);

    if (thread != RT_NULL)
        rt_thread_startup(thread);
}
~~~

这个例子的程序创建了一个nwt线程，这个线程在启动运行后，将阻塞在nw_bh_sem信号上，一旦这个信号量被释放，将执行接下来的nw_packet_parser过程，开始Bottom Half的事件处理。接下来让我们来看一下demo_nw_isr中是如何处理Top Half，并开启Bottom Half的，如下例。

~~~{.c}
void demo_nw_isr(int vector)
{
	/* 当network设备接收到数据后，陷入中断异常，开始执行此ISR */
	/* 开始Top Half部分的处理，如读取硬件设备的状态以判断发生了何种中断*/
	nw_device_status_read();

	/*..其他一些数据操作等..*/

	/* 释放nw_bh_sem，发送信号给demo_nw_thread，准备开始Bottom Half */
	rt_sem_release(nw_bh_sem);

	/* 然后退出中断的Top Half部分，结束device的ISR */
}
~~~

从上面例子的两个代码片段可以看出，中断服务例程通过对一个信号量对象的等待和释放，来完成中断Bottom Half的起始和终结。由于将中断处理划分为Top和Bottom两个部分后，使得中断处理过程变为异步过程。这部分系统开销需要用户在使用RT-Thread时，必须认真考虑中断服务的处理时间是否大于给Bottom Half发送通知并处理的时间。

## 中断相关接口 ##

为了尽量的使用户和系统底层异常、中断隔离开来，RT-Thread把中断和异常封装起来，以更友好的接口的形式提供给用户。（注：这部分的API由BSP提供，在某些处理器支持分支中并不一定存在，例如ARM Cortex-M0/M3分支中，请查询相关移植分支获得详细的实现情况）

### 装载中断服务例程 ###

可调用如下的接口挂载一个新的中断服务例程：

    rt_isr_handler_t rt_hw_interrupt_install(int vector,
                             rt_isr_handler_t  handler,
                             void *param,
                             char *name);

调用rt_hw_interrupt_install后，系统将把用户的中断服务例程(new_handler)和指定的中断号关联起来，当这个中断源产生中断时，系统将自动调用装载的中断服务例程。如果old_handler不为空，程序则返回之前关联的这个中断服务例程。

**函数参数**

--------------------------------------------------------------------------
                       参数  描述
---------------------------  ---------------------------------------------
                 int vector  vector是挂载的中断号。

   rt_isr_handler_t handler  新挂载的中断服务例程。
 
                void *param  param会作为参数传递给中断服务例程。

                 char *name  中断的名称。
--------------------------------------------------------------------------


**函数返回**

函数返回挂载这个中断服务例程之前挂载的中断服务例程。

**注意事项**

这个API并不会出现在每一个移植分支中，例如通常Cortex-M0/M3/M4的移植分支中就没有这个API。

### 屏蔽中断源 ###

通常在ISR准备处理某个中断信号之前，我们需要先屏蔽该中断源，以保证在接下来的处理过程中硬件状态或者数据不会受到干扰，我们可调用下面这个函数接口：

    void rt_hw_interrupt_mask(int vector);

调用rt_hw_interrupt_mask函数接口后，相应的中断将会被屏蔽（通常当这个中断触发时，中断状态寄存器会有相应的变化，但并不送达到处理器进行处理）。


**函数参数**

--------------------------------------------------------------------------
                       参数  描述
---------------------------  ---------------------------------------------
                 int vector  vector是要屏蔽的中断号
--------------------------------------------------------------------------

**注意事项**

这个API并不会出现在每一个移植分支中，例如通常Cortex-M0/M3/M4的移植分支中就没有这个API。

### 打开被屏蔽的中断源 ###

在ISR处理完状态或数据以后，需要及时的打开之前被屏蔽的中断源，使得尽可能的不丢失硬件中断信号，我们可调用下面的函数接口：

    void rt_hw_interrupt_umask(int vector);

调用rt_hw_interrupt_umask函数接口后，如果中断（及对应外设）被正确时，中断触发后，将送到到处理器进行处理。

**函数参数**

--------------------------------------------------------------------------
                     参数    描述
---------------------------  ---------------------------------------------
                 int vector  vector是要打开屏蔽的中断号
--------------------------------------------------------------------------

**注意事项**

这个API并不会出现在每一个移植分支中，例如通常Cortex-M0/M3/M4的移植分支中就没有这个API。

### 关闭中断 ###

当需要关闭整个系统的中断时，可调用下面的函数接口：

    rt_base_t rt_hw_interrupt_disable(void);

当系统关闭了中断时，就意味着当前任务/代码不会被其他事件所打断（因为整个系统已经不再对外部事件响应），也就是当前任务不会被抢占，除非这个任务主动让出处理器。

**函数返回**

函数返回中断前的系统中断状态。

### 打开中断 ###

打开中断往往是和关闭中断成对使用的，用于恢复关闭中断前的状态。调用的函数接口如下：

    void rt_hw_interrupt_enable(rt_base_t level);

调用这个函数接口将恢复调用rt_hw_interrupt_disable前的中断状态，level是上一次关闭中断时返回的值。

**函数参数**

--------------------------------------------------------------------------
      参数           描述
-------------------  -----------------------------------------------------
    rt_base_t level  level 是rt_hw_interrupt_disable函数返回的中断状态。
--------------------------------------------------------------------------

**注意事项**

调用这个接口并不代表着肯定打开中断，而是恢复关闭中断前的状态，如果调用rt_hw_interrupt_disable（）前是关中断状态，那么调用此函数后依然是关中断状态。

### 与OS相关的中断接口 ###

当整个系统被中断打断，进入中断处理函数时，OS需要知道当前已经进入到中断状态。针对这种情况，OS提供了两个函数：

    void rt_interrupt_enter(void);
    void rt_interrupt_leave(void);

rt_interrupt_enter函数用于通知OS，当前已经进入了中断状态；rt_interrupt_leave函数用于通知OS，已经离开中断状态。通常来说，OS需要知道这样的运行状态，这样在中断服务例程中，如果调用了OS相关的调用，OS好及时调整相应的行为，例如进行任务切换时应该采取中断中任务切换的策略，而不是立即进行切换。但是如果中断服务例程很显然、很必然地，它不会去调用OS相关的函数，这个时候，也可以不调用rt_interrupt_enter/leave函数。

    rt_uint8_t rt_interrupt_get_nest(void);

这个函数提供给上层应用，当前嵌套的中断深度。即，如果当前是中断上下文环境中，这个函数的返回值会大于0。

**函数返回**

函数返回当前系统中中断嵌套的深度，如果当前系统处于中断上下文环境中，返回值大于0。如果返回值大于1，说明当前出现了中断嵌套。

## ARM Cortex-M中的中断与异常 ##

ARM Cortex-M系列处理器与以往的ARM7TDMI、ARM920T相差很多，以往中断控制器都由IP授权的各家芯片厂商自行定义，而ARM Cortex-M则把中断控制器统一起来，命名为NVIC（嵌套向量中断控制）。正如其名，ARM Cortex-M NVIC支持中断嵌套功能：当一个中断触发并且系统进行响应时，处理器硬件会将当前运行的部分上下文寄存器自动压入中断栈中，这部分的寄存器包括PSR，R0，R1，R2，R3以及R12寄存器。当系统正在服务一个中断时，如果有一个更高优先级的中断触发，那么处理器同样的会打断当前运行的中断服务例程，然后把老的中断服务例程上下文的PSR，R0，R1，R2，R3和R12寄存器自动保存到中断栈中。这些部分上下文寄存器保存到中断栈的行为完全是硬件行为，这一点是与其他ARM处理器最大的区别（以往都需要依赖于软件保存上下文）。

另外，在ARM Cortex-M系列处理器上，所有中断都采用中断向量表的方式进行处理，即当一个中断触发时，处理器将直接判定是哪个中断源，然后直接跳转到相应的固定位置进行处理。而在ARM7、ARM9中，一般是先跳转进入IRQ入口，然后再由软件进行判断是哪个中断源触发，获得了相对应的中断服务例程入口地址后，再进行后续的中断处理。ARM7、ARM9的好处在于，所有中断它们都有统一的入口地址，便于OS的统一管理。而ARM Cortex-M系列处理器则恰恰相反，每个中断服务例程必须排列在一起放在统一的地址上（这个地址必须要设置到NVIC的中断向量偏移寄存器中）。

中断向量表一般由一个数组定义（或在起始代码中给出）。在STM32上，默认采用起始代码给出：

代码清单：初始化代码中的中断向量表

~~~{.c}
__Vectors       DCD     __initial_sp        	; Top of Stack
                DCD     Reset_Handler           ; Reset Handler
                DCD     NMI_Handler            	; NMI Handler
                DCD     HardFault_Handler      	; Hard Fault Handler
                DCD     MemManage_Handler      	; MPU Fault Handler
                DCD     BusFault_Handler      	; Bus Fault Handler
                DCD     UsageFault_Handler    	; Usage Fault Handler
                DCD     0                        	; Reserved
                DCD     0                       	; Reserved
                DCD     0                       	; Reserved
                DCD     0                       	; Reserved
                DCD     SVC_Handler           	; SVCall Handler
                DCD     DebugMon_Handler      	; Debug Monitor Handler
                DCD     0                        	; Reserved
                DCD     PendSV_Handler         	; PendSV Handler
                DCD     SysTick_Handler         ; SysTick Handler

… … 

NMI_Handler     		PROC
                EXPORT NMI_Handler             	[WEAK]
                B       .
                ENDP
HardFault_Handler	PROC
                EXPORT HardFault_Handler       	[WEAK]
                B       .
                ENDP

… … 
~~~

请注意代码后面的[WEAK]标识，它是符号弱化标识，在[WEAK]前面的符号如NMI_Handler、HardFault_Handler）将被执行弱化处理，如果整个代码在链接时遇到了名称相同的符号（例如与NMI_Handler相同名称的函数），那么代码将使用未被弱化定义的符号（与NMI_Handler相同名称的函数），而与弱化符号相关的代码将被自动丢弃。

RT-Thread在Cortex-M系列上也遵循这样的方法，当用户需要使用自定义的中断服务例程时，只需要定义相同名称的函数覆盖弱化符号即可。例如用户需要自定义自己的串口2中断处理函数，那么可以在代码中自己实现USART2_IRQHandler函数，在系统编译链接时，中断向量表中将只保留这份USART2_IRQHandler函数，而不是经过WAEK修饰的USART2_IRQHandler函数。

## 外设中的中断模式与轮询模式 ##
当编写一个外设驱动时，其编程模式到底采用中断模式触发还是轮询模式触发往往是驱动开发人员首先要考虑的问题，并且这个问题在实时操作系统与分时操作系统中差异还非常大。因为轮询模式本身采用顺序执行的方式：查询到相应的事件然后进行对应的处理。所以轮询模式从实现上来说，相对简单清晰。例如往串口中写入数据，仅当串口控制器写完一个数据时，程序代码才写入下一个数据（否则这个数据丢弃掉）。
相应的代码可以是这样的：

~~~{.c}
	/* 轮询模式向串口写入数据 */
	while (size)
	{
		/* 判断UART外设中数据是否发送完毕 */
		while (!(uart->uart_device->SR & USART_FLAG_TXE));
		/* 当所有数据发送完毕后，才发送下一个数据 */
		uart->uart_device->DR = (*ptr & 0x1FF);
	
		++ptr; --size;
	}
~~~

但是在实时系统中轮询模式可能会出现非常大问题，因为在实时操作系统中，当一个程序持续地执行时（轮询时），它所在的线程会一直运行，比它优先级低的线程都不会得到运行。而分时系统中，这点恰恰相反，几乎没有优先级之分，可以在一个时间片运行这个程序，然后在另外一段时间片上运行另外一段程序。

所以通常情况下，实时系统中更多采用的是中断模式来驱动外设。当数据达到时，由中断唤醒相关的处理线程，再继续进行后续的动作。例如一些携带FIFO（包含一定数据量的先进先出队列）的串口外设，其写入过程可以是这样的，如下图所示：
 
线程先向串口的FIFO中写入数据，当FIFO满时，线程主动挂起。串口控制器持续地从FIFO中取出数据并以配置的波特率（例如115200bps）发送出去。当FIFO中所有数据都发送完成时，将向处理器触发一个中断；当中断服务例程得到执行时，可以唤醒这个线程。这里举例的是FIFO类型的设备，在现实中也有DMA类型的设备，原理类似。

对于低速设备这种模式非常好，因为在串口外设把FIFO中的数据发送出去前，处理器可以运行其他的线程，这样就提高了系统的整体运行效率（甚至对于分时系统来说，这样的设计也是非常必要）。但是对于一些高速设备，例如传输速度达到10Mbps的时候，假设一次发送的数据量是32字节，我们可以计算出发送这样一段数据量需要的时间是：

    (32*8) * 1/10Mbps = 25us

当数据需要持续传输时，系统将在25us后触发一个中断以唤醒上层线程继续下次传递。假设系统的任务切换时间是8us（通常实时操作系统的任务上下文切换时间只有几个us），那么当整个系统运行时，对于数据带宽利用率将只有25/(25 + 8) = 75.8%。但是采用轮询模式，数据带宽的利用率则可能达到100%。这个也是大家普遍认为实时系统中数据吞吐量不足的缘故，系统开销消耗在了任务切换上（有些实时系统甚至会如本章前面说的，采用底半处理，分级的中断处理方式，相当于再行拉长中断到发送线程的时间开销，效率会更进一步下降）。

通过上述的计算过程，我们可以看出其中的一些关键因素：发送数据量越小，发送速度越快，对于数据吞吐量的影响也将越大。归根结底，系统中产生中断的频度如何。当一个实时系统想要提升数据吞吐量时，可以考虑的几种方式：

* 增加每次数据量发送的长度，每次尽量让外设尽量多地发送数据；
* 必要情况下更改中断模式为轮询模式。同时为了解决轮询方式一直抢占处理机，其他低优先级线程得不到运行的情况，可以把轮询线程的优先级适当降低。
