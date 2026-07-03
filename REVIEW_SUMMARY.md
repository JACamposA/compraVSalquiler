# compraVSalquiler — Review Summary for Fable

> Token-efficient briefing. All file locations are relative to repo root.

---

## What is this?

A single-page interactive Monte Carlo simulator for Argentine housing decisions: **"¿Comprar con crédito UVA o alquilar e invertir?"** (Buy with UVA mortgage vs Rent & Invest). R runs entirely in the browser via **WebR (WASM)** — no server.

**Live at:** GitHub Pages (deployed from `_site/` via GitHub Actions on push to `main`).

---

## File map

```
index.qmd          ~1680 lines — entire app (R engine + HTML sliders + JS orchestration)
styles.css           885 lines — all custom CSS; dark theme (#11151c)
_quarto.yml           10 lines — site config (title, favicon, css reference)
images/bg-desktop.png + bg-mobile.png — background images
_extensions/coatless/webr/ — WebR Quarto extension (vendor, don't touch)
_site/             — built output, committed and deployed
.github/workflows/publish.yml — CI: render + deploy to gh-pages branch
```

---

## Architecture

```
User browser
  ├── Quarto HTML page (pre-built _site/index.html)
  │     ├── WebR WASM runtime (loads R in-browser, ~5s first load)
  │     │     └── Setup chunk: loads MASS, ggplot2, dplyr, tidyr, scales, Matrix
  │     │         and defines the entire simulation engine
  │     └── Hidden Monaco editor (qwebrEditorInstances[0])
  │
  ├── Custom JS (inline, bottom of index.qmd)
  │     ├── Reads all control values → builds R code string via buildCode()
  │     ├── On "Calcular" click: injects code into Monaco editor → calls qwebrExecuteCode()
  │     ├── Intercepts R stdout (@@TAG@@...@@ lines) → renders styled HTML cards
  │     └── Moves graph blurbs before their matching <canvas> elements
  │
  └── Results area (#sim-app)
        ├── .qwebr-output-code-area — text output (beautified to cards)
        └── .qwebr-output-graph-area — ggplot2 canvases
```

**Critical constraint — this is Quarto/WebR, NOT Shiny:**
- There are no reactive cells. Controls are plain HTML; JS wires them manually.
- Nothing auto-recomputes on input change. User must click "Calcular" to re-run.
- `buildCode()` assembles a complete R script string from current control values and passes it to the hidden WebR editor for execution.
- If a new control needs to feed a value into R, it must: (a) be read in `buildCode()` and (b) injected as an R variable assignment in the code string. No other mechanism exists.

---

## Two modes (mode-toggle pill buttons)

| Mode | Entry point | What it simulates |
|---|---|---|
| Comprar vs Alquilar | `ejecutar_y_mostrar()` | UVA mortgage vs renting + investing for N years |
| Simulador de Inversión | `ejecutar_inversion_y_mostrar()` | Standalone portfolio growth |

Mode switch clears the output area. Inputs are persisted to `localStorage` (key: `compraVSalquiler:inputs`).

---

## R engine (index.qmd lines 23–797, `context: setup` chunk, hidden from user)

### Core functions

| Function | Location (approx. line) | Purpose |
|---|---|---|
| `crear_matriz_correlacion()` | 41 | 6×6 correlation matrix for economic variables |
| `generar_shocks_correlacionados()` | 56 | Multivariate normal shocks via `mvrnorm` |
| `simular_regimen_crisis()` | 62 | Markov-like crisis regime; monthly Bernoulli draws |
| `generar_eventos_vida()` | 81 | Annual life events per simulation path |
| `calcular_utilidad_crra()` | 108 | CRRA utility U(W)=(W^(1-γ)-1)/(1-γ) |
| `simular_compra_completa()` | 114 | UVA mortgage path simulation (month-by-month loop) |
| `simular_alquiler_completa()` | 271 | Rent + invest path simulation |
| `ejecutar_simulacion_completa()` | 376 | Runs N sim pairs, returns combined dataframe |
| `construir_params()` | 398 | Maps UI slider values → full parameter list |
| `ejecutar_y_mostrar()` | 500 | Main orchestrator: build params → simulate → print @@markers → print 3 plots |
| `simular_inversion_completa()` | 723 | Single investment path |
| `ejecutar_simulacion_inversion()` | 745 | N investment paths |
| `ejecutar_inversion_y_mostrar()` | 756 | Investment mode orchestrator |
| `tema_oscuro()` | 479 | ggplot2 dark theme (matches UI) |

