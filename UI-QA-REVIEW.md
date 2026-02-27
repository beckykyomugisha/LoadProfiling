# UI QA Review: Solar C&I Load Profile Generator

**Date:** 2026-02-27
**Reviewer:** Claude (AI-assisted QA)
**File under review:** `solar-load-profile-tool.html` (1,365 lines)
**Application type:** Single-file HTML/CSS/JS web application

---

## Executive Assessment

This tool generates electrical load profiles for commercial and industrial facilities in Sub-Saharan Africa. While it has a polished dark-theme visual appearance, deep inspection reveals significant issues across data integrity, usability, accessibility, and domain accuracy that would cause engineering professionals, energy consultants, and enterprise users to avoid this tool.

**Overall UI/UX Score: 5.5/10**

---

## CATEGORY 1: Reasons You Wouldn't Trust the Outputs

### 1.1 Only 3 out of 50+ industries have actual templates (CRITICAL)

The tool advertises **50+ industry types** across 17 categories, but only **3 have real equipment templates**:

- Cement factory (line 967)
- Hotel — luxury/resort (line 988)
- Gold mine (line 1005)

Every other selection silently falls through to a **generic "default template"** (line 1024) with vague entries like "Main process equipment (45%)" and "Secondary equipment (20%)." A user selecting "Pharmaceutical plant" gets the same profile as "Bus depot." **The tool never discloses this.**

**Impact:** Misleading output that appears industry-specific but isn't.

### 1.2 Confidence system is shallow and misleading

The confidence assessment (line 1180) is a single check:

```javascript
const confidence = metrics.loadFactor > 30 && metrics.loadFactor < 90 ? 'medium' : 'low';
```

- No "HIGH" confidence level exists — the best result is "MEDIUM"
- Does not factor in whether a real template was used vs. generic fallback
- Does not consider completeness of user-provided data
- A generic template with plausible load factors shows "MEDIUM" confidence

### 1.3 Operating time inputs are collected but never used (CRITICAL)

The optional "Operating start time" and "Operating end time" inputs (lines 722–728) are collected but **never used in any calculation**. The profile generation hardcodes fixed time periods:

```javascript
// line 1138-1142
if (hour >= 0 && hour <= 5) period = 'night';
else if (hour === 6) period = 'ramp';
else if (hour >= 7 && hour <= 17) period = 'prod';
else if (hour === 18) period = 'wind';
else period = 'eve';
```

A factory operating 22:00–06:00 (night shift) gets the exact same profile as one operating 06:00–18:00.

### 1.4 Equipment quantities are always 1

Every equipment line in the Load Analysis tab shows `Qty: 1` (line 1229). A cement factory with "Cement mill (primary)" at 2,200 kW is represented as a single 2,200 kW item rather than realistic quantities (e.g., 4 × 550 kW). This undermines report credibility.

### 1.5 Country data is static and unsourced

Grid availability, voltage, and diesel costs (lines 952–963) are hardcoded with:
- No source attribution
- No date stamp
- No regional granularity (e.g., Nigeria grid varies drastically by region)
- No mechanism for updates

---

## CATEGORY 2: Reasons You Wouldn't Use the UI

### 2.1 No data persistence — all work lost on refresh (CRITICAL)

No `localStorage`, `sessionStorage`, or any persistence mechanism. If a user accidentally closes the tab or refreshes, all inputs and generated results are permanently lost.

### 2.2 Alert-based error handling

Validation uses `alert()` (line 1356) — only one check exists (average > peak). There is:
- No inline error messaging
- No field highlighting
- No guidance for corrective action
- No validation for edge cases (e.g., peak load of 0.5 rounds all equipment to 0 kW)

### 2.3 No loading state or generation feedback

Clicking "Generate load profile" produces no visual feedback — no spinner, no button disable, no progress indicator.

### 2.4 Chart is rudimentary and non-interactive

The 24-hour bar chart (line 1240) is built from `<div>` elements:
- No Y-axis labels or scale reference
- No gridlines
- No click interaction
- Hover tooltips via CSS `::after` only (not keyboard accessible)
- No weekday/weekend toggle
- No zoom or pan

### 2.5 Export is CSV-only

No PDF report generation, no branded output for client delivery, no charts in exports. Energy consultants need deliverable reports, not raw CSV.

### 2.6 XSS vulnerability in output rendering (CRITICAL)

User inputs are interpolated directly into `innerHTML` without sanitization (line 1191):

```javascript
`<tr><td>Company location</td><td>${inputs.city ? inputs.city + ', ' : ''}${inputs.country}</td></tr>`
```

