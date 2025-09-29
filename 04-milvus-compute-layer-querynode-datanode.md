# Milvus 计算层剖析：QueryNode 和 DataNode 的技术实现

## 前言

在 Milvus 的分布式架构中，计算层是执行实际数据处理和向量搜索的核心组件。QueryNode 负责处理查询请求，DataNode 负责数据写入和存储。本文将深入剖析这两个关键组件的技术实现，了解它们如何协同工作来提供高性能的向量搜索能力。

## 计算层架构概览

### 计算层的定位

Milvus 的计算层是系统的执行引擎，主要负责：

1. **向量搜索**：执行高维向量的相似性搜索
2. **数据处理**：处理数据的写入、更新和删除
3. **索引管理**：管理各种类型的向量索引
4. **缓存优化**：通过多级缓存提升查询性能
5. **负载均衡**：支持动态负载分配和故障转移

### 计算层组件架构

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
    │ QueryCoord │
    └────┬────┘
         │
    ┌────┴────┐
    │ DataCoord  │
    └────┬────┘
         │
┌────────┴────────┐
│  Compute Layer  │
├─────────────────┤
│  QueryNode 1    │
│  QueryNode 2    │
│  QueryNode 3    │
│  DataNode 1     │
│  DataNode 2     │
│  DataNode 3     │
│  StreamingNode  │
└─────────────────┘
         │
┌────────┴────────┐
│  Storage Layer  │
│ (Object Storage)│
└─────────────────┘
```

## QueryNode：向量搜索的执行引擎

### QueryNode 的核心职责

QueryNode 是向量搜索的主要执行单元，负责：

1. **段管理**：加载和管理数据段
2. **查询执行**：执行向量搜索和过滤操作
3. **索引查询**：使用各种索引类型进行高效搜索
4. **结果聚合**：聚合多个段的查询结果
5. **缓存管理**：管理查询相关的缓存数据

### QueryNode 的架构设计

```go
// QueryNode 核心结构
type QueryNode struct {
    // 基础设施
    ctx    context.Context
    config *Config

    // 核心组件
    segmentMgr     *segment.Manager     // 段管理器
    queryExecutor  *query.Executor      // 查询执行器
    indexMgr       *index.Manager       // 索引管理器
    cacheMgr       *cache.Manager       // 缓存管理器

    // 存储组件
    storage        storage.Storage      // 存储接口
    metaStore      meta.MetaStore       // 元数据存储

    // 资源管理
    memoryMgr      *memory.Manager      // 内存管理器
    resourceMgr    *resource.Manager    // 资源管理器

    // 监控和指标
    metrics        *Metrics             // 指标收集器
    healthChecker  *health.Checker      // 健康检查器

    // 服务器
    server         *grpc.Server
}

// QueryNode 初始化
func (qn *QueryNode) Init() error {
    // 1. 初始化存储组件
    qn.storage = storage.NewStorage(qn.config.Storage)
    qn.metaStore = meta.NewMetaStore(qn.config.MetaStore)

    // 2. 初始化核心管理器
    qn.segmentMgr = segment.NewManager(qn.metaStore, qn.storage)
    qn.queryExecutor = query.NewExecutor(qn.config.Query)
    qn.indexMgr = index.NewManager(qn.config.Index)
    qn.cacheMgr = cache.NewManager(qn.config.Cache)

    // 3. 初始化资源管理器
    qn.memoryMgr = memory.NewManager(qn.config.Memory)
    qn.resourceMgr = resource.NewManager(qn.config.Resource)

    // 4. 初始化监控组件
    qn.metrics = metrics.NewCollector(qn.config.Metrics)
    qn.healthChecker = health.NewChecker(qn.config.Health)

    // 5. 启动后台任务
    qn.startBackgroundTasks()

    return nil
}
```

### 段管理系统

```go
// 段管理器
type SegmentManager struct {
    // 段缓存
    segments *sync.Map // map[int64]*Segment

    // 加载器
    loader *SegmentLoader

    // 卸载器
    unloader *SegmentUnloader

    // 状态管理
    stateMgr *SegmentStateManager

    // 监控指标
    metrics *SegmentMetrics
}

// 段信息
type Segment struct {
    ID           int64
    CollectionID int64
    PartitionID  int64
    State        SegmentState
    Type         SegmentType
    Size         int64
    RowCount     int64
    IndexInfo    *IndexInfo
    LoadedNodes  []int64
    CreatedAt    int64
    UpdatedAt    int64
}

// 段状态
type SegmentState int

const (
    SegmentState_Creating SegmentState = iota
    SegmentState_Growing
    SegmentState_Flushed
    SegmentState_Flushing
    SegmentState_Compacting
    SegmentState_Dropped
)

// 段类型
type SegmentType int

const (
    SegmentType_Memory SegmentType = iota
    SegmentType_Persistent
    SegmentType_Indexed
)

// 段加载器
type SegmentLoader struct {
    // 存储接口
    storage storage.Storage

    // 内存管理器
    memoryMgr *memory.Manager

    // 索引管理器
    indexMgr *index.Manager

    // 加载策略
    strategy LoadStrategy
}

// 加载策略
type LoadStrategy interface {
    ShouldLoad(segment *Segment) bool
    GetLoadPriority(segment *Segment) int
    SelectLoadingNode(segment *Segment, nodes []*QueryNode) *QueryNode
}

// 段加载
func (sl *SegmentLoader) LoadSegment(segment *Segment) error {
    // 1. 检查是否已经加载
    if sl.isSegmentLoaded(segment.ID) {
        return nil
    }

    // 2. 检查内存资源
    if !sl.memoryMgr.CheckMemory(segment.Size) {
        return fmt.Errorf("insufficient memory for segment %d", segment.ID)
    }

    // 3. 准备内存空间
    memoryHandle, err := sl.memoryMgr.Allocate(segment.Size)
    if err != nil {
        return fmt.Errorf("failed to allocate memory: %v", err)
    }

    // 4. 从存储加载段数据
    segmentData, err := sl.storage.LoadSegment(segment.ID)
    if err != nil {
        sl.memoryMgr.Free(memoryHandle)
        return fmt.Errorf("failed to load segment data: %v", err)
    }

    // 5. 加载索引（如果有的话）
    if segment.IndexInfo != nil {
        if err := sl.indexMgr.LoadIndex(segment, segmentData); err != nil {
            sl.memoryMgr.Free(memoryHandle)
            return fmt.Errorf("failed to load index: %v", err)
        }
    }

    // 6. 初始化段内存结构
    if err := sl.initializeSegmentMemory(segment, segmentData, memoryHandle); err != nil {
        sl.memoryMgr.Free(memoryHandle)
        return fmt.Errorf("failed to initialize segment memory: %v", err)
    }

    // 7. 更新段状态
    segment.State = SegmentState_Loaded

    return nil
}

// 段卸载器
type SegmentUnloader struct {
    // 内存管理器
    memoryMgr *memory.Manager

    // 卸载策略
    strategy UnloadStrategy
}

// 卸载策略
type UnloadStrategy interface {
    ShouldUnload(segment *Segment) bool
    GetUnloadPriority(segment *Segment) int
}

