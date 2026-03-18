# 状態遷移と連鎖解決フロー

既存設計書で暗黙のままになっている状態遷移・連鎖処理の順序を厳密に定義する。
各フローは「この順序で処理する」という実装上の契約であり、逸脱する場合は本ドキュメントを先に更新する。

関連: [ゲームループ](17_game_loop_and_phases.md) / [イベント・トリガー](18_event_and_trigger.md) / [召喚・バースト](21_summon_and_burst.md) / [サブシステム](22_subsystems.md) / [ステータス・ダメージ・効果](20_stat_damage_effect.md)

---

## 1. イベントキューの基本原則

すべての連鎖処理は **EventQueue（FIFO）** を通じて処理される。
以下の原則を全フローに共通して適用する。

### 1.1 即時発火 vs キュー

| 分類 | 処理方式 | 例 |
|---|---|---|
| **状態変更** | 即時（in-place） | HP減少、isDown=true、shield=null |
| **イベント発行** | EventQueue に Enqueue | DamageDealt、CharacterDowned、StatusExpired |
| **トリガースキャン** | イベント処理後に実行 | アイテム自動発動、パッシブ誘発 |
| **勝敗判定** | フェイズ末尾でのみ実行 | バトルフェイズ全行動解決後、ターン終了時 |

### 1.2 原子性（Atomicity）

- **1つの行動解決**（PlannedAction の実行）は **原子的単位** として扱う
- 行動解決中に発生したイベントは即座に Enqueue されるが、**トリガースキャンは行動解決の完了後** に実行する
- 割り込みトリガーは例外：行動解決後のトリガースキャンで発火し、**次の行動解決の前に** 即座に解決する

### 1.3 連鎖の深さ制限

- 同一フェイズ内で同一エンティティからの誘発回数: **上限10回**（既存ルール）
- トリガースキャンの再帰深度: **上限5レベル**（トリガーがトリガーを呼ぶ連鎖）
- 上限到達時はそれ以上の誘発を無視し、ログに `TriggerDepthLimitReached` を記録

---

## 2. ダメージ適用 → ダウン判定 → 連鎖処理

### 2.1 単体ダメージの完全フロー

```
ApplyDamageToTarget(state, targetId, attackContext):
  ── Step 1: ダメージ算出 ──
  1a. DamageCalculator.CalculateDamage(attackContext) → DamageResult

  ── Step 2: シールド吸収（真ダメージ以外） ──
  2a. DamageType == True → シールドを貫通（Step 2 をスキップ）
  2b. ShieldProcessor.AbsorbDamage(state, targetId, finalDamage)
      → absorbed, throughDamage, shieldDepleted
  2c. shieldDepleted == true:
      → target.shield = null（即時）
      → ShieldExpired(Depleted) を Enqueue

  ── Step 3: HP 減少 ──
  3a. target.hp = max(0, target.hp - throughDamage)
  3b. DamageDealt イベントを Enqueue

  ── Step 4: ダウン判定 ──
  4a. target.hp == 0 && !target.isDown:
      → target.isDown = true（即時）
      → CharacterDowned イベントを Enqueue
      → Step 5（召喚消滅）へ進む
  4b. target.hp > 0:
      → ダウンなし。フロー終了

  ── Step 5: キャラダウン時の連鎖 ──
  5a. 紐づく召喚枠の召喚（オブジェクト/ユニット）を消滅:
      → SummonSlot.Object != null:
          Object を banished へ移動
          SummonObjectDestroyed を Enqueue
      → SummonSlot.Unit != null:
          Unit を banished へ移動
          SummonUnitDestroyed を Enqueue
          ★ Unit の未解決行動を予約リストから除外
      → SummonSlot = null
  5b. キャラにセット中のアイテムを破棄:
      → Item != null:
          Item を graveyard へ移動
          ItemExhausted(characterDowned) を Enqueue
          Item = null
  5c. キャラのパッシブを無効化:
      → 全パッシブの IsActive = false
      → パッシブ由来の StatModifier を StatCalculator から除去
  5d. StatCalculator.RecomputeAllStats は不要（ダウン済みキャラは計算対象外）

  ── Step 6: 勝敗判定は行わない ──
  ※ 勝敗判定はバトルフェイズ全行動解決後にのみ実行する
  ※ ダウン直後に蘇生される可能性があるため、即時判定しない
```

