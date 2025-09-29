# Milvus 混合搜索：向量与结构化数据的完美融合

## 前言

在实际应用场景中，单纯依靠向量相似度搜索往往无法满足复杂的业务需求。我们需要结合向量搜索和传统结构化查询，实现更精确、更灵活的搜索能力。Milvus 的混合搜索功能正是为了解决这一需求而设计的，它将向量搜索与标量过滤、元数据查询等功能完美融合，为用户提供强大的复合查询能力。

## 混合搜索概览

### 什么是混合搜索

混合搜索（Hybrid Search）是指在向量搜索的基础上，结合结构化数据查询条件，实现多维度、多条件的复合搜索。它允许用户在寻找语义相似内容的同时，对数据的属性进行精确过滤。

```go
// 混合搜索查询结构
type HybridSearchRequest struct {
    // 向量查询条件
    VectorQuery *VectorQuery

    // 标量过滤条件
    ScalarFilters []*ScalarFilter

    // 元数据过滤条件
    MetadataFilters []*MetadataFilter

    // 时间范围过滤
    TimeRange *TimeRangeFilter

    // 地理位置过滤
    GeoFilters []*GeoFilter

    // 排序条件
    SortConditions []*SortCondition

    // 分页参数
    Pagination *PaginationParams

    // 聚合参数
    Aggregations []*AggregationParams
}

// 向量查询条件
type VectorQuery struct {
    // 查询向量
    Vectors [][]float32

    // 搜索参数
    SearchParams *SearchParams

    // 距离类型
    MetricType string

    // 期望返回的 TopK
    TopK int

    // 最小相似度阈值
    MinScore float32
}

// 标量过滤条件
type ScalarFilter struct {
    // 字段名
    FieldName string

    // 操作符
    Operator string

    // 值
    Value interface{}

    // 是否取反
    Negated bool
}
```

### 混合搜索的应用场景

```go
// 混合搜索应用场景枚举
type HybridSearchScenario string

const (
    // 电商商品搜索
    EcommerceProductSearch HybridSearchScenario = "ecommerce_product_search"

    // 文档智能检索
    DocumentRetrieval HybridSearchScenario = "document_retrieval"

    // 图像相似搜索
    ImageSimilaritySearch HybridSearchScenario = "image_similarity_search"

    // 推荐系统
    RecommendationSystem HybridSearchScenario = "recommendation_system"

    // 视频内容搜索
    VideoContentSearch HybridSearchScenario = "video_content_search"

    // 用户画像搜索
    UserProfileSearch HybridSearchScenario = "user_profile_search"
)

// 场景特征描述
type ScenarioFeatures struct {
    // 向量维度
    VectorDimension int

    // 常用过滤字段
    CommonFilterFields []string

    // 排序需求
    SortingRequirements []string

    // 聚合需求
    AggregationRequirements []string

    // 性能要求
    PerformanceRequirements *PerformanceRequirements
}
```

## 混合搜索实现原理

### 1. 查询解析与优化

```go
// 混合搜索查询解析器
type HybridSearchParser struct {
    // 语法分析器
    syntaxParser *SyntaxParser

    // 语义分析器
    semanticAnalyzer *SemanticAnalyzer

    // 查询优化器
    queryOptimizer *QueryOptimizer

    // 执行计划生成器
    planGenerator *ExecutionPlanGenerator
}

// 查询解析流程
func (hsp *HybridSearchParser) ParseQuery(query *HybridSearchRequest) (*OptimizedQueryPlan, error) {
    // 1. 语法分析
    parsedQuery, err := hsp.syntaxParser.Parse(query)
    if err != nil {
        return nil, fmt.Errorf("syntax parsing failed: %w", err)
    }

    // 2. 语义分析
    analyzedQuery, err := hsp.semanticAnalyzer.Analyze(parsedQuery)
    if err != nil {
        return nil, fmt.Errorf("semantic analysis failed: %w", err)
    }

    // 3. 查询优化
    optimizedQuery, err := hsp.queryOptimizer.Optimize(analyzedQuery)
    if err != nil {
        return nil, fmt.Errorf("query optimization failed: %w", err)
    }

    // 4. 生成执行计划
    executionPlan, err := hsp.planGenerator.Generate(optimizedQuery)
    if err != nil {
        return nil, fmt.Errorf("execution plan generation failed: %w", err)
    }

    return executionPlan, nil
}

// 语法分析器
type SyntaxParser struct {
    // 词法分析器
    lexer *Lexer

    // 语法规则
    grammar *Grammar

    // 错误处理器
    errorHandler *ErrorHandler
}

// 语法分析
func (sp *SyntaxParser) Parse(query *HybridSearchRequest) (*ParsedQuery, error) {
    parsedQuery := &ParsedQuery{
        VectorClauses:    make([]*VectorClause, 0),
        FilterClauses:    make([]*FilterClause, 0),
        SortClauses:      make([]*SortClause, 0),
        AggregationClauses: make([]*AggregationClause, 0),
    }

    // 解析向量查询部分
    if query.VectorQuery != nil {
        vectorClause, err := sp.parseVectorQuery(query.VectorQuery)
        if err != nil {
            return nil, err
        }
        parsedQuery.VectorClauses = append(parsedQuery.VectorClauses, vectorClause)
    }

    // 解析标量过滤条件
    for _, filter := range query.ScalarFilters {
        filterClause, err := sp.parseScalarFilter(filter)
        if err != nil {
            return nil, err
        }
        parsedQuery.FilterClauses = append(parsedQuery.FilterClauses, filterClause)
    }

    // 解析排序条件
    for _, sort := range query.SortConditions {
        sortClause, err := sp.parseSortCondition(sort)
        if err != nil {
            return nil, err
        }
        parsedQuery.SortClauses = append(parsedQuery.SortClauses, sortClause)
    }

    // 解析聚合条件
    for _, agg := range query.Aggregations {
        aggClause, err := sp.parseAggregation(agg)
        if err != nil {
            return nil, err
        }
        parsedQuery.AggregationClauses = append(parsedQuery.AggregationClauses, aggClause)
    }

    return parsedQuery, nil
}
```

