# C# クラス/インターフェース設計

ルールエンジン設計の一部。Phase 1 実装で使用する全 C# クラス・インターフェース・データモデル・enum の定義。

関連: [アーキテクチャ概要](16_architecture_overview.md) / [効果解決](20_stat_damage_effect.md) / [実装ロードマップ](26_implementation_roadmap.md)

---

## 12. C# クラス/インターフェース設計

### GameManager

#### IGameManager / GameManager

```csharp
public class MatchConfig
{
    public RegulationDefinition Regulation { get; set; }
    public TeamConfig Team1 { get; set; }
    public TeamConfig Team2 { get; set; }
    public long RngSeed { get; set; }
}

public class TeamConfig
{
    public string TeamId { get; set; }
    public List<CharacterConfig> Characters { get; set; }  // [3]
    public List<string> SupportCardIds { get; set; }       // max 3
}

public class CharacterConfig
{
    public string CharacterId { get; set; }
    public List<string> DeckCardIds { get; set; }          // 10-15
    public string UniqueCardId { get; set; }
}

public class MatchResult
{
    public string WinnerId { get; set; }
    public string LoserId { get; set; }
    public string Reason { get; set; }  // "allDown" / "leaderDown" / "timeout_hpRatio" / "timeout_initiative"
    public int FinalTurn { get; set; }
    public List<GameEvent> EventLog { get; set; }
    public GameState FinalState { get; set; }
}

public interface IGameManager
{
    /// <summary>マッチを初期化する</summary>
    GameState InitializeMatch(MatchConfig config);

    /// <summary>マッチを実行する（初期化→ターンループ→終了）</summary>
    MatchResult RunMatch(MatchConfig config);

    /// <summary>勝敗判定</summary>
    VictoryResult CheckVictory(GameState state);
}

public class VictoryResult
{
    public bool IsDecided { get; set; }
    public string WinnerId { get; set; }
    public string LoserId { get; set; }
    public string Reason { get; set; }
}
```

### コア

#### IGameState / GameState

ゲーム全体の状態を保持する。

```csharp
public interface IGameState
{
    int Turn { get; }
    Phase CurrentPhase { get; }
    string InitiativeTeamId { get; }
    IEventQueue EventQueue { get; }
    TeamState[] Teams { get; }  // [2]
    int GlobalSeqCounter { get; }
    IRngProvider Rng { get; }
}

public class GameState : IGameState
{
    public int Turn { get; set; }
    public Phase CurrentPhase { get; set; }
    public string InitiativeTeamId { get; set; }
    public IEventQueue EventQueue { get; set; }
    public TeamState[] Teams { get; set; }
    public int GlobalSeqCounter { get; set; }
    public IRngProvider Rng { get; set; }

    /// <summary>globalSeqCounter をインクリメントして返す</summary>
    public int NextSeq() => ++GlobalSeqCounter;
}

public enum Phase
{
    MatchSetup,
    TurnStart,
    Draw,
    Preparation,
    Battle,
    TurnEnd,
    GameEnd
}
```

#### IEventQueue / EventQueue

```csharp
public interface IEventQueue
{
    void Enqueue(GameEvent evt);
    GameEvent Dequeue();
    bool HasPending { get; }
    int Count { get; }
}

public class EventQueue : IEventQueue
{
    private readonly Queue<GameEvent> queue = new();

    public void Enqueue(GameEvent evt) => queue.Enqueue(evt);
    public GameEvent Dequeue() => queue.Dequeue();
    public bool HasPending => queue.Count > 0;
    public int Count => queue.Count;
}
```

#### GameEvent（基底）+ 各イベント型

