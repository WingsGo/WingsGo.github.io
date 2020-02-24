---
title: Doris源码解析[三、BE存储引擎]
key: 2020022403
tags: Doris
---

# BE存储引擎部分代码设计文档(2019)

<!--more-->

## 主要工作
1. 类名和⽂文件名尽量量⼀一致，看代码的时候⽐比较⽅方便便。
2. 逻辑上分层，理理论上只能从上往下调⽤用，不不能从下往上调⽤用，明确每个层次的作⽤用。
3. 划分概念，现在是乱的，看了了类名不不知道意思。
4. 写好注释，看懂了了的代码就加上注释，新来的同学看代码⽐比较容易易⼀一些，如果英⽂文不不⾏行行就使⽤用中⽂文注释
吧。
5. 重构完之后，理理论上各个模块应该可以单独测试。
6. 本次不不涉及功能裁剪，只是重新组合各个模块的功能让模块之间的职责划分更更清晰，现在接⼝口内部的⾏行行
为的实现代码能不不动就不不动了了。

## 主要概念
1. storageengine --> tablet --> rowset --> segment group --> segment, storageengine 包含多个tablet， tablet包含多个rowset。
2. 将BE原来的table这个概念换成tablet，每个tablet 有唯⼀一的tabletid和schema hash， rollup 和 schema change都是产⽣生不不同的tablet。
3. rowset相当于原来的⼀一个版本，rowset可以是⼀一个版本的数据，rowset也可以包含多个版本的数据。
4. version:只是代表⼀一个版本号，version不不能表示数据⽂文件。
5. Delta 特定指导⼊入的⼀一个批次，delta这个概念可能只是出现在我们的讨论中，不不会出现在代码中。

## 接口设计

### 物理存储抽象
整个BE是建⽴立在以下2个⼦子存储系统上来完成存储的管理理⼯工作的:

- 文件系统，这个主要是底层的rowset读写⽂文件需要这个，⽐比如创建⽂文件，读取⽂文件，删除⽂文件，拷⻉贝⽂文 件等。这块考虑到未来我们可能⽤用BOS作为我们的存储系统，所以这块我们做了了⼀一个抽象叫做 FsEnv。 
- 元数据存储系统，是底层存储tablet和rowset的元数据的⼀一个系统，⽬目前我们使⽤用rocksdb实现，未来⽐比 如⽤用BOS作为存储的时候这个可能也会变化，所以我们我们把这块也做了了抽象叫做 MetaEnv。 
- FsEnv和MetaEnv其实都跟底层的存储介质相关，并且它们经常需要⼀一起使⽤用，⽽而且需要在多个对象之 间传递，所以在FsEnv和MetaEnv之上我们构建了了⼀一个DataDir层，封装了了FsEnv和MetaEnv，每个 DataDir都绑定了了⼀一个存储介质StorageMedium。下⾯面分别介绍⼀一下它们的主要接⼝。

```
// StoreType 描述底层的物理理存储的类型 StoreType
{
    POSIX; // 基于本地⽂文件系统的⼀一种dir实现
    BOS; // 基于百度对象存储系统的dir实现 
}

// data dir 并不不仅仅封装了了FsEnv和MetaEnv， 还封装了部分磁盘⽬录管理的逻辑，⽐如当前我们⼀个磁盘目录下最多允许1000个子目录
// 当我们要create⼀个新的tablet的时候，data dir就需要做⼀下这个⽬录的选择了
// 未来汇报存储状态的时候我们也直接让DataDir来汇报就可以了了
DataDir {
    Status init();
    // 根据tablet的元数据，创建tablet存储的⽂文件结构，并不不修改元数据 Status create_tablet_dir(TabletMeta* tablet_meta);
    // 删除tablet的物理理⽂文件结构
    Status delete_tablet_dir(TabletMeta* tablet_meta); Status get_fs_env(FsEnv* fs_env);
    Status get_meta_env(MetaEnv* meta_env);
}
// 抽象⼀下文件系统操作的逻辑，未来我们可能使⽤用bos作为我们的存储介质，到时候只要重新实现⼀下fs env就可以了
FsEnv
{
    create_file(); create_dir(); pread(); pwrite();
}
// 抽象tablet和rowset的元数据操作的逻辑，未来假如对接bos的话需要实现⼀个基于bos的元数据管理模块 
// 比如我们基于bos做管理的时候，metastore可能实现为⼀个⽂件，到时候把本地的数据直接保存到bos上作为⽂件就可以了
MetaEnv
{
    Status get_all_tablet_meta(vector<TabletMeta*> tablet_metas); 
    Status get_all_rowset_meta(vector<RowsetMeta*> rowset_metas); 
    Status save_tablet_meta(TabletMeta* tablet_meta);
    Status delete_tablet_meta(TabletMeta* tablet_meta);
    Status save_rowset_meta(RowsetMeta* rowset_meta);
    Status delete_rowset_meta(RowsetMeta* rowset_meta);
}
```

