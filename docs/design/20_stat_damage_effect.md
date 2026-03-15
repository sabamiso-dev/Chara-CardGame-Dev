# ステータス計算・ダメージ計算・効果解決 設計

ルールエンジン設計の一部。StatCalculator（5段階パイプライン）、DamageCalculator（ダメージ計算式）、EffectResolver（カード効果のデータ駆動解決）を定義する。

関連: [行動解決](19_action_resolver.md) / [召喚・バースト](21_summon_and_burst.md) / [サブシステム](22_subsystems.md)

---

## 5. ステータス修正パイプライン（StatCalculator）

### 5段階パイプライン

```
base → additive → multiplicative → clamp → finalFloor
```

| 段階 | 内容 | 例 |
|---|---|---|
| **1. base** | キャラ/召喚の素ステータス + ロードアウト（構築時パッシブ）補正 + 召喚オブジェクト上乗せ | pAtk = 300 |
| **2. additive** | バフ/デバフの固定値加算 | pAtk + 20（攻撃UP）, pAtk - 15（デバフ） |
| **3. multiplicative** | バースト倍率、割合バフ/デバフの乗算 | pAtk × 1.15（バースト）, pAtk × 1.10（割合バフ） |
| **4. clamp** | 特殊効果による上限/下限設定 | 「pAtk を 500 以下にする」等 |
| **5. finalFloor** | `floor` で整数化。ステータス下限 = 0（maxHp のみ下限 1） | floor(345.5) = 345 |

### 同カテゴリ内の適用順

- 同じカテゴリ（例: additive 同士）の中では、**`createdSeq` 昇順**（効果が付与された順）で適用
- 同 `createdSeq` の場合は、`initiativeTeamId` 側を先に適用

### 置換型（setTo）の特殊処理

- 「ステータスを N 固定にする」型の効果は、**clamp 前に適用** し、以後の additive/multiplicative を **受けない**
- 同一ステータスに複数の setTo が存在する場合、`createdSeq` が **最も新しい** ものが有効

### フィールド効果と状態効果の相互作用

- フィールド効果と状態効果が同名パラメータを修正する場合も、**共通パイプラインで処理**
- 種別ごとの優先度分離等の特例は設けない

### StatRecomputed イベント

ステータス再計算時にデバッグ/リプレイ検証用イベントを発行。

```
StatRecomputed {
  entityType: character | summonUnit | summonObject
  entityId: string
  stat: pAtk | pDef | mAtk | mDef | spd | tech | maxHp
  baseValue: number
  appliedEffects: [{ id, layer, delta }]
  finalValue: number
}
```

### 再計算トリガー

以下のタイミングで再計算を実行する。

- バフ/デバフの付与時
- バフ/デバフの解除時
- バフ/デバフの期限切れ時
- バースト状態の適用/解除時
- 召喚オブジェクトの生成/消滅時
- フィールド効果の展開/消滅時

---

## 6. ダメージ計算（DamageCalculator）

### 基本式

```
最終ダメージ = max(1, floor(ATK * カード倍率 * 100 / (100 + DEF * 防御係数)))
```

- **防御係数**: レギュレーション定義の `defenseCoefficient`（初期値 **0.6**）
- **最低保証**: ダメージの最低値は **1**

### ダメージ種別分岐

| 種別 | ATK | DEF | 計算 |
|---|---|---|---|
| **物理** | `pAtk`（バフ/デバフ込み） | `pDef`（バフ/デバフ込み） | 基本式 |
| **魔法** | `mAtk`（バフ/デバフ込み） | `mDef`（バフ/デバフ込み） | 基本式 |
| **真ダメージ** | — | — | `max(1, floor(固定値 or ATK * カード倍率))`。DEF 無視、シールド/軽減も無効 |

### クリティカル判定

```
critRate = clamp((attackerTech - defenderTech * 0.5) / 500, 0.02, 0.30)
```

- **最低クリティカル率**: 2%
- **最大クリティカル率**: 30%
- **クリティカルダメージ倍率**: **1.5**（固定）
- クリティカル発生時、最終ダメージに × 1.5 を乗算（端数切り捨て）
- 判定は `rngSeed` から決定的に行う

### 状態異常命中率

```
statusHitRate = clamp(baseRate + (attackerTech - defenderTech) * 0.001, 0.10, 0.95)
```

- **baseRate**: カード/状態異常定義で指定（例: 毒=0.80, 凍結=0.50）
- **最低命中率**: 10%
- **最大命中率**: 95%
- 判定は `rngSeed` から決定的に行う

### 端数処理

- 中間計算は **小数保持**
- 最終結果でまとめて **floor**（中間で floor しない）

### 計算フロー