```csharp
public abstract class GameEvent
{
    public int Turn { get; set; }
    public Phase Phase { get; set; }
    public long Timestamp { get; set; }
}

// 代表的なイベント型
public class TurnStartedEvent : GameEvent { }
public class TurnEndedEvent : GameEvent { }
public class PhaseChangedEvent : GameEvent
{
    public Phase NewPhase { get; set; }
}

public class ActionResolvedEvent : GameEvent
{
    public string ActionId { get; set; }
    public ActorType ActorType { get; set; }
    public string ActorId { get; set; }
    public string CardId { get; set; }
    public ActionOutcome ResolvedAs { get; set; }  // Executed / Fizzled
    public FizzleReason? FizzleReason { get; set; }
    public ActivationMode ActivationMode { get; set; }  // Normal / Interrupt
    public int ApBefore { get; set; }
    public int ApCostApplied { get; set; }
    public int ApAfter { get; set; }
    public ActionSortKey SortKey { get; set; }
    public TieBreakLevel TieBreakLevelUsed { get; set; }
}

public class DamageDealtEvent : GameEvent
{
    public string SourceId { get; set; }
    public string TargetId { get; set; }
    public int Amount { get; set; }
    public DamageType DamageType { get; set; }
    public bool IsCritical { get; set; }
}

public class StatusAppliedEvent : GameEvent
{
    public string TargetId { get; set; }
    public string StatusId { get; set; }
    public string SourceId { get; set; }
    public int Duration { get; set; }
}

public class CardDrawnEvent : GameEvent
{
    public string TeamId { get; set; }
    public string CardId { get; set; }
}

public class DeckReshuffledEvent : GameEvent
{
    public string TeamId { get; set; }
    public int TrashReturnedCount { get; set; }
    public int BanishedIgnoredCount { get; set; }
    public int RngStepBefore { get; set; }
    public int RngStepAfter { get; set; }
}

public class SummonEnteredEvent : GameEvent
{
    public string SummonInstanceId { get; set; }
    public string CardId { get; set; }
    public string OwnerCharacterId { get; set; }
    public SummonType SummonType { get; set; }  // Object / Unit
}

public class SummonExpiredEvent : GameEvent
{
    public string SummonInstanceId { get; set; }
    public SummonType SummonType { get; set; }
}

public class SummonReplacedEvent : GameEvent
{
    public string OldSummonInstanceId { get; set; }
    public string NewSummonInstanceId { get; set; }
}

public class StatRecomputedEvent : GameEvent
{
    public EntityType EntityType { get; set; }
    public string EntityId { get; set; }
    public StatType Stat { get; set; }
    public int BaseValue { get; set; }
    public List<AppliedEffect> AppliedEffects { get; set; }
    public int FinalValue { get; set; }
}

public class ShieldAppliedEvent : GameEvent
{
    public string TargetId { get; set; }
    public string SourceId { get; set; }
    public int Amount { get; set; }
    public int Duration { get; set; }
}

public class ShieldConsumedEvent : GameEvent
{
    public string TargetId { get; set; }
    public int Absorbed { get; set; }
    public int ThroughDamage { get; set; }
}

public class ShieldExpiredEvent : GameEvent
{
    public string TargetId { get; set; }
    public ShieldRemovalReason Reason { get; set; }
}

public class CharacterResurrectedEvent : GameEvent
{
    public string TargetId { get; set; }
    public string SourceId { get; set; }
    public int RestoredHp { get; set; }
}

public class ElementConvertedEvent : GameEvent
{
    public string CharacterId { get; set; }
    public string FromElement { get; set; }
    public string ToElement { get; set; }
    public int Amount { get; set; }
}

public class APModifiedEvent : GameEvent
{
    public string CharacterId { get; set; }
    public int Amount { get; set; }
    public int ApBefore { get; set; }
    public int ApAfter { get; set; }
}

public class ItemSetEvent : GameEvent
{
    public string CharacterId { get; set; }
    public string ItemCardId { get; set; }
}

public class ItemTriggeredEvent : GameEvent
{
    public string CharacterId { get; set; }
    public string ItemCardId { get; set; }
    public string TriggerEvent { get; set; }
    public int CountRemaining { get; set; }
}

public class ItemExhaustedEvent : GameEvent
{
    public string CharacterId { get; set; }
    public string ItemCardId { get; set; }
}

public class ItemOverwrittenEvent : GameEvent
{
    public string CharacterId { get; set; }
    public string OldItemCardId { get; set; }
    public string NewItemCardId { get; set; }
}

public enum ShieldRemovalReason { Expired, Overwritten, Depleted }
```

#### IRngProvider / SeededRng

```csharp
public interface IRngProvider
{
    int CurrentStep { get; }
    double NextDouble();
    int NextInt(int minInclusive, int maxExclusive);

    /// <summary>Fisher-Yates シャッフル</summary>
    void Shuffle<T>(IList<T> list);
}

public class SeededRng : IRngProvider
{
    private readonly long seed;
    public int CurrentStep { get; private set; }

    public SeededRng(long seed)
    {
        this.seed = seed;
        CurrentStep = 0;
    }

    public double NextDouble() { /* xoshiro256 等 */ CurrentStep++; /* ... */ }
    public int NextInt(int min, int max) { /* ... */ }

    public void Shuffle<T>(IList<T> list)
    {
        for (int i = list.Count - 1; i > 0; i--)
        {
            int j = NextInt(0, i + 1);
            (list[i], list[j]) = (list[j], list[i]);
        }
    }
}
```

### プロセッサ

#### ITurnProcessor / TurnProcessor

```csharp
public interface ITurnProcessor
{
    /// <summary>1ターン全体を処理する</summary>
    GameState ProcessTurn(GameState state);

    /// <summary>指定フェイズを処理する</summary>
    GameState ProcessPhase(GameState state, Phase phase);
}

public class TurnProcessor : ITurnProcessor
{
    private readonly IDeckManager deckManager;
    private readonly IActionResolver actionResolver;
    private readonly IEffectResolver effectResolver;
    private readonly IStatusEffectProcessor statusEffectProcessor;
    private readonly IBurstManager burstManager;
    private readonly ISummonManager summonManager;

    public TurnProcessor(
        IDeckManager deckManager,
        IActionResolver actionResolver,
        IEffectResolver effectResolver,
        IStatusEffectProcessor statusEffectProcessor,
        IBurstManager burstManager,
        ISummonManager summonManager)
    {
        this.deckManager = deckManager;
        this.actionResolver = actionResolver;
        this.effectResolver = effectResolver;
        this.statusEffectProcessor = statusEffectProcessor;
        this.burstManager = burstManager;
        this.summonManager = summonManager;
    }

    public GameState ProcessTurn(GameState state)
    {
        state = ProcessPhase(state, Phase.TurnStart);
        state = ProcessPhase(state, Phase.Draw);
        state = ProcessPhase(state, Phase.Preparation);
        state = ProcessPhase(state, Phase.Battle);
        state = ProcessPhase(state, Phase.TurnEnd);
        return state;
    }

    public GameState ProcessPhase(GameState state, Phase phase) { /* ... */ }
}
```