### 6 correlated economic variables (per month, per simulation)

1. **inflacion** — Argentine peso CPI shock
2. **tc** — USD/ARS exchange rate shock
3. **salario** — real salary growth shock
4. **inmobiliario** — real estate appreciation shock
5. **bolsa** — stock market return shock
6. **alquiler** — rent price shock

Correlations: inflation↔FX=0.80, salary↔inflation=0.30, real estate↔stocks=0.40, rent↔inflation=0.60, real estate↔inflation=0.50.

**Path symmetry guarantees** (both paths per simulation receive identical):
- The 6 shock vectors above
- The crisis regime vector
- Life-event draws
- FX-jump occurrence and magnitude (pre-drawn per month as `shocks$salto_ocurre` / `shocks$salto_magnitud` in `ejecutar_simulacion_completa`)
- Stock-crash dynamics during crisis: both portfolios take `−caida_bolsa/duracion_media` per crisis month
- Expensas: both buyer and renter pay `expensas_mensual_usd` monthly (they cancel in `diferencia_mensual`)

### Crisis model (Argentine-specific)

- Default 12% annual probability (configurable 2–30%)
- Average 18-month duration (σ=6 months)
- On crisis start: 65% chance of FX jump (+30–65%), salary -12% real, real estate -25%, stocks -35%
- 30% chance government caps UVA adjustment for 6 months (historical precedent)
- Separate crisis inflation (default 12%/month vs 4%/month base)

### Life events (annual, per path)

- **Job loss:** 5% annual, avg 4 months unemployment
- **Big raise:** 10% annual (+20% salary)
- **Career change:** 3% annual (-15% temporary)
- **Major repair:** 8% annual (5% of property value)
- **Legal problem:** 2% annual (3% of property value)
- **Lease non-renewal:** 12% at contract end (+8% premium on new lease + moving costs)
- **Space upgrade:** 5% annual (+25% rent increase + moving costs)

### UVA mortgage mechanics

- Real rate formula: `cuota = deuda × (r(1+r)^n)/((1+r)^n - 1)` where r = `tasa_real_uva_anual/12`; at r = 0 falls back to `deuda/n` (straight-line)
- UVA index tracks peso inflation monthly; cuota in USD = `cuota_uva × uva_index / tipo_cambio`
- Financial stress threshold: cuota/income > 40%
- Operating costs on top of cuota: maintenance 1.5%/yr + property tax 1%/yr + insurance 0.3%/yr + expensas

### Tax model (symmetric between buyer and renter portfolios)

