# Select

1. 多个channel都可用时，会随机选择一个channel执行，因为如果按顺序执行，则下面的channel得不到执行，因此为了避免饥饿使用随机执行

2. 使用default可以让select不阻塞

3. 数据结构：select并没有直接的数据结构，下面这个是case段的数据结构，因为case操作都有channel有关，所以含有一个channel结构体

   ```go
   type scase struct {
   	c    *hchan         // chan
   	elem unsafe.Pointer // data element
   }
   ```

4. select中会包含如下几种情况

    - `select` 不存在任何的 `case`；会直接调用block函数进行阻塞，block函数中会调用schedule()让出本goroutine对处理器的使用权
    - `select` 只存在一个 `case`；会先判断ch是否为nil，然后继续阻塞
    - `select` 存在两个 `case`，其中一个 `case` 是 `default`；此时调用[`runtime.selectnbsend`](https://draveness.me/golang/tree/runtime.selectnbsend)或者[`runtime.selectnbrecv`](https://draveness.me/golang/tree/runtime.selectnbrecv)进行非阻塞的调用
    - `select` 存在多个 `case`；最后一种情况，单独分析

5. 分析上文的`select` 存在多个 `case`的情况，默认情况下，编译器分以下步骤处理

    - 将所有的 `case` 转换成包含 Channel 以及类型等信息的 [`runtime.scase`](https://draveness.me/golang/tree/runtime.scase) 结构体；

    - 调用运行时函数 [`runtime.selectgo`](https://draveness.me/golang/tree/runtime.selectgo) 从多个准备就绪的 Channel 中选择一个可执行的 [`runtime.scase`](https://draveness.me/golang/tree/runtime.scase) 结构体；

    - 通过 `for` 循环生成一组 `if` 语句，在语句中判断自己是不是被选中的 `case`；

   ```go
   selv := [3]scase{}
   order := [6]uint16
   for i, cas := range cases {
       c := scase{}
       c.kind = ...
       c.elem = ...
       c.c = ...
   }
   chosen, revcOK := selectgo(selv, order, 3)
   if chosen == 0 {
       ...
       break
   }
   if chosen == 1 {
       ...
       break
   }
   if chosen == 2 {
       ...
       break
   }
   ```

   主要的代码逻辑在selectgo函数中，下面分析该函数

6. `selectgo`包含两大主要部分

    - 初始化：通过引入随机数确定轮询顺序和加锁顺序

    - 循环：包含三个阶段

    1. 根据 `pollOrder` 遍历所有的 `case` 查看是否有可以立刻处理的 Channel；
    2. 如果存在，直接获取 `case` 对应的索引并返回；如果不存在，创建 [`runtime.sudog`](https://draveness.me/golang/tree/runtime.sudog) 结构体，将当前 Goroutine 加入到所有相关 Channel 的收发队列，并调用 [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) 挂起当前 Goroutine 等待调度器的唤醒；
    3. 当调度器唤醒当前 Goroutine 时，会再次按照 `lockOrder` 遍历所有的 `case`，从中查找需要被处理的 [`runtime.sudog`](https://draveness.me/golang/tree/runtime.sudog) 对应的索引；

   

   