```
1. ダメージ種別を判定（物理/魔法/真）
2. ATK / DEF を StatCalculator から取得（バフ/デバフ込み最終値）
3. 基本式でダメージ算出（小数保持）
4. クリティカル判定（RngProvider 使用）
   → クリティカルなら × 1.5
5. floor で整数化
6. max(1, ...) で最低保証
7. DamageDealt イベント発行
```

---

## 6.5. カード効果解決（EffectResolver）

### 責務

`EffectSpec`（`07_data_model.md`）をデータ駆動で解決するモジュール。
ActionResolver の「4d. 効果解決」ステップから呼び出され、EffectType に応じて既存モジュールへ処理を委譲する。

### 設計原則

- EffectResolver 自身は **ディスパッチャ** であり、実際の計算ロジックは持たない
- 各 EffectType → 既存モジュール（DamageCalculator, StatusEffectProcessor 等）へ委譲
- `Composite` は再帰的に子効果を解決、`Conditional` は条件を評価して分岐

### EffectContext（発動元情報）

効果解決時に必要なコンテキスト情報を保持する。

- `sourceTeamId`: string（発動元チーム）
- `sourceCharacterId`: string（発動元キャラ）
- `sourceCardId`: string（発動元カード）
- `sourceType`: `"skill" | "support" | "unique" | "summon" | "item" | "passive" | "setEffect"`
- `activationMode`: `"normal" | "interrupt"`
- `parentTarget?`: TargetSpec（親効果から継承された対象。Composite/Conditional 内のサブ効果用）

### EffectType → モジュール ディスパッチ対応表

| EffectType | 委譲先モジュール | 主な処理 |
|---|---|---|
| `Damage` | **DamageCalculator** | ダメージ種別判定 → ATK/DEF 取得 → 基本式 → クリティカル → DamageDealt イベント |
| `Heal` | **EffectResolver 内部** | calcMode に応じて回復量算出 → HP 加算 → Healed イベント |
| `Draw` | **DeckManager** | DrawCards(count) |
| `StatusApply` | **StatusEffectProcessor** | ApplyStatus(statusId, duration, ...) |
| `Dispel` | **StatusEffectProcessor** | DispelStatus(category, count, filterTags) |
| `Summon` | **SummonManager** | CreateSummon(summonCardId) |
| `ElementGain` | **EffectResolver 内部** | ElementCounters の該当色を加算 → ElementGained イベント |
| `ElementConsume` | **EffectResolver 内部** | ElementCounters の該当色を減算 → ElementConsumed イベント |
| `ElementConvert` | **EffectResolver 内部** | from を減算 → to を加算 → イベント発行 |
| `BurstGain` | **BurstManager** | AddBurstPoints(points) |
| `Shield` | **EffectResolver 内部** | シールド量算出 → 対象にシールド付与 → イベント発行 |
| `Resurrect` | **EffectResolver 内部** | ダウン状態解除 → HP を maxHp × hpPercent で設定 → イベント発行 |
| `FieldEffectDeploy` | **EffectResolver 内部** | FieldEffectInstance 生成 → TeamState.fieldEffect に配置 → イベント発行 |
| `ItemSet` | **EffectResolver 内部** | ItemInstance 生成 → キャラにセット（既存上書き） → イベント発行 |
| `APModify` | **EffectResolver 内部** | AP 増減 → clamp(0, unlockedMaxAP) → イベント発行 |
| `Composite` | **再帰呼び出し** | children を順次 ResolveEffect() |
| `Conditional` | **条件評価 → 再帰** | ConditionSpec を評価 → thenEffect or elseEffect を ResolveEffect() |

### Target 継承ルール

サブ効果（Composite の子要素 / Conditional の then/else）の対象指定:

1. サブ効果自身が `target` を持つ場合 → **そちらを使用**
2. サブ効果の `target` が null の場合 → **親効果の target を継承**
3. 親効果も target が null の場合 → **EffectContext.parentTarget** を参照
4. すべて null の場合 → **発動元カードの TargetSpec** を使用

### Composite の解決フロー

```
ResolveComposite(state, compositeSpec, context):
  1. children を先頭から順に取得
  2. 各 child について:
     a. child.target が null なら parentTarget を設定
     b. ResolveEffect(state, child, context) を再帰呼び出し
     c. 結果を state に反映
  3. 全 children 解決後に return
```

- 子効果は **記述順（配列順）** で解決される
- 途中で対象がダウンしても、残りの子効果は **スキップせず** 続行する（対象不在の効果は個別に無効化）

### Conditional の解決フロー

```
ResolveConditional(state, conditionalSpec, context):
  1. EvaluateCondition(state, conditionalSpec.condition, context) を呼び出し
  2. 結果が true:
     a. thenEffect を ResolveEffect(state, thenEffect, context)
  3. 結果が false:
     a. elseEffect が null でなければ ResolveEffect(state, elseEffect, context)
     b. elseEffect が null なら何もしない
```

