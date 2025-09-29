# Milvus 协调器详解：系统的大脑指挥中心

## 前言

在 Milvus 的分布式架构中，协调器（Coordinators）扮演着系统的"大脑"角色，它们负责协调整个系统的运行，管理资源分配，调度任务执行，确保数据的一致性和可靠性。本文将深入剖析 Milvus 三大核心协调器的设计原理、工作机制和最佳实践。

## 协调器架构概览

### 协调器的定位和作用

Milvus 的协调器层是系统的控制中枢，主要负责：

1. **资源管理**：分配和监控系统资源
2. **任务调度**：协调数据的读写操作
3. **元数据管理**：维护系统的元数据信息
4. **负载均衡**：优化系统的资源利用率
5. **故障恢复**：处理节点故障和数据一致性

### 协调器分类

Milvus 包含三种主要的协调器：

1. **RootCoord**：根协调器，全局元数据管理
2. **QueryCoord**：查询协调器，查询任务调度
3. **DataCoord**：数据协调器，数据生命周期管理

```
┌─────────────────┐
│   Client SDK    │
└─────────────────┘
         │
    ┌────┴────┐
    │  Proxy  │
    └────┬────┘
         │
    ┌────┴────┐
    │  RootCoord  │
    └────┬────┘
         │
    ┌────┴────┐
    │ QueryCoord │
    └────┬────┘
         │
    ┌────┴────┐
    │ DataCoord  │
    └────┬────┘
         │
┌────────┴────────┐
│  Compute Nodes  │
└─────────────────┘
```

## RootCoord：系统的总指挥

### RootCoord 的核心职责

RootCoord 是 Milvus 的最高级别的协调器，负责全局性的管理任务：

1. **元数据管理**：管理集合、分区、字段等元数据
2. **时间戳分配**：为所有操作分配全局时间戳
3. **权限控制**：实现 RBAC 权限管理
4. **配额管理**：控制资源使用配额
5. **服务发现**：维护集群节点信息

### RootCoord 的架构设计

```go
// RootCoord 核心结构
type RootCoord struct {
    // 基础设施
    ctx      context.Context
    config   *Config
    metrics  *Metrics

    // 核心组件
    metaStore     *meta.Store          // 元数据存储
    tso           *tso.TimestampOracle // 时间戳分配器
    idAllocator   *id.Allocator        // ID 分配器
    sessionMgr    *session.Manager     // 会话管理器

    // 管理器
    collectionMgr *collection.Manager   // 集合管理器
    partitionMgr  *partition.Manager    // 分区管理器
    credentialMgr *credential.Manager   // 凭据管理器
    quotaMgr      *quota.Manager       // 配额管理器
    rbacMgr       *rbac.Manager        // RBAC 管理器

    // 服务器
    server *grpc.Server
}

// RootCoord 初始化
func (r *RootCoord) Init() error {
    // 1. 初始化元数据存储
    if err := r.initMetaStore(); err != nil {
        return err
    }

    // 2. 初始化时间戳分配器
    r.tso = tso.NewTimestampOracle(r.config.TSO)

    // 3. 初始化 ID 分配器
    r.idAllocator = id.NewAllocator(r.metaStore)

    // 4. 初始化各个管理器
    r.collectionMgr = collection.NewManager(r.metaStore, r.idAllocator)
    r.partitionMgr = partition.NewManager(r.metaStore)
    r.credentialMgr = credential.NewManager(r.metaStore)
    r.quotaMgr = quota.NewManager(r.metaStore)
    r.rbacMgr = rbac.NewManager(r.metaStore)

    // 5. 初始化会话管理器
    r.sessionMgr = session.NewManager(r.config.Session)

    // 6. 启动后台任务
    r.startBackgroundTasks()

    return nil
}
```

### 时间戳分配机制

```go
// 时间戳分配器
type TimestampOracle struct {
    // 时间戳生成器
    generator *tso.Generator

    // 配置
    epoch int64
    step  int64

    // 保护机制
    mu sync.RWMutex
}

// 分配时间戳
func (t *TimestampOracle) Allocate(count int64) (int64, int64) {
    t.mu.Lock()
    defer t.mu.Unlock()

    // 1. 获取当前时间戳
    current := t.generator.Current()

    // 2. 分配指定数量的时间戳
    start := current
    end := current + count

    // 3. 更新生成器状态
    t.generator.Update(end)

    return start, end
}

// 时间戳结构
type Timestamp struct {
    // 时间戳主体
    Time uint64

    // 逻辑时间戳
    Logical uint16

    // 节点ID
    NodeID uint16
}

// 时间戳序列化
func (ts *Timestamp) Serialize() uint64 {
    // 节点ID (16位) + 逻辑时间戳 (16位) + 时间戳主体 (32位)
    return (uint64(ts.NodeID) << 48) |
           (uint64(ts.Logical) << 32) |
           ts.Time
}
```

### 集合管理流程