// 段卸载
func (su *SegmentUnloader) UnloadSegment(segment *Segment) error {
    // 1. 检查是否可以卸载
    if !su.strategy.ShouldUnload(segment) {
        return nil
    }

    // 2. 停止查询访问
    su.stopQueryAccess(segment.ID)

    // 3. 持久化内存数据（如果需要）
    if err := su.persistSegmentData(segment); err != nil {
        return fmt.Errorf("failed to persist segment data: %v", err)
    }

    // 4. 释放内存
    if err := su.memoryMgr.FreeSegmentMemory(segment.ID); err != nil {
        return fmt.Errorf("failed to free memory: %v", err)
    }

    // 5. 更新段状态
    segment.State = SegmentState_Unloaded

    return nil
}
```

### 查询执行引擎

```go
// 查询执行器
type QueryExecutor struct {
    // 查询计划生成器
    planner *QueryPlanner

    // 查询优化器
    optimizer *QueryOptimizer

    // 查询执行器
    executor *QueryExecutionEngine

    // 结果处理器
    resultProcessor *ResultProcessor
}

// 查询请求
type SearchRequest struct {
    CollectionID int64
    PartitionIDs []int64
    QueryVector  []float32
    TopK         int
    MetricType   MetricType
    Expr         string
    Params       map[string]interface{}
}

// 查询响应
type SearchResponse struct {
    Results      []SearchResult
    TotalHits    int
    QueryLatency time.Duration
    ExecutionLatency time.Duration
}

// 搜索结果
type SearchResult struct {
    ID      int64
    Score   float32
    Fields  map[string]interface{}
}

// 执行搜索
func (qe *QueryExecutor) ExecuteSearch(req *SearchRequest) (*SearchResponse, error) {
    start := time.Now()

    // 1. 查询计划生成
    plan, err := qe.planner.CreatePlan(req)
    if err != nil {
        return nil, fmt.Errorf("failed to create query plan: %v", err)
    }

    // 2. 查询优化
    optimizedPlan, err := qe.optimizer.Optimize(plan)
    if err != nil {
        return nil, fmt.Errorf("failed to optimize query: %v", err)
    }

    // 3. 执行查询
    executionStart := time.Now()
    results, err := qe.executor.Execute(optimizedPlan)
    if err != nil {
        return nil, fmt.Errorf("failed to execute query: %v", err)
    }

    executionLatency := time.Since(executionStart)

    // 4. 处理结果
    processedResults, err := qe.resultProcessor.Process(results, req.TopK)
    if err != nil {
        return nil, fmt.Errorf("failed to process results: %v", err)
    }

    // 5. 构建响应
    response := &SearchResponse{
        Results:          processedResults,
        TotalHits:        len(results),
        QueryLatency:     time.Since(start),
        ExecutionLatency: executionLatency,
    }

    return response, nil
}

// 查询计划生成器
type QueryPlanner struct {
    // 元数据存储
    metaStore meta.MetaStore

    // 查询解析器
    parser *QueryParser
}

// 查询计划
type QueryPlan struct {
    ID          string
    CollectionID int64
    Steps       []QueryStep
    Cost        float64
}

// 查询步骤
type QueryStep struct {
    Type      QueryStepType
    SegmentID int64
    IndexType IndexType
    Filter    *FilterExpr
    Limit     int
    Cost      float64
}

// 查询步骤类型
type QueryStepType int

const (
    QueryStepType_Scan QueryStepType = iota
    QueryStepType_IndexSearch
    QueryStepType_Filter
    QueryStepType_Limit
    QueryStepType_Aggregate
)

// 创建查询计划
func (qp *QueryPlanner) CreatePlan(req *SearchRequest) (*QueryPlan, error) {
    // 1. 获取集合信息
    collection, err := qp.metaStore.GetCollection(req.CollectionID)
    if err != nil {
        return nil, fmt.Errorf("failed to get collection: %v", err)
    }

    // 2. 获取相关段
    segments, err := qp.metaStore.GetSegmentsByCollection(req.CollectionID)
    if err != nil {
        return nil, fmt.Errorf("failed to get segments: %v", err)
    }

    // 3. 过滤目标段
    targetSegments := qp.filterSegments(segments, req)

    // 4. 构建查询步骤
    steps := make([]QueryStep, 0, len(targetSegments))
    totalCost := 0.0

    for _, segment := range targetSegments {
        step := QueryStep{
            SegmentID: segment.ID,
            Limit:     req.TopK,
        }

        // 5. 选择查询策略
        if segment.IndexInfo != nil {
            step.Type = QueryStepType_IndexSearch
            step.IndexType = segment.IndexInfo.Type
            step.Cost = qp.estimateIndexSearchCost(segment, req)
        } else {
            step.Type = QueryStepType_Scan
            step.Cost = qp.estimateScanCost(segment, req)
        }

        // 6. 添加过滤条件
        if req.Expr != "" {
            filter, err := qp.parser.ParseFilter(req.Expr)
            if err != nil {
                return nil, fmt.Errorf("failed to parse filter: %v", err)
            }
            step.Filter = filter
        }

        steps = append(steps, step)
        totalCost += step.Cost
    }

    // 7. 添加聚合步骤
    if len(steps) > 1 {
        steps = append(steps, QueryStep{
            Type: QueryStepType_Aggregate,
            Limit: req.TopK,
            Cost: qp.estimateAggregateCost(len(steps), req.TopK),
        })
        totalCost += steps[len(steps)-1].Cost
    }

    // 8. 添加限制步骤
    if req.TopK > 0 {
        steps = append(steps, QueryStep{
            Type: QueryStepType_Limit,
            Limit: req.TopK,
            Cost: qp.estimateLimitCost(req.TopK),
        })
        totalCost += steps[len(steps)-1].Cost
    }

    return &QueryPlan{
        ID:          qp.generatePlanID(),
        CollectionID: req.CollectionID,
        Steps:       steps,
        Cost:        totalCost,
    }, nil
}

// 查询优化器
type QueryOptimizer struct {
    // 统计信息
    statistics *StatisticsManager

    // 成本估算器
    costEstimator *CostEstimator

    // 优化规则
    rules []OptimizationRule
}

// 优化规则
type OptimizationRule interface {
    Apply(plan *QueryPlan) (*QueryPlan, error)
    GetName() string
}

// 查询优化
func (qo *QueryOptimizer) Optimize(plan *QueryPlan) (*QueryPlan, error) {
    optimizedPlan := plan

    // 应用所有优化规则
    for _, rule := range qo.rules {
        var err error
        optimizedPlan, err = rule.Apply(optimizedPlan)
        if err != nil {
            log.Warnf("Optimization rule %s failed: %v", rule.GetName(), err)
            continue
        }
    }

    return optimizedPlan, nil
}

// 并行执行优化规则
type ParallelExecutionRule struct{}

