# KEDA 原始碼結構分析

**文件生成日期**: 2025-12-18

---

## 專案根目錄結構

```
keda/
├── apis/                      # CRD API 類型定義
│   ├── eventing/              # CloudEvents 相關 CRDs
│   │   └── v1alpha1/          # CloudEventSource, ClusterCloudEventSource
│   └── keda/                  # 核心 KEDA CRDs
│       └── v1alpha1/          # ScaledObject, ScaledJob, TriggerAuthentication
│
├── cmd/                       # 應用程式入口點
│   ├── adapter/               # Metrics Adapter 入口 (main.go)
│   ├── operator/              # KEDA Operator 入口 (main.go)
│   └── webhooks/              # Admission Webhooks 入口 (main.go)
│
├── config/                    # Kubernetes/Kustomize 配置
│   ├── crd/                   # CRD manifests
│   │   └── bases/             # 生成的 CRD YAML
│   ├── default/               # 預設 kustomize overlay
│   ├── e2e/                   # E2E 測試部署配置
│   ├── grafana/               # Grafana dashboard 配置
│   ├── manager/               # Operator deployment
│   ├── metrics-server/        # Metrics adapter deployment
│   ├── minimal/               # 最小化部署配置
│   ├── rbac/                  # RBAC 規則
│   ├── samples/               # 範例 CRD 資源
│   ├── service_account/       # Service Account 配置
│   └── webhooks/              # Webhook 配置
│
├── controllers/               # Kubernetes controllers
│   ├── eventing/              # CloudEventSource controllers
│   └── keda/                  # ScaledObject/Job controllers
│
├── hack/                      # 開發和建構腳本
│   ├── boilerplate.go.txt     # 程式碼生成的版權頭
│   ├── update-codegen.sh      # 更新生成的 client 程式碼
│   ├── verify-codegen.sh      # 驗證生成的程式碼
│   ├── verify-manifests.sh    # 驗證 CRD manifests
│   └── verify-schema.sh       # 驗證 scaler schema
│
├── images/                    # 圖片資源
│   └── logos/                 # KEDA logos
│
├── pkg/                       # 共享庫程式碼
│   ├── certificates/          # TLS 憑證管理
│   ├── common/                # 共用工具函式
│   ├── eventemitter/          # 事件發射器
│   ├── eventreason/           # 事件原因常數
│   ├── fallback/              # Fallback 邏輯
│   ├── generated/             # 生成的 client-go 程式碼
│   │   ├── clientset/         # Typed clientset
│   │   ├── informers/         # Shared informers
│   │   └── listers/           # Resource listers
│   ├── k8s/                   # Kubernetes 工具函式
│   ├── metricscollector/      # Prometheus/OTEL metrics 收集
│   ├── metricsservice/        # gRPC metrics 服務
│   ├── mock/                  # 測試用 mock 物件
│   │   ├── mock_client/       # controller-runtime client mock
│   │   ├── mock_eventemitter/ # EventEmitter mock
│   │   ├── mock_scale/        # Scale client mock
│   │   ├── mock_scaler/       # Scaler interface mock
│   │   ├── mock_scaling/      # ScaleHandler mock
│   │   └── mock_secretlister/ # SecretLister mock
│   ├── provider/              # External metrics provider
│   ├── scalers/               # 【核心】Scaler 實作
│   │   ├── authentication/    # 認證輔助函式
│   │   ├── aws/               # AWS 相關工具
│   │   ├── externalscaler/    # External scaler gRPC
│   │   ├── gcp/               # GCP 相關工具
│   │   ├── kafka/             # Kafka 相關工具
│   │   ├── liiklus/           # Liiklus gRPC
│   │   ├── openstack/         # OpenStack 相關工具
│   │   ├── scalersconfig/     # Scaler 配置類型
│   │   ├── splunk/            # Splunk 相關工具
│   │   ├── sumologic/         # SumoLogic 相關工具
│   │   └── *.go               # 各個 scaler 實作
│   ├── scaling/               # 擴縮邏輯核心
│   │   ├── cache/             # Scalers 快取
│   │   ├── executor/          # Scale executor
│   │   └── modifiers/         # Scaling modifiers
│   ├── status/                # 狀態管理
│   └── util/                  # 通用工具函式
│
├── schema/                    # JSON Schema
│   ├── generate_scaler_schema.go  # Schema 生成器
│   └── generated/             # 生成的 scaler schemas
│
├── tests/                     # E2E 測試
│   ├── scalers/               # 各 scaler 的 E2E 測試
│   │   ├── activemq/          # ActiveMQ scaler 測試
│   │   ├── aws-cloudwatch/    # AWS CloudWatch 測試
│   │   ├── azure-*            # Azure 相關測試
│   │   ├── kafka/             # Kafka 測試
│   │   ├── prometheus/        # Prometheus 測試
│   │   └── ...                # 其他 scaler 測試
│   ├── run-all.go             # 測試執行器
│   ├── utils/                 # 測試工具函式
│   └── README.md              # 測試說明
│
├── tools/                     # 開發工具
│
├── vendor/                    # Go module vendor 目錄
│
├── version/                   # 版本資訊
│   └── version.go             # 版本常數
│
├── .devcontainer/             # VS Code Dev Container 配置
├── .github/                   # GitHub 配置
│   ├── workflows/             # GitHub Actions workflows
│   └── ISSUE_TEMPLATE/        # Issue 範本
│
├── Dockerfile                 # KEDA Operator Dockerfile
├── Dockerfile.adapter         # Metrics Adapter Dockerfile
├── Dockerfile.webhooks        # Webhooks Dockerfile
├── go.mod                     # Go module 定義
├── go.sum                     # Go module checksum
├── Makefile                   # 建構腳本
├── PROJECT                    # kubebuilder project 配置
│
├── BUILD.md                   # 建構說明
├── CHANGELOG.md               # 變更日誌
├── CONTRIBUTING.md            # 貢獻指南
├── CREATE-NEW-SCALER.md       # 新增 Scaler 指南
├── LICENSE                    # Apache 2.0 授權
├── README.md                  # 專案 README
├── RELEASE-PROCESS.md         # 發布流程
├── ROADMAP.md                 # 路線圖
├── SECURITY.md                # 安全政策
└── TESTING.md                 # 測試策略
```

