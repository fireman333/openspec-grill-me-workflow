---
name: spec
description: >
  跨 session 的 OpenSpec 生命週期 wrapper（vibe-coding 多 session 專案 / 長期 refactor / 自由開發）。
  5 個 gate：assess / init / retro（git history 回填 specs）/ resume / handoff。
  **底層執行**（propose / apply / archive）走 OpenSpec 原生 `/opsx:*` + `openspec` CLI，**本 skill 不重複實作**。
  **退讓**：有 research_plan.json 走 research-plan；有 01_protocol/ 走 ma-end-to-end。
  Trigger: `/spec`、「裝 OpenSpec」「起一個 spec」「spec 評估」「spec retro」「resume spec」「spec handoff」。
  SessionStart hook 偵測 cwd 自動建議 assess / resume；PreCompact hook 在已 init 專案建議 handoff。
---

# spec — OpenSpec 生命週期 Wrapper

> 原名 `spec-driven`，2026-04-21 改為 `spec`（消除 `/spec` 與 `/spec-driven` 雙 slash 入口的 confusion）。遷移史見 [MIGRATION.md](MIGRATION.md)。

## 設計哲學

**問題**：OpenSpec（Fission-AI）是成熟的 spec-driven development 工具，但有兩個痛點：
1. **沒有 gate** — 預設你已經決定要 spec-driven，會被 RA / 國考 / 教學等非 coding 專案誤觸發
2. **沒有 retro** — mature project 已有 git tag + release 歷史，從零起 propose 浪費

**解法**：`spec` skill 補這兩塊 + 整個 lifecycle 的 thin wrapper。**不重複 OpenSpec 已實現的功能**（propose / apply / archive 等），只做：

| 模式 | 角色 | 是否呼叫 OpenSpec |
|---|---|---|
| `assess` | Gate keeper：判斷該不該裝 OpenSpec | ❌ |
| `clarify` | Pre-init grill：呼叫 `grill-me` skill（Deep + software-project pack）寫 `openspec/project-clarifications.md` | ❌（純文件，不碰 OpenSpec CLI） |
| `init` | Bootstrap：包裝 `openspec init --tools claude` + 寫 project.md / config.yaml；偵測到 `project-clarifications.md` 自動預填 | ✅ |
| `retro` | 從 git history 回填初始 `openspec/specs/<domain>/spec.md` | ✅（寫 specs，不走 changes/） |
| `resume` | Session 開場：`openspec list` + `openspec status` 三句話總結 | ✅（read-only） |
| `handoff` | Session 結束前：`openspec validate` + 提醒 archive 已完成 changes | ✅（read-only） |

**核心鐵律**：
- 不重複實作 OpenSpec 已有功能。`propose / apply / archive` 一律走 `/opsx:*`
- 改動專案前必先確認 `openspec` CLI 安裝（若無則導向 `npm install -g @fission-ai/openspec@latest`）
- Curator 模式：所有 spec 寫入都要使用者確認，不自動 commit、不自動 archive
- 退讓邏輯：偵測到 `research_plan.json` / `01_protocol/+09_qa/` 一律不啟動，讓位給 statistics / MA workflow

---

## 觸發條件

| 場景 | 行為 |
|---|---|
| `/spec`（無參數）| 預設走 `assess`；若已偵測到 `openspec/` 走 `resume` |
| `/spec assess` | 跑 rubric 評估專案是否該裝 OpenSpec |
| `/spec clarify` | Pre-init grill：呼叫 `grill-me` Deep + software-project pack，寫 `openspec/project-clarifications.md`，供 init 預填 |
| `/spec init` | Bootstrap OpenSpec：跑 `openspec init --tools claude`、寫初始 project.md + config.yaml；偵測到 clarifications.md 自動預填 |
| `/spec retro` | Mature project（≥ 3 git tag）從 release notes / commit messages 回填 `openspec/specs/<domain>/` |
| `/spec resume` | Session 開場讀 OpenSpec 狀態 + context warm-up（讀 project.md / in-progress proposal+design / capability specs / 最近 archives） |
| `/spec handoff` | Session 結束前驗證、提醒 archive 完成的 changes |
| `/spec note "<text>"` | 即時決策捕捉：自動路由到 in-progress change 的 `design.md ## Decisions`，或 `openspec/decisions/<YYYY-MM-DD>.md` |
| `/spec clear` | Session 切換捷徑：pending decisions 檢查 → `/spec handoff` 流程 → 提示使用者手動 `/clear`，下次 `/spec resume` 接續 |
| 「寫 spec」「起一個 spec」 | → `assess` 或 `init`（依是否已有 openspec/ ） |
| 「先 grill 一下」「project 太模糊」「用 grill 釐清」「想先想清楚 audience / NFR / failure mode」 | → `clarify` |
| 「spec retro」「mature project 寫 spec」 | → `retro` |
| 「resume spec」「spec 接續」 | → `resume` |
| 「spec handoff」「spec 交接」 | → `handoff` |
| 「先記一下」「等等記下來」「決定是」「這個要記住」「待會別忘」「存一下決策」 | → `note` |
| 「切 session」「清 context」「換新 session」「spec clear」 | → `clear` |
| SessionStart hook 注入提醒 | 主模型在第一輪 response 主動提議對應模式 |

---

## Preflight：OpenSpec CLI 檢查

任何呼叫 `openspec` 的模式（init / retro / resume / handoff）執行前先跑：

```bash
which openspec || command -v openspec
openspec --version
```