func (r *ParallelExecutionRule) Apply(plan *QueryPlan) (*QueryPlan, error) {
    // 1. 识别可以并行执行的步骤
    parallelGroups := r.identifyParallelGroups(plan.Steps)

    // 2. 重新组织执行顺序
    newSteps := make([]QueryStep, 0, len(plan.Steps))

    for _, group := range parallelGroups {
        if len(group) > 1 {
            // 3. 添加并行执行标记
            for i, step := range group {
                step.Params = make(map[string]interface{})
                step.Params["parallel_group"] = group[0].SegmentID
                step.Params["parallel_index"] = i
                newSteps = append(newSteps, step)
            }
        } else {
            newSteps = append(newSteps, group[0])
        }
    }

    // 4. 更新计划
    plan.Steps = newSteps
    return plan, nil
}

// 索引选择优化规则
type IndexSelectionRule struct {
    statistics *StatisticsManager
}

func (r *IndexSelectionRule) Apply(plan *QueryPlan) (*QueryPlan, error) {
    for i, step := range plan.Steps {
        if step.Type == QueryStepType_IndexSearch {
            // 5. 评估是否使用索引更优
            if r.shouldUseScanInstead(step) {
                step.Type = QueryStepType_Scan
                step.IndexType = ""
                plan.Steps[i] = step
            }
        }
    }
    return plan, nil
}

// 查询执行引擎
type QueryExecutionEngine struct {
    // 线程池
    threadPool *ThreadPool

    // 段执行器
    segmentExecutor *SegmentExecutor

    // 结果聚合器
    resultAggregator *ResultAggregator
}

// 线程池
type ThreadPool struct {
    workers  []*Worker
    taskChan chan Task
    wg       sync.WaitGroup
}

// 工作线程
type Worker struct {
    id     int
    pool   *ThreadPool
    stopCh chan struct{}
}

// 执行查询计划
func (qee *QueryExecutionEngine) Execute(plan *QueryPlan) ([]SearchResult, error) {
    // 6. 并行执行查询步骤
    resultChan := make(chan []SearchResult, len(plan.Steps))
    errorChan := make(chan error, len(plan.Steps))

    var wg sync.WaitGroup

    for _, step := range plan.Steps {
        wg.Add(1)
        go func(s QueryStep) {
            defer wg.Done()

            // 7. 提交任务到线程池
            result, err := qee.threadPool.Submit(func() ([]SearchResult, error) {
                return qee.segmentExecutor.ExecuteStep(s)
            })

            if err != nil {
                errorChan <- err
                return
            }

            resultChan <- result
        }(step)
    }

    // 8. 等待所有步骤完成
    go func() {
        wg.Wait()
        close(resultChan)
        close(errorChan)
    }()

    // 9. 收集结果
    var allResults []SearchResult
    for results := range resultChan {
        allResults = append(allResults, results...)
    }

    // 10. 检查错误
    if err := <-errorChan; err != nil {
        return nil, err
    }

    // 11. 聚合结果
    finalResults := qee.resultAggregator.Aggregate(allResults, plan.Steps)

    return finalResults, nil
}

// 段执行器
type SegmentExecutor struct {
    // 索引执行器
    indexExecutors map[IndexType]IndexExecutor

    // 扫描执行器
    scanExecutor *ScanExecutor

    // 过滤执行器
    filterExecutor *FilterExecutor
}

// 索引执行器接口
type IndexExecutor interface {
    Execute(segment *Segment, queryVector []float32, topK int) ([]SearchResult, error)
    GetName() string
}

// 执行查询步骤
func (se *SegmentExecutor) ExecuteStep(step QueryStep) ([]SearchResult, error) {
    // 12. 获取段信息
    segment, err := se.getSegment(step.SegmentID)
    if err != nil {
        return nil, err
    }

    // 13. 执行查询
    switch step.Type {
    case QueryStepType_Scan:
        return se.scanExecutor.Execute(segment, step)

    case QueryStepType_IndexSearch:
        executor, ok := se.indexExecutors[step.IndexType]
        if !ok {
            return nil, fmt.Errorf("unsupported index type: %v", step.IndexType)
        }
        return executor.Execute(segment, step.QueryVector, step.Limit)

    default:
        return nil, fmt.Errorf("unsupported query step type: %v", step.Type)
    }
}

// HNSW 索引执行器
type HNSWExecutor struct {
    // HNSW 参数
    efConstruction int
    efSearch       int
    maxConnections int

    // 量化器
    quantizer *Quantizer
}

func (he *HNSWExecutor) Execute(segment *Segment, queryVector []float32, topK int) ([]SearchResult, error) {
    // 14. 获取 HNSW 索引
    hnswIndex, err := he.getHNSWIndex(segment)
    if err != nil {
        return nil, err
    }

    // 15. 量化查询向量
    quantizedQuery := he.quantizer.Quantize(queryVector)

    // 16. 执行 HNSW 搜索
    candidates := he.searchHNW(hnswIndex, quantizedQuery, he.efSearch)

    // 17. 重排序
    results := he.rerank(candidates, queryVector, topK)

    return results, nil
}

// IVF 索引执行器
type IVFExecutor struct {
    // IVF 参数
    nlist int
    nprobe int

    // 量化器
    quantizer *Quantizer
}

func (ie *IVFExecutor) Execute(segment *Segment, queryVector []float32, topK int) ([]SearchResult, error) {
    // 18. 获取 IVF 索引
    ivfIndex, err := ie.getIVFIndex(segment)
    if err != nil {
        return nil, err
    }

    // 19. 量化查询向量
    quantizedQuery := ie.quantizer.Quantize(queryVector)

    // 20. 计算查询向量到聚类中心的距离
    distances := ie.calculateClusterDistances(quantizedQuery, ivfIndex.centroids)

    // 21. 选择最近的聚类
    selectedClusters := ie.selectClusters(distances, ie.nprobe)

    // 22. 在选定的聚类中搜索
    results := ie.searchInClusters(ivfIndex, quantizedQuery, selectedClusters, topK)

    return results, nil
}

// 结果聚合器
type ResultAggregator struct {
    // 排序器
    sorter *ResultSorter

    // 去重器
    deduplicator *ResultDeduplicator
}

// 聚合结果
func (ra *ResultAggregator) Aggregate(results []SearchResult, steps []QueryStep) []SearchResult {
    // 23. 去重
    uniqueResults := ra.deduplicator.Deduplicate(results)

    // 24. 排序
    sortedResults := ra.sorter.Sort(uniqueResults)

    // 25. 限制结果数量
    maxResults := ra.getMaxResults(steps)
    if len(sortedResults) > maxResults {
        sortedResults = sortedResults[:maxResults]
    }

    return sortedResults
}
```

### 索引管理系统

```go
// 索引管理器
type IndexManager struct {
    // 索引缓存
    indices *sync.Map // map[int64]*Index

    // 索引构建器
    builders map[IndexType]IndexBuilder

    // 索引加载器
    loader *IndexLoader

    // 索引统计
    statistics *IndexStatistics
}

// 索引信息
type IndexInfo struct {
    ID          int64
    SegmentID   int64
    Type        IndexType
    Params      map[string]interface{}
    State       IndexState
    Size        int64
    CreatedAt   int64
    UpdatedAt   int64
}

// 索引状态
type IndexState int

const (
    IndexState_Building IndexState = iota
    IndexState_Built
    IndexState_Loading
    IndexState_Loaded
    IndexState_Failed
)

