# Globe.GL CDN互換性修正計画

**日付**: 2026-04-07  
**ステータス**: 計画中  
**優先度**: 高（アプリケーションが起動しない）

---

## 現状の問題

### エラー一覧

| # | エラー | 原因 |
|---|--------|------|
| 1 | `THREE.Timer is not a constructor` | `globe.gl@0.8.5` が Three.js の `Timer` クラスを参照。このクラスは r150+ で**削除済み** |
| 2 | `Globe is not defined` | 上記エラーで globe.gl の初期化が失敗 → `Globe` グローバルが生成されない |
| 3 | `Multiple instances of Three.js being imported` | CDN読み込みの重複（globe.glが内部にThree.jsを抱え込み、別途読み込んだThree.jsと衝突） |
| 4 | `Scripts "build/three.js" are deprecated with r150+` | Three.js r150+ はUMDビルドを非推奨に |

### 根本原因

**globe.gl@0.8.5** が Three.js r150+ と互換性がない。
- globe.gl 内部で `THREE.Timer` を使用（r150で削除）
- globe.gl のUMDビルドは Three.js を内包しており、別途 `<script>` で読み込むと重複する
- 結果: `Globe` がグローバルに公開されない

---

## 修正案（3つ）

### A. Three.js r149 にダウングレード（最小変更）

**内容**: Three.js を `0.149.0` に固定。globe.gl が依存する `Timer` が存在する最終版。

| メリット | デメリット |
|----------|------------|
| 変更はCDN URL 1行のみ | Three.js の非推奨警告は残る（r149では警告なし） |
| globe.gl@0.8.5 と確実に互換 | 将来的に globe.gl アップグレードが必要 |

**影響範囲**: `index.html` の `<script>` タグ 2行

---

### B. ES Modules 版 globe.gl に移行（中規模）

**内容**: `importmap` + ES Modules で Three.js と globe.gl を読み込み。

```html
<script type="importmap">
{ "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
    "globe.gl": "https://cdn.jsdelivr.net/npm/globe.gl@0.8.5/dist/globe.gl.mjs"
}}
</script>
<script type="module">
  import Globe from 'globe.gl';
  import * as THREE from 'three';
  // ...
</script>
```

| メリット | デメリット |
|----------|------------|
| Three.js最新版が使える | 全コードを `<script type="module">` に書き直す必要がある |
| 重複インスタンス問題が解消 | `window.Globe` / `window.THREE` 依存箇所をすべて修正 |
| 将来のアップグレードが容易 | テスト工数が大きい |

**影響範囲**: `index.html` 全体（1000行超）のモジュール化

---

### C. globe.gl を最新ベータ版にアップグレード（調査が必要）

**内容**: globe.gl の最新バージョン（r150+対応版）があれば使用。

| メリット | デメリット |
|----------|------------|
| 最新のThree.jsと互換 | 最新ベータの安定性が不明 |
| UMD非推奨問題も解消 | 既存APIの破壊的変更がある可能性 |

**要確認**: globe.gl の最新リリースノートで r150+ 対応状況

---

## 推奨: 案A（Three.js r149 ダウングレード）

**理由**:
1. **影響範囲最小**: CDN URL 2行のみの変更
2. **即時動作確認可能**: 既存コードの修正不要
3. **機能に影響なし**: Three.js r149 で現在使っているAPIはすべて動作
4. **将来の移行パス**: 案B/C は別途Issueで計画的に実施

### 実施内容

```diff
- <script src="https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.min.js"></script>
- <script src="https://cdn.jsdelivr.net/npm/globe.gl@0.8.5/dist/globe.gl.min.js"></script>
+ <script src="https://cdn.jsdelivr.net/npm/three@0.149.0/build/three.min.js"></script>
+ <script src="https://cdn.jsdelivr.net/npm/globe.gl@0.8.5/dist/globe.gl.min.js"></script>
```

- Three.js 単独で読み込む（globe.gl に内包させない）
- 両方とも `jsdelivr` で統一（CDN混在による問題を回避）

### 検証項目

- [ ] `Globe is not defined` エラーが解消
- [ ] `THREE.Timer is not a constructor` エラーが解消
- [ ] `Multiple instances of Three.js` 警告が解消
- [ ] 地球儀が正常に表示される
- [ ] スライダーで年代変更可能
- [ ] 移民フローArcが表示される
- [ ] 年号表示がNaNにならない（前回修正済み）

---

## 今後の課題（別Issue）

| 課題 | 優先度 | 備考 |
|------|--------|------|
| ES Modules 移行（案B） | 低 | 大規模リファクタリング |
| globe.gl 最新版への追随 | 中 | 新バージョンの安定性確認後 |
| deprecated 警告の完全解消 | 低 | Three.js がUMD完全削除(r160)まで猶予あり |
