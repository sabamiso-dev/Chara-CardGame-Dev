# 設計書（`docs/design/`）

Phase 1（ルールエンジン実装）の設計図。
既存仕様書（`docs/requirements/`）の処理ルールを統合し、C# 実装の全体像を定義する。

---

## ファイル構成

| ファイル | 内容 | 主要セクション |
|---|---|---|
| [16_architecture_overview.md](16_architecture_overview.md) | アーキテクチャ概要 | 設計原則、全体構成図、モジュール責務一覧、依存関係 |
| [17_game_loop_and_phases.md](17_game_loop_and_phases.md) | ゲームループとフェイズ設計 | フェイズ遷移図、各フェイズ処理、GameManager、AP管理ルール |
| [18_event_and_trigger.md](18_event_and_trigger.md) | イベントシステムとトリガー判定 | EventQueue、トリガーソース、スキャンフロー、割り込み解決、無限ループ対策 |
| [19_action_resolver.md](19_action_resolver.md) | 行動解決パイプライン | ソートキー（タイブレークチェーン）、不発チェック、解決フロー |
| [20_stat_damage_effect.md](20_stat_damage_effect.md) | ステータス・ダメージ・効果解決 | StatCalculator（5段階パイプライン）、DamageCalculator、EffectResolver、データフロー例 |
| [21_summon_and_burst.md](21_summon_and_burst.md) | 召喚管理・バーストシステム | SummonManager、BurstManager、召喚ユニット行動管理 |
| [22_subsystems.md](22_subsystems.md) | サブシステム | DeckManager、StatusEffectProcessor、ShieldProcessor、ItemManager、FieldEffectManager |
| [23_validation_rng_loader.md](23_validation_rng_loader.md) | バリデーション・乱数・データローダー | LoadoutValidator、RngProvider、DefinitionLoader/Repository |
| [24_1v1_passive_misc.md](24_1v1_passive_misc.md) | 1vs1・パッシブ・降参/切断 | 1vs1方式B対応、キャラパッシブシステム、降参/切断/タイムアウト |
| [25_csharp_classes.md](25_csharp_classes.md) | C# クラス/インターフェース設計 | 全データモデル、インターフェース、enum、EffectSpec関連 |
| [26_implementation_roadmap.md](26_implementation_roadmap.md) | Phase 1 実装ロードマップ | Step 1〜7 の実装順序、テスト方針 |
| [27_state_transitions.md](27_state_transitions.md) | 状態遷移と連鎖解決フロー | ダメージ→ダウン連鎖、バースト適用/解除、ターン終了処理順、蘇生 |
| [28_error_log_replay.md](28_error_log_replay.md) | エラーハンドリング・ログ・リプレイ | エラー3層分類、公開/内部ログ二層、リプレイデータ、診断機能 |

---

## 読み方ガイド

- **初めて読む人**: `16_architecture_overview.md` → `17_game_loop_and_phases.md` → `19_action_resolver.md` の順で全体像を把握
- **実装に着手する人**: `26_implementation_roadmap.md` で実装順序を確認 → `25_csharp_classes.md` でクラス設計を参照
- **特定システムを調べたい人**: 上記の表から該当ファイルへ直接アクセス

## 仕様書との対応

設計書は以下の仕様書（`docs/requirements/`）を根拠としている。

| 設計書 | 主な参照仕様書 |
|---|---|
| ゲームループ | `06_turn_flow.md`, `10_team_and_3v3_battle.md` |
| 行動解決 | `06_turn_flow.md`, `02_cards.md` |
| ステータス・ダメージ | `07_data_model.md`, `15_damage_formula.md` |
| 効果解決 | `07_data_model.md`, `02_cards.md` |
| 召喚・バースト | `03_summons.md`, `05_burst_and_support.md` |
| サブシステム | `02_cards.md`, `07_data_model.md` |
| 1vs1・パッシブ | `12_1v1_regulation.md`, `10_team_and_3v3_battle.md` |
| データモデル | `07_data_model.md` |