// 索引构建器接口
type IndexBuilder interface {
    Build(segment *Segment, params map[string]interface{}) (*Index, error)
    GetName() string
    GetSupportedTypes() []IndexType
}

// HNSW 索引构建器
type HNSWBuilder struct {
    // 构建参数
    efConstruction int
    maxConnections int
    metricType    MetricType
}

func (hb *HNSWBuilder) Build(segment *Segment, params map[string]interface{}) (*Index, error) {
    // 1. 提取向量数据
    vectors, err := hb.extractVectors(segment)
    if err != nil {
        return nil, err
    }

    // 2. 构建参数
    if efConstruction, ok := params["efConstruction"].(int); ok {
        hb.efConstruction = efConstruction
    }
    if maxConnections, ok := params["maxConnections"].(int); ok {
        hb.maxConnections = maxConnections
    }

    // 3. 构建 HNSW 图
    hnswIndex := &HNSWIndex{
        MaxConnections: hb.maxConnections,
        EfConstruction: hb.efConstruction,
        MetricType:    hb.metricType,
        Vectors:       vectors,
    }

    // 4. 插入向量
    for i, vector := range vectors {
        if err := hnswIndex.Insert(vector, i); err != nil {
            return nil, fmt.Errorf("failed to insert vector %d: %v", i, err)
        }
    }

    // 5. 优化图结构
    hnswIndex.Optimize()

    // 6. 返回索引
    return &Index{
        Type:  IndexType_HNSW,
        Data:  hnswIndex,
        Size:  hb.calculateIndexSize(hnswIndex),
    }, nil
}

// IVF 索引构建器
type IVFBuilder struct {
    // 构建参数
    nlist       int
    metricType  MetricType
    quantizer   *Quantizer
}

func (ib *IVFBuilder) Build(segment *Segment, params map[string]interface{}) (*Index, error) {
    // 1. 提取向量数据
    vectors, err := ib.extractVectors(segment)
    if err != nil {
        return nil, err
    }

    // 2. 构建参数
    if nlist, ok := params["nlist"].(int); ok {
        ib.nlist = nlist
    }

    // 3. K-means 聚类
    centroids, assignments := ib.kmeansClustering(vectors, ib.nlist)

    // 4. 构建 IVF 索引
    ivfIndex := &IVFIndex{
        Nlist:      ib.nlist,
        Centroids:  centroids,
        Assignments: assignments,
        Vectors:    vectors,
        Quantizer:  ib.quantizer,
    }

    // 5. 返回索引
    return &Index{
        Type:  IndexType_IVF,
        Data:  ivfIndex,
        Size:  ib.calculateIndexSize(ivfIndex),
    }, nil
}

// 索引加载器
type IndexLoader struct {
    // 存储接口
    storage storage.Storage

    // 内存管理器
    memoryMgr *memory.Manager

    // 索引工厂
    factory *IndexFactory
}

// 加载索引
func (il *IndexLoader) LoadIndex(indexInfo *IndexInfo) (*Index, error) {
    // 1. 检查是否已经加载
    if loadedIndex, ok := il.indices.Load(indexInfo.ID); ok {
        return loadedIndex.(*Index), nil
    }

    // 2. 从存储加载索引数据
    indexData, err := il.storage.LoadIndex(indexInfo.ID)
    if err != nil {
        return nil, fmt.Errorf("failed to load index data: %v", err)
    }

    // 3. 检查内存资源
    if !il.memoryMgr.CheckMemory(indexInfo.Size) {
        return nil, fmt.Errorf("insufficient memory for index %d", indexInfo.ID)
    }

    // 4. 分配内存
    memoryHandle, err := il.memoryMgr.Allocate(indexInfo.Size)
    if err != nil {
        return nil, fmt.Errorf("failed to allocate memory: %v", err)
    }

    // 5. 反序列化索引
    index, err := il.factory.Deserialize(indexInfo.Type, indexData, memoryHandle)
    if err != nil {
        il.memoryMgr.Free(memoryHandle)
        return nil, fmt.Errorf("failed to deserialize index: %v", err)
    }

    // 6. 缓存索引
    il.indices.Store(indexInfo.ID, index)

    return index, nil
}
```

### 缓存管理系统

```go
// 缓存管理器
type CacheManager struct {
    // L1 缓存（段缓存）
    segmentCache *lru.Cache

    // L2 缓存（查询缓存）
    queryCache *QueryCache

    // L3 缓存（向量缓存）
    vectorCache *VectorCache

    // 缓存策略
    policy CachePolicy
}

// 段缓存
type SegmentCache struct {
    // LRU 缓存
    cache *lru.Cache

    // 预取器
    prefetcher *SegmentPrefetcher

    // 淘汰策略
    evictionPolicy SegmentEvictionPolicy
}

// 查询缓存
type QueryCache struct {
    // LRU 缓存
    cache *lru.Cache

    // 查询标准化器
    normalizer *QueryNormalizer

    // 缓存键生成器
    keyGenerator *CacheKeyGenerator
}

// 向量缓存
type VectorCache struct {
    // 缓存分区
    partitions []*VectorPartition

    // 替换策略
    replacementPolicy VectorReplacementPolicy
}

// 向量分区
type VectorPartition struct {
    // 分区 ID
    ID int

    // 缓存空间
    vectors [][]float32

    // LRU 链表
    lruList *list.List

    // 索引
    index map[int]*list.Element
}

// 缓存查询
func (qc *QueryCache) Get(req *SearchRequest) ([]SearchResult, bool) {
    // 1. 生成缓存键
    cacheKey := qc.keyGenerator.Generate(req)

    // 2. 检查缓存
    if cached, found := qc.cache.Get(cacheKey); found {
        return cached.([]SearchResult), true
    }

    return nil, false
}

// 缓存查询结果
func (qc *QueryCache) Put(req *SearchRequest, results []SearchResult) {
    // 3. 生成缓存键
    cacheKey := qc.keyGenerator.Generate(req)

    // 4. 计算缓存成本
    cost := qc.calculateCacheCost(results)

    // 5. 检查是否值得缓存
    if qc.shouldCache(req, cost) {
        qc.cache.Put(cacheKey, results)
    }
}

// 向量缓存查询
func (vc *VectorCache) Get(vectorID int) ([]float32, bool) {
    // 6. 选择分区
    partition := vc.selectPartition(vectorID)

    // 7. 检查缓存
    if vector, found := partition.vectors[vectorID]; found {
        // 8. 更新 LRU
        partition.updateLRU(vectorID)
        return vector, true
    }

    return nil, false
}

// 向量缓存存储
func (vc *VectorCache) Put(vectorID int, vector []float32) {
    // 9. 选择分区
    partition := vc.selectPartition(vectorID)

    // 10. 检查空间
    if len(partition.vectors) >= vc.getMaxPartitionSize() {
        // 11. 淘汰向量
        vc.evictVector(partition)
    }

    // 12. 存储向量
    partition.vectors[vectorID] = vector
    partition.updateLRU(vectorID)
}

### 内存管理系统

