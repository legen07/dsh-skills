---
name: open-sea-skin
description: |
  Real-time WebGPU ocean rendering algorithm with Gerstner wave physics, FBM procedural noise,
  dynamic day/night cycle, and glassmorphism theming for DeepSeek Harness.
  DSH plugin + Chrome/Edge extension; serves a self-contained ocean renderer at /open-sea-skin
  with adjustable waves (sea), daylight (time), and glass opacity (glass) parameters.
  Two-mode architecture: full page showcase and embedded skin mode via iframe injection.
  Uses TSL node shaders, Gerstner swells, and adaptive quality scaling.
  Preserves host page CSS variables for seamless theme integration.
license: MIT
metadata:
  author: Joe Legen <https://github.com/legen07>
  version: 1.2.4
  domain: visual-theme
  runtime: node + webgpu + three.js 0.178 + TSL
  triggers:
    - open-sea-skin
    - ocean skin
    - ocean theme
    - sea skin
    - threejs ocean
    - webgpu ocean
    - gerstner wave
    - glassmorphism
    - dsh skin
    - dsh theme
    - ocean renderer
    - animated ocean background
    - day night cycle
    - water shader
    - webgpu rendering
---

# Open Sea Skin — Real-Time Ocean Rendering Algorithm

A WebGPU-accelerated ocean skin for DeepSeek Harness that renders a persistent, interactive ocean background behind any host page. The algorithm combines Gerstner wave physics with fractal Brownian motion noise, dynamic day/night color palettes, and a glassmorphism CSS overlay system — all driven by three adjustable parameters: sea state, daylight, and glass opacity.

---

## 1. Algorithm Overview

The ocean engine runs in two modes:

- **Full page showcase** — Standalone demo with UI panel, sliders, FPS readout, drift controls, loader/error states
- **Skin mode** (`?skin=1`) — Bare ocean rendered behind the host UI in an isolated iframe, controlled via `postMessage` from the settings panel

Both modes share the same rendering pipeline but differ in how they receive parameters and whether they display chrome.

### Parameter Map

| Parameter | Range | Default | Controls |
|-----------|-------|---------|----------|
| `sea` | 0–100 | 45 | Gerstner wave amplitude multiplier |
| `time` | 0–100 | 55 | Day/night cycle position (0=dusk, 100=midday) |
| `glass` | 40–90 | 72 | CSS backdrop-filter opacity for glassmorphism |
| `autoCycle` | boolean | true | Whether time auto-animates through the day cycle |

---

## 2. Core Rendering Pipeline

### 2.1 Gerstner Wave Physics

Five directional Gerstner waves with different wavelengths, steepness, and directions simulate realistic ocean swells. Each wave is defined as:

```
{ dir: [dx, dy], wavelength: λ, steepness: s }
```

The wave generation converts each definition into shader primitives:

```
k = 2π / λ          // wave number
c = √(g·k)          // phase speed (g = 9.8)
a0 = s / k          // amplitude scaled by steepness
```

The five wave definitions:

| Direction | Wavelength | Steepness |
|-----------|-----------|-----------|
| [1.0, 0.0] | 60.0 | 0.12 |
| [0.6, 0.8] | 31.0 | 0.12 |
| [-0.7, 0.7] | 18.0 | 0.09 |
| [0.3, -0.95] | 9.5 | 0.07 |
| [-0.35, -0.94] | 5.0 | 0.05 |

The vertex displacement function iterates over all waves, computing for each:

```
f = k · (dir · xz - t · c)
x += a0 · dir.x · cos(f) · sea
y += a0 · sin(f) · sea
z += a0 · dir.y · cos(f) · sea
```

The normal is computed analytically from the same wave sum, using partial derivatives for the tangent and binormal vectors, then cross-product and normalize.

### 2.2 Fractal Brownian Motion (FBM) Noise

Three octaves of gradient noise with harmonic seeding produce micro-surface detail:

```
noise(p) = gradNoise(p) + gradNoise(2.04p + (17.3, 9.1)) × 0.5
         + gradNoise(4.11p + (42.7, 28.6)) × 0.25
```

The hash function uses sine-based pseudo-random generation on compact ranges to avoid float precision banding on integrated GPUs:

