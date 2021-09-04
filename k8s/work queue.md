# 概述

`channel`是`go`协程间通信的主要方式，但使用channel时通常需要预设`channel`的buffer大小。在实际业务中，通常难以准确的预知需要的`channel`大小，若预设较大buffer的channel可能会造成内存浪费，若预设的buffer较小则可能造成处理任务的阻塞，而`channel`不支持动态扩容。因此需要一种动态调节缓冲大小的数据结构来支持协程间协作，`k8s`的`client-go`中提供了基于切片的支持线程安全的并发队列，解耦生产者与消费者，并且提供了去重、限速、重试加入队列等功能，代码简洁设计巧妙，可作为除`channel`外另一种便捷的协程通信手段。

# 基本用法

[k8s/client-go代码仓](https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go)中以一个controller处理任务为例子，提供了`work queue`队列的官方example，`work queue`部分主要代码如下

## 生产者部分

```go
// 创建一个work queue
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
// 将一个待处理的对象添加work queue
queue.Add(key)
```

## 消费者部分

```go
// workers代表处理并发处work queue中任务的协程数
for i := 0; i < workers; i++ {
	go wait.Until(c.runWorker, time.Second, stopCh)
}
// runWorker是一个死循环，通过processNextItem从队列中取出key进行处理，然后取出next key继续处理
func (c *Controller) runWorker() {
	for c.processNextItem() {
	}
}
// processNextItem就是真正的处理逻辑了，
func (c *Controller) processNextItem() bool {
	// quit为true代表队列已关闭
	key, quit := c.queue.Get()
	if quit {
		return false
	}

    // 处理完后将key从队列中移除，并且该操作是并发安全的，后续会进行详细分析
	defer c.queue.Done(key)

	// 调用实际业务代码对每个key进行处理
	err := c.syncToStdout(key.(string))
	// 如果处理异常则进行错误处理
	c.handleErr(err, key)
	return true
}
// 错误处理代码，对上一函数中可能处理失败的key进行重试等操作
func (c *Controller) handleErr(err error, key interface{}) {
	if err == nil {
        // 如果处理没有错误，调用Forget方法将key从限速队列中移除
		c.queue.Forget(key)
		return
	}

	// NumRequeues方法返回了一个key重入队列的次数，若超过设定的阈值，则从限速队列中移除，不再处理
	if c.queue.NumRequeues(key) < 5 {
		klog.Infof("Error syncing pod %v: %v", key, err)

        // 若重入队列次数没超过阈值，则添加到限速队列
		c.queue.AddRateLimited(key)
		return
	}

	c.queue.Forget(key)
	// 错误上报
	runtime.HandleError(err)
}
```

可以看到，`work queue`的使用比较简单，且官方已经给了使用模板，调用者不需要处理元素去重、重入队列、并发安全、错误处理等逻辑，只需关注自己的业务即可。

限速队列由多个队列一起完成，下文将逐一介绍其具体实现。

# 基本队列`work queue`

## 接口

该`Interface`定义了一个最基本的队列的基本方法

```go
type Interface interface {
    // 添加元素
	Add(item interface{})
    // 队列长度
	Len() int
    // 返回队首元素，并返队列队列是否关闭
	Get() (item interface{}, shutdown bool)
    // 处理完一个元素后从队列中删除
	Done(item interface{})
    // 关闭队列
	ShutDown()
    // 返回队列是否关闭
	ShuttingDown() bool
}
```

## 实现

```go
type Type struct {
    // interface{}类型的切片，存储具体的key
	queue []t
    // 存储需要被处理的元素
	dirty set
    // 存储正在处理的元素
	processing set
    // 用于唤醒其他协程队列已满足条件可以继续处理，如队列由空变为非空
	cond *sync.Cond
    // 标记队列是否关闭
	shuttingDown bool

    // 监控指标相关字段
	metrics queueMetrics
	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}
```

这里重点介绍下`shutdown`标记，该标记主要为了通知其他`goroutine`如消费者、监控组件等队列是否关闭状态，其他`goroutine`处理是检查到队列为关闭状态时停止工作，避免发生未知错误。

### 主要方法

#### `Add`

```go
func (q *Type) Add(item interface{}) {
    // 加锁互斥保证线程安全
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	if q.shuttingDown {
		return
	}
    // 去重
	if q.dirty.has(item) {
		return
	}

    // 记录监控指标
	q.metrics.add(item)

	q.dirty.insert(item)
    // 当有相同元素正在处理时，同样进行去重操作，不予入队
	if q.processing.has(item) {
		return
	}

    // 最终添加到队列
	q.queue = append(q.queue, item)
	q.cond.Signal()
}
```

`Add`方法会先key放入`dirty`和queue中，加入时都会对这两个字段存储的元素进行去重操作，同时若该key正在被处理，则不会加入，防止同一时刻有同一个key被多个worker处理，从而导致业务逻辑错误。所以dirty集合的设计既有去重功能，又保证一个元素至多被处理一次。

