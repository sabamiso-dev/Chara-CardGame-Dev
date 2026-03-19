## データモデル案（実装用の叩き台）

### 方針
- **カード定義（Definition）** と **ゲーム中の実体（Instance）** を分離する。
- ルール処理は **イベント駆動**（`06_turn_flow.md`）で統一する。

---

## 1) エンティティ一覧（概念）

### GameState
- `turn`: number
- `phase`: enum（TurnStart/Draw/Preparation/Battle/TurnEnd）
- `initiativeTeamId`: string（同時解決/同時誘発の最終タイブレーク用。ゲーム開始時に乱数で決定し、以降固定）
- `eventQueue`: EventQueue（優先権モデルを入れるなら Stack に差し替え可能）
- `teams`: TeamState[2]
- `rngSeed`: number（リプレイ/検証用：TBD）
- `globalSeqCounter`: number（ターン終了処理順を決定するためのグローバル生成連番。永続効果が生成されるたびにインクリメントされ、その効果の `createdSeq` に記録される）

### TeamState
- `id`
- `characters`: CharacterState[3]
- `resources`: ResourcesState（Burst等）
- `deck`: CardId[]（= チームデッキ。キャラデッキ合体後の山札）
- `hand`: CardId[]
- `graveyard`: CardId[]
- `banished`: CardId[]（追放ゾーン。graveyardへ戻さない）
- `supportSet`: SupportSlotState[]（デッキ外 / 最大3。クールダウン管理を含む）
- `plannedActions`: PlannedAction[]（準備フェイズで入力された予約）
- `decision`: PreparationDecisionState
- `fieldEffect`: FieldEffectInstance | null（**フィールド効果**。チームごとに1枠だけ保持でき、新規展開で上書き）

### CharacterState
- `id`
- `hp`, `maxHp`
- `isDown`: boolean
- `stats`: CharacterStats（`10_team_and_3v3_battle.md`）
- `statuses`: StatusEffect[]（バフ/デバフ）
- `ap`: number（現在AP）
- `unlockedMaxAP`: number（解放済み最大AP）
- `apCap`: number（AP上限）
- `apRegen`: number（毎ターン回復量）
- `passives`: CharacterPassiveInstance[]（解放条件つき）
- `loadoutPassives`: LoadoutPassiveInstance[]（カードのセット効果＝構築時パッシブ。編成時点で確定し、バトル中は原則変動しない想定）
- `unique`: UniqueSlotState（装備ユニークとクールダウン）
- `items`: ItemInstance[]（キャラにセットされた装備。**最大1**。新規セット時は既存アイテムを **上書きで除去**）
- `summonSlot`: SummonSlotState | null（キャラに紐づく召喚枠：召喚オブジェクト＋召喚ユニット）
- `burst`: BurstState
- `elements`: ElementCounters
- `shield`: ShieldInstance | null（シールド。最大1枠。新規付与時は既存を上書き）

### CharacterStats（最小）
- `pAtk`, `pDef`
- `mAtk`, `mDef`
- `spd`
- `tech`

### BurstState（キャラのバースト状態）
- `isBurst`: boolean
- `remainingTurns`: number（突入ターンを含めて3）
- `createdSeq`: number（ターン終了処理順の決定に使用）
- `statModifier`: BurstStatMultiplier（倍率式。確定）

### BurstStatMultiplier（確定）
バースト状態のステータス上昇は **倍率式** で適用する。端数は **切り捨て（floor）**。
- `pAtkMultiplier`: **1.15**（pAtk ×1.15）
- `mAtkMultiplier`: **1.15**（mAtk ×1.15）
- `pDefMultiplier`: **1.10**（pDef ×1.10）
- `mDefMultiplier`: **1.10**（mDef ×1.10）
- `spdMultiplier`: **1.10**（spd ×1.10）
- `techMultiplier`: **1.10**（tech ×1.10）
- `maxHp`：**変動なし**（バースト状態でもmaxHpは変わらない）

（例）pAtk=300 のキャラがバースト状態 → `floor(300 × 1.15)` = **345**

### SummonSlotState（召喚枠）
召喚枠は「召喚オブジェクト（配置物）」と「召喚ユニット（戦闘体）」を分けて持つ。

- `object`: SummonObjectInstance | null
- `unit`: SummonUnitInstance | null

（ルール）
- 召喚枠は「オブジェクト/ユニット」で **同一スロットを共有** するため、原則として **`object` と `unit` が同時に非nullになることはない**。
  - 新しい召喚（オブジェクト/ユニット）が展開された場合、既存の召喚は **上書きで除去**される（`SummonReplaced` 等）。

### SummonObjectInstance（召喚オブジェクト）
召喚枠に配置されるオブジェクト。攻撃しない/攻撃されないが、寿命やカード効果で消滅する。

- `instanceId`
- `cardId`（どの召喚カード由来か）
- `ownerTeamId`
- `ownerCharacterId`
- `durationTurnsRemaining`
- `createdSeq`: number（ターン終了処理順の決定に使用）
- `stats`（算出した"オブジェクトのステータス"）
- `appliedStatBonus`（このオブジェクトがキャラへ上乗せしている分：任意。ログ/再現性のために保持してよい）
- `isBurstVariant`: boolean
- `baseVariantCardId`: string | null
- `effectiveSummonCost`: number（このオブジェクトの"現在のコスト"。入れ替え軽減に参照）

### SummonUnitInstance（召喚ユニット）
攻撃/行動するユニット。召喚ユニットが存在する場合、紐づくキャラへの攻撃はユニットへ差し替わる（`03_summons.md`）。

- `instanceId`
- `cardId`（どの召喚カード由来か）
- `ownerTeamId`
- `ownerCharacterId`
- `durationTurnsRemaining`
- `hp`, `maxHp`
- `stats`
- `passives`: PassiveId[]（カード定義から展開）
- `actionSkills`: ActionSkillId[]（カード定義から展開）
- `configuredActionSkillId`: ActionSkillId | null（バトルフェイズで使う「設定済み行動」）
- `hasActedThisTurn`: boolean（ターン開始フェイズでリセット）
- `createdSeq`: number（ターン終了処理順の決定に使用）
- `isBurstVariant`: boolean
- `baseVariantCardId`: string | null

