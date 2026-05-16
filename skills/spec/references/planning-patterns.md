# Planning Patterns — Manus 風格四大行為守則

Session 內的 context 管理 + error recovery 補充指引，給 `spec-driven` 的 `resume` / `handoff` mode 引用。四個 pattern 不強制每個 task 都跑，但遇到對應情境時照這個做能有效防止 context 流失與無效迭代。

來源概念：[othmanadi/planning-with-files](https://github.com/othmanadi/planning-with-files)（Manus 風格 file-based planning）。本文件只抽方法論，不引入該 skill 的 hooks 或額外檔案結構。

---

## 1. 2-Action Rule — Multimodal Context Flush

**規則**：每執行 **2 次** 以下類型的動作後，**立刻** 把重點 flush 到磁碟：

- 讀 PDF / image / 其他視覺型 artifact
- 跑 browser automation（Claude in Chrome / Preview / computer-use）取資料
- 拉多篇文獻（OE 查詢、lit-review、PubMed）
- Deep research 連續拉 web 結果

**寫去哪**：

| 工作流 | 目的地 |
|---|---|
| `spec-driven` 專案 | `spec.md` 的 Current status 或獨立 `findings.md` |
| `research-plan` 專案 | `research_plan.json` 的 notes 欄位 |
| MA 專案 | `07_extraction/` 對應欄位 |
| Ad-hoc | 當前 CWD 的 `notes.md` |

**理由**：Multimodal（圖、網頁截圖、PDF 內容）在 context 中位置易被 compact 擠掉；text 形式才穩定持久。兩次是經驗閾值 —— 一次還記得、三次已經遺失細節。

**不適用**：純文字對話、單次 file read、single tool call。

---

## 2. 3-Strike Error Protocol

遇 error 時**不允許用相同方法重試**。三次失敗後強制升給使用者。

```
Attempt 1: 診斷根因
  → 仔細讀 error message
  → 定位問題（是 input / env / logic / dependency？）
  → Targeted fix（只改會影響該 error 的東西）

Attempt 2: 換方法
  → 相同 error 再現 → 改用不同 tool / library / 角度
  → 絕不重複完全相同的失敗動作
  → 若原本用 A，就改 B；若 flag 不對，改整個策略而不是微調 flag

Attempt 3: 質疑假設
  → 重新審視 task 本身的假設是否成立
  → 檢查 precondition（檔案存在？網路通？權限對？）
  → 可能要重新規劃，而不是繼續修這個 step

第 4 次：升給使用者
  → 明確說「試過什麼」+「具體錯誤訊息」+「卡在哪個假設」
  → 不再嘗試 workaround
```

**鐵律**：
- `if action_failed: next_action != same_action`
- 不在 silent retry 迴圈消耗 context
- 不把 error 藏起來重試到成功 —— log 到 spec.md / progress / notes
- 與 `~/.claude/imports/coding_principles.md` 第 4 條「Goal-Driven 可獨立 loop」互補：Loop 的退出條件不只是「成功」，也包括「3 次仍失敗 → 主動升級」

---

## 3. Read-vs-Write Decision Matrix

減少冗餘 file read、確保多模態資訊被 text 化。

| 情境 | 正確動作 | 為什麼 |
|---|---|---|
| 剛寫完一個檔（Write / Edit 完）| **不要** 重讀該檔 | 內容還在 context，重讀只是浪費 token |
| 看過 image / PDF 後 | **立刻** write 摘要到 findings/notes | Multimodal 易流失，text 才穩定 |
| Browser / screenshot 取回資料後 | Write to file | Screenshot 不 persist 到下一輪 |
| 開新 phase / 回到 task | Read plan + findings | Context 可能已 stale，需 re-orient |
| 遇 error 時 | Read 當前相關檔 | 修 bug 需要真實當前狀態，不能靠記憶 |
| Session 中斷後 resume | Read 所有 planning 檔 | 還原狀態是第一要務 |
| Subagent 回來後 | 看 agent summary，不重跑 | Agent 已摘要，重跑是重複工 |

---

## 4. 5-Question Reboot Test

**用途**：`/spec resume` 模式開場必跑的自檢 checklist。若五題都能回答，代表 context 管理扎實、可直接開始工作；若有一題答不出，先補 read 相關檔再動手。

| 問題 | 答案來源 |
|---|---|
| **Where am I?** | `spec.md` 的 Current status（Done / In progress / Blocked） |
| **Where am I going?** | `spec.md` 的 Next + Acceptance criteria 未勾選項 |
| **What's the goal?** | `spec.md` 的 Goal section |
| **What have I learned?** | `spec.md` 的 Decision log / findings 檔 |
| **What have I done?** | `spec.md` 的 Current status / git log |

**流程**：
1. Read `spec.md`
2. 心中跑過五題
3. 把答案壓成三句話 summary 給使用者（Goal / Last status / Next 建議）
4. 使用者確認後才開始做事

若某題答不出：先 read 對應檔（git log、findings、decision log），補齊再動。**不在不知道答案的狀態下動手。**

---

## 與 `spec-driven` 各 mode 的對應

| Mode | 主要使用哪些 pattern |
|---|---|
| `assess` | （無，純評估） |
| `init` | 無，專注蒐集 Goal / Constraints / Acceptance |
| `handoff` | Pattern 2（若 session 內有 error 要 log） + Pattern 3 決定哪些寫進 decision log |
| `resume` | **Pattern 4（5-Question Reboot Test）必跑** + Pattern 3 Matrix 決定 read 哪些檔 |
| Session 中（非 spec mode）| Pattern 1（2-Action Rule）+ Pattern 2（3-Strike） |

---

## 不做的事

- ❌ 不把這四個 pattern 變成 hook 強制注入（使用者 context economy 優先）
- ❌ 不建 `findings.md` / `progress.md` 另外兩檔（spec.md 單檔已足夠；除非專案複雜到三檔才裝得下，否則不拆）
- ❌ 不在每次 tool call 前 cat spec.md（planning-with-files 原作的作法；開銷過高）
