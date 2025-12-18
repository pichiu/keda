# KEDA 架構文件

**文件生成日期**: 2025-12-18
**專案版本**: v2.x (Go module: github.com/kedacore/keda/v2)

---

## 1. 執行摘要

KEDA (Kubernetes Event-Driven Autoscaling) 是一個 Kubernetes operator，擴展了 Kubernetes 的原生 HPA 能力，使其能夠基於事件驅動的指標進行擴縮容決策，包括從零擴展和縮放到零。

---

## 2. 系統架構

### 2.1 高層架構

```
                                    ┌─────────────────────────────────────┐
                                    │         Kubernetes Cluster          │
                                    │                                     │
┌─────────────────────┐            │  ┌─────────────────────────────────┐ │
│  Event Sources      │            │  │        KEDA Namespace           │ │
│  ─────────────────  │            │  │                                 │ │
│  - Message Queues   │            │  │  ┌─────────┐  ┌──────────────┐  │ │
│  - Databases        │◄───────────┼──┤  │ KEDA    │  │ Metrics      │  │ │
│  - Cloud Services   │            │  │  │ Operator│  │ Adapter      │  │ │
│  - Prometheus       │            │  │  └────┬────┘  └──────┬───────┘  │ │
│  - Custom (gRPC)    │            │  │       │              │          │ │
└─────────────────────┘            │  │       ▼              ▼          │ │
                                    │  │  ┌─────────────────────────┐   │ │
                                    │  │  │    Admission Webhooks   │   │ │
                                    │  │  └─────────────────────────┘   │ │
                                    │  └─────────────────────────────────┘ │
                                    │                  │                   │
                                    │                  ▼                   │
                                    │  ┌─────────────────────────────────┐ │
                                    │  │    HPA (auto-managed by KEDA)   │ │
                                    │  └────────────────┬────────────────┘ │
                                    │                   │                  │
                                    │                   ▼                  │
                                    │  ┌─────────────────────────────────┐ │
                                    │  │   Workloads (Deployment/       │ │
                                    │  │   StatefulSet/Jobs)            │ │
                                    │  └─────────────────────────────────┘ │
                                    └─────────────────────────────────────┘
```

### 2.2 組件架構

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              KEDA System                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                        cmd/operator/main.go                        │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │  │
│  │  │ ScaledObject     │  │ ScaledJob        │  │ TriggerAuth      │  │  │
│  │  │ Reconciler       │  │ Reconciler       │  │ Reconciler       │  │  │
│  │  └────────┬─────────┘  └────────┬─────────┘  └──────────────────┘  │  │
│  │           │                     │                                  │  │
│  │           ▼                     ▼                                  │  │
│  │  ┌────────────────────────────────────────────────────────────┐    │  │
│  │  │                    pkg/scaling/                             │    │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐                  │    │  │
│  │  │  │ ScaleHandler    │  │ ScaleExecutor   │                  │    │  │
│  │  │  └────────┬────────┘  └────────┬────────┘                  │    │  │
│  │  │           │                    │                           │    │  │
│  │  │           ▼                    ▼                           │    │  │
│  │  │  ┌─────────────────────────────────────────────────────┐   │    │  │
│  │  │  │                 pkg/scalers/                         │   │    │  │
│  │  │  │  70+ Scaler implementations                         │   │    │  │
│  │  │  │  (Kafka, RabbitMQ, Prometheus, AWS, Azure, GCP...)  │   │    │  │
│  │  │  └─────────────────────────────────────────────────────┘   │    │  │
│  │  └────────────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                       cmd/adapter/main.go                          │  │
│  │  ┌────────────────────────────────────────────────────────────┐    │  │
│  │  │  External Metrics API Server (k8s.io/metrics)              │    │  │
│  │  │  - Exposes metrics to HPA                                  │    │  │
│  │  │  - gRPC server for operator communication                  │    │  │
│  │  └────────────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                       cmd/webhooks/main.go                         │  │
│  │  ┌────────────────────────────────────────────────────────────┐    │  │
│  │  │  Admission Webhooks                                        │    │  │
│  │  │  - ValidatingWebhook for CRD validation                    │    │  │
│  │  └────────────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心模組

