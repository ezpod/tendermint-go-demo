## Tendermint Core Golang应用开发指南

Tendermint Core是一个用Go语言开发的支持拜占庭容错/BFT的区块链中间件，
用于在一组节点之间安全地复制状态机/FSM。Tendermint Core的出色之处
在于它是第一个实现BFT的区块链共识引擎，并且始终保持这一清晰的定位。
这个指南将介绍如何使用Go语言开发一个基于Tendermint Core的区块链应用。

Tendermint Core为区块链应用提供了极其简洁的开发接口，支持各种开发语言，
是开发自有公链/联盟链/私链的首选方案，例如Cosmos、[Binance Chain](http://sc.hubwiz.com/codebag/bnbtool/)、
Hyperledger Burrow、Ethermint等均采用Tendermint Core共识引擎。

虽然Tendermint Core支持任何语言开发的状态机，但是如果采用Go之外的
其他开发语言编写状态机，那么应用就需要通过套接字或gRPC与Tendermint 
Core通信，这会造成额外的性能损失。而采用Go语言开发的状态机可以和
Tendermint Core运行在同一进程中，因此可以得到最好的性能。

>相关链接：[ Tendermint 区块链开发详解 ](http://xc.hubwiz.com/course/5bdec63ac02e6b6a59171df3?affid=blog7878) | [本教程源代码下载](https://github.com/ezpod/tendermint-go-demo)

## 1、安装Go开发环境

请参考[官方文档](https://golang.org/doc/install)安装Go开发环境。

确认你已经安装了最新版的Go：

```
$ go version
go version go1.12.7 darwin/amd64
```

确认你正确设置了`GOPATH`环境变量：

```
$ echo $GOPATH
/Users/melekes/go
```

## 2、创建Go项目

首先创建一个新的Go语言项目：

```
$ mkdir -p $GOPATH/src/github.com/me/kvstore
$ cd $GOPATH/src/github.com/me/kvstore
```

在`example`目录创建`main.go`文件，内容如下：

```
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, Tendermint Core")
}
```

运行上面代码，将在标准输出设备显示指定的字符串：

```
$ go run main.go
Hello, Tendermint Core
```

## 3、编写Tendermint Core应用

Tendermint Core与应用之间通过ABCI（Application Blockchain Interface）通信，
使用的报文消息类型都定义在protobuf文件中，因此基于Tendermint Core可以运行任何
语言开发的应用。

创建文件`app.go`，内容如下：

```
package main
import (
	abcitypes "github.com/tendermint/tendermint/abci/types"
)

type KVStoreApplication struct {}

var _ abcitypes.Application = (*KVStoreApplication)(nil)

func NewKVStoreApplication() *KVStoreApplication {
	return &KVStoreApplication{}
}

func (KVStoreApplication) Info(req abcitypes.RequestInfo) abcitypes.ResponseInfo {
	return abcitypes.ResponseInfo{}
}

func (KVStoreApplication) SetOption(req abcitypes.RequestSetOption) abcitypes.ResponseSetOption {
	return abcitypes.ResponseSetOption{}
}

func (KVStoreApplication) DeliverTx(req abcitypes.RequestDeliverTx) abcitypes.ResponseDeliverTx {
	return abcitypes.ResponseDeliverTx{Code: 0}
}

func (KVStoreApplication) CheckTx(req abcitypes.RequestCheckTx) abcitypes.ResponseCheckTx {
	return abcitypes.ResponseCheckTx{Code: 0}
}

func (KVStoreApplication) Commit() abcitypes.ResponseCommit {
	return abcitypes.ResponseCommit{}
}

func (KVStoreApplication) Query(req abcitypes.RequestQuery) abcitypes.ResponseQuery {
	return abcitypes.ResponseQuery{Code: 0}
}

func (KVStoreApplication) InitChain(req abcitypes.RequestInitChain) abcitypes.ResponseInitChain {
	return abcitypes.ResponseInitChain{}
}

func (KVStoreApplication) BeginBlock(req abcitypes.RequestBeginBlock) abcitypes.ResponseBeginBlock {
	return abcitypes.ResponseBeginBlock{}
}

func (KVStoreApplication) EndBlock(req abcitypes.RequestEndBlock) abcitypes.ResponseEndBlock {
	return abcitypes.ResponseEndBlock{}
}
```

接下来我们逐个解读上述方法并添加必要的实现逻辑。

## 3、CheckTx

当一个新的交易进入Tendermint Core时，它会要求应用先进行检查，比如验证格式、签名等。

```
func (app *KVStoreApplication) isValid(tx []byte) (code uint32) {
	// check format
	parts := bytes.Split(tx, []byte("="))
	if len(parts) != 2 {
		return 1
	}	
  
  key, value := parts[0], parts[1]	
  
  // check if the same key=value already exists
	err := app.db.View(func(txn *badger.Txn) error {
		item, err := txn.Get(key)
		if err != nil && err != badger.ErrKeyNotFound {
			return err
		}
		if err == nil {
			return item.Value(func(val []byte) error {
				if bytes.Equal(val, value) {
					code = 2
				}
				return nil
			})
		}
		return nil
	})
  
	if err != nil {
		panic(err)
	}	
  
  return code
}

func (app *KVStoreApplication) CheckTx(req abcitypes.RequestCheckTx) abcitypes.ResponseCheckTx {
	code := app.isValid(req.Tx)
	return abcitypes.ResponseCheckTx{Code: code, GasWanted: 1}
}
```

如果进来的交易格式不是`{bytes}={bytes}`，我们将返回代码`1`。如果指定的key和value
已经存在，我们返回代码`2`。对于其他情况我们返回代码`0`表示交易有效 —— 注意Tendermint
Core会将返回任何非零代码的交易视为无效交易。

有效的交易最终将被提交，我们使用[badger](https://github.com/dgraph-io/badger)
作为底层的键/值库，badger是一个嵌入式的快速KV数据库。

```
import "github.com/dgraph-io/badger"type KVStoreApplication struct {
	db           *badger.DB
	currentBatch *badger.Txn
}func NewKVStoreApplication(db *badger.DB) *KVStoreApplication {
	return &KVStoreApplication{
		db: db,
	}
}
```

## 4、BeginBlock -> DeliverTx -> EndBlock -> Commit

当Tendermint Core确定了新的区块后，它会分三次调用应用：

- BeginBlock：区块开始时调用
- DeliverTx：每个交易时调用
- EndBlock：区块结束时调用

注意，DeliverTx是异步调用的，但是响应是有序的。

```
func (app *KVStoreApplication) BeginBlock(req abcitypes.RequestBeginBlock) abcitypes.ResponseBeginBlock {
	app.currentBatch = app.db.NewTransaction(true)
	return abcitypes.ResponseBeginBlock{}
}
```

下面的代码创建一个数据操作批次，用来存储区块交易：

```
func (app *KVStoreApplication) DeliverTx(req abcitypes.RequestDeliverTx) abcitypes.ResponseDeliverTx {
	code := app.isValid(req.Tx)
	if code != 0 {
		return abcitypes.ResponseDeliverTx{Code: code}
	}	
  parts := bytes.Split(req.Tx, []byte("="))
	
  key, value := parts[0], parts[1]	
  
  err := app.currentBatch.Set(key, value)
	if err != nil {
		panic(err)
	}	
  
  return abcitypes.ResponseDeliverTx{Code: 0}
}
```

如果交易的格式错误，或者已经存在相同的键/值对，那么我们仍然返回
非零代码，否则，我们将该交易加入操作批次。

在目前的设计中，区块中可以包含不正确的交易 —— 那些通过了CheckTx检查
但是DeliverTx失败的交易，这样做是出于性能的考虑。

注意，我们不能在DeliverTx中提交交易，因为在这种情况下Query可能会
由于被并发调用而返回不一致的数据，例如，Query会提示指定的值已经存在，
而实际的区块还没有真正提交。

`Commit`用来通知应用来持久化新的状态。

```
func (app *KVStoreApplication) Commit() abcitypes.ResponseCommit {
	app.currentBatch.Commit()
	return abcitypes.ResponseCommit{Data: []byte{}}
}
```

## 5、查询 - Query

当客户端应用希望了解指定的键/值对是否存在时，它会调用Tendermint Core
的RPC接口[ /abci_query ](http://cw.hubwiz.com/card/c/tendermint-rpc-api/1/1/2/)进行查询，
该接口会调用应用的`Query`方法。

基于Tendermint Core的应用可以自由地提供其自己的API。不过使用Tendermint Core
作为代理，客户端应用利用Tendermint Core的统一API的优势。另外，客户端也不需要
调用其他额外的Tendermint Core API来获得进一步的证明。

注意在下面的代码中我们没有包含证明数据。

```
func (app *KVStoreApplication) Query(reqQuery abcitypes.RequestQuery) (resQuery abcitypes.ResponseQuery) {
	resQuery.Key = reqQuery.Data
	err := app.db.View(func(txn *badger.Txn) error {
		item, err := txn.Get(reqQuery.Data)
		if err != nil && err != badger.ErrKeyNotFound {
			return err
		}
		if err == badger.ErrKeyNotFound {
			resQuery.Log = "does not exist"
		} else {
			return item.Value(func(val []byte) error {
				resQuery.Log = "exists"
				resQuery.Value = val
				return nil
			})
		}
		return nil
	})
	if err != nil {
		panic(err)
	}
	return
}
```


## 6、在同一进程内启动Tendermint Core和应用实例

将以下代码加入`main.go`文件：

```
package main
import (
	"flag"
	"fmt"
	"os"
	"os/signal"
	"path/filepath"
	"syscall"	
  "github.com/dgraph-io/badger"
	"github.com/pkg/errors"
	"github.com/spf13/viper"	
  abci "github.com/tendermint/tendermint/abci/types"
	cfg "github.com/tendermint/tendermint/config"
	tmflags "github.com/tendermint/tendermint/libs/cli/flags"
	"github.com/tendermint/tendermint/libs/log"
	nm "github.com/tendermint/tendermint/node"
	"github.com/tendermint/tendermint/p2p"
	"github.com/tendermint/tendermint/privval"
	"github.com/tendermint/tendermint/proxy"
)
var configFile string

func init() {
	flag.StringVar(&configFile, "config", "$HOME/.tendermint/config/config.toml", "Path to config.toml")
}

func main() {
	db, err := badger.Open(badger.DefaultOptions("/tmp/badger"))
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to open badger db: %v", err)
		os.Exit(1)
	}
	defer db.Close()
	
  app := NewKVStoreApplication(db)	
  
  flag.Parse()	
  
  node, err := newTendermint(app, configFile)	
  if err != nil {
		fmt.Fprintf(os.Stderr, "%v", err)
		os.Exit(2)
	}	
  
  node.Start()
	
  defer func() {
		node.Stop()
		node.Wait()
`	}()	

  c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt, syscall.SIGTERM)
	<-c
	os.Exit(0)
}

func newTendermint(app abci.Application, configFile string) (*nm.Node, error) {
	// read config
	config := cfg.DefaultConfig()
	config.RootDir = filepath.Dir(filepath.Dir(configFile))
	viper.SetConfigFile(configFile)
	if err := viper.ReadInConfig(); err != nil {
		return nil, errors.Wrap(err, "viper failed to read config file")
	}
	if err := viper.Unmarshal(config); err != nil {
		return nil, errors.Wrap(err, "viper failed to unmarshal config")
	}
	if err := config.ValidateBasic(); err != nil {
		return nil, errors.Wrap(err, "config is invalid")
	}	
  
  // create logger
	logger := log.NewTMLogger(log.NewSyncWriter(os.Stdout))
	var err error
	logger, err = tmflags.ParseLogLevel(config.LogLevel, logger, cfg.DefaultLogLevel())
	if err != nil {
		return nil, errors.Wrap(err, "failed to parse log level")
	}	
  
  // read private validator
	pv := privval.LoadFilePV(
		config.PrivValidatorKeyFile(),
		config.PrivValidatorStateFile(),
	)	
  
  // read node key
	nodeKey, err := p2p.LoadNodeKey(config.NodeKeyFile())
	if err != nil {
		return nil, errors.Wrap(err, "failed to load node's key")
	}	
  
  // create node
	node, err := nm.NewNode(
		config,
		pv,
		nodeKey,
		proxy.NewLocalClientCreator(app),
		nm.DefaultGenesisDocProviderFunc(config),
		nm.DefaultDBProvider,
		nm.DefaultMetricsProvider(config.Instrumentation),
		logger)
	if err != nil {
		return nil, errors.Wrap(err, "failed to create new Tendermint node")
	}	
  
  return node, nil
}
```

这段代码很长，让我们分开来介绍。

首先，初始化Badger数据库，然后创建应用实例：

```
db, err := badger.Open(badger.DefaultOptions("/tmp/badger"))
if err != nil {
	fmt.Fprintf(os.Stderr, "failed to open badger db: %v", err)
	os.Exit(1)
}
defer db.Close()
app := NewKVStoreApplication(db)
```

接下来使用下面的代码创建Tendermint Core的Node实例：

```
flag.Parse()node, err := newTendermint(app, configFile)
if err != nil {
	fmt.Fprintf(os.Stderr, "%v", err)
	os.Exit(2)
}

...

// create node
node, err := nm.NewNode(
	config,
	pv,
	nodeKey,
	proxy.NewLocalClientCreator(app),
	nm.DefaultGenesisDocProviderFunc(config),
	nm.DefaultDBProvider,
	nm.DefaultMetricsProvider(config.Instrumentation),
	logger)
  
if err != nil {
	return nil, errors.Wrap(err, "failed to create new Tendermint node")
}
```

`NewNode`方法用来创建一个全节点实例，它需要传入一些参数，例如配置文件、
私有验证器、节点密钥等。

注意我们使用`proxy.NewLocalClientCreator`来创建一个本地客户端，而不是使用
套接字或gRPC来与Tendermint Core通信。

下面的代码使用[viper](https://github.com/spf13/viper)来读取配置文件，我们
将在下面使用tendermint的init命令来生成。

```
config := cfg.DefaultConfig()
config.RootDir = filepath.Dir(filepath.Dir(configFile))
viper.SetConfigFile(configFile)
if err := viper.ReadInConfig(); err != nil {
	return nil, errors.Wrap(err, "viper failed to read config file")
}
if err := viper.Unmarshal(config); err != nil {
	return nil, errors.Wrap(err, "viper failed to unmarshal config")
}
if err := config.ValidateBasic(); err != nil {
	return nil, errors.Wrap(err, "config is invalid")
}
```

我们使用`FilePV`作为私有验证器，通常你应该使用SignerRemote链接到
一个外部的HSM设备。

```
pv := privval.LoadFilePV(
	config.PrivValidatorKeyFile(),
	config.PrivValidatorStateFile(),
)
```

`nodeKey`用来在Tendermint的P2P网络中标识当前节点。

```
nodeKey, err := p2p.LoadNodeKey(config.NodeKeyFile())
if err != nil {
	return nil, errors.Wrap(err, "failed to load node's key")
}
```

我们使用内置的日志记录器：

```
logger := log.NewTMLogger(log.NewSyncWriter(os.Stdout))
var err error
logger, err = tmflags.ParseLogLevel(config.LogLevel, logger, cfg.DefaultLogLevel())
if err != nil {
	return nil, errors.Wrap(err, "failed to parse log level")
}
```

最后，我们启动节点并添加一些处理逻辑，以便在收到SIGTERM或Ctrl-C时
可以优雅地关闭。

```
node.Start()
defer func() {
	node.Stop()
	node.Wait()
}()

c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt, syscall.SIGTERM)
<-c
os.Exit(0)
```

## 7、项目依赖管理、构建、配置生成和启动

我们使用go module进行项目依赖管理：

```
$ go mod init hubwiz.com/tendermint-go/demo
$ go build
```

上面的命令将解析项目依赖并执行构建过程。

要创建默认的配置文件，可以执行`tendermint init`命令。但是
在开始之前，我们需要安装Tendermint Core。

```
$ rm -rf /tmp/example
$ cd $GOPATH/src/github.com/tendermint/tendermint
$ make install
$ TMHOME="/tmp/example" tendermint init

I[2019-07-16|18:40:36.480] Generated private validator                  module=main keyFile=/tmp/example/config/priv_validator_key.json stateFile=/tmp/example2/data/priv_validator_state.json
I[2019-07-16|18:40:36.481] Generated node key                           module=main path=/tmp/example/config/node_key.json
I[2019-07-16|18:40:36.482] Generated genesis file                       module=main path=/tmp/example/config/genesis.json
```

现在可以启动我们的一体化Tendermint Core应用了：

```
$ ./demo -config "/tmp/example/config/config.toml"

badger 2019/07/16 18:42:25 INFO: All 0 tables opened in 0s
badger 2019/07/16 18:42:25 INFO: Replaying file id: 0 at offset: 0
badger 2019/07/16 18:42:25 INFO: Replay took: 695.227s

E[2019-07-16|18:42:25.818] Couldn't connect to any seeds                module=p2p
I[2019-07-16|18:42:26.853] Executed block                               module=state height=1 validTxs=0 invalidTxs=0
I[2019-07-16|18:42:26.865] Committed state                              module=state height=1 txs=0 appHash=
```

现在可以打开另一个终端，尝试发送一个交易：

```
$ curl -s 'localhost:26657/broadcast_tx_commit?tx="tendermint=rocks"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "check_tx": {
      "gasWanted": "1"
    },
    "deliver_tx": {},
    "hash": "1B3C5A1093DB952C331B1749A21DCCBB0F6C7F4E0055CD04D16346472FC60EC6",
    "height": "128"
  }
}
```

响应中应当会包含交易提交的区块高度。

现在让我们检查指定的键是否存在并返回其对应的值：

```
$ curl -s 'localhost:26657/abci_query?data="tendermint"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "response": {
      "log": "exists",
      "key": "dGVuZGVybWludA==",
      "value": "cm9ja3M="
    }
  }
}
```

“dGVuZGVybWludA==” 和“cm9ja3M=” 都是base64编码的，分别对应于
“tendermint” 和“rocks” 。

## 8、小结

在这个指南中，我们学习了如何使用Go开发一个内置Tendermint Core
共识引擎的区块链应用，源代码可以[从github下载](https://github.com/ezpod/tendermint-go-demo)，如果希望进一步系统
学习Tendermint的应用开发，推荐汇智网的[Tendermint区块链开发详解](http://xc.hubwiz.com/course/5bdec63ac02e6b6a59171df3?affid=blog7878)。

---
原文链接：[Tendermint Core应用开发指南 —— 汇智网](http://blog.hubwiz.com/2020/01/03/tendermint-app-golang/)