---

## 關鍵目錄說明

### `/apis/` - CRD 類型定義

包含所有 Kubernetes Custom Resource Definition 的 Go 類型定義：

```
apis/
├── eventing/v1alpha1/
│   ├── cloudeventsource_types.go        # CloudEventSource CRD
│   ├── clustercloudeventsource_types.go # ClusterCloudEventSource CRD
│   └── zz_generated.deepcopy.go         # 自動生成的 DeepCopy 方法
└── keda/v1alpha1/
    ├── scaledobject_types.go            # ScaledObject CRD
    ├── scaledjob_types.go               # ScaledJob CRD
    ├── triggerauthentication_types.go   # TriggerAuthentication CRD
    ├── condition_types.go               # Status conditions
    ├── groupversion_info.go             # API group/version info
    ├── *_webhook.go                     # Webhook handlers
    └── zz_generated.deepcopy.go
```

### `/cmd/` - 應用程式入口點

三個獨立的二進位程式：

| 目錄 | 說明 | 預設埠號 |
|------|------|----------|
| `cmd/operator/` | KEDA 主 operator | 8080 (metrics), 8081 (health), 9666 (gRPC) |
| `cmd/adapter/` | Kubernetes Metrics Server | 443 (HTTPS) |
| `cmd/webhooks/` | Admission webhooks | 9443 (webhook) |

### `/controllers/` - Reconcilers

Kubernetes controller 實作：

```
controllers/
├── eventing/
│   ├── cloudeventsource_controller.go        # CloudEventSource reconciler
│   └── clustercloudeventsource_controller.go
└── keda/
    ├── scaledobject_controller.go            # ScaledObject reconciler
    ├── scaledjob_controller.go               # ScaledJob reconciler
    ├── triggerauthentication_controller.go
    └── clustertriggerauthentication_controller.go
```

### `/pkg/scalers/` - Scaler 實作

這是 KEDA 的核心，包含 70+ 種 scaler 實作：