```
hash2(p) = fract(sin(p · (127.1, 311.7)) × 43758.5453)
         , fract(sin(p · (269.5, 183.3)) × 43758.5453)
```

The gradient noise uses quintic interpolation (`u = f²·f·(6f² - 15f + 10)`) between four seeded gradient vectors.

### 2.3 TSL Node Shaders (Three.js Shader Language)

The ocean material is built entirely from TSL nodes — `Fn`, `pass`, `uniform`, `float`, `vec2`, `vec3`, `vec4`, `If`, and math functions (`sin`, `cos`, `dot`, `cross`, `normalize`, `mix`, `pow`, `max`, `clamp`, `fract`, `floor`, `smoothstep`, `distance`, `reflect`).

Key TSL constructs:

- **`uniform()`** — Time, sea amplitude, sun direction/color, sky colors
- **`Fn(...)`** — Wraps shader logic into reusable node functions
- **`If(condition, () => { ... })`** — Conditional GPU execution (distance-gated)
- **`pass`** — Renders the final output to screen

### 2.4 WebGPU Rendering

Uses Three.js 0.178 with WebGPU backend. The ocean is a large flat plane geometry rendered with a custom TSL material. Key optimizations:

- **No MSAA** — Low-power adapter in skin mode
- **Distance-gated fragment work** — FBM detail normals, sparkle, and foam only compute within 140 units of the camera
- **Adaptive resolution** — `navigator.hardwareConcurrency ≤ 4` or `deviceMemory ≤ 4` triggers low quality
- **60 fps frame cap** — Frame-limited in skin mode
- **3× lighter geometry** — Compared to the standalone site
- **Bloom post-processing** — `bloom()` node from TSL display adds glow to crests

---

## 3. Dynamic Day/Night Cycle

The sky and ocean colors interpolate through four time-of-day palettes: **Dusk**, **Golden Hour**, **Afternoon**, **Midday**.

### Color Palettes

Each palette defines five color targets:

| Palette | Zenith | Horizon | Sun | Deep | Shallow |
|---------|--------|---------|-----|------|---------|
| **Day** | [0.07, 0.20, 0.42] | [0.52, 0.68, 0.82] | [1.0, 0.93, 0.80] | [0.015, 0.09, 0.11] | [0.06, 0.32, 0.36] |
| **Dusk** | [0.03, 0.05, 0.16] | [0.85, 0.36, 0.16] | [1.0, 0.42, 0.14] | [0.02, 0.045, 0.075] | [0.09, 0.15, 0.20] |

### Cycle Computation

```
elevation = lerp(-0.05, 0.62, t)    // sun angle
azimuth   = lerp(-0.9, 0.9, t)      // sun position
sunDir    = (cos(elevation)·sin(azimuth), sin(elevation), -cos(elevation)·cos(azimuth))

daylight = smoothstep(0, 0.42, elevation)   // 0=dusk, 1=midday

lerpRGB(target, dusk, day, daylight)  // interpolate all five color targets
```

The sun intensity also interpolates: `intensity = lerp(2.6, 1.6, daylight)`.

### Sky Rendering

The sky color is computed from:
1. **Zenith-to-horizon gradient** — `mix(uHorizonColor, uZenithColor, pow(max(up, 0), 0.42))`
2. **Deep color injection** — `mix(col, uDeepColor.mul(1.4) + uHorizonColor.mul(0.25), smoothstep(0, -0.15, d.y))`
3. **Sun specular** — `pow(max(dot(d, uSunDir), 0), 10) × 0.18` + `smoothstep(0.9994, 0.9998, s) × 30`
4. **Cloud layer** — Band-limited FBM cloud noise masked by `smoothstep(0.03, 0.16, d.y) × smoothstep(0.6, 0.22, d.y)`

### Ocean Material

The ocean color combines:
1. **Base color** — `mix(uDeepColor, uShallowColor, clamp(crest·0.35 + 0.45, 0, 1))`
2. **Subsurface scattering** — `pow(max(dot(V, uSunDir), 0), 3) × max(crest, 0) × 0.18`
3. **Sky reflection** — Fresnel-driven reflection with `reflect(V.negate(), N)`
4. **Sun specular** — Tight noise-modulated sparkle (`pow(ndh, 500)`) + broad gloss (`pow(ndh, 48) × 0.12`)
5. **Crest foam** — `smoothstep(0.5, 0.95, foamNoise) × smoothstep(1.0, 2.0, crest) × 0.85`
6. **Horizon fade** — `mix(col, uHorizonColor, smoothstep(150, 290, dist))`