若 CLI 不存在：
1. 告知使用者「OpenSpec 未安裝。安裝指令：`npm install -g @fission-ai/openspec@latest`（需 Node ≥ 20.19）」
2. 用 AskUserQuestion 問「要現在裝嗎？」
3. 不要 silent 跳過 — 使用者要知道 dependency 缺失

---

## 複雜度評估 Rubric（Assess 模式）

判斷專案是否該裝 OpenSpec 的訊號為主，跟舊版相同（OpenSpec 是更重的工具，所以 gate 比舊單檔 spec.md 更重要）。

### 加分項（推薦裝 OpenSpec）

| 訊號 | 分數 | 偵測方式 |
|---|---|---|
| 有 `package.json` 且 dependencies 含 web framework（react/vue/svelte/next/vite/astro） | +3 | `jq -r '.dependencies // {} \| keys[]' package.json` |
| 有 `.git/` 且 commit 數 ≥ 5 | +2 | `git log --oneline \| wc -l` |
| 源碼檔案數（`.py` / `.ts` / `.tsx` / `.js` / `.jsx` / `.vue` / `.svelte` / `.go` / `.rs`）≥ 10 | +2 | Glob |
| 有 README.md 且長度 > 500 字 | +1 | `wc -c README.md` |
| 有 tests 目錄 / `__tests__` / `*.test.*` / `*.spec.*` | +1 | Glob |
| 使用者本 session 已編輯 ≥ 3 個不同檔案 | +1 | session log |
| 多人協作（`.git/config` 有 multiple authors，或 `CONTRIBUTORS.md`） | +2 | OpenSpec 對 team 收益更大 |
| 計畫建立 ≥ 3 個互相依賴的 skill / module cluster（共用 hook / shared contract / ecosystem doc） | +4 | 使用者自述、或偵測到已有 `ECOSYSTEM.md` / 多個 skill 在 intake queue。每個 skill 天然對應一個 capability，跨 session 設計 contract 正是 OpenSpec 強項 |

### 減分項（不推薦）

| 訊號 | 分數 | 偵測方式 |
|---|---|---|
| 專案根有 `research_plan.json` | -10 | 強烈退讓給 research-plan |
| 專案根有 `01_protocol/` + `09_qa/` | -10 | 強烈退讓給 ma-end-to-end |
| 只有一個原始碼檔案 | -3 | 一次性 script |
| CWD 是 `~/Desktop` / `~/Downloads` | -2 | 可能是臨時任務 |
| CWD 在 `~/.claude/` 或其子目錄（非 skill cluster 規劃） | -3 | meta-work 通常不需 OpenSpec；例外：若命中「skill cluster ≥ 3」加分項則此項不扣 |
| 完全沒 source code（純 markdown / docs） | -2 | 走 manuscript skill |
| 專案規模 < 100 行 source code | -2 | OpenSpec overhead 太重 |

### 判定

- **總分 ≥ 5**：強烈推薦 `/spec init`（bootstrap OpenSpec）
- **總分 2–4**：邊緣，列出加減分讓使用者決定
- **總分 ≤ 1**：不推薦，建議用其他工具或手動 README

**Skill cluster 專用規則**（命中 skill cluster 加分項時）：
- `openspec/` 目錄放在 **cluster root**（例：`~/.claude/skills/<cluster-name>/openspec/`）而非 `~/.claude/` 或個別 skill 內
- 每個 skill 對應一個 `openspec/specs/<skill-name>/spec.md` capability
- 共用 hook / shared contract 另開 capability（例：`shared-hook`、`ecosystem-contract`）
- Init 時 `openspec/project.md` 的 Purpose 寫 cluster 整體目標，不是單一 skill

---

## 主流程

### Mode: `assess`

1. `pwd` 取 CWD
2. 跑 preflight：`openspec --version`（不阻塞，只報告）
3. 跑上方 rubric，算分數
4. 用 AskUserQuestion 呈現結果：

```
專案複雜度評估結果：

📊 分數：<N>（加分 <X> / 減分 <Y>）
📈 訊號：
  + [加分訊號 1]
  + [加分訊號 2]
  - [減分訊號 1]

🔧 OpenSpec 狀態：[已安裝 vX.Y.Z / 未安裝]

💡 推薦：[強烈推薦裝 OpenSpec / 邊緣 / 不推薦]
   理由：[一句話]

選項：
  1. 現在跑 /spec init（bootstrap OpenSpec）
  2. 先跑 /spec clarify 用 grill-me 把 audience / NFR / scope 釐清，再 init（推薦給 description 模糊的非 CS 背景使用者）
  3. 先不裝，之後再看
  4. 這類專案有更適合的工作流 → [指向對應 skill]
  5. 已有 git release 歷史 → 改跑 /spec retro 直接回填
```

### Mode: `clarify`

**觸發時機**：使用者在 `/spec init` 之前想先 grill 清楚 audience / NFR / failure mode 等面向；或 `/spec assess` 結果建議裝 OpenSpec 但 project 描述還太模糊。**Pre-init only**，已 init 過就不該再用（用 `/spec note` / `/opsx:propose` 取代）。

**為什麼要這個模式**：原本 `/spec init` 只問 2 題（用途 + stack），對非 CS 背景使用者太薄。`grill-me` skill 的 software-project pack 涵蓋 6 個 facet（audience / scope / NFR / failure tolerance / tech constraints / deploy model），補齊 OpenSpec 的 clarification gap。

