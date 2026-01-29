# GitLab AI Code Review 設定

本文檔說明如何在 GitLab 中整合 QodoAI 的 PR Agent 進行自動化 Code Review。

## 概述

使用 **PR Agent** (由 QodoAI 提供) 來自動分析 Merge Request，提供：
- 📝 MR 描述生成與總結
- 🔍 自動化代碼審查
- ✨ 代碼改進建議

## 架構

```
GitLab Merge Request
        ↓
  .gitlab-ci.yml
        ↓
  PR Agent Container (Qodo/pr-agent)
        ↓
  LLM Provider (gpt / gemini  / claude / 其他)
        ↓
  Comments 回傳至 MR
```

## 前置準備

### 1. 環境變數設定

在 GitLab 專案的 **Settings > CI/CD > Variables** 中添加：

| 變數名 | 說明 | 備註 |
|--------|------|------|
| `GITLAB_PERSONAL_ACCESS_TOKEN` | GitLab 個人訪問令牌 | 需要 `api` 和 `read_api` 權限 |
| `OPENAI_KEY` | OpenAI API 金鑰 | 用於 GPT 模型調用 |

**建立 GitLab 令牌步驟：**
1. 進入 **Settings > Access Tokens**
2. 建立新令牌，勾選：`api`, `read_api`, `write_repository`
3. 複製令牌到 `GITLAB_PERSONAL_ACCESS_TOKEN`

### 2. 依賴配置

確保專案根目錄包含以下文件：
- `.gitlab-ci.yml` - CI/CD 流程配置
- `.pr_agent.toml` - PR Agent 行為設定

⚠️ **重要提示：**
- `.pr_agent.toml` **必須放在專案根目錄**
- `.pr_agent.toml` **只能在 default 分支生效**

## 配置文件說明

📚 **配置參數完整列表：**

### .gitlab-ci.yml

```yaml
tests:
  stage: test
  image:
    name: codiumai/pr-agent:latest
    entrypoint: [""]
  script:
    - cd /app
    - echo "Running PR Agent action step"
    - export MR_URL="$CI_MERGE_REQUEST_PROJECT_URL/merge_requests/$CI_MERGE_REQUEST_IID"
    - echo "MR_URL=$MR_URL"
    - export gitlab__url=$CI_SERVER_PROTOCOL://$CI_SERVER_FQDN
    - export gitlab__PERSONAL_ACCESS_TOKEN=$GITLAB_PERSONAL_ACCESS_TOKEN
    - export config__git_provider="gitlab"
    - export openai__key=$OPENAI_KEY
    - python -m pr_agent.cli --pr_url="$MR_URL" describe
    - python -m pr_agent.cli --pr_url="$MR_URL" review
    - python -m pr_agent.cli --pr_url="$MR_URL" improve
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**配置說明：**

| 參數 | 說明 |
|------|------|
| `MR_URL` | Merge Request URL，PR Agent 需要此參數 |
| `gitlab__url` | GitLab 服務器地址 |
| `gitlab__PERSONAL_ACCESS_TOKEN` | GitLab API 認證 |
| `config__git_provider` | 配置 Git 提供商為 GitLab |
| `openai__key` | OpenAI API 金鑰 |

**執行命令：**
- `describe` - 生成 MR 描述和總結
- `review` - 執行代碼審查，找出潛在問題
- `improve` - 提供代碼改進建議

**觸發條件：**
- 僅在 Merge Request 事件時執行（`CI_PIPELINE_SOURCE == "merge_request_event"`）

### .pr_agent.toml

詳見 [PR Agent 配置選項以及預設](https://github.com/qodo-ai/pr-agent/blob/main/pr_agent/settings/configuration.toml)

```toml
[config]
model = "gpt-4o-mini-2024-07-18"
model_turbo = "gpt-4o-mini-2024-07-18"
fallback_models = ["gpt-4o-mini-2024-07-18", "gpt-4o-mini"]
response_language="zh-TW"

[ignore]
regex = [
    '^(?!src)',
]
```

**配置說明：**

| 區域 | 參數 | 說明 |
|------|------|------|
| `config` | `model` | 主要使用的 AI 模型 |
| | `model_turbo` | 快速任務使用的模型 |
| | `fallback_models` | 備用模型列表 |
| | `response_language` | 回應語言（繁體中文） |
| `ignore` | `regex` | 排除的文件正則表達式 |

**支援的模型列表：**
詳見 [PR Agent 支援的模型](https://github.com/qodo-ai/pr-agent/blob/main/pr_agent/algo/__init__.py)

**Ignore 規則說明：**
```regex
^(?!src)
```
- 排除除 `src` 資料夾外的所有文件
- 只有 `src` 目錄中的代碼會被 AI 審查

## .pr_agent.toml 配置層級與優先順序

PR Agent 支持多層級配置，優先級由高到低：

1. **環境變數配置** - Runner 中直接設定（優先級最高）
2. **本地配置** - 各專案根目錄的 `.pr_agent.toml`
3. **全域配置** - Group/Subgroup 級別的 `pr-agent-settings` 倉庫中的 `.pr_agent.toml`
4. **預設配置** - PR Agent 內置預設值（優先級最低）

### 環境變數配置

在 `.gitlab-ci.yml` 或 GitLab Variables 中直接設定 PR Agent 參數。環境變數優先級最高，會覆蓋所有配置文件設定。

**語法規則：** `config__<section>__<parameter>=value`

**常用環境變數設定範例：**

```yaml
script:
    # 模型配置
    - export config__model="gpt-4o-mini-2024-07-18"
    - export config__model_turbo="gpt-4o-mini-2024-07-18"
    - export config__response_language="zh-TW"
    
    # Ignore 規則 (排除非 src 資料夾)
    - export config__ignore__regex="^(?!src)"
    
    # API 和認證
    - export openai__key=$OPENAI_KEY
    - export gitlab__PERSONAL_ACCESS_TOKEN=$GITLAB_PERSONAL_ACCESS_TOKEN
    - export gitlab__url=$CI_SERVER_PROTOCOL://$CI_SERVER_FQDN
    - export config__git_provider="gitlab"
    
    # Review 工具特定設定
    - export pr_reviewer__extra_instructions="檢查代碼品質和安全性"
    - export pr_reviewer__num_code_suggestions=5
    
    # 執行 PR Agent
    - python -m pr_agent.cli --pr_url="$MR_URL" review
