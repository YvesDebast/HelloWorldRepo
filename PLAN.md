# Exoplanet Presentation — Implementation Plan

_Last updated: 2026-06-01 · Derived from `RESEARCH.md`_

This is the **single input** for the implementation session. It restates the
facts an implementer needs (so the plan stands alone), then defines phases,
deliverables, and verification criteria. Read `RESEARCH.md` only if you need the
"why"; everything required to build is here.

---

## 0. Context an implementer needs (self-contained)

**Goal.** One scrollable, presentation-style HTML page that explores
`exoplanets.csv` across four angles — Discovery, Planet science, Habitability,
Superlatives — plus a Caveats section. **Descriptive, not predictive.**

**Audience.** Software engineers at Moody's: comfortable with data and charts,
**not** astronomers. Define each domain term on first use (equilibrium
temperature, parsec, transit, metallicity, AU). Every chart gets a **one-line
plain-language takeaway**. Let visuals carry the story.

**Hard constraints (non-negotiable):**
- **HTML + vanilla JS in one file.** No build system, no framework, no package
  manager.
- **Zero dependencies.** Charts hand-drawn with `<canvas>` (or inline SVG). **No
  CDN, no charting library** — a third-party/CDN lib would require TSG +
  Cybersecurity approval under Moody's software policy. Avoid it entirely.
- **Dual-mode CSV loading**, same as `coffee_comparison.html` one folder up:
  - Served over HTTP (`python -m http.server`): `fetch('./exoplanets.csv')`
    succeeds → table/charts render automatically.
  - `file://` (double-click): fetch fails silently → user clicks a **"Load
    CSV"** button and picks the file. Document this on the page.
- **Honest data handling** (see §6) is part of the spec, not optional polish.

**Output filename:** `exoplanets.html` in the project root.

### Dataset facts (verified 2026-06-01)
- `exoplanets.csv`, **1,174 data rows** (1,175 lines incl. header), 19 columns.
- One row per planet. **A star/system is keyed by `star_id`**, not by name
  (multi-planet systems repeat the host star). Planet-level stats use row count;
  star-level stats group by `star_id`.
- CSV may contain quoted fields → need a `parseCSVLine` that respects quotes
  (reuse the approach from `coffee_comparison.html`).
- Many cells are **empty** (notably `sy_dist`, `pl_orbsmax`, `st_met`). Empty =
  missing, must be handled, never silently dropped.

**Columns (header order):**
`pl_name, star_id, hostname, pl_rade, pl_bmasse, pl_eqt, pl_eqt_computed,
disc_year, discoverymethod, ra, dec, sy_dist, pl_orbper, pl_orbsmax, st_teff,
st_rad, st_mass, st_met, sy_pnum`

| Column | Meaning / unit |
|---|---|
| `pl_rade` | Planet radius, Earth radii (Earth=1, Jupiter≈11) |
| `pl_bmasse` | Planet mass, Earth masses (Earth=1, Jupiter≈318) |
| `pl_eqt` | Equilibrium temperature, Kelvin |
| `pl_eqt_computed` | `True` = temp **estimated**, not measured |
| `disc_year` | Discovery year |
| `discoverymethod` | Transit / Radial Velocity / Imaging / … |
| `sy_dist` | Distance, parsecs (1 pc ≈ 3.26 ly) |
| `pl_orbper` | Orbital period, days |
| `pl_orbsmax` | Semi-major axis, AU (Earth–Sun = 1) |
| `st_teff` | Host star surface temp, K (Sun ≈ 5,778) |
| `st_rad`, `st_mass` | Host star radius / mass, solar units (=1) |
| `st_met` | Metallicity [Fe/H], dex (Sun = 0) |
| `sy_pnum` | Number of known planets in the system |