```go
// 内存管理器
type MemoryManager struct {
    // 总内存限制
    totalLimit int64

    // 已使用内存
    usedMemory int64

    // 内存分配器
    allocator MemoryAllocator

    // 内存监控
    monitor *MemoryMonitor

    // 垃圾回收器
    gc *MemoryGC

    // 保护机制
    mu sync.RWMutex
}

// 内存分配器接口
type MemoryAllocator interface {
    Allocate(size int64) (MemoryHandle, error)
    Free(handle MemoryHandle) error
    GetStatistics() MemoryStatistics
}

// 内存句柄
type MemoryHandle struct {
    ID     int64
    Size   int64
    Data   unsafe.Pointer
    Type   MemoryType
}

// 内存类型
type MemoryType int

const (
    MemoryType_System MemoryType = iota
    MemoryType_MMap
    MemoryType_GPU
)

// 分配内存
func (mm *MemoryManager) Allocate(size int64) (MemoryHandle, error) {
    mm.mu.Lock()
    defer mm.mu.Unlock()

    // 1. 检查内存限制
    if mm.usedMemory+size > mm.totalLimit {
        // 2. 尝试垃圾回收
        if err := mm.gc.Collect(size); err != nil {
            return MemoryHandle{}, fmt.Errorf("insufficient memory: requested %d, available %d", size, mm.totalLimit-mm.usedMemory)
        }
    }

    // 3. 分配内存
    handle, err := mm.allocator.Allocate(size)
    if err != nil {
        return MemoryHandle{}, err
    }

    // 4. 更新使用统计
    mm.usedMemory += size

    return handle, nil
}

// 释放内存
func (mm *MemoryManager) Free(handle MemoryHandle) error {
    mm.mu.Lock()
    defer mm.mu.Unlock()

    // 5. 释放内存
    if err := mm.allocator.Free(handle); err != nil {
        return err
    }

    // 6. 更新使用统计
    mm.usedMemory -= handle.Size

    return nil
}

// 内存垃圾回收器
type MemoryGC struct {
    // 回收策略
    policy GCPolicy

    // 统计信息
    statistics *GCStatistics
}

// 回收策略
type GCPolicy interface {
    ShouldCollect(handle MemoryHandle) bool
    GetPriority(handle MemoryHandle) int
}

// 垃圾回收
func (gc *MemoryGC) Collect(required int64) error {
    // 7. 获取所有内存句柄
    handles := gc.getAllHandles()

    // 8. 按优先级排序
    sortedHandles := gc.sortByPriority(handles)

    // 9. 逐步回收内存
    freed := int64(0)
    for _, handle := range sortedHandles {
        if freed >= required {
            break
        }

        if gc.policy.ShouldCollect(handle) {
            if err := gc.freeHandle(handle); err != nil {
                log.Warnf("Failed to free handle %d: %v", handle.ID, err)
                continue
            }
            freed += handle.Size
        }
    }

    if freed < required {
        return fmt.Errorf("insufficient memory after GC: freed %d, required %d", freed, required)
    }

    return nil
}
```

## DataNode：数据写入的处理中心

### DataNode 的核心职责

DataNode 是数据写入的主要执行单元，负责：

1. **数据写入**：处理数据的插入、更新和删除操作
2. **数据刷新**：将内存数据刷新到持久存储
3. **数据压缩**：执行数据压缩和合并操作
4. **数据复制**：实现数据的多副本管理
5. **WAL 管理**：管理预写日志确保数据持久性

### DataNode 的架构设计

```go
// DataNode 核心结构
type DataNode struct {
    // 基础设施
    ctx    context.Context
    config *Config

    // 核心组件
    dataWriter    *data.Writer        // 数据写入器
    dataFlusher   *data.Flusher       // 数据刷新器
    walManager    *wal.Manager        // WAL 管理器
    compactor     *compaction.Compactor // 压缩器
    replicator    *replication.Manager // 复制管理器

    // 存储组件
    storage       storage.Storage      // 存储接口
    metaStore     meta.MetaStore       // 元数据存储

    // 缓存组件
    cacheMgr      *cache.Manager       // 缓存管理器
    bufferPool    *buffer.Pool         // 缓冲池

    // 监控和指标
    metrics       *Metrics             // 指标收集器
    healthChecker *health.Checker      // 健康检查器

    // 服务器
    server        *grpc.Server
}

// DataNode 初始化
func (dn *DataNode) Init() error {
    // 1. 初始化存储组件
    dn.storage = storage.NewStorage(dn.config.Storage)
    dn.metaStore = meta.NewMetaStore(dn.config.MetaStore)

    // 2. 初始化核心组件
    dn.dataWriter = data.NewWriter(dn.config.Data)
    dn.dataFlusher = data.NewFlusher(dn.config.Data)
    dn.walManager = wal.NewManager(dn.config.WAL)
    dn.compactor = compaction.NewCompactor(dn.config.Compaction)
    dn.replicator = replication.NewManager(dn.config.Replication)

    // 3. 初始化缓存组件
    dn.cacheMgr = cache.NewManager(dn.config.Cache)
    dn.bufferPool = buffer.NewPool(dn.config.Buffer)

    // 4. 初始化监控组件
    dn.metrics = metrics.NewCollector(dn.config.Metrics)
    dn.healthChecker = health.NewChecker(dn.config.Health)

    // 5. 启动后台任务
    dn.startBackgroundTasks()

    return nil
}
```

### 数据写入器

