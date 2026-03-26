# v2 移民フローシナリオ設計

> Issue #3 / 作成: 2026-03-26 / 更新: 2026-03-27

## 1. 要件の再確認

### 目的
2024〜2100年の人口シミュレーションに「移民フロー」を追加する。
余剰人口（自然増加分の一定割合）が、豊かで近い国に流れるモデル。

### 理論的裏付け
- **Ravenstein の移民法則 (1885)**: 大多数の移民は短距離、経済的動機が支配的
- **重力モデル**: Flow(A→B) ∝ Pop_A × Pop_B × GDP_B / Distance^β — 国連移民データとの相関が高い
- 言語・植民地関係を加えると「ナイジェリア→英国」等の長距離移民も再現できる

### 現状
- `computeFuturePop(m, year, scenario)` が**国ごと独立**に人口を計算
- 3シナリオ（Linear / Stable / Soft）は TFR の変化パターンだけが異なる
- ドット生成 `generateFutureDots(year, scenario)` は人口からドット数を決め、基準年の座標にジッターを加える

### 移民フローで変わること
- 計算が**国ごと独立 → 全国同時**（A国の流出 = B国の流入）
- 新しいデータ: GDP per capita（豊かさの指標）— SSP シナリオで2100年まで予測あり
- 新しいパラメータ: 移民率（余剰の何%が移動するか）

## 2. データモデル

### 必要なデータ

| データ | 用途 | ソース | 状態 |
|--------|------|--------|------|
| GDP (PPP) 2020〜2100 | 流入先の魅力度 | IIASA SSP Database | **✅ 取得済み** (`ssp_gdp_ppp.csv`) |
| 国の緯度・経度 | 距離計算 | `countries_coords.csv` | 既存 |
| 国別人口モデル | 自然増加計算 | `pop_model_data.json` | 既存 |
| 言語圏マッピング | 言語ボーナス | 手作成 | 未作成 |

### GDP データ: IIASA SSP Database

**`ssp_gdp_ppp.csv`** — 193カ国 × 6シナリオ（SSP1-5 + Historical Reference）
- 期間: 2020〜2100（5年刻み）
- 単位: billion USD_2017/yr (PPP)
- 国名表記: "Japan", "United States" 形式 → **ISO3 への変換が必要**

GDP が時代とともに変化するため、移民先が将来変わることを再現できる。
（例: インドの GDP 成長 → 2060年代には流出が減り、流入が増える可能性）

#### SSP シナリオの使い分け
- **SSP2（Middle of the Road）をデフォルトとして使用**
- 将来的に SSP シナリオ切替を UI に追加する可能性はあるが、Phase 1 では固定

#### 前処理で生成するファイル: `gdp_ssp.json`

```json
{
  "AFG": { "2020": 78.5, "2025": 89.2, ..., "2100": 450.3 },
  "USA": { "2020": 20937, "2025": 23150, ..., "2100": 65000 },
  ...
}
```
- SSP2 のみ抽出、国名→ISO3 変換、年→GDP のマップ
- per capita への変換は実行時に pop で割る（pop が動的に変わるため）

### 言語圏データ: `lang_groups.json`

重力モデルに言語圏ボーナスを加える（Ravenstein の法則 + 植民地つながりの簡易近似）。

```json
{
  "en": ["USA","GBR","CAN","AUS","NZL","IRL","IND","NGA","GHA","KEN","ZAF","PHL",...],
  "fr": ["FRA","BEL","CHE","CAN","SEN","CIV","CMR","COD","MLI","BFA","HTI",...],
  "es": ["ESP","MEX","COL","ARG","PER","VEN","CHL","ECU","GTM","CUB","DOM",...],
  "pt": ["PRT","BRA","AGO","MOZ","GNB",...],
  "ar": ["SAU","ARE","QAT","KWT","BHR","OMN","EGY","MAR","DZA","TUN","JOR","LBN",...]
}
```

