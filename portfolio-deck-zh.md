# Portfolio Deck — 辛忠翰 (Reed)

## 從打造 App 到打造 AI 系統

> Senior iOS Engineer → AI Technical Program Manager

- 8 年以上規模化產品交付經驗：TikTok、Crypto.com、17Live、Pinkoi
- 從零打造 production AI 平台：17 個模組、4 個 LLM 供應商、24/7 運行
- 帶領 10+ 人跨職能團隊，橫跨新加坡、台灣、美國、英國時區

---

## 職涯軌跡

### 時間線

| 時期 | 角色 | 核心成就 |
|------|------|----------|
| 2017-2019 | iOS 工程師，外交部（貝里斯） | 跨文化協作，交付 3 個政府 App |
| 2019-2020 | iOS 工程師，Pinkoi | 登入系統 & 物流系統架構翻新 |
| 2021-2022 | Senior iOS，17Live | 直播間架構 MVC→MVVM-C，美國團隊 Sprint 完成率 60%→85% |
| 2023-2024 | Senior iOS，Crypto.com | Onboarding 模組，測試覆蓋率 42%→75% |
| 2024-2025 | Senior iOS，TikTok Live | Tech Lead（品質效率），Project Owner（10 人跨職能團隊） |
| 2025-至今 | AI 平台建構者 | Muse Platform — 獨立擔任 Architect + PM + SWE + QA |

### 規律

每一站都在往 **技術深度 × 交付領導力** 的交叉點靠近。TikTok 的 Project Owner 經驗是轉折點——我發現自己最擅長的是 *驅動技術專案從零到完成*。

---

## 為什麼是 AI TPM？

### AI 專案的真正挑戰

AI 交付的難點 **不在 model 本身**，而在於：

| 挑戰 | 意義 |
|------|------|
| **Pipeline 可靠性** | LLM 非確定性；需要重試、fallback、品質檢查機制 |
| **成本不可預測** | Token 用量波動大；需要 routing + 預算控制 |
| **品質閘門** | 如何大規模 QA AI 的產出？ |
| **交付節奏** | AI 實驗 ≠ 傳統 Sprint；需要不同的規劃方式 |
| **跨團隊依賴** | ML × Backend × Product × Data — 每個人說不同語言 |

### 我的優勢

我不是從管理角度「聽說」這些挑戰，而是作為 **builder** 親身經歷，並為每一個設計了解法。

---

## Muse Platform — 架構全景

### 四層 Clean Architecture

```
┌───────────────────────────────────────────────────────┐
│  INTERFACES（介面層）                                   │
│  Ops UI (Next.js) │ Telegram Agent │ CLI │ MCP Server │
├───────────────────────────────────────────────────────┤
│  FEATURES（功能層）— 17 個自動化 AI 模組                 │
│  內容採集 │ 個人生產力 │ 知識管理                        │
├───────────────────────────────────────────────────────┤
│  PLATFORM（平台層）— 共用函式庫                          │
│  LLM 路由 │ Catalog │ 通知 │ HTTP │ 採集器框架          │
├───────────────────────────────────────────────────────┤
│  INFRA（基礎設施層）                                    │
│  路徑 │ 密鑰 │ 日誌 │ 部署 │ 排程                      │
└───────────────────────────────────────────────────────┘
```

### 關鍵數字

| 指標 | 數值 |
|------|------|
| Feature 模組 | 17 個 |
| 內容來源 | 4 個（小紅書、YouTube、Twitter、RSS） |
| LLM 供應商 | 4 個（Anthropic、DeepSeek、Groq、Gemini） |
| 排程任務 | 每日 20+ 個 |
| 使用介面 | 4 種（Ops UI、Telegram、CLI、MCP） |
| 架構層數 | 4 層（CI 強制依賴方向） |

---

## 深入案例 #1 — LLM 成本優化

### 問題

Claude Agent SDK 每次呼叫 **~27,000 token overhead** = **83% 浪費在 system prompt**

### 做法

1. **棄用 SDK** → 直接使用 Anthropic Messages API
2. **Task-based model routing** — TOML config 將任務類型映射到最佳供應商
3. **A/B 測試框架** — 同時比較 4 種配置：
   - Baseline（純 Claude）
   - Current-prod（混合配置）
   - Groq-free（免費方案）
   - Mixed-optimal（最佳組合）

### 結果

**成本降低 83%** — 品質透過 A/B 測試驗證維持

### TPM 觀點

> 量化 trade-off → 讓數據驅動決策 → 驗證假設。
> 這就是 AI TPM 每天在做的事。

---

## 深入案例 #2 — 規模化並行交付

### 問題

17 個 feature 待遷移。序列開發 = **數週**。

### 做法

1. **6-wave 交付計畫**（A-G）：每 wave 3-5 個 features
2. **Git worktree 隔離**：3-5 個 Claude subagent 並行工作
3. **衝突解決 SOP**：`pyproject.toml` 是必現衝突點 — 有文件化的 merge protocol
4. **品質閘門不打折**：每個 agent 走完整 10 步開發儀式

### 結果

| 指標 | 改善前 | 改善後 |
|------|--------|--------|
| 每 wave 耗時 | ~27 分鐘 | ~9 分鐘 |
| 提升幅度 | — | **-66%** |
| 成功交付 | — | 17/17 |
| 生產事故 | — | 0 |

