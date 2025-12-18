# KEDA 專案概述

**文件生成日期**: 2025-12-18
**專案名稱**: KEDA (Kubernetes Event-Driven Autoscaling)
**Repository**: github.com/kedacore/keda

---

## 執行摘要

KEDA 是一個 CNCF 畢業專案，為 Kubernetes 提供事件驅動的自動擴縮容能力。它可以根據事件來源（如訊息佇列深度、資料庫查詢結果等）精細控制工作負載的自動擴縮，包括從零擴展和縮放到零。

### 關鍵特點

- **事件驅動擴縮**: 基於 70+ 種事件來源進行擴縮決策
- **從零擴展/縮放到零**: 支援將工作負載縮放至零副本以節省資源
- **Kubernetes 原生**: 與 HPA (Horizontal Pod Autoscaler) 無縫整合
- **無外部依賴**: 可在雲端和邊緣環境運行
- **CNCF 畢業專案**: 生產就緒的成熟專案

---

## 技術堆疊摘要

| 類別 | 技術 | 版本 |
|------|------|------|
| 主要語言 | Go | 1.25.5 |
| 框架 | controller-runtime / kubebuilder | v0.21.0 |
| 容器化 | Docker | Multi-arch (amd64, arm64, s390x) |
| Kubernetes | client-go, apimachinery | v0.33.5 |
| 建構工具 | Make, Kustomize | - |
| 測試框架 | Ginkgo/Gomega, gotestsum | v2.27.2 / v1.13.0 |
| CI/CD | GitHub Actions | 20+ workflows |
| 程式碼品質 | golangci-lint, pre-commit | v2.7.1 |
| Observability | Prometheus, OpenTelemetry | - |

---

## 架構類型

| 屬性 | 值 |
|------|-----|
| Repository 類型 | Monolith |
| 專案類型 | Backend (Kubernetes Operator) |
| 架構模式 | Kubernetes Operator Pattern |
| 授權 | Apache License 2.0 |

---

## 專案組件

KEDA 由三個主要組件組成：

### 1. KEDA Operator (`cmd/operator/`)
- 主要的 Kubernetes controller
- 監控 ScaledObject 和 ScaledJob CRDs
- 管理 HPA 生命週期
- 處理 scaler 的實例化和生命週期

### 2. Metrics Adapter (`cmd/adapter/`)
- 實作 Kubernetes External Metrics API
- 向 HPA 提供自定義指標
- 支援 gRPC 和 HTTP 端點

### 3. Admission Webhooks (`cmd/webhooks/`)
- 驗證 CRD 資源的正確性
- 確保配置符合預期格式
- 提供安全驗證層

---

## 核心 CRDs

| CRD | 描述 |
|-----|------|
| ScaledObject | 定義如何擴縮 Deployment/StatefulSet |
| ScaledJob | 定義如何擴縮 Kubernetes Jobs |
| TriggerAuthentication | 定義觸發器認證配置 |
| ClusterTriggerAuthentication | 叢集範圍的觸發器認證 |
| CloudEventSource | 定義 CloudEvents 來源 |

---

## Scalers 概覽

KEDA 支援 **70+ 種 scalers**，包括：

### 雲端服務
- AWS: SQS, Kinesis, CloudWatch, DynamoDB
- Azure: Service Bus, Event Hubs, Blob Storage, Monitor
- GCP: Pub/Sub, Stackdriver, Cloud Storage

### 訊息佇列
- Apache Kafka, RabbitMQ, NATS JetStream
- Redis, ActiveMQ, Artemis, Beanstalk

### 資料庫
- PostgreSQL, MySQL, MSSQL, MongoDB
- Cassandra, CouchDB, Elasticsearch

### 監控系統
- Prometheus, Datadog, New Relic
- Dynatrace, Splunk, SumoLogic

### 其他
- Kubernetes Workload/Resources
- External Scaler (自定義 gRPC)
- Cron, GitHub Runner 等

---

## 相關文件

| 文件 | 描述 |
|------|------|
| [架構文件](./architecture.md) | 詳細的系統架構說明 |
| [原始碼結構](./source-tree-analysis.md) | 目錄結構分析 |
| [開發指南](./development-guide.md) | 本地開發環境設置 |
| [API 文件](./api-documentation.md) | CRD 和 API 規格 |

---

## 快速開始

### 前置需求
- Go 1.25.5+
- Docker
- Kubernetes cluster (或 kind/minikube)
- kubectl
- Make

### 建構專案
```bash
git clone git@github.com:kedacore/keda.git
cd keda
make build
```

### 部署到 Kubernetes
```bash
make deploy
```

### 運行測試
```bash
make test
```

---

## 外部資源

- **官方網站**: https://keda.sh
- **GitHub**: https://github.com/kedacore/keda
- **文檔**: https://keda.sh/docs/
- **Slack**: #KEDA on Kubernetes Slack
