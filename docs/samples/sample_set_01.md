# サンプルセット01: カイン（物理アタッカー / 赤）

仕様の認識合わせ用。全カード種別（スキル/召喚オブジェクト/召喚ユニット/アイテム/ユニーク）を1体分のデッキで網羅する。
サポートカードはチーム共通枠のためデッキ外で別途定義。

参照: `docs/requirements/14_content_baselines_and_data_workflow.md`（基礎ステータス目安）

---

## キャラ定義

```
ID:            char_kain_striker
名前:          カイン
色:            赤
ロール:        物理アタッカー

── ステータス ──
HP:            1200
pAtk:          310
pDef:          170
mAtk:          140
mDef:          160
spd:           265
tech:          200

── AP ──
apStart:       3
apMaxStart:    3
apCap:         9
apRegen:       2

── パッシブ（3枠） ──
パッシブ1:  闘志         解放条件: unlockedMaxAP >= 4
            効果: pAtk +15（常時。statMod / additive）
パッシブ2:  血の加速     解放条件: unlockedMaxAP >= 6
            効果: 行動チャージ量が +30 → +40 に増加（バーストゲージ蓄積率UP）
パッシブ3:  覚醒の一撃   解放条件: unlockedMaxAP >= 8
            効果: HP50%以下の時、物理ダメージ倍率 +0.2（条件付き常時）

── ユニーク候補（3択から1） ──
1. unique_kain_flash_strike   （閃撃）
2. unique_kain_adrenaline     （アドレナリン）
3. unique_kain_final_blow     （最後の一撃）
```

---

## キャラデッキ（12枚）

### スキルカード（7枚）

```
────────────────────────────────────────
ID:       skill_red_slash
名前:     斬撃
色:       赤
種別:     スキル
コスト:   2 AP
priority: 0
タグ:     [攻撃, 物理]

効果:
  effectType: Damage
  damageType: physical
  cardMultiplier: 1.0
  target: { mode: enemyCharacter }

テキスト: 敵1体に物理ダメージ（pAtk×1.0）を与える。

────────────────────────────────────────
ID:       skill_red_double_strike
名前:     連続斬り
色:       赤
種別:     スキル
コスト:   3 AP
priority: -1
タグ:     [攻撃, 物理, 多段]

効果:
  effectType: Damage
  damageType: physical
  cardMultiplier: 0.6
  hitCount: 2
  target: { mode: enemyCharacter }

テキスト: 敵1体に物理ダメージ（pAtk×0.6）を2回与える。

────────────────────────────────────────
ID:       skill_red_flame_burst
名前:     爆炎撃
色:       赤
種別:     スキル
コスト:   4 AP
priority: -2
elementCost: { red: 2 }
タグ:     [攻撃, 物理, フィニッシャー]

効果:
  effectType: Damage
  damageType: physical
  cardMultiplier: 1.8
  target: { mode: enemyCharacter }

テキスト: 赤エレメント2消費。敵1体に物理ダメージ（pAtk×1.8）を与える。

────────────────────────────────────────
ID:       skill_red_quick_strike
名前:     先制突き
色:       赤
種別:     スキル
コスト:   1 AP
priority: +2
タグ:     [攻撃, 物理, 速攻]

効果:
  effectType: Damage
  damageType: physical
  cardMultiplier: 0.5
  target: { mode: enemyCharacter }

テキスト: 敵1体に物理ダメージ（pAtk×0.5）を与える。

────────────────────────────────────────
ID:       skill_red_war_cry
名前:     雄叫び
色:       赤
種別:     スキル
コスト:   2 AP
priority: +1
タグ:     [バフ, 自己強化]

効果:
  effectType: StatusApply
  statusId: status_patk_up
  duration: 3
  overrideValue: 30  （pAtk +30）
  target: { mode: self }

テキスト: 自分にpAtk+30（3ターン）を付与する。

セット効果:
  effectType: StatusApply
  statusId: status_patk_up_loadout
  duration: -1（永続）
  overrideValue: 10（pAtk +10）
  target: { mode: self }

テキスト（セット効果）: pAtk +10（永続）。

────────────────────────────────────────
ID:       skill_colorless_guard
名前:     防御姿勢
色:       無色
種別:     スキル
コスト:   1 AP
priority: +1
タグ:     [防御, シールド]

効果:
  effectType: Shield
  shieldType: fixed
  value: 80
  duration: 2
  target: { mode: self }

テキスト: 自分にシールド80（2ターン）を付与する。

────────────────────────────────────────
ID:       skill_red_counter_stance
名前:     反撃の構え
色:       赤
種別:     スキル
コスト:   2 AP
priority: +1
タグ:     [攻撃, 物理, トリガー]

trigger:
  name: カウンタートリガー
  event: onDamaged
  condition: null（ダメージを受ければ発動）

効果:
  effectType: Damage
  damageType: physical
  cardMultiplier: 0.5
  target: { mode: enemyCharacter }
  ※ トリガー発動時は攻撃してきた敵を対象にする

テキスト:
  通常: 敵1体に物理ダメージ（pAtk×0.5）を与える。
  （カウンタートリガー）: このターン中、このキャラがダメージを受けた時、
  割り込みで発動する。
```

