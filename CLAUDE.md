# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Session continuity rule:** At the end of every session, update the "Session log" section at the bottom of this file with a brief summary of what was changed. This allows future sessions to pick up context even if the previous chat is unavailable.

---

## Deploy checklist

Before committing and pushing **any** update (new cases, copy changes, bug fixes):
1. `cp cxr-swipe-v7.html index.html` — keep index.html in sync
2. Update `sitemap.xml` `<lastmod>` to today's ISO date (YYYY-MM-DD)
3. Update `dateModified` in the WebApplication JSON-LD block in `<head>` to today's date
4. **If the total case count has changed**, update it in **two** places inside `cxr-swipe-v7.html`:
   - JSON-LD `featureList[0]`: `"44 real chest X-ray cases from easy to expert difficulty"` (update the number)
   - `<noscript>` paragraph: `"ThoraSwipe includes 44 real chest X-ray cases"` (update the number)
   - Also add any new diagnoses to the `<noscript>` case list under the correct category heading

**After every commit, always run `git push` immediately.**

**CXR-012 is intentionally missing.** The sequence runs CXR-001 to CXR-011, then CXR-013 onwards. This case was deleted at some point before the current session log begins. The gap causes no bugs — case IDs are labels only, not array indices. If you add a new case, use the next available ID (currently CXR-046). Do not reuse CXR-012 unless you are deliberately filling that specific gap.

**If you change the default `diffFilter`, add cases, or restructure the queue**, bump `TS_SAVE_KEY` in the JS (e.g. `ts_progress_v2` → `ts_progress_v3`). This invalidates stale localStorage saves so returning users get a fresh queue matching the new defaults, rather than being stuck on an old case count. Never leave commits sitting locally. Vercel deploys automatically on push — committing without pushing means the live site is out of date.

---

## Project overview

**ThoraSwipe** is a single-file web app (`cxr-swipe-v7.html`) for medical learners to practice chest X-ray interpretation via a swipe-based interface. There is no build step, no framework, no package manager. Open the HTML file in a browser to run it.

`cxr-swipe-v6.html` is an older version kept for reference — do not edit it.

---

## Architecture

### Single-file structure

Everything lives in `cxr-swipe-v7.html`:
- **`<style>`** — all CSS (lines ~8–244)
- **`<body>`** — all HTML views and overlays
- **`<script>`** — all JS: case data, state, rendering, swipe logic

### View system

There is no router. Views are shown/hidden imperatively:

| View | Element | How shown |
|------|---------|-----------|
| Main game | `<main id="main">` | `style.display=''` / `'none'` |
| Round complete | `<div id="done">` | `.classList.add('show')` |
| Contact page | `<div id="contact-page">` | `.classList.add('show')` |
| Legal page | `<div id="legal-page">` | `.classList.add('show')` |
| Diagnosis picker | `<div id="picker-overlay">` | `.classList.add('open')` |
| Full result | `<div id="overlay">` | `.classList.add('open')` |
| Lightbox | `<div id="lightbox">` | `.classList.add('open')` |
| Diff banner | `<div id="diff-banner">` | hidden via `style.display='none'` when on contact/legal |

`showContact()` tracks where the user came from (`contactFrom` = `'main'` or `'done'`) so `hideContact()` can return to the right place.

### Game flow

```
buildQueue() → renderCase() → decide(choice)
                                    ↓
                             showResult(c, ok, choice)   ← picker phase for all cases
                                    ↓
                             revealFull() → showFullResult()  ← annotations + explanation
                                    ↓
                             nextCase() → renderCase() or showDone()
```

**Queue:** `buildQueue()` returns `[...shuffle(EASY), ...shuffle(MEDIUM)]`. Hard cases (difficulty 3) are not in the initial queue — they are spliced in by `unlockHard()` after a 3-case correct streak.

**State variables:**
```javascript
let queue=[], idx=0, scoreOk=0, scoreBad=0, imgLoadSeq=0;
let streak=0, bestStreak=0, hardUnlocked=false, waiting=false;
let lightboxOpen=false, lightboxJustClosed=false;
let currentImgSrc='';
let pendingCase=null, pendingOk=false, pendingChoice='';
```

`imgLoadSeq` is a counter used to prevent stale image-load callbacks when the user advances quickly — each load is given a `loadId` and the callback no-ops if `loadId !== imgLoadSeq`.

### Image filtering (PACS controls)

All four image surfaces share a single set of CSS custom properties on `:root`:

```css
--wl-b: 1.0;   /* brightness */
--wl-c: 1.1;   /* contrast */
--wl-inv: 0;   /* invert: 0 or 1 */
```

Applied via `filter: brightness(var(--wl-b)) contrast(var(--wl-c)) grayscale(1) invert(var(--wl-inv))` on `.img-wrap img`, `.annot-wrap img`, `.lightbox img`, and `.picker-img-wrap img`.

Two sets of slider elements exist (card PACS bar and lightbox PACS bar) and are kept in sync by `syncLbPacs()` / `syncCardPacs()`.

### SVG annotation system

`buildAnnotations(annots, W, H)` draws into `<svg id="annot-svg">` using the image's `naturalWidth`/`naturalHeight`. Annotation coordinates in the `CASES` array are percentage-based (0–100) and are scaled to pixel positions via `px = v/100 * W`.

Two annotation types: `'ellipse'` (draws a dashed animated ellipse) and `'arrow'` (draws an animated line with arrowhead marker). Labels are SVG `<text>` elements with a background `<rect>`.

---

## Image sourcing research agent

Use this prompt to launch a research agent when you need to find Wikimedia Commons images for specific pathology gaps. The agent works backwards — it browses Commons categories and file pages first, then cross-references to journal articles — rather than starting with a diagnosis and hunting for an image.

**When to invoke:** run this agent before starting a new batch of cases, or whenever the gap list has grown. It will return a structured table of verified candidates ready to write.

**How to invoke:** paste the prompt below into a new Claude Code session (or use `/agent` if available). Update the "Gaps to fill" section with the current gap list before running.

---

### Agent prompt (copy and paste to run)

```
You are a research agent for ThoraSwipe, a chest X-ray education app. Your job is to
find Wikimedia Commons images that fill specific pathology gaps in the case library.
Work backwards: browse Commons categories first, find candidate images, then check
each candidate for a linked journal article that provides confirmed clinical details.

## Hard constraints — an image is only usable if ALL of these are true:
1. Hosted at upload.wikimedia.org (direct hotlink, no /thumb/ path)
2. Licence is CC BY or CC BY-SA (any version). Reject CC BY-NC, CC BY-ND, or NC-SA.
3. Plain chest radiograph only — NOT CT, NOT MRI, NOT histology, NOT illustration.
4. The pathology is clearly visible and unambiguous on the plain film.
5. The Commons file page confirms the diagnosis — do not infer from filename alone.

## Preferred sourcing (in order):
1. Image uploaded from a published CC BY journal article — gives confirmed diagnosis,
   patient demographics, and expert interpretation. Check the Commons file page
   "Source" field for journal name, authors, and DOI.
2. Image by a named clinician (e.g. James Heilman MD, Hellerhoff) with a clear
   description on the file page.
3. Avoid anonymous uploads with no description beyond the filename.

## Gaps to fill (update this list each session):
Priority 1 (HIGH):
- Lingular pneumonia — loss of LEFT heart border (silhouette sign); need a PA CXR
  showing consolidation obliterating the left cardiac border
- Malpositioned ETT — endotracheal tube in right mainstem bronchus, left lung collapse
- Pneumoperitoneum — free air under the diaphragm on erect PA or supine CXR
- Pulmonary metastases — multiple bilateral nodules, "cannonball" pattern

Priority 2 (MEDIUM):
- Left lower lobe collapse — sail sign (opacity behind heart, volume loss)
- Primary TB / Ghon complex — peripheral focus + ipsilateral hilar lymphadenopathy
- Bronchiectasis — tram-track sign, ring shadows, mucus plugging
- Anterior mediastinal mass — thymoma, lymphoma, or teratoma
- Deep sulcus sign — supine pneumothorax on ICU portable AP CXR

## Search strategy:
For each gap, check these Commons categories (fetch the page, list all files, then
check promising individual file pages for licence and description):
- https://commons.wikimedia.org/wiki/Category:X-rays_of_pneumonia
- https://commons.wikimedia.org/wiki/Category:Pneumoperitoneum
- https://commons.wikimedia.org/wiki/Category:X-rays_of_atelectasis
- https://commons.wikimedia.org/wiki/Category:Pulmonary_metastases
- https://commons.wikimedia.org/wiki/Category:Bronchiectasis
- https://commons.wikimedia.org/wiki/Category:X-rays_of_tuberculosis
- https://commons.wikimedia.org/wiki/Category:Mediastinal_tumors
- https://commons.wikimedia.org/wiki/Category:X-rays_of_the_chest
- https://commons.wikimedia.org/wiki/Category:X-rays_of_pneumothorax

Also try Wikimedia Commons MediaSearch for each gap:
https://commons.wikimedia.org/w/index.php?search=TERM&title=Special:MediaSearch&type=image

If a category yields nothing useful after 2–3 file checks, note it and move on.
Do not spend more than 3 fetches on any single gap.

## Output format — for each usable candidate found:
| Gap | Commons file URL | Direct image URL | Licence | Author/Source | Journal article? | Findings | Projection | Demographics |
|-----|-----------------|-----------------|---------|---------------|-----------------|----------|------------|--------------|

If no usable image is found for a gap, state: "Gap X — no usable Commons image found.
Consider: [specific PMC article URL if found, with its licence]" so the user knows
whether to upload a CC BY image to Commons.

## Do not:
- Use Radiopaedia URLs (hotlink-blocked)
- Use NIH Open-i URLs (service discontinued)
- Use PMC blob URLs directly in the case (unstable) — flag them for Commons upload instead
- Guess the diagnosis from a filename — confirm from the file description page
- Return CT images as chest X-rays
```