### 2. 执行计划生成

```go
// 执行计划生成器
type ExecutionPlanGenerator struct {
    // 计划节点工厂
    nodeFactory *PlanNodeFactory

    // 成本估算器
    costEstimator *CostEstimator

    // 统计信息管理器
    statsManager *StatisticsManager
}

// 执行计划节点
type PlanNode interface {
    // 执行节点
    Execute(ctx context.Context, session *Session) (*ResultSet, error)

    // 估算成本
    EstimateCost() *Cost

    // 获取节点类型
    GetType() NodeType

    // 获取子节点
    GetChildren() []PlanNode

    // 设置子节点
    SetChildren(children []PlanNode)
}

// 生成执行计划
func (epg *ExecutionPlanGenerator) Generate(query *OptimizedQuery) (*ExecutionPlan, error) {
    // 1. 创建扫描节点
    scanNode, err := epg.createScanNode(query)
    if err != nil {
        return nil, err
    }

    // 2. 创建过滤节点
    filterNode, err := epg.createFilterNode(query, scanNode)
    if err != nil {
        return nil, err
    }

    // 3. 创建向量搜索节点
    vectorSearchNode, err := epg.createVectorSearchNode(query, filterNode)
    if err != nil {
        return nil, err
    }

    // 4. 创建排序节点
    sortNode, err := epg.createSortNode(query, vectorSearchNode)
    if err != nil {
        return nil, err
    }

    // 5. 创建聚合节点
    aggNode, err := epg.createAggregationNode(query, sortNode)
    if err != nil {
        return nil, err
    }

    // 6. 创建分页节点
    paginationNode, err := epg.createPaginationNode(query, aggNode)
    if err != nil {
        return nil, err
    }

    return &ExecutionPlan{
        Root:      paginationNode,
        PlanNodes: epg.collectAllNodes(paginationNode),
        EstimatedCost: epg.estimateTotalCost(paginationNode),
    }, nil
}

// 向量搜索节点
type VectorSearchNode struct {
    // 向量查询条件
    vectorQuery *VectorQuery

    // 索引管理器
    indexManager *IndexManager

    // 相似度计算器
    similarityCalculator *SimilarityCalculator

    // 子节点
    child PlanNode
}

// 执行向量搜索
func (vsn *VectorSearchNode) Execute(ctx context.Context, session *Session) (*ResultSet, error) {
    // 1. 执行子节点获取候选集
    childResult, err := vsn.child.Execute(ctx, session)
    if err != nil {
        return nil, err
    }

    // 2. 加载向量索引
    index, err := vsn.indexManager.LoadIndex(session.CollectionID)
    if err != nil {
        return nil, err
    }

    // 3. 执行向量搜索
    searchResults := make([]SearchResult, 0)
    for _, vector := range vsn.vectorQuery.Vectors {
        results, err := index.Search(vector, vsn.vectorQuery.TopK)
        if err != nil {
            return nil, err
        }
        searchResults = append(searchResults, results...)
    }

    // 4. 合并结果
    mergedResults := vsn.mergeResults(searchResults)

    // 5. 过滤和排序
    filteredResults := vsn.filterByChildResult(mergedResults, childResult)

    return &ResultSet{
        Results: filteredResults,
        Metadata: map[string]interface{}{
            "search_type": "vector",
            "num_results": len(filteredResults),
        },
    }, nil
}
```

## 过滤引擎实现

### 1. 标量过滤引擎