**流程**：

1. **Preflight**：
   - CWD 不能已有 `openspec/`（已 init 過的不該重 grill）— 偵測到就問「已 init 過的專案要不要 skip clarify 直接 `/spec resume`？」
   - **若 `openspec/project-clarifications.md` 已存在**（前次 clarify 留下的）→ 用 AskUserQuestion 問：
     - 選項 A: 重新 grill → 改名舊檔為 `project-clarifications.md.bak.<timestamp>` 後走 step 2（保留歷史不直接刪）
     - 選項 B: 接續上次 → 中止 clarify、提示「`/spec init` 會自動用既有 clarifications 預填」
     - 選項 C: 取消
   - 不檢查 `openspec` CLI（clarify 純文件，不碰 OpenSpec）
2. **呼叫 `grill-me` skill** — 透過 Skill tool invoke，傳入：
   - `mode: deep`（強制 Deep，不偵測）
   - `pack: software-project`（鎖死，不偵測）
   - `output: <cwd>/openspec/project-clarifications.md`（覆寫 default scratch path）
   - `recommended_next_skill: /spec init`
3. **grill-me 跑完後**：自動回到本 skill，印「✅ 已寫入 `openspec/project-clarifications.md`，下一步：`/spec init` 會自動讀此檔預填 `project.md`」
4. **不主動觸發 `/spec init`**（curator 哲學）：使用者自己決定何時 init

### Mode: `init`

1. Preflight：確認 `openspec` CLI 存在，沒有就引導安裝
2. 檢查 CWD 是否已有 `openspec/` 目錄 — 有就問「要重 init 嗎？還是切到 resume 模式？」
3. **Read clarifications.md if exists**：
   - 若 `openspec/project-clarifications.md` 存在 → Read 全文進 context
   - 抽取對應欄位寫入 `project.md` 對應 section（精確對齊 `templates/project.md`）：
     | Facet | 寫入 project.md section |
     |---|---|
     | F1 Audience | `## Purpose`（短目標）+ `## Target Users`（user persona） |
     | F2 Scope | `## Roadmap`（MVP / nice-to-have / future）+ `## Out of Scope`（永不做的明確邊界） |
     | F3 NFR | `## Non-Functional Requirements` |
     | F4 Failure Tolerance | `## Failure Modes & Constraints` |
     | F5 Tech Constraints | `## Stack & Constraints` |
     | F6 Deploy Model | `## Deploy & Distribution` |
   - **Parsing strategy**（從 clarifications.md 抽 facet answer）：
     - Regex `(?m)^## Facet (\d+):\s*(.+?)$` 抽 facet number + name
     - 對每 facet，讀到下個 `^##` 或 EOF；用 `\*\*A\*\*:\s*(.+?)(?=\n\n|^##|\Z)` 抽答案 paragraph
     - 多個 sub-facet 答案合併成 bullet list
   - **仍跑 AskUserQuestion confirm**（curator 哲學：使用者要看到並 confirm 預填內容）— 對每個 section 顯示「已從 clarifications 預填，可改 / 接受 / skip」
   - 若不存在 → 跑下方 step 4 蒐集 minimal 2 欄；結尾建議「下次可先 `/spec clarify` 補完整 context」
4. 用 AskUserQuestion 蒐集（若 step 3 有 clarifications，這裡是 confirm；若沒有，這裡是收集）：
   - **Project purpose**（1–3 句長期目標，不綁單次改動）
   - **Stack / Constraints**（語言、框架、deadline、相容性）
   - **是否要啟用 expanded profile**（含 verify / continue / ff 等 6 個額外指令；推薦「是」）
4. 跑 `openspec init --tools claude`（init OpenSpec 結構與 Claude Code skills/commands）
5. 若使用者選 expanded profile：
   - 提醒手動跑 `openspec config profile`（互動式）或直接編輯 `~/.config/openspec/config.json` 設定 `profile: custom` + `workflows: [...10 個...]`，然後 `openspec update --force`
6. 寫 `openspec/project.md`（**這個檔案 OpenSpec 預設不會建**，是我們自訂作為 project context；可被 OpenSpec 的 config.yaml `context` 欄位引用）— 用 `~/.claude/skills/spec/templates/project.md` 作範本，讓使用者填欄位
7. 寫 `openspec/config.yaml`（curator rules + project defaults，見下方範本）
8. **寫 project-level `<project>/CLAUDE.md`**：
   - 為什麼：Claude Code 原生讀 `CLAUDE.md` 不讀 `AGENTS.md`，所以 retreat rules + pipeline 要寫進 project CLAUDE.md 才會被每個 session 讀到
   - 把 `~/.claude/skills/spec/templates/project-claude-md-append.md` 的內容 **append** 到 `<project>/CLAUDE.md`（若檔不存在就 create）
   - 若既有 CLAUDE.md 已含 `BEGIN: spec skill` marker → 先告知使用者並用 AskUserQuestion 問是否覆寫 managed block
   - 若不在 git repo 內 → 提醒使用者把 CLAUDE.md 加進版控
9. 提醒使用者：
   - **重啟 Claude Code** — 不重啟 `/opsx:*` slash command 不會出現（GitHub issue #237）
   - 第一個 change 用 `/opsx:propose "your idea"` 開始
   - 之後 session 開場用 `/spec resume`、結束前用 `/spec handoff`

### Mode: `retro`

