---
title: Doris源码解析[二、异步任务之Schema Change]
key: 2020022402
tags: Doris
---

# Doris提交Schema Change策略

<!--more-->

在Doris中，许多任务都是异步执行的，例如创建tablet，删除tablet，修改表结构，Broker Load等等，这一类的作业的主要流程为:

1. 在FE根据task_type可以分为多种任务，`AgentTask`为所有任务的父类，`AgentTask`及其子类封装了包括`backendId`, `signature`, `taskType`等任务的信息，在RPC调用时FE通过调用`toAgentTaskRequest`方法，将该类转化为`TAgentTaskRequest`然后将调用RPC方法接口`submit_task`将请求下发给BE执行。
2. `AgentServer`在初始化时，会调用类中根据`task_type`划分的多个`std::unique_ptr<TaskWorkerPool>`的实例的`start`方法，该方法会根据`task_type`为每个实例绑定不同的回调函数，并根据参数，对每种task生成指定数量的线程去执行对应的任务。BE接收任务请求参数`TAgentTaskRequest`，`AgentServer::submit_task`方法会解析`TAgentTaskRequest`，并根据`task_type`转发给不同类型的`TaskWorkPool`实例，该实例会调用`TaskWorkPool::submit_task`方法，将`TAgentTaskRequest`提交到`std::deque<TAgentTaskRequest>`队列中等待调度，而`TaskWorkPool`类中有多种回调函数，当一个任务提交完成后，同时会调用`_worker_thread_condition_lock.notify()`方法唤醒一个消费者线程去执行任务。
3. 当线程对应回调方法的函数中的条件变量唤醒时（即`worker_pool_this->_worker_thread_condition_lock.wait();`），该线程会开始执行任务，并在执行完成后在方法`TaskWorkerPool::_finish_task`中进行RPC调用，向FE汇报任务完成的结果`TFinishTaskRequest`，最后调用`TaskWorkerPool::_remove_task_info`方法将该任务从队列中移除，完成整个Schema Change的任务逻辑


## FE端逻辑
以下从源码层面进一步剖析整个作业的执行逻辑，首先从FE入手，在用户修改表结构时，任务有4个状态，即`pending`，`waitingTxn`, `running`及`finished`，在进行类型转换时，Doris首先会在`pending`阶段，下发给BE创建`replicas`的任务，具体的代码如下:

