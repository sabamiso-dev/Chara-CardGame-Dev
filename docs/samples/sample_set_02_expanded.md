# サンプルセット02: CardSample_mycreate.md を展開したカード群

`CardSample_mycreate.md` のカードアイデアを EffectSpec 形式に変換し、
さらに同系統の追加カードを作成する。

---

## 新メカニクスの仕様整理

`CardSample_mycreate.md` から読み取れる、既存仕様に対する拡張提案。

### M1. 段階追加効果（Tiered Element Bonus）

カード効果に「追加エレメントを消費すると効果が強化される」仕組み。
1枚のカードが複数段階の追加効果を持つ。

```
設計案:
  effectType: Composite
  children:
    - (基本効果: エレメント消費なし)
    - effectType: Conditional
      condition: ElementThreshold(色, >=, N)  ← 保有チェック
      thenEffect:
        effectType: Composite
        children:
          - ElementConsume(色, N)
          - (追加効果)
```

### M2. デッキ操作（Deck Manipulation）

- **ボトムドロー**: デッキの一番下からカードを手札に加える
- **デッキサーチ（発掘）**: デッキ上からX枚を確認し、条件に合うカードを手札に加える
- → 新 EffectType `DeckManipulate` の追加が必要

### M3. バウンス（Bounce）

- フィールドの召喚/アイテムを手札に戻す
- → 新 EffectType `Bounce` の追加が必要

### M4. トークン生成（Token Creation）

- 一時的なカードを生成して手札に加える。使用後は追放（banished）
- → 既存の追放ゾーン設計と整合（`01_core_rules.md`）

### M5. キーワード能力（Keyword Ability）

- 【発掘:X】【消失】等、複数カードで共有される定型効果
- → EffectSpec の再利用（共通定義を参照する仕組み）

---

## CardSample_mycreate.md のカードを EffectSpec 形式に変換

### 観測

```
────────────────────────────────────────
ID:       skill_blue_observation
名前:     観測
色:       青
種別:     スキル
コスト:   1 AP
priority: +2
タグ:     [妨害, 情報, バフ]

効果:
  effectType: Composite
  children:
    - effectType: Reveal           ← 新EffectType
      revealParams:
        target: { mode: enemyTeam }
        scope: hand               （手札を公開）
        duration: 1               （このターンのみ）
    - effectType: StatusApply
      statusId: status_accuracy_up
      duration: 1
      overrideValue: 20           （命中率+20%）
      target: { mode: self }      （自分の召喚ユニット含む）

テキスト: 相手の手札を確認する。このターン、自身のユニットの攻撃命中率を20%アップ。

備考: 「手札公開」は新メカニクス。Phase 1 では内部ログに手札情報を
      記録するだけでも機能する（UI実装はPhase 2）。
────────────────────────────────────────
```

### 荷物整理

```
────────────────────────────────────────
ID:       skill_colorless_inventory_sort
名前:     荷物整理
色:       無色
種別:     スキル
コスト:   2 AP
priority: -2
タグ:     [ドロー, 手札操作]

効果:
  effectType: Composite
  children:
    - effectType: Draw
      drawParams:
        count: 2
    - effectType: Discard          ← 新EffectType
      discardParams:
        count: 2
        mode: lowestCost          （コストが最も低いカードから）
        selection: random         （同コスト内はランダム）
        destination: graveyard

テキスト: カードを2枚ドローする。その後、手札のコストが最も低いカード2枚を
          ランダムにトラッシュへ送る。

備考: ドロー後の手札全体から対象を選ぶ。
      コスト0が複数あればその中からランダム（RngProvider使用）。
────────────────────────────────────────
```

### 野営準備

```
────────────────────────────────────────
ID:       skill_green_camp_prep
名前:     野営準備
色:       緑
種別:     スキル
コスト:   3 AP
priority: +1
タグ:     [防御, 回復, 持続]

効果:
  effectType: Composite
  children:
    - effectType: StatusApply
      statusId: status_damage_reduction
      duration: 2
      overrideValue: 20           （被ダメージ20%軽減）
      target: { mode: self }
    - effectType: StatusApply
      statusId: status_regen
      duration: 2
      target: { mode: self }
      ※ status_regen: ターン終了時にmaxHPの20%回復（dot/isHeal=true）

テキスト: 受けるダメージを20%軽減し、ターン終了時にmaxHPの20%を回復する（2ターン）。

── 数値例（HP1200のキャラ） ──
  被ダメ軽減: 200ダメージ → 160ダメージ
  ターン終了回復: floor(1200 × 0.2) = 240回復/ターン
────────────────────────────────────────
```