**觸發時機**：mature project（≥ 3 git tag）已有 release 歷史但無 `openspec/`。比 init→propose 更高效：架構決策資訊已在 commit messages + release notes，只需 mine + 寫成 OpenSpec spec format。

**適用條件**（全命中才走 retro）：
- CWD 有 `.git/`
- `git tag -l | wc -l` ≥ 3
- 沒有 `openspec/`（有就問合併還是取消）
- `openspec` CLI 已安裝

**流程**：

1. **Bootstrap**：跑 `openspec init --tools claude`（建立空骨架）
2. **Harvest**（read-only，可平行跑）：
   - `git tag -l --sort=-creatordate`
   - `gh release list --limit 50`（若 gh 可用）
   - 對每個 minor / major tag 跑 `gh release view <tag>`
   - 對關鍵 tag 間跑 `git show --stat <tag>`
   - `git log --all --oneline` 補抓重要但未 tag 的 commit
3. **Identify capabilities**（LLM 處理，不呼叫 user）：
   - 從 README + 主要源碼檔結構推斷 1–5 個 capability（kebab-case：例 `auth`、`search`、`export`）
   - 每個 capability 對應一個將寫入的 `openspec/specs/<capability>/spec.md`
4. **Extract requirements per capability**（LLM 處理）：
   - 從 commit messages + release notes 挑跟此 capability 相關的 normative behavior（「the system MUST...」「user CAN...」）
   - 寫成 RFC 2119 格式（SHALL / MUST / SHOULD / MAY）
   - 每個 requirement 至少 1 個 BDD scenario（`#### Scenario:` + `WHEN/THEN`）
5. **Review with user**：
   - 呈現草稿：候選 capabilities + 每個的 requirements + scenarios
   - 用 AskUserQuestion 讓 user 修 / 刪 / 補
6. **Write**：
   - `openspec/project.md`（從 README 推斷 + 手動補）
   - `openspec/specs/<capability>/spec.md` × N（依使用者 confirm 後的草稿）
   - **不寫 `openspec/changes/`** — retro 是回填現狀，不是 propose 新東西
7. **Validate**：跑 `openspec validate --all`，有 error 修到通過
8. **下一步建議**：
   - 重啟 Claude Code 讓 slash command 生效
   - 之後新功能用 `/opsx:propose`
   - 提醒：若想記錄歷史 architecture decisions，可手動建 `openspec/decisions/<YYYY-MM-DD>-<topic>.md`（OpenSpec 不強制此目錄，但 config.yaml 可宣告）

### Mode: `resume`

**觸發時機**：新 session 開場，使用者手動或 SessionStart hook 注入 openspec/ 存在訊號後主模型自動建議。

**目標**：把 OpenSpec 檔案系統當成跨 session working memory。開場一次 load 足夠 context，之後 session 中**不需要**反覆問使用者「上次到哪了 / 為什麼這樣決定」— 這是跨 session 省 token 的核心路徑。

1. Preflight：`openspec --version`
2. 取狀態（並行）：
   - `openspec list` — 列出所有 changes（in-progress + archived recent）
   - 對每個 in-progress change 跑 `openspec status --change <name>` — 取進度
   - `openspec list --specs` — 列出已穩定的 capabilities
   - `git log --oneline -10` — 最近 commit
3. **Context warm-up（必做，不可省）**：主動用 Read tool 把下列檔案一次性倒進 context：
   - `openspec/project.md` 全文 — 穩定 project purpose + stack + out-of-scope
   - **每個** in-progress change 的 `proposal.md` + `design.md` 全文 — 決策 rationale 與實作 how
   - **每個** in-progress change 觸及的 `openspec/specs/<capability>/spec.md` 全文 — 當前 main truth
   - 最近 3 個 archived change 的 `proposal.md`（取第一段 summary 即可，不用全文） — recent context
   - 若 `openspec/decisions/` 目錄存在，讀最新 3 個 `.md` 檔全文 — 跨 change 的歷史決策
   
   **代價**：開場多花 1–3K token load context。
   **收益**：session 中免去「之前我們決定什麼 / 為什麼選 X」的反覆解釋，對話只承擔新思考，檔案承擔記憶。
4. 產出三句話 summary：
   - 第一句：**Project**（從 `openspec/project.md` 第一段摘出 purpose）
   - 第二句：**Last known state**（最新 change 的進度 + 最後一個 archived change 的 timestamp + headline）
   - 第三句：**Suggested next**（in-progress change 的下一個未完成 task；若無 in-progress，建議用 `/opsx:propose` 起新 change）
5. 問使用者：「這方向對嗎？還是要先看別的？」
6. 確認後再開始實作。

**退化情境**：若 `openspec/` 存在但 `project.md` 是空的 / 全 template placeholder → 在 summary 裡 flag「project.md 尚未填寫，建議先補齊 Purpose / Stack」再繼續。

### Mode: `handoff`

**觸發時機**：session 結束前手動呼叫；PreCompact hook 在已 init OpenSpec 的專案會自動建議。

1. Preflight：`openspec --version`
2. **Validate**：`openspec validate --all` — 有結構錯誤先報警
3. **Check completed changes**：跑 `openspec list`，篩選所有 4/4 artifacts complete + tasks 全勾的 changes
4. **For each completed-but-unarchived change**：
   - 用 AskUserQuestion 問「要 archive `<change>` 嗎？這會把 delta sync 進主 specs/ 並移到 archive/ 目錄」
   - 使用者確認 → 建議走 `/opsx:archive <change>` slash command（workflow 會主動問「先 sync 嗎」並把 delta 寫入 `openspec/specs/<capability>/`），**不**直接呼叫 raw `openspec archive --yes`（後者會跳過 sync gate）
   - 取得 archive 路徑 → 寫入 commit message draft
