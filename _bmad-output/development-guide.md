# KEDA 開發指南

**文件生成日期**: 2025-12-18

---

## 1. 開發環境設置

### 1.1 前置需求

| 工具 | 版本需求 | 用途 |
|------|---------|------|
| Go | 1.25.5+ | 主要開發語言 |
| Docker | 最新穩定版 | 容器建構 |
| kubectl | 與叢集版本相容 | Kubernetes 操作 |
| Make | - | 建構自動化 |
| Kind/Minikube | 最新版 | 本地 K8s 叢集 |
| golangci-lint | 2.7.1 | 程式碼 linting |
| pre-commit | 最新版 | Git hooks |

### 1.2 快速開始 (Dev Containers)

使用 VS Code Dev Containers 是最快的開發環境設置方式：

```bash
git clone git@github.com:kedacore/keda.git
cd keda
code .
# 在 VS Code 中: Ctrl+Shift+P -> Dev Containers: Reopen in container
make build
```

### 1.3 本地直接開發

```bash
# Clone repository
git clone git@github.com:kedacore/keda.git
cd keda

# 設置 Go 環境 (如果遇到 checksum 錯誤)
go env -w GOPROXY=https://proxy.golang.org,direct GOSUMDB=sum.golang.org

# 建構
make build
```

---

## 2. 建構與編譯

### 2.1 建構指令

```bash
# 完整建構 (包含 generate, fmt, vet)
make build

# 個別建構
make manager     # 建構 KEDA Operator
make adapter     # 建構 Metrics Adapter
make webhooks    # 建構 Admission Webhooks

# 更新依賴
make update-mod

# 生成程式碼
make generate    # DeepCopy, mocks, proto
make manifests   # CRD manifests
```

### 2.2 Docker 映像檔

```bash
# 建構 Docker images
make docker-build

# 建構並推送 (需設置 IMAGE_REGISTRY 和 IMAGE_REPO)
IMAGE_REGISTRY=docker.io IMAGE_REPO=yourusername make publish

# 多架構建構
make publish-multiarch
```

---

## 3. 本地運行與除錯

### 3.1 憑證設置

KEDA 需要 TLS 憑證才能運行。本地開發時需要手動生成：

```bash
mkdir -p /certs
openssl req -newkey rsa:2048 -subj '/CN=localhost' \
    -addext "subjectAltName = DNS:localhost" \
    -nodes -keyout /certs/tls.key -x509 -days 3650 -out /certs/tls.crt
cp /certs/tls.crt /certs/ca.crt
```

### 3.2 本地運行 Operator

```bash
# 1. 部署 CRDs 和 KEDA 基礎設施
make deploy

# 2. 縮放 keda-operator 至 0
kubectl scale deployment/keda-operator --replicas=0 -n keda

# 3. 本地運行 operator
make run ARGS="--zap-log-level=debug"
```

### 3.3 VS Code 除錯

在 `.vscode/launch.json` 中配置：

**Operator 除錯:**
```json
{
  "configurations": [
    {
      "name": "Launch operator",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/cmd/operator/main.go",
      "env": {
        "WATCH_NAMESPACE": "",
        "KEDA_CLUSTER_OBJECT_NAMESPACE": "keda"
      }
    }
  ]
}
```

**Metrics Server 除錯:**
```json
{
  "configurations": [
    {
      "name": "Launch metrics-server",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/cmd/adapter/main.go",
      "env": {
        "WATCH_NAMESPACE": "",
        "KEDA_CLUSTER_OBJECT_NAMESPACE": "keda"
      },
      "args": [
        "--authentication-kubeconfig=~/.kube/config",
        "--authentication-skip-lookup",
        "--authorization-kubeconfig=~/.kube/config",
        "--lister-kubeconfig=~/.kube/config",
        "--secure-port=6443",
        "--v=5"
      ]
    }
  ]
}
```

---

## 4. 測試

### 4.1 單元測試

```bash
# 運行單元測試
make test

# 帶 race 檢測的測試
make test-race
```

### 4.2 E2E 測試

```bash
# 本地 K8s 叢集運行 E2E 測試
make e2e-test-local

# 運行 smoke tests
make smoke-test

# 驗證 E2E 測試 regex
make e2e-regex-check
```

### 4.3 編寫 E2E 測試

每個 scaler 都必須有 E2E 測試。測試位於 `tests/scalers/` 目錄：

```go
// tests/scalers/example/example_test.go
//go:build e2e
// +build e2e

package example_test

import (
    "testing"
    . "github.com/kedacore/keda/v2/tests/utils"
)

func TestExampleScaler(t *testing.T) {
    // 測試邏輯
}
```

---

## 5. 程式碼品質

### 5.1 Linting

```bash
# 運行 golangci-lint
make golangci

# 手動安裝 linter
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin v2.7.1
```

### 5.2 Pre-commit Hooks

```bash
# 安裝 pre-commit
pip install pre-commit
# 或
brew install pre-commit

# 啟用 hooks
pre-commit install

# 手動運行
pre-commit run --all-files
```

### 5.3 驗證指令

```bash
make verify-manifests      # 驗證 CRD manifests
make clientset-verify      # 驗證生成的 client 程式碼
make verify-scalers-schema # 驗證 scaler schema
```

---

## 6. 新增 Scaler

