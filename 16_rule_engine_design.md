# ルールエンジン設計書

Phase 1（ルールエンジン実装）の設計図。
既存仕様書（`06_turn_flow.md`, `07_data_model.md`, `15_damage_formula.md` 等）の処理ルールを統合し、
C# 実装の全体像を定義する。

---

## 1. アーキテクチャ概要

### 設計原則

| 原則 | 内容 |
|---|---|
| **純粋関数に近い形** | 各処理は GameState を受け取り、変更後の GameState を返す。副作用はイベント発行で表現する |
| **イベント駆動** | すべてのゲーム内変化は GameEvent として記録・処理される（`06_turn_flow.md`） |
| **決定的処理** | rngSeed ベースの乱数により、同一入力から同一結果を再現可能（リプレイ対応） |
| **テスト容易性** | UI 非依存。インターフェース経由で各モジュールを差し替え・モック可能 |

### 全体構成図

```
┌─────────────────────────────────────────────────────────┐
│                     GameManager                         │
│  （ゲーム全体の進行制御・マッチ開始〜終了）              │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│                    TurnProcessor                        │
│  （フェイズ遷移・各フェイズ処理のオーケストレーション）  │
│                                                         │
│  ┌──────────┐ ┌──────────────┐ ┌─────────────────────┐  │
│  │DeckMgr   │ │ActionResolver│ │StatusEffectProcessor│  │
│  │(ドロー/  │ │(行動ソート/  │ │(状態異常/バフの     │  │
│  │ シャッフル│ │ 不発判定/    │ │ 付与/解除/期限切れ) │  │
│  │ /ゾーン)  │ │ 順次解決)    │ │                     │  │
│  └──────────┘ └──────┬───────┘ └─────────────────────┘  │
│                      │                                   │
│       ┌──────────────┼──────────────┐                   │
│       ▼              ▼              ▼                   │
│  ┌──────────┐ ┌──────────────┐ ┌───────────┐           │
│  │DamageCal │ │StatCalculator│ │TriggerSys │           │
│  │(ダメージ │ │(5段階パイプ  │ │(トリガー  │           │
│  │ 計算)    │ │ ライン)      │ │ 判定/割込)│           │
│  └──────────┘ └──────────────┘ └───────────┘           │
│                                                         │
│  ┌──────────────┐ ┌──────────┐ ┌───────────┐           │
│  │EffectResolver│ │SummonMgr │ │BurstManager│          │
│  │(EffectSpec   │ │(生成/消滅│ │(ゲージ管理/│          │
│  │ ディスパッチ)│ │ /上書き) │ │ バースト)  │          │
│  └──────────────┘ └──────────┘ └───────────┘           │
│                                                         │
│  ┌──────────┐ ┌───────────────┐ ┌───────────┐           │
│  │ItemMgr   │ │FieldEffectMgr │ │ShieldProc │           │
│  │(セット/  │ │(展開/消滅/   │ │(付与/吸収/│           │
│  │ 自動発動)│ │ 上書き)       │ │ 期限切れ) │           │
│  └──────────┘ └───────────────┘ └───────────┘           │
│                                                         │
│  ┌───────────┐                                          │
│  │RngProvider│                                          │
│  │(決定的   │                                          │
│  │ 乱数生成)│                                          │
│  └───────────┘                                          │
└─────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────┐
│                     EventQueue                          │
│  （イベント駆動の中核。FIFO キュー）                    │
└─────────────────────────────────────────────────────────┘
```

### モジュール責務一覧

