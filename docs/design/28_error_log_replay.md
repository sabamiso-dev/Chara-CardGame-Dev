# エラーハンドリング・ログ・リプレイ設計

ルールエンジンの堅牢性・デバッグ・リプレイ再現を支える横断的関心事を定義する。

関連: [アーキテクチャ概要](16_architecture_overview.md) / [イベント・トリガー](18_event_and_trigger.md) / [バリデーション・RNG](23_validation_rng_loader.md) / [C#クラス設計](25_csharp_classes.md)

---

## 1. エラーハンドリング戦略

### 1.1 エラー分類

ルールエンジンで発生するエラーを3層に分類し、それぞれ異なる処理方針を適用する。

| 層 | 分類 | 発生タイミング | 処理方針 | 例 |
|---|---|---|---|---|
| **L1: 致命的** | 処理続行不能 | MatchSetup / 定義ロード | 例外を throw → マッチ不成立 | 定義ファイル不正、必須定義不在 |
| **L2: 論理的** | ルール違反 | MatchSetup / バトル中 | Result 型で返す → 呼び出し元が判断 | 編成違反、不正な予約 |
| **L3: 回復可能** | 想定内の異常 | バトル中 | フォールバック処理 → ログ記録 → 続行 | 対象不在で不発、定義参照ミス |

### 1.2 例外を throw するケース（L1）

ゲームの前提が壊れており、続行しても意味がないケース。

```csharp
/// <summary>定義データが存在しない場合にスローする</summary>
public class DefinitionNotFoundException : Exception
{
    public string DefinitionType { get; }  // "Card" / "Character" / etc.
    public string DefinitionId { get; }

    public DefinitionNotFoundException(string type, string id)
        : base($"Definition not found: {type} '{id}'")
    {
        DefinitionType = type;
        DefinitionId = id;
    }
}

/// <summary>GameState が不整合な場合にスローする</summary>
public class GameStateCorruptedException : Exception
{
    public string Detail { get; }

    public GameStateCorruptedException(string detail)
        : base($"GameState corrupted: {detail}")
    {
        Detail = detail;
    }
}
```

throw するケース一覧:

| 状況 | 例外型 | 発生箇所 |
|---|---|---|
| `DefinitionRepository.GetCard(id)` で id が存在しない | `DefinitionNotFoundException` | 全モジュール |
| `DefinitionRepository.GetCharacter(id)` で id が存在しない | `DefinitionNotFoundException` | GameManager |
| RngProvider の seed が初期化されていない | `InvalidOperationException` | RngProvider |
| GameState.Teams が null or 要素数 != 2 | `GameStateCorruptedException` | TurnProcessor |
| 定義ファイルのデシリアライズ失敗 | `DefinitionLoadException` | DefinitionLoader |

### 1.3 Result 型で返すケース（L2）

```csharp
/// <summary>処理結果の汎用型</summary>
public class GameResult<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public string ErrorCode { get; }
    public string ErrorMessage { get; }

    private GameResult(bool success, T value, string code, string msg)
    {
        IsSuccess = success;
        Value = value;
        ErrorCode = code;
        ErrorMessage = msg;
    }

    public static GameResult<T> Success(T value)
        => new(true, value, null, null);

    public static GameResult<T> Failure(string code, string message)
        => new(false, default, code, message);
}
```

Result 型で返すケース一覧:

| 状況 | ErrorCode | 発生箇所 |
|---|---|---|
| 編成バリデーション失敗 | `LOADOUT_INVALID` | LoadoutValidator |
| 予約合計APが現在APを超過 | `RESERVATION_AP_EXCEEDED` | 準備フェイズ入力検証 |
| サポート発動でBurst不足 | `BURST_INSUFFICIENT` | BurstManager |
| クールダウン中のカード使用 | `COOLDOWN_ACTIVE` | ActionResolver |

### 1.4 フォールバック処理するケース（L3）

ゲームを止めず、フォールバック動作で続行するケース。全てログに記録する。

| 状況 | フォールバック | ログレベル |
|---|---|---|
| 攻撃対象がダウン済み | 不発（秘匿） | Info |
| エレメント不足 | 不発（秘匿） | Info |
| 召喚入れ替え時に旧召喚が既に消滅 | 入れ替え軽減なしで続行 | Warning |
| EffectSpec の target が null かつ親もなし | 効果スキップ | Warning |
| StatusEffect の duration が負値 | duration = 0 として処理 | Warning |
| トリガー誘発上限（10回）到達 | 以後の誘発を無視 | Warning |
| トリガー再帰深度上限（5レベル）到達 | 以後の再帰を無視 | Warning |

### 1.5 DefinitionRepository のエラー戦略

```csharp
public class DefinitionRepository : IDefinitionRepository
{
    /// <summary>
    /// 定義を取得する。存在しない場合は DefinitionNotFoundException をスローする。
    /// Phase 1 では「定義が存在しない = データ不整合」として致命的エラーとする。
    /// </summary>
    public CardDefinition GetCard(string cardId)
    {
        if (!cards.TryGetValue(cardId, out var card))
            throw new DefinitionNotFoundException("Card", cardId);
        return card;
    }

    /// <summary>
    /// 定義の存在チェック（例外を投げない）。
    /// 条件分岐で使用する場合はこちらを使う。
    /// </summary>
    public bool TryGetCard(string cardId, out CardDefinition card)
        => cards.TryGetValue(cardId, out card);
}
```

---

## 2. ログシステム

### 2.1 ログの二層構造

対戦ゲームでは「不発の秘匿」があるため、ログを2層に分離する。

| 層 | 用途 | 含む情報 | 閲覧者 |
|---|---|---|---|
| **公開ログ（PublicLog）** | 観戦・対戦中のUI表示 | 発動宣言後の行動、ダメージ、公開情報の変化 | 両プレイヤー・観戦者 |
| **内部ログ（InternalLog）** | デバッグ・リプレイ完全再現 | 不発の予約、手札変化、乱数消費、全イベント | 開発者・リプレイシステム |

### 2.2 ログレベル

```csharp
public enum LogLevel
{
    Debug,    // 開発中のみ。パイプライン中間値、ソートキー詳細
    Info,     // 通常動作。行動解決、ダメージ、状態変化
    Warning,  // フォールバック発動。対象不在、上限到達
    Error     // 致命的エラー直前。定義不在、状態不整合
}
```

### 2.3 GameLogger インターフェース

```csharp
public interface IGameLogger
{
    /// <summary>内部ログに記録（リプレイ再現用）</summary>
    void LogInternal(LogLevel level, string category, string message, object details = null);

    /// <summary>公開ログに記録（UI表示・観戦用）</summary>
    void LogPublic(string category, string message, object details = null);

    /// <summary>内部ログを取得</summary>
    IReadOnlyList<LogEntry> GetInternalLog();

    /// <summary>公開ログを取得</summary>
    IReadOnlyList<LogEntry> GetPublicLog();
}

public class LogEntry
{
    public int Turn { get; set; }
    public Phase Phase { get; set; }
    public LogLevel Level { get; set; }
    public string Category { get; set; }
    public string Message { get; set; }
    public object Details { get; set; }
    public long TimestampMs { get; set; }
}
```

### 2.4 ログカテゴリ

| カテゴリ | 公開/内部 | 例 |
|---|---|---|
| `Action` | 両方 | "キャラA が スキルX を使用" |
| `Action.Fizzle` | 内部のみ | "キャラA のスキルX が不発（AP不足）" |
| `Damage` | 両方 | "キャラA → キャラB に 136 物理ダメージ" |
| `Status` | 両方 | "キャラB に 毒（3ターン）を付与" |
| `Summon` | 両方 | "キャラA が 召喚ユニットX を召喚" |
| `Burst` | 両方 | "チームA がバースト状態を発動" |
| `Trigger` | 両方 | "アイテムX が自動発動" |
| `Trigger.Interrupt` | 両方 | "スキルY が割り込み発動" |
| `Draw` | 内部のみ | "チームA がカードX, Y, Z をドロー" |
| `RNG` | 内部のみ | "クリティカル判定: step=42, roll=0.18, threshold=0.26 → hit" |
| `StatRecompute` | 内部のみ | "キャラA.pAtk: base=300 +20(buff) ×1.15(burst) = 368" |
| `Validation` | 内部のみ | "編成バリデーション: エラー0件" |
| `System` | 内部のみ | "TriggerDepthLimitReached: entity=item_001, depth=5" |

### 2.5 イベントログとの関係

- **GameEvent**（EventQueue に積まれるもの）は **処理駆動** のためのデータ構造
- **LogEntry**（GameLogger に記録されるもの）は **可読性重視** の記録
- 1つの GameEvent が 0〜複数の LogEntry を生成する
- GameEvent は全て InternalLog にも記録する（リプレイ用）

```
GameEvent 発行
  │
  ├→ EventQueue に Enqueue（処理駆動）
  │
  ├→ GameLogger.LogInternal（内部ログ）
  │
  └→ 公開可能なイベントの場合 → GameLogger.LogPublic（公開ログ）
```

---

## 3. リプレイシステム

### 3.1 リプレイの設計方針

- **入力再現方式**: MatchConfig + 各ターンの PlannedAction を保存し、再実行する
- RNG は seed 固定のため、同一入力 → 同一出力が保証される
- イベントログは「検証用」であり、リプレイの駆動には使わない

### 3.2 リプレイデータ構造

```csharp
/// <summary>リプレイ再生に必要な最小データ</summary>
public class ReplayData
{
    /// <summary>リプレイフォーマットのバージョン</summary>
    public int Version { get; set; } = 1;

    /// <summary>マッチ設定（レギュレーション、チーム編成、RNG seed）</summary>
    public MatchConfig Config { get; set; }

    /// <summary>各ターンの入力データ</summary>
    public List<TurnInput> TurnInputs { get; set; } = new();

    /// <summary>最終結果（検証用）</summary>
    public MatchResult ExpectedResult { get; set; }
}

/// <summary>1ターン分の入力</summary>
public class TurnInput
{
    public int Turn { get; set; }

    /// <summary>チーム1の準備フェイズ入力</summary>
    public PreparationInput Team1Input { get; set; }

    /// <summary>チーム2の準備フェイズ入力</summary>
    public PreparationInput Team2Input { get; set; }
}

/// <summary>準備フェイズの入力データ</summary>
public class PreparationInput
{
    /// <summary>行動予約リスト</summary>
    public List<PlannedAction> Actions { get; set; } = new();

    /// <summary>確定ボタンの押下時刻（ms）</summary>
    public long? SubmittedAtMs { get; set; }

    /// <summary>バースト状態の予約（対象キャラID。null=予約なし）</summary>
    public string BurstTargetCharacterId { get; set; }

    /// <summary>サポート発動予約（カードID。null=発動なし）</summary>
    public string SupportActivateCardId { get; set; }

    /// <summary>1vs1方式B: AP供給（供給元キャラID、供給量）</summary>
    public ApSupplyInput ApSupply { get; set; }
}

public class ApSupplyInput
{
    public string SupportCharacterId { get; set; }
    public int Amount { get; set; }
}
```

### 3.3 リプレイ記録フロー

```
マッチ開始時:
  1. ReplayData を生成
  2. Config = 現在の MatchConfig をディープコピー

各ターンの準備フェイズ完了時:
  3. TurnInput を生成
  4. 両チームの PreparationInput を記録
  5. ReplayData.TurnInputs に追加

マッチ終了時:
  6. ExpectedResult = MatchResult をコピー
  7. ReplayData をシリアライズして保存
```

### 3.4 リプレイ再生フロー

```
ReplayMatch(replayData):
  1. GameManager.InitializeMatch(replayData.Config) で GameState を構築
  2. 各ターン:
     a. TurnStart, Draw フェイズを通常処理
     b. Preparation フェイズ:
        → replayData.TurnInputs[turn] から PlannedAction を注入
        → SubmittedAtMs も復元
     c. Battle, TurnEnd フェイズを通常処理
  3. マッチ終了後:
     a. 実際の MatchResult と ExpectedResult を比較
     b. 不一致 → ReplayMismatchError（再現性バグ）
```

### 3.5 リプレイ検証ポイント

再現性を保証するために、以下のチェックポイントを設ける。

```csharp
public class ReplayCheckpoint
{
    public int Turn { get; set; }
    public Phase Phase { get; set; }
    public int RngStep { get; set; }
    public string GameStateHash { get; set; }  // GameState の軽量ハッシュ
}
```

- 各ターン終了時に ReplayCheckpoint を記録
- リプレイ再生時に同じチェックポイントで比較
- `GameStateHash` は以下のフィールドから算出:
  - 両チームの各キャラ HP / AP / IsDown
  - Burst Points
  - デッキ枚数 / 手札枚数 / 墓地枚数
  - RNG の CurrentStep

```csharp
public static string ComputeGameStateHash(GameState state)
{
    var sb = new StringBuilder();
    sb.Append($"T{state.Turn}|");
    foreach (var team in state.Teams)
    {
        sb.Append($"BP{team.Resources.BurstPoints}|");
        sb.Append($"D{team.Deck.Count}H{team.Hand.Count}G{team.Graveyard.Count}|");
        foreach (var ch in team.Characters)
        {
            sb.Append($"{ch.Hp}/{ch.MaxHp}/{ch.Ap}/{(ch.IsDown ? 1 : 0)}|");
        }
    }
    sb.Append($"RNG{state.Rng.CurrentStep}");
    // SHA256 等でハッシュ化
    return ComputeSha256(sb.ToString());
}
```

---

## 4. GameState シリアライズ

### 4.1 用途

| 用途 | タイミング | 形式 |
|---|---|---|
| **リプレイデータ保存** | マッチ終了時 | JSON |
| **デバッグ用スナップショット** | 任意（テスト時） | JSON |
| **テストのアサーション** | テスト実行時 | インメモリ比較 |

### 4.2 シリアライズ方針

- Phase 1 では **JSON** を使用（`System.Text.Json` or `Newtonsoft.Json`）
- GameState の全フィールドをシリアライズ可能にする
- ただし以下は **除外**:
  - `IEventQueue`（処理中のキュー。リプレイには不要）
  - `IRngProvider`（seed + currentStep のみ保存）
  - 循環参照を避けるため、エンティティ間の参照は ID で表現

### 4.3 RNG 状態のシリアライズ

```csharp
/// <summary>RNG のシリアライズ用スナップショット</summary>
public class RngSnapshot
{
    public long Seed { get; set; }
    public int CurrentStep { get; set; }
}
```

- リプレイデータに `MatchConfig.RngSeed` を含めれば、RNG は再構築可能
- デバッグ用スナップショットでは `RngSnapshot` を GameState と共に保存

### 4.4 MatchResult のシリアライズ

```csharp
public class SerializableMatchResult
{
    public string WinnerId { get; set; }
    public string LoserId { get; set; }
    public string Reason { get; set; }
    public int FinalTurn { get; set; }
    public int TotalEvents { get; set; }
    public List<ReplayCheckpoint> Checkpoints { get; set; }
    public RngSnapshot FinalRngState { get; set; }
}
```

---

## 5. 診断機能（Diagnostics）

### 5.1 ゲーム状態の整合性チェック

デバッグ・テスト用に、GameState の内部整合性を検証する機能を提供する。
本番実行時はパフォーマンスのためオフにできる。

```csharp
public interface IGameDiagnostics
{
    /// <summary>GameState の整合性を検証する</summary>
    DiagnosticResult ValidateState(GameState state);

    /// <summary>診断を有効/無効にする</summary>
    bool IsEnabled { get; set; }
}

public class DiagnosticResult
{
    public bool IsConsistent => Warnings.Count == 0 && Errors.Count == 0;
    public List<string> Warnings { get; set; } = new();
    public List<string> Errors { get; set; } = new();
}
```

### 5.2 整合性チェック項目

```
ValidateState(state):
  ── キャラ状態 ──
  1. 各キャラの HP が 0〜MaxHp の範囲内
  2. IsDown == true のキャラは HP == 0
  3. IsDown == false のキャラは HP > 0
  4. AP が 0〜UnlockedMaxAP の範囲内
  5. UnlockedMaxAP が 0〜ApCap の範囲内

  ── チーム状態 ──
  6. BurstPoints が 0〜BurstMaxPoints の範囲内
  7. Deck + Hand + Graveyard + Banished のカード総数が初期デッキ枚数と一致
     （トークン生成を除く）
  8. 同一カードIDが複数ゾーンに存在しない

  ── 召喚状態 ──
  9. SummonSlot に Object と Unit が同時に存在しない
  10. 召喚の HP が 0〜MaxHp の範囲内
  11. IsDown == true のキャラに召喚が存在しない

  ── バースト状態 ──
  12. Burst.IsBurst == true なら RemainingTurns > 0
  13. Burst.IsBurst == false なら RemainingTurns == 0

  ── 永続効果 ──
  14. 各永続効果の RemainingTurns >= 0（-1 は無限のみ許可）
  15. CreatedSeq が GlobalSeqCounter 以下

  ── RNG ──
  16. RNG の CurrentStep が単調増加（前回チェック時より大きい）
```

### 5.3 使用タイミング

| タイミング | 推奨 | 備考 |
|---|---|---|
| **テスト実行時** | 常にオン | 各フェイズ処理後に ValidateState を呼ぶ |
| **デバッグビルド** | 常にオン | ターン終了時に ValidateState |
| **リリースビルド** | オフ | パフォーマンス優先 |
| **リプレイ検証時** | オン | チェックポイントごとに ValidateState |

---

## 6. 全体アーキテクチャへの統合

### 6.1 依存関係

```
GameManager
  ├→ IGameLogger（ログ記録）
  ├→ IGameDiagnostics（整合性チェック）
  ├→ ReplayRecorder（リプレイデータ記録）
  └→ TurnProcessor
       ├→ IGameLogger（各モジュールから利用）
       └→ 各モジュール → IGameLogger.LogInternal / LogPublic
```

### 6.2 初期化フロー

```
RunMatch(config):
  1. GameState 初期化
  2. GameLogger 初期化
  3. GameDiagnostics 初期化（デバッグ時のみ有効化）
  4. ReplayRecorder 初期化（ReplayData.Config = config）
  5. ターンループ開始
     → 各フェイズ処理でログ記録 + 診断チェック
  6. マッチ終了
     → ReplayData.ExpectedResult = MatchResult
     → ReplayData をシリアライズ
     → MatchResult + InternalLog + PublicLog を返す
```

### 6.3 C# インターフェース追加一覧

```csharp
// 新規追加
public interface IGameLogger { /* セクション2.3 参照 */ }
public interface IGameDiagnostics { /* セクション5.1 参照 */ }

public class GameLogger : IGameLogger { /* 実装 */ }
public class GameDiagnostics : IGameDiagnostics { /* 実装 */ }
public class ReplayRecorder { /* セクション3.3 参照 */ }
public class ReplayPlayer { /* セクション3.4 参照 */ }

// 新規例外
public class DefinitionNotFoundException : Exception { /* セクション1.2 参照 */ }
public class GameStateCorruptedException : Exception { /* セクション1.2 参照 */ }
public class DefinitionLoadException : Exception { /* DefinitionLoader 用 */ }
public class ReplayMismatchException : Exception { /* リプレイ検証用 */ }

// 新規データ
public class ReplayData { /* セクション3.2 参照 */ }
public class TurnInput { /* セクション3.2 参照 */ }
public class PreparationInput { /* セクション3.2 参照 */ }
public class ReplayCheckpoint { /* セクション3.5 参照 */ }
public class RngSnapshot { /* セクション4.3 参照 */ }
public class LogEntry { /* セクション2.3 参照 */ }
public class DiagnosticResult { /* セクション5.1 参照 */ }
public class GameResult<T> { /* セクション1.3 参照 */ }
```
