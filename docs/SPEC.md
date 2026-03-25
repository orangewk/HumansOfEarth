# People of Earth — 現状の仕様書

## システム構成

```
┌─────────────────────────────────────────────────────┐
│                   index.html                         │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │ Globe.GL │  │PointCloud│  │HistoricalPointCloud│  │
│  │ (地球儀)  │  │(NASAドット)│  │(歴史ドット)        │  │
│  └──────────┘  └──────────┘  └───────────────────┘  │
│       ↑              ↑               ↑               │
│  earth_night   city_dots.json   historical_dots.json │
│  _base.jpg     + pop_model      + pop_history.json   │
│                _data.json       + timeline.json      │
└─────────────────────────────────────────────────────┘
```

## 2つの独立した描画システム

### システムA: NASAドット（PointCloud）
- **データ源**: `city_dots.json` — NASA夜景から抽出した都市光ピクセル (~40K dots)
- **用途**: 近現代 (1800〜2100) の表示
- **属性**: 位置(固定), brightness(NASAから), country, RGB色
- **バッファ**: base dots (40K) + growth slots (80K) = **120K 頂点**
- **制御変数**: `baseDotOpacity`, `era`, `ratio`, `brightness_factor`, `size_factor`

### システムB: 歴史ドット（HistoricalPointCloud）
- **データ源**: `historical_dots.json` — 1ドット=5万人で事前計算 (最大~19K dots/年)
- **用途**: 古代〜近代 (~10000 BC〜1800) の表示
- **属性**: 位置(年ごとに変わる), サイズ/色は全ドット同一
- **バッファ**: 全年の最大ドット数 (~19K 頂点)
- **制御変数**: `hcOpacity`, `era`

## 時代ごとの動作

```
      era=0                              era=0→1                    era=1
  ◄──────────────────►  ◄───────────────────────────►  ◄──────────────────►
  -10000            1800                             1950                2100
  │                  │                                │                   │
  │  歴史ドットのみ   │    歴史ドット(フェードアウト)      │  NASAドットのみ     │
  │  (赤い火, wobble)│    NASAドット(フェードイン)        │  (年齢色, 固定)     │
  │                  │                                │                   │
  │  baseDotOpacity  │    baseDotOpacity               │  baseDotOpacity    │
  │  = 0             │    = (year-1800)/150            │  = 1.0             │
  │                  │                                │                   │
  │  hcOpacity       │    hcOpacity                    │  hcOpacity         │
  │  = 0.8           │    = 0.8*(1-era)               │  = 0               │
```

## ★ 問題点

問題の性質を3段階に分類: **矛盾**(設計が自己矛盾), **非効率**(動くが無駄がある), **トレードオフ**(意図的だが改善余地あり)

### 非効率1: NASAドットの古代用ロジック

```
NASAドット更新ループ (L645-693):
  ├── density weight (countryWeight) ← 古代用に導入、era でフェードするが複雑
  ├── relative/absolute blend (L657-659) ← 古代用。era=1では無害だが死コード
  ├── sizeMultiplier = 1+(1-era)*0.8 (L671) ← 古代用。era=1では1.0で無害だが死コード
  ├── eraColor: fire→electric blend (L620-638) ← era<1でのみ発動。era=1では不要分岐
  └── dotVisible + dotRank (L674-683) ← 衰退表現。OK
```

NASAドットは 1800〜1950 の過渡期にも表示される（baseDotOpacity > 0）ので、fire→electric ブレンド等は過渡期の演出として意味がある。ただし era=0 (〜1800) では baseDotOpacity=0 で完全非表示なのに全ロジックが走る**非効率**。

### 非効率2: 成長スロットの VRAM 使用量

```
Base dots:    40,736
Growth slots: 81,472 (各国のドット数×2)
Total buffer: 122,208 頂点
```

- 2025年 (ratio ≈ 1.0) では成長スロットはほぼ全部 alpha=0
- setDrawRange で描画範囲を制限したが、バッファ自体が巨大
- setDrawRange で描画は制限されるが、VRAM に 120K 頂点分のバッファを確保している
- alpha=0 + additive blending での光漏れは setDrawRange により発生しない

### トレードオフ3: 歴史ドットが1800年で固定

```
historical_dots.json: 年キー = [-10000, ..., 1800]
timeline.json: [-10000, ..., 1800, 1801, 1802, ..., 2100]

1801〜1950:
  - histDots は 1800 にスナップ → 1800年のドットが150年間固定表示
  - hcOpacity が 0.8→0 にフェード → 見た目は消えていく
  - hcOpacity でフェードアウトするので視覚的には許容される設計上の妥協
  - ただし1801〜1949年のデータを生成すれば解消可能
```

