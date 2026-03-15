# サブシステム設計（デッキ・状態異常・シールド・アイテム・フィールド効果）

ルールエンジン設計の一部。DeckManager、StatusEffectProcessor、ShieldProcessor、ItemManager、FieldEffectManager を定義する。

関連: [効果解決](20_stat_damage_effect.md) / [イベント・トリガー](18_event_and_trigger.md) / [召喚・バースト](21_summon_and_burst.md)

---

## 9. デッキ管理（DeckManager）

### ドロー

```
DrawCards(state, teamId, count):
  1. drawCount = count（通常: 生存キャラ数）
  2. while drawCount > 0:
     a. deck が空なら ReshuffleDeck(state, teamId)
     b. deck の先頭からカードを取り出し hand へ追加
     c. drawCount--
  3. CardDrawn イベント発行（各枚）
```

### デッキ再構築

```
ReshuffleDeck(state, teamId):
  1. rngStepBefore = rng.CurrentStep
  2. graveyard のカード全てを deck に移動
  3. Fisher-Yates シャッフル（rngSeed ベース）
  4. rngStepAfter = rng.CurrentStep
  5. DeckReshuffledFromTrash イベント発行:
     - trashReturnedCount
     - banishedIgnoredCount
     - rngSeedSnapshot, rngStepBefore, rngStepAfter
     - shuffleAlgorithm: "fisher-yates"
```

### ゾーン移動

| ゾーン | 役割 |
|---|---|
| **deck** | 山札（ドロー元） |
| **hand** | 手札（使用可能なカード） |
| **graveyard** | トラッシュ（使用済みカード。デッキ再構築時に戻る） |
| **banished** | 追放（召喚実体消滅先。デッキ再構築の対象外） |

- ゾーン間の移動は常にイベントとして記録する

---

## 10. 状態異常/バフ処理（StatusEffectProcessor）

### 付与フロー

```
ApplyStatus(state, target, statusDef, source):
  1. immune チェック:
     - target に immuneTo タグが一致する immune 効果があれば無効化
     - StatusApplied(immune) イベント発行
     - return
  2. 命中判定（デバフの場合）:
     - statusHitRate を算出（tech 依存）
     - RngProvider で判定
     - 失敗なら return
  3. stackingRule による分岐:
     a. replace:
        - 既存の同名状態を除去
        - 新しい状態を付与
     b. extend:
        - 既存の同名状態の remainingTurns に duration を加算
     c. stack:
        - 既存スタック数 < maxStacks なら新しいスタックを追加
        - maxStacks に達している場合は無視
  4. createdSeq を付与（globalSeqCounter）
  5. StatusApplied イベント発行
```

### 期限切れ

- ターン終了時に `duration` を 1 減少（`createdSeq` 昇順）
- 0 になったら除去し、`StatusExpired` イベント発行

### 解除（dispel）

```
DispelStatus(state, target, category, count):
  1. 対象の purgeable タグ付き状態を収集
  2. category (buff/debuff) でフィルタ
  3. createdSeq 昇順（古い方から）で count 件を解除
  4. StatusExpired(dispelled) イベント発行（各件）
```

### 効果タイプ分岐

| type | 処理 |
|---|---|
| **dot** | ターン開始/終了等のトリガータイミングで固定ダメージ or %ダメージ。`isHeal=true` なら回復 |
| **statMod** | StatCalculator のパイプラインに additive/multiplicative として参加 |
| **actionBlock** | 行動解決前に判定。`skip` なら行動スキップ、`chance` なら確率で不発 |
| **misc** | 上記に当てはまらない特殊効果（個別実装） |

---

## 10.5. シールド処理

### 概要

シールドはダメージ吸収用の一時的な防御膜。`CharacterState.shield` にキャラごと最大1枠保持する。

### 付与フロー

```
ApplyShield(state, targetId, shieldParams, context):
  1. シールド量を算出:
     - Fixed: value をそのまま使用
     - MaxHpPercent: floor(target.maxHp * value)
     - StatScale: floor(source の stat * value)
  2. 既存シールドがある場合 → ShieldExpired(overwritten) イベント発行
  3. 新規 ShieldInstance を生成:
     - remainingAmount = 算出値
     - remainingTurns = duration
     - createdSeq = state.NextSeq()
  4. target.shield = 新規 ShieldInstance
  5. ShieldApplied イベント発行
```