5. **Suggest commit（硬互動點，不能 silent skip）**：
   - 偵測 git status，若有 dirty changes（含剛 archive 的 changes/）→ **必須**用 AskUserQuestion 明確列出：
     - 選項 1：現在 commit（走 auto-git skill，template：`spec(archive): merge <change-name> — <proposal headline>`）
     - 選項 2：先不 commit，手動處理
     - 選項 3：取消 handoff
   - 使用者選 1 → 走 auto-git skill commit（**不**直接呼叫 `git commit`；不加 `--no-verify`）
   - 使用者選 2 / 3 → 尊重、不推
   - 為什麼是硬互動點：archive 完沒 commit → 下次 session 會看到 main specs/ 已變動但 git 沒記錄，混淆歷史
6. **Decision log reminder**：
   - 若本 session 有非 trivial 技術決策（從 conversation 推斷）但未寫入任何 archived change 的 proposal.md
   - 提示使用者：「以下決策建議寫進 `openspec/decisions/<date>-<topic>.md`」或用 `/spec note "..."` 即時捕捉
   - 不自動寫，使用者確認後 Edit

### Mode: `note`

**觸發時機**：
- `/spec note "<decision text>"` — 一句到一段話的即時決策捕捉
- 使用者說「先記一下」「等等記下來」「決定是 ...」「這個要記住」「待會別忘」「存一下決策」等意圖 → 主模型提示走此模式

**為什麼需要這個模式**：對話中的非 trivial 決策（例：「我們用 X 不用 Y，理由：...」）如果不當場寫進檔，compact 或關 session 就會流失。`/opsx:propose` 成本太高（要寫完整 proposal/design/tasks），`/spec note` 是**低成本 decision capture 通道**，專治「決策在對話裡但沒寫檔」的漏洞。

**流程**：

1. Preflight：確認 CWD 有 `openspec/`。沒有 → 提醒「先跑 /spec init」後中止
2. **Parse note text**：
   - `/spec note` 後的全部字串作為 note body
   - 若使用者沒給（只打 `/spec note`）→ 用 AskUserQuestion 收一段文字（open-ended）
   - 空字串 → 中止並提醒「note 至少要一句話」
3. **Auto-derive short tag**：從 note body 第一個逗號/句號前的片段取前 ~40 字作為條目標題（可用 Python 處理）
4. **偵測當前 in-progress change**：
   ```bash
   openspec list --json 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print('\n'.join(c['name'] for c in d.get('changes',[]) if c.get('status') != 'archived'))"
   ```
   若 `openspec list --json` 不支援，降級用 `ls openspec/changes/`（未 archive 的都還在此目錄）
5. **決定寫入位置**：
   - **0 個 in-progress change** → 走 `openspec/decisions/<YYYY-MM-DD>.md`（檔案不存在就建含 `# Decisions — YYYY-MM-DD\n` 標題）
   - **1 個 in-progress change** → 預設 append 到該 `openspec/changes/<change>/design.md` 的 `## Decisions` 區段
   - **≥ 2 個 in-progress change** → AskUserQuestion 列出所有 in-progress change 名稱 + 選項「都不屬於，進 openspec/decisions/」讓使用者選
6. **寫入格式**：
   - 若寫入 change 的 `design.md`：
     ```markdown
     ## Decisions

     ### YYYY-MM-DD HH:MM — <auto-tag>
     
     <note body>
     ```
     （若 design.md 已有 `## Decisions` 區段，append entry；沒有就新增區段在檔尾）
   - 若寫入 `openspec/decisions/<YYYY-MM-DD>.md`：
     ```markdown
     ## HH:MM — <auto-tag>

     <note body>
     ```
     （append 到檔尾，保持時序）
7. **回顯**：打印寫入路徑 + 擷取的前 80 字，讓使用者看一眼確認無誤
8. **不自動 commit**（Curator 哲學）；session 結束 `/spec handoff` 會一併處理。若使用者明確要求 commit 則走 auto-git skill

**失敗模式**：
- 使用者給空字串 → 中止，提醒「note 至少要一句話」
- 寫入失敗（檔案權限 / 磁碟滿） → 報告錯誤 + 不 retry
- `openspec list --json` 失敗 → 降級用 `ls openspec/changes/`；仍失敗就全走 `decisions/`
- 偵測到 in-progress change 但使用者的 note 顯然跨 change（例「總結整個 Q2 的決策方向」） → AskUserQuestion 確認是否改走 `decisions/`

**好的 note 例子**：
- `"用 Postgres 不用 SQLite，理由：未來 multi-user concurrent 與需要 FTS"`
- `"API rate limit 上限暫定每 IP 100 req/min，沒有強依據，先跑一週觀察"`
- `"決定用 Zustand 不用 Redux，Zustand 體積小、對本專案規模夠用"`

**不適合 note 的例子**（這些該走 `/opsx:propose`）：
- 新 capability 的 BDD scenario（要進 spec 的 delta）
- 多步驟實作計畫（要進 tasks.md）
- 會改 main spec 語意的變更（要走完整 propose → archive 流程）

### Mode: `clear`

**觸發時機**：
- `/spec clear` 手動呼叫
- `spec-context-watch.sh` strong tier（≥ 2.0 MB）注入提醒後，主模型在 auto mode 直接走此流程
- 使用者說「切 session」「清 context」「換新 session」