#### `Get`

```go
func (q *Type) Get() (item interface{}, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
    // 如果队列元素为空且没有关闭则等待其他goroutine唤醒
	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait()
	}
    // 若已经关闭则应该返回
	if len(q.queue) == 0 {
		// We must be shutting down.
		return nil, true
	}

    // 从queue字段取出一个元素
	item, q.queue = q.queue[0], q.queue[1:]

    // 监控相关
	q.metrics.get(item)

    // 把key加入processing集合并从dirty删除，这样相同的key可以在当前key处理完后继续入队处理
	q.processing.insert(item)
	q.dirty.delete(item)

	return item, false
}
```

`Get`操作也比较简单，需要注意的是Get时key也会从dirty集合中移除

#### `Done` & `ShutDown`

```go
func (q *Type) Done(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.metrics.done(item)

	q.processing.delete(item)
    // 若key在处理的过程中，又再次被加入到队列，由Add方法可知，当key在processing中时，Add操作只是把key放到了dirty集合，并没有放入queue中，因此
    // 相同的key处理完从processing中移除后，需要把key再放入到queue中，防止key被遗漏
	if q.dirty.has(item) {
		q.queue = append(q.queue, item)
		q.cond.Signal()
	}
}

func (q *Type) ShutDown() {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	q.shuttingDown = true
	q.cond.Broadcast()
}
```

以上就是`work queue`的基本逻辑，可以看到通过设置dirty和processing两个集合，`work queue`实现了去重功能，并防止了相同key被同时处理的错误。接下来将介绍`work queue`如何实现延迟已经限速。

# 延迟队列`DelayingQueue`

## 接口

```go
type DelayingInterface interface {
	Interface
    // 经过duration时间后item被重新加入队列
	AddAfter(item interface{}, duration time.Duration)
}
```

延迟队列在上文介绍的`work queue`基础上实现，继承了`Interface`接口，多了一个`AddAfter`方法，通过设置指定的duration来达到限速的目的。

## 实现

```go
func newDelayingQueue(clock clock.Clock, q Interface, name string) *delayingType {
	ret := &delayingType{
		Interface:       q,
		clock:           clock,
		heartbeat:       clock.NewTicker(maxWait),
		stopCh:          make(chan struct{}),
		waitingForAddCh: make(chan *waitFor, 1000),
		metrics:         newRetryMetrics(name),
	}

	go ret.waitingLoop()
	return ret
}
```

`newDelayingQueue`返回一个`delayingType`类型的限速队列，同时会启动一个`waitingLoop`协程处理被添加的key，接下来详细分析`AddAfter`和`waitingLoop`的实现。

### 主要方法

#### `AddAfter`

```go
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
	if q.ShuttingDown() {
		return
	}

	q.metrics.retry()

	// 没有延迟直接加入queue
	if duration <= 0 {
		q.Add(item)
		return
	}

	select {
	case <-q.stopCh:
        // 将封装后的key放入waitingForAddCh channel
	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:
	}
}
```

`AddAfter`的主要逻辑就是将key和key的延迟时间封装成一个`waitFor` `struct`，其中`readyAt`即为key应该加入到队列的时间，然后将该`struct`放入到`waitingForAddCh`中，`waitingLoop`协程会异步进行处理。默认`waitingForAddCh`的大小为1000，当`channel`满时添加key会被block。

#### `waitingLoop`

```go
func (q *delayingType) waitingLoop() {
	defer utilruntime.HandleCrash()

	never := make(<-chan time.Time)

    // 记录等待队列中第一个key需要的等待的时间
	var nextReadyAtTimer clock.Timer

    //  基于堆实现的优先级队列，需要最早被加入到队列的key放在最前面
	waitingForQueue := &waitForPriorityQueue{}
	heap.Init(waitingForQueue)

	waitingEntryByData := map[t]*waitFor{}

	for {
		if q.Interface.ShuttingDown() {
			return
		}

		now := q.clock.Now()

		// 判断堆顶元素是否到期需要加入队列
		for waitingForQueue.Len() > 0 {
			entry := waitingForQueue.Peek().(*waitFor)
            // 如果还没到期，继续等待或者继续监听后续key加入事件
			if entry.readyAt.After(now) {
				break
			}

            // 从堆顶弹出元素添加到队列
			entry = heap.Pop(waitingForQueue).(*waitFor)
			q.Add(entry.data)
			delete(waitingEntryByData, entry.data)
		}

		nextReadyAt := never
		if waitingForQueue.Len() > 0 {
			if nextReadyAtTimer != nil {
				nextReadyAtTimer.Stop()
			}
			entry := waitingForQueue.Peek().(*waitFor)
			nextReadyAtTimer = q.clock.NewTimer(entry.readyAt.Sub(now))
			nextReadyAt = nextReadyAtTimer.C()
		}

		select {
		case <-q.stopCh:
			return

		case <-q.heartbeat.C():
			// 等待心跳时间过期

		case <-nextReadyAt:
			// 等待对顶元素时间过期

            // 从waitingForAddCh中取元素，若已经到期直接加入到队列，否则加入堆中等待处理
		case waitEntry := <-q.waitingForAddCh:
			if waitEntry.readyAt.After(q.clock.Now()) {
				insert(waitingForQueue, waitingEntryByData, waitEntry)
			} else {
				q.Add(waitEntry.data)
			}

            // 尝试再取一个
			drained := false
			for !drained {
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
						insert(waitingForQueue, waitingEntryByData, waitEntry)
					} else {
						q.Add(waitEntry.data)
					}
				default:
					drained = true
				}
			}
		}
	}
}
```