- 同じ言語圏ペアに `LANG_BONUS = 3.0`（重みを3倍）を掛ける
- 複数言語圏に属する国（CAN: en+fr）は両方でボーナスを受ける
- データは小さいのでハードコード or 小JSON

### 距離マトリクス

`countries_coords.csv` の緯度経度から Haversine 距離を計算。
236カ国 × 236カ国 = ~56K ペアだが、対称行列なので ~28K 計算。

**判断: 実行時計算**。ファイルを増やすより、起動時に1回計算する方がシンプル。

## 3. 計算ロジック

### アーキテクチャ変更の核心

現在:
```
computeFuturePop(country, year, scenario) → pop
  // 各国独立。2024→year のループ
```

変更後:
```
computeAllFuturePop(year, scenario) → { countryCode: pop, ... }
  // 全国同時。2024→year のループ内で移民フローを計算
```

### アルゴリズム

```
for each year from 2024 to targetYear:
  ① 全国の自然増加を計算（既存ロジック）
    naturalPop[c] = pop[c] * (1 + birthRate - deathRate)

  ② 余剰人口を計算
    surplus[c] = max(0, naturalPop[c] - pop[c])  // 自然増加分
    emigrants[c] = surplus[c] * EMIGRATION_RATE   // 例: 0.05 (5%)

  ③ 流入先の割り振り（各流出国について）
    for each emigrant-sending country c:
      gdpPC[dest] = interpolateGDP(dest, year) / pop[dest]  // SSP2 から補間
      langBonus = shareLang(c, dest) ? LANG_BONUS : 1.0
      weight[dest] = gdpPC[dest] * (1 / distance(c, dest)^DISTANCE_DECAY) * langBonus
      normalize weights → probabilities
      distribute emigrants[c] to destinations proportionally

  ④ 人口を更新
    pop[c] = naturalPop[c] - emigrants[c] + immigrants[c]
```

### パラメータ

| パラメータ | 初期値 | 意味 |
|-----------|--------|------|
| `EMIGRATION_RATE` | 0.03 (3%) | 自然増加分のうち移民する割合 |
| `GDP_WEIGHT` | 1.0 | GDP の影響度（べき乗） |
| `DISTANCE_DECAY` | 1.5 | 距離の減衰指数（大きい = 近隣重視） |
| `LANG_BONUS` | 3.0 | 同一言語圏ペアの重みボーナス |
| `MIN_DISTANCE_KM` | 500 | 距離の下限（隣接国の異常な引力を防ぐ） |

### 移民の子供の TFR

**決定: 移民先の TFR に同化（即時）**
- 理由:
  - 2世代目には同化するのが一般的な人口統計の知見
  - 1世代目の TFR を追跡するには出身国ごとの人口を分離管理する必要があり、複雑度が爆発する
  - このプロジェクトは「思考実験」であり、精密なモデリングは目的ではない
- 実装: 移民した人口は流入先国の `pop` に加算するだけ（既存のコホート計算が自動的に流入先の TFR を適用）

## 4. 既存シナリオとの統合

**決定: 移民を独立トグル**
- 既存3シナリオ（Linear/Stable/Soft）× 移民 ON/OFF = 6パターン
- UI: 移民トグルボタンを設定パネルに追加
- **理由**: 移民の有無を比較できるのがこのフィーチャの価値

## 5. 描画方式

**Phase 1: ドットの再配分（静的）**
- `generateFutureDots` で移民後の人口を使ってドット数を決める
- 移民でドットが増えた国は新しい座標にドットが出現、減った国はドットが消える
- **現在のパラダイム「1ドット = N人」に完全に一致**

**Phase 2 候補: アーク表示**
- 主要な移民ルートを弧線（arc）で表示
- Globe.GL の `arcsData` で実装可能
- Phase 1 の結果を見てから判断

## 6. 実装ステップ

