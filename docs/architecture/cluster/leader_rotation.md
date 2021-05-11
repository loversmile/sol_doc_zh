## Leader 轮询
在任何给定时刻，一个集群只需要一个Validator 来产生分类账条目。由于一次只有一个leader，所有的validator 都能够重放相同的账本副本。然而，每次只有一个领导的缺点是，一个恶意的领导有能力审查投票和交易。然而，每次只有一个领导的缺点是，一个恶意的领导有能力审查投票和交易。由于审查和网络丢包不能区分，集群不能简单地选出一个节点无限期地担任领导角色。相反，集群通过轮询哪个节点带头来最小化恶意leader的影响。

每个Validator使用相同的算法选择预期的领导者，如下所述。当Validator接收到一个新的签名分类账entry 时，可以确定该entry是由预期的领导产生的。每个领导被分配一个slot 的顺序被称为 *leader schedule*。

### Leader 时间表轮询(Leader Schedule Rotation)
Validator 拒绝没有由 *slot leader*签名的区块。所有*slot leader*的身份列表称为*leader schedule*。leader schedule 被局部周期性地重新计算。它在一段称为epoch的时间内分配slot leader。 进度表必须在它分配的 slot 之前计算，以便最终确定其用于计算进度表的分类账状态。这个持续时间称为*leader schedule offset*。Solana将 slot 到下一个epoch 的持续时间设置为偏移量(offset)。也就是说，一个epoch 的 leader schedule 是从前一个纪元开始的分类账状态计算出来的。一个epoch的偏移量是任意的，并且假设足够长，以便所有Validator 在生成下一个进度表之前最终确定他们的分类账状态。集群可以选择缩短偏移量，以减少stake(权益)变更和leader调度更新之间的时间间隔。

在不使用分区持续时间超过epoch的情况下，只需要在root fork 跨越epoch边界时生成调度。因为时间表是为下一个epoch 准备的，所以任何提交给root fork的新stake 在下一个epoch 之前都不会被激活。用于生成leader调度的块是第一个跨越epoch 边界的块。

如果没有一个分区持续的时间超过一个epoch，集群的工作方式如下:
1. Validator 在投票时不断更新自己的根分支。
2. 每次slot 高度超过epoch边界时，Validator就更新它的leader schedule。

例如：

epoch持续时间为100个slot 。根分叉从槽高度99处计算的fork更新到槽高度102处计算的fork。slot 高度为100、101的fork由于失败而被跳过。新的leader调度是使用slot 高度102的fork计算的。从第200位开始，直到再次更新。

不可能存在不一致，因为与集群一起投票的每个验证器在根通过102时都跳过了100和101。所有验证器，无论投票模式如何，都将提交给102或102的后代。

#### Leader Schedule Rotation with Epoch Sized Partitions(使用Epoch大小的分区的Leader调度 )
leader调度偏移的持续时间直接关系到集群拥有不一致的正确leader调度视图的可能性。

考虑以下场景:

### Leader Schedule Generation at Genesis

### Leader Schedule Generation Algorithm

### Schedule Attack Vectors
#### Seed
#### Active Set
#### Staking
#### Validator operational key loss

### 追加 Entries