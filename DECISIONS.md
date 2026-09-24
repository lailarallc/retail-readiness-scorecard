# Retail Readiness Scorecard — Decisions Log

Permanent record of choices that should survive session turnover.
If a decision is reversed, strike it through and add the replacement
below — don't delete.

---

## Format

Each entry:
- **Date** — when decided
- **Decision** — one sentence, imperative voice
- **Why** — the reasoning, including what was tried and rejected
- **Scope** — what this applies to (file, chunk, deliverable, or "global")
- **Do not** — explicit anti-instructions, if any

---

## Architecture & Pipeline

### 2026-05-26 — Use single-file HTML (Vite + vite-plugin-singlefile) as the build architecture
- **Why:** The offline-first constraint (must work with no internet connection, no loading spinner) eliminates every server-based option. Streamlit was considered and rejected: zero design control, wrong interaction model for a questionnaire, outputs look like data science demos rather than consulting deliverables. Single-file HTML with all assets inlined is the only approach that satisfies offline + instant load + Lailara design fidelity + professional PDF export.
- **Scope:** Global — this is the project's foundational architecture.
- **Do not:** Do not introduce a server, CDN dependency, or external fetch at any point. All CSS, JS, fonts, and libraries must be inlined in the output HTML.

### 2026-07-31 — Enforce the offline guarantee at build time

- **Why:** The offline-first constraint had no automated guard — a future change (swapping `doc.save()` for an HTML-based PDF render, or a font that failed to inline) could silently reintroduce a network request. `scripts/check-offline.mjs` runs as a `postbuild` step and fails the build if the bundled HTML contains any external resource-loading reference (script/link/img/iframe src+href, CSS `url()`/`@import`) or if fonts aren't inlined.
- **Scope:** Build pipeline (`package.json` `postbuild`; `scripts/check-offline.mjs`).
- **Do not:** Do not remove the `postbuild` hook or weaken the check to make a build pass. Anchor links (`<a href>`) and string URLs inside bundled libraries are correctly ignored — they are not resource loads.

---

## Data & Schema

[Decisions about data sources, schemas, transformations]

### 2026-07-31 — A launch-blocking gap overrides the numeric score band