---

**After the agent returns:** take the table of candidates and start a new session saying "write case for [diagnosis] using [Commons URL]". The agent's output is the sourcing brief; writing the case is a separate step.

**If the agent finds a CC BY PMC image not yet on Commons:** upload it via [commons.wikimedia.org/wiki/Special:UploadWizard](https://commons.wikimedia.org/wiki/Special:UploadWizard) using the journal article's CC BY licence, author names, and DOI as the source, then return to write the case.

---



### Case data structure

Each case is an object in the `CASES` array. The fields are:

```javascript
{
  imageUrl:    '',   // direct upload.wikimedia.org URL (no /thumb/ path)
  credit:      '',   // short attribution shown on the card, e.g. 'Wikimedia Commons — CC BY-SA 3.0'
  sourceUrl:   '',   // Wikimedia Commons file page URL
  sourceLabel: '',   // 'Wikimedia Commons'
  isNormal:    false,
  difficulty:  1,    // 1=EASY, 2=MEDIUM, 3=HARD
  title:       '',   // '67M — presenting complaint.'
  sub:         '',   // 'PA chest radiograph — [context]'
  diagnosis:   '',   // exact string — must not appear in its own findings differential
  explanation: '',   // 2–3 sentences: finding → mechanism → clinical implication
  findings:    [],   // 5 items: primary → supporting signs → notable negatives → clinical context
  annots:      [],
}
```

### Finding and verifying images

All images must come from **Wikimedia Commons** (`upload.wikimedia.org`) — it is the only source that reliably hotlinks from a browser for a static app with no backend. Radiopaedia blocks hotlinking (403); NIH Open-i URLs go dead; large ML datasets (NIH ChestX-ray14, MIMIC-CXR, CheXpert, PadChest) require bulk download and a proxy backend, which ThoraSwipe does not have.

#### Preferred sourcing strategy: published open-access papers

The richest cases come from **published open-access case reports** whose images have been uploaded to Wikimedia Commons. These give you confirmed diagnosis, exact patient demographics, expert clinical interpretation, and a citable source — far more than an anonymously-uploaded scan.

**Where to find them:**
- Search Wikimedia Commons for the pathology name — many open-access journals (BMJ Case Reports, PLOS ONE, Journal of Medical Case Reports) require CC BY, so their figures get uploaded.
- Search PubMed Central (`pmc.ncbi.nlm.nih.gov`) for the pathology + "chest radiograph" + "case report". Open-access papers have CC BY figures; check whether the CXR figure has been uploaded to Commons, or upload it yourself if the licence permits.
- The Commons category tree (`Category:Chest X-rays`) and its subcategories by condition are the fastest browsing path.

**What a paper-sourced case gives you that a generic upload does not:**
- Confirmed diagnosis (not inferred from filename)
- Patient age, sex, and presenting complaint
- Clinical context (how the diagnosis was made, what treatment was given)
- Expert interpretation to draw findings and explanation from
- Authors to credit

#### Image verification checklist

- Fetch the Commons file page to confirm: direct URL (not `/thumb/`), which side pathology is on, patient demographics, exact licence.
- Never add `crossorigin="anonymous"` to `<img>` tags — use `referrerpolicy="no-referrer"` only.
- **If the image comes from a published case report**, the `credit` field must name the actual authors (e.g. `'Wikimedia Commons — CC BY 4.0 (Herreros et al.)'`), not just the licence. Check the Commons page — the uploader and the original authors are often different people.
- **Verify the diagnosis matches what the image actually shows.** For situs-dependent findings (dextrocardia, situs inversus, organ position), confirm explicitly from the Commons description or Wikipedia usage — do not infer from the filename alone. An image described only as "cardiac apex facing right" does not confirm situs inversus totalis.
- **Never reuse the same `imageUrl` across two cases.** Grep the existing `CASES` array for the filename before adding a case.

### PA CXR anatomical convention

**Patient's RIGHT = viewer's LEFT (x < 50%)**
**Patient's LEFT = viewer's RIGHT (x > 50%)**

This applies to every label in `annots`, `findings`, `title`, and `diagnosis`. Never label by screen position.

### Annotation coordinates

```javascript
{type:'ellipse', cx:50, cy:60, rx:14, ry:10, color:'#ef4444', label:'Finding'}
{type:'arrow',   x1:30, y1:40, x2:38, y2:50, color:'#f59e0b', label:'Sign'}
```

Approximate landmarks (% of image):
- Right lung (viewer's left): x 10–45%, y 15–75%
- Left lung (viewer's right): x 55–90%, y 15–75%
- Heart: cx 40–52%, cy 50–68%
- Right costophrenic angle: x 22–28%, y 80–87%
- Left costophrenic angle: x 72–78%, y 80–87%

Colour conventions: `#ef4444` red = pathology · `#f59e0b` amber = sign · `#22c55e` green = normal · `#60a5fa` blue = landmark/variant

### Distractor pool

After adding a case, add its `diagnosis` (and any useful wrong-answer variants) to `ALL_DIAGNOSES` so it can appear as a distractor in other cases.

### Difficulty guidelines

| `difficulty` | Use when |
|---|---|
| 1 (EASY) | Classic presentation, single obvious finding |
| 2 (MEDIUM) | Requires identifying a named sign (silhouette, meniscus) or subtle finding |
| 3 (HARD) | Subtle finding AND requires clinical context integration AND broad differential |

Normal cases default to difficulty 1. Use 2 only if a normal variant (e.g. pectoralis shadows) could be mistaken for pathology.

**The named-sign rule:** If the primary teaching point is recognising a named radiological sign (silhouette sign, meniscus sign, Hampton's hump, water bottle sign, etc.), that is difficulty **2**, not 3 — even if the sign is subtle. Difficulty 3 requires the sign to be subtle *and* the diagnosis to depend on integrating clinical context. When in doubt, use 2.

---

## Writing normal CXR cases

Normal cases are harder to write well than pathological ones. The single most common failure mode is a vignette that tells the learner the answer before they look at the image.

### The vignette must not give away the diagnosis

**Never write:**
- "Routine screening. Completely asymptomatic."
- "Occupational health baseline. No symptoms."
- "Annual health review. No cardiac or respiratory history."
- "Pre-operative assessment. No respiratory complaints."
- "Clinical trial enrolment. No abnormality expected."

These are dead giveaways. A learner reads them and immediately swipes Normal without engaging with the film.

**The correct approach:** Give the patient a symptom or a referral reason that could plausibly indicate pathology, but which is also consistent with a normal CXR result. The learner should have to look at the image to decide.

### The ambiguity rule

**The vignette symptom must be explainable by a CXR-negative diagnosis.** Ask yourself: *could a real patient with these symptoms have a completely normal CXR?* If the answer is no, the hook is too alarming.

Good presentations for normal CXR cases:
- Persistent dry cough after viral URTI — GP querying early pneumonia or atypical infection
- Mild exertional breathlessness (e.g. "mild breathlessness climbing two flights of stairs") — GP excluding cardiac or pulmonary cause
- Intermittent chest tightness on exertion — no pleuritic quality, SpO₂ normal, querying cardiac/musculoskeletal
- Referred for "abnormal appearance on recent chest film" — the referral concern is what generates the ambiguity
- Occupational exposure (dust inhalation, fume exposure) with mild cough — hypersensitivity pneumonitis possible but CXR may be normal early
- Pre-operative workup with a new mild symptom — deconditioning or anxiety explains normal CXR
- Post-viral cough 3–4 weeks — classic CXR-negative diagnosis

Bad presentations (too alarming — would produce CXR findings in reality):
- Haemoptysis — always warrants CT; a normal CXR doesn't close the case and creates too much disconnect
- SpO₂ ≤ 97% — implies genuine hypoxia; hard to justify with a completely normal CXR
- Breathlessness limiting daily activities — severity implies something should be visible
- Bilateral reduced breath sounds on auscultation — a significant examination finding that usually has a CXR correlate
- Progressive dyspnoea over weeks or months that is still worsening — severity implies radiographic abnormality

### SpO₂ and vital signs for normal cases

- SpO₂ should be **98–99%** on air. Never use ≤97% for a normal CXR case.
- Heart rate should be **70–95 bpm** at most. Tachycardia (>100 bpm) implies an acute process.
- Temperature: use "Afebrile" only — never quote a fever for a normal CXR case.

### Findings array for normal cases

The five findings should annotate what IS normal, not just say "nothing abnormal". Useful slots:
1. Lung fields — explicitly confirm clear bilaterally
2. Cardiac size — CTR < 0.5 (confirm on PA film)
3. Mediastinum — trachea midline, no widening
4. Pleural spaces — sharp costophrenic angles, no effusion
5. Teaching point — what does a normal CXR NOT exclude, or what is the specific variant that could be mistaken for pathology?

Finding 5 must add genuine educational value. **Do not repeat finding 1 in different words.** Common failure: finding 1 says "Clear lungs — no consolidation" and finding 5 says "No acute pulmonary abnormality identified." These teach the same thing. Use finding 5 for the clinical teaching point: what CXR-negative diagnoses still need pursuing given the presenting symptom.

### The explanation for normal cases

The explanation should:
1. Confirm the CXR is normal (one sentence)
2. Explain the specific variant visible (if any) — e.g. pectoralis shadows, breast tissue, diaphragmatic flattening
3. State what a normal CXR does and does **not** exclude in this clinical context

Example for a cough case: *"A normal CXR does not exclude post-viral cough, upper airway cough syndrome, or GORD — all CXR-negative diagnoses. Chest radiography was indicated to exclude pneumonia, pleural effusion, and mediastinal pathology, which have been excluded here."*

Never end a normal case explanation with "Cleared for [procedure/duty]." This is a clinical outcome statement, not a teaching point.

### Normal case checklist

Before committing a new normal CXR case, confirm:
- [ ] Vignette has a symptom or referral reason that could indicate pathology
- [ ] Symptom is mild enough that a completely normal CXR is clinically plausible
- [ ] SpO₂ ≥ 98% on air
- [ ] No alarming vital signs (no tachycardia, no fever)
- [ ] Finding 5 is a genuine teaching point, not a restatement of another finding
- [ ] Explanation states what the normal CXR excludes AND what it does not exclude
- [ ] Explanation does not contain "cleared for [procedure/duty]"
- [ ] If the teaching point is a normal variant (pectoralis shadows, breast shadows, prominent thymus, etc.), difficulty is set to 2

---

## Common pitfalls

- **`credit` and `sourceUrl` must match.** If the image is on Wikimedia Commons, both must point there — not to the site where you first encountered the image.
- **Do not list the confirmed diagnosis in its own differential** in the findings array.
- **Do not include UI/implementation notes in findings text** (e.g. "(viewer's left)") — findings are clinical statements.
- **Sex-specific teaching points matter.** E.g. catamenial pneumothorax for young women, not Marfan (a male-predominant association). But do not add sex-specificity where none exists — PCD/Kartagener affects males and females equally; the sex-specific feature is male infertility (immotile sperm), not the condition itself.
- **`sourceUrl` must be a page URL, not a raw image URL.** The "View source" link in the result overlay opens this URL.
- **Never reuse the same `imageUrl` across two cases.** Before adding a case, grep the existing `CASES` array for the filename to confirm it is not already in use. Duplicate images break the educational integrity of the game — a learner who has seen the film once will recognise it immediately on a second pass.
- **The `sub` field must always specify the projection: `'PA chest radiograph — ...'` or `'AP chest radiograph — ...'`.** Omitting the projection is always wrong. Check the Commons filename or description to determine which projection was used (`pa` = posteroanterior; `ap` = anteroposterior). If the image was taken in an ICU or emergency setting, it is almost certainly AP (portable). PA is the standard upright outpatient/inpatient film.

---

## Medical writing rules (from audit)

These rules were added after a formal accuracy review. Violating them produces incorrect or misleading teaching content.

### Language
- **Never write "pathognomonic"** for a CXR finding. Virtually no plain-film appearance is pathognomonic — miliary pattern, water bottle sign, and others all have differentials. Use "highly characteristic of", "consistent with", or "classic for".
- **Use precise anatomical descriptors.** Do not write "perihilar" when you mean "paracardiac" (RML/lingular disease abuts the cardiac border, not the hilum). Common errors: RML opacity described as perihilar; lower-lobe collapse called "basal". Check lobar anatomy before writing.
- **Colloquial terms are allowed but must be paired with the proper term.** Write `lobulated bilateral lymphadenopathy ("potato nodes")`, not just `"potato nodes"`.
- **Match the source's severity language exactly.** If the Commons description says "subtotale Atelektase" (near-total atelectasis), do not write "complete" or "total" collapse throughout the case. Overstating severity is a factual error even if the distinction seems minor.

### AP vs PA projection
AP (portable/anteroposterior) films require specific caveats that PA films do not:
- **Cardiac silhouette is magnified** — the CTR cannot be used to reliably assess cardiomegaly on an AP film.
- **Mediastinum appears wider** — do not overcall mediastinal widening on AP films.
- **Scapulae project over the lung fields** — can mimic or obscure apical pathology.

Whenever a case uses an AP film, the `sub` field must say `'AP chest radiograph — ...'` and the `explanation` must note the projection and its implications (at minimum: "this is a portable AP film — cardiac silhouette is magnified").

ICU, emergency department, and resuscitation CXRs are **almost always AP**. Identify this from the Commons filename (`ap` suffix) or the clinical context before writing.

### Clinical accuracy
- **If the source image comes from a published case report, use the actual patient demographics and presentation** — do not fabricate details that contradict the source. If the source says the patient was immunocompetent, do not write a teaching point saying immunocompromised patients are at highest risk.
- **Do not claim a CXR finding alone distinguishes two conditions if echocardiography or CT is actually required.** E.g. a globular cardiac silhouette with clear lungs is seen in both pericardial effusion *and* compensated dilated cardiomyopathy — CXR cannot differentiate them; echo is mandatory.
- **Investigation recommendations must reflect current guidelines** (year of last review: 2025–26). Specific example: for sampling hilar/mediastinal lymph nodes, EBUS-TBNA is now the standard first-line procedure — not bronchoscopic transbronchial biopsy of the lung parenchyma.
- **State the limitations of supporting tests.** E.g. IGRA can be falsely negative in miliary TB due to immunological anergy; serum ACE is neither sensitive nor specific for sarcoidosis. Don't present a test as definitively useful without its caveat.
- **Do not teach folk rules or informal clinical heuristics as validated criteria.** Example: "NG tube must cross the midline" is a bedside teaching habit, not a published safety guideline. The validated criteria are aspirate pH <5.5 or confirmed gastric placement on CXR. If you are unsure whether a criterion is evidence-based, do not include it.

### ECG and non-radiographic correlates
- When listing ECG correlates of a radiographic finding, be complete. Dextrocardia inverts **all** waveforms in lead I (P, QRS, **and T**) — not just P and QRS.

### Differential diagnosis
- When listing a differential, ensure each item is a genuine **radiographic** mimic on CXR — not just a clinical differential. The learner is interpreting a film, not a symptom complex.
- **Qualify opportunistic infection differentials.** PCP (Pneumocystis jirovecii pneumonia) is only a valid differential in immunocompromised patients (HIV CD4 <200, transplant, high-dose steroids). Do not list it as a general differential for bilateral pneumonia without stating the immune status requirement.
- **Match the differential to the specific radiographic pattern, not just the lobe or density.** Example: mesothelioma is not a differential for ipsilateral white-out with *ipsilateral* mediastinal shift (which indicates volume loss/collapse). Mesothelioma causes a frozen hemithorax with variable or no shift. Ensure each differential item could actually produce the exact pattern shown — shift direction, lobar distribution, and associated signs all matter.

---

## Additional rules (from audit round 3)

### Tension vs simple pneumothorax
- **Always verify the image before writing the clinical narrative.** A CXR described as "spontaneous pneumothorax" may still show mediastinal shift or cardiac displacement consistent with tension physiology. Specifically check the Wikimedia Commons description for phrases like "Mediastinalshift" or "Spannungspneumothorax" before writing "no tracheal deviation — not under tension." An image originally created to illustrate tension PTX must never be used to teach absence of tension.

### Annotation accuracy
- **Only annotate findings that are definitively visible in the image.** If a finding (e.g. air-fluid level, subtle cavity) is mentioned as "may be present" or "often seen" in the findings text, do **not** add an annotation arrow pointing to it — annotation implies confirmation. Use `findings` text with hedging language ("may be visible", "if present") for uncertain features; reserve `annots` for unambiguous, confirmed findings.

### British English consistency
- **Use British English spellings throughout all case fields.** Key pairs: dyspnoea (not dyspnea), oedema (not edema), haemothorax (not hemothorax), haemoptysis (not hemoptysis), anaemia (not anemia). Inconsistent spelling within the same case or across cases is a quality error.

### CTR notation
- **Express cardiothoracic ratio as a decimal, not a percentage.** Write "CTR > 0.5" not "CTR > 50%". The ratio is dimensionless (transverse cardiac diameter ÷ maximum transverse thoracic diameter); expressing it as a percentage is factually incorrect. The findings array and the explanation must use the same notation.

### Credit for published case report images
- **When a Wikimedia Commons image originates from a published paper, the credit field must name the actual authors** in the format: `'Wikimedia Commons — CC BY X.X (Surname1 & Surname2, Journal Year)'`. The uploader and the original authors are often different people — always check the Commons file page "Source" field, not just the uploader name.

---

## Additional rules (from audit round 4)

### AP projection — James Heilman files
- **James Heilman MD images with the suffix `M.jpg` (e.g. `RLL_pneumoniaM.jpg`, `LLL_pneumonia_with_effusionM.jpg`) are AP films**, confirmed from their Commons descriptions. Any case using one of these files must say `'AP chest radiograph — ...'` in `sub` and must include the AP caveat in `explanation`: "Note this is a portable AP film — the cardiac silhouette is magnified and CTR cannot be reliably used."

### Diagnosis must never appear in its own differential
- **This rule is routinely violated in pneumonia cases.** "Bacterial lobar pneumonia" is not a valid differential item for a confirmed bacterial lobar pneumonia diagnosis — it IS the diagnosis. The differential slot must list genuine radiographic alternates: aspiration, pulmonary infarction, lobar collapse, organising pneumonia. Check every new case before committing.

### Every `sub` field must specify projection
- **Three cases (CXR-008, CXR-009, CXR-013) were found with `'Chest radiograph — ...'` — the projection was omitted entirely.** The projection word (`PA` or `AP`) is mandatory. Use the Commons filename suffix (`_pa_`, `_ap_`), Commons description, or clinical context (ICU/emergency → AP; outpatient/standing → PA) to determine projection.

### Differential must match the specific radiographic pattern
- **Do not include items that require cavitation, volume loss, or shift in a differential for a solid opacity without those features.** Example: "lung abscess" is only a radiographic mimic when cavitation is present — for a solid lobulated mass, the mimic should be "organising pneumonia" or "carcinoid tumour." Match every differential item to the *exact* CXR pattern, not just the clinical diagnosis category.

### Government source images: name the specific agency
- **"NIH" is an umbrella term covering many agencies.** When an image originates from the National Cancer Institute, write `'Public Domain (NCI / National Cancer Institute)'` not `'Public Domain (NIH)'`. Check the Commons page Source field for the exact originating agency.

### Anatomical specificity in findings
- **Describe mass and opacity locations to the lobe or zone level, not just "left lung field" or "right mid-zone."** Write "left upper lobe" or "right lower zone" — this is clinically meaningful (upper-lobe predominance in lung cancer, lower-lobe predominance in aspiration) and is what a radiologist would document.

---

## Additional rules (from audit round 5)

### Credit must always name the creator — not just for published papers

The existing rule only calls out published case report images. The requirement is broader: **every `credit` field must name the creator if they are identifiable on the Commons file page**, regardless of whether the image comes from a paper.

- Always open the Commons file page and check the **Author** field before writing the credit.
- For Hellerhoff images — identifiable by the naming pattern `[Description]_[age][sex]_-_CR_[projection]_-_001.jpg` (e.g. `Pektoralisschatten_im_Roentgenbild_38M_-_CR_pa_-_001.jpg`) — credit must read `'Wikimedia Commons — CC BY-SA 4.0 (Hellerhoff)'`.
- A credit that reads only `'Wikimedia Commons — CC BY-SA 4.0'` with no author name is **always wrong** unless the Commons page genuinely lists no identifiable creator.

### Normal case difficulty — pectoralis shadows are difficulty 2, not 1

The difficulty rule already states: *"Use 2 only if a normal variant (e.g. pectoralis shadows) could be mistaken for pathology."* This rule is being violated. To make it explicit:

- **Any normal case where the teaching point is a variant that mimics pathology** (pectoralis shadows, breast shadows, prominent thymus, diaphragmatic humps, etc.) must be `difficulty:2`.
- `difficulty:1` is reserved for textbook-clean normal films with no notable variants — a film where a competent student would find nothing to second-guess.

### No duplicate findings — every slot must teach something distinct

**Each of the 5 findings must be substantively different.** Before committing, read all 5 findings back to back and ask: does any one of them teach the same point as another, just reworded? If yes, that slot is wasted.

The canonical failure mode: Finding 2 says "Bilateral pectoralis shadows — normal variant" and Finding 5 says "Soft tissue density in lower zones — do not mistake for pathology." These are the same teaching point. Replace the duplicate with content that adds value — a distinguishing feature, a management pearl, a clinical caveat, or a systematic review point.

### Teaching caveats must match the case's own clinical context

**If the explanation or a finding contains a teaching caveat (e.g. "a normal CXR does not exclude…", "consider X in patients with Y"), verify the caveat is consistent with the patient described in `title`.**

- Do not write "A normal CXR in a *symptomatic* patient does not exclude pathology" when the patient is described as *asymptomatic*.
- Do not write "this is a common incidental finding in *elderly* patients" when the patient is 28.
- The patient in the `title` is reading the teaching point. If the caveat refers to a different patient type, drop the qualifier or broaden it.

### Differentials: causes are not mimics

**A disease that *causes* a condition is not a radiographic differential for that condition.**

- Alpha-1 antitrypsin deficiency *causes* emphysema — it should **not** appear as a differential item in an emphysema case. It belongs in the `explanation` as a clinical caveat: "check serum A1AT in younger or non-smoking patients."
- Likewise: sarcoidosis as a cause of pulmonary fibrosis is not a differential for a fibrosis case; TB as a cause of apical scarring is not a differential for an old TB sequelae case.
- The differential slot must contain entities that produce an **identical or near-identical CXR appearance** but have a different underlying pathology. Ask: *could a radiologist, looking only at this film, reasonably favour this alternative?* If distinguishing the two requires serum levels, biopsy, or CT rather than the plain film itself, it belongs in the explanation, not the differential.

---

## Additional rules (from audit round 6)

### "Pathognomonic" — rule already exists but is routinely violated

The existing rule ("Never write 'pathognomonic'") was violated in CXR-041, where eggshell calcification was described as "pathognomonic of silicosis." This is wrong on two counts: it violates the rule, and it is medically inaccurate (eggshell calcification also occurs in sarcoidosis and post-radiation lymphoma).

**Signs that are commonly but incorrectly called pathognomonic — never use this word for any of these:**
- Eggshell calcification → "highly characteristic of silicosis and coal worker's pneumoconiosis"
- Miliary pattern → "consistent with miliary TB, haematogenous metastases, or sarcoidosis"
- Water bottle sign → "classic for pericardial effusion, but also seen in dilated cardiomyopathy"
- Hampton's hump → "highly characteristic of pulmonary infarction"
- Air crescent sign → "highly characteristic of aspergilloma or angioinvasive aspergillosis"
- Sail sign (thymic) → "classic for normal thymus in infancy"

**Pre-commit check:** Search the text of each new case for the word "pathognomonic" before committing. If found, it is always wrong — rewrite.

### Differential: conditions causing volume loss or structural change are not mimics for solid masses

This extends the existing "differential must match the specific radiographic pattern" rule (audit round 4) with a concrete failure mode caught in CXR-041.

**The rule:** Do not include a condition in the differential if it produces a fundamentally different radiographic morphology (e.g. volume loss, cavitation, or ground-glass) rather than the solid mass/opacity pattern shown.

**The specific failure (CXR-041):** Sarcoidosis Stage IV was listed as a differential for bilateral upper zone conglomerate masses (PMF). Stage IV sarcoidosis causes upper lobe fibrosis and volume loss — not conglomerate masses. It is not a genuine radiographic mimic of PMF and was replaced with chronic berylliosis, which produces an identical plain-film appearance.

**Test before including any differential item:** Ask — *does this condition produce a solid opacity/mass at the same location, without volume loss, cavitation, or shift that the target image does not show?* If no, it belongs in the explanation as a clinical distinction, not the differential.

### ICU and lines-and-tubes cases: all visible tubes must be accounted for

**When an image is from an ICU, resuscitation, or lines-and-tubes context, every tube and line visible in the image must appear in the findings.**

Systematic ICU CXR review requires identifying: ETT, central venous line, arterial line, nasogastric/orogastric tube, chest drains, pacing wires, Swan-Ganz catheter (if present). Omitting a visible tube is a teaching error — the learner sees it in the image and gets no guidance.

**The specific failure (CXR-043):** The Commons description for ARDSSevere.png explicitly states "Person is intubated with an OG in place." Only the ET tube was mentioned in the original findings; the OG tube was omitted and has since been added.

**Before committing any ICU case:**
1. Read the Commons description for mentions of specific lines/tubes.
2. Look at the image and list every tube/line visible.
3. Verify every item on that list appears somewhere in the findings array.
4. For each tube, state the expected correct position and the consequence of malposition.

---

## Additional rules (from audit round 7)

This round audited CXR-006, 007, 009, 025, 031. The recurring theme: **the case text and annotations must be verified against the pixels of the actual image, not just against the Commons description or the diagnosis label.** Three cases asserted findings that the image does not show.

### Every annotation must overlie a finding that is actually visible at those coordinates

**The specific failure (CXR-031, Aspergilloma):** the annotation labelled "Aspergilloma" was placed over the right upper lobe, but on inspection (and after contrast-enhancing and zooming that region) the right upper lobe is clear, aerated lung — there is no fungal ball, cavity, or air crescent there. The annotation pointed at normal lung.

**The rule:** before committing any case, open the actual image and confirm that *each* annotation's `cx/cy` lands on the feature named in its `label`. If you cannot see the finding at those coordinates, the annotation is wrong — do not ship it on the assumption that "the diagnosis says it's there." This is the annotation-side counterpart to the existing "only annotate findings that are definitively visible" rule (round 3).

### Confirm laterality and lobe from the source — and if the source does not state them, do not invent them

**The specific failure (CXR-031):** the Commons file page and the original Flickr source gave *no* laterality or lobe at all ("Aspergilloma X-ray" was the entire description). "Right upper lobe" was inferred, written into the findings and annotation, and turned out not to match the image.

**The rule:** when the source does not confirm which side/lobe a finding is on, you may **not** assert one. Either confirm laterality from the image itself with high confidence, find a source that states it, or do not write a lateralised finding. Never let an unconfirmed side reach `title`, `findings`, `diagnosis`, or `annots`. (Extends the existing "confirm from the file description page, do not infer" rule.)

### Never assert mediastinal shift — or its direction — without seeing it; absent expected shift is itself the teaching point

**The specific failure (CXR-007, Pleural Effusion):** the case stated "mediastinal shift to the right — expected with massive effusion" as a confirmed finding. The trachea on the actual film is midline; there is no shift. A large effusion that does **not** push the mediastinum away should instead raise suspicion of underlying lobar collapse or a fixed/infiltrated mediastinum (e.g. malignancy) — that is the real teaching point, and it is the opposite of what the case originally taught.

**The rule:** mediastinal/tracheal shift (presence *and* direction) must be read off the image, never assumed from the diagnosis. Trace the trachea and the cardiac borders before writing any shift statement. If the expected shift is absent, teach its absence and what that implies — do not paper over it with "expected."

### Match severity language to the pixels, not just the source title

**The specific failure (CXR-007):** the Commons title says "massive," and the case escalated that to "obliterating the left hemithorax / left hemithorax obliterated." The film's left upper zone is clearly still aerated — it is a large effusion, not a whole-hemithorax white-out.

**The rule:** the existing round-1 rule ("match the source's severity language exactly") cuts both ways — the *image* is the final authority. If the source label (here, a one-line Commons title) overstates what the film shows, describe what the film shows. Do not write "obliterated / complete / total / whole-hemithorax" unless the image genuinely shows it.

### Credit must name the creator even for CC0 / public-domain images

**The specific failure (CXR-006):** the image is CC0, and the credit read only "Wikimedia Commons — CC0 Public Domain" — but the Commons Author field names **Mikael Häggström, M.D.** The round-5 rule ("name the creator if identifiable") was assumed not to apply because CC0 requires no attribution. Attribution being *legally optional* does not make it *editorially optional*: if the creator is named on the file page, name them in `credit`. This applies to CC0 and US-Government public-domain images too (name the author and, per round 4, the specific agency).

### Process note: avoid images with baked-in arrows or measurements for the swipe game

When re-sourcing, an image that already has arrows, circles, or "0.80 cm"-style measurement overlays pointing at the lesion is a poor fit for a *swipe-to-diagnose* format — it reveals the abnormality (and its location) before the learner engages. Prefer a clean film and let the app's own SVG `annots` reveal the finding *after* the user answers. (CXR-009 ships with a faint author-drawn circle; that is tolerable, but do not actively choose such images when a clean alternative exists. **CXR-001 has a baked-in circle *and* arrow plus a "SITTING" burn-in — flagged for re-source when a clean cardiogenic-oedema film is found.**)

---

## Additional rules (from audit round 8)

This round audited the full 45-case library for internal consistency and re-verified two source images against their Commons pages / pixels.

### The `sub` projection must match the projection the case body actually teaches

**The specific failure (CXR-001):** `sub` read `'PA chest radiograph …'` while the `explanation` and `findings` both described a *"portable AP film … the cardiac silhouette is magnified, CTR cannot be reliably used."* The two directly contradicted each other. Viewing the image resolved it — the film has **"SITTING" burned into the top-right corner**, confirming a portable AP acquisition, so `sub` was the error and is now `'AP chest radiograph — portable, …'`.

**The rule:** the projection word in `sub` is not decorative — it must agree with every AP/PA claim in the `explanation`, `findings`, and any CTR statement. Before committing, read the projection in `sub` and the projection implied by the body back to back; if the body invokes AP magnification caveats, `sub` must say AP (and vice-versa). Burned-in positioning labels ("SITTING", "SUPINE", "PORTABLE", "ERECT") on the image are authoritative — use them.

### Diagnosis label, annotation label, and body severity must all agree

**The specific failure (CXR-007):** audit round 7 softened the body from "massive / obliterating the left hemithorax" to "large … upper zone still aerated" and relabelled the annotation "Large L effusion" — but the `diagnosis` field was left as **"Massive Left Pleural Effusion."** The headline label then overstated the corrected text.

**The rule:** severity is expressed in three places — `diagnosis`, `annots[].label`, and the prose. When you soften (or escalate) severity in one, sweep the other two in the same edit. A `diagnosis` string is a claim like any other and must match the pixels and the body.

### Normal-CXR vital-sign limits — the one sanctioned exception (PE / acute rule-out)

The normal-case checklist requires SpO₂ ≥ 98% and no tachycardia. **CXR-035 is a deliberate, permitted exception:** its whole teaching point is *"a normal CXR does not exclude pulmonary embolism,"* which requires a vignette of pleuritic pain with mild hypoxia (SpO₂ 97%) and tachycardia (HR 104) — the physiology of a real PE. A normal-film case whose lesson is *rule-out of an acute vascular/embolic cause* may carry SpO₂ 96–97% and HR up to ~105 **only when the explanation explicitly names PE (or the acute cause) as the reason the CXR is normal**. This exception does **not** license alarming vitals on routine/incidental normal cases (screening, pre-op, post-viral cough) — those still follow the SpO₂ ≥ 98%, no-tachycardia rule. `review.html` does not lint vitals, so this judgement is on the writer.

### "NIH" is acceptable only when the source names no sub-agency

Clarifying the round-4 rule: name the specific agency **when the Commons page identifies one**. For CXR-039 (`PCPxray.jpg`) the Commons Source/Author fields name only "National Institutes of Health" with no institute (NCI/NHLBI/etc.), so `'Public Domain (NIH / National Institutes of Health)'` is correct and was left unchanged. Do not invent a sub-agency the source does not state.

---

## Pending: contact form (revisit later)

The contact page currently has a **skeleton contact form** (topic dropdown + message textarea) wired to a placeholder Formspree endpoint. The design and backend are intentionally deferred.

### Current state (as of May 2026)
- CSS classes `.cf-select`, `.cf-textarea`, `.cf-submit`, `.cf-status`, `.cf-inline-link` are defined and styled.
- HTML form is in the `#contact-page` `<div class="cp-card">` block.
- `submitContactForm()` JS function exists but calls `https://formspree.io/f/REPLACE_WITH_YOUR_FORMSPREE_ID` — this **must be replaced** before the form works.
- Legal page email mentions have been removed and replaced with `hideLegal()` links back to the contact form.

### Backend options (all free)
| Service | Limit | Notes |
|---|---|---|
| **FormSubmit** | Unlimited | No account. First submission triggers a one-time activation email to your address. Endpoint: `https://formsubmit.co/ajax/<email-or-hash>`. Hash is visible in source but doesn't expose email directly. |
| **Web3Forms** | 250/month | No account needed — get an access key at web3forms.com. Clean opaque key, no activation step. |
| **Formspree** | 50/month | Requires free account. Most polished dashboard. |

### To activate the form
1. Pick a backend (FormSubmit recommended for unlimited free tier).
2. Replace the `fetch` URL in `submitContactForm()` — search for `REPLACE_WITH_YOUR_FORMSPREE_ID`.
3. For FormSubmit: use your real email first; it converts to a hash after activation.
4. Test end-to-end, then re-sync `index.html` and push.

### Design TODOs (deferred)
- Consider whether a topic dropdown is the right UX, or whether three separate entry points (general / bug report / case suggestion) work better as distinct buttons that pre-fill the form.
- Consider adding a name/handle field (optional) so replies can be personalised.
- The form currently lives inside a `.cp-card` block — evaluate whether it fits visually once real content is in place.

---

---

## SEO audit (2026-06-06) — pending tasks

A full SEO audit was run on 2026-06-06. Items below are unfixed as of that date.

### 🔴 Bugs — fix before next push

- **`dateCreated: "2025-01-01"`** in WebApplication schema — use the actual git first-commit date if different.

~~Case count mismatch (42→44)~~ — fixed 2026-06-06.

### 🟠 On-page changes (high impact, low effort)

- **Title tag**: rewrite to front-load the primary query.
  - Current: `ThoraSwipe — Free Chest X-Ray Quiz & CXR Practice for Medical Students`
  - Target: `Free Chest X-Ray Quiz — CXR Interpretation Practice | ThoraSwipe`
  - Reason: "Free" first = CTR signal; query-first = better keyword match; brand last.

- **Meta description**: fix case count, shorten to ≤155 chars, drop PACS jargon.
  - Target (~152 chars): `Free CXR quiz — 44 real chest X-rays with annotated feedback, progressive difficulty, and no login. For medical students, junior doctors and radiographers.`

- **OG tags**: make shareable for WhatsApp/Slack study group sharing.
  - `og:title` target: `ThoraSwipe — Free Chest X-Ray Quiz (44 Real Cases)`
  - `og:description` target: `Swipe-based CXR trainer used by medical students and junior doctors. 44 real radiographs, annotated findings, PACS controls. No login.`
  - Same changes apply to `twitter:title` / `twitter:description`.

- **`Organization` sameAs**: currently `[]`. Add GitHub repo URL and any social profile URL. Even one URL helps Google cross-reference the brand entity.

- **Twitter handle**: `meta name="twitter:site" content="@thoraswipe"` — verify this X/Twitter account exists. If not, remove the tag.

- **`application-name` meta tag**: add `<meta name="application-name" content="ThoraSwipe">` — matches manifest short_name.

### 🟠 Schema additions

- **`AggregateRating`** on WebApplication and/or LearningResource — star ratings appear in SERPs and increase CTR by ~15-30%. Even a static rating based on early feedback is valid.
  ```json
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.8",
    "ratingCount": "47",
    "bestRating": "5"
  }
  ```
  Only add when you have real user feedback to back up the numbers.

- **`author` Person schema** on WebApplication — critical for E-E-A-T on medical content. Google classifies CXR interpretation as YMYL (Your Money or Your Life). Anonymous medical tools are actively downgraded.
  ```json
  "author": {
    "@type": "Person",
    "name": "YOUR NAME",
    "jobTitle": "Medical Doctor / Radiologist / etc.",
    "affiliation": { "@type": "Organization", "name": "Your institution" }
  }
  ```

### 🟡 Content additions (medium impact)

- **American English variants**: the site uses correct British English (oedema, haemothorax, haemoptysis, anaemia) but US searchers type "edema", "hemothorax", "hemoptysis", "anemia". Add parenthetical US variants once per key term in the noscript section and FAQ answers: e.g. "pulmonary oedema (pulmonary edema)". This doubles keyword surface without duplicate pages.

- **"What does CXR stand for?" FAQ**: high-volume, low-competition featured-snippet opportunity. Add as FAQ #12:
  - Q: "What does CXR stand for?"
  - A: "CXR stands for chest X-ray (also written chest xray). It refers to a plain-film radiograph of the thorax used to assess the lungs, heart, mediastinum, pleura, and bony thorax. CXR is the standard abbreviation used in clinical notes, referral letters, and radiology reports worldwide."

- **Additional FAQ questions** not yet covered:
  - "What does cardiomegaly look like on a chest X-ray?"
  - "What is the silhouette sign in radiology?"
  - "What does TB look like on a chest X-ray?"
  - "How do you identify a pleural effusion on CXR?" (more direct than the current phrasing)
  - "What is Hampton's Hump?"

- **Noscript H1 variant**: add "chest xray" (no hyphen) once in the opening paragraph — one of the highest-volume search variants and currently absent from all crawlable content.

### 🟡 Technical

- **LCP preload**: the first CXR image loads via JS `img.onload`, making it undiscoverable by the browser preloader. This likely causes a slow LCP score (Core Web Vitals). Fix: add `<link rel="preload" as="image" href="[first-case-image-url]">` in `<head>` for the first case in the queue.

### 🔵 Content strategy (longer term — highest ranking ceiling)

- **Pathology landing pages**: create static HTML pages for the top 10 pathologies (pneumothorax, pleural effusion, pneumonia, pulmonary oedema, etc.). Each page = ~400 words of clinical content + CTA to the quiz. These are independently indexable and compete directly with Radiopaedia on long-tail queries like "pneumothorax chest x-ray quiz".

- **"How to Read a Chest X-Ray" guide**: ~12k searches/month globally. A free, structured HTML guide covering ABCDE, with a CTA to the quiz, would rank and funnel traffic. Biggest single content opportunity.

- **Off-page / backlinks**: domain authority is the core gap vs Radiopaedia (DA 72) and Wikipedia. Priority targets: Reddit r/medicalstudents, r/radiology, The Student Room, Almostadoctor.co.uk, BMA Student resources, medical school Moodle link-sharing, NHS learning resource directories. A single high-DA backlink from an NHS or university domain would be transformative.

---

## Admin case-review dashboard (`review.html`)

`review.html` is a standalone, unlisted admin tool (deployed at `/review`, `noindex`) for auditing the case library. It is **not** part of the game and is **not** in the deploy-checklist `index.html` sync — it ships as its own route.

- **Data source:** it does **not** duplicate `CASES`. On load it `fetch('/')`s the live homepage and extracts the `CASES` array (always in sync, read-only). It never needs editing when you add cases — it re-reads the source.
- **Editing:** click **Connect folder to edit** and pick the **`CXR Swipe` project folder**. Uses the browser **File System Access API** (Chrome/Edge, https or localhost). The serializer reproduces the repo's single-quote style and round-trips all 45 cases byte-identically (verified); before any write it re-parses the whole file and aborts if the case count changes. Safari/Firefox lack the API → read-only.
- **Save is one-click publish-ready.** Each Save: splices the edit into `cxr-swipe-v7.html` → bumps its `dateModified` to today → writes the identical text to `index.html` → bumps `sitemap.xml` `<lastmod>` to today → reloads. That covers deploy-checklist steps 1–3 automatically. Editing an existing case never changes the case count, so the count/`TS_SAVE_KEY` steps never apply. **The only remaining step is `git push`** — a web page cannot run Git, so review with `git diff` and have Claude (or your terminal) commit & push.
- **Lint flags** on each card encode CLAUDE.md rules: `pathognomonic` usage, missing PA/AP projection, findings count ≠ 5, diagnosis appearing in its own findings, credit with no named author, US spellings, annot coords outside 0–100, duplicate `imageUrl`. Use the **Flagged only** toggle to triage.
- **Annotations** are drawn on each film (SVG overlay, percent coords) so you can eyeball whether each `annot` overlies its finding — the audit-round-7 check.
- **Caveat:** it carries a `noindex` meta tag and is unlinked/not in the sitemap; the global `vercel.json` CSP/`X-Robots-Tag` were left unchanged (the meta tag is the authoritative noindex signal).
- **CSP gotcha:** the site CSP has no `'unsafe-eval'`, so `eval`/`new Function` are blocked on the deployed site. Both `review.html` and `library.html` parse the extracted `CASES` array with a small hand-written **literal parser** (`parseArray`/`parseLiteral`) — not eval. The parser skips `//` and `/* */` comments (the `CASES` array contains `// ── MEDIUM ──` section dividers) and tolerates trailing commas; verified to deep-equal the old eval output for all 45 cases. If you ever need to parse the array in a new tool, reuse this parser — do not reach for `new Function`.

## Shareable read-only case library (`library.html`)

`library.html` (deployed `/library`, `noindex`) is the **public, view-only** version of the review dashboard — meant to be shared by link. Same card rendering and annotations as `/review`, plus search/difficulty/normal filters and an annotations on/off toggle (hide to self-test). **No editing surface at all** (no folder connect, no serializer, no lint flags). Reads live `CASES` via `fetch('/')`. Includes a "Try the quiz" link back to the app and an educational-use disclaimer footer.

## Case audit log (`audit.html`)

`audit.html` (deployed `/audit`, `noindex`) is a **fully offline, self-contained** record of the medical audits Claude has run on the case library — it has **no `fetch`** and bakes its data inline, so it opens straight from disk (`file://`). It is **not** generated from `CASES`; its content is compiled by hand from the audit history in this file (audit rounds 1–7 + the dated audit sessions). Three parts: summary stats (cases audited / clean / fixed / removed); the standing "what gets checked" checklist distilled from the seven audit rounds; and a per-case log (id, diagnosis, outcome badge, when, what was *checked*, what was *changed*). Search + outcome filter. **When you audit more cases, update the `AUDITS` array in `audit.html`** to keep it current — it does not auto-sync.

---

## Preview server

A local preview server is configured in `.claude/launch.json` (project root, not CXR Swipe subdirectory):

```json
{
  "name": "thoraswipe",
  "runtimeExecutable": "npx",
  "runtimeArgs": ["serve", "-l", "7823", "."],
  "cwd": "/Users/yashverma/Documents/Cowork Playground/App Ideas/CXR Swipe",
  "port": 7823
}
```

Start via `preview_start("thoraswipe")` in Claude Code. Serves the CXR Swipe directory at `http://localhost:7823`. Use `preview-beauty.html` (or any temp file) for visual experiments before touching the main file. Always delete temp files before committing.

---

## Session log

### 2026-07-06 (full-library consistency audit + 6 fixes)

**Read all 45 cases end-to-end and audited for internal consistency; verified two source images.** Fixed six defects (case count unchanged at 45; no `TS_SAVE_KEY` bump):

- **CXR-001 (Pulmonary Oedema)** — `sub` said "PA" while the body taught a portable AP film with CTR caveats. Viewed the image: it has **"SITTING" burned into the corner** → confirmed portable AP. Changed `sub` → `'AP chest radiograph — portable, cardiac assessment'` and reworded the explanation to cite the on-image annotation. (Image also has a baked-in circle+arrow — flagged for re-source in the process note.)
- **CXR-024 (Left Tension Pneumothorax)** — removed UI note `"(patient's left = viewer's right)"` from finding 1 (no implementation notes in findings; the identical note was stripped from the mesothelioma case in June but this one was missed).
- **CXR-007 (Left Pleural Effusion)** — `diagnosis` still read "Massive…" after round 7 softened the body to "large"; changed to **"Large Left Pleural Effusion"** so diagnosis / annotation / prose agree.
- **CXR-042 (SVC Syndrome)** — its diagnosis was absent from `ALL_DIAGNOSES`; added `'Superior Vena Cava Syndrome — Right Lung Carcinoma'` + `'SVC Obstruction — Mediastinal Mass'` to the distractor pool.
- **CXR-039 (PCP)** — re-checked the Commons page; it names only "National Institutes of Health" (no institute), so the `NIH` credit is correct — **left unchanged**.
- **CXR-035 (Normal AP)** — SpO₂ 97% / HR 104 violate the normal-case vitals checklist but are pedagogically required (teaching "normal CXR does not exclude PE"). **Kept the case**; documented a sanctioned exception in CLAUDE.md instead.

**CLAUDE.md:** added **"Additional rules (from audit round 8)"** — `sub` projection must match the body's AP/PA claims (burned-in positioning labels are authoritative); diagnosis + annotation + prose severity must agree; the one permitted normal-case vitals exception (PE / acute rule-out); and clarified that "NIH" is acceptable when the source names no sub-agency.

**Non-defects noted (not changed, judgement left to owner):** redundant teaching pairs (CXR-006 & CXR-019 both RML-pneumonia/silhouette; CXR-003 & CXR-038 both normal breast-shadows); difficulty skew (7 easy / 21 medium / 17 hard, 10 normal); open coverage gaps (lingular pneumonia, malpositioned ETT, cannonball metastases, bronchiectasis, primary TB/Ghon, anterior mediastinal mass, deep sulcus sign, aspergilloma re-source).

**Deploy checklist:** `index.html` re-synced; `dateModified` + `sitemap.xml` lastmod → 2026-07-06. No count change → no `TS_SAVE_KEY` bump. Not yet committed/pushed (awaiting go-ahead — push auto-deploys live).

### 2026-06-16 (offline case audit log)

**Added `audit.html`** (`/audit`, `noindex`) — fully offline, self-contained page recording every case Claude has audited and what was checked/changed. Data compiled by hand from this file's audit history (rounds 1–7 + dated sessions); **20 cases** logged: 3 verified clean (CXR-039/040/042), 16 fixed, 1 removed (CXR-031). Includes the distilled "what gets checked" checklist. No `fetch`/`eval` — opens from disk. Update the `AUDITS` array when more cases are audited. Diagnosis labels pulled from the live source so they match exactly.

### 2026-06-16 (shareable library + CSP-safe parser fix)

**Added `library.html`** (`/library`, `noindex`) — public, read-only, shareable case browser. Same annotated cards/filters as `/review` minus all editing and lint internals; links back to the quiz; educational-use disclaimer.

**Fixed CSP breakage:** both `review.html` and `library.html` originally parsed the `CASES` array with `new Function(...)`, which the site CSP blocks (no `'unsafe-eval'`) — the deployed pages errored with *"Evaluating a string as JavaScript violates… 'unsafe-eval'"*. Replaced eval with a hand-written literal parser (`parseArray`/`parseLiteral`) that skips `//`/`/* */` comments and trailing commas; verified in Node to deep-equal the old eval output for all 45 cases. See the CSP gotcha note in the dashboard section.

**Current state (end of 2026-06-16 sessions):**
- Total cases: **45**; `TS_SAVE_KEY`: `ts_progress_v7` (unchanged — no cases added/removed this session).
- New routes/files: `review.html` → **`/review`** (admin, editing) and `library.html` → **`/library`** (shareable, read-only); both `noindex`, unlinked, not in sitemap. Neither is part of the `index.html` deploy sync.
- `dateModified` / `sitemap.xml` lastmod: **2026-06-16**. `index.html` in sync with `cxr-swipe-v7.html`.
- All work committed and pushed to `origin/main`; Vercel auto-deployed.

### 2026-06-16 (admin case-review dashboard)

**Added `review.html`** — standalone unlisted admin tool (deployed `/review`, `noindex`) to review/audit all cases at once. See the "Admin case-review dashboard" section above for full behaviour. Key points: reads live `CASES` via `fetch('/')` (no data duplication); File System Access API editing; per-card lint flags encode CLAUDE.md rules; annotations drawn on each film. **Save is one-click publish-ready**: writes the edit to `cxr-swipe-v7.html`, syncs `index.html`, bumps `dateModified` + `sitemap.xml` to today (deploy-checklist steps 1–3); only `git push` remains. Editing connects to the **project folder** (not a single file) so it can reach all three files.

Verified via Node against the real file (browser preview unavailable — sandboxed preview server falls back to Python `http.server` → `PermissionError`): parse/scan alignment 45/45; serializer round-trips all 45 cases byte-identical; `dateModified` (1) and `sitemap` `<lastmod>` (2) regex bumps confirmed; full Save simulation re-parses to 45 with the edit applied, other cases intact, and divergence only at the edited region. JS syntax-checked. Does not touch the game.

### 2026-06-16 (committed pending accuracy/credit fixes)

**Committed and pushed previously-uncommitted working changes** (15 edits across 9 cases; case count unchanged at 45):

- **CXR-001 (Pulmonary Oedema)** — reframed from PA "cardiomegaly / CTR > 0.5" to **portable AP**: CTR unreliable on the magnified silhouette, added echo correlation, added a cardiogenic-vs-ARDS distinguishing finding. Annotation label "Cardiomegaly" → "Cardiac silhouette".
- **CXR-003 (Normal)** — patient **45M → 45F** (hernia repair → laparoscopic cholecystectomy), **difficulty 1 → 2**, teaching point reframed around **breast shadows** mimicking basal consolidation. Credit now names **Mikael Häggström**.
- **Credit creator names added** (audit round-5 rule): CXR-004 (Stillwaterising), CXR-010/011/013 (Hellerhoff), CXR-014 (Doctoroftcm), CXR-015 (SCiardullo).
- **Mesothelioma case** — removed UI note `"(patient's right = viewer's left)"` from a finding (no implementation notes in findings).

**Deploy checklist:** `index.html` synced (byte-identical), `sitemap.xml` lastmod + `dateModified` → 2026-06-16. No `TS_SAVE_KEY` bump (count unchanged). Committed `36f883d`, pushed to origin/main — Vercel auto-deploys.

**Note:** preview server (`preview_start("thoraswipe")`) failed this session — sandboxed Python `http.server` returns `PermissionError: Operation not permitted`. Verified instead by static check (45 caseIds intact, edited lines quote-clean). Untracked files left alone: `.claude/`, `CXR-Review-039-043.docx`, `cases-by-pathology.md`, `logo-showcase.html`, and a `~$` Office temp lock file.

### 2026-06-11 (medical audit of 5 random cases + CXR-031 removal)

**Audited CXR-006, 007, 009, 025, 031** by viewing each radiograph (downloaded + contrast-enhanced the regions in question) and cross-checking every Commons file page for diagnosis, laterality, projection, licence, and author. Two cases asserted findings the image does not show.

- **CXR-006 (RML pneumonia)** — accurate; Commons confirms 67M, right middle lobe. Fix: `credit` now names the creator **Mikael Häggström** (CC0 still warrants editorial attribution).
- **CXR-007 (left pleural effusion)** — removed the false **"mediastinal shift to the right"** claim (trachea is midline on the film; verified by cropping the upper mediastinum) and reframed it as the *absent-shift* teaching point (suspect underlying collapse / fixed mediastinum). Softened "obliterating the left hemithorax" → "large… upper zone still aerated" (it is not a whole-hemithorax white-out). Credit now names **Clinical Cases / Ves Dimov MD**. Differential converted from *causes* of effusion to *radiographic mimics* of a unilateral white-out.
- **CXR-009 (LLL pneumonia + effusion)** — verified clean; AP correctly identified (portable markers visible), licence/author correct.
- **CXR-025 (cavitary TB)** — accurate; cavitation annotations nudged cy:22 → cy:32 onto the densest mid-zone disease.
- **CXR-031 (aspergilloma)** — annotation pointed at **clear, aerated right upper lobe**; the Commons/Flickr source gives **no laterality** (it was inferred). Commons has only two plain-film aspergillomas and neither is clean for a swipe-to-diagnose deck (the other has answer-revealing arrows + coned framing). **Case removed pending re-source** (user decision).

**CXR-031 removal bookkeeping:** count **46 → 45** across meta description, JSON-LD `featureList`, LearningResource description, OG/Twitter tags, `<noscript>` paragraph, and the "Pneumonia & Pulmonary Infection (8→7 cases)" heading; **manifest.json** also corrected (was stale at 44). Removed the aspergilloma `<li>` from the noscript case list **and** the "Air crescent sign" entry from the radiographic-signs glossary (no remaining case teaches it). **Kept** `Aspergilloma` / `Invasive Pulmonary Aspergillosis` in `ALL_DIAGNOSES` — still useful distractors for cavitary/haemoptysis cases. **`TS_SAVE_KEY` v6 → v7** (case count changed).

**CLAUDE.md:** added **"Additional rules (from audit round 7)"** — verify each annotation overlies a finding visible at its coordinates; confirm laterality from the source and never invent it; never assert mediastinal shift (or direction) without seeing it; match severity language to the pixels, not the source title; credit creators even for CC0 / public-domain images; avoid images with baked-in arrows/measurements for the swipe game.

**Deploy checklist:** `index.html` synced (byte-identical); `sitemap.xml` lastmod + `dateModified` → 2026-06-11. Verified in preview — 45 cases parse, no console errors, root URL renders.

**NOT yet committed/pushed** — awaiting user go-ahead (push auto-deploys to live thoraswipe.com). **Pending:** re-source a clean CC BY aspergilloma plain film (air crescent / Monod sign), then re-add the case and reverse the count/glossary changes.

### 2026-06-06 (new case + gap analysis session)

**Gap analysis completed** — 44 cases audited, gaps identified across 8 categories. Full gap list in the previous chat session (not reproduced here). Top priorities: lingular pneumonia, malpositioned ETT, pulmonary fibrosis, pneumoperitoneum, pulmonary metastases.

**Added CXR-046: Amiodarone-Induced Pulmonary Toxicity** (difficulty 3, ILD category)
- First case in new "Interstitial & Fibrotic Lung Disease" category — fills the entire ILD gap
- Image: James Heilman MD, CC BY-SA 3.0, `IPF_amiodarone.JPG`
- PA CXR: bilateral diffuse coarse reticulonodular opacification, mid-lower zone predominance, necklace artefact
- Teaching: drug-induced ILD vs cardiogenic oedema vs IPF/UIP; DLCO as functional marker; amiodarone half-life and mechanism
- 5 ILD-related distractors added to ALL_DIAGNOSES
- TS_SAVE_KEY bumped v4 → v5; case count 44 → 45 across all metadata

**ETT malposition & lingular pneumonia**: images not currently on Wikimedia Commons.
- ETT malposition: Voucharas et al., *Cureus* 2024, PMC10981442 — CC BY 4.0, 67M post-cardiac surgery — user must upload Figure 3 to Wikimedia Commons before case can be added.
- Lingular pneumonia: no suitable Commons image found after extensive search. Needs new sourcing.

**Current state:**
- Total cases: **45** (CXR-001 to CXR-046, CXR-012 intentionally missing)
- `TS_SAVE_KEY`: `ts_progress_v5`
- Next case ID: **CXR-047**



### 2026-06-06 (SEO fixes session)

**Fixed SEO items 1–3 from audit:**
- **Title tag** rewritten to query-first: `Free Chest X-Ray Quiz — CXR Interpretation Practice | ThoraSwipe`
- **Meta description** trimmed to ≤155 chars, PACS jargon removed
- **`dateCreated`** in WebApplication schema corrected from `2025-01-01` to `2026-05-24` (actual first git commit date)
- `index.html` synced; committed and pushed.

**Pending SEO items:** 4–17 from audit list (OG/Twitter tags, schema additions, content, technical, content strategy)



### 2026-06-02

**What was done across two sessions (second session recovered from a crashed first):**

- **Added 5 new cases** (CXR-039 to CXR-043), all difficulty 3 (HARD):
  - CXR-039: Pneumocystis jirovecii Pneumonia (PCP)
  - CXR-040: Right Pancoast Tumour — Superior Sulcus Carcinoma
  - CXR-041: Complicated Silicosis — Progressive Massive Fibrosis
  - CXR-042: Superior Vena Cava Syndrome — Right Lung Carcinoma
  - CXR-043: Acute Respiratory Distress Syndrome (ARDS)

- **Updated case count** from 37 → 42 in:
  - JSON-LD `featureList[0]`
  - `<noscript>` paragraph and category headings
  - This CLAUDE.md deploy checklist

- **Bumped `TS_SAVE_KEY`** from `ts_progress_v2` → `ts_progress_v3` (required when case count changes)

- **Updated metadata**: `dateModified` and `sitemap.xml` `<lastmod>` both set to `2026-06-02`

- **Synced `index.html`** from `cxr-swipe-v7.html`

- **Contact form**: still deferred — placeholder endpoint `REPLACE_WITH_YOUR_FORMSPREE_ID` not yet replaced

**Current state:**
- Total cases: **42** (CXR-001 to CXR-043)
- `TS_SAVE_KEY`: `ts_progress_v3`
- All changes committed and pushed; Vercel auto-deployed

### 2026-06-02 (audit session)

**In-depth medical audit of CXR-039 to CXR-043. Three fixes applied:**

- **CXR-039 (PCP)**: No issues — verified clean ✅
- **CXR-040 (Pancoast)**: No issues — Commons confirms 47F, right lung, NSCLC, smoker ✅
- **CXR-041 (Silicosis), Finding 3**: Removed "pathognomonic" (CLAUDE.md violation + medically wrong — eggshell calcification also occurs in sarcoidosis and post-radiation lymphoma). Replaced with "highly characteristic of silicosis and coal worker's pneumoconiosis."
- **CXR-041 (Silicosis), Finding 5**: Replaced sarcoidosis Stage IV (not a genuine radiographic mimic of PMF) with **chronic berylliosis**, which produces an identical plain-film appearance indistinguishable by CXR.
- **CXR-042 (SVC Syndrome)**: No issues — verified clean ✅
- **CXR-043 (ARDS), Finding 3**: Added OG tube (visible in image per Commons description) alongside ET tube; added teaching point about expected OG tip position below the left hemidiaphragm.

**Contact form**: still deferred — `REPLACE_WITH_YOUR_FORMSPREE_ID` not yet replaced.

### 2026-06-03 (new cases session)

**Added 2 new cases (CXR-044–045). Case count 42 → 44. TS_SAVE_KEY v3 → v4.**

- **CXR-044**: Chilaiditi Syndrome — Hepatic Flexure Colonic Interposition
  - 85F, difficulty 2, PA CXR, isNormal: false
  - Hellerhoff, CC BY-SA 4.0
  - Teaches: bowel gas above right hemidiaphragm (Chilaiditi sign) vs pneumoperitoneum; haustral folds as the discriminator; no acute peritonism
  - Image: `Chilaiditi-Syndrom_bei_Zwerchfellhochstand_rechts_85W_-_CR_pa_-_001.jpg`

- **CXR-045**: Traumatic Diaphragmatic Rupture — Left Intrathoracic Spleen
  - 32M, difficulty 3, PA CXR, isNormal: false
  - Hariharan et al., BMC Gastroenterol 2006, CC BY 2.0
  - Teaches: loss of left hemidiaphragm silhouette; delayed post-trauma presentation (missed as pleural effusion acutely); haematemesis as presenting feature of intrathoracic herniation
  - Image: `Diaphragmatic_rupture_spleen_herniation.jpg`

**Note:** These cases originated from a search for "poor inspiration" CXRs. Wikimedia Commons has no images explicitly labeled as poor inspiration — this was confirmed after exhaustive search. The diaphragm/subdiaphragmatic category was substituted as the closest teachable topic with confirmed Commons sourcing.

**Deploy checklist run:** dateModified → 2026-06-03, sitemap lastmod → 2026-06-03, noscript updated with new "Diaphragm & Subdiaphragmatic (2 cases)" category, 8 new distractor diagnoses added to ALL_DIAGNOSES.

### 2026-06-02 (SEO session)

**Full SEO optimisation pass. Changes across 4 files:**

- **LearningResource schema** added — makes ThoraSwipe eligible for Google's educational rich results; lists 13 specific skills taught
- **Organization schema** added — establishes brand in Google's Knowledge Graph
- **FAQPage expanded** from 6 → 11 questions, adding: pleural effusion CXR, pulmonary oedema signs, ABCDE approach, pneumonia on CXR, how to learn chest X-ray reading
- **Meta description** updated to mention 42 cases and no-login requirement (CTR signal)
- **Keywords meta** expanded with long-tail terms (ABCDE, OSCE, foundation doctor, etc.)
- **hreflang tags** added (`en` + `x-default`)
- **Noscript section** expanded with 3 new sections: how it works (3-step sequence), 10 key radiographic signs with descriptions, who it is for (5 audience types) — this is the primary crawlable content for a JS-heavy app
- **manifest.json** updated: added `id`, `lang`, `dir`, `categories: ["medical","education"]`, expanded description
- **vercel.json** updated: added `X-Robots-Tag` response header

### 2026-06-06 (UI polish + analytics + SEO audit session)

**Vercel Web Analytics enabled:**
- Added `<script defer src="https://cdn.vercel-insights.com/v1/script.js">` before `</body>`
- Updated CSP in both HTML meta tag and `vercel.json` to allow `cdn.vercel-insights.com` (script-src) and `vitals.vercel-insights.com` (connect-src)
- Updated legal/privacy page: replaced "no analytics" bullet with accurate Vercel Web Analytics disclosure (cookieless, no personal data)

**UI/design polish — 11 items across 5 commits:**
- High priority: font-size floor raised to 12px (card-sub, findings list, hint); Normal/Abnormal button contrast increased; button icon symmetry (both lead with symbol); `?` header button replaced with SVG info-circle icon
- Medium priority: score pill restructured to `✓ 0 | ✕ 0`; difficulty filter chips 9→10px with hover/active states; PACS invert `⊡` replaced with half-circle SVG
- Beauty items 1–3: ambient background gradient strengthened (7→14% opacity) + warm red accent; SVG noise texture overlay (4% opacity); swipe card gets subtle top-lit gradient + lighter top border; image area bottom fade; blue CTA buttons glow on hover; Normal/Abnormal buttons glow green/red on hover
- Beauty items 4–6: verdict icon 40→52px with green/red glow; `v-text p` 11→12px; "Round Complete" heading gets blue→green gradient text; score box pulses blue on entry; score number count-up animation (0→final, 650ms ease-out cubic); card-title 13→14px/700; difficulty dots 6→7px with gold glow on active
- Beauty items 7–9: findings `→` arrow replaced with CSS counter 01–05 in Space Mono; gap 5→8px; findings box gets blue left-accent border; progress bar 4→5px; `progPulse` keyframe fires on each case advance
- Beauty items 10–11: `pillPulse` keyframe on score pill fires on every `updateScore()` call; contact page redundant logo/wordmark hero removed

**SEO audit run** — see "SEO audit (2026-06-06)" section above for full findings and pending tasks. Key bugs identified: meta description and LearningResource schema still say "42 cases" (should be 44); manifest.json also says 42.

**Preview server configured** in `.claude/launch.json` at project root — see "Preview server" section above.

**Current state:**
- Total cases: **44** (CXR-001 to CXR-045, CXR-012 intentionally missing)
- `TS_SAVE_KEY`: `ts_progress_v4`
- Vercel Analytics: **live**
- All changes committed and pushed; Vercel auto-deployed
- **SEO bugs pending** (42→44 count in meta description, LearningResource, manifest)
