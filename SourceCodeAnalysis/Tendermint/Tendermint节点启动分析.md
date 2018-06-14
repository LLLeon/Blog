# Tendermint 节点启动源码分析

本文以官方示例公链 Basecoin 的 `basecoind start` 命令为入口，结合日志与源码分析 Tendermint 节点创建、启动及生成区块等过程。

文中代码细节较多，但限于篇幅没有太深入某些细节，比如共识过程。如只想快速了解整体过程，可只阅读有 `basecoind start` **日志**部分的内容。

## start 命令入口

执行 `basecoind init` 命令初始化 genesis 配置、priv-validator 文件及 p2p-node 文件后，在命令行执行 `basecoind start` 启动节点。在不带有任何参数时，Tendermint 会与 ABCI 应用一起执行：

**日志**：`Starting ABCI with Tendermint    module=main`

```go
func(cmd *cobra.Command, args []string) error {
			if !viper.GetBool(flagWithTendermint) {
				ctx.Logger.Info("Starting ABCI without Tendermint")
				return startStandAlone(ctx, appCreator)
			}
			ctx.Logger.Info("Starting ABCI with Tendermint")
			return startInProcess(ctx, appCreator)
		}
```

## 创建节点

### 创建三条连接的客户端

`startInProcess` 会创建 Tendermint 节点，然后启动节点。

创建节点时 Tendermint 与 ABCI 应用会创建建立三条连接所需的客户端：即 `query`、`mempool` 以及 `consensus` 连接。

**注意：**所有要启动服务都是通过 `BaseService.Start` 来执行的。

**日志**：`Starting multiAppConn    module=proxy impl=multiAppConn`

 `NewNode()` 中相关代码：

```go
proxyApp := proxy.NewAppConns(clientCreator, handshaker)

// 实际执行的是 multiAppConn.multiAppConn()
if err := proxyApp.Start(); err != nil {
		return nil, fmt.Errorf("Error starting proxy app connections: %v", err)
	}
```

三条连接的客户端的建立在 `multiAppConn.OnStart` 方法中完成，在这里三条连接的客户端都是 `localClient` 结构，由于 `localClient` 没有实现 `OnStart` 方法，它们的 `Start` 方法调用的都是 `cmn.BaseService` 的 `OnStart` 方法，也就是 `querycli.Start()` （另两个也一样）除了打印日志外，什么都不做。

接下来分别把 `localClient` 封装成了 `appConnQuery` 、`appConnMempool` 和 `appConnConsensus` 结构。

**日志**：`Starting localClient    module=abci-client connection=query impl=localClient`

`multiAppConn.OnStart()` 中相关代码：

```go
// 在创建节点时传入的 clientCreator 的具体实现是 localClientCreator 结构，
// 这里最终返回的是 localClient 结构
querycli, err := app.clientCreator.NewABCIClient()

// 执行的是 cmn.BaseService 的 OnStart 方法，啥都没做
if err := querycli.Start(); err != nil {
		return errors.Wrap(err, "Error starting ABCI client (query connection)")
	}
```

其它两个连接的客户端与 `query` 连接客户端的创建相同。

### 握手同步

在建立完毕以上三条连接的客户端后，会执行 `app.handshaker.Handshake(app)` 来握手，确保 Tendermint 节点与应用程序的状态是同步的。

**日志**：

```bash
ABCI Handshake                               module=consensus appHeight=169313 appHash=DC1ED303D0D1EE403CC010B911D7A991E3EAE7E3

ABCI Replay Blocks                           module=consensus appHeight=169313 storeHeight=169313 stateHeight=169313

Completed ABCI Handshake - Tendermint and App are synced module=consensus appHeight=169313 appHash=DC1ED303D0D1EE403CC010B911D7A991E3EAE7E3
```

在 `query` 连接上通过 ABCI Info 查询 ABCI 应用的 blockstore 中的最新状态，然后将 Tendermint 节点的区块重放到此状态：

`multiAppConn.OnStart()` 中相关代码：

```go
if app.handshaker != nil {
   return app.handshaker.Handshake(app)
}
```

`Handshaker.Handshake()` 中相关代码：

```go
// 从 ABCI 应用的 blockstore 中获取最新状态
res, err := proxyApp.Query().InfoSync(abci.RequestInfo{version.Version})

// 重放所有区块到最新状态
_, err = h.ReplayBlocks(h.initialState, appHash, blockHeight, proxyApp)
```

重放完毕后，Tendermint 节点已经和 ABCI 应用同步到了相同的区块高度。

### 快速同步设置及验证人节点确认

在进入代码细节之前，先了解一下快速同步的概念。

**快速同步（FastSync）**：在当前节点落后于区块链的最新状态时，需要进行节点间同步，快速同步只下载区块并检查验证人的默克尔树，比运行实时一致性八卦协议快得多。一旦追上其它节点的状态，守护进程将切换出快速同步并进入正常共识模式。在运行一段时间后，如果此节点至少有一个 peer，并且其区块高度至少与最大的报告的 peer 高度一样高，则认为该节点已追上区块链最新状态（即 `caught up`）。

