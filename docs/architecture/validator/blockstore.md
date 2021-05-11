## Blockstore
当一个区块到达终点后，从这个区块到起源区块的所有区块形成一个线性链，其名字为区块链。然而，在此之前，validator必须维护所有可能有效的链，称为fork。在 [fork generation](../cluster/fork_generation.html)中描述了由于领导者轮换而产生fork的过程。这里描述的块存储数据结构是验证器如何处理这些fork，直到块最终完成。

blockstore允许validator以任何顺序记录它在网络上观察到的每个碎片，只要碎片是由给定槽的预期领导者签署的。

分片被移动到一个可分叉的键空间，即领导槽+分片索引(槽内)的元组。这允许Solana协议的跳过列表结构被完整地存储，而不需要先验地选择要遵循哪个fork、持久化哪个Entries或者什么时候持久化它们。

对最近的碎片的修复请求从RAM或最近的文件中提供，从更深层的储存中取出近期的碎片，这是由支持Blockstore的存储实现的。

### blockstore 的功能
1. Persistence(持久化)：Blockstore位于节点验证管道的前端，就在网络接收和签名验证之后。如果收到的碎片与Leader时间表一致(即由Leader 签名为指定的插槽签署)，它将立即存储。

2. Repair(修复)：修复是和上面相同的窗口修复，但能够服务任何已收到碎片。blokckstore 存储签名碎片，保存起源链。

3. Forks：blockstore 支持随机访问碎片，因此可以支持Validator从Bank检查点回滚和重放的需要。

4. Restart：通过适当的修剪/剔除，Blockstore可以通过从slot 0有序枚举条目来重新播放。重放阶段的逻辑(即处理forks)将必须用于Blockstore中最新的条目。

### blockstore 设计
1. Blockstore中的Entries以键值对的形式存储，其中键是一个条目的连接 slot 索引和 shred 索引,值是entry 的数据。注意，每个slot 的 shred索引都是基于0的(即它们是相对的槽)。

2. Blockstore维护每个slot 的元数据，在```SlotMeta```结构中包含:
    - slot_index: slot 索引
    - num_blocks: slot 中 block 的数量(用于链接到前一个slot)
    - consumed: 最高的shred 指数n，使得对于所有```m < n```，在这个```slot```中存在一个shred指数等于n的shred(即最高的连续shred指数)。 
    - received: 该 slot 收到的最高 shred 索引。
    - next_slots: 这个slot 可以链接到的feature slot 的列表。在重新构建总账以查找可能的分叉点时使用。
    - last_index: 标记为此slot 的最后一个shred 的shred 索引。shred 上的标记将由 Leader 为一个 slot 设置，当他们为一个 slot 发送最后一个shred 时。
    - is_rooted: 仅当每个 block 从0开始，slot 形成一个没有洞的完整的序列的时候，为True。我们可以使用以下规则为每个槽派生is_rooted。假设slot(n)是索引为n的slot，如果索引为n的slot拥有该slot期望的所有ticks，则slot(n).is_full()为真。让is_rooted(n)语句为“slot(n).is_rooted是真的”。然后:
    is_rooted(0) is_rooted(n+1) iff (is_rooted(n) and slot(n).is_full()

3. Chaining - 当一个新的slot x的 shred到达时，我检查这个新 slot的区块(num_blocsk)(信息在shred中被编码)。然后我们知道这个新的slot链到slot ```x - num_blocks```。

4. Subscriptions(订阅)：Blockstore记录了一组已经“订阅”的slot。这意味着链接到这些 slot 的 entries 将被发送到Blockstore通道，供ReplayStage消费。查看```Blockstore APIs```了解更多。

5. Update notifications(更新通知)：blockstore 对于任意 n 在slot(n).is_rooted 从 false变成 true 时候通知到 listener。

### Blockstore APIs
Blockstore提供了一个基于订阅的API, ReplayStage用它来请求它感兴趣的entries。这些 entries 将通过Blockstore公开的通道发送。这些订阅API 类似于如下： 1. ```fn get_slots_since(slot_indexes: &[u64]) -> Vec<SlotMeta>```: 返回连接到列表任意元素的新slot ```slot_indexes```.
    1. ```fn get_slot_entries(slot_index: u64, entry_start_index: usize, max_entries: Option<u64>) -> Vec<Entry>```：返回从```entry_start_index```开始的一个 entry vector，如果```if max_entries == Some(max)```,限制为max，否则，返回没有上限的vector长度。

注意：累积起来，这意味着重放阶段现在必须知道一个插槽何时结束，并订阅它感兴趣的下一个插槽，以获得下一组entries。在此之前，链接 slot 的责任落在Blockstore身上。

### Bank 接口
bank 公开重放(The bank exposes to replay stage:)：
1. ```prev_hash```: 它在哪个PoH链上工作，这是由它处理的最后一个条目的哈希表示的。
2. ```tick_height```: PoH链中目前正在被Bank验证的tick。
3. ```votes```: 一堆记录，包含了：

    1. ```prev_hashes```: 这次投票之后的任何事情都必须链接在PoH

    2. ```tick_height```: 投票时候的tick 高度
    
    3. ```lockout period```: 在账本上要观察多长时间才能在这个投票下面被链接

Replay 阶段使用Blockstore api来找到最长的entries链，它可以挂起之前的投票。如果该 entries 链没有挂起最新的投票，Replay阶段将Bank 回滚到该投票，并从那里Replay该链。

### Pruning Blockstore(剪枝 blockstore)
当 blockstore 中的entries 足够老，表示所有可能的分叉变得不那么有用，甚至可能对重新启动时的重放造成问题。然而，一旦验证者的投票达到了最大锁定，任何不在PoH链上的Blockstore内容都可以被剪枝、删除。