### Phase 1: データ準備 ✅ 完了
1. ✅ IIASA SSP Database から GDP (PPP) を取得 → `ssp_gdp_ppp.csv`
2. ✅ 前処理: 国名→ISO3 変換 + SSP2 抽出 → `gdp_ssp.json` (192カ国)
3. ✅ `lang_groups.json` を作成（5言語圏、109カ国）
4. ✅ `countries_coords.csv` の ISO3 マッピング確認 — 192カ国フルカバレッジ

### Phase 2: 計算エンジン（コア） ✅ 完了
1. ✅ `computeAllFuturePop(year, scenario)` — 全国同時計算エンジン
2. ✅ Haversine 距離 + 事前計算済み距離マトリクス
3. ✅ 移民割り振り（重力モデル + 言語ボーナス + GDP/cap キャップ 150K）
4. ✅ 既存 `computeFuturePop` をフォールバックとして維持
5. ✅ キャッシュ: `scenario_migration_year` → 全国人口マップ

### Phase 3: UI 統合 ✅ 完了
1. ✅ 「移民フロー」トグルボタン（i18n: EN "Migration" / JA "移民フロー"）
2. ✅ `generateFutureDots` が `showMigration` フラグで新旧エンジンを切替
3. ✅ シナリオ変更 or 移民トグルでキャッシュ無効化
4. ✅ 移民ボタン: 過去年ではグレーアウト
5. ✅ 世界人口ラベル: 移民ON時は `computeAllFuturePop` の合計値を使用

### Phase 4: チューニング & 検証 ✅ 完了
1. ✅ GDP/cap キャップ (150,000) を追加 — 小国石油国 (ガイアナ等) の異常値を抑制
2. ✅ 定性的検証 — 主要移民回廊が現実的:
   - NGA → CMR, GHA, IRL (英語圏+近隣)
   - IND → SGP, UAE, QAT (英語+湾岸)
   - MEX → PAN, GTM, USA (スペイン語+近隣)
   - DZA → TUN, LBY, MAR (アラビア語+近隣)
   - SYR → ISR, IRQ, SAU (アラビア語+高GDP)
3. ✅ パフォーマンス: 2024→2100 の77年ループ × 192カ国、初回計算後キャッシュ

## 7. リスク評価

| リスク | レベル | 対策 |
|--------|--------|------|
| 全国同時計算のパフォーマンス | LOW | 193国 × 77年 = ~15K ステップ。キャッシュすれば初回のみ |
| GDP データの欠損国 | LOW | SSP は193カ国カバー。欠損国は GDP=0 扱い（移民先にならないだけ）|
| 国名→ISO3 の変換ミス | LOW | pop_model_data.json の ISO3 と SSP の国名を照合。手動補完 |
| 移民パラメータの妥当性 | MEDIUM | 思考実験と割り切り。固定値で定性的に正しい方向を |
| 小国・島国への過剰流入 | MEDIUM | GDP が高くて近い小国に集中するリスク → 人口キャップ or 吸収力上限で対策 |
| 現行コードとの衝突 | LOW | 移民トグル OFF = 既存ロジックそのまま |

## 8. 複雑度

**中** — 計算ロジックの変更は大きい（独立→同時）が、描画側は既存パラダイムに乗る。

## 9. 決定済み事項

| 項目 | 決定 | 理由 |
|------|------|------|
| GDP データソース | IIASA SSP Database (SSP2) | 193カ国、2100年まで、無料、IPCC お墨付き |
| GDP の将来予測 | SSP2 の予測値を使用（固定値にしない） | データが手に入ったので |
| 移民モデル | 重力モデル + 言語圏ボーナス | Ravenstein の法則に基づく。学術的に裏付けあり |
| 移民の子供の TFR | 移民先に即時同化 | 複雑度とのトレードオフ |
| シナリオ統合 | 独立トグル（ON/OFF） | 比較が価値 |
| 描画方式 | Phase 1 は再配分、Phase 2 でアーク検討 | 既存パラダイムに乗せる |
