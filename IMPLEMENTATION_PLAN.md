# Implementation Plan — The Best Is Yet to Come

A record of each development phase for the Three.js birthday journey.

---

## Phase 1: Three.js Environment (committed on feature/stability-navigation)

- Three.js r0.161.0 (ES module, jsDelivr CDN).
- CatmullRomCurve3 spline, 241 control points, Y=0 road.
- ACESFilmicToneMapping, PCFSoftShadowMap, SRGBColorSpace.
- First-person camera following the spline.
- HUD with hero title, prev/next controls, progress dots.
- Five configurable milestone stops driven by content.json.
- Marker rings at each stop.

---

## Phase 2: Scenery and Performance

- InstancedMesh for trees (5 batches) and mountains (6 batches).
- Mulberry32 seeded PRNG for deterministic layout.
- Target: ~21 draw calls, ~18,000 triangles.
- Fog, vignette, sky gradient.

---

## Phase 3: Mountain Road Elevation (reverted)

- Elevated terrain with curve elevation was implemented (commit 02ac0bc)
  but reverted (commit f26f494) — the flat winding road was preferred.
- History preserved on branch.

---

## Phase 4: Milestone Storytelling Panels (commit 7c8d04a)

- Side-panel design on desktop, bottom-sheet on mobile.
- Paragraph splitting (blank-line separated).
- Subtitle display, hidden-when-empty Telugu section.
- Noto Sans Telugu font via Google Fonts.
- Focus management, backdrop click, panelReturnBtn.

---

## Phase 5a: Photograph Gallery (commit 8342af4)

- One-photo-at-a-time slideshow inside the milestone popup.
- object-fit:contain, 4:3 aspect-ratio stage (no layout shift).
- Prev/Next arrows, caption, counter.
- Lightbox (z-index:50) with full-screen view, arrow key nav, swipe.
- Broken-image fallback placeholder.
- fillGallery() and openLightbox() infrastructure.

---

## Phase 5b: Popup Restoration + Future Experiences

### Part A — Restored Centred Cinematic Popup (commit 335aaef)

- Replaced side-panel/bottom-sheet with original centred popup design.
- Dark navy gradient, warm gold border, 24px border-radius, deep shadow.
- Fade + scale(0.96) → scale(1) animation on both open and close.
- prefers-reduced-motion disables all transitions.
- Mobile: max-height:90dvh, safe-area padding, 18px border-radius.
- All IDs preserved: mEyebrow, mTitle, mSubtitle, mBody, mTelugu,
  photoGallery, futureGallery, closeBtn, continueBtn, modalWrap.
- Backdrop click (outside .modal article) closes popup.

### Part B — Future Experiences Gallery (commit 8a4de41)

- Five new futureScenes entries in content.json with enabled flag and caption.
- Stop 4 (beyond-work): futureScene = "unhurried-mornings".
- Stop 5 (future): futureScene = "all" (full gallery).
- openModal() respects enabled !== false; uses src||image and caption||title.
- Five SVG placeholder illustrations in assets/future/:
    family-adventures.svg, unhurried-mornings.svg,
    rediscovering-interests.svg, more-family-memories.svg, years-ahead.svg.

### Part C — Documentation (commit 106bb18)

- README.md created.
- FUTURE_IMAGE_PROMPTS.md created.

---

## Phase 6: Birthday Letter Overlay (commit 1f09858)

A dedicated birthday-letter experience at the final destination.

### What was built

Letter overlay (#letterWrap / .letter):
- Centred, z-index:7 (above popup 5, below lightbox 50).
- Dark warm-navy gradient, gold border, 22px border-radius.
- Fade + scale animation with prefers-reduced-motion support.
- Header: ornament, title, dateline, centred gold rule.
- Body: Georgia serif, 1.92 line-height for comfortable reading.
- Telugu closing line (Noto Sans Telugu, centred, hidden if empty).
- Signature line: right-aligned italic.
- Action buttons: "Return to Final Milestone" + "Restart Journey" (primary).

### Content source

- Letter body: content.site.finalLetter (never hardcoded).
- Telugu: content.site.finalTelugu (hidden if empty).
- Paragraph splitting: identical to milestone popup (/\n{2,}/).

### Interactions

- "Read Your Birthday Letter" button (letterBtn): appears only at Stop 5.
- Opens letter overlay, closes milestone popup.
- "Return to Final Milestone": closes letter, re-opens milestone popup.
- "Restart Journey": closes letter, travels camera to Stop 1.
- Escape: lightbox > letter > milestone popup (priority order).
- Backdrop click: clicking outside .letter article closes letter.
- Raycaster: ignores canvas clicks when letter is open.

### Changes to openModal()

- Removed isFinal story concatenation (letter no longer in popup body).
- Removed isFinal Telugu special-case (Telugu lives in letter overlay).
- letterBtn shown only on final stop; continueBtn hidden on final stop.

### Structural checks

48/48 assertions pass (46 automated, 2 verified manually).

### Known limitations

- Browser testing unavailable in this environment (no browser automation).
  Manual verification required using `python -m http.server 8080`.
- Telugu rendering depends on Noto Sans Telugu Google Font loading
  (requires internet connection on first load; cached thereafter).

---

## Files Modified Per Phase

  index.html         — All UI, CSS, JavaScript (primary file)
  content.json       — Story content and configuration
  assets/future/     — Five SVG placeholder illustrations (Phase 5b)
  FUTURE_IMAGE_PROMPTS.md — AI generation prompts (Phase 5b)
  README.md          — Updated through Phase 6
  IMPLEMENTATION_PLAN.md — This file
