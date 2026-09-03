# EQD2/BED Uncertainty Calculator — Full Build Specification

Build a single self-contained file called `index.html`. No frameworks, no external libraries, no build step, no localStorage, no sessionStorage. Plain HTML5, CSS3, and vanilla JavaScript only. Everything in one file.

---

## Application overview

A radiotherapy biological dose calculator that computes BED and EQD2 using the linear-quadratic model, propagates uncertainty in the α/β ratio through Monte Carlo simulation, and supports cumulative dose calculation for re-irradiation planning. The tool must be fully functional offline in any modern browser.

---

## Formulas — implement exactly these

```
BED  = n × d × (1 + d / ab)
EQD2 = D × (d + ab) / (2 + ab)   where D = n × d

Cumulative BED  = BED_current  + BED_prior
Cumulative EQD2 = EQD2_current + r × EQD2_prior
  where r = recovery factor (1 = no recovery, 0 = full recovery)
  and BED_prior and EQD2_prior use the same sampled ab as the current course
```

---

## Section 1 — Current course inputs

- Number of fractions (n): integer, minimum 1
- Dose per fraction d (Gy): decimal, minimum 0.01
- Display total dose D = n × d in Gy, updated live
- Tissue / α/β dropdown with these options and data:

```
Tumor — generic         point=10,  lo=8,    hi=12
Head & neck mucosa      point=10,  lo=7,    hi=12
Prostate tumor          point=1.5, lo=1.5,  hi=3
Breast tumor            point=4,   lo=3,    hi=4.6
Late-reacting normal    point=3,   lo=2,    hi=5
Spinal cord             point=2,   lo=0.87, hi=4.9
Lung (late)             point=3,   lo=2.5,  hi=6
Custom (manual entry)
```

- Add a code comment above this data block:
  `// PLACEHOLDER VALUES — replace with cited tissue-specific literature sources before clinical use`

- When Custom is selected, show three number inputs: α/β point estimate, range low, range high
- Show selected α/β point and range as small text below the dropdown: e.g. "α/β = 2 Gy, range 0.87 to 4.9 Gy"
- Validate all inputs: show an inline error message rather than NaN if inputs are invalid

---

## Section 2 — Point estimate results

Display these as large prominent cards, updated live on every input change:

- Total Dose (D) — full-width teal highlight card
- BED — card with value in Gy to 2 decimal places
- EQD2 — card with value in Gy to 2 decimal places

---

## Section 3 — Uncertainty analysis (Monte Carlo)

Add a toggle switch labelled "Uncertainty analysis". Default: ON.

When ON, run Monte Carlo on every input change:

**Triangular sampling function:**
```javascript
function triangularSample(lo, mode, hi) {
  if (hi <= lo) return mode;
  const u = Math.random();
  const c = (mode - lo) / (hi - lo);
  if (u < c) return lo + Math.sqrt(u * (hi - lo) * (mode - lo));
  return hi - Math.sqrt((1 - u) * (hi - lo) * (hi - mode));
}
```

**Monte Carlo loop:**
- 5000 iterations
- Each iteration: sample ab = triangularSample(lo, point, hi), compute EQD2 sample
- Sort samples ascending
- Report 5th, 50th, 95th percentiles using linear interpolation between order statistics
- Return both the three percentiles AND the full sorted samples array

**Display:**
- EQD2 Median card (50th percentile)
- 90% Interval card showing "Xth – Yth Gy, 5th–95th percentile"
- Histogram: inline SVG, 28 bins, teal colour, x-axis labelled "EQD2 (Gy)" with 6 numeric tick labels spanning actual min to max of samples. Handle the degenerate case (all samples identical) by showing a single centred bar.

When OFF: hide all Monte Carlo output and histogram.

---

## Section 4 — Prior course (re-irradiation)

Add a toggle switch labelled "Prior course" with subtitle "(re-irradiation)". Default: OFF.

When ON, show:
- Prior fractions (n_prior): integer, minimum 1
- Prior dose per fraction d_prior (Gy): decimal, minimum 0.01
- Recovery factor γ: two inputs side by side labelled "Low" and "High", defaults 1.0 and 1.0
- Add hint text below: "γ = 1: no recovery (conservative, recommended for spinal cord). γ = 0: full recovery."