### 2.2 召喚ユニット経由ダメージの完全フロー

攻撃対象キャラに召喚ユニットが存在する場合、攻撃は召喚ユニットへ差し替わる。

```
ApplyDamageViaSummonUnit(state, targetCharId, attackContext):
  summonUnit = state.GetSummonUnit(targetCharId)

  ── Step 1: 召喚ユニットへのダメージ ──
  1a. DamageCalculator.CalculateDamage(attackContext) → DamageResult
  1b. summonUnit.hp = max(0, summonUnit.hp - finalDamage)
  1c. DamageDealt(target=summonUnit) を Enqueue

  ── Step 2: キャラへの返しダメージ（真ダメージ） ──
  2a. reflectDamage = floor(finalDamage / 2)
  2b. character.hp = max(0, character.hp - reflectDamage)
      ※ シールド/防御で軽減されない（真ダメージ扱い）
      ※ ただし「ダメージを受けた」判定にはなる
  2c. DamageDealt(target=character, type=True, isReflect=true) を Enqueue

  ── Step 3: 召喚ユニットのダウン判定 ──
  3a. summonUnit.hp == 0:
      → summonUnit を banished へ移動
      → SummonUnitDestroyed を Enqueue
      → SummonSlot.Unit = null
      → Unit の未解決行動を予約リストから除外

  ── Step 4: キャラのダウン判定 ──
  4a. character.hp == 0 && !character.isDown:
      → character.isDown = true
      → CharacterDowned を Enqueue
      → セクション2.1の Step 5 連鎖を実行
         （※ 召喚ユニットは既に Step 3 で処理済みの場合あり。二重除去しない）

  ── Step 5: イベント処理順 ──
  Enqueue された順序で処理:
    DamageDealt(summonUnit) → DamageDealt(character) → SummonUnitDestroyed（あれば） → CharacterDowned（あれば）
  各イベント処理後にトリガースキャンを実行
```

### 2.3 勝敗判定のタイミング（確定）

```
勝敗判定が実行されるタイミング（これ以外では判定しない）:
  1. バトルフェイズの全行動解決後（全 PlannedAction + 全トリガー割り込みの解決完了後）
  2. ターン終了フェイズの全処理完了後（maxTurns 到達時）

判定しないタイミング:
  - 個々の行動解決中（ダウンが発生しても続行）
  - トリガー割り込み解決中
  - ターン終了フェイズの途中（期限切れ処理中）
```

---

## 3. バースト状態の適用と解除

### 3.1 バースト状態適用の完全フロー

バトルフェイズ開始時、行動解決の前に実行。

```
ApplyBurstMode(state, characterId):
  ── Step 1: バーストゲージ消費 ──
  1a. team.Resources.BurstPoints -= 100
  1b. character.Burst.IsBurst = true
  1c. character.Burst.RemainingTurns = 3
  1d. character.Burst.CreatedSeq = state.NextSeq()

  ── Step 2: AP回復・解放 ──
  2a. character.Ap = min(character.Ap + 2, character.UnlockedMaxAP + 1)
      ※ +1は次ステップの解放を先取り
  2b. character.UnlockedMaxAP = min(character.UnlockedMaxAP + 1, character.ApCap)

  ── Step 3: ステータス上昇（倍率式） ──
  3a. バースト倍率を StatCalculator のパイプラインに追加:
      pAtk × 1.15, mAtk × 1.15
      pDef × 1.10, mDef × 1.10
      spd × 1.10, tech × 1.10
      maxHp: 変動なし
  3b. StatCalculator.RecomputeAllStats(state, characterId, Character)

  ── Step 4: 召喚のバースト変化 ──
  4a. SummonSlot に Object がある場合:
      → SummonManager.TransformToBurstVariant(state, characterId, Object)
      → HP比維持で再計算、寿命/状態維持
  4b. SummonSlot に Unit がある場合:
      → SummonManager.TransformToBurstVariant(state, characterId, Unit)
      → HP比維持で再計算、寿命/状態維持
      → ConfiguredActionSkillId をバースト版定義に切替

  ── Step 5: ユニークカードの強化版切替 ──
  5a. Unique.EffectiveUniqueCardId = burstVariantCardId
      ※ クールダウンは共有（値を変更しない）

  ── Step 6: イベント発行 ──
  6a. BurstModeApplied を Enqueue
```

