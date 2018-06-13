# event

## 简介

1. 提供订阅某种类型的事件，向订阅者发送事件等功能。
2. 通过 TypeMuxEvent.Data 的类型来区分不同类型事件。

## TypeMuxEvent 结构

```go
// 推送给订阅者的带有时间标记的事件
type TypeMuxEvent struct {
	Time time.Time
	Data interface{} // Post 时根据这个字段来区分事件类型
}
```

## TypeMux 结构

```go
// TypeMux 是事件分发路由，接收者来处理某类型的事件，可以被注册到这里。
// 此结构的零值可以直接使用。
type TypeMux struct {
	mutex   sync.RWMutex
	subm    map[reflect.Type][]*TypeMuxSubscription // TypeMux 订阅的事件类型及其对应的订阅
	stopped bool
}
```

### Subscribe 方法

```go
// Subscribe 为给定类型的事件创建一个订阅。
// 当取消订阅或 mux 关闭时，该订阅的 channel 也关闭。
func (mux *TypeMux) Subscribe(types ...interface{}) *TypeMuxSubscription {
	sub := newsub(mux)
	mux.mutex.Lock()
	defer mux.mutex.Unlock()
	if mux.stopped {
         // mux 停止时，将 sub 的状态设置为关闭，其后的取消订阅操作将被短路
		sub.closed = true
		close(sub.postC)
	} else {
         // 检查 subm 是否初始化
		if mux.subm == nil {
			mux.subm = make(map[reflect.Type][]*TypeMuxSubscription)
		}
         // mux 不能重复订阅同一类型的事件
		for _, t := range types {
			rtyp := reflect.TypeOf(t)
			oldsubs := mux.subm[rtyp] // 从 map 中获取订阅该类型的数组
             // 如果订阅该类型的数组中已经有 sub，说明 mux 已经订阅过该类型，直接 panic
			if find(oldsubs, sub) != -1 {
				panic(fmt.Sprintf("event: duplicate type %s in Subscribe", rtyp))
			}
             // 如果 mux 之前没有订阅当前类型，则将 sub 添加到订阅该类型的数组中
			subs := make([]*TypeMuxSubscription, len(oldsubs)+1) // 复制到比原 slice 长度大的 slice 的用法
			copy(subs, oldsubs)
			subs[len(oldsubs)] = sub
			mux.subm[rtyp] = subs
		}
	}
    // 即使 mux 停止后也返回 sub
	return sub
}
```

### Post 方法

```go
// 向订阅某类型的所有接收者发送一个此类型事件。
// 如果 mux 已经停止，返回 ErrMuxClosed 错误。
func (mux *TypeMux) Post(ev interface{}) error {
    // 创建事件
	event := &TypeMuxEvent{
		Time: time.Now(),
		Data: ev,
	}
	rtyp := reflect.TypeOf(ev) // 获取该事件的类型，以便向订阅此类型的接收者发送事件
	mux.mutex.RLock()
	if mux.stopped { // 检测 mux 是否关闭
		mux.mutex.RUnlock()
		return ErrMuxClosed
	}
	subs := mux.subm[rtyp] // 获取订阅此类型事件的数组
	mux.mutex.RUnlock()
	for _, sub := range subs {
		sub.deliver(event) // 向所有接收者投递该事件
	}
	return nil
}
```

### Stop 方法

```go
// 关闭 mux， 此 mux 不能再被使用。随后的 Post 调用将失败，返回 ErrMuxClosed。
// Stop 将阻塞，直到所有当前的投递都完成。
func (mux *TypeMux) Stop() {
	mux.mutex.Lock()
	for _, subs := range mux.subm {
		for _, sub := range subs {
			sub.closewait() // 关闭每个订阅，相关内容清零
		}
	}
	mux.subm = nil // 关闭每个订阅后将订阅 map 清空
	mux.stopped = true
	mux.mutex.Unlock()
}
```

### del 方法