```go
// 数据写入器
type DataWriter struct {
    // 写入队列
    writeQueue *WriteQueue

    // 批量处理器
    batchProcessor *BatchProcessor

    // WAL 写入器
    walWriter *wal.Writer

    // 内存缓冲区
    memoryBuffer *MemoryBuffer

    // 写入策略
    strategy WriteStrategy
}

// 写入策略
type WriteStrategy interface {
    ShouldBatch(requests []*WriteRequest) bool
    GetBatchSize() int
    GetBatchTimeout() time.Duration
}

// 写入请求
type WriteRequest struct {
    CollectionID int64
    PartitionID  int64
    RowData      map[string]interface{}
    Timestamp    int64
    RequestID    string
}

// 写入响应
type WriteResponse struct {
    Success      bool
    ID           int64
    Timestamp    int64
    ErrorMessage string
}

// 数据写入
func (dw *DataWriter) Write(req *WriteRequest) (*WriteResponse, error) {
    // 1. 验证请求
    if err := dw.validateRequest(req); err != nil {
        return &WriteResponse{
            Success:      false,
            ErrorMessage: err.Error(),
        }, err
    }

    // 2. 生成 ID 和时间戳
    if req.ID == 0 {
        req.ID = dw.generateID()
    }
    if req.Timestamp == 0 {
        req.Timestamp = dw.generateTimestamp()
    }

    // 3. 写入 WAL
    if err := dw.walWriter.Write(req); err != nil {
        return &WriteResponse{
            Success:      false,
            ErrorMessage: fmt.Sprintf("failed to write WAL: %v", err),
        }, err
    }

    // 4. 写入内存缓冲区
    if err := dw.memoryBuffer.Write(req); err != nil {
        return &WriteResponse{
            Success:      false,
            ErrorMessage: fmt.Sprintf("failed to write memory buffer: %v", err),
        }, err
    }

    // 5. 更新元数据
    if err := dw.updateMetadata(req); err != nil {
        log.Warnf("Failed to update metadata: %v", err)
    }

    // 6. 返回成功响应
    return &WriteResponse{
        Success:   true,
        ID:        req.ID,
        Timestamp: req.Timestamp,
    }, nil
}

// 批量写入处理器
type BatchProcessor struct {
    // 批量队列
    queue chan []*WriteRequest

    // 批量大小
    batchSize int

    // 批量超时
    batchTimeout time.Duration

    // 处理函数
    processFunc func([]*WriteRequest) error
}

// 启动批量处理
func (bp *BatchProcessor) Start() {
    ticker := time.NewTicker(bp.batchTimeout)
    defer ticker.Stop()

    batch := make([]*WriteRequest, 0, bp.batchSize)

    for {
        select {
        case requests := <-bp.queue:
            // 7. 添加到批次
            batch = append(batch, requests...)

            // 8. 检查批次大小
            if len(batch) >= bp.batchSize {
                if err := bp.processBatch(batch); err != nil {
                    log.Errorf("Failed to process batch: %v", err)
                }
                batch = batch[:0]
            }

        case <-ticker.C:
            // 9. 处理超时的批次
            if len(batch) > 0 {
                if err := bp.processBatch(batch); err != nil {
                    log.Errorf("Failed to process batch: %v", err)
                }
                batch = batch[:0]
            }
        }
    }
}

// 处理批次
func (bp *BatchProcessor) processBatch(requests []*WriteRequest) error {
    if len(requests) == 0 {
        return nil
    }

    start := time.Now()
    defer func() {
        log.Debugf("Processed batch of %d requests in %v", len(requests), time.Since(start))
    }()

    // 10. 处理批量请求
    return bp.processFunc(requests)
}
```

### WAL 管理器

```go
// WAL 管理器
type WALManager struct {
    // WAL 存储接口
    storage WALStorage

    // WAL 写入器
    writer *WALWriter

    // WAL 读取器
    reader *WALReader

    // 检查点管理器
    checkpointManager *CheckpointManager

    // 恢复管理器
    recoveryManager *RecoveryManager
}

// WAL 存储接口
type WALStorage interface {
    Append(entry *WALEntry) error
    Read(offset int64, limit int64) ([]*WALEntry, error)
    Truncate(offset int64) error
    GetSize() int64
}

// WAL 条目
type WALEntry struct {
    ID         int64
    Type       WALType
    Data       []byte
    Timestamp  int64
    Checksum   uint32
}

// WAL 类型
type WALType int

const (
    WALType_Data WALType = iota
    WALType_Metadata
    WALType_Checkpoint
)

// WAL 写入器
type WALWriter struct {
    // 存储接口
    storage WALStorage

    // 缓冲区
    buffer []byte

    // 压缩器
    compressor *Compressor

    // 批量写入器
    batchWriter *BatchWriter
}

// 写入 WAL 条目
func (ww *WALWriter) Write(entry *WALEntry) error {
    // 1. 序列化条目
    data, err := ww.serializeEntry(entry)
    if err != nil {
        return fmt.Errorf("failed to serialize entry: %v", err)
    }

    // 2. 计算校验和
    entry.Checksum = ww.calculateChecksum(data)

    // 3. 压缩数据
    if ww.compressor != nil {
        compressedData, err := ww.compressor.Compress(data)
        if err != nil {
            return fmt.Errorf("failed to compress data: %v", err)
        }
        data = compressedData
    }

    // 4. 添加到缓冲区
    ww.buffer = append(ww.buffer, data...)

    // 5. 检查是否需要刷新
    if len(ww.buffer) >= ww.getBufferSize() {
        if err := ww.flush(); err != nil {
            return fmt.Errorf("failed to flush buffer: %v", err)
        }
    }

    return nil
}

// 刷新缓冲区
func (ww *WALWriter) flush() error {
    if len(ww.buffer) == 0 {
        return nil
    }

    // 6. 写入存储
    if err := ww.storage.Append(&WALEntry{
        ID:        ww.generateID(),
        Type:      WALType_Data,
        Data:      ww.buffer,
        Timestamp: time.Now().Unix(),
    }); err != nil {
        return err
    }

    // 7. 清空缓冲区
    ww.buffer = ww.buffer[:0]

    return nil
}

// WAL 读取器
type WALReader struct {
    // 存储接口
    storage WALStorage

    // 当前偏移量
    offset int64

    // 解压器
    decompressor *Decompressor
}

// 读取 WAL 条目
func (wr *WALReader) Read(count int64) ([]*WALEntry, error) {
    // 8. 从存储读取
    entries, err := wr.storage.Read(wr.offset, count)
    if err != nil {
        return nil, fmt.Errorf("failed to read from storage: %v", err)
    }

    // 9. 处理每个条目
    var result []*WALEntry
    for _, entry := range entries {
        // 10. 验证校验和
        if !wr.validateChecksum(entry) {
            log.Warnf("Invalid checksum for entry %d", entry.ID)
            continue
        }

        // 11. 解压数据
        if wr.decompressor != nil {
            decompressedData, err := wr.decompressor.Decompress(entry.Data)
            if err != nil {
                log.Warnf("Failed to decompress entry %d: %v", entry.ID, err)
                continue
            }
            entry.Data = decompressedData
        }

        // 12. 反序列化条目
        if err := wr.deserializeEntry(entry); err != nil {
            log.Warnf("Failed to deserialize entry %d: %v", entry.ID, err)
            continue
        }

        result = append(result, entry)
    }

    // 13. 更新偏移量
    if len(result) > 0 {
        wr.offset += int64(len(result))
    }

    return result, nil
}
```

### 数据刷新器

