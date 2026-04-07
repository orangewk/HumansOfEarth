# Globe.GL CDN 互換性問題 — 調査・修正ログ

**作成日**: 2026-04-07  
**ステータス**: 未解決（調査継続中）  
**リポジトリ**: https://github.com/orangewk/HumansOfEarth

---

## 概要

`index.html` で読み込んでいる Three.js / globe.gl の CDN 互換性エラーにより、
アプリケーションが起動しない問題。

---

## 時系列ログ

### 【2026-04-07 21:29】 初回コミット（既存状態）

コミット: `78ea6e8`
```
fix: CDN バージョン固定 — three@0.152.2 + globe.gl@0.8.5
```

この時点でのCDN指定:
```html
<script src="https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/globe.gl@0.8.5/dist/globe.gl.min.js"></script>
```

---

### 【セッション 1 — ユーザー初期報告】

**ユーザー報告**:
- 年代を進めると年号表示が `NaN` になる
- 移民表示がうまく動いていない
- `E:\orange-wks\HumansOfEarth` を見てほしい

#### 修正 1: NaN 年号バグ

**原因**: `formatYear(yr)` にバリデーションがなく、`TIMELINE` 配列の範囲外アクセスで `undefined` が渡されると `Math.round(undefined) → NaN` になる。

**修正**:
```javascript
function formatYear(yr) {
  if (typeof yr !== 'number' || isNaN(yr)) return '—';
  yr = Math.round(yr);
  // ...既存の処理
}
```

#### 移民表示仕様確認

移民表示機能はすでに実装済み:
- `showMigration` フラグでトグル
- 2024年以降の未来の年でのみ有効（過去ではグレーアウト）
- 送出率3%、距離減衰、言語ボーナス、GDP引力でモデル化
- 上位25フローをArcアニメーションで表示

#### 確認結果

- ✅ NaN バグ修正完了
- ✅ 移民表示は仕様通り実装済み
- ❌ **この時点でCDNエラーは未対応のまま**

---

### 【セッション 2 — CDN エラー報告】

**ユーザー報告（コンソールエラー）**:
```
three.min.js:1 Scripts "build/three.js" and "build/three.min.js" are deprecated
  with r150+, and will be removed with r160.

（インデックス）:241 Uncaught ReferenceError: Globe is not defined
  at （インデックス）:241:15
  at （インデックス）:1088:3
```

#### 試行 1: Three.js バージョン更新 + CDN 変更

```html
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
<script src="https://unpkg.com/globe.gl"></script>
```

**結果**: ❌ 改善せず。さらにエラー追加:
```
THREE.WARNING: Multiple instances of Three.js being imported.
z0.Timer is not a constructor
```

---

### 【セッション 3 — 計画ドキュメント要求】

**ユーザー要求**: 「まず実装ではなく計画を書いてください」

#### 作成ドキュメント

`docs/planning/2026-04-07-globe-cdn-fix-plan.md`

3つの修正案を提示:
- **案A**: Three.js r149 にダウングレード（最小変更）
- **案B**: ES Modules 版 globe.gl に移行（中規模）
- **案C**: globe.gl を最新ベータ版にアップグレード

**推奨**: 案A

---

### 【セッション 4 — 調査と図入りドキュメント要求】

**ユーザー要求**:
1. 「エラー原因を調査してください」
2. 「mermaid図、そしてドキュメントにしてください」
3. 「エラー対策計画を書いて、図を多めに」

#### 調査結果（web_fetch）

| 確認項目 | 結果 |
|---------|------|
| globe.gl 依存 three.js（最新版） | `>=0.179 <1` |
| globe.gl 最新バージョン | `2.45.3` |
| `globe.gl@0.8.5` package.json | **404 Not Found** |
| THREE.Timer の存在（r149） | **存在しない** |
| THREE.Timer の削除（r150） | **確認不能** |
| three-globe の依存に 3d-tiles-renderer | **含まれていない** |

#### ハルシネーションの発生

以下の記述に根拠なし:
- ❌ 「Timer は r150 で削除された」
- ❌ 「r149 にダウングレードすれば動く」
- ❌ 「3d-tiles-renderer が Timer を使っている」

→ ユーザーより指摘を受け、事実のみの整理に切り替え

---

## 現在の状態（2026-04-07 時点）

### 未解決のエラー

| # | エラー | 状態 |
|---|--------|------|
| 1 | `THREE.Timer is not a constructor` | ❌ 未解決 |
| 2 | `Globe is not defined` | ❌ 未解決（#1 の結果） |
| 3 | `Multiple instances of Three.js being imported` | ❌ 未解決 |
| 4 | `Scripts "build/three.min.js" are deprecated with r150+` | ⚠️ 警告 |

### 修正済み

| # | 修正 | 状態 |
|---|------|------|
| 1 | `formatYear()` バリデーション追加 | ✅ 完了（コミット済み） |
| 2 | CDN URL 変更（複数回試行） | ❌ 効果なし |

### index.html の現在のCDN指定

```html
<script src="https://cdn.jsdelivr.net/npm/three@0.149.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/globe.gl@0.8.5/dist/globe.gl.min.js"></script>
```

（※ これはローカル未コミット状態。コミットしていない可能性あり）

---

## 次のセッションへの引き継ぎメモ

### やったこと
- NaN バグ修正（`formatYear` にバリデーション追加）
- 移民表示の確認（仕様通り実装済み）
- CDN の複数回変更（いずれも効果なし）
- 調査ドキュメント作成（ハルシネーション含む）

### やること（未着手）
- ✅ globe.gl の実コードを読んで `Timer` の呼び出し元を特定する
- ✅ globe.gl の正しいバージョンを特定する（`0.8.5` が存在するか？）
- ✅ 動作する CDN 組み合わせを特定する
- ✅ テスト

### 重要な制約
- グローバル `THREE` が必要（`THREE.BufferGeometry` 等を直接使用）
- グローバル `Globe` が必要（`new Globe(...)` を使用）
- ES Modules 移行は大規模（1000行超の書き直し）

### 参考ファイル
- `index.html` — メインのHTML（1094行）
- `docs/planning/2026-04-07-globe-cdn-fix-plan.md` — 修正案ドキュメント
- `docs/SPEC.md` — 既存の仕様書
