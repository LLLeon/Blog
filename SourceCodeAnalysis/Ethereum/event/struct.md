# 数据结构

## event

event 文件主要提供了 TypeMux 对某类型事件的订阅、投递。

### TypeMuxEvent

```go
// 表示某类型事件
type TypeMuxEvent struct {
	Time time.Time
	Data interface{} // 调用 Post 方法时根据这个字段区分事件类型，投递给订阅此类型事件的订阅者
}
```

### TypeMux

```go
// 事件管理的路由，通过 subm 字段将某类型事件及其订阅数组存储起来
type TypeMux struct {
	mutex   sync.RWMutex
	subm    map[reflect.Type][]*TypeMuxSubscription
	stopped bool
}
```

### TypeMuxSubscription

```go
// 某类型事件的具体订阅，通过 postC 和 readC 字段来发送、接收事件
type TypeMuxSubscription struct {
	mux     *TypeMux
	created time.Time
	closeMu sync.Mutex
	closing chan struct{}
	closed  bool

  	// 以下两个字段初始化时使用同一个 channel，
  	// 将 postC 设为 nil 不会影响从 readC 读取事件
	postMu sync.RWMutex
	readC  <-chan *TypeMuxEvent
	postC  chan<- *TypeMuxEvent
}
```



## subscription

### Subscription

```go
// 订阅接口
type Subscription interface {
	Err() <-chan error // 返回错误信息
	Unsubscribe()      // 取消事件发送，关闭错误通道
}
```

### funcSub

```go
// 实现了 Subscription 接口，给 NewSubscription 函数提供取消订阅通道
// NewSubscription 函数在新协程中运行一个生产者函数，返回一个订阅
// NewSubscription 函数在函数 Resubscribe 中使用
type funcSub struct {
	unsub        chan struct{}
	err          chan error
	mu           sync.Mutex
	unsubscribed bool
}
```

### ResubscribeFunc

```go
// 用来提供重新订阅的实际业务逻辑
type ResubscribeFunc func(context.Context) (Subscription, error)
```

### resubscribeSub

```go
// 重新订阅结构，供 Resubscribe 函数使用
// 主要函数：loop、subscribe
type resubscribeSub struct {
	fn                   ResubscribeFunc
	err                  chan error
	unsub                chan struct{}
	unsubOnce            sync.Once // 确保取消订阅只执行一次
	lastTry              mclock.AbsTime
	waitTime, backoffMax time.Duration
}
```

### SubscriptionScope

```go
// 用来追踪订阅，要追踪某个订阅时调用 Track 方法，会将该订阅添加到 subs 字段的 map 中
type SubscriptionScope struct {
	mu     sync.Mutex
	subs   map[*scopeSub]struct{}
	closed bool
}
```

### scopeSub

```go
// 这个结构中的 s 字段是具体要追踪的订阅
type scopeSub struct {
	sc *SubscriptionScope // 指向所属的 SubscriptionScope
	s  Subscription
}
```



## feed

### Feed

```go
// 一对多订阅，发送给 Feed 的事件将通过通道发送给订阅者，只能供一种类型的事件使用
type Feed struct {
	once      sync.Once        // 保证只初始化一次
	sendLock  chan struct{}    // Send 时持有此字段
	removeSub chan interface{} // 移除订阅：中断对此订阅的发送
	sendCases caseList         // 发送时从这个字段读取 case

	mu     sync.Mutex
	inbox  caseList // 暂时存放新添加的订阅，发送时会放到 sendCases 中
	etype  reflect.Type
	closed bool
}
```
### feedSub

```go
// 供 Feed 的 Subscribe 方法使用，它实现了 Subscription 接口，channel 字段是事件发送的通道
type feedSub struct {
	feed    *Feed
	channel reflect.Value
	errOnce sync.Once
	err     chan error
}
```

### caseList

```go
// SelectCase 实际存储订阅事件的 channel 及要在此 channel 发送的值
type caseList []reflect.SelectCase
```