现在回到 `NewNode()` 源码，当节点同步到 ABCI 应用的最新状态后，会检查当前状态的验证人集合中是否只有当前节点一个验证人，如果是，则无需快速同步。

```go
// 重新从数据库加载状态，因为可能在握手时有更新
state = sm.LoadState(stateDB)

// Decide whether to fast-sync or not
// We don't fast-sync when the only validator is us.

// 此字段用来指定当此节点在区块链末端有许多区块需要同步时，是否启用快速同步功能，默认为 true
fastSync := config.FastSync
if state.Validators.Size() == 1 {
   addr, _ := state.Validators.GetByIndex(0)
   if bytes.Equal(privValidator.GetAddress(), addr) {
      fastSync = false
   }
}
```

**日志**：

```bash
This node is a validator                     module=consensus addr=FF2AF6957DCD4B2FA8373D2157DE67278C5F0E41 pubKey=PubKeyEd25519{476CA31AE9AFB1FDC2BE1A00C32796FE5EEDBE3BD4C3C05CD1A76A4E975FB975}
```

 `NewNode()` 中相关代码，显示当前节点是否是验证人：

```go
// Log whether this node is a validator or an observer
if state.Validators.HasAddress(privValidator.GetAddress()) {
   consensusLogger.Info("This node is a validator", "addr", privValidator.GetAddress(), "pubKey", privValidator.GetPubKey())
} else {
   consensusLogger.Info("This node is not a validator", "addr", privValidator.GetAddress(), "pubKey", privValidator.GetPubKey())
}
```

### 创建各种 Reactor

`Reactor` 是处理各类传入消息的结构，一共有 5 种类型，通过将其添加到 `Switch` 中来实现。先熟悉一下 `Switch` 的结构：

`Switch` 处理 peer 连接，并暴露一个 API 以在各类 `Reactor` 上接收传入的消息。每个 `Reactor` 负责处理一个或多个 “Channels” 的传入消息。因此，发送传出消息通常在 peer 执行，传入的消息在 `Reactor` 上接收。

```go
type Switch struct {
   cmn.BaseService

   config       *config.P2PConfig
   listeners    []Listener
   // 添加的 Reactor 都存在这里
   reactors     map[string]Reactor
   chDescs      []*conn.ChannelDescriptor
   reactorsByCh map[byte]Reactor
   peers        *PeerSet
   dialing      *cmn.CMap
   reconnecting *cmn.CMap
   nodeInfo     NodeInfo // our node info
   nodeKey      *NodeKey // our node privkey
   addrBook     AddrBook

   filterConnByAddr func(net.Addr) error
   filterConnByID   func(ID) error

   rng *cmn.Rand // seed for randomizing dial times and orders
}
```

#### MempoolReactor

`Mempool` 是一个有序的内存池，交易在被共识提议之前会存储在这里，而在存储到这里之前会通过 ABCI 应用的 CheckTx 方法检查其合法性。

先看 `NewNode()` 中相关代码：

```go
mempoolLogger := logger.With("module", "mempool")
// 创建 Mempool
mempool := mempl.NewMempool(config.Mempool, proxyApp.Mempool(), state.LastBlockHeight)

// 初始化 Mempool 的  write-ahead log（确保可以从任何形式的崩溃中恢复过来）
mempool.InitWAL() // no need to have the mempool wal during tests
mempool.SetLogger(mempoolLogger)

// 创建 MempoolReactor
mempoolReactor := mempl.NewMempoolReactor(config.Mempool, mempool)
mempoolReactor.SetLogger(mempoolLogger)

// 这里根据配置，判断是否要等待有交易时才生成新区块
if config.Consensus.WaitForTxs() {
   mempool.EnableTxsAvailable()
}
```

`MempoolReactor` 用来在 peer 之间对 `mempool` 交易进行广播。看一下它的数据结构：

```go
type MempoolReactor struct {
   p2p.BaseReactor
   config  *cfg.MempoolConfig
   Mempool *Mempool
}

func NewMempoolReactor(config *cfg.MempoolConfig, mempool *Mempool) *MempoolReactor {
	memR := &MempoolReactor{
		config:  config,
		Mempool: mempool,
	}
	memR.BaseReactor = *p2p.NewBaseReactor("MempoolReactor", memR)
	return memR
}
```

#### EvidenceReactor

`Evidence` 是一个接口，表示验证人的任何可证明的恶意活动，主要有 `DuplicateVoteEvidence` （包含验证人签署两个相互矛盾的投票的证据。）这种实现。

 `NewNode()` 中相关代码：

```go
evidenceDB, err := dbProvider(&DBContext{"evidence", config})
if err != nil {
   return nil, err
}
evidenceLogger := logger.With("module", "evidence")

// EvidenceStore 用来存储见过的所有 Evidence，包括已提交的、已经过验证但没有广播的以及已经广播但未提交的
evidenceStore := evidence.NewEvidenceStore(evidenceDB)
// EvidencePool 在 EvidenceStore 中维护一组有效的 Evidence
evidencePool := evidence.NewEvidencePool(stateDB, evidenceStore)
evidencePool.SetLogger(evidenceLogger)

// 创建 EvidenceReactor
evidenceReactor := evidence.NewEvidenceReactor(evidencePool)
evidenceReactor.SetLogger(evidenceLogger)
```