### 3.2 バースト状態解除の完全フロー

ターン終了フェイズの createdSeq 順処理中に実行。

```
RemoveBurstMode(state, characterId):
  ── Step 1: ステータス倍率の解除 ──
  1a. バースト倍率を StatCalculator のパイプラインから除去
  1b. StatCalculator.RecomputeAllStats(state, characterId, Character)
      ※ この再計算でキャラのHPが0以下になることはない（maxHpは変動しないため）

  ── Step 2: AP上限の戻し ──
  2a. if character.UnlockedMaxAP > 非バースト最大上限:
      character.UnlockedMaxAP = 非バースト最大上限
  2b. if character.Ap > character.UnlockedMaxAP:
      character.Ap = character.UnlockedMaxAP

  ── Step 3: 召喚の通常版戻し ──
  3a. SummonSlot に Object がある場合:
      baseVariantCardId != null:
        → SummonManager.TransformToBaseVariant(state, characterId, Object)
        → HP比維持で再計算
        ★ 再計算後に召喚の HP == 0 の場合:
          → 召喚は消滅する（SummonObjectExpired を Enqueue）
          → SummonSlot.Object = null
      baseVariantCardId == null:
        → 召喚はそのまま維持（バーストボーナスのみ解除）
  3b. SummonSlot に Unit がある場合:
      （Object と同様の処理）
      ★ HP == 0 の場合:
        → SummonUnitExpired を Enqueue
        → SummonSlot.Unit = null

  ── Step 4: ユニークカードの通常版戻し ──
  4a. Unique.EffectiveUniqueCardId = Unique.EquippedUniqueCardId
      ※ クールダウンは維持

  ── Step 5: バースト状態フラグの解除 ──
  5a. character.Burst.IsBurst = false
  5b. character.Burst.RemainingTurns = 0

  ── Step 6: イベント発行 ──
  6a. BurstModeRemoved を Enqueue
```

---

## 4. ターン終了フェイズの処理順序

### 4.1 完全フロー