### TPM 觀點

> 就像同時管理 3-5 個 scrum team：
> 依賴映射 + 明確介面契約 + 衝突解決機制 + 不因速度而妥協的品質閘門。

---

## 深入案例 #3 — 平台工程

### 問題

17 個模組，每個都需要 routing、scheduling、monitoring、QA。**手動配置不 scale。**

### 解法：`muse.toml` Manifest 系統

```toml
[module]
layer = "features"
kind = ["job"]

[job]
schedule = "0 */6 * * *"
entry = "run_pipeline"

[verify]
status = "ok"
max_errors = 0
duration_max = "5m"
```

### 運作方式

```
muse.toml（每個模組一份）
    ↓ muse_catalog 讀取所有 manifest
    ↓ ops-server 自動掛載 router
    ↓ scheduler 自動註冊任務
    ↓ CLI 在 CI 自動驗證
    ↓ Ops UI 自動產生 feature card
```

**新增模組 = 寫一個 muse.toml。其他全部自動。**

### TPM 觀點

> IDP 核心價值 = **護欄 + 自助服務**。
> 團隊快速交付，同時維持品質邊界。
> 這就是 Google、Meta、Uber 的 platform team 在做的事——我從零打造了一個。

---

## 品質系統設計

### 多層品質閘門

```
第 1 層：本地
  └─ 4-agent 自動 review
     （代碼 / 測試 / 錯誤處理 / 註釋）

第 2 層：CI
  └─ Gemini 代碼 review + Ruff lint
     （required merge checks）

第 3 層：驗證
  └─ Log-Based Verification Gate
     （解析 JSONL logs → 驗證 pipeline 健康狀態）

第 4 層：人工
  └─ Ops UI QA Checklist
     （人工確認 → Mark Ready gate）

第 5 層：合併
  └─ 前 4 層全綠
     → auto-merge → branch 自動刪除
```

### 結果

- 17 個模組 **零生產事故**
- 品質和速度不是 trade-off — **前提是把閘門自動化**

---

## 銜接：TikTok Project Owner 經驗

### Interactive Creator Competition

- 跨職能團隊：**10 人**（PM、Designer、Backend、iOS、Android、QA）
- 帶動：每日送禮用戶數↑、追蹤轉換↑、創作者留存↑

### 學到什麼

| 經驗 | TPM 應用 |
|------|----------|
| 對齊 10 個人需要共同指標 | OKR 驅動的專案管理 |
| 技術決策直接影響時程 | TPM 必須理解工程 trade-off |
| AB testing 驗證一切 | 數據驅動決策 |
| 跨時區協調（新加坡/台灣） | 大規模利害關係人管理 |

### 17Live：美國團隊 Sprint 改善

- 問題：Sprint 完成率 **60%**（vs 台灣團隊 90%+）
- 方案：導入 Design Review + API Spec 對齊 + 輪班 Sync-up
- 結果：6 個 Sprint 內 **60% → 85%**

---

## 我能帶來什麼

### 能力矩陣

| iOS 工程經驗 | Muse Platform 經驗 | → AI TPM 價值 |
|---|---|---|
| Clean Architecture, MVVM-C | 四層系統設計 | 與工程師對話的技術可信度 |
| Project Owner（10 人團隊） | 6-wave 並行交付 | 規模化專案交付 |
| Sprint 60%→85% 改善 | Workflow gate、自動化 | 流程改善 |
| 效能優化（記憶體、動畫） | LLM 成本優化、A/B 測試 | 數據驅動決策 |
| 跨時區（SG/HK/UK/US） | 多 Agent 協調 | 利害關係人管理 |

---

## 關鍵差異化

### vs. PM 候選人
> 我能讀代碼、設計架構、量化技術 trade-off。

### vs. 工程師候選人
> 我帶過跨職能交付、設計過流程、優化過團隊速度。

### vs. 其他 AI TPM 候選人
> 我 **親手打造** 了一個 production AI 系統。
> 我對 AI 交付挑戰的理解來自第一手經驗，不是投影片。

### 一句話

> 「多數 AI TPM 候選人管理過 AI 專案。我 *build* 了一個——然後建了管理它的 platform。」

---

## 我在尋找的

- **技術深度受重視**的 AI 產品團隊
- **大規模交付 AI/ML features** 的機會
- 重視**系統性流程改善**的團隊
- 能讓我**銜接工程與產品**的環境

---

## 附錄：技術棧

| 類別 | 技術 |
|------|------|
| 語言 | Python 3.12+ |
| 套件管理 | uv（workspace monorepo） |
| Web 框架 | FastAPI |
| 前端 | Next.js（Ops UI） |
| LLM | Anthropic Messages API、DeepSeek、Groq、Gemini |
| 資料庫 | SQLite（per-feature，含 FTS5 全文搜尋） |
| 瀏覽器自動化 | Playwright + CDP |
| CI/CD | GitHub Actions（Gemini review + Ruff lint） |
| 排程 | launchd（macOS） |
| 通知 | Telegram Bot API |
| 測試 | pytest、VCR.py |
| 硬體 | Mac Mini M2 Pro 32GB（prod） |