| モジュール | 責務 |
|---|---|
| **GameManager** | マッチの開始・終了制御、GameState 初期化、勝敗判定 |
| **TurnProcessor** | フェイズ遷移、各フェイズ処理の呼び出し |
| **EventQueue** | イベントの蓄積と FIFO 順次処理 |
| **StatCalculator** | ステータス修正パイプライン（5段階） |
| **DamageCalculator** | ダメージ計算（物理/魔法/真ダメージ） |
| **ActionResolver** | 行動ソート・不発判定・順次解決 |
| **EffectResolver** | EffectSpec のディスパッチ・Composite/Conditional の再帰解決 |
| **TriggerSystem** | トリガー条件判定・割り込み処理 |
| **SummonManager** | 召喚の生成/消滅/上書き/バースト変化 |
| **BurstManager** | バーストゲージ管理・バースト状態の適用/解除 |
| **DeckManager** | デッキ操作（ドロー/再構築/ゾーン移動） |
| **StatusEffectProcessor** | 状態異常/バフの付与/解除/期限切れ |
| **ItemManager** | アイテムのセット/自動発動/破棄/上書き |
| **FieldEffectManager** | フィールド効果の展開/消滅/上書き |
| **ShieldProcessor** | シールドの付与/ダメージ吸収/期限切れ |
| **RngProvider** | 決定的乱数生成（seedベース） |

### 依存関係

```
GameManager → TurnProcessor → ActionResolver → EffectResolver → DamageCalculator
                                                               → StatusEffectProcessor
                                                               → DeckManager
                                                               → SummonManager
                                                               → BurstManager
                                                               → ShieldProcessor
                                                               → ItemManager
                                                               → FieldEffectManager
                                             → StatCalculator
                                             → TriggerSystem → ItemManager（アイテム自動発動）
                            → DeckManager
                            → StatusEffectProcessor
                            → SummonManager → StatCalculator
                            → BurstManager → SummonManager
                                           → StatCalculator
                            → ShieldProcessor
                            → ItemManager
                            → FieldEffectManager → StatCalculator
全モジュール → EventQueue（イベント発行）
全モジュール → RngProvider（乱数が必要な場合）
```

---

## 2. ゲームループとステートマシン

### GameState の方針

GameState は **ミュータブル（in-place 変更）** 方式を採用する。
ただし、以下の制約を設ける。

- 各処理メソッドは GameState を引数に受け取り、変更後の同一 GameState を返す
- 変更前の状態が必要な場合（リプレイ差分、ロールバック等）は、呼び出し元で明示的にスナップショットを取る
- テスト時はディープコピーで状態の前後比較が可能

### フェイズ遷移図

```
MatchSetup → TurnStart → Draw → Preparation → Battle → TurnEnd
                 ▲                                         │
                 │                                         │
                 └─── (次ターンへ) ◄────────────────────────┘
                                                           │
                                              (勝敗確定) → GameEnd
```

### 各フェイズの処理責務

#### MatchSetup（対戦開始時）

```
1. RegulationDefinition を読み込み
2. 両チームの編成（キャラ/デッキ/ユニーク/サポート）をロード
3. セット効果（構築時パッシブ）を適用
   - キャラデッキ内カードの setEffects → キャラへ適用
   - サポートカードの setEffects → チームへ適用
4. チームデッキを作成（3キャラのキャラデッキを合体）
5. デッキをシャッフル（rngSeed 使用）
6. 初期手札ドロー（5枚）
7. セット効果による追加ドロー（初期手札5枚の後に処理）
8. initiativeTeamId を rngSeed から決定（以降固定）
9. 初期 Burst = 0（セット効果で初期値がある場合はそれを適用）
10. 初期エレメント = 各色 0（同上）
11. 初期 AP = apStart（標準: 3）
    - unlockedMaxAP = apStart
```

**イベント**: `MatchStarted`

#### TurnStart（ターン開始フェイズ）

```
1. turn カウンタをインクリメント
2. 「ターン開始時」トリガー解決（アイテム/パッシブ/状態/フィールド効果）
3. 召喚ユニットの hasActedThisTurn をリセット（false）
4. AP 解放: unlockedMaxAP += 1（ただし apCap まで）
5. AP 回復: ap += apRegen（ただし unlockedMaxAP まで）
   ※ 1ターン目開始時は AP 解放/回復を行わない（初期値のまま）
```