### ItemInstance
- `instanceId`
- `cardId`
- `ownerTeamId`
- `ownerCharacterId`
- `countRemaining`
- `createdSeq`: number（ターン終了処理順の決定に使用）
- `triggers`: TriggerSpec[]（カード定義から展開）
（運用メモ）
- `countRemaining==0` になったアイテムは **破棄** され、スロットが空く。
- 既存アイテムを別アイテムで置き換える場合、既存アイテムは **上書きで除去**（「破棄」とは別扱い）とする。

### CardDefinition（共通）
- `id`, `name`, `type`, `color`, `cost`, `priority`, `rulesText`, `tags`
- `trigger?`: CardTrigger（トリガー条件。任意。詳細は `02_cards.md` セクション7参照）
- `setEffects?`: EffectSpec[]（セット効果＝構築時パッシブ。対戦開始前のロードアウトにセットされているだけで適用される効果）
  - 例：恒常ステ補正、ゲーム開始時に追加ドロー、デメリット付与など
（TBD）トークンカードを「デッキ外生成するカード」として扱う場合、`CardDefinition` と `CardInstance` の分離が必要になる可能性がある。
  - 最低限の方針：トークンはカード効果で生成され、使用後は `banished` に送る（使い切り）。

### CardTrigger（カードのトリガー条件）
トリガーを持つカードは、バトルフェイズ中に条件が満たされると割り込みで早期発動する（詳細は `02_cards.md` セクション7）。

- `name`: string（トリガーの表示名。カード設計で自由に命名。例：「カウンタートリガー」「ダメージトリガー」）
- `event`: string（トリガーイベント。例：`"onDamaged"`, `"onAllyDamaged"`, `"onEnemyAction"`, `"onAllyHpBelow"`, `"onEnemySummon"` 等）
- `condition?`: string（追加条件。省略可。例：`"damageAmount >= 100"`）

### SkillCardDefinition（追加）
- `effect`: EffectSpec（データ駆動にするなら）

### SummonObjectCardDefinition（追加）
召喚オブジェクトを召喚するカード定義（=召喚枠に配置するカード）。

- `durationTurns`
- `objectScaling`: ScalingSpec（召喚オブジェクト用：キャラステータス×補正）
- `objectFlatStats`: FlatStatBonus（召喚オブジェクト用：調整弁）
- `onEnterEffects`: EffectSpec[]
- `burstVariantCardId`: string | null（バースト状態時のオブジェクトカード差し替え）

### SummonUnitCardDefinition（追加）
召喚ユニットを召喚するカード定義（=戦闘に参加するカード）。

- `durationTurns`
- `unitScaling`: ScalingSpec（召喚ユニット用：キャラステータス×補正）
- `unitFlatStats`: FlatStatBonus（召喚ユニット用：調整弁）
- `onEnterEffects`: EffectSpec[]
- `burstVariantCardId`: string | null（バースト状態時のユニットカード差し替え）

### SupportCardDefinition（追加）
- `burstCostPoints`（例：200）
- `cooldownTurns`: number（使用後のクールダウンターン数。例：3）
- `effect`: EffectSpec
- `setEffects?`: EffectSpec[]（セット効果＝構築時パッシブ。サポートカードはチームに紐づくため、スコープは原則 `team`）

### SupportSlotState（サポート枠の実行時状態）
- `cardId`: string
- `cooldownRemainingTurns`: number（0なら使用可能）
- `createdSeq`: number（ターン終了処理順の決定に使用。クールダウン減少の順序に影響）

### UniqueCardDefinition（追加）
- `cooldownTurns`
- `effect`: EffectSpec
- `burstVariantCardId`: string | null（バースト状態時の強化版ユニーク）

### PlannedAction（準備フェイズの予約）
- `id`
- `teamId`
- `actorType`: `character | summon`
- `actorId`: string
- `source`: `hand | support | unique | summonConfigured`
- `cardId`: string（sourceに応じて CardId/SupportCardId/UniqueCardId 等）
- `targets`: TargetSpec（対象指定。未指定/ランダム/自動等：TBD）
- `sequence`: number（同一actor内の実行順）
- `createdAtMs`: number（30秒制限・UI/リプレイ用：任意）
- `cardPriority`: number（予約時点のpriority。ソート用に冗長保持：任意）
- `willConsumeApOnExecute`: number（実行時に消費するAP。=cost：任意）

### ResourcesState（例）
- `burstPoints`: number（0〜burstMaxPoints）
- `burstMaxPoints`: number（BurstMax=5の場合は500）
（UI表示用に「ゲージ本数」が欲しい場合は `floor(burstPoints / 100)` で算出する）

### PreparationDecisionState（例）
- `submittedAtMs`: number | null
- `decisionSpeedBonusPoints`: number（例：70/50/30/0。未確定なら0）

### ElementCounters（キャラのエレメント）
- `red`: number
- `blue`: number
- `green`: number
- `orange`: number
- `yellow`: number
- `purple`: number
（無色はカウンターを持たない）

### UniqueSlotState（例）
- `equippedUniqueCardId`: string | null
- `cooldownRemainingTurns`: number
- `createdSeq`: number（ターン終了処理順の決定に使用。クールダウン減少の順序に影響）
- `effectiveUniqueCardId`: string | null（バースト中はburstVariantへ差し替えた結果：任意）

### CharacterPassiveInstance（例）
- `passiveId`
- `unlockAtUnlockedMaxAP`: number（例：4/6/7）
- `isActive`: boolean（または動的に計算）

