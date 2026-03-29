# Data Sources — Provenance and References

> Document date: 2026-03-28
> Japanese version: [../data-sources.md](../data-sources.md)

## 1. NASA Blue Marble (Daytime Earth Texture)

| Item | Detail |
|------|--------|
| File | `blue_marble_day.jpg` |
| Source | NASA Earth Observatory, Blue Marble: Next Generation |
| URL | https://eoimages.gsfc.nasa.gov/images/imagerecords/73000/73909/world.topo.bathy.200412.3x5400x2700.jpg |
| License | NASA public domain (no copyright) |
| Processing | 5400×2700 → night conversion (50% desaturation + teal tint + 12% brightness) → resized to 4096×2048 |
| Output | `earth_night_base.jpg` |
| Note | Night color palette sampled from NASA Black Marble Antarctic dark areas (land: R≈8,G≈42,B≈62 / ocean: R≈6,G≈32,B≈50) |

Reference: Stöckli, R., Vermote, E., Saleous, N., Simmon, R. & Herring, D. (2005). *The Blue Marble Next Generation — A true color earth dataset including seasonal dynamics from MODIS*. NASA Earth Observatory.

## 2. NASA Black Marble / VIIRS (Nighttime City Lights)

| Item | Detail |
|------|--------|
| File | `earth-night.jpg` (source image) |
| Source | NASA VIIRS Black Marble / three-globe example image |
| URL | CDN: `three-globe/example/img/earth-night.jpg` |
| License | NASA public domain |
| Processing | City light pixel extraction (brightness>50, R>B×0.7, \|R-B\|<40) → 31,089 dots → 2,726 ocean dots removed → 11,423 supplementary dots added for developing countries → final 39,786 dots |
| Output | `city_dots.json` |
| Known Issues | Includes fishing lights, oil platforms, and ship lights (removed via ocean filtering). Countries with limited electrical infrastructure have fewer dots (corrected via population-proportional supplementation) |

Reference: Román, M.O. et al. (2018). *NASA's Black Marble nighttime lights product suite*. Remote Sensing of Environment, 210, 113-143.

## 3. OWID Historical Population

| Item | Detail |
|------|--------|
| File | `owid_hist.json`, `owid_hist_meta.json` (gitignored), `pop_history.json` |
| Source | Our World in Data, indicator 953903 |
| URL | https://ourworldindata.org/ (API: `owid.cloud/v1/indicators/953903`) |
| License | CC-BY 4.0 |
| Coverage | 10,000 BC to AD 2023, 168–247 countries |
| Processing | Extracted per-country population for all years, combined with TIMELINE array into `pop_history.json` |
| Known Issues | **Ancient data is HYDE 3.2 model estimates** (see §10), not measurements. Pre-Columbian Americas population estimates are highly debated in academia |

Reference: Gapminder, HYDE & UN (2022). *Population*. Our World in Data. https://ourworldindata.org/population

## 4. OWID Total Fertility Rate

| Item | Detail |
|------|--------|
| File | `tfr_data.json`, `tfr_meta.json` (gitignored) |
| Source | Our World in Data, indicator 950920 |
| URL | https://ourworldindata.org/ (API: `owid.cloud/v1/indicators/950920`) |
| License | CC-BY 4.0 |
| Coverage | 1950–2023, estimates |
| Processing | Linear regression on 2016-2023 values → slope (`s`). 2023 value stored as `t` in `pop_model_data.json` |

Reference: UN Population Division (2024). *World Population Prospects 2024*. United Nations.

## 5. OWID Life Expectancy

| Item | Detail |
|------|--------|
| File | `le_data.json`, `le_meta.json` (gitignored) |
| Source | Our World in Data, indicator 1118466 |
| License | CC-BY 4.0 |
| Processing | 2023 (or latest) value stored as `l` in `pop_model_data.json` |

Reference: UN Population Division (2024). *World Population Prospects 2024*. United Nations.

## 6. OWID Infant Mortality Rate

| Item | Detail |
|------|--------|
| File | `im_data.json`, `im_meta.json` (gitignored) |
| Source | Our World in Data, indicator 1027774 |
| License | CC-BY 4.0 |
| Processing | Converted from per 100 live births to fraction (÷100). 2023 value stored as `i` |

Reference: IGME (2024). *Levels and Trends in Child Mortality*. UN Inter-agency Group for Child Mortality Estimation.

