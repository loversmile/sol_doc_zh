## 同步 (Synchronization)
快速、可靠的同步是Solana能够实现如此高吞吐量的最大原因。传统的区块链在被称为区块的大块上同步交易。对块进行同步时，事务在持续时间(称为“块时间”)过去之前不能被处理。在Proof of Work共识中，这些块时间需要非常大(~10分钟)，以最小化多个验证器同时产生一个新的有效块的可能性。在Proof of Stake 共识中没有这样的约束，但是如果没有可靠的时间戳，验证器就不能确定传入块的顺序。流行的解决方案是用[wallclock timestamp](https://en.bitcoin.it/wiki/Block_timestamp)时间戳标记每个块。由于时钟漂移和网络延迟的变化，时间戳仅在一两个小时内准确。为了解决这个问题，这些系统延长了阻塞时间，以合理地确定每个块上的中间时间戳总是在增加。

Solana采取了一种非常不同的方法，叫做Proof of History 或者 PoH。前导节点的“时间戳”块具有密码证明，表明自上次证明以来已经经过了一段时间。所有被Hash到证明中的数据都是在证明生成之前发生的。然后，节点与Validator节点共享新块，Validator节点能够验证这些证明。这些块可以以任何顺序到达验证器，甚至可以在数年后重新播放。有了这样可靠的同步保证，Solana能够将块分解成更小的事务批次，称为entries。在任何块共识之前，entryies被实时流到Validator。

从技术上讲，Solana从不发送一个区块，而是使用这个术语来描述Validator通过投票来实现确认的entries序列。这样，Solana的确认时间与基于区块的系统完全可以相提并论。当前实现将块时间设置为800ms。

在幕后发生的是，entries被流到验证器的速度就像Leader节点可以将一组有效事务批处理到一个entry一样快。Validator在对这些条目的有效性进行投票之前就会处理它们。乐观的来看，处理事务时候，从收到最后一个条目到节点可以投票的时间之间实际上没有延迟。在没有达成事件共识的情况下，节点只是回滚其状态。这种乐观处理技术于1981年引入，称为乐观并发控制（[Optimistic Concurrency Control](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.65.4735)）。它可以应用于区块链架构，在这种架构中，集群对一个哈希进行投票，哈希代表完整的分类帐，最高可达某个块高度。在Solana中，它是使用最后一个entry的PoH哈希简单地实现的。

### 与VDFs的关系(Relationship to VDFs)
2017年11月，Solana首次描述了在区块链中使用的历史证明技术(PoH)。第二年6月，斯坦福大学描述了一种类似的技术，称为可验证延迟函数(VDF)。[verifiable delay function](https://eprint.iacr.org/2018/601.pdf)

VDF的一个理想特性是验证时间非常快。Solana验证其延迟函数的方法与创建它所花费的时间成正比。通过4000个核心GPU，它的速度足以满足Solana的需求，但如果你问上面引用的论文的作者，他们可能会告诉你(并且已经告诉你)Solana的方法在算法上很慢，它不应该被称为VDF。我们认为VDF应该代表可验证延迟函数的范畴，而不仅仅是具有某些性能特征的子集。在这一问题得到解决之前，Solana很可能会继续使用PoH这个术语来表示其专用VDF。

PoH和VDF之间的另一个区别是，VDF仅用于跟踪持续时间。另一方面，PoH的哈希链包括应用程序观察到的任何数据的哈希。这些数据是一把双刃剑。一方面，数据“证明了历史”——数据肯定在哈希之前就存在了。另一方面，这意味着应用程序可以通过更改数据散列时对散列链进行操作。PoH链因此不能作为一个很好的随机来源，而一个没有数据的VDF可以。例如，Solana的leader rotation算法仅来自于VDF的高度，而不是该高度上的哈希值。

### 与共识机制的关系(Relationship to Consensus Mechanisms)
历史证明不是一种共识机制，但它被用来提高索拉纳的股权证明共识的性能。它还用于提高data plane 协议的性能。

### More on Proof of History
[water clock analogy](https://medium.com/solana-labs/proof-of-history-explained-by-a-water-clock-e682183417b8)

[Proof of History overview](https://medium.com/solana-labs/proof-of-history-a-clock-for-blockchain-cf47a61a9274)