### Rowset设计
Rowset代表了一个或多个批次的数据，抽象它的原因是想以后能支持更多的文件存储格式。在设计过程中考虑了一下几点:
- tablet是不不感知磁盘的，真正感知存储路路径的是rowset，每个rowset都应该有一个单独的磁盘⽬目录，比如我们在做磁盘迁移的时候，一个tablet的数据可能跨越了多个磁盘，这样实际上就要求每个rowset有⾃己的存储⽬目录。 
- rowset⼀旦生成了就应该是⼀个不可变的结构，理论上任何状态都不能改变，这样有利于控制我们系统的读写并发。 
- rowset的元数据尽量量少⼀些，因为⼀个节点可能有几十万个tablet，每个tablet⼜可能有⼏百到几千个rowset，而rowset的元数据位于内存中，如果数据量太⼤内存可能爆掉。

- rowset的设计要满⾜我们**磁盘拷⻉**的需求，⽐如我们把⼀个磁盘的数据和元数据都拷⻉一下到另外⼀个磁盘上或者把磁盘目录变更一下，系统应该是能够加载的。所以不能直接在rowset meta中存储data dir。

- rowset的id在⼀个be内要唯一， 但是在不同的be上会有重复。rowset id只是⼀个唯⼀的标识，并不代表任何含义，两个BE上如果有两个rowset的id相同，他们的数据内容不一定一样。


```
// 我们的⽂文件格式可能会升级，也可能引⼊入别的⽂文件格式，所以我们引⼊入rowset type来表示不不同的⽂文件格式 
enum RowsetType {
    ALPHA_ROWSET = 0, // doris原有的列存
    BETA_ROWSET = 1, // 新列存 
};

RowsetState {
    PREPARING, // 表示rowset正在写入
    COMMITTED, // 表示rowset已经写入完成，但是⽤户还不可见;这个状态下的rowset，BE不能⾃行判断是否删除，必须由FE的指令
    VISIBLE; // 表示rowset 已经对⽤户可见 
}

RowsetMeta {
    version; // 当rowset meta更新的时候修改一下这个version，当两个元数据冲突的时候以version大的为基准
    txn_id; // 当前rowset关联的txn是哪个，当系统重启的时候storage engine会加载所有的rowset，根据rowset关联的txn恢复内存中的txnmap
    tablet_id; 
    tablet_schemahash; 
    rowset_id; 
    rowset_type; 
    start_version; 
    end_version; 
    row_num; 
    total_disk_size; 
    data_disk_size; 
    index_disk_size;
    // 下⾯这一堆最好直接封装成binary
    // min/max/null等统计信息 
    column_meta; 
    delete_condition;
    // 用于存储不同rowset的一些额外信息
    // 为了方便扩展，Property是⼀一个key/value对，类型都是string vector<std::string> properties;
}
Rowset { 
方法:
    // 主要用来通过rowset_id获取元素信息，初始化_rowset_meta 
    // 加载rowset的时候要不要做⽂文件的check?
    Status init(RowsetMeta* rowset_meta);
    // 构造RowsetReader，用于读取rowset 
    RowsetReader* create_reader();
    // ⽤于rowset重命名使⽤，比如迁盘
    // 我们不提供move接口，move可以⽤用copy + delete 实现
    Status copy(RowsetWriter* dest_rowset_writer);
    // 删除rowset底层所有的文件和其他资源 
    Status delete();
属性:
    // rowset 记录它所在的磁盘目录的句柄，从这个句柄中可以获得metaenv和fsenv 
    DataDir dir;
    RowsetMeta rowset_meta;
};

// 当要创建⼀个rowset的时候必须调⽤用这个api，⽐如clone，restore，ingest 
// 除了系统加载之外，只有这个类能够⽣成⼀个新的rowset
RowsetWriter {
方法:
    // 初始化RowsetWriter
    Status init(std::string rowset_id, const std::string& rowset_path_prefix, Sche ma* schema) = 0;
    // 写⼊一个rowblock数据
    Status add_row_block(RowBlock* row_block) = 0;
    // 当要向⼀个rowset中增加⼀个文件的时候调⽤这个api，这个⽅方法看着有点别扭，有更更好的⽅方法可以替换一下
    // 1. 根据传⼊的file name，rowset writer会⽣成⼀个对应的文件名 
    // 2. 上游系统根据生成的文件名把文件拷⻉到这个目录下
    Status generate_written_path(string src_path);
    // ⽣成一个rowset，⾸先要检查是否close了，如果close了才能调⽤用这个API 
    Status build(Rowset* rowset);

    // 释放所有的资源，此时代表不能写⼊新的数据文件了，但是可以修改元数据
    close();
属性:
    DataDir dir;
};

/**
* ReadContext中的参数是在要读取rowset的逻辑的地方
* 拿到RowsetReader对象之后，进行init的时候传⼊的。
* 这些参数区别于构造函数中的传⼊入参数在于构造函数的参数是在
* new ReadContext的对象的时候就会确定，也就是在Rowset::new_reader() 
* 的时候决定
*/
class ReadContext {
private:
    Schema* projection; // 投影列信息
    std::unordered_map<std::string, ColumnPredicate> predicates; //过滤条件 const EncodedKey* lower_bound_key; // key下界
    const EncodedKey* exclusive_upper_bound_key; // key上界
};

class RowsetReader { 
方法:
    // reader初始化函数
    // init逻辑中需要判断rowset是否可以直接被统计信息过滤删除、是否被删除条件删除。 
    // 如果被删除，则has_next直接返回false
    Status init(ReadContext* read_context) = 0;
    // 判断是否还有数据 bool has_next() = 0;
    // 读取下⼀个Block的数据
    Status next_block(RowBlock* row_block) = 0;
    // 关闭reader
    // 会触发下层数据读取逻辑的close操作，进行类似关闭⽂文件， 
    // 更新统计信息等操作
    void close() = 0;
属性:
    DataDir dir;
    RowsetMeta rowset_meta; 
};
```

