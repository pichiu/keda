# KEDA API 文件

**文件生成日期**: 2025-12-18
**API 版本**: keda.sh/v1alpha1, eventing.keda.sh/v1alpha1

---

## 1. 核心 CRDs

### 1.1 ScaledObject

ScaledObject 定義如何根據事件來源自動擴縮 Deployment、StatefulSet 或其他 scale subresource。

**API Group**: `keda.sh/v1alpha1`
**Kind**: `ScaledObject`
**Short Name**: `so`

#### Spec 欄位

| 欄位 | 類型 | 必填 | 預設值 | 描述 |
|------|------|------|--------|------|
| scaleTargetRef | ScaleTarget | 是 | - | 擴縮目標參照 |
| pollingInterval | *int32 | 否 | 30 | 輪詢間隔（秒） |
| cooldownPeriod | *int32 | 否 | 300 | 冷卻期（秒） |
| initialCooldownPeriod | *int32 | 否 | 0 | 初始冷卻期（秒） |
| idleReplicaCount | *int32 | 否 | - | 閒置時的副本數 |
| minReplicaCount | *int32 | 否 | 1 | 最小副本數 |
| maxReplicaCount | *int32 | 否 | 100 | 最大副本數 |
| triggers | []ScaleTriggers | 是 | - | 觸發器列表 |
| fallback | *Fallback | 否 | - | Fallback 配置 |
| advanced | *AdvancedConfig | 否 | - | 進階配置 |

#### ScaleTarget

```yaml
scaleTargetRef:
  name: my-deployment          # 目標資源名稱 (必填)
  apiVersion: apps/v1          # API 版本 (選填, 預設: apps/v1)
  kind: Deployment             # 資源類型 (選填, 預設: Deployment)
  envSourceContainerName: app  # 環境變數來源容器 (選填)
```

#### ScaleTriggers

```yaml
triggers:
  - type: prometheus           # Scaler 類型 (必填)
    name: my-trigger           # 觸發器名稱 (選填)
    useCachedMetrics: false    # 使用快取指標 (選填)
    metadata:                  # Scaler 專用配置 (必填)
      serverAddress: http://prometheus:9090
      metricName: http_requests
      threshold: "100"
      query: sum(rate(http_requests_total[2m]))
    authenticationRef:         # 認證參照 (選填)
      name: my-trigger-auth
      kind: TriggerAuthentication  # 或 ClusterTriggerAuthentication
    metricType: AverageValue   # 指標類型: Value | AverageValue
```

#### Fallback

```yaml
fallback:
  failureThreshold: 3          # 失敗閾值 (必填)
  replicas: 5                  # Fallback 副本數 (必填)
  behavior: static             # static | currentReplicas | currentReplicasIfHigher | currentReplicasIfLower
```

#### AdvancedConfig

```yaml
advanced:
  restoreToOriginalReplicaCount: true  # ScaledObject 刪除時還原副本數
  horizontalPodAutoscalerConfig:
    name: keda-hpa-my-deployment       # 自定義 HPA 名稱
    behavior:                          # HPA 行為設置
      scaleDown:
        stabilizationWindowSeconds: 300
        policies:
          - type: Percent
            value: 10
            periodSeconds: 60
  scalingModifiers:                    # 進階擴縮修改器
    formula: (trig1 + trig2) / 2
    target: "50"
    activationTarget: "10"
    metricType: AverageValue
```

#### 完整範例

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
    apiVersion: apps/v1
    kind: Deployment
  pollingInterval: 15
  cooldownPeriod: 300
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_total
        threshold: "100"
        query: sum(rate(http_requests_total{namespace="default"}[2m]))
  fallback:
    failureThreshold: 3
    replicas: 6
