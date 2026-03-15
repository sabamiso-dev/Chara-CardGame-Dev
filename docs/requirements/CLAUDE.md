# 要件定義書の運用ルール

このフォルダ（`docs/requirements/`）はゲームの **仕様・ルール・データモデル** を定義する。
設計書（`docs/design/`）の根拠となるドキュメント群。

---

## ファイル構成と役割

| ファイル | 役割 | 変更頻度 |
|---|---|---|
| `00_overview.md` | ゲーム概要・コンセプト・用語定義 | 低 |
| `01_core_rules.md` | 勝敗条件・デッキ構成・リソース等の基本ルール | 中 |
| `02_cards.md` | カード種別（スキル/召喚/アイテム/サポート/ユニーク）・トリガー・priority | 高 |
| `03_summons.md` | 召喚オブジェクト/ユニットの仕様・入れ替え・ゾーン管理 | 中 |
| `04_colors.md` | 7色の方向性・デッキ制約 | 低 |
| `05_burst_and_support.md` | バーストゲージ・バースト状態・サポートカード | 中 |
| `06_turn_flow.md` | ターン進行（5フェイズ）・解決順・同時操作 | 高 |
| `07_data_model.md` | データモデル・イベント定義・EffectSpec・ステータス修正パイプライン | 高 |
| `08_nonfunctional_and_milestones.md` | 非機能要件・マイルストーン | 低 |
| `09_open_questions.md` | **TBD集約先** — 未決定事項はすべてここに記録 | 高 |
| `10_team_and_3v3_battle.md` | 3vs3編成・AP・同時操作・キャラステータス | 中 |
| `11_questions_and_proposals.md` | **Q&A集約先** — 疑問→回答の確定記録 | 高 |
| `12_1v1_regulation.md` | 1vs1方式Bの固有ルール | 低 |
| `13_github_and_claude_workflow.md` | GitHub・Claude運用ガイド | 低 |
| `14_content_baselines_and_data_workflow.md` | 基礎ステータス目安・データ追加手順 | 中 |
| `15_damage_formula.md` | ダメージ計算式・クリティカル・状態異常命中率 | 低 |

---

## 編集ルール

### 仕様変更時の整合性
- 1つのファイルを変更したら、**関連ファイルも同時に更新**する
- 特に以下の組み合わせは相互参照が多いため注意:
  - `02_cards.md` ↔ `07_data_model.md`（カード定義 ↔ データモデル）
  - `06_turn_flow.md` ↔ `07_data_model.md`（フェイズ処理 ↔ イベント定義）
  - `03_summons.md` ↔ `05_burst_and_support.md`（召喚バースト変化）
  - `01_core_rules.md` ↔ `10_team_and_3v3_battle.md`（基本ルール ↔ 3vs3詳細）

### TBD管理
- 未決定事項は **`09_open_questions.md` に集約**する（各ファイル内にTBDを散在させない）
- 各ファイル内に「TBD」と書く場合は、`09_open_questions.md` への参照リンクを付ける

### 疑問・回答の記録
- 仕様に関する疑問と回答（確定事項）は **`11_questions_and_proposals.md` に集約**する
- セクション0「決定済み（反映済み）」に確定事項のサマリーがある — 既に反映済みの項目を再議論しない

### 確定事項の保護
- `11_questions_and_proposals.md` のセクション0に列挙されている**確定事項を勝手に変更しない**
- 確定事項を覆す場合は、理由を明記して `11_questions_and_proposals.md` に新しい疑問として起票する

### 新規データ追加
- キャラ/カード/召喚の追加は `14_content_baselines_and_data_workflow.md` の手順に従う
- ID命名規約（`skill_<color>_<name>` 等）を守る

---

## 設計書との関係

- 要件定義書は **「何を作るか」** を定義する
- 設計書（`docs/design/`）は **「どう作るか」** を定義する
- 要件と設計にずれが見つかった場合、**要件を先に確認・更新**してから設計を修正する