```
### GitLab 全域配置設置

在 GitLab **Group** 下建立全域配置倉庫，所有子倉庫會自動使用：

**步驟：**
1. 在 GitLab Group 下建立新專案：`pr-agent-settings`
2. 在該專案根目錄建立 `.pr_agent.toml`
3. 該 Group 下的所有專案都會自動繼承此配置

**架構示例：**
```
GitLab Group (mygroup)
├── pr-agent-settings/
│   └── .pr_agent.toml (全域配置)
├── project-A/
│   └── 自動使用全域配置
├── project-B/
│   └── 如有本地 .pr_agent.toml 會覆蓋全域配置
└── project-C/
    └── 自動使用全域配置
```

**⚠️ 重要：GitLab 子組層級查詢規則**

在多層子組結構中，PR Agent 只會在**直接上層子組**查找 `pr-agent-settings`，不會向上遞歸查找：

```
Organization (GitLab Instance)
├── Group A (第1層)
│   ├── Subgroup A1 (第2層)
│   │   ├── Subgroup A1-1 (第3層) ← 倉庫在這裡
│   │   │   └── my-project/
│   │   │
│   │   └── pr-agent-settings/ ✅ 只會查找這裡（上方1級）
│   │
│   └── pr-agent-settings/ ❌ 不會查找這裡（上方2級）
│
└── pr-agent-settings/ ❌ 不會查找這裡（上方3級）
```

## 配置文件說明

### .gitlab-ci.yml

```yaml
tests:
  stage: test
  image:
    name: codiumai/pr-agent:latest
    entrypoint: [""]
  script:
    - cd /app
    - echo "Running PR Agent action step"
    - export MR_URL="$CI_MERGE_REQUEST_PROJECT_URL/merge_requests/$CI_MERGE_REQUEST_IID"
    - echo "MR_URL=$MR_URL"
    - export gitlab__url=$CI_SERVER_PROTOCOL://$CI_SERVER_FQDN
    - export gitlab__PERSONAL_ACCESS_TOKEN=$GITLAB_PERSONAL_ACCESS_TOKEN
    - export config__git_provider="gitlab"
    - export openai__key=$OPENAI_KEY
    - python -m pr_agent.cli --pr_url="$MR_URL" describe
    - python -m pr_agent.cli --pr_url="$MR_URL" review
    - python -m pr_agent.cli --pr_url="$MR_URL" improve
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**配置說明：**

| 參數 | 說明 |
|------|------|
| `image` | 使用 CodiumAI 官方 Docker 鏡像 |
| `stage` | CI 流程階段（test） |
| `MR_URL` | Merge Request URL，PR Agent 需要此參數 |
| `gitlab__url` | GitLab 服務器地址 |
| `gitlab__PERSONAL_ACCESS_TOKEN` | GitLab API 認證 |
| `config__git_provider` | 配置 Git 提供商為 GitLab |
| `openai__key` | OpenAI API 金鑰 |

**執行命令：**
- `describe` - 生成 MR 描述和總結
- `review` - 執行代碼審查，找出潛在問題
- `improve` - 提供代碼改進建議

**觸發條件：**
- 僅在 Merge Request 事件時執行（`CI_PIPELINE_SOURCE == "merge_request_event"`）

### .pr_agent.toml

```toml
[config]
model = "gpt-4o-mini-2024-07-18"
model_turbo = "gpt-4o-mini-2024-07-18"
fallback_models = ["gpt-4o-mini-2024-07-18", "gpt-4o-mini"]
response_language="zh-TW"

[ignore]
regex = [
    '^(?!src)',
]
```

**配置說明：**

| 區域 | 參數 | 說明 |
|------|------|------|
| `config` | `model` | 主要使用的 AI 模型 |
| | `model_turbo` | 快速任務使用的模型 |
| | `fallback_models` | 備用模型列表 |
| | `response_language` | 回應語言（繁體中文） |
| `ignore` | `regex` | 排除的文件正則表達式 |

**Ignore 規則說明：**
```regex
^(?!src)
```
- 排除除 `src` 資料夾外的所有文件
- 只有 `src` 目錄中的代碼會被 AI 審查

## 工作流程

### 1. 建立 Merge Request
```bash
git checkout -b feature/my-feature
git commit -m "feat: 新增功能"
git push origin feature/my-feature
```

### 2. GitLab CI 自動觸發
- 推送後，GitLab 自動執行 `.gitlab-ci.yml`
- PR Agent 容器啟動並連接到 MR

### 3. AI 分析過程
```
Step 1: Describe
  └─ 自動生成 MR 描述和總結

Step 2: Review
  └─ 逐行代碼審查
  └─ 提出潛在問題和改進建議

Step 3: Improve
  └─ 提供具體的代碼優化建議
  └─ 提出最佳實踐
```

### 4. 查看結果
- 在 Merge Request **Comments** 中查看 AI 的審查意見
- 根據建議進行代碼修改
- 推送更新，CI 會再次執行


## 常用命令

| 命令 | 功能 |
|------|------|
| `describe` | 生成 MR 描述 |
| `review` | 代碼審查 |
| `improve` | 代碼改進建議 |