```go
// 标量过滤引擎
type ScalarFilterEngine struct {
    // 过滤器注册表
    filterRegistry *FilterRegistry

    // 表达式计算器
    expressionEvaluator *ExpressionEvaluator

    // 索引管理器
    filterIndexManager *FilterIndexManager
}

// 过滤器接口
type Filter interface {
    // 过滤数据
    Filter(data interface{}) (bool, error)

    // 获取过滤条件
    GetCondition() interface{}

    // 估算选择性
    EstimateSelectivity() float64

    // 优化过滤器
    Optimize() (Filter, error)
}

// 等值过滤器
type EqualFilter struct {
    field     string
    value     interface{}
    fieldName string
}

func (ef *EqualFilter) Filter(data interface{}) (bool, error) {
    // 获取字段值
    fieldValue, err := ef.getFieldValue(data)
    if err != nil {
        return false, err
    }

    // 比较值
    return reflect.DeepEqual(fieldValue, ef.value), nil
}

// 范围过滤器
type RangeFilter struct {
    field     string
    min       interface{}
    max       interface{}
    inclusive bool
    fieldName string
}

func (rf *RangeFilter) Filter(data interface{}) (bool, error) {
    // 获取字段值
    fieldValue, err := rf.getFieldValue(data)
    if err != nil {
        return false, err
    }

    // 类型断言和比较
    switch v := fieldValue.(type) {
    case int:
        minVal, ok := rf.min.(int)
        if !ok {
            return false, fmt.Errorf("type mismatch for min value")
        }
        maxVal, ok := rf.max.(int)
        if !ok {
            return false, fmt.Errorf("type mismatch for max value")
        }

        if rf.inclusive {
            return v >= minVal && v <= maxVal, nil
        }
        return v > minVal && v < maxVal, nil

    case float64:
        minVal, ok := rf.min.(float64)
        if !ok {
            return false, fmt.Errorf("type mismatch for min value")
        }
        maxVal, ok := rf.max.(float64)
        if !ok {
            return false, fmt.Errorf("type mismatch for max value")
        }

        if rf.inclusive {
            return v >= minVal && v <= maxVal, nil
        }
        return v > minVal && v < maxVal, nil

    default:
        return false, fmt.Errorf("unsupported field type: %T", fieldValue)
    }
}

// 模糊匹配过滤器
type LikeFilter struct {
    field     string
    pattern   string
    fieldName string
}

func (lf *LikeFilter) Filter(data interface{}) (bool, error) {
    // 获取字段值
    fieldValue, err := lf.getFieldValue(data)
    if err != nil {
        return false, err
    }

    // 转换为字符串
    strValue, ok := fieldValue.(string)
    if !ok {
        return false, fmt.Errorf("field value is not a string")
    }

    // 转换 LIKE 模式为正则表达式
    regexPattern := lf.likeToRegex(lf.pattern)

    // 编译正则表达式
    regex, err := regexp.Compile(regexPattern)
    if err != nil {
        return false, err
    }

    // 执行匹配
    return regex.MatchString(strValue), nil
}

// LIKE 模式转正则表达式
func (lf *LikeFilter) likeToRegex(pattern string) string {
    regex := regexp.QuoteMeta(pattern)
    regex = strings.ReplaceAll(regex, "%", ".*")
    regex = strings.ReplaceAll(regex, "_", ".")
    return "^" + regex + "$"
}
```

### 2. 地理位置过滤

```go
// 地理位置过滤器
type GeoFilter struct {
    // 过滤类型
    FilterType GeoFilterType

    // 地理位置数据
    GeoData *GeoData

    // 距离阈值（米）
    DistanceThreshold float64

    // 字段名
    FieldName string
}

// 地理过滤类型
type GeoFilterType string

const (
    GeoDistance GeoFilterType = "distance"
    GeoInCircle GeoFilterType = "in_circle"
    GeoInPolygon GeoFilterType = "in_polygon"
)

// 地理位置数据
type GeoData struct {
    // 经度
    Longitude float64

    // 纬度
    Latitude float64

    // 圆形半径（米）
    Radius float64

    // 多边形顶点
    Polygon []*GeoPoint
}

// 地理点
type GeoPoint struct {
    Longitude float64
    Latitude  float64
}

// 地理距离计算
func (gf *GeoFilter) calculateDistance(point1, point2 *GeoPoint) float64 {
    // 使用 Haversine 公式计算球面距离
    const earthRadius = 6371000 // 地球半径（米）

    // 转换为弧度
    lat1 := point1.Latitude * math.Pi / 180
    lat2 := point2.Latitude * math.Pi / 180
    deltaLat := (point2.Latitude - point1.Latitude) * math.Pi / 180
    deltaLon := (point2.Longitude - point1.Longitude) * math.Pi / 180

    // Haversine 公式
    a := math.Sin(deltaLat/2)*math.Sin(deltaLat/2) +
        math.Cos(lat1)*math.Cos(lat2)*
            math.Sin(deltaLon/2)*math.Sin(deltaLon/2)

    c := 2 * math.Atan2(math.Sqrt(a), math.Sqrt(1-a))

    return earthRadius * c
}

// 圆形区域过滤
func (gf *GeoFilter) inCircleFilter(point *GeoPoint) bool {
    if gf.FilterType != GeoInCircle {
        return false
    }

    distance := gf.calculateDistance(point, &GeoPoint{
        Longitude: gf.GeoData.Longitude,
        Latitude:  gf.GeoData.Latitude,
    })

    return distance <= gf.GeoData.Radius
}

// 多边形区域过滤
func (gf *GeoFilter) inPolygonFilter(point *GeoPoint) bool {
    if gf.FilterType != GeoInPolygon {
        return false
    }

    // 使用射线法判断点是否在多边形内
    return gf.pointInPolygon(point, gf.GeoData.Polygon)
}

// 射线法判断点是否在多边形内
func (gf *GeoFilter) pointInPolygon(point *GeoPoint, polygon []*GeoPoint) bool {
    if len(polygon) < 3 {
        return false
    }

    count := 0
    for i := 0; i < len(polygon); i++ {
        j := (i + 1) % len(polygon)

        if gf.rayIntersectsSegment(point, polygon[i], polygon[j]) {
            count++
        }
    }

    return count%2 == 1
}

// 射线与线段相交检测
func (gf *GeoFilter) rayIntersectsSegment(point, segStart, segEnd *GeoPoint) bool {
    if (segStart.Latitude > point.Latitude) != (segEnd.Latitude > point.Latitude) {
        if point.Longitude < (segEnd.Longitude-segStart.Longitude)*
            (point.Latitude-segStart.Latitude)/(segEnd.Latitude-segStart.Latitude)+segStart.Longitude {
            return true
        }
    }
    return false
}
```