### 矛盾4: 過渡期でNASAドットがほぼ見えない

```
未来 (2024+): ratio = computeState() → pop / pop_2023
過去 (〜2023): ratio = pop_history[year] / pop_model.p (= pop_2023)

→ 2025年の ratio ≈ 1.0（2年しか経ってないので）
→ 1800年の ratio ≈ world平均 0.13（2023年比で人口が少ない）
→ -10000年の ratio ≈ 0.00001

問題: baseDotOpacity が 1800〜1950 でフェードインするが、
      ratio は 0.13 → brightness_factor = 0.13^1.5 = 0.047
      → ほぼ見えない
      → 1950年に突然 ratio ≈ 0.8 くらいで明るくなる
```

### 矛盾5: countryWeight と relative blend の複雑な相互依存

```
L612: w = 1.0 + ((countryWeight - 1.0) * (1 - era))
L614: s.weightedRatio = s.ratio * w

L658: relativeRatio = ratio / maxWeightedRatio
L659: ratio = relativeRatio * (1-era) + ratio * era
```

- countryWeight は era でフェード
- relativeRatio も era でフェード
- **era の中間値（過渡期）で二重に補正がかかる**

### トレードオフ6: 2つのシステムの「人口の意味」が違う

```
システムA (NASA): 1ドット = 都市光ピクセル（明るさが人口の代理）
                  ドット数 ∝ 電化率（先進国が多い）
                  明るさ = ratio × power curve

システムB (歴史): 1ドット = 5万人（ドット数が人口そのもの）
                  ドット数 ∝ 人口
                  明るさ = 全ドット同一

過渡期 (1800-1950) で2つが重なるが、「人口の表し方」が根本的に違う。
NASAドットは「どこが光ってるか」、歴史ドットは「何人いるか」。
```

## データフロー図

```
updateYear(year) が呼ばれると:

  year → era 計算
       → baseDotOpacity 計算
       → hcOpacity 計算

  ┌─────────────── 各国ループ ───────────────┐
  │ getState(code, year)                      │
  │   year <= 2023: popHistory[year] / pop2023│
  │   year > 2023:  computeState(model, year) │
  │   → {ratio, elderlyPct}                  │
  │                                           │
  │ × countryWeight (era でフェード)           │
  │   → weightedRatio                        │
  └───────────────────────────────────────────┘

  ┌─────── NASA ドット (i = 0..N) ────────────┐
  │ ratio → relative/absolute blend            │
  │ ratio → brightness_factor (power curve)    │
  │ ratio → size_factor                        │
  │ era → sizeMultiplier                       │
  │ era → eraColor (fire or age)               │
  │ brightness_factor → dotVisible (rank cull)  │
  │ finalAlpha = brightness × baseDotOpacity   │
  └────────────────────────────────────────────┘

  ┌─────── 成長スロット (i = N..N+80K) ───────┐
  │ era < 1 → ratio = 0 → 全非表示            │
  │ era = 1 & ratio > 1 → growthFraction 分表示│
  └────────────────────────────────────────────┘

  ┌─────── 歴史ドット (別PointCloud) ──────────┐
  │ year → histDots[closest year] のドット配列  │
  │ 位置を全書き換え                            │
  │ hcOpacity → alpha (全ドット同一)            │
  │ era → 色 (fire or warm)                    │
  │ setDrawRange(0, nVisible)                  │
  └────────────────────────────────────────────┘
```

## 仕様書に未記載だった重要な動作

1. **popHistory の線形補間**: getState() 内で、指定年のデータがない場合に隣接年で線形補間（L276-289）
2. **elderlyPct の非対称性**: 1950年以前は固定値 0.05（warm）、1950年以降のみコホートモデルで計算（L294-298）
3. **stateCache のメモリ**: year × country × scenario でキャッシュが増え続ける。シナリオ変更時のみリセット
4. **computeState のループコスト**: 2100年だと2024年から77回のコホートシミュレーションループ（キャッシュで初回のみ）
5. **drawRange 計算の CPU コスト**: 毎フレーム全成長スロットを走査して最後尾を探す O(totalGrowthSlots)

## 推奨: 簡素化の方向性

1. **NASAドットから古代用ロジックを全削除** — era<1 では baseDotOpacity=0 で非表示なのだから不要
2. **成長スロットを廃止または大幅縮小** — 12万頂点バッファは過剰。必要時だけ動的生成するか、上限を制限
3. **countryWeight を廃止** — 統一ドットモデル導入で古代のバランス問題は解決済み。NASAドットに適用する意味がない
4. **relative/absolute blend を廃止** — 同上
5. **過渡期のロジックを明確化** — 歴史ドットのフェードアウトとNASAドットのフェードインの関係を整理