### 6.1 Scaler 介面

每個 scaler 必須實作 `Scaler` 介面：

```go
type Scaler interface {
    // 取得指標值和活動狀態
    GetMetricsAndActivity(ctx context.Context, metricName string) ([]external_metrics.ExternalMetricValue, bool, error)

    // 取得用於 HPA 的 metric spec
    GetMetricSpecForScaling(ctx context.Context) []v2.MetricSpec

    // 清理資源
    Close(ctx context.Context) error
}
```

### 6.2 實作步驟

1. **建立 scaler 檔案**
   ```bash
   touch pkg/scalers/my_scaler.go
   touch pkg/scalers/my_scaler_test.go
   ```

2. **實作 scaler**
   ```go
   package scalers

   type myScaler struct {
       metricType v2.MetricTargetType
       metadata   *myScalerMetadata
       logger     logr.Logger
   }

   type myScalerMetadata struct {
       // 配置欄位
   }

   func NewMyScaler(config *scalersconfig.ScalerConfig) (Scaler, error) {
       // 解析 metadata
       // 驗證配置
       // 返回 scaler 實例
   }

   func (s *myScaler) GetMetricsAndActivity(ctx context.Context, metricName string) ([]external_metrics.ExternalMetricValue, bool, error) {
       // 實作指標取得邏輯
   }

   func (s *myScaler) GetMetricSpecForScaling(ctx context.Context) []v2.MetricSpec {
       // 定義 metric spec
   }

   func (s *myScaler) Close(ctx context.Context) error {
       // 清理資源
   }
   ```

3. **註冊 scaler** (`pkg/scaling/scalers_builder.go`)
   ```go
   case "my-scaler":
       return scalers.NewMyScaler(config)
   ```

4. **新增 E2E 測試** (`tests/scalers/my-scaler/`)

5. **更新文檔** (keda-docs repository)

### 6.3 最佳實踐

- 使用 `scalersconfig.ScalerConfig` 處理配置
- 使用 `InitializeLogger` 初始化 logger
- 實作適當的錯誤處理和重試邏輯
- 遵循 metrics 命名規範
- 支援 TLS 和認證選項

---

## 7. 版本控制

### 7.1 Commit 簽署

所有 commit 必須使用 DCO (Developer Certificate of Origin) 簽署：

```bash
git commit -s -m "feat: add new feature"
```

### 7.2 Changelog

每個變更都應該更新 `CHANGELOG.md`：

```markdown
## Unreleased

### New
- **General:** Description (#PR_NUMBER)
- **ScalerName:** Description (#ISSUE_NUMBER)

### Improvements
...

### Fixes
...
```

### 7.3 分支策略

- `main`: 主開發分支
- `release-x.y`: 版本發布分支
- `feat/*`, `fix/*`: 功能/修復分支

---

## 8. 部署

### 8.1 部署到 Kubernetes

```bash
# 使用預設映像檔
make deploy

# 使用自定義映像檔
IMAGE_REGISTRY=docker.io IMAGE_REPO=yourrepo make deploy
```

### 8.2 卸載

```bash
make undeploy
```

### 8.3 生成發布 YAML

```bash
# 生成 release manifests
VERSION=2.x.x make release

# 輸出檔案:
# - keda-2.x.x.yaml (完整部署)
# - keda-2.x.x-core.yaml (最小化部署)
# - keda-2.x.x-crds.yaml (僅 CRDs)
```

---

## 9. 環境變數

### 9.1 Operator 環境變數

| 變數 | 預設值 | 描述 |
|------|--------|------|
| WATCH_NAMESPACE | "" | 監控的 namespace（空=所有） |
| KEDA_CLUSTER_OBJECT_NAMESPACE | keda | KEDA 物件所在 namespace |
| KEDA_HTTP_DEFAULT_TIMEOUT | 3000 | HTTP 預設超時 (ms) |
| KEDA_SCALEDOBJECT_CTRL_MAX_RECONCILES | 5 | ScaledObject 最大並發 reconcile |
| KEDA_SCALEDJOB_CTRL_MAX_RECONCILES | 1 | ScaledJob 最大並發 reconcile |

### 9.2 日誌設置

| 參數 | 選項 | 預設 |
|------|------|------|
| --zap-log-level | debug, info, error | info |
| --zap-encoder | json, console | console |
| --zap-time-encoding | epoch, iso8601, rfc3339 | rfc3339 |

---

## 10. 故障排除

### 10.1 常見問題

**建構失敗 - checksum mismatch**
```bash
go env -w GOPROXY=https://proxy.golang.org,direct GOSUMDB=sum.golang.org
```

**CRD 驗證失敗**
```bash
make verify-manifests
make manifests
```

**E2E 測試失敗**
```bash
# 清理測試資源
make e2e-test-clean-crds
kubectl delete ns -l type=e2e
```

### 10.2 取得協助

- Slack: [#KEDA](https://kubernetes.slack.com/archives/CKZJ36A5D)
- GitHub Issues: [kedacore/keda](https://github.com/kedacore/keda/issues)
- Community Meetings: [keda.sh/community](https://keda.sh/community/)

---

## 相關文件

- [專案概述](./project-overview.md)
- [架構文件](./architecture.md)
- [原始碼結構](./source-tree-analysis.md)
