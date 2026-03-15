# 設計書の運用ルール

このフォルダ（`docs/design/`）は Phase 1（ルールエンジン実装）の **C# 設計図** を格納する。
要件定義書（`docs/requirements/`）を根拠とし、クラス・メソッド・処理フローを定義する。

---

## ファイル構成と依存関係

| ファイル | 内容 | 主な参照元（requirements） |
|---|---|---|
| `README.md` | 設計書一覧・読み方ガイド | — |
| `16_architecture_overview.md` | アーキテクチャ概要・モジュール責務・依存関係 | 全体 |
| `17_game_loop_and_phases.md` | フェイズ遷移・GameManager・AP管理 | `06_turn_flow.md`, `10_team_and_3v3_battle.md` |
| `18_event_and_trigger.md` | EventQueue・TriggerSystem・無限ループ対策 | `06_turn_flow.md`, `02_cards.md` |
| `19_action_resolver.md` | ソートキー・不発チェック・解決フロー | `06_turn_flow.md` |
| `20_stat_damage_effect.md` | StatCalculator・DamageCalculator・EffectResolver | `07_data_model.md`, `15_damage_formula.md` |
| `21_summon_and_burst.md` | SummonManager・BurstManager・召喚ユニット行動 | `03_summons.md`, `05_burst_and_support.md` |
| `22_subsystems.md` | Deck・Status・Shield・Item・FieldEffect | `02_cards.md`, `07_data_model.md` |
| `23_validation_rng_loader.md` | LoadoutValidator・RNG・DefinitionLoader | `07_data_model.md` |
| `24_1v1_passive_misc.md` | 1vs1方式B・パッシブ・降参/切断 | `12_1v1_regulation.md`, `10_team_and_3v3_battle.md` |
| `25_csharp_classes.md` | 全C#クラス/インターフェース/enum | `07_data_model.md` |
| `26_implementation_roadmap.md` | 実装ロードマップ（Step 1〜7） | `08_nonfunctional_and_milestones.md` |

---

## 編集ルール

### 要件との整合性
- 設計を変更する前に、**根拠となる要件定義書を必ず確認**する（上表の「主な参照元」を参照）
- 要件と設計にずれを見つけた場合、**要件側を先に確認**する
  - 要件が正しい → 設計を修正
  - 設計が正しい（要件が古い）→ 要件を更新してから設計を修正
- 要件にない独自仕様を設計書に追加しない（必要なら要件側に起票してから）

### ファイル追加・ナンバリング
- 新しい設計書を追加する場合、**連番（27〜）** で命名する
- `README.md` のファイル構成表に追加する
- ルート `CLAUDE.md` の設計書セクションにも追加する

### C# コーディング規約
- 命名: **PascalCase**（クラス/メソッド/プロパティ）、**camelCase**（ローカル変数/フィールド）
- インターフェース接頭辞: `I`（例: `IGameManager`, `IStatCalculator`）
- enum は PascalCase（例: `Phase.TurnStart`, `DamageType.Physical`）
- nullable 参照型はコメントで明示（例: `// nullable`）
- 各モジュールはインターフェース経由で依存し、テスト時にモック差し替え可能にする

### 設計原則
- **純粋関数に近い形**: GameState を受け取り、変更後の GameState を返す
- **イベント駆動**: すべてのゲーム内変化は GameEvent として記録
- **決定的処理**: rngSeed ベースで同一入力→同一結果（リプレイ対応）
- **テスト容易性**: UI非依存、インターフェース経由でモック可能

### 相互参照のルール
- 設計書間で参照する場合は相対リンク（例: `[EffectResolver](20_stat_damage_effect.md)`）を使う
- 要件定義書を参照する場合はパス付き（例: `docs/requirements/07_data_model.md`）で記載
