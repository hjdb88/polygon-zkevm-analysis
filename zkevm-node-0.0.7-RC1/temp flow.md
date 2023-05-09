# Polygon zkEVM

## 流程整理

### Sequencer（定序器）

1. createSequencer，Start（cmd/run.go）: 启动定序器
   1. 判断是否同步完成（sequencer/sequencer.go#isSynced） 
   2. 启动dbManager协程（sequencer/dbmanager#Start）
      1. 不断从池中加载交易（sequencer/dbmanager#loadFromPool）
         1. <font color="red">待补充</font>
      2. 检查是否发生重组（sequencer/dbmanager#checkIfReorg）
      3. 将tx存储到状态中并更改它在池中的状态（sequencer/dbmanager#storeProcessedTxAndDeleteFromPool）
         1. 循环处理
         2. 获取数据，txToStore := <-d.txsStore.Ch <font color="red">待补充数据来源</font>
         3. 检查是否发生重组
         4. 刷新状态数据库
         5. 在状态中存储一笔交易（sequencer/dbmanager#StoreProcessedTransaction）
         6. 更新批次L2数据（state/pgstatestorage.go#UpdateBatchL2Data）
         7. 将交易池中对应交易状态更改为选中（pool/pool.go#UpdateTxStatus）
   3. 启动终结器（sequencer/finalizer.go#Start）
      1. 关闭信号接收器（sequencer/finalizer.go#listenForClosingSignals）
         1. <font color="red">待补充</font>
      2. 处理交易并完成批次（sequencer/finalizer.go#finalizeBatches）
         1. 循环处理
         2. 获取适合可用批次资源的最高效tx（sequencer/worker.go#GetBestFittingTx） <font color="red">待补充详细规则</font>
         3. tx不为空，执行单笔交易处理逻辑（sequencer/finalizer.go#processTransaction）
            1. <font color="red">调用Prover的executor处理一个批次</font>，返回NewStateRoot、NewAccInputHash等信息（state/state.go#ProcessBatch）
               1. 使用grpc调用Prover的executor模块的ProcessBatch方法
            2. 执行成功，处理交易执行结果（sequencer/finalizer.go#handleTxProcessResp）
               1. 处理交易错误
               2. 检查剩余资源，校验交易使用的资源是否少于批处理中的剩余资源（sequencer/finalizer.go#checkRemainingResources）
               3. 存储已处理的交易，将其添加到批处理中并以原子方式更新池中的状态（sequencer/finalizer.go#storeProcessedTx）
                  1. 存储处理后的交易，f.txsStore.Ch <- &txToStore{...}
                  2. 从效率列表中删除交易（sequencer/worker.go#DeleteTx）
                  3. 在Executor上执行成功tx后更新地址信息（sequencer/worker.go#UpdateAfterSingleSuccessfulTxExecution）
                  4. 更新交易状态（sequencer/dbmanager#UpdateTxStatus）
            3. 更新内存中的批次和processRequest
         4. tx为空，等待新tx
         5. 遇到任何关闭信号的最后期限或者批次交易数量已满或者当前批次剩余资源在最有效时刻关闭批的约束阈值范围内（满足isDeadlineEncountered或者isBatchFull或者isBatchAlmostFull时执行sequencer/finalizer.go#finalizeBatch）
            1. 重试直到成功关闭当前批次并打开一个新批次，可能会在批次关闭和生成的新空批次之间处理强制批次
   4. 关闭信号管理器（sequencer/closingsignalsignal#Start）
      1. checkForcedBatches
      2. checkGERUpdate
      3. checkSendToL1Timeout
   5. trackOldTxs
   6. 尝试发送批次（sequencer/sequencesender.go#isSynced#tryToSendSequence）
      1. 在开始下一个循环之前处理监控的交易（#ProcessPendingMonitoredTxs）
      2. 检查同步器是否是最新的（sequencer/sequencer.go#isSynced）
      3. 检查是否应该将序列发送到L1（sequencer/sequenccer.go#getSequencesToSend）
      4. 构建要发送到 PoE 智能合约方法SequenceBatches的 []bytes 数据，并发送到L1（etherman/etherman.go#BuildSequenceBatchesTxData）
         1. 调用合约方法sequenceBatches（etherman/etherman.go#sequenceBatches）
      5. 将序列添加到ethTxManager，ethTxManager放置要发送和监控的交易（ethtxmanager/ethtxmanager#Add）
   7. 将worker中太旧的交易过期（sequencer/worker.go#ExpireTransactions、pool/pool.go#UpdateTxStatus）

