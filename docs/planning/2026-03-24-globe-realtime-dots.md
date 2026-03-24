# 実装計画: Globe.GL リアルタイムドット描画

## 要件

暗い地球ベーステクスチャの上に、31,000個の都市光ドットを THREE.Points でリアルタイム描画する。
- ドットは globe の子オブジェクトとしてズーム・回転に追従
- 年スライダーで人口変化に応じてドットが増減（暗い順に消灯）
- warm tint + glow シェーダー（additive blending）
- 地球はゆっくり自動回転

## Globe.GL 内部構造（調査結果）

```
THREE.Scene
├── Skysphere (background)
├── Lights
└── ThreeGlobe (extends THREE.Group)  ← ここの子にすれば追従する
    ├── Group (globeLayer: mesh + atmosphere)
    ├── Group (pointsLayer)
    ├── Group (customLayer)
    └── ... 他のレイヤー群
```

- Globe radius: `100` (Three.js world units)
- 座標変換: `phi = (90 - lat) * DEG2RAD`, `theta = (90 - lng) * DEG2RAD`
- CDN バージョン未固定 (`//cdn.jsdelivr.net/npm/globe.gl`) — API 挙動がバージョンで変わるリスク

## 現在の問題の原因分析

### 1. Loading dots... で止まる
- **真因未特定** — ブラウザ Console のエラーメッセージをまず確認する必要がある
- [推測] `.animateIn(false)` の追加が原因。CDN バージョンによって chainable setter として存在しない or `undefined` を返してチェーンが壊れる可能性
- 前バージョン（テクスチャ方式、`.animateIn` なし）は正常動作していた

### 2. 経度ずれ
- **原因**: 座標変換式が three-globe 内部と不一致だった（修正済みだが未確認）
- three-globe: `theta = (90 - lng) * DEG2RAD`
- X/Z 割り当ての一致も要確認

### 3. ズーム追従しない
- **原因**: `world.scene().add(pointCloud)` は Scene 直下に追加するので globe 回転と独立
- ThreeGlobe グループの子にすれば追従する

## 実装計画

### Phase 0: 安定起動点の確保
- `.animateIn(false)` を削除して**動いていたテクスチャ版**をベースにする
- globe + ベーステクスチャ + 自動回転 + UI が動く状態を確認
- **ここが通らなければ先に進まない**

### Phase 1: ドット描画（最小限）
- Phase 0 の動作コードに THREE.Points を追加
- ThreeGlobe グループを確実に取得して `.add(pointCloud)`:
  - `world.scene().traverse()` で ThreeGlobe を探す（type/children 数で判別）
  - 取得失敗なら `customLayerData` にフォールバック
- 座標変換: three-globe と同じ `polar2Cartesian` 式
- ShaderMaterial 注意点:
  - `attribute vec3 color;` を vertex shader で明示宣言（Three.js バージョン依存を回避）
  - `gl_PointSize` のクランプ（min/max で制限）
- **まず全ドット表示で位置・追従・見た目を確認**

### Phase 2: 年スライダー連動
- buffer attribute の alpha/size を直接更新（`needsUpdate = true`）
- `customThreeObjectUpdate` は使わない（直接更新の方がシンプル確実）
- 国ごとの visible count を年から計算、暗い順に消灯

### Phase 3: 仕上げ
- 自動回転速度調整
- UI ラベル更新
- `controls()` の null チェック

## リスク

| リスク | 深刻度 | 対策 |
|--------|--------|------|
| CDN バージョン未固定で API 挙動が変わる | HIGH | バージョンを固定（`globe.gl@2.45.1` 等） |
| ThreeGlobe グループの取得失敗 | MEDIUM | traverse + fallback to customLayerData |
| ShaderMaterial の color attribute 暗黙定義 | MEDIUM | vertex shader で明示宣言 |
| gl_PointSize のドライバ上限 | LOW | clamp(min, max) で制限 |
| controls() が null を返す | LOW | null チェック追加 |
| 31K 点の buffer update 負荷 | LOW | Float32Array ループのみ、問題なし |

## 複雑度: 低〜中
