## 性能指标(Performance Metrics)
Solana集群的性能是用网络每秒能够维持的平均事务数(TPS)来衡量的。以及由绝大多数集群确认一个事务所需的时间(Confirmation Time)。

每个集群节点维护各种计数器，这些计数器在某些事件上递增。这些计数器定期上传到基于云的数据库。Solana的指标仪表板获取这些计数器，计算性能指标并将其显示在dashboard上。

### TPS
每个节点的bank运行时维护它已处理的事务的计数。Dashboard首先计算集群中所有启用metric的节点的事务中位数。然后在2秒的时间段内平均集群事务计数，并在TPS时间序列图中显示。Dashboard还显示了平均TPS、最大TPS和总事务计数统计数据，这些数据都是从中值事务计数计算出来的。

### 确认时间(Confirmation Time)
每个验证器节点维护一个对该节点可见的活动分类账本fork列表。当节点接收并处理了所有与该fork对应的条目时，就认为该fork已被冻结。当一个fork获得了累积的绝对多数投票，并且它的一个子fork被冻结时，它就被认为是被确认的。

节点为每个新fork分配一个时间戳，并计算确认fork所需的时间。这个时间反映为性能指标中的验证器确认时间。性能Dashboard以时间序列图的形式显示每个验证器节点的确认时间的平均值。

### **硬件设定(Hardware setup)**
验证软件部署在GCP n1-standard-16实例中，配有1TB pd-ssd磁盘和2x Nvidia V100图形处理器。这些部署在美国-西方-1地区。

当网络从带有n1-standard-16 CPU-only实例的客户端机器聚合后，启动solanna -bench-tps，该实例带有以下参数```--tx\_count=50000 --thread-batch-sleep 1000```

TPS和确认指标是在```bench-tps```传输阶段开始，平均5分钟，从仪表板上的数字中捕获的，
