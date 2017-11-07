# accounts

## Account 结构

```go
// 表示一个以太坊账户，由可选的 URL 字段来标志其位置。
type Account struct {
	Address common.Address `json:"address"`
	URL     URL            `json:"url"`
}
```

## Wallet 接口

```go
// 表示一个软件或硬件钱包，可能包含一个或多个账户。
type Wallet interface {
  	// 检索可访问钱包的路径。它是由上层用户从多个后端的所有钱包定义的排序顺序。
	URL() URL

  	// 获取钱包的当前状态。
	Status() (string, error)

  	// 初始化对钱包实例的访问。这不是解锁或解密账户密钥，而是简单地建立到硬件钱包的连接。
  	// passphrase 参数对特殊钱包的实例来说可以使用也可以不使用。
  	// 注意，打开钱包后，必须关闭它以释放相关资源（对硬件钱包来说尤其重要）。
	Open(passphrase string) error

  	// 释放由一个打开的钱包实例持有的任何资源。
	Close() error

  	// 检索钱包当前知道的签名账户列表。对于分层确定的钱包，此列表不全面，只包含在账户派生期间显式的标记的账户。
	Accounts() []Account

  	// 判断一个账户是否是此钱包的一部分。
	Contains(account Account) bool

  	// 尝试在指定的派生路径来显式的派生一个层次确定的账户。如果请求，派生的账户将被添加到钱包的追踪账户列表中。
	Derive(path DerivationPath, pin bool) (Account, error)

  	// 设置一个基本帐户派生路径，从该路径中，钱包试图发现非零帐户，并自动将其添加到跟踪帐户列表中。
	SelfDerive(base DerivationPath, chain ethereum.ChainStateReader)

  	// 请求钱包对给定哈希值进行签名。
	SignHash(account Account, hash []byte) ([]byte, error)

  	// 请求钱包对给定交易进行签名。
	SignTx(account Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)

  	// 请求钱包对给定的哈希值进行签名，与给定的密码一起来作为额外的身份验证信息。
	SignHashWithPassphrase(account Account, passphrase string, hash []byte) ([]byte, error)

  	// 请求钱包对给定的交易进行签名，与给定的密码一起作为额外的身份验证信息。
	SignTxWithPassphrase(account Account, passphrase string, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
}
```

## Backend 接口

```go
type Backend interface {
  	// 检索后端目前所知道的钱包列表。返回的钱包默认不是打开的。
	Wallets() []Wallet

  	// 创建一个异步订阅，接收当后端检测到钱包到达或离开的通知。
	Subscribe(sink chan<- WalletEvent) event.Subscription
}
```

## WalletEventType 类型

```go
// 表示可以通过钱包订阅子系统来发射的不同的事件类型。
type WalletEventType int
```

## WalletEvent 结构

```go
// 一个当账户后端检测到钱包到达或离开时发射的事件。
type WalletEvent struct {
	Wallet Wallet          // 钱包实例
	Kind   WalletEventType // 在此系统中发生的事件类型
}
```