### ダメージ吸収フロー

DamageCalculator の計算結果をHPに適用する際、シールドによる吸収を挟む。

```
ApplyDamageWithShield(state, targetId, damage, damageType):
  1. damageType == True → シールドを貫通（吸収なし）、HP に直接適用
  2. target.shield == null → HP に直接適用
  3. target.shield != null:
     a. absorbed = min(shield.remainingAmount, damage)
     b. shield.remainingAmount -= absorbed
     c. throughDamage = damage - absorbed
     d. target.hp -= throughDamage
     e. ShieldConsumed イベント発行（absorbed, throughDamage）
     f. shield.remainingAmount == 0 → shield を null に、ShieldExpired イベント発行
```

### 期限切れ

- ターン終了時に `remainingTurns` を 1 減少（`createdSeq` 昇順、他の永続効果と統一）
- 0 になったらシールド消滅 → `ShieldExpired` イベント発行

---

## 10.6. アイテム管理（ItemManager）

### 概要

アイテムカードはキャラにセットされ、トリガー条件が満たされると自動発動する。
アイテム枠は **キャラごとに1枠**。

### セットフロー

```
SetItem(state, characterId, itemCardDef, context):
  1. 既存アイテムがある場合:
     a. ItemOverwritten イベント発行（既存アイテムの除去）
     b. 既存アイテムは graveyard へ移動
  2. ItemInstance を生成:
     - countRemaining = itemCardDef.count
     - triggers = itemCardDef.triggers
     - createdSeq = state.NextSeq()
  3. character.item = 新規 ItemInstance
  4. ItemSet イベント発行
```

### 自動発動フロー

バトルフェイズ中、各イベント処理後にアイテムのトリガー条件をスキャンする（TriggerSystem と連携）。

```
CheckItemTriggers(state, evt):
  1. 全キャラの item を走査
  2. 各 item.triggers をイベントと照合
  3. 条件成立:
     a. ItemTriggered イベント発行
     b. EffectResolver.ResolveEffect(itemCardDef.effect, context)
     c. item.countRemaining -= 1
     d. countRemaining == 0:
        - item を null に
        - カードを graveyard へ
        - ItemExhausted イベント発行
```

### 上書きと破棄の区別

| 状況 | イベント | カードの移動先 |
|---|---|---|
| 新アイテムで上書き | `ItemOverwritten` | graveyard |
| 残回数 0 で破棄 | `ItemExhausted` | graveyard |

- 上書きは「破棄」とは別扱い（将来「破棄時」誘発があっても上書きでは発火しない想定）

---

## 10.7. フィールド効果管理（FieldEffectManager）

### 概要

フィールド効果はチーム単位で保持される場の効果。各チーム1枠で、新規展開時は既存を上書きする。

### 展開フロー

```
DeployFieldEffect(state, ownerTeamId, fieldEffectDef, context):
  1. 既存フィールド効果がある場合:
     a. FieldEffectOverwritten イベント発行
     b. 既存を null に
  2. FieldEffectInstance を生成:
     - ownerTeamId, sourceTeamId = context から決定
     - duration = fieldEffectDef.defaultDurationTurns
     - modifiers = fieldEffectDef.modifiers
     - triggers = fieldEffectDef.triggers
     - createdSeq = state.NextSeq()
  3. team.fieldEffect = 新規 FieldEffectInstance
  4. FieldEffectDeployed イベント発行
  5. modifiers がある場合 → StatCalculator で影響キャラのステータス再計算
```

### 期限切れ

- ターン終了時に `remainingTurns` を 1 減少（`createdSeq` 昇順）
- 0 になったら消滅 → `FieldEffectExpired` イベント発行 → ステータス再計算

### 解除（dispel）

- `purgeable` タグ付きのフィールド効果のみ dispel 可能
- 解除時 → `FieldEffectRemoved` イベント発行 → ステータス再計算

---