```go
// 集合管理器
type CollectionManager struct {
    metaStore    *meta.Store
    idAllocator  *id.Allocator
    cache        *cache.LRUCache
    eventChannel chan *CollectionEvent
}

// 创建集合
func (cm *CollectionManager) CreateCollection(req *CreateCollectionRequest) (*Collection, error) {
    // 1. 参数验证
    if err := cm.validateRequest(req); err != nil {
        return nil, err
    }

    // 2. 权限检查
    if err := cm.checkPermission(req.User, "create_collection"); err != nil {
        return nil, err
    }

    // 3. 配额检查
    if err := cm.checkQuota(req.User, "collections"); err != nil {
        return nil, err
    }

    // 4. 分配集合ID
    collectionID, err := cm.idAllocator.Allocate()
    if err != nil {
        return nil, err
    }

    // 5. 创建集合元数据
    collection := &Collection{
        ID:           collectionID,
        Name:         req.Name,
        Description:  req.Description,
        Schema:       req.Schema,
        State:        CollectionState_Creating,
        CreatedAt:    time.Now().Unix(),
        CreatedBy:    req.User,
        Properties:   req.Properties,
    }

    // 6. 持久化元数据
    if err := cm.metaStore.CreateCollection(collection); err != nil {
        return nil, err
    }

    // 7. 更新缓存
    cm.cache.Put(collectionID, collection)

    // 8. 发送创建事件
    cm.eventChannel <- &CollectionEvent{
        Type:       EventType_Create,
        Collection: collection,
    }

    return collection, nil
}

// 删除集合
func (cm *CollectionManager) DropCollection(req *DropCollectionRequest) error {
    // 1. 权限检查
    if err := cm.checkPermission(req.User, "drop_collection"); err != nil {
        return err
    }

    // 2. 获取集合元数据
    collection, err := cm.metaStore.GetCollectionByName(req.Name)
    if err != nil {
        return err
    }

    // 3. 检查集合状态
    if collection.State != CollectionState_Normal {
        return fmt.Errorf("collection is not in normal state")
    }

    // 4. 标记集合为删除中
    collection.State = CollectionState_Dropping
    if err := cm.metaStore.UpdateCollection(collection); err != nil {
        return err
    }

    // 5. 通知其他协调器
    if err := cm.notifyOtherCoordinators(collection); err != nil {
        return err
    }

    // 6. 清理相关资源
    if err := cm.cleanupCollectionResources(collection); err != nil {
        return err
    }

    // 7. 从存储中删除
    if err := cm.metaStore.DeleteCollection(collection.ID); err != nil {
        return err
    }

    // 8. 从缓存中删除
    cm.cache.Remove(collection.ID)

    return nil
}
```

### RBAC 权限管理

```go
// RBAC 管理器
type RBACManager struct {
    metaStore   *meta.Store
    roleManager *role.Manager
    policyCache *cache.LRUCache
}

// 检查权限
func (rm *RBACManager) CheckPermission(user string, resource string, action string) error {
    // 1. 获取用户角色
    roles, err := rm.metaStore.GetUserRoles(user)
    if err != nil {
        return err
    }

    // 2. 检查每个角色的权限
    for _, role := range roles {
        // 3. 获取角色策略
        policies, err := rm.metaStore.GetRolePolicies(role)
        if err != nil {
            return err
        }

        // 4. 检查是否有匹配的策略
        for _, policy := range policies {
            if rm.matchPolicy(policy, resource, action) {
                return nil
            }
        }
    }

    return fmt.Errorf("permission denied: user %s cannot %s on %s", user, action, resource)
}

// 策略匹配
func (rm *RBACManager) matchPolicy(policy *Policy, resource string, action string) bool {
    // 1. 检查资源匹配
    if !rm.matchResource(policy.Resource, resource) {
        return false
    }

    // 2. 检查动作匹配
    if !rm.matchAction(policy.Action, action) {
        return false
    }

    // 3. 检查条件（如果有的话）
    if policy.Condition != nil {
        return rm.checkCondition(policy.Condition)
    }

    return true
}
```

## QueryCoord：查询任务的调度大师

### QueryCoord 的核心职责

QueryCoord 负责查询相关的任务调度和资源管理：

1. **查询节点管理**：监控查询节点的状态
2. **查询任务调度**：分配查询任务到合适的节点
3. **段管理**：管理数据段的加载和卸载
4. **负载均衡**：优化查询节点的负载分布
5. **缓存管理**：优化查询性能

### QueryCoord 的架构设计

```go
// QueryCoord 核心结构
type QueryCoord struct {
    // 基础设施
    ctx    context.Context
    config *Config

    // 核心组件
    cluster      *cluster.Cluster      // 集群管理
    metaStore    *meta.Store           // 元数据存储
    distributor  *distributor.Distributor // 分布式管理器

    // 管理器
    nodeManager    *node.Manager        // 节点管理器
    segmentMgr     *segment.Manager     // 段管理器
    loadBalancer  *loadbalancer.Balancer // 负载均衡器
    scheduler     *scheduler.Scheduler   // 任务调度器
    cacheManager  *cache.Manager        // 缓存管理器

    // 服务器
    server *grpc.Server
}

// QueryCoord 初始化
func (q *QueryCoord) Init() error {
    // 1. 初始化集群管理
    q.cluster = cluster.NewCluster(q.config.Cluster)

    // 2. 初始化元数据存储
    q.metaStore = meta.NewStore(q.config.MetaStore)

    // 3. 初始化分布式管理器
    q.distributor = distributor.NewDistributor(q.cluster, q.metaStore)

    // 4. 初始化各个管理器
    q.nodeManager = node.NewManager(q.cluster, q.metaStore)
    q.segmentMgr = segment.NewManager(q.metaStore, q.nodeManager)
    q.loadBalancer = loadbalancer.NewBalancer(q.nodeManager)
    q.scheduler = scheduler.NewScheduler(q.nodeManager, q.segmentMgr)
    q.cacheManager = cache.NewManager(q.config.Cache)

    // 5. 启动后台任务
    q.startBackgroundTasks()

    return nil
}
```

### 查询任务调度

