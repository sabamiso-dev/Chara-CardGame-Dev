# ゲームループとフェイズ設計

ルールエンジン設計の一部。ゲーム全体の進行制御・フェイズ遷移・GameManager・AP管理を定義する。

関連: [アーキテクチャ概要](16_architecture_overview.md) / [行動解決](19_action_resolver.md) / [C#クラス設計](25_csharp_classes.md)

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