## 排序与聚合

### 1. 多维度排序

```go
// 排序引擎
type SortEngine struct {
    // 排序器注册表
    sorterRegistry *SorterRegistry

    // 内存管理器
    memoryManager *MemoryManager

    // 并行排序器
    parallelSorter *ParallelSorter
}

// 排序器接口
type Sorter interface {
    // 排序数据
    Sort(data interface{}, condition *SortCondition) error

    // 获取排序类型
    GetType() SortType

    // 支持的数据类型
    SupportedTypes() []reflect.Type
}

// 向量相似度排序器
type VectorSimilaritySorter struct {
    // 相似度计算器
    similarityCalculator *SimilarityCalculator

    // 排序方向
    ascending bool
}

func (vss *VectorSimilaritySorter) Sort(data interface{}, condition *SortCondition) error {
    results, ok := data.([]SearchResult)
    if !ok {
        return fmt.Errorf("invalid data type for vector similarity sorting")
    }

    // 根据相似度分数排序
    sort.Slice(results, func(i, j int) bool {
        if vss.ascending {
            return results[i].Score < results[j].Score
        }
        return results[i].Score > results[j].Score
    })

    return nil
}

// 多维度复合排序器
type MultiFieldSorter struct {
    // 排序条件
    conditions []*SortCondition

    // 字段提取器
    fieldExtractor *FieldExtractor
}

func (mfs *MultiFieldSorter) Sort(data interface{}, condition *SortCondition) error {
    results, ok := data.([]SearchResult)
    if !ok {
        return fmt.Errorf("invalid data type for multi-field sorting")
    }

    // 多维度排序
    sort.Slice(results, func(i, j int) bool {
        for _, cond := range mfs.conditions {
            // 提取字段值
            valI, err := mfs.fieldExtractor.ExtractField(results[i].Data, cond.FieldName)
            if err != nil {
                continue
            }

            valJ, err := mfs.fieldExtractor.ExtractField(results[j].Data, cond.FieldName)
            if err != nil {
                continue
            }

            // 比较值
            cmp := mfs.compareValues(valI, valJ, cond.Ascending)
            if cmp != 0 {
                return cmp < 0
            }
        }
        return false
    })

    return nil
}

// 值比较
func (mfs *MultiFieldSorter) compareValues(valI, valJ interface{}, ascending bool) int {
    switch vi := valI.(type) {
    case int:
        vj, ok := valJ.(int)
        if !ok {
            return 0
        }
        if ascending {
            return vi - vj
        }
        return vj - vi
    case float64:
        vj, ok := valJ.(float64)
        if !ok {
            return 0
        }
        if ascending {
            if vi < vj {
                return -1
            } else if vi > vj {
                return 1
            }
            return 0
        }
        if vi > vj {
            return -1
        } else if vi < vj {
            return 1
        }
        return 0
    case string:
        vj, ok := valJ.(string)
        if !ok {
            return 0
        }
        if ascending {
            return strings.Compare(vi, vj)
        }
        return strings.Compare(vj, vi)
    default:
        return 0
    }
}
```

### 2. 结果聚合

```go
// 聚合引擎
type AggregationEngine struct {
    // 聚合器注册表
    aggregatorRegistry *AggregatorRegistry

    // 分组器
    grouper *Grouper

    // 内存管理器
    memoryManager *MemoryManager
}

// 聚合器接口
type Aggregator interface {
    // 聚合数据
    Aggregate(data interface{}, params *AggregationParams) (*AggregationResult, error)

    // 获取聚合类型
    GetType() AggregationType

    // 重置聚合状态
    Reset()
}

// 计数聚合器
type CountAggregator struct {
    count int64
}

func (ca *CountAggregator) Aggregate(data interface{}, params *AggregationParams) (*AggregationResult, error) {
    switch v := data.(type) {
    case []SearchResult:
        ca.count += int64(len(v))
    case SearchResult:
        ca.count++
    default:
        return nil, fmt.Errorf("unsupported data type for count aggregation")
    }

    return &AggregationResult{
        Type:  "count",
        Value: ca.count,
    }, nil
}

func (ca *CountAggregator) Reset() {
    ca.count = 0
}

// 平均值聚合器
type AverageAggregator struct {
    sum   float64
    count int64
}

func (aa *AverageAggregator) Aggregate(data interface{}, params *AggregationParams) (*AggregationResult, error) {
    switch v := data.(type) {
    case []SearchResult:
        for _, result := range v {
            value, err := aa.extractFieldValue(result.Data, params.FieldName)
            if err != nil {
                continue
            }
            aa.sum += value
            aa.count++
        }
    case SearchResult:
        value, err := aa.extractFieldValue(v.Data, params.FieldName)
        if err != nil {
            return nil, err
        }
        aa.sum += value
        aa.count++
    default:
        return nil, fmt.Errorf("unsupported data type for average aggregation")
    }

    if aa.count == 0 {
        return &AggregationResult{
            Type:  "average",
            Value: 0.0,
        }, nil
    }

    return &AggregationResult{
        Type:  "average",
        Value: aa.sum / float64(aa.count),
    }, nil
}

// 分组聚合
type GroupingAggregator struct {
    // 分组字段
    groupField string

    // 子聚合器
    subAggregators map[string]Aggregator

    // 分组结果
    groupResults map[string]*AggregationResult
}

func (ga *GroupingAggregator) Aggregate(data interface{}, params *AggregationParams) (*AggregationResult, error) {
    switch v := data.(type) {
    case []SearchResult:
        for _, result := range v {
            // 提取分组键
            groupKey, err := ga.extractGroupKey(result.Data)
            if err != nil {
                continue
            }

            // 初始化分组
            if ga.groupResults[groupKey] == nil {
                ga.groupResults[groupKey] = &AggregationResult{
                    Type:     "group",
                    GroupKey: groupKey,
                    SubResults: make(map[string]interface{}),
                }
            }

            // 执行子聚合
            for aggName, agg := range ga.subAggregators {
                subResult, err := agg.Aggregate(result, params)
                if err != nil {
                    continue
                }
                ga.groupResults[groupKey].SubResults[aggName] = subResult
            }
        }
    default:
        return nil, fmt.Errorf("unsupported data type for grouping aggregation")
    }

    return &AggregationResult{
        Type:         "grouping",
        GroupResults: ga.groupResults,
    }, nil
}
```