### Aggregator（聚合器）

1. runAggregator: 启动聚合器
   1. ProcessPendingMonitoredTxs: 开始前处理监控批次验证
   2. DeleteUngeneratedProofs: 删除未生成的递归证明
   3. cleanupLockedProofs
      1. CleanupLockedProofs: 从存储中删除锁定在生成状态并且超过设定阈值的证明
   4. sendFinalProof: 等待从证明者那里接收最终证明
      1. startProofVerification: 将verifyingProof变量设置为true表示正在进行证明验证
      2. GetBatchByNumber: 获取给定编号的批次
      3. BuildTrustedVerifyBatchesTxData: 向L1合约提交证明验证
      4. ProcessPendingMonitoredTxs: 在开始下一个周期之前处理受监控的批次验证
      5. resetVerifyProofTime: 更新超时以验证证明
      6. endProofVerification: 将verifyingProof变量设置为false表示没有正在进行的证明验证


1. Channel: 实现证明者客户端和聚合器服务器之间的双向通信通道
   1. tryBuildFinalProof: 检查提供的证明是否有资格用于构建最终证明
   2. tryAggregateProofs: 尝试聚合证明
   3. tryGenerateBatchProof: 尝试生成批次证明


1. tryBuildFinalProof: 检查提供的证明是否有资格用于构建最终证明
   1. 目前没有证明生成，检查是否有准备好验证的证明
      1. getAndLockProofReadyToVerify
      2. UpdateGeneratedProof
   2. 目前有证明生成，检查它是否有资格被验证
      1. validateEligibleFinalProof: 验证最终证明是否合格
   3. buildFinalProof: 构建并返回聚合批证明的最终证明
      1. FinalProof: 指示证明者为给定的输入生成最终证明。它返回正在计算的证明的ID
      2. WaitFinalProof: 等待证明者生成证明并返回证明者响应
   4. 通过channel返回证明结果


1. tryAggregateProofs: 尝试聚合证明
   1. AggregatedProof: 指示证明者从提供的两个输入生成聚合证明。它返回正在计算的证明的ID
   2. WaitRecursiveProof: 等待证明者生成递归证明并将其返回
   3. 通过删除2个聚合证明并存储新生成的递归证明来更新状态
   4. tryBuildFinalProof: 状态是最新的，请检查我们是否可以使用刚制作的证明发送最终证明
   5. UpdateGeneratedProof: 最终证明还没有生成，更新递归证明


1. tryGenerateBatchProof: 尝试生成批次证明
   1. getAndLockBatchToProve
   2. buildInputProver
   3. BatchProof: 指示证明者为提供的输入生成批量证明。它返回正在计算的证明的ID
   4. WaitRecursiveProof: 等待证明者生成递归证明并将其返回
   5. tryBuildFinalProof: 检查提供的证明是否有资格用于构建最终证明
   6. UpdateGeneratedProof: 最终证明还没有生成，更新批量证明
   7. DeleteGeneratedProofs: 从存储中删除落在批号范围内的生成证明


-----


### Synchronizer（同步器）流程