**Cumulative calculation:**
Using the same Monte Carlo loop as Section 3, for each iteration also compute:
```
BED_prior_sample  = n_prior × d_prior × (1 + d_prior / ab_sample)
EQD2_prior_sample = (n_prior × d_prior) × (d_prior + ab_sample) / (2 + ab_sample)
r_sample          = midpoint of (γ_low + γ_high) / 2
cumBED_sample     = BED_current_sample  + BED_prior_sample
cumEQD2_sample    = EQD2_current_sample + r_sample × EQD2_prior_sample
```

Collect, sort, and report median and 90% interval for both cumBED and cumEQD2 samples.

**Display (only when Prior course is ON):**
- Cumulative BED Median card
- Cumulative BED 90% Interval card
- Cumulative EQD2 Median card (scalar surrogate)
- Cumulative EQD2 90% Interval card
- Warning box directly below in red/amber: "⚠ This assumes spatial coincidence of dose distributions between courses. This is a worst-case scalar surrogate, not a spatially-registered cumulative dose. Recovery factor is uncertain and organ-specific."

---

## Section 5 — OAR constraint flagging

For spinal cord and lung late presets, after the cumulative section show a constraint flag:

```
Spinal cord: illustrative EQD2 tolerance 50 Gy
Lung late:   illustrative EQD2 tolerance 20 Gy
```

- If prior course is OFF: flag against the current EQD2 95th percentile
- If prior course is ON: flag against the cumulative EQD2 95th percentile
- If the 95th percentile EXCEEDS the threshold: show amber warning "⚠ Upper bound of 90% interval (X Gy) exceeds illustrative EQD2 constraint of Y Gy."
- If within: show green "✓ 90% interval within illustrative EQD2 constraint of Y Gy."
- Add note below: "Constraint values are illustrative. Always refer to current QUANTEC/HyTEC guidance and consult medical physics."

---

## Section 6 — Validation panel

Add a section at the bottom titled "Self-test — Reference cases". Run on page load. Assert all to ±0.01 Gy tolerance.

Run these cases using the core BED and EQD2 formulas directly (not via the UI):

```
n=30, d=2,  ab=10  → BED 72.00,     EQD2 60.00
n=25, d=2,  ab=10  → BED 60.00,     EQD2 50.00
n=5,  d=4,  ab=3   → BED 46.6667,   EQD2 28.00
n=20, d=3,  ab=1.5 → BED 180.00,    EQD2 77.1429
n=35, d=2,  ab=10  → BED 84.00,     EQD2 70.00
n=15, d=3,  ab=10  → BED 58.50,     EQD2 48.75
```

Display each row showing: case label, computed BED, computed EQD2, PASS (green) or FAIL (red).
Show summary line: "6/6 reference cases pass (tolerance ±0.01 Gy)" in green if all pass.

---

## Section 7 — Footer

Show:
```
BED  = n × d × (1 + d / (α/β))
EQD2 = D × (d + α/β) / (2 + α/β)
Cumulative EQD2 = EQD2(current) + γ × EQD2(prior)
```

Add disclaimer: "Educational tool — Masters in Cancer Care Informatics project. Not for clinical decision-making. Always consult a qualified medical physicist."

---

## Visual design

- Page background: light grey (#F6F7F9)
- Cards: white (#FFFFFF) with light border (#DDE3EA), border-radius 10px
- Primary accent: teal (#0C8599)
- Dark card background (main results): deep navy (#152238)
- Warning/amber: #C8790F on #FBEEDB background
- Success/green: #2F9E58 on #E9F7EE background
- Typography: system font stack (-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif)
- Headings: Space Grotesk from Google Fonts (load via link tag)
- Monospace values: IBM Plex Mono from Google Fonts
- Generous whitespace, clean clinical look
- Responsive: single column on mobile (<768px), two-column grid (inputs left, results right) on desktop
- Toggle switches: custom CSS, teal when on
- All results update live on every input change with no submit button

---

## Code quality requirements

- All functions named clearly: `triangularSample`, `runMonteCarlo`, `computeBED`, `computeEQD2`, `drawHistogram`, `runValidation`
- Monte Carlo and histogram recompute together on every input change
- No console errors on load
- No NaN displayed anywhere — always show a clean error message for invalid inputs
- Comment the AB_RANGES object with the placeholder warning
- Comment the triangular distribution section explaining why triangular was chosen
- Do not use localStorage, sessionStorage, or any browser storage API
