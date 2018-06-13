# feed

## Feed 结构

```go
// Feed 实现了一对多的订阅，事件的载体是一个 channel。
// 发送到 Feed 的值将同时投递到订阅的 channel。
// Feed 中仅可持有一个类型，类型由第一个 Send 或 Subscribe 操作决定。
// 随后对这些方法的调用如果类型不匹配，将 panic。
// 此结构无需初始化即可使用。
type Feed struct {
	once      sync.Once        // 保证初始化只运行一次
  
  	// sendLock 的缓冲为 1，持有它时它为空。确保每次只有一个 Send 能对 sendCases 进行操作
	sendLock  chan struct{} 
	removeSub chan interface{} // 中断 Send 操作
  	sendCases caseList         // 由 Send 方法使用的活动的选择案例集合

	mu     sync.Mutex
	inbox  caseList // 持有新订阅的 channel，发送时会添加到 sendCases
	etype  reflect.Type // element type，channel 中元素的类型
	closed bool
}

// 这是 sendCases 中第一个实际订阅 channel 的索引。
// sendCases[0] 是一个 SelectRecv case，用于 removeSub。
const firstSubSendCase = 1

// 初始化
func (f *Feed) init() {
	f.removeSub = make(chan interface{})
	f.sendLock = make(chan struct{}, 1)
	f.sendLock <- struct{}{} // 要 Send 时先从它接收，Send 完毕后向它发送以阻塞再次发送
  	// 初始化时将 sendCases 中第一个 case 设置为移除订阅用的 chnnel，被移除的 case 会发送到这个 channel
	f.sendCases = caseList{{Chan: reflect.ValueOf(f.removeSub), Dir: reflect.SelectRecv}}
}
```

### Subscribe 方法

```go
// Subscribe 将一个 channel 添加到 Feed 中。未来的发送将通过 channel 来投递，直到取消订阅。
// 所有添加的 channel 必须具有一样的元素类型。
// channel 必须具有足够的缓冲空间，避免阻塞其它的订阅者。
// TODO:  Slow subscribers are not dropped.
func (f *Feed) Subscribe(channel interface{}) Subscription {
	f.once.Do(f.init)
     // 检查 channel 类型，如果不是 Chan 或 Chan 不是发送类型则 panic
	chanval := reflect.ValueOf(channel)
	chantyp := chanval.Type()
	if chantyp.Kind() != reflect.Chan || chantyp.ChanDir()&reflect.SendDir == 0 {
		panic(errBadChannel)
	}
	sub := &feedSub{feed: f, channel: chanval, err: make(chan error, 1)}

	f.mu.Lock()
	defer f.mu.Unlock()
  	// 检查 channel 中元素的类型，元素类型前后不一致将 panic
	if !f.typecheck(chantyp.Elem()) {
		panic(feedTypeError{op: "Subscribe", got: chantyp, want: reflect.ChanOf(reflect.SendDir, f.etype)})
	}
  	// 将此 case 添加到 inbox。
  	// 下一次调用 Send 时将把 inbox 中的 case 添加到 f.sendCases。
	cas := reflect.SelectCase{Dir: reflect.SelectSend, Chan: chanval}
	f.inbox = append(f.inbox, cas) 
	return sub
}
```

### typecheck 方法

```go
// 检查 channel 中元素是否与前面调用者的一致。
// 调用者必须先对 f.mu 加锁
func (f *Feed) typecheck(typ reflect.Type) bool {
	if f.etype == nil { // 表示是第一次调用，这时类型还没有值
		f.etype = typ
		return true
	}
	return f.etype == typ // 如果类型不一致，调用者将 panic
}
```

### remove 方法

```go
// 从 Feed 中移除 sub
func (f *Feed) remove(sub *feedSub) {
  	// 首先从 inbox 中移除，可能涉及还没有添加到 f.sendCases 的 channel。
  	ch := sub.channel.Interface() // TODO: 以 interface{} 的形式返回 sub.channel 的当前值
	f.mu.Lock()
	index := f.inbox.find(ch) // ch 在 inbox 中的索引
	if index != -1 { // 表示 ch 在 inbox 中，移除它
		f.inbox = f.inbox.delete(index)
		f.mu.Unlock()
		return
	}
	f.mu.Unlock()

	select {
	case f.removeSub <- ch:
		// 有 Send 在执行时，将从 f.sendCases 中移除此 channel
	case <-f.sendLock:
      	// 没有 select 到第一个 case，表示当前没有 Send 操作，需要在这里移除 ch，
      	// 现在持有发送锁，可以移除此 channel。
		f.sendCases = f.sendCases.delete(f.sendCases.find(ch))
		f.sendLock <- struct{}{} // 发送一个值来阻塞 sendLock
	}
}
```