`waitingLoop`监听`waitingForAddCh` `channel`，从中取出待添加到队列的key，如果已经到期则直接加入，否则将key放入到堆中，然后每次从堆中取出最先过期的key进行判断处理。

# 限速队列`RateLimitingQueue`

## 接口

```go
type RateLimitingInterface interface {
	DelayingInterface

    // 限速器rate limiter指定的时间到期后将item加入队列
	AddRateLimited(item interface{})

	// 将item从限速器删除，不再进行重试加入，但还是需要调用Done方法删除item
	Forget(item interface{})

	// 返回item被重新入队列的次数
	NumRequeues(item interface{}) int
}
```

`RateLimitingInterface`同样是在·的基础上多了三个方法，使用限速队列时需要调用`NewNamedRateLimitingQueue`方法传入`RateLimiter`，调用时可以传入不同的限速器`ratelimiter`实现，官方提供了四种rate Limiter实现，分别是`BucketRateLimiter`、`ItemExponentialFailureRateLimiter`、`ItemFastSlowRateLimiter`和`MaxOfRateLimiter。`

`RateLimiter`需要实现三个方法

```go
type RateLimiter interface {
   // 返回key需要等待加入队列的时间
   When(item interface{}) time.Duration
   // 取消key的重试
   Forget(item interface{})
   // 记录一个key被重试了多少次
   NumRequeues(item interface{}) int
}
```

## `BucketRateLimiter`

`BucketRateLimiter`是基于token令牌桶的限速方法，通过三方库`golang.org/x/time/rate`实现，令牌桶的算法原理是将固定数目的token放入桶中，桶满时则不再添加，然后元素需要拿到token才能被处理，后续元素需要等待有空闲的token被释放。使用时通过如下代码初始化一个令牌桶。

```go
rate.NewLimiter(rate.Limit(10), 100)
```

10代表每秒往桶中放入token的数量，100代表token数量，加入有102个元素在，则前100个元素直接通过，而对于第100个元素，由于每秒放入10个token，因此处理一个token需要`100ms`，所以第101个元素需要等待`100ms`，同理第102个元素需要等待`200ms`。

## `ItemExponentialFailureRateLimiter`

`ItemExponentialFailureRateLimiter`就是指数退避算法，有两个主要参数`baseDelay`，`maxDelay`，`baseDelay`代表需要推迟的基数，每次添加相同的key其对应的延迟加入时间会指数递增，但也不能无限递增，因此`maxDelay`规定了延迟时间的上限。指数退避部分主要代码如下。

```go
// 每次进来exp指数加1
exp := r.failures[item]
r.failures[item] = r.failures[item] + 1

backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
if backoff > math.MaxInt64 {
   return r.maxDelay
}
```

利润`baseDelay`为10，`maxDelay`为1000，同一个key第一次进行需要等待的时间为10*2^1，第二次为10\*2^2，以此类推。

## `ItemFastSlowRateLimiter`

`ItemFastSlowRateLimiter`定义了两个时间`fastDelay`、`lowDelay`以及达到`fastDelay`的阈值`maxFastAttempts。`

```go
r.failures[item] = r.failures[item] + 1

if r.failures[item] <= r.maxFastAttempts {
   return r.fastDelay
}
return r.slowDelay
```

当重新加入队列的次数小于阈值`maxFastAttempts`，需要等待的时间为`fastDelay`，超过阈值则需要等待更长的时间`slowDelay。`

## `MaxOfRateLimiter`

`MaxOfRateLimiter`则是多个`RateLimiter`的组合，需要延迟的时间为各个`RateLimiter`的时间最大值。

# 总结

`client-go`的`work queue`首先实现了一个最基础的队列，包含最基本的`Add`、`Get`等方法，然后在该队列基础上实现了`DelayingQueue`，实现了延迟队列的功能，最后在延迟队列的基础上实现了`RateLimitingQueue`，层层嵌套，最终实现了功能完善的限速队列，使得使用者只需关注业务逻辑，不需要自己实现底层逻辑。