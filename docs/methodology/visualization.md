# 可視化 — 技術的方法論

> 作成日: 2026-03-28
> English version: [en/visualization.md](en/visualization.md)

## 1. 都市光ドットの生成

### 1.1 光ピクセル抽出

NASA 夜景画像 (`earth-night.jpg`, 4096×2048 equirectangular) から都市光ピクセルを抽出。

フィルタ条件:
```python
city_mask = (
    (brightness > 50) &          # 十分な明るさ
    (R > B * 0.7) &               # 赤が青より優位（雪原・氷を除外）
    (abs(R - B) < 40)             # R≈G≈B の白い光（オレンジ砂漠を除外）
)
```
brightness = (R + G + B) / 3

結果: 31,089ピクセルを都市光ドットとして抽出。

### 1.2 国割り当て

GeoJSON (Natural Earth) のポリゴンで point-in-polygon 判定。Shapely STRtree で空間インデックス。

- 割り当て成功: 28,387 / 31,089 (91.3%)
- 未割り当て: 2,702 → 最寄り国に割り当て（buffer=2°で最近傍ポリゴンを探索）

### 1.3 海上ドット除去

GeoJSON 陸地ポリゴン内にないドットを除去。

- 除去: 2,726ドット（漁火、油田プラットフォーム、船舶の光）
- 残存: 39,786ドット（補正後の最終値、§1.4参照）

### 1.4 発展途上国ドット補正

NASA 夜景は電力インフラに依存するため、発展途上国のドット数が人口に比して過少。

補正基準:
- グローバル平均: 285K人/dot
- 3倍以上不足 かつ 50ドット以上不足 の国を対象
- GeoJSON ポリゴン内にランダム配置（既存ドット近傍70% + ランダム30%）
- 補完ドットの brightness: 35-60（低め、非電化地域を表す）

主な補正: IND 371→5,046 / NGA 37→799 / BGD 15→601 / ETH 13→451

### 1.5 密度正規化 (countryWeight)

ドット数/人口の国間格差を補正する重み:

```javascript
globalAvgPPD = totalPop / totalDots  // グローバル平均 pop-per-dot
countryWeight[code] = sqrt(ppd / globalAvgPPD)
```

sqrt を使用して極端な値を抑制。結果例:
- JPN (943dots/124M): weight ≈ 0.8 (過剰表現を抑制)
- IND (5046dots/1438M): weight ≈ 1.2 (適正化)
- NGA (799dots/228M): weight ≈ 1.3

この重みは `updateYear()` で ratio に乗算される。

## 2. 描画

### 2.1 THREE.Points + カスタム ShaderMaterial

42K+ ドットを 1 draw call で描画。バッファに成長スロット（既存ドット数×2）を事前確保し、合計約127Kスロット。

Vertex Shader:
```glsl
attribute vec3 aColor;
attribute float aSize;
attribute float aAlpha;
varying vec3 vColor;
varying float vAlpha;
void main() {
  vColor = aColor;
  vAlpha = aAlpha;
  vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
  gl_PointSize = clamp(aSize * (200.0 / -mvPosition.z), 1.0, 64.0);
  gl_Position = projectionMatrix * mvPosition;
}
```

Fragment Shader:
```glsl
varying vec3 vColor;
varying float vAlpha;
void main() {
  float d = length(gl_PointCoord - vec2(0.5));
  if (d > 0.5) discard;
  float glow = 1.0 - smoothstep(0.0, 0.5, d);
  glow = glow * glow;  // sharper falloff
  gl_FragColor = vec4(vColor * glow, glow * vAlpha);
}
```

- **Additive blending** (`THREE.AdditiveBlending`): 重なったドットの光が加算される
- **depthWrite: false**: 透明オブジェクトの描画順序問題を回避
- **gl_PointSize clamp**: 1.0〜64.0 でドライバ依存の上限を制御

### 2.2 暖色ティント

都市光ドットの色にオレンジ方向のティントを適用:
```javascript
R = min(1.0, originalR * 1.3)  // R を30%ブースト
G = originalG                   // G そのまま
B = originalB * 0.7              // B を30%カット
```

根拠: NASA 夜景の都市光は実際にはほぼニュートラルグレー（R≈G≈B）だが、暗い地球ベースのティール背景との対比でオレンジに見える効果を意図的に強調。比較テストで `warm core+glow r=6 ×2.5` がユーザー選好（2026-03-24）。

### 2.3 暗い地球ベース

Blue Marble 昼画像を夜景化:

```python
# 1. 彩度50%低下 (Purkinje効果)
night = 0.5 * day + 0.5 * luminance

# 2. ティールティント
R *= 0.4, G *= 0.5, B *= 1.0

# 3. 地形に応じた明暗
night = land_tint * (0.6 + 0.7 * luminance)  # 陸地
night = ocean_tint * (0.7 + 0.3 * luminance)  # 海洋

# 4. 氷雪ブースト
night[ice] *= 1.2
```

