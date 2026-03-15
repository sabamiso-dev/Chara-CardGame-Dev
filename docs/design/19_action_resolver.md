# 行動解決パイプライン設計

ルールエンジン設計の一部。バトルフェイズでの行動ソート・不発判定・順次解決を定義する。

関連: [ゲームループ](17_game_loop_and_phases.md) / [イベント・トリガー](18_event_and_trigger.md) / [効果解決](20_stat_damage_effect.md)

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