---

## 4. Two-Mode Architecture

### 4.1 DSH Host Plugin (`plugin/index.js`)

The plugin registers a web server route at `/open-sea-skin` that serves the self-contained renderer. It uses `ctx.effect()` to hook into the DSH `webServer`:

```js
ctx.effect(() => ctx.webServer.register({
  kind: 'prefix',
  path: '/open-sea-skin',
  handler: serveAsset,
}), 'open-sea-skin: static renderer route')
```

The route serves:
- `skin.html` — The ocean canvas at `/open-sea-skin/`
- `styles.css` — Glassmorphism styles
- `ocean.js` — The WebGPU renderer (with `?skin=1` query)
- `vendor/**/*` — Three.js, TSL, and addons
- `loader.js` — The settings panel controller

### 4.2 Skin Mode (`?skin=1`)

The ocean iframe is injected at `z-index: 0` with `pointer-events: none` behind the host UI. Parameters arrive via:
1. **URL search params** — Initial values set by the host content script
2. **postMessage** — Live updates from the settings panel

```js
frame.contentWindow.postMessage({
  type: 'oss-set', sea: state.sea, t: state.time, auto: state.autoCycle,
}, targetOrigin)
```

The iframe URL format: `${assetUrl('skin.html')}?skin=1&sea=45&t=55&auto=1&parentOrigin=${location.origin}`

### 4.3 Settings Panel

The panel (`plugin/client.js`) creates:
- A floating toggle button (wave SVG icon) positioned near the settings trigger
- A dialog panel with: Sea state slider, Daylight slider, Glass opacity slider, Enable checkbox, Auto-cycle checkbox, Reset button
- CSS injection for glassmorphism (`glassCss()` function)
- Chrome storage/localStorage persistence
- i18n support (zh/en)
- Keyboard navigation (Tab cycling, Escape to close)
- `MutationObserver` for settings trigger detection
- `prefers-reduced-motion` respect

---

## 5. Glassmorphism CSS System

The `glassCss(glass)` function generates CSS custom property overrides that transform the host page's theme tokens into translucent glass surfaces:

```css
body { --dsw-alias-bg-base: rgba(255,255,255,${light}) !important; ... }
body[data-ds-dark-theme] { --dsw-alias-bg-base: rgba(9,12,16,${dark}) !important; ... }
```

The glass opacity parameter (40–90) maps to:
- Light mode: `light = glass / 100`, `dark = max(0.40, light - 0.12)`
- Dark mode: alpha values are adjusted for deeper translucency

The button and panel use `backdrop-filter: blur(14px) saturate(130%)` and `backdrop-filter: blur(22px) saturate(140%)` respectively.

---

## 6. Adaptive Quality System

```js
const LOW_END_DEVICE = QUALITY === 'low' || (QUALITY === 'auto' && (
  (navigator.hardwareConcurrency || 8) <= 4
  || ('deviceMemory' in navigator && Number(navigator.deviceMemory) <= 4)
));
```

In low-end mode, the renderer reduces geometry density and fragment shader complexity.

---

## 7. Storage & Persistence

The settings panel supports two storage adapters:
- **Chrome storage** — For Chrome/Edge extension installations
- **LocalStorage** — For standalone/web installations

Both adapters share the same `save(patch)` and `load()` interface. The `normalize()` function clamps values to valid ranges:
- `sea`: 0–100 (default 45)
- `time`: 0–100 (default 55)
- `glass`: 40–90 (default 72)
- `autoCycle`: boolean (default true)

Manually setting `time` disables `autoCycle`.

---

## 8. Similar Architectures on GitHub

The following repositories share algorithmic patterns with open-sea-skin:

