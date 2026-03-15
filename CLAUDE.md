# プロジェクト指示（CLAUDE.md）

## プロジェクト概要
- 「デュアル・レイヤー・バトル（仮）」— Unity (C#) 製デジタルカードゲーム
- 3vs3チーム戦 + 1vs1レギュレーション / 同時操作 / バーストシステム
- やり取りは日本語で行う

## ドキュメント構成

### 要件定義書（`docs/requirements/`）— 大まかな設計要件
詳細は [`docs/requirements/CLAUDE.md`](docs/requirements/CLAUDE.md) を参照。

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

### 設計書（`docs/design/`）— 具体的なクラス・メソッド設計
詳細は [`docs/design/CLAUDE.md`](docs/design/CLAUDE.md) および [`docs/design/README.md`](docs/design/README.md) を参照。

| ファイル | 内容 |
|---|---|
| README.md | 設計書一覧・読み方ガイド・仕様書との対応 |
| 16_architecture_overview.md | アーキテクチャ概要（設計原則・全体構成図・モジュール責務・依存関係） |
| 17_game_loop_and_phases.md | ゲームループとフェイズ設計（フェイズ遷移・GameManager・AP管理） |
| 18_event_and_trigger.md | イベントシステムとトリガー判定（EventQueue・TriggerSystem・無限ループ対策） |
| 19_action_resolver.md | 行動解決パイプライン（ソートキー・不発チェック・解決フロー） |
| 20_stat_damage_effect.md | ステータス・ダメージ・効果解決（StatCalculator・DamageCalculator・EffectResolver） |
| 21_summon_and_burst.md | 召喚管理・バーストシステム（SummonManager・BurstManager・召喚ユニット行動） |
| 22_subsystems.md | サブシステム（DeckManager・StatusEffect・Shield・Item・FieldEffect） |
| 23_validation_rng_loader.md | バリデーション・乱数・データローダー（LoadoutValidator・RNG・DefinitionLoader） |
| 24_1v1_passive_misc.md | 1vs1レギュレーション・パッシブ・降参/切断 |
| 25_csharp_classes.md | C# クラス/インターフェース設計（全データモデル・enum・EffectSpec） |
| 26_implementation_roadmap.md | Phase 1 実装ロードマップ（Step 1〜7・テスト方針） |

## 開発フェーズ
- Phase 0: 仕様固め（現在）
- Phase 1: ルールエンジン（UIなし）
- Phase 2: UIプロトタイプ
- Phase 3: コンテンツ投入

## コーディング規約（最小）
- 言語: C# / Unity
- 命名: PascalCase（クラス/メソッド）、camelCase（変数/フィールド）
- 詳細は [`docs/design/CLAUDE.md`](docs/design/CLAUDE.md) を参照

## Git運用（個人開発）
- 秘密情報（APIキー、トークン、.env等）をコミットしない
- コミットメッセージは変更内容がわかる簡潔な記述
- ブランチ/PRは推奨だが必須ではない
