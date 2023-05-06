流程：
定序器以MATIC代币的形式支付费用，获得创建和提议批次的权利。
聚合器从定序器接收所有交易信息，并将其发送给Prover，Prover在复杂的多项式计算后提供一个小的zk-Proof。
智能合约验证此证明。

交易生命周期：
桥：存入以太币，等待**globalExitRoot**在L2上发布，然后在L2进行认领资金
L2交易：用户在钱包发起交易并发送给定序器，定序器承诺添加该交易并在L2上完成，此时L1上还未完成（这时称为**可信状态**），定序器将批次数据发送到L1智能合约（这时称为**虚拟状态**），聚合器对未决交易进行验证并构建证明实现L1的最终确定性，此时证明得到验证，用户交易达到L1最终确定性（这时称为**合并状态**）


PolygonZkEVM.sol合约sequencedBatches函数必须包含一个globalExitRoot存在于桥的L1合约PolygonZkEVMGlobalExitRoot.sol的GlobalExitRootMap中。只有包含有效的globalExitRoot，批次才有效。

As L2 Network mentioned, all L2 network history can be recomputed from L1 smart contracts. So, for the Polygon ZkEVM case, how do I get the transaction history of L2 transactions from the Polygon ZkEVM contract on L1?
