# 召喚管理・バーストシステム・召喚ユニット行動 設計

ルールエンジン設計の一部。SummonManager（生成/消滅/上書き/バースト変化）、BurstManager（ゲージ管理/バースト状態）、召喚ユニットの0コスト行動を定義する。

関連: [ゲームループ](17_game_loop_and_phases.md) / [効果解決](20_stat_damage_effect.md) / [サブシステム](22_subsystems.md)

---

## 7. 召喚管理（SummonManager）

### 生成フロー

```
1. 召喚カード使用（手札 → graveyard）
2. 入れ替え判定:
   a. 既存召喚がある場合 → コスト軽減計算
      軽減量 = floor(既存召喚の effectiveSummonCost / 2)
      実行コスト = max(0, 新カードcost - 軽減量)
   b. 既存召喚を SummonReplaced で除去（→ banished）
3. ステータス算出（召喚時に1回だけ確定）:
   summon.stat = character.stat * scaling + flat
   ※ character.stat はバフ/デバフ込みの「バトル中ステータス」
4. 召喚実体を生成（SummonSlotState に配置）
5. createdSeq を付与（globalSeqCounter からインクリメント）
6. 登場時効果（onEnterEffects）を解決
7. SummonObjectEntered / SummonUnitEntered イベント発行
```

### 上書きルール

- 召喚枠は **オブジェクト/ユニットで同一スロットを共有**
- 新しい召喚が展開されると、既存の召喚は **上書きで除去**
- 上書きは `SummonReplaced` イベントを発火
- on-expire トリガーは **上書き時に発火しない**（明示的に記載がない限り）
- 消滅した召喚実体は **追放ゾーン（banished）** へ

### 期限切れ

- ターン終了時に `durationTurnsRemaining` を 1 減少
- 0 になったら消滅（`SummonObjectExpired` / `SummonUnitExpired`）
- 消滅した召喚実体は **追放ゾーン（banished）** へ

### バースト変化

バースト状態適用時:

```
1. 既存召喚の burstVariantCardId を参照
2. バースト版カード定義からステータスを再算出
3. HP: 最大HP比で現在HPを再計算
   newHp = floor(currentHp / oldMaxHp * newMaxHp)
4. 寿命（durationTurnsRemaining）: 維持
5. 状態異常/バフ: 維持
6. 召喚ユニットの configuredActionSkillId: バースト版定義を使用
7. isBurstVariant = true
```

バースト状態解除時:

```
1. baseVariantCardId から通常版カード定義を特定
2. 通常版で再算出（HP比/寿命/状態の引き継ぎは同様）
3. baseVariantCardId == null の場合（フォールバック）:
   - 召喚はそのまま維持
   - バースト状態によるステータスボーナスのみ解除
```

### ゾーン管理

| 対象 | 移動先 |
|---|---|
| 召喚カード（使用時） | hand → graveyard |
| 召喚実体（消滅時） | field → banished |

- banished のカード/実体は「トラッシュ戻し」の対象外

---

## 8. バーストシステム（BurstManager）

### ゲージ管理

- **ポイント制**: 0 〜 burstMaxPoints（標準: 500）
- **1ゲージ = 100ポイント**
- UI表示用: `floor(burstPoints / 100)` でゲージ本数を算出
- 溢れは切り捨て

### 増加条件

| 条件 | 増加量 |
|---|---|
| **決定速度ボーナス**（10秒以内） | +70pt |
| **決定速度ボーナス**（20秒以内） | +50pt |
| **決定速度ボーナス**（30秒以内） | +30pt |
| **決定速度ボーナス**（時間切れ） | +0pt |
| **行動チャージ**（キャラ or 召喚ユニットが行動） | +30pt（デフォルト） |

- パッシブ等で蓄積率倍率がある場合: `floor(burstDeltaPoints * multiplier)`

### バースト状態の適用

