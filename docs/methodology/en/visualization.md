# Visualization — Technical Methodology

> Document date: 2026-03-28
> Japanese version: [../visualization.md](../visualization.md)

## 1. City Light Dot Generation

### 1.1 Light Pixel Extraction

City light pixels are extracted from the NASA night image (`earth-night.jpg`, 4096×2048 equirectangular).

Filter criteria:
```python
city_mask = (
    (brightness > 50) &          # sufficient brightness
    (R > B * 0.7) &               # red dominates blue (excludes snow/ice)
    (abs(R - B) < 40)             # R≈G≈B white light (excludes orange desert)
)
```
brightness = (R + G + B) / 3

Result: 31,089 pixels extracted as city light dots.

### 1.2 Country Assignment

Point-in-polygon assignment using GeoJSON (Natural Earth) polygons with Shapely STRtree spatial index.

- Successfully assigned: 28,387 / 31,089 (91.3%)
- Unassigned: 2,702 → assigned to nearest country (buffer=2° nearest polygon search)

### 1.3 Ocean Dot Removal

Dots not contained within any GeoJSON land polygon are removed.

- Removed: 2,726 dots (fishing lights, oil platforms, ship lights)
- Remaining: 39,786 dots (final count after supplementation, see §1.4)

### 1.4 Developing Country Dot Supplementation

NASA night imagery depends on electrical infrastructure, causing developing countries to have disproportionately few dots relative to population.

Correction criteria:
- Global average: 285K people/dot
- Countries needing 3× or more dots AND 50+ dot deficit are targeted
- Random placement within GeoJSON polygons (70% near existing dots + 30% random)
- Supplementary dot brightness: 35-60 (lower, representing non-electrified areas)

Major corrections: IND 371→5,046 / NGA 37→799 / BGD 15→601 / ETH 13→451

### 1.5 Density Normalization (countryWeight)

Weight correcting inter-country dot density variance:

```javascript
globalAvgPPD = totalPop / totalDots  // global average pop-per-dot
countryWeight[code] = sqrt(ppd / globalAvgPPD)
```

sqrt used to suppress extreme values. Example results:
- JPN (943 dots / 124M): weight ≈ 0.8 (suppresses over-representation)
- IND (5,046 dots / 1,438M): weight ≈ 1.2 (normalized)
- NGA (799 dots / 228M): weight ≈ 1.3

This weight is multiplied into ratio during `updateYear()`.

## 2. Rendering

### 2.1 THREE.Points + Custom ShaderMaterial

42K+ dots rendered in a single draw call. Buffer pre-allocates growth slots (2× existing dot count), totaling ~127K slots.

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

- **Additive blending** (`THREE.AdditiveBlending`): overlapping dot lights sum together
- **depthWrite: false**: avoids transparent object draw order issues
- **gl_PointSize clamp**: 1.0–64.0 controls driver-dependent limits

### 2.2 Warm Tint

Orange-direction tint applied to city light dot colors:
```javascript
R = min(1.0, originalR * 1.3)  // R boosted 30%
G = originalG                   // G unchanged
B = originalB * 0.7              // B cut 30%
```

Rationale: NASA night image city lights are nearly neutral gray (R≈G≈B), but the teal dark earth base creates a contrast effect making them appear orange. This effect is intentionally enhanced. User preference test selected `warm core+glow r=6 ×2.5` (2026-03-24).

### 2.3 Dark Earth Base

Blue Marble daytime image converted to nighttime:

```python
# 1. 50% desaturation (Purkinje effect)
night = 0.5 * day + 0.5 * luminance

# 2. Teal tint
R *= 0.4, G *= 0.5, B *= 1.0

# 3. Terrain-based brightness variation
night = land_tint * (0.6 + 0.7 * luminance)  # land
night = ocean_tint * (0.7 + 0.3 * luminance)  # ocean

# 4. Ice/snow boost
night[ice] *= 1.2
```

Color palette sampled from NASA night image Antarctic dark areas:
- Land: R≈8, G≈42, B≈62
- Ocean: R≈6, G≈32, B≈50

## 3. Era-Based Visual Effects

### 3.1 Era Transition Variable