```
protected void runPendingJob() throws AlterCancelException {
    Preconditions.checkState(jobState == JobState.PENDING, jobState);
    LOG.info("begin to send create replica tasks. job: {}", jobId);
    Database db = Catalog.getCurrentCatalog().getDb(dbId);
    if (db == null) {
        throw new AlterCancelException("Databasee " + dbId + " does not exist");
    }

    // 1. create replicas
    AgentBatchTask batchTask = new AgentBatchTask();
    // count total replica num
    int totalReplicaNum = 0;
    for (MaterializedIndex shadowIdx : partitionIndexMap.values()) {
        for (Tablet tablet : shadowIdx.getTablets()) {
            totalReplicaNum += tablet.getReplicas().size();
        }
    }
    MarkedCountDownLatch<Long, Long> countDownLatch = new MarkedCountDownLatch<>(totalReplicaNum);
    db.readLock();
    try {
        OlapTable tbl = (OlapTable) db.getTable(tableId);
        if (tbl == null) {
            throw new AlterCancelException("Table " + tableId + " does not exist");
        }

        boolean isStable = tbl.isStable(Catalog.getCurrentSystemInfo(),
                Catalog.getCurrentCatalog().getTabletScheduler(),
                db.getClusterName());
        if (!isStable) {
            errMsg = "table is unstable";
            LOG.warn("doing schema change job: " + jobId + " while table is not stable.");
            return;
        }

        Preconditions.checkState(tbl.getState() == OlapTableState.SCHEMA_CHANGE);
        for (long partitionId : partitionIndexMap.rowKeySet()) {
            Partition partition = tbl.getPartition(partitionId);
            if (partition == null) {
                continue;
            }
            TStorageMedium storageMedium = tbl.getPartitionInfo().getDataProperty(partitionId).getStorageMedium();
            
            Map<Long, MaterializedIndex> shadowIndexMap = partitionIndexMap.row(partitionId);
            for (Map.Entry<Long, MaterializedIndex> entry : shadowIndexMap.entrySet()) {
                long shadowIdxId = entry.getKey();
                MaterializedIndex shadowIdx = entry.getValue();
                
                short shadowShortKeyColumnCount = indexShortKeyMap.get(shadowIdxId);
                List<Column> shadowSchema = indexSchemaMap.get(shadowIdxId);
                int shadowSchemaHash = indexSchemaVersionAndHashMap.get(shadowIdxId).second;
                int originSchemaHash = tbl.getSchemaHashByIndexId(indexIdMap.get(shadowIdxId));
                
                for (Tablet shadowTablet : shadowIdx.getTablets()) {
                    long shadowTabletId = shadowTablet.getId();
                    List<Replica> shadowReplicas = shadowTablet.getReplicas();
                    for (Replica shadowReplica : shadowReplicas) {
                        long backendId = shadowReplica.getBackendId();
                        countDownLatch.addMark(backendId, shadowTabletId);
                        CreateReplicaTask createReplicaTask = new CreateReplicaTask(
                                backendId, dbId, tableId, partitionId, shadowIdxId, shadowTabletId,
                                shadowShortKeyColumnCount, shadowSchemaHash,
                                Partition.PARTITION_INIT_VERSION, Partition.PARTITION_INIT_VERSION_HASH,
                                tbl.getKeysType(), TStorageType.COLUMN, storageMedium,
                                shadowSchema, bfColumns, bfFpp, countDownLatch);
                        createReplicaTask.setBaseTablet(partitionIndexTabletMap.get(partitionId, shadowIdxId).get(shadowTabletId), originSchemaHash);
                        
                        batchTask.addTask(createReplicaTask);
                    } // end for rollupReplicas
                } // end for rollupTablets
            }
        }
    } finally {
        db.readUnlock();
    }

    if (!FeConstants.runningUnitTest) {
        // send all tasks and wait them finished
        AgentTaskQueue.addBatchTask(batchTask);
        AgentTaskExecutor.submit(batchTask);
        // max timeout is 1 min
        long timeout = Math.min(Config.tablet_create_timeout_second * 1000L * totalReplicaNum, 60000);
        boolean ok = false;
        try {
            ok = countDownLatch.await(timeout, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            LOG.warn("InterruptedException: ", e);
            ok = false;
        }

        if (!ok) {
            // create replicas failed. just cancel the job
            // clear tasks and show the failed replicas to user
            AgentTaskQueue.removeBatchTask(batchTask, TTaskType.CREATE);
            String errMsg = null;
            if (!countDownLatch.getStatus().ok()) {
                errMsg = countDownLatch.getStatus().getErrorMsg();
            } else {
                List<Entry<Long, Long>> unfinishedMarks = countDownLatch.getLeftMarks();
                // only show at most 3 results
                List<Entry<Long, Long>> subList = unfinishedMarks.subList(0, Math.min(unfinishedMarks.size(), 3));
                errMsg = "Error replicas:" + Joiner.on(", ").join(subList);
            }
            LOG.warn("failed to create replicas for job: {}, {}", jobId, errMsg);
            throw new AlterCancelException("Create replicas failed. Error: " + errMsg);
        }
    }

    // create all replicas success.
    // add all shadow indexes to catalog
    db.writeLock();
    try {
        OlapTable tbl = (OlapTable) db.getTable(tableId);
        if (tbl == null) {
            throw new AlterCancelException("Table " + tableId + " does not exist");
        }
        Preconditions.checkState(tbl.getState() == OlapTableState.SCHEMA_CHANGE);
        addShadowIndexToCatalog(tbl);
    } finally {
        db.writeUnlock();
    }

    this.watershedTxnId = Catalog.getCurrentGlobalTransactionMgr().getTransactionIDGenerator().getNextTransactionId();
    this.jobState = JobState.WAITING_TXN;

    // write edit log
    Catalog.getCurrentCatalog().getEditLog().logAlterJob(this);
    LOG.info("transfer schema change job {} state to {}, watershed txn id: {}", jobId, this.jobState, watershedTxnId);
}
```

