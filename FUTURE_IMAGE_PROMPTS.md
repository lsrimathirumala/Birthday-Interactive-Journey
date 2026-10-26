# Future Experience — AI Image Generation Prompts

Use these prompts with an AI image generator (e.g. DALL-E 3, Midjourney, Stable Diffusion, Adobe Firefly)
to create beautiful, personalised illustrations for the future-experience gallery.

**Important constraints:**
- Do NOT include recognisable real faces or identifying personal details.
- Do NOT use images of specific real people.
- Prefer wide-format / landscape images (4:3 ratio, 800×600 px or larger).
- Aim for a warm, cinematic, hopeful aesthetic consistent with the rest of the site.
- Save the resulting image as the filename listed under each prompt, then update `content.json` if needed.

---

## 1. Family Adventures
**File:** `assets/future/family-adventures.svg` → replace with `assets/future/family-adventures.jpg` (or `.png`)

**Prompt:**
> Cinematic aspirational illustration of a multigenerational family on a mountain holiday — wide panoramic valley at golden hour, winding path through green hills, distant snow-capped peaks catching warm light. Silhouette figures walking together. No identifiable faces. Warm and hopeful atmosphere. Photorealistic or painterly, 4:3 landscape.

**Update `content.json`:** Change `"src": "assets/future/family-adventures.svg"` → `"src": "assets/future/family-adventures.jpg"`.

---

## 2. Unhurried Mornings
**File:** `assets/future/unhurried-mornings.svg` → replace with `assets/future/unhurried-mornings.jpg`

**Prompt:**
> A quiet sunlit veranda overlooking misty green hills at dawn. A weathered wooden chair, a steaming cup of chai on a small side table, potted plants catching the early light. Sunrise glow reflecting off the mist below. No people or only an indistinct seated silhouette from behind. Tranquil, unhurried, cinematic natural light. 4:3 landscape.

**Update `content.json`:** Change `"src"` path accordingly.

---

## 3. Rediscovering Interests
**File:** `assets/future/rediscovering-interests.svg` → replace with `assets/future/rediscovering-interests.jpg`

**Prompt:**
> An inviting, warmly lit workspace bathed in window light — open sketchbook with pencil sketches in progress, a small succulent plant, stacked books with colourful spines, a reading lamp, camera on a shelf. No person, just the atmosphere of a curious mind at rest. Warm evening light, cinematic colour grading, shallow depth of field. 4:3 landscape.

**Update `content.json`:** Change `"src"` path accordingly.

---

## 4. More Family Memories
**File:** `assets/future/more-family-memories.svg` → replace with `assets/future/more-family-memories.jpg`

**Prompt:**
> Warm Indian festival celebration with lit clay diyas arranged in a row, marigold garlands, soft candlelight reflections. Silhouette figures of family members (no identifiable faces) gathered together. Rangoli pattern on the ground, warm amber and saffron tones. Atmospheric, candid, celebratory. Cinematic 4:3 landscape.

**Update `content.json`:** Change `"src"` path accordingly.

---

## 5. The Years Ahead
**File:** `assets/future/years-ahead.svg` → replace with `assets/future/years-ahead.jpg`

**Prompt:**
> Golden sunrise over a long, gently winding road stretching toward an open glowing horizon. Cypress trees lining the road in silhouette. Stars still visible in the upper sky fading into the first light of dawn. Warm and hopeful, the road ahead full of possibility. No people. Cinematic, painterly, wide landscape.

**Update `content.json`:** Change `"src"` path accordingly.

---

## How to Replace a Placeholder

1. Generate the image using the prompt above.
2. Save it to the `assets/future/` folder (e.g. `assets/future/family-adventures.jpg`).
3. Open `content.json` and find the matching entry in `futureScenes`.
4. Update the `"src"` field to point to the new file.
5. Optionally update `"caption"` to personalise the text shown in the gallery.
6. Refresh the page — the new image will appear automatically.

## How to Disable an Experience

In `content.json`, find the entry in `futureScenes` and set `"enabled": false`:

```json
{
  "id": "rediscovering-interests",
  "title": "Rediscovering Interests",
  "src": "assets/future/rediscovering-interests.svg",
  "caption": "...",
  "enabled": false
}
```

The experience will be hidden in the gallery without needing to delete anything.