### Tablet的设计
- 在新的架构里tablet只是⼀个rowset的容器器，tablet可以根据rowset创建相应的reader。
- tablet里所有的rowset都是⽣效的rowset，不生效的rowset不放在这⾥面，这样简化了tablet的逻辑。

```
TabletState
{ 

}

TabletMeta 
{
    int64_t version; // 表示meta修改的版本号，当有多个属于同一个tablet的meta存在时，选择version最⼤的那个
    tableid; // ⽬前没想到有什么用
    partition_id; // publish version的时候需要根据partition id来获取所有关联的rowset， 因为version是绑定在partition上的，不能按照tablet来partition的原因是怕tabletid太多了
    shardid; // tablet 所在的shard的id，这个是跟磁盘管理理相关的逻辑，是个optional 
    TabletState state;
    vector<RowsetMeta> rowsets;
    serialize();
    deserialize();
}

Tablet { 
方法:
    // 当compaction、写⼊结束、clone结束的时候调用这个api来修改
    // 这⾥把要增加的rowset和要删除的rowset放在⼀起可以保持⼀个原子性
    // 当compaction的时候可能会增加⼀些rowset，删除⼀些rowset，并且希望这个原⼦性
    Status modify_rowsets(vector<Rowset*> added_rowsets, vector<Rowset*> deleted_r
    owsets);

    Status create_snapshot(); 
    
    Status release_snapshot();

    // 这个API 只有在垃圾清理理的时候⽤用
    Status get_all_rowsets(vector<Rowset*> rowsets) 
    {
        // 根据⽂文件列列表的前缀返回所有的独⽴立的rowsetid 
    }
属性:
    DataDir data_dir; // 当前tablet 默认的data dir，当写⼊入新的rowset的时候，需要⽤用这个dat
adir来存储数据 
}

TabletReader
{
    set_read_version(int64_t end_version);
    set_start_key();
    set_end_key();
    set_filter_expr();
}
```
TabletMeta和RowsetMeta的持久化存储的⽅式:
- 当前存在的问题
    - 过去tablet meta中保存了一个rowset meta的列表，每次更新rowset都需要更新整个tablet meta，这样写放大⾮常严重。
    - 在这种⽅方式下未生效的rowset【COMMITTED】和 已生效的rowset【VISIBLE】都存储在tablet meta中，⽐较混乱。