这段代码的主要逻辑就是首先在元数据中检查db、tbl、副本数、表的状态等信息，由于一个be需要生成多个tablet，因此使用`MarkedCountDownLatch`记录任务数量，并生成对应的`CreateReplicaTask`任务，然后将多个`CreateReplicaTask`添加到实现了`Runnable`接口的`AgentBatchTask`类的私有成员变量`private Map<Long, List<AgentTask>> backendIdToTasks`中，在`AgentBatchTask.run`方法中，会将`List<AgentTask>`转化为`List<TAgentTaskRequest>`，然后通过thrift接口`AgentService.TAgentResult submit_tasks(1:list<AgentService.TAgentTaskRequest> tasks);`向BE发送RPC调用，BE完成后，会调用thrift接口`MasterService.TMasterResult finishTask(1:MasterService.TFinishTaskRequest request);
`向FE汇报BE的任务执行结果。

随后，该方法会调用`AgentTaskQueue.addBatchTask(batchTask);`及`AgentTaskExecutor.submit(batchTask);`方法，记录并提交`batchTask`，`AgentTaskExecutor`内部是通过`Executors.newCachedThreadPool`来创造线程并执行实现了`Runnable`接口的`AgentBatchTask`。

在生成了多个执行任务线程后，主线程会调用`ok = countDownLatch.await(timeout, TimeUnit.MILLISECONDS);`方法，阻塞并等待所有执行提交任务的子线程完成。

## BE端逻辑
在BE端，我们首先查看`AgentServer`类，该类的主要私有成员及构造函数如下，在构造函数中，首先有多个`std::unique_ptr<TaskWorkerPool>`类的实例化对象，每个实例化对象代表一类任务，在`AgentServer`类初始化时，会根据配置参数，对每个实例化对象调用`TaskWorkerPool::start`函数，生成指定数量的线程并绑定对应的回调函数。

```
std::unique_ptr<TaskWorkerPool> _create_tablet_workers;
std::unique_ptr<TaskWorkerPool> _drop_tablet_workers;
std::unique_ptr<TaskWorkerPool> _push_workers;
std::unique_ptr<TaskWorkerPool> _publish_version_workers;
std::unique_ptr<TaskWorkerPool> _clear_transaction_task_workers;
std::unique_ptr<TaskWorkerPool> _delete_workers;
std::unique_ptr<TaskWorkerPool> _alter_tablet_workers;
std::unique_ptr<TaskWorkerPool> _clone_workers;
std::unique_ptr<TaskWorkerPool> _storage_medium_migrate_workers;
std::unique_ptr<TaskWorkerPool> _check_consistency_workers;

// These 3 worker-pool do not accept tasks from FE.
// It is self triggered periodically and reports to Fe master
std::unique_ptr<TaskWorkerPool> _report_task_workers;
std::unique_ptr<TaskWorkerPool> _report_disk_state_workers;
std::unique_ptr<TaskWorkerPool> _report_tablet_workers;

std::unique_ptr<TaskWorkerPool> _upload_workers;
std::unique_ptr<TaskWorkerPool> _download_workers;
std::unique_ptr<TaskWorkerPool> _make_snapshot_workers;
std::unique_ptr<TaskWorkerPool> _release_snapshot_workers;
std::unique_ptr<TaskWorkerPool> _move_dir_workers;
std::unique_ptr<TaskWorkerPool> _recover_tablet_workers;
std::unique_ptr<TaskWorkerPool> _update_tablet_meta_info_workers;
```