`EvidenceReactor` 用来在 peer 间对 `EvidencePool` 中 `Evidence` 进行广播。

```go
type EvidenceReactor struct {
   p2p.BaseReactor
   evpool   *EvidencePool
    
   eventBus *types.EventBus
}
```

#### BlockchainReactor

 `NewNode()` 中相关代码：

```go
blockExecLogger := logger.With("module", "state")

// BlockExecutor 用来处理区块执行和状态更新。
// 它暴露一个 ApplyBlock() 方法，用来验证并执行区块、更新状态和 ABCI 应答，然后以原子方式提交
// 并更新 mempool，最后保存状态。
blockExec := sm.NewBlockExecutor(stateDB, blockExecLogger, proxyApp.Consensus(), mempool, evidencePool)

// 创建 BlockchainReactor
bcReactor := bc.NewBlockchainReactor(state.Copy(), blockExec, blockStore, fastSync)
bcReactor.SetLogger(logger.With("module", "blockchain"))
```

`BlockchainReactor` 用来处理长期的 catchup 同步。

```go
type BlockchainReactor struct {
	p2p.BaseReactor

	// immutable
	initialState sm.State

	blockExec *sm.BlockExecutor
    
    // 区块的底层存储。主要存储三种类型的信息：BlockMeta、Block part 和 Commit
	store     *BlockStore
    
    // 当加入到 BlockPool 时，peer 自己报告它们的高度。
    // 从当前节点最新的 pool.height 开始，从报告的高于我们高度的 peer 顺序请求区块。
    // 节点经常问 peer 他们当前的高度，这样我们就可以继续前进。
    // 不断请求更高的区块直到到达限制。如果大多数请求没有可用的 peer，并且没有处在 peer 限制，可以
    // 切换到 consensus reactor
	pool      *BlockPool
	fastSync  bool

	requestsCh <-chan BlockRequest
	errorsCh   <-chan peerError
}
```

#### ConsensusReactor

 `NewNode()` 中相关代码：

`ConsensusState` 用来处理共识算法的执行。它处理投票和提案，一旦达成一致，将区块提交给区块链并针对 ABCI 应用执行它们。内部状态机从 peer、内部验证人和定时器接收输入。

```go
// 创建 ConsensusState
consensusState := cs.NewConsensusState(config.Consensus, state.Copy(),
   blockExec, blockStore, mempool, evidencePool)
consensusState.SetLogger(consensusLogger)

if privValidator != nil {
    // 设置 PrivValidator 用来投票
   consensusState.SetPrivValidator(privValidator)
}

// 创建 ConsensusReactor
consensusReactor := cs.NewConsensusReactor(consensusState, fastSync)
consensusReactor.SetLogger(consensusLogger)
```

`ConsensusReactor` 用于共识服务。

```go
type ConsensusReactor struct {
   p2p.BaseReactor // BaseService + p2p.Switch

   conS *ConsensusState

   mtx      sync.RWMutex
   fastSync bool
   eventBus *types.EventBus
}
```

#### 把 reactor 添加到 Switch

 `NewNode()` 中相关代码：

把上面创建的四个 Reactor 添加到 Switch 中。

```go
p2pLogger := logger.With("module", "p2p")

sw := p2p.NewSwitch(config.P2P)
sw.SetLogger(p2pLogger)
sw.AddReactor("MEMPOOL", mempoolReactor)
sw.AddReactor("BLOCKCHAIN", bcReactor)
sw.AddReactor("CONSENSUS", consensusReactor)
sw.AddReactor("EVIDENCE", evidenceReactor)
```

#### PEXReactor（可选的）

看代码之前先了解几个概念：

- seeds：启动节点时可通过 `--p2p.seeds` 标签来指定种子节点，可以从中获得许多其它 peer 的地址。

  ```bash
  tendermint node --p2p.seeds "f9baeaa15fedf5e1ef7448dd60f46c01f1a9e9c4@1.2.3.4:46656,0491d373a8e0fcf1023aaf18c51d6a1d0d4f31bd@5.6.7.8:46656"
  ```

- persistent_peers：可指定与当前节点保持持久连接的一组节点。或使用 RPC 端点 `/dial_peers` 来指定而无需停止 Tendermint 实例。

  ```bash
  tendermint node --p2p.persistent_peers "429fcf25974313b95673f58d77eacdd434402665@10.11.12.13:46656,96663a3dd0d7b9d17d4c8211b191af259621c693@10.11.12.14:46656"
  curl 'localhost:46657/dial_peers?persistent=true&peers=\["429fcf25974313b95673f58d77eacdd434402665@10.11.12.13:46656","96663a3dd0d7b9d17d4c8211b191af259621c693@10.11.12.14:46656"\]'
  ```

- PEX：peer-exchange 协议的缩写，默认是开启的，在第一次启动后通常不需要种子。peer 之间将传播已知的 peer 并形成一个网络。peer 地址存储在 addrbook 中。

