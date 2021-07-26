# channel

1. 数据结构

   ```go
   type hchan struct {
      qcount   uint           // total data in the queue
      dataqsiz uint           // size of the circular queue
      buf      unsafe.Pointer // points to an array of dataqsiz elements
      elemsize uint16
      closed   uint32
      elemtype *_type // element type
      sendx    uint   // send index
      recvx    uint   // receive index
      recvq    waitq  // list of recv waiters
      sendq    waitq  // list of send waiters
   
      // lock protects all fields in hchan, as well as several
      // fields in sudogs blocked on this channel.
      //
      // Do not change another G's status while holding this lock
      // (in particular, do not ready a G), as this can deadlock
      // with stack shrinking.
      lock mutex
   }
   ```

2. 发送数据，调用的是[`runtime.chansend`](https://draveness.me/golang/tree/runtime.chansend)，主要流程

    - 如果当前 Channel 的 `recvq` 上存在已经被阻塞的 Goroutine，那么会直接将数据发送给当前 Goroutine 并将其设置成下一个运行的 Goroutine；
    - 如果 Channel 存在缓冲区并且其中还有空闲的容量，我们会直接将数据存储到缓冲区 `sendx` 所在的位置上；
    - 如果不满足上面的两种情况，会创建一个 [`runtime.sudog`](https://draveness.me/golang/tree/runtime.sudog) 结构并将其加入 Channel 的 `sendq` 队列中，当前 Goroutine 也会陷入阻塞等待其他的协程从 Channel 接收数据；

3. 接收数据

    - 如果 Channel 为空，那么会直接调用 [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) 挂起当前 Goroutine；

    - 如果 Channel 已经关闭并且缓冲区没有任何数据，[`runtime.chanrecv`](https://draveness.me/golang/tree/runtime.chanrecv) 会直接返回；

    - 如果 Channel 的 `sendq` 队列中存在挂起的 Goroutine，会将 `recvx` 索引所在的数据拷贝到接收变量所在的内存空间上并将 `sendq` 队列中 Goroutine 的数据拷贝到缓冲区；此时又分两种情况：
        - 如果 Channel 不存在缓冲区；
            1. 调用 [`runtime.recvDirect`](https://draveness.me/golang/tree/runtime.recvDirect) 将 Channel 发送队列中 Goroutine 存储的 `elem` 数据拷贝到目标内存地址中；
        - 如果 Channel 存在缓冲区；
            1. 将队列中的数据拷贝到接收方的内存地址；
            2. 将发送队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方；

    - 如果 Channel 的缓冲区中包含数据，那么直接读取 `recvx` 索引对应的数据；

    - 在默认情况下会挂起当前的 Goroutine，将 [`runtime.sudog`](https://draveness.me/golang/tree/runtime.sudog) 结构加入 `recvq` 队列并陷入休眠等待调度器的唤醒；

4. 注意每次唤醒时，只是调用 [`runtime.goready`](https://draveness.me/golang/tree/runtime.goready) 将等待接收数据的 Goroutine 标记成可运行状态 `Grunnable` 并把该 Goroutine 放到发送方所在的处理器的 `runnext` 上等待执行，该处理器在下一次调度时会立刻唤醒数据的接收方；