```
AgentServer::AgentServer(ExecEnv* exec_env, const TMasterInfo& master_info) :
        _exec_env(exec_env),
        _master_info(master_info),
        _topic_subscriber(new TopicSubscriber()) {
    for (auto& path : exec_env->store_paths()) {
        try {
            string dpp_download_path_str = path.path + DPP_PREFIX;
            boost::filesystem::path dpp_download_path(dpp_download_path_str);
            if (boost::filesystem::exists(dpp_download_path)) {
                boost::filesystem::remove_all(dpp_download_path);
            }
        } catch (...) {
            LOG(WARNING) << "boost exception when remove dpp download path. path=" << path.path;
        }
    }

    // It is the same code to create workers of each type, so we use a macro
    // to make code to be more readable.

#ifndef BE_TEST
#define CREATE_AND_START_POOL(type, pool_name)         \
    pool_name.reset(new TaskWorkerPool(                \
                TaskWorkerPool::TaskWorkerType::type,  \
                _exec_env,                             \
                master_info));                         \
    pool_name->start();
#else
#define CREATE_AND_START_POOL(type, pool_name)
#endif // BE_TEST

    CREATE_AND_START_POOL(CREATE_TABLE, _create_tablet_workers);
    CREATE_AND_START_POOL(DROP_TABLE, _drop_tablet_workers);
    // Both PUSH and REALTIME_PUSH type use _push_workers
    CREATE_AND_START_POOL(PUSH, _push_workers);
    CREATE_AND_START_POOL(PUBLISH_VERSION, _publish_version_workers);
    CREATE_AND_START_POOL(CLEAR_TRANSACTION_TASK, _clear_transaction_task_workers);
    CREATE_AND_START_POOL(DELETE, _delete_workers);
    CREATE_AND_START_POOL(ALTER_TABLE, _alter_tablet_workers);
    CREATE_AND_START_POOL(CLONE, _clone_workers);
    CREATE_AND_START_POOL(STORAGE_MEDIUM_MIGRATE, _storage_medium_migrate_workers);
    CREATE_AND_START_POOL(CHECK_CONSISTENCY, _check_consistency_workers);
    CREATE_AND_START_POOL(REPORT_TASK, _report_task_workers);
    CREATE_AND_START_POOL(REPORT_DISK_STATE, _report_disk_state_workers);
    CREATE_AND_START_POOL(REPORT_OLAP_TABLE, _report_tablet_workers);
    CREATE_AND_START_POOL(UPLOAD, _upload_workers);
    CREATE_AND_START_POOL(DOWNLOAD, _download_workers);
    CREATE_AND_START_POOL(MAKE_SNAPSHOT, _make_snapshot_workers);
    CREATE_AND_START_POOL(RELEASE_SNAPSHOT, _release_snapshot_workers);
    CREATE_AND_START_POOL(MOVE, _move_dir_workers);
    CREATE_AND_START_POOL(RECOVER_TABLET, _recover_tablet_workers);
    CREATE_AND_START_POOL(UPDATE_TABLET_META_INFO, _update_tablet_meta_info_workers);
#undef CREATE_AND_START_POOL

#ifndef BE_TEST
    // Add subscriber here and register listeners
    TopicListener* user_resource_listener = new UserResourceListener(exec_env, master_info);
    LOG(INFO) << "Register user resource listener";
    _topic_subscriber->register_listener(doris::TTopicType::type::RESOURCE, user_resource_listener);
#endif
}
```

接下来在`AgentServer`中最重要的是`AgentServer::submit_task`方法，该方法是thrift调用接口的具体实现，接收`List<TAgentTaskRequest>`参数，并根据每个`task`的`task_type`将请求转发给`TaskWorkPool::submit_task`方法

