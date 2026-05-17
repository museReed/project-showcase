# 面試技術知識手冊 — TPM-AI @ Phison

> 以 muse-platform 為實例，涵蓋全棧系統設計 + AI 工程 + TPM 指標。
> 每個主題：30 秒概念 → 你的實作 → 設計決策 → 預期追問。

---

## 1. API 設計

### 30 秒概念
RESTful API 的核心是資源導向：用 URL 表示資源、HTTP method 表示動作、status code 表示結果。好的 API 設計讓消費者不需讀文件就能猜到用法。

### muse-platform 實作
**Ops Server（FastAPI）**：
```
GET  /health                          → 系統健康度 + catalog 驗證狀態
GET  /api/modules                     → 列出已掛載模組（從 muse.toml 自動發現）
GET  /api/features                    → Feature cards（含 actions, kinds, layer）
POST /api/qa/sessions                 → 建立 QA checklist session
PATCH /api/qa/marks/{mark_id}         → 勾選/取消 checklist item
POST /api/qa/sessions/{id}/mark-ready → 觸發 PR comment + commit status
```

**MCP Knowledge Server（stdio transport）**：
```
search_articles(query, sources?, min_score?, days?, limit)
get_article(source, id)
list_recent(sources?, days, category?, limit)
```

### 設計決策
| 決定 | 原因 |
|------|------|
| FastAPI 而非 Flask | 原生 async、自動 OpenAPI doc、Pydantic 驗證 |
| Auto-mount router | 新模組加 `muse.toml [router]` 就自動掛載，不改 app.py |
| MCP 用 stdio transport | Claude Desktop / Claude Code 原生支援，零網路設定 |

### 預期追問
- **Q: 如何處理 API versioning？**
  A: 目前 single-version（內部工具），但 URL prefix `/api/v1/` 已預留。若需 breaking change，新增 `/api/v2/` 並行跑，舊版 deprecation header 通知。

- **Q: 怎麼做 rate limiting？**
  A: 內部工具不需對外 rate limit。對 LLM provider 的 rate limit 在 adapter 層處理：捕獲 429 → exponential backoff → 觸發 fallback provider。

---

## 2. 資料庫設計

### 30 秒概念
Schema 設計的核心是正規化（避免冗餘）和查詢效能（索引策略）之間的平衡。在 OLTP 場景，先正規化再針對 hot path 反正規化。

### muse-platform 實作
**4 個 SQLite DB**，每個 collector 獨立：
- `ai-news.db` — AI 新聞文章
- `xhs.db` — 小紅書帖子（最複雜，三表結構）
- `videos.db` — YouTube 影片
- `tweets.db` — Twitter 推文

**XHS 三表設計**：
```
posts（原始資料）
  ├─ red_id (UK)        ← 平台原生 ID，防重複
  ├─ content_hash       ← SHA256(title+content)，跨來源去重
  └─ *_json fields      ← 半結構化資料用 JSON 欄位

post_enrichment（LLM 加工結果）
  ├─ llm_summary, personal_insights, expert_review
  └─ 與 posts 用 red_id JOIN

post_metadata（評分 + 分類）
  ├─ ai_value_score, muse_value_score, career_value_score
  └─ category, subcategory
```

**FTS5 全文搜尋 + CJK fallback**：
```sql
-- 先嘗試 FTS5
SELECT ... FROM posts_fts WHERE posts_fts MATCH ?
-- 若 0 結果（CJK 字元問題）→ fallback
SELECT ... FROM posts WHERE title LIKE ? OR content LIKE ?
```

### 設計決策
| 決定 | 原因 |
|------|------|
| SQLite 而非 PostgreSQL | 單人 IDP，不需 concurrent write；部署零依賴 |
| 三表拆分 | 原始資料與 LLM 加工分離，重新 enrich 不影響原始 |
| JSON 欄位 | author、engagement 結構頻繁變動，JSON 避免 migration |
| content_hash 索引 | O(1) 去重查詢，防止重複 LLM 呼叫浪費成本 |

### 預期追問
- **Q: 為什麼不用 PostgreSQL？**
  A: 單人專案、無 concurrent write 需求、SQLite 零部署成本。如果需要多用戶，遷移路徑明確：SQLAlchemy adapter pattern，換 DB engine 不改業務邏輯。