### 非常への備え

```
────────────────────────────────────────
ID:       skill_yellow_emergency_prep
名前:     非常への備え
色:       黄
種別:     スキル
コスト:   3 AP
priority: -1
タグ:     [アイテム付与, 支援]

効果:
  effectType: Composite
  children:
    - effectType: ItemSet
      itemSetParams:
        itemCardId: item_token_ration （「非常食」トークンアイテム）
      target: { mode: self }
    - effectType: ItemSet
      itemSetParams:
        itemCardId: item_token_ration
      target: { mode: allyCharacter }  （味方1体を指定）

テキスト: 自身と味方1体に「非常食」をセットする。

── 「非常食」トークンアイテム ──
  ID:     item_token_ration
  名前:   非常食
  count:  1
  trigger: { event: onAllyHpBelow, condition: "hpPercent <= 0.4" }
  効果:   Heal { calcMode: maxHpPercent, value: 0.25 }
  ※ 使用後は追放（banished）。トークンのため再利用不可。

備考: 味方へのアイテムセットは「allyCharacter」対象。
      既にアイテムがセット済みの場合は上書きされる。
      プレイヤーが上書きリスクを判断する必要がある。
────────────────────────────────────────
```

### 秘伝ゴーレム召喚

```
────────────────────────────────────────
ID:       skill_orange_golem_summon
名前:     秘伝ゴーレム召喚
色:       橙
種別:     スキル
コスト:   4 AP
priority: 0
タグ:     [召喚, 段階強化]

効果:
  effectType: Composite
  children:
    ── 基本効果 ──
    - effectType: Summon
      summonParams:
        summonCardId: summon_unit_orange_golem

    ── ①追加(土-3): 上位ゴーレム召喚 ──
    - effectType: Conditional
      condition:
        conditionType: ElementThreshold
        element: orange
        operator: gte
        threshold: 3
      thenEffect:
        effectType: Composite
        children:
          - effectType: ElementConsume { element: orange, amount: 3 }
          - effectType: Summon
            summonParams:
              summonCardId: summon_unit_orange_iron_golem
            ※ この Summon が基本の golem を上書きする

    ── ②追加(土-2): ゴーレム強化 ──
    - effectType: Conditional
      condition:
        conditionType: ElementThreshold
        element: orange
        operator: gte
        threshold: 2
      thenEffect:
        effectType: Composite
        children:
          - effectType: ElementConsume { element: orange, amount: 2 }
          - effectType: StatusApply
            statusId: status_golem_boost  （「ゴーレム」タグ持ち召喚のATK,DEF+100%）
            duration: -1（永続）
            target: { mode: allySummonUnit }
            ※ filterTags: ["ゴーレム"] で対象を絞る

    ── ③追加(土-1): ドロー ──
    - effectType: Conditional
      condition:
        conditionType: ElementThreshold
        element: orange
        operator: gte
        threshold: 1
      thenEffect:
        effectType: Composite
        children:
          - effectType: ElementConsume { element: orange, amount: 1 }
          - effectType: Draw { count: 1 }

テキスト:
  ゴーレムを召喚する。
  ①追加(橙-3): ゴーレムではなく鉄のゴーレムを召喚する。
  ②追加(橙-2): 自身の「ゴーレム」を持つ召喚ユニットのATK,DEFを+100%。
  ③追加(橙-1): カードを1枚ドロー。

備考:
  - ①②③は独立した Conditional のため、エレメントがあれば複数同時に発動可能
  - ①と②を両方発動する場合、橙エレメント5消費
  - ①で上位ゴーレム召喚 → ②でそのゴーレムも強化、という連鎖が成立
  - 段階追加効果のパターンとして、他のカードにも応用できる
────────────────────────────────────────
```

### 相棒コール

```
────────────────────────────────────────
ID:       skill_colorless_buddy_call
名前:     相棒コール
色:       無色
種別:     スキル
コスト:   X AP（可変コスト: 発動時の残りAPをすべて消費）
priority: -1
タグ:     [召喚, サーチ, 可変コスト]

効果:
  effectType: DeckSearch          ← 新EffectType
  deckSearchParams:
    costMode: allRemainingAP      （残りAPすべてを消費。X = 消費したAP）
    filterTags: ["バディ"]         （「バディ」タグを持つ召喚カード）
    filterCost: lte               （コスト ≤ X）
    action: play                  （見つけたカードをそのまま発動）

テキスト: 残りAPをすべて支払う（X）。デッキからコストX以下の
          「バディ」召喚カードを1枚選んで発動する。

備考:
  - 可変コスト（X）は新メカニクス。EffectSpec に costMode を追加する必要あり
  - デッキサーチ → 即座発動は強力なので、対象を「バディ」タグに限定
  - 見つからなかった場合は不発（APは消費済み）
  - サーチしたカードはデッキから除去 → 発動 → graveyard へ
────────────────────────────────────────
```

