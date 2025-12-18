# KEDA 專案文件索引

**文件生成日期**: 2025-12-18
**專案**: KEDA (Kubernetes Event-Driven Autoscaling)
**掃描模式**: Exhaustive Scan

---

## 文件概覽

本文件集由 BMAD Document Project Workflow 自動生成，提供 KEDA 專案的全面技術文件，適用於 AI 輔助開發和人類開發者參考。

---

## 文件目錄

| 文件 | 描述 | 適用對象 |
|------|------|----------|
| [專案概述](./project-overview.md) | 專案介紹、技術堆疊摘要、快速開始 | 所有開發者 |
| [架構文件](./architecture.md) | 系統架構、組件設計、資料流 | 架構師、資深開發者 |
| [原始碼結構](./source-tree-analysis.md) | 目錄結構、模組說明 | 新加入的開發者 |
| [開發指南](./development-guide.md) | 開發環境設置、建構、測試 | 貢獻者 |
| [API 文件](./api-documentation.md) | CRD 規格、Scaler 配置 | 使用者、整合開發者 |

---

## 專案摘要

### 關鍵資訊

| 屬性 | 值 |
|------|-----|
| 專案類型 | Kubernetes Operator |
| 主要語言 | Go 1.25.5 |
| 框架 | controller-runtime / kubebuilder |
| 授權 | Apache License 2.0 |
| CNCF 狀態 | Graduated Project |

### 核心功能

- **事件驅動擴縮**: 支援 70+ 種事件來源
- **從零擴展**: 可將工作負載縮放至零副本
- **Kubernetes 原生**: 與 HPA 無縫整合
- **多雲支援**: AWS、Azure、GCP 等雲端服務

### 主要組件

| 組件 | 目的 |
|------|------|
| KEDA Operator | 監控 CRDs，管理擴縮邏輯 |
| Metrics Adapter | 向 HPA 提供外部指標 |
| Admission Webhooks | 驗證 CRD 配置 |

---

## 快速導航

### 我是新手開發者
1. 閱讀 [專案概述](./project-overview.md) 了解專案背景
2. 參考 [原始碼結構](./source-tree-analysis.md) 熟悉程式碼組織
3. 按照 [開發指南](./development-guide.md) 設置開發環境

### 我要貢獻程式碼
1. 查看 [開發指南](./development-guide.md) 中的開發流程
2. 如果要新增 Scaler，參考 [開發指南 - 新增 Scaler](./development-guide.md#6-新增-scaler)
3. 確保通過所有測試並更新 CHANGELOG

### 我要使用 KEDA
1. 參考 [API 文件](./api-documentation.md) 了解 CRD 規格
2. 查看常用 [Scaler Metadata](./api-documentation.md#3-常用-scaler-metadata) 配置

### 我要理解架構
1. 閱讀 [架構文件](./architecture.md) 了解系統設計
2. 查看 [組件架構](./architecture.md#22-組件架構) 圖示

---

## 文件元資料

```yaml
workflow_version: "1.2.0"
scan_type: "exhaustive"
project_type: "backend"
language: "Go"
framework: "controller-runtime"
generated_documents:
  - index.md
  - project-overview.md
  - architecture.md
  - source-tree-analysis.md
  - development-guide.md
  - api-documentation.md
  - project-scan-report.json
```

---

## 外部資源

- **官方網站**: https://keda.sh
- **GitHub Repository**: https://github.com/kedacore/keda
- **官方文件**: https://keda.sh/docs/
- **Slack**: #KEDA on Kubernetes Slack
- **社群會議**: https://keda.sh/community/