### 召喚オブジェクトカード（1枚）

```
────────────────────────────────────────
ID:       summon_object_red_flame_totem
名前:     炎のトーテム
色:       赤
種別:     召喚オブジェクト
コスト:   3 AP
priority: 0
タグ:     [召喚, オブジェクト, 自己強化]

durationTurns: 3
objectScaling:
  pAtk: 0.3   （キャラpAtkの30%を上乗せ）
  pDef: 0.2
objectFlatStats:
  pAtk: 10
  pDef: 5

onEnterEffects: なし
burstVariantCardId: summon_object_red_flame_totem_burst

テキスト: 召喚オブジェクトを配置する（3ターン）。
          キャラにpAtk（キャラpAtk×0.3 +10）、pDef（キャラpDef×0.2 +5）を上乗せする。

── 数値例（カインの場合） ──
  pAtk上乗せ: floor(310 × 0.3) + 10 = 103
  pDef上乗せ: floor(170 × 0.2) + 5 = 39
  → カインのpAtk: 310 + 103 = 413, pDef: 170 + 39 = 209
```

### 召喚ユニットカード（1枚）

```
────────────────────────────────────────
ID:       summon_unit_red_fire_wolf
名前:     炎狼
色:       赤
種別:     召喚ユニット
コスト:   4 AP
priority: 0
タグ:     [召喚, ユニット, 攻撃]

durationTurns: 2
unitScaling:
  pAtk: 0.7
  pDef: 0.4
  hp:   0.5
  spd:  0.8
unitFlatStats:
  pAtk: 20
  pDef: 10
  hp:   100
  spd:  0

actionSkills:
  - id: action_fire_wolf_bite
    名前: 火炎咬み
    priority: 0
    効果:
      effectType: Damage
      damageType: physical
      cardMultiplier: 0.8
      target: { mode: enemyCharacter }

passives: なし

onEnterEffects:
  - effectType: Damage
    damageType: physical
    cardMultiplier: 0.4
    target: { mode: enemyCharacter }

burstVariantCardId: summon_unit_red_fire_wolf_burst

テキスト: 召喚ユニット「炎狼」を召喚する（2ターン）。
          登場時: 敵1体に物理ダメージ（pAtk×0.4）。
          行動: 火炎咬み — 敵1体に物理ダメージ（pAtk×0.8）。

── 数値例（カインの場合） ──
  hp:   floor(1200 × 0.5) + 100 = 700
  pAtk: floor(310 × 0.7) + 20 = 237
  pDef: floor(170 × 0.4) + 10 = 78
  spd:  floor(265 × 0.8) + 0 = 212

── 召喚入れ替え（炎のトーテムが存在する場合） ──
  軽減量: floor(3 / 2) = 1
  実行コスト: max(0, 4 - 1) = 3 AP
  ※ 炎のトーテム（オブジェクト）は上書きで除去される

── 攻撃対象の差し替え ──
  炎狼が存在する間、カインへの攻撃は炎狼へ差し替わる。
  炎狼がダメージD を受けた場合:
    炎狼: D ダメージ
    カイン: floor(D/2) の真ダメージ（軽減されない）
```

### アイテムカード（1枚）

