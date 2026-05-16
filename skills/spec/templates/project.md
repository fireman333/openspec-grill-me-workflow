# {{PROJECT_NAME}} — Project Context

> 由 `/spec init` 產生。OpenSpec 不強制這個檔案，但 `openspec/config.yaml` 會 reference 進去作為所有 artifact 生成時的 AI context。**手動維護**，不要讓 OpenSpec 自動覆寫。

## Purpose

<!-- 1–3 句長期目標。描述「這個專案是什麼、解決誰的什麼問題」，不綁單次改動。
範例：
- 給 RA 用的心衰竭資料庫互動式 dashboard，方便指導教授一次看 cohort 篩選 + KM 曲線
- Vibe-coding side project：把考古題錯題本自動化進 RemNote，每週 push reminder
-->

## Target Users

<!-- 誰會用這個 app（從 clarifications facet 1）。寫實際 user persona 而非「想像中的 user」。
範例：
- 自己 + 實驗室 5 人（內部工具）
- 心臟科 RA / 主治醫師（≤ 30 人，預期 6 個月內成長到 80 人）
-->

## Stack & Constraints

<!-- 列當前用的 language / framework / key libraries，以及硬限制（deadline / 相容性 / 部署環境）。
範例：
- Stack: Python 3.11 + Streamlit + DuckDB
- Deploy: 院內 intranet 機器，無外網存取
- Constraint: PHI 零容忍 — 所有 patient ID 在進入 spec 前必須 deid
-->

## Non-Functional Requirements

<!-- 效能 / 跨平台 / 資料規模 / 維護期限（從 clarifications facet 3）。
範例：
- 互動回應 < 5 秒可接受；batch 跑一晚 OK
- 跨平台：只 Mac
- 資料量級：cohort < 5000 patients
- 維護期限：1+ 年（畢業後接手者：學弟妹）
-->

## Failure Modes & Constraints

<!-- 出錯場景 + 容忍度（從 clarifications facet 4）。No Silent Errors（見 coding_principles.md §1.5）。
範例：
- 病人安全層級：本工具不直接影響臨床決策，error 可容忍但要 log
- BOM / 編碼錯：Skip + 印 skip 總數（不 silent swallow）
- 第三方 API 掛掉：fallback 用本地 cache，告知使用者
-->

## Out of Scope

<!-- 明確列出「這專案不做什麼」。OpenSpec propose 時若使用者要求超出範圍，AI 應先提出 retreat 建議。
範例：
- 不做 multi-tenant（單機單使用者）
- 不做 user auth（純內網信任）
-->

## Roadmap

<!-- MVP 必須功能 + nice-to-have + 後續階段（從 clarifications facet 2）。Out of Scope 是「永不做」，Roadmap 是「現在不做但之後做」。
範例：
- MVP（必須，~ 4 週）：cohort filter UI / KM 曲線 / Table 1 export
- Phase 2 nice-to-have：competing risk plot / forest plot integration
- Phase 3 future：multi-cohort comparison
-->

## Deploy & Distribution

<!-- 使用者如何取得 app + 更新機制 + 安裝門檻（從 clarifications facet 6）。
範例：
- 取得方式：git clone + `uv run streamlit run app.py`
- 更新機制：手動 git pull
- 安裝門檻：使用者要會 terminal（接手者：學弟妹有受過訓練）
-->

## Key People & Sources

<!-- 誰會看 spec、誰是 stakeholder、外部資料來源。
範例：
- Owner: 我（RA）
- Reviewer: 心臟科 W 主任、生統 N 老師
- Data source: HF database snapshot 2026-Q1，IRB# 202601-XXX
-->

## Conventions

<!-- 專案內部慣例（命名 / 目錄 / commit message style）。OpenSpec scenario 寫作會參考。
範例：
- Capability slug 一律 kebab-case（survival-analysis、competing-risks）
- Commit prefix: feat: / fix: / spec: / chore:
- Test naming: `test_<capability>_<behavior>.py`
-->

## Related Projects / References

<!-- 相關 repo、paper、或其他 spec。
範例：
- Sister project: ~/Desktop/ttm-cooling-cohort/
- Reference paper: Rohman 2024 (HFrEF mortality model)
-->
