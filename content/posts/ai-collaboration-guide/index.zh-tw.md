---
title: "AI 協作開發指南：Agent.md + Skills 實戰手冊"
date: 2026-03-31
draft: false
summary: "如何用 Agent.md（CLAUDE.md / AGENTS.md）與 Skills 提升 AI 協同開發效率：核心概念、meta-prompt、skill-creator，以及 Claude Code 與 Codex 的差異。"
tags: ["AI", "Claude Code", "Codex", "Skills"]
---

本指南整理我導入 AI 協同開發的實作經驗，幫助開發者快速理解 **Agent.md**（CLAUDE.md / AGENTS.md）與 **Skills** 這兩個功能，並善用 **meta-prompt**（讓 AI 幫你生成設定檔）把它們導入實際工作流。

---

## 1. 核心概念

> Agent.md = 專案永久記憶（always loaded）
>
> Skills = 按需求提取的專家技能（on-demand loaded）

|  | Agent.md | Skills |
|---|---|---|
| 載入時機 | 每次對話自動載入 | 呼叫時才載入 |
| 內容性質 | 專案架構、規範、指令 | 特定任務的 SOP |
| 更新頻率 | 隨專案演進 | 隨工作流程優化 |
| Token 影響 | 固定消耗 | 按需消耗 |

**跨平台使用**