#### IActionResolver / ActionResolver

```csharp
public interface IActionResolver
{
    /// <summary>予約行動を解決し、結果リストを返す</summary>
    List<ActionResult> ResolveActions(GameState state, List<PlannedAction> actions);

    /// <summary>不発チェック</summary>
    bool CheckFizzle(GameState state, PlannedAction action, out FizzleReason reason);

    /// <summary>行動をソートキーで並び替える</summary>
    List<PlannedAction> SortActions(GameState state, List<PlannedAction> actions);
}
```

#### IStatCalculator / StatCalculator

```csharp
public interface IStatCalculator
{
    /// <summary>5段階パイプラインで最終ステータスを算出</summary>
    int CalculateFinalStat(StatType stat, int baseValue, List<StatModifier> modifiers);

    /// <summary>キャラの全ステータスを再計算</summary>
    void RecomputeAllStats(GameState state, string entityId, EntityType entityType);
}

public class StatCalculator : IStatCalculator
{
    public int CalculateFinalStat(StatType stat, int baseValue, List<StatModifier> modifiers)
    {
        // 1. setTo チェック（最新の createdSeq が有効）
        // 2. additive を createdSeq 昇順で加算
        // 3. multiplicative を createdSeq 昇順で乗算
        // 4. clamp 適用
        // 5. floor + 下限0（maxHp は下限1）
        return 0; // placeholder
    }

    public void RecomputeAllStats(GameState state, string entityId, EntityType entityType)
    {
        // 全ステータスに対して CalculateFinalStat を呼び出し
        // StatRecomputed イベントを発行
    }
}
```

#### IDamageCalculator / DamageCalculator

```csharp
public interface IDamageCalculator
{
    /// <summary>ダメージを計算する</summary>
    DamageResult CalculateDamage(AttackContext context);

    /// <summary>クリティカル判定</summary>
    bool RollCritical(AttackContext context, IRngProvider rng);

    /// <summary>状態異常命中判定</summary>
    bool RollStatusHit(float baseRate, int attackerTech, int defenderTech, IRngProvider rng);
}

public struct AttackContext
{
    public string AttackerId;
    public string DefenderId;
    public DamageType DamageType;
    public float CardMultiplier;
    public int Atk;           // StatCalculator で算出済みの最終値
    public int Def;           // 同上
    public int AttackerTech;
    public int DefenderTech;
    public float DefenseCoefficient;
}

public struct DamageResult
{
    public int FinalDamage;
    public bool IsCritical;
    public int RawDamage;      // クリティカル前
}
```

#### ITriggerSystem / TriggerSystem

```csharp
public interface ITriggerSystem
{
    /// <summary>イベントに対してトリガー条件をスキャンし、発動対象を返す</summary>
    List<TriggeredAction> ScanTriggers(GameState state, GameEvent evt);

    /// <summary>無限ループ対策用の誘発カウンタを管理</summary>
    bool CanTrigger(string entityId, Phase currentPhase);
}
```

#### ISummonManager / SummonManager

```csharp
public interface ISummonManager
{
    /// <summary>召喚を生成する</summary>
    GameState CreateSummon(GameState state, string characterId, string summonCardId);

    /// <summary>召喚を除去する</summary>
    GameState RemoveSummon(GameState state, string summonInstanceId, SummonRemovalReason reason);

    /// <summary>バースト版への変化</summary>
    GameState TransformToBurstVariant(GameState state, string characterId);

    /// <summary>通常版への変化（バースト解除時）</summary>
    GameState TransformToBaseVariant(GameState state, string characterId);

    /// <summary>入れ替えコスト軽減を算出</summary>
    int CalculateReplacementCostReduction(GameState state, string characterId);
}
```

#### IBurstManager / BurstManager

```csharp
public interface IBurstManager
{
    /// <summary>バーストポイントを加算</summary>
    GameState AddBurstPoints(GameState state, string teamId, int points);

    /// <summary>バースト状態を適用</summary>
    GameState ApplyBurstMode(GameState state, string characterId);

    /// <summary>バースト状態を解除</summary>
    GameState RemoveBurstMode(GameState state, string characterId);

    /// <summary>サポートカードを発動</summary>
    GameState ActivateSupport(GameState state, string teamId, string supportCardId);

    /// <summary>決定速度ボーナスを算出</summary>
    int CalculateDecisionSpeedBonus(int decisionTimeMs);
}
```

#### IDeckManager / DeckManager

```csharp
public interface IDeckManager
{
    /// <summary>カードをドローする</summary>
    GameState DrawCards(GameState state, string teamId, int count);

    /// <summary>デッキを再構築する（トラッシュ→デッキ）</summary>
    GameState ReshuffleDeck(GameState state, string teamId);

    /// <summary>カードをゾーン間で移動する</summary>
    GameState MoveCard(GameState state, string teamId, string cardId, CardZone from, CardZone to);
}

public enum CardZone
{
    Deck,
    Hand,
    Graveyard,
    Banished
}
```

#### IStatusEffectProcessor / StatusEffectProcessor

