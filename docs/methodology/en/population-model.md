# Population Model — Methodology

> Document date: 2026-03-28
> Japanese version: [../population-model.md](../population-model.md)

## 1. Overview

National populations are projected from 2024 to 2100 using a discrete annual demographic update. The driving variable is **TFR (Total Fertility Rate)**, with three scenarios determining its future trajectory. Life expectancy, infant mortality, and initial age structure are held constant at 2023 values.

**This model is a thought experiment, not a forecast.** It asks: "What happens if current fertility trends continue without recovery?" The UN medium-variant projection (which assumes TFR convergence toward replacement level) is intentionally not used.

## 2. Input Parameters

Per-country entry in `pop_model_data.json`:

| Field | Description | Source |
|-------|-------------|--------|
| `t` | TFR 2023 (births/woman) | OWID indicator 950920 |
| `s` | TFR annual slope (2016-2023 linear regression) | Derived from above |
| `l` | Life expectancy 2023 (years) | OWID indicator 1118466 |
| `i` | Infant mortality 2023 (fraction; original unit: per 100 live births) | OWID indicator 1027774 |
| `p` | Population 2023 | OWID historical population |
| `y0` | Age 0-14 population share 2023 | World Bank SP.POP.0014.TO.ZS |
| `e0` | Age 65+ population share 2023 | World Bank SP.POP.65UP.TO.ZS |

## 3. Population Projection

### 3.1 Replacement-Level Birth Rate

```
deathRate = 1 / le
birthRate = deathRate × (yearTFR / 2.1) × (1 − im)
pop(t+1)  = pop(t) × (1 + birthRate − deathRate)
```

- `2.1` = replacement-level TFR (value at which population sustains itself across generations)
- TFR = 2.1 → birthRate ≈ deathRate (stable population)
- TFR < 2.1 → population decline, TFR > 2.1 → population growth

### 3.2 TFR Scenarios

| Scenario | Formula | Description |
|----------|---------|-------------|
| **Linear** | `max(0, t + s × (yr − 2023))` | Continue 2016-2023 linear trend. Floor at 0 |
| **Stable** | `t` | TFR frozen at 2023 value |
| **Soft** | `max(0.8, t + s × elapsed × exp(−0.03 × elapsed))` | Decline decelerates exponentially. Floor at 0.8 |

### 3.3 Population Cap

```
popCap = p × 3
pop(t+1) = max(0, min(pop(t+1), popCap))
```

A uniform cap of 3× the 2023 population. See §6.2 for design history.

## 4. 3-Layer Cohort Model

Used for age-based color encoding (elderlyPct → dot color). Does not feed back into total population calculations.

Initialization (2023 data):
```
young   = p × y0
working = p × (1 − y0 − e0)
elderly = p × e0
```

Annual update:
```
births      = (working + young × 0.3) × deathRate × fertilityRatio × (1 − im)
youngToWork = young / 15      // transitions to working after ~15 years
workToElder = working / 50    // transitions to elderly after ~50 years

youngDeaths  = young   × deathRate × 0.3
workDeaths   = working × deathRate × 0.5
elderDeaths  = elderly × deathRate × 2.5

young   += births − youngToWork − youngDeaths
working += youngToWork − workToElder − workDeaths
elderly += workToElder − elderDeaths
```

Mortality coefficients (0.3, 0.5, 2.5) are heuristic values, not calibrated against life tables.

## 5. Known Limitations

1. **TFR linear extrapolation is not a prediction** — historical TFR recoveries do exist
2. **Simplified age structure** — only 3 layers, no age-specific mortality tables
3. **No migration** — all scenarios assume closed populations
4. **LE and IM fixed at 2023** — no assumption of future medical improvement
5. **War, pandemic, and technological disruption not modeled**
6. **`1/le` is a crude approximation** — assumes age-uniform stable population

## 6. Design Decisions

### 6.1 Why Not UN Medium-Variant

The UN medium-variant assumes TFR will eventually revert to replacement level. This project asks "what if it doesn't?" — using UN projections would contradict the question itself.

### 6.2 Population Cap Evolution

1. **Agricultural carrying capacity**: arable area × Rwanda density (1,130/km²) → failed for Egypt (desert country importing food, actual pop 115M)
2. **Industrialized country whitelist**: World Bank high-income + upper-middle-income exempt → Bangladesh classification difficulty (agricultural but extremely dense due to rice paddies)
3. **Uniform 3× (current)**: simple, applies to all cases. No theoretical basis; purely heuristic

### 6.3 Migration to Replacement-Level Model

Old model: `birthRate = TFR / 30 × 0.5 × (1 − im)`
- Bug: population grew even at TFR < 2.1 (Japan TFR=1.208 → +0.83%/year growth)
- Cause: birth rate applied to entire population, not accounting for reproductive-age fraction
- Fix: scale birth rate by TFR/2.1 ratio → TFR=2.1 yields stability, below yields decline