```
1. Burst 1本（100pt）消費
2. バトルフェイズ開始時に対象キャラをバースト状態にする
3. ステータス上昇（倍率式、端数floor）:
   - pAtk × 1.15, mAtk × 1.15
   - pDef × 1.10, mDef × 1.10
   - spd × 1.10, tech × 1.10
   - maxHp: 変動なし
4. AP +2 回復（unlockedMaxAP まで）
5. unlockedMaxAP +1 解放
6. 召喚のバースト版への変化（SummonManager 連携）
7. ユニークカードの強化版への変化
8. 持続: 突入ターンを含め3ターン
9. createdSeq を付与（ターン終了処理順に使用）
```

### バースト状態の解除

```
1. ターン終了時に remainingTurns を 1 減少
2. remainingTurns == 0 で解除:
   - unlockedMaxAP が非バースト最大AP上限を超えている場合、元に戻す
   - 召喚のバースト版 → 通常版への変化
   - ユニークカードの強化版 → 通常版への変化
   - ステータス倍率の解除
```

### サポートカードの発動

```
1. burstPoints >= burstCostPoints を確認
2. burstPoints -= burstCostPoints
3. 効果解決
4. cooldownRemainingTurns = cooldownTurns（クールダウン開始）
5. SupportActivated イベント発行
```

---


## 15. 召喚ユニット行動管理

### 概要

召喚ユニットはバトルフェイズ中に **0コストのアクションスキル** を使用して行動する。
この行動はキャラの行動と同じソートに参加し、priority/spd で順番が決まる。

### 行動の予約と設定

召喚ユニットの行動は「準備フェイズで予約」ではなく、**召喚時に設定されたアクションスキル** を使用する。

```
PrepareForBattle 時:
  各チームの各キャラの召喚枠を走査:
    if summonSlot.Unit != null && !summonSlot.Unit.HasActedThisTurn:
      actionSkillId = summonSlot.Unit.ConfiguredActionSkillId
      if actionSkillId != null:
        → PlannedAction を生成:
          - actorType: Summon
          - actorId: summonUnit.InstanceId
          - source: SummonConfigured
          - cardId: actionSkillId
          - cardPriority: actionSkillDef.Priority
          - willConsumeApOnExecute: 0
```

### ソートへの参加

召喚ユニットの行動は、キャラの行動と **同じソートリスト** に混在する。

```
ソートキー:
  priority = アクションスキルの priority
  spd = 召喚ユニットの spd（StatCalculator 最終値）
  decisionSubmittedAtMs = 所有チームの決定時刻
  initiativeTeamId = ゲーム開始時決定
  createdSeq = 召喚ユニットの createdSeq
```

### 解決フロー

```
ResolveSummonAction(state, action):
  1. 召喚ユニットの存在チェック:
     - summonUnit が消滅済み → スキップ
  2. hasActedThisTurn チェック:
     - true → スキップ（1ターン1回制限）
  3. アクションスキルの効果解決:
     - EffectResolver.ResolveEffect(state, actionSkillDef.Effect, context)
     - context.SourceType = "summon"
  4. hasActedThisTurn = true
  5. Burst 増加: +30pt（行動チャージ）
  6. UnitActed イベント発行
  7. トリガー判定、アイテム自動発動チェック
```

### 召喚入れ替え時の行動取消し

同一ターンに召喚ユニットが入れ替わった場合の処理（`11_questions_and_proposals.md` 1.5 確定済み）。

```
OnSummonReplaced(state, oldUnitId, newUnit):
  1. 旧召喚ユニットの未解決行動を予約リストから検索
  2. 旧ユニットが未行動（HasActedThisTurn == false）:
     → 旧ユニットの予約を除外
  3. 新ユニットの行動を予約リストに追加:
     → PrepareForBattle と同様に PlannedAction を生成
     → ソート順に挿入（現在の解決位置より後のみ有効）
```

### アクションスキル定義

```csharp
public class ActionSkillDefinition
{
    public string Id { get; set; }
    public string Name { get; set; }
    public int Priority { get; set; }
    public EffectSpec Effect { get; set; }
    public string RulesText { get; set; }
}
```

### 追加イベント

```csharp
public class UnitActedEvent : GameEvent
{
    public string SummonInstanceId { get; set; }
    public string ActionSkillId { get; set; }
    public string OwnerCharacterId { get; set; }
}
```

---