```csharp
public interface IStatusEffectProcessor
{
    /// <summary>状態を付与する</summary>
    GameState ApplyStatus(GameState state, string targetId, StatusEffectDefinition statusDef,
                          string sourceId, int duration);

    /// <summary>状態を解除する（dispel）</summary>
    GameState DispelStatus(GameState state, string targetId, string category, int count);

    /// <summary>ターン終了時の期限切れ処理</summary>
    GameState ProcessExpiration(GameState state);

    /// <summary>DoT/HoT の効果を適用する</summary>
    GameState ProcessDotEffects(GameState state, string targetId);
}
```

#### IItemManager / ItemManager

```csharp
public interface IItemManager
{
    /// <summary>アイテムをキャラにセットする</summary>
    GameState SetItem(GameState state, string characterId, string itemCardId);

    /// <summary>アイテムの自動発動をチェック・実行する</summary>
    GameState CheckAndTriggerItems(GameState state, GameEvent evt);

    /// <summary>アイテムを除去する（破棄 or 上書き）</summary>
    GameState RemoveItem(GameState state, string characterId, ItemRemovalReason reason);
}

public enum ItemRemovalReason { Exhausted, Overwritten }
```

#### IFieldEffectManager / FieldEffectManager

```csharp
public interface IFieldEffectManager
{
    /// <summary>フィールド効果を展開する</summary>
    GameState DeployFieldEffect(GameState state, string ownerTeamId,
                                 string fieldEffectDefId, EffectContext context);

    /// <summary>フィールド効果を解除する（dispel）</summary>
    GameState RemoveFieldEffect(GameState state, string teamId);

    /// <summary>ターン終了時の期限切れ処理</summary>
    GameState ProcessFieldEffectExpiration(GameState state);
}
```

#### IShieldProcessor

```csharp
public interface IShieldProcessor
{
    /// <summary>シールドを付与する</summary>
    GameState ApplyShield(GameState state, string targetId,
                           ShieldEffectParams shieldParams, EffectContext context);

    /// <summary>ダメージにシールド吸収を適用する</summary>
    ShieldAbsorbResult AbsorbDamage(GameState state, string targetId,
                                     int damage, DamageType damageType);

    /// <summary>ターン終了時の期限切れ処理</summary>
    GameState ProcessShieldExpiration(GameState state);
}

public struct ShieldAbsorbResult
{
    public int Absorbed;
    public int ThroughDamage;
    public bool ShieldDepleted;
}
```

#### ITriggerSystem / TriggerSystem（詳細）

```csharp
public interface ITriggerSystem
{
    /// <summary>イベントに対してトリガー条件をスキャンし、発動対象を返す</summary>
    List<TriggeredAction> ScanTriggers(GameState state, GameEvent evt);

    /// <summary>トリガー条件をイベントと照合する</summary>
    bool MatchTriggerCondition(CardTrigger trigger, GameEvent evt, GameState state);

    /// <summary>無限ループ対策用の誘発カウンタを管理</summary>
    bool CanTrigger(string entityId);

    /// <summary>フェイズ開始時にカウンタをリセット</summary>
    void ResetCounters();
}

public class TriggeredAction
{
    public string EntityId { get; set; }
    public string SourceType { get; set; }  // "item" / "passive" / "status" / "fieldEffect" / "cardTrigger"
    public string CardId { get; set; }
    public PlannedAction Action { get; set; }  // cardTrigger の場合のみ
}
```

#### ILoadoutValidator

```csharp
public interface ILoadoutValidator
{
    /// <summary>編成がレギュレーションに適合しているか検証する</summary>
    LoadoutValidationResult Validate(TeamConfig teamConfig, RegulationDefinition regulation);
}

public class LoadoutValidationResult
{
    public bool IsValid => Errors.Count == 0;
    public List<string> Errors { get; set; } = new();
}
```

### データモデル（`07_data_model.md` 準拠）

