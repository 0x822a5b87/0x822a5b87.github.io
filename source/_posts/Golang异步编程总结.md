---
title: Golang异步编程总结
date: 2024-04-24 17:19:10
tags:
---

# Golang异步编程总结

## 常用方式

### 基于go关键词

> 匿名函数

```go
func main() {
	for i := 0; i < 10; i++ {
		go func(val int) {
			fmt.Println(val)
		}(i)
	}

	// 这里其实是一个未定义行为
	time.Sleep(1 * time.Second)
}
```

> 使用函数

```go
func main() {

	f := func(val int) {
		fmt.Println(val)
	}

	for i := 0; i < 10; i++ {
		go f(i)
	}

	// 这里其实是一个未定义行为
	time.Sleep(1 * time.Second)
}
```

### 基于goroutine和channel

```go
func TestChan(t *testing.T) {
	input := 10

	// 不带缓冲区
	ch := make(chan int)
	//ch := make(chan int, 1)
	go func() {
		ch <- input
	}()

	val := <-ch

	assert.Equal(t, input, val)
}
```

### 使用sync.WaitGroup

```go
func TestWaitGroup(t *testing.T) {
	wg := sync.WaitGroup{}
	var goroutineSize = 10
	wg.Add(goroutineSize)
	for i := 0; i < goroutineSize; i++ {
		go func() {
			wg.Done()
		}()
	}
	wg.Wait()
}
```

### 使用errorgoup.Wait

#### errorgroup.Group

> A Group is a collection of goroutines working on subtasks that are part of the same overall task.

```go
func TestErrorGroup(t *testing.T) {
	var eg errgroup.Group
	var errorIndex = 8
	for i := 0; i < GoroutineSize; i++ {
		// 这里特别容易出错，因为 eg.Go() 是在内部调用 go func() 产生了一个闭包
		localVal := i
		eg.Go(func() error {
			// 在特定的位置抛出异常
			if localVal == errorIndex {
				return errors.New(strconv.Itoa(localVal))
			}
			return nil
		})
	}

	err := eg.Wait()

	assert.Error(t, err, strconv.Itoa(errorIndex))
}
```

## 一些使用技巧

### 使用channel的range和close操作

```go
func TestRangeAndClose(t *testing.T) {
	ch := make(chan int)
	var sum int
	go func() {
		for val := 0; val < GoroutineSize; val++ {
			sum += val
			ch <- val
		}
		close(ch)
	}()

	var sum2 int
	for val := range ch {
		sum2 += val
	}

	assert.Equal(t, sum, sum2)
}
```

### 使用select实现多个异步操作的等待

```go
func TestSelect(t *testing.T) {
	ch0 := make(chan int)
	ch1 := make(chan int)
	stop := make(chan *int)

	go func() {
		for i := 0; i < GoroutineSize; i++ {
			if i%2 == 0 {
				ch0 <- i
			} else {
				ch1 <- i
			}
		}

		stop <- nil
	}()

	running := true
	var sum int
	for running {
		select {
		case val := <-ch0:
			sum += val
		case val := <-ch1:
			sum += val
		case <-stop:
			running = false
		}
	}

	var sum2 int
	for i := 0; i < GoroutineSize; i++ {
		sum2 += i
	}

	assert.Equal(t, sum, sum2)
}
```

### 使用select和time.After()实现超时控制

```go
func TestAfter(t *testing.T) {
	// 这个里面的assert大部分情况下是对的，然而实际上是未定义行为
	// 因为这里存在两个协程的竞争关系
	ch := make(chan int)

	go func() {
		time.Sleep(10 * time.Second)
		ch <- 0
	}()

	var selectedVal int
	select {
	case selectedVal = <-ch:
		break
	case _ = <-time.After(1 * time.Second):
		// 这个case有一个很特殊的地方，就是我们在select里初始化了Timer，
		// 在这个位置不会有问题，因为我们这里是没有循环的
		selectedVal = 10
		break
	}

	assert.Equal(t, 0, selectedVal)
}

```



### 使用time.Tick()和time.After()进行定时操作

```go
func TestTick(t *testing.T) {
	tick := time.Tick(1 * time.Second)
	after := time.After(5 * time.Second)

	var sum = 0
	var running = true
	for running {
		select {
		case <-tick:
			sum++
		case <-after:
			//case <-time.After(5 * time.Second):
			// 这里我们的代码是在一个for循环中，这意味着我们每次进入循环都新建了一个Timer对象
			running = false
		}
	}
	fmt.Println(sum)
}
```

### 使用sync.Mutex()和sync.RWMutex()进行并发安全访问