| Repository | Stars | Shared Algorithm |
|------------|-------|-----------------|
| `madblade/waves-gerstner` | 22 | Gerstner wave model with vertex shaders |
| `nguyencongnamit/threejs-ocean` | 0 | Photorealistic ocean, Gerstner waves, baked-sky reflection |
| `cpl121/ocean-shader` | 0 | WebGL ocean with Gerstner waves, foam, lighting |
| `christianpasinrey/drift` | 0 | Real-time WebGL open ocean, Gerstner swells, atmospheric sky |
| `Twarga/Open-Sea-Sim-Grok4.6` | 0 | WebGPU open-sea, Gerstner swell, FBM micro-surface |

Common patterns across these repos:
- Gerstner wave formulations for ocean displacement
- FBM noise for micro-surface detail
- Fresnel-based sky reflection
- Time-of-day color interpolation
- Wave steepness/wavelength parameterization

---

## 9. File Structure

```
project/
├── plugin/
│   ├── index.js              # DSH host plugin entry (web server route)
│   └── client.js             # Settings panel controller (shared core)
├── native-dist/
│   ├── ocean.js              # WebGPU renderer (TSL shaders, Gerstner, FBM)
│   ├── skin.html             # Bare ocean canvas for skin mode
│   ├── styles.css            # Glassmorphism CSS variables
│   ├── loader.js             # Generated from shared/skin-core.js
│   ├── vendor/               # Three.js 0.178, TSL, OrbitControls, BloomNode
│   └── version/              # Version metadata
├── cordis.patch.yml          # DSH plugin manifest
├── package.json              # Module config, DSH build hooks
└── shared/                   # Source files (build concatenates to native-dist)
    └── skin-core.js          # Shared controller source
```

---

## 10. Implementation Checklist

- [ ] Use WebGPU backend with Three.js 0.178 and TSL node shaders
- [ ] Implement 5-directional Gerstner wave physics with configurable wavelength/steepness/direction
- [ ] Build 3-octave FBM gradient noise for micro-surface detail
- [ ] Create TSL node functions for wave position, normal, crest, sky color, ocean color
- [ ] Implement day/night cycle with smoothstep interpolation between Dusk/Golden Hour/Afternoon/Midday palettes
- [ ] Add distance-gated fragment work (140-unit radius) for performance
- [ ] Add adaptive quality detection via `navigator.hardwareConcurrency` and `deviceMemory`
- [ ] Build two-mode architecture: full showcase + `?skin=1` iframe mode
- [ ] Implement `postMessage` communication between settings panel and ocean iframe
- [ ] Create glassmorphism CSS system with `backdrop-filter` and CSS custom property injection
- [ ] Support Chrome storage and localStorage adapters
- [ ] Build DSH host plugin that registers `/open-sea-skin` route on `ctx.webServer`
- [ ] Add i18n support (zh/en)
- [ ] Respect `prefers-reduced-motion`
- [ ] Add keyboard accessibility (Tab, Escape) to settings panel
- [ ] Use `MutationObserver` to detect settings trigger position for button placement

---

## 11. Anti-Patterns

| Don't | Do |
|-------|-----|
| Use external animation libraries | Use TSL node shaders exclusively |
| Render ocean on main thread | Keep WebGPU rendering in isolated iframe |
| Hard-code color values | Define palettes as objects, interpolate via `lerpRGB` |
| Use CSS transitions for gesture feedback | Use instant CSS class toggles |
| Skip distance-gating | Always gate FBM detail work by camera distance |
| Use MSAA in skin mode | Use no-MSAA, adaptive resolution |
| Create new entry point for disabled skin | Always render button/panel so skin can be re-enabled |
| Double-render WebGPU | Whichever path mounts first owns the surface |
| Use `@keyframes` for gesture-driven motion | Use springs or instant class toggles |

---

## 12. Key Dependencies

- **Three.js 0.178** — 3D rendering
- **Three.js TSL** — Node-based shader construction
- **WebGPU** — GPU compute backend
- **OrbitControls** — Camera interaction (showcase mode)
- **BloomNode** — Post-processing glow effect
- **CDIR (Cordis)** — DSH plugin framework

---

## 13. AI-Generic

This skill has no dependency on any specific AI platform, model, or provider. It covers a complete real-time ocean rendering algorithm that works independently of any AI integration. Any AI integration should be built on top of the DSH plugin layer, typically in the settings panel controller or the storage adapter.