```go
// 任务调度器
type Scheduler struct {
    nodeManager   *node.Manager
    segmentMgr    *segment.Manager
    queue         *task.Queue
    loadBalancer  *loadbalancer.Balancer
    priorityMap   map[task.Priority]int
}

// 调度搜索任务
func (s *Scheduler) ScheduleSearch(req *SearchRequest) (*ScheduleResult, error) {
    // 1. 获取目标集合
    collection, err := s.segmentMgr.GetCollection(req.CollectionID)
    if err != nil {
        return nil, err
    }

    // 2. 获取可用的段
    segments, err := s.segmentMgr.GetSearchableSegments(collection)
    if err != nil {
        return nil, err
    }

    // 3. 获取可用节点
    availableNodes := s.nodeManager.GetAvailableNodes()

    // 4. 计算节点负载
    nodeLoads := s.calculateNodeLoads(availableNodes, segments)

    // 5. 分配段到节点
    nodeAssignments := s.assignSegmentsToNodes(segments, availableNodes, nodeLoads)

    // 6. 创建调度结果
    result := &ScheduleResult{
        CollectionID: collection.ID,
        Tasks:        make(map[int64]*SearchTask),
    }

    for nodeID, assignedSegments := range nodeAssignments {
        result.Tasks[nodeID] = &SearchTask{
            NodeID:    nodeID,
            Segments: assignedSegments,
            Query:     req.Query,
            Limit:     req.Limit,
        }
    }

    return result, nil
}

// 段分配算法
func (s *Scheduler) assignSegmentsToNodes(segments []*Segment, nodes []*Node, nodeLoads map[int64]*NodeLoad) map[int64][]*Segment {
    assignments := make(map[int64][]*Segment)

    // 1. 按段大小排序（从大到小）
    sort.Slice(segments, func(i, j int) bool {
        return segments[i].Size > segments[j].Size
    })

    // 2. 使用贪心算法分配段
    for _, segment := range segments {
        // 3. 找到最合适的节点
        bestNode := s.findBestNodeForSegment(segment, nodes, nodeLoads)

        // 4. 分配段到节点
        assignments[bestNode.ID] = append(assignments[bestNode.ID], segment)

        // 5. 更新节点负载
        nodeLoads[bestNode.ID].AssignedSize += segment.Size
        nodeLoads[bestNode.ID].AssignedCount++
    }

    return assignments
}

// 找到最佳节点
func (s *Scheduler) findBestNodeForSegment(segment *Segment, nodes []*Node, nodeLoads map[int64]*NodeLoad) *Node {
    var bestNode *Node
    bestScore := math.MaxFloat64

    for _, node := range nodes {
        // 1. 计算负载分数
        loadScore := s.calculateLoadScore(node, nodeLoads[node.ID])

        // 2. 计算位置分数（考虑段的位置亲和性）
        locationScore := s.calculateLocationScore(segment, node)

        // 3. 计算总分数
        totalScore := loadScore + locationScore

        // 4. 选择最佳节点
        if totalScore < bestScore {
            bestScore = totalScore
            bestNode = node
        }
    }

    return bestNode
}
```

### 负载均衡策略

```go
// 负载均衡器
type LoadBalancer struct {
    nodeManager *node.Manager
    algorithm   LoadBalanceAlgorithm
    config      *LoadBalanceConfig
}

// 负载均衡算法类型
type LoadBalanceAlgorithm interface {
    SelectNode(nodes []*Node, req *Request) (*Node, error)
}

// 轮询算法
type RoundRobinAlgorithm struct {
    currentIndex int
    mu           sync.Mutex
}

func (rr *RoundRobinAlgorithm) SelectNode(nodes []*Node, req *Request) (*Node, error) {
    rr.mu.Lock()
    defer rr.mu.Unlock()

    if len(nodes) == 0 {
        return nil, fmt.Errorf("no available nodes")
    }

    // 选择下一个节点
    selectedNode := nodes[rr.currentIndex%len(nodes)]
    rr.currentIndex++

    return selectedNode, nil
}

// 加权轮询算法
type WeightedRoundRobinAlgorithm struct {
    currentWeights map[int64]int
    mu            sync.Mutex
}

func (wrr *WeightedRoundRobinAlgorithm) SelectNode(nodes []*Node, req *Request) (*Node, error) {
    wrr.mu.Lock()
    defer wrr.mu.Unlock()

    // 1. 计算当前权重
    totalWeight := 0
    for _, node := range nodes {
        currentWeight := wrr.currentWeights[node.ID] + node.Weight
        wrr.currentWeights[node.ID] = currentWeight
        totalWeight += currentWeight
    }

    // 2. 选择权重最大的节点
    var selectedNode *Node
    maxWeight := 0

    for _, node := range nodes {
        if wrr.currentWeights[node.ID] > maxWeight {
            maxWeight = wrr.currentWeights[node.ID]
            selectedNode = node
        }
    }

    // 3. 减少选中节点的权重
    if selectedNode != nil {
        wrr.currentWeights[selectedNode.ID] -= totalWeight
    }

    return selectedNode, nil
}

// 最少连接算法
type LeastConnectionsAlgorithm struct{}

func (lc *LeastConnectionsAlgorithm) SelectNode(nodes []*Node, req *Request) (*Node, error) {
    if len(nodes) == 0 {
        return nil, fmt.Errorf("no available nodes")
    }

    // 选择连接数最少的节点
    selectedNode := nodes[0]
    minConnections := selectedNode.Connections

    for _, node := range nodes[1:] {
        if node.Connections < minConnections {
            minConnections = node.Connections
            selectedNode = node
        }
    }

    return selectedNode, nil
}
```

## DataCoord：数据生命周期的管理者

### DataCoord 的核心职责

DataCoord 负责数据相关的管理任务：

1. **数据节点管理**：监控数据节点的状态
2. **数据段管理**：管理数据段的生命周期
3. **压缩调度**：调度数据压缩任务
4. **索引管理**：协调索引构建任务
5. **垃圾回收**：清理无用数据

### DataCoord 的架构设计