## 查询优化技术

### 1. 过滤条件下推

```go
// 过滤条件下推优化器
type FilterPushdownOptimizer struct {
    // 过滤条件分析器
    filterAnalyzer *FilterAnalyzer

    // 索引选择器
    indexSelector *IndexSelector

    // 成本估算器
    costEstimator *CostEstimator
}

// 执行过滤条件下推优化
func (fpo *FilterPushdownOptimizer) Optimize(plan *ExecutionPlan) (*ExecutionPlan, error) {
    // 1. 分析过滤条件
    filterConditions := fpo.filterAnalyzer.AnalyzeFilters(plan)

    // 2. 选择可下推的过滤条件
    pushdownFilters := fpo.selectPushdownFilters(filterConditions)

    // 3. 生成下推执行计划
    optimizedPlan, err := fpo.generatePushdownPlan(plan, pushdownFilters)
    if err != nil {
        return nil, err
    }

    // 4. 验证优化效果
    if fpo.validateOptimization(plan, optimizedPlan) {
        return optimizedPlan, nil
    }

    return plan, nil
}

// 选择可下推的过滤条件
func (fpo *FilterPushdownOptimizer) selectPushdownFilters(conditions []*FilterCondition) []*FilterCondition {
    pushdownFilters := make([]*FilterCondition, 0)

    for _, condition := range conditions {
        // 检查是否支持下推
        if fpo.isPushdownSupported(condition) {
            // 估算下推收益
            benefit := fpo.estimatePushdownBenefit(condition)
            if benefit > fpo.pushdownThreshold {
                pushdownFilters = append(pushdownFilters, condition)
            }
        }
    }

    return pushdownFilters
}

// 检查过滤条件下推支持
func (fpo *FilterPushdownOptimizer) isPushdownSupported(condition *FilterCondition) bool {
    // 检查字段类型
    fieldType := fpo.getFieldType(condition.FieldName)
    if fieldType == nil {
        return false
    }

    // 检查操作符支持
    switch condition.Operator {
    case "=", "!=", ">", "<", ">=", "<=", "IN", "NOT IN":
        return true
    case "LIKE", "NOT LIKE":
        return fpo.isLikePushdownSupported(fieldType)
    default:
        return false
    }
}

// 生成下推执行计划
func (fpo *FilterPushdownOptimizer) generatePushdownPlan(
    originalPlan *ExecutionPlan,
    pushdownFilters []*FilterCondition,
) (*ExecutionPlan, error) {
    // 为每个下推过滤条件创建下推节点
    pushdownNodes := make([]PlanNode, 0)
    for _, filter := range pushdownFilters {
        pushdownNode := fpo.createPushdownNode(filter)
        pushdownNodes = append(pushdownNodes, pushdownNode)
    }

    // 构建优化后的执行计划
    optimizedPlan := &ExecutionPlan{
        Root:      fpo.buildOptimizedPlan(originalPlan.Root, pushdownNodes),
        PlanNodes: fpo.collectAllNodes(originalPlan.Root),
    }

    return optimizedPlan, nil
}
```

### 2. 查询重写优化

```go
// 查询重写优化器
type QueryRewriteOptimizer struct {
    // 重写规则引擎
    rewriteRules *RewriteRules

    // 统计信息管理器
    statsManager *StatisticsManager

    // 成本模型
    costModel *CostModel
}

// 执行查询重写
func (qro *QueryRewriteOptimizer) Rewrite(query *HybridSearchRequest) (*HybridSearchRequest, error) {
    // 1. 应用重写规则
    rewrittenQuery := qro.applyRewriteRules(query)

    // 2. 基于统计信息优化
    optimizedQuery := qro.optimizeByStatistics(rewrittenQuery)

    // 3. 成本验证
    if qro.validateRewrite(query, optimizedQuery) {
        return optimizedQuery, nil
    }

    return query, nil
}

// 应用重写规则
func (qro *QueryRewriteOptimizer) applyRewriteRules(query *HybridSearchRequest) *HybridSearchRequest {
    rewrittenQuery := qro.deepCopyQuery(query)

    // 规则1：常量折叠
    rewrittenQuery = qro.constantFolding(rewrittenQuery)

    // 规则2：谓词下推
    rewrittenQuery = qro.predicatePushdown(rewrittenQuery)

    // 规则3：冗余过滤消除
    rewrittenQuery = qro.redundantFilterElimination(rewrittenQuery)

    // 规则4：连接顺序优化
    rewrittenQuery = qro.joinOrderOptimization(rewrittenQuery)

    return rewrittenQuery
}

// 常量折叠
func (qro *QueryRewriteOptimizer) constantFolding(query *HybridSearchRequest) *HybridSearchRequest {
    for _, filter := range query.ScalarFilters {
        // 检查是否为常量表达式
        if qro.isConstantExpression(filter) {
            // 计算常量值
            result, err := qro.evaluateConstantExpression(filter)
            if err == nil {
                // 替换为计算结果
                filter.Value = result
            }
        }
    }
    return query
}

// 冗余过滤消除
func (qro *QueryRewriteOptimizer) redundantFilterElimination(query *HybridSearchRequest) *HybridSearchRequest {
    optimizedFilters := make([]*ScalarFilter, 0)

    for i, filter1 := range query.ScalarFilters {
        isRedundant := false

        // 检查是否与其他过滤条件冗余
        for j, filter2 := range query.ScalarFilters {
            if i != j && qro.isRedundantFilter(filter1, filter2) {
                isRedundant = true
                break
            }
        }

        if !isRedundant {
            optimizedFilters = append(optimizedFilters, filter1)
        }
    }

    query.ScalarFilters = optimizedFilters
    return query
}
```

