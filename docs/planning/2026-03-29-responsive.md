# 実装計画: レスポンシブ対応・スマホ対応

## 要件

Humans of Earth の UI がスマホ・縦長画面で使えるようにする。

### 現状の問題
1. **スライダー幅が固定** (`width: 300px`) — 狭い画面ではみ出す
2. **フッタの横並びが崩れる** — 年表示 + コントロール + 人口が1行に収まらない
3. **設定パネルが右下固定** (`right: 20px`) — 狭い画面で地球を隠す
4. **タイトルの文字サイズ** — スマホで大きすぎる
5. **ボタンのタッチターゲット** — 36px は指で押しにくい（推奨48px）
6. **ツールチップ (data-tip)** — hover ベースなのでタッチデバイスで見えない
7. **Globe.GL のタッチ操作** — ピンチズーム等は Globe.GL が対応済み

### ブレークポイント方針
- **デスクトップ** (≥768px): 現状維持
- **モバイル** (<768px): フッタを複数行に、スライダーをフル幅に

## 実装フェーズ

### Phase 1: CSS メディアクエリ
- スライダー: `width: 300px` → `flex: 1; min-width: 80px;`（既に `style` 属性で `flex:1` が指定されてるが CSS 側の `width:300px` が競合）
- フッタ: 768px 未満で `flex-wrap: wrap` + 年表示を上段・スライダーをフル幅に
- タイトル: スマホで `font-size: 0.9rem`
- ボタン: スマホで `min-width: 44px; min-height: 44px`

### Phase 2: 設定パネルのモバイル対応
- 768px 未満: 右下固定 → 画面下からスライドアップするシート
- またはフッタの上に重ねる形式

### Phase 3: ツールチップ代替
- タッチデバイスでは hover ツールチップが出ない
- シナリオボタンのタップで info-label に説明を表示する（既に実装済みの可能性）

### Phase 4: 動作確認
- Chrome DevTools のデバイスエミュレーションで確認
- iPhone SE (375px), iPhone 14 (390px), iPad (768px) で検証

## 影響範囲
- `index.html` の `<style>` セクション
- 一部 HTML 構造の微調整（フッタの行分割）

## リスク

| リスク | 深刻度 | 対策 |
|--------|--------|------|
| CSS 変更がデスクトップ表示を崩す | MEDIUM | メディアクエリで分離、デスクトップは現状維持 |
| Globe.GL のタッチ操作との干渉 | LOW | Globe.GL は OrbitControls でタッチ対応済み |
| 設定パネルのスライドアップが複雑 | LOW | 最初は単純な全幅表示で十分 |

## Globe.GL リサイズ制御（調査結果 2026-03-29）

### 発見
- Globe.GL は **auto-resize をしない**（window resize listener なし）
- 初期化時に `window.innerWidth` / `window.innerHeight` をデフォルト取得し、以後固定
- `world.width(n)` / `world.height(n)` で明示的にサイズ変更可能（chainable setter）
- 内部で `renderer.setSize()` + `camera.aspect` 更新 + CSS サイズ設定を行う

### 失敗した方法
1. CSS `min-height` on body/html → Globe.GL が無視（内部で `window.innerHeight` を使用）
2. CSS `min-height` on `#globeViz` → Globe.GL が container サイズを上書き
3. JS で `el.style.height` を設定 → Globe.GL が独自にサイズ管理

### 正解: Globe.GL の `width()` / `height()` API を使う
```javascript
window.addEventListener('resize', function() {
  var w = Math.max(320, window.innerWidth);
  var h = Math.max(500, window.innerHeight);
  world.width(w).height(h);
  document.body.style.overflowX = (window.innerWidth < 320) ? 'auto' : 'hidden';
  document.body.style.overflowY = (window.innerHeight < 500) ? 'auto' : 'hidden';
});
```

## 複雑度: 低