### 3.1 Controllers (`controllers/`)

| Controller | 檔案位置 | 描述 |
|------------|---------|------|
| ScaledObjectReconciler | `controllers/keda/scaledobject_controller.go` | 處理 ScaledObject CRD 的 reconciliation |
| ScaledJobReconciler | `controllers/keda/scaledjob_controller.go` | 處理 ScaledJob CRD 的 reconciliation |
| TriggerAuthenticationReconciler | `controllers/keda/triggerauthentication_controller.go` | 處理認證資源 |
| CloudEventSourceReconciler | `controllers/eventing/cloudeventsource_controller.go` | 處理 CloudEvents 來源 |

### 3.2 Scaling Engine (`pkg/scaling/`)

```go
// pkg/scaling/scale_handler.go
type ScaleHandler interface {
    HandleScalableObject(ctx context.Context, scalableObject interface{}) error
    DeleteScalableObject(ctx context.Context, scalableObject interface{}) error
    GetScalersCache() *cache.ScalersCache
}
```

關鍵職責：
- 管理 Scaler 實例的生命週期
- 處理指標收集和快取
- 與 HPA 協調擴縮決策

### 3.3 Scalers (`pkg/scalers/`)

每個 scaler 實作 `Scaler` 介面：

```go
// pkg/scalers/scaler.go
type Scaler interface {
    GetMetricsAndActivity(ctx context.Context, metricName string) ([]external_metrics.ExternalMetricValue, bool, error)
    GetMetricSpecForScaling(ctx context.Context) []v2.MetricSpec
    Close(ctx context.Context) error
}
```

**Push Scaler 擴展介面**:
```go
type PushScaler interface {
    Scaler
    Run(ctx context.Context, active chan<- bool)
}
```

### 3.4 APIs (`apis/`)

CRD 定義位於 `apis/keda/v1alpha1/` 和 `apis/eventing/v1alpha1/`：

| CRD | 檔案 | 短名稱 |
|-----|------|--------|
| ScaledObject | `scaledobject_types.go` | so |
| ScaledJob | `scaledjob_types.go` | sj |
| TriggerAuthentication | `triggerauthentication_types.go` | ta |
| ClusterTriggerAuthentication | `clustertriggerauthentication_types.go` | cta |

---

## 4. 資料流

### 4.1 ScaledObject 處理流程

```
1. User creates ScaledObject
          │
          ▼
2. ScaledObjectReconciler detects change
          │
          ▼
3. ScaleHandler initializes Scalers based on triggers
          │
          ▼
4. Scalers fetch metrics from external sources
          │
          ▼
5. KEDA creates/updates HPA with external metrics
          │
          ▼
6. HPA queries Metrics Adapter for current values
          │
          ▼
7. HPA makes scaling decision → scales Deployment/StatefulSet
```

### 4.2 指標服務架構

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   KEDA Operator │◄────►│  gRPC Server     │◄────►│  Metrics Adapter│
│                 │      │  (port 9666)     │      │                 │
│  - Scalers      │      │                  │      │  - External     │
│  - ScaleHandler │      │  pkg/            │      │    Metrics API  │
│                 │      │  metricsservice/ │      │                 │
└─────────────────┘      └──────────────────┘      └─────────────────┘
                                                            │
                                                            ▼
                                                   ┌─────────────────┐
                                                   │       HPA       │
                                                   │                 │
                                                   │ queries metrics │
                                                   │ for scaling     │
                                                   └─────────────────┘