```go
// 数据刷新器
type DataFlusher struct {
    // 刷新队列
    flushQueue *FlushQueue

    // 刷新策略
    strategy FlushStrategy

    // 刷新执行器
    executor *FlushExecutor

    // 监控指标
    metrics *FlushMetrics
}

// 刷新策略
type FlushStrategy interface {
    ShouldFlush(buffer *MemoryBuffer) bool
    GetFlushPriority(buffer *MemoryBuffer) int
    GetFlushTarget(buffer *MemoryBuffer) FlushTarget
}

// 刷新目标
type FlushTarget struct {
    Type      FlushTargetType
    Location  string
    Options   map[string]interface{}
}

// 刷新目标类型
type FlushTargetType int

const (
    FlushTargetType_ObjectStorage FlushTargetType = iota
    FlushTargetType_LocalStorage
    FlushTargetType_Distributed
)

// 内存缓冲区
type MemoryBuffer struct {
    // 数据段
    segments map[int64]*SegmentBuffer

    // 元数据
    metadata *BufferMetadata

    // 统计信息
    statistics *BufferStatistics

    // 保护机制
    mu sync.RWMutex
}

// 段缓冲区
type SegmentBuffer struct {
    ID        int64
    Data      []byte
    Index     []byte
    Metadata  *SegmentMetadata
    Size      int64
    RowCount  int64
    CreatedAt int64
    UpdatedAt int64
}

// 刷新执行器
type FlushExecutor struct {
    // 存储接口
    storage storage.Storage

    // 压缩器
    compressor *Compressor

    // 校验和计算器
    checksumCalculator *ChecksumCalculator
}

// 执行刷新
func (fe *FlushExecutor) ExecuteFlush(buffer *MemoryBuffer, target FlushTarget) error {
    // 1. 验证缓冲区
    if err := fe.validateBuffer(buffer); err != nil {
        return fmt.Errorf("invalid buffer: %v", err)
    }

    // 2. 准备刷新数据
    flushData := fe.prepareFlushData(buffer)

    // 3. 压缩数据
    compressedData, err := fe.compressor.Compress(flushData)
    if err != nil {
        return fmt.Errorf("failed to compress data: %v", err)
    }

    // 4. 计算校验和
    checksum := fe.checksumCalculator.Calculate(compressedData)

    // 5. 创建刷新文件
    flushFile := &FlushFile{
        ID:         fe.generateFileID(),
        Data:       compressedData,
        Checksum:   checksum,
        Target:     target,
        CreatedAt:  time.Now().Unix(),
        Size:       int64(len(compressedData)),
    }

    // 6. 写入存储
    if err := fe.storage.WriteFlushFile(flushFile); err != nil {
        return fmt.Errorf("failed to write flush file: %v", err)
    }

    // 7. 更新元数据
    if err := fe.updateMetadata(buffer, flushFile); err != nil {
        log.Warnf("Failed to update metadata: %v", err)
    }

    return nil
}

// 刷新文件
type FlushFile struct {
    ID         int64
    Data       []byte
    Checksum   uint32
    Target     FlushTarget
    CreatedAt  int64
    Size       int64
    Metadata   map[string]interface{}
}
```

### 数据压缩器

```go
// 数据压缩器
type Compactor struct {
    // 压缩队列
    compactionQueue *CompactionQueue

    // 压缩策略
    strategy CompactionStrategy

    // 压缩执行器
    executor *CompactionExecutor

    // 监控指标
    metrics *CompactionMetrics
}

// 压缩策略
type CompactionStrategy interface {
    ShouldCompact(segments []*Segment) bool
    GetCompactionPriority(segments []*Segment) int
    SelectCompactionSegments(segments []*Segment) []*Segment
}

// 压缩任务
type CompactionTask struct {
    ID          int64
    SegmentIDs  []int64
    TargetSegment *Segment
    Priority    CompactionPriority
    State       CompactionState
    CreatedAt   int64
    StartedAt   int64
    CompletedAt int64
    ErrorMessage string
}

// 压缩优先级
type CompactionPriority int

const (
    CompactionPriority_Low CompactionPriority = iota
    CompactionPriority_Medium
    CompactionPriority_High
    CompactionPriority_Critical
)

// 压缩执行器
type CompactionExecutor struct {
    // 存储接口
    storage storage.Storage

    // 合并器
    merger *SegmentMerger

    // 索引构建器
    indexBuilder *IndexBuilder
}

// 执行压缩
func (ce *Compactor) ExecuteCompaction(task *CompactionTask) error {
    // 1. 更新任务状态
    task.State = CompactionState_Running
    task.StartedAt = time.Now().Unix()

    // 2. 获取压缩段
    segments := ce.getSegments(task.SegmentIDs)
    if len(segments) == 0 {
        return fmt.Errorf("no segments found for compaction")
    }

    // 3. 合并段数据
    mergedSegment, err := ce.merger.MergeSegments(segments)
    if err != nil {
        task.State = CompactionState_Failed
        task.ErrorMessage = fmt.Sprintf("failed to merge segments: %v", err)
        return err
    }

    // 4. 构建索引
    if mergedSegment.IndexInfo != nil {
        if err := ce.indexBuilder.BuildIndex(mergedSegment); err != nil {
            task.State = CompactionState_Failed
            task.ErrorMessage = fmt.Sprintf("failed to build index: %v", err)
            return err
        }
    }

    // 5. 写入压缩后的段
    if err := ce.storage.WriteSegment(mergedSegment); err != nil {
        task.State = CompactionState_Failed
        task.ErrorMessage = fmt.Sprintf("failed to write segment: %v", err)
        return err
    }

    // 6. 更新元数据
    if err := ce.updateMetadata(task, mergedSegment); err != nil {
        task.State = CompactionState_Failed
        task.ErrorMessage = fmt.Sprintf("failed to update metadata: %v", err)
        return err
    }

    // 7. 清理原始段
    if err := ce.cleanupOriginalSegments(task.SegmentIDs); err != nil {
        log.Warnf("Failed to cleanup original segments: %v", err)
    }

    // 8. 更新任务状态
    task.State = CompactionState_Completed
    task.CompletedAt = time.Now().Unix()
    task.TargetSegment = mergedSegment

    return nil
}

// 段合并器
type SegmentMerger struct {
    // 合并策略
    strategy MergeStrategy

    // 过滤器
    filter *DataFilter

    // 排序器
    sorter *DataSorter
}

// 合并段
func (sm *SegmentMerger) MergeSegments(segments []*Segment) (*Segment, error) {
    // 9. 验证段
    if err := sm.validateSegments(segments); err != nil {
        return nil, err
    }

    // 10. 过滤重复数据
    filteredData, err := sm.filter.FilterDuplicates(segments)
    if err != nil {
        return nil, fmt.Errorf("failed to filter duplicates: %v", err)
    }

    // 11. 排序数据
    sortedData, err := sm.sorter.SortData(filteredData)
    if err != nil {
        return nil, fmt.Errorf("failed to sort data: %v", err)
    }

    // 12. 创建新段
    newSegment := &Segment{
        ID:          sm.generateSegmentID(),
        CollectionID: segments[0].CollectionID,
        PartitionID:  segments[0].PartitionID,
        State:       SegmentState_Compacted,
        Size:        int64(len(sortedData)),
        RowCount:    sm.calculateRowCount(sortedData),
        CreatedAt:   time.Now().Unix(),
    }

    // 13. 设置段数据
    newSegment.Data = sortedData

    return newSegment, nil
}
```

### 数据复制管理器

