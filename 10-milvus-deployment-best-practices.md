# Milvus 部署最佳实践：从测试环境到生产环境的完整指南

## 前言

将 Milvus 从测试环境部署到生产环境是一个系统性的工程，涉及架构设计、资源配置、安全策略、监控告警等多个方面。本文将为您提供完整的 Milvus 部署指南，帮助您构建稳定、高效、安全的向量数据库生产环境。

## 部署架构设计

### 1. 环境规划

```go
// 部署环境规划器
type DeploymentPlanner struct {
    // 环境分析器
    environmentAnalyzer *EnvironmentAnalyzer

    // 容量规划器
    capacityPlanner *CapacityPlanner

    // 架构设计器
    architectDesigner *ArchitectDesigner

    // 风险评估器
    riskAssessor *RiskAssessor
}

// 部署环境类型
type DeploymentEnvironment string

const (
    // 开发环境
    Development DeploymentEnvironment = "development"

    // 测试环境
    Testing DeploymentEnvironment = "testing"

    // 预生产环境
    Staging DeploymentEnvironment = "staging"

    // 生产环境
    Production DeploymentEnvironment = "production"
)

// 环境配置
type EnvironmentConfig struct {
    // 环境类型
    Environment DeploymentEnvironment

    // 规模等级
    ScaleLevel ScaleLevel

    // 可用性要求
    AvailabilityRequirement AvailabilityLevel

    // 性能要求
    PerformanceRequirement *PerformanceRequirement

    // 安全要求
    SecurityRequirement *SecurityRequirement

    // 成本约束
    CostConstraint *CostConstraint

    // 运维要求
    OperationsRequirement *OperationsRequirement
}

// 规模等级
type ScaleLevel string

const (
    // 小规模（单机）
    SmallScale ScaleLevel = "small"

    // 中规模（集群）
    MediumScale ScaleLevel = "medium"

    // 大规模（多集群）
    LargeScale ScaleLevel = "large"

    // 超大规模（跨地域）
    ExtraLargeScale ScaleLevel = "extra_large"
)

// 环境分析
func (dp *DeploymentPlanner) AnalyzeEnvironment(config *EnvironmentConfig) (*EnvironmentAnalysis, error) {
    analysis := &EnvironmentAnalysis{
        Environment: config.Environment,
        ScaleLevel:  config.ScaleLevel,
        Timestamp:  time.Now(),
    }

    // 1. 分析基础设施
    infrastructure := dp.environmentAnalyzer.AnalyzeInfrastructure()
    analysis.Infrastructure = infrastructure

    // 2. 分析网络拓扑
    networkTopology := dp.environmentAnalyzer.AnalyzeNetworkTopology()
    analysis.NetworkTopology = networkTopology

    // 3. 分析存储能力
    storageCapacity := dp.environmentAnalyzer.AnalyzeStorageCapacity()
    analysis.StorageCapacity = storageCapacity

    // 4. 分析计算能力
    computeCapacity := dp.environmentAnalyzer.AnalyzeComputeCapacity()
    analysis.ComputeCapacity = computeCapacity

    // 5. 分析安全环境
    securityEnvironment := dp.environmentAnalyzer.AnalyzeSecurityEnvironment()
    analysis.SecurityEnvironment = securityEnvironment

    return analysis, nil
}

// 基础设施分析
type InfrastructureAnalysis struct {
    // 硬件资源
    HardwareResources *HardwareResources

    // 操作系统
    OperatingSystem *OSInfo

    // 容器平台
    ContainerPlatform *ContainerPlatformInfo

    // 编排系统
    OrchestrationSystem *OrchestrationInfo

    // 监控系统
    MonitoringSystem *MonitoringInfo

    // 日志系统
    LoggingSystem *LoggingInfo
}

// 硬件资源
type HardwareResources struct {
    // CPU 信息
    CPU *CPUInfo

    // 内存信息
    Memory *MemoryInfo

    // 存储信息
    Storage *StorageInfo

    // 网络信息
    Network *NetworkInfo

    // GPU 信息
    GPU *GPUInfo
}
```

### 2. 容量规划

```go
// 容量规划器
type CapacityPlanner struct {
    // 工作负载分析器
    workloadAnalyzer *WorkloadAnalyzer

    // 资源计算器
    resourceCalculator *ResourceCalculator

    // 增长预测器
    growthPredictor *GrowthPredictor

    // 成本估算器
    costEstimator *CostEstimator
}

// 工作负载分析
type WorkloadAnalysis struct {
    // 数据特征
    DataCharacteristics *DataCharacteristics

    // 查询模式
    QueryPattern *QueryPattern

    // 并发特征
    ConcurrencyProfile *ConcurrencyProfile

    // 性能要求
    PerformanceTargets *PerformanceTargets

    // 可用性要求
    AvailabilityTargets *AvailabilityTargets
}

// 数据特征
type DataCharacteristics struct {
    // 数据量
    DataVolume DataVolume

    // 增长率
    GrowthRate GrowthRate

    // 数据类型分布
    DataTypeDistribution DataTypeDistribution

    // 向量维度分布
    VectorDimensionDistribution VectorDimensionDistribution

    // 数据访问模式
    DataAccessPattern DataAccessPattern
}

// 数据量
type DataVolume struct {
    // 当前数据量
    CurrentVolume int64

    // 预期月增长量
    MonthlyGrowth int64

    // 预期年增长量
    YearlyGrowth int64

    // 保留期
    RetentionPeriod time.Duration

    // 压缩比
    CompressionRatio float64
}

// 资源需求计算
func (cp *CapacityPlanner) CalculateResourceRequirements(
    workload *WorkloadAnalysis,
    growthPeriod time.Duration,
) (*ResourceRequirements, error) {
    requirements := &ResourceRequirements{
        Timestamp:     time.Now(),
        GrowthPeriod:  growthPeriod,
        CurrentPhase:   PhaseInitial,
        FuturePhases:  make([]*ResourcePhase, 0),
    }

    // 1. 计算当前资源需求
    currentResources := cp.calculateCurrentResources(workload)
    requirements.CurrentResources = currentResources

    // 2. 预测未来增长
    growthProjections := cp.growthPredictor.PredictGrowth(workload, growthPeriod)

    // 3. 计算各阶段资源需求
    for _, projection := range growthProjections {
        phaseResources := cp.calculatePhaseResources(projection)
        requirements.FuturePhases = append(requirements.FuturePhases, phaseResources)
    }

    // 4. 添加安全缓冲
    cp.addSafetyBuffer(requirements)

    // 5. 计算成本
    requirements.CostEstimate = cp.costEstimator.EstimateCost(requirements)

    return requirements, nil
}

// 计算当前资源需求
func (cp *CapacityPlanner) calculateCurrentResources(workload *WorkloadAnalysis) *ResourceAllocation {
    allocation := &ResourceAllocation{}

    // 1. 计算存储需求
    storageAllocation := cp.calculateStorageAllocation(workload.DataCharacteristics)
    allocation.Storage = storageAllocation

    // 2. 计算内存需求
    memoryAllocation := cp.calculateMemoryAllocation(workload.DataCharacteristics, workload.QueryPattern)
    allocation.Memory = memoryAllocation

    // 3. 计算CPU需求
    cpuAllocation := cp.calculateCPUAllocation(workload.QueryPattern, workload.ConcurrencyProfile)
    allocation.CPU = cpuAllocation

    // 4. 计算GPU需求
    gpuAllocation := cp.calculateGPUAllocation(workload.QueryPattern)
    allocation.GPU = gpuAllocation

    // 5. 计算网络需求
    networkAllocation := cp.calculateNetworkAllocation(workload.QueryPattern, workload.ConcurrencyProfile)
    allocation.Network = networkAllocation

    return allocation
}

// 计算存储分配
func (cp *CapacityPlanner) calculateStorageAllocation(dataChar *DataCharacteristics) *StorageAllocation {
    allocation := &StorageAllocation{}

    // 1. 计算向量数据存储
    vectorStorage := dataChar.DataVolume.CurrentVolume * int64(dataChar.VectorDimensionDistribution.AverageDimension*4) / 1024 / 1024 // MB
    allocation.VectorData = vectorStorage

    // 2. 计算索引存储
    indexStorage := vectorStorage * 2 // 索引通常是原始数据的2倍
    allocation.IndexData = indexStorage

    // 3. 计算元数据存储
    metadataStorage := vectorStorage * 0.1 // 元数据约占总存储的10%
    allocation.Metadata = metadataStorage

    // 4. 计算日志存储
    logStorage := vectorStorage * 0.05 // 日志约占总存储的5%
    allocation.Logs = logStorage

    // 5. 计算备份存储
    backupStorage := (vectorStorage + indexStorage + metadataStorage) * 3 // 3个备份副本
    allocation.Backup = backupStorage

    // 6. 计算总存储需求
    allocation.Total = vectorStorage + indexStorage + metadataStorage + logStorage + backupStorage

    // 7. 添加增长缓冲
    allocation.Total = int64(float64(allocation.Total) * 1.3) // 30%增长缓冲

    return allocation
}

// 计算内存分配
func (cp *CapacityPlanner) calculateMemoryAllocation(
    dataChar *DataCharacteristics,
    queryPattern *QueryPattern,
) *MemoryAllocation {
    allocation := &MemoryAllocation{}

    // 1. 计算向量索引内存
    indexMemory := dataChar.DataVolume.CurrentVolume * int64(dataChar.VectorDimensionDistribution.AverageDimension*4) / 1024 / 1024 // MB
    allocation.IndexMemory = indexMemory

    // 2. 计算缓存内存
    cacheMemory := indexMemory * 0.3 // 缓存为索引内存的30%
    allocation.CacheMemory = cacheMemory

    // 3. 计算查询处理内存
    queryMemory := cp.calculateQueryMemory(queryPattern)
    allocation.QueryMemory = queryMemory

    // 4. 计算系统开销内存
    systemMemory := 4096 // 系统基础内存4GB
    allocation.SystemMemory = systemMemory

    // 5. 计算总内存需求
    allocation.Total = indexMemory + cacheMemory + queryMemory + systemMemory

    // 6. 添加安全缓冲
    allocation.Total = int64(float64(allocation.Total) * 1.2) // 20%安全缓冲

    return allocation
}
```

