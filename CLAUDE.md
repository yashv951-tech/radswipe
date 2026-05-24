# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

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
| 3 (HARD) | Subtle, requires clinical context integration, broad differential |

Normal cases default to difficulty 1. Use 2 only if a normal variant (e.g. pectoralis shadows) could be mistaken for pathology.

---

## Common pitfalls

- **`credit` and `sourceUrl` must match.** If the image is on Wikimedia Commons, both must point there — not to the site where you first encountered the image.
- **Do not list the confirmed diagnosis in its own differential** in the findings array.
- **Do not include UI/implementation notes in findings text** (e.g. "(viewer's left)") — findings are clinical statements.
- **Sex-specific teaching points matter.** E.g. catamenial pneumothorax for young women, not Marfan (a male-predominant association).
- **`sourceUrl` must be a page URL, not a raw image URL.** The "View source" link in the result overlay opens this URL.