```go
// 从 mux 的订阅 map 中删除某订阅
func (mux *TypeMux) del(s *TypeMuxSubscription) {
	mux.mutex.Lock()
	for typ, subs := range mux.subm {
		if pos := find(subs, s); pos >= 0 { // 如果 s 在订阅数组 subs 中
			if len(subs) == 1 { // 如果订阅数组中只有 s 一个订阅，直接删除 typ 类型的订阅
				delete(mux.subm, typ)
			} else {
                 // 如果不是只有一个订阅，将该订阅从数组中删除
				mux.subm[typ] = posdelete(subs, pos)
			}
		}
	}
	s.mux.mutex.Unlock()
}
```

### find 函数

```go
// 判断订阅 item 是否在 slice 数组中，如果在则返回其索引，不在则返回 -1
func find(slice []*TypeMuxSubscription, item *TypeMuxSubscription) int {
	for i, v := range slice {
		if v == item {
			return i
		}
	}
	return -1
}
```

### posdelete 函数

```go
// 从订阅数组 slice 中删除索引为 pos 的元素
func posdelete(slice []*TypeMuxSubscription, pos int) []*TypeMuxSubscription {
	news := make([]*TypeMuxSubscription, len(slice)-1)
	copy(news[:pos], slice[:pos]) // 将 pos 位置之前的元素复制到新数组中
	copy(news[pos:], slice[pos+1:]) // 将 pos 位置之后的元素复制到新数组中
	return news
}
```



## TypeMuxSubscription 结构

```go
// 通过 TypeMux 建立的订阅
type TypeMuxSubscription struct {
	mux     *TypeMux // 指示此订阅属于哪个 mux
	created time.Time
	closeMu sync.Mutex // 关闭
	closing chan struct{}
	closed  bool

    // readC 和 postC 是同一个 channel。
    // 它们分两个字段存储，所以 postC 可以设置为 nil 而不影响 Chan 方法从 readC 读取值。
	postMu sync.RWMutex
	readC  <-chan *TypeMuxEvent
	postC  chan<- *TypeMuxEvent
}
```

### newsub 函数

```go
// 新建 mux 类型的订阅
func newsub(mux *TypeMux) *TypeMuxSubscription {
    // channel 声明时可以指定方向，增加代码可读性，make 时可以是双向的
	c := make(chan *TypeMuxEvent)
	return &TypeMuxSubscription{
		mux:     mux,
		created: time.Now(),
		readC:   c,
		postC:   c,
		closing: make(chan struct{}),
	}
}
```

### Chan 方法

```go
// 返回 s.readC channel，读取事件
func (s *TypeMuxSubscription) Chan() <-chan *TypeMuxEvent {
	return s.readC
}
```

### Unsubscribe 方法

```go
// 将 s 从 mux 的订阅 map 中删除，即 mux 取消订阅 s
func (s *TypeMuxSubscription) Unsubscribe() {
	s.mux.del(s)
	s.closewait()
}
```

### closewait 方法

```go
func (s *TypeMuxSubscription) closewait() {
	s.closeMu.Lock()
	defer s.closeMu.Unlock()
	if s.closed {
		return
	}
	close(s.closing) // 发送关闭信号
	s.closed = true

  	s.postMu.Lock() // 等待 deliver 释放读锁后才能加锁
	close(s.postC) // 停止投递新事件
	s.postC = nil // 归零
	s.postMu.Unlock()
}
```

### deliver 方法

```go
// 向接收者投递事件
func (s *TypeMuxSubscription) deliver(event *TypeMuxEvent) {
    // 如果订阅发生在事件创建之后，则不向该订阅投递此事件
	if s.created.After(event.Time) {
		return
	}
    // 否则就投递该事件
	s.postMu.RLock() // 加读锁，closewait 方法要等待这里完成后才能关闭
	defer s.postMu.RUnlock()

	select {
	case s.postC <- event: // 阻塞的投递事件
	case <-s.closing: // 等待订阅关闭的信号
	}
}
```

