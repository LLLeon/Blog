# Observable

## Observable struct

```go
// 只是一个 interface 类型的 channel，用来向 observer 发射项目
type Observable <-chan interface{}
```
## Observable Operators

### Creating Observables

#### From

```go
// 从迭代器读取数据通过 Observable 发射，最终效果是将其它类型的数据转换为 Observable，从而可以与其它 
// Observables 交互
func From(it rx.Iterator) Observable {
	source := make(chan interface{})
	go func() {
		for {
			val, err := it.Next()
			if err != nil {
				break
			}
			source <- val
		}
		close(source)
	}()
	return Observable(source)
}
```

#### Empty

```go
// 创建一个空的 Observable
func Empty() Observable {
	source := make(chan interface{})
	go func() {
		close(source)
	}()
	return Observable(source)
}
```

#### Interval

```go
// 每隔指定时间间隔，就发送升序的整数序列，直到能够从 term 读取到数据时停止
func Interval(term chan struct{}, interval time.Duration) Observable {
	source := make(chan interface{})
	go func(term chan struct{}) {
		i := 0
	OuterLoop:
		for {
			select {
			case <-term: // 等待退出信号
				break OuterLoop
			case <-time.After(interval): // 阻塞，直到指定时间间隔后才发送整数给 source
				source <- i
			}
			i++
		}
		close(source)
	}(term)
	return Observable(source)
}
```

#### Repeat

```go
// 重复发射指定项目，可以选择是否指定重复发射的次数
func Repeat(item interface{}, ntimes ...int) Observable {
	source := make(chan interface{})
	
	// 未指定 ntimes 时，无限重复
	if len(ntimes) == 0 {
		go func() {
			for {
				source <- item
			}
			close(source)
		}()
		return Observable(source)
	}

	// 重复发射 n 次
	if len(ntimes) > 0 {
		count := ntimes[0]
		if count <= 0 {
			return Empty() // 如果次数小于等于 0，返回空的 Observable
		}
		go func() {
			for i := 0; i < count; i++ {
				source <- item
			}
			close(source)
		}()
		return Observable(source)
	}

	return Empty()
}
```

#### Range

```go
// 创建一个发射指定范围内的整数序列的 Observable
func Range(start, end int) Observable {
	source := make(chan interface{})
	go func() {
		i := start
		for i < end {
			source <- i
			i++
		}
		close(source)
	}()
	return Observable(source)
}
```

#### Just

```go
// 用来创建 Observable，传入的参数可以是任意类型的数据
func Just(item interface{}, items ...interface{}) Observable {
	source := make(chan interface{}) // make 一个没有缓冲的 channel
  
    // 判断传入参数 items 的数量，有传入则将其追加到存放 item 的 interface 切片
	if len(items) > 0 {
		items = append([]interface{}{item}, items...)
	} else {
		items = []interface{}{item}
	}

    // 启动一个协程，遍历传入的 items，将 item 放入 channel，
    // 因为 channel 无缓冲，里面的 item 被读取后才能放入下一个
	go func() {
		for _, item := range items {
			source <- item
		}
		close(source) // 全部 item 都放入 channel 后将其关闭，不影响读取里面的内容
	}()

	return Observable(source) // 强制类型转换
}
```
#### Start

```go
// 根据一个或多个类指令函数创建一个 Observable，并且异步的发射每次操作的返回值
func Start(f fx.EmittableFunc, fs ...fx.EmittableFunc) Observable {
	if len(fs) > 0 {
		fs = append([]fx.EmittableFunc{f}, fs...)
	} else {
		fs = []fx.EmittableFunc{f}
	}

	source := make(chan interface{})

	var wg sync.WaitGroup
	for _, f := range fs {
		wg.Add(1)
         // 启动协程来发射每次的执行结果
		go func(f fx.EmittableFunc) {
			source <- f()
			wg.Done()
		}(f)
	}

    // 非阻塞的等待所有 goroutine 完成
	go func() {
		wg.Wait()
		close(source)
	}()

	return Observable(source)
}
```



### Transforming Observables items