如果 PEX 模式是打开的，它应该处理种子的拨号，否则将由 Switch 来处理。

 `NewNode()` 中相关代码：

```go
// 创建 addrBook
addrBook := pex.NewAddrBook(config.P2P.AddrBookFile(), config.P2P.AddrBookStrict)
addrBook.SetLogger(p2pLogger.With("book", config.P2P.AddrBookFile()))

// 如果开启了 PEX 模式
if config.P2P.PexReactor {
   // 创建 PEX reactor
   pexReactor := pex.NewPEXReactor(addrBook,
      &pex.PEXReactorConfig{
         Seeds:          cmn.SplitAndTrim(config.P2P.Seeds, ",", " "),
         SeedMode:       config.P2P.SeedMode,
         PrivatePeerIDs: cmn.SplitAndTrim(config.P2P.PrivatePeerIDs, ",", " ")})
   pexReactor.SetLogger(p2pLogger)
   // 添加到 Switch
   sw.AddReactor("PEX", pexReactor)
}

sw.SetAddrBook(addrBook)
```

`PEXReactor` 处理 PEX 并保证足够数量的 peer 连接到 Switch。用 AddrBook 存储 peer 的 NetAddress。为防止滥用，只接受来自 peer 的 `pexAddrsMsg`，我们也发送了相应的 `pexRequestMsg`。每个 `defaultEnsurePeersPeriod` 时间段内只接收一个 `pexRequestMsg`。

### FilterPeers

配置中的 `config.FilterPeers` 字段用来指定当连接到一个新 peer 时是否要查询 ABCI 应用，由应用用来决定是否要保持连接。

使用 ABCI 查询通过 addr 或 pubkey 来过滤 peer，如果返回 OK 则添加 peer。

 `NewNode()` 中相关代码：

```go
if config.FilterPeers {
   // 设置两种类型的 Filter
   sw.SetAddrFilter(func(addr net.Addr) error {
      resQuery, err := proxyApp.Query().QuerySync(abci.RequestQuery{Path: cmn.Fmt("/p2p/filter/addr/%s", addr.String())})
      if err != nil {
         return err
      }
      if resQuery.IsErr() {
         return fmt.Errorf("Error querying abci app: %v", resQuery)
      }
      return nil
   })
   sw.SetIDFilter(func(id p2p.ID) error {
      resQuery, err := proxyApp.Query().QuerySync(abci.RequestQuery{Path: cmn.Fmt("/p2p/filter/pubkey/%s", id)})
      if err != nil {
         return err
      }
      if resQuery.IsErr() {
         return fmt.Errorf("Error querying abci app: %v", resQuery)
      }
      return nil
   })
}
```

### 设置 EventBus

`EventBus` 是一个通过此系统的所有事件的事件总线，所有的调用都代理到底层的 pubsub 服务器。所有事件都必须用 `EventBus` 来发布，以保证正确的数据类型。

```go
type EventBus struct {
	cmn.BaseService
    
    // Server 允许 client 订阅/取消订阅消息，带或不带 tag 发布消息，并管理内部状态
	pubsub *tmpubsub.Server
}
```

 `NewNode()` 中相关代码：

```go
eventBus := types.NewEventBus()
eventBus.SetLogger(logger.With("module", "events"))

// 将要发布和/或订阅消息（事件）的服务
// consensusReactor 将在 consensusState 和 blockExecutor 上设置 eventBus
consensusReactor.SetEventBus(eventBus)
```

### 设置 TxIndexer

`TxIndexer` 是一个接口，定义了索引和搜索交易的方法。它有两种实现，一个是 `kv.TxIndex`，由键值存储支持（levelDB），另一个是 `null.TxIndex`，即不设置索引（默认的）。

`IndexerService` 会把 `TxIndexer` 和 `EventBus` 连接在一起，以对来自 `EventBus` 的交易进行索引。

```go
type IndexerService struct {
	cmn.BaseService

	idr      TxIndexer
	eventBus *types.EventBus
}
```

 `NewNode()` 中相关代码：

```go
var txIndexer txindex.TxIndexer
switch config.TxIndex.Indexer {
// 键值存储索引
case "kv":
   // 创建 DB
   store, err := dbProvider(&DBContext{"tx_index", config})
   if err != nil {
      return nil, err
   }
   // 有指定要索引的标签列表（以逗号分隔）
   if config.TxIndex.IndexTags != "" {
      txIndexer = kv.NewTxIndex(store, kv.IndexTags(cmn.SplitAndTrim(config.TxIndex.IndexTags, ",", " ")))
   // 要索引所有标签
   } else if config.TxIndex.IndexAllTags {
      txIndexer = kv.NewTxIndex(store, kv.IndexAllTags())
   // 不索引标签
   } else {
      txIndexer = kv.NewTxIndex(store)
   }
default:
   txIndexer = &null.TxIndex{}
}

// 创建 IndexerService
indexerService := txindex.NewIndexerService(txIndexer, eventBus)
indexerService.SetLogger(logger.With("module", "txindex"))
```

### 创建节点的 BaseService