```csharp
// --- チーム/キャラ ---
public class TeamState
{
    public string Id { get; set; }
    public CharacterState[] Characters { get; set; }  // [3]
    public ResourcesState Resources { get; set; }
    public List<string> Deck { get; set; }
    public List<string> Hand { get; set; }
    public List<string> Graveyard { get; set; }
    public List<string> Banished { get; set; }
    public SupportSlotState[] SupportSet { get; set; }
    public List<PlannedAction> PlannedActions { get; set; }
    public PreparationDecisionState Decision { get; set; }
    public FieldEffectInstance FieldEffect { get; set; }  // nullable
}

public class CharacterState
{
    public string Id { get; set; }
    public int Hp { get; set; }
    public int MaxHp { get; set; }
    public bool IsDown { get; set; }
    public CharacterStats Stats { get; set; }
    public List<StatusEffect> Statuses { get; set; }
    public int Ap { get; set; }
    public int UnlockedMaxAP { get; set; }
    public int ApCap { get; set; }
    public int ApRegen { get; set; }
    public List<CharacterPassiveInstance> Passives { get; set; }
    public List<LoadoutPassiveInstance> LoadoutPassives { get; set; }
    public UniqueSlotState Unique { get; set; }
    public ItemInstance Item { get; set; }  // nullable, max 1
    public SummonSlotState SummonSlot { get; set; }  // nullable
    public BurstState Burst { get; set; }
    public ElementCounters Elements { get; set; }
    public ShieldInstance Shield { get; set; }  // nullable, max 1
}

public class CharacterStats
{
    public int PAtk { get; set; }
    public int PDef { get; set; }
    public int MAtk { get; set; }
    public int MDef { get; set; }
    public int Spd { get; set; }
    public int Tech { get; set; }
}

// --- バースト ---
public class BurstState
{
    public bool IsBurst { get; set; }
    public int RemainingTurns { get; set; }
    public int CreatedSeq { get; set; }
}

public class ResourcesState
{
    public int BurstPoints { get; set; }
    public int BurstMaxPoints { get; set; }
}

// --- 召喚 ---
public class SummonSlotState
{
    public SummonObjectInstance Object { get; set; }  // nullable
    public SummonUnitInstance Unit { get; set; }       // nullable
}

public class SummonObjectInstance
{
    public string InstanceId { get; set; }
    public string CardId { get; set; }
    public string OwnerTeamId { get; set; }
    public string OwnerCharacterId { get; set; }
    public int DurationTurnsRemaining { get; set; }
    public int CreatedSeq { get; set; }
    public CharacterStats Stats { get; set; }
    public bool IsBurstVariant { get; set; }
    public string BaseVariantCardId { get; set; }
    public int EffectiveSummonCost { get; set; }
}

public class SummonUnitInstance
{
    public string InstanceId { get; set; }
    public string CardId { get; set; }
    public string OwnerTeamId { get; set; }
    public string OwnerCharacterId { get; set; }
    public int DurationTurnsRemaining { get; set; }
    public int Hp { get; set; }
    public int MaxHp { get; set; }
    public CharacterStats Stats { get; set; }
    public List<string> Passives { get; set; }
    public List<string> ActionSkills { get; set; }
    public string ConfiguredActionSkillId { get; set; }
    public bool HasActedThisTurn { get; set; }
    public int CreatedSeq { get; set; }
    public bool IsBurstVariant { get; set; }
    public string BaseVariantCardId { get; set; }
}

// --- カード定義 ---
public abstract class CardDefinition
{
    public string Id { get; set; }
    public string Name { get; set; }
    public CardType Type { get; set; }
    public string Color { get; set; }
    public int Cost { get; set; }
    public int Priority { get; set; }
    public string RulesText { get; set; }
    public List<string> Tags { get; set; }
    public CardTrigger Trigger { get; set; }  // nullable
    public Dictionary<string, int> ElementCost { get; set; }
    public Dictionary<string, int> ElementGain { get; set; }
    public List<EffectSpec> SetEffects { get; set; }
}

public class SkillCardDefinition : CardDefinition
{
    public EffectSpec Effect { get; set; }
}

public class SummonObjectCardDefinition : CardDefinition
{
    public int DurationTurns { get; set; }
    public ScalingSpec ObjectScaling { get; set; }
    public FlatStatBonus ObjectFlatStats { get; set; }
    public List<EffectSpec> OnEnterEffects { get; set; }
    public string BurstVariantCardId { get; set; }
}

public class SummonUnitCardDefinition : CardDefinition
{
    public int DurationTurns { get; set; }
    public ScalingSpec UnitScaling { get; set; }
    public FlatStatBonus UnitFlatStats { get; set; }
    public List<EffectSpec> OnEnterEffects { get; set; }
    public string BurstVariantCardId { get; set; }
}

public class SupportCardDefinition : CardDefinition
{
    public int BurstCostPoints { get; set; }
    public int CooldownTurns { get; set; }
    public EffectSpec Effect { get; set; }
}

public class ItemCardDefinition : CardDefinition
{
    public int Count { get; set; }
    public List<CardTrigger> Triggers { get; set; }
    public EffectSpec Effect { get; set; }
}

public class UniqueCardDefinition : CardDefinition
{
    public int CooldownTurns { get; set; }
    public EffectSpec Effect { get; set; }
    public string BurstVariantCardId { get; set; }
}

public enum CardType
{
    Skill,
    SummonObject,
    SummonUnit,
    Item,
    Support,
    Unique
}

// --- 状態異常 ---
public class StatusEffect
{
    public string Id { get; set; }
    public string SourceId { get; set; }
    public string TargetId { get; set; }
    public int Duration { get; set; }
    public int CreatedSeq { get; set; }
    public StatusEffectType EffectType { get; set; }
    public StackingRule StackingRule { get; set; }
    public List<string> Tags { get; set; }
}

public class StatusEffectDefinition
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Category { get; set; }  // buff / debuff
    public StackingRule StackingRule { get; set; }
    public int DefaultDuration { get; set; }
    public StatusEffectType Effect { get; set; }
    public List<string> Triggers { get; set; }
    public float BaseHitRate { get; set; }
    public List<string> Tags { get; set; }
}

public class StackingRule
{
    public StackingMode Mode { get; set; }  // Replace / Extend / Stack
    public int? MaxStacks { get; set; }
}

public enum StackingMode { Replace, Extend, Stack }

// --- 行動 ---
public class PlannedAction
{
    public string Id { get; set; }
    public string TeamId { get; set; }
    public ActorType ActorType { get; set; }
    public string ActorId { get; set; }
    public ActionSource Source { get; set; }
    public string CardId { get; set; }
    public TargetSpec Targets { get; set; }
    public int Sequence { get; set; }
    public long CreatedAtMs { get; set; }
    public int CardPriority { get; set; }
    public int WillConsumeApOnExecute { get; set; }
}

public class ActionResult
{
    public string ActionId { get; set; }
    public ActionOutcome Outcome { get; set; }
    public FizzleReason? FizzleReason { get; set; }
    public ActivationMode ActivationMode { get; set; }
    public int ApBefore { get; set; }
    public int ApCostApplied { get; set; }
    public int ApAfter { get; set; }
}

public enum ActorType { Character, Summon }
public enum ActionSource { Hand, Support, Unique, SummonConfigured }
public enum ActionOutcome { Executed, Fizzled }
public enum ActivationMode { Normal, Interrupt }
public enum FizzleReason { InsufficientAP, InvalidTarget, InsufficientElement, CooldownActive }
public enum TieBreakLevel { Priority, Spd, SubmitTime, Initiative, CreatedSeq }

// --- 補助型 ---
public class ActionSortKey
{
    public int Priority { get; set; }
    public int Spd { get; set; }
    public long DecisionSubmittedAtMs { get; set; }
    public string InitiativeTeamId { get; set; }
    public int CreatedSeq { get; set; }
}

public class CardTrigger
{
    public string Name { get; set; }
    public string Event { get; set; }
    public string Condition { get; set; }
}

public class TargetSpec
{
    public TargetMode Mode { get; set; }
    public List<string> TargetIds { get; set; }
}

public enum TargetMode
{
    Self, AllyCharacter, EnemyCharacter,
    AllySummonUnit, EnemySummonUnit,
    AllySummonObject, EnemySummonObject,
    AllAllies, AllEnemies, Random,
    AllyDowned  // 蘇生専用: ダウン済み味方キャラを対象
}

public class ScalingSpec
{
    public float PAtk { get; set; }
    public float PDef { get; set; }
    public float MAtk { get; set; }
    public float MDef { get; set; }
    public float Hp { get; set; }
    public float Spd { get; set; }
    public float Tech { get; set; }
}

public class FlatStatBonus
{
    public int PAtk { get; set; }
    public int PDef { get; set; }
    public int MAtk { get; set; }
    public int MDef { get; set; }
    public int Hp { get; set; }
    public int Spd { get; set; }
    public int Tech { get; set; }
}

public class StatModifier
{
    public string SourceId { get; set; }
    public ModifierLayer Layer { get; set; }
    public float Value { get; set; }
    public int CreatedSeq { get; set; }
}

public enum ModifierLayer { Additive, Multiplicative, Clamp, SetTo }
public enum StatType { PAtk, PDef, MAtk, MDef, Spd, Tech, MaxHp }
public enum EntityType { Character, SummonUnit, SummonObject }
public enum DamageType { Physical, Magical, True }
public enum SummonType { Object, Unit }
public enum SummonRemovalReason { Expired, Destroyed, Replaced }

public struct AppliedEffect
{
    public string Id { get; set; }
    public ModifierLayer Layer { get; set; }
    public int Delta { get; set; }
}

public class TriggeredAction
{
    public string EntityId { get; set; }
    public string CardId { get; set; }
    public PlannedAction Action { get; set; }
}

// --- その他 ---
public class ElementCounters
{
    public int Red { get; set; }
    public int Blue { get; set; }
    public int Green { get; set; }
    public int Orange { get; set; }
    public int Yellow { get; set; }
    public int Purple { get; set; }
}

public class ItemInstance
{
    public string InstanceId { get; set; }
    public string CardId { get; set; }
    public string OwnerTeamId { get; set; }
    public string OwnerCharacterId { get; set; }
    public int CountRemaining { get; set; }
    public int CreatedSeq { get; set; }
    public List<CardTrigger> Triggers { get; set; }
}

public class FieldEffectInstance
{
    public string InstanceId { get; set; }
    public string OwnerTeamId { get; set; }
    public string SourceTeamId { get; set; }
    public string FieldEffectDefId { get; set; }
    public int CreatedSeq { get; set; }
    public int RemainingTurns { get; set; }  // -1 for infinite
    public List<StatModifier> Modifiers { get; set; }
    public List<CardTrigger> Triggers { get; set; }
    public List<string> Tags { get; set; }
}

public class FieldEffectDefinition
{
    public string Id { get; set; }
    public string Name { get; set; }
    public int DefaultDurationTurns { get; set; }
    public string Scope { get; set; }  // "ownTeam" / "enemyTeam" / "bothTeams"
    public List<StatModifier> Modifiers { get; set; }
    public List<CardTrigger> Triggers { get; set; }
    public List<string> Tags { get; set; }
    public string RulesText { get; set; }
}

public class ShieldInstance
{
    public string SourceId { get; set; }
    public string SourceCharacterId { get; set; }
    public int RemainingAmount { get; set; }
    public int RemainingTurns { get; set; }
    public int CreatedSeq { get; set; }
}

public class SupportSlotState
{
    public string CardId { get; set; }
    public int CooldownRemainingTurns { get; set; }
    public int CreatedSeq { get; set; }
}

public class UniqueSlotState
{
    public string EquippedUniqueCardId { get; set; }
    public int CooldownRemainingTurns { get; set; }
    public int CreatedSeq { get; set; }
    public string EffectiveUniqueCardId { get; set; }  // バースト中の差し替え結果
}

public class PreparationDecisionState
{
    public long? SubmittedAtMs { get; set; }
    public int DecisionSpeedBonusPoints { get; set; }
}

public class RegulationDefinition
{
    public string Id { get; set; }
    public string Name { get; set; }
    public int TeamSize { get; set; }
    public int SupportSetMax { get; set; }
    public int MaxTurns { get; set; }
    public int PreparationTimeSec { get; set; }
    public int InitialHandSize { get; set; }
    public int ApStart { get; set; }
    public int BurstMax { get; set; }
    public int BurstMaxPoints { get; set; }
    public int ActionChargePoints { get; set; }
    public float DefenseCoefficient { get; set; }
    public string VictoryCondition { get; set; }
}
```