- **Q: JSON 欄位的查詢效能？**
  A: JSON 欄位不做 WHERE 條件，只做 SELECT 讀取。需篩選的欄位（score、category）獨立到 post_metadata 表並建索引。

---

## 3. LLM 工程 — Multi-Model Routing

### 30 秒概念
不同 LLM 任務對品質和成本的要求不同。把 task → model 的映射抽成可配置的 routing layer，就能根據任務特性選擇最適合的模型，同時內建 fallback 應對 rate limit。

### muse-platform 實作
**TOML 配置驅動**：
```toml
[defaults]
provider = "anthropic"
model = "claude-haiku-4-5"
max_tokens = 2048

[tasks.classify_xhs]
model = "claude-sonnet-4-6"      # 分類需要較強理解力
max_tokens = 1024

[tasks.expert_review]
model = "claude-opus-4-6"        # 專家審閱需要最強推理
max_tokens = 8192
fallback_provider = "deepseek"
fallback_model = "deepseek-reasoner"
```

**呼叫鏈**：
```
Feature.call(task="classify_xhs")
  → LLMClient.call()
    → RoutingConfig.resolve(task) → (provider, model, params)
      → AdapterFactory.get(provider)
        → AnthropicProvider.complete()
          → [429 RateLimit] → FallbackProvider.complete()
```

**Provider 實作**：
- 不用 Anthropic SDK，直接 `httpx` 打 API → 完全掌控 retry、timeout、cost tracking
- Exponential backoff: 2^attempt 秒，最多 3 次重試
- 每次呼叫記錄 input/output tokens → 自動算成本

**A/B 測試驗證**（`scripts/llm_ab_test_v2.py`）：
- 相同 prompt 分別打 Claude / DeepSeek
- 人工盲評品質 → 決定哪些 task 可以用便宜模型
- 結果：分類任務 DeepSeek 品質相當，月費降 83%

### 設計決策
| 決定 | 原因 |
|------|------|
| TOML 而非 DB | 宣告式、版本控制、人類可讀 |
| httpx 而非 SDK | 自訂 retry、cost tracking、request-ID 注入 |
| Task-based routing | 同一模組不同任務用不同模型（classify 用 Sonnet，enrich 用 Haiku）|
| Defaults 打包進 wheel | `importlib.resources` 載入，無外部依賴即可運行 |

### 預期追問
- **Q: 83% 成本降低是怎麼算的？**
  A: 用 A/B 測試框架比較 Claude vs DeepSeek 在各任務的品質（人工盲評），發現 classify/enrich 兩類任務 DeepSeek 品質足夠。這兩類佔 80% 以上的 LLM 呼叫量，改用 DeepSeek（價格約 Claude 的 1/10）→ 整體月費從 ~$60 降到 ~$10。

- **Q: Fallback 怎麼保證一致性？**
  A: Prompt 不變、output schema 不變，只有 provider 不同。分類結果用 JSON schema 驗證，不合格的重試一次。

---

## 4. 資料收集 Pipeline — 7 Phase 架構

### 30 秒概念
ETL pipeline 的關鍵是：每個階段明確分工、錯誤不傳播、全鏈路可追蹤。用 correlation ID 串聯所有階段，失敗時能精確定位是哪個 phase 出問題。

### muse-platform 實作
```
Phase 1 — Collect     : 從外部 API 抓取原始資料（paginated）
Phase 2 — Normalize   : 去重（SHA256 content_hash）、JSON 解析
Phase 3 — Store       : INSERT / UPDATE 到 SQLite
Phase 4 — Score       : 5 維度評分（ai_value, muse_value, career, project, social）
Phase 5 — Classify    : LLM 分類（category, impact, time_sensitivity）
Phase 6 — Enrich      : LLM 摘要 + 個人洞察 + 深度分析
Phase 7 — Expert Review: 4 個專家平行審閱（ThreadPoolExecutor）
```

**原子性保證**：
- 所有事件 buffer 在記憶體中
- Pipeline 結束時一次寫入 JSONL log 檔
- `run_summary` 永遠是第一行（不需額外 index 檔案）