```go
func TestMutex(t *testing.T) {
	wg := sync.WaitGroup{}
	mutex := sync.Mutex{}
	mutex.Lock()

	countFunc := func(ret *int) {
		for i := 0; i < LoopSize; i++ {
			*ret = *ret + i
		}
	}

	var sum = 0
	countFunc(&sum)

	var mutexSum = 0
	wg.Add(2)
	go func() {
		defer wg.Done()
		defer mutex.Unlock()
		// 休眠1秒等其他的协程执行
		time.Sleep(1 * time.Second)
		countFunc(&mutexSum)
		// 确认数据是否正常
		assert.Equal(t, sum, mutexSum)
		fmt.Printf("sum = %d, mutexSum = %d\n", sum, mutexSum)
	}()

	go func() {
		mutex.Lock()
		defer wg.Done()
		defer mutex.Unlock()
		countFunc(&mutexSum)
		assert.Equal(t, 2*sum, mutexSum)
		fmt.Printf("sum = %d, mutexSum = %d\n", sum, mutexSum)
	}()

	wg.Wait()
}
```

### 使用sync.Cond进行条件变量控制

> ```go
> // Cond implements a condition variable, a rendezvous point
> // for goroutines waiting for or announcing the occurrence
> // of an event.
> //
> // Each Cond has an associated Locker L (often a *Mutex or *RWMutex),
> // which must be held when changing the condition and
> // when calling the Wait method.
> //
> // A Cond must not be copied after first use.
> type Cond struct {
> 	noCopy noCopy
> 
> 	// L is held while observing or changing the condition
> 	L Locker
> 
> 	notify  notifyList
> 	checker copyChecker
> }
> ```



1. `sync.Cond` 基于互斥锁/读写锁。
2. `sync.Mutex` 主要用于保护共享资源的并发访问，确保在任何时刻只有一个 goroutine 可以访问共享资源。它用于实现临界区的互斥访问。`sync.Cond` 主要用于在 goroutine 之间进行**通信和同步**。它允许 goroutine 在某个条件满足时等待通知，以及在条件变量发生变化时发送通知给等待的 goroutine。
3. `sync.Mutex` 可以直接用于保护共享资源的临界区。`sync.Cond` 则需要和 `sync.Mutex` 结合使用。在使用 `sync.Cond` 时，通常会先获取一个 `sync.Mutex` 锁，然后使用该锁创建一个 `sync.Cond` 对象，并在 `sync.Cond` 上调用 `Wait`、`Signal` 和 `Broadcast` 方法。

实际上，sync.Cond 就是基于 sync.Mutex 实现的，也就是说 sync.Cond 能提供的能力，sync.Mutex 一定能实现。但是他们有什么区别呢，可以看下面的例子。

下面的例子中，我们存在两个问题：

1. **生产者和消费者同时访问queue**
2. **消费者需要等待生产者达到某个状态**

通过 sync.Mutex 我们可以很简单的实现 <1>。

但是如果要实现 <2> 我们就必须使用 `runtime_notifyListAdd` 和 `runtime_notifyListWait` 来挂起自身。

```go
func TestCond0(t *testing.T) {
	var mutex = sync.Mutex{}
	var cond = sync.NewCond(&mutex)
	var queue []int

	producer := func() {
		for i := 0; ; i++ {
			cond.L.Lock()
			queue = append(queue, i)
			cond.L.Unlock()
			cond.Signal()
			time.Sleep(100 * time.Millisecond)
		}
	}

	consumer := func(name string) {
		for {
			cond.L.Lock()
			// 第一次进入时，consumer 的 lock 和 producer 的 lock不能确定具体发生顺序
			// 但是我们可以确保通过 cond.Wait() 进入等待
			// 注意，这里必须是for循环去判断
			// 因为 cond.Wait() 中会先调用 cond.L.Unlock()
			// 也就是说，此时这段代码可能存在另外一个进程可以开始访问临界区
			// 举个例子
			// 1. consumer-0 Wait()，此时 consumer-0 释放锁，并且通过runtime_notifyListWait挂起自身
			// 2. producer 写入一条数据，但是尚未调用 Signal()
			// 3. consumer-1 判断 len(queue) != 0 消费掉 queue 中的数据并释放锁
			// 4. producer 调用 Signal() 唤醒 consumer-0
			// 5. consumer-0 此时操作 queue 就会导致空指针
			// 而如果当所有的进程都进入 Wait() 状态，那么此时反而不会有这个问题，
			// 因为每次 Signal() 只会启动一个协程
			for len(queue) == 0 {
				fmt.Printf("%s wait\n", name)
				cond.Wait()
			}
			val := queue[0]
			fmt.Printf("%s receive : %d\n", name, val)
			queue = queue[1:]
			cond.L.Unlock()
		}
	}

	go producer()

	go consumer("consumer-0")
	go consumer("consumer-1")
	go consumer("consumer-2")
	go consumer("consumer-3")
	go consumer("consumer-4")
	go consumer("consumer-5")
	go consumer("consumer-6")
	go consumer("consumer-7")
	go consumer("consumer-8")
	go consumer("consumer-9")

	time.Sleep(1 * time.Minute)
}
```

我们也可以自己在for循环里加锁后判断队列长度来实现，但是很明显这是一个忙等待，会有大量的CPU消耗在 `len(queue) == 0` 这个条件上。