节点所需的所有字段内容已在以上部分创建完毕，在创建 `BaseService` 后，就可以启动节点了。

```go
node.BaseService = *cmn.NewBaseService(logger, "Node", node)
```

### 创建节点总结

总结一下 `NewNode` 函数都做了哪些事情：

- 创建 Tendermint 与 ABCI 应用建立 `mempool` 、`consensus` 和 `query` 连接所需的客户端。
- Tendermint 节点与应用程序执行握手，确保其状态是同步的。
- 根据配置 `config.FastSync` 及 `privValidator.GetAddress()` 方法，判断是否需要快速同步，是否是验证人节点。
- 创建并在 `Switch` 中设置 `Reactor`，即 `MempoolReactor`、`EvidenceReactor`、`BlockchainReactor`、`ConsensusReactor` 以及 `PEXReactor` 这五种。用来在 peer 上接收不同类型的消息。
- 根据配置 `config.FilterPeers` 判断是否要用 ABCI 查询通过 addr 或 pubkey 来过滤要新连接的 peer，如果返回 OK 则添加 peer。
- 创建并设置 `EventBus`，订阅/发布事件。
- 设置 `TxIndexer`，对交易进行索引。

## 启动节点

回到 `startInProcess` 函数，在这里创建完毕节点后，执行 `Start` 方法，实际执行的是节点的 `OnStart` 方法。

接下来就看这个方法：

```go
// 日志：Starting Node    module=node impl=Node
func (n *Node) OnStart() error {
    // 日志：Starting EventBus    module=events impl=EventBus
	err := n.eventBus.Start()
	if err != nil {
		return err
	}

     // 监听 P2P 连接的端口
	// 日志：Local listener    module=p2p ip=:: port=46656
    // 日志：Could not perform UPNP discover    module=p2p err="write udp4 0.0.0.0:63731->239.255.255.250:1900: i/o timeout"
    // 日志：Starting DefaultListener    module=p2p impl=Listener(@169.254.237.47:46656)
	protocol, address := cmn.ProtocolAndAddress(n.config.P2P.ListenAddress)
	l := p2p.NewDefaultListener(protocol, address, n.config.P2P.SkipUPNP, n.Logger.With("module", "p2p"))
	n.sw.AddListener(l)

	// 生成节点的 PrivKey
    // 日志：P2P Node ID    module=node ID=3158bddb4e03bef94bf4a99543af5beab4733368 file=/Users/LLLeon/.basecoind/config/node_key.json
	nodeKey, err := p2p.LoadOrGenNodeKey(n.config.NodeKeyFile())
	if err != nil {
		return err
	}
	n.Logger.Info("P2P Node ID", "ID", nodeKey.ID(), "file", n.config.NodeKeyFile())

	nodeInfo := n.makeNodeInfo(nodeKey.ID())
	n.sw.SetNodeInfo(nodeInfo)
	n.sw.SetNodeKey(nodeKey)

	// 将自己添加到 addrbook 以防止连接自己
    // 日志： Add our address to book    module=p2p book=/Users/LLLeon/.basecoind/config/addrbook.json addr=3158bddb4e03bef94bf4a99543af5beab4733368@169.254.237.47:46656
	n.addrBook.AddOurAddress(nodeInfo.NetAddress())
    
    // 在 P2P 服务器之前启动 RPC 服务器，所以可以例如：接收第一个区块的交易
    // 日志：Starting RPC HTTP server on tcp://0.0.0.0:46657 module=rpc-server
	if n.config.RPC.ListenAddress != "" {
		listeners, err := n.startRPC()
		if err != nil {
			return err
		}
		n.rpcListeners = listeners
	}

	// 启动 switch (P2P 服务器)，函数内部依次启动了各 Reactor，
    // Switch 会在 46656 端口（默认的）监听 peer 的连接，连接后会通过 Reactor 的 Receive 方法
    // 接收来自 peer 的消息
    
    // 日志：Starting P2P Switch    module=p2p impl="P2P Switch"
    // 日志：Starting BlockchainReactor    module=blockchain impl=BlockchainReactor
    // 日志：Starting ConsensusReactor    module=consensus impl=ConsensusReactor
    // 日志：ConsensusReactor    module=consensus fastSync=false
    // 日志：Starting ConsensusState    module=consensus impl=ConsensusState
    // 日志：Starting baseWAL    module=consensus wal=/Users/LLLeon/.basecoind/data/cs.wal/wal impl=baseWAL
    // 日志：Catchup by replaying consensus messages    module=consensus height=169315
    // 日志：Replay: Done    module=consensus
    // 日志：Starting EvidenceReactor    module=evidence impl=EvidenceReactor
    // 日志：Starting PEXReactor    module=p2p impl=PEXReactor
    // 日志：Starting AddrBook    module=p2p book=/Users/LLLeon/.basecoind/config/addrbook.json impl=AddrBook
    // 日志：enterNewRound(169315/0). Current: 169315/0/RoundStepNewHeight module=consensus height=169315 round=0
    // 日志：enterPropose(169315/0). Current: 169315/0/RoundStepNewRound module=consensus height=169315 round=0
    // 日志：enterPropose: Our turn to propose            module=consensus height=169315 round=0 proposer=FF2AF6957DCD4B2FA8373D2157DE67278C5F0E41 privValidator="PrivValidator{FF2AF6957DCD4B2FA8373D2157DE67278C5F0E41 LH:169314, LR:0, LS:3}"
    // 日志：Starting MempoolReactor    module=mempool impl=MempoolReactor
	err = n.sw.Start()
	if err != nil {
		return err
	}

	// 始终连接到持久 peers
	if n.config.P2P.PersistentPeers != "" {
		err = n.sw.DialPeersAsync(n.addrBook, cmn.SplitAndTrim(n.config.P2P.PersistentPeers, ",", " "), true)
		if err != nil {
			return err
		}
	}

	// 启动交易的 indexer
    // 日志：Starting IndexerService    module=txindex impl=IndexerService
	return n.indexerService.Start()
}
```