## Kubernetes 部署

### 1. Helm Chart 配置

```yaml
# milvus-values.yaml
# Milvus 集群配置
global:
  # 命名空间
  namespace: "milvus-prod"

  # 镜像仓库配置
  image:
    repository: "milvusdb/milvus"
    tag: "v2.3.0"
    pullPolicy: "IfNotPresent"

  # 资源限制
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "8"
      memory: "16Gi"

# 协调器配置
coordinator:
  # 副本数
  replicas: 3

  # 资源配置
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

  # 高可用配置
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - milvus-coordinator
        topologyKey: "kubernetes.io/hostname"

  # 污点容忍
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "milvus"
    effect: "NoSchedule"

# 查询节点配置
queryNode:
  # 副本数
  replicas: 5

  # 资源配置
  resources:
    requests:
      cpu: "4"
      memory: "16Gi"
    limits:
      cpu: "16"
      memory: "64Gi"

  # GPU 配置
  gpu:
    enabled: true
    count: 1
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1

  # 存储配置
  storage:
    # 内存类型
    type: "memory"
    size: "32Gi"

    # SSD 缓存
    ssdCache:
      enabled: true
      size: "100Gi"

# 数据节点配置
dataNode:
  # 副本数
  replicas: 3

  # 资源配置
  resources:
    requests:
      cpu: "2"
      memory: "8Gi"
    limits:
      cpu: "8"
      memory: "32Gi"

  # 存储配置
  storage:
    # 数据存储
    data:
      size: "1Ti"
      storageClass: "fast-ssd"

    # WAL 存储
    wal:
      size: "500Gi"
      storageClass: "ultra-ssd"

# 代理配置
proxy:
  # 副本数
  replicas: 3

  # 资源配置
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

  # 服务配置
  service:
    type: "LoadBalancer"
    port: 19530
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

# 依赖服务配置
dependencies:
  # etcd 配置
  etcd:
    enabled: true
    replicaCount: 3
    persistence:
      enabled: true
      size: "100Gi"
      storageClass: "fast-ssd"

  # MinIO 配置
  minio:
    enabled: true
    mode: "distributed"
    replicas: 4
    persistence:
      enabled: true
      size: "1Ti"
      storageClass: "fast-ssd"

  # Pulsar 配置
  pulsar:
    enabled: true
    broker:
      replicaCount: 3
      resources:
        requests:
          cpu: "1"
          memory: "2Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
    zookeeper:
      replicaCount: 3
    bookkeeper:
      replicaCount: 3

# 监控配置
monitoring:
  # Prometheus 配置
  prometheus:
    enabled: true
    retention: "30d"
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "4"
        memory: "8Gi"

  # Grafana 配置
  grafana:
    enabled: true
    adminPassword: "your-secure-password"
    persistence:
      enabled: true
      size: "10Gi"

# 日志配置
logging:
  # 日志级别
  level: "info"

  # 日志格式
  format: "json"

  # 日志轮转
  rotation:
    enabled: true
    maxAge: "7d"
    maxSize: "100Mi"

  # 日志输出
  output:
    # 文件输出
    file:
      enabled: true
      path: "/var/log/milvus"

    # Elasticsearch 输出
    elasticsearch:
      enabled: true
      hosts:
        - "http://elasticsearch:9200"
      index: "milvus-logs"

# 安全配置
security:
  # TLS 配置
  tls:
    enabled: true
    certSecret: "milvus-tls-certs"

  # 认证配置
  authentication:
    enabled: true
    method: "jwt"
    jwtSecret: "your-jwt-secret"

  # 网络策略
  networkPolicy:
    enabled: true
    ingressRules:
      - from:
        - namespaceSelector:
            matchLabels:
              name: "milvus-client"
        ports:
          - port: 19530
            protocol: TCP

# 备份配置
backup:
  enabled: true
  schedule: "0 2 * * *"  # 每天凌晨2点备份
  retention: "7d"
  storage:
    type: "s3"
    s3:
      bucket: "milvus-backup"
      region: "us-east-1"
      endpoint: "s3.amazonaws.com"
```

### 2. 部署脚本