而如果我们使用 `sync.Cond`，则只会在满足条件时唤醒某一个goroutine来操作。

```go
func TestCond1(t *testing.T) {
	var mutex = sync.Mutex{}
	var queue []int

	producer := func() {
		for i := 0; ; i++ {
			mutex.Lock()
			queue = append(queue, i)
			mutex.Unlock()
			time.Sleep(100 * time.Millisecond)
		}
	}

	consumer := func(name string) {
		for {
			mutex.Lock()

			if len(queue) != 0 {
				val := queue[0]
				fmt.Printf("%s receive : %d\n", name, val)
				queue = queue[1:]
			} else {
				fmt.Printf("%s : queue is empty\n", name)
			}

			mutex.Unlock()
		}
	}

	go producer()

	go consumer("consumer-0")
	go consumer("consumer-1")
	go consumer("consumer-2")
	go consumer("consumer-3")
	go consumer("consumer-4")
	go consumer("consumer-5")
	go consumer("consumer-6")
	go consumer("consumer-7")
	go consumer("consumer-8")
	go consumer("consumer-9")

	time.Sleep(1 * time.Minute)
}

```

### 使用sync.Pool管理对象池

```go
func TestPool(t *testing.T) {
	type User struct{ Id int }
	var autoIncrementId = 0
	pool := sync.Pool{New: func() any {
		autoIncrementId++
		return User{autoIncrementId}
	}}

	for i := 0; i < LoopSize; i++ {
		u := pool.Get().(User)
		pool.Put(u)
		// user 应该一直被复用
		assert.Equal(t, autoIncrementId, u.Id)
	}
}

```

### 使用sync.Once()实现只执行一次的操作

sync.Once() 的简单操作

```go
func TestOnce(t *testing.T) {
	once0 := sync.Once{}
	var sum = 0
	for i := 0; i < LoopSize; i++ {
		once0.Do(func() {
			sum++
		})
	}
	assert.Equal(t, sum, 1)

	once1 := sync.Once{}
	var sum2 = 0
	once1.Do(func() {
		sum2++
	})
	once1.Do(func() {
		sum2 += LoopSize
	})
	assert.Equal(t, 1, sum2)
}
```

也可以用来实现资源的初始化控制。

```go
func TestOnce1(t *testing.T) {
	type Resource struct {
		Id int
	}

	var id int
	once := sync.Once{}
	var resource *Resource
	initResource := func() {
		resource = &Resource{Id: id}
		id++
	}
	getResource := func() *Resource {
		once.Do(func() {
			initResource()
		})
		return resource
	}

	wg := sync.WaitGroup{}
	wg.Add(LoopSize)
	for i := 0; i < LoopSize; i++ {
		go func() {
			defer wg.Done()
			r := getResource()
			assert.Equal(t, r.Id, 0)
		}()
	}
	wg.Wait()
}
```

### 使用sync.Once()和context.Context()实现资源清理

```go
func TestOnceAndContext(t *testing.T) {
	once := sync.Once{}
	var val int
	cleanup := func() {
		val = LoopSize
	}

	doTask := func(ctx context.Context) {
		select {
		case <-ctx.Done():
			once.Do(cleanup)
		}
	}

	background := context.Background()
	timeout, cancelFunc := context.WithTimeout(background, 1*time.Second)
	defer cancelFunc()
	doTask(timeout)
	assert.Equal(t, val, LoopSize)
}
```

### 使用sync.Map实现并发安全的映射

```go
func TestSyncMap(t *testing.T) {
	syncMap := sync.Map{}
	valueStore := func(name string) {
		for i := 0; i < LoopSize; i++ {
			syncMap.Store(name+strconv.Itoa(i), "")
		}
	}

	wg := sync.WaitGroup{}
	wg.Add(LoopSize)
	for i := 0; i < LoopSize; i++ {
		go func(val int) {
			defer wg.Done()
			valueStore("setter-" + strconv.Itoa(val) + "-")
		}(i)
	}
	wg.Wait()

	var l int
	syncMap.Range(func(k, v any) bool {
		l++
		return true
	})
	assert.Equal(t, LoopSize*LoopSize, l)
}

```

### 使用context.Context进行协程管理和取消

> context.Context用于在协程之间传递上下文信息，并可用于取消或超时控制。可以使用context.WithCancel()创建一个可取消的上下文，并使用context.WithTimeout()创建一个带有超时的上下文。

### 使用context.WithDeadline()或者context.WithTimeout()设置截止时间

> - WithDeadline returns a copy of the parent context with the deadline adjusted to be no later than d.
> - WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).

### 使用context.WithValue()传递上下文

## 引用

[Golang异步编程方式和技巧](https://mp.weixin.qq.com/s/PuNu65ggHyB6jxRqhbN_VQ)

[sync.Cond 的使用场景](https://geektutu.com/post/hpg-sync-cond.html)

[Golang sync.Cond 条件变量源码分析](https://www.cyhone.com/articles/golang-sync-cond/)

