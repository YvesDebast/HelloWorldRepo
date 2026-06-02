# Exoplanet Presentation — Implementation Plan

_Last updated: 2026-06-02 · Derived from `RESEARCH.md` · Expanded to incorporate `stars.csv`_

This is the **single input** for the implementation session. It restates the
facts an implementer needs (so the plan stands alone), then defines phases,
deliverables, and verification criteria. Read `RESEARCH.md` only if you need the
"why"; everything required to build is here.

> **What changed in this revision (2026-06-02).** A second dataset, `stars.csv`
> (one row per host star), arrived. It **joins to `exoplanets.csv` on `star_id`**
> and lets us tell a host-star/system story the first plan only hinted at. The
> scope grows from four angles to **five** (new **§C Host stars & systems**), the
> loader now ingests **two CSVs and joins them**, and the caveats gain a
> data-source rule. Everything else — zero-dependency canvas charts, dual-mode
> loading, honest N-used/N-missing handling, `star_id` grouping — is unchanged.

---

## 0. Context an implementer needs (self-contained)

**Goal.** One scrollable, presentation-style HTML page that explores
`exoplanets.csv` **joined to `stars.csv`** across five angles — Discovery,
Planet science, Host stars & systems, Habitability, Superlatives — plus a
Caveats section. **Descriptive, not predictive.**

**Audience.** Software engineers at Moody's: comfortable with data and charts,
**not** astronomers. Define each domain term on first use (equilibrium
temperature, parsec, transit, metallicity, AU, solar units, [Fe/H]). Every chart
gets a **one-line plain-language takeaway**. Let visuals carry the story.

**Hard constraints (non-negotiable):**
- **HTML + vanilla JS in one file.** No build system, no framework, no package
  manager.
- **Zero dependencies.** Charts hand-drawn with `<canvas>` (or inline SVG). **No
  CDN, no charting library** — a third-party/CDN lib would require TSG +
  Cybersecurity approval under Moody's software policy. Avoid it entirely.
- **Dual-mode loading of BOTH CSVs:**
  - Served over HTTP (`python -m http.server`): `fetch('./exoplanets.csv')`
    **and** `fetch('./stars.csv')` both succeed → join + render automatically.
  - `file://` (double-click): fetches fail silently → user uses a **"Load CSVs"**
    file input (accepts **multiple** files) and picks *both* files; code routes
    each by sniffing its header. Document this on the page.
- **Honest data handling** (see §6) is part of the spec, not optional polish.

**Output filename:** `exoplanets.html` in the project root.

### Dataset facts — `exoplanets.csv` (verified 2026-06-01)
- **1,174 data rows** (1,175 lines incl. header), 19 columns.
- One row per planet. **A star/system is keyed by `star_id`**, not by name
  (multi-planet systems repeat the host star). Planet-level stats use row count;
  star-level stats group by `star_id` (or use the `stars.csv` table directly).
- CSV may contain quoted fields → need a `parseCSVLine` that respects quotes.
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

### Dataset facts — `stars.csv` (verified 2026-06-02)
- **972 data rows** (973 lines incl. header), 10 columns. **One row per host
  star**, keyed by `star_id`.
- **Columns (header order):**
  `star_id, star_name, teff_k, mass_solar, radius_solar, metallicity_fe_h,
  distance_pc, ra_deg, dec_deg, num_planets`

| Column | Meaning / unit |
|---|---|
| `star_id` | Integer key; **the join key** to `exoplanets.csv` |
| `star_name` | Host name (== `hostname` in the planet file) |
| `teff_k` | Surface temperature, K (Sun ≈ 5,778) |
| `mass_solar` | Mass, solar units (Sun = 1) |
| `radius_solar` | Radius, solar units (Sun = 1) |
| `metallicity_fe_h` | Metallicity [Fe/H], dex (Sun = 0) |
| `distance_pc` | Distance, parsecs |
| `num_planets` | Known planets in the system |
| `ra_deg`, `dec_deg` | Sky coordinates (present, not central) |

