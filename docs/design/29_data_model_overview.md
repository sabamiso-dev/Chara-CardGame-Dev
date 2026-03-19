# データモデル設計（全体マップ）

全エンティティの関係・制約・ライフサイクルを1箇所に整理する。
要件定義（`docs/requirements/07_data_model.md`）とC#クラス設計（`25_csharp_classes.md`）の橋渡し。

関連: [要件: データモデル](../requirements/07_data_model.md) / [C#クラス設計](25_csharp_classes.md) / [状態遷移](27_state_transitions.md)

---

## 1. エンティティ関係図

```
GameState
├── Turn, Phase, InitiativeTeamId, GlobalSeqCounter
├── Rng (IRngProvider)
├── EventQueue (IEventQueue)
│
└── Teams[2] ─── TeamState
                  ├── Id
                  ├── Resources ─── ResourcesState
                  │                  └── BurstPoints / BurstMaxPoints
                  │
                  ├── Deck[]  ──┐
                  ├── Hand[]    ├── CardId（ゾーン間を移動する）
                  ├── Graveyard[] ┤
                  └── Banished[]──┘
                  │
                  ├── SupportSet[max 3] ─── SupportSlotState
                  │                          ├── CardId
                  │                          ├── CooldownRemainingTurns
                  │                          └── CreatedSeq
                  │
                  ├── PlannedActions[] ─── PlannedAction（準備フェイズの予約）
                  ├── Decision ─── PreparationDecisionState
                  ├── FieldEffect ─── FieldEffectInstance（1枠。nullable）
                  ├── LeaderCharacterId（1vs1方式Bのみ。nullable）
                  │
                  └── Characters[3] ─── CharacterState
                                        ├── Id, Hp, MaxHp, IsDown
                                        ├── Stats ─── CharacterStats
                                        │              └── pAtk, pDef, mAtk, mDef, spd, tech
                                        ├── Ap, UnlockedMaxAP, ApCap, ApRegen
                                        │
                                        ├── Statuses[] ─── StatusEffect（0〜N件）
                                        ├── Passives[] ─── CharacterPassiveInstance[3]
                                        ├── LoadoutPassives[] ─── LoadoutPassiveInstance
                                        │
                                        ├── Unique ─── UniqueSlotState（1枠）
                                        ├── Item ─── ItemInstance（1枠。nullable）
                                        ├── Shield ─── ShieldInstance（1枠。nullable）
                                        ├── Burst ─── BurstState
                                        ├── Elements ─── ElementCounters
                                        │
                                        └── SummonSlot ─── SummonSlotState（1枠。nullable）
                                                           ├── Object ─── SummonObjectInstance（nullable）
                                                           └── Unit ─── SummonUnitInstance（nullable）
                                                                         ※ Object と Unit は同時に存在しない
```

---

## 2. 定義データ（マスターデータ）

バトル前に読み込まれ、バトル中は変更されない。

### 2.1 キャラ定義

```
CharacterDefinition
├── Id                     例: "char_kain_striker"
├── Name                   例: "カイン"
├── ColorIdentity          例: "red"
├── Role                   例: "物理アタッカー"
├── BaseStats ─── CharacterStats
│                 └── pAtk, pDef, mAtk, mDef, spd, tech
├── MaxHp                  例: 1200
├── ApStart                例: 3（初期AP）
├── ApMaxStart             例: 3（初期最大AP）
├── ApCap                  例: 9（AP上限）
├── ApRegen                例: 2（毎ターン回復量）
├── Passives[3] ─── PassiveConfig
│                    ├── PassiveId
│                    └── UnlockAtUnlockedMaxAP  例: 4, 6, 8
└── UniqueChoiceIds[3]     例: ["unique_kain_flash_strike", ...]
```

### 2.2 カード定義（共通 + 種別固有）

```
CardDefinition（共通フィールド）
├── Id                     例: "skill_red_slash"
├── Name                   例: "斬撃"
├── Type ─── CardType      Skill / SummonObject / SummonUnit / Item / Support / Unique
├── Color                  "red" / "blue" / ... / "colorless"
├── Cost                   AP消費量
├── Priority               -3 〜 +3
├── RulesText              ルールテキスト
├── Tags[]                 ["攻撃", "物理", "遺物", "バディ", ...]
├── Trigger? ─── CardTrigger（トリガー付きカードのみ）
├── ElementCost?           { red: 2 } 等
├── ElementGain?           { red: 1 } 等（通常はルール側で自動処理）
└── SetEffects? ─── EffectSpec[]（セット効果）

── 種別ごとの追加フィールド ──

SkillCardDefinition
└── Effect ─── EffectSpec

SummonObjectCardDefinition
├── DurationTurns          寿命
├── ObjectScaling ─── ScalingSpec    { pAtk: 0.3, pDef: 0.2, ... }
├── ObjectFlatStats ─── FlatStatBonus { pAtk: 10, pDef: 5, ... }
├── OnEnterEffects[] ─── EffectSpec  登場時効果
└── BurstVariantCardId?    バースト版のカードID

SummonUnitCardDefinition
├── DurationTurns
├── UnitScaling ─── ScalingSpec
├── UnitFlatStats ─── FlatStatBonus
├── OnEnterEffects[]
├── ActionSkills[] ─── ActionSkillDefinition   0コスト行動スキル
├── Passives[]         パッシブID
└── BurstVariantCardId?

SupportCardDefinition
├── BurstCostPoints        例: 200（2ゲージ）
├── CooldownTurns          例: 3
└── Effect ─── EffectSpec

ItemCardDefinition
├── Count                  使用回数（例: 3）
├── Triggers[] ─── CardTrigger    自動発動条件
└── Effect ─── EffectSpec

UniqueCardDefinition
├── CooldownTurns          例: 3
├── Effect ─── EffectSpec
└── BurstVariantCardId?    バースト強化版
```

### 2.3 その他の定義データ

```
StatusEffectDefinition         状態異常/バフのテンプレート
├── Id, Name, Category (buff/debuff)
├── StackingRule (replace/extend/stack + maxStacks)
├── DefaultDuration, BaseHitRate
├── Effect ─── StatusEffectType (dot/statMod/actionBlock/misc)
├── Triggers[], Tags[]

FieldEffectDefinition          フィールド効果のテンプレート
├── Id, Name, DefaultDurationTurns
├── Scope (ownTeam/enemyTeam/bothTeams)
├── Modifiers[], Triggers[], Tags[]

ActionSkillDefinition          召喚ユニットの0コスト行動
├── Id, Name, Priority
├── Effect ─── EffectSpec

PassiveDefinition              キャラパッシブの定義
├── Id, Name, Type (Constant/Triggered/OnActivate)
├── Modifiers[], Triggers[]
├── OnActivateEffect? ─── EffectSpec

RegulationDefinition           レギュレーション設定
├── TeamSize, CharacterDeckSize, SupportSetMax
├── MaxTurns, PreparationTimeSec, InitialHandSize
├── ApStart, BurstMax, BurstMaxPoints, ActionChargePoints
├── DefenseCoefficient, VictoryCondition
├── ColorRestriction, SameNameRule, ColorlessCardLimit
├── BannedCards[], RestrictedCards[]
├── OneVsOneRules? ─── OneVsOneRegulation
```

---

## 3. バトル中インスタンス（実行時データ）

バトル中に生成・変更・消滅するデータ。

### 3.1 スロット制約

| エンティティ | 所属先 | 最大数 | 上書きルール |
|---|---|---|---|
| **StatusEffect** | CharacterState.Statuses | 無制限 | StackingRule に従う |
| **ItemInstance** | CharacterState.Item | **1** | 新規セットで既存を上書き除去 |
| **ShieldInstance** | CharacterState.Shield | **1** | 新規付与で既存を上書き |
| **SummonObjectInstance** | SummonSlot.Object | **1** | 新規召喚で上書き除去 |
| **SummonUnitInstance** | SummonSlot.Unit | **1** | 新規召喚で上書き除去 |
| **FieldEffectInstance** | TeamState.FieldEffect | **1** | 新規展開で上書き |
| **SupportSlotState** | TeamState.SupportSet | **3** | 編成時に固定 |
| **UniqueSlotState** | CharacterState.Unique | **1** | 編成時に固定 |
| **CharacterPassiveInstance** | CharacterState.Passives | **3** | 編成時に固定 |

### 3.2 ゾーン管理（カード）

```
カードのライフサイクル:

  Deck（山札）
    │
    ├──(ドロー)──→ Hand（手札）
    │                │
    │                ├──(スキル/召喚/アイテム使用)──→ Graveyard（トラッシュ）
    │                │
    │                └──(Discard効果)──→ Graveyard
    │
    └──(デッキ0枚時)──← Graveyard（トラッシュ戻し→シャッフル）

  Banished（追放ゾーン）
    ← 召喚実体の消滅時
    ← トークンカード（【消失】）の使用後
    ※ トラッシュ戻しの対象外
```

| ゾーン | 公開情報 | トラッシュ戻し対象 |
|---|---|---|
| Deck | 非公開 | — |
| Hand | 非公開 | — |
| Graveyard | 公開 | 対象 |
| Banished | 公開 | **対象外** |

### 3.3 CreatedSeq の用途

`GlobalSeqCounter` からインクリメントして付与される生成連番。以下の用途で使用:

| 用途 | 参照箇所 |
|---|---|
| ターン終了時の処理順 | 全永続効果を createdSeq 昇順で処理 |
| StatCalculator 同カテゴリ内の適用順 | additive/multiplicative 内で createdSeq 昇順 |
| 同時誘発の最終タイブレーク | 同一チーム内の createdSeq 昇順 |
| 行動ソートの最終タイブレーク | 同一チーム内の createdSeq 昇順 |

**CreatedSeq を持つエンティティ一覧:**
BurstState, SummonObjectInstance, SummonUnitInstance, ItemInstance, ShieldInstance, FieldEffectInstance, StatusEffect, SupportSlotState, UniqueSlotState

---

## 4. EffectSpec 体系（効果定義）

### 4.1 EffectType 一覧（23種）

| # | EffectType | カテゴリ | 入力 | 出力 | 委譲先 |
|---|---|---|---|---|---|
| 1 | `Damage` | 戦闘 | damageType, multiplier, hitCount | HP減少, DamageDealt | DamageCalculator |
| 2 | `Heal` | 戦闘 | calcMode, value | HP増加, Healed | EffectResolver内部 |
| 3 | `Draw` | 手札 | count | 手札増加, CardDrawn | DeckManager |
| 4 | `Discard` | 手札 | count, targetScope, selection | 手札減少, CardDiscarded | DeckManager |
| 5 | `StatusApply` | 状態 | statusId, duration, stacks | 状態付与, StatusApplied | StatusEffectProcessor |
| 6 | `Dispel` | 状態 | category, count, tags | 状態解除, StatusExpired | StatusEffectProcessor |
| 7 | `Summon` | 召喚 | summonCardId | 召喚生成, SummonEntered | SummonManager |
| 8 | `ElementGain` | リソース | element, amount | エレメント増, ElementGained | EffectResolver内部 |
| 9 | `ElementConsume` | リソース | element, amount | エレメント減, ElementConsumed | EffectResolver内部 |
| 10 | `ElementConvert` | リソース | from, to, amount | エレメント変換, ElementConverted | EffectResolver内部 |
| 11 | `BurstGain` | リソース | points | バースト増, — | BurstManager |
| 12 | `Shield` | 防御 | shieldType, value, duration | シールド付与, ShieldApplied | ShieldProcessor |
| 13 | `Resurrect` | 状態 | hpPercent | 蘇生, CharacterResurrected | EffectResolver内部 |
| 14 | `FieldEffectDeploy` | フィールド | fieldEffectId, duration | 展開, FieldEffectDeployed | FieldEffectManager |
| 15 | `ItemSet` | アイテム | itemCardId | セット, ItemSet | ItemManager |
| 16 | `APModify` | リソース | amount, targetStat | AP増減, APModified | EffectResolver内部 |
| 17 | `Bounce` | フィールド | targetType, count, destination | バウンス, CardBounced | EffectResolver内部 |
| 18 | `DeckManipulate` | デッキ | action, count, filterTags | デッキ操作, DeckManipulated | DeckManager |
| 19 | `DeckSearch` | デッキ | filterTags, filterType, action | サーチ, DeckSearched | DeckManager |
| 20 | `TokenCreate` | 手札 | templateCardId, color, tags | トークン生成, TokenCreated | EffectResolver内部 |
| 21 | `Reveal` | 情報 | targetScope, count, duration | 情報公開, HandRevealed | EffectResolver内部 |
| 22 | `Composite` | 制御 | children[] | 子効果を順次解決 | 再帰呼び出し |
| 23 | `Conditional` | 制御 | condition, then, else | 条件分岐 | 条件評価→再帰 |

### 4.2 セット効果で使用可能な EffectType

| 許可 | 禁止 |
|---|---|
| StatusApply, Draw, BurstGain, ElementGain, APModify, Composite, Conditional | 上記以外の全て（バトル中のリアルタイム効果を前提とするため） |

### 4.3 ConditionType 一覧（6種）

| ConditionType | 判定対象 | 例 |
|---|---|---|
| `ElementThreshold` | キャラのエレメント数 | 赤エレメント≥3 |
| `HpThreshold` | HP割合 | HP≤50% |
| `StatusCheck` | 状態の有無 | 対象が毒状態か |
| `BurstCheck` | バースト状態 | 自分がバースト中か |
| `AllyDownCount` | 味方ダウン数 | 味方が2体以上ダウン |
| `HandCount` | 手札枚数 | 手札≤2枚 |

---

## 5. イベント一覧

### 5.1 全イベント（カテゴリ別）

**ゲーム進行:**
`MatchStarted`, `TurnStarted`, `TurnEnded`, `PhaseChanged`

**行動:**
`ActionReserved`, `ActionResolved`, `CardPlayed`, `UnitActed`, `SupportActivated`

**戦闘:**
`DamageDealt`, `Healed`, `CharacterDowned`, `CharacterResurrected`

**状態:**
`StatusApplied`, `StatusExpired`

**シールド:**
`ShieldApplied`, `ShieldConsumed`, `ShieldExpired`

**召喚:**
`SummonObjectEntered`, `SummonObjectExpired`, `SummonObjectDestroyed`
`SummonUnitEntered`, `SummonUnitExpired`, `SummonReplaced`

**アイテム:**
`ItemSet`, `ItemTriggered`, `ItemExhausted`, `ItemOverwritten`

**フィールド効果:**
`FieldEffectDeployed`, `FieldEffectOverwritten`, `FieldEffectExpired`, `FieldEffectRemoved`

**リソース:**
`ElementGained`, `ElementConsumed`, `ElementConverted`, `APModified`

**手札・デッキ:**
`CardDrawn`, `CardDiscarded`, `CardBounced`, `DeckManipulated`, `DeckSearched`, `TokenCreated`, `HandRevealed`, `DeckReshuffledFromTrash`

**バースト:**
`BurstModeApplied`, `BurstModeRemoved`

**パッシブ:**
`PassiveUnlocked`

**診断/ログ:**
`StatRecomputed`

---

## 6. 編成データ（MatchConfig）

マッチ開始時に必要な入力データ。

```
MatchConfig
├── Regulation ─── RegulationDefinition
├── RngSeed                    リプレイ用シード
├── Team1 ─── TeamConfig
│              ├── TeamId
│              ├── Characters[3] ─── CharacterConfig
│              │                      ├── CharacterId
│              │                      ├── DeckCardIds[10〜15]
│              │                      └── UniqueCardId
│              └── SupportCardIds[max 3]
└── Team2 ─── TeamConfig
```

### バリデーション（LoadoutValidator）

| チェック | 条件 |
|---|---|
| チーム人数 | == teamSize |
| デッキ枚数 | characterDeckSize.min ≤ N ≤ max |
| サポート枠 | ≤ supportSetMax |
| 同名制限 | キャラデッキ内: perCharacterDeck / チーム全体: perTeam |
| 色制約 | maxColors / singleOnly |
| 無色制限 | ratio / fixed |
| 禁止/制限カード | bannedCards / restrictedCards |
| ユニーク候補 | キャラの UniqueChoiceIds に含まれるか |

---

## 7. リプレイデータ

```
ReplayData
├── Version                    フォーマットバージョン
├── Config ─── MatchConfig     マッチ設定（RngSeed含む）
├── TurnInputs[] ─── TurnInput
│                      ├── Turn
│                      ├── Team1Input ─── PreparationInput
│                      │                   ├── Actions[] ─── PlannedAction
│                      │                   ├── SubmittedAtMs
│                      │                   ├── BurstTargetCharacterId?
│                      │                   ├── SupportActivateCardId?
│                      │                   └── ApSupply? (1vs1のみ)
│                      └── Team2Input ─── PreparationInput
└── ExpectedResult ─── MatchResult
```

---

## 8. データフロー概要

```
【編成時】
  CharacterDefinition + CardDefinition
    → LoadoutValidator で検証
    → MatchConfig として GameManager に渡す

【MatchSetup】
  MatchConfig
    → GameState 初期化
    → セット効果解決（LoadoutPassive → StatCalculator）
    → チームデッキ作成（合体→シャッフル）
    → 初期手札ドロー

【バトル中】
  PlannedAction（入力）
    → ActionResolver（ソート→不発チェック→実行）
      → EffectResolver（EffectSpec → 各モジュール委譲）
        → GameState 更新
        → GameEvent 発行
        → TriggerSystem スキャン
    → EventQueue 処理（連鎖解決）

【ターン終了】
  全永続効果（createdSeq 昇順）
    → 持続カウント減少
    → 期限切れ処理（消滅→イベント→トリガー）
    → 勝敗判定
```
