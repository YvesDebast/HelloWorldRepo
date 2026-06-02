# Exoplanet Analysis — Research & Design

_Last updated: 2026-06-01_

This document records **what we decided to build, and why**, before writing any
code. It is the shared reference for goal, audience, hypotheses, scope,
visualization choices, and risks.

---

## 1. Goal

Turn `exoplanets.csv` (1,174 planets, 19 columns) into a **small single-page web
presentation** that explores the dataset and surfaces what is interesting in it,
across four angles: the discovery story, planet science, habitability, and
superlatives.

It is **descriptive / exploratory**, not predictive.

---

## 2. Audience & explanation level

- **Who:** people with a **small scientific background** — e.g. software
  engineers at Moody's. Comfortable with data, numbers, and charts; **not**
  astronomers.
- **Explanation level:** define each domain term on first use (equilibrium
  temperature, parsec, transit, metallicity). Concise and technical-but-
  accessible. **Let the visuals carry the story**, not jargon. Every chart gets
  a one-line plain-language takeaway.

---

## 3. Hypotheses (what we expect to find)

Stating these up front so the analysis can confirm or refute them — not just
describe.

- **H1 — Discovery is dominated by Transit, and surged after ~2014.** The launch
  era of Kepler/TESS should show a steep rise in transit discoveries.
- **H2 — Detection method biases the sample.** Imaging-discovered planets will
  cluster at **high mass and wide orbits**; Transit/Radial-Velocity will favour
  large, close-in planets ("hot Jupiters").
- **H3 — The mass–radius plane is lopsided.** Most catalogued planets are
  giants; genuinely small, temperate planets are rare.
- **H4 — Very few habitability candidates.** Planets with radius ≈ 0.5–1.5 Earth
  **and** equilibrium temperature ≈ 180–310 K will be a small handful, and many
  of those will rely on an **estimated** (not measured) temperature.

These are expectations, not conclusions; the presentation reports what the data
actually shows.

---

## 4. Key columns used

One row per planet. The columns the presentation actually depends on:

| Group | Columns | Units / meaning |
|---|---|---|
| Identification | `pl_name`, `star_id`, `hostname` | Planets sharing a `star_id` orbit the same star |
| Planet physical | `pl_rade`, `pl_bmasse`, `pl_eqt`, `pl_eqt_computed` | Earth radii (Earth=1, Jupiter≈11); Earth masses (Earth=1, Jupiter≈318); equilibrium temp in K; `True`=temp estimated, not measured |
| Discovery | `disc_year`, `discoverymethod` | Year; Transit / Radial Velocity / Imaging / … |
| Position & orbit | `sy_dist`, `pl_orbper`, `pl_orbsmax` | Distance in parsecs (1 pc≈3.26 ly); orbital period in days; semi-major axis in AU (Earth–Sun=1) |
| Host star | `st_teff`, `st_rad`, `st_mass`, `st_met` | Surface temp K (Sun≈5,778); radius/mass in solar units (=1); metallicity [Fe/H] dex (Sun=0) |
| System | `sy_pnum` | Number of known planets in the system |

`ra` / `dec` (sky coordinates) are present but **not central** to this analysis.

---

## 5. Scope — the four angles

| # | Angle | Question | Charts | Why it's in |
|---|---|---|---|---|
| A | **Discovery story** | How did we find these? | Discoveries/year; method-mix over time | Most complete, reliable data; frames everything (tests H1) |
| B | **Planet science** | How do planets & stars relate? | Mass–radius scatter; temperature vs. orbit | The meaningful physics; shows real relationships (tests H2, H3) |
| C | **Habitability** | Which look Earth-like? | Filter result: radius 0.5–1.5 ⊕ **and** eqt 180–310 K | The question people actually ask (tests H4) |
| D | **Superlatives** | What are the records? | Table: closest, hottest, most massive, widest orbit, biggest system | Memorable, concrete anchors |

---

## 6. Visualization choice & layout

- **Output:** a single scrollable HTML page, presentation-style.
- **Tooling:** **HTML + vanilla JS** (JS as the code-behind), no build system,
  no framework.
