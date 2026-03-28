# Data Sources — Provenance and References

> Document date: 2026-03-28

## 1. NASA Blue Marble (昼間地球テクスチャ)

| Item | Detail |
|------|--------|
| File | `blue_marble_day.jpg` |
| Source | NASA Earth Observatory, Blue Marble: Next Generation |
| URL | https://eoimages.gsfc.nasa.gov/images/imagerecords/73000/73909/world.topo.bathy.200412.3x5400x2700.jpg |
| License | NASA public domain (no copyright) |
| Processing | 5400×2700 → 夜景化（彩度50%低下 + ティールティント + 明度12%）→ 4096×2048 にリサイズ |
| Output | `earth_night_base.jpg` |
| Note | 夜景化の色調は NASA Black Marble の南極暗部をサンプリングして合わせた（陸: R≈8,G≈42,B≈62 / 海: R≈6,G≈32,B≈50） |

Reference: Stöckli, R., Vermote, E., Saleous, N., Simmon, R. & Herring, D. (2005). *The Blue Marble Next Generation — A true color earth dataset including seasonal dynamics from MODIS*. NASA Earth Observatory.

## 2. NASA Black Marble / VIIRS (夜間都市光)

| Item | Detail |
|------|--------|
| File | `earth-night.jpg` (元画像、gitignore 対象外) |
| Source | NASA VIIRS Black Marble / three-globe example image |
| URL | CDN: `three-globe/example/img/earth-night.jpg` |
| License | NASA public domain |
| Processing | 都市光ピクセル抽出（brightness>50, R>B×0.7, |R-B|<40）→ 31,089ドット → 海上ドット2,726除去 → 発展途上国補正で11,423追加 → 最終39,786ドット |
| Output | `city_dots.json` |
| Known Issues | 漁火・油田プラットフォーム・船舶の光が含まれる（海上ドットとして除去で対応）。電力インフラが未整備の国はドット数が過少（人口比補正で対応） |

Reference: Román, M.O. et al. (2018). *NASA's Black Marble nighttime lights product suite*. Remote Sensing of Environment, 210, 113-143.

## 3. OWID Historical Population (歴史人口)

| Item | Detail |
|------|--------|
| File | `owid_hist.json`, `owid_hist_meta.json` (gitignore), `pop_history.json` |
| Source | Our World in Data, indicator 953903 |
| URL | https://ourworldindata.org/ (API: `owid.cloud/v1/indicators/953903`) |
| License | CC-BY 4.0 |
| Coverage | 紀元前10,000年〜AD2023、168〜247カ国 |
| Processing | 全年の国別人口を抽出、TIMELINE 配列と合わせて `pop_history.json` に出力 |
| Known Issues | **古代データは HYDE 3.2 モデル推計値**（§10参照）。実測ではない。特に先コロンブス期アメリカの人口推計は学界で大きな議論がある |

Reference: Gapminder, HYDE & UN (2022). *Population*. Our World in Data. https://ourworldindata.org/population

## 4. OWID Total Fertility Rate (合計特殊出生率)

| Item | Detail |
|------|--------|
| File | `tfr_data.json`, `tfr_meta.json` (gitignore) |
| Source | Our World in Data, indicator 950920 |
| URL | https://ourworldindata.org/ (API: `owid.cloud/v1/indicators/950920`) |
| License | CC-BY 4.0 |
| Coverage | 1950〜2023、推計値 |
| Processing | 2016-2023の値で線形回帰 → slope (`s`) を算出。2023年値を `t` として `pop_model_data.json` に格納 |

Reference: UN Population Division (2024). *World Population Prospects 2024*. United Nations.

## 5. OWID Life Expectancy (平均寿命)

| Item | Detail |
|------|--------|
| File | `le_data.json`, `le_meta.json` (gitignore) |
| Source | Our World in Data, indicator 1118466 |
| License | CC-BY 4.0 |
| Processing | 2023年（or 最新）の値を `l` として `pop_model_data.json` に格納 |

Reference: UN Population Division (2024). *World Population Prospects 2024*. United Nations.

## 6. OWID Infant Mortality Rate (乳幼児死亡率)

| Item | Detail |
|------|--------|
| File | `im_data.json`, `im_meta.json` (gitignore) |
| Source | Our World in Data, indicator 1027774 |
| License | CC-BY 4.0 |
| Processing | per 100 live births → 分率（÷100）に変換。2023年値を `i` として格納 |

Reference: IGME (2024). *Levels and Trends in Child Mortality*. UN Inter-agency Group for Child Mortality Estimation.