```go
// DataCoord 核心结构
type DataCoord struct {
    // 基础设施
    ctx    context.Context
    config *Config

    // 核心组件
    cluster     *cluster.Cluster     // 集群管理
    metaStore   *meta.Store          // 元数据存储
    metaTable   *metacache.MetaTable // 元数据表缓存

    // 管理器
    nodeManager       *node.Manager       // 节点管理器
    segmentMgr        *segment.Manager    // 段管理器
    compactionMgr     *compaction.Manager // 压缩管理器
    indexMgr          *index.Manager      // 索引管理器
    garbageCollector  *gc.Collector       // 垃圾回收器
    channelMgr        *channel.Manager    // 通道管理器

    // 服务器
    server *grpc.Server
}

// DataCoord 初始化
func (d *DataCoord) Init() error {
    // 1. 初始化集群管理
    d.cluster = cluster.NewCluster(d.config.Cluster)

    // 2. 初始化元数据存储
    d.metaStore = meta.NewStore(d.config.MetaStore)

    // 3. 初始化元数据表缓存
    d.metaTable = metacache.NewMetaTable(d.metaStore)

    // 4. 初始化各个管理器
    d.nodeManager = node.NewManager(d.cluster, d.metaStore)
    d.segmentMgr = segment.NewManager(d.metaStore, d.nodeManager)
    d.compactionMgr = compaction.NewManager(d.metaStore, d.segmentMgr)
    d.indexMgr = index.NewManager(d.metaStore, d.segmentMgr)
    d.garbageCollector = gc.NewCollector(d.metaStore, d.segmentMgr)
    d.channelMgr = channel.NewManager(d.config.Channel, d.metaStore)

    // 5. 启动后台任务
    d.startBackgroundTasks()

    return nil
}
```

### 数据压缩管理

```go
// 压缩管理器
type CompactionManager struct {
    metaStore    *meta.Store
    segmentMgr   *segment.Manager
    nodeManager  *node.Manager
    compactionQueue *task.Queue
    metrics      *CompactionMetrics
}

// 压缩任务
type CompactionTask struct {
    ID           int64
    SegmentIDs   []int64
    TargetNode   int64
    Priority     Priority
    State        CompactionState
    CreatedAt    int64
    StartedAt    int64
    CompletedAt  int64
    ErrorMessage string
}

// 调度压缩任务
func (cm *CompactionManager) ScheduleCompaction() {
    // 1. 获取压缩候选段
    candidates := cm.getCompactionCandidates()

    // 2. 按优先级排序
    sortedCandidates := cm.sortByPriority(candidates)

    // 3. 创建压缩任务
    for _, candidate := range sortedCandidates {
        // 4. 检查是否需要压缩
        if cm.needCompaction(candidate) {
            // 5. 创建压缩任务
            task := cm.createCompactionTask(candidate)

            // 6. 添加到队列
            cm.compactionQueue.Push(task)
        }
    }
}

// 获取压缩候选段
func (cm *CompactionManager) getCompactionCandidates() []*CompactionCandidate {
    var candidates []*CompactionCandidate

    // 1. 获取所有活跃的段
    segments := cm.segmentMgr.GetActiveSegments()

    // 2. 按集合分组
    segmentsByCollection := cm.groupSegmentsByCollection(segments)

    // 3. 对每个集合的段进行分析
    for collectionID, collectionSegments := range segmentsByCollection {
        // 4. 计算压缩指标
        metrics := cm.calculateCompactionMetrics(collectionSegments)

        // 5. 创建压缩候选
        if metrics.ShouldCompact {
            candidate := &CompactionCandidate{
                CollectionID: collectionID,
                Segments:     collectionSegments,
                Metrics:      metrics,
                Priority:     cm.calculatePriority(metrics),
            }
            candidates = append(candidates, candidate)
        }
    }

    return candidates
}

// 执行压缩任务
func (cm *CompactionManager) executeCompaction(task *CompactionTask) error {
    // 1. 更新任务状态
    task.State = CompactionState_Running
    task.StartedAt = time.Now().Unix()

    // 2. 分配任务到目标节点
    node := cm.nodeManager.GetNode(task.TargetNode)
    if node == nil {
        return fmt.Errorf("target node not found")
    }

    // 3. 执行压缩
    result, err := node.Compact(task.SegmentIDs)
    if err != nil {
        task.State = CompactionState_Failed
        task.ErrorMessage = err.Error()
        return err
    }

    // 4. 更新段元数据
    if err := cm.updateSegmentMetadata(task, result); err != nil {
        task.State = CompactionState_Failed
        task.ErrorMessage = err.Error()
        return err
    }

    // 5. 标记任务完成
    task.State = CompactionState_Completed
    task.CompletedAt = time.Now().Unix()

    // 6. 清理原始段
    if err := cm.cleanupOriginalSegments(task); err != nil {
        log.Warnf("Failed to cleanup original segments: %v", err)
    }

    return nil
}
```

### 索引管理

```go
// 索引管理器
type IndexManager struct {
    metaStore   *meta.Store
    segmentMgr  *segment.Manager
    nodeManager *node.Manager
    indexQueue  *task.Queue
    metrics     *IndexMetrics
}

// 索引任务
type IndexTask struct {
    ID           int64
    SegmentID    int64
    IndexType    IndexType
    IndexParams  map[string]interface{}
    TargetNode   int64
    State        IndexState
    CreatedAt    int64
    StartedAt    int64
    CompletedAt  int64
    ErrorMessage string
}

// 创建索引
func (im *IndexManager) CreateIndex(req *CreateIndexRequest) error {
    // 1. 验证请求
    if err := im.validateRequest(req); err != nil {
        return err
    }

    // 2. 获取段信息
    segment, err := im.segmentMgr.GetSegment(req.SegmentID)
    if err != nil {
        return err
    }

    // 3. 检查是否已存在索引
    if segment.Index != nil {
        return fmt.Errorf("index already exists")
    }

    // 4. 选择目标节点
    targetNode := im.selectTargetNode(segment)
    if targetNode == nil {
        return fmt.Errorf("no available node for index building")
    }

    // 5. 创建索引任务
    task := &IndexTask{
        ID:          im.generateTaskID(),
        SegmentID:   req.SegmentID,
        IndexType:   req.IndexType,
        IndexParams: req.IndexParams,
        TargetNode:  targetNode.ID,
        State:       IndexState_Pending,
        CreatedAt:   time.Now().Unix(),
    }

    // 6. 添加到队列
    im.indexQueue.Push(task)

    return nil
}

// 执行索引构建
func (im *IndexManager) executeIndexBuild(task *IndexTask) error {
    // 1. 更新任务状态
    task.State = IndexState_Running
    task.StartedAt = time.Now().Unix()

    // 2. 获取目标节点
    node := im.nodeManager.GetNode(task.TargetNode)
    if node == nil {
        return fmt.Errorf("target node not found")
    }

    // 3. 构建索引
    result, err := node.BuildIndex(task.SegmentID, task.IndexType, task.IndexParams)
    if err != nil {
        task.State = IndexState_Failed
        task.ErrorMessage = err.Error()
        return err
    }

    // 4. 更新段元数据
    if err := im.updateSegmentIndex(task, result); err != nil {
        task.State = IndexState_Failed
        task.ErrorMessage = err.Error()
        return err
    }

    // 5. 标记任务完成
    task.State = IndexState_Completed
    task.CompletedAt = time.Now().Unix()

    return nil
}
```