- **Charting:** **zero dependencies** — charts hand-drawn with `<canvas>`/SVG.
  No external libraries, so **no TSG/Cybersecurity approval required** (a CDN /
  open-source charting lib would need it under Moody's software policy).
- **CSV loading:** two-mode behavior —
  `fetch('./exoplanets.csv')` works when **served over HTTP**
  (`python -m http.server`); opening via **`file://`** (double-click) needs a
  manual **"Load CSV"** button. This is documented on the page.

### Text diagram — page layout

```
+------------------------------------------------------------+
|  HEADER                                                    |
|  "Exploring 1,174 Exoplanets"   [Load CSV] (file:// only)  |
|  one-line intro + how-to-read note                         |
+------------------------------------------------------------+
|  § A  DISCOVERY STORY                                      |
|  takeaway sentence                                         |
|  +--------------------+   +--------------------------+     |
|  | discoveries / year |   | method mix over time     |     |
|  | (bar, canvas)      |   | (stacked area/line)      |     |
|  +--------------------+   +--------------------------+     |
+------------------------------------------------------------+
|  § B  PLANET SCIENCE                                       |
|  takeaway sentence                                         |
|  +--------------------+   +--------------------------+     |
|  | mass vs radius     |   | eq. temp vs orbit (AU)   |     |
|  | (scatter, log axes)|   | (scatter, colored=method)|     |
|  +--------------------+   +--------------------------+     |
+------------------------------------------------------------+
|  § C  HABITABILITY                                         |
|  takeaway: "N of 1,174 fall in the box; M use est. temp"  |
|  +-----------------------------------------------+         |
|  | radius vs temp scatter, habitable box shaded  |         |
|  +-----------------------------------------------+         |
|  small table of the candidates                            |
+------------------------------------------------------------+
|  § D  SUPERLATIVES                                         |
|  table: closest | hottest | most massive | widest | biggest|
+------------------------------------------------------------+
|  § E  CAVEATS  (data completeness, selection bias, est temp)|
+------------------------------------------------------------+
|  FOOTER  source + "data describes what we can detect"      |
+------------------------------------------------------------+
```

---

## 7. Risks & honest-handling rules

These shape every result, agreed up front so the analysis is honest.

- **Missing data is heavy** in some columns (`sy_dist`, `pl_orbsmax`, `st_met`).
  → Report missingness per column; **never silently drop rows** — each chart
  states how many planets it is actually based on.
- **`pl_eqt_computed = True` means estimated, not measured.** → Habitability
  results flag how many candidates rely on estimated temperatures.
- **Selection bias is real.** Each method detects certain planets. → Every
  conclusion is framed as *what we can detect*, not what exists. Gets its own
  caveat (§ E).
- **Multi-planet systems repeat the host star.** → Star-level stats group by
  `star_id`; planet-level stats use the row count.
- **Hand-drawn charts are more code.** → Risk of bugs in axis scaling; keep the
  chart helpers small and test them visually against known values.

---

## 8. Alternatives considered & rejected

| Considered | Rejected because |
|---|---|
| Python + matplotlib report / Jupyter notebook | Audience wants a **presentation** to read in a browser, not a notebook; HTML/JS matches the existing repo style |
| Interactive web app (live sort/filter UI) | Heavier; the goal is a guided *story*, not a data-exploration tool. Possible later |
| Charting library (Chart.js / D3 via CDN) | Faster to build, but any third-party/CDN lib needs **TSG + Cybersecurity approval** under Moody's policy; zero-dep avoids that |
| Grouping shops/planets by name | A system is keyed by `star_id`; names alone would mis-merge multi-planet systems |

---

## 9. What's next

1. **Commit `RESEARCH.md` before any code** (see note below on repo state).
2. Build the page skeleton (header, 5 sections, footer) + CSV loader with the
   `file://` / HTTP dual-mode behavior.
3. Implement the small chart helpers (bar, scatter) in canvas, tested visually.
4. Fill each section A–D with its chart + takeaway; compute habitability filter.
5. Write the caveats section from § 7.
6. Verify by serving over HTTP and confirming all charts render.

---

## 10. Out of scope (for now)

- Interactive sort/filter web app.
- External data beyond `exoplanets.csv`.
- Statistical modeling / prediction — this pass is descriptive only.