- Agent.md（CLAUDE.md）：Claude Code、Codex 與多數工具都通用。
- SKILL.md 遵循 [Agent Skills Standard](https://agentskills.io)，30+ 工具通用（Claude Code、Codex、Cursor、Gemini CLI、GitHub Copilot 等）。一份 SKILL.md 寫一次，到處都能用。

---

## 2. Agent.md / CLAUDE.md 建置

### 2.1 結構骨架

一份開發專案的 CLAUDE.md 建議包含以下 section（每個 section 都有明確職責）：

```
## Conversation — 語言偏好與回覆風格
## Architecture Overview — 分層架構、目錄結構（表格）
## Technology Stack — 語言版本、框架、核心依賴
## Build and Development Commands — 建置/執行/測試指令（code block）
## Configuration — 環境配置、profiles、環境變數
## Development Notes — 架構慣例、命名規範、常見模式
## Key Business Domains — 核心業務領域
## Testing Strategy — 測試框架、Mock 策略、已知陷阱（❌/✅ 對比）
```

### 2.2 Meta-Prompt 模板

不需要手寫 CLAUDE.md！以下 prompt 可直接貼進 AI 工具，讓它掃描 codebase 自動產出（Claude Code 和 Codex 也都能用 `/init` 初始化專案）：

```
請你掃描整個 codebase，然後依照以下結構產出 markdown：

1. Conversation — 開發者的語言偏好、回覆風格
2. Architecture Overview — 專案類型、目錄結構、各模組用途（用表格）
3. Technology Stack — 語言版本、框架、主要依賴
4. Build and Development Commands — 建置、執行、測試的常用指令
5. Configuration — 環境配置方式（profiles、環境變數）
6. Development Notes — 架構慣例、命名規範、常見模式
7. Key Business Domains — 核心業務領域簡述
8. Testing Strategy — 測試框架、Mock 策略、已知陷阱

要求：
- 目錄結構用表格呈現（模組名 | 用途）
- 指令用 code block 加註解
- 陷阱/踩坑經驗用 ❌/✅ 對比格式
- 總長度控制在 200-400 行
- 只寫 AI 協作時真正需要的資訊，不要寫顯而易見的內容
```

### 2.3 重點呈現技巧

以一個 Java 微服務專案為例，幾個讓 CLAUDE.md 更好用的做法：

**Architecture Overview** — 用表格列出各微服務模組，一目瞭然：

```
| 服務名稱 | 用途 |
|---------|------|
| account-service | 帳戶管理（用戶帳戶、餘額、交易記錄） |
| order-service   | 訂單管理（訂單建立、狀態追蹤） |
| ...             | ... |
```

**Testing Strategy** — 用 ❌/✅ 對比記錄踩坑經驗，AI 每次都會遵守：

```java
// ❌ 錯誤：null 會被 MyBatis Plus 跳過，DB NOT NULL 欄位報錯
config.setRemark(null);

// ✅ 正確：空字串會被包含在 INSERT SQL 中
config.setRemark("");
```

**Build Commands** — code block 加註解，AI 需要建置時直接複製：

```bash
# Use project Java version
source ~/.sdkman/bin/sdkman-init.sh && sdk use java <version>

# Build all modules
mvn clean install

# Run specific test class
mvn test -Dtest=OrderControllerTest
```

---

## 3. Skill 建置

### 3.1 SKILL.md 結構

每個 Skill 是一個 `.claude/skills/{name}/SKILL.md`（或 `.codex/skills/{name}/SKILL.md`）檔案，結構很簡單：

```yaml
---
name: skill-name          # 必填：呼叫名稱
description: 一段描述...    # 必填：用途說明（也是自動觸發的比對依據）
---

# Skill 標題

這裡放 Markdown 格式的指令內容。
可以用 $ARGUMENTS 接收使用者傳入的參數。
```

可選：在 `references/` 子目錄放範本檔案，SKILL.md 內可引用。

**解剖範例 — 一個 code-review skill 的 frontmatter**：

```yaml
---
name: code-review
description: 對後端系統程式碼進行全面 Code Review，涵蓋架構規範符合性、
  命名規範、回傳包裝、安全性、業務邏輯、分散式事務、性能
  （N+1 查詢、快取）、Null 安全及測試覆蓋。當完成功能開發、準備提交 PR、
  或需要程式碼品質審查時使用，輸出 🔴🟡🟢 分級問題報告。
---
```

description 越精確，自動觸發就越準確。

**最簡範例 — grill-me skill**（僅數行 body）：

```yaml
---
name: grill-me
description: Interview the user relentlessly about a plan or design until
  reaching shared understanding. Use when user wants to stress-test a plan.
---

Interview me relentlessly about every aspect of this plan until we reach
a shared understanding. Walk down each branch of the design tree.
If a question can be answered by exploring the codebase, explore instead.
```

**進階範例 — 含 references 子目錄 + MCP 整合**：

```
.claude/skills/task-flow/
├── SKILL.md                              # 主指令（含多個 Step）
└── references/
    ├── api-change-template.md            # API 改動說明 HTML 範本
    └── test-report-template.md           # 自測報告 HTML 範本
```

SKILL.md 內可引用 `references/api-change-template.md` 來確保輸出格式一致——這是 Skill 管理範本檔的推薦做法。Skill 內整合的 MCP 工具也可替換成你習慣的工具（如 Notion、Obsidian 等支援 MCP 的工具）。

### 3.2 用 skill-creator 產生 Skill（6 步流程）

不需要手寫 SKILL.md！Claude Code 和 Codex 都內建了 `/skill-creator`：

1. 輸入 `/skill-creator`
2. 用自然語言描述需求（例如：「我需要一個 skill，在我提到任務編號時自動從任務系統讀取卡片、分析需求、產出 API 文件和測試報告」）
3. skill-creator 互動詢問細節（觸發條件？輸出格式？參數？）
4. 自動產出 `.claude/skills/{name}/SKILL.md`
5. 用 `/skill-name` 測試
6. 跨工具通用：同一份 SKILL.md 直接被 Codex、Cursor 等工具讀取

---

## 4. Claude Code 使用教學

### 4.1 設置 CLAUDE.md

- 放在**專案根目錄**，Claude Code 啟動時自動讀取。
- 支援**子目錄放額外 CLAUDE.md**（就近覆蓋），例如 `services/order-service/CLAUDE.md` 可放該服務特有的規範。
- 不需要任何額外設定，放了就生效。

### 4.2 用 /skill-creator 產生 skill

實際操作流程（對話範例）：

```
你：/skill-creator
AI：你想建立什麼類型的 skill？請描述它的用途。
你：我需要一個 code review skill，針對後端系統的架構規範、
    安全性、業務邏輯、分散式事務進行全面審查
AI：好的，我有幾個問題：
    1. 審查目標怎麼指定？路徑還是 PR？
    2. 輸出格式偏好？
    3. 需要哪些審查維度？
你：[回答...]
AI：[產出 .claude/skills/code-review/SKILL.md]
```

### 4.3 呼叫 Skill

**明確呼叫**（斜線命令）：

```
/code-review services/order-service
/write-test OrderServiceImpl.createOrder
/task-flow TASK-1234
```

**自動觸發**（根據 description 文字比對）：

- 在對話中提到任務編號 → 自動觸發 task-flow skill
- 提到「做個 code review」→ 自動觸發 code-review skill
- 提到「需要新 API」→ 自動觸發 new-api skill

**關鍵**：description 要寫清楚觸發條件和邊界。

### 4.4 一組常見的團隊 Skills 範例

實務上可以針對團隊的重複性工作各做一個 skill。以下是一組示意：

| Skill | 用途 | 觸發時機 |
|---|---|---|
| `code-review` | 全面 Code Review（多維度、🔴🟡🟢 分級報告） | 提交 PR 前 |
| `write-test` | 撰寫 JUnit 5 單元測試（Given/When/Then） | 需要新增測試 |
| `task-flow` | 任務完整工作流（分析→spec→API 文件→自測報告） | 提到任務編號 |
| `new-api` | 新增 API 端點骨架（Controller→Service→VO→DTO→ErrorCode） | 需要新 API |
| `new-service` | 建立新微服務完整骨架 | 需要新服務 |
| `new-feign-client` | 新增 Feign Client（自動查服務註冊中心，如 Eureka） | 跨服務呼叫 |
| `review-security` | 安全性專項審查（IDOR、敏感資源存取、Token 驗證） | 重要操作審查 |
| `review-transaction` | 分散式事務審查（事務邊界、undolog） | 多服務寫入 |
| `annual-review` | 全面安全掃描（OWASP Top 10、P0–P3 分級） | 安全審計 |
| `grill-me` | 計畫/設計深度訪談（逐層拆解決策樹） | 壓力測試方案 |

---

## 5. Codex 使用教學

### 5.1 差異對照表

| 項目 | Claude Code | Codex |
|---|---|---|
| 專案記憶檔 | `CLAUDE.md` | `AGENTS.md` |
| 層級覆蓋 | 子目錄 CLAUDE.md | 子目錄 AGENTS.md + override 檔 |
| 大小限制 | 無明確限制 | 32 KiB 預設 |
| Skills 格式 | `.claude/skills/*/SKILL.md` | 相同（通用標準）`.codex/skills/*/SKILL.md` |
| Skill 產生器 | `/skill-creator` 內建 | `$skill-creator` 內建 |
| 自動觸發設定 | description 文字比對 | `agents/openai.yaml` 可額外設定 |

### 5.2 要點

- **Skills 完全通用**，不需修改。用 Claude Code 的 `/skill-creator` 產生後，Codex 直接讀取同一份 SKILL.md。
- Codex 有 `agents/openai.yaml` 可設定 `allow_implicit_invocation: false` 關閉自動觸發。
- AGENTS.md 的內容結構跟 CLAUDE.md 類似，但兩者**不通用**（檔名不同、各自讀各自的）。
- 建議做法：維護一份 CLAUDE.md 作為主版本，需要時複製為 AGENTS.md 並微調。

---

## 6. 實戰案例：用 grill-me Skill 產生本指南

這份指南本身就是透過 `/grill-me` skill 產生的，以下是這次體驗的分析。

**流程回顧**：

1. 輸入 `/grill-me` + 大綱草稿
2. grill-me 逐層拆解決策分支（目標受眾 → 範圍 → 格式 → 內容深度）
3. 每輪 2–3 個問題，附帶建議答案
4. 所有分支解決後自動整合為完整計畫

**優點**：

- **強迫思考**：每個「我以為想清楚了」的部分都被追問出盲點（如 AGENTS.md vs CLAUDE.md 不通用）。
- **決策留痕**：一連串 Q&A 形成完整的設計決策記錄。
- **減少返工**：在動手寫之前就解決了格式、比重、範例呈現方式等爭議。
- **自動研究**：grill-me 過程中主動查了 agentskills.io 確認 Skills 通用性。

**缺點 / 注意事項**：

- **耗時**：數輪問答約 15–20 分鐘，簡單任務不需要這麼重的流程。
- **Token 消耗**：深度訪談 + 網路搜尋 + codebase 探索，Token 用量較高。
- **適用場景**：適合「方向不明確」或「影響範圍大」的任務；明確的 bug fix 或小功能直接做即可。

**結論**：grill-me 適合用在「寫之前需要想清楚」的場景，如技術文件、架構設計、新功能規劃。

---

## 7. 外部學習資源

- [Claude Code 官方文件](https://docs.anthropic.com/en/docs/claude-code)
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Codex 官方文件](https://developers.openai.com/codex)
- [Codex AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
- [Agent Skills 開放標準](https://agentskills.io)
- [The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
- [Claude Code Skills：讓 AI 變身專業工匠](https://kaochenlong.com/claude-code-skills)
