---
title: perf在线查问题常用方式
date: 2018-07-28 11:57:32
tags:
---
虽然转后台已经有一年，但主要做玩法这一块，所以在线查问题主要查的是崩溃问题，因为线上部署的ds是debug版本，所以主要通过gdb查看堆栈来解决问题，前不久查询系统组的一个进程问题，有关心得记录一下,以下文字并非原创，梳理自其他文档并摘抄

参考：https://blog.csdn.net/zhangskd/article/details/37902159

<h2>perf的基本介绍</h2>

Perf是一个linux下的性能调试工具，能够帮助开发者发现哪些代码会消耗更多的CPU时间以及这些代码为什么会消耗额外的CPU时间，同时具有开销小、适用面广等优点。Perf对判断系统的性能瓶颈非常有用。它可以被配置以中断或定期方式获取事件实例数来收集代码的运行信息。在许多体系结构上，Perf提供了对性能计数器的访问。

<h4>性能计数器</h4>

性能计数器是现代CPU中一些特殊的硬件寄存器。这些寄存器对特定的硬件事件进行计数：比如指令的执行，cache miss或者是分支预测错误等，同时不会对内核或应用的性能产生影响。这些寄存器中的内容可以被定期收集，同时这些寄存器也可以在特定的事件数量超过一定数值时触发中断。而在这些中断中，特定的事件数量被记录，并在应用层需要时返回这些数据。这样，使得开发者能够收集代码执行过程中某些事件比如cache miss，内存引用以及CPU clock或cycle等的相关信息，并使得开发者能够根据它们来判断代码的运行情况及其原因
性能计数器可以通过特殊的文件描述符访问。这些特殊的文件描述符通过特定的系统调用perf_event_open打开，并可以通过通常的文件访问接口进行访问和设置。多个性能计数器可以被一次打开，并支持poll操作。

<h4>perf的运行原理</h4>

　　性能调优工具如 perf，Oprofile 等的基本原理都是对被监测对象进行采样，最简单的情形是根据 tick 中断进行采样，即在 tick 中断内触发采样点，在采样点里判断程序当时的上下文。假如一个程序 90% 的时间都花费在函数 foo() 上，那么 90% 的采样点都应该落在函数 foo() 的上下文中。运气不可捉摸，但我想只要采样频率足够高，采样时间足够长，那么以上推论就比较可靠。因此，通过 tick 触发采样，我们便可以了解程序中哪些地方最耗时间，从而重点分析。

　　稍微扩展一下思路，就可以发现改变采样的触发条件使得我们可以获得不同的统计数据：

　　以时间点 ( 如 tick) 作为事件触发采样便可以获知程序运行时间的分布。

　　以 cache miss 事件触发采样便可以知道 cache miss 的分布，即 cache 失效经常发生在哪些程序代码中。如此等等。

　　因此让我们先来了解一下 perf 中能够触发采样的事件有哪些。

　　使用perf list（在root权限下运行），可以列出所有的采样事件

<h4>Perf工具集</h4>

perf 是一个包含22种子工具的工具集，以下是最常用的5种：
* perf list // 用来查看perf所支持的性能事件，有软件的也有硬件的
* perf stat
* perf top // top 类似于 Linux 的 top 命令，对系统性能进行实时分析
* perf record
* perf report

<h2>Perf能够观察的事件类型</h2>

事件分为以下三种：
1）Hardware Event 是由 PMU 硬件产生的事件，比如 cache 命中，当您需要了解程序对硬件特性的使用情况时，便需要对这些事件进行采样；
2）Software Event 是内核软件产生的事件，比如进程切换，tick 数等 ;
3）Tracepoint event 是内核中的静态 tracepoint 所触发的事件，这些 tracepoint 用来判断程序运行期间内核的行为细节，比如 slab 分配器的分配次数等。

上述每一个事件都可以用于采样，并生成一项统计数据

其中，PERF_TYPE_HARDWARE类事件包括：
（1）       cpu-cycles：某段时间内的CPU cycle数；  // cpu周期
（2）       instructions：某段时间内的CPU所执行的指令数；
（3）       cache misses：cache miss次数；
（4）       branch misses：分支预测错误次数；
而PERF_TYPE_SOFTWARE包含的事件包括：
（1）       cpu-clock：某段时间内的cpu时钟数；
（2）       page faults：页错误次数；
（3）       context switches：上下文交换次数；

以上是一些常用的事件，其中CPU cycles事件和cpu-clock事件因比较常用，我们说一下它们的区别：cpu-clock可以用来表示程序执行经过的真实时间，而无论CPU处于什么状态（Pn（n非0）或者是C状态）；而CPU cycles则用来表示执行程序指令花费的时钟周期数，如果CPU处于Pn（n非0）或者是C状态，则cycles的产生速度会减慢。也即，如果你想查看哪些代码消耗的真实时间多，则可以使用cpu-clock事件；而如果你想查看哪些代码消耗的时钟周期多，则可以使用CPU cycles事件。

Perf工具的常用命令包括stat，record，report等。Perf stat命令用来显示程序运行的整体状况；Perf record命令则用来记录指定事件在程序运行过程中的信息，而Perf report命令则用来报告基于前面record命令记录的事件信息生成的程序运行状况报告。