```go
// 部署管理器
type DeploymentManager struct {
    // Kubernetes 客户端
    kubeClient *kubernetes.Clientset

    // Helm 客户端
    helmClient *helm.Client

    // 配置管理器
    configManager *ConfigManager

    // 验证器
    validator *DeploymentValidator

    // 监控器
    monitor *DeploymentMonitor
}

// 部署配置
type DeploymentConfig struct {
    // 集群名称
    ClusterName string

    // 命名空间
    Namespace string

    // Helm Chart 配置
    HelmChart *HelmChartConfig

    // 资源配置
    Resources *ResourceConfig

    // 网络配置
    Network *NetworkConfig

    // 存储配置
    Storage *StorageConfig

    // 安全配置
    Security *SecurityConfig

    // 监控配置
    Monitoring *MonitoringConfig
}

// Helm Chart 配置
type HelmChartConfig struct {
    // Chart 名称
    ChartName string

    // Chart 版本
    ChartVersion string

    // 仓库地址
    Repository string

    // Values 文件
    ValuesFile string

    // Values 覆盖
    ValuesOverride map[string]interface{}
}

// 执行部署
func (dm *DeploymentManager) Deploy(config *DeploymentConfig) (*DeploymentResult, error) {
    // 1. 验证部署配置
    if err := dm.validator.ValidateConfig(config); err != nil {
        return nil, fmt.Errorf("config validation failed: %w", err)
    }

    // 2. 准备部署环境
    if err := dm.prepareEnvironment(config); err != nil {
        return nil, fmt.Errorf("environment preparation failed: %w", err)
    }

    // 3. 部署依赖服务
    if err := dm.deployDependencies(config); err != nil {
        return nil, fmt.Errorf("dependencies deployment failed: %w", err)
    }

    // 4. 部署 Milvus 集群
    deploymentResult, err := dm.deployMilvusCluster(config)
    if err != nil {
        return nil, fmt.Errorf("Milvus cluster deployment failed: %w", err)
    }

    // 5. 部署监控组件
    if err := dm.deployMonitoring(config); err != nil {
        return nil, fmt.Errorf("monitoring deployment failed: %w", err)
    }

    // 6. 验证部署结果
    if err := dm.validator.ValidateDeployment(deploymentResult); err != nil {
        return nil, fmt.Errorf("deployment validation failed: %w", err)
    }

    // 7. 启动监控
    dm.monitor.StartMonitoring(deploymentResult)

    return deploymentResult, nil
}

// 准备部署环境
func (dm *DeploymentManager) prepareEnvironment(config *DeploymentConfig) error {
    // 1. 创建命名空间
    namespace := &corev1.Namespace{
        ObjectMeta: metav1.ObjectMeta{
            Name: config.Namespace,
            Labels: map[string]string{
                "app":       "milvus",
                "env":       "production",
                "managed-by": "milvus-operator",
            },
        },
    }

    if _, err := dm.kubeClient.CoreV1().Namespaces().Create(context.TODO(), namespace, metav1.CreateOptions{}); err != nil {
        if !k8serrors.IsAlreadyExists(err) {
            return fmt.Errorf("failed to create namespace: %w", err)
        }
    }

    // 2. 创建服务账户
    serviceAccount := &corev1.ServiceAccount{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "milvus-service-account",
            Namespace: config.Namespace,
        },
    }

    if _, err := dm.kubeClient.CoreV1().ServiceAccounts(config.Namespace).Create(context.TODO(), serviceAccount, metav1.CreateOptions{}); err != nil {
        if !k8serrors.IsAlreadyExists(err) {
            return fmt.Errorf("failed to create service account: %w", err)
        }
    }

    // 3. 创建 RBAC 角色
    if err := dm.createRBACRoles(config); err != nil {
        return fmt.Errorf("failed to create RBAC roles: %w", err)
    }

    // 4. 创建存储类
    if err := dm.createStorageClasses(config); err != nil {
        return fmt.Errorf("failed to create storage classes: %w", err)
    }

    // 5. 创建网络策略
    if err := dm.createNetworkPolicies(config); err != nil {
        return fmt.Errorf("failed to create network policies: %w", err)
    }

    return nil
}

// 部署依赖服务
func (dm *DeploymentManager) deployDependencies(config *DeploymentConfig) error {
    // 1. 部署 etcd 集群
    if err := dm.deployEtcd(config); err != nil {
        return fmt.Errorf("failed to deploy etcd: %w", err)
    }

    // 2. 部署 MinIO
    if err := dm.deployMinIO(config); err != nil {
        return fmt.Errorf("failed to deploy MinIO: %w", err)
    }

    // 3. 部署 Pulsar
    if err := dm.deployPulsar(config); err != nil {
        return fmt.Errorf("failed to deploy Pulsar: %w", err)
    }

    return nil
}

// 部署 etcd 集群
func (dm *DeploymentManager) deployEtcd(config *DeploymentConfig) error {
    etcdChart := &HelmReleaseConfig{
        Name:       "milvus-etcd",
        Namespace:  config.Namespace,
        Chart:      "bitnami/etcd",
        Version:    "8.3.0",
        Values: map[string]interface{}{
            "auth": map[string]interface{}{
                "rbac": map[string]interface{}{
                    "create": true,
                    "rules": []map[string]interface{}{
                        {
                            "apiGroups": []string{""},
                            "resources": []string{"pods", "services", "endpoints"},
                            "verbs":     []string{"get", "list", "watch"},
                        },
                    },
                },
            },
            "replicaCount": 3,
            "persistence": map[string]interface{}{
                "enabled": true,
                "size":    "100Gi",
                "storageClass": "fast-ssd",
            },
            "resources": map[string]interface{}{
                "requests": map[string]interface{}{
                    "cpu":    "200m",
                    "memory": "256Mi",
                },
                "limits": map[string]interface{}{
                    "cpu":    "500m",
                    "memory": "512Mi",
                },
            },
            "metrics": map[string]interface{}{
                "enabled": true,
                "serviceMonitor": map[string]interface{}{
                    "enabled": true,
                },
            },
        },
    }

    if err := dm.helmClient.InstallRelease(etcdChart); err != nil {
        return fmt.Errorf("failed to install etcd: %w", err)
    }

    return nil
}

// 部署 Milvus 集群
func (dm *DeploymentManager) deployMilvusCluster(config *DeploymentConfig) (*DeploymentResult, error) {
    // 1. 准备 Helm Values
    values, err := dm.prepareHelmValues(config)
    if err != nil {
        return nil, fmt.Errorf("failed to prepare Helm values: %w", err)
    }

    // 2. 安装 Milvus Helm Chart
    milvusChart := &HelmReleaseConfig{
        Name:       "milvus",
        Namespace:  config.Namespace,
        Chart:      "milvus/milvus",
        Version:    "4.0.1",
        Values:     values,
        Wait:       true,
        Timeout:    30 * time.Minute,
    }

    if err := dm.helmClient.InstallRelease(milvusChart); err != nil {
        return nil, fmt.Errorf("failed to install Milvus: %w", err)
    }

    // 3. 获取部署状态
    deploymentStatus, err := dm.getDeploymentStatus(config)
    if err != nil {
        return nil, fmt.Errorf("failed to get deployment status: %w", err)
    }

    // 4. 构建部署结果
    result := &DeploymentResult{
        Success:      true,
        Timestamp:    time.Now(),
        ClusterName:  config.ClusterName,
        Namespace:    config.Namespace,
        Status:       deploymentStatus,
        Components:   dm.getDeployedComponents(config),
        Endpoints:    dm.getServiceEndpoints(config),
        Resources:    dm.getDeployedResources(config),
    }

    return result, nil
}

// 准备 Helm Values
func (dm *DeploymentManager) prepareHelmValues(config *DeploymentConfig) (map[string]interface{}, error) {
    values := make(map[string]interface{})

    // 全局配置
    values["global"] = map[string]interface{}{
        "namespace": config.Namespace,
        "image": map[string]interface{}{
            "repository": "milvusdb/milvus",
            "tag":        "v2.3.0",
            "pullPolicy": "IfNotPresent",
        },
    }

    // 协调器配置
    values["coordinator"] = map[string]interface{}{
        "replicas": 3,
        "resources": map[string]interface{}{
            "requests": map[string]interface{}{
                "cpu":    "1",
                "memory": "2Gi",
            },
            "limits": map[string]interface{}{
                "cpu":    "4",
                "memory": "8Gi",
            },
        },
        "affinity": map[string]interface{}{
            "podAntiAffinity": map[string]interface{}{
                "requiredDuringSchedulingIgnoredDuringExecution": []map[string]interface{}{
                    {
                        "labelSelector": map[string]interface{}{
                            "matchExpressions": []map[string]interface{}{
                                {
                                    "key":      "app",
                                    "operator": "In",
                                    "values":   []string{"milvus-coordinator"},
                                },
                            },
                        },
                        "topologyKey": "kubernetes.io/hostname",
                    },
                },
            },
        },
    }

    // 查询节点配置
    values["queryNode"] = map[string]interface{}{
        "replicas": 5,
        "resources": map[string]interface{}{
            "requests": map[string]interface{}{
                "cpu":    "4",
                "memory": "16Gi",
            },
            "limits": map[string]interface{}{
                "cpu":    "16",
                "memory": "64Gi",
            },
        },
        "gpu": map[string]interface{}{
            "enabled": true,
            "count":   1,
            "resources": map[string]interface{}{
                "requests": map[string]interface{}{
                    "nvidia.com/gpu": 1,
                },
                "limits": map[string]interface{}{
                    "nvidia.com/gpu": 1,
                },
            },
        },
    }

    // 数据节点配置
    values["dataNode"] = map[string]interface{}{
        "replicas": 3,
        "resources": map[string]interface{}{
            "requests": map[string]interface{}{
                "cpu":    "2",
                "memory": "8Gi",
            },
            "limits": map[string]interface{}{
                "cpu":    "8",
                "memory": "32Gi",
            },
        },
        "storage": map[string]interface{}{
            "data": map[string]interface{}{
                "size":         "1Ti",
                "storageClass": "fast-ssd",
            },
            "wal": map[string]interface{}{
                "size":         "500Gi",
                "storageClass": "ultra-ssd",
            },
        },
    }

    // 合并自定义 Values
    if config.HelmChart.ValuesOverride != nil {
        values = dm.mergeValues(values, config.HelmChart.ValuesOverride)
    }

    return values, nil
}
```