Typing `<img src=x onerror=alert(1)>` in the City field executes arbitrary JavaScript.

---

## CATEGORY 3: Reasons You Wouldn't Deploy It

### 3.1 Minimal mobile experience

Single breakpoint at 900px (line 521). Below that:
- Weekly profile table (9 columns) overflows
- Tab buttons become cramped
- Chart becomes too small to be useful
- No touch optimization

### 3.2 Zero accessibility compliance (WCAG 2.1 AA failures)

- No ARIA attributes anywhere
- Tabs lack `role="tablist"`, `role="tab"`, `aria-selected`
- Color-only status indicators (green/yellow/red)
- No keyboard navigation for tabs
- No skip links
- Emoji icons (☀, 📊) carry no alt text
- Inline `onclick` handlers instead of proper event delegation

### 3.3 No offline capability despite targeting Sub-Saharan Africa

Google Fonts loaded from CDN (line 7). If offline — common in the target market — fonts fail silently. No service worker, no font fallback strategy, no offline-first design.

### 3.4 Single-file architecture doesn't scale

At 1,365 lines in one file:
- CSS, JS, and HTML are co-located
- No build process, minification, or linting
- No test suite
- Adding a new industry template means editing UI markup file
- No version tracking of template data

---

## CATEGORY 4: Domain-Specific Deal-Breakers

### 4.1 No seasonal or climate variation

Load profiles should account for dry vs. rainy season, temperature variations (HVAC), and daylight hours (lighting). The tool generates a single flat profile.

### 4.2 No solar generation modeling

For a tool branded "Solar C&I," there is **zero solar generation modeling**:
- No PV generation curves
- No self-consumption ratios
- No battery storage sizing
- No grid import/export estimation
- No financial modeling

It's a load profiling tool, not a solar sizing tool — the branding is misleading.

### 4.3 No multi-scenario comparison

Professional energy assessments compare scenarios (e.g., "current" vs. "with solar" vs. "with solar + battery"). This tool generates one profile at a time with no comparison capability.

### 4.4 Weekend load model is unrealistically simplistic

Weekend load is a flat 30% multiplier (line 1150):

```javascript
weekendLoad = load * 0.3;
```

Hospitals, data centers, and other continuous-operation facilities don't drop to 30% on weekends. This makes weekend profiles unreliable.

---

## Severity Summary

| Severity | Count | Key Issues |
|----------|-------|-----------|
| **Critical** | 4 | 47/50 industries use generic template silently; operating times ignored; no data persistence; XSS vulnerability |
| **Major** | 7 | No solar modeling; CSV-only export; no accessibility; flat weekend model; no seasonal variation; unsourced data; no scenario comparison |
| **Moderate** | 5 | Alert-based errors; no loading states; basic chart; no mobile UX; no offline support |
| **Minor** | 3 | Inline onclick handlers; emoji icons; single-file architecture |

---

## Expert Recommendation

**This tool should not be used for client-facing deliverables or investment decisions** for five core reasons:

1. **False sense of precision.** The 6-tab output with detailed equipment tables, hourly charts, and weekly profiles looks authoritative — but 94% of industry selections silently use a generic template. The output is templated fiction dressed as engineering analysis.

2. **Ignores its own inputs.** Operating start/end times are collected but discarded. This violates the fundamental contract between a tool and its user.

3. **Lacks the "solar" in Solar C&I.** Generates load demand only. No solar yield estimation, PV sizing, storage analysis, or financial modeling. Answers only half the question.

4. **Not auditable.** No source attribution for country data, no methodology documentation, no version history for templates, and no test coverage means results cannot be defended professionally.

5. **Prototype shipped as product.** Single-file architecture, absence of tests, security vulnerabilities, and lack of persistence indicate a proof-of-concept presented as a finished tool.

**Suitable for:** Internal rough estimates by someone who already understands load profiling.
**Not suitable for:** Client reports, investment decisions, regulatory submissions, or bankable feasibility studies.

---

## Recommended Next Steps (Priority Order)

1. Add real equipment templates for all listed industries (or remove unsupported ones from the dropdown)
2. Wire up operating time inputs to the profile generation algorithm
3. Add data persistence (localStorage at minimum)
4. Sanitize user inputs before innerHTML rendering (XSS fix)
5. Replace `alert()` with inline validation messages
6. Add ARIA attributes and keyboard navigation for accessibility
7. Integrate a proper charting library (Chart.js / Plotly)
8. Add solar generation overlay capability
9. Implement PDF export for professional deliverables
10. Add seasonal/climate variation modeling