### EffectSpec 関連

#### EffectSpec / EffectType

```csharp
public enum EffectType
{
    Damage,
    Heal,
    Draw,
    StatusApply,
    Dispel,
    Summon,
    ElementGain,
    ElementConsume,
    ElementConvert,
    BurstGain,
    Shield,
    Resurrect,
    FieldEffectDeploy,
    ItemSet,
    APModify,
    Composite,
    Conditional
}

public class EffectSpec
{
    public EffectType EffectType { get; set; }
    public TargetSpec Target { get; set; }  // nullable — null なら親/カードの target を継承

    // 型別パラメータ（EffectType に応じていずれか1つを使用）
    public DamageEffectParams DamageParams { get; set; }
    public HealEffectParams HealParams { get; set; }
    public DrawEffectParams DrawParams { get; set; }
    public StatusApplyEffectParams StatusApplyParams { get; set; }
    public DispelEffectParams DispelParams { get; set; }
    public SummonEffectParams SummonParams { get; set; }
    public ElementGainEffectParams ElementGainParams { get; set; }
    public ElementConsumeEffectParams ElementConsumeParams { get; set; }
    public ElementConvertEffectParams ElementConvertParams { get; set; }
    public BurstGainEffectParams BurstGainParams { get; set; }
    public ShieldEffectParams ShieldParams { get; set; }
    public ResurrectEffectParams ResurrectParams { get; set; }
    public FieldEffectDeployEffectParams FieldEffectDeployParams { get; set; }
    public ItemSetEffectParams ItemSetParams { get; set; }
    public APModifyEffectParams APModifyParams { get; set; }
    public CompositeEffectParams CompositeParams { get; set; }
    public ConditionalEffectParams ConditionalParams { get; set; }
}
```

