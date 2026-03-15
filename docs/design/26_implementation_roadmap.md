# Phase 1 実装ロードマップ

ルールエンジン設計の一部。モジュール実装順序・テスト方針・統合テスト計画を定義する。

関連: [アーキテクチャ概要](16_architecture_overview.md) / [C#クラス設計](25_csharp_classes.md)

---

## 10.10. Phase 1 実装ロードマップ

### 目的

Phase 1（ルールエンジン実装）のモジュール実装順序を定義する。
依存関係の少ない下位モジュールから積み上げ、各ステップで単体テストを書きながら進める。

### 実装順序

```
Step 1: 基盤層（依存なし）
├── RngProvider（SeededRng）
├── EventQueue
├── GameState / TeamState / CharacterState 等のデータモデル
└── 各種 enum / 定数定義
    テスト: RNG の決定性, EventQueue の FIFO 動作

Step 2: 計算層（基盤に依存）
├── StatCalculator（5段階パイプライン）
├── DamageCalculator（基本式 + クリティカル + 状態異常命中）
└── ShieldProcessor（付与 / 吸収 / 期限切れ）
    テスト: ステータス計算の精度, ダメージ計算の境界値, シールド吸収

Step 3: 状態管理層
├── StatusEffectProcessor（付与 / 解除 / 期限切れ / DoT）
├── DeckManager（ドロー / 再構築 / ゾーン移動）
└── LoadoutValidator（編成バリデーション）
    テスト: 重ねがけルール, デッキ再構築の決定性, バリデーション全パターン

Step 4: エンティティ管理層
├── SummonManager（生成 / 消滅 / 上書き / バースト変化）
├── BurstManager（ゲージ管理 / バースト状態適用・解除）
├── ItemManager（セット / 自動発動 / 破棄）
└── FieldEffectManager（展開 / 消滅 / 上書き）
    テスト: 召喚入れ替え, バースト状態遷移, アイテム自動発動

Step 5: 効果解決層
├── EffectResolver（17種の EffectType ディスパッチ）
│   ├── 単純型（Damage, Heal, Draw, StatusApply, Dispel, etc.）を先に
│   └── 再帰型（Composite, Conditional）を後に
└── TriggerSystem（スキャン / 条件マッチ / 割り込み解決）
    テスト: 各 EffectType の解決, Composite 再帰, Conditional 分岐,
           トリガー割り込み, 無限ループ対策

Step 6: オーケストレーション層
├── ActionResolver（ソート / 不発判定 / 順次解決）
├── TurnProcessor（フェイズ遷移 / 各フェイズ処理）
└── GameManager（MatchSetup / ターンループ / 勝敗判定）
    テスト: 行動ソートの全タイブレーク, 不発パターン,
           1ターン通しテスト, 複数ターン通しテスト,
           勝敗判定（allDown / timeout / 蘇生後）

Step 7: 統合テスト
├── フルマッチシミュレーション（3〜5ターン）
├── リプレイ再現テスト（同一 seed → 同一結果）
├── エッジケース:
│   ├── 全キャラ同時ダウン → タイブレーク
│   ├── 蘇生後の勝敗再判定
│   ├── デッキ再構築を跨ぐドロー
│   ├── Composite 内で対象ダウン → 後続効果の処理
│   └── 同一イベントで複数トリガー同時成立
└── パフォーマンス: 15ターンフルマッチの処理時間計測
```

### テスト方針

| 方針 | 内容 |
|---|---|
| **純粋関数テスト** | 各モジュールは GameState を受け取り変更後を返す。入出力で検証 |
| **決定的テスト** | RngProvider の seed 固定で乱数依存テストも決定的に |
| **インターフェース経由** | モック差し替えで単体テストを独立実行可能 |
| **イベントログ検証** | 処理後の EventQueue に期待するイベント列が記録されているかで検証 |
| **スナップショット比較** | GameState のディープコピーで処理前後の差分を検証 |

---


## 18. 実装ロードマップ改訂（セクション13〜17の統合）

セクション10.10の実装ロードマップに、新規セクションを統合する。

```
Step 1: 基盤層（変更なし）
├── RngProvider, EventQueue, データモデル, enum/定数
└── ★ DefinitionRepository / DefinitionLoader を追加

Step 2: 計算層（変更なし）
├── StatCalculator, DamageCalculator, ShieldProcessor

Step 3: 状態管理層（変更なし）
├── StatusEffectProcessor, DeckManager, LoadoutValidator

Step 4: エンティティ管理層
├── SummonManager, BurstManager, ItemManager, FieldEffectManager
└── ★ PassiveManager（パッシブ解放判定）を追加

Step 5: 効果解決層
├── EffectResolver, TriggerSystem
└── ★ ActionSkillDefinition のディスパッチ対応を追加

Step 6: オーケストレーション層
├── ActionResolver, TurnProcessor, GameManager
├── ★ 1vs1分岐ロジック（TeamStateExtensions）を追加
├── ★ AP供給処理を追加
└── ★ 降参/切断/タイムアウト処理を追加

Step 7: 統合テスト
├── 既存テストケース
├── ★ 1vs1方式Bのフルマッチテスト
├── ★ パッシブ解放タイミングテスト
├── ★ 召喚ユニット行動のソート・解決テスト
├── ★ 召喚入れ替え時の行動取消しテスト
└── ★ 降参/タイムアウトテスト
```
