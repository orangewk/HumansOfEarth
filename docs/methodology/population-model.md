# 人口モデル — 方法論

> 作成日: 2026-03-28
> English version: [en/population-model.md](en/population-model.md)

## 1. 概要

2024〜2100年の国別人口を、1年刻みの離散的な人口動態更新で推計する。駆動変数は **TFR（合計特殊出生率）** で、3つのシナリオに応じて将来の軌道が決まる。平均寿命・乳幼児死亡率・初期年齢構成は2023年値で固定。

**このモデルは思考実験であり予測ではない。** 「現在の出生率トレンドが回復なしに続いたらどうなるか」を問う。UN 中位推計（TFR が置換水準に収束する前提）は意図的に使用していない。

## 2. 入力パラメータ

`pop_model_data.json` の各国エントリ:

| フィールド | 説明 | 出所 |
|-----------|------|------|
| `t` | TFR 2023 (出生/女性) | OWID indicator 950920 |
| `s` | TFR 年間勾配 (2016-2023 線形回帰) | 同上から算出 |
| `l` | 平均寿命 2023 (年) | OWID indicator 1118466 |
| `i` | 乳幼児死亡率 2023 (分率、元データは per 100 live births) | OWID indicator 1027774 |
| `p` | 人口 2023 | OWID historical population |
| `y0` | 0-14歳人口比率 2023 | World Bank SP.POP.0014.TO.ZS |
| `e0` | 65歳以上人口比率 2023 | World Bank SP.POP.65UP.TO.ZS |

## 3. 人口推計

### 3.1 置換水準ベースの出生率

```
deathRate = 1 / le
birthRate = deathRate × (yearTFR / 2.1) × (1 − im)
pop(t+1)  = pop(t) × (1 + birthRate − deathRate)
```

- `2.1` = 置換水準TFR（人口が世代交代で維持される値）
- TFR = 2.1 → birthRate ≈ deathRate（人口安定）
- TFR < 2.1 → 人口減少、TFR > 2.1 → 人口増加

### 3.2 TFR シナリオ

| シナリオ | 数式 | 説明 |
|---------|------|------|
| **Linear** | `max(0, t + s × (yr − 2023))` | 2016-2023の線形トレンドを継続。床=0 |
| **Stable** | `t` | TFR を2023年値で固定 |
| **Soft** | `max(0.8, t + s × elapsed × exp(−0.03 × elapsed))` | 減衰が指数的に鈍化。床=0.8 |

### 3.3 人口キャップ

```
popCap = p × 3
pop(t+1) = max(0, min(pop(t+1), popCap))
```

一律「現在人口の3倍」をキャップとする。§6.2 で経緯を記述。

## 4. 3層コホートモデル

年齢色変え（elderlyPct → ドット色）のために使用。人口計算本体にはフィードバックしない。

初期化（2023年データ）:
```
young   = p × y0
working = p × (1 − y0 − e0)
elderly = p × e0
```

年次更新:
```
births      = (working + young × 0.3) × deathRate × fertilityRatio × (1 − im)
youngToWork = young / 15      // 15年で労働層へ
workToElder = working / 50    // 50年で高齢層へ

youngDeaths  = young   × deathRate × 0.3
workDeaths   = working × deathRate × 0.5
elderDeaths  = elderly × deathRate × 2.5

young   += births − youngToWork − youngDeaths
working += youngToWork − workToElder − workDeaths
elderly += workToElder − elderDeaths
```

死亡率係数 (0.3, 0.5, 2.5) はヒューリスティック値であり、生命表から較正していない。

## 5. 既知の限界

1. **TFR 線形外挿は予測ではない** — 歴史上 TFR 回復のケースは存在する
2. **年齢構造の簡略化** — 3層のみ、年齢別死亡率テーブルなし
3. **移民を考慮していない** — 全シナリオで閉鎖人口
4. **LE・IM は2023年固定** — 将来の医療改善を想定しない
5. **戦争・パンデミック・技術革新は未考慮**
6. **`1/le` は粗い近似** — 年齢一様な安定人口を仮定

## 6. 設計判断

### 6.1 UN 中位推計を使わない理由

UN 中位推計は「TFR が最終的に置換水準に回帰する」前提を置く。このプロジェクトは「回帰しなかったら？」を問うものなので、UN 推計を使うと問い自体と矛盾する。

### 6.2 人口キャップの変遷

1. **農業収容力キャップ**: 耕作可能面積 × ルワンダ密度(1,130人/km²) → エジプト(砂漠国だが貿易で食料調達、実人口1.15億)で破綻
2. **工業国ホワイトリスト**: World Bank 高所得+上位中所得を exempt → バングラデシュ(農業国だが稲作で密度超高)の分類困難
3. **一律3倍（現行）**: シンプルで全ケースに適用可能。理論的根拠はなく、ヒューリスティック

### 6.3 置換水準モデルへの移行

旧モデル: `birthRate = TFR / 30 × 0.5 × (1 − im)`
- バグ: TFR < 2.1 でも birthRate > deathRate となり人口が増加（日本 TFR=1.208 で年+0.83%増）
- 原因: 全人口に対して出生率を適用しており、出産可能年齢人口の比率を考慮していない
- 修正: 置換水準(2.1)との比で出生率を調整 → TFR=2.1 で安定、それ以下で減少