1. 启动同步器（cmd/run.go#runSynchronizer）
   1. 创建同步器并开启同步（synchronizer/synchronizer.go#Sync）
      1. 查询最后同步的以太坊区块lastEthereumBlock，如果没有lastEthereumBlock意味着需要从头开始同步。如果不是，则继续从检索到的以太坊块获取最新的同步块。如果数据库上没有块，使用创世块
      2. 定时器
         1. 同步L1区块（synchronizer/synchronizer.go#syncBlocks）
            1. 检查是否存在重组（synchronizer/synchronizer.go#checkReorg）
            2. 存在重组则重置状态，并返回（synchronizer/synchronizer.go#resetState）
               1. 重置，即删除blockNumber之后的区块数据
               2. 将提供的blockNumber直到最新blockNumber的所有受监控的tx更新到Reorged状态（ethtxmanager/ethtxmanager.go#Reorg）
            3. 调用以太坊区块链检索数据
            4. 查找包含在以太坊块中的rollup信息和一个名为order的额外参数，解析并处理event log（GetRollupInfoByBlockRange）
            5. 处理区块，使用状态将新信息包含到数据库中（processBlockRange）
               1. etherman.SequenceBatchesOrder类型: synchronizer/synchronizer.go#processSequenceBatches
               2. etherman.ForcedBatchesOrder类型: synchronizer/synchronizer.go#processForcedBatch
               3. etherman.GlobalExitRootsOrder类型: synchronizer/synchronizer.go#processGlobalExitRoot
               4. etherman.SequenceForceBatchesOrder类型: synchronizer/synchronizer.go#processSequenceForceBatch
               5. etherman.TrustedVerifyBatchOrder类型: synchronizer/synchronizer.go#processTrustedVerifyBatches
               6. etherman.ForkIDsOrder类型: synchronizer/synchronizer.go#processForkID
         2. 获取最后定序的批次和最后同步的批次，判断最后同步批次大于等于最后定序批次，则L1状态完整同步完成。
         3. 当节点同步了来自L1的所有信息时，从与可信状态相关的可信定序器同步信息（synchronizer/synchronizer.go#syncTrustedState）
            1. 处理可信的批次数据（synchronizer/synchronizer.go#processTrustedBatch）
               1. 检查批次是否需要同步
               2. 从数据库中删除编号大于给定批次的批次（state/pgstatestorage.go#ResetTrustedState）
               3. 由定序器用来将交易处理成一个开放的批次（state/state.go#ProcessSequencerBatch）
                  1. 最终调用Prover的executor模块执行交易并返回结果
               4. 由定序器用于将已处理的交易添加到打开的批处理中（state/state.go#StoreTransactions）


-----


### 其他

1. 查询rollup信息（etherman/etherman.go#GetRollupInfoByBlockRange）
   1. etherman/etherman.go#processEvent
      1. sequencedBatchesEventSignatureHash事件: ehterman/etherman.go#sequencedBatchesEvent
         1. 
      2. updateGlobalExitRootSignatureHash事件: ehterman/etherman.go#updateGlobalExitRootEvent
         1.
      3. forcedBatchSignatureHash事件: ehterman/etherman.go#forcedBatchEvent
         1.
      4. verifyBatchesTrustedAggregatorSignatureHash事件: ehterman/etherman.go#verifyBatchesTrustedAggregatorEvent
         1.
      5. forceSequencedBatchesSignatureHash事件: ehterman/etherman.go#forceSequencedBatchesEvent
         1.
      6. updateZkEVMVersionSignatureHash事件: ehterman/etherman.go#updateZkevmVersion
         1.


# 备注

1. L2的交易数据提交到L1交易的InputData中
2. Sequencer（定序器）同步数据先从L1合约event log同步，然后才从可信定序器同步信息
3. Sequencer（定序器）从可信定序器同步的数据格式是types.Batch，可信定序器从它的PostgreSQL数据库查询得到（表state.batch、表state.transaction等）

   ```go
   // Batch structure
   type Batch struct {
       Number              ArgUint64           `json:"number"`
       Coinbase            common.Address      `json:"coinbase"`
       StateRoot           common.Hash         `json:"stateRoot"`
       GlobalExitRoot      common.Hash         `json:"globalExitRoot"`
       MainnetExitRoot     common.Hash         `json:"mainnetExitRoot"`
       RollupExitRoot      common.Hash         `json:"rollupExitRoot"`
       LocalExitRoot       common.Hash         `json:"localExitRoot"`
       AccInputHash        common.Hash         `json:"accInputHash"`
       Timestamp           ArgUint64           `json:"timestamp"`
       SendSequencesTxHash *common.Hash        `json:"sendSequencesTxHash"`
       VerifyBatchTxHash   *common.Hash        `json:"verifyBatchTxHash"`
       Transactions        []TransactionOrHash `json:"transactions"`
   }
   ```
4. state/state.go#StoreTransactions: StoreTransactions由定序器用于将已处理的交易添加到打开的批处理中。如果批处理已经有txs，则处理的Txs必须是现有Txs的超集，保持顺序。
5. Prover生成证明需要什么数据
