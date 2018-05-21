# HL.Develop

## Hyperledger Fabric结构体系说明

Hyperledger Fabric架构具有以下优点：

- Chaincode信任灵活性。
  该体系结构将chiancodes（区块链应用）的信任假设与ordering信任假定相分离。换句话说，ordering 服务由专用的一组节点提供，并可以容忍其中的一些失败或者不当的行为，并且对于每个chaincode代码，其信任背书者可以是不同的。

- 可扩展的。
  作为负责特定chaincode的背书策略的节点与orderers 节点垂向沟通，所以这套系统可以比那些功能集中在一个节点上的情况更容易扩展。特别是当不同的chaincode代码分容器执行，这将可以允许不同背书策略的（被认可）节点调用同一chaincode容器。

- 保密。
  该体系结构有助于部署，具有*关于其交易的内容*和*状态更新*的机密性要求的chaincode代码。

- 共识模块化。
  这个结构通过orderers 服务实现可插拔式的共识。

## 1. 系统构架

对于管理功能和参数, 可能存在一个或多个特殊 chaincodes, 统称为 '系统 chaincodes'。

## 1.1 Fabric 交易

>在资产交易过程中的事务机制

首先，Transactions有两种类型（更多类型基于这两种）：

1. 部署交易（Deploy transactions）
  创建新的chaincode并以这个程序作为参数。当部署交易成功执行时，这个chaincode已经被安装到了这个区块链上。
2. 调用交易（Invoke transactions）
  在已经部署的chaincode程序上执行操作。一个invoke交易会调用一个chaincode和它提供的函数之一。调用成功时，由chaincode执行指定的功能（涉及修改相应的状态并return an output）

部署交易是调用交易的特例，其中的部署创建的新chaincode代码，对应于在系统chaincode代码中调用交易。

## 1.2 区块链数据结构

### 1.2.1 状态表示

  KVS(key-value store) K -> (V X N)

N 是一个单向无限的版本号数字。 单射函数 next: N -> N 取一个元素的N并返回下一个版本。

在 N 的最低元素的情况下，V 和 N 都包含一个特殊元素⊥ (空类型)。

最初所有的key都映射到 (⊥, ⊥)。对于 s(k) = (v, ver), 我们用 s(k).value 表示 v，用 s(k).version 表示ver

由peers 维护状态，不由orderers 和 clients 维护。

KVS 中的键可以从它们的名称识别为属于特定的 chaincode, 在某种意义上, 只有某一 chaincode 的交易可以修改属于此 chaincode 的键。原则上, 任何 chaincode 都可以读取属于其他 chaincodes 的键。

>post-v1 将支持跨 chaincode 事务, 即修改属于两个或多个 chaincodes 的状态。

### 1.2.2 Ledger账目

Ledger账目提供了所有成功状态更改 (valid transactions) 的可验证历史记录, 以及在系统运行期间发生的更改状态的失败尝试 (invalid transactions) 。

Ledger账目由orderer服务, 作为完全有序的 (valid有效 或 invalid无效) transactions的 hashchain。hashchain 在分类账中强加块的总顺序, 每个块包含一个完全有序事务的数组。这会在所有交易记录中强加总订单。

Ledger保存在所有的 peer 或者在orderers的子集中。在orderer的内容中引用的Ledger被称作`OrdererLedger`。在peer的内容中引用的Ledger被称作`PeerLedger`。

PeerLedger 与 OrdererLedger 的不同之处在于, Peer 在本地维护一个位掩码, 它将有效事务从无效的交易记录中分离出来。

在post-v1 版中，Peer 可以删减PeerLedger。Orderers 为了容错性和对于PeerLedger的可用性，保存维护OrdererLedger，并可以决定在ordering服务属性被维护时随时删减OrdererLedger。

## 1.3 节点

节点是 blockchain 的通信实体。一个“节点”只是一个逻辑函数, 在这个意义上, 不同类型的多个节点可以在同一物理服务器上运行。重要的是节点在 '信任域' 中的分组方式, 以及与控制它们的逻辑实体的关联。

有三种类型的节点：