```
void AgentServer::submit_tasks(TAgentResult& agent_result, const vector<TAgentTaskRequest>& tasks) {
    Status ret_st;

    // TODO check master_info here if it is the same with that of heartbeat rpc
    if (_master_info.network_address.hostname == "" || _master_info.network_address.port == 0) {
        Status ret_st = Status::Cancelled("Have not get FE Master heartbeat yet");
        ret_st.to_thrift(&agent_result.status);
        return;
    }

    for (auto task : tasks) {
        VLOG_RPC << "submit one task: " << apache::thrift::ThriftDebugString(task).c_str();
        TTaskType::type task_type = task.task_type;
        int64_t signature = task.signature;

#define HANDLE_TYPE(t_task_type, work_pool, req_member)                         \
    case t_task_type:                                                           \
        if (task.__isset.req_member) {                                          \
            work_pool->submit_task(task);                                       \
        } else {                                                                \
            ret_st = Status::InvalidArgument(strings::Substitute(               \
                    "task(signature=$0) has wrong request member", signature)); \
        }                                                                       \
        break;

        // TODO(lingbin): It still too long, divided these task types into several categories
        switch (task_type) {
        HANDLE_TYPE(TTaskType::CREATE, _create_tablet_workers, create_tablet_req);
        HANDLE_TYPE(TTaskType::DROP, _drop_tablet_workers, drop_tablet_req);
        HANDLE_TYPE(TTaskType::PUBLISH_VERSION, _publish_version_workers, publish_version_req);
        HANDLE_TYPE(TTaskType::CLEAR_TRANSACTION_TASK,
                    _clear_transaction_task_workers,
                    clear_transaction_task_req);
        HANDLE_TYPE(TTaskType::CLONE, _clone_workers, clone_req);
        HANDLE_TYPE(TTaskType::STORAGE_MEDIUM_MIGRATE,
                    _storage_medium_migrate_workers,
                    storage_medium_migrate_req);
        HANDLE_TYPE(TTaskType::CHECK_CONSISTENCY,
                    _check_consistency_workers,
                    check_consistency_req);
        HANDLE_TYPE(TTaskType::UPLOAD, _upload_workers, upload_req);
        HANDLE_TYPE(TTaskType::DOWNLOAD, _download_workers, download_req);
        HANDLE_TYPE(TTaskType::MAKE_SNAPSHOT, _make_snapshot_workers, snapshot_req);
        HANDLE_TYPE(TTaskType::RELEASE_SNAPSHOT, _release_snapshot_workers, release_snapshot_req);
        HANDLE_TYPE(TTaskType::MOVE, _move_dir_workers, move_dir_req);
        HANDLE_TYPE(TTaskType::RECOVER_TABLET, _recover_tablet_workers, recover_tablet_req);
        HANDLE_TYPE(TTaskType::UPDATE_TABLET_META_INFO,
                    _update_tablet_meta_info_workers,
                    update_tablet_meta_info_req);

        case TTaskType::REALTIME_PUSH:
        case TTaskType::PUSH:
            if (!task.__isset.push_req) {
                ret_st = Status::InvalidArgument(strings::Substitute(
                        "task(signature=$0) has wrong request member", signature));
                break;
            }
            if (task.push_req.push_type == TPushType::LOAD
                    || task.push_req.push_type == TPushType::LOAD_DELETE) {
                _push_workers->submit_task(task);
            } else if (task.push_req.push_type == TPushType::DELETE) {
                _delete_workers->submit_task(task);
            } else {
                ret_st = Status::InvalidArgument(strings::Substitute(
                        "task(signature=$0, type=$1, push_type=$2) has wrong push_type",
                        signature, task_type, task.push_req.push_type));
            }
            break;
        case TTaskType::ALTER:
            if (task.__isset.alter_tablet_req || task.__isset.alter_tablet_req_v2) {
                _alter_tablet_workers->submit_task(task);
            } else {
                ret_st = Status::InvalidArgument(strings::Substitute(
                        "task(signature=$0) has wrong request member", signature));
            }
            break;
        default:
            ret_st = Status::InvalidArgument(strings::Substitute(
                    "task(signature=$0, type=$1) has wrong task type", signature, task_type));
            break;
        }
#undef HANDLE_TYPE

        if (!ret_st.ok()) {
            LOG(WARNING) << "fail to submit task. reason: " << ret_st.get_error_msg()
                    << ", task: " << task;
            // For now, all tasks in the batch share one status, so if any task
            // was failed to submit, we can only return error to FE(even when some
            // tasks have already been successfully submitted).
            // However, Fe does not check the return status of submit_tasks() currently,
            // and it is not sure that FE will retry when something is wrong, so here we
            // only print an warning log and go on(i.e. do not break current loop),
            // to ensure every task can be submitted once. It is OK for now, because the
            // ret_st can be error only when it encounters an wrong task_type and
            // req-member in TAgentTaskRequest, which is basically impossible.
            // TODO(lingbin): check the logic in FE again later.
        }
    }

    ret_st.to_thrift(&agent_result.status);
}
```

然后我们进入`TaskWorkPool`函数中，可以看到主要有一下的函数指针，每个函数指针会绑定一个回调函数，该回调函数在`TaskWorkPool::start`方法中被绑定

```
static void* _create_tablet_worker_thread_callback(void* arg_this);
static void* _drop_tablet_worker_thread_callback(void* arg_this);
static void* _push_worker_thread_callback(void* arg_this);
static void* _publish_version_worker_thread_callback(void* arg_this);
static void* _clear_transaction_task_worker_thread_callback(void* arg_this);
static void* _alter_tablet_worker_thread_callback(void* arg_this);
static void* _clone_worker_thread_callback(void* arg_this);
static void* _storage_medium_migrate_worker_thread_callback(void* arg_this);
static void* _check_consistency_worker_thread_callback(void* arg_this);
static void* _report_task_worker_thread_callback(void* arg_this);
static void* _report_disk_state_worker_thread_callback(void* arg_this);
static void* _report_tablet_worker_thread_callback(void* arg_this);
static void* _upload_worker_thread_callback(void* arg_this);
static void* _download_worker_thread_callback(void* arg_this);
static void* _make_snapshot_thread_callback(void* arg_this);
static void* _release_snapshot_thread_callback(void* arg_this);
static void* _move_dir_thread_callback(void* arg_this);
static void* _recover_tablet_thread_callback(void* arg_this);
static void* _update_tablet_meta_worker_thread_callback(void* arg_this);
```