```

#### 重要 Annotations

| Annotation | 描述 |
|------------|------|
| `autoscaling.keda.sh/paused` | 暫停擴縮 (true/false) |
| `autoscaling.keda.sh/paused-replicas` | 暫停時的固定副本數 |
| `autoscaling.keda.sh/paused-scale-in` | 暫停縮容 |
| `autoscaling.keda.sh/paused-scale-out` | 暫停擴容 |
| `autoscaling.keda.sh/force-activation` | 強制啟用 |

---

### 1.2 ScaledJob

ScaledJob 定義如何根據事件來源自動擴縮 Kubernetes Jobs。

**API Group**: `keda.sh/v1alpha1`
**Kind**: `ScaledJob`
**Short Name**: `sj`

#### Spec 欄位

| 欄位 | 類型 | 必填 | 預設值 | 描述 |
|------|------|------|--------|------|
| jobTargetRef | batchv1.JobSpec | 是 | - | Job 規格 |
| pollingInterval | *int32 | 否 | 30 | 輪詢間隔（秒） |
| successfulJobsHistoryLimit | *int32 | 否 | 100 | 保留成功 Job 數量 |
| failedJobsHistoryLimit | *int32 | 否 | 100 | 保留失敗 Job 數量 |
| maxReplicaCount | *int32 | 否 | 100 | 最大並行 Job 數 |
| scalingStrategy | ScalingStrategy | 否 | default | 擴縮策略 |
| triggers | []ScaleTriggers | 是 | - | 觸發器列表 |

#### ScalingStrategy

```yaml
scalingStrategy:
  strategy: "default"              # default | custom | accurate
  customScalingQueueLengthDeduction: 1  # 自定義佇列長度扣除
  customScalingRunningJobPercentage: "0.5"  # 自定義運行 Job 百分比
  pendingPodConditions:            # 等待 Pod 條件
    - "Ready"
    - "PodScheduled"
  multipleScalersCalculation: "max"  # max | min | avg | sum
```

#### 完整範例

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: rabbitmq-consumer
  namespace: default
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    activeDeadlineSeconds: 600
    backoffLimit: 6
    template:
      spec:
        containers:
          - name: consumer
            image: my-consumer:latest
            env:
              - name: RABBITMQ_HOST
                value: rabbitmq
        restartPolicy: Never
  pollingInterval: 30
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  maxReplicaCount: 20
  scalingStrategy:
    strategy: default
  triggers:
    - type: rabbitmq
      metadata:
        host: amqp://rabbitmq:5672
        queueName: my-queue
        queueLength: "5"
```

---

### 1.3 TriggerAuthentication

TriggerAuthentication 提供 namespace 範圍的認證配置。

**API Group**: `keda.sh/v1alpha1`
**Kind**: `TriggerAuthentication`
**Short Name**: `ta`

#### Spec 欄位

| 欄位 | 類型 | 描述 |
|------|------|------|
| podIdentity | PodIdentityProvider | Pod Identity 配置 |
| secretTargetRef | []AuthSecretTargetRef | Secret 參照 |
| env | []AuthEnvironment | 環境變數參照 |
| hashiCorpVault | HashiCorpVault | HashiCorp Vault 配置 |
| azureKeyVault | AzureKeyVault | Azure Key Vault 配置 |
| awsSecretManager | AwsSecretManager | AWS Secrets Manager 配置 |
| gcpSecretManager | GCPSecretManager | GCP Secret Manager 配置 |

#### 認證方式

**1. Secret 參照**
```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: secret-auth
spec:
  secretTargetRef:
    - parameter: password
      name: my-secret
      key: password
```

**2. Pod Identity (Azure)**
```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-pod-identity
spec:
  podIdentity:
    provider: azure-workload
    identityId: <identity-client-id>
```

**3. Pod Identity (AWS)**
```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-pod-identity
spec:
  podIdentity:
    provider: aws
    roleArn: arn:aws:iam::123456789:role/my-role
```

**4. Pod Identity (GCP)**
```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: gcp-pod-identity
spec:
  podIdentity:
    provider: gcp
```

**5. HashiCorp Vault**
```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: vault-auth
spec:
  hashiCorpVault:
    address: https://vault.example.com
    authentication: token
    credential:
      token: <token>
    secrets:
      - parameter: password
        key: secret/data/myapp
        path: password
```

---

### 1.4 ClusterTriggerAuthentication

與 TriggerAuthentication 相同，但在 cluster 範圍。

**API Group**: `keda.sh/v1alpha1`
**Kind**: `ClusterTriggerAuthentication`
**Short Name**: `cta`

```yaml
apiVersion: keda.sh/v1alpha1
kind: ClusterTriggerAuthentication
metadata:
  name: cluster-wide-auth
spec:
  # 與 TriggerAuthentication 相同的 spec
```

---

