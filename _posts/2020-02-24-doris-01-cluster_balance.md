---
title: Doris源码解析[一、负载均衡]
key: 2020022401
tags: Doris
---

 # Doris 副本修复和均衡策略

<!--more-->

 # 名词解释
 1. Tablet：Doris 表的逻辑分片，一个表有多个分片
 2. Replica：分片的副本，默认一个分片有3个副本
 3. Healthy Replica：健康副本，副本所在 Backend 存活，且副本的版本完整
 4. TabletChecker（TC）：是一个常驻的后台线程，用于定期扫描所有的 Tablet，检查这些 Tablet 的状态，并根据检查结果，决定是否将 tablet 发送给 TabletScheduler。
 5. TabletScheduler（TS）：是一个常驻的后台线程，用于处理由 TabletChecker 发来的需要修复的 Tablet。同时也会进行集群副本均衡的工作。
 6. TabletSchedCtx（TSC）：是一个 tablet 的封装。当 TC 选择一个 tablet 后，会将其封装为一个 TSC，发送给 TS。
 
 # TabletChecker
 ## 检查规则
 TC 根据以下规则进行检查：
 
 1. 只检查 OLAP 类型的表
 2. 只检查状态为 NORMAL 的表（SCHEMA_CHANGE, ROLLUP 等状态都不检查）
 3. 对每一个 tablet 进行健康检查 [tablet.getHealthStatusWithPriority()]，检查结果如下：
    
    1. REPLICA_MISSING
       存活副本数小于期望副本数
    2. VERSION_INCOMPLETE
       存活副本数大于等于期望副本数，但其中健康副本数小于期望副本数
    3. REPLICA_MISSING_IN_CLUSTER
       健康副本数大于等于期望副本数，但在对应 cluster 内的副本数小于期望副本数
    4. REDUNDANT
       健康副本都在对应 cluster 内，但数量大于期望副本数
    5. HEALTHY
       健康分片，即条件[1-4]都不满足。
 
 ## 优先级
 TC 在检查 tablet 状态的同时，也会对非 HEALTHY 状态的 tablet 分配一个初始优先级。该优先级决定了在 TS 中的处理优先级。
 
 1. VERY_HIGH
    
    * REDUNDANT。因为这种情况处理最快且不占资源，所以我们优先搞定他。
 2. HIGH
    
    * REPLICA_MISSING 且多数不存活
    * VERSION_INCOMPLETE 且多数版本缺失
 3. NORMAL
    
    * REPLICA_MISSING 但多数存活
    * VERSION_INCOMPLETE 但多数版本完整
 4. LOW
    
    * REPLICA_MISSING_IN_CLUSTER
 
 > **手动优先修复**
 
 > 我们提供了一个优先修复某个表或分区的命令：
 
 > `ADMIN REPAIR TABLE tbl [PARTITION (p1, p2, ...)];`
 
 > 这个命令，可以告诉 TC，在扫描 Tablet 时，对需要优先修复的表或分区中的有问题的 Tablet，给予 VERY_HIGH 的优先级。
 
 > 这个命令只是一个 hint，并不能保证一定能修复成功，并且优先级也会随 TS 的调度而发生变化。并且当 Master FE 切换或重启后，这些信息都会丢失。
 
 > 可以通过 `show proc "/cluster_balance/priority_repair";` 查看设为优先修复的表或分区。
 
 ## 检查调度
 1. 默认每 20 秒进行一次全量检查。(TabletChecker.CHECK_INTERVAL_MS)
 2. 在每次检查前，如果发现 TS 中正在调度或排队的 Tablet 数量超过 5000（TabletChecker.MAX_SCHEDULING_TABLETS），则放弃本轮检查。
 3. 每轮检查会将已经修复完成的分区，从**优先修复队列**中移除。
 
 # TabletSchedCtx
 TabletSchedCtx 包含了一个 Tablet 在 TS 处理过程中，所有的资源占用和中间结果。当 Tablet 从 TS 中移除时，会调用 releaseResource() 释放所有占用的资源（比如 Slot）。
 
 同时，TSC 好包括一些副本选择和优先级调整的逻辑，我们会在 TabletScheduler 中介绍。
 
 # TabletScheduler
 TabletScheduler 也是一个后台常驻线程，默认每五秒（SCHEDULE_INTERVAL_MS）运行一次。TS 有两个主要工作，一是处理由 TC 发来的 tablet，尝试修复他们。二是进行集群均衡。
 
 ## 副本修复
 由 TC 发来的 Tablet，封装为 TabletSchedCtx，加入到一个优先级队列中（pendingTablets）。TS 每次会从队列中取出最多 10 个（MIN_BATCH_NUM），对每一个 tablet，进行如下操作：
 
 1. 重新检查 tablet 的副本状态，根据副本状态，进行对应的修复操作
 2. 对副本的处理（scheduleTablet()）如果没有抛出异常，则会将这个 tablet 标记为 RUNNING，放入 runningTablets 队列。对于该对了中的 tablet，只要当任务结束，或任务超时后，才会移出。
 3. 对副本的处理也可能抛出 SchedException 异常，有以下几种类型：
    
    1. SCHEDULE_FAILED：处理失败。则会适当调整优先级，然后重新加入 pending 队列。
    2. FINISHED：已经处理完成，并且没有任何需要等待执行的任务，则直接移除 tablet
    3. UNRECOVERABLE：遇到不能自动处理的情况（比如对应的 table 不存在了），直接移除 tablet。
 
 ### PathSlot
 为了保证在执行副本修复或均衡过程中，不会导致某些机器因分配任务太多而被打满，我们为 BE 上的每块盘指定了固定个数的 Slot。对于每块盘（对应一个 BE 上的 root path），用于副本修复的 slot 数量为 1 (Config.schedule_slot_num_per_path)，而用于均衡的 slot 数量为 2（BALANCE_SLOT_NUM_FOR_PATH）。
 
 副本恢复和均衡，都是对一个副本，从源端拷贝到目的端的操作。这个操作，会对源端和目的端各占用一个 slot。如果在调度时拿不到 slot，则会抛出 SCHEDULE_FAILED 异常。
 
 ### 各种恢复操作
 1. REPLICA_MISSING：`handleReplicaMissing()`
    尽量选择一个负载低的 path 作为目的端，同时选择一个可用副本作为源端，启动一个 CloneTask。
 2. VERSION_INCOMPLETE：`handleReplicaMissing()`
    选择一个版本缺失的副本作为目的端。优先选择：`last success version > last failed version` 或 `更小的 last failed version` 或 `last failed version 大于 0`，同时选择一个可用副本作为源端，启动一个 CloneTask。
 3. REDUNDANT：`handleRedundantReplica()`
    选择一个多余的副本，删除对应的元数据，按以下顺序，直到选择一个：
    
    1. BE 已经被 drop
    2. BE 不可用
    3. 副本的状态是 CLONE
    4. 副本的 last failed version > 0 （版本缺失）
    5. 低版本副本
    6. 不在对应 cluster 中的副本
    7. 高负载 BE 上的副本
 4. REPLICA_MISSING_IN_CLUSTER：`handleReplicaClusterMigration()`
    基本同 REPLICA_MISSING，只是在选择目的端时，需要选择对应的 cluster 的BE。
 
 ### 优先级调整
 我们使用一个优先级队列 (pendingTablets) 来存放待调度的 Tablet。当调度失败时，Tablet 会被重新放回优先级队列。如果不调整优先级，则可能导致重复调度一个一直会失败的 tablet，而导致其他 tablet 得不到调度。
 
 TS 每隔 1min，会检查一遍优先级队列中 tablet 的优先级，并按需调整（TabletSchedCtx.adjustPriority()）。
 
 1. 对于同一个 tablet，优先级至少间隔 5min（MIN_ADJUST_PRIORITY_INTERVAL_MS）才会调整一次。
 2. 如果一个 tablet 调度失败 5 次（SCHED_FAILED_COUNTER_THRESHOLD），则降一级。
 3. 如果一个 tablet 自上次调度超过 30min 未被调度（MAX_NOT_BEING_SCHEDULED_INTERVAL_MS），则升一级。
 4. 初始优先级（origPriority）为 VERY_HIGH 的，至多被降级为 NORMAL，初始优先级为 LOW 的，至多提升为 HIGH。
 
 ### 超时和任务失败
 对于在 pendingTablets 中等待被调度的 tablet，没有超时设置，如果调度不成功，则会一直尝试调度，或者因为 tablet 已经恢复而终止调度。
 
 对于 runningTablets 中等待 Clone 任务完成的 tablet，设有一个任务超时时间。当任务超时后，会移除这个 tablet。任务超时时间根据副本的大小预估（TabletSchedCtx.getApproximateTimeoutMs()），在 3min(MIN_CLONE_TASK_TIMEOUT_MS) 到 120min(MAX_CLONE_TASK_TIMEOUT_MS) 之间，按 5MB/s(MIN_CLONE_SPEED_MB_PER_SECOND) 的速度，预估超时时间。
 
 如果cloneTask失败次数超过3次（TabletSchedCtx.RUNNING_FAILED_COUNTER_THRESHOLD）,则该任务会被移除。防止一个一直失败的任务占用资源。
 
 ## 集群均衡
 TS 在每轮调度的同时，也会尝试进行集群均衡的操作。均衡操作使用独立的 slot，防止副本恢复操作因集群不均衡而无法进行，而均衡操作又因没有 slot 而无法触发。
 
 ### ClusterLoadStatistics
 ClusterLoadStatistics（CLS）用于表示一个 cluster 中各个 Backend 的负载均衡情况（BackendLoadStatistic）。TS 根据这个统计值，来触发集群均衡。我们当前通过 **磁盘使用率** 和 **副本数量** 两个指标，为每个BE计算一个 loadScore，作为 BE 的负载分数。分数越高，表示该 BE 的负载越重。
 
 磁盘使用率和副本数量各有一个权重系数，分别为 **capacityCoefficient** 和 **replicaNumCoefficient**，其 **和衡为1**。其中 capacityCoefficient 会根据实际磁盘使用率动态调整。当一个 BE 的总体磁盘使用率在 50% 以下，则 capacityCoefficient 值为 0.5，如果磁盘使用率在 75%（Config.capacity_used_percent_high_water）以上，则值为 1。如果使用率介于 50% ~ 75% 之间，则改权重系数平滑增加，公式为：
 
 `capacityCoefficient= 2 * 磁盘使用率 - 0.5`
 
 该权重系数保证当磁盘使用率过高时，该 Backend 的负载分数会更高，以保证尽快均衡这个 BE。
 
 可以通过 `show proc "/cluster_balance/cluster_load_stat";` 查看整个集群的负载分数。
 
 TS 会每隔 1 分钟（STAT_UPDATE_INTERVAL_MS）更新一次 CLS。
 
 ### 均衡策略
 我们在 LoadBalancer 类中处理均衡任务。TS 每轮调度，会通过 LoadBalancer 选择一些合适的 Tablet，封装为 BALANCE 类型的 TabletSchedCtx，放入 pendingTablets。
 
 1. 将 BE 根据 cluster 的平均 loadScore，换分为 high,mid,low 三挡。如果高于平均 loadScore 10% (Config.balance_load_score_threshold)，则标记为 high，如果低于平均 10%，则标记为 low，否则，标记位 mid。如果所有 BE 都为 mid，则我们认为集群是均衡的。
 2. 同样，我们会将每个 BE 上的 path，根据使用容量，也划分为 high,mid,low 三挡。
 3. 我们根据以下规则选择 tablet（注意这里只是选择 tablet，而不确定具体的源端或目的端副本，这些操作交由 TS 在调度时完成。）
    
    1. 有副本在 high 档的BE上，并且
    2. 有副本在 high 或 mid 档的 path 上。
    
    > 我们不选择在 high 档BE上，但在 low 档 path 上的tablet，主要考虑到，这会让 low 档的 path 负载更低。
 4. 对于选择出来的 tablet，我们随机打乱后，封装为 BALANCE 类型的 TabletSchedCtx，优先级为 LOW，放入 pendingTablets。
 5. 同时我们会检查，处于 low 档的BE 是否都不可用。如果都不可用，则我们也不会触发均衡，因为那会导致均衡任务无法成功调度。
 
 ### 均衡调度
 当 TS 调度到 BALANCE 类型的 tablet 时，会进行以下操作：
 
 1. 检查这个 tablet 是否是 healthy，如果不是，终止 balance。
 2. 选择一个可用的 replica 作为源端。
 3. 选择一个 low 档的 BE 上 low 会 mid 的 path 作为目的端。（当只有 high 和 mid 时，mid 会被当做 low。当只有 mid 和 low 时，mid 会被当做 high）。
 4. 选择好后，会调用 `isMoreBalanced()` 检查这次均衡，是否会使集群更均衡（所有 BE 的 loadScore 与平均 loadScore 的差值的和变小）。
 5. 生成一个 CloneTask 开始执行。
 
 > 整个 Balance 过程只是在 low 档的 BE 上添加了一个副本。之后，TC会检查到副本冗余（REDUNDANT），然后交由 TS，删除高负载 BE 上的副本。
 
 # 附录
 ## TabletSchedulerStat
 一个简单的统计类，用于统计 scheduler 过程中的各种情况计数。可以通过 `show proc "/cluster_balance/sched_stat";` 查看，同时，Master FE 日志一会定期打印改统计值的周期变化情况。
 
 ## 副本调度与 Alter 流程
 Alter 流程会设置 table 的状态，从而阻止 TC 或 TS 处理非 NORMAL 状态 table 的 tablet。这个规则用于简化两个两种交叉导致的各种问题。在实际使用中，建议用户通过 **手动优先修复** 先恢复集群，再进行 Alter 作业。
 
 ## 持久化
 所有和调度相关的信息都不会持久化，包括处于 CLONE 状态的副本。如果 MASTER FE 宕机或者切主，则当前的所有调度任务丢失。