## 运维管理

### 1. 监控告警

```go
// 监控管理器
type MonitoringManager struct {
    // Prometheus 客户端
    prometheusClient *prometheus.Client

    // Grafana 客户端
    grafanaClient *grafana.Client

    // 告警管理器
    alertManager *AlertManager

    // 指标收集器
    metricsCollector *MetricsCollector

    // 仪表板管理器
    dashboardManager *DashboardManager
}

// 告警规则
type AlertRule struct {
    // 规则名称
    Name string

    // 规则表达式
    Expression string

    // 持续时间
    For time.Duration

    // 严重级别
    Severity AlertSeverity

    // 告警描述
    Description string

    // 告警标签
    Labels map[string]string

    // 告警注释
    Annotations map[string]string
}

// 告警严重级别
type AlertSeverity string

const (
    // 紧急
    Emergency AlertSeverity = "emergency"

    // 严重
    Critical AlertSeverity = "critical"

    // 警告
    Warning AlertSeverity = "warning"

    // 信息
    Info AlertSeverity = "info"
)

// 创建监控配置
func (mm *MonitoringManager) CreateMonitoringConfig() (*MonitoringConfig, error) {
    config := &MonitoringConfig{
        Prometheus: &PrometheusConfig{
            Retention: "30d",
            Resources: &ResourceRequirements{
                Requests: &Resource{
                    CPU:    "2",
                    Memory: "4Gi",
                },
                Limits: &Resource{
                    CPU:    "8",
                    Memory: "16Gi",
                },
            },
            ScrapeConfigs: mm.createScrapeConfigs(),
            AlertRules:    mm.createAlertRules(),
        },
        Grafana: &GrafanaConfig{
            AdminPassword: "your-secure-password",
            Resources: &ResourceRequirements{
                Requests: &Resource{
                    CPU:    "200m",
                    Memory: "512Mi",
                },
                Limits: &Resource{
                    CPU:    "2",
                    Memory: "2Gi",
                },
            },
            Dashboards: mm.createDashboards(),
        },
        AlertManager: &AlertManagerConfig{
            Config: mm.createAlertManagerConfig(),
        },
    }

    return config, nil
}

// 创建抓取配置
func (mm *MonitoringManager) createScrapeConfigs() []*ScrapeConfig {
    configs := make([]*ScrapeConfig, 0)

    // Milvus 组件抓取配置
    configs = append(configs, &ScrapeConfig{
        JobName:        "milvus-components",
        ScrapeInterval: "15s",
        ScrapeTimeout:  "10s",
        Scheme:         "http",
        MetricsPath:    "/metrics",
        StaticConfigs: []*StaticConfig{
            {
                Targets: []string{
                    "milvus-proxy:9091",
                    "milvus-rootcoord:9091",
                    "milvus-querycoord:9091",
                    "milvus-datacoord:9091",
                    "milvus-querynode:9091",
                    "milvus-datanode:9091",
                },
                Labels: map[string]string{
                    "app": "milvus",
                },
            },
        },
    })

    // 依赖服务抓取配置
    configs = append(configs, &ScrapeConfig{
        JobName:        "milvus-dependencies",
        ScrapeInterval: "30s",
        ScrapeTimeout:  "20s",
        Scheme:         "http",
        MetricsPath:    "/metrics",
        StaticConfigs: []*StaticConfig{
            {
                Targets: []string{
                    "milvus-etcd:2379",
                    "milvus-minio:9000",
                    "pulsar-broker:8080",
                },
                Labels: map[string]string{
                    "app": "milvus-dependencies",
                },
            },
        },
    })

    // Kubernetes 资源抓取配置
    configs = append(configs, &ScrapeConfig{
        JobName:        "kubernetes-pods",
        ScrapeInterval: "30s",
        ScrapeTimeout:  "20s",
        Scheme:         "https",
        MetricsPath:    "/metrics/cadvisor",
        TLSConfig: &TLSConfig{
            InsecureSkipVerify: true,
        },
        KubernetesSDConfigs: []*KubernetesSDConfig{
            {
                Role: "pod",
            },
        },
    })

    return configs
}

// 创建告警规则
func (mm *MonitoringManager) createAlertRules() []*AlertRule {
    rules := make([]*AlertRule, 0)

    // 高 CPU 使用率告警
    rules = append(rules, &AlertRule{
        Name:        "HighCPUUsage",
        Expression:  "sum(rate(container_cpu_usage_seconds_total{namespace=\"milvus-prod\"}[5m])) by (pod) / sum(kube_pod_container_resource_requests_cpu_cores{namespace=\"milvus-prod\"}) > 0.8",
        For:         5 * time.Minute,
        Severity:    Warning,
        Description: "High CPU usage detected in Milvus pod {{ $labels.pod }}",
        Labels: map[string]string{
            "severity": "warning",
            "category": "resource",
        },
        Annotations: map[string]interface{}{
            "summary":     "High CPU usage",
            "description": "Pod {{ $labels.pod }} has high CPU usage: {{ $value }}%",
        },
    })

    // 高内存使用率告警
    rules = append(rules, &AlertRule{
        Name:        "HighMemoryUsage",
        Expression:  "sum(container_memory_working_set_bytes{namespace=\"milvus-prod\"}) by (pod) / sum(kube_pod_container_resource_requests_memory_bytes{namespace=\"milvus-prod\"}) > 0.9",
        For:         5 * time.Minute,
        Severity:    Critical,
        Description: "High memory usage detected in Milvus pod {{ $labels.pod }}",
        Labels: map[string]string{
            "severity": "critical",
            "category": "resource",
        },
        Annotations: map[string]interface{}{
            "summary":     "High memory usage",
            "description": "Pod {{ $labels.pod }} has high memory usage: {{ $value }}%",
        },
    })

    // 高查询延迟告警
    rules = append(rules, &AlertRule{
        Name:        "HighQueryLatency",
        Expression:  "histogram_quantile(0.95, rate(milvus_proxy_query_latency_seconds_bucket[5m])) > 1.0",
        For:         2 * time.Minute,
        Severity:    Warning,
        Description: "High query latency detected in Milvus proxy",
        Labels: map[string]interface{}{
            "severity": "warning",
            "category": "performance",
        },
        Annotations: map[string]interface{}{
            "summary":     "High query latency",
            "description": "95th percentile query latency is {{ $value }} seconds",
        },
    })

    // 磁盘空间不足告警
    rules = append(rules, &AlertRule{
        Name:        "LowDiskSpace",
        Expression:  "(1 - (kube_node_status_capacity_bytes{resource=\"ephemeral_storage\"} - kube_node_status_capacity_bytes{resource=\"ephemeral_storage\"} - kube_node_status_allocatable_bytes{resource=\"ephemeral_storage\"}) / kube_node_status_capacity_bytes{resource=\"ephemeral_storage\"}) * 100 > 85",
        For:         5 * time.Minute,
        Severity:    Critical,
        Description: "Low disk space detected on node {{ $labels.node }}",
        Labels: map[string]interface{}{
            "severity": "critical",
            "category": "storage",
        },
        Annotations: map[string]interface{}{
            "summary":     "Low disk space",
            "description": "Node {{ $labels.node }} has low disk space: {{ $value }}% used",
        },
    })

    // 组件崩溃告警
    rules = append(rules, &AlertRule{
        Name:        "ComponentDown",
        Expression:  "up{job=\"milvus-components\"} == 0",
        For:         1 * time.Minute,
        Severity:    Emergency,
        Description: "Milvus component {{ $labels.instance }} is down",
        Labels: map[string]interface{}{
            "severity": "emergency",
            "category": "availability",
        },
        Annotations: map[string]interface{}{
            "summary":     "Component down",
            "description": "Milvus component {{ $labels.instance }} is not responding",
        },
    })

    return rules
}

// 创建告警管理器配置
func (mm *MonitoringManager) createAlertManagerConfig() *AlertManagerConfig {
    return &AlertManagerConfig{
        Global: &GlobalConfig{
            SMTPSmarthost: "smtp.gmail.com:587",
            SMTPFrom:      "alerts@example.com",
            SMTPAuthUsername: "alerts@example.com",
            SMTPAuthPassword: "your-smtp-password",
        },
        Route: &RouteConfig{
            GroupBy:        []string{"alertname", "severity"},
            GroupWait:      "10s",
            GroupInterval:   "10s",
            RepeatInterval: "1h",
            Receiver:       "web.hook",
            Routes: []*RouteConfig{
                {
                    Match: map[string]string{
                        "severity": "emergency",
                    },
                    Receiver: "emergency",
                },
                {
                    Match: map[string]string{
                        "severity": "critical",
                    },
                    Receiver: "critical",
                },
                {
                    Match: map[string]string{
                        "severity": "warning",
                    },
                    Receiver: "warning",
                },
            },
        },
        Receivers: []*ReceiverConfig{
            {
                Name: "web.hook",
                WebhookConfigs: []*WebhookConfig{
                    {
                        URL:      "http://webhook-receiver:8080/webhook",
                        SendResolved: true,
                    },
                },
            },
            {
                Name: "emergency",
                EmailConfigs: []*EmailConfig{
                    {
                        To:       "emergency-team@example.com",
                        Subject:   "[EMERGENCY] Milvus Alert: {{ .GroupLabels.alertname }}",
                        Body:      `{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}`,
                    },
                },
            },
            {
                Name: "critical",
                EmailConfigs: []*EmailConfig{
                    {
                        To:       "ops-team@example.com",
                        Subject:   "[CRITICAL] Milvus Alert: {{ .GroupLabels.alertname }}",
                        Body:      `{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}`,
                    },
                },
            },
            {
                Name: "warning",
                EmailConfigs: []*EmailConfig{
                    {
                        To:       "dev-team@example.com",
                        Subject:   "[WARNING] Milvus Alert: {{ .GroupLabels.alertname }}",
                        Body:      `{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}`,
                    },
                },
            },
        },
    }
}
```