**イベント**: `TurnStarted`, `PhaseChanged(TurnStart)`

#### Draw（ドローフェイズ）

```
1. ドロー枚数 = ダウンしていないキャラの数（最大3枚）
2. デッキ枚数 ≥ ドロー枚数 → 通常ドロー
3. デッキ枚数 < ドロー枚数 →
   a. 残りデッキから引けるだけ引く
   b. graveyard のカードをすべてデッキに戻す
   c. Fisher-Yates シャッフル（rngSeed）
   d. DeckReshuffledFromTrash イベント発行
   e. 不足分をドロー
4. 「ドロー時」トリガー解決
```

**イベント**: `PhaseChanged(Draw)`, `CardDrawn`, `DeckReshuffledFromTrash`（必要時）

#### Preparation（準備フェイズ / 30秒）

```
1. 両プレイヤーに行動予約の入力を受付
2. 各予約の妥当性チェック（推奨）:
   - AP 合計がキャラの currentAP 以下か
   - クールダウン中でないか
   - 対象が存在するか
3. バースト状態の予約も受付（Burst 1本消費予約）
4. decisionSubmittedAtMs を記録
5. 制限時間（30秒）終了時、未確定の場合はパス扱い
6. 決定速度ボーナスの算出:
   - 10秒以内: +70pt
   - 20秒以内: +50pt
   - 30秒以内: +30pt
   - 時間切れ: +0pt
```

**イベント**: `PhaseChanged(Preparation)`, `ActionReserved`

#### Battle（バトルフェイズ）

```
1. 決定速度ボーナスを burstPoints に加算
2. バースト状態の予約があれば適用（行動解決の前に）
   - ステータス上昇（倍率式）
   - AP +2 回復
   - unlockedMaxAP +1 解放
   - 既存召喚のバースト版への変化
   - ユニークカードの強化版への変化
3. 全予約行動をソート（タイブレークチェーン）
4. 各行動を順次解決:
   a. 不発チェック（AP不足/エレメント不足/対象不在/CD中）
   b. 不発なら ActionResolved(fizzled) を発行しスキップ
   c. コスト支払い（AP 消費、エレメント消費）
   d. 効果解決
   e. カードをトラッシュへ（hand → graveyard）
   f. トリガー判定（予約済みトリガー付きカードの条件チェック）
      → 条件成立なら割り込みで早期発動
   g. Burst 増加（行動チャージ: +30pt）
   h. 自動発動チェック（アイテム条件）
   i. ステータス再計算（バフ/デバフ変動時）
   j. ActionResolved(executed) イベント発行
5. 全行動解決後に勝敗判定
```

**イベント**: `PhaseChanged(Battle)`, `ActionResolved`, `CardPlayed`, `DamageDealt`, `Healed`, `StatusApplied`, 他多数

#### TurnEnd（ターン終了フェイズ）

```
1. 現在存在する全永続効果を収集:
   - 召喚オブジェクト/ユニット
   - バースト状態
   - フィールド効果
   - 状態異常/バフ
   - シールド
   - アイテム（持続ターン減少のみ）
   - ユニークCD / サポートCD
2. createdSeq 昇順にソート
3. 各効果について:
   a. 持続カウント（remainingTurns 等）を 1 減少
   b. 0 になった場合は消滅/期限切れ処理:
      - 召喚: SummonObjectExpired / SummonUnitExpired
      - バースト状態: 解除（AP上限/召喚/ユニークを元に戻す）
      - フィールド効果: FieldEffectExpired
      - シールド: ShieldExpired
      - 状態異常: StatusExpired
      - CD: cooldownRemainingTurns-- のみ（0で使用可能に戻る）
   c. 消滅で誘発トリガーが発生した場合、イベントキューへ積む
4. ターン終了フェイズ中に新たに生成された永続効果は
   そのターンの処理リストには含めない（次ターンから）
5. 手札上限チェックなし
6. maxTurns に到達した場合 → GameEnd（タイブレーク判定）
```