**為什麼需要這個模式**：`/clear` 是 Claude Code 內建命令，**只能使用者輸入**（slash command 不能由 hook 或 skill 執行）。但除此之外的收尾步驟都可以自動化。`/spec clear` 把「handoff + 提示 /clear + 下次 resume 銜接」打包，讓切換 session 變一鍵。

**流程**：

1. **Preflight**：確認 CWD 有 `openspec/`。無 → 告知使用者「無 OpenSpec 專案，直接 /clear 即可」並中止
2. **Pending decisions scan**（保障）：
   - 掃 conversation（本輪 response 可見範圍）看有無使用者確認但未寫檔的非 trivial 決策
   - 有 → 列 ≤ 5 bullet，**問使用者**：「這些要先 `/spec note` 嗎？（Y=幫你寫 / N=直接 clear）」
   - 無 → 進 step 3
3. **執行 handoff 流程**（等同 `Mode: handoff`）：
   - `openspec validate --all`
   - 偵測 completed 但未 archive 的 changes → 問要不要 `/opsx:archive`
   - git status dirty → 走 auto-git 問要不要 commit
4. **Clear instruction**（硬互動點）：
   - 回顯訊息（繁中）：
     ```
     ✅ Handoff 完成。狀態已全部寫進 openspec/（specs / changes / decisions / snapshot）。
     
     現在請手動執行：/clear
     
     下次開始工作時打 /spec resume，我會 auto-load：
       - project.md 全文
       - 所有 in-progress change 的 proposal + design + spec
       - 最近 archive + decisions 最新 3 個
     
     工作記憶全部從檔案還原，不需要你重新解釋上次到哪。
     ```
   - **不**自己嘗試 invoke `/clear`（會失敗 — slash command 是 user-only）
5. **退化情境**：
   - 若無 in-progress change、無 dirty git、無未寫決策 → 整個 step 3 skip，直接進 step 4 告訴使用者可 `/clear`
   - 若 handoff 流程被使用者中途取消 → 不強迫 clear，respect 使用者節奏

**自動化層級**：
- **手動**：使用者直接打 `/spec clear`
- **半自動（strong warning 觸發時）**：context-watch hook 在 2.0 MB 注入 → auto mode 下主模型自動跑 `Mode: clear` 完整流程，停在 step 4 等使用者打 `/clear`
- **不可自動**：`/clear` 本身必須使用者輸入（Claude Code 安全限制）

**為什麼不能自動打 /clear**：Claude Code 把 slash command 當作使用者意圖表達，skill / hook 不能代替使用者 invoke，避免 AI 意外清掉對話。這是硬限制，接受即可。

---

## 修改既有 main spec（delta 語法）

`openspec/specs/<capability>/spec.md` 是 **main truth**，**不直接編輯語意**。任何 requirement 的新增 / 修改 / 刪除 / 改名都走 change delta 流程。

### 語意修改（99% 情況走這條）

```bash
/opsx:propose <change-name>
```

OpenSpec 在 `openspec/changes/<change>/specs/<capability>/spec.md` 建 delta 檔，用以下區段描述改動：

| 區段 | 用途 |
|---|---|
| `## ADDED Requirements` | 新 requirement（加 BDD scenario） |
| `## MODIFIED Requirements` | 修改既有 requirement 的 normative 文字 / scenario |
| `## REMOVED Requirements` | 刪除不再適用的 requirement（建議附一句 reason） |
| `## RENAMED Requirements` | 重新命名（format：`old-id` → `new-id`） |

走完 `/opsx:apply` → `/opsx:verify` → `/opsx:archive`，delta 自動 merge 進 main spec，change 搬進 `archive/` 永久保留 rationale。

### 純 typo / markdown 修復

Non-semantic 的錯字、格式壞、超連結壞可以**直接手動編** `openspec/specs/<cap>/spec.md`。但：
- 不可改變任何 `SHALL / MUST / SHOULD` 的語意
- 改完跑 `openspec validate --all` 確認結構沒壞

### 撤回已 archive 的錯誤

**絕不動 `archive/` 目錄**（歷史不可變）。開新 change 補救：

```bash
/opsx:propose revert-<broken-change>
```

在新 delta 用 `MODIFIED` / `REMOVED` 把錯的倒回。archive 該次 revert 時 main spec 被正確 merge。

### 不該做的事

- ❌ 為了「清理」手動編 main spec 語意 — 會跟未來 archive 的 delta 衝突
- ❌ 直接刪 `archive/<date>-<change>/` 資料夾 — 丟失決策歷史
- ❌ 在同一個 change 同時 ADDED + MODIFIED 同一 requirement — 分兩個 change 提案

---

## 相關檔案 & 範本

### `openspec/project.md`（使用者寫的 project context）

由 `/spec init` 產出。OpenSpec 不強制這個檔案存在，但 config.yaml 可 reference 進去作為 AI context。範本見 `templates/project.md`。

### `openspec/config.yaml`（curator rules + project defaults）

由 `/spec init` 產出，包含：
- `context.project_md` — reference project.md 內容到所有 artifact 生成
- `rules` — curator 硬規則（不自動 commit、所有非 trivial 決策 AskUserQuestion、PHI / 機密資料零容忍）
- `retreat` — 自訂偵測：若同目錄有 `research_plan.json` 或 `01_protocol/`，OpenSpec 應警告退出