- **Why:** Some low-weight "no" answers describe conditions that hard-block transacting (Item 360 incomplete → item setup rejected; GS1-128/SSCC-18 labels non-compliant → receiving failures; Costco direct-thermal → labels rejected at depot; missing FSMA 204 KDEs → Walmart won't accept the ASN). Under pure point-weighting these could still round to Green, producing a "Ready" verdict next to a "will be rejected" finding — the worst failure for an exec-facing tool. User decided (reviewing the four cases by name) to override the score.
- **Scope:** `scoreDimension` in `src/engine/scoring.js`. Item 360, EDI labels, and Costco thermal use `capAtYellow()` (a would-be Green becomes Yellow). FSMA 204 is a hard Red gate (early return, `numeric: 0`), mirrored as `isGate` in `questions.js`. Chart legend on both screen + PDF notes that a blocking gap can cap/fail a dimension regardless of score.
- **Do not:** Do not let any of these four "no" answers read Green. Do not "simplify" fulfillment back to default 70/30 thresholds or drop the caps — tests in `scoring.test.js` ("blocking-gap caps (C2)") guard this. EDI asn-timing "no" is intentionally NOT capped (fine-risk, not a hard reject).

### 2026-07-31 — A non-green dimension must never contradict its own badge

- **Why:** Cards rendered "No critical gaps identified." under a Red/Yellow badge whenever a dimension's findings array was empty (which happened for most "partial" answers). And "Top Priorities" padded to three by appending Green dimensions. Both put self-contradicting content in front of a buyer.
- **Scope:** `emptyStateText(status)` in `scoring.js` (shared by `screens.js` + `pdf.js`) returns status-appropriate copy — only Green may say "no gaps." Every "partial"/"no" answer branch pushes a concrete finding. `getTopBlockers` returns only Red/Yellow (0–3), never Greens.
- **Do not:** Do not reintroduce a hardcoded "No critical gaps" fallback in either renderer, and do not pad the priorities list with Greens. Guarded by the findings-invariant sweep test.

---

## Visualization

### 2026-05-26 — Use hand-written SVG for the results bar chart, not Chart.js
- **Why:** Chart.js tree-shaken is ~160KB; combined with jsPDF (~350KB) the file would exceed the 600KB budget. SVG is also the better technical fit: these are static R/Y/G status bars, not interactive data charts — no animation, no tooltip library, no axes needed. SVG renders crisper at any zoom, works natively in jsPDF for PDF export, and keeps the rendering ~5KB vs ~160KB.
- **Scope:** Results screen chart (`src/ui/results.js` and `src/ui/chart.js`).
- **Do not:** Do not add Chart.js, D3, or any charting library. The bar chart is intentionally hand-coded SVG.

---

## Output Formats

### 2026-05-26 — Use jsPDF v4 programmatic drawing for PDF export, not html2canvas
- **Why:** html2canvas rasterizes the page — the PDF looks like a screenshot, not a document, and degrades when printed or zoomed. Programmatic jsPDF drawing (doc.rect(), doc.text(), TTF fonts via addFileToVFS) produces crisp vector output that scales cleanly and forwards professionally. html2canvas also adds ~90KB to the bundle. The "CEO forwards this PDF" use case demands vector quality.
- **Scope:** PDF export module (`src/ui/pdf.js`).
- **Do not:** Do not add html2canvas. jsPDF draws all PDF content directly — no DOM capture. TTF fonts (not woff2) must be imported via Vite's `?base64` query and registered with jsPDF's VFS.

### 2026-05-26 — Apply the 600KB file size budget to gzip size, not raw HTML size
- **Why:** jsPDF ES min is 343KB alone; three TTF fonts add ~169KB base64; woff2 CSS fonts add ~73KB base64; app code adds ~40KB. Raw total ~1,072KB but gzip 395KB — well under budget. Single-file HTML is always served over HTTP with Content-Encoding: gzip; the uncompressed size is never what the user downloads. Text-heavy payloads (base64, minified JS) compress at 3–4×. Treating raw size as the metric would force dropping jsPDF (breaking PDF export), dropping fonts (breaking design system), or abandoning the single-file architecture — none of which is acceptable.
- **Scope:** Build output constraint for the lifetime of this project.
- **Do not:** Do not use raw uncompressed size as the budget metric. The build target is `gzip < 600KB`. Current: 395KB gzip.

---

### 2026-05-26 — Use CSS @keyframes for screen fade-in, not opacity class toggling

- **Why:** The original approach (`.screen { opacity: 0 }` → `.screen.visible { opacity: 1 }`) failed in the headless preview browser — opacity-0 elements were treated as non-interactive even after `.visible` was applied. `@keyframes screen-fade-in` animation makes elements always rendered at full opacity (animation is purely cosmetic), so they're always clickable and `display: flex` is always active. No JS toggle needed.
- **Scope:** `src/styles/layout.css` screen transition. All screen mounts use this pattern.
- **Do not:** Do not revert to opacity class toggling or `display: none` approaches for screen transitions. The @keyframes approach is simpler (no JS state for visibility), more accessible, and works correctly in both headless and real browsers.

### 2026-05-26 — Use window.__rrs_initialized guard to prevent HMR listener duplication

- **Why:** Vite HMR can re-execute `main.js` multiple times during dev reconnection. Each execution adds another click/keydown listener to `#app`, stacking multiple handlers against different `appState` instances. The guard (`if (!window[APP_INIT_KEY])`) ensures listeners and initial render run exactly once. `import.meta.hot.dispose()` resets the flag on module invalidation so a full re-render on the next manual reload is still clean.
- **Scope:** `src/main.js` boot block.
- **Do not:** Do not remove the HMR guard or move event listener registration outside it. Any re-execution of the boot block without the guard will stack listeners.

### 2026-09-24 — Re-copy the build to lailara-website after every release; the Pages deploy is not the public page
- **Why:** `lailarallc.com/scorecard` embeds `lailara-website/site/public/tools/retail-readiness-scorecard.html` (`?embedded&gatepdf`), a hand-copied build. Pushing this repo deploys only `retail-readiness-scorecard.pages.dev`. On 2026-09-24 the public copy was found 2 fixes behind (last copied 2026-07-08) after this repo's deploy had gone green.
- **Scope:** Every release of `dist/retail-readiness-scorecard.html`.
- **Do not:** Do not treat a green deploy here as "live for prospects". Verify `lailarallc.com/tools/retail-readiness-scorecard.html` contains the change. Before copying, confirm lailara-website has nothing else undeployed (last successful deploy SHA = HEAD), and keep the `scorecard-request-pdf` / `scorecard-proceed-pdf` postMessage contract intact.

## Writing & Voice

[Voice, style, terminology decisions specific to this project]

### 2026-09-24 — Score Walmart OTIF against the targets that apply to the supplier, not the retired 98% composite
- **Why:** Walmart replaced the 98% composite on 2024-02-01 (SPS Commerce): on-time 90% for prepaid (arrival by MABD) or 98% for collect (ready for pickup), and 95% in-full per category. A supplier ships prepaid or collect, not both, so asking about all three made an honest "yes" impossible for most suppliers. Partial = short on an applicable target but within 5 points (preserves the old 93–97 vs 98 band). Rejected: listing all three targets as required (user review, 2026-09-24).
- **Scope:** `ff_otif_rate` Walmart text, labels, findings in `src/data/questions.js`, `src/engine/scoring.js`, `scoring_engine/score.py`, `scoring_engine/retailers/walmart.yaml`, `src/data/retailers.js`.
- **Do not:** Do not reintroduce "98% composite" or a single blended OTIF %. Do not state the fine as "per PO" — it is "3% of COGS on non-compliant cases", matching the-question-engine q13. Change the wording in all five files above together.


---

## Reversed / Superseded

When a decision is overturned:
1. Strike through the original entry above (don't delete)
2. Add a new entry below with the replacement decision
3. Note the link in both directions

This preserves the history of why something is the way it is.