```
────────────────────────────────────────
ID:       item_red_berserker_band
名前:     狂戦士の腕輪
色:       赤
種別:     アイテム
コスト:   2 AP
priority: -2
タグ:     [アイテム, 自己強化, 攻撃]

count: 3（3回発動で破棄）

triggers:
  - event: onDamaged
    condition: null（ダメージを受けたら発動）

効果:
  effectType: StatusApply
  statusId: status_patk_up
  duration: 2
  overrideValue: 15（pAtk +15）
  target: { mode: self }

テキスト: キャラにセットする。
          このキャラがダメージを受けた時、pAtk+15（2ターン）を得る。（残り3回）

── 動作例 ──
  ターン3: アイテムセット（2AP消費）→ キャラにセット
  ターン3バトル: カインが敵から攻撃を受ける
    → 腕輪が自動発動: pAtk+15（2ターン）付与。count: 3→2
  ターン4バトル: カインが再度攻撃を受ける
    → 腕輪が自動発動: pAtk+15（2ターン）付与。count: 2→1
    ※ stackingRule=stack の場合、2つのバフが重なる
  ターン5バトル: カインが攻撃を受ける
    → 腕輪が自動発動: count: 1→0 → アイテム破棄（ItemExhausted）
```

### アイテムカード（2枚目 — 回復系）

```
────────────────────────────────────────
ID:       item_colorless_emergency_heal
名前:     緊急回復キット
色:       無色
種別:     アイテム
コスト:   2 AP
priority: -2
タグ:     [アイテム, 回復]

count: 2

triggers:
  - event: onAllyHpBelow
    condition: "hpPercent <= 0.3"（HPが30%以下になった時）

効果:
  effectType: Heal
  calcMode: maxHpPercent
  value: 0.15（maxHPの15%回復）
  target: { mode: self }

テキスト: キャラにセットする。
          このキャラのHPが30%以下になった時、maxHPの15%を回復する。（残り2回）

── 数値例 ──
  カインのmaxHP=1200の場合:
  HP 360以下になった時 → 180回復
```

---

## ユニークカード（3択）

```
────────────────────────────────────────
ID:       unique_kain_flash_strike
名前:     閃撃
色:       赤
種別:     ユニーク
コスト:   2 AP
cooldownTurns: 3
priority: +2
タグ:     [攻撃, 物理, 速攻]

効果:
  effectType: Damage
  damageType: physical
  cardMultiplier: 0.7
  target: { mode: enemyCharacter }

burstVariantCardId: unique_kain_flash_strike_burst

テキスト: 敵1体に物理ダメージ（pAtk×0.7）を与える。CD: 3ターン。

── バースト強化版 ──
ID:       unique_kain_flash_strike_burst
名前:     閃撃・極
コスト:   2 AP
cooldownTurns: 3（共有）
priority: +2

効果:
  effectType: Composite
  children:
    - effectType: Damage
      damageType: physical
      cardMultiplier: 1.2
      target: { mode: enemyCharacter }
    - effectType: StatusApply
      statusId: status_pdef_down
      duration: 2
      overrideValue: 20（pDef -20）
      target: { mode: enemyCharacter }

テキスト: 敵1体に物理ダメージ（pAtk×1.2）を与え、pDef-20（2ターン）を付与する。

────────────────────────────────────────
ID:       unique_kain_adrenaline
名前:     アドレナリン
色:       赤
種別:     ユニーク
コスト:   1 AP
cooldownTurns: 4
priority: +1
タグ:     [バフ, 自己強化]

効果:
  effectType: Composite
  children:
    - effectType: StatusApply
      statusId: status_patk_up
      duration: 2
      overrideValue: 40
      target: { mode: self }
    - effectType: StatusApply
      statusId: status_spd_up
      duration: 2
      overrideValue: 20
      target: { mode: self }

burstVariantCardId: unique_kain_adrenaline_burst

テキスト: 自分にpAtk+40, spd+20（2ターン）を付与する。CD: 4ターン。

── バースト強化版 ──
ID:       unique_kain_adrenaline_burst
名前:     アドレナリン・極

効果:
  effectType: Composite
  children:
    - effectType: StatusApply （pAtk+60, 2ターン）
    - effectType: StatusApply （spd+30, 2ターン）
    - effectType: APModify
      amount: 1
      targetStat: current

テキスト: 自分にpAtk+60, spd+30（2ターン）を付与し、AP+1回復する。

────────────────────────────────────────
ID:       unique_kain_final_blow
名前:     最後の一撃
色:       赤
種別:     ユニーク
コスト:   3 AP
cooldownTurns: 5
priority: -3
タグ:     [攻撃, 物理, フィニッシャー]

効果:
  effectType: Conditional
  condition:
    conditionType: HpThreshold
    scope: target
    operator: lte
    thresholdPercent: 0.3
  thenEffect:
    effectType: Damage
    damageType: physical
    cardMultiplier: 2.5
    target: { mode: enemyCharacter }
  elseEffect:
    effectType: Damage
    damageType: physical
    cardMultiplier: 1.0
    target: { mode: enemyCharacter }

burstVariantCardId: null（バースト強化版なし）

テキスト: 敵1体に物理ダメージを与える。
          対象のHPが30%以下なら倍率2.5、それ以外は倍率1.0。CD: 5ターン。
```

