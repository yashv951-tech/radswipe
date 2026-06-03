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
   - JSON-LD `featureList[0]`: `"42 real chest X-ray cases from easy to expert difficulty"` (update the number)
   - `<noscript>` paragraph: `"ThoraSwipe includes 42 real chest X-ray cases"` (update the number)
   - Also add any new diagnoses to the `<noscript>` case list under the correct category heading

**After every commit, always run `git push` immediately.**

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

## Adding a new case

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

- Use only **Wikimedia Commons** (`upload.wikimedia.org`). Radiopaedia hotlink-blocks; NIH Open-i URLs go dead.
- Fetch the Commons file page to confirm: direct URL (not `/thumb/`), which side pathology is on, patient demographics, exact licence.
- Never add `crossorigin="anonymous"` to `<img>` tags — use `referrerpolicy="no-referrer"` only.
- **If the image comes from a published case report**, the `credit` field must name the actual authors (e.g. `'Wikimedia Commons — CC BY 4.0 (Herreros et al.)'`), not just the licence. Check the Commons page — the uploader and the original authors are often different people.
- **Verify the diagnosis matches what the image actually shows.** For situs-dependent findings (dextrocardia, situs inversus, organ position), confirm explicitly from the Commons description or Wikipedia usage — do not infer from the filename alone. An image described only as "cardiac apex facing right" does not confirm situs inversus totalis.

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

## Session log

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