### 垃圾回收

```go
// 垃圾回收器
type GarbageCollector struct {
    metaStore   *meta.Store
    segmentMgr  *segment.Manager
    storage     storage.Storage
    config      *GCConfig
    metrics     *GCMetrics
}

// 垃圾回收任务
type GCTask struct {
    ID          int64
    Files       []string
    State       GCState
    CreatedAt   int64
    StartedAt   int64
    CompletedAt int64
    ErrorMessage string
}

// 执行垃圾回收
func (gc *GarbageCollector) Run() {
    // 1. 获取待删除的文件
    filesToDelete := gc.getFilesToDelete()

    if len(filesToDelete) == 0 {
        return
    }

    // 2. 创建垃圾回收任务
    task := &GCTask{
        ID:        gc.generateTaskID(),
        Files:     filesToDelete,
        State:     GCState_Pending,
        CreatedAt: time.Now().Unix(),
    }

    // 3. 执行垃圾回收
    if err := gc.executeGCTask(task); err != nil {
        log.Errorf("Failed to execute GC task: %v", err)
    }
}

// 获取待删除的文件
func (gc *GarbageCollector) getFilesToDelete() []string {
    var files []string

    // 1. 获取存储中的所有文件
    allFiles := gc.storage.List()

    // 2. 获取活跃文件列表
    activeFiles := gc.getActiveFiles()

    // 3. 找出待删除的文件
    for _, file := range allFiles {
        if !gc.isActiveFile(file, activeFiles) {
            // 4. 检查文件是否超过保留期
            if gc.isExpired(file) {
                files = append(files, file)
            }
        }
    }

    return files
}

// 执行垃圾回收任务
func (gc *GarbageCollector) executeGCTask(task *GCTask) error {
    // 1. 更新任务状态
    task.State = GCState_Running
    task.StartedAt = time.Now().Unix()

    // 2. 批量删除文件
    deletedCount := 0
    failedCount := 0

    for _, file := range task.Files {
        if err := gc.storage.Delete(file); err != nil {
            log.Errorf("Failed to delete file %s: %v", file, err)
            failedCount++
        } else {
            deletedCount++
        }
    }

    // 3. 更新任务状态
    if failedCount > 0 {
        task.State = GCState_PartiallyFailed
        task.ErrorMessage = fmt.Sprintf("%d files failed to delete", failedCount)
    } else {
        task.State = GCState_Completed
    }

    task.CompletedAt = time.Now().Unix()

    // 4. 更新指标
    gc.metrics.IncrementDeleted(deletedCount)
    gc.metrics.IncrementFailed(failedCount)

    return nil
}
```

## 协调器间通信

### 事件驱动架构

```go
// 事件总线
type EventBus struct {
    subscribers map[string][]Subscriber
    mu          sync.RWMutex
}

// 订阅者接口
type Subscriber interface {
    HandleEvent(event *Event)
    GetSubscribedTopics() []string
}

// 发布事件
func (eb *EventBus) Publish(topic string, event *Event) error {
    eb.mu.RLock()
    defer eb.mu.RUnlock()

    subscribers := eb.subscribers[topic]
    if len(subscribers) == 0 {
        return nil
    }

    // 异步发送事件给所有订阅者
    for _, subscriber := range subscribers {
        go func(s Subscriber) {
            defer func() {
                if r := recover(); r != nil {
                    log.Errorf("Subscriber panicked: %v", r)
                }
            }()
            s.HandleEvent(event)
        }(subscriber)
    }

    return nil
}

// 订阅事件
func (eb *EventBus) Subscribe(topic string, subscriber Subscriber) error {
    eb.mu.Lock()
    defer eb.mu.Unlock()

    if eb.subscribers == nil {
        eb.subscribers = make(map[string][]Subscriber)
    }

    eb.subscribers[topic] = append(eb.subscribers[topic], subscriber)
    return nil
}
```

### 事件类型定义

```go
// 事件类型
type EventType int

const (
    EventType_CreateCollection EventType = iota
    EventType_DropCollection
    EventType_CreatePartition
    EventType_DropPartition
    EventType_LoadSegment
    EventType_UnloadSegment
   EventType_BuildIndex
    EventType_Compaction
    EventType_NodeAdd
    EventType_NodeRemove
    EventType_NodeHeartbeat
)

// 事件结构
type Event struct {
    ID        int64
    Type      EventType
    Timestamp int64
    Source    string
    Payload   interface{}
}

// 集合事件
type CollectionEvent struct {
    Collection *Collection
    EventType  EventType
    User       string
}

// 段事件
type SegmentEvent struct {
    Segment   *Segment
    EventType EventType
    NodeID    int64
}

// 节点事件
type NodeEvent struct {
    Node      *Node
    EventType EventType
}
```

### 协调器间协作示例

