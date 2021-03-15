# Spark

stage是根据shuffle来划分的, shuffle算子之前的代码被划分为一个stage, 之后的被划分为一个stage

shuffle算子包括: groupByKey, join

shuffle算子宽窄依赖

窄依赖与宽依赖的区别: 是否发生shuffle(洗牌)操作, 宽依赖会发生shuffle操作
- 窄依赖是子RDD的各个分片(partition), 不依赖于其他分片, 能够独立计算得到结果
- 宽依赖指子RDD的各个分片会依赖于父RDD的多个分片, 所以会造成父RDD的各个分片在集群中的重新分片
  - 宽依赖主要有两个过程: shuffle write和shuffle fetch, 类似于Hadoop的Map和Reduce阶段.
  - shuffle write将Shuffle Map Task任务产生的中间结果缓存到内存中
  - shuffle fetch获取Shuffle Map Task缓存的中间结果进行Shuffle Reduce Task计算

## 资源分配
越大越好
1. executor 进程数
- 增大executor能够提高并行度, 比如3个executor, 2个cores, 能够并行的task是6个
2. cores 每个机器配的CPU核数
- 也是增大并行的task数
3. memory 每个机器的内存大小
- 对RDD进行cache, 更多的内存可以缓存更多的数据, 可以减少磁盘IO
- 对于shuffle操作, reduce端, 需要内存来存放拉取的数据进行聚合. 如果内存不够, 也会写入磁盘, 增大IO
- 对于task的执行, 会创建很多对象, 如果内存不足, 会频繁导致JVM堆内存溢出, 然后频繁GC, minor GC和full GC 的速度很慢
4. driver-memory driver的内存(影响不大)

## 调节并行度
1. task数量, 至少设置成总的cpu cores数相等(最理想情况下, 总共150个cpu cores, 分配了150个task一起运行, 差不多同一时间运行完成)
2. 官方推荐, task的数量设置成spark application总cpu cores的数量的2-3倍, 比如150个cpu cores, 需要将task数量设置成300-500个
3. 实际情况下, 有些task运行的快一些, 比如50s完成了, 有些task慢一些, 要1分50s完成, 如果设置的task数量, 刚好等于cpu cores的数量, 那么就会造成资源的浪费
   - 因为比如150个task, 其中的10个很快就运行完了, 还有140个还在运行, 这个时候, 就有10个cpu cores空闲出来, 就导致了浪费.
   - 如果将task数量设置成cpu cores总数2-3倍, 其中的一些task运行完了, 其他的task就会补进来, 没有cpu空闲
4. 设置并行度
```
spark.default.parallelism=500
```

## 重构RDD架构以及RDD持久化
1. 复用RDD, 差不多的RDD, 抽取为一个共同的RDD, 供后面的RDD计算, 重复使用
2. 公共RDD持久化, 将RDD缓存到内存中
3. 双副本机制(在内存够用的情况下), 一个副本丢失了, 不用重复计算, 直接使用另一个副本

## 广播大变量
1. 默认情况下, 