最后是被`AgentServer::submit_task`转发调用的`TaskWorkPool::submit_task`方法，在该方法中，每个`task`会被依次塞入`std::deque<TAgentTaskRequest> _tasks;`队列中（先进先出），然后调用`_worker_thread_condition_lock.notify();`方法唤醒一个`task`线程执行任务，即一个典型的`生产-消费者模型`

```
void TaskWorkerPool::submit_task(const TAgentTaskRequest& task) {
    const TTaskType::type task_type = task.task_type;
    int64_t signature = task.signature;

    std::string type_str;
    EnumToString(TTaskType, task_type, type_str);
    LOG(INFO) << "submitting task. type=" << type_str << ", signature=" << signature;

    if (_register_task_info(task_type, signature)) {
        // Set the receiving time of task so that we can determine whether it is timed out later
        (const_cast<TAgentTaskRequest&>(task)).__set_recv_time(time(nullptr));
        size_t task_count_in_queue = 0;
        {
            lock_guard<Mutex> worker_thread_lock(_worker_thread_lock);
            _tasks.push_back(task);
            task_count_in_queue = _tasks.size();
            _worker_thread_condition_lock.notify();
        }
        LOG(INFO) << "success to submit task. type=" << type_str << ", signature=" << signature
                << ", task_count_in_queue=" << task_count_in_queue;
    } else {
        LOG(INFO) << "fail to register task. type=" << type_str << ", signature=" << signature;
    }
}
```

以`_alter_tablet_worker_thread_callback`任务为例，该任务是一个消费者，当生产者队列为空时会调用`worker_pool_this->_worker_thread_condition_lock.wait();`进行阻塞，直到被生产唤醒后，执行具体的任务逻辑，然后调用`TaskWorkPool::_finish_task`向FE汇报任务完成情况，最后通过`TaskWorkPool::_remote_task_info`将任务从队列`std::deque<TAgentTaskRequest> _tasks;`中移除，至此便完成了整个创建replica的逻辑。

```
void* TaskWorkerPool::_alter_tablet_worker_thread_callback(void* arg_this) {
    TaskWorkerPool* worker_pool_this = (TaskWorkerPool*)arg_this;

#ifndef BE_TEST
    while (true) {
#endif
        TAgentTaskRequest agent_task_req;
        {
            lock_guard<Mutex> worker_thread_lock(worker_pool_this->_worker_thread_lock);
            while (worker_pool_this->_tasks.empty()) {
                worker_pool_this->_worker_thread_condition_lock.wait();
            }

            agent_task_req = worker_pool_this->_tasks.front();
            worker_pool_this->_tasks.pop_front();
        }
        int64_t signatrue = agent_task_req.signature;
        LOG(INFO) << "get alter table task, signature: " <<  agent_task_req.signature;
        bool is_task_timeout = false;
        if (agent_task_req.__isset.recv_time) {
            int64_t time_elapsed = time(nullptr) - agent_task_req.recv_time;
            if (time_elapsed > config::report_task_interval_seconds * 20) {
                LOG(INFO) << "task elapsed " << time_elapsed
                          << " seconds since it is inserted to queue, it is timeout";
                is_task_timeout = true;
            }
        }
        if (!is_task_timeout) {
            TFinishTaskRequest finish_task_request;
            TTaskType::type task_type = agent_task_req.task_type;
            switch (task_type) {
            case TTaskType::ALTER:
                worker_pool_this->_alter_tablet(worker_pool_this,
                                            agent_task_req,
                                            signatrue,
                                            task_type,
                                            &finish_task_request);
                break;
            default:
                // pass
                break;
            }
            worker_pool_this->_finish_task(finish_task_request);
        }
        worker_pool_this->_remove_task_info(agent_task_req.task_type, agent_task_req.signature);
#ifndef BE_TEST
    }
#endif
    return (void*)0;
}
```