```go
// 处理集合创建事件
func (qc *QueryCoord) handleCollectionCreate(event *Event) {
    collectionEvent := event.Payload.(*CollectionEvent)
    collection := collectionEvent.Collection

    // 1. 为新集合创建段管理器
    qc.segmentMgr.CreateCollectionManager(collection)

    // 2. 预加载段（如果需要）
    if collection.Properties["preload"] == "true" {
        qc.schedulePreload(collection)
    }

    // 3. 更新缓存
    qc.cacheManager.UpdateCollection(collection)
}

// 处理段加载事件
func (qc *QueryCoord) handleSegmentLoad(event *Event) {
    segmentEvent := event.Payload.(*SegmentEvent)
    segment := segmentEvent.Segment

    // 1. 检查是否需要加载
    if qc.shouldLoadSegment(segment) {
        // 2. 选择目标节点
        targetNode := qc.selectNodeForSegment(segment)
        if targetNode != nil {
            // 3. 调度加载任务
            qc.scheduleSegmentLoad(segment, targetNode)
        }
    }
}

// 处理节点故障事件
func (qc *QueryCoord) handleNodeFailure(event *Event) {
    nodeEvent := event.Payload.(*NodeEvent)
    node := nodeEvent.Node

    // 1. 获取节点上的所有段
    segments := qc.segmentMgr.GetSegmentsByNode(node.ID)

    // 2. 重新分配段到其他节点
    for _, segment := range segments {
        // 3. 选择新的目标节点
        targetNode := qc.selectNodeForSegment(segment)
        if targetNode != nil {
            // 4. 重新加载段
            qc.scheduleSegmentLoad(segment, targetNode)
        }
    }
}
```

## 协调器高可用性

### 主备选举机制

```go
// 主备选举器
type LeaderElector struct {
    nodeID       int64
    electionPath string
    client       *clientv3.Client
    leadershipCh chan bool
    isLeader     bool
    mu           sync.RWMutex
}

// 开始选举
func (le *LeaderElector) Start() error {
    // 1. 创建选举会话
    session, err := concurrency.NewSession(le.client, concurrency.WithTTL(10))
    if err != nil {
        return err
    }

    // 2. 创建选举
    election := concurrency.NewElection(session, le.electionPath)

    // 3. 参与选举
    go func() {
        if err := election.Campaign(le.ctx, fmt.Sprintf("%d", le.nodeID)); err != nil {
            log.Errorf("Failed to campaign for leadership: %v", err)
            return
        }

        // 4. 成为领导者
        le.becomeLeader()
    }()

    // 5. 监听领导者变化
    go le.observeLeadership()

    return nil
}

// 观察领导者变化
func (le *LeaderElector) observeLeadership() {
    session, err := concurrency.NewSession(le.client, concurrency.WithTTL(10))
    if err != nil {
        log.Errorf("Failed to create session: %v", err)
        return
    }

    election := concurrency.NewElection(session, le.electionPath)

    for {
        // 6. 获取当前领导者
        leader, err := election.Leader(le.ctx)
        if err != nil {
            log.Errorf("Failed to get leader: %v", err)
            time.Sleep(1 * time.Second)
            continue
        }

        currentLeader, err := strconv.ParseInt(string(leader), 10, 64)
        if err != nil {
            log.Errorf("Failed to parse leader ID: %v", err)
            time.Sleep(1 * time.Second)
            continue
        }

        // 7. 更新领导者状态
        le.updateLeadership(currentLeader)

        time.Sleep(1 * time.Second)
    }
}

// 更新领导者状态
func (le *LeaderElector) updateLeadership(leaderID int64) {
    le.mu.Lock()
    defer le.mu.Unlock()

    wasLeader := le.isLeader
    le.isLeader = (leaderID == le.nodeID)

    // 8. 通知状态变化
    if wasLeader != le.isLeader {
        if le.isLeader {
            le.becomeLeader()
        } else {
            le.becomeFollower()
        }
    }
}

// 成为领导者
func (le *LeaderElector) becomeLeader() {
    le.mu.Lock()
    le.isLeader = true
    le.mu.Unlock()

    log.Infof("Node %d became leader", le.nodeID)

    // 9. 通知外部
    if le.leadershipCh != nil {
        le.leadershipCh <- true
    }
}

// 成为跟随者
func (le *LeaderElector) becomeFollower() {
    le.mu.Lock()
    le.isLeader = false
    le.mu.Unlock()

    log.Infof("Node %d became follower", le.nodeID)

    // 10. 通知外部
    if le.leadershipCh != nil {
        le.leadershipCh <- false
    }
}
```

### 故障恢复机制