至此，节点中各模块已经启动完毕，接下来进入 catchup、共识协议等核心处理逻辑。

## Tendermint 核心处理逻辑

这部分包括区块的 catchup、共识协议处理等内容。

这部分逻辑是在 `ConsensusReactor` 启动时处理的。

**日志**：

```bash
Starting ConsensusReactor                    module=consensus impl=ConsensusReactor
ConsensusReactor                             module=consensus fastSync=false
Starting ConsensusState                      module=consensus impl=ConsensusState
Starting baseWAL                             module=consensus wal=/Users/LLLeon/.basecoind/data/cs.wal/wal impl=baseWAL
Starting TimeoutTicker                       module=consensus impl=TimeoutTicker
Catchup by replaying consensus messages      module=consensus height=169315
Replay: Done                                 module=consensus
```

先看 `ConsensusReactor` 启动的代码：

```go
func (conR *ConsensusReactor) OnStart() error {
   conR.Logger.Info("ConsensusReactor ", "fastSync", conR.FastSync())
   if err := conR.BaseReactor.OnStart(); err != nil {
      return err
   }

   // 订阅这几种类型的事件：EventNewRoundStep、EventVote 和 EventProposalHeartbeat，一旦接收到这些
   // 类型的消息就会用在 state 中定义的 pubsub 广播给 peer
   conR.subscribeToBroadcastEvents()

    // 由于只有我们一个节点，所以会设置 FastSync = false，会启动 ConsensusState
   if !conR.FastSync() {
      err := conR.conS.Start()
      if err != nil {
         return err
      }
   }

   return nil
}
```

现在看 `ConsensusState` 的启动：

```go
func (cs *ConsensusState) OnStart() error {
	if err := cs.evsw.Start(); err != nil {
		return err
	}

	// we may set the WAL in testing before calling Start,
	// so only OpenWAL if its still the nilWAL
	if _, ok := cs.wal.(nilWAL); ok {
		walFile := cs.config.WalFile()
        
         // 这里会启动 baseWAL，它在处理消息之前会将其写入磁盘。可用于崩溃恢复和确定性重放
		wal, err := cs.OpenWAL(walFile)
		if err != nil {
			cs.Logger.Error("Error loading ConsensusState wal", "err", err.Error())
			return err
		}
		cs.wal = wal
	}

	// we need the timeoutRoutine for replay so
	// we don't block on the tick chan.
	// NOTE: we will get a build up of garbage go routines
	// firing on the tockChan until the receiveRoutine is started
	// to deal with them (by that point, at most one will be valid)
    
    // 用来对每一步的超时进行控制
    // 里面是在协程里面执行 timeoutRoutine 方法，内部会监听 timeoutTicker.tickChan，进行到下一步时
    // 会中止并重置旧 timer，并更新 timeoutInfo。tickChan 上超时时间为 0 时，会立即转发到 tockChan
    // 日志：Starting TimeoutTicker    module=consensus impl=TimeoutTicker
	if err := cs.timeoutTicker.Start(); err != nil {
		return err
	}

	// we may have lost some votes if the process crashed
	// reload from consensus log to catchup
    
    // 新建 ConsensusState 时已设置为 true
	if cs.doWALCatchup {
         // catchup：仅重放自上一个区块以来的那些消息
         // 日志：Catchup by replaying consensus messages      module=consensus height=169315
         // 日志：Replay: Done                                 module=consensus
		if err := cs.catchupReplay(cs.Height); err != nil {
			cs.Logger.Error("Error on catchup replay. Proceeding to start ConsensusState anyway", "err", err.Error())
			// NOTE: if we ever do return an error here,
			// make sure to stop the timeoutTicker
		}
	}

    // 核心处理逻辑，主要的协程，详细解释见下面
	go cs.receiveRoutine(0)

	// schedule the first round!
	// use GetRoundState so we don't race the receiveRoutine for access
	cs.scheduleRound0(cs.GetRoundState())

	return nil
}
```
这个方法里面主要做了区块的 catchup、在协程中启动 `receiveRoutine()` 方法来接收并处理消息，以及执行 `scheduleRound0()` 方法来进入新回合，下面逐一说明。

### catchup