### 2. 备份恢复

```go
// 备份管理器
type BackupManager struct {
    // 备份调度器
    backupScheduler *BackupScheduler

    // 备份执行器
    backupExecutor *BackupExecutor

    // 恢复执行器
    restoreExecutor *RestoreExecutor

    // 备份存储管理器
    backupStorageManager *BackupStorageManager

    // 备份验证器
    backupValidator *BackupValidator
}

// 备份配置
type BackupConfig struct {
    // 备份计划
    Schedule *BackupSchedule

    // 存储配置
    Storage *BackupStorageConfig

    // 保留策略
    Retention *BackupRetention

    // 压缩配置
    Compression *BackupCompression

    // 加密配置
    Encryption *BackupEncryption

    // 验证配置
    Verification *BackupVerification
}

// 备份计划
type BackupSchedule struct {
    // 备份类型
    BackupType BackupType

    // 调度表达式
    CronExpression string

    // 备份时间窗口
    TimeWindow *TimeWindow

    // 并发限制
    MaxConcurrentBackups int

    // 重试配置
    RetryPolicy *RetryPolicy
}

// 备份类型
type BackupType string

const (
    // 完整备份
    FullBackup BackupType = "full"

    // 增量备份
    IncrementalBackup BackupType = "incremental"

    // 差异备份
    DifferentialBackup BackupType = "differential"
)

// 创建备份计划
func (bm *BackupManager) CreateBackupPlan(config *BackupConfig) (*BackupPlan, error) {
    plan := &BackupPlan{
        ID:          bm.generateBackupPlanID(),
        Config:      config,
        Status:      BackupPlanStatusCreated,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
        BackupJobs:  make([]*BackupJob, 0),
        Statistics:  &BackupStatistics{},
    }

    // 1. 验证备份配置
    if err := bm.validateBackupConfig(config); err != nil {
        return nil, fmt.Errorf("backup config validation failed: %w", err)
    }

    // 2. 生成备份时间表
    schedule, err := bm.generateBackupSchedule(config.Schedule)
    if err != nil {
        return nil, fmt.Errorf("failed to generate backup schedule: %w", err)
    }
    plan.Schedule = schedule

    // 3. 预估备份大小
    estimatedSize, err := bm.estimateBackupSize()
    if err != nil {
        return nil, fmt.Errorf("failed to estimate backup size: %w", err)
    }
    plan.EstimatedSize = estimatedSize

    // 4. 创建存储位置
    storageLocation, err := bm.backupStorageManager.CreateStorageLocation(config.Storage)
    if err != nil {
        return nil, fmt.Errorf("failed to create storage location: %w", err)
    }
    plan.StorageLocation = storageLocation

    return plan, nil
}

// 执行备份
func (bm *BackupManager) ExecuteBackup(plan *BackupPlan) (*BackupResult, error) {
    // 1. 创建备份作业
    backupJob := bm.createBackupJob(plan)

    // 2. 验证前置条件
    if err := bm.validatePrerequisites(backupJob); err != nil {
        return nil, fmt.Errorf("prerequisites validation failed: %w", err)
    }

    // 3. 执行备份
    backupResult, err := bm.backupExecutor.Execute(backupJob)
    if err != nil {
        return nil, fmt.Errorf("backup execution failed: %w", err)
    }

    // 4. 验证备份
    if err := bm.backupValidator.ValidateBackup(backupResult); err != nil {
        return nil, fmt.Errorf("backup validation failed: %w", err)
    }

    // 5. 更新备份计划状态
    plan.BackupJobs = append(plan.BackupJobs, backupJob)
    plan.Statistics.TotalBackups++
    plan.Statistics.LastBackupTime = time.Now()
    plan.Statistics.TotalSize += backupResult.Size

    // 6. 清理过期备份
    if err := bm.cleanupExpiredBackups(plan); err != nil {
        log.Warnf("Failed to cleanup expired backups: %v", err)
    }

    return backupResult, nil
}

// 备份执行器
type BackupExecutor struct {
    // 数据备份器
    dataBackuper *DataBackuper

    // 索引备份器
    indexBackuper *IndexBackuper

    // 元数据备份器
    metadataBackuper *MetadataBackuper

    // 配置备份器
    configBackuper *ConfigBackuper

    // 进度跟踪器
    progressTracker *ProgressTracker
}

// 执行备份作业
func (be *BackupExecutor) Execute(job *BackupJob) (*BackupResult, error) {
    result := &BackupResult{
        JobID:      job.ID,
        StartTime:  time.Now(),
        Status:     BackupStatusRunning,
        Components: make([]*BackupComponentResult, 0),
    }

    // 1. 创建临时目录
    tempDir, err := be.createTempDirectory()
    if err != nil {
        return nil, fmt.Errorf("failed to create temp directory: %w", err)
    }
    defer os.RemoveAll(tempDir)

    // 2. 备份配置
    configResult, err := be.configBackuper.BackupConfig(tempDir)
    if err != nil {
        return nil, fmt.Errorf("config backup failed: %w", err)
    }
    result.Components = append(result.Components, configResult)

    // 3. 备份元数据
    metadataResult, err := be.metadataBackuper.BackupMetadata(tempDir)
    if err != nil {
        return nil, fmt.Errorf("metadata backup failed: %w", err)
    }
    result.Components = append(result.Components, metadataResult)

    // 4. 备份索引
    indexResult, err := be.indexBackuper.BackupIndexes(tempDir, job.Plan.Config.Indexes)
    if err != nil {
        return nil, fmt.Errorf("index backup failed: %w", err)
    }
    result.Components = append(result.Components, indexResult)

    // 5. 备份数据
    dataResult, err := be.dataBackuper.BackupData(tempDir, job.Plan.Config.DataScope)
    if err != nil {
        return nil, fmt.Errorf("data backup failed: %w", err)
    }
    result.Components = append(result.Components, dataResult)

    // 6. 压缩备份文件
    if err := be.compressBackup(tempDir, result); err != nil {
        return nil, fmt.Errorf("backup compression failed: %w", err)
    }

    // 7. 上传到存储
    if err := be.uploadToStorage(result, job.Plan.StorageLocation); err != nil {
        return nil, fmt.Errorf("storage upload failed: %w", err)
    }

    // 8. 更新结果
    result.EndTime = time.Now()
    result.Status = BackupStatusCompleted
    result.Duration = result.EndTime.Sub(result.StartTime)

    return result, nil
}

// 数据备份器
type DataBackuper struct {
    // 数据连接器
    dataConnector *DataConnector

    // 数据导出器
    dataExporter *DataExporter

    // 增量备份管理器
    incrementalBackupManager *IncrementalBackupManager
}

// 备份数据
func (db *DataBackuper) BackupData(outputDir string, dataScope *DataScope) (*BackupComponentResult, error) {
    result := &BackupComponentResult{
        Component: "data",
        StartTime: time.Now(),
        Status:    BackupStatusRunning,
    }

    // 1. 连接数据源
    connection, err := db.dataConnector.Connect()
    if err != nil {
        return nil, fmt.Errorf("failed to connect to data source: %w", err)
    }
    defer connection.Close()

    // 2. 导出数据
    exportResult, err := db.dataExporter.ExportData(connection, outputDir, dataScope)
    if err != nil {
        return nil, fmt.Errorf("failed to export data: %w", err)
    }

    // 3. 创建备份清单
    manifest := &BackupManifest{
        Component:    "data",
        Version:      "1.0",
        CreatedAt:    time.Now(),
        DataScope:    dataScope,
        Files:        exportResult.Files,
        TotalSize:    exportResult.TotalSize,
        Checksum:     exportResult.Checksum,
    }

    // 4. 保存清单
    if err := db.saveManifest(outputDir, manifest); err != nil {
        return nil, fmt.Errorf("failed to save manifest: %w", err)
    }

    // 5. 更新结果
    result.EndTime = time.Now()
    result.Status = BackupStatusCompleted
    result.Duration = result.EndTime.Sub(result.StartTime)
    result.Size = exportResult.TotalSize
    result.Files = exportResult.Files
    result.Checksum = exportResult.Checksum

    return result, nil
}

// 恢复执行器
type RestoreExecutor struct {
    // 数据恢复器
    dataRestorer *DataRestorer

    // 索引恢复器
    indexRestorer *IndexRestorer

    // 元数据恢复器
    metadataRestorer *MetadataRestorer

    // 配置恢复器
    configRestorer *ConfigRestorer

    // 验证器
    restoreValidator *RestoreValidator
}

// 执行恢复
func (re *RestoreExecutor) ExecuteRestore(backupID string, target *RestoreTarget) (*RestoreResult, error) {
    result := &RestoreResult{
        BackupID:   backupID,
        StartTime: time.Now(),
        Status:     RestoreStatusRunning,
        Target:     target,
    }

    // 1. 下载备份文件
    backupFiles, err := re.downloadBackup(backupID)
    if err != nil {
        return nil, fmt.Errorf("failed to download backup: %w", err)
    }

    // 2. 验证备份完整性
    if err := re.validateBackupIntegrity(backupFiles); err != nil {
        return nil, fmt.Errorf("backup integrity validation failed: %w", err)
    }

    // 3. 解压备份文件
    if err := re.extractBackupFiles(backupFiles); err != nil {
        return nil, fmt.Errorf("failed to extract backup files: %w", err)
    }

    // 4. 恢复配置
    if err := re.configRestorer.RestoreConfig(target); err != nil {
        return nil, fmt.Errorf("config restore failed: %w", err)
    }

    // 5. 恢复元数据
    if err := re.metadataRestorer.RestoreMetadata(target); err != nil {
        return nil, fmt.Errorf("metadata restore failed: %w", err)
    }

    // 6. 恢复索引
    if err := re.indexRestorer.RestoreIndexes(target); err != nil {
        return nil, fmt.Errorf("index restore failed: %w", err)
    }

    // 7. 恢复数据
    if err := re.dataRestorer.RestoreData(target); err != nil {
        return nil, fmt.Errorf("data restore failed: %w", err)
    }

    // 8. 验证恢复结果
    if err := re.restoreValidator.ValidateRestore(result); err != nil {
        return nil, fmt.Errorf("restore validation failed: %w", err)
    }

    // 9. 更新结果
    result.EndTime = time.Now()
    result.Status = RestoreStatusCompleted
    result.Duration = result.EndTime.Sub(result.StartTime)

    return result, nil
}
```