### 条件評価（EvaluateCondition）

```
EvaluateCondition(state, conditionSpec, context):
  conditionType に応じて分岐:
  - ElementThreshold: キャラの ElementCounters[element] を operator で threshold と比較
  - HpThreshold: scope に応じて自分 or 対象の (currentHp / maxHp) を比較
  - StatusCheck: scope に応じて自分 or 対象の Statuses を statusId/tags で検索
  - BurstCheck: scope に応じて自分 or 対象の BurstState.IsBurst を判定
  - AllyDownCount: チームの IsDown==true のキャラ数を比較
  - HandCount: チームの Hand.Count を比較
```

### セット効果（SetEffects）の解決

セット効果は MatchSetup フェイズで解決される（`16_rule_engine_design.md` セクション2 参照）。

```
ResolveSetEffects(state, setEffects, context):
  1. 各 EffectSpec について EffectType の制約チェック（許可リストに含まれるか）
  2. 制約違反の場合はエラーログを出力してスキップ
  3. 許可された EffectSpec を ResolveEffect() で解決
```

### 複数対象の解決

`TargetSpec.mode` が `allAllies` / `allEnemies` など複数対象を指す場合の処理:

```
ResolveMultiTarget(state, effect, context, targets):
  1. 対象リストを生成:
     - allAllies → 味方チームの isDown==false の全キャラ
     - allEnemies → 敵チームの isDown==false の全キャラ
     - random → RngProvider で isDown==false のキャラから1体選択
  2. 対象リストを createdSeq 昇順でソート（処理順の決定性保証）
  3. 各対象について:
     - effect のコピーに target = { mode: 特定キャラ, targetIds: [id] } を設定
     - ResolveEffect(state, copiedEffect, context) を呼び出し
  4. 途中で対象がダウンしても残りの対象への効果は継続する
```

- Composite 内の子効果が allEnemies を持つ場合、**子効果ごとに** 全対象を処理する（親の対象を共有しない）
- ダメージ効果で `hitCount > 1` の場合、**同一対象に** hitCount 回ダメージ判定を繰り返す

### 蘇生処理（Resurrect）

```
ResolveResurrect(state, targetId, params, context):
  1. 対象チェック:
     - target.isDown == false → 効果なし（既に生存）
     - target.isDown == true → 蘇生処理に進む
  2. 蘇生実行:
     a. target.isDown = false
     b. target.hp = max(1, floor(target.maxHp * params.hpPercent))
     c. target.shield = null（シールドはリセット）
     d. target.statuses は維持（蘇生前の状態異常/バフを引き継ぐ）
     e. target.ap は変更なし（蘇生前の値を維持）
  3. CharacterResurrected イベント発行
  4. StatCalculator で蘇生キャラのステータス再計算
```

注意事項:
- 蘇生はカード効果として **制限なし** で許可（`01_core_rules.md` 確定済み）
- 勝敗判定は **バトルフェイズの全行動解決後** に行うため、蘇生で覆る
- 蘇生対象の `target` 指定は `allyCharacter`（ダウン済み味方を指定可能にする必要あり）
  - TargetSpec の `allyCharacter` は **ダウン済みキャラも対象候補に含む**（Resurrect 専用の例外）
  - Resurrect 以外の効果で TargetSpec がダウン済みキャラを指す場合、その効果は無効

### ダメージ適用フロー（DamageCalculator → ShieldProcessor → HP）

DamageCalculator が算出した最終ダメージを HP に適用するまでの統合フロー:

```
ApplyDamageToTarget(state, targetId, attackContext):
  1. DamageCalculator.CalculateDamage(attackContext)
     → DamageResult { finalDamage, isCritical, rawDamage }

  2. ダメージ種別分岐:
     a. DamageType == True（真ダメージ）:
        - シールド貫通: target.hp -= finalDamage
        - ShieldProcessor は呼ばない
     b. DamageType == Physical or Magical:
        - ShieldProcessor.AbsorbDamage(state, targetId, finalDamage, damageType)
        → ShieldAbsorbResult { absorbed, throughDamage, shieldDepleted }
        - target.hp -= throughDamage

  3. HP 下限: target.hp = max(0, target.hp)

  4. ダウン判定:
     target.hp == 0 → target.isDown = true → CharacterDowned イベント発行

  5. DamageDealt イベント発行:
     - amount = finalDamage
     - absorbed = shieldAbsorbed（0 if 真ダメージ）
     - isCritical
     - damageType

  6. 召喚ユニット経由のダメージ（03_summons.md）:
     対象が召喚ユニットを持つキャラの場合、攻撃は召喚ユニットへ差し替わる:
     a. 召喚ユニットに finalDamage を適用
     b. 紐づくキャラに floor(finalDamage / 2) の真ダメージ（半減ダメージ）
     c. 召喚ユニットの hp == 0 → SummonUnitExpired イベント
```