## 实际应用案例

### 1. 电商商品搜索

```go
// 电商商品搜索实现
type EcommerceProductSearch struct {
    // 混合搜索引擎
    hybridSearchEngine *HybridSearchEngine

    // 商品特征提取器
    featureExtractor *ProductFeatureExtractor

    // 结果处理器
    resultProcessor *ProductResultProcessor
}

// 商品搜索请求
type ProductSearchRequest struct {
    // 搜索关键词
    Keywords string

    // 商品类别
    Category string

    // 价格范围
    PriceRange *PriceRange

    // 品牌过滤
    Brands []string

    // 评分过滤
    MinRating float64

    // 地理位置过滤
    Location *GeoFilter

    // 排序条件
    SortBy string

    // 分页参数
    Page int
    PageSize int
}

// 执行商品搜索
func (eps *EcommerceProductSearch) SearchProducts(req *ProductSearchRequest) (*ProductSearchResponse, error) {
    // 1. 构建混合搜索查询
    hybridQuery := eps.buildHybridQuery(req)

    // 2. 执行搜索
    searchResults, err := eps.hybridSearchEngine.ExecuteSearch(hybridQuery)
    if err != nil {
        return nil, err
    }

    // 3. 处理搜索结果
    processedResults := eps.resultProcessor.ProcessResults(searchResults)

    // 4. 构建响应
    response := &ProductSearchResponse{
        Products: processedResults,
        Total:    len(processedResults),
        Page:     req.Page,
        PageSize: req.PageSize,
        Facets:   eps.generateFacets(searchResults),
    }

    return response, nil
}

// 构建混合搜索查询
func (eps *EcommerceProductSearch) buildHybridQuery(req *ProductSearchRequest) *HybridSearchRequest {
    query := &HybridSearchRequest{
        VectorQuery:      eps.buildVectorQuery(req),
        ScalarFilters:    eps.buildScalarFilters(req),
        SortConditions:   eps.buildSortConditions(req),
        Pagination:       eps.buildPagination(req),
        Aggregations:     eps.buildAggregations(req),
    }

    return query
}

// 构建向量查询
func (eps *EcommerceProductSearch) buildVectorQuery(req *ProductSearchRequest) *VectorQuery {
    // 从搜索关键词提取向量
    featureVector := eps.featureExtractor.ExtractFeatures(req.Keywords)

    return &VectorQuery{
        Vectors:     [][]float32{featureVector},
        SearchParams: &SearchParams{
            Nprobe: 64,
            Ef:     200,
        },
        MetricType: "cosine",
        TopK:       100,
        MinScore:   0.3,
    }
}

// 构建标量过滤条件
func (eps *EcommerceProductSearch) buildScalarFilters(req *ProductSearchRequest) []*ScalarFilter {
    filters := make([]*ScalarFilter, 0)

    // 类别过滤
    if req.Category != "" {
        filters = append(filters, &ScalarFilter{
            FieldName: "category",
            Operator:  "=",
            Value:     req.Category,
        })
    }

    // 价格范围过滤
    if req.PriceRange != nil {
        if req.PriceRange.Min > 0 {
            filters = append(filters, &ScalarFilter{
                FieldName: "price",
                Operator:  ">=",
                Value:     req.PriceRange.Min,
            })
        }
        if req.PriceRange.Max > 0 {
            filters = append(filters, &ScalarFilter{
                FieldName: "price",
                Operator:  "<=",
                Value:     req.PriceRange.Max,
            })
        }
    }

    // 品牌过滤
    if len(req.Brands) > 0 {
        filters = append(filters, &ScalarFilter{
            FieldName: "brand",
            Operator:  "IN",
            Value:     req.Brands,
        })
    }

    // 评分过滤
    if req.MinRating > 0 {
        filters = append(filters, &ScalarFilter{
            FieldName: "rating",
            Operator:  ">=",
            Value:     req.MinRating,
        })
    }

    return filters
}
```

### 2. 文档智能检索