## 安全配置

### 1. 访问控制

```go
// 安全管理器
type SecurityManager struct {
    // 认证管理器
    authenticationManager *AuthenticationManager

    // 授权管理器
    authorizationManager *AuthorizationManager

    // 加密管理器
    encryptionManager *EncryptionManager

    // 审计管理器
    auditManager *AuditManager

    // 网络安全管理器
    networkSecurityManager *NetworkSecurityManager
}

// 认证配置
type AuthenticationConfig struct {
    // 认证方式
    Method AuthenticationMethod

    // JWT 配置
    JWT *JWTConfig

    // OAuth 配置
    OAuth *OAuthConfig

    // LDAP 配置
    LDAP *LDAPConfig

    // 多因素认证配置
    MFA *MFAConfig
}

// 认证方式
type AuthenticationMethod string

const (
    // 用户名密码
    UsernamePassword AuthenticationMethod = "username_password"

    // JWT Token
    JWT AuthenticationMethod = "jwt"

    // OAuth 2.0
    OAuth AuthenticationMethod = "oauth2"

    // LDAP
    LDAP AuthenticationMethod = "ldap"

    // 证书认证
    Certificate AuthenticationMethod = "certificate"
)

// 创建认证配置
func (sm *SecurityManager) CreateAuthenticationConfig(config *AuthenticationConfig) error {
    // 1. 验证配置参数
    if err := sm.validateAuthenticationConfig(config); err != nil {
        return fmt.Errorf("authentication config validation failed: %w", err)
    }

    // 2. 创建认证后端
    authBackend, err := sm.createAuthenticationBackend(config)
    if err != nil {
        return fmt.Errorf("failed to create authentication backend: %w", err)
    }

    // 3. 配置认证中间件
    if err := sm.configureAuthenticationMiddleware(authBackend); err != nil {
        return fmt.Errorf("failed to configure authentication middleware: %w", err)
    }

    // 4. 创建会话管理
    if err := sm.createSessionManager(config); err != nil {
        return fmt.Errorf("failed to create session manager: %w", err)
    }

    // 5. 配置令牌管理
    if err := sm.configureTokenManagement(config); err != nil {
        return fmt.Errorf("failed to configure token management: %w", err)
    }

    return nil
}

// JWT 配置
type JWTConfig struct {
    // 密钥
    Secret string

    // 签名算法
    SigningAlgorithm string

    // 令牌过期时间
    TokenExpiry time.Duration

    // 刷新令牌过期时间
    RefreshTokenExpiry time.Duration

    // 发行者
    Issuer string

    // 受众
    Audience string

    // 声明
    Claims map[string]interface{}
}

// 配置 JWT 认证
func (sm *SecurityManager) ConfigureJWTAuth(config *JWTConfig) error {
    // 1. 生成 JWT 密钥对
    if config.Secret == "" {
        secret, err := sm.generateJWTSecret()
        if err != nil {
            return fmt.Errorf("failed to generate JWT secret: %w", err)
        }
        config.Secret = secret
    }

    // 2. 创建 JWT 验证器
    jwtValidator := &JWTValidator{
        Secret:           []byte(config.Secret),
        SigningAlgorithm:  config.SigningAlgorithm,
        Issuer:          config.Issuer,
        Audience:        config.Audience,
        TokenExpiry:     config.TokenExpiry,
    }

    // 3. 创建 JWT 签发器
    jwtSigner := &JWTSigner{
        Secret:          []byte(config.Secret),
        SigningAlgorithm: config.SigningAlgorithm,
        Issuer:         config.Issuer,
        Audience:       config.Audience,
        TokenExpiry:    config.TokenExpiry,
    }

    // 4. 配置认证中间件
    authMiddleware := &JWTAuthenticationMiddleware{
        Validator: jwtValidator,
        Signer:    jwtSigner,
        TokenExtractor: &BearerTokenExtractor{},
        ErrorHandler: &DefaultAuthenticationErrorHandler{},
    }

    // 5. 注册中间件
    if err := sm.registerAuthenticationMiddleware(authMiddleware); err != nil {
        return fmt.Errorf("failed to register authentication middleware: %w", err)
    }

    return nil
}

// 授权配置
type AuthorizationConfig struct {
    // 授权模型
    Model AuthorizationModel

    // 角色定义
    Roles []*Role

    // 权限定义
    Permissions []*Permission

    // 策略定义
    Policies []*Policy

    // 属性定义
    Attributes []*Attribute
}

// 授权模型
type AuthorizationModel string

const (
    // RBAC (基于角色的访问控制)
    RBAC AuthorizationModel = "rbac"

    // ABAC (基于属性的访问控制)
    ABAC AuthorizationModel = "abac"

    // ReBAC (基于关系的访问控制)
    ReBAC AuthorizationModel = "rebac"

    // PBAC (基于策略的访问控制)
    PBAC AuthorizationModel = "pbac"
}

// 创建授权配置
func (sm *SecurityManager) CreateAuthorizationConfig(config *AuthorizationConfig) error {
    // 1. 验证配置参数
    if err := sm.validateAuthorizationConfig(config); err != nil {
        return fmt.Errorf("authorization config validation failed: %w", err)
    }

    // 2. 创建授权引擎
    authEngine, err := sm.createAuthorizationEngine(config)
    if err != nil {
        return fmt.Errorf("failed to create authorization engine: %w", err)
    }

    // 3. 创建策略存储
    policyStore, err := sm.createPolicyStore(config)
    if err != nil {
        return fmt.Errorf("failed to create policy store: %w", err)
    }

    // 4. 创建权限检查器
    permissionChecker := &PermissionChecker{
        AuthEngine:   authEngine,
        PolicyStore:  policyStore,
        CacheManager: sm.createPermissionCache(),
    }

    // 5. 配置授权中间件
    authzMiddleware := &AuthorizationMiddleware{
        PermissionChecker: permissionChecker,
        ErrorHandler:     &DefaultAuthorizationErrorHandler{},
    }

    // 6. 注册中间件
    if err := sm.registerAuthorizationMiddleware(authzMiddleware); err != nil {
        return fmt.Errorf("failed to register authorization middleware: %w", err)
    }

    return nil
}

// RBAC 策略存储
type RBACPolicyStore struct {
    // 角色存储
    roleStore RoleStore

    // 权限存储
    permissionStore PermissionStore

    // 策略存储
    policyStore PolicyStore

    // 缓存管理器
    cacheManager *CacheManager
}

// 角色存储接口
type RoleStore interface {
    // 创建角色
    CreateRole(role *Role) error

    // 获取角色
    GetRole(roleID string) (*Role, error)

    // 更新角色
    UpdateRole(role *Role) error

    // 删除角色
    DeleteRole(roleID string) error

    // 列出角色
    ListRoles() ([]*Role, error)

    // 分配权限到角色
    AssignPermissionToRole(roleID string, permissionID string) error

    // 从角色移除权限
    RemovePermissionFromRole(roleID string, permissionID string) error

    // 获取角色权限
    GetRolePermissions(roleID string) ([]*Permission, error)
}

// 角色定义
type Role struct {
    // 角色ID
    ID string

    // 角色名称
    Name string

    // 角色描述
    Description string

    // 角色权限
    Permissions []*Permission

    // 创建时间
    CreatedAt time.Time

    // 更新时间
    UpdatedAt time.Time

    // 元数据
    Metadata map[string]interface{}
}

// 权限定义
type Permission struct {
    // 权限ID
    ID string

    // 权限名称
    Name string

    // 权限描述
    Description string

    // 资源类型
    ResourceType string

    // 操作类型
    Action string

    // 权限条件
    Conditions []PermissionCondition

    // 创建时间
    CreatedAt time.Time

    // 更新时间
    UpdatedAt time.Time
}

// 权限条件
type PermissionCondition struct {
    // 条件类型
    Type string

    // 条件字段
    Field string

    // 条件操作符
    Operator string

    // 条件值
    Value interface{}
}
```