```
ProcessTurnEnd(state):
  ── Phase A: 永続効果リストの構築 ──
  A1. ゲーム中に存在する全永続効果を収集:
      - 召喚オブジェクト（各キャラの SummonSlot.Object）
      - 召喚ユニット（各キャラの SummonSlot.Unit）
      - バースト状態（各キャラの Burst where IsBurst==true）
      - フィールド効果（各チームの FieldEffect）
      - 状態異常/バフ（各キャラの Statuses）
      - シールド（各キャラの Shield）
      - ユニークCD（各キャラの Unique where CooldownRemainingTurns > 0）
      - サポートCD（各チームの SupportSet where CooldownRemainingTurns > 0）
  A2. createdSeq 昇順にソート → expiryList
  A3. ★ この時点のリストを「処理対象スナップショット」として固定
      → 以後の処理で新たに生成された永続効果はリストに追加しない

  ── Phase B: 順次処理 ──
  B1. expiryList の各要素について:
      a. 持続カウントを 1 減少
         - remainingTurns-- / durationTurnsRemaining-- / cooldownRemainingTurns--
      b. 減少後に 0 になった場合 → 消滅/期限切れ処理:

         【召喚オブジェクト（durationTurnsRemaining == 0）】
         → Object を banished へ移動
         → SummonObjectExpired を Enqueue
         → SummonSlot.Object = null
         → StatCalculator.RecomputeAllStats（上乗せ解除）

         【召喚ユニット（durationTurnsRemaining == 0）】
         → Unit を banished へ移動
         → SummonUnitExpired を Enqueue
         → SummonSlot.Unit = null

         【バースト状態（remainingTurns == 0）】
         → RemoveBurstMode（セクション3.2 の完全フロー実行）

         【フィールド効果（remainingTurns == 0、kind=="turns"の場合）】
         → FieldEffectExpired を Enqueue
         → team.FieldEffect = null
         → StatCalculator: 影響キャラのステータス再計算

         【状態異常/バフ（duration == 0）】
         → StatusExpired を Enqueue
         → target.Statuses から除去
         → StatCalculator: 影響キャラのステータス再計算（statMod の場合）

         【シールド（remainingTurns == 0）】
         → ShieldExpired(Expired) を Enqueue
         → target.Shield = null

         【ユニークCD / サポートCD（cooldownRemainingTurns == 0）】
         → カウントが0になるだけ（消滅はしない＝使用可能に戻る）

      c. ★ 消滅処理で Enqueue されたイベントに対してトリガースキャンを実行:
         → 条件成立したトリガーの効果を解決
         → 新たに生成された永続効果は expiryList に追加しない（次ターンから処理対象）
         → 無限ループ対策: エンティティ別の誘発上限10回を適用

  ── Phase C: 最終処理 ──
  C1. 手札上限チェック: なし（手札上限は設けない）
  C2. maxTurns 判定:
      state.Turn >= regulation.MaxTurns → 勝敗判定（タイブレーク）→ GameEnd
  C3. TurnEnded イベント発行
```

### 4.2 ターン終了中の連鎖に関する制約

| ルール | 内容 |
|---|---|
| **新規永続効果** | ターン終了フェイズ中に生成されたものは今ターンの処理リストに含めない |
| **消滅連鎖** | 消滅Aのトリガーが消滅Bを引き起こす場合、Bも即座に処理する（ただしBは expiryList の順番とは無関係） |
| **ステータス再計算** | 消滅ごとに必要に応じて実行する（まとめてバッチ処理しない） |
| **ダウン判定** | ターン終了中にキャラのHPが0になった場合もダウンは発生する（例: DoTで死亡）|
| **勝敗判定** | ターン終了フェイズの **全処理完了後** にのみ判定する |

---

## 5. 召喚入れ替え時の行動リスト更新

### 5.1 バトルフェイズ中の召喚入れ替えフロー

```
OnSummonReplaced(state, oldSummon, newSummon, actionResolverContext):
  ── Step 1: 旧召喚の行動除外 ──
  1a. 予約リスト（未解決分）から oldSummon.InstanceId の行動を検索
  1b. 見つかった場合:
      → 旧行動を予約リストから除外
      → ActionCancelled(reason=SummonReplaced) を Enqueue
  1c. 見つからない場合（既に解決済み or 行動なし）:
      → 何もしない

  ── Step 2: 新召喚の行動追加 ──
  2a. newSummon.ConfiguredActionSkillId が存在する場合:
      → 新しい PlannedAction を生成:
        actorType: Summon
        actorId: newSummon.InstanceId
        cardPriority: actionSkillDef.Priority
        spd: newSummon.Stats.Spd
  2b. ★ 挿入位置の決定:
      → 現在の解決位置（resolveIndex）より後ろの未解決行動リストに挿入
      → 挿入後、未解決リスト全体をソートキーで再ソート
      → ★ 新行動の priority が「現在解決中の行動」より高くても、
        現在の行動の後に実行される（割り込みはしない）
  2c. newSummon.HasActedThisTurn = false

  ── Step 3: イベント発行 ──
  3a. SummonReplaced を Enqueue
```