### Send 方法

```go
// Send 同时投递给所有订阅的 channel，返回发送给的订阅者的数量。
func (f *Feed) Send(value interface{}) (nsent int) {
	f.once.Do(f.init)
	<-f.sendLock // 对发送加锁

  	// 获取发送锁后，将 inbox 中的 channel 添加到 sendCases
	f.mu.Lock()
	f.sendCases = append(f.sendCases, f.inbox...)
	f.inbox = nil // inbox 清零
	f.mu.Unlock()

  	// 在所有 channel 上设置发送的值。如果类型与前面的类型不一致， sendLock 加锁后 panic
	rvalue := reflect.ValueOf(value)
	if !f.typecheck(rvalue.Type()) {
		f.sendLock <- struct{}{}
		panic(feedTypeError{op: "Send", got: rvalue.Type(), want: f.etype})
	}
	for i := firstSubSendCase; i < len(f.sendCases); i++ {
		f.sendCases[i].Send = rvalue // 依次为每个 channel 设置要发送的值，给下面的 Select 操作使用
	}

  	
	cases := f.sendCases // f.sendCases 和 cases 是两个 slice，但底层引用的是同一个数组
  	// 对 sendCases 中除 removeSub 外的所有 channel 执行 Send
	for {
      	// 快速路径：在添加到 select 集合之前尝试非阻塞的发送。
      	// 如果订阅者处理的足够快，并且有足够的缓冲空间的话，通常会成功。
		for i := firstSubSendCase; i < len(cases); i++ {
          	 // 在 Chan 上发送 rvalue，不阻塞，发送失败后继续尝试发送。
          	 // 如果一直失败，就会往下执行 Select 操作。
			if cases[i].Chan.TrySend(rvalue) {
				nsent++ // 发送成功后计数器加 1
				cases = cases.deactivate(i) // 将成功发送的 case 移动到不可访问的部分，避免重复发送
				i-- // 因为上一行已经将发送过的 case "屏蔽" 了，让 i 一直都等于 1，避免漏掉 case
			}
		}
      	// 说明 cases 中除 removeSub 外都成功发送，停止循环
		if len(cases) == firstSubSendCase {
			break
		}
      	// 在 cases 中所有的 channel 执行发送操作，
      	// 一直阻塞直到当至少有一个 case 可处理时，伪随机的选择一个 case 执行。
      	// chosen, recv: 返回选中的 case 的索引以及从该 case 接收的值。
		chosen, recv, _ := reflect.Select(cases) // 这个函数在处理一个 case 前是阻塞的
		if chosen == 0 { // recv 是从 <-f.removeSub 读取的值，表示有接收者取消了订阅，要取消对它的发送
          	 index := f.sendCases.find(recv.Interface()) // 是在 sendCases 中的索引
			f.sendCases = f.sendCases.delete(index) // 从 sendCases 中移除此 case
          	 if index >= 0 && index < len(cases) {
               	 // 两个 slice 的底层存储一样，只是长度不一样，因为 sendCases 长度少了 1，
               	 // 所以 cases 的长度也要减去 1。
				cases = f.sendCases[:len(cases)-1]
			}
		} else {
			cases = cases.deactivate(chosen) // 成功发送后将该元素 "屏蔽"
			nsent++
		}
	}

  	// 把发送过的值清零，释放发送锁
	for i := firstSubSendCase; i < len(f.sendCases); i++ {
		f.sendCases[i].Send = reflect.Value{}
	}
	f.sendLock <- struct{}{}
	return nsent
}
```

## feedSub 结构

```go
// 实现了 Subscription 接口
type feedSub struct {
	feed    *Feed // 指向其所属的 Feed
	channel reflect.Value
	errOnce sync.Once
	err     chan error
}
```

### Unsubscribe 方法

```go
func (sub *feedSub) Unsubscribe() {
	sub.errOnce.Do(func() {
		sub.feed.remove(sub) // 从其所属的 Feed 中移除自己
		close(sub.err)
	})
}
```

## caseList 类型

```go
// 存储 case 的数组
type caseList []reflect.SelectCase
```

### delete 方法

```go
// 从 cs 中移除给定的 case
func (cs caseList) delete(index int) caseList {
	return append(cs[:index], cs[index+1:]...)
}
```

### deactivate 方法

```go
// 将 index 位置的元素移动到 slice 最后的位置并将长度减 1，容量没变
func (cs caseList) deactivate(index int) caseList {
	last := len(cs) - 1
	cs[index], cs[last] = cs[last], cs[index] // 将其放到最后的位置
	return cs[:last] // 使最后位置的元素不可访问
}
```