- Real estate capital gains: 15% (waived if primary residence + ≥2 years)
- Personal property tax: 0.5% on value above $100k USD (configurable, waivable for primary residence)
- Financial gains tax at exit: 15% on **actual gains above cost basis** — both paths track `base_costo_inv` (deposits add to basis; withdrawals clamp basis to portfolio value; market losses don't reduce basis)
- Dividend tax: 7% annually on 2% assumed yield — applied to **both** portfolios
- Emergency debt accrues at 10%/yr if cash runs out (buyer only)

### Behavioral finance (renter)

- Tracks max portfolio value; if drawdown < -25%, 30% annual chance of panic-selling
- After panic sale: switches to risk-free rate (2%/yr) for remainder

---

## User-facing controls and defaults

### Buy vs Rent mode (index.qmd lines 800–992)

| Control | ID | Default | Range / options |
|---|---|---|---|
| Ahorros actuales | sl-ahorros | $30,000 | 0–200,000 |
| Salario mensual | sl-salario | $2,333 | 300–10,000 |
| % salario para vivienda+ahorro | sl-tasa_ahorro | 50% | 0–100% |
| Valor propiedad | sl-propiedad | $120,000 | 20k–500k |
| Anticipo | sl-enganche | 20% | 0–80% |
| Tasa real UVA | sl-tasa_uva | 6% | 0–15% |
| Plazo crédito | sl-plazo | 25 años | 5–35 |
| Alquiler mensual | sl-alquiler | $437 | 100–3,000 |
| Expensas | sl-expensas | $200 | 50–800 |
| Ubicación | sl-ubicacion | GBA | CABA / GBA |
| Tipo de empleo | sl-empleado-publico | 0 (particular/privado) | 0 = privado → `tope_ratio=0.25`; 1 = empleado público/CERA → `tope_ratio=0.30` |

**Advanced (collapsed behind "Avanzado ▼" button):**

| Control | Default |
|---|---|
| Escenario macro | Continuidad (base) — also: Estabilización, Hiperinflación |
| Inflación mensual peso | 4% (Continuidad) / 2% (Estabilización) / 10% (Hiperinflación) |
| Inflación anual dólar | 3% |
| Retorno bolsa | Moderado 7% (also: Conservador 3%, Agresivo 10%) |
| Probabilidad crisis anual | 12% (Continuidad) / 6% (Estabilización) / 20% (Hiperinflación) |

### Investment simulator mode

| Slider | ID | Default | Range |
|---|---|---|---|
| Capital inicial | sl-inv_capital | $30,000 | 0–200k |
| Aporte mensual | sl-inv_aporte | $500 | 0–5,000 |
| Años a invertir | sl-inv_anos | 20 | 1–40 |
| Crecimiento del aporte | sl-inv_crecaporte | 8% | 0–20% |
| Rendimiento esperado | sl-inv_rendimiento | 7% | 0–15% |
| Precisión (N sim) | sl-inv_nsim | 500 | 50–1,000 |

---

## Eligibility tope — design notes

The bank qualification check uses a single `tope_ratio` variable rather than a hardcoded percentage. No Argentine bank publishes a 28% cap; dependencia/privado ≈ 25%, empleados públicos and cuentas CERA ≈ 30%.

**Two separate ratios — important distinction:**

| Field | Formula | What it measures |
|---|---|---|
| `cuota_ingreso` = `ratio_cuota_ingreso` | `costo_mensual_compra / salario * 100` | Total housing burden (cuota + taxes + insurance + maintenance) as % of income — financial stress indicator |
| `cuota_declarado` = `cuota_sobre_declarado` | `cuota_mensual_credito / salario_declarado * 100` | Mortgage payment only as % of declared salary — bank eligibility indicator |

`cuota_ingreso` is displayed as "Cuota / ingreso (real)" with thresholds <33% green / <50% yellow / ≥50% red.  
`cuota_declarado` drives the eligibility tile: green if ≤ `tope_ratio×100`, red otherwise.

**Why they're separate:** Banks only check the mortgage payment against salary (not operating costs). Operating costs belong in the financial stress indicator, not the eligibility gate.

**R:** `califica_banco <- cuota_sobre_declarado <= (tope_ratio * 100)` where `cuota_sobre_declarado = cuota_mensual_credito / salario_declarado_usd * 100`.

**No-califica warning** (shown in `@@RECOMENDACION@@` card when `tipo=comprar` and `califica=no`):
- `cuota_usd` = `cuota_mensual_credito`; `salario_minimo_usd` = `cuota_mensual_credito / tope_ratio`
- Banner: "Para una cuota de $Y/mes necesitás ganar al menos $Z/mes. Tu salario actual ($X/mes) no alcanza."
- Followed by actionable fix list (JS-computed from emitted fields):
  - `propiedad_max` = `round(credito_max / (1-enganche_pct))` where `credito_max = target_cuota / factor_amort`
  - `enganche_min_pct` = `ceiling((1 - credito_max / valor_propiedad_usd) * 100)`
  - `plazo_min_anos` = `ceiling(log(k/(k-r)) / log(1+r) / 12)` where `k = target_cuota / monto_credito`
- JS only shows each suggestion if within slider bounds (propMax ≥ $20k; enganMin ≤ 80%; plazoMin ≤ 35yr)

The input is **not clamped** to `tope_ratio` — it is a derived output signal (green/red), not an input limiter. Clamping would hide the "no calificás" case.

**Data flow:**
```
HTML toggle (#preset-empleo buttons)
  → sets #sl-empleado-publico.value ("0" or "1")
  → persisted to localStorage under key "empleado-publico"

buildCode() [on "Calcular" click]
  → injects: tope_ratio <- 0.25  (privado) or  tope_ratio <- 0.30  (público/CERA)

R: ejecutar_y_mostrar(..., tope_ratio = 0.25)
  → cuota_sobre_declarado <- cuota_mensual_credito / salario_declarado_usd * 100
  → califica_banco <- cuota_sobre_declarado <= (tope_ratio * 100)
  → emits @@INDICADORES@@...|tope_ratio=0.25@@
  → emits @@RECOMENDACION@@...|califica=si/no|cuota_usd=X|salario_usd=Y|salario_minimo_usd=Z@@

JS: renderIndicadoresCard(f)
  → topeRatioPct = Math.round(Number(f.tope_ratio || 0.25) * 100)
  → eligText  = 'Cuota: XX.X% del sueldo declarado (máx. YY%)'
  → eligColor = green if califica === 'si', red otherwise

JS: renderRecomendacionCard(f)
  → if tipo==='comprar' && califica==='no': prepend red banner with salary/cuota figures
```

The `#preset-empleo` click handler follows the identical pattern as `#preset-rendimiento` and `#preset-escenario` — sets a hidden `<input>`, updates `.preset-active` class, calls `saveInputs()`. The preset restore on page load uses the same `["key", "group-id"]` pair array that handles `escenario` and `rendimiento`.

---

## Output protocol (@@TAG@@ markers)

R prints structured markers to stdout; JS parses and renders them as styled cards.

| Tag | Fields | Rendered as |
|---|---|---|
| `@@RESUMEN@@` | plazo, compra_inicial, compra_cuota, compra_gastos, compra_total, alquiler_inicial, alquiler_cuota, alquiler_total | Two-column card: Comprando vs Alquilando |
| `@@INDICADORES@@` | ratio_pr, cuota_ingreso, cuota_declarado, califica, diferencia_mensual, emergencia, meses_emergencia, **tope_ratio** | Grid of 5 key-indicator tiles; tope_ratio drives eligibility display |
| `@@RECOMENDACION@@` | tipo, diferencia, prob, prob_alquiler, califica (si/no), cuota_usd, salario_usd, salario_minimo_usd, propiedad_usd, propiedad_max, enganche_min_pct, plazo_min_anos | Badge + detail sentence; red warning banner with actionable fix suggestions when tipo=comprar and califica=no |
| `@@BREAKEVEN@@` | ano (number or "nunca") | Yellow-accented callout card |
| `@@CONFIGURACION@@` | ahorros, salario, propiedad, credito, plazo, horizonte, alquiler, ubicacion | Grid of input echoes |
| `@@PATRIMONIO@@` | plazo, compra_media, compra_q10, compra_q90, compra_peor, alquiler_* | Two-column card with P10/P50/P90/worst |
| `@@CHECKPOINTS@@` | y5_c, y5_a, y10_c, y10_a, y15_c, y15_a | Table of median wealth at years 5/10/15 |
| `@@STRESS@@` | pct_sims, stress_med | Stress card (green/yellow/red) |
| `@@GRAFICOBLURB@@` | texto | Explanation text, relocated before matching canvas |
| `@@INVRESUMEN@@` | capital, aporte, anos, total_aportado, final_media, final_q10, final_q90, peor, multiplo | Investment summary card |

**Chart output order (3 plots for buy/rent mode):**
1. Histogram: distribution of (Buy patrimony − Rent patrimony) at horizon year
2. Ribbon chart: median + P25/P75 + P10/P90 wealth evolution over time for both options (with breakeven year dashed line)
3. Line chart: UVA monthly payment trajectory in USD (P10/P90 ribbon + median)

---

## Location-specific assumptions (`construir_params()` line 398)

| Parameter | CABA | GBA |
|---|---|---|
| Annual property appreciation (USD) | ~3% (estabilizacion=4%, hiper=1%) | ~0.5% (estabilizacion=2%, hiper=-1%) |

---

## Key JS functions (index.qmd lines 1025–1650)

| Function | Purpose |
|---|---|
| `buildCode()` | Assembles R code string from all current control values |
| `beautifyOutput()` | Parses @@TAG@@ lines → renders HTML cards in output area |
| `relocateGraphBlurbs()` | Moves `.rs-graph-blurb` divs before their matching `<canvas>` |
| `updateDiaUno()` | Live "día uno" cost card: anticipo + escritura + emergency reserve check |
| `parseCardMarker(text)` | Regex: `/^@@([A-Z]+)@@(.*)@@$/` → `{tag, fields}` |
| `setMode(mode)` | Toggles comprar/invertir, clears output, updates button states |
| `saveInputs()` / `loadInputs()` | localStorage persistence for all controls |
| `renderIndicadoresCard(f)` | Renders 5 indicator tiles; derives `topeRatioPct` from `f.tope_ratio` |

**Preset button pattern** (used by escenario, rendimiento, and empleo controls):
```js
document.querySelectorAll('#preset-X .preset-btn').forEach(function (btn) {
  btn.addEventListener('click', function () {
    document.getElementById('sl-X').value = btn.dataset.val;
    document.querySelectorAll('#preset-X .preset-btn').forEach(b => b.classList.remove('preset-active'));
    btn.classList.add('preset-active');
    saveInputs();
  });
});
```
Restore on load uses a `["key", "group-id", ...]` pair array iterated in the `savedInputs` block.

---

## CSS architecture (styles.css)

All dark theme. Key sections:

- **`:root`** — CSS vars: `--bg:#f3f5f9`, `--card:#fff`, `--ink:#1a1f2b`, `--accent:#2e86c1`, `--accent-grad: linear-gradient(135deg,#2e86c1,#6c5ce7)`, `--radius:18px`
- **`.app-hero`** — dark (#11151c) hero banner with gradient top border
- **`.app-card`** — dark card container for slider sections
- **`input[type="range"]`** — custom styled with CSS `--fill` var for progress track
- **`.dia-uno-card`** / `.duno-*` — upfront cost breakdown card
- **`.preset-btn`** / `.preset-active` — pill preset buttons (purple accent); used by escenario, rendimiento, and empleo toggles
- **`#sim-app .qwebr-output-code-area`** — dark console output area with all `.rs-*` card styles
- **`.avanzado-panel`** — collapsed by default (`max-height:0`), animated open
- **`@media (max-width: 767px)`** — bg-mobile.png background

**What's hidden via CSS:**
- `.qwebr-non-interactive-loading-container` — setup chunk loading indicator
- `#sim-app .qwebr-editor-toolbar`, `.qwebr-editor` — Monaco editor chrome
- `#quarto-margin-sidebar` — WebR "Code Links" sidebar widget
- `.quarto-title-block` — Quarto default title block

---

## CI/CD (.github/workflows/publish.yml)

Triggers: push to `main`
Steps: checkout → R 4.3.2 → system deps (libuv1-dev, libsass-dev) → knitr/rmarkdown → Quarto (pre-release) → WebR extension → `quarto render index.qmd` → deploy `_site/` to gh-pages branch via `peaceiris/actions-gh-pages@v3`.

---

## Callouts at bottom of page

- **Collapsible "¿Qué incluye esta simulación?"** — explains: UVA mechanics, crisis model, life events, investment model (S&P 500-like), taxes, location assumptions
- **Disclaimer callout (warning)** — not financial advice

---

## Default simulation run (on page load)

The second `{webr-r}` chunk auto-runs with defaults:
- ahorros=$30k, salario=$2,333, tasa_ahorro=50%, propiedad=$120k, enganche=20%, plazo=25yr, tasa_uva=6%, alquiler=$437, expensas=$200, ubicacion=GBA, rendimiento=7%, escenario=base, inflacion_pesos=4%/mo, inflacion_dolar=3%/yr, prob_crisis=12%, **tope_ratio=0.25**

---

## Known model constants (not exposed as controls)

- N simulations: 500 (buy/rent), configurable 50–1000 (investment mode)
- Salary growth: 8%/yr real USD
- Deposit: 3 months rent; commission: 1 month rent
- Escritura (closing costs): 10% of property value
- Sale costs: 6% of property value
- Contract duration: 2 years
- Random seed: 42
- Panic sell threshold: -25% drawdown; risk-free fallback: 2%/yr
- CRRA risk aversion γ=2.0
- Utility of ownership: $150/mo; utility of flexibility (renting): $100/mo