### 5.2 挿入ソートの保証

- 新行動は **未解決行動の中で** ソートキーに従った位置に挿入される
- 既に解決済みの行動は影響を受けない
- 同一ソートキーの場合、新行動は **末尾** に配置される（createdSeq が最新のため）

---

## 6. 行動解決中のイベント処理フロー

### 6.1 1つの行動解決の完全なイベント処理順

```
ResolveOneAction(state, action, resolverContext):
  ── Phase 1: 行動実行 ──
  1. 不発チェック → 不発なら ActionResolved(fizzled) を Enqueue して終了
  2. コスト支払い（AP、エレメント）
  3. エレメント獲得（有色カード使用時）
  4. 効果解決（EffectResolver）
     → 効果解決中に発生するイベントは即座に Enqueue:
       DamageDealt, Healed, StatusApplied, SummonEntered, etc.
  5. カードを graveyard へ移動
  6. ActionResolved(executed) を Enqueue

  ── Phase 2: イベントキュー処理 ──
  7. EventQueue.ProcessAll():
     → キューが空になるまで FIFO で処理
     → 各イベント処理後にトリガースキャンを実行:
       - アイテム自動発動
       - パッシブ誘発
       - 状態異常誘発
       - フィールド効果誘発
     → トリガー発火で新イベントが生成された場合、キュー末尾に追加
     → 無限ループ対策: エンティティ別上限10回 + 再帰深度上限5レベル

  ── Phase 3: 割り込みトリガー判定（バトルフェイズのみ） ──
  8. 予約済みカードのトリガー条件をスキャン
  9. 条件成立したカードがある場合:
     → 複数成立: イニシアチブ側優先 → 同チーム内は createdSeq 昇順
     → 各割り込みカードについて:
       a. 不発チェック（AP不足等）
       b. AP消費
       c. 効果解決
       d. カードを graveyard へ
       e. ActionResolved(executed, interrupt) を Enqueue
       f. 予約リストから除外
       g. Phase 2 を再実行（割り込みで発生したイベントを処理）

  ── Phase 4: バーストゲージ加算 ──
  10. team.BurstPoints += actionChargePoints（デフォルト +30）
      ※ 割り込み発動も行動チャージの対象

  ── Phase 5: ステータス再計算 ──
  11. 変動があった場合のみ StatCalculator.RecomputeAllStats
```

### 6.2 割り込みの再帰に関する制約

- 割り込みカードの効果解決後のトリガースキャンで、**さらに別の割り込みカードが成立する場合がある**
- この場合、**再帰的に割り込みを解決する**（Phase 3 の再入）
- ただし再帰深度は **上限5レベル** とする
- 割り込みカードがトリガーとして自分自身を再発火することはない（発動済みのため予約リストから除外済み）

---

## 7. 蘇生処理の状態遷移

### 7.1 蘇生の完全フロー

```
ResolveResurrect(state, targetId, params, context):
  ── Step 1: 前提チェック ──
  1a. target.IsDown == false → 効果なし（既に生存）。終了
  1b. target.IsDown == true → 蘇生処理に進む

  ── Step 2: 基本復帰 ──
  2a. target.IsDown = false
  2b. target.Hp = max(1, floor(target.MaxHp * params.HpPercent))
  2c. target.Shield = null（シールドはリセット）

  ── Step 3: 状態の引き継ぎ ──
  3a. target.Statuses → 維持（蘇生前に付与されていたバフ/デバフは残る）
      ★ actionBlock 系（凍結/スタン等）も維持される
      ★ 「蘇生時にCC解除」が必要な場合はカード効果側で Dispel を組み合わせる
  3b. target.Ap → 維持（ダウン時のAP値がそのまま残る）
  3c. target.Elements → 維持
  3d. target.UnlockedMaxAP → 維持

  ── Step 4: パッシブの再有効化 ──
  4a. 各パッシブの解放条件を再チェック:
      UnlockedMaxAP >= UnlockThreshold → IsActive = true
  4b. 有効化されたパッシブの StatModifier を StatCalculator に追加

  ── Step 5: 召喚枠 ──
  5a. 蘇生時に召喚は復帰しない（ダウン時に消滅済み）
  5b. SummonSlot は null のまま

  ── Step 6: ステータス再計算 ──
  6a. StatCalculator.RecomputeAllStats(state, targetId, Character)

  ── Step 7: イベント発行 ──
  7a. CharacterResurrected を Enqueue
```