範本見 `templates/config.yaml.tmpl`。

### `openspec/decisions/<date>-<topic>.md`（optional）

OpenSpec 不強制此目錄。但跨版本的 architecture decisions（例：「為什麼 v0.5 從 Vite 換到 Next」）放這比塞進 individual change 的 proposal.md 更易檢索。`/spec handoff` 會偵測未記錄的決策並提醒。

---

## SessionStart Hook 行為

Hook script: `~/.claude/scripts/spec-session-start.sh`（已從舊版改寫）

**新邏輯**：
1. 讀 CLAUDE_PROJECT_DIR（或 fallback 到 `pwd`）
2. 跑快速 gate（命中即 exit 0 不注入）：
   - CWD 是 `~/Desktop` / `~/Downloads` / 沒 code 訊號 → skip
   - 已有 `research_plan.json` → skip（統計專案走 research-plan）
   - 有 `01_protocol/` + `09_qa/` → skip（MA 走 ma-end-to-end）
   - CWD 是 `~/.claude/` 或 home root → skip
3. **Case A**：偵測到 `openspec/` 目錄 → 提醒主模型建議 `/spec resume`
4. **Case B**：偵測到 code 訊號（package.json / pyproject.toml / Cargo.toml / go.mod / requirements.txt）但無 `openspec/`：
   - `git tag -l | wc -l` ≥ 3：提醒主模型建議 `/spec retro`（mature-but-no-spec）
   - tag < 3：提醒建議 `/spec assess`

**不做的事**：
- Hook 不跑 rubric（留給 skill）
- Hook 不呼叫 `openspec` CLI（避免 SessionStart 慢）
- Hook 不寫檔案

## PreCompact Hook 行為（保底 checkpoint）

Hook script: `~/.claude/scripts/spec-pre-compact.sh`

**觸發時機**：auto-compact 或 manual `/compact` 即將發生前，Claude Code 觸發 PreCompact event。

**動作**（偵測到 `openspec/` 存在時）：
1. **背景 auto-checkpoint**（不阻塞 compact）：
   - 跑 `openspec validate --all` → 寫 `~/.claude/logs/openspec-precompact-<slug>-<timestamp>.txt`
   - Dump `openspec list --json`（in-progress changes 快照）→ 寫 `openspec/decisions/<timestamp>-precompact-snapshot.md`
   - 快照裡含 validate 摘要 + 復原提示「下次 `/spec resume` 會讀到」
2. **注入 additional context**：提醒主模型在下一輪列出本 session 未寫進 OpenSpec 的關鍵決策（≤5 bullets），建議使用者對每個跑 `/spec note`

**為什麼需要 auto-checkpoint**：PreCompact 觸發時 context 已接近滿，主模型可能沒 budget 做完整 `/spec handoff`。至少先把 validate 結果 + in-progress state **落地到磁碟**，下個 session 即使 compact 丟失對話細節，`/spec resume` 仍能從 `openspec/decisions/<ts>-precompact-snapshot.md` 重建。

## UserPromptSubmit Hook 行為（主動預警）

Hook script: `~/.claude/scripts/spec-context-watch.sh`

**觸發時機**：每次使用者送訊息時。

**動作**（偵測到 `openspec/` 存在時）：
1. 讀取當前 session 的 transcript jsonl 大小（`transcript_path` 由 Claude Code 提供）
2. 跟 threshold 比對（預設可用 env var 覆寫）：
   - `< 1.2 MB`：靜默
   - `≥ 1.2 MB`（warn tier，~40–50% context）：注入提醒「建議現在 `/spec note "<decision>"` 把尚未落地的決策寫進檔」
   - `≥ 2.0 MB`（strong tier，~65–75% context）：注入強提醒「現在跑 `/spec handoff` + 手動 `/compact "focus on <change>"`，auto-compact 即將觸發」
3. **Throttle**：每個 tier 每個 session 只 fire 一次（state file `~/.claude/logs/context-watch-<session_id>.state`）

**為什麼用 jsonl 大小當 proxy**：Claude Code 沒有對 hook 暴露 token 計數。jsonl 檔 monotonic 增長，byte size 是最好的可得 proxy。經驗校準：2.3 MB jsonl ≈ ~75% context usage（JSON structure + tool outputs overhead ~3–4x）。

**Threshold 覆寫**：可在 shell 或 `.env` 設 `CONTEXT_WATCH_WARN_BYTES` / `CONTEXT_WATCH_STRONG_BYTES`。

## 三層防禦總覽（auto-compact 前保存決策）

| 層 | 檔案 | 時機 | 行為 |
|---|---|---|---|
| 1. 主動預警 | `spec-context-watch.sh` | 每次送訊息 | 50-70% 提醒 note、70-85% 強推 handoff+compact |
| 2. 保底 checkpoint | `spec-pre-compact.sh` | auto-compact 觸發前 | 背景 validate + 寫 snapshot 到 decisions/ |
| 3. Session 結束 | `/spec handoff` 手動 | 使用者打 `/spec handoff` | validate + archive completed + auto-git commit |

三層互補：第 1 層預防、第 2 層保底、第 3 層收尾。正常使用只需記得 session 結束打 `/spec handoff` 或接受 hook 的提醒即可。

### Meta-work exception（2026-04-21 補）

Hook script 的 skip gate 原本會把 `~/Desktop` / `~/Downloads` / `~/.claude*` 當 home-root 一律退讓。**副作用**：使用者在 `~/Desktop` 做 meta-work（編 `~/.claude/` 內的 skill / scripts / CLAUDE.md / settings.json）時三層防禦全失效 — 2026-04-21 session 踩到。

