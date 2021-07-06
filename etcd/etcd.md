# version

revision：全局递增的，每次删除等修改操作都会递增
每个key有create_revision、mod_revision、version字段
create_revision是创建该key时的全局revision值
mod_revision是最后一次修改时的revision值
version：每次key修改会递增，从1开始

# 一致性

一致性分强弱，从强到若如下

- 线性一致性Linearizability consistency ，也叫强一致性、严格一致性、原子一致性。是程序能实现的最高的一致性模型
- 顺序一致性 Sequential consistency
- 因果一致性 Causal consistency 弱一致性
- 最终一致性 Eventual consistency 弱一致性，最终会达到一致性，不考虑中间的结果

1)严格一致性（Strict Consistency)：需要有全局统一时钟来调配，而在分布式系统中要做到不同地方时钟完全同步不可能实现，因此是一种理想状态

2)顺序一致性：不关注结果正确与否，不同线程看到的结果一致就行不管对错，要对一起对，要错一起错，即不同线程看到的顺序不应冲突