1. Client: 向背书者提交实际交易调用, 并将事务建议广播到orderer服务。

2. Peer: 提交交易并维护该PeerLedger的状态和副本的节点。此外, Peer可以有一个特殊的背书者角色。

3. Orderer: 运行确保传递(如原子或总订单广播)的通信服务的节点。

### 1.3.1 Client

客户端Client表示代表最终使用者的所有操作。它必须连接Peer以与 blockchain 通信。客户端Client可以连接到其选择的任何Peer。客户端创建并从而调用交易(invoke transactions)。

### 1.3.2 Peer

Peer以块(blocks)的形式从orderer服务接收有序状态更新, 并维护状态和Ledger。

Peer还可以担当一个 endorsing peer 或一个 endorser 的特殊角色。Endorsing peer的特殊功能发生在特定的 chaincode 上, 并在提交交易之前支持它。每个 chaincode 都可以指定一个背书策略, 可以引用一组Endorsing peer。该策略定义了有效交易背书 (通常是一组endorser签名) 的必要和充分条件(the necessary and sufficient conditions)。

在安装新 chaincode 的部署事务的特殊情况下, (部署) 背书策略被指定为系统 chaincode 的背书策略。

### 1.3.3 Ordering service nodes (Orderers)

Orderers 构成Ordering service, 即提供交付保证的通讯结构。Ordering service可以采用不同的方式实现: 从集中式服务 (例如开发和测试) 到针对不同网络和节点故障模型的分布式协议。

Ordering service为Client和Peer提供了一个共享的通信通道, 为包含交易的消息提供广播服务。Client连接到channel, 并可以在channel上广播消息, 然后将其传递给所有Peer。该channel支持所有消息的原子(atomic)传递, 即消息通信与总订单传递和(具体实现)可靠性。换言之, channel向所有连接的Peer输出相同的消息, 并以相同的逻辑顺序输出到所有Peer。这种原子通信保证在分布式系统的语境中也称为*全序广播total-order broadcast*、*原子广播atomic broadcast*或*协商一致consensus*。所传递的消息是包含在 blockchain 状态中的候选交易。

*分区Partitioning* (Ordering service channels)。Ordering service可能支持多个channels, 类似于发布/订阅 (pub/sub) 信息系统的*topics*。Client可以连接到给定的channel, 然后可以发送消息并获取到达的消息。channel可以被认为是分区-连接到一个通道的客户端不知道其他通道的存在, 但客户端可以连接到多个通道。尽管 Hyperledger 结构中包含的一些Ordering service实现支持多个通道, 但为了简单起见, 在本文的其余部分中, 我们假定Ordering service由单个通道/主题组成。

#### **Ordering service API**

Peer通过Ordering service提供的接口连接到Ordering service提供的channel通道。Ordering service API 由两个基本操作 (通常是异步事件) 组成:

1. **TODO** 添加用于在Client/Peer指定序列号下获取特定Block的 API 部分。

    - `broadcast(blob-数据块的二进制形式)`: Client调用此项来广播任意消息 `blob` 以在channel上传播。向服务发送请求时，也在BFT-二进制文件传输（Binary File Transfer）环境中被称为`request(blob)`
    - `deliver(seqno, prevhash, blob)`: orderer 在Peer上调用此功能来传递消息，以便使用指定的非负整数序列号（seqno）和最近交付的blob（prevhash）的散列来传递消息blob。换句话说，它是来自Ordering service的输出事件。 deliver（）有时也被称为pub-sub系统中的notify()或BFT-二进制文件传输（Binary File Transfer）系统中的commit()。

#### Ledger 和形成区块

通过一系列事件，`deliver(seqno, prevhash, blob)`其中 prevhash 来自于上一个区块。

#### Ordering 服务的属性

1. 安全性（持续保证）

2. 活跃性（传输保证）

## 交易流程

## 2.1 客户端创立一笔交易和将它发送到所选择的背书节点

Invoke 一笔交易，客户端发送一条 PROPOSE 消息选择一组认可的peers，对于给定chaincodeID的一组 endorsing peers 经由peer而被提供给客户机，该peer又从认可政策中知道一组标识peer