**イベント**: `PhaseChanged(TurnEnd)`, `TurnEnded`, 各種 Expired イベント

#### 勝敗判定（VictoryCheck）

勝敗判定は **バトルフェイズの全行動解決後** と **ターン終了時（maxTurns到達）** に行う。

```
CheckVictory(state, regulation):
  1. 各チームの状態を判定:
     a. victoryCondition == "allDown":
        - チームの全キャラが isDown == true → そのチームは敗北
     b. victoryCondition == "leaderDown":
        - チームのリーダーが isDown == true → そのチームは敗北
  2. 判定結果:
     a. 片方だけ敗北条件を満たす → 他方の勝利
     b. 両方同時に敗北条件を満たす → タイブレーク判定
     c. どちらも満たさない + maxTurns 到達 → タイブレーク判定
     d. どちらも満たさない + maxTurns 未到達 → 続行
  3. タイブレーク判定（hpRatioSum）:
     a. 各チームの hpRatioSum を算出:
        hpRatioSum = Σ (isDown でないキャラの currentHp / maxHp)
     b. hpRatioSum が高い方が勝利
     c. 完全同値 → initiativeTeamId 側が勝利
  4. GameEnd イベント発行（勝者/敗者/判定理由）
```

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

## 4. 行動解決パイプライン（ActionResolver）

### ソートキー（タイブレークチェーン）

上位の基準で差がつけば、以降は参照しない。

| 優先度 | キー | 順序 | 説明 |
|---|---|---|---|
| 1 | `priority` | **降順** | カード優先度が高い行動を先に解決 |
| 2 | `spd` | **降順** | 行動主体（キャラ/召喚ユニット）の速度 |
| 3 | `decisionSubmittedAtMs` | **昇順** | 準備フェイズの確定ボタンが早いチームが先 |
| 4 | `initiativeTeamId` | — | 上記まで同値ならイニシアチブ側が先 |
| 5 | `createdSeq` | **昇順** | 同一チーム内で全て同値ならエンティティ生成連番 |

- 時間切れ（未確定）の行動は、同 priority/spd 帯の **末尾** に配置
- 召喚ユニットの行動もキャラと同じソートに参加（0コストスキルの priority と召喚ユニットの spd を使用）
- 試合中に乱数を使った最終タイブレークは行わない

### 不発チェック（FizzleCheck）

行動の **発動宣言前** に以下を順にチェックする。

| チェック | 不発理由（FizzleReason） | 条件 |
|---|---|---|
| AP 不足 | `insufficientAP` | `currentAP < cost`（軽減後） |
| エレメント不足 | `insufficientElement` | 必要エレメント数を満たさない |
| 対象不在 | `invalidTarget` | 対象キャラがダウン済み or 対象不在 |
| クールダウン中 | `cooldownActive` | ユニーク/サポートの CD > 0 |

- 不発カードは **手札に残る**（相手に公開されない）
- AP・エレメントは消費しない
- コスト軽減の下限: `max(1, originalCost - reduction)`（0コストカードは除く）

### 解決フロー

```
1. 全予約行動を収集
2. ソートキーで並び替え
3. バースト状態の適用（バトルフェイズ開始時、行動解決の前）
4. 各行動を順次解決:
   ┌─────────────────────────────────────────┐
   │ 4a. 不発チェック                        │
   │     → 不発なら ActionResolved(fizzled)   │
   │       を発行してスキップ                 │
   │ 4b. コスト支払い                        │
   │     - AP 消費                            │
   │     - エレメント消費（elementCost）       │
   │ 4c. エレメント獲得（有色カード→その色+1）│
   │ 4d. 効果解決（EffectResolver）            │
   │     - EffectSpec を EffectResolver へ    │
   │       渡し、各モジュールへ委譲           │
   │ 4e. カード処理                          │
   │     - hand → graveyard                   │
   │ 4f. ActionResolved(executed) 発行        │
   │ 4g. トリガー割り込み判定                │
   │     → TriggerSystem で予約済みトリガー  │
   │       付きカードの条件チェック            │
   │     → 成立なら割り込みで即座に解決      │
   │ 4h. Burst 増加（行動チャージ: +30pt）    │
   │ 4i. 自動発動チェック（アイテム条件）     │
   │ 4j. ステータス再計算（変動があれば）     │
   └─────────────────────────────────────────┘
5. 全行動解決後に勝敗判定
```

