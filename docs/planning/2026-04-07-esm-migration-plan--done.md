# ES Modules 移行計画

**日付**: 2026-04-07
**ステータス**: proposed
**前提**: Issue #6（CDN互換性問題）の根本対応

---

## 背景

### なぜ動かなくなったか

```
元のコード（バージョン指定なし）
  three/build/three.min.js    ← CDN latest を取得
  globe.gl                    ← CDN latest を取得

Three.js r160 で build/three.min.js（UMD）を削除
  → CDN latest が 0.183.2 に進行 → three.min.js が 404
  → window.THREE が undefined

前任が globe.gl@0.8.5 を指定（npm に存在しないバージョン）
  → globe.gl も 404 → window.Globe が undefined
```

### なぜ ES Modules か

Three.js は r150 で UMD ビルドを非推奨、r160 で削除した。
公式の推奨移行先は **`<script type="importmap">` + `<script type="module">`**。
（参照: https://threejs.org/manual/en/installation.html）

r159（UMD最終版）に固定する暫定案もあるが、Three.js / globe.gl 両方の
アップデートパスが閉じるため、本来の修正（ESM化）を選択する。

---

## 変更の範囲

### 変更するもの

| 対象 | 変更内容 |
|------|---------|
| `index.html` L110-111 | `<script>` 2行 → `<script type="importmap">` + `<script type="module">` |
| `index.html` L167-1091 | IIFE 内の先頭に `import` 文を追加。IIFE → module スコープに変更 |

### 変更しないもの

| 対象 | 理由 |
|------|------|
| ロジック全体 | import 追加のみ。計算・描画・UI ロジックは一切変更しない |
| `THREE.xxx` の呼び出し | `import * as THREE from 'three'` で同名に束縛 |
| `Globe(...)` の呼び出し | `import Globe from 'globe.gl'` で同名に束縛 |
| HTML/CSS | 変更なし |
| データファイル (.json, .csv, .jpg) | 変更なし |

---

## ターゲットバージョン

| パッケージ | バージョン | 根拠 |
|-----------|-----------|------|
| `three` | **0.183.0** | 最新安定版（0.183.2 でも可）。ESM ビルド `build/three.module.js` を使用 |
| `globe.gl` | **2.45.3** | 最新安定版。ESM ビルド `dist/globe.gl.mjs`（28KB、Three.js 非バンドル）。`three >=0.179 <1` を要求 |

---

## 具体的な差分

### Step 1: CDN 読み込みの置換（`<head>` 内）

```diff
- <script src="https://cdn.jsdelivr.net/npm/three@0.149.0/build/three.min.js"></script>
- <script src="https://cdn.jsdelivr.net/npm/globe.gl@0.8.5/dist/globe.gl.min.js"></script>
+ <script type="importmap">
+ {
+   "imports": {
+     "three": "https://cdn.jsdelivr.net/npm/three@0.183.0/build/three.module.js",
+     "globe.gl": "https://cdn.jsdelivr.net/npm/globe.gl@2.45.3/dist/globe.gl.mjs"
+   }
+ }
+ </script>
```

### Step 2: インラインスクリプトの module 化

```diff
- <script>
- (function() {
+ <script type="module">
+ import * as THREE from 'three';
+ import Globe from 'globe.gl';
+
+ {
    // ── i18n ──
    var urlLang = ...
    ...（全ロジックそのまま）...
- })();
- </script>
+ }
+ </script>
```

**補足**: `<script type="module">` は自動的に strict mode かつモジュールスコープになるため、
IIFE の即時実行ラッパーは不要になる。ただし `var` 宣言のスコープ挙動を維持するため、
ブロック `{ }` で囲む（任意。module スコープだけでも十分だが安全策）。

### Step 3: 確認が必要な点

| 項目 | 懸念 | 対応 |
|------|------|------|
| `<script type="module">` は defer 的に動作する | DOM 要素への参照（`getElementById` 等）は `<body>` 末尾にあるため問題なし | 確認のみ |
| globe.gl ESM が `Globe` コンストラクタを default export するか | npm メタデータで確認済み: `"exports": { "default": "./dist/globe.gl.mjs" }` | 確認済み |
| Three.js ESM と globe.gl ESM で Three.js インスタンスが一致するか | importmap で同じ specifier `"three"` を使うため、ブラウザが同一モジュールを共有する。**Multiple instances 問題が解消される** | これが ESM 化の最大のメリット |
| `three-globe` の背景画像 URL | L243: `//cdn.jsdelivr.net/npm/three-globe/example/img/night-sky.png` — JS import ではなく画像 URL なので影響なし | 変更不要 |

---

## リスク

| リスク | 影響度 | 緩和策 |
|--------|--------|--------|
| globe.gl 2.45.3 の API に破壊的変更がある | 中 | globe.gl の Changelog を確認。使用 API（Globe コンストラクタ、arcsData、pointOfView 等）は v1 から安定している |
| importmap の [ブラウザ対応](https://caniuse.com/import-maps) | 低 | Chrome 89+, Firefox 108+, Safari 16.4+ で対応。2026年時点ではほぼ全ブラウザ対応 |
| three@0.183 で ShaderMaterial / BufferGeometry の API が変わっている | 低 | これらは Three.js のコア API。非推奨になっていないか確認する |

---

## 検証チェックリスト

- [ ] コンソールにエラーがないこと
- [ ] 「Multiple instances of Three.js」警告が出ないこと
- [ ] 地球儀が表示されること
- [ ] 背景の星空が表示されること
- [ ] ドット（人口光）が表示されること
- [ ] スライダーで年代変更が機能すること
- [ ] 古代（-10000〜1800）のドット wobble が動作すること
- [ ] 年齢色分けトグルが機能すること
- [ ] 移民フロー Arc が表示されること（2024年以降 + Migration ON）
- [ ] 再生/一時停止ボタンが機能すること
- [ ] キーボード操作（← → Space R）が機能すること
- [ ] モバイル表示が崩れないこと

---

## 作業見積もり

| 作業 | 内容 |
|------|------|
| コード変更 | `index.html` の `<head>` 3行置換 + `<script>` タグ属性変更 + import 2行追加 + IIFE 除去 |
| ロジック変更 | なし |
| テスト | ブラウザで上記チェックリストを手動確認 |
