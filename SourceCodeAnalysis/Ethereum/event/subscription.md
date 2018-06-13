# subscription

## Subscription 接口

```go
// Subscription 表示一个事件流。事件的载体通常是一个 channel，但不是此接口的一部分。
// Subscription 在创建时可以失败。失败信息通过一个 error channel 记录。如果订阅有问题此 channel 接收一
// 个值。只有一个值将被发送。
// 如果订阅成功结束或取消订阅时，error channel 将关闭。
// Unsubscribe 方法取消发送事件。必须在所有实例中调用 Unsubscribe，以确保与此订阅有关的资源被释放。
type Subscription interface {
	Err() <-chan error // 返回 error channel
	Unsubscribe()      // 取消事件发送，关闭 error channel
}
```

### NewSubscription 函数

```go
// NewSubscription 在新协程中运行一个生产者函数来作为一个订阅。
// 当取消订阅时，提供给生产者函数的 channel 将关闭。
// 如果 fn 返回错误，将发送到该订阅的 error channel。
func NewSubscription(producer func(<-chan struct{}) error) Subscription {
	s := &funcSub{unsub: make(chan struct{}), err: make(chan error, 1)} // 注意 err 通道有缓冲
	go func() {
		defer close(s.err) // 协程退出前关闭 error channel
		err := producer(s.unsub) // 这里执行
		s.mu.Lock()
		defer s.mu.Unlock()
		if !s.unsubscribed { // 执行完毕生产任务后，如果没有取消订阅则标记为已取消订阅
			if err != nil {
				s.err <- err // 如果有错误信息则发送
			}
			s.unsubscribed = true
		}
	}()
	return s
}
```

## funcSub 结构

```go
type funcSub struct {
	unsub        chan struct{} // 发送取消订阅信号的 channel
	err          chan error
	mu           sync.Mutex // 无需初始化
	unsubscribed bool
}
```

### Unsubscribe 方法

```go
// 取消订阅，可以在传给 NewSubscription 的 producer 函数中调用（在单独的 goroutine 中）
func (s *funcSub) Unsubscribe() {
	s.mu.Lock()
	if s.unsubscribed {
		s.mu.Unlock()
		return
	}
	s.unsubscribed = true
	close(s.unsub)
	s.mu.Unlock()
  	// 等待执行生产任务的协程退出
	<-s.err // 不管 producer 有没有错误，当其完成后这里都会接收到值
}
```

### Err 方法

```go
// 用来读取错误信息
func (s *funcSub) Err() <-chan error {
	return s.err
}
```

### Resubscribe 函数

```go
// Resubscribe 重复调用 fn 来保持始终有一个订阅。
// 当订阅已建立，Resubscribe 等待它失败并再次调用 fn。
// 这个过程将一直重复直到调用 Unsubscribe 或活动的订阅成功结束。
// Resubscribe 应用指数退避算法来重复调用 fn。调用的间隔时间取决于错误率，但永远不会超过 backoffMax 值。
func Resubscribe(backoffMax time.Duration, fn ResubscribeFunc) Subscription {
	s := &resubscribeSub{
		waitTime:   backoffMax / 10,
		backoffMax: backoffMax,
		fn:         fn,
		err:        make(chan error),
		unsub:      make(chan struct{}),
	}
	go s.loop()
	return s
}
```

## ResubscribeFunc 类型的函数

```go
// ResubscribeFunc 用来尝试建立订阅。
type ResubscribeFunc func(context.Context) (Subscription, error)
```

## resubscribeSub 结构

```go
type resubscribeSub struct {
	fn                   ResubscribeFunc
	err                  chan error
	unsub                chan struct{}
	unsubOnce            sync.Once // 确保取消订阅只执行一次
	lastTry              mclock.AbsTime
	waitTime, backoffMax time.Duration
}
```

### Unsubscribe 和 Err 方法

```go
// 取消订阅
func (s *resubscribeSub) Unsubscribe() {
	s.unsubOnce.Do(func() {
		s.unsub <- struct{}{} // 发送取消订阅信号（注意：不是关闭此 channel）
		<-s.err // 等待 loop 方法执行完毕
	})
}

// 读取错误信息
func (s *resubscribeSub) Err() <-chan error {
	return s.err
}
```

### loop 方法

```go
// 循环执行订阅
func (s *resubscribeSub) loop() {
	defer close(s.err)
	var done bool
    // 一直订阅，直到没有错误或取消重新订阅
	for !done {
		sub := s.subscribe()
		if sub == nil { // 说明取消了重新订阅
			break
		}
         // sub 有错误时 done 为 false，继续循环
         // sub 没有错误或取消重新订阅后 done 为 true，停止循环
		done = s.waitForError(sub)
		sub.Unsubscribe() // 不管本次是否完成，都取消订阅
	}
}
```