#### Map

```go
// 由 apply 函数对 Observable 中每个项目进行操作，然后将转换完毕后的项目通过新 Observable 发射
func (o Observable) Map(apply fx.MappableFunc) Observable {
    // make 一个无缓冲的 channel
	out := make(chan interface{})
  
	go func() {
		for item := range o {
			out <- apply(item)
		}
		close(out)
	}()
  
    // 由于上面开启了协程，马上返回新 Observable 供其它地方使用
	return Observable(out)
}
```
#### Scan

```go
// 读取原 Observable 中的项目，由 apply 函数处理后，再将当前处理结果与下个项目一起传给 apply 函数进行处理
func (o Observable) Scan(apply fx.ScannableFunc) Observable {
	out := make(chan interface{})
	go func() {
		var current interface{} // 用来记录当前处理结果
		for item := range o {
			out <- apply(current, item) // apply 函数处理后发给新的 Observable
			current = apply(current, item) // 记录当前处理结果
		}
		close(out)
	}()
	return Observable(out)
}
```


### Filtering Observables

#### Take

```go
// 获取前 n 个项目，通过新的 Observable 发射
func (o Observable) Take(nth uint) Observable {
	out := make(chan interface{})
	go func() {
		takeCount := 0
		for item := range o {
			if (takeCount < int(nth)) {
				takeCount += 1
				out <- item
				continue
			}
			break
		}
		close(out)
	}()
	return Observable(out)
}
```
#### TakeLast
```go
// 获取后 n 个项目，通过新的 Observable 发射
func (o Observable) TakeLast(nth uint) Observable {
	out := make(chan interface{})
	go func() {
		buf := make([]interface{}, nth)
         // 每次遍历先判断 buf 内元素数量，如果大于等于 n，丢弃第一个，append 操作将后一个元素追加到 buf
         // 最后达到获取后 n 个元素的效果
		for item := range o {
			if (len(buf) >= int(nth)) {
				buf = buf[1:]
			}
			buf = append(buf, item)
		}
		for _, takenItem := range buf {
			out <- takenItem
		}
		close(out)
	}()
	return Observable(out)
}
```
#### Filter
```go
// 用 apply 函数过滤原 Observable 中的元素，将过滤好的项目通过新的 Observable 发射
func (o Observable) Filter(apply fx.FilterableFunc) Observable {
	out := make(chan interface{})
	go func() {
		for item := range o {
			if apply(item) {
				out <- item
			}
		}
		close(out)
	}()
	return Observable(out)
}
```
#### First
```go
// 返回一个新的 Observable，仅发射第一个元素
func (o Observable) First() Observable {
	out := make(chan interface{})
	go func() {
		for item := range o {
			out <- item
			break
		}
		close(out)
	}()
	return Observable(out)
}
```
#### Last

```go
// 返回一个新的 Observable，仅发射最后一个元素的
func (o Observable) Last() Observable {
	out := make(chan interface{})
	go func() {
		var last interface{}
		for item := range o {
			last = item // 这里每次遍历都将值付给协程中的全局变量 last，一直到最后一个元素，遍历结束
		}
		out <- last
		close(out)
	}()
	return Observable(out)
}
```

#### Distinct

```go
// 用 apply 函数过滤原 Observable 中的重复项目，将结果通过新 Observable 发射，只允许没有发射过的元素通过
func (o Observable) Distinct(apply fx.KeySelectorFunc) Observable {
	out := make(chan interface{})
	go func() {
		keysets := make(map[interface{}]struct{})
		for item := range o {
			key := apply(item) // 对元素执行 apply 函数
             // 将 keysets 中没有的元素放进 out，下面在 map 中以此元素为 key 赋值，这样再遍历到相同的
             // 元素时，就不会在放进 out，达到过滤重复元素的效果
			_, ok := keysets[key]
			if !ok {
				out <- item
			}
             // map 里面存在的元素肯定是已经放进过 out 的
			keysets[key] = struct{}{} // 这里 value 的格式及内容不重要，只是起到一个赋值的作用
		}
		close(out)
	}()
	return Observable(out)
}
```

