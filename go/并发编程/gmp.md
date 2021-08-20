# GMP

1. 最多创建10000个M，也就是内核级线程，但最多有`GOMAXPROCS` 个活跃线程能够正常运行

2. 数据结构

   ```go
   type m struct {
   	g0   *g
   	curg *g
   	...
   }
   ```

   其中 g0 是持有调度栈的 Goroutine，也就是自举协程。`curg` 是在当前线程上运行的用户 Goroutine，这也是操作系统线程唯一关心的两个 Goroutine。

   g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行

3. p在空闲时会唤醒一个m。

4. 当执行非阻塞调用时，m和g与p解绑，此时m会记着p，当系统调用结束继续找之前那个p，若p空闲则接着执行，若p有任务在执行，则获取其他p，若仍没有空闲p，则将g放入全局队列。

5. 系统中有个`sysmon`线程(不与任何p绑定，相当于一个独立的内核线程)进行一下系统监控

   - 检查死锁
   - 运行计时器(`goready`唤醒一个到期的计时器执行)
   - 网络轮询
   - 抢占处理器
   - 垃圾回收

# 抢占式调度

1. `goready`和`gopark`函数

   `goready`：唤醒goroutine，并设置为runnable状态，放入到p的runq里等待被调度，相当于进程的就绪，不等于运行状态

   `gopark`：解除goroutine和m的绑定关系，将当前goroutine切换我等待状态，切换到m的g0栈，然后执行park_m函数，最后调用schedule()函数重新触发调度。调用`gopark`后goroutine不可被调度，下次需要先调用`goeady`切换成可调度状态才能被掉调度运行。

   https://blog.csdn.net/u010853261/article/details/85887948

2. 分为协作式抢占和基于信号的抢占

# 协作式抢占

1. 两种方式触发

   - 用户手动调用Gosched()
   - 通过在函数前后插入抢占检测指令来进行抢占，当检测到当前 Goroutine 被标记为被应该被抢占时， 则主动中断执行，让出执行权利。若stackguard0设置为stackPreempt会抢占，这种情况是在栈分段时会判断，所以都是在发生函数调用时判断栈是否需要扩张时才会可能进行抢占，因此如果没有发生函数调用就不会发生抢占。

2. 使用加入抢占指令方式时何时会被置为stackPreempt：

   进入系统调用；任何运行时不再持有锁的时候（m.locks == 0）；圾回收器需要停止所有用户 Goroutine 时(STW时)

# 基于信号的抢占

基于信号的抢占在sysmon中体现，系统线程会监控goroutine的运行当g发生系统调用或者goroutine运行时间超过10ms时都会调用函数preempt给M发送SIGURG 信号进行抢占。m通过doSigPreempt函数的抢占处理函数asyncPreempt2处理。

- 发生syscall时设置p为idle，调用handoffp主动让出处理器处理权，供其他m使用。
- 运行超过10ms时，发送SIGURG 通知m进行抢占调度，然后m最终调用asyncPreempt2进行处理，有两种处理方式，如果g的preemptStop状态为true，需要m放弃g，让出线程，然后调用schedule继续调度其他goroutine。否则解绑m和g，然后把g放到全局调度队列，这种相当于不是抢占，而是m主动放弃g，让出线程