### subscribe 方法

```go
// 执行重新订阅操作
func (s *resubscribeSub) subscribe() Subscription {
	subscribed := make(chan error)
	var sub Subscription
retry:
	for {
		s.lastTry = mclock.Now() // 设置最后重试时间为当前时间
		ctx, cancel := context.WithCancel(context.Background()) // 带有退出函数的 ctx
         // 在 goroutine 中执行订阅操作
		go func() {
			rsub, err := s.fn(ctx)
			sub = rsub
			subscribed <- err
		}()
		select {
		case err := <-subscribed:
			cancel() // 接收到订阅操作返回的 err 后，放弃 s.fn(ctx) 的订阅操作
			if err != nil {
                 // 订阅失败，backoffWait 中有定时器，阻塞的等待执行下次订阅
                 // 如果等待过程中取消了重新订阅，返回 nil
				if s.backoffWait() {
					return nil
				}
				continue retry
			}
			if sub == nil {
				panic("event: ResubscribeFunc returned nil subscription and no error")
			}
			return sub
		case <-s.unsub: // 取消了重新订阅
			cancel()
			return nil
		}
	}
}
```

### waitForError

```go
// 等待传入的订阅返回错误或取消重新订阅，有错误会返回 false，没有错误或取消重新订阅后返回 true
func (s *resubscribeSub) waitForError(sub Subscription) bool {
	defer sub.Unsubscribe() // 取消订阅传入的 sub
	select {
	case err := <-sub.Err():
		return err == nil
	case <-s.unsub: // 等待取消重新订阅的信号
		return true
	}
}
```

### backoffWait 方法

```go
// 计算重新订阅的间隔时间，如果到了重新订阅的时间，返回 false，如果在重试之前取消了重新订阅，返回 true
func (s *resubscribeSub) backoffWait() bool {
    // 根据最大间隔时间及上次重试间隔时间计算本次重试间隔时间，不得超过最大间隔时间
	if time.Duration(mclock.Now()-s.lastTry) > s.backoffMax {
		s.waitTime = s.backoffMax / 10
	} else {
		s.waitTime *= 2
		if s.waitTime > s.backoffMax {
			s.waitTime = s.backoffMax
		}
	}

	t := time.NewTimer(s.waitTime)
	defer t.Stop()
	select {
	case <-t.C: // 等待到达重试时间，调用此方法的地方执行重试操作
		return false
	case <-s.unsub: // 等待取消重新订阅的信号，调用此方法的地方停止重试
		return true
	}
}
```

## SubscriptionScope 和 scopeSub 结构

```go
// SubscriptionScope 提供了一次取消多个订阅的工具。
// 其零值可以直接使用，因为 Track 中对 subs 字段做了检查。
type SubscriptionScope struct {
	mu     sync.Mutex
	subs   map[*scopeSub]struct{} // 注意这种用法，只是为了存储 *scopeSub
	closed bool
}

type scopeSub struct {
	sc *SubscriptionScope // 指向它所属的 SubscriptionScope
	s  Subscription
}
```

### Track 方法

```go
// 把订阅 s 添加到 scope 中，如果 scope 已经关闭，Track 返回 nil。
// 返回的订阅是一个包含订阅 s 的 scopeSub ，取消订阅这个 scopeSub 将把它从 scope 中移除。
func (sc *SubscriptionScope) Track(s Subscription) Subscription {
	sc.mu.Lock()
	defer sc.mu.Unlock()
	if sc.closed {
		return nil
	}
	if sc.subs == nil {
		sc.subs = make(map[*scopeSub]struct{})
	}
	ss := &scopeSub{sc, s}
	sc.subs[ss] = struct{}{}
	return ss
}
```

### Close 方法

```go
// Close 对所有追踪的订阅执行取消订阅操作，并防止更多订阅添加到追踪集合。
// 执行 Close 后再执行 Track 将返回 nil。
func (sc *SubscriptionScope) Close() {
	sc.mu.Lock()
	defer sc.mu.Unlock()
	if sc.closed {
		return
	}
	sc.closed = true
	for s := range sc.subs {
		s.s.Unsubscribe() // 对集合中的订阅依次执行取消操作
	}
	sc.subs = nil // 将 map 清零
}
```

### Unsubscribe 方法

```go
func (s *scopeSub) Unsubscribe() {
	s.s.Unsubscribe()
	s.sc.mu.Lock()
	defer s.sc.mu.Unlock()
	delete(s.sc.subs, s) // 从追踪集合中移除 s
}
```