**JSON 解析四層 fallback**（處理 LLM 輸出不穩定）：
1. 嘗試 Markdown fence 提取 (\`\`\`json...\`\`\`)
2. 直接 `json.loads()`
3. 提取第一個 `{...}` 子字串
4. 修復無效的 escape sequences
→ 全部失敗才回傳空 dict（logged at ERROR）

### 預期追問
- **Q: 為什麼 Expert Review 用 ThreadPoolExecutor 而非 asyncio？**
  A: LLM 呼叫底層是 sync httpx（非 async），用 ThreadPoolExecutor 可以 4 個專家同時跑，延遲從 4×T 降到 ~1×T。

- **Q: Pipeline 中間失敗怎麼辦？**
  A: 每個 phase 有 `phase_start/phase_end` 標記，失敗的 phase 記錄在 pipeline log。`pipeline_status` 欄位記錄進度（pending → classified → enriched），下次跑只處理未完成的。

---

## 5. 系統架構 — Clean Architecture

### 30 秒概念
Clean Architecture 的核心規則：依賴只能向內（向下）。外層可以依賴內層，反之不行。這讓核心業務邏輯不受 UI、DB、外部 API 變動影響。

### muse-platform 實作
```
Interfaces（FastAPI, Telegram, MCP, CLI）
     ↓ imports
Features（25+ collectors, agents, pipelines）
     ↓ imports
Platform（core, llm, collectors, notify, http_clients, catalog）
     ↓ imports
Infra（paths, secrets, scheduler, logging, permissions）
```

**強制執行**：
- CI 用 `deptry` 掃描 import，違反依賴規則 → build fail
- Features 之間禁止直接 import（透過 Platform 或 Interfaces router）
- Boundary modules（被 ≥2 個上游依賴的模組）有額外 hardening 規範

**模組自動發現**：
- 每個模組有 `muse.toml` manifest（聲明 layer、dependencies、router path）
- `muse_catalog` 掃描所有 `muse.toml` → 自動掛載 router、註冊 scheduler job、產生 UI card
- 新增模組不需改任何 central config

### 預期追問
- **Q: 41 個模組不會太碎嗎？**
  A: 每個模組是獨立可部署單元，有自己的 `pyproject.toml`、test suite、muse.toml。好處是：CI 只跑受影響的模組（dependency graph filter）、新模組零摩擦加入、單模組 bug 不影響其他。

- **Q: 跟 microservices 有什麼區別？**
  A: muse-platform 是 monorepo + modular monolith。所有模組共享同一個 process，透過 Python import 溝通（不走網路）。比 microservices 簡單但保持了模組隔離。如果未來需要拆成 service，模組邊界已經明確。

---

## 6. 監控與可觀測性

### 30 秒概念
可觀測性三支柱：Logs（事件）、Metrics（聚合數字）、Traces（請求鏈路）。在沒有重量級 APM 的情況下，用 correlation ID + 結構化日誌可以做到 80% 的追蹤能力。

### muse-platform 實作
**Correlation ID**（`contextvars.ContextVar`）：
- 格式：`run-<8 hex>`
- 在 `BaseCollector.run()` 入口分配
- 自動繼承到所有子 async task / thread
- 不需手動傳參，全鏈路自動附帶

**結構化 JSONL 日誌**：
```json
{
  "ts": "2026-05-15T14:30:45.123456+00:00",
  "level": "INFO",
  "module": "muse.collectors.red_collector",
  "correlation_id": "run-abc12345",
  "msg": "collected 42 items",
  "ctx": {"source": "xiaohongshu", "items_count": 42, "duration_s": 23.5}
}
```

**LLM 成本追蹤**（Observatory）：
```python
with track_llm_call("enrich", "claude-haiku-4-5") as tracker:
    response = client.messages.create(...)
    tracker.record(response.usage.input_tokens, response.usage.output_tokens)
# → 自動算成本、記錄延遲、寫入 llm-calls.jsonl
```

### 設計決策
| 決定 | 原因 |
|------|------|
| contextvars 而非 thread-local | 原生支援 asyncio 繼承 |
| JSONL 而非 OpenTelemetry | 零外部依賴、schema 自控、版本穩定 |
| 雙 handler（.log + .jsonl）| 人類 debug 用 .log，機器分析用 .jsonl |
| 觀測失敗靜默 | 觀測系統不能影響主要業務流程 |

---

## 7. 測試策略

### 30 秒概念
測試金字塔：大量 unit test（快、便宜）、少量 integration test（慢、貴）、極少 E2E test。對外部 API，用 cassette 錄製取代真實呼叫，CI 永遠不打外部 API。

### muse-platform 實作
**三層分級**：
| 層級 | 路徑 | 觸發時機 | 特性 |
|------|------|----------|------|
| Unit | `tests/unit/` | 每次 PR | Mock 隔離、< 1s per test |
| P0 Integration | `tests/integration/p0/` | 每次 PR | 核心路徑、真實 DB (in-memory) |
| P1 Integration | `tests/integration/p1/` | Tag `v*` release | 外部 API (VCR 錄製)、長時間 |

**VCR.py cassette**：
- 第一次跑：錄製真實 HTTP 交互到 YAML 檔
- 之後跑：replay 錄製內容（不打 API）
- CI 環境永遠用 replay mode

**CI 智慧範圍**（`dorny/paths-filter`）：
- 偵測哪些 package 被改動（含依賴圖）
- 只跑受影響的 package 測試
- 例：改 `muse_llm` → 跑 llm tests + 依賴 llm 的 collector tests

---

## 8. CI/CD Pipeline

### 30 秒概念
好的 CI/CD 是品質閘門：每一道 gate 驗證一個面向，全部通過才能合併。自動化越多，人類出錯機會越少。

### muse-platform 實作
**5 道 Merge Gate（全綠才能合併）**：
1. ✅ CI unit + P0 integration pass
2. ✅ Gemini AI Review 無 P0/P1 issue
3. ✅ Ops UI Mark Ready（QA checklist 簽核）
4. ✅ `muse catalog validate`（schema 驗證）
5. ✅ 影響文件在同一個 PR

**Workflow 並行防衝突**：
```yaml
concurrency:
  group: ci-tests-${{ github.head_ref }}
  cancel-in-progress: true
# 同 branch 新 push → 自動取消舊 workflow
```

**Release Flow**：
```
develop → tag v* → full integration test → release PR → deploy prod
```

---

## 9. Vector Database & Embeddings（知識準備）

### 30 秒概念
Vector DB 把文字/圖片轉成高維向量（embedding），用近似最近鄰搜尋（ANN）找語意相似的內容。跟傳統關鍵字搜尋（BM25/FTS5）的差別是：vector search 理解語意，「機器學習」能找到「deep learning」。

### 核心知識點

**Embedding 流程**：
```
原始文字 → Embedding Model（如 text-embedding-3-small）→ [0.012, -0.034, ...] (1536 維)
→ 存入 Vector DB → Query 時同樣轉 embedding → ANN 搜尋 → Top-K 結果
```

**主流 Vector DB 比較**：
| DB | 特性 | 適合場景 |
|----|------|----------|
| Pinecone | Managed、serverless | 快速上線、不想管 infra |
| Weaviate | 內建 hybrid search | 需要 keyword + vector 混合 |
| Qdrant | Rust 高效能、filter 強 | 需要複雜 metadata filter |
| ChromaDB | 輕量、Python native | 原型開發、單機場景 |
| pgvector | PostgreSQL 擴充 | 已有 PG、不想加新 DB |
| SQLite-vec | SQLite 擴充 | 極輕量、嵌入式 |

**RAG（Retrieval-Augmented Generation）架構**：
```
User Query
  → Embedding(query)
  → Vector Search(top_k=10)
  → Rerank(cross-encoder, top_k=3)
  → LLM(context=retrieved_docs, question=query)
  → Answer
```

**Chunking 策略**（影響 RAG 品質的關鍵）：
- Fixed size（500 tokens + 50 overlap）— 簡單但粗暴
- Semantic chunking — 按段落/標題分割
- Parent-child — 小 chunk 搜尋，返回大 chunk 給 LLM context

### muse-platform 的演進路徑
目前用 FTS5 全文搜尋（keyword-based），已規劃 vector 層：
```
現狀：FTS5 MATCH → LIKE fallback（處理 CJK）
未來：FTS5 + Vector Hybrid Search
  → 先 FTS5 粗篩（keyword recall）
  → 再 embedding similarity 精排（semantic rerank）
  → SQLite-vec 或 Qdrant（視規模決定）
```

### 預期追問
- **Q: FTS5 和 Vector Search 什麼時候該用哪個？**
  A: 精確搜尋（找特定名詞）→ FTS5 更準；語意搜尋（找相關概念）→ Vector 更好。實務上 hybrid 最強：FTS5 召回 + Vector rerank。

- **Q: Embedding 維度越高越好嗎？**
  A: 不是。高維度增加計算和儲存成本，且有 curse of dimensionality。1536（OpenAI）或 1024（Cohere）在多數場景已足夠。

- **Q: 怎麼評估 RAG 品質？**
  A: RAGAS 框架：Faithfulness（答案是否忠於 context）、Answer Relevancy（答案是否相關）、Context Precision（找到的文件是否精準）、Context Recall（是否漏掉重要文件）。

---

## 10. 前端經驗（不只 iOS）

### 30 秒概念
現代前端不只是寫 UI，還包括狀態管理、API 整合、效能優化、無障礙設計。Web 前端和 iOS 的核心差異是：Web 要處理更多瀏覽器差異、SEO、響應式設計。

### muse-platform 實作
- **Ops UI（Next.js）**：模組管理、QA checklist 簽核、feature card 瀏覽
- **Telegram Bot UI**：inline keyboard、callback query、chunk 訊息
- **Content Viewer**：Markdown 渲染、SVG 圖表嵌入
- **MCP 整合**：Claude Desktop 原生整合、FTS5 搜尋結果呈現

### iOS → Web 的技能遷移
| iOS 概念 | Web 對應 |
|----------|----------|
| UIKit / SwiftUI | React / Next.js |
| Core Data | IndexedDB / SQLite (WASM) |
| URLSession | fetch / axios |
| GCD / async-await | Promise / async-await |
| Auto Layout | CSS Flexbox / Grid |
| Combine | RxJS / React Query |

### 預期追問
- **Q: 你做 Web 前端多久了？**
  A: muse-platform 的 Ops UI 是我用 Next.js 做的全棧應用。核心是把 iOS 的架構思維（MVVM、狀態管理、離線優先）帶到 Web。技術細節不同，但設計模式相通。

---

## 11. Edge AI（對應 Phison aiDAPTIV+）

### 30 秒概念
Edge AI 是把 AI 推理從雲端搬到終端設備（SSD controller、手機、IoT）。好處是低延遲、隱私保護、離線運作。挑戰是模型壓縮、記憶體限制、功耗控制。

### 核心知識點

**模型優化技術**：
| 技術 | 原理 | 壓縮率 |
|------|------|--------|
| Quantization（量化）| FP32 → INT8/INT4 | 2-4× |
| Pruning（剪枝）| 移除低權重連接 | 2-10× |
| Knowledge Distillation | 大模型教小模型 | 模型相關 |
| LoRA / QLoRA | 只訓練低秩分解的權重 | 訓練成本降 |

**Phison aiDAPTIV+ 架構理解**：
- SSD controller 內建 AI 推理引擎
- 數據在儲存層直接處理，不需傳到 CPU/GPU
- 適合：即時資料過濾、異常偵測、邊緣推理
- 優勢：降低資料搬運延遲、省電、可處理敏感資料（不出設備）

**你可以連結的經驗**：
- muse-platform 的 multi-model routing 概念可以延伸到 edge：
  - 簡單任務 → edge model（on-SSD）
  - 複雜任務 → cloud model（fallback）
  - 跟你做的 task-based routing 一樣：根據任務複雜度選擇推理位置

### 預期追問
- **Q: Edge AI 最大的挑戰是什麼？**
  A: 記憶體和功耗限制。一個 7B 參數模型 FP16 需要 14GB，SSD controller 可能只有幾百 MB。所以量化（INT4 → ~3.5GB）和模型選型（選用 small model + fine-tune）是關鍵。

- **Q: 你怎麼看 on-SSD AI 的應用場景？**
  A: 三大方向：(1) 即時資料過濾/壓縮（減少 I/O 頻寬）、(2) 安全異常偵測（勒索軟體行為模式識別）、(3) 個人化推薦/搜尋（隱私不出設備）。Phison 的定位是把 SSD 從被動儲存變成智慧計算節點。

---

## 12. PM vs TPM vs AI-TPM 角色定位

### 三者職能比較

| 面向 | PM（Product Manager） | TPM（Technical PM） | AI-TPM |
|------|----------------------|---------------------|--------|
| **核心職責** | 定義「做什麼」— 產品願景、roadmap、用戶價值 | 定義「怎麼做」— 技術方案、跨團隊協調、交付節奏 | 「怎麼做 AI」— AI 系統設計、模型選型、資料策略、成本控制 |
| **日常關注** | 用戶研究、市場分析、feature prioritization | 架構設計、技術風險、依賴管理、release planning | 模型效能、推理延遲、資料品質、AI 倫理、成本優化 |
| **決策權** | 產品 scope、feature 取捨、launch timing | 技術選型、系統架構、開發流程、品質標準 | AI/ML 技術選型、模型部署策略、AI infra 架構 |
| **技術深度** | 理解技術可行性，不需寫 code | 能讀 code、做 design review、debug 跨系統問題 | 能評估模型、設計 prompt、理解 training/inference pipeline |
| **跨團隊協調** | PM ↔ Design ↔ Engineering | Engineering ↔ Infra ↔ QA ↔ Security | ML Engineering ↔ Data ↔ Platform ↔ Product |
| **成功指標** | DAU、retention、NPS、revenue | On-time delivery、system uptime、tech debt ratio | Model accuracy、inference cost、latency P99、data freshness |
| **典型交付物** | PRD、wireframe、user story | Tech spec、architecture diagram、project timeline | Model evaluation report、AI roadmap、cost analysis、A/B test plan |

### 為什麼 Phison 需要 AI-TPM 而非純 PM/TPM

```
PM 知道「用戶要什麼」但不懂 model serving
TPM 知道「系統怎麼跑」但不懂 prompt engineering
AI-TPM = TPM 技術功底 + AI/ML 領域知識 + 產品思維
```

**Phison aiDAPTIV+ 的特殊性**：
- 硬體（SSD controller）+ 軟體（firmware）+ AI（on-device inference）三方交會
- 需要能同時跟硬體團隊談 memory constraint、跟 ML 團隊談 quantization、跟產品團隊談 user scenario
- 純 PM 做不到技術深度，純 TPM 缺 AI 視角 → AI-TPM 是最適合的角色

### 你的定位（面試敘事）

> 我的背景是 iOS 工程師 → 技術主管 → 獨立建構 muse-platform（41 模組全棧系統）。
> 這讓我同時具備：
> - **TPM 的技術深度**：能讀 code、做 architecture review、設計 CI/CD pipeline
> - **AI 工程經驗**：multi-model routing、prompt 設計、LLM 成本優化（降 83%）、RAG pipeline
> - **產品思維**：從用戶需求反推技術方案（不是為了技術而技術）
>
> 在 Phison，我能把 aiDAPTIV+ 的技術能力翻譯成產品語言，
> 同時確保 AI 落地方案在技術上可行、成本上合理、時程上可控。

### 預期追問
- **Q: 你沒有硬體團隊管理經驗，怎麼跟 firmware 團隊合作？**
  A: 在 ByteDance 我跟 AI 團隊協作（需求收集 + workflow 設計），雖然不是硬體，但跨領域協作的方法相同：(1) 先學他們的語言（memory layout、DMA、NAND），(2) 用共同的介面定義合作邊界（API spec），(3) 定期 sync 確保雙方理解一致。我在 muse-platform 的模組化設計就是這種「定義清楚邊界、各自負責」的實踐。

- **Q: PM 和 TPM 意見不同時你怎麼處理？**
  A: 用數據和 prototype 說話。例如 PM 想要一個功能但工程預估 3 個月，我會：(1) 拆解成 MVP（2 週可交付的最小版本），(2) 用數據估算 MVP 能覆蓋多少用戶場景，(3) 讓 PM 決定是否 MVP 先上。muse-platform 的 A/B test 框架就是這種「用數據決策」的工具。

---

## 13. TPM 核心指標

### 業務指標（Business KPIs）

| 指標 | 定義 | 為什麼重要 | 目標範例 |
|------|------|-----------|----------|
| **Time-to-Market** | Feature 從 PRD 到上線的天數 | TPM 的核心交付價值 | < 2 週 / feature |
| **Feature Adoption Rate** | 上線後 30 天內活躍使用率 | 做出來要有人用 | > 60% |
| **Customer Satisfaction (CSAT)** | 用戶滿意度評分 | 產品品質的最終衡量 | > 4.2 / 5.0 |
| **Revenue Impact** | Feature 對營收的貢獻 | 商業價值對齊 | 可量化 ROI |
| **Stakeholder Alignment** | 跨部門需求衝突的解決率 | TPM 的協調能力 | 90%+ 首輪對齊 |
| **Roadmap Accuracy** | 計畫 vs 實際交付的吻合度 | 預測能力 | > 80% on-time |

### 技術指標（Engineering KPIs）

| 指標 | 定義 | 為什麼重要 | 目標範例 |
|------|------|-----------|----------|
| **Deployment Frequency** | 每週部署次數 | 持續交付能力 | ≥ 2x / week |
| **Lead Time for Changes** | Commit → Production 的時間 | 開發效率 | < 1 天 |
| **Change Failure Rate** | 部署後導致 incident 的比例 | 品質把關 | < 5% |
| **MTTR** | Mean Time to Recovery | 系統韌性 | < 1 小時 |
| **Test Coverage** | 程式碼覆蓋率 | 品質底線 | ≥ 70% (unit) |
| **Tech Debt Ratio** | 技術債佔總開發時間比例 | 長期健康度 | < 20% |
| **API Latency P99** | 99th percentile 回應時間 | 用戶體驗 | < 500ms |
| **LLM Cost per Request** | 每次 AI 推理的平均成本 | AI 產品的獨特指標 | 持續優化 |

### AI 產品特有指標

| 指標 | 定義 | 為什麼重要 |
|------|------|-----------|
| **Model Accuracy / F1** | 模型預測準確率 | AI 產品的核心品質 |
| **Inference Latency** | 模型推理延遲 | 用戶體驗 |
| **Token Cost Efficiency** | 每個有效輸出的 token 成本 | AI 的邊際成本控制 |
| **Hallucination Rate** | 模型產生錯誤資訊的比例 | 信任度 |
| **Fallback Trigger Rate** | 降級到 fallback model 的頻率 | 系統穩定性 |
| **Data Freshness** | 訓練/索引資料的更新頻率 | RAG 品質 |

### muse-platform 的指標實踐
```
✅ Deployment Frequency → 2-3x/week（auto-merge on PR pass）
✅ Lead Time → < 4hr（worktree → PR → auto-merge）
✅ Change Failure Rate → ~2%（5-gate merge protection）
✅ LLM Cost Tracking → Observatory 自動記錄每次呼叫成本
✅ Token Efficiency → A/B test 驗證 → routing config 優化
✅ Fallback Rate → routing log 可計算觸發率
```

### 預期追問
- **Q: TPM 怎麼跟工程團隊溝通這些指標？**
  A: Dashboard 可視化（不用工程師手動報告）、週會只看趨勢變化和 outlier、每個 sprint 的 retro 看哪些指標退步了。重點是指標要能 actionable，不是只看數字。

- **Q: 業務指標和技術指標衝突時怎麼辦？**
  A: 例如 Time-to-Market 壓力下 Tech Debt 上升。TPM 的角色是量化 trade-off：「如果現在不還債，3 個月後每個 feature 多花 30% 時間」。用數據說服 stakeholder 分配 20% 時間給 tech debt。

---

## 13. 面試快速回答框架

每個問題用 **STAR-T** 回答：

| 步驟 | 說明 | 時間 |
|------|------|------|
| **S**ituation | 背景（1 句） | 10 秒 |
| **T**ask | 你的職責（1 句） | 10 秒 |
| **A**ction | 你做了什麼（2-3 句，技術細節） | 30 秒 |
| **R**esult | 量化結果 | 10 秒 |
| **T**ie-back | 連結到 Phison 的需求 | 10 秒 |

**範例**：
> **Q: 你怎麼處理 AI 的成本問題？**
>
> S: muse-platform 月 LLM 費用持續增長，需要控制。
> T: 我負責設計成本優化方案。
> A: 建了 task-based routing（TOML 配置）+ A/B 測試框架（blind eval），發現分類任務 DeepSeek 品質等同 Claude，改用 fallback routing。
> R: 月費降 83%（$60→$10），品質不變。
> T: 這個 routing + A/B 測試的方法也適用於 Phison 的 edge AI 場景——在 cloud 和 on-device inference 之間做 task-based routing。