**Discovery-method counts (verified):** Transit 1,129 · Radial Velocity 24 ·
Imaging 17 · Orbital Brightness Modulation 3 · Transit Timing Variations 1.
(So the method-mix chart is dominated by Transit — expect this; don't "fix" it.)

### Hypotheses the page should confirm/refute (not just illustrate)
- **H1** Discovery dominated by Transit, surged after ~2014 (Kepler/TESS).
- **H2** Method biases the sample: Imaging → high mass / wide orbit;
  Transit & RV → large, close-in ("hot Jupiters").
- **H3** Mass–radius plane is lopsided toward giants; small temperate planets rare.
- **H4** Very few habitability candidates (radius 0.5–1.5 ⊕ **and** eqt 180–310 K),
  and many of those rely on **estimated** temperature.

---

## 1. Phases, deliverables, and verification

Build in order. Each phase has a concrete deliverable and a verification gate;
do not advance until the gate passes.

### Phase 1 — Skeleton + CSV loader
**Build:**
- `exoplanets.html` with header, five `<section>`s (A Discovery, B Planet
  science, C Habitability, D Superlatives, E Caveats), and a footer.
- Header: title "Exploring 1,174 Exoplanets", one-line intro, a **how-to-read**
  note, and a **"Load CSV"** file input (visible/used in `file://` mode).
- `parseCSVLine` (quote-aware) + a `parseCSV` that returns array-of-objects
  keyed by header.
- Dual-mode load: on page load try `fetch('./exoplanets.csv')`; on failure show
  a prompt to use the Load CSV button. Both paths feed the same `render(rows)`.
- A typed accessor that converts numeric columns to `Number` and treats `""` as
  `null` (missing), and `pl_eqt_computed` to a boolean.

**Deliverable:** page loads in both modes; `render` receives 1,174 parsed rows.

**Verify:**
- Served over HTTP → console logs `rows.length === 1174`, no errors.
- `file://` → auto-fetch fails gracefully, Load CSV button parses the same 1,174.
- Spot-check one row (e.g. `55 Cnc e`): `pl_rade≈1.875`, `sy_pnum=7`,
  `pl_eqt_computed===true`.

### Phase 2 — Chart helpers (canvas), tested visually
**Build small, reusable helpers** — keep them tiny; this is the highest bug-risk
area (axis scaling):
- `barChart(canvas, {labels, values, ...})`.
- `scatter(canvas, {points:[{x,y,group}], xLog, yLog, colorBy, axisLabels})` —
  must support **log axes** (mass–radius spans orders of magnitude) and
  per-group colour.
- `stackedSeries(...)` or stacked bars for method-mix-over-time.
- Shared axis/tick/label utilities. A small fixed colour map per discovery
  method (consistent across all charts + a legend).

**Deliverable:** helpers render correct axes/ticks against known inputs.

**Verify (visual + sanity):**
- Feed a known toy dataset; confirm a point at a known (x,y) lands at the
  expected pixel and tick labels read correctly on both linear and log axes.
- Resize/redraw doesn't distort (handle canvas pixel ratio if used).

### Phase 3 — Section A: Discovery story (tests H1)
**Build:**
- **Discoveries per year** (bar) — counts by `disc_year`.
- **Method mix over time** (stacked bars/area by year, coloured by method).
- Takeaway sentence stating whether H1 holds (Transit share, post-~2014 surge).

**Verify:** bar counts sum to the number of rows with a valid `disc_year`;
caption states **N used / N missing**; Transit visibly dominates.

### Phase 4 — Section B: Planet science (tests H2, H3)
**Build:**
- **Mass–radius scatter**, log–log axes (`pl_bmasse` vs `pl_rade`), coloured by
  method, with a legend. Earth & Jupiter reference markers if cheap.
- **Equilibrium temp vs orbit**: `pl_eqt` vs `pl_orbsmax` (AU), coloured by method.
- Takeaways referencing H2 (Imaging at high mass/wide orbit) and H3 (giant-heavy).

**Verify:** each chart caption reports N plotted / N dropped for missing x or y;
Imaging points sit at high mass / wide orbit as H2 predicts (or note if not).

### Phase 5 — Section C: Habitability (tests H4)
**Build:**
- **Radius vs temperature scatter** with the habitable box shaded:
  radius **0.5–1.5 ⊕** AND eqt **180–310 K**.
- Compute the candidate set; render a **small table** of candidates
  (`pl_name`, `pl_rade`, `pl_eqt`, `pl_eqt_computed`, `sy_dist`).
- Takeaway: "**N of 1,174** fall in the box; **M** of those rely on an
  **estimated** temperature (`pl_eqt_computed=True`)."

**Verify:** candidate count is reproducible from the filter; the est-temp count
matches `pl_eqt_computed===true` within the set; box bounds match the numbers above.

### Phase 6 — Section D: Superlatives
**Build a table** of record-holders, each with the planet name and value:
- **Closest** (min `sy_dist`), **hottest** (max `pl_eqt`), **most massive**
  (max `pl_bmasse`), **widest orbit** (max `pl_orbsmax`), **biggest system**
  (max `sy_pnum`, by `star_id`).

**Verify:** each value is the true extremum over **non-missing** entries; ties
handled deterministically; biggest-system uses `star_id` grouping.

### Phase 7 — Section E: Caveats + final polish
**Build** the caveats section from §6 below, footer with source line
("data describes what we can **detect**, not what exists"), and a per-column
**missingness** note (or small table).

**Verify:** every chart on the page shows its N-used/N-missing; caveats present;
page reads top-to-bottom as a coherent story.

---

## 2. Honest-handling rules (apply to every section) {#honest}
<a id="honest"></a>

These are spec, enforced in verification:
1. **Never silently drop rows.** Each chart/stat states **N used / N missing**
   for the columns it depends on.
2. **`pl_eqt_computed=True` = estimated.** Habitability flags how many
   candidates rely on estimated temperature.
3. **Selection bias is real.** Frame every conclusion as *what we can detect*,
   not what exists. Section E owns this.
4. **Multi-planet systems repeat the host star.** Star-level stats group by
   `star_id`; planet-level stats use row count.
5. **Hand-drawn charts = more code.** Keep chart helpers small; test axis
   scaling visually against known values (Phase 2 gate).

---

## 3. Page layout (target)

```
HEADER  "Exploring 1,174 Exoplanets" + intro + how-to-read + [Load CSV]
§A DISCOVERY     discoveries/year (bar)      | method mix over time (stacked)
§B PLANET SCI    mass vs radius (log scatter)| eq.temp vs orbit (scatter)
§C HABITABILITY  radius vs temp + shaded box ; candidates table
§D SUPERLATIVES  closest | hottest | most massive | widest | biggest system
§E CAVEATS       completeness · selection bias · estimated temp
FOOTER  source + "data describes what we can detect"
```

---

## 4. Definition of done
- `exoplanets.html` renders all five sections in **both** load modes, no console
  errors, **zero external requests** (no CDN/library).
- Every chart has a plain-language takeaway and an **N-used/N-missing** caption.
- Habitability count + estimated-temp count are reproducible from the stated
  filter; superlatives are true extrema over non-missing data.
- Each hypothesis H1–H4 is explicitly **confirmed or refuted** in prose.
- Caveats section present; star-level stats grouped by `star_id`.

## 5. Out of scope
Interactive sort/filter app · external data beyond `exoplanets.csv` ·
statistical modeling/prediction · charting libraries / CDNs.