```go
// 复制管理器
type ReplicationManager struct {
    // 复制组
    replicationGroups map[int64]*ReplicationGroup

    // 复制策略
    strategy ReplicationStrategy

    // 复制执行器
    executor *ReplicationExecutor

    // 监控指标
    metrics *ReplicationMetrics
}

// 复制组
type ReplicationGroup struct {
    ID          int64
    LeaderID    int64
    FollowerIDs []int64
    State       ReplicationState
    Config      *ReplicationConfig
    Statistics  *ReplicationStatistics
}

// 复制状态
type ReplicationState int

const (
    ReplicationState_Creating ReplicationState = iota
    ReplicationState_Active
    ReplicationState_Failover
    ReplicationState_Recovering
)

// 复制策略
type ReplicationStrategy interface {
    ShouldReplicate(data []byte) bool
    GetReplicationTargets(group *ReplicationGroup) []int64
    HandleFailure(group *ReplicationGroup, failedNode int64) error
}

// 复制执行器
type ReplicationExecutor struct {
    // 网络接口
    network NetworkInterface

    // 复制队列
    queue *ReplicationQueue

    // 重试机制
    retryManager *RetryManager
}

// 执行复制
func (re *ReplicationExecutor) ExecuteReplication(group *ReplicationGroup, data []byte) error {
    // 1. 验证复制组
    if group.State != ReplicationState_Active {
        return fmt.Errorf("replication group is not active")
    }

    // 2. 获取复制目标
    targets := re.strategy.GetReplicationTargets(group)
    if len(targets) == 0 {
        return fmt.Errorf("no replication targets available")
    }

    // 3. 创建复制任务
    task := &ReplicationTask{
        ID:        re.generateTaskID(),
        GroupID:   group.ID,
        Data:      data,
        Targets:   targets,
        CreatedAt: time.Now().Unix(),
    }

    // 4. 并行复制
    results := make(chan *ReplicationResult, len(targets))
    var wg sync.WaitGroup

    for _, target := range targets {
        wg.Add(1)
        go func(nodeID int64) {
            defer wg.Done()

            result := re.replicateToNode(task, nodeID)
            results <- result
        }(target)
    }

    // 5. 等待复制完成
    go func() {
        wg.Wait()
        close(results)
    }()

    // 6. 收集结果
    var successCount int
    var errors []error

    for result := range results {
        if result.Success {
            successCount++
        } else {
            errors = append(errors, result.Error)
        }
    }

    // 7. 检查复制成功数量
    requiredSuccesses := re.getRequiredSuccesses(len(targets))
    if successCount < requiredSuccesses {
        return fmt.Errorf("replication failed: %d/%d nodes succeeded", successCount, len(targets))
    }

    return nil
}

// 复制到节点
func (re *ReplicationExecutor) replicateToNode(task *ReplicationTask, nodeID int64) *ReplicationResult {
    // 8. 重试机制
    var lastError error
    maxRetries := re.retryManager.GetMaxRetries()

    for attempt := 0; attempt < maxRetries; attempt++ {
        // 9. 发送复制数据
        err := re.network.SendToNode(nodeID, &ReplicationRequest{
            TaskID:   task.ID,
            GroupID:  task.GroupID,
            Data:     task.Data,
            Attempt:  attempt,
        })

        if err == nil {
            return &ReplicationResult{
                Success: true,
                NodeID:  nodeID,
                Attempt: attempt,
            }
        }

        lastError = err
        re.retryManager.WaitBeforeRetry(attempt)
    }

    return &ReplicationResult{
        Success: false,
        NodeID:  nodeID,
        Error:   lastError,
        Attempt: maxRetries - 1,
    }
}

// 复制结果
type ReplicationResult struct {
    Success  bool
    NodeID   int64
    Error    error
    Attempt  int
    Duration time.Duration
}
```

## 计算层性能优化

### 并行查询优化

```go
// 并行查询执行器
type ParallelQueryExecutor struct {
    // 查询分发器
    dispatcher *QueryDispatcher

    // 结果聚合器
    aggregator *ResultAggregator

    // 并行控制
    parallelControl *ParallelControl
}

// 查询分发器
type QueryDispatcher struct {
    // 节点选择器
    nodeSelector *NodeSelector

    // 负载均衡器
    loadBalancer *LoadBalancer

    // 重试机制
    retryManager *RetryManager
}

// 分发查询
func (qd *QueryDispatcher) DispatchQuery(plan *QueryPlan) (*QueryResult, error) {
    // 1. 选择执行节点
    nodes := qd.nodeSelector.SelectNodes(plan)

    // 2. 分发查询任务
    results := make(chan *QueryResult, len(nodes))
    errors := make(chan error, len(nodes))

    var wg sync.WaitGroup
    for _, node := range nodes {
        wg.Add(1)
        go func(n *QueryNode) {
            defer wg.Done()

            result, err := qd.executeOnNode(n, plan)
            if err != nil {
                errors <- err
                return
            }
            results <- result
        }(node)
    }

    // 3. 等待所有查询完成
    go func() {
        wg.Wait()
        close(results)
        close(errors)
    }()

    // 4. 收集结果
    var allResults []QueryResult
    for result := range results {
        allResults = append(allResults, *result)
    }

    // 5. 检查错误
    if err := <-errors; err != nil {
        return nil, err
    }

    // 6. 聚合结果
    finalResult := qd.aggregator.AggregateResults(allResults)
    return finalResult, nil
}
```

### 内存优化

```go
// 内存优化器
type MemoryOptimizer struct {
    // 内存分析器
    analyzer *MemoryAnalyzer

    // 优化策略
    strategies []MemoryOptimizationStrategy

    // 执行器
    executor *OptimizationExecutor
}

// 内存分析器
type MemoryAnalyzer struct {
    // 内存监控器
    monitor *MemoryMonitor

    // 统计分析器
    statistics *StatisticsAnalyzer
}

// 分析内存使用
func (ma *MemoryAnalyzer) AnalyzeMemoryUsage() (*MemoryAnalysis, error) {
    // 1. 收集内存使用数据
    usageData := ma.monitor.CollectMemoryUsage()

    // 2. 分析内存分布
    distribution := ma.analyzeDistribution(usageData)

    // 3. 识别内存热点
    hotspots := ma.identifyHotspots(usageData)

    // 4. 计算优化潜力
    potential := ma.calculateOptimizationPotential(usageData)

    // 5. 生成优化建议
    recommendations := ma.generateRecommendations(usageData, hotspots, potential)

    return &MemoryAnalysis{
        Distribution:    distribution,
        Hotspots:        hotspots,
        Potential:       potential,
        Recommendations: recommendations,
        Timestamp:       time.Now().Unix(),
    }, nil
}
```

## 总结

Milvus 的计算层通过 QueryNode 和 DataNode 的紧密协作，实现了高性能的向量搜索和数据处理能力：

1. **QueryNode**：专注于查询执行，通过段管理、索引优化、缓存策略等机制提供毫秒级的向量搜索能力
2. **DataNode**：专注于数据处理，通过 WAL 机制、批量处理、压缩优化等确保数据的高效写入和持久化
3. **并行处理**：通过并行查询、批量处理、异步刷新等机制充分利用系统资源
4. **内存优化**：通过多级缓存、内存池、垃圾回收等机制优化内存使用效率
5. **容错能力**：通过复制机制、故障恢复、重试策略等确保系统的高可用性

计算层的设计充分体现了现代分布式系统的架构思想，通过精心设计的组件和算法，确保了 Milvus 能够在海量数据和复杂查询场景下保持优秀的性能表现。

在下一篇文章中，我们将深入探讨 Milvus 的存储层设计，了解它是如何管理海量向量数据的存储和检索的。

---

**作者简介**：本文作者专注于高性能计算和分布式系统设计，拥有丰富的向量数据库和搜索引擎开发经验。

**相关链接**：
- [Milvus 计算层文档](https://milvus.io/docs/compute_layer.md)
- [向量搜索算法研究](https://arxiv.org/list/cs.IR/recent)
- [分布式计算最佳实践](https://en.wikipedia.org/wiki/Distributed_computing)