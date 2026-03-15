# 1vs1レギュレーション・キャラパッシブ・降参/切断 設計

ルールエンジン設計の一部。1vs1方式B対応、キャラパッシブシステム（AP上限連動解放）、降参・切断・タイムアウト処理を定義する。

関連: [ゲームループ](17_game_loop_and_phases.md) / [召喚・バースト](21_summon_and_burst.md) / [C#クラス設計](25_csharp_classes.md)

---

## 13. 1vs1レギュレーション対応

### 概要

1vs1方式B（3vs3編成流用1vs1）は、3キャラ編成をそのまま使いつつ、戦闘の意思決定を1キャラ（リーダー）に集約する。
既存の3vs3エンジンを **レギュレーション分岐** で対応し、専用のサブシステムは設けない。

### TeamState への追加

```csharp
public class TeamState
{
    // ... 既存フィールド ...

    /// <summary>1vs1方式B: リーダーキャラのID（null なら3vs3）</summary>
    public string LeaderCharacterId { get; set; }
}
```

### RegulationDefinition への追加（既存）

```csharp
public class OneVsOneRegulation
{
    public int? SupportSetMaxOverride { get; set; }   // 1vs1用サポート上限（例: 2）
    public bool ApSupplyEnabled { get; set; }          // AP供給の有無
    public int? ApSupplyMaxPerTurn { get; set; }       // AP供給上限/ターン（null=無制限）
    public int SummonSlotsForLeader { get; set; }      // リーダー召喚枠数（例: 3）
}
```

### 分岐ポイント一覧

| 処理 | 3vs3 | 1vs1方式B | 分岐条件 |
|---|---|---|---|
| **勝利条件** | 全キャラダウン | リーダーダウン | `regulation.VictoryCondition` |
| **カード使用主体** | 各キャラ | リーダーのみ | `LeaderCharacterId != null` |
| **召喚枠** | 各キャラ1枠 | リーダー3枠 | `oneVsOneRules.SummonSlotsForLeader` |
| **ドロー** | 生存キャラ数 | 常に3（サポートはダウンしない） | 共通ルール適用 |
| **AP供給** | なし | サポート→リーダー | `oneVsOneRules.ApSupplyEnabled` |
| **サポートの被ダメ** | 受ける | 受けない（攻撃対象外） | `IsSupport(charId)` |
| **ユニーク使用** | 各キャラ | リーダーのみ | `IsLeader(charId)` |
| **セット効果** | 全キャラ適用 | 全キャラ適用（サポート含む） | 変更なし |
| **パッシブ** | 全キャラ適用 | 全キャラ適用 | 変更なし |

### ヘルパーメソッド

```csharp
public static class TeamStateExtensions
{
    public static bool Is1v1(this TeamState team)
        => team.LeaderCharacterId != null;

    public static bool IsLeader(this TeamState team, string characterId)
        => team.LeaderCharacterId == characterId;

    public static bool IsSupport(this TeamState team, string characterId)
        => team.Is1v1() && !team.IsLeader(characterId);

    public static CharacterState GetLeader(this TeamState team)
        => team.Characters.First(c => c.Id == team.LeaderCharacterId);

    public static IEnumerable<CharacterState> GetSupports(this TeamState team)
        => team.Characters.Where(c => c.Id != team.LeaderCharacterId);
}
```

### AP供給処理

準備フェイズ中に実行される。サポートからリーダーへAPを譲渡する。

```
ProcessApSupply(state, teamId, supportCharacterId, amount):
  1. バリデーション:
     - oneVsOneRules.ApSupplyEnabled == true
     - support = team.GetCharacterById(supportCharacterId)
     - support が実際にサポートポジションであること
     - leader = team.GetLeader()
     - amount >= 0 && amount <= support.Ap
  2. 供給実行:
     - actualAmount = min(amount, leader.UnlockedMaxAP - leader.Ap)
       ※ リーダーの上限を超える分は切り捨て
     - support.Ap -= actualAmount
     - leader.Ap += actualAmount
  3. APSupplied イベント発行:
     - supportId, leaderId, requestedAmount, actualAmount
```

### 攻撃対象判定の分岐

```
IsValidAttackTarget(state, teamId, targetCharacterId):
  team = state.GetTeam(teamId)
  target = team.GetCharacterById(targetCharacterId)

  if target.IsDown → false
  if team.Is1v1() && team.IsSupport(targetCharacterId) → false
  → true
```

### 召喚枠の拡張（1vs1方式B）

リーダーは3つの召喚枠を使用できる。

```csharp
public class CharacterState
{
    // 3vs3: SummonSlot は1枠（既存）
    // 1vs1方式B: リーダーは SummonSlots[3] を使用
    public SummonSlotState SummonSlot { get; set; }          // 3vs3用（互換）
    public SummonSlotState[] SummonSlots { get; set; }       // 1vs1方式B用（リーダーのみ）
}
```

```
GetSummonSlots(state, characterId):
  team = state.GetTeamByCharacter(characterId)
  if team.Is1v1() && team.IsLeader(characterId):
    return character.SummonSlots  // [3]
  else:
    return [character.SummonSlot]  // [1]
```

- 1vs1方式Bの召喚入れ替えコスト軽減は「どの召喚枠に出すか」で判定
- 空き枠に出す場合は軽減なし

### 追加イベント

```csharp
public class APSuppliedEvent : GameEvent
{
    public string SupportId { get; set; }
    public string LeaderId { get; set; }
    public int RequestedAmount { get; set; }
    public int ActualAmount { get; set; }
}
```

---

## 14. キャラパッシブシステム

### 概要

キャラは **パッシブスキル枠を3つ** 持ち、AP上限（unlockedMaxAP）の解放に応じてパッシブが順次有効化される（`10_team_and_3v3_battle.md`）。

### データモデル（既存の補足）

```csharp
public class CharacterPassiveInstance
{
    public string PassiveId { get; set; }
    public int UnlockAtUnlockedMaxAP { get; set; }  // 解放閾値（例: 4, 6, 7）
    public bool IsActive { get; set; }
}
```

### パッシブ定義

```csharp
public class PassiveDefinition
{
    public string Id { get; set; }
    public string Name { get; set; }
    public PassiveType Type { get; set; }
    public List<StatModifier> Modifiers { get; set; }      // 常時ステ補正
    public List<CardTrigger> Triggers { get; set; }         // 誘発型
    public EffectSpec OnActivateEffect { get; set; }        // 解放時の即時効果（任意）
    public string RulesText { get; set; }
}

public enum PassiveType
{
    Constant,   // 常時有効（ステータス補正など）
    Triggered,  // イベント駆動（条件で発動）
    OnActivate  // 解放時に1回だけ発動
}
```

### 解放判定フロー

ターン開始フェイズのAP解放後に判定する。

```
CheckPassiveUnlocks(state):
  各チーム、各キャラについて:
    for each passive in character.Passives:
      if !passive.IsActive
         && character.UnlockedMaxAP >= passive.UnlockAtUnlockedMaxAP:
        passive.IsActive = true
        PassiveUnlocked イベント発行

        // 常時型: StatCalculator に修正を追加 → ステータス再計算
        if passiveDef.Type == Constant:
          passiveDef.Modifiers → StatCalculator パイプラインに参加
          StatCalculator.RecomputeAllStats(state, character.Id, Character)

        // 解放時型: 即時効果を解決
        if passiveDef.Type == OnActivate && passiveDef.OnActivateEffect != null:
          EffectResolver.ResolveEffect(state, passiveDef.OnActivateEffect, context)

        // 誘発型: TriggerSystem のスキャン対象に追加される（自動）
```

### TurnStart フェイズへの統合

セクション2のTurnStart処理を拡張する。

```
TurnStart:
  1. turn++
  2. 「ターン開始時」トリガー解決
  3. 召喚ユニットの hasActedThisTurn リセット
  4. AP解放: unlockedMaxAP += 1（apCap まで）
  5. AP回復: ap += apRegen（unlockedMaxAP まで）
  6. ★ パッシブ解放判定: CheckPassiveUnlocks(state)  ← 追加
     ※ AP解放後に判定することで、新たに閾値に到達したパッシブが即座に有効化される
```

### 追加イベント

```csharp
public class PassiveUnlockedEvent : GameEvent
{
    public string CharacterId { get; set; }
    public string PassiveId { get; set; }
    public int UnlockThreshold { get; set; }
    public int CurrentUnlockedMaxAP { get; set; }
}
```

---


## 16. 降参・切断・タイムアウト処理

### 概要

対戦中の中断処理（降参・切断・放置）を定義する（`01_core_rules.md` 確定済み）。

### 降参（Surrender）

```
ProcessSurrender(state, teamId):
  1. 降参はゲーム開始時から可能（ターン/フェイズの制限なし）
  2. 降参したチームは即座に敗北
  3. GameEnd 処理:
     - reason: "surrender"
     - winnerId: 相手チーム
     - loserId: 降参チーム
  4. SurrenderEvent 発行
```

### 切断/放置

```
ProcessDisconnect(state, teamId):
  1. 猶予時間（disconnectGracePeriodSec）を開始
     ※ 値は RegulationDefinition で定義（推奨: 30〜60秒）
  2. 猶予時間内に復帰:
     → 通常続行
  3. 猶予時間を超過:
     → 負け扱い
     - reason: "disconnect"
```

### 準備フェイズのタイムアウト

```
ProcessPreparationTimeout(state, teamId):
  1. 30秒の制限時間内に確定しなかった場合
  2. 未確定チームの処理:
     - 行動予約は空（パス扱い）
     - 決定速度ボーナス: +0pt
  3. 確定済みチームは通常通り処理
  4. バトルフェイズへ進行
```

### 追加イベント / データ

```csharp
public class SurrenderEvent : GameEvent
{
    public string TeamId { get; set; }
}

public class DisconnectEvent : GameEvent
{
    public string TeamId { get; set; }
    public int GracePeriodSec { get; set; }
    public bool Reconnected { get; set; }
}
```

---

