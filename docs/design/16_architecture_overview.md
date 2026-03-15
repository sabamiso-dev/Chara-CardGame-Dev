# アーキテクチャ概要

Phase 1（ルールエンジン実装）の設計図。
既存仕様書（`docs/requirements/`）の処理ルールを統合し、C# 実装の全体像を定義する。

関連: [ゲームループ](17_game_loop_and_phases.md) / [C#クラス設計](25_csharp_classes.md) / [実装ロードマップ](26_implementation_roadmap.md) / [設計書一覧](README.md)

---

## 1. アーキテクチャ概要

### 設計原則

| 原則 | 内容 |
|---|---|
| **純粋関数に近い形** | 各処理は GameState を受け取り、変更後の GameState を返す。副作用はイベント発行で表現する |
| **イベント駆動** | すべてのゲーム内変化は GameEvent として記録・処理される（`06_turn_flow.md`） |
| **決定的処理** | rngSeed ベースの乱数により、同一入力から同一結果を再現可能（リプレイ対応） |
| **テスト容易性** | UI 非依存。インターフェース経由で各モジュールを差し替え・モック可能 |

### 全体構成図

```
┌─────────────────────────────────────────────────────────┐
│                     GameManager                         │
│  （ゲーム全体の進行制御・マッチ開始〜終了）              │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│                    TurnProcessor                        │
│  （フェイズ遷移・各フェイズ処理のオーケストレーション）  │
│                                                         │
│  ┌──────────┐ ┌──────────────┐ ┌─────────────────────┐  │
│  │DeckMgr   │ │ActionResolver│ │StatusEffectProcessor│  │
│  │(ドロー/  │ │(行動ソート/  │ │(状態異常/バフの     │  │
│  │ シャッフル│ │ 不発判定/    │ │ 付与/解除/期限切れ) │  │
│  │ /ゾーン)  │ │ 順次解決)    │ │                     │  │
│  └──────────┘ └──────┬───────┘ └─────────────────────┘  │
│                      │                                   │
│       ┌──────────────┼──────────────┐                   │
│       ▼              ▼              ▼                   │
│  ┌──────────┐ ┌──────────────┐ ┌───────────┐           │
│  │DamageCal │ │StatCalculator│ │TriggerSys │           │
│  │(ダメージ │ │(5段階パイプ  │ │(トリガー  │           │
│  │ 計算)    │ │ ライン)      │ │ 判定/割込)│           │
│  └──────────┘ └──────────────┘ └───────────┘           │
│                                                         │
│  ┌──────────────┐ ┌──────────┐ ┌───────────┐           │
│  │EffectResolver│ │SummonMgr │ │BurstManager│          │
│  │(EffectSpec   │ │(生成/消滅│ │(ゲージ管理/│          │
│  │ ディスパッチ)│ │ /上書き) │ │ バースト)  │          │
│  └──────────────┘ └──────────┘ └───────────┘           │
│                                                         │
│  ┌──────────┐ ┌───────────────┐ ┌───────────┐           │
│  │ItemMgr   │ │FieldEffectMgr │ │ShieldProc │           │
│  │(セット/  │ │(展開/消滅/   │ │(付与/吸収/│           │
│  │ 自動発動)│ │ 上書き)       │ │ 期限切れ) │           │
│  └──────────┘ └───────────────┘ └───────────┘           │
│                                                         │
│  ┌───────────┐                                          │
│  │RngProvider│                                          │
│  │(決定的   │                                          │
│  │ 乱数生成)│                                          │
│  └───────────┘                                          │
└─────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│                     EventQueue                          │
│  （イベント駆動の中核。FIFO キュー）                    │
└─────────────────────────────────────────────────────────┘
```

### モジュール責務一覧

| モジュール | 責務 |
|---|---|
| **GameManager** | マッチの開始・終了制御、GameState 初期化、勝敗判定 |
| **TurnProcessor** | フェイズ遷移、各フェイズ処理の呼び出し |
| **EventQueue** | イベントの蓄積と FIFO 順次処理 |
| **StatCalculator** | ステータス修正パイプライン（5段階） |
| **DamageCalculator** | ダメージ計算（物理/魔法/真ダメージ） |
| **ActionResolver** | 行動ソート・不発判定・順次解決 |
| **EffectResolver** | EffectSpec のディスパッチ・Composite/Conditional の再帰解決 |
| **TriggerSystem** | トリガー条件判定・割り込み処理 |
| **SummonManager** | 召喚の生成/消滅/上書き/バースト変化 |
| **BurstManager** | バーストゲージ管理・バースト状態の適用/解除 |
| **DeckManager** | デッキ操作（ドロー/再構築/ゾーン移動） |
| **StatusEffectProcessor** | 状態異常/バフの付与/解除/期限切れ |
| **ItemManager** | アイテムのセット/自動発動/破棄/上書き |
| **FieldEffectManager** | フィールド効果の展開/消滅/上書き |
| **ShieldProcessor** | シールドの付与/ダメージ吸収/期限切れ |
| **LoadoutValidator** | 編成バリデーション（RegulationDefinition 準拠） |
| **RngProvider** | 決定的乱数生成（seedベース） |

### 依存関係

```
GameManager → TurnProcessor → ActionResolver → EffectResolver → DamageCalculator
                                                               → StatusEffectProcessor
                                                               → DeckManager
                                                               → SummonManager
                                                               → BurstManager
                                                               → ShieldProcessor
                                                               → ItemManager
                                                               → FieldEffectManager
                                             → StatCalculator
                                             → TriggerSystem → ItemManager（アイテム自動発動）
                            → DeckManager
                            → StatusEffectProcessor
                            → SummonManager → StatCalculator
                            → BurstManager → SummonManager
                                           → StatCalculator
                            → ShieldProcessor
                            → ItemManager
                            → FieldEffectManager → StatCalculator
全モジュール → EventQueue（イベント発行）
全モジュール → RngProvider（乱数が必要な場合）
```

---