```go
// 故障恢复管理器
type FailureRecoveryManager struct {
    nodeManager *node.Manager
    segmentMgr  *segment.Manager
    recoveryMap map[int64]*RecoveryTask
    mu          sync.RWMutex
}

// 故障恢复任务
type RecoveryTask struct {
    NodeID         int64
    FailureTime    int64
    Segments       []*Segment
    RecoveryTime   int64
    State          RecoveryState
    TargetNodes    map[int64][]*Segment
    ErrorMessage   string
}

// 处理节点故障
func (frm *FailureRecoveryManager) HandleNodeFailure(nodeID int64) {
    // 1. 创建恢复任务
    task := &RecoveryTask{
        NodeID:      nodeID,
        FailureTime: time.Now().Unix(),
        State:       RecoveryState_Detecting,
    }

    // 2. 获取故障节点的段
    segments := frm.segmentMgr.GetSegmentsByNode(nodeID)
    task.Segments = segments

    // 3. 标记段为不可用
    frm.segmentMgr.MarkSegmentsUnavailable(segments)

    // 4. 开始恢复过程
    frm.startRecovery(task)
}

// 开始恢复过程
func (frm *FailureRecoveryManager) startRecovery(task *RecoveryTask) {
    // 5. 更新任务状态
    task.State = RecoveryState_Planing

    // 6. 计算恢复计划
    recoveryPlan := frm.calculateRecoveryPlan(task)

    // 7. 分配段到新节点
    task.TargetNodes = frm.assignSegmentsToNodes(task.Segments, recoveryPlan)

    // 8. 执行恢复
    frm.executeRecovery(task)
}

// 执行恢复
func (frm *FailureRecoveryManager) executeRecovery(task *RecoveryTask) {
    // 9. 更新任务状态
    task.State = RecoveryState_Executing

    // 10. 并行恢复段
    var wg sync.WaitGroup
    errorChan := make(chan error, len(task.TargetNodes))

    for targetNodeID, segments := range task.TargetNodes {
        wg.Add(1)
        go func(nodeID int64, segs []*Segment) {
            defer wg.Done()

            // 11. 在目标节点加载段
            if err := frm.loadSegmentsOnNode(nodeID, segs); err != nil {
                errorChan <- err
                return
            }
        }(targetNodeID, segments)
    }

    // 12. 等待所有恢复完成
    wg.Wait()
    close(errorChan)

    // 13. 检查错误
    var errors []error
    for err := range errorChan {
        errors = append(errors, err)
    }

    if len(errors) > 0 {
        // 14. 部分失败
        task.State = RecoveryState_PartiallyFailed
        task.ErrorMessage = fmt.Sprintf("%d segments failed to recover", len(errors))
    } else {
        // 15. 完全恢复
        task.State = RecoveryState_Completed
        task.RecoveryTime = time.Now().Unix()
    }

    // 16. 更新段状态
    if task.State == RecoveryState_Completed {
        frm.segmentMgr.MarkSegmentsAvailable(task.Segments)
    }
}
```

## 协调器性能优化

### 缓存策略

```go
// 多级缓存管理器
type CacheManager struct {
    // L1 缓存（内存缓存）
    l1Cache *lru.Cache

    // L2 缓存（分布式缓存）
    l2Cache *redis.Client

    // 缓存策略
    config *CacheConfig

    // 指标
    metrics *CacheMetrics
}

// 获取缓存项
func (cm *CacheManager) Get(key string) (interface{}, bool) {
    // 1. 首先检查 L1 缓存
    if value, found := cm.l1Cache.Get(key); found {
        cm.metrics.IncrementL1Hit()
        return value, true
    }

    // 2. 检查 L2 缓存
    if cm.l2Cache != nil {
        if value, err := cm.l2Cache.Get(key).Result(); err == nil {
            // 3. 回填 L1 缓存
            cm.l1Cache.Add(key, value)
            cm.metrics.IncrementL2Hit()
            return value, true
        }
    }

    cm.metrics.IncrementMiss()
    return nil, false
}

// 设置缓存项
func (cm *CacheManager) Set(key string, value interface{}, ttl time.Duration) {
    // 1. 设置 L1 缓存
    cm.l1Cache.Add(key, value)

    // 2. 设置 L2 缓存
    if cm.l2Cache != nil {
        cm.l2Cache.Set(key, value, ttl)
    }

    cm.metrics.IncrementSet()
}

// 缓存预热
func (cm *CacheManager) WarmUp(items map[string]interface{}) {
    // 3. 并行预热缓存
    var wg sync.WaitGroup
    semaphore := make(chan struct{}, cm.config.Concurrency)

    for key, value := range items {
        wg.Add(1)
        go func(k string, v interface{}) {
            defer wg.Done()

            semaphore <- struct{}{}
            defer func() { <-semaphore }>()

            cm.Set(k, v, cm.config.DefaultTTL)
        }(key, value)
    }

    wg.Wait()
}
```

### 批量操作优化

```go
// 批量操作管理器
type BatchManager struct {
    // 批量队列
    queue *task.Queue

    // 批量处理器
    processor BatchProcessor

    // 配置
    config *BatchConfig

    // 指标
    metrics *BatchMetrics
}

// 批量处理器接口
type BatchProcessor interface {
    ProcessBatch(items []interface{}) error
    GetBatchSize() int
    GetBatchTimeout() time.Duration
}

// 批量处理循环
func (bm *BatchManager) Start() {
    ticker := time.NewTicker(bm.processor.GetBatchTimeout())
    defer ticker.Stop()

    batch := make([]interface{}, 0, bm.processor.GetBatchSize())

    for {
        select {
        case item := <-bm.queue.Chan():
            // 1. 添加到批次
            batch = append(batch, item)

            // 2. 检查批次大小
            if len(batch) >= bm.processor.GetBatchSize() {
                if err := bm.processBatch(batch); err != nil {
                    log.Errorf("Failed to process batch: %v", err)
                }
                batch = batch[:0]
            }

        case <-ticker.C:
            // 3. 处理超时的批次
            if len(batch) > 0 {
                if err := bm.processBatch(batch); err != nil {
                    log.Errorf("Failed to process batch: %v", err)
                }
                batch = batch[:0]
            }
        }
    }
}

// 处理批次
func (bm *BatchManager) processBatch(items []interface{}) error {
    if len(items) == 0 {
        return nil
    }

    start := time.Now()
    defer func() {
        bm.metrics.ObserveBatchSize(len(items))
        bm.metrics.ObserveBatchDuration(time.Since(start))
    }()

    // 4. 处理批次
    if err := bm.processor.ProcessBatch(items); err != nil {
        bm.metrics.IncrementFailedBatches()
        return err
    }

    bm.metrics.IncrementSuccessfulBatches()
    return nil
}
```

## 协调器监控和诊断

### 指标收集

