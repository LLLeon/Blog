# manager

## Manager 结构

```go
// 账户管理器，可以与各种后端通信，用于交易签名。
type Manager struct {
	backends map[reflect.Type][]Backend // 当前注册的后端索引
	updaters []event.Subscription       // 对于所有后端的钱包升级订阅
	updates  chan WalletEvent           // 后端钱包变更的订阅接收器
	wallets  []Wallet                   // 从所有注册的后端缓存所有钱包

	feed event.Feed // 钱包到达或离开的一对多通知

	quit chan chan error
	lock sync.RWMutex
}
```

### NewManager 函数

```go
// 创建通用的账户管理器，通过各种支持的后端来签署交易。
func NewManager(backends ...Backend) *Manager {
  	// 所有后端订阅钱包到达或离开的通知
	updates := make(chan WalletEvent, 4*len(backends)) // 为什么要乘以 4？

	subs := make([]event.Subscription, len(backends))
	for i, backend := range backends {
		subs[i] = backend.Subscribe(updates)
	}
  
  	// 从后端取出原始钱包列表，并按 URL 排序
	var wallets []Wallet
	for _, backend := range backends {
		wallets = merge(wallets, backend.Wallets()...)
	}

  	// 创建账户管理器并返回
	am := &Manager{
		backends: make(map[reflect.Type][]Backend),
		updaters: subs,
		updates:  updates,
		wallets:  wallets,
		quit:     make(chan chan error),
	}
  	// 根据 backend 类型将其添加到相应类型的 backend 列表中
	for _, backend := range backends {
		kind := reflect.TypeOf(backend)
		am.backends[kind] = append(am.backends[kind], backend)
	}
  	// 钱包事件循环监听来自后台的通知，并更新钱包缓存。
	go am.update()

	return am
}
```

### Close 方法

```go
// 终止管理器的内部通知处理程序。
func (am *Manager) Close() error {
	errc := make(chan error)
	am.quit <- errc // 传递一个 channel
	return <-errc // 返回从 channel 中读取到的值
}
```

### update 方法

```go
// 钱包事件循环监听来自后台的通知，并更新钱包缓存。
func (am *Manager) update() {
  	// 当管理器关闭时，关闭所有订阅。
	defer func() {
		am.lock.Lock()
		for _, sub := range am.updaters {
			sub.Unsubscribe()
		}
		am.updaters = nil // 取消订阅后要将切片清零
		am.lock.Unlock()
	}()

  	// 循环直到关闭
	for {
		select {
		case event := <-am.updates:
          	 // 钱包事件到达，更新本地缓存
			am.lock.Lock()
			switch event.Kind {
			case WalletArrived:
				am.wallets = merge(am.wallets, event.Wallet)
			case WalletDropped:
				am.wallets = drop(am.wallets, event.Wallet)
			}
			am.lock.Unlock()

          	 // 通知此事件的监听者
			am.feed.Send(event)

		case errc := <-am.quit:
          	 // 管理器关闭，返回
			errc <- nil
			return
		}
	}
}
```

### Backends 方法

```go
// 从管理器中获取给定类型的后台。
func (am *Manager) Backends(kind reflect.Type) []Backend {
	return am.backends[kind]
}
```

### Wallets

```go
// 返回在该帐户管理器下注册的所有签名者帐户。
func (am *Manager) Wallets() []Wallet {
	am.lock.RLock()
	defer am.lock.RUnlock()

	cpy := make([]Wallet, len(am.wallets))
	copy(cpy, am.wallets)
	return cpy
}
```

### Wallet 方法

```go
// 获取与特定 URL 关联的钱包。
func (am *Manager) Wallet(url string) (Wallet, error) {
	am.lock.RLock()
	defer am.lock.RUnlock()

	parsed, err := parseURL(url)
	if err != nil {
		return nil, err
	}
  	// 遍历管理器下所有钱包与 url 进行比较
	for _, wallet := range am.Wallets() {
		if wallet.URL() == parsed {
			return wallet, nil
		}
	}
	return nil, ErrUnknownWallet

```

### Find 方法

```go
// 尝试找到特定账户对应的钱包。因为账户可以动态的添加到钱包和从钱包移除，
func (am *Manager) Find(account Account) (Wallet, error) {
	am.lock.RLock()
	defer am.lock.RUnlock()

	for _, wallet := range am.wallets {
		if wallet.Contains(account) { // 遍历查找包含此帐号的钱包
			return wallet, nil
		}
	}
	return nil, ErrUnknownAccount
}
```

### merge 方法

```go
// 分类的在切片中追加钱包，通过在正确的位置插入新的钱包来保存原始列表的排序。
// 原始切片假定已经按照 URL 进行了排序。
func merge(slice []Wallet, wallets ...Wallet) []Wallet {
  // 用二分查找对 wallets 中的钱包 URL 与切片中的 URL 比较，返回 func(i) 为 true 时最小的索引,
  // 如果没有找到则返回 len(slice)
	for _, wallet := range wallets {
		n := sort.Search(len(slice), func(i int) bool { return slice[i].URL().Cmp(wallet.URL()) >= 0 })
      // 在 [0, len(slice)) 中没有找到满足 func(i) 为 true 的索引，直接追加到切片后面
		if n == len(slice) {
			slice = append(slice, wallet)
			continue
		}
      	// 找到了则将 wallet 放到正确的位置
		slice = append(slice[:n], append([]Wallet{wallet}, slice[n:]...)...)
	}
	return slice
}
```

### drop 方法

```go
// 从已排序的缓存中查找钱包，并删除指定的钱包。
func drop(slice []Wallet, wallets ...Wallet) []Wallet {
	for _, wallet := range wallets {
		n := sort.Search(len(slice), func(i int) bool { return slice[i].URL().Cmp(wallet.URL()) >= 0 })
		if n == len(slice) {
          	 // 没找到钱包
			continue
		}
		slice = append(slice[:n], slice[n+1:]...) // 从切片中删除了找到的钱包
	}
	return slice
}
```

