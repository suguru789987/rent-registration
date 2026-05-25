# agent-rails/ — 地代家賃 エージェント・レール仕様（検証雛形 v0）

- 作成: 2026-05-25
- ステータス: **検証用雛形（PoC）。本番仕様ではない。**【要本人確認】
- 目的: 「UI主眼の仕様を、AIエージェントが参照するレール仕様（ドメイン/計算/制約/検証）に再記述できるか」を rent 1本で検証する

## この雛形が検証する仮説

> 既存のUI仕様（`mock/spec.md`・`verification/`・`modal/`・`JUDGMENT_LOG.md`）の素材は、
> **作り直しではなく視点変更だけ**で、エージェントが安全に地代家賃を扱うための
> 「レール（事実・制約・計算・検証）」に再記述できる。

検証の結論メモ（記入欄）:
- 移せた箇所: …
- 移せなかった/追加ドメイン知識が要った箇所: …（←ここが最大の学び）

## レール6層と本雛形のファイル対応

| 層 | 役割（AIの失敗をどう防ぐか） | ファイル | 由来（既存成果物） |
|---|---|---|---|
| ① グラウンディング（②照合） | 事実は真実源に当てる | `rent_schema.yaml` の `sources` | `mock/spec.md §5-2 参照元` |
| ② ケイパビリティ/状態（③把握） | 現行システムの能力/制約を知る | `capabilities.yaml` | rent R-002（予算期間機能なし） |
| ③ 不変条件/制約（④線引き） | 無効な行動を禁止 | `guardrails.yaml` | rent R-004 / `mock/spec.md §8` |
| ④ 決定論計算（演算分離） | LLMに算術させない | `calc_rent.tool.yaml` | `mock/spec.md §6 計算ロジック` |
| ⑤ ポリシー/エスカレーション（⑤先読み） | リスク時に止める/人に渡す | `guardrails.yaml` の `policies` | rent R-002/R-003、暫定仕様 |
| ⑥ 評価/クリティック（①却下） | 出力を現実基準で採点 | `rent_eval.tsv` + `critic_rubric.md` | `verification/` + `JUDGMENT_LOG.md` |
| ⑦ 実行状態機械（ワークフロー） | 中断・再開・スキップ・上書きガード・冪等性 | `workflow.yaml` | timerecord `save_draft_spec_v2.tsv` の状態遷移設計 |

## 使い方（エージェント開発時の参照イメージ）

1. エージェントは賃料額を **自分で計算しない**。`calc_rent` ツール（決定論）を呼ぶ
2. 行動前に `guardrails.yaml` の invariants を満たすか検証（過去月誤適用などを禁止）
3. `capabilities.yaml` を前提知識として読み、「予算期間機能はまだ無い」を知った上で振る舞う
4. マルチステップ実行は `workflow.yaml` の状態機械に従う（tentative→commit_gate→committed、中断時は resume_contract で再開、未分類状態は escalate）
5. 出力は `rent_eval.tsv` のゴールデンセットと `critic_rubric.md` で自動採点

## 要確認ポイント（推測を含む箇所）

- `calc_rent.tool.yaml` の **段階別歩合（tiered）の段階境界・限界税率方式（marginal）か総額方式か** は実仕様で要確認
- `rent_eval.tsv` の期待値は `mock/spec.md §6` の式から **本雛形で導出**したもの。`verification/` の実検証値と突合して確定すること
- `capabilities.yaml` の各 `available: false` は 2026-03 暫定仕様時点の推定。現在の本番状態は要確認