### トリック・ドロー

```
────────────────────────────────────────
ID:       skill_blue_trick_draw
名前:     トリック・ドロー
色:       青
種別:     スキル
コスト:   2 AP
priority: -1
タグ:     [ドロー, 手札操作]

効果:
  effectType: Composite
  children:
    - effectType: Draw
      drawParams:
        count: 3
    - effectType: Discard
      discardParams:
        count: 1
        mode: random
        destination: graveyard

テキスト: カードを3枚ドローする。その後、手札からランダムに1枚をトラッシュへ送る。

── 期待値 ──
  実質+2枚のハンドアドバンテージ。
  ランダム捨てのリスクがあるが、手札が少ない時はリスクが低い。
────────────────────────────────────────
```

### アクセスダイブ

```
────────────────────────────────────────
ID:       skill_blue_access_dive
名前:     アクセスダイブ
色:       青
種別:     スキル
コスト:   2 AP
priority: -1
タグ:     [ドロー, デッキ操作, 段階強化]

効果:
  effectType: Composite
  children:
    ── 基本効果 ──
    - effectType: DeckManipulate    ← 新EffectType
      deckManipulateParams:
        action: drawFromBottom
        count: 1

    ── ①追加(無-3): トークン生成 ──
    - effectType: Conditional
      condition: ElementThreshold(colorless... )
      ※ 無色エレメントは存在しないため、別の条件に置き換えが必要
      → 提案: 「任意の色のエレメント3消費」or 「APを追加消費」

      thenEffect:
        effectType: TokenCreate     ← 新EffectType
        tokenCreateParams:
          templateColor: casterColor （使用キャラの色と同じ）
          tags: ["消失"]             （使用後 → banished）
          addToHand: true

    ── ②追加(水-2): 相手召喚バウンス ──
    - effectType: Conditional
      condition: ElementThreshold(blue, >=, 2)
      thenEffect:
        effectType: Composite
        children:
          - effectType: ElementConsume { element: blue, amount: 2 }
          - effectType: Bounce
            bounceParams:
              target: { mode: enemySummonUnit, selection: random }
              count: 1
              filterCost: lte        （手札に加えたカードのコスト以下）
              destination: deckBottom （デッキの一番下へ）

テキスト:
  デッキの一番下にあるカードを手札に加える。
  ①追加(任意色-3): キャラと同じ色のカードを1枚作成して手札に加える（【消失】付き）。
  ②追加(青-2): 手札に加えたカードのコスト以下の相手召喚を1体、デッキの一番下へ戻す。

備考:
  - 「無色エレメント」は仕様上存在しないため、①の条件を要検討
  - 「手札に加えたカードのコスト」を参照する処理は、
    直前の効果の結果を次の効果で参照する仕組みが必要
────────────────────────────────────────
```

### 地層調査

```
────────────────────────────────────────
ID:       skill_orange_excavation
名前:     地層調査
色:       橙
種別:     スキル
コスト:   2 AP
priority: -2
タグ:     [デッキ操作, 発掘, 遺物]

効果:
  effectType: Composite
  children:
    ── 基本: 発掘2 ──
    - effectType: DeckManipulate
      deckManipulateParams:
        action: excavate           （発掘）
        count: 2                   （上から2枚確認）
        filterTags: ["遺物"]       （遺物タグを持つカードを手札へ）
        remainderAction: deckBottom（残りはデッキの一番下へ）

    ── ①追加(橙-3): 追加発掘2 ──
    - effectType: Conditional
      condition: ElementThreshold(orange, >=, 3)
      thenEffect:
        effectType: Composite
        children:
          - effectType: ElementConsume { element: orange, amount: 3 }
          - effectType: DeckManipulate
            deckManipulateParams:
              action: excavate
              count: 2
              filterTags: ["遺物"]
              remainderAction: deckBottom

テキスト:
  【発掘:2】を行う。
  ①追加(橙-3): さらに【発掘:2】を行う。

  ※【発掘:X】→ デッキの上からX枚確認し、
    【遺物】タグを持つカードをすべて手札に加える。
    手札に加えなかったカードはデッキの一番下に戻す。
────────────────────────────────────────
```