在这次启动节点之前，已经运行过一段时间，区块最新高度为 169315。首先看节点在重启后是如何 catchup 到此高度的。

`catchupReplay` 源码：

```go
func (cs *ConsensusState) catchupReplay(csHeight int64) error {

   // Set replayMode to true so we don't log signing errors.
   cs.replayMode = true
   defer func() { cs.replayMode = false }()

   // Ensure that #ENDHEIGHT for this height doesn't exist.
   // NOTE: This is just a sanity check. As far as we know things work fine
   // without it, and Handshake could reuse ConsensusState if it weren't for
   // this check (since we can crash after writing #ENDHEIGHT).
   //
   // Ignore data corruption errors since this is a sanity check.
   gr, found, err := cs.wal.SearchForEndHeight(csHeight, &WALSearchOptions{IgnoreDataCorruptionErrors: true})
   if err != nil {
      return err
   }
   if gr != nil {
      if err := gr.Close(); err != nil {
         return err
      }
   }
   if found {
      return fmt.Errorf("WAL should not contain #ENDHEIGHT %d", csHeight)
   }

   // Search for last height marker.
   //
   // Ignore data corruption errors in previous heights because we only care about last height
   gr, found, err = cs.wal.SearchForEndHeight(csHeight-1, &WALSearchOptions{IgnoreDataCorruptionErrors: true})
   if err == io.EOF {
      cs.Logger.Error("Replay: wal.group.Search returned EOF", "#ENDHEIGHT", csHeight-1)
   } else if err != nil {
      return err
   }
   if !found {
      return fmt.Errorf("Cannot replay height %d. WAL does not contain #ENDHEIGHT for %d", csHeight, csHeight-1)
   }
   defer gr.Close() // nolint: errcheck

   cs.Logger.Info("Catchup by replaying consensus messages", "height", csHeight)

   var msg *TimedWALMessage
   dec := WALDecoder{gr}

   for {
      msg, err = dec.Decode()
      if err == io.EOF {
         break
      } else if IsDataCorruptionError(err) {
         cs.Logger.Debug("data has been corrupted in last height of consensus WAL", "err", err, "height", csHeight)
         panic(fmt.Sprintf("data has been corrupted (%v) in last height %d of consensus WAL", err, csHeight))
      } else if err != nil {
         return err
      }

      // NOTE: since the priv key is set when the msgs are received
      // it will attempt to eg double sign but we can just ignore it
      // since the votes will be replayed and we'll get to the next step
      if err := cs.readReplayMessage(msg, nil); err != nil {
         return err
      }
   }
   cs.Logger.Info("Replay: Done")
   return nil
}
```

### receiveRoutine

**核心处理逻辑**：需要重点看一下 `cs.receiveRoutine()` 方法。

这个方法处理可能引起状态转换的消息，参数 `maxSteps` 表示退出前要处理的消息数量，0 表示永不退出。它持有 `RoundState` 并且是唯一更新它的地方。更新发生在超时、完成提案和 2/3 多数时。 `ConsensusState` 在任意内部状态更新之前必须是锁定的。

这个方法在单独的协程里面，接收并处理来自 peer、节点内部或超时消息。

```go
func (cs *ConsensusState) receiveRoutine(maxSteps int) {
   defer func() {
      if r := recover(); r != nil {
         cs.Logger.Error("CONSENSUS FAILURE!!!", "err", r, "stack", string(debug.Stack()))
      }
   }()

   for {
      // 到达设定的 maxSteps 时退出
      if maxSteps > 0 {
         if cs.nSteps >= maxSteps {
            cs.Logger.Info("reached max steps. exiting receive routine")
            cs.nSteps = 0
            return
         }
      }
      rs := cs.RoundState
      var mi msgInfo

      select {
      // TxsAvailable 返回一个通道，每添加一个交易到 mempool 后会触发一次，
      // 并且只有在 mempool 中的交易可用时才会触发。
      // 如果没有调用 EnableTxsAvailable，返回的 channel 可能为 nil。
      case height := <-cs.mempool.TxsAvailable():
          
          // 这里会执行 enterPropose()，进入提议环节，完成后会执行 enterPrevote 进入 Prevote 环节
          // 日志：enterPropose(169315/0). Current: 169315/0/RoundStepNewRound module=consensus height=169315 round=0
          // 日志：enterPropose: Our turn to propose            module=consensus height=169315 round=0 proposer=FF2AF6957DCD4B2FA8373D2157DE67278C5F0E41 privValidator="PrivValidator{FF2AF6957DCD4B2FA8373D2157DE67278C5F0E41 LH:169314, LR:0, LS:3}"
         cs.handleTxsAvailable(height)
          
      // 接收到来自 peer 的消息
      case mi = <-cs.peerMsgQueue:
         cs.wal.Write(mi)
         // handles proposals, block parts, votes
         // may generate internal events (votes, complete proposals, 2/3 majorities)
         // 处理消息
         cs.handleMsg(mi)
          
      // 接收到来自内部的消息
      case mi = <-cs.internalMsgQueue:
         // 发送签名的消息前写入磁盘
         cs.wal.WriteSync(mi) // NOTE: fsync
         // handles proposals, block parts, votes
         cs.handleMsg(mi)
          
      // 接收到超时消息
      case ti := <-cs.timeoutTicker.Chan(): // tockChan:
         cs.wal.Write(ti)
         // if the timeout is relevant to the rs
         // go to the next step
         cs.handleTimeout(ti, rs)
      case <-cs.Quit():

         // NOTE: the internalMsgQueue may have signed messages from our
         // priv_val that haven't hit the WAL, but its ok because
         // priv_val tracks LastSig

         // close wal now that we're done writing to it
         cs.wal.Stop()

         close(cs.done)
         return
      }
   }
}
```