### ScalingSpec / FlatStatBonus（例）
- `scale`: { pAtk?: number; pDef?: number; mAtk?: number; mDef?: number; hp?: number; spd?: number; tech?: number }
- `flat`:  { pAtk?: number; pDef?: number; mAtk?: number; mDef?: number; hp?: number; spd?: number; tech?: number }

### TargetSpec（例）
- `mode`: `self | allyCharacter | enemyCharacter | allySummonUnit | enemySummonUnit | allySummonObject | enemySummonObject | allAllies | allEnemies | random | allyDowned`
- `targetIds`: string[]（必要な場合）

（補足）
- `allyDowned`：ダウン済みの味方キャラを対象にする（蘇生専用）。Resurrect 以外の EffectType でこのモードを使用した場合、効果は無効となる。
- `allAllies` / `allEnemies`：対象は `isDown == false` のキャラのみ（ダウン済みを含まない）。
- `random`：`isDown == false` のキャラから RngProvider で1体選択。

### FieldEffectInstance（チームのフィールド効果）
カード効果などで「チーム全体に恩恵」「相手チームに不利」を与えるための、**バトル中にチーム単位で保持される効果**。
フィールド効果は **各チーム1枠** で、追加で展開されると既存のものは上書きされる。

- `instanceId`
- `ownerTeamId`: string（このフィールド効果を"保持している"チーム。=スロットの所属）
- `sourceTeamId`: string（このフィールド効果を"発動させた"チーム。敵に付与する効果もあり得るため分離）
- `source`: `{ kind: "card" | "support" | "unique" | "summonObject" | "summonUnit" | "passive" | "system"; sourceId?: string; sourceCharacterId?: string }`
- `createdSeq`: number（ターン終了処理順の決定に使用）
- `duration`: `{ kind: "turns"; remainingTurns: number } | { kind: "infinite" }`
  - `kind: "turns"` の場合、`remainingTurns` は **ターン終了時** に減少し、0になったら期限切れ（Expired）で除去する。
- `modifiers`（状態と同様：例「全味方被ダメ-1」「敵全体の回復量-30%」など）
- `triggers`: TriggerSpec[]（状態と同様：誘発型のフィールド効果がある場合）
- `rulesText?`: string（ログ/デバッグ/リプレイ向けの任意保持）

（運用メモ）
- 新規展開時は `TeamState.fieldEffect` を丸ごと差し替える（旧フィールド効果は **消滅（上書き）** 扱い）
- 期限切れ（duration==0）・解除（dispel等）・上書きは、**除去理由として区別**できるようにイベントログも分ける（下記 `Event` 参照）
- 除去後は `null` へ

### FieldEffectDefinition（フィールド効果の定義テンプレート）
フィールド効果をデータ駆動で定義するためのテンプレート。`FieldEffectInstance` の生成元となる。

- `id`: string（例：`field_flame_zone`）
- `name`: string（例：「炎域」）
- `defaultDurationTurns`: number（デフォルト持続ターン数。-1 で無限）
- `scope`: `"ownTeam" | "enemyTeam" | "bothTeams"`（効果の適用範囲）
- `modifiers`: StatModifier[]（ステータス修正。StatCalculator パイプラインに参加）
- `triggers`: TriggerSpec[]（誘発型効果がある場合）
- `tags`: string[]（dispel 対象判定などに使用。標準タグ：`purgeable`）
- `rulesText`: string

### ShieldInstance（キャラのシールド状態）
ダメージを肩代わりする一時的な防御膜。**キャラごとに最大1枠**。新規付与時は既存を上書きする。

- `sourceId`: string（付与元カード/効果の ID）
- `sourceCharacterId`: string（付与元キャラ）
- `remainingAmount`: number（残りシールド量。ダメージ吸収で減少）
- `remainingTurns`: number（残り持続ターン数。ターン終了時に減少）
- `createdSeq`: number（ターン終了処理順の決定に使用）

（ルール）
- ダメージ処理時、DEF計算後・HP減少前にシールドで吸収する
- `remainingAmount` がダメージ以上 → シールドだけ減少、HP変動なし
- `remainingAmount` がダメージ未満 → シールドを全消費し、超過分をHPから減算
- **真ダメージはシールドを貫通する**（`15_damage_formula.md` 準拠）
- `remainingAmount == 0` または `remainingTurns == 0` で消滅
- 新規シールド付与時、既存シールドは **上書き（消滅）** される（加算ではない）

### ItemCardDefinition（アイテムカードの定義）
アイテムカード固有の定義。キャラにセットされ、トリガー条件で自動発動する。

- `count`: number（使用回数。例：3）
- `triggers`: CardTrigger[]（自動発動のトリガー条件）
- `effect`: EffectSpec（発動時の効果）

---

## 2) 状態（StatusEffect）モデル（案）

### 最小プロパティ
- `id`
- `source`（カード/召喚/キャラ/サポート/ユニーク）
- `target`（キャラ or 召喚）
- `duration`（ターン数 or `infinite`）
- `createdSeq`: number（ターン終了処理順の決定に使用）
- `modifiers`（例：STR+2、被ダメ-1、ドロー+1 など）
- `triggers`（誘発型なら）
- `stackingRule`: StackingRule（重ねがけルール）

### StackingRule（確定）
同一状態が既に付与されている場合の挙動を定義する。

- `mode`: `replace | extend | stack`
  - **replace**：既存の同名状態を **上書き**（残りターンもリセット）
  - **extend**：既存の同名状態の **持続ターンを延長**（効果量はそのまま）
  - **stack**：既存の同名状態に **重ね掛け**（スタック数が増え、効果量が加算される）
- `maxStacks?`: number（`mode=stack` の場合のみ。**カード/状態定義ごとに個別設定可能**。例：毒は5、攻撃バフは3 等）

（補足）
- `mode` はカード/状態定義ごとに設定する。同じ名前の状態でも、付与元カードが異なれば異なるルールを持ちうる。
- `maxStacks` に達した状態にさらにスタックを付与した場合、**新しいスタック分は無視**される（上限で止まる）。