### 巫術・扇風乱舞

```
────────────────────────────────────────
ID:       skill_purple_fan_dance
名前:     巫術・扇風乱舞
色:       紫
種別:     スキル
コスト:   4 AP
priority: +1
タグ:     [妨害, バウンス, 攻撃]

効果:
  effectType: Composite
  children:
    ── バウンス ──
    - effectType: Bounce
      bounceParams:
        target: { mode: enemyField }   （敵の召喚ユニット/オブジェクト/アイテムから）
        count: 2
        selection: random
        destination: hand              （相手の手札に戻す）
        trackCost: true                （戻したカードのコスト合計を記録）

    ── コスト連動ダメージ ──
    - effectType: Damage
      damageParams:
        damageType: physical
        cardMultiplier: 0             （固定値ではなくコスト連動）
        fixedDamage: bouncedCostSum * 0.1 * attackerPAtk
        ※ 戻したカードのコスト合計 × 10% × pAtk
      target: { mode: allEnemies }

テキスト:
  相手フィールドの召喚ユニット・召喚オブジェクト・アイテムからランダムに2つを
  手札に戻す。戻したカードのコスト合計×10%のpAtkダメージを相手チーム全体に与える。

備考:
  - 「戻したカードのコスト」を参照する処理は、
    効果解決中のコンテキスト変数として保持する設計が必要
  - 戻す対象がいない場合、ダメージは0（不発ではない）
  - 高コスト召喚を戻すほどダメージが大きくなる報酬設計
────────────────────────────────────────
```

### 巫術・輝石の盾

```
────────────────────────────────────────
ID:       skill_yellow_jewel_shield
名前:     巫術・輝石の盾
色:       黄
種別:     スキル
コスト:   3 AP
priority: +2
タグ:     [防御, シールド, デバフ]

効果:
  effectType: Composite
  children:
    - effectType: Shield
      shieldParams:
        shieldType: maxHpPercent
        value: 0.8                 （現在のmaxHPの80%）
        duration: 2
      target: { mode: self }
    - effectType: StatusApply
      statusId: status_patk_down
      duration: 2
      overrideValue: -30%          （pAtk -30%。multiplicative）
      target: { mode: allEnemies }

テキスト:
  自身のmaxHPの80%のシールドを得る（2ターン）。
  相手チーム全体のpAtkを30%ダウン（2ターン）。

── 数値例（HP1200のキャラ） ──
  シールド量: floor(1200 × 0.8) = 960
  ※ 非常に強力なシールドだが、コスト3AP + priority+2 で早期発動
────────────────────────────────────────
```

---

## 追加カード（同系統の展開）

上記のメカニクスを応用した追加カード。

### 情報戦系（青）

```
────────────────────────────────────────
ID:       skill_blue_misinformation
名前:     偽情報
色:       青
種別:     スキル
コスト:   2 AP
priority: +1
タグ:     [妨害, 手札操作]

効果:
  effectType: Composite
  children:
    - effectType: Discard
      discardParams:
        count: 1
        mode: random
        target: { mode: enemyTeam }   （相手の手札から）
        destination: graveyard
    - effectType: Draw
      drawParams:
        count: 1
      target: { mode: self }

テキスト: 相手の手札からランダムに1枚をトラッシュへ送る。自分はカードを1枚ドローする。
────────────────────────────────────────
```

### 段階強化系（橙）

```
────────────────────────────────────────
ID:       skill_orange_chain_blast
名前:     連鎖爆破
色:       橙
種別:     スキル
コスト:   3 AP
priority: -1
タグ:     [攻撃, 魔法, 段階強化]

効果:
  effectType: Composite
  children:
    ── 基本: 1体に魔法ダメージ ──
    - effectType: Damage
      damageParams:
        damageType: magical
        cardMultiplier: 0.8
      target: { mode: enemyCharacter }

    ── ①追加(橙-1): 対象追加 ──
    - effectType: Conditional
      condition: ElementThreshold(orange, >=, 1)
      thenEffect:
        effectType: Composite
        children:
          - effectType: ElementConsume { element: orange, amount: 1 }
          - effectType: Damage
            damageParams: { damageType: magical, cardMultiplier: 0.8 }
            target: { mode: enemyCharacter }  （別の敵1体）

    ── ②追加(橙-3): 全体化 ──
    - effectType: Conditional
      condition: ElementThreshold(orange, >=, 3)
      thenEffect:
        effectType: Composite
        children:
          - effectType: ElementConsume { element: orange, amount: 3 }
          - effectType: Damage
            damageParams: { damageType: magical, cardMultiplier: 0.6 }
            target: { mode: allEnemies }

テキスト:
  敵1体に魔法ダメージ（mAtk×0.8）。
  ①追加(橙-1): さらに別の敵1体に同じダメージ。
  ②追加(橙-3): 敵全体に魔法ダメージ（mAtk×0.6）。
────────────────────────────────────────
```

