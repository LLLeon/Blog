# Tendermint ABCI 应用 KVStore 源码详解

**Tendermint abci** [项目主页](https://github.com/tendermint/abci)

这篇文章以 ABCI 示例 `KVStore` 应用及默认的 `socket` 连接为例说明 ABCI 应用的启动及 abci-cli 客户端与其交互的过程，以加深开发 ABCI 应用的模式及源码组织方式的理解。

## 整体流程说明

1. **ABCI 应用服务端**：在命令行执行 `abci-cli kvstore` 启动应用后，它会在 46658 端口等待客户端的 TCP 连接。
2. **abci-cli 客户端**：这个客户端指的是 `echo` 、`info` 和 `deliverTx` 等子命令。执行这些命令会建立一条与 ABCI 应用服务端的 TCP 连接，并将子命令后面的参数当作请求消息发送给应用服务端进行处理（这里的 ABCI 应用与 Tendermint 节点绑定在一起）。

## 构建命令过程

程序入口在 `abci/cmd/abci-cli/main.go` 的 `Execute` 函数。

这里面做事情有：

- 构建 `RootCmd` 命令，即 `abci-cli` 命令及各子命令。

- 注册全局 Flags，主要包括：
  - ABCI 应用服务端监听地址 `flagAddress` ，默认为：`tcp://0.0.0.0:46658`
  - abci-cli 客户端与 ABCI 应用服务端的通信协议 `flagAbci`，默认为：`socket`
  - 日志等级 `flagLogLevel`，默认为：`debug`
- 添加 ABCI 应用 `kvstore` 和 `dummy` 以及 `echoCmd`、`infoCmd`、`deliverTxCmd` 和 `commitCmd` 等客户端命令。

### RootCmd、kvstore 命令的实现及启动
#### RootCmd 主命令逻辑

所有子命令都要添加到 `RootCmd` 主命令下面。

执行 `abci-cli` 命令只会列出其使用文档，在执行具体子命令时才会执行其定义的相应应用逻辑。

这里只需看这段代码：

```go
// 执行主命令时如果 client 为空，会创建 client 并启动，会根据 flagAbci 参数来判断要创建 socket 客户端
// 还是 RPC 客户端
if client == nil {
			var err error
			client, err = abcicli.NewClient(flagAddress, flagAbci, false)
			if err != nil {
				return err
			}
			client.SetLogger(logger.With("module", "abci-client"))
    
    		// 启动客户端，实际执行的是 client.OnStart() 函数
			if err := client.Start(); err != nil {
				return err
			}
		}
```

#### abci-cli 客户端

在命令行执行 `abci-cli echo hello`，会与 ABCI 应用服务端建立一条 TCP 连接并将 “abc” 发送到服务端进行处理，收到应后断开连接。但这样比较麻烦，可以使用 `abci-cli console` 命令可以在交互式命令行中与应用服务端交互。

通过调用 `abcicli.NewClient` 函数来创建客户端。

返回的是接口 `abci/client/Client`，这个接口有 `socket`、`grpc` 和 `local` 三种实现，但 flag 只可以指定前两种，默认为 `socket`。

  ```go
  func NewClient(addr, transport string, mustConnect bool) (client Client, err error) {
  	switch transport {
  	case "socket":
  		client = NewSocketClient(addr, mustConnect)
  	case "grpc":
  		client = NewGRPCClient(addr, mustConnect)
  	default:
  		err = fmt.Errorf("Unknown abci transport %s", transport)
  	}
  	return
  }
  ```

`abci/client/Client` 接口继承了 `tmlibs/common/service/Service` 接口，可以启动、停止和重置。客户端和服务端都需要这些功能，使用时可以通过把 `BaseService` 作为自定义结构的匿名字段来实现。

首先看 `socketClient` 函数的结构，它实现了 `abci/client/Client` 接口，由于 `cmn.BaseService` 结构是它的匿名字段，也间接实现了 `tmlibs/common/service/Service` 接口：

```go
type socketClient struct {
    // 这个结构实现了 Service 接口，内部包含 impl Service字段，即具体实现 Service 的结构
	cmn.BaseService

    // 用来传递请求及应答消息的通道，
	reqQueue    chan *ReqRes
	flushTimer  *cmn.ThrottleTimer
	mustConnect bool

	mtx     sync.Mutex
	addr    string
	conn    net.Conn
	err     error
    
    // 标准库中的双向链表，从 reqQueue 读取到的请求消息都会先放入这个链表的尾端
	reqSent *list.List
	resCb   func(*types.Request, *types.Response) // listens to all callbacks
}
```

创建 socketClient 的函数：

```go
func NewSocketClient(addr string, mustConnect bool) *socketClient {
	cli := &socketClient{
		reqQueue:    make(chan *ReqRes, reqQueueSize),
		flushTimer:  cmn.NewThrottleTimer("socketClient", flushThrottleMS),
		mustConnect: mustConnect,

		addr:    addr,
		reqSent: list.New(),
		resCb:   nil,
	}
    
    // 这里传入了具体实现 Service 的结构 cli
	cli.BaseService = *cmn.NewBaseService(nil, "socketClient", cli)
	return cli
}
```

在命令行执行 `abci-cli kvstore` 命令时会执行 `client.Start()` 启动服务，这里用默认的 `socket` 连接及 `kvstore` 应用举例说明。这里执行的是 socketClient 结构中匿名字段 cmn.BaseService 的方法：

```go
func (bs *BaseService) Start() error {
	if atomic.CompareAndSwapUint32(&bs.started, 0, 1) {
		if atomic.LoadUint32(&bs.stopped) == 1 {
			bs.Logger.Error(Fmt("Not starting %v -- already stopped", bs.name), "impl", bs.impl)
			return ErrAlreadyStopped
		}
		bs.Logger.Info(Fmt("Starting %v", bs.name), "impl", bs.impl)
        
        // 这里实际执行上面传入的 cli（即 socketClient 结构） 的 OnStart 函数来启动服务
		err := bs.impl.OnStart()
		if err != nil {
			// revert flag
			atomic.StoreUint32(&bs.started, 0)
			return err
		}
		return nil
	}
	bs.Logger.Debug(Fmt("Not starting %v -- already started", bs.name), "impl", bs.impl)
	return ErrAlreadyStarted
}
```

以上就是客户端的启动过程。

#### ABCI 应用服务端

现在看一下执行 `abci-cli kvstore` 命令都做了什么。

执行此命令时，实际执行的是 `cmdKVStore` 函数，启动了应用服务端，在 `tcp://0.0.0.0:46658` 监听连接。

启动服务端：

```go
// 默认创建 socket 服务端
srv, err := server.NewServer(flagAddrD, flagAbci, app)
	if err != nil {
		return err
	}
	srv.SetLogger(logger.With("module", "abci-server"))
	if err := srv.Start(); err != nil {
		return err
	}
```

`NewSocketServer` 创建服务端：

```go
func NewSocketServer(protoAddr string, app types.Application) cmn.Service {
	proto, addr := cmn.ProtocolAndAddress(protoAddr)
	s := &SocketServer{
		proto:    proto,
		addr:     addr,
		listener: nil,
		app:      app,
		conns:    make(map[int]net.Conn),
	}
    // 这里使用的模式与 Client 段一致
	s.BaseService = *cmn.NewBaseService(nil, "ABCIServer", s)
	return s
}
```

执行 `srv.Start()` 函数时，实际执行的是 `SocketServer` 的实现，通过 `BaseService` 结构的 `Start` 方法调用：

```go
func (s *SocketServer) OnStart() error {
	if err := s.BaseService.OnStart(); err != nil {
		return err
	}
	ln, err := net.Listen(s.proto, s.addr)
	if err != nil {
		return err
	}
	s.listener = ln
    // 启动一个协程来监听连接
	go s.acceptConnectionsRoutine()
	return nil
}
```

至此已经把 ABCI 应用服务端是如何启动的说明了，下面的部分会详细说明请求及应答处理的细节。

## 请求及应答处理

这部分以 `deliver_tx` 命令为例来进行说明。

### abci-cli 客户端

为了方便，这里再看一下 `socketClient` 的数据结构：

```go
type socketClient struct {
	cmn.BaseService

	reqQueue    chan *ReqRes
	flushTimer  *cmn.ThrottleTimer
	mustConnect bool

	mtx     sync.Mutex
	addr    string
	conn    net.Conn
	err     error
    
    // 这里会把请求写入双向链表 reqSent 的尾端，在 recvResponseRoutine 函数中接收到应答时会从此链表取出
    // 第一个请求进行类型比较，如果与应答类型一样则返回给前端
	reqSent *list.List
	resCb   func(*types.Request, *types.Response) // listens to all callbacks

}
```

OnStart 函数所做的就是与服务端建立连接，启动两个协程来处理请求与应答。

先看处理请求的函数  `sendRequestsRoutine`：

```go
func (cli *socketClient) sendRequestsRoutine(conn net.Conn) {
	w := bufio.NewWriter(conn)
	for {
		select {
            // 发送 flush 类型请求的定时器
		case <-cli.flushTimer.Ch:
			select {
			case cli.reqQueue <- NewReqRes(types.ToRequestFlush()):
			default:
				// Probably will fill the buffer, or retry later.
			}
		case <-cli.Quit():
			return
		case reqres := <-cli.reqQueue:
             // 这里会把请求写入双向链表 reqSent 的尾端
			cli.willSendReq(reqres)
             // 把请求消息写入连接缓冲，这时还没有发送给服务端
			err := types.WriteMessage(reqres.Request, w)
			if err != nil {
				cli.StopForError(fmt.Errorf("Error writing msg: %v", err))
				return
			}
             // 如果请求是 flush 类型，会把缓冲的请求消息 (包括此 flush 请求) 写入连接，
             // 由 kvstore 服务端接收并处理。
             // 有两种方式发送 flush 类型的请求：1) 定时器触发；2) DeliverTxSync 函数中主动发送
			if _, ok := reqres.Request.Value.(*types.Request_Flush); ok {
				err = w.Flush()
				if err != nil {
					cli.StopForError(fmt.Errorf("Error flushing writer: %v", err))
					return
				}
			}
		}
	}
}
```

现在看处理应答的 `recvResponseRoutine` 函数：

```go
func (cli *socketClient) recvResponseRoutine(conn net.Conn) {

	r := bufio.NewReader(conn) // Buffer reads
	for {
		var res = &types.Response{}
        // 从连接中读取应答，出错时会关闭连接并执行 flushQueue() 释放 wg.WaitGroup
		err := types.ReadMessage(r, res)
		if err != nil {
			cli.StopForError(err)
			return
		}
		switch r := res.Value.(type) {
		case *types.Response_Exception:
			cli.StopForError(errors.New(r.Exception.Error))
			return
		default:
             // 应答处理逻辑在这里
			err := cli.didRecvResponse(res)
			if err != nil {
				cli.StopForError(err)
				return
			}
		}
	}
}

func (cli *socketClient) didRecvResponse(res *types.Response) error {
	cli.mtx.Lock()
	defer cli.mtx.Unlock()

    // 从双向链表 reqSent 中取出第一个请求
	next := cli.reqSent.Front()
	if next == nil {
		return fmt.Errorf("Unexpected result type %v when nothing expected", reflect.TypeOf(res.Value))
	}
	reqres := next.Value.(*ReqRes)
     // 检查请求与应答的类型是否匹配
	if !resMatchesReq(reqres.Request, res) {
		return fmt.Errorf("Unexpected result type %v when response to %v expected",
			reflect.TypeOf(res.Value), reflect.TypeOf(reqres.Request.Value))
	}

	reqres.Response = res    // Set response
    reqres.Done()            // 释放此请求创建时执行的 wg.Add(1)
	cli.reqSent.Remove(next) // 从链表中删除第一个请求

	// Notify reqRes listener if set
	if cb := reqres.GetCallback(); cb != nil {
		cb(res)
	}

	// Notify client listener if set
	if cli.resCb != nil {
		cli.resCb(reqres.Request, res)
	}

	return nil
}
```

### ABCI 应用服务端

**创建应用细节**：

先看应用的数据结构 `KVStoreApplication`：

```go
// 这个结构只实现了 Info、DeliverTx、CheckTx、Commit 和 Query 方法
type KVStoreApplication struct {
    // 这个基础结构实现了 "tendermint/abci/types/Application" 接口（此项目中基本都是用的这种模式）。
    // 这个结构实现的接口的方法中没有具体应用逻辑，以供开发者在自己的应用结构中继承此结构后，可以只实现
    // 必须的方法，而无需实现接口的全部方法
	types.BaseApplication

	state State
}
```

创建应用：

```go
func NewKVStoreApplication() *KVStoreApplication {
	state := loadState(dbm.NewMemDB())
	return &KVStoreApplication{state: state}
}
```

主要看 `loadState` 函数，它根据键 `stateKey` 从内存存储 `MemDB` 结构中获取对应状态，因为是初始化，肯定没有对应值，返回的是一个带有新建 `MemDB` （就是一个带锁的 map）的 `State`：

```go
func loadState(db dbm.DB) State {
	stateBytes := db.Get(stateKey)
	var state State
	if len(stateBytes) != 0 {
		err := json.Unmarshal(stateBytes, &state)
		if err != nil {
			panic(err)
		}
	}
	state.db = db
	return state
}
```
**服务端处理请求及应答细节**：

重点看接受连接的函数：

```go
func (s *SocketServer) acceptConnectionsRoutine() {
	for {
		// 接受连接，下面这些日志就是命令行启动 kvstore 后看到的信息
		s.Logger.Info("Waiting for new connection...")
		conn, err := s.listener.Accept()
		if err != nil {
			if !s.IsRunning() {
				return // Ignore error from listener closing.
			}
			s.Logger.Error("Failed to accept connection: " + err.Error())
			continue
		}

		s.Logger.Info("Accepted a new connection")

        // 可以接受多条连接并记录
		connID := s.addConn(conn)

		closeConn := make(chan error, 2)              // Push to signal connection closed
		responses := make(chan *types.Response, 1000) // A channel to buffer responses

         // 从连接读取请求并处理
		go s.handleRequests(closeConn, conn, responses)
		
         // 从 'responses' 获取应答并写到连接中
		go s.handleResponses(closeConn, conn, responses)

		// 等待信号来关闭连接
		go s.waitForClose(closeConn, connID)
	}
}
```

先看处理请求的 handleRequests 函数：

```go
func (s *SocketServer) handleRequests(closeConn chan error, conn net.Conn, responses chan<- *types.Response) {
	var count int
	var bufReader = bufio.NewReader(conn)
    
	for {
		var req = &types.Request{}
        // 从连接上读取请求消息，读取完毕或出错后要通知 waitForClose 协程来关闭连接
		err := types.ReadMessage(bufReader, req)
		if err != nil {
			if err == io.EOF {
				closeConn <- err
			} else {
				closeConn <- fmt.Errorf("Error reading message: %v", err.Error())
			}
			return
		}
		s.appMtx.Lock()
		count++
        // 处理请求时要加锁，这个函数会根据请求的类型调用具体函数来处理，
        // 比如 types.Request_DeliverTx 类型时就会调用 KVStoreApplication.DeliverTx 函数来处理，
        // 应答会写入 responses 通道，以便 handleResponses 函数处理
		s.handleRequest(req, responses)
		s.appMtx.Unlock()
	}
}
```

现在看 handleResponses 函数：

```go
func (s *SocketServer) handleResponses(closeConn chan error, conn net.Conn, responses <-chan *types.Response) {
	var count int
	var bufWriter = bufio.NewWriter(conn)
	for {
        // 从 responses 通道读取应答并写入连接。同样，出错时要通知 waitForClose 来关闭连接
		var res = <-responses
		err := types.WriteMessage(res, bufWriter)
		if err != nil {
			closeConn <- fmt.Errorf("Error writing message: %v", err.Error())
			return
		}
        
         // flush 类型的应答是哪里来的？
         // 与客户端处理类似，如果是此类型要进行 Flush 处理，把缓冲的数据写入连接
		if _, ok := res.Value.(*types.Response_Flush); ok {
			err = bufWriter.Flush()
			if err != nil {
				closeConn <- fmt.Errorf("Error flushing write buffer: %v", err.Error())
				return
			}
		}
		count++
	}
}
```

### deliver_tx 子命令执行过程

现在以此命令为例说明发起请求及收到应答的整体过程。

1. 服务端启动，启动两个协程，一个处理请求，一个处理应答。
2. 在命令行输入 `abci-cli console` 进入交互模式，创建客户端并与服务端建立了持久连接。服务端和客户端各启动两个协程，一个处理请求，一个处理应答。
3. 输入 `deliver_tx "abc"`，客户端会识别子命令，调用 `cmdDeliverTx` 函数，此函数在解析到 tx 后进行编码，随后会调用 `cli.DeliverTxSync(txBytes)`  (同步的) 函数。
4. 请求消息写入连接后，服务端读取到此请求。
5. 根据请求类型，调用 KVStore 应用的 `DeliverTx` 函数进行处理。
6. 服务端处理完毕后的应答写入连接，客户端接收到应答，前端打印显示到命令行。