### 2. 网络安全

```go
// 网络安全管理器
type NetworkSecurityManager struct {
    // 防火墙管理器
    firewallManager *FirewallManager

    // TLS 配置管理器
    tlsConfigManager *TLSConfigManager

    // VPN 配置管理器
    vpnConfigManager *VPNConfigManager

    // 入侵检测系统
    intrusionDetectionSystem *IntrusionDetectionSystem

    // 网络策略管理器
    networkPolicyManager *NetworkPolicyManager
}

// 网络策略
type NetworkPolicy struct {
    // 策略名称
    Name string

    // 策略类型
    Type NetworkPolicyType

    // 规则
    Rules []*NetworkPolicyRule

    // 优先级
    Priority int

    // 状态
    Status NetworkPolicyStatus

    // 创建时间
    CreatedAt time.Time

    // 更新时间
    UpdatedAt time.Time
}

// 网络策略类型
type NetworkPolicyType string

const (
    // 入站策略
    Ingress NetworkPolicyType = "ingress"

    // 出站策略
    Egress NetworkPolicyType = "egress"

    // 负载均衡策略
    LoadBalancer NetworkPolicyType = "load_balancer"

    // DDoS 防护策略
    DDoSProtection NetworkPolicyType = "ddos_protection"
)

// 网络策略规则
type NetworkPolicyRule struct {
    // 规则ID
    ID string

    // 协议
    Protocol string

    // 源地址
    SourceAddresses []string

    // 源端口
    SourcePorts []int

    // 目标地址
    DestinationAddresses []string

    // 目标端口
    DestinationPorts []int

    // 动作
    Action NetworkPolicyAction

    // 日志记录
    Log bool

    // 优先级
    Priority int
}

// 网络策略动作
type NetworkPolicyAction string

const (
    // 允许
    Allow NetworkPolicyAction = "allow"

    // 拒绝
    Deny NetworkPolicyAction = "deny"

    // 丢弃
    Drop NetworkPolicyAction = "drop"

    // 重定向
    Redirect NetworkPolicyAction = "redirect"

    // 限流
    RateLimit NetworkPolicyAction = "rate_limit"
)

// 创建网络策略
func (nsm *NetworkSecurityManager) CreateNetworkPolicy(policy *NetworkPolicy) error {
    // 1. 验证策略配置
    if err := nsm.validateNetworkPolicy(policy); err != nil {
        return fmt.Errorf("network policy validation failed: %w", err)
    }

    // 2. 创建 Kubernetes NetworkPolicy
    if err := nsm.createKubernetesNetworkPolicy(policy); err != nil {
        return fmt.Errorf("failed to create kubernetes network policy: %w", err)
    }

    // 3. 创建云安全组规则
    if err := nsm.createSecurityGroupRules(policy); err != nil {
        return fmt.Errorf("failed to create security group rules: %w", err)
    }

    // 4. 配置防火墙规则
    if err := nsm.configureFirewallRules(policy); err != nil {
        return fmt.Errorf("failed to configure firewall rules: %w", err)
    }

    // 5. 配置负载均衡器
    if policy.Type == LoadBalancer {
        if err := nsm.configureLoadBalancer(policy); err != nil {
            return fmt.Errorf("failed to configure load balancer: %w", err)
        }
    }

    // 6. 启用 DDoS 防护
    if policy.Type == DDoSProtection {
        if err := nsm.enableDDoSProtection(policy); err != nil {
            return fmt.Errorf("failed to enable DDoS protection: %w", err)
        }
    }

    return nil
}

// TLS 配置管理器
type TLSConfigManager struct {
    // 证书管理器
    certificateManager *CertificateManager

    // 私钥管理器
    privateKeyManager *PrivateKeyManager

    // 证书颁发机构
    certificateAuthority *CertificateAuthority

    // 证书轮换管理器
    certificateRotationManager *CertificateRotationManager
}

// TLS 配置
type TLSConfig struct {
    // 证书文件路径
    CertFile string

    // 私钥文件路径
    KeyFile string

    // CA 证书文件路径
    CAFile string

    // 最小 TLS 版本
    MinVersion string

    // 密码套件
    CipherSuites []string

    // 证书轮换配置
    Rotation *CertificateRotationConfig
}

// 证书轮换配置
type CertificateRotationConfig struct {
    // 轮换间隔
    RotationInterval time.Duration

    // 提前过期时间
    ExpiryLeadTime time.Duration

    // 自动轮换
    AutoRotate bool

    // 通知配置
    NotificationConfig *NotificationConfig
}

// 配置 TLS
func (tcm *TLSConfigManager) ConfigureTLS(config *TLSConfig) error {
    // 1. 验证证书
    if err := tcm.validateCertificates(config); err != nil {
        return fmt.Errorf("certificate validation failed: %w", err)
    }

    // 2. 创建 TLS 上下文
    tlsConfig := &tls.Config{
        MinVersion:         tcm.parseTLSVersion(config.MinVersion),
        CipherSuites:       tcm.parseCipherSuites(config.CipherSuites),
        Certificates:       []tls.Certificate{tcm.loadCertificate(config)},
        ClientAuth:         tls.RequireAndVerifyClientCert,
        ClientCAs:          tcm.loadCA(config),
        InsecureSkipVerify: false,
    }

    // 3. 配置证书轮换
    if config.Rotation != nil && config.Rotation.AutoRotate {
        if err := tcm.certificateRotationManager.SetupRotation(config.Rotation); err != nil {
            return fmt.Errorf("failed to setup certificate rotation: %w", err)
        }
    }

    // 4. 应用 TLS 配置
    if err := tcm.applyTLSConfig(tlsConfig); err != nil {
        return fmt.Errorf("failed to apply TLS config: %w", err)
    }

    return nil
}

// 证书管理器
type CertificateManager struct {
    // 证书存储
    certificateStore *CertificateStore

    // 证书生成器
    certificateGenerator *CertificateGenerator

    // 证书验证器
    certificateValidator *CertificateValidator

    // 证书监控器
    certificateMonitor *CertificateMonitor
}

// 生成证书
func (cm *CertificateManager) GenerateCertificate(req *CertificateRequest) (*Certificate, error) {
    // 1. 验证请求
    if err := cm.validateCertificateRequest(req); err != nil {
        return nil, fmt.Errorf("certificate request validation failed: %w", err)
    }

    // 2. 生成私钥
    privateKey, err := cm.generatePrivateKey(req.KeyAlgorithm)
    if err != nil {
        return nil, fmt.Errorf("failed to generate private key: %w", err)
    }

    // 3. 创建证书签名请求
    csr, err := cm.createCertificateSigningRequest(req, privateKey)
    if err != nil {
        return nil, fmt.Errorf("failed to create CSR: %w", err)
    }

    // 4. 签发证书
    certificate, err := cm.certificateGenerator.SignCertificate(csr, req)
    if err != nil {
        return nil, fmt.Errorf("failed to sign certificate: %w", err)
    }

    // 5. 验证证书
    if err := cm.certificateValidator.ValidateCertificate(certificate); err != nil {
        return nil, fmt.Errorf("certificate validation failed: %w", err)
    }

    // 6. 存储证书
    if err := cm.certificateStore.StoreCertificate(certificate); err != nil {
        return nil, fmt.Errorf("failed to store certificate: %w", err)
    }

    return certificate, nil
}

// VPN 配置管理器
type VPNConfigManager struct {
    // VPN 客户端
    vpnClient *VPNClient

    // VPN 配置生成器
    vpnConfigGenerator *VPNConfigGenerator

    // VPN 连接管理器
    vpnConnectionManager *VPNConnectionManager

    // VPN 监控器
    vpnMonitor *VPNMonitor
}

// VPN 配置
type VPNConfig struct {
    // VPN 类型
    Type VPNType

    // 服务器地址
    ServerAddress string

    // 服务器端口
    ServerPort int

    // 认证方式
    Authentication *VPNAuthentication

    // 加密配置
    Encryption *VPNEncryption

    // 网络配置
    Network *VPNNetwork

    // 路由配置
    Routes []*VPNRoute
}

// VPN 类型
type VPNType string

const (
    // OpenVPN
    OpenVPN VPNType = "openvpn"

    // WireGuard
    WireGuard VPNType = "wireguard"

    // IPsec
    IPsec VPNType = "ipsec"

    // SSL VPN
    SSLVPN VPNType = "ssl_vpn"
)

// 配置 VPN
func (vcm *VPNConfigManager) ConfigureVPN(config *VPNConfig) error {
    // 1. 验证 VPN 配置
    if err := vcm.validateVPNConfig(config); err != nil {
        return fmt.Errorf("VPN config validation failed: %w", err)
    }

    // 2. 生成 VPN 配置文件
    vpnConfig, err := vcm.vpnConfigGenerator.GenerateConfig(config)
    if err != nil {
        return fmt.Errorf("failed to generate VPN config: %w", err)
    }

    // 3. 部署 VPN 服务器
    if err := vcm.deployVPNServer(vpnConfig); err != nil {
        return fmt.Errorf("failed to deploy VPN server: %w", err)
    }

    // 4. 配置客户端连接
    if err := vcm.configureClientConnections(vpnConfig); err != nil {
        return fmt.Errorf("failed to configure client connections: %w", err)
    }

    // 5. 设置网络路由
    if err := vcm.setupNetworkRoutes(config.Routes); err != nil {
        return fmt.Errorf("failed to setup network routes: %w", err)
    }

    // 6. 启动 VPN 监控
    if err := vcm.vpnMonitor.StartMonitoring(); err != nil {
        return fmt.Errorf("failed to start VPN monitoring: %w", err)
    }

    return nil
}
```