## 7. World Bank Age Structure (年齢構成)

| Item | Detail |
|------|--------|
| File | `age_structure.json`, `wb_young.json`, `wb_elderly.json` (後者2つは gitignore) |
| Source | World Bank Development Indicators |
| URL | `https://api.worldbank.org/v2/country/all/indicator/SP.POP.0014.TO.ZS` (young), `SP.POP.65UP.TO.ZS` (elderly) |
| License | CC-BY 4.0 |
| Coverage | 2022年データ（261カ国） |
| Processing | 百分率 → 分率に変換。working = 1 − young − elderly として算出 |

Reference: World Bank (2024). *World Development Indicators*. https://data.worldbank.org/

## 8. Natural Earth GeoJSON (国境ポリゴン)

| Item | Detail |
|------|--------|
| File | `countries.geojson` |
| Source | Natural Earth |
| URL | https://geojson.xyz/ or Natural Earth Data |
| License | Public domain |
| Coverage | 258 countries/territories, ISO 3166-1 Alpha-3 codes |
| Usage | ドットの国割り当て（point-in-polygon）、海上判定（陸地ポリゴン内かどうか） |
| Known Issues | 一部の領土（フランス海外県等）でポリゴンが欠落。係争地域の境界は Natural Earth の判断に依存 |

Reference: Natural Earth (2024). *Free vector and raster map data*. https://www.naturalearthdata.com/

## 9. Reba et al. (2016) Historical City Data (歴史都市人口)

| Item | Detail |
|------|--------|
| File | `chandler.csv`, `modelski_ancient.csv`, `historical_cities.json` |
| Source | Reba, M., Reitsma, F., & Seto, K.C. (2016) — Chandler (1987) + Modelski (2003) のデジタル化・ジオコーディング |
| URL | Chandler: https://ndownloader.figshare.com/files/5407640 / Modelski Ancient: https://ndownloader.figshare.com/files/5356132 |
| License | CC-BY 4.0 |
| Coverage | Chandler: 1,587都市, BC2250〜AD1975 / Modelski: 154都市, BC3700〜AD1000 |
| Processing | 2データセットをマージ（近接0.5°以内で重複排除）→ 1,577都市。年ごとの人口を補間 |
| Known Issues | 古代の人口推計は元文献（Chandler, Modelski）の判断に依存。都市の定義が時代により異なる。Certainty フィールドあり（使用していない） |

References:
- Reba, M., Reitsma, F. & Seto, K.C. (2016). *Spatializing 6,000 years of global urbanization from 3700 BC to AD 2000*. Scientific Data, 3, 160034. https://doi.org/10.1038/sdata.2016.34
- Chandler, T. (1987). *Four Thousand Years of Urban Growth: An Historical Census*. Edwin Mellen Press.
- Modelski, G. (2003). *World Cities: −3000 to 2000*. FAROS 2000.

## 10. HYDE 3.2 (OWID 歴史人口の背後のモデル)

| Item | Detail |
|------|--------|
| Source | History Database of the Global Environment, Utrecht University / PBL Netherlands |
| URL | https://doi.org/10.17026/dans-25g-gez3 |
| License | CC-BY 3.0 |
| Usage | OWID の歴史人口データ（§3）の主要ソース。直接使用はしていないが、データの出自として記録 |
| Known Issues | **先コロンブス期アメリカの人口推計は学界で最も議論の多いトピック**。HYDE は Denevan (1992) 寄りの「高め推計」を採用。例: メキシコ BC1000 = 604万人。低め推計（Kroeber 1939）では全米大陸で800〜1500万人 |

References:
- Klein Goldewijk, K. et al. (2017). *Anthropogenic land use estimates for the Holocene — HYDE 3.2*. Earth System Science Data, 9, 927-953.
- Denevan, W.M. (1992). *The Native Population of the Americas in 1492*. University of Wisconsin Press.
- Kroeber, A.L. (1939). *Cultural and Natural Areas of Native North America*. University of California Press.
- Dobyns, H.F. (1966). *Estimating Aboriginal American Population*. Current Anthropology, 7(4), 395-416.

## 11. World Bank Country Classification (工業国判定、廃止)

| Item | Detail |
|------|--------|
| File | `industrialized.json`, `hic_countries.json` (gitignore) |
| Source | World Bank income classification |
| Usage | 人口キャップの工業国ホワイトリスト用に取得したが、一律3倍キャップに変更したため廃止 |
| Note | 高所得(86カ国) + 上位中所得(54カ国) = 140カ国を「工業国」と分類していた |