## 2. Eventing CRDs

### 2.1 CloudEventSource

定義 CloudEvents 來源以發送擴縮事件。

**API Group**: `eventing.keda.sh/v1alpha1`
**Kind**: `CloudEventSource`

```yaml
apiVersion: eventing.keda.sh/v1alpha1
kind: CloudEventSource
metadata:
  name: my-cloudevents
  namespace: default
spec:
  clusterName: my-cluster
  destination:
    http:
      uri: http://cloudevents-sink.example.com
  eventSubscription:
    includedEventTypes:
      - keda.scaledobject.ready.v1
      - keda.scaledobject.failed.v1
```

### 2.2 ClusterCloudEventSource

Cluster 範圍的 CloudEventSource。

**API Group**: `eventing.keda.sh/v1alpha1`
**Kind**: `ClusterCloudEventSource`

---

## 3. 常用 Scaler Metadata

### 3.1 Prometheus

```yaml
type: prometheus
metadata:
  serverAddress: http://prometheus:9090
  query: sum(rate(http_requests_total[2m]))
  threshold: "100"
  activationThreshold: "5"
  namespace: default
  customHeaders: "X-Scope-OrgID=tenant1"
  ignoreNullValues: "true"
  unsafeSsl: "false"
```

### 3.2 Kafka

```yaml
type: kafka
metadata:
  bootstrapServers: kafka.example.com:9092
  consumerGroup: my-group
  topic: my-topic
  lagThreshold: "10"
  activationLagThreshold: "0"
  offsetResetPolicy: latest
  allowIdleConsumers: "false"
  scaleToZeroOnInvalidOffset: "false"
  version: "1.0.0"
```

### 3.3 RabbitMQ

```yaml
type: rabbitmq
metadata:
  host: amqp://user:password@rabbitmq:5672
  queueName: my-queue
  queueLength: "20"
  activationQueueLength: "0"
  mode: QueueLength  # QueueLength | MessageRate
  value: "20"
  protocol: amqp     # amqp | http
  vhostName: /
```

### 3.4 AWS SQS

```yaml
type: aws-sqs-queue
metadata:
  queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
  queueLength: "5"
  activationQueueLength: "0"
  awsRegion: us-east-1
  scaleOnInFlight: "true"
  scaleOnDelayed: "false"
```

### 3.5 Azure Service Bus

```yaml
type: azure-servicebus
metadata:
  queueName: my-queue
  namespace: my-servicebus
  messageCount: "5"
  activationMessageCount: "0"
  connectionFromEnv: SERVICE_BUS_CONNECTION
```

### 3.6 Kubernetes Workload

```yaml
type: kubernetes-workload
metadata:
  podSelector: 'app=my-app'
  value: "5"
  activationValue: "0"
```

### 3.7 Cron

```yaml
type: cron
metadata:
  timezone: Asia/Taipei
  start: 0 6 * * *
  end: 0 22 * * *
  desiredReplicas: "10"
```

---

## 4. Status 欄位

### 4.1 ScaledObject Status

```yaml
status:
  scaleTargetKind: Deployment
  scaleTargetGVKR:
    group: apps
    version: v1
    kind: Deployment
    resource: deployments
  originalReplicaCount: 3
  lastActiveTime: "2025-12-18T10:30:00Z"
  externalMetricNames:
    - s0-prometheus
  hpaName: keda-hpa-my-deployment
  triggersTypes: prometheus
  authenticationsTypes: ""
  conditions:
    - type: Ready
      status: "True"
      reason: ScaledObjectReady
    - type: Active
      status: "True"
      reason: ScalerActive
    - type: Fallback
      status: "False"
      reason: NoFallbackFound
    - type: Paused
      status: "False"
      reason: NotPaused
  health:
    s0-prometheus:
      numberOfFailures: 0
      status: Happy
```

### 4.2 Conditions

| Condition | 描述 |
|-----------|------|
| Ready | ScaledObject 是否就緒 |
| Active | 是否有活動的 scalers |
| Fallback | 是否處於 fallback 模式 |
| Paused | 是否被暫停 |

---

## 5. 相關文件

- [專案概述](./project-overview.md)
- [架構文件](./architecture.md)
- [開發指南](./development-guide.md)
- [官方文件](https://keda.sh/docs/)