- 所以这次我们计划更新tablet meta和rowset meta的管理方式，具体逻辑如下:
    - 当rowset commit的时候，需要修改rowset的状态为COMMITTED，需要把rowsetmeta信息持久化到meta store中。
    - 当rowset publish的时候，需要修改rowset的状态为VISIBLE，修改rowset的version，把rowset信息持久化到meta env中。然后把rowset meta添加到tablet meta中，此时tablet meta可以不持久化。
    - 这会带来一个问题，就是系统重启的时候如果rowset特别多，那么启动会⾮常慢。所以需要engine定期的把tablet meta持久化，⼀旦持久化，那么就把已经visible的rowset meta从meta store中删除，因为他们已经保存到tablet meta中了。 这样做还带来⼀个好处，当rowset向tablet中增加时，只是内存操作，加⼀个lock就可以，不涉及io，速度可以做到⾮常快。

### 系统启动加载过程
在上述结构下，整个系统重启的时候的tablet和rowset加载过程如下:

- 根据系统的配置⽂件读取data dir，目前doris的架构⾥，每个data dir就是配置⽂件的一个磁盘⽬录。 

- 读取每个data dir下的meta store中的tablet meta和rowset meta，根据tablet meta中的shard id构建 rowset对象的shard id信息。

- 根据加载后的所有的tablet meta构建tablet对象
    - 如果在不同的data dir⽬录下存在多个tablet meta，那么选择version最大的tablet meta构建tablet对象。
    - 如果rowset的状态是PUBLISHED，那么就把他追加到对应的tablet里。 
    - 如果rowset的状态是COMMITTED，那么就根据txn id，把他加到txn map里。


## Service 和 StorageEngine设计
我们把BE向外部暴露的能⼒封装为⼀个的Service，Service有两类:⼀类是轮询fe获得task，然后执行行 task;另一类是作为rpc的服务端供rpc调用的。Service假如需要访问rowset或者tablet的时候，我们都⾸先注册⼀个task，由storageengine负责⽣成task的各种状态，然后暴露对应的task给service，然后service调⽤ task的execute方法来执行。这样就把所有的并发和状态控制都封装在StorageEngine中，简单的逻辑如下:
- service模块调用StorageEngine对应的⽅法create一个task，在create task中storageengine做⼀系列的检查，加锁的逻辑

- service拿到task之后调⽤task的execute方法，来执⾏具体的⼯作。service调用storageengine的finish⽅法结束⼀个task。

- 如果task执⾏失败，那么调用cancel task方法来取消task，storageengine需要清除对应的状态。在上述逻辑下，所有的状态维护，锁的维护都由storageengine完成，外部不再看到锁，看到状态。以⽬前为例我们有以下⼏种task:
```
IngestService
{
    // batch ingest的逻辑 storageengine->create_rowset_writer();while(1) {
        download_file();
        rowset_writer.write_block(); 
    }
    storageengine->finish_rowset_writer(rowset_writer);
    
    // 流式导⼊的逻辑 
    storageengine->create_rowset_writer();
    // 每次写⼊都调
    rowset_writer.write_block();
    // 写完之后 
    storageengine->finish_rowset_writer(rowset_writer);
}

AlterService
{
    storageengine->create_alter_Task();
    alter_task.execute() {
    } 
    storageengine->finish_alter_Task(alter_Task);
}

// 从⽂文件恢复⼀一个rowset
CloneService
{
    // 当全量clone的时候，tablet不存在，那么先调⽤create tablet创建⼀一个tablet 
    // 增量clone的时候，tablet存在，不需要create tablet
    if (tablet not exists)
    {
        storageengine->create_tablet();
    }
    storageengine->create_clone_task(); clone_Task.execute();
    storageengine->finish_clone_task(clone_task);
}

AlterTask 
{
    Status execute() { 
        // 1. 读取⽼老老的数据
        // 2. 把数据写⼊入到新的tablet中
    }

    Status get_rowsets(vector<RowSet*> rowsets); 
}
    
CloneTask
{
    // 开始执⾏一个clone Task
    // 1. 调⽤远程的节点，make snapshot
    // 2. 从远程节点下载⽂文件，放到本地临时⽂文件夹，包括meta，index，data 
    Status execute()
    {
        make_snapshot(); 
    }

    // 当clone任务结束后，上层调用这个API，这个API返回clone的rowset的列表 
    // 上层API把对应的rowset根据条件添加到tablet中
    Status get_rowsets(vector<RowSet*> rowsets);
}
    
MigrateTask
{
    Status execute()
    {
        // 修改tablet的meta，把tablet meta持久化到⽬目标的 
        // 1. 遍历migrate task中的rowset 列列表
        // 2. ⼀一个个rowset进⾏行行转化。
    }

    Status get_rowsets(vector<RowSet*> rowsets); 
}

CompactionTask
{
    Status execute();
    Status get_rowsets(vector<RowSet*> rowsets); 
}

```

