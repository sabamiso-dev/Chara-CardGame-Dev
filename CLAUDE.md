# プロジェクト指示（CLAUDE.md）

## プロジェクト概要
- 「デュアル・レイヤー・バトル（仮）」— Unity (C#) 製デジタルカードゲーム
- 3vs3チーム戦 + 1vs1レギュレーション / 同時操作 / バーストシステム
- やり取りは日本語で行う

## 仕様書の構成
| ファイル | 内容 |
|---|---|
| 00_overview.md | ゲーム全体の概要・コンセプト |
| 01_core_rules.md | コアルール（バトルの基本） |
| 02_cards.md | カード種別・構造 |
| 03_summons.md | 召喚獣の仕様 |
| 04_colors.md | 属性・色システム |
| 05_burst_and_support.md | バースト・サポートシステム |
| 06_turn_flow.md | ターン進行フロー |
| 07_data_model.md | データモデル・イベント定義・EffectSpec・ShieldInstance・FieldEffectDefinition |
| 08_nonfunctional_and_milestones.md | 非機能要件・マイルストーン |
| 09_open_questions.md | 未決事項（TBD集約先） |
| 10_team_and_3v3_battle.md | チーム戦・3vs3ルール |
| 11_questions_and_proposals.md | 質疑応答・提案の記録 |
| 12_1v1_regulation.md | 1vs1レギュレーション |
| 13_github_and_claude_workflow.md | GitHub・Claude連携ワークフロー |
| 14_content_baselines_and_data_workflow.md | コンテンツ基準値・データ投入手順 |
| 15_damage_formula.md | ダメージ計算式 |
| 16_rule_engine_design.md | ルールエンジン設計（Phase 1 実装の設計図・全22セクション・C#クラス設計含む） |

## 仕様変更ルール
- 仕様変更時は関連ドキュメントの整合性を同時に取る
- TBDは 09_open_questions.md に集約
- 疑問・回答は 11_questions_and_proposals.md に集約
- 新規データ追加は 14_content_baselines_and_data_workflow.md の手順に従う

## 開発フェーズ
- Phase 0: 仕様固め（現在）
- Phase 1: ルールエンジン（UIなし）
- Phase 2: UIプロトタイプ
- Phase 3: コンテンツ投入

## コーディング規約（最小・Phase 1以降で拡充）
- 言語: C# / Unity
- ルールエンジンは純粋関数に近い形でユニットテスト可能にする
- イベント駆動アーキテクチャ（06_turn_flow.md / 07_data_model.md 準拠）
- 命名: PascalCase（クラス/メソッド）、camelCase（変数/フィールド）

## Git運用（個人開発）
- 秘密情報（APIキー、トークン、.env等）をコミットしない
- コミットメッセージは変更内容がわかる簡潔な記述
- ブランチ/PRは推奨だが必須ではない