---

## 8. ステータス再計算と連鎖の制約

### 8.1 再計算をトリガーする条件

| トリガー | 影響範囲 |
|---|---|
| バフ/デバフの付与 | 対象キャラ |
| バフ/デバフの解除/期限切れ | 対象キャラ |
| バースト状態の適用/解除 | 対象キャラ + 召喚 |
| 召喚オブジェクトの生成/消滅 | 紐づくキャラ |
| フィールド効果の展開/消滅 | scope に応じた全対象キャラ |
| パッシブの有効化/無効化 | 対象キャラ |
| 蘇生 | 蘇生されたキャラ |

### 8.2 再計算の連鎖制約

- ステータス再計算自体は **イベントを発行しない**（StatRecomputed はデバッグ用ログ）
- 再計算の結果、**パッシブの解放条件が満たされることはない**
  - 理由: パッシブの解放条件は `UnlockedMaxAP`（AP解放値）であり、ステータス値ではない
  - よってステータス再計算 → パッシブ有効化の連鎖は発生しない
- 再計算の結果、**HPが0以下になることはない**
  - 理由: ステータス再計算は maxHp を変更しうるが、currentHp > maxHp の場合に currentHp を切り詰める処理は行わない
  - 例外: maxHp が下がった結果 currentHp > maxHp となる場合は、**currentHp = maxHp に切り詰める**（ただし maxHp >= 1 が保証されるため HP=0 にはならない）

### 8.3 複数キャラの再計算順序

フィールド効果など複数キャラに影響する再計算の場合:
1. **チーム内**: キャラの createdSeq 昇順（= CharacterState の配列順）
2. **チーム間**: initiativeTeamId 側を先に処理
3. 各キャラの再計算は独立（キャラAの再計算結果がキャラBの再計算に影響しない）

---

## 9. 状態遷移図（サマリー）

### 9.1 キャラの状態遷移

```
Alive ──(HP=0)──→ Down ──(Resurrect)──→ Alive
  │                 │
  │                 └──(GameEnd)──→ 最終状態
  │
  ├──(BurstApply)──→ Alive+Burst ──(3ターン経過)──→ Alive
  │                    │
  │                    └──(HP=0)──→ Down（バーストも同時解除）
  │
  └──(Surrender/Disconnect)──→ 強制敗北
```

### 9.2 召喚の状態遷移

```
NotExist ──(SummonEntered)──→ Active ──(寿命0)──→ Expired → banished
                                │
                                ├──(カード効果で破壊)──→ Destroyed → banished
                                │
                                ├──(上書き入れ替え)──→ Replaced → banished
                                │
                                ├──(キャラダウン)──→ Destroyed → banished
                                │
                                ├──(BurstApply)──→ Active(BurstVariant)
                                │                    │
                                │                    └──(BurstRemove)──→ Active(通常版)
                                │                                        or Expired(HP0)
                                │
                                └──(BurstRemove+HP0)──→ Expired → banished
```

### 9.3 シールドの状態遷移

```
None ──(ShieldApply)──→ Active ──(ダメージ吸収で残量0)──→ Depleted → None
                          │
                          ├──(残りターン0)──→ Expired → None
                          │
                          ├──(新シールド付与)──→ Overwritten → None → Active(新)
                          │
                          └──(キャラダウン)──→ None（リセット）
```