## 交易流

为了确保数据的一致性和完整性，Hyperledger Fabric在整个交易流程中实施了多个检查点，包括客户端认证-`client authentication`，背书-`endorsement`，`ordering`和对ledger的承诺。

在这个场景中假设有两个客户，A和B，在进行橙子买卖。这两个客户各自有自己的网络节点（Peer），通过他们的节点发送交易并且和账本进行互动。

### 假设 (Assumptions)

这个流程假设了channel已经被建立并运行，这个应用程序的用户已经被注册和加入了组织，拥有CA证书并且获得了必要的可用于在网络中提供身份验证的加密材料。

这个chaincode已经被安装在peers 中并且被在此channel中实例化。

在这个chaincode中已经包含了橙子市场的初始化键值对和商业逻辑，设置了认可策略，规定peerA 和 PeerB 必须认可任何交易。

### 1. 客户A 发起交易

客户 A 发送了一个购买橙子的请求，这条请求的目标是 PeerA 和 PeerB ,他们分别代表客户A和客户B 根据背书策略要求两个peer 必须支持任何交易，因此这条请求会发送到这两个节点。

接下来这个交易提案被构建。利用受支持的SDK（Node，Java，Python）的应用程序，会用生成交易提议的可用API之一。该提议是调用chaincode代码的函数，以便数据可以被读取/写入账本（即为资产编写新的键值对）的请求。 SDK可用作将交易提议打包为正确架构的格式（通过gRPC的协议缓冲区）并采用用户的加密凭证为此交易提议生成唯一签名。

### 2. 赞成peer节点验证的签名并且执行这笔交易

Endorsing peers要验证 (1) 交易提交格式良好, (2) 以前尚未提交 (重播攻击保护), (3) 签名有效 (使用 MSP), (4) (如示例中的客户端 A) 已正确授权在该通道上执行建议的操作 (即, 每个Endorsing peers都确保提交者满足信道的编写者策略)。

Endorsing peers将transaction proposal输入作为参数提供给调用的 chaincode 函数。然后, 对当前状态数据库执行 chaincode, 以生成包括响应值、读集和写集在内的事务结果。此时未对ledger进行更新。这些值的集合以及Endorsing peers的签名将作为 'proposal response' 传递给 SDK, 分析要使用的应用程序的有效负载。

>MSP是一个peer组件，它允许同行验证来自client的交易请求并签署交易结果（endorsements）。书写策略在频道创建时定义，并确定哪些用户有权向该频道提交交易。

### 3. 对proposal response的检查

应用程序验证Endorsing peers签名并比较proposal response, 以确定方案响应是否相同。如果 chaincode 只查询ledger, 则应用程序将检查查询响应, 通常不会将事务提交到ordering服务。如果客户端应用程序打算将事务提交到ordering服务以更新ledger, 则应用程序将确定, 指定的背书策略在提交前是否已完成 (即peerA 和 peerB 都认可)。体系结构是这样的, 即使应用程序选择不检查响应或以其他方式转发 unendorsed 事务, 背书策略仍将由peer执行, 并在提交验证阶段得到维护。

### 4. Client将背书组装成交易

应用程序将“交易消息”中的交易提议和响应“广播”给 Ordering Service。该事务将包含读/写集，the endorsing peers签名和通道ID。Ordering Service不需要检查交易的全部内容以便执行其操作，它仅接收来自网络中所有channel的交易，按channel按时间顺序排序，并为每个channel创建交易块。

### 5. 交易已经过验证并提交

交易的区块被“delivered”给channel上的所有peer。对区块中的交易进行验证，以确保批注策略得到满足，并确保读取集变量的分类账状态没有变化，因为读取集是由事务执行生成的。块中的事务被标记为有效或无效。

### 6. 账本更新

每个peer将块追加到通道的链中, 对于每个有效交易, 写入集都提交到当前状态数据库。发出事件, 通知客户端应用程序将交易 (调用) 一成不变追加到链中, 并通知该事务是否已验证或无效。