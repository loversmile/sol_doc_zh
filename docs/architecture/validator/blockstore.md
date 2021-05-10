## Blockstore
当一个区块到达终点后，从这个区块到起源区块的所有区块形成一个线性链，其名字为区块链。然而，在此之前，validator必须维护所有可能有效的链，称为fork。在 [fork generation](../cluster/fork_generation.html)中描述了由于领导者轮换而产生fork的过程。这里描述的块存储数据结构是验证器如何处理这些fork，直到块最终完成。

blockstore允许validator以任何顺序记录它在网络上观察到的每个碎片，只要碎片是由给定槽的预期领导者签署的。

分片被移动到一个可分叉的键空间，即领导槽+分片索引(槽内)的元组。这允许Solana协议的跳过列表结构被完整地存储，而不需要先验地选择要遵循哪个fork、持久化哪个Entries或者什么时候持久化它们。

对最近的碎片的修复请求从RAM或最近的文件中提供，从更深层的储存中取出近期的碎片，这是由支持Blockstore的存储实现的。

### blockstore 的功能
1. Persistence(持久化)：
2. Repair(修复)：
3. Forks
4. Restart

### blockstore 设计