修補：兩個 hook（`spec-context-watch.sh` / `spec-pre-compact.sh`）都加 `is_meta_work()` heuristic — 若 `find -L ~/.claude` 在最近 30 分鐘內有 skill/config/script 檔被編輯，即使 CWD 在 skip list 內也**不 skip**，改走 meta-mode 分支：
- context-watch 發 meta-mode 專屬警示（建議寫 scratch note 或直接寫進 affected SKILL.md / CLAUDE.md）
- pre-compact 把 recently-touched file list 快照到 `~/.claude/scratch/precompact-snapshots/<ts>-meta-snapshot.md`（因為 `~/.claude/` 不是 git repo、沒 openspec/，這是唯一的 paper trail）

`find -L` 是關鍵：許多人會把 `~/.claude/{CLAUDE.md,imports,scripts,skills}` symlink 到外部 dotfiles repo（例 `~/dotfiles/claude-config/`）做版控，沒 `-L` follow symlink 會完全抓不到真正的 meta-work 編輯。

---

## 與其他 skill 的互動

| skill | 互動方式 |
|---|---|
| `grill-me` | `/spec clarify` 是 grill-me 的 thin wrapper（強制 Deep + software-project pack + 輸出到 `openspec/project-clarifications.md`）；`/spec init` 偵測 clarifications.md 自動預填 project.md |
| `research-plan` | `spec` 偵測 `research_plan.json` 退讓；不搶統計專案 |
| `ma-end-to-end` | 偵測 MA 結構退讓 |
| `/verify` | `/spec handoff` validate 通過後可建議接 `/verify` 做 code-level 端到端驗證 |
| `auto-git` | `/spec handoff` 內 commit 步驟走 auto-git skill 而非直接 `git commit` |
| `auto-skill-eval` | `/spec init` / `/spec retro` 完成後是 non-trivial 任務，會被自動評估 |
| `no-wheels` | vibe-coding 專案 `/spec assess` 推薦裝 OpenSpec 前可先建議跑 `/no-wheels` 確認不是造輪 |
| `/opsx:*` (OpenSpec native) | `spec` 不重複實作；propose / apply / archive 一律走 OpenSpec |

---

## 失敗模式與回退

| 症狀 | 處理 |
|---|---|
| `openspec` CLI 不存在 | 提醒安裝 `npm install -g @fission-ai/openspec@latest`，用 AskUserQuestion 問是否現在裝 |
| `/spec init` 時 CWD 已有 `openspec/` | 問使用者：重 init / 改走 resume / 取消 |
| `/spec retro` 時 CWD 已有 `openspec/` | 問使用者：在現有 specs/ 上 append capability / 取消 |
| `/spec retro` 時 tag < 3 | 改建議 `/spec init`（沒歷史可挖） |
| `/spec retro` 時 `gh` 未安裝或未登入 | 降級成僅用 `git log` + `git show`，跳過 release notes |
| `/spec resume` 時 `openspec/` 不存在 | 退到 `/spec assess` |
| `/spec handoff` 時 `openspec validate` 失敗 | 報告錯誤 + 不繼續，要使用者手動修 |
| Slash command `/opsx:*` 不出現（GitHub #237） | 提醒「重啟 Claude Code」，若仍無 → 檢查 `<project>/.claude/commands/opsx/` 檔案是否存在 |
| Raw `openspec archive <name> --yes` 跳過 sync gate（main specs 缺新 capability） | 走 `/opsx:archive` slash command 而非 raw CLI — workflow 版本會主動問「要先 sync 嗎」；若已用 raw CLI archive 過、發現 specs/ 缺檔，跑 `openspec sync-specs` 補回 |
| OpenSpec 升版後 commands 不見（GitHub #769） | 跑 `openspec update --force` 重生成 |
| Opus 跳過 OpsX workflow 直接寫 code（GitHub #869） | 在 `openspec/config.yaml` 的 rules 區段加硬規則「禁止跳過 propose / apply 直接 implement」 |
| SessionStart hook 執行失敗 | 靜默不影響 session（stderr 丟 log） |

---

## References

- [Planning Patterns](references/planning-patterns.md) — Manus 風格四大 pattern（2-Action Rule / 3-Strike Error Protocol / Read-vs-Write Matrix / 5-Question Reboot Test）。`resume` 必跑 5-Question Reboot Test
- OpenSpec 官方文件：[github.com/Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)、[openspec.dev](https://openspec.dev/)
- 已知 GitHub issues 影響：#237（slash command 不出現）、#769（升版 command 不見）、#834（workflow 順序未強制）、#869（LLM 跳過 OpsX）— 寫 config.yaml rules 時要主動 mitigate

---

## 不做的事（Out of Scope）

- **不重複 OpenSpec 已有功能** — propose / apply / archive 一律走 `/opsx:*`
- **不自動寫 spec / change** — 永遠要使用者確認
- **不取代 TodoWrite** — short-term tasks 用 TodoWrite，cross-session lifecycle 才用 `spec`
- **不取代 research_plan.json / MA 目錄結構**
- **不自動 commit** — `/spec handoff` 內 commit 經 auto-git skill 並等 user confirm
- **不 sync 到雲端 / 外部系統** — 純 local file
- **不做進度條 / Gantt** — 進度交給 OpenSpec `openspec status` 顯示