## 7. World Bank Age Structure

| Item | Detail |
|------|--------|
| File | `age_structure.json`, `wb_young.json`, `wb_elderly.json` (latter two gitignored) |
| Source | World Bank Development Indicators |
| URL | `https://api.worldbank.org/v2/country/all/indicator/SP.POP.0014.TO.ZS` (young), `SP.POP.65UP.TO.ZS` (elderly) |
| License | CC-BY 4.0 |
| Coverage | 2022 data (261 countries) |
| Processing | Percentage → fraction. working = 1 − young − elderly |

Reference: World Bank (2024). *World Development Indicators*. https://data.worldbank.org/

## 8. Natural Earth GeoJSON (Country Boundaries)

| Item | Detail |
|------|--------|
| File | `countries.geojson` |
| Source | Natural Earth |
| URL | https://geojson.xyz/ or Natural Earth Data |
| License | Public domain |
| Coverage | 258 countries/territories, ISO 3166-1 Alpha-3 codes |
| Usage | Dot-to-country assignment (point-in-polygon), ocean detection (land polygon containment) |
| Known Issues | Some territories (e.g. French overseas departments) have missing polygons. Disputed boundary decisions follow Natural Earth's judgment |

Reference: Natural Earth (2024). *Free vector and raster map data*. https://www.naturalearthdata.com/

## 9. Reba et al. (2016) Historical City Data

| Item | Detail |
|------|--------|
| File | `chandler.csv`, `modelski_ancient.csv`, `historical_cities.json` |
| Source | Reba, M., Reitsma, F., & Seto, K.C. (2016) — digitized and geocoded Chandler (1987) + Modelski (2003) |
| URL | Chandler: https://ndownloader.figshare.com/files/5407640 / Modelski Ancient: https://ndownloader.figshare.com/files/5356132 |
| License | CC-BY 4.0 |
| Coverage | Chandler: 1,587 cities, 2250 BC–AD 1975 / Modelski: 154 cities, 3700 BC–AD 1000 |
| Processing | Merged two datasets (deduplication within 0.5° proximity) → 1,577 unique cities. Population interpolated between available years |
| Known Issues | Ancient population estimates depend on original source judgment (Chandler, Modelski). City definitions vary across eras. Certainty field available but not used |

References:
- Reba, M., Reitsma, F. & Seto, K.C. (2016). *Spatializing 6,000 years of global urbanization from 3700 BC to AD 2000*. Scientific Data, 3, 160034. https://doi.org/10.1038/sdata.2016.34
- Chandler, T. (1987). *Four Thousand Years of Urban Growth: An Historical Census*. Edwin Mellen Press.
- Modelski, G. (2003). *World Cities: −3000 to 2000*. FAROS 2000.

## 10. HYDE 3.2 (Model Behind OWID Historical Population)

| Item | Detail |
|------|--------|
| Source | History Database of the Global Environment, Utrecht University / PBL Netherlands |
| URL | https://doi.org/10.17026/dans-25g-gez3 |
| License | CC-BY 3.0 |
| Usage | Primary source behind OWID historical population data (§3). Not directly used, but recorded as data provenance |
| Known Issues | **Pre-Columbian Americas population estimates are among the most debated topics in academia.** HYDE adopts "high count" estimates aligned with Denevan (1992). Example: Mexico at 1000 BC = 6.04M. Low-count estimates (Kroeber 1939) place the entire Americas at 8–15M |

References:
- Klein Goldewijk, K. et al. (2017). *Anthropogenic land use estimates for the Holocene — HYDE 3.2*. Earth System Science Data, 9, 927-953.
- Denevan, W.M. (1992). *The Native Population of the Americas in 1492*. University of Wisconsin Press.
- Kroeber, A.L. (1939). *Cultural and Natural Areas of Native North America*. University of California Press.
- Dobyns, H.F. (1966). *Estimating Aboriginal American Population*. Current Anthropology, 7(4), 395-416.

## 11. World Bank Country Classification (Deprecated)

| Item | Detail |
|------|--------|
| File | `industrialized.json`, `hic_countries.json` (gitignored) |
| Source | World Bank income classification |
| Usage | Originally obtained for industrialized-country whitelist in population cap. Deprecated after switching to uniform 3× cap |
| Note | High-income (86 countries) + upper-middle-income (54 countries) = 140 countries classified as "industrialized" |