```go
// 文档智能检索实现
type DocumentRetrievalSystem struct {
    // 混合搜索引擎
    hybridSearchEngine *HybridSearchEngine

    // 文档索引管理器
    documentIndexManager *DocumentIndexManager

    // 语义理解引擎
    semanticEngine *SemanticEngine

    // 相关性计算器
    relevanceCalculator *RelevanceCalculator
}

// 文档搜索请求
type DocumentSearchRequest struct {
    // 查询文本
    QueryText string

    // 文档类型过滤
    DocumentTypes []string

    // 时间范围过滤
    TimeRange *TimeRangeFilter

    // 作者过滤
    Authors []string

    // 标签过滤
    Tags []string

    // 相关性阈值
    RelevanceThreshold float64

    // 排序条件
    SortBy string

    // 聚合参数
    Aggregations []string
}

// 执行文档搜索
func (drs *DocumentRetrievalSystem) SearchDocuments(req *DocumentSearchRequest) (*DocumentSearchResponse, error) {
    // 1. 语义理解
    semanticQuery, err := drs.semanticEngine.UnderstandQuery(req.QueryText)
    if err != nil {
        return nil, err
    }

    // 2. 构建混合搜索查询
    hybridQuery := drs.buildHybridQuery(req, semanticQuery)

    // 3. 执行搜索
    searchResults, err := drs.hybridSearchEngine.ExecuteSearch(hybridQuery)
    if err != nil {
        return nil, err
    }

    // 4. 相关性重排序
    rerankedResults := drs.relevanceCalculator.Rerank(searchResults, semanticQuery)

    // 5. 构建响应
    response := &DocumentSearchResponse{
        Documents: rerankedResults,
        Total:     len(rerankedResults),
        Query:     req.QueryText,
        Highlights: drs.generateHighlights(rerankedResults),
        Suggestions: drs.generateSuggestions(req.QueryText),
    }

    return response, nil
}

// 语义理解
func (se *SemanticEngine) UnderstandQuery(queryText string) (*SemanticQuery, error) {
    // 1. 文本预处理
    processedText := se.preprocessText(queryText)

    // 2. 实体识别
    entities := se.extractEntities(processedText)

    // 3. 意图识别
    intent := se.recognizeIntent(processedText)

    // 4. 关键词提取
    keywords := se.extractKeywords(processedText)

    // 5. 语义向量生成
    semanticVector := se.generateSemanticVector(processedText)

    return &SemanticQuery{
        OriginalText:   queryText,
        ProcessedText:  processedText,
        Entities:       entities,
        Intent:         intent,
        Keywords:       keywords,
        SemanticVector: semanticVector,
    }, nil
}

// 相关性重排序
func (rc *RelevanceCalculator) Rerank(
    results []SearchResult,
    query *SemanticQuery,
) []SearchResult {
    // 1. 计算相关性分数
    for i, result := range results {
        relevanceScore := rc.calculateRelevanceScore(result, query)
        results[i].Score = relevanceScore
    }

    // 2. 按相关性分数排序
    sort.Slice(results, func(i, j int) bool {
        return results[i].Score > results[j].Score
    })

    // 3. 多样性优化
    diversifiedResults := rc.diversifyResults(results)

    return diversifiedResults
}

// 计算相关性分数
func (rc *RelevanceCalculator) calculateRelevanceScore(
    result SearchResult,
    query *SemanticQuery,
) float64 {
    // 1. 向量相似度分数
    vectorScore := result.Score

    // 2. 文本匹配分数
    textScore := rc.calculateTextMatchScore(result, query)

    // 3. 实体匹配分数
    entityScore := rc.calculateEntityMatchScore(result, query)

    // 4. 时间新鲜度分数
    freshnessScore := rc.calculateFreshnessScore(result)

    // 5. 综合分数计算
    totalScore := rc.combineScores(
        vectorScore,
        textScore,
        entityScore,
        freshnessScore,
    )

    return totalScore
}
```

## 性能优化策略

### 1. 索引优化

```go
// 混合搜索索引优化器
type HybridSearchIndexOptimizer struct {
    // 向量索引管理器
    vectorIndexManager *VectorIndexManager

    // 标量索引管理器
    scalarIndexManager *ScalarIndexManager

    // 复合索引管理器
    compositeIndexManager *CompositeIndexManager

    // 统计信息管理器
    statsManager *StatisticsManager
}

// 优化索引配置
func (hsio *HybridSearchIndexOptimizer) OptimizeIndexes(collection *Collection) error {
    // 1. 分析数据特征
    dataCharacteristics := hsio.analyzeDataCharacteristics(collection)

    // 2. 优化向量索引
    if err := hsio.optimizeVectorIndexes(collection, dataCharacteristics); err != nil {
        return err
    }

    // 3. 优化标量索引
    if err := hsio.optimizeScalarIndexes(collection, dataCharacteristics); err != nil {
        return err
    }

    // 4. 优化复合索引
    if err := hsio.optimizeCompositeIndexes(collection, dataCharacteristics); err != nil {
        return err
    }

    return nil
}

// 优化向量索引
func (hsio *HybridSearchIndexOptimizer) optimizeVectorIndexes(
    collection *Collection,
    dataCharacteristics *DataCharacteristics,
) error {
    // 1. 选择最优向量索引类型
    optimalIndexType := hsio.selectOptimalVectorIndex(dataCharacteristics)

    // 2. 优化索引参数
    optimalParams := hsio.optimizeVectorIndexParameters(optimalIndexType, dataCharacteristics)

    // 3. 创建或重建索引
    if err := hsio.vectorIndexManager.CreateIndex(
        collection.ID,
        optimalIndexType,
        optimalParams,
    ); err != nil {
        return err
    }

    return nil
}

// 优化标量索引
func (hsio *HybridSearchIndexOptimizer) optimizeScalarIndexes(
    collection *Collection,
    dataCharacteristics *DataCharacteristics,
) error {
    // 分析标量字段特征
    for _, field := range collection.Schema.Fields {
        if field.IsScalar {
            // 1. 选择索引类型
            indexType := hsio.selectScalarIndexType(field, dataCharacteristics)

            // 2. 创建标量索引
            if indexType != "" {
                if err := hsio.scalarIndexManager.CreateIndex(
                    collection.ID,
                    field.Name,
                    indexType,
                ); err != nil {
                    return err
                }
            }
        }
    }

    return nil
}
```