### StatusEffectDefinition（状態異常/バフの定義テンプレート）

状態異常やバフを **順次追加可能なフレームワーク** として定義するためのテンプレート。
新しい状態異常/バフを追加する場合、このテンプレートに沿って定義する。

- `id`: string（例：`status_poison`）
- `name`: string（例：「毒」）
- `category`: `"buff" | "debuff"`
- `stackingRule`: StackingRule（上記参照）
- `defaultDuration`: number（デフォルト持続ターン数）
- `effect`: StatusEffectType
- `triggers`: string[]（処理タイミング。例：`["turnEnd"]`）
- `baseHitRate`: number（tech計算式の `baseRate`。`15_damage_formula.md` 参照）
- `tags`: string[]（参照用タグ。dispel/immune の対象判定にも使用する。標準タグ：`purgeable`（dispel解除可能）, `crowdControl`（行動制限）, `statModifier`（ステータス増減）, `dot`（持続ダメ）, `hot`（持続回復）。詳細は `02_cards.md` セクション9）

### StatusEffectType（効果の種類）

- `type`: `"dot" | "statMod" | "actionBlock" | "misc"`

#### dot（持続ダメージ/持続回復）
- `damageType`: `"physical" | "magical" | "true"`
- `damageCalc`: `"fixed" | "maxHpPercent" | "atkPercent"`
- `value`: number（固定値 or パーセント）
- `isHeal?`: boolean（trueなら回復として扱う）

#### statMod（ステータス増減）
- `stat`: string（`"pAtk" | "pDef" | "mAtk" | "mDef" | "spd" | "tech"` 等）
- `modType`: `"flat" | "percent"`
- `value`: number

#### actionBlock（行動制限）
- `blockType`: `"skip" | "chance"`
  - **skip**: 行動を完全にスキップ
  - **chance**: 確率で不発にする
- `chance?`: number（`blockType=chance` の場合の確率）

### 初期状態異常セット（サンプル・順次追加可能）

| 名前 | category | effect.type | stackingMode | maxStacks | defaultDuration | baseHitRate | 備考 |
|---|---|---|---|---|---|---|---|
| 毒 | debuff | dot (maxHP×5%, true) | stack | 3 | 3 | 0.80 | 紫の基本デバフ |
| 火傷 | debuff | dot＋statMod(pAtk-10%) | replace | — | 3 | 0.70 | 赤の副次効果 |
| 凍結 | debuff | actionBlock(skip) | replace | — | 1 | 0.50 | 青の強CC |
| 攻撃UP | buff | statMod(pAtk+固定値) | stack | 3 | 3 | — | 汎用バフ |
| リジェネ | buff | dot(回復, maxHP×8%) | replace | — | 3 | — | 緑/黄の回復 |

（注）上記はサンプル。テンプレートに沿って定義すれば、新しい状態異常をいくらでも追加できる。

---

## 2.5) カード効果定義（EffectSpec）

カード効果をデータ駆動で表現する共通フレームワーク。
`StatusEffectType` と同じ **判別共用体パターン**（`EffectType` enum + 型別パラメータクラス）を採用し、
継承階層ではなく flat な構成にすることで JSON / ScriptableObject シリアライズに最適化する。

### EffectType（効果種別 enum）

| EffectType | 用途 | 代表例 |
|---|---|---|
| `Damage` | ダメージ（物理/魔法/真） | 攻撃スキル |
| `Heal` | 回復（固定/割合/スケーリング） | 回復スキル |
| `Draw` | カードドロー | ドロー系スキル |
| `StatusApply` | 状態付与（バフ/デバフ/DoT） | 毒付与、攻撃UP |
| `Dispel` | 状態解除（タグベース） | 解除スキル |
| `Summon` | 召喚の参照（SummonManager へ委譲） | 召喚カード |
| `ElementGain` | エレメント獲得 | エレメント蓄積 |
| `ElementConsume` | エレメント消費 | エレメント消費型スキル |
| `ElementConvert` | エレメント変換 | 橙の色変換 |
| `BurstGain` | バーストポイント増加 | バースト蓄積 |
| `Shield` | シールド付与 | 防御スキル |
| `Resurrect` | 蘇生 | 蘇生スキル |
| `FieldEffectDeploy` | フィールド効果展開 | フィールドスキル |
| `ItemSet` | アイテムセット | アイテムカード |
| `APModify` | AP 増減 | AP回復スキル |
| `Discard` | 手札捨て（自分/相手） | 荷物整理、偽情報 |
| `Bounce` | フィールド→手札戻し | 巫術・扇風乱舞 |
| `DeckManipulate` | デッキ操作（ボトムドロー/発掘/並べ替え） | アクセスダイブ、地層調査 |
| `DeckSearch` | デッキサーチ→即発動/手札追加 | 相棒コール |
| `TokenCreate` | 一時カード生成（使用後→追放） | アクセスダイブ、非常への備え |
| `Reveal` | 手札/デッキの情報公開 | 観測 |
| `Composite` | 複合効果（子 EffectSpec のリスト） | ダメージ+デバフ |
| `Conditional` | 条件分岐（条件→Then/Else） | エレメント≥3で追加ダメージ |

### EffectSpec（効果定義の本体）

- `effectType`: EffectType（上記 enum）
- `target?`: TargetSpec（対象指定。省略時は発動元カードの target を継承）
- `params`: 型別パラメータ（EffectType に応じて以下の1つを使用）

### 型別パラメータクラス

#### DamageEffectParams（Damage）
- `damageType`: `"physical" | "magical" | "true"`
- `cardMultiplier`: number（カード倍率。例：1.2）
- `fixedDamage?`: number（真ダメージ固定値。`damageType=true` かつ倍率ではない場合）
- `hitCount?`: number（多段ヒット数。省略時=1）
- `atkStat?`: `"pAtk" | "mAtk"`（参照する攻撃ステータス。`damageType` から自動判定可。真ダメージで倍率参照時に使用）