```go
// 指标收集器
type MetricsCollector struct {
    // 系统指标
    systemMetrics *SystemMetrics

    // 协调器指标
    coordinatorMetrics *CoordinatorMetrics

    // 任务指标
    taskMetrics *TaskMetrics

    // 存储指标
    storageMetrics *StorageMetrics

    // 指标导出器
    exporter *prometheus.Exporter
}

// 收集系统指标
func (mc *MetricsCollector) CollectSystemMetrics() {
    // 1. CPU 使用率
    cpuUsage := mc.getCPUUsage()
    mc.systemMetrics.CPUUsage.Set(cpuUsage)

    // 2. 内存使用
    memoryUsage := mc.getMemoryUsage()
    mc.systemMetrics.MemoryUsage.Set(memoryUsage)

    // 3. 磁盘使用
    diskUsage := mc.getDiskUsage()
    mc.systemMetrics.DiskUsage.Set(diskUsage)

    // 4. 网络使用
    networkUsage := mc.getNetworkUsage()
    mc.systemMetrics.NetworkUsage.Set(networkUsage)
}

// 收集协调器指标
func (mc *MetricsCollector) CollectCoordinatorMetrics() {
    // 5. 节点数量
    nodeCount := mc.getNodeCount()
    mc.coordinatorMetrics.NodeCount.Set(nodeCount)

    // 6. 段数量
    segmentCount := mc.getSegmentCount()
    mc.coordinatorMetrics.SegmentCount.Set(segmentCount)

    // 7. 任务队列长度
    queueLength := mc.getQueueLength()
    mc.coordinatorMetrics.QueueLength.Set(queueLength)

    // 8. 请求处理时间
    requestLatency := mc.getRequestLatency()
    mc.coordinatorMetrics.RequestLatency.Observe(requestLatency)
}
```

### 健康检查

```go
// 健康检查器
type HealthChecker struct {
    // 检查项目
    checks map[string]HealthCheck

    // 配置
    config *HealthConfig

    // 健康状态
    status map[string]HealthStatus

    // 保护机制
    mu sync.RWMutex
}

// 健康检查接口
type HealthCheck interface {
    Check() (HealthStatus, error)
    GetName() string
}

// 健康状态
type HealthStatus struct {
    Status    HealthState
    Message   string
    Timestamp int64
    Duration  time.Duration
}

// 健康状态类型
type HealthState int

const (
    HealthState_Healthy HealthState = iota
    HealthState_Degraded
    HealthState_Unhealthy
)

// 运行健康检查
func (hc *HealthChecker) Run() {
    ticker := time.NewTicker(hc.config.Interval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            hc.runChecks()
        }
    }
}

// 运行检查
func (hc *HealthChecker) runChecks() {
    hc.mu.Lock()
    defer hc.mu.Unlock()

    results := make(map[string]HealthStatus)

    // 并行执行检查
    var wg sync.WaitGroup
    for name, check := range hc.checks {
        wg.Add(1)
        go func(n string, c HealthCheck) {
            defer wg.Done()

            start := time.Now()
            status, err := c.Check()
            duration := time.Since(start)

            if err != nil {
                status = HealthStatus{
                    Status:    HealthState_Unhealthy,
                    Message:   err.Error(),
                    Timestamp: time.Now().Unix(),
                    Duration:  duration,
                }
            } else {
                status.Timestamp = time.Now().Unix()
                status.Duration = duration
            }

            results[n] = status
        }(name, check)
    }

    wg.Wait()

    // 更新状态
    hc.status = results

    // 计算整体健康状态
    overallStatus := hc.calculateOverallStatus(results)
    hc.updateOverallHealth(overallStatus)
}

// 计算整体健康状态
func (hc *HealthChecker) calculateOverallStatus(results map[string]HealthStatus) HealthStatus {
    healthyCount := 0
    degradedCount := 0
    unhealthyCount := 0

    for _, status := range results {
        switch status.Status {
        case HealthState_Healthy:
            healthyCount++
        case HealthState_Degraded:
            degradedCount++
        case HealthState_Unhealthy:
            unhealthyCount++
        }
    }

    total := len(results)
    if total == 0 {
        return HealthStatus{
            Status:    HealthState_Unhealthy,
            Message:   "No health checks configured",
            Timestamp: time.Now().Unix(),
        }
    }

    // 计算健康度
    healthRatio := float64(healthyCount) / float64(total)
    degradedRatio := float64(degradedCount) / float64(total)

    var overallStatus HealthState
    var message string

    switch {
    case healthRatio >= 0.8:
        overallStatus = HealthState_Healthy
        message = fmt.Sprintf("System is healthy (%d/%d checks passed)", healthyCount, total)
    case healthRatio >= 0.5 || degradedRatio <= 0.3:
        overallStatus = HealthState_Degraded
        message = fmt.Sprintf("System is degraded (%d healthy, %d degraded, %d unhealthy)", healthyCount, degradedCount, unhealthyCount)
    default:
        overallStatus = HealthState_Unhealthy
        message = fmt.Sprintf("System is unhealthy (%d unhealthy checks)", unhealthyCount)
    }

    return HealthStatus{
        Status:    overallStatus,
        Message:   message,
        Timestamp: time.Now().Unix(),
    }
}
```

## 总结

Milvus 的协调器层是整个系统的核心控制中枢，通过 RootCoord、QueryCoord、DataCoord 三大协调器的紧密协作，实现了：

1. **全局资源管理**：统一管理系统的所有资源
2. **智能任务调度**：根据负载和资源状况智能调度任务
3. **数据一致性**：通过时间戳和版本控制确保数据一致性
4. **高可用性**：通过主备选举和故障恢复确保服务连续性
5. **性能优化**：通过缓存、批量处理等机制提升系统性能

协调器的设计充分体现了分布式系统的复杂性管理，通过精心设计的架构和算法，确保了 Milvus 能够在大规模分布式环境中稳定运行。

在下一篇文章中，我们将深入探讨 Milvus 的计算层组件，了解 QueryNode 和 DataNode 如何协同工作来处理海量向量数据。

---

**作者简介**：本文作者专注于分布式系统架构和协调器设计，拥有丰富的分布式协调服务开发经验。

**相关链接**：
- [Milvus 协调器文档](https://milvus.io/docs/coordinator_overview.md)
- [分布式系统设计模式](https://martinfowler.com/articles/patterns-of-distributed-systems/)
- [Raft 一致性算法](https://raft.github.io/)