---


## 11.7. データフロー例（EffectResolver トレース）

Phase 1 実装の参考として、代表的なカードが EffectResolver を通じてどう解決されるかをトレースする。

### 例1: 複合攻撃カード「毒炎斬」

カード定義:
```yaml
id: skill_poison_flame
name: 毒炎斬
type: Skill
color: purple
cost: 3
priority: 0
effect:
  effectType: Composite
  target: { mode: enemyCharacter }
  compositeParams:
    children:
      - effectType: Damage
        damageParams:
          damageType: physical
          cardMultiplier: 1.2
      - effectType: StatusApply
        statusApplyParams:
          statusId: status_poison
          duration: 3
```

解決トレース:
```
1. ActionResolver: 行動をソート → priority=0 で実行順確定
2. ActionResolver: 不発チェック（AP≥3, 対象存在） → OK
3. ActionResolver: AP消費（ap -= 3）, エレメント獲得（purple += 1）
4. ActionResolver → EffectResolver.ResolveEffect(compositeSpec, context)

5. EffectResolver: EffectType=Composite → ResolveComposite()
   5a. Target 解決: compositeSpec.target = { mode: enemyCharacter } → 対象確定

   5b. child[0]: Damage
       - Target: null → 親の target を継承（enemyCharacter）
       - → DamageCalculator.CalculateDamage():
         ATK = source.pAtk（StatCalculator 最終値）
         DEF = target.pDef（StatCalculator 最終値）
         damage = max(1, floor(ATK * 1.2 * 100 / (100 + DEF * 0.6)))
         → クリティカル判定（RngProvider）
         → DamageDealt イベント発行
         → target.hp -= damage（ShieldInstance がある場合はシールド吸収を先に処理）

   5c. child[1]: StatusApply
       - Target: null → 親の target を継承
       - → StatusEffectProcessor.ApplyStatus():
         immune チェック → 命中判定（tech 依存, RngProvider）
         → 命中: StatusEffect を target.statuses に追加
         → StatusApplied イベント発行

6. ActionResolver: カードを graveyard へ移動
7. ActionResolver: ActionResolved(executed) イベント発行
8. ActionResolver: トリガー割り込み判定、Burst+30pt、アイテム自動発動チェック
```

### 例2: 条件分岐カード「炎嵐」

カード定義:
```yaml
id: skill_fire_storm
name: 炎嵐
type: Skill
color: red
cost: 4
priority: -1
effect:
  effectType: Conditional
  target: { mode: allEnemies }
  conditionalParams:
    condition:
      conditionType: ElementThreshold
      elementThresholdParams:
        element: red
        operator: gte
        threshold: 3
    thenEffect:
      effectType: Damage
      damageParams:
        damageType: magical
        cardMultiplier: 1.8
    elseEffect:
      effectType: Damage
      damageParams:
        damageType: magical
        cardMultiplier: 0.9
```

解決トレース:
```
1. EffectResolver: EffectType=Conditional → ResolveConditional()
2. EvaluateCondition: ElementThreshold
   → source.elements.red >= 3 ?
   → 例: red=4 → true
3. true → thenEffect を解決:
   Damage(magical, 1.8) を allEnemies に対して実行
   → 各敵キャラに対して DamageCalculator を呼び出し
4. （もし red=2 だった場合）
   false → elseEffect を解決:
   Damage(magical, 0.9) を allEnemies に対して実行
```

### 例3: セット効果（構築時パッシブ）

カード定義:
```yaml
id: skill_heavy_blade
name: 豪剣
type: Skill
cost: 4
setEffects:
  - effectType: Composite
    compositeParams:
      children:
        - effectType: StatusApply
          target: { mode: self }
          statusApplyParams:
            statusId: status_patk_up_loadout
            duration: -1  # 永続
        - effectType: StatusApply
          target: { mode: self }
          statusApplyParams:
            statusId: status_spd_down_loadout
            duration: -1  # 永続（デメリット）
```

解決タイミング: MatchSetup ステップ3a
```
1. EffectResolver: Composite → children を順次解決
2. child[0]: StatusApply(status_patk_up_loadout, 永続)
   → pAtk に additive +20 の永続バフを付与
   → LoadoutPassiveInstance として記録
3. child[1]: StatusApply(status_spd_down_loadout, 永続)
   → spd に additive -10 の永続デバフを付与（デメリット）
   → LoadoutPassiveInstance として記録
4. StatCalculator で影響キャラのステータス再計算
```

---

