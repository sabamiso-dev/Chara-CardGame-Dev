# 編成バリデーション・乱数管理・定義データローダー 設計

ルールエンジン設計の一部。LoadoutValidator（編成バリデーション）、RngProvider（決定的乱数）、DefinitionLoader/Repository（マスターデータ読み込み）を定義する。

関連: [ゲームループ](17_game_loop_and_phases.md) / [C#クラス設計](25_csharp_classes.md) / [実装ロードマップ](26_implementation_roadmap.md)

---

## 10.9. 編成バリデーション（LoadoutValidator）

### 責務

MatchSetup 時に、両チームの編成が `RegulationDefinition` のルールに適合しているかを検証する。
不正な編成はマッチ開始前にリジェクトする。

### バリデーションチェック一覧

```
ValidateLoadout(teamConfig, regulation):
  errors = []

  ── 1. チーム人数 ──
  teamConfig.characters.length == regulation.teamSize
  → 不一致: "チーム人数が不正（期待: {teamSize}, 実際: {actual}）"

  ── 2. キャラデッキ枚数 ──
  各キャラの deckCardIds.length が regulation.characterDeckSize の [min, max] 範囲内
  → 範囲外: "キャラ {id} のデッキ枚数が不正（{min}〜{max}枚）"

  ── 3. サポート枠数 ──
  teamConfig.supportCardIds.length <= regulation.supportSetMax
  → 超過: "サポート枚数が上限超過（最大: {max}）"

  ── 4. 同名カード制限 ──
  4a. perCharacterDeck: 各キャラデッキ内で同名カードが regulation.sameNameRule.perCharacterDeck 枚以下
  4b. perTeam: チーム全体（全キャラデッキ + サポート）で同名カードが perTeam 枚以下
  → 超過: "カード {name} が同名制限を超過"

  ── 5. 色制約 ──
  5a. mode == "maxColors":
      チーム全体で使用している有色カードの色数 <= regulation.colorRestriction.maxColors
      ※ 無色カードは色数にカウントしない
  5b. mode == "singleOnly":
      有色カードは1色のみ許可
  → 違反: "色制約違反（最大 {maxColors} 色）"

  ── 6. 無色カード制限 ──
  6a. mode == "ratio":
      各キャラデッキ内の無色カード枚数 / デッキ枚数 <= maxRatio
  6b. mode == "fixed":
      各キャラデッキ内の無色カード枚数 <= maxCount
  → 超過: "無色カード制限超過"

  ── 7. 禁止/制限カード ──
  7a. regulation.bannedCards に含まれるカードが使用されていないか
  7b. regulation.restrictedCards の枚数制限が守られているか
  → 違反: "禁止カード {id} が使用されている" / "制限カード {id} が枚数超過"

  ── 8. ユニークカードの妥当性 ──
  各キャラが選択したユニークカードが、そのキャラの候補3枚に含まれるか
  → 不正: "キャラ {id} のユニーク {uniqueId} は候補に含まれない"

  return errors（空なら合格）
```

### バリデーション結果

```csharp
public class LoadoutValidationResult
{
    public bool IsValid => Errors.Count == 0;
    public List<string> Errors { get; set; } = new();
}
```

### MatchSetup での呼び出し

MatchSetup のステップ1（基盤初期化）の直後、ステップ2（チーム構築）の前に実行する。

```
InitializeMatch(config):
  1a. GameState 生成
  1b. バリデーション:
      result1 = ValidateLoadout(config.team1, config.regulation)
      result2 = ValidateLoadout(config.team2, config.regulation)
      → いずれかが不正 → MatchSetupFailed エラーを返す
  2. チーム構築（以降は既存フロー）
```

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

## 11. 乱数管理（RngProvider）

### 決定的 RNG

- **seed ベース** の疑似乱数生成器を使用
- 同一 seed からは常に同一の乱数列が生成される
- リプレイ時に同一 seed を与えれば完全再現可能

### 用途

| 用途 | タイミング |
|---|---|
| **initiativeTeamId 決定** | MatchSetup |
| **デッキシャッフル** | MatchSetup, DeckReshuffle |
| **クリティカル判定** | DamageCalculator |
| **状態異常命中判定** | StatusEffectProcessor |

### ステップ記録

- 各乱数使用時に `rngStepBefore` / `rngStepAfter` を記録
- イベントログに含めることで、リプレイ検証時に乱数の消費を追跡可能

### 実装方針

- C# の `System.Random` は seed 対応だが、バージョン間で挙動が異なる可能性がある
- **推奨**: xoshiro256 等の明確な仕様を持つアルゴリズムを採用し、プラットフォーム非依存を保証する

---


## 17. 定義データローダー（DefinitionLoader）

### 概要

カード定義・キャラ定義・レギュレーション定義などのマスターデータを読み込む。
Phase 1 では YAML / JSON ファイルから読み込み、将来的に ScriptableObject へ移行可能な設計とする。

### 責務

```
DefinitionLoader
├── LoadCardDefinitions(): Dictionary<string, CardDefinition>
├── LoadCharacterDefinitions(): Dictionary<string, CharacterDefinition>
├── LoadStatusEffectDefinitions(): Dictionary<string, StatusEffectDefinition>
├── LoadFieldEffectDefinitions(): Dictionary<string, FieldEffectDefinition>
├── LoadActionSkillDefinitions(): Dictionary<string, ActionSkillDefinition>
├── LoadPassiveDefinitions(): Dictionary<string, PassiveDefinition>
└── LoadRegulationDefinition(id): RegulationDefinition
```

### キャラ定義

```csharp
public class CharacterDefinition
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string ColorIdentity { get; set; }
    public string Role { get; set; }
    public CharacterStats BaseStats { get; set; }
    public int MaxHp { get; set; }
    public int ApStart { get; set; }
    public int ApMaxStart { get; set; }
    public int ApCap { get; set; }
    public int ApRegen { get; set; }
    public List<PassiveConfig> Passives { get; set; }       // [3]
    public List<string> UniqueChoiceIds { get; set; }        // [3]
}

public class PassiveConfig
{
    public string PassiveId { get; set; }
    public int UnlockAtUnlockedMaxAP { get; set; }
}
```

### DefinitionRepository（実行時の定義参照）

```csharp
public interface IDefinitionRepository
{
    CardDefinition GetCard(string cardId);
    CharacterDefinition GetCharacter(string characterId);
    StatusEffectDefinition GetStatusEffect(string statusId);
    FieldEffectDefinition GetFieldEffect(string fieldEffectId);
    ActionSkillDefinition GetActionSkill(string actionSkillId);
    PassiveDefinition GetPassive(string passiveId);
    RegulationDefinition GetRegulation(string regulationId);
}

public class DefinitionRepository : IDefinitionRepository
{
    private readonly Dictionary<string, CardDefinition> cards;
    private readonly Dictionary<string, CharacterDefinition> characters;
    private readonly Dictionary<string, StatusEffectDefinition> statusEffects;
    private readonly Dictionary<string, FieldEffectDefinition> fieldEffects;
    private readonly Dictionary<string, ActionSkillDefinition> actionSkills;
    private readonly Dictionary<string, PassiveDefinition> passives;

    public DefinitionRepository(DefinitionLoader loader)
    {
        cards = loader.LoadCardDefinitions();
        characters = loader.LoadCharacterDefinitions();
        statusEffects = loader.LoadStatusEffectDefinitions();
        fieldEffects = loader.LoadFieldEffectDefinitions();
        actionSkills = loader.LoadActionSkillDefinitions();
        passives = loader.LoadPassiveDefinitions();
    }

    public CardDefinition GetCard(string cardId) => cards[cardId];
    public CharacterDefinition GetCharacter(string characterId) => characters[characterId];
    // ... 他の Get メソッドも同様
}
```

### 全モジュールへの注入

DefinitionRepository は **全モジュールが参照する** ため、DI コンテナまたはコンストラクタ注入で配布する。

```
GameManager
  └→ new DefinitionRepository(loader)
       ├→ TurnProcessor(repo, ...)
       ├→ ActionResolver(repo, ...)
       ├→ EffectResolver(repo, ...)
       ├→ SummonManager(repo, ...)
       ├→ StatusEffectProcessor(repo, ...)
       ├→ ItemManager(repo, ...)
       └→ LoadoutValidator(repo, ...)
```

### Phase 1 実装方針

- YAML / JSON をデシリアライズして `Dictionary<string, T>` に格納
- Unity 統合時は `ScriptableObject` + `Addressables` に差し替え可能
- テスト時はインメモリで直接 Dictionary を構築（ファイルI/O不要）

---

