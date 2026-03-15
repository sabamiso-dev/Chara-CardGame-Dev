# イベントシステムとトリガー判定

ルールエンジン設計の一部。イベント駆動の中核（EventQueue）とトリガー判定処理（TriggerSystem）を定義する。

関連: [行動解決](19_action_resolver.md) / [効果解決](20_stat_damage_effect.md) / [サブシステム](22_subsystems.md)

---

## 3. イベントシステム

### EventQueue の構造

```
EventQueue（FIFO キュー）
├── Enqueue(event): イベントを末尾に追加
├── Dequeue(): 先頭のイベントを取り出して処理
├── ProcessAll(): キューが空になるまで順次処理
└── Count: 現在キュー内のイベント数
```

- キューは **FIFO**（先入先出）で処理する
- 処理中に追加されたイベントは **末尾** に積まれる
- 1つのイベント処理が完了してから次のイベントを処理する

### イベントの発火 → 収集 → 解決フロー

```
アクション実行
    │
    ▼
イベント生成（例: DamageDealt）
    │
    ▼
EventQueue に Enqueue
    │
    ▼
イベント処理開始（Dequeue）
    │
    ▼
トリガー条件スキャン
├── アイテムのトリガー条件をチェック
├── パッシブの常時参照
├── 状態異常の誘発条件をチェック
├── フィールド効果のトリガーをチェック
└── 予約済みカードのトリガー条件をチェック
    │
    ▼
条件成立したトリガーの追加イベントを Enqueue
    │
    ▼
次のイベントを Dequeue（繰り返し）
```

### トリガー判定の仕組み

各イベント処理後に、以下の順序でトリガー条件をスキャンする。

1. **アイテムのトリガー**: キャラにセットされたアイテムの `triggers` をチェック
2. **パッシブの常時参照**: 解放済みパッシブの参照条件をチェック
3. **状態異常/バフの誘発**: `triggers` フィールドを持つ StatusEffect をチェック
4. **フィールド効果の誘発**: FieldEffectInstance の `triggers` をチェック
5. **予約済みカードのトリガー**: バトルフェイズ中、予約済みカードの `trigger` 条件をチェック

### 無限ループ対策

- 同一フェイズ内で、**同一エンティティから発生する誘発回数は上限 10 回**
- 上限に達した場合、それ以上の誘発は無視する
- カウンタはフェイズ開始時にリセットされる

### 同時誘発の解決順

1. **イニシアチブ側が先** に解決順を決める
2. 同一プレイヤー内の複数誘発は、**所有者が解決順を選択** する
3. 最終タイブレークは `initiativeTeamId` を使用

---


## 10.8. トリガー判定処理（TriggerSystem）

### 責務

イベント発生時にゲーム中の全トリガーソースをスキャンし、条件が成立したトリガーを収集・実行する。
セクション3（イベントシステム）で概要を記載したが、ここでは処理ロジックの詳細を定義する。

### トリガーソース一覧

| ソース | 保持場所 | フェイズ | 備考 |
|---|---|---|---|
| **アイテム** | `CharacterState.item.triggers` | 全フェイズ | 条件成立で自動発動。ItemManager と連携 |
| **キャラパッシブ** | `CharacterState.passives` | 全フェイズ | `isActive == true` のもののみ |
| **状態異常/バフ** | `StatusEffect.triggers` | 全フェイズ | 誘発型の状態効果 |
| **フィールド効果** | `FieldEffectInstance.triggers` | 全フェイズ | 誘発型のフィールド効果 |
| **予約済みカード** | `PlannedAction` の `CardTrigger` | バトルフェイズのみ | 割り込み早期発動 |

### スキャンフロー

```
ScanTriggers(state, evt):
  1. triggeredActions = []
  2. triggerCounters を参照（無限ループ対策）

  ── アイテムトリガー ──
  3. 全キャラの item を走査:
     item != null && item.triggers がイベント evt に一致
     → triggerCounters[item.instanceId] < 10 を確認
     → 一致: { source: "item", entityId, action } を triggeredActions に追加

  ── パッシブトリガー ──
  4. 全キャラの passives を走査:
     passive.isActive == true && passive に誘発条件がある && evt に一致
     → 一致: { source: "passive", entityId, action } を追加

  ── 状態異常/バフトリガー ──
  5. 全キャラの statuses を走査:
     status.triggers が evt に一致
     → 一致: { source: "status", entityId, action } を追加

  ── フィールド効果トリガー ──
  6. 両チームの fieldEffect を走査:
     fieldEffect != null && fieldEffect.triggers が evt に一致
     → 一致: { source: "fieldEffect", entityId, action } を追加

  ── 予約済みカードトリガー（バトルフェイズのみ） ──
  7. state.currentPhase == Battle の場合:
     未解決の PlannedAction で CardTrigger を持つものを走査:
     trigger.event が evt に一致 && trigger.condition を評価
     → 一致: { source: "cardTrigger", entityId, action } を追加

  8. return triggeredActions
```

### トリガー条件マッチング

```
MatchTriggerCondition(trigger, evt):
  1. trigger.event と evt の型を照合:
     - "onDamaged" → evt is DamageDealtEvent && target == trigger所有者
     - "onAllyDamaged" → evt is DamageDealtEvent && target は味方キャラ
     - "onEnemyAction" → evt is ActionResolvedEvent && actor は敵チーム
     - "onAllyHpBelow" → evt is DamageDealtEvent && 味方の hp/maxHp < threshold
     - "onEnemySummon" → evt is SummonEnteredEvent && 敵チーム
     - "onTurnStart" → evt is TurnStartedEvent
     - "onTurnEnd" → evt is TurnEndedEvent
     - etc.
  2. trigger.condition が存在する場合:
     - 条件文字列を評価（例: "damageAmount >= 100"）
     - Phase 1 では固定パターンのパーサーで対応
       サポートする条件パターン:
       - "damageAmount >= N"
       - "hpPercent <= N"
       - "elementCount(color) >= N"
     - 条件不成立 → false
  3. 全条件成立 → true
```

### 割り込み解決フロー（予約済みカードトリガー）

```
ResolveInterrupt(state, triggeredCardActions):
  1. 複数のトリガーが同時成立した場合:
     a. イニシアチブ側のトリガーを先に処理
     b. 同一チーム内で複数 → 所有者が解決順を選択
        （Phase 1 では createdSeq 昇順で自動決定）
  2. 各トリガーカードについて:
     a. 不発チェック（AP不足等）
     b. AP消費
     c. EffectResolver.ResolveEffect(cardDef.effect, context{activationMode: Interrupt})
     d. カードを graveyard へ
     e. ActionResolved(executed, interrupt) イベント発行
     f. 該当カードを予約リストから除外（以後の priority 解決に参加しない）
  3. 割り込み解決後、通常の行動解決を再開
```

### 無限ループ対策の実装

```csharp
public class TriggerCounterMap
{
    private readonly Dictionary<string, int> counters = new();
    private const int MaxTriggersPerEntity = 10;

    public bool CanTrigger(string entityId)
        => !counters.ContainsKey(entityId) || counters[entityId] < MaxTriggersPerEntity;

    public void Increment(string entityId)
    {
        if (!counters.ContainsKey(entityId)) counters[entityId] = 0;
        counters[entityId]++;
    }

    public void ResetAll() => counters.Clear();
}
```

- `TriggerCounterMap` はフェイズ開始時に `ResetAll()` で初期化
- 各トリガー発動時に `Increment()` で加算
- `CanTrigger()` が false なら、そのエンティティからの誘発は無視

---