```javascript
if (year <= 1800) era = 0;                         // ancient: firelight
else if (year <= 1950) era = (year - 1800) / 150;  // transition: 0→1
else era = 1;                                       // modern: electric light
```

### 3.2 Ancient Firelight (era < 1)

Color: deep red-orange (firelight)
```javascript
fireR = min(1.0, origR * 1.2 + 0.3)
fireG = origR * 0.3
fireB = origR * 0.05
// linear interpolation with electric light by era
color = fire * (1 - era) + electric * era
```

Size: ancient dots are larger and softer
```javascript
sizeMultiplier = 1.0 + (1 - era) * 0.8  // up to 1.8× larger
```

### 3.3 Population Normalization by Era

- **Ancient (year < 1950)**: relative normalization. Maximum population country per year = ratio 1.0. No power curve (linear scale)
- **Modern (year >= 1950)**: ratio relative to 2023 population. Power curve applied

### 3.4 Age Color Shift (era = 1 only)

```javascript
function ageColorShift(origR, origG, origB, elderlyPct) {
  var t = clamp((elderlyPct - 0.05) / 0.35, 0, 1);  // 5%→0, 40%→1
  R = origR * (1 - t * 0.6);     // R cut up to 60%
  G = origG * (1 - t * 0.2);     // G cut up to 20%
  B = origB + (1 - origB) * t * 0.7;  // B boosted up to 70%
}
```

Result: young countries → orange (warm), aging countries → blue-white (cool)

## 4. Population Change Visualization

### 4.1 Decline (ratio < 1)

```javascript
brightness_factor = ratio ^ 1.5  // rapid dimming
size_factor = ratio ^ 1.2        // size reduction
```

When ratio < 0.5, dimmest dots (low dotRank) are turned off:
```javascript
keepFraction = brightness_factor * 2  // 0.5→keep all, 0→keep none
keepCount = floor(total * keepFraction)
// dots with dotRank < (total - keepCount) set to alpha=0
```

### 4.2 Growth (ratio > 1)

Existing dots slightly increase in brightness and size:
```javascript
brightness_factor = min(2.0, ratio ^ 0.6)
size_factor = min(1.8, ratio ^ 0.4)
```

Additionally, pre-allocated growth slots are activated:
```javascript
growthFraction = min(1.0, (ratio - 1) / 2.0)  // ratio 1→3 maps to 0→1
nShow = floor(nSlots * growthFraction)
```

Growth dots are randomly placed near the brightest existing dots (top 1/3), offset by lat/lon ±0.4°.

## 5. Historical City Dots

### 5.1 Data

1,577 cities from Reba et al. (2016), merging Chandler and Modelski datasets. Rendered as a separate `THREE.Points` object.

### 5.2 Population → Visual Mapping

```javascript
function getHistCityPop(pops, year)  // linear interpolation to nearest year

// Normalization: normalized against maximum city population for the year
norm = pop / maxCityPop  // 0..1
alpha = 0.3 + norm * 0.7
size = 2.0 + norm * 5.0
```

Color follows era (ancient: red-orange, modern: warm white).

## 6. Design Decisions

### 6.1 Texture Baking → Realtime Dots

Initial implementation: population changes baked into NASA night image, generating per-year static textures → swapped on Globe.GL.

Reasons for change:
- Cannot support dot movement (v2 migration flows)
- Scenario switching impossible (requires pre-generation)
- Difficult to add attribute overlays (age coloring, etc.)

### 6.2 Bloom (Sharp Core + Gaussian Glow)

Explored during static image generation. Dots plotted as points, Gaussian blur for glow → sharp core + glow composited.

Result: warm bloom r=6 ×2.5 was user-preferred. However, after migrating to Globe.GL realtime rendering, post-process bloom was not adopted. Instead, fragment shader glow falloff (`smoothstep`) provides a pseudo-glow effect.

### 6.3 Dots: Remove → Dim

Initial: population decline removed dimmest dots (alpha=0).
Changed to: continuous brightness/size variation (dots not removed).

Reason: future age-composition coloring cannot apply color attributes to removed dots. Keeping all dots allows arbitrary attributes to be added later.

Exception: when ratio is extremely low (brightness_factor < 0.5), dimmest dots are turned off (to represent dark ancient earth).