### scheduleRound0

`scheduleRound0()` 方法会将当前 consensus H/R/S 封装成 `timeoutInfo` 发送给 `tickChan`，这就进入了 `timeoutTicker.timeoutRoutine` 的逻辑。

这里发送的信息第 0 回合，步骤为 `RoundStepNewHeight`。

`scheduleRound0` 源码：


```go
func (cs *ConsensusState) scheduleRound0(rs *cstypes.RoundState) {
	
	sleepDuration := rs.StartTime.Sub(time.Now())
	cs.scheduleTimeout(sleepDuration, rs.Height, 0, cstypes.RoundStepNewHeight)
}
```

`timeoutTicker.timeoutRoutine` 源码：

```go
func (t *timeoutTicker) timeoutRoutine() {
   t.Logger.Debug("Starting timeout routine")
   var ti timeoutInfo
   for {
      select {
      case newti := <-t.tickChan:
         t.Logger.Debug("Received tick", "old_ti", ti, "new_ti", newti)

         // ignore tickers for old height/round/step
         if newti.Height < ti.Height {
            continue
         } else if newti.Height == ti.Height {
            if newti.Round < ti.Round {
               continue
            } else if newti.Round == ti.Round {
               if ti.Step > 0 && newti.Step <= ti.Step {
                  continue
               }
            }
         }

         // stop the last timer
         t.stopTimer()

         // update timeoutInfo and reset timer
         // NOTE time.Timer allows duration to be non-positive
         ti = newti
         t.timer.Reset(ti.Duration)
         t.Logger.Debug("Scheduled timeout", "dur", ti.Duration, "height", ti.Height, "round", ti.Round, "step", ti.Step)
      case <-t.timer.C:
         t.Logger.Info("Timed out", "dur", ti.Duration, "height", ti.Height, "round", ti.Round, "step", ti.Step)
         // go routine here guarantees timeoutRoutine doesn't block.
         // Determinism comes from playback in the receiveRoutine.
         // We can eliminate it by merging the timeoutRoutine into receiveRoutine
         // and managing the timeouts ourselves with a millisecond ticker
         go func(toi timeoutInfo) { t.tockChan <- toi }(ti)
      case <-t.Quit():
         return
      }
   }
}
```

在 `receiveRoutine` 中，如果触发了超时，会执行 `handleTimeout`，进行状态转移相关操作：

```go
func (cs *ConsensusState) handleTimeout(ti timeoutInfo, rs cstypes.RoundState) {
	cs.Logger.Debug("Received tock", "timeout", ti.Duration, "height", ti.Height, "round", ti.Round, "step", ti.Step)

	// timeouts must be for current height, round, step
	if ti.Height != rs.Height || ti.Round < rs.Round || (ti.Round == rs.Round && ti.Step < rs.Step) {
		cs.Logger.Debug("Ignoring tock because we're ahead", "height", rs.Height, "round", rs.Round, "step", rs.Step)
		return
	}

	// the timeout will now cause a state transition
	cs.mtx.Lock()
	defer cs.mtx.Unlock()

	switch ti.Step {
	case cstypes.RoundStepNewHeight:
		// NewRound event fired from enterNewRound.
		// XXX: should we fire timeout here (for timeout commit)?
		cs.enterNewRound(ti.Height, 0)
	case cstypes.RoundStepNewRound:
		cs.enterPropose(ti.Height, 0)
	case cstypes.RoundStepPropose:
		cs.eventBus.PublishEventTimeoutPropose(cs.RoundStateEvent())
		cs.enterPrevote(ti.Height, ti.Round)
	case cstypes.RoundStepPrevoteWait:
		cs.eventBus.PublishEventTimeoutWait(cs.RoundStateEvent())
		cs.enterPrecommit(ti.Height, ti.Round)
	case cstypes.RoundStepPrecommitWait:
		cs.eventBus.PublishEventTimeoutWait(cs.RoundStateEvent())
		cs.enterNewRound(ti.Height, ti.Round+1)
	default:
		panic(cmn.Fmt("Invalid timeout step: %v", ti.Step))
	}

}
```

## 共识协议处理

共识投票和生成区块等环节是在 `receiveRoutine` 中收到消息或超时后由 `handleMsg` 和 `handleTimeout` 执行相应方法进行处理。

本文只提供一个阅读 Tendermint 源码的入口，其它核心逻辑，比如创建区块等逻辑，需要自己深入 `consensus` 包。