#### 型別パラメータクラス

```csharp
public class DamageEffectParams
{
    public DamageType DamageType { get; set; }
    public float CardMultiplier { get; set; }
    public int? FixedDamage { get; set; }
    public int HitCount { get; set; } = 1;
    public StatType? AtkStat { get; set; }
}

public class HealEffectParams
{
    public HealCalcMode CalcMode { get; set; }
    public float Value { get; set; }
    public StatType? Stat { get; set; }
}

public enum HealCalcMode { Fixed, MaxHpPercent, StatScale }

public class DrawEffectParams
{
    public int Count { get; set; }
}

public class StatusApplyEffectParams
{
    public string StatusId { get; set; }
    public int? Duration { get; set; }
    public float? OverrideValue { get; set; }
    public int Stacks { get; set; } = 1;
}

public class DispelEffectParams
{
    public string Category { get; set; }  // "buff" / "debuff" / "any"
    public int Count { get; set; }
    public List<string> FilterTags { get; set; }
}

public class SummonEffectParams
{
    public string SummonCardId { get; set; }
}

public class ElementGainEffectParams
{
    public string Element { get; set; }
    public int Amount { get; set; }
}

public class ElementConsumeEffectParams
{
    public string Element { get; set; }
    public int Amount { get; set; }
}

public class ElementConvertEffectParams
{
    public string From { get; set; }
    public string To { get; set; }
    public int Amount { get; set; }
}

public class BurstGainEffectParams
{
    public int Points { get; set; }
}

public class ShieldEffectParams
{
    public ShieldCalcMode ShieldType { get; set; }
    public float Value { get; set; }
    public int Duration { get; set; } = 1;
    public StatType? Stat { get; set; }
}

public enum ShieldCalcMode { Fixed, MaxHpPercent, StatScale }

public class ResurrectEffectParams
{
    public float HpPercent { get; set; }
}

public class FieldEffectDeployEffectParams
{
    public string FieldEffectId { get; set; }
    public int DurationTurns { get; set; }
}

public class ItemSetEffectParams
{
    public string ItemCardId { get; set; }
}

public class APModifyEffectParams
{
    public int Amount { get; set; }
    public APTargetStat TargetStat { get; set; }
}

public enum APTargetStat { Current, UnlockedMax }

public class CompositeEffectParams
{
    public List<EffectSpec> Children { get; set; }
}

public class ConditionalEffectParams
{
    public ConditionSpec Condition { get; set; }
    public EffectSpec ThenEffect { get; set; }
    public EffectSpec ElseEffect { get; set; }  // nullable
}
```

#### ConditionSpec