#### DistinctUntilChanged

```go
// DistinctUntilChanged 是 Distinct 函数的一个变体，只过滤连续的重复项，将结果通过新 Observable 发射
func (o Observable) DistinctUntilChanged(apply fx.KeySelectorFunc) Observable {
	out := make(chan interface{})
	go func() {
		var current interface{} // current 用来记录当前项目经 apply 函数处理后的值
		for item := range o {
			key := apply(item)
			if current != key { // 当前项目仅与上一个项目进行比较，连续的重复项目不执行任何操作
				out <- item
				current = key
			}
		}
		close(out)
	}()
	return Observable(out)
}
```

#### Skip

```go
// 跳过原 Observable 的前 n 个项目，只通过新的 Observable 发射其余的项目
func (o Observable) Skip(nth uint) Observable {
	out := make(chan interface{})
	go func() {
		skipCount := 0 // 最后会等于 n
		for item := range o {
			if (skipCount < int(nth)) { // 这里实现了跳过前 n 个项目的效果
				skipCount += 1
				continue
			}
			out <- item
		}
		close(out)
	}()
	return Observable(out)
}
```

#### SkipLast

```go
// 忽略原 Observable 的后 n 个项目，只通过新的 Observable 发射前面的项目
func (o Observable) SkipLast(nth uint) Observable {
	out := make(chan interface{})
	go func() {
		buf := make(chan interface{}, nth) // 只能缓冲 n 个项目
		for item := range o {
			select {
				case buf <- item: // 从第一个项目开始往 buf 发送，放满 n 个后进入 default 分支
				default:
					out <- (<- buf) // 开始从 buf 读取项目并发往 out
					buf <- item // out 中的项目被读取后，下一个项目继续发往 buf，如此循环直到遍历完毕
			}
		}
		close(buf)
		close(out)
	}()
	return Observable(out)
}
```



### Utility Operators

#### Subscribe

```go
// 向 observable 注册 EventHandler，即订阅该 observable，返回一个 channel，用于其它地方监听项目处理完毕
//  的信号。handlers 和 Observer 实现了 EventHandler 接口
func (o Observable) Subscribe(handler rx.EventHandler) <-chan subscription.Subscription {
	done := make(chan subscription.Subscription)
	sub := subscription.New().Subscribe()

    // 这里检查 handler 类型并将其赋值给 Observer 对应的字段，如果是 Observer 类型则直接赋值给 ob
    // 由 Observer 中相应的 handler 对数据进行处理
	ob := CheckEventHandler(handler)

	go func() {
	OuterLoop:
		for item := range o {
			switch item := item.(type) { // item 是 interface，可以用此方式来判断类型并获取值
			case error:
				ob.OnError(item)

				// 记录错误并跳出循环
				sub.Error = item
				break OuterLoop
			default:
				ob.OnNext(item) // 处理项目
			}
		}

		// 每次只会执行 OnDone 和 OnError 中的一个
		if sub.Error == nil {
			ob.OnDone()
		}

      // 正常执行完毕后当前记录时间到 sub 并将该 Subscription 通过 done channel 发送，其它地方接收
      // 到取消订阅信号后，此协程返回
		done <- sub.Unsubscribe()
		return
	}()

	return done
}
```


### Others

#### Next

```go
// observer 主动从 observable 读取数据时使用此方法
func (o Observable) Next() (interface{}, error) {
	if next, ok := <-o; ok { // 如果有数据则返回
		return next, nil
	}
	return nil, errors.New(errors.EndOfIteratorError)
}
```
#### CheckEventHandler

```go
// 检查事件处理函数的类型
func CheckEventHandler(handler rx.EventHandler) observer.Observer {
	ob := observer.DefaultObserver
	switch handler := handler.(type) {
	case handlers.NextFunc:
		ob.NextHandler = handler
	case handlers.ErrFunc:
		ob.ErrHandler = handler
	case handlers.DoneFunc:
		ob.DoneHandler = handler
	case observer.Observer:
		ob = handler
	}
	return ob
}
```

