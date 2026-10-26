# The Best Is Yet to Come
### A Three.js First-Person Birthday Journey

A handcrafted, interactive birthday experience for Nannagaru's 57th birthday.
The visitor drives a first-person camera along a winding mountain road, stopping at five
milestone memories before reaching an open golden horizon — and a personal birthday letter.

---

## How to Preview Locally

Open `index.html` in a modern browser. Because the page loads `content.json` via `fetch()`,
you need a local server (browsers block `file://` requests to JSON):

```
python -m http.server 8080
# Then open: http://localhost:8080
```

Or use the VS Code Live Server extension, or any static file server.

---

## Journey Flow

1. Camera arrives at the final milestone (Stop 5: "The Best Is Yet to Come").
2. Open the milestone popup to explore five future-experience illustrations.
3. Select "Read Your Birthday Letter" to open the dedicated letter overlay.
4. Read the letter, then choose to return to the final milestone or restart the journey.

---

## How the Milestone Popup Works

Each stop opens a centred cinematic popup over the 3D scene.

- Design: Dark navy gradient, warm gold border, 24px border-radius, deep shadow.
- Animation: Fade in + scale(0.96) translateY(10px) to scale(1). Reversed on close.
- Accessibility: role="dialog", aria-modal, focus management, keyboard navigation.
- prefers-reduced-motion: All transitions disabled.
- Backdrop click: Clicking outside the article closes it.
- Keyboard: Escape closes; arrow keys navigate photo galleries.

---

## How the Birthday Letter Works

At Stop 5, the milestone popup shows a "Read Your Birthday Letter" button (final stop only).

Selecting it:
1. Closes the milestone popup gracefully.
2. Reveals a separate letter overlay (z-index 7, above the milestone popup).
3. The 3D scene remains visible around the letter.
4. The camera does not move.

Letter interactions:
  - "Return to Final Milestone": closes letter, re-opens the milestone popup.
  - "Restart Journey": closes letter, travels camera back to Stop 1.
  - Escape key closes the letter (returns to milestone popup).
  - Clicking the dark backdrop outside the article closes the letter.
  - The letter can be opened and closed repeatedly.

The letter reads content from `content.json` fields:
  - `site.finalLetter`  - the letter body (blank-line separated paragraphs)
  - `site.finalTelugu`  - optional Telugu closing line (hidden if empty)

---

## Editing Story Content

All story content lives in `content.json`. Find the "stops" array.

  title          - Large heading shown in the popup
  subtitle       - Italic tagline below the title
  year           - Appears in the gold eyebrow (leave empty to omit)
  story          - Narrative for this stop (blank lines separate paragraphs)
  teluguMessage  - Optional Telugu message in the popup (hidden if empty)
  photos         - Array of { "src": "...", "caption": "..." } entries
  futureScene    - ID of a future scene to show, or "all" for all scenes

To edit the birthday letter, find "site.finalLetter" in content.json.
Separate paragraphs with \n\n in the JSON string.

---

## Adding or Replacing Photographs

1. Copy your photo into assets/photos/ (create it if needed).
2. In content.json, find the stop and add to "photos":
   { "src": "assets/photos/my-photo.jpg", "caption": "A beautiful memory." }
3. Refresh the page.

The gallery shows one photo at a time with Prev/Next, captions, and a lightbox.
Photos use object-fit:contain — never cropped.

---

## Future Experiences (Stops 4 and 5)

Stop 4 shows a single aspirational illustration (unhurried-mornings).
Stop 5 shows all five future-experience categories as a gallery.

Configuration is in content.json under "site.futureScenes":

  id       - Unique identifier (referenced by stop futureScene field)
  title    - Gallery caption header
  src      - Path to the illustration (SVG or JPG/PNG)
  caption  - Short description shown below the image
  enabled  - Set to false to hide without deleting

To disable: set "enabled": false.
To reorder: move entries up/down in the array.
To add new: append an entry and reference its id in a stop's futureScene field.

---

## Replacing Placeholder Illustrations

Five SVG placeholders live in assets/future/. See FUTURE_IMAGE_PROMPTS.md
for AI generation prompts for each category.

To replace: save the new image to assets/future/, then update the "src"
field in the matching futureScenes entry in content.json.

---

## File Structure

  index.html                - Main file: Three.js scene + all UI + logic
  content.json              - All editable story content and configuration
  assets/future/            - Future-experience illustrations (SVG placeholders)
  assets/photos/            - Add personal photographs here (create folder)
  original-prototype.html   - READ ONLY reference — do not modify
  FUTURE_IMAGE_PROMPTS.md   - AI prompts for replacement illustrations
  IMPLEMENTATION_PLAN.md    - Development history and phase records
  README.md                 - This file

---

## Technical Notes

- Three.js r0.161.0 via jsDelivr CDN (ES module).
- CatmullRomCurve3 spline with 241 control points drives the camera.
- InstancedMesh for trees and mountains (GPU-efficient).
- No framework — pure vanilla JavaScript, HTML, and CSS.
- No paid APIs — all content is local.
- No build step — open index.html with a local server for JSON loading.
- z-index layers: scene(0) < HUD < progress(2) < popup(5) < letter(7) < lightbox(50).