### ActionResolved イベントの生成

すべての行動（不発含む）について `ActionResolved` イベントを生成する。

記録内容:
- `actionId`, `actorType`, `actorId`, `cardId`
- `resolvedAs`: `executed | fizzled`
- `fizzleReason`: 不発理由（不発時のみ）
- `activationMode`: `normal | interrupt`
- `apBefore`, `apCostApplied`, `apAfter`
- `sortKey`: 5項目のソートキー
- `tieBreakLevelUsed`: どこで順序が確定したか

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

## 11.5. GameManager（マッチライフサイクル）

### 責務

- マッチの開始から終了までの全体制御
- GameState の初期化（MatchSetup）
- ターンループの駆動（TurnProcessor へ委譲）
- 勝敗判定と GameEnd 処理

### マッチライフサイクル

```
RunMatch(regulation, team1Config, team2Config, rngSeed):
  1. MatchSetup: GameState を初期化
  2. ターンループ:
     while (state.CurrentPhase != GameEnd):
       state = turnProcessor.ProcessTurn(state)
       state = CheckVictoryAndAdvance(state)
  3. return MatchResult（勝者/敗者/イベントログ）
```

### MatchSetup 詳細処理フロー

```
InitializeMatch(regulation, team1Config, team2Config, rngSeed):
  ── 1. 基盤初期化 ──
  1a. GameState を生成（turn=0, phase=MatchSetup）
  1b. RngProvider を rngSeed で初期化
  1c. globalSeqCounter = 0

  ── 2. チーム構築 ──
  各チーム (team1, team2) について:
    2a. TeamState を生成（id, deck=[], hand=[], graveyard=[], banished=[]）
    2b. ResourcesState を初期化:
        - burstPoints = 0
        - burstMaxPoints = regulation.burstMaxPoints（標準: 500）
    2c. 各キャラ (3体) の CharacterState を生成:
        - hp = maxHp（キャラ定義から）
        - ap = regulation.apStart（標準: 3）
        - unlockedMaxAP = regulation.apStart
        - apCap = キャラ定義の上限（標準: 10）
        - apRegen = キャラ定義の回復量（標準: 1）
        - isDown = false
        - elements = 全色 0
        - shield = null
        - summonSlot = null
    2d. ユニークカードの装備:
        - unique.equippedUniqueCardId = 選択されたユニークID
        - unique.cooldownRemainingTurns = 0
        - unique.createdSeq = state.NextSeq()
    2e. サポートカードの配置:
        - supportSet に最大3枚を配置
        - 各スロットの cooldownRemainingTurns = 0

  ── 3. セット効果の解決 ──
  3a. キャラデッキのセット効果（scope: self）:
      各キャラの各カードについて:
        card.setEffects が存在する場合
          → EffectResolver.ResolveEffect(state, setEffect, context{sourceType: "setEffect"})
          → 主に StatusApply（永続 statMod）を想定
      ※ 解決順: キャラ1 → キャラ2 → キャラ3（チーム内順序）
  3b. サポートカードのセット効果（scope: team）:
      各サポートカードについて:
        card.setEffects が存在する場合
          → EffectResolver.ResolveEffect(...)
      ※ 解決順: スロット1 → スロット2 → スロット3
  3c. セット効果で付与された LoadoutPassiveInstance を character.loadoutPassives に記録

  ── 4. デッキ構築 ──
  4a. 各キャラのキャラデッキ（10〜15枚）を結合 → チームデッキ
  4b. Fisher-Yates シャッフル（RngProvider 使用）
  4c. チームデッキを team.deck に設定

  ── 5. 初期手札 ──
  5a. DeckManager.DrawCards(state, teamId, regulation.initialHandSize)
      ※ 標準: 5枚
  5b. セット効果による追加ドロー:
      セット効果で Draw が予約されている場合、ここで追加実行
      ※ 初期手札5枚の「後」に処理（確定済みルール）

  ── 6. イニシアチブ決定 ──
  6a. state.initiativeTeamId = RngProvider で2チームからランダム選択
      ※ 以降固定（更新なし）

  ── 7. 完了 ──
  7a. MatchStarted イベント発行
  7b. state.CurrentPhase = TurnStart に遷移
  7c. return state
```

