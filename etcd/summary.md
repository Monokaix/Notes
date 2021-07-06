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

# 总结

- etcd一致性读用的是ReadIndex的算法，只有状态机的apply index大于等于commit index时才返回读数据，否则就等待，而commit index的正确性是通过每次都向leader请求保证的
- etcd通过ttl设置key时，会把当前节点时间加上ttl时间作为key的expiration，这个过期时间会被写到一个有序map。
  leader中会运行一个tick，每500ms触发一次，同时tick会产生一个sync消息，里面包含leader系统当前时间，leader把sync消息通过raft广播到所有节点，节点查询有序map，然后删除过期时间小于当前时间的key
- etcdctl的watch可分为一次和持续watch，可以加上参数指定从哪个版本号开始，当是持续watch时，etcd返回第一个大于等于watchindex的key的信息
- etcd支持cas操作，可以在put请求中设定key的条件(是否存在，值是否为指定值等)进行key的设置，只有在满足请求中的条件时才会进行set key操作
- 集群内全部节点都会记录修改存储状态的操作，如创建、删除、更新等操作，watch和get只会被本地节点记录
- etcd v3使用lease机制取代了v2的ttl机制
- 对目录的存储使用的是线段树结构，相当于把同一目录下的key看做是具有相同前缀的key处理