### 2. 缓存优化

```go
// 混合搜索缓存优化器
type HybridSearchCacheOptimizer struct {
    // 查询缓存
    queryCache *QueryCache

    // 结果缓存
    resultCache *ResultCache

    // 索引缓存
    indexCache *IndexCache

    // 预热管理器
    warmupManager *WarmupManager
}

// 优化缓存策略
func (hsco *HybridSearchCacheOptimizer) OptimizeCaching(workload *WorkloadAnalysis) error {
    // 1. 分析查询模式
    queryPatterns := hsco.analyzeQueryPatterns(workload)

    // 2. 优化查询缓存
    if err := hsco.optimizeQueryCache(queryPatterns); err != nil {
        return err
    }

    // 3. 优化结果缓存
    if err := hsco.optimizeResultCache(queryPatterns); err != nil {
        return err
    }

    // 4. 优化索引缓存
    if err := hsco.optimizeIndexCache(queryPatterns); err != nil {
        return err
    }

    // 5. 执行缓存预热
    if err := hsco.warmupManager.ExecuteWarmup(queryPatterns); err != nil {
        return err
    }

    return nil
}

// 分析查询模式
func (hsco *HybridSearchCacheOptimizer) analyzeQueryPatterns(workload *WorkloadAnalysis) *QueryPatternAnalysis {
    analysis := &QueryPatternAnalysis{
        FrequentQueries:    make(map[string]int),
        FrequentFilters:    make(map[string]int),
        FrequentSorts:      make(map[string]int),
        FrequentAggregations: make(map[string]int),
    }

    // 分析查询频率
    for _, query := range workload.Queries {
        analysis.FrequentQueries[query.Fingerprint]++

        // 分析过滤条件
        for _, filter := range query.Filters {
            filterKey := fmt.Sprintf("%s:%s:%v", filter.Field, filter.Operator, filter.Value)
            analysis.FrequentFilters[filterKey]++
        }

        // 分析排序条件
        for _, sort := range query.Sorts {
            sortKey := fmt.Sprintf("%s:%s", sort.Field, sort.Direction)
            analysis.FrequentSorts[sortKey]++
        }

        // 分析聚合条件
        for _, agg := range query.Aggregations {
            aggKey := fmt.Sprintf("%s:%s", agg.Field, agg.Function)
            analysis.FrequentAggregations[aggKey]++
        }
    }

    return analysis
}

// 优化查询缓存
func (hsco *HybridSearchCacheOptimizer) optimizeQueryCache(patterns *QueryPatternAnalysis) error {
    // 获取高频查询
    topQueries := hsco.getTopQueries(patterns.FrequentQueries, 100)

    // 预编译查询
    for queryFingerprint := range topQueries {
        if err := hsco.queryCache.PrecompileQuery(queryFingerprint); err != nil {
            log.Warnf("Failed to precompile query %s: %v", queryFingerprint, err)
        }
    }

    return nil
}
```

## 总结

Milvus 的混合搜索功能为用户提供了强大的复合查询能力，将向量搜索与结构化数据查询完美结合。通过灵活的过滤条件、多维度排序、丰富的聚合功能和智能的查询优化，混合搜索能够满足各种复杂的业务场景需求。

### 核心优势：

1. **查询灵活性**：支持向量搜索、标量过滤、地理位置查询等多种查询类型的组合
2. **性能优化**：通过过滤条件下推、查询重写、索引优化等技术提升查询性能
3. **结果丰富性**：支持排序、聚合、分页等功能，提供丰富的查询结果
4. **易于使用**：提供简洁的 API 接口和灵活的查询语法
5. **高可扩展性**：支持自定义过滤器、聚合器和排序器

### 应用场景：

- **电商搜索**：商品相似性搜索 + 价格/品牌/类别过滤
- **文档检索**：语义搜索 + 文档类型/时间/作者过滤
- **推荐系统**：向量相似度 + 用户属性/行为过滤
- **图像搜索**：图像相似性 + 元数据过滤
- **社交网络**：用户相似性 + 兴趣标签/地理位置过滤

通过本文介绍的技术和最佳实践，您可以充分利用 Milvus 的混合搜索功能，构建出功能强大、性能优异的智能搜索系统。

在下一篇文章中，我们将深入探讨 Milvus 的流式处理架构，了解如何实现实时的数据流处理和增量索引更新。

---

**作者简介**：本文作者专注于搜索技术和数据处理领域，拥有丰富的搜索引擎和推荐系统开发经验。

**相关链接**：
- [Milvus 混合搜索文档](https://milvus.io/docs/hybridsearch.md)
- [向量数据库最佳实践](https://milvus.io/docs/best_practices.md)
- [搜索引擎优化技术](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)