### 遺物カード（橙 — 発掘で手札に入る）

```
────────────────────────────────────────
ID:       skill_orange_ancient_blade
名前:     古代の刃
色:       橙
種別:     スキル
コスト:   1 AP
priority: 0
タグ:     [攻撃, 物理, 遺物]

効果:
  effectType: Damage
  damageParams:
    damageType: physical
    cardMultiplier: 1.2
  target: { mode: enemyCharacter }

セット効果:
  effectType: StatusApply
  statusId: status_patk_up_loadout
  duration: -1
  overrideValue: 8（pAtk +8）

テキスト: 敵1体に物理ダメージ（pAtk×1.2）を与える。
備考: 低コスト高倍率だが【遺物】タグのため通常ドローでは引きにくい設計を想定。
      【発掘】でサーチすることで安定して手札に入る。
────────────────────────────────────────
```

### 犠牲系（紫）

```
────────────────────────────────────────
ID:       skill_purple_life_drain
名前:     命の収奪
色:       紫
種別:     スキル
コスト:   2 AP
priority: 0
タグ:     [攻撃, 魔法, 吸収, 自傷]

効果:
  effectType: Composite
  children:
    - effectType: Damage
      damageParams:
        damageType: true          （真ダメージ）
        fixedDamage: null
        cardMultiplier: 0.1       （自身のmaxHPの10%を自傷）
        atkStat: null
      target: { mode: self }      ← 自傷
    - effectType: Damage
      damageParams:
        damageType: magical
        cardMultiplier: 1.4
      target: { mode: enemyCharacter }
    - effectType: Heal
      healParams:
        calcMode: fixed
        value: 0                  （ダメージの30%を回復 — 要: damageRef）
      ※ 与えたダメージの30%回復は新メカニクス（吸収）。
        Heal の value を「直前のダメージ結果の30%」にする仕組みが必要

テキスト: 自身のmaxHPの10%を失う。敵1体に魔法ダメージ（mAtk×1.4）を与え、
          与えたダメージの30%を回復する。
────────────────────────────────────────
```

### 支援系（緑）

```
────────────────────────────────────────
ID:       skill_green_natures_gift
名前:     自然の恵み
色:       緑
種別:     スキル
コスト:   3 AP
priority: +1
タグ:     [回復, 全体, バフ]

効果:
  effectType: Composite
  children:
    - effectType: Heal
      healParams:
        calcMode: maxHpPercent
        value: 0.15
      target: { mode: allAllies }
    - effectType: StatusApply
      statusId: status_pdef_up
      duration: 2
      overrideValue: 20
      target: { mode: allAllies }

テキスト: 味方全体のHPをmaxHPの15%回復し、pDef+20（2ターン）を付与する。
────────────────────────────────────────
```

---

## 新メカニクスのまとめ（要件定義への反映候補）

| メカニクス | 新EffectType | 影響範囲 | 優先度 |
|---|---|---|---|
| **Discard（手札捨て）** | `Discard` | 手札操作全般 | 高 |
| **Bounce（バウンス）** | `Bounce` | 召喚/アイテムの除去手段 | 高 |
| **DeckManipulate（デッキ操作）** | `DeckManipulate` | ボトムドロー、発掘 | 中 |
| **TokenCreate（トークン生成）** | `TokenCreate` | 一時カード生成 | 中 |
| **Reveal（手札公開）** | `Reveal` | 情報アドバンテージ | 低 |
| **DeckSearch（デッキサーチ）** | `DeckSearch` | 相棒コール等 | 中 |
| **吸収（ダメージ参照回復）** | 既存Healの拡張 | 紫の自傷+吸収 | 中 |
| **可変コスト（X AP）** | コストシステムの拡張 | 相棒コール等 | 低 |
| **段階追加効果** | 既存Conditionalで表現可能 | 橙の強化分岐 | — |
| **キーワード能力（発掘等）** | 共通定義の参照 | カードテキスト簡略化 | 低 |

「段階追加効果」は既存の `Conditional + ElementConsume` で表現可能なため、新EffectTypeは不要です。