#### HealEffectParams（Heal）
- `calcMode`: HealCalcMode（下記 enum）
- `value`: number（固定値 or パーセント値 or 倍率）
- `stat?`: `"pAtk" | "mAtk" | "pDef" | "mDef" | "tech"`（`calcMode=statScale` 時に参照するステータス）

#### HealCalcMode（enum）
- `fixed`：固定値回復（value がそのまま回復量）
- `maxHpPercent`：最大HPの割合回復（value=0.1 → maxHP×10%）
- `statScale`：ステータス×倍率で回復（value=0.5, stat=mAtk → mAtk×0.5）

#### DrawEffectParams（Draw）
- `count`: number（ドロー枚数）

#### StatusApplyEffectParams（StatusApply）
- `statusId`: string（StatusEffectDefinition の id を参照。例：`status_poison`）
- `duration?`: number（省略時は StatusEffectDefinition の defaultDuration）
- `overrideValue?`: number（効果量を上書きする場合。省略時は定義のデフォルト）
- `stacks?`: number（一度に付与するスタック数。省略時=1）

#### DispelEffectParams（Dispel）
- `category`: `"buff" | "debuff" | "any"`
- `count`: number（解除件数）
- `filterTags?`: string[]（特定タグのみ対象。省略時は `purgeable` 全般）

#### SummonEffectParams（Summon）
- `summonCardId`: string（召喚カード定義の ID。SummonManager へ委譲）

#### ElementGainEffectParams（ElementGain）
- `element`: `"red" | "blue" | "green" | "orange" | "yellow" | "purple"`
- `amount`: number

#### ElementConsumeEffectParams（ElementConsume）
- `element`: `"red" | "blue" | "green" | "orange" | "yellow" | "purple"`
- `amount`: number

#### ElementConvertEffectParams（ElementConvert）
- `from`: `"red" | "blue" | "green" | "orange" | "yellow" | "purple"`
- `to`: `"red" | "blue" | "green" | "orange" | "yellow" | "purple"`
- `amount`: number

#### BurstGainEffectParams（BurstGain）
- `points`: number（バーストポイント増加量）

#### ShieldEffectParams（Shield）
- `shieldType`: ShieldType（下記 enum）
- `value`: number（シールド量。固定値 or 倍率）
- `duration?`: number（持続ターン数。省略時=1）
- `stat?`: string（`shieldType=statScale` 時に参照するステータス）

#### ShieldType（enum）
- `fixed`：固定値シールド
- `maxHpPercent`：最大HPの割合
- `statScale`：ステータス×倍率

#### ResurrectEffectParams（Resurrect）
- `hpPercent`: number（蘇生時のHP割合。例：0.3 → maxHP×30%）

#### FieldEffectDeployEffectParams（FieldEffectDeploy）
- `fieldEffectId`: string（展開するフィールド効果の定義 ID）
- `durationTurns`: number（持続ターン数。-1 で無限）

#### ItemSetEffectParams（ItemSet）
- `itemCardId`: string（セットするアイテムカードの定義 ID）

#### APModifyEffectParams（APModify）
- `amount`: number（正で回復、負で減少）
- `targetStat`: `"current" | "unlockedMax"`（現在AP or 解放済み最大AP）

#### DiscardEffectParams（Discard）
- `count`: number（捨てる枚数）
- `targetScope`: `"self" | "enemy"`（自分の手札 or 相手の手札）
- `selection`: DiscardSelection（下記 enum）
- `destination`: `"graveyard" | "banished" | "deckBottom"`（捨て先）

#### DiscardSelection（enum）
- `random`：ランダムに選択
- `lowestCost`：コストが最も低いカードから選択（同コスト内はランダム）
- `highestCost`：コストが最も高いカードから選択（同コスト内はランダム）
- `ownerChoice`：所有者が選択（Phase 1 ではランダムで代用可）

#### BounceEffectParams（Bounce）
- `targetType`: `"summonUnit" | "summonObject" | "item" | "fieldEffect" | "any"`（戻す対象の種別。`any` は召喚/アイテムからランダム）
- `count`: number（戻す数）
- `selection`: `"random" | "lowestCost" | "highestCost" | "ownerChoice"`
- `destination`: `"hand" | "deckTop" | "deckBottom"`（戻し先）
- `filterCostOperator?`: `"lte" | "gte" | "eq"`（コストフィルタ。省略時はフィルタなし）
- `filterCostValue?`: number（コストフィルタの基準値。`contextRef` で直前の効果結果を参照可能）

#### DeckManipulateEffectParams（DeckManipulate）
- `action`: DeckManipulateAction（下記 enum）
- `count`: number（操作枚数）
- `filterTags?`: string[]（対象カードのタグフィルタ。発掘で「遺物」タグ等）
- `remainderAction?`: `"deckBottom" | "deckTop" | "graveyard"`（フィルタに一致しなかったカードの行き先。省略時は `deckBottom`）

#### DeckManipulateAction（enum）
- `drawFromBottom`：デッキの一番下からカードをドロー
- `excavate`：デッキの上からcount枚を確認し、filterTagsに一致するカードを手札へ。残りは remainderAction へ
- `peek`：デッキの上からcount枚を確認（手札には加えない。情報取得のみ）

#### DeckSearchEffectParams（DeckSearch）
- `filterTags?`: string[]（検索対象のタグフィルタ。例：`["バディ"]`）
- `filterType?`: `"skill" | "summonUnit" | "summonObject" | "item" | "any"`（カード種別フィルタ）
- `filterCostOperator?`: `"lte" | "gte" | "eq"`（コストフィルタ）
- `filterCostValue?`: number | `"remainingAP"`（コスト基準値。`"remainingAP"` は発動時の残りAPを参照）
- `action`: `"addToHand" | "play"`（見つけたカードを手札に加える or 即座に発動）
- `maxResults?`: number（検索結果の上限。省略時=1）
- `onNotFound?`: `"fizzle" | "ignore"`（見つからなかった場合。`fizzle` は不発扱い、`ignore` は何も起きない）