```
pkg/scalers/
├── scaler.go                    # Scaler 介面定義
├── scalers_builder.go           # Scaler 工廠
│
├── # Cloud Providers
├── aws_cloudwatch_scaler.go
├── aws_dynamodb_scaler.go
├── aws_kinesis_scaler.go
├── aws_sqs_scaler.go
├── azure_eventhub_scaler.go
├── azure_monitor_scaler.go
├── azure_pipelines_scaler.go
├── azure_servicebus_scaler.go
├── gcp_pubsub_scaler.go
├── gcp_stackdriver_scaler.go
│
├── # Message Queues
├── kafka_scaler.go
├── rabbitmq_scaler.go
├── nats_jetstream_scaler.go
├── redis_scaler.go
├── activemq_scaler.go
├── artemis_scaler.go
│
├── # Databases
├── postgresql_scaler.go
├── mysql_scaler.go
├── mssql_scaler.go
├── mongodb_scaler.go
├── cassandra_scaler.go
├── couchdb_scaler.go
├── elasticsearch_scaler.go
│
├── # Monitoring
├── prometheus_scaler.go
├── datadog_scaler.go
├── newrelic_scaler.go
├── dynatrace_scaler.go
│
├── # Kubernetes Native
├── kubernetes_workload_scaler.go
├── kubernetes_resource_scaler.go
├── cpu_memory_scaler.go
│
├── # Others
├── external_scaler.go           # 自定義 gRPC scaler
├── cron_scaler.go
├── github_runner_scaler.go
└── ...
```

### `/pkg/scaling/` - 擴縮邏輯

核心擴縮邏輯實作：

```
pkg/scaling/
├── scale_handler.go             # 主要 handler interface
├── scalers_builder.go           # Scaler 建構邏輯
├── cache/
│   └── scalers_cache.go         # Scalers 快取管理
├── executor/
│   └── scale_executor.go        # 實際執行擴縮
└── modifiers/
    └── modifiers.go             # 進階擴縮修改器
```

### `/config/` - Kubernetes 配置

使用 Kustomize 管理的 Kubernetes manifests：

```
config/
├── crd/bases/                   # 生成的 CRD YAML (由 controller-gen 產生)
├── default/                     # 完整部署配置
├── minimal/                     # 最小化部署 (無 webhooks)
├── e2e/                         # E2E 測試專用配置
├── manager/                     # Operator deployment
├── metrics-server/              # Metrics adapter deployment
├── webhooks/                    # Webhook deployment
├── rbac/                        # ClusterRole, ClusterRoleBinding
└── samples/                     # 範例 ScaledObject/Job
```

### `/tests/` - E2E 測試

每個 scaler 都有對應的 E2E 測試：

```
tests/
├── run-all.go                   # 測試執行入口
├── utils/
│   ├── setup_test.go           # 測試設置
│   └── helpers.go              # 測試輔助函式
└── scalers/
    ├── kafka/                  # Kafka E2E 測試
    │   └── kafka_test.go
    ├── prometheus/
    │   └── prometheus_test.go
    └── ...
```

---

## 程式碼生成

KEDA 使用以下工具生成程式碼：

| 工具 | 用途 | 輸出位置 |
|------|------|----------|
| controller-gen | CRD manifests, RBAC, DeepCopy | `config/crd/bases/`, `config/rbac/` |
| client-gen | Typed clientset | `pkg/generated/clientset/` |
| informer-gen | Shared informers | `pkg/generated/informers/` |
| lister-gen | Resource listers | `pkg/generated/listers/` |
| mockgen | Mock objects for testing | `pkg/mock/` |
| protoc | gRPC code | `pkg/scalers/externalscaler/`, `pkg/scalers/liiklus/` |

執行程式碼生成：
```bash
make generate          # 生成 DeepCopy, mock, proto
make manifests         # 生成 CRD manifests
make clientset-generate # 生成 client-go 程式碼
```

---

## CI/CD Workflows

`.github/workflows/` 包含以下主要 workflows：

| Workflow | 觸發條件 | 用途 |
|----------|---------|------|
| `main-build.yml` | Push to main | 建構並推送 Docker images |
| `pr-validation.yml` | PR opened/updated | 驗證 PR (lint, test, build) |
| `pr-e2e.yml` | Maintainer 觸發 | 執行 E2E 測試 |
| `nightly-e2e.yml` | 每日排程 | 完整 E2E 測試套件 |
| `release-build.yml` | Tag push | 發布版本建構 |
| `static-analysis-*.yml` | PR/Push | 程式碼靜態分析 |

---

## 相關文件

- [專案概述](./project-overview.md)
- [架構文件](./architecture.md)
- [開發指南](./development-guide.md)