- **Missing cells exist here too** (e.g. `CT Cha`, `PDS 70`, `HD 203030` have
  blank temp/mass/radius). Same rule: empty = missing, handled, never dropped.

### The join (verified 2026-06-02)
- **Join on `star_id`.** It is a clean 1:1 key: all **972** distinct `star_id`s
  referenced by the 1,174 planets exist in `stars.csv`, and every star has ≥1
  planet. **Zero orphans** on either side.
- A name join (`hostname` == `star_name`) also works here (names are unique,
  zero mismatches) — but `star_id` is canonical and robust to catalog-name drift
  (e.g. planet `HD 21749 c` has `star_id 46`, host name `GJ 143`). **Use
  `star_id`.** The requirement text's "join on host_name" is satisfied by this.
- **Source of truth.** `stars.csv` is the **authoritative star-level table**. The
  star attributes are *also* embedded per-row in `exoplanets.csv`
  (`st_teff/st_rad/st_mass/st_met/sy_dist/sy_pnum`); in the ~4 rows where they
  disagree (e.g. `star_id 502`: planet 5213 K vs star 5151 K), **prefer
  `stars.csv`** for star-level stats. Star-level charts read the **972-row star
  table**, not planet rows, so multi-planet systems aren't double-counted.

**Discovery-method counts (verified):** Transit 1,129 · Radial Velocity 24 ·
Imaging 17 · Orbital Brightness Modulation 3 · Transit Timing Variations 1.
(So the method-mix chart is dominated by Transit — expect this; don't "fix" it.)

### Hypotheses the page should confirm/refute (not just illustrate)
Planet-focused (unchanged):
- **H1** Discovery dominated by Transit, surged after ~2014 (Kepler/TESS).
- **H2** Method biases the sample: Imaging → high mass / wide orbit;
  Transit & RV → large, close-in ("hot Jupiters").
- **H3** Mass–radius plane is lopsided toward giants; small temperate planets rare.
- **H4** Very few habitability candidates (radius 0.5–1.5 ⊕ **and** eqt 180–310 K),
  and many of those rely on **estimated** temperature.

Star/system-focused (new, enabled by `stars.csv`):
- **H5** Host stars cluster near Sun-like (G/K) temperatures (~4,500–6,500 K);
  very hot or very cool hosts are rare — a selection effect of Transit & RV.
- **H6** Most systems have **one** known planet; multi-planet systems are a
  minority but real (up to 6–7 planets).
- **H7** **Metal-rich hosts skew toward giant planets** — high-mass planets
  concentrate at metallicity [Fe/H] ≳ 0 (the giant-planet–metallicity correlation).
- **H8** The sample is dominated by relatively **nearby** stars, with a long tail
  past 1,000 pc (the Kepler field); distance reflects what we can observe.

---

## 1. Phases, deliverables, and verification

Build in order. Each phase has a concrete deliverable and a verification gate;
do not advance until the gate passes.

### Phase 1 — Skeleton + dual-CSV loader + join
**Build:**
- `exoplanets.html` with header, six `<section>`s (A Discovery, B Planet science,
  **C Host stars & systems**, D Habitability, E Superlatives, F Caveats), and a
  footer.
- Header: title "Exploring 1,174 Exoplanets across 972 Systems", one-line intro,
  a **how-to-read** note, and a **"Load CSVs"** multi-file input (used in
  `file://` mode).
- `parseCSVLine` (quote-aware) + a `parseCSV` returning array-of-objects keyed by
  header.
- **Dual-mode, two-file load:** on page load `Promise.all([fetch('./exoplanets.csv'),
  fetch('./stars.csv')])`; on failure prompt the user to use the Load CSVs button.
  The button takes `multiple` files and **routes each by header sniff** (`pl_name`
  present → planets; `star_id`+`star_name` present → stars). Both paths feed the
  same `render(planets, stars)`.
- **Join:** build `starsById` (Map keyed by `star_id`) from the star table; attach
  `planet._star = starsById.get(planet.star_id)` to each planet row for
  star-enriched charts. Keep the 972-row `stars` array for star-level charts.
- Typed accessors: numeric columns → `Number`, `""` → `null` (missing);
  `pl_eqt_computed` → boolean.

**Deliverable:** page loads in both modes; `render` receives **1,174** planets and
**972** stars, fully joined.

**Verify:**
- Served over HTTP → console logs `planets.length === 1174`, `stars.length === 972`,
  and join coverage `1174/1174` (every planet has `_star`), no errors.
- `file://` → auto-fetch fails gracefully; Load CSVs button (both files) yields the
  same counts and join.
- Spot-check `55 Cnc e`: `pl_rade≈1.875`, `sy_pnum=7`, `pl_eqt_computed===true`,
  and `_star.star_name === "55 Cnc"`, `_star.num_planets === 7`.

### Phase 2 — Chart helpers (canvas), tested visually
**Build small, reusable helpers** — keep them tiny; this is the highest bug-risk
area (axis scaling):
- `barChart(canvas, {labels, values, ...})`.
- `scatter(canvas, {points:[{x,y,group}], xLog, yLog, colorBy, axisLabels})` —
  must support **log axes** (mass–radius spans orders of magnitude) and
  per-group colour.
- `stackedSeries(...)` or stacked bars for method-mix-over-time.
- `histogram(canvas, {values, bins, log})` — for distance and num_planets
  distributions (or pre-bin and reuse `barChart`).
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

### Phase 5 — Section C: Host stars & systems (tests H5, H6, H7, H8) — NEW
All four star angles requested. Star-level charts read the **972-row star table**;
the metallicity chart is planet-level (joined).
**Build:**
1. **HR-style scatter (H5)** — host `teff_k` (x) vs `radius_solar` (y, **log**),
   one point per star; Sun reference marker at (5,778 K, 1 R☉). For this
   non-astronomer audience plot temperature **cool→hot left→right** with a clear
   axis label (this deviates from the astronomer convention; note it). Takeaway:
   where do host stars cluster vs the Sun.
2. **System architecture (H6)** — bar chart: count of **stars** by `num_planets`
   (1, 2, 3, …). Callout: how many planets live in multi-planet systems vs
   singletons. Builds on the §E biggest-system superlative.
3. **Metallicity vs planet mass (H7)** — scatter: host `metallicity_fe_h` (x) vs
   `pl_bmasse` (y, **log**), one point per **planet** (the join-powered chart);
   mark the giant threshold (~50 ⊕) so the eye can test whether giants concentrate
   at [Fe/H] ≳ 0. Takeaway states whether H7 holds.
4. **Distance distribution (H8)** — histogram of `distance_pc` across the 972
   stars (consider log bins; range ≈ 6.5–2,500 pc). Takeaway: how local the sample
   is and the long tail to the Kepler field.
- **Optional enrichment** (only if cheap): colour the Phase-4 mass–radius points
  by host `teff_k`. Skip if it complicates the legend.

**Verify:** each chart caption reports **N used / N missing** over the relevant
column (star table for 1/2/4, planet rows for 3). System-architecture bar counts
sum to 972 stars. Metallicity chart's planet count + missing-`st_met`/`pl_bmasse`
count reconcile to 1,174. Each of H5–H8 is stated **confirmed or refuted**.

### Phase 6 — Section D: Habitability (tests H4)
**Build:**
- **Radius vs temperature scatter** with the habitable box shaded:
  radius **0.5–1.5 ⊕** AND eqt **180–310 K**.
- Compute the candidate set; render a **small table** of candidates
  (`pl_name`, `pl_rade`, `pl_eqt`, `pl_eqt_computed`, `sy_dist`) — and, now that
  we have the join, the host `star_name` and `num_planets`.
- Takeaway: "**N of 1,174** fall in the box; **M** of those rely on an
  **estimated** temperature (`pl_eqt_computed=True`)."

**Verify:** candidate count is reproducible from the filter; the est-temp count
matches `pl_eqt_computed===true` within the set; box bounds match the numbers above.

### Phase 7 — Section E: Superlatives
**Build a table** of record-holders, each with the planet/star name and value:
- **Closest** (min `sy_dist`), **hottest** (max `pl_eqt`), **most massive**
  (max `pl_bmasse`), **widest orbit** (max `pl_orbsmax`), **biggest system**
  (max `num_planets`, from the **star table**), and now **hottest / coolest host
  star** and **most metal-rich host** (from `stars.csv`).

**Verify:** each value is the true extremum over **non-missing** entries; ties
handled deterministically; system + star superlatives come from the 972-row table.

### Phase 8 — Section F: Caveats + final polish
**Build** the caveats section from §6 below, footer with source line
("data describes what we can **detect**, not what exists"), and a per-column
**missingness** note (or small table) covering **both** files.

**Verify:** every chart on the page shows its N-used/N-missing; caveats present
(incl. the two-source rule); page reads top-to-bottom as a coherent story.

---

## 2. Honest-handling rules (apply to every section) {#honest}
<a id="honest"></a>

These are spec, enforced in verification:
1. **Never silently drop rows.** Each chart/stat states **N used / N missing**
   for the columns it depends on.
2. **`pl_eqt_computed=True` = estimated.** Habitability flags how many
   candidates rely on estimated temperature.
3. **Selection bias is real.** Frame every conclusion as *what we can detect*,
   not what exists. Section F owns this (now incl. stellar-temperature and
   distance selection effects, H5/H8).
4. **Multi-planet systems repeat the host star.** Star-level stats use the
   **972-row star table** (or group planet rows by `star_id`); planet-level
   stats use the planet row count. Never count a 6-planet system's star 6 times.
5. **Two sources for star attributes.** `stars.csv` is authoritative for
   star-level stats; where it disagrees with the embedded `st_*` columns (~4
   rows), prefer `stars.csv` and note it.
6. **Hand-drawn charts = more code.** Keep chart helpers small; test axis
   scaling visually against known values (Phase 2 gate).

---

## 3. Page layout (target)

```
HEADER  "Exploring 1,174 Exoplanets across 972 Systems" + intro + how-to-read + [Load CSVs]
§A DISCOVERY      discoveries/year (bar)        | method mix over time (stacked)
§B PLANET SCI     mass vs radius (log scatter)  | eq.temp vs orbit (scatter)
§C HOST STARS &   HR-style temp vs radius (log) | system size: stars by #planets (bar)
   SYSTEMS        metallicity vs planet mass    | host distance distribution (hist)
§D HABITABILITY   radius vs temp + shaded box ; candidates table (+ host, #planets)
§E SUPERLATIVES   closest | hottest | most massive | widest | biggest system | host extremes
§F CAVEATS        completeness · selection bias · estimated temp · two-source note
FOOTER  source + "data describes what we can detect"
```

---

## 4. Definition of done
- `exoplanets.html` renders all **six** sections in **both** load modes, no
  console errors, **zero external requests** (no CDN/library).
- Both CSVs load and **join on `star_id`** (1,174 planets ↔ 972 stars, 100%
  coverage); join verified in console.
- Every chart has a plain-language takeaway and an **N-used/N-missing** caption.
- Habitability count + estimated-temp count are reproducible from the stated
  filter; superlatives are true extrema over non-missing data.
- Each hypothesis **H1–H8** is explicitly **confirmed or refuted** in prose.
- Caveats section present (incl. two-source rule); star-level stats use the
  972-row table, never double-counting multi-planet hosts.

## 5. Out of scope
Interactive sort/filter app · external data beyond `exoplanets.csv` /
`stars.csv` · statistical modeling/prediction · charting libraries / CDNs.
</content>
</invoke>