### StorageEngine部分
```
StorageEngine
{
    // 当上游的导⼊入任务需要导⼊一份数据的时候调⽤这个api，这个api会创建⼀个rowset writer给上层
    // 上层调⽤用这个api完成数据的写⼊入
    Status open_rowset_writer(int64_t tablet_id, int64_t schema_hash, int64_t txnId, RowsetWriter* rowset_writer) 
    {
        // 1. 检测当前tablet是否在做schema change，如果在做，那么就把schema change的tablet也拿出来
        // 2. ⽣成rowset id 和 rowset writer
        // 3. 根据tablet的属性，设置rowset writer写⼊文件的⽬录
        // 4. 在txn map中注册⼀下txn与rowset，tablet的关系
        // 5. 在一些场景下【clone，schema change，rollup】的时候没有txn id，此时就不需要建立关联了。
    }

    // 当向⼀个rowset writer 写完数据之后调⽤用这个API
    Status finish_rowset_writer(RowsetWriter* rowset_writer) 
    {
        // 1. 生成对应的rowset, 如果有alter task在执⾏那么就生成两个tablet的rowset
        // 2. 将rowset的状态变更为committed状态。
        // 3. 当⽤户调用finish的时候，需要把rowset元数据持久化，因为调⽤完finish之后，⽤户会调用txn.commit
        // ⼀旦commit成功，那么数据就不能丢失了。 
    }

    // 当写⼊错误的时候调用这个API，表示取消一个rowset writer 
    Status cancel_rowset_writer(RowsetWriter* rowset_writer) 
    {
        // 1. 关闭rowset 打开的文件句柄 
        // 2. 把rowset 从txn map中移除
        // 3. 这⾥可以不删除rowset对应的⽂件，因为垃圾清理线程会清理它们 
    }

    // 当service 层收到publish version task的时候调⽤用这个api 
    Status publish_txn(int64_t txn_id)
    {
        // 1. 找到txn对应的rowset，把rowset的状态变更更为 VISIBLE
        // 2. 持久化rowset meta到meta store中
        // 3. 把rowset添加到tablet对应的rowset 列列表中
    }

    // 清理掉txn关联的rowset
    Status clear_txn(int64_t txn_id) {
        // 1. 如果txn 关联的rowset meta已经持久化到meta store中，那么需要从meta store中清理。
    }

    // 创建⼀个alter task, 在be端不区分schema change和rollup，统一用alter task来区分。 
    Status create_alter_task(AlterTableReq* req, AlterTask* alter_task)
    {
        // 1. 选择⼀个data dir
        // 2. 在data dir下创建⼀个tablet
        // 3. 把tablet信息填充到 alter_task ⾥
        // 4. 把tablet信息添加到storage engine的map⾥
        // 5. 在tablet的meta中记录一个状态，这个状态要持久化，否则会有问题。
    }

    // 当alter task结束后，调用这个api，把新的rowset添加到tablet的rowset列列表中
    Status finish_alter_task(AlterTask* alter_task)
    {
        // 1. 读取创建的新的rowset，把他们加⼊入到新的tablet中 
        // 2. 清理一下alter task的状态
        // 3. 持久化2个tablet的meta状态
    }

    Status cancel_alter_task(AlterTask* alter_task) {
        // 1. 删除创建的新的tablet
        // 2. 清理正在操作的tablet的alter status 
    }

    // 清除一个tablet上标记的alter状态
    Status clear_alter_status(int64_t tablet_id);

    Status create_clone_task(CloneTask* clone_task) 
    {
    }

    // clone task本质上是从远端复制一堆rowset到本地，当cloneTask结束的时候，调用这个api
    // 这个api会拿到clone Task⽣成的rowset，把他们加入到tablet的rowset列列表中。
    Status finish_clone_task(CloneTask* clone_task)
    {
        // 1. 取出clone task中生成的rowset 把rowset 添加到tablet的meta中 
        // 2. 持久化tabletmeta
        // 3. 把rowset 添加到tablet的active rowset列表中
    }

    Status make_snapshot(int64_t tablet_id, TabletSnapshot* snapshot) 
    {
        // 创建⼀一个snapshot 
    }

    // 把⼀个tablet从一个⽬录移动到另外一个⽬录。
    Status create_migrate_task(int64_t tablet_id, Path dest_path) 
    {
        // 1. 修改tablet的default dir，这样新来的数据就都可以往新的dir⾥写⼊了 
        // 2. 持久化tablet meta
        // 3. 返回所有visible的rowset和正在写⼊的rowset。
        // 4. 把要转化的rowset的信息填充到migrate task中。
    }

    Status finish_migrate_task(MigrateTask* migrate_task);

    Status create_compaction_task(CompactionTask* compaction_task) 
    {
        // 根据compaction policy选择⼀个最合适的tablet
        // 选择一堆rowset，创建一个rowset writer 填充到task中 
    }

    Status finish_compaction_task() 
    {
        // 读取compaction task ⽣成的rowset 
        // 变更tablet meta，持久化tabletmeta 
        // 变更tablet 中的rowset列表
    }

    Status cancel_compaction_task();

    Status drop_tablet() 
    {
        tablet_meta->set_state(TabletState.DELETING); 
        store = tablet_info->get_store();
        store->save_meta(tablet_meta);
        store->delete_tablet(tablet_meta);
        tablet_meta->set_state(TabletState.DELETED);
        store->save_meta(tablet_meta); 
    }

    Status create_tablet(CreateTabletReq* req) 
    {
        OlapStore* store = _select_store();
        TabletMeta* tablet_meta = _create_new_meta(req);
        store->create_tablet(tablet_meta);store->save_meta(tablet_meta);
    }

    // 系统启动的时候只是根据data dir的配置，读取meta store中所有的tablet和rowset的元数据信息
    // 并没有读取具体的物理⽂件的信息，因为读取物理文件信息在rowset规模特别大的时候会非常慢
    Status load()
    {
        vector<DataDir> data_dirs = 根据配置⽂文件读取所有的dir 
        for (DataDir dir : data_dirs) {
            dir.load_tablets(vector<TabletMeta*> tablets); 
            for (TabletMeta* talbet_meta : tablets) {
                // 构建tablet 对象
                for (RowsetMeta* rowset_meta : tablet_meta -> get_rowsets) {
                    // 构建txn map 
                }
            }
        }
    }

    // 这是⼀个垃圾回收的操作，BE运⾏过程中会产⽣许多垃圾，⽐如当rowset在写⼊的时候，写⼊了⼀部分,此时文件系统core掉了会产⽣
    // rowset 的垃圾文件;当tablet被删除的时候可能会产生tablet的垃圾⽂文件 
    //
    Status garbge_collection()
    {
        for (TabletInfo* tablet_info : tablet_infos) 
        {
            // 清理理所有的垃圾的tablet
            if (tablet_info->tablet_meta->tablet_state == TabletState.DELETING) 
            {
                tablet_info->store()->delete_tablet(tablet_meta); 
                tablet_meta->set_state(TabletState.DELETED); 
                store->save_meta(tablet_meta);
                continue;
            }
            // 清理tablet中⽆用的rowset文件, get all tablets 的实现目前考虑的就是根据文件名的前缀作为rowsetid
            rowset_ids = tablet_info->get_tablet()->get_all_rowsets(); 
            for (int64_t rowset_id : rowset_ids)
            {
                if (rowset_id not in committed rowsets 
                    && rowset id not in prepare rowsets 
                    && rowset id not in writing rowsets
                ) 
                {
                    tablet_info->get_tablet()->delete_rowset(rowset_id); 
                }
            }
        }
        // 垃圾的txn不要在这⾥清理， storage engine会**定时的把txn向fe汇报，fe会下发清理txn的任务，收到clear txn任务的时候清理**
    }
}
```

## 错误处理逻辑
抽象⼀个错误处理的队列，发⽣错误后，如果在当前线程中不能直接处理，那么需要把错误消息封装一下直接发送给错误队列，由各个不同的任务线程，从队列中读取错误信息来处理。