```csharp
public enum ConditionType
{
    ElementThreshold,
    HpThreshold,
    StatusCheck,
    BurstCheck,
    AllyDownCount,
    HandCount
}

public class ConditionSpec
{
    public ConditionType ConditionType { get; set; }

    public ElementThresholdParams ElementThresholdParams { get; set; }
    public HpThresholdParams HpThresholdParams { get; set; }
    public StatusCheckParams StatusCheckParams { get; set; }
    public BurstCheckParams BurstCheckParams { get; set; }
    public AllyDownCountParams AllyDownCountParams { get; set; }
    public HandCountParams HandCountParams { get; set; }
}

public class ElementThresholdParams
{
    public string Element { get; set; }
    public ComparisonOp Operator { get; set; }
    public int Threshold { get; set; }
}

public class HpThresholdParams
{
    public ConditionScope Scope { get; set; }
    public ComparisonOp Operator { get; set; }
    public float ThresholdPercent { get; set; }
}

public class StatusCheckParams
{
    public ConditionScope Scope { get; set; }
    public string StatusId { get; set; }
    public List<string> FilterTags { get; set; }
    public bool Exists { get; set; }
}

public class BurstCheckParams
{
    public ConditionScope Scope { get; set; }
    public bool IsBurst { get; set; }
}

public class AllyDownCountParams
{
    public ComparisonOp Operator { get; set; }
    public int Count { get; set; }
}

public class HandCountParams
{
    public ComparisonOp Operator { get; set; }
    public int Count { get; set; }
}

public enum ComparisonOp { Gte, Lte, Eq }
public enum ConditionScope { Self, Target }
```

#### IEffectResolver / EffectResolver

```csharp
public class EffectContext
{
    public string SourceTeamId { get; set; }
    public string SourceCharacterId { get; set; }
    public string SourceCardId { get; set; }
    public string SourceType { get; set; }  // "skill" / "support" / "unique" / etc.
    public ActivationMode ActivationMode { get; set; }
    public TargetSpec ParentTarget { get; set; }  // Composite/Conditional 内のサブ効果用
}

public interface IEffectResolver
{
    /// <summary>EffectSpec を解決する</summary>
    GameState ResolveEffect(GameState state, EffectSpec effect, EffectContext context);

    /// <summary>条件を評価する</summary>
    bool EvaluateCondition(GameState state, ConditionSpec condition, EffectContext context);
}

public class EffectResolver : IEffectResolver
{
    private readonly IDamageCalculator damageCalculator;
    private readonly IStatusEffectProcessor statusEffectProcessor;
    private readonly IDeckManager deckManager;
    private readonly ISummonManager summonManager;
    private readonly IBurstManager burstManager;
    private readonly IStatCalculator statCalculator;

    public EffectResolver(
        IDamageCalculator damageCalculator,
        IStatusEffectProcessor statusEffectProcessor,
        IDeckManager deckManager,
        ISummonManager summonManager,
        IBurstManager burstManager,
        IStatCalculator statCalculator)
    {
        this.damageCalculator = damageCalculator;
        this.statusEffectProcessor = statusEffectProcessor;
        this.deckManager = deckManager;
        this.summonManager = summonManager;
        this.burstManager = burstManager;
        this.statCalculator = statCalculator;
    }

    public GameState ResolveEffect(GameState state, EffectSpec effect, EffectContext context)
    {
        // 1. Target 解決（effect.Target ?? context.ParentTarget）
        // 2. EffectType に応じてディスパッチ:
        //    - Damage → damageCalculator.CalculateDamage(...)
        //    - StatusApply → statusEffectProcessor.ApplyStatus(...)
        //    - Draw → deckManager.DrawCards(...)
        //    - Composite → children を順次 ResolveEffect()
        //    - Conditional → EvaluateCondition() → then/else を ResolveEffect()
        //    - etc.
        return state; // placeholder
    }

    public bool EvaluateCondition(GameState state, ConditionSpec condition, EffectContext context)
    {
        // ConditionType に応じて分岐して評価
        return false; // placeholder
    }
}
```

### データフロー例: 1ターンの処理

```
GameManager.StartMatch(regulation, team1, team2)
  └→ GameState 初期化
  └→ TurnProcessor.ProcessTurn(state)
       ├→ ProcessPhase(TurnStart)
       │    ├→ StatusEffectProcessor: ターン開始トリガー
       │    ├→ SummonManager: hasActedThisTurn リセット
       │    └→ AP 解放/回復
       ├→ ProcessPhase(Draw)
       │    └→ DeckManager.DrawCards(state, teamId, aliveCount)
       ├→ ProcessPhase(Preparation)
       │    └→ 外部入力: PlannedAction[] を受け取る
       ├→ ProcessPhase(Battle)
       │    ├→ BurstManager: 決定速度ボーナス加算
       │    ├→ BurstManager.ApplyBurstMode (予約あれば)
       │    │    └→ SummonManager.TransformToBurstVariant
       │    ├→ ActionResolver.SortActions
       │    └→ ActionResolver.ResolveActions
       │         ├→ CheckFizzle → skip or continue
       │         ├→ DamageCalculator.CalculateDamage
       │         ├→ StatCalculator.RecomputeAllStats
       │         ├→ TriggerSystem.ScanTriggers
       │         ├→ BurstManager.AddBurstPoints (+30)
       │         └→ StatusEffectProcessor.ApplyStatus
       └→ ProcessPhase(TurnEnd)
            ├→ 全永続効果を createdSeq 昇順で処理
            ├→ StatusEffectProcessor.ProcessExpiration
            ├→ SummonManager.RemoveSummon (期限切れ)
            ├→ BurstManager.RemoveBurstMode (期限切れ)
            └→ 勝敗判定 / maxTurns 判定
```

---