### GameEnd 処理

```
ProcessGameEnd(state, victoryResult):
  1. 勝者/敗者を確定
  2. GameEndedEvent を発行:
     - winnerId: string
     - loserId: string
     - reason: "allDown" | "leaderDown" | "timeout_hpRatio" | "timeout_initiative"
     - finalTurn: number
  3. 全イベントログを MatchResult に包んで返す
```

---

## 11.6. AP管理ルール

AP（アクションポイント）に関するルールが複数仕様書に分散しているため、ここに統合する。

### AP 関連プロパティ

| プロパティ | 初期値 | 説明 |
|---|---|---|
| `ap` | `apStart`（標準: 3） | 現在AP。行動で消費、ターン開始で回復 |
| `unlockedMaxAP` | `apStart`（標準: 3） | 解放済みの最大AP。毎ターン+1で解放 |
| `apCap` | キャラ定義（標準: 10） | AP上限。unlockedMaxAP はこれを超えない |
| `apRegen` | キャラ定義（標準: 1） | 毎ターン回復量 |

### AP ライフサイクル

```
── ターン開始フェイズ ──
（1ターン目は処理しない: 初期値のまま）
1. AP解放: unlockedMaxAP = min(unlockedMaxAP + 1, apCap)
2. AP回復: ap = min(ap + apRegen, unlockedMaxAP)

── 準備フェイズ ──
3. 予約上限: 各キャラの予約コスト合計 ≤ 現在AP
   ※ APは予約時に仮押さえしない

── バトルフェイズ ──
4. バースト状態適用時（行動解決前）:
   - ap = min(ap + 2, unlockedMaxAP)
   - unlockedMaxAP = min(unlockedMaxAP + 1, apCap)

5. 各行動の実行時:
   a. 不発チェック: currentAP < cost → 不発（AP消費なし）
   b. コスト軽減: effectiveCost = max(1, originalCost - reduction)
      ※ 0コストカードは除く（0のまま）
   c. AP消費: ap -= effectiveCost

6. APModify 効果（EffectSpec 経由）:
   - targetStat == "current": ap = clamp(ap + amount, 0, unlockedMaxAP)
   - targetStat == "unlockedMax": unlockedMaxAP = clamp(unlockedMaxAP + amount, 0, apCap)

── ターン終了フェイズ ──
7. バースト状態解除時:
   - unlockedMaxAP が非バースト最大上限を超えている場合 → 非バースト上限に戻す
   - ap > unlockedMaxAP の場合 → ap = unlockedMaxAP（超過分は切り捨て）
```

### AP 計算例

```
ターン1開始: ap=3, unlockedMaxAP=3（初期値。1ターン目は解放/回復なし）
ターン1バトル: スキル(cost=2)使用 → ap=1
ターン2開始: unlockedMaxAP=4, ap=min(1+1, 4)=2
ターン2バトル: スキル(cost=1)×2使用 → ap=0
ターン3開始: unlockedMaxAP=5, ap=min(0+1, 5)=1
ターン3バトル: バースト適用 → ap=min(1+2, 5+1)=3, unlockedMaxAP=6
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
    AllAllies, AllEnemies, Random
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