## 总结

Milvus 的生产环境部署是一个系统工程，需要综合考虑架构设计、资源配置、安全策略、监控告警、备份恢复等多个方面。通过本文介绍的部署最佳实践，您可以构建出稳定、高效、安全的向量数据库生产环境。

### 关键部署要点：

1. **环境规划**：根据业务需求合理规划部署环境和资源配置
2. **容器化部署**：使用 Kubernetes 和 Helm Chart 实现标准化部署
3. **高可用架构**：通过多副本、负载均衡、故障转移确保服务可用性
4. **安全配置**：实现身份认证、权限控制、网络安全等多层安全防护
5. **监控告警**：建立完善的监控体系和告警机制
6. **备份恢复**：制定完善的备份策略和恢复流程
7. **性能优化**：根据业务特点优化系统配置和参数

### 部署检查清单：

- [ ] 环境规划完成
- [ ] 容量规划评估
- [ ] Kubernetes 集群部署
- [ ] 依赖服务部署
- [ ] Milvus 集群部署
- [ ] 网络策略配置
- [ ] 安全策略配置
- [ ] 监控告警配置
- [ ] 备份恢复配置
- [ ] 性能测试验证
- [ ] 故障演练测试
- [ ] 文档和培训

通过遵循这些最佳实践，您可以在生产环境中成功部署和管理 Milvus 集群，为业务提供稳定、高效的向量数据库服务。

在下一篇文章中，我们将深入探讨 Milvus 的安全特性和访问控制机制，帮助您构建更加安全的向量数据库系统。

---

**作者简介**：本文作者专注于云原生技术和 DevOps 领域，拥有丰富的大规模系统部署和运维经验。

**相关链接**：
- [Milvus 部署文档](https://milvus.io/docs/install_standalone-docker.md)
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Helm Chart 最佳实践](https://helm.sh/docs/chart_best_practices/)
- [Docker 容器安全最佳实践](https://docs.docker.com/engine/security/)