色調は NASA 夜景の南極暗部からサンプリング:
- 陸地: R≈8, G≈42, B≈62
- 海洋: R≈6, G≈32, B≈50

## 3. 時代別視覚効果

### 3.1 時代遷移変数

```javascript
if (year <= 1800) era = 0;                         // 古代: 火の光
else if (year <= 1950) era = (year - 1800) / 150;  // 過渡期: 0→1
else era = 1;                                       // 近現代: 電気光
```

### 3.2 古代の火の光 (era < 1)

色: 深い赤橙（焚火の色）
```javascript
fireR = min(1.0, origR * 1.2 + 0.3)
fireG = origR * 0.3
fireB = origR * 0.05
// era で電気光と線形補間
color = fire * (1 - era) + electric * era
```

サイズ: 古代ドットは大きくぼんやり
```javascript
sizeMultiplier = 1.0 + (1 - era) * 0.8  // 最大1.8倍
```

### 3.3 時代別人口正規化

- **古代 (year < 1950)**: 相対正規化。その年の最大人口国 = ratio 1.0。べき乗カーブなし（線形スケール）
- **近現代 (year >= 1950)**: 2023年人口比。ratio にべき乗カーブ適用

### 3.4 年齢色シフト (era = 1 のみ)

```javascript
function ageColorShift(origR, origG, origB, elderlyPct) {
  var t = clamp((elderlyPct - 0.05) / 0.35, 0, 1);  // 5%→0, 40%→1
  R = origR * (1 - t * 0.6);     // R を最大60%カット
  G = origG * (1 - t * 0.2);     // G を最大20%カット
  B = origB + (1 - origB) * t * 0.7;  // B を最大70%ブースト
}
```

結果: 若い国 → オレンジ（暖色）、高齢化した国 → 青白（寒色）

## 4. 人口変化の視覚表現

### 4.1 減少 (ratio < 1)

```javascript
brightness_factor = ratio ^ 1.5  // 急激に暗くなる
size_factor = ratio ^ 1.2        // サイズも縮小
```

ratio < 0.5 の場合、暗いドット（dotRank が低い）から消灯:
```javascript
keepFraction = brightness_factor * 2  // 0.5→全残存, 0→全消灯
keepCount = floor(total * keepFraction)
// dotRank < (total - keepCount) のドットは alpha=0
```

### 4.2 増加 (ratio > 1)

既存ドットの明るさ・サイズを微増:
```javascript
brightness_factor = min(2.0, ratio ^ 0.6)
size_factor = min(1.8, ratio ^ 0.4)
```

加えて、事前確保した成長スロットを有効化:
```javascript
growthFraction = min(1.0, (ratio - 1) / 2.0)  // ratio 1→3 で 0→1
nShow = floor(nSlots * growthFraction)
```

成長ドットは既存の明るいドット（上位1/3）の近傍にランダム配置（lat/lon ±0.4°）。

## 5. 歴史都市ドット

### 5.1 データ

Reba et al. (2016) の Chandler + Modelski データを統合した1,577都市。別の `THREE.Points` として描画。

### 5.2 人口 → 視覚マッピング

```javascript
function getHistCityPop(pops, year)  // 最近傍の年を線形補間

// Normalization: 同年の最大都市人口で正規化
norm = pop / maxCityPop  // 0..1
alpha = 0.3 + norm * 0.7
size = 2.0 + norm * 5.0
```

色は era に連動（古代: 赤橙、近代: 暖色白）。

## 6. 設計判断

### 6.1 テクスチャ焼き込み → リアルタイムドット

初期実装: 人口変化を NASA 夜景画像に焼き込み、年ごとの静止画テクスチャを生成 → Globe.GL で切り替え。

変更理由:
- ドットの移動（v2 移民フロー）に対応できない
- シナリオ切替が不可能（事前生成が必要）
- 年齢色変え等の属性追加が困難

### 6.2 Bloom（シャープコア + ガウシアングロー）の検討

静止画生成時に検討。ドットを点として打ち、ガウシアンブラーでグロー → シャープコア + グロー合成。

結果: warm bloom r=6 ×2.5 がユーザー選好。ただし Globe.GL のリアルタイム描画に移行したため、ポストプロセスとしてのブルームは不採用。代わりに fragment shader の glow falloff (`smoothstep`) で擬似的にグローを表現。

### 6.3 ドット: 消去 → 減光

初期: 人口減少で暗いドットから消去（alpha=0）。
変更: 明るさ・サイズで連続変化（消さない）。

理由: 将来の年齢構成色変えで、消えたドットには色属性を適用できない。全ドットを残すことで、任意の属性を後から追加可能。

例外: ratio が極端に低い場合 (brightness_factor < 0.5) は暗いドットから消灯する（古代の暗い地球を表現するため）。