在程序运行出现问题时，我们希望快速定位热点代码段，以使得后面的优化和问题分析有的放矢，实现问题的快速定位与解决。我们可以使用Perf record命令记录合适的事件信息，并使用Perf report命令生成程序运行状况报告

<h2>perf top</h2>

对于一个指定的性能事件(默认是CPU周期)，显示消耗最多的函数或指令
perf top主要用于实时分析各个函数在某个性能事件上的热度，能够快速的定位热点函数，包括应用程序函数、模块函数与内核函数，甚至能够定位到热点指令。默认的性能事件为cpu cycles。

<h4>使用例子</h4>

	```c++
	perf top -e cycles:k   显示内核和模块中，消耗最多CPU周期的函数：
    perf top -e kmem:kmem_cache_alloc 显示分配高速缓存最多的函数
    可能因为权限问题要加sudo
	```
perf top加上-p参数可以指定某一个进程
<h2>perf stat——概览程序的运行情况</h2>

　　面对一个问题程序，最好采用自顶向下的策略。先整体看看该程序运行时各种统计事件的大概，再针对某些方向深入细节。而不要一下子扎进琐碎细节，会一叶障目的。
　　有些程序慢是因为计算量太大，其多数时间都应该在使用CPU进行计算，这叫做CPUbound型；有些程序慢是因为过多的IO，这种时候其CPU利用率应该不高，这叫做IObound型；对于CPUbound程序的调优和IObound的调优是不同的。

　　如果您认同这些说法的话，Perfstat应该是您最先使用的一个工具。它通过概括精简的方式提供被调试程序运行的整体情况和汇总数据。如下测试代码：

	
        ```c++
         void longa() 
         { 
           int i,j; 
           for(i = 0; i < 1000000; i++) 
           j=i; //am I silly or crazy? I feel boring and desperate. 
         }

         void foo2() 
         { 
           int i; 
           for(i=0 ; i < 10; i++) 
                longa(); 
         }

         void foo1() 
         { 
           int i; 
           for(i = 0; i< 100; i++) 
              longa(); 
         }

         int main(void) 
         { 
           foo1(); 
           foo2(); 
         } 
        ```
        
将它编译为可执行文件 test1
	```c++
    > gcc -o test1 -g test.c // 注意：此处一定要加-g选项，加入调试和符号表信息。
	```

下面演示了 perf stat 针对程序 test1 的输出：

   ```c++
   >perf stat ./test1
   
   Performance counter stats for './test1':
        268.420706 task-clock                #    1.000 CPUs utilized          
                 1 context-switches          #    0.000 M/sec                  
                19 CPU-migrations            #    0.000 M/sec                  
               117 page-faults               #    0.000 M/sec                  
         776123911 cycles                    #    2.891 GHz                    
         555285891 stalled-cycles-frontend   #   71.55% frontend cycles idle   
     <not counted> stalled-cycles-backend  
         551088144 instructions              #    0.71  insns per cycle        
                                             #    1.01  stalled cycles per insn
         110223807 branches                  #  410.638 M/sec                  
              8162 branch-misses             #    0.01% of all branches        

       0.268504557 seconds time elapsed

   ```
   
结果分析： 对 test1进行调优应该要找到热点 ( 即最耗时的代码片段 )，再看看是否能够提高热点代码的效率,缺省情况下，除了 task-clock-msecs 之外，perf stat 还给出了其他几个最常用的统计信息：
* Task-clock-msecs：CPU 利用率，该值高，说明程序的多数时间花费在 CPU 计算上而非 IO。
* Context-switches：进程切换次数，记录了程序运行过程中发生了多少次进程切换，频繁的进程切换是应该避免的。
* Cache-misses：程序运行过程中总体的 cache 利用情况，如果该值过高，说明程序的 cache 利用不好
* CPU-migrations：表示进程 t1 运行过程中发生了多少次 CPU 迁移，即被调度器从一个 CPU 转移到另外一个 CPU 上运行。
* Cycles：处理器时钟，一条机器指令可能需要多个 cycles，
* Instructions: 机器指令数目。
* IPC：是 Instructions/Cycles 的比值，该值越大越好，说明程序充分利用了处理器的特性。
* Cache-references: cache 命中的次数
* Cache-misses: cache 失效的次数。
* branches：遇到的分支指令数。branch-misses是预测错误的分支指令数。
  
<h2>Perf示例1, perf的基本使用方式</h2>

某业务网络服务程序在运行时出现CPU占用率100%的问题，同时服务性能很差。为了揭开问题产生的原因，我们使用Perf record程序来记录CPU cycles事件，希望能发现是哪些代码占用了大量的CPU时间。为了达到此目的，我们需要执行如下命令：

    ```c++
    >perf record -e cycles -a ./XXX 
    或
    >perf record -e cycles  -g -a -p PID
    ```

注意上方，加入-g则可以在生成的报告中加入-g命令获取详细信息，生成报告如下：
	```c++
    >perf report // 可以看到每个函数的耗时，看不到堆栈
    或
    > perf report -g // 详细
    // 报告中可以使用方向键和enter键查询展开，或者使用如下方式自动全部展开
    > perf report -g >1.txt
    ```

<h2>实例2，综合使用Perf的多个命令与多种事件</h2>




