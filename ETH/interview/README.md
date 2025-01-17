# 面试题

## 区块链底层

### 数据结构

#### UTXO

Q：什么是 UTXO？



Q：为什么比特币需要 UTXO（它实际解决了什么问题）？



Q：UTXO 跟账户模型的区别与联系？





#### Merkel Tree 及其变种

Q：讲讲 Merkel Tree 的工作原理



Q：讲讲 Merkel Tree 的实现方式



Q：Merkel Tree 有哪些应用？



### 密码学

#### 哈希算法

Q：什么是哈希冲突？如果出现哈希冲突，意味着什么？

A：哈希冲突意味着两个不同的值 x 与 y，能够得到 Hash(x) == Hash(y)，这便是哈希冲突。在哈希表中，如果出现哈希冲突，则会导致旧值被覆盖。



Q：你能实现一个简单的哈希表吗？

A：[参见此文](https://github.com/TheStarBoys/implement-algorithms-with-golang/blob/master/book/dataStructure/hashTable.md)

#### 加密算法

Q：什么是对称加密？



Q：什么是非对称加密？



Q：如果业务要求较高安全性的同时，兼顾高效率，请你结合对称加密与非对称加密设计一个加密方案来满足需求。



Q：什么是数字签名？



Q：讲一讲什么是 RSA



Q：讲一讲什么是 ECDSA



### 共识

#### 非拜占庭容错算法

Paxos

Raft

#### 拜占庭容错算法

PBFT

Proof of Work

Proof of Stake

Delegated Proof of Stake

Proof of Activity

Proof of Reputation

### 工程

**Q：当你调用某区块链节点的 api 时，发现返回值一会儿正常，一会儿错误。例如，你去获取当前区块号，有时返回正确的值，有时候返回的值比该值要小，有时候没有响应。请分析原因，并给出解决方案。**

A：该现象是由负载均衡导致的。你的每个请求每次都负载均衡的分配给了某一个机器，当有些机器出现故障时，就会发生该现象。方案一：自己维护一个全节点，简单粗暴，但坏处是维护成本高。方案二：使用付费服务，例如 infura，相对比较稳定，费用比维护全节点低，如果 api 请求较少，则费用可能为零。方案三：每隔一段时间，通过调用类似获取当前最新区块号的 api，比较单/多个公开节点返回值的差异，并与 infura 的接口数据比对，选择表现更好的节点。



Q：什么是拜占庭容错算法？



Q：简单讲一讲 PoW 的实现思路



**Q：PoW 的分叉选择规则是什么（如何确定最长合法链）？**

A：工作量最大的那条链为最长合法链。



Q：讲一讲 PoS 的工作原理



Q：PoS 的分叉选择规则是什么（如何确定最长合法链）？



Q：PoS 与 DPoS 的相同点与不同点是什么？



**Q：DPoS 中如果一个节点同时在两个分叉投票，如何惩罚？**

A：投票行为需要节点进行签名，当检举者收到一个投票广播，发现该投票人已经投过票，即此时检举者收到了两份投票的证明，将此证明包含进区块链，即可触发惩罚。



## 智能合约

### 开发

Q：分别讲一讲 storage、memory、calldata 这几个存储位置



Q：call、delegate_call、static_call、call_code 你的理解是？



Q：如何实现一个 Iterable Mapping？



Q：send()、transfer()的区别与联系



Q：讲一讲你对 private 关键字的理解？



Q：private 修饰的状态变量无法直接调用接口获取到，因此存储一些敏感数据是安全的。这句话是对还是错？



Q：require、assert、revert 的作用分别是什么？



Q：external 函数与 public 函数的区别与联系？



Q：合约如何接收 Ether 转入



Q：了解过汇编吗？



Q：合约地址是如何得出的？



Q：对 abi 了解多少？



Q：你能实现一个安全的随机数合约吗？



Q：事件加 index 与不加有什么区别？什么情况该加，什么情况不加？



Q：列举一下哪些情况下，合约调用会失败。



Q：了解过 opcode 吗？



Q：如何实现可升级合约？



Q：如何实现 Proxy？



Q：

### 测试

Q：你有为合约编写过测试吗？你是如何进行测试的？





### 部署

Q：你如何部署你的合约的？



Q：如果你确信你的合约没有 bug，部署脚本也没有问题，什么情况可能会导致部署失败（部署时提示 XXX ran out of gas）



Q：知道如何验证合约吗？



### 优化

**Q：你了解过合约优化吗？一般的优化思路是什么？**

A：



[优化详情](../smart_contract/Optimization.md)

### 安全性

Q：什么是重入攻击？



Q：如何避免整数溢出？



Q：你用过哪些安全检测工具？



[安全性详情](../smart_contract/Security/)

### 标准与协议

#### ERC20

**Q：你知道 ERC20 是什么吗？**

A：



**Q：你一般如何使用 `transfer` 函数（如何使用更安全）？**

A：在调用该函数前，需要遵循 Checks-Effects-Interactions 原则。由于标准的定义，`transfer` 函数有一个返回值，来返回 token 是否转移成功。判断该返回值，若为 false，则需要 revert。



**Q：你对 ERC20 的一些扩展了解多少？**

A：ERC20Permit，允许使用签名的方式进行 `approve` 。ERC20Votes，允许记录和查询历史持币数量，以便用于 DAO 投票治理。



**Q：对任意 ERC20 合约调用 name()、symbol()、decimals() 是什么现象？**

A：一部分合约的返回值是正常的，一部分是异常的。因为这三个接口并不是必定会实现的，因此，假设你使用 js 调用这些函数，而目标合约并未实现时，可能会抛出异常。



[ERC20详情](../EIPs/ERC/ERC20.md)

#### ERC721

**Q：ERC721 标准中的 tokenUri 决定了其元数据存储位置，能讲讲为什么不直接存储在链上吗？常见的数据存储方式有哪些？它们各自的优点以及缺点？**

A：由于链上的所有数据，所有全节点都会存储，存储成本高昂，NFT 往往有大量的描述性数据，因此往往存储在链下。一般来说，有三种方式：1. 存储在中心化服务器中，例如 google 空间 2. 存储在 IPFS 中 3. 存储在 arweave 链上。中心化服务器成本低，方便修改元数据，但有单点故障、恶意篡改等风险。IPFS是分布式存储，根据内容寻址，因此不用担心元数据被篡改，但它本身没有激励层，不能保证它存储的数据不会丢弃。arweave 链采用 Proof of Access 共识，实现了数据永久存储。



[ERC721详情](../../Introduction/004 什么是 NFT.md)

#### ERC1155



#### ERC712



#### Uniswap



#### IPFS

**Q：IPFS 上 cid 对应的数据是不可篡改的吗？**

A：是的，cid 就是哈希值，一旦数据哪怕做很小的改动，cid 也是截然不同的，因此不可篡改。



**Q：IPFS 在上传文件时，会分散的自动备份多份吗？**

A：不是，IPFS 协议本身不提供副本功能。



**Q：IPFS 上存储的文件会永久存在吗？**

A：不会，IPFS 只聚焦于内容寻址，不承诺数据永存。需要类似 FileCoin 这样的激励层帮助实现指定存储时长。



**Q：普通家用电脑（无公网 IP）成为 IPFS 节点后，上传文件到自己本地 IPFS 节点后，能够通过 https://ipfs.io/ipfs/xxx 直接访问到你上传的文件吗？**

A：不能，没有公网 IP 会导致其他节点无法发现你的节点，进而在检索文件时无法找到对应内容。



**Q：IPFS 上能发布网页吗？如果能，能支持哪些网页？**

A：能发布，只支持静态页面。



**Q：NFT 中的 IPFS 承担了什么角色？**

A：NFT 数据/元数据存储的地方，以减轻链上存储成本。NFT 的 tokenUri 会设置为对应元数据的 ipfs 链接，从而保证元数据不可篡改。



### 应用

#### Yield Farming

#### [DAO](../../Introduction/017 什么是DAO.md)

#### [元交易](../smart_contract/MetaTx/元交易（Metatransaction）系列一，什么是元交易？.md)

#### 多重签名钱包

#### 预言机

**Q：预言机是什么？**

A：预言机是真实世界与区块链的桥梁。它们扮演一个链上 API 的角色，以查询链下数据。这些数据可以是价格信息、天气报告、体育比赛结果等。

**Q：讲一讲简单的预言机的架构实现**

A：下面我将讲述一个简单的示例，但显然有更多触发链下计算的方式：

1. 在你的合约上发出一个事件
2. 链下服务订阅这些事件
3. 链下服务根据事件执行某些任务
4. 链下服务发起新的交易，带上请求的数据，以响应智能合约。

更进一步的话，可能是建立这些节点的网络，对不同的api和源进行调用，并聚合链上的数据。



#### 跨链

#### 侧链

**Q：什么是侧链？**

A：侧链是与以太坊主网并行运行并独立运行的独立区块链。它有自己的[共识算法](https://ethereum.org/en/developers/docs/consensus-mechanisms/)（例如 Proof of Authority、Delegated Proof of Stake，拜占庭容错算法）。它通过双向网桥连接到主网。它与 EVM 兼容，不受 Layer 1 保护（因此，严格来说，它不是 Layer 2）。



#### 事件索引服务



### 工程

**Q：你有使用过哪些 Solidity 开发框架？你一般使用什么框架，为什么？（它们的优点与缺点？）**

A：主要使用的框架是 Truffle 与 Hardhat。Truffle 的 console 让你更容易与合约交互，Boxes 生态有助于快速构建 DApp，是**快速开发 DApp 的不错选择**。Hardhat 有丰富的插件体系、更好的类型支持（TypeScript），**更适合专业人士使用**。小型的合约项目，我一般使用 Truffle，更复杂的项目，优先选用 Hardhat。



**Q：当你在 mainnet 使用合约 call 时，发现总是 revert，你确信合约代码没错，那么可能是什么原因导致的以及如何解决？**

A：首先确定你使用 call 操作时，from 指定的地址是什么，查询其 ETH 余额是否足够一次 call 调用。如果余额不足，call 会由于 gas 不够而失败。两种办法：1. 给该地址转足够的 ETH 2. 如果这个 call 的结果跟 from 无关的话，可以在 etherscan 上找到一个有足够余额的地址，替换原来的 from 即可



**Q：你在有的链上调用合约失败会带有 require 函数中的提示信息，而有的链没有，这是为什么以及如何解决？**

A：考虑是目标链版本过低，这是曾经的一个 go-ethereum 的 [issue 为21083](https://github.com/ethereum/go-ethereum/pull/21083) ，并在该 [commit](https://github.com/ethereum/go-ethereum/commit/0b3f3be2b5dde72c6292bfb16915ad763d4aa0bd) 中被修复。原因是老版本不支持返回 revert 时的错误信息。



**Q：在智能合约中实现一个安全的随机数是很困难的，目前有解决方案吗？**

A：Chainlin VRF（Verifiable Random Function可验证随机函数）是为智能合约设计的可证明公平和可验证的随机性来源。



**Q：EVM 链的智能合约不能在任意时间或任意条件下触发或执行自己的函数。只有当另一个帐户发起交易（例如用户、oracle或合约）时才会发生状态更改。如何才能做到让智能合约在任意时间或任意条件下触发或执行自己的函数？**

A：直接参见 [Chainlink Keepers](https://docs.chain.link/docs/chainlink-keepers/introduction/) 文档。



**Q：truffle 出现错误 “Transaction exited with an error (status 0) after consuming all gas” 如何解决？**

A：[参考](https://ethereum.stackexchange.com/questions/71481/transaction-exited-with-an-error-status-0-after-consuming-all-gas)，硬编码 gas、gasPrice 值。