#### TokenCreateEffectParams（TokenCreate）
- `templateCardId?`: string（テンプレートとなるカード定義のID。省略時は動的生成）
- `color?`: `"red" | "blue" | "green" | "orange" | "yellow" | "purple" | "colorless" | "casterColor"`（トークンの色。`casterColor` は使用キャラの色を継承）
- `tags`: string[]（トークンに付与するタグ。`["消失"]` など）
- `count?`: number（生成枚数。省略時=1）
- `addTo`: `"hand" | "deckTop" | "deckBottom"`（生成先）
- `vanishOnUse`: boolean（使用後に追放するか。`true` = 【消失】相当）

#### RevealEffectParams（Reveal）
- `targetScope`: `"enemyHand" | "enemyDeckTop" | "selfDeckTop"`（公開対象）
- `count?`: number（`deckTop` 系の場合の枚数。`hand` の場合は全枚数）
- `duration?`: number（公開情報の持続ターン数。省略時=そのターンのみ）

#### CompositeEffectParams（Composite）
- `children`: EffectSpec[]（子効果のリスト。順次解決）

#### ConditionalEffectParams（Conditional）
- `condition`: ConditionSpec（条件判定。下記参照）
- `thenEffect`: EffectSpec（条件成立時に実行する効果）
- `elseEffect?`: EffectSpec（条件不成立時に実行する効果。省略可）

### ConditionSpec（条件定義）

- `conditionType`: ConditionType（下記 enum）
- `params`: 条件別パラメータ

#### ConditionType（enum）

| ConditionType | 用途 | 例 |
|---|---|---|
| `ElementThreshold` | エレメント数の閾値判定 | 赤エレメント≥3 |
| `HpThreshold` | HP割合の閾値判定 | HP≤50% |
| `StatusCheck` | 特定状態の有無判定 | 対象が毒状態か |
| `BurstCheck` | バースト状態判定 | 自分がバースト中か |
| `AllyDownCount` | 味方ダウン数判定 | 味方が2体以上ダウン |
| `HandCount` | 手札枚数判定 | 手札≤2枚 |

#### ElementThresholdParams
- `element`: `"red" | "blue" | "green" | "orange" | "yellow" | "purple"`
- `operator`: `"gte" | "lte" | "eq"`（以上/以下/等しい）
- `threshold`: number

#### HpThresholdParams
- `scope`: `"self" | "target"`（判定対象。自分 or 効果対象）
- `operator`: `"gte" | "lte"`
- `thresholdPercent`: number（例：0.5 → 50%）

#### StatusCheckParams
- `scope`: `"self" | "target"`
- `statusId?`: string（特定状態 ID。省略時はタグで判定）
- `filterTags?`: string[]（タグベース判定）
- `exists`: boolean（true=存在する場合に成立、false=存在しない場合に成立）

#### BurstCheckParams
- `scope`: `"self" | "target"`
- `isBurst`: boolean

#### AllyDownCountParams
- `operator`: `"gte" | "lte" | "eq"`
- `count`: number

#### HandCountParams
- `operator`: `"gte" | "lte" | "eq"`
- `count`: number

### セット効果（SetEffects）で使用可能な EffectType の制約

セット効果（構築時パッシブ）は **バトル開始前に適用される** ため、バトル中の対象指定やリアルタイム効果を使用できない。

#### 許可される EffectType
| EffectType | 用途例 |
|---|---|
| `StatusApply` | 恒常ステ補正（永続 statMod バフ/デバフ） |
| `Draw` | ゲーム開始時に追加ドロー |
| `BurstGain` | 初期バーストポイント付与 |
| `ElementGain` | 初期エレメント付与 |
| `APModify` | 初期AP補正 |
| `Composite` | 上記の複合（例：ステ補正+追加ドロー） |
| `Conditional` | 条件付きセット効果（例：特定色デッキの場合のみ発動） |

#### 禁止される EffectType
- `Damage`, `Heal`, `Dispel`, `Summon`, `ElementConsume`, `ElementConvert`, `Shield`, `Resurrect`, `FieldEffectDeploy`, `ItemSet`, `Discard`, `Bounce`, `DeckManipulate`, `DeckSearch`, `TokenCreate`, `Reveal`
  - 理由：これらはバトル中の対象指定・リアルタイム効果を前提としており、構築時パッシブとして適用する意味がない。

---

## 3) イベント（Event）モデル（案）

### 代表イベント
- `TurnStarted`, `TurnEnded`
- `PhaseChanged`（TurnStart/Draw/Preparation/Battle/TurnEnd）
- `ActionReserved`（準備フェイズで予約された）
- `CardPlayed`（Skill/Summon/Item/Unique）
- `SupportActivated`
- `FieldEffectDeployed`（フィールド効果が展開された）
- `FieldEffectOverwritten`（既存のフィールド効果が上書きで消滅した）
- `FieldEffectExpired`（フィールド効果が期限切れ（duration==0）で消滅した）
- `FieldEffectRemoved`（フィールド効果が解除（dispel等）で消滅した）
- `SummonObjectEntered`, `SummonObjectExpired`, `SummonObjectDestroyed`
- `SummonUnitEntered`, `SummonUnitExpired`
- `SummonReplaced`（任意：入れ替えを明示したい場合。オブジェクト入れ替え＋ユニット同時消滅を含む）
- `UnitActed`（キャラ/召喚）
- `CharacterDowned`
- `DamageDealt`, `Healed`
- `StatusApplied`, `StatusExpired`
- `ElementGained`, `ElementConsumed`, `ElementConverted`
- `ShieldApplied`（シールド付与）
- `ShieldConsumed`（シールドがダメージを吸収した）
- `ShieldExpired`（シールドが期限切れで消滅した）
- `CharacterResurrected`（蘇生）
- `ItemSet`（アイテムがキャラにセットされた）
- `ItemTriggered`（アイテムが自動発動した）
- `ItemExhausted`（アイテムの残回数が0になり破棄された）
- `ItemOverwritten`（アイテムが上書きで除去された）
- `APModified`（AP増減）
- `CardDiscarded`（手札からカードが捨てられた）
- `CardBounced`（フィールドのカード/召喚/アイテムが手札に戻された）
- `DeckManipulated`（デッキ操作が行われた：ボトムドロー/発掘/確認）
- `DeckSearched`（デッキサーチが行われた）
- `TokenCreated`（トークンカードが生成された）
- `HandRevealed`（手札が公開された）
- `DeckReshuffledFromTrash`（詳細は下記）
- `ActionResolved`（詳細は下記）
- `StatRecomputed`（詳細は下記。セクション5参照）