---

## サポートカード（チーム共通枠 / デッキ外）

```
────────────────────────────────────────
ID:       support_red_inferno
名前:     業火
色:       赤
種別:     サポート
burstCostPoints: 200（2ゲージ）
cooldownTurns: 3
priority: -1
タグ:     [攻撃, 全体, 物理]

効果:
  effectType: Damage
  damageType: physical
  cardMultiplier: 0.8
  target: { mode: allEnemies }

setEffects:
  effectType: StatusApply
  statusId: status_patk_up_loadout
  duration: -1
  overrideValue: 5（pAtk +5。チーム全体）
  scope: team

テキスト: Burst 2ゲージ消費。敵全体に物理ダメージ（pAtk×0.8）を与える。
セット効果: チーム全体のpAtk +5（永続）。

── 数値例（カイン pAtk=310 で使用） ──
  各敵キャラに対して:
    damage = max(1, floor(310 × 0.8 × 100 / (100 + 敵pDef × 0.6)))
    敵pDef=200 の場合: floor(248 × 100 / 220) = floor(112.7) = 112 ダメージ
```

---

## デッキ構成まとめ

| # | ID | 種別 | コスト | priority | 色 |
|---|---|---|---|---|---|
| 1 | skill_red_slash | スキル | 2 | 0 | 赤 |
| 2 | skill_red_double_strike | スキル | 3 | -1 | 赤 |
| 3 | skill_red_flame_burst | スキル | 4 | -2 | 赤 |
| 4 | skill_red_quick_strike | スキル | 1 | +2 | 赤 |
| 5 | skill_red_war_cry | スキル | 2 | +1 | 赤 |
| 6 | skill_colorless_guard | スキル | 1 | +1 | 無色 |
| 7 | skill_red_counter_stance | スキル（トリガー付き） | 2 | +1 | 赤 |
| 8 | summon_object_red_flame_totem | 召喚オブジェクト | 3 | 0 | 赤 |
| 9 | summon_unit_red_fire_wolf | 召喚ユニット | 4 | 0 | 赤 |
| 10 | item_red_berserker_band | アイテム | 2 | -2 | 赤 |
| 11 | item_colorless_emergency_heal | アイテム | 2 | -2 | 無色 |

**計11枚**（キャラデッキ10〜15枚の範囲内）
**色構成**: 赤9枚 + 無色2枚（2色制限に適合、無色比率 2/11 ≈ 18% < 50%）

---

## 仕様確認ポイント

このサンプルで以下の仕様が正しく表現されているか確認してください:

1. **EffectSpec の構造** — 各カードの効果が EffectType + params で表現できているか
2. **priority の使い方** — 速攻(+2) / 標準(0) / 重い技(-2,-3) のグラデーション
3. **トリガーシステム** — `反撃の構え` の割り込み発動（通常時はpriority+1で普通に発動）
4. **召喚オブジェクトの上乗せ** — scaling + flat のステータス加算
5. **召喚ユニットの行動** — 0コストアクションスキル、登場時効果
6. **召喚入れ替え** — オブジェクト→ユニットの上書き、コスト軽減
7. **アイテムの自動発動** — トリガー条件、count消費、破棄
8. **ユニークの通常版/バースト強化版** — クールダウン共有、効果の変化
9. **Conditional（条件分岐）** — 最後の一撃のHP閾値判定
10. **セット効果** — 雄叫びの永続pAtk補正、サポートのチーム全体補正
11. **エレメントコスト** — 爆炎撃の赤エレメント2消費
12. **数値例** — 召喚ステータスの算出結果、ダメージ試算