```

---

## 5. 認證架構

### 5.1 認證類型

| 類型 | 描述 | 使用場景 |
|------|------|----------|
| TriggerAuthentication | Namespace 範圍的認證 | 單一 namespace 內共享認證 |
| ClusterTriggerAuthentication | Cluster 範圍的認證 | 跨 namespace 共享認證 |
| Pod Identity | Azure/AWS/GCP Workload Identity | 雲端原生認證 |

### 5.2 Secret 解析

```go
// pkg/scalers/authentication/authentication_types.go
type AuthClientSet struct {
    CoreV1Interface kubernetes.CoreV1Interface
    SecretLister    corev1listers.SecretLister
}
```

KEDA 使用 informer 快取 secrets 以提高效能，並限制只監控 KEDA namespace 中的 secrets。

---

## 6. 部署架構

### 6.1 Kubernetes 資源

```yaml
# keda namespace 中部署的資源
Namespace: keda
├── Deployment: keda-operator
│   └── Container: keda-operator
│       ├── Port: 8080 (metrics)
│       ├── Port: 8081 (health probes)
│       └── Port: 9666 (gRPC metrics service)
├── Deployment: keda-metrics-apiserver
│   └── Container: keda-metrics-apiserver
│       ├── Port: 443 (HTTPS)
│       └── Port: 8080 (metrics)
├── Deployment: keda-admission-webhooks
│   └── Container: keda-admission-webhooks
│       └── Port: 9443 (webhook)
├── Service: keda-operator
├── Service: keda-metrics-apiserver
├── Service: keda-admission-webhooks
├── APIService: v1beta1.external.metrics.k8s.io
├── ValidatingWebhookConfiguration: keda-admission
└── ClusterRole/ClusterRoleBinding: keda-operator
```

### 6.2 憑證管理

KEDA 支援自動憑證輪替：
- `--enable-cert-rotation=true` 啟用
- 憑證儲存在 `kedaorg-certs` secret
- 支援自定義 CA 目錄用於 scaler TLS 連線

---

## 7. 可觀測性

### 7.1 Prometheus Metrics

KEDA 暴露以下類型的 metrics：
- Operator metrics (scaling 活動, reconciliation 狀態)
- Scaler metrics (每個 trigger 的活動狀態)
- 錯誤和延遲 metrics

### 7.2 OpenTelemetry

透過 `--enable-opentelemetry-metrics=true` 啟用 OTLP 匯出。

### 7.3 Logging

使用 zap logger，支援：
- Log levels: debug, info, error
- Log formats: json, console
- Time encodings: epoch, iso8601, rfc3339

---

## 8. 擴展性

### 8.1 新增 Scaler

1. 在 `pkg/scalers/` 建立新的 scaler 檔案
2. 實作 `Scaler` 介面
3. 在 `pkg/scaling/scalers_builder.go` 註冊
4. 新增 e2e 測試在 `tests/scalers/`
5. 更新 keda-docs

### 8.2 External Scaler

使用 gRPC 介面實作自定義 scaler：

```protobuf
// pkg/scalers/externalscaler/externalscaler.proto
service ExternalScaler {
    rpc IsActive(ScaledObjectRef) returns (IsActiveResponse) {}
    rpc StreamIsActive(ScaledObjectRef) returns (stream IsActiveResponse) {}
    rpc GetMetricSpec(ScaledObjectRef) returns (GetMetricSpecResponse) {}
    rpc GetMetrics(GetMetricsRequest) returns (GetMetricsResponse) {}
}
```

---

## 9. 安全考量

### 9.1 RBAC

KEDA 需要以下權限：
- 讀取 ScaledObjects, ScaledJobs, TriggerAuthentications
- 建立/更新/刪除 HPAs
- 讀取 Secrets (僅 KEDA namespace)
- 讀取/更新 Deployments, StatefulSets, Jobs

### 9.2 Admission Control

ValidatingWebhook 確保：
- CRD 格式正確
- Replica count 邊界有效
- Trigger 配置正確

---

## 10. 效能考量

### 10.1 Polling Interval

- 預設: 30 秒
- 可在 ScaledObject 中自定義
- 影響對事件來源的查詢頻率

### 10.2 Cooldown Period

- 預設: 300 秒
- 防止頻繁的縮放操作
- `initialCooldownPeriod` 用於啟動保護

### 10.3 資源限制

- 可配置 `--kube-api-qps` 和 `--kube-api-burst`
- 支援回應壓縮控制

---

## 11. 相關文件

- [原始碼結構](./source-tree-analysis.md)
- [開發指南](./development-guide.md)
- [API 文件](./api-documentation.md)