### ActionResolved（行動解決イベント）（確定）
バトルフェイズで各行動が解決された際に記録するイベント。不発を含むすべての行動について生成する。

- `actionId`: string（PlannedAction.id に対応）
- `actorType`: `character | summon`
- `actorId`: string
- `cardId`: string
- `resolvedAs`: `executed | fizzled`（正常実行 or 不発）
- `fizzleReason?`: `insufficientAP | invalidTarget | insufficientElement | cooldownActive`（不発時の理由）
- `activationMode`: `normal | interrupt`（通常発動 or トリガー割り込み発動）
- `apBefore`: number（実行前のAP）
- `apCostApplied`: number（実際に消費されたAP。不発時は0）
- `apAfter`: number（実行後のAP）
- `sortKey`: `{ priority: number; spd: number; decisionSubmittedAtMs: number; initiativeTeamId: string; createdSeq: number }`
- `tieBreakLevelUsed`: `priority | spd | submitTime | initiative | createdSeq`（どこで順序が確定したか）

### DeckReshuffledFromTrash（デッキ再構築イベント）（確定）
デッキ枚数が0の状態でドローが発生した際のトラッシュ戻し＋シャッフルを記録するイベント。

- `turn`: number
- `teamId`: string
- `oldDeckCount`: number（イベント前のデッキ枚数。通常0）
- `trashReturnedCount`: number（graveyardから戻されたカード枚数）
- `banishedIgnoredCount`: number（追放ゾーンにあり戻されなかったカード/実体数）
- `rngSeedSnapshot`: number（シャッフル時点のRNG状態）
- `rngStepBefore`: number（シャッフル前のRNGステップ数）
- `rngStepAfter`: number（シャッフル後のRNGステップ数）
- `shuffleAlgorithm`: `"fisher-yates"`（推奨。決定的シャッフルアルゴリズム）

### イベント処理の流れ
1. アクション実行でイベント生成（例：CardPlayed）
2. イベントに反応するトリガーを収集（アイテム/パッシブ/状態/フィールド効果）
3. 追加イベントをキューに積む
4. すべて解決するまで繰り返す

**無限ループ対策**：同一フェイズ内で同一エンティティから発生する誘発回数は **上限10回** とする。上限に達した場合、それ以上の誘発は無視する。

---

## 4) 計算式（初期案）

### 召喚ステータス（`03_summons.md`）
- 例：`summon.pAtk = character.pAtk * scale.pAtk + flat.pAtk`
- 例：`summon.hp   = character.maxHp * scale.hp + flat.hp`
- スケーリング元は「キャラのバトル中ステータス（バフ/デバフ込み）」を参照する。

### バースト増加（`05_burst_and_support.md`）
- `on CardPlayed`: `burstPoints += burstDeltaPoints`
- `on UnitActed (Summon)`: `burstPoints += burstDeltaPoints`
  - デフォルト `burstDeltaPoints=30`（1ターンで1〜2ゲージ蓄積を目安に設定）
  - パッシブ等で蓄積率倍率がある場合は `burstDeltaPoints * multiplier` のように補正してよい
    - multiplierで小数が出る場合は **切り捨て（floor）**

（補足）色/カード個性で加算量を変える場合、イベント側に `burstDeltaPoints` を付与する。

### ダメージ計算式（`15_damage_formula.md`）
- `最終ダメージ = max(1, floor(ATK × カード倍率 × 100 / (100 + DEF × 防御係数)))`
- 防御係数の初期値：**0.6**
- 詳細は `15_damage_formula.md` を参照。

---

## 5) ステータス修正パイプライン（確定）

### 目的
バフ/デバフ/パッシブ/バースト倍率/フィールド効果など、複数の継続効果が同一ステータスに影響する場合の **合成順と丸め** を定義する。

### パイプライン（5段階）
同一計算対象への修正は、以下の順序で適用する。

1. **base**（基礎値）：キャラ/召喚の素のステータス＋ロードアウト（構築時パッシブ）補正
2. **additive**（加算補正）：バフ/デバフの固定値（例：pAtk+20、pDef-15）
3. **multiplicative**（乗算補正）：バースト倍率（×1.15等）、割合バフ/デバフ（例：pAtk×1.10）
4. **clamp**（上下限制約）：特殊効果による上限/下限設定がある場合
5. **finalFloor**（最終切り捨て）：`floor` で整数化。**ステータス下限=0**（HP最大値のみ下限1）

### 同カテゴリ内の適用順
- 同じカテゴリ（例：additive同士）の中では、**`createdSeq` 昇順**（効果が付与された順）で適用する。
- 同 `createdSeq` の場合は、所有者チームの `initiativeTeamId` 側を先に適用する。

### 置換型（setTo）効果
- 「ステータスをN固定にする」型の効果は、**clamp 前に適用** し、以後の additive/multiplicative を **受けない**。
- 同一ステータスに複数の setTo が存在する場合、`createdSeq` が最も新しい（最後に付与された）ものが有効。

### フィールド効果と状態効果の相互作用
- フィールド効果と状態効果が同名パラメータを修正する場合も、**共通パイプラインで処理** する。特例（種別ごとの優先度分離等）は設けない。

### StatRecomputed イベント
ステータスが再計算された際に記録するイベント（デバッグ/リプレイ検証用）。

