# 计算机体系结构

1. CPU有三级缓存，L1和L2cache是每个核私有的，L3cache在同一个CPU插槽上（对应多个核）是共享的。为了解决不同核之间数据一致性问题，提出了MESI协议

    - M：Modified 修改
    - E：Exclusive 互斥、独占
    - S：Shared 共享
    - I：Invalid 无效

   每个cache line有两个bit位来表示该cache的四个状态

   在进行读写操作时，通过发送**总线事务**通知其他CPU更改状态，并等待这些状态返回。

   eg：本CPU执行读操作，发现local cache没有数据，因此通过read发起一次bus transaction，来自其他的cpu local cache或者memory会通过read response回应，从而将该 cache line 从Invalid状态迁移到shared状态。

   当cache line处于shared状态的时候，说明在多个cpu的local cache中存在副本，因此，这些cacheline中的数据都是read only的，一旦其中一个cpu想要执行数据写入的动作，必须先通过invalidate获取该数据的独占权，而其他的CPU会以invalidate acknowledge回应，清空数据并将其cacheline从shared状态修改成invalid状态。

2. 但是每次通过消息传递到其他CPU核心并等待这些CPU核心返回消息比较耗费CPU周期，因此引入了store buffer 和 invalidate queue（相当于延迟对数据的写入或读取），但随之带来的问题是CPU级别的指令重排，导致多核并发编程出现问题。解决的办法抛给应用层，在应用层加入读或写屏障，保证顺序执行与数据在内存的可见性

3. 总结一下就是：指令重排（内存）包括硬件和软件两个方面，硬件就是上面说的store buffer机制引起的，软件层面就是编译器对指令进行编译重排

4. go语言中如何避免指令重排

   由于go的哲学是通过通信实现共享内存，因此并没有直接提供CPU级别的指令进行编程，而是通过happen-before来控制代码的执行顺序，比如

    - **init和包级变量初始化**：包引用其他包时，执行顺序：依赖包包级变量初始化 < 依赖包init函数执行 < 当前包包级变量初始化 < 当前包init函数执行（<表示先于
    - **Channel**：对一个元素的send操作Happens Before对应的**receive完成**操作；对channel的close操作Happens Before receive 端的收到关闭通知操作；
    - **Lock**：简单理解就是这一次的Lock总是Happens After上一次的Unlock，读写锁的RLock HappensAfter上一次的UnLock，其对应的RUnlock Happens Before 下一次的Lock。
    - **Once**：once中的函数一定是在都执行完后，别的协程才有机会拿到这个要执行的函数，即保证仅有一个goroutine可以调用f()，其**余goroutine的调用会阻塞直至f()返回。**

5. 