- `entityType`: `character | summonUnit | summonObject`
- `entityId`: string
- `stat`: string（`pAtk | pDef | mAtk | mDef | spd | tech | maxHp` 等）
- `baseValue`: number
- `appliedEffects`: `{ id: string; layer: "additive" | "multiplicative" | "clamp" | "setTo"; delta: number }[]`
- `finalValue`: number

---

## （追加）ロードアウト（構築時パッシブ）モデル（案）

### LoadoutPassiveInstance（例）
- `id`
- `sourceCardId`: string
- `ownerCharacterId`: string
- `scope`: `self | team`（キャラ単体/チーム全体）
- `stacking`: `stackable | nonStackable`
- `nonStackingKey?`: string（`stacking=nonStackable` の場合に同一判定するキー）
- `effects`: EffectSpec[]（ステ補正、開始時ドロー等）

（注）「同一キャラに同一カードをセットできない」ため、self-scope での同名重複は発生しない想定。

---

## （追加）レギュレーション定義（RegulationDefinition）

対戦ルールをパラメータ化し、様々なレギュレーション（3vs3標準、1vs1方式B、カジュアル、大会用等）を柔軟に定義するためのデータモデル。

### RegulationDefinition
- `id`: string（例：`reg_standard_3v3`, `reg_1v1_leader`）
- `name`: string（例：「標準3vs3」「1vs1リーダー戦」）
- `teamSize`: number（チーム人数。例：3）
- `characterDeckSize`: `{ min: number; max: number }`（キャラデッキ枚数。例：`{ min: 10, max: 15 }`）
- `supportSetMax`: number（サポート枠の上限。例：3）
- `colorRestriction`: `{ mode: "none" | "maxColors" | "singleOnly"; maxColors?: number }`（色制約。例：`{ mode: "maxColors", maxColors: 2 }`）
- `sameNameRule`: `{ perCharacterDeck: number; perTeam: number | null }`（同名制限。例：`{ perCharacterDeck: 1, perTeam: null }`）
- `bannedCards`: CardId[]（禁止カードリスト）
- `restrictedCards`: `{ cardId: string; maxPerTeam: number }[]`（制限カードリスト）
- `maxTurns`: number | null（最大ターン数。nullなら無制限）
- `preparationTimeSec`: number（準備フェイズの制限時間。例：30）
- `initialHandSize`: number（初期手札枚数。例：5）
- `apStart`: number（初期AP。例：3）
- `burstMax`: number（BurstMaxゲージ数。例：5）
- `burstMaxPoints`: number（BurstMax内部ポイント。例：500）
- `actionChargePoints`: number（行動チャージ量。例：30）
- `defenseCoefficient`: number（ダメージ計算の防御係数。例：0.6）
- `victoryCondition`: `"allDown" | "leaderDown"`（勝利条件）
- `tiebreaker`: `"hpRatioSum"`（タイブレーク方式）
- `oneVsOneRules?`: OneVsOneRegulation | null（1vs1方式B固有ルール。nullなら3vs3標準）

### RegulationDefinition（追加フィールド）

以下は既存フィールドに加え、バランス調整・レギュレーション分岐のために追加するフィールド。

- `cardMultiplierGuide`: CardMultiplierGuide（カード倍率のガイドライン。設計メタ情報）
- `colorRestriction`: ColorRestriction（色制約）
- `sameNameRule`: SameNameRule（同名カード制限）
- `colorlessCardLimit`: ColorlessCardLimit（無色カード制限）
- `timeoutRule`: TimeoutRule（ターン上限到達時の処理）

### CardMultiplierGuide（カード倍率ガイドライン）
レギュレーションごとのカード設計指針。実装上の制約ではなく、カード調整時の参照情報。

- `low`: `{ min: number; max: number }`（例：`{ min: 0.3, max: 0.5 }`）
- `standard`: `{ min: number; max: number }`（例：`{ min: 0.6, max: 1.0 }`）
- `high`: `{ min: number; max: number }`（例：`{ min: 1.2, max: 1.6 }`）
- `ultra`: `{ min: number; max: number }`（例：`{ min: 1.8, max: 2.5 }`）

### ColorlessCardLimit（無色カード制限）
- `mode`: `"none" | "ratio" | "fixed"`
  - **none**：制限なし
  - **ratio**：キャラデッキ枚数に対する比率で制限（例：0.5＝デッキの半分まで）
  - **fixed**：固定枚数で制限（例：5枚まで）
- `maxRatio?`: number（`mode=ratio` の場合）
- `maxCount?`: number（`mode=fixed` の場合）

### TimeoutRule（ターン上限到達時の処理）
- `method`: `"hpRatioSum"`（ダウンしていないキャラのHP割合の合計で判定）
- `finalTiebreak`: `"initiative"`（完全同値時は `initiativeTeamId` で判定）

### 標準レギュレーション初期値（参考）

| 項目 | 値 | 備考 |
|---|---|---|
| maxTurns | 15 | 膠着防止 |
| colorRestriction | `{ mode: "maxColors", maxColors: 2 }` | 2色＋無色 |
| sameNameRule.perTeam | 2 | チーム全体で同名2枚まで |
| colorlessCardLimit | `{ mode: "ratio", maxRatio: 0.5 }` | デッキの半数まで |
| timeoutRule | `{ method: "hpRatioSum", finalTiebreak: "initiative" }` | HP割合合算 |

（注）全てレギュレーションごとに変更可能。特殊ルール（単色限定、無色禁止、ターン無制限等）も定義可能。

### OneVsOneRegulation（1vs1方式B固有）
- `supportSetMaxOverride?`: number（1vs1用サポート上限。例：2）
- `apSupplyEnabled`: boolean（サポートからのAP供給の有無）
- `apSupplyMaxPerTurn`: number | null（AP供給の上限。nullなら無制限）
- `summonSlotsForLeader`: number（リーダーの召喚枠数。例：3）


