# Implementation Plan — Persona 3 Reload Portfolio Edits

## Project context (discovered during exploration)

This is a **static HTML/CSS/JS site** — no build system, no package manager, no test framework.
- `index.html` loads `css/style.css?v=38` and `js/main.js?v=37` (cache-busting query strings).
- `js/main.js` is a single ES file that renders all dynamic content (menu options, project cards, skill tabs/rows, contact rows are wired in markup but selected via JS).
- The site is **desktop-only**: `css/style.css` shows a `.mobile-warning` overlay for viewports `< 1024px` (`@media (max-width: 1023px)`). Verification MUST use a viewport ≥ 1024px.
- `main.js` supports **deep-link URL params** that skip the loading screen, which is how each change is verified:
  - `?page=project` → opens PROJECT screen directly
  - `?page=skills&tab=languages` → opens SKILLS screen on the LANGUAGES tab
  - `?page=contact` → opens CONTACT screen directly
  - `?page=about` → opens ABOUT screen directly
- There is **no automated test suite**. Verification is manual browser inspection via a local HTTP server + the deep-link URLs above. Grep is not verification.

**Verification setup (run once, reuse for every item):**
1. Start a local server in the project root: `python -m http.server 8000` (or `npx serve` if Node is available). Use a fresh server or hard-refresh to avoid caching.
2. Open `http://localhost:8000/?page=project` (etc.) in a browser window at least 1024px wide.
3. Because the HTML pins `js/main.js?v=37` / `css/style.css?v=38`, after editing those files either hard-refresh (Ctrl+Shift+R) or bump the `?v=` numbers in `index.html` so the browser fetches the edited files. Bumping is safer; see item 4.

All four changes are **independent** (different files / sections). They can be applied in any order; the ordering below groups the two `main.js` project changes together for convenience.

---

## Key structures found (ground truth from the files)

### Projects (CHANGE 1 & 2) — `js/main.js`
The 5 project cards are rendered dynamically from the `slinkData` array. The roman numeral lives in its own box (`.card-numeral-box` → `.card-numeral-text` = `${item.numeral}`), and the title/subtitle are separate elements. Current array:

```js
const slinkData = [
  { numeral: "I",   title: "P3R PORTFOLIO",         subtitle: "Persona 3 Reload UI & Web Audio Engine", url: "#" },
  { numeral: "II",  title: "PERSONA 5 PORTO",       subtitle: "Phantom Thieves Comic Showcase",         url: "../porto-persona/index.html" },
  { numeral: "III", title: "P5 FONT GENERATOR",     subtitle: "Ransom Cutout Typography Tool",          url: "../persona5-font/index.html" },
  { numeral: "IV",  title: "AIOS UTBK PLATFORM",    subtitle: "Neumorphic SNBT Exam Simulator",        url: "../aios-app/index.html" },
  { numeral: "V",   title: "CREATIX AGENCY",        subtitle: "High-Performance Modern Web Platform",  url: "../web-app/index.html" }
];
```

Navigation on click happens in `confirmSlinkSelection()` (the cards are `<button>` elements created in `renderSlinkCards`, so there is **no `href`** on them — navigation is pure JS):

```js
function confirmSlinkSelection() {
  const item = slinkData[selectedSlinkIndex];
  if (item && item.url && item.url !== "#") {
    playSFX();
    setTimeout(() => {
      window.location.href = item.url;
    }, 200);
  } else {
    playSFX();
  }
}
```

`confirmSlinkSelection` is the **only** place that navigates for projects. It is triggered by: clicking an already-selected card, pressing Enter/Space on a card, and the `slinkConfirmBtn` (ENTER) button. `playSFX()` plays the existing navigation sound (`sfx/navigation.wav`). The cards also call `selectSlinkCard(idx)` on hover/arrow-keys, which already calls `playSFX()`. No `data-link` attributes exist.

### Skills / Languages (CHANGE 3) — `js/main.js`
The LANGUAGES tab data lives in `skillGroupsData`, in the `id: "languages"` group. The bar width is driven entirely by `skill.level` (rendered as `width: ${skill.level}%` in `renderSkillStats`). The numeric `LV` indicator is hidden via CSS (`.p3r-stat-lv { display: none !important; }`), so **only the bar fill width is visible**. Current group:

```js
{
  id: "languages",
  title: "LANGUAGE PROFICIENCY",
  code: "04",
  skills: [
    { name: "Bahasa Indonesia (Native / Fluent)", level: 100 },
    { name: "English (Professional Working Proficiency)", level: 86 },
    { name: "English (Technical & Documentation)", level: 90 },
    { name: "English (Spoken & Conversational)", level: 82 }
  ]
}
```

The tab label (`skillTabsList` → `{ id: "languages", code: "04", label: "LANGUAGES" }`) and group title `"LANGUAGE PROFICIENCY"` are **unchanged** — only the `skills` array entries change. No CSS/HTML change needed; the existing `.p3r-stat-bar-fill` styling applies automatically.

### Contact (CHANGE 4) — `index.html` + `css/style.css`
The contact "phone pad" is static markup in `index.html` inside `#contact-page`. Key findings:
- The big header label is a watermark: `<div class="p3r-mail-watermark" aria-hidden="true">MAIL</div>`. CSS `.p3r-mail-watermark` sizes it at `clamp(5.6rem, 6.5vw, 7.6rem)`. "CONTACT" (7 chars) is wider than "MAIL" (4 chars) but still fits the phone-card width (`clamp(760px,50vw,980px)`); only the text node changes.
- **The Gmail `mailto:` link is already present** — no change needed for that part:
  ```html
  <a href="mailto:rakesaa17@gmail.com" class="p3r-mail-row active" data-idx="0" role="listitem" aria-label="Email via Gmail">
    <div class="p3r-mail-icon-block p3r-mail-gmail" aria-hidden="true">
      <svg viewBox="0 0 24 24" width="34" height="34" fill="currentColor">
        <path d="M20 4H4c-1.1 0-1.99.9-1.99 2L2 18c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2zm0 4l-8 5-8-5V6l8 5 8-5v2z"/>
      </svg>
    </div>
    <div class="p3r-mail-content-block">
      <div class="p3r-mail-sender-row">
        <svg class="p3r-mail-env-icon" width="18" height="14" viewBox="0 0 24 18" fill="currentColor" aria-hidden="true">
          <path d="M22 0H2C0.9 0 0 0.9 0 2v14c0 1.1 0.9 2 2 2h20c1.1 0 2-.9 2-2V2c0-1.1-.9-2-2-2zm0 4.2l-10 6.3L2 4.2V2.5l10 6.3 10-6.3v1.7z"/>
        </svg>
        <span class="p3r-mail-sender-name">Gmail</span>
      </div>
      <div class="p3r-mail-row-line" aria-hidden="true"></div>
      <div class="p3r-mail-msg-text">rakesaa17@gmail.com</div>
    </div>
  </a>
  ```
  Clicking it already opens the email client to compose to `rakesaa17@gmail.com`. Keep this block exactly as-is.
- Below the Gmail row there are **4 "empty slot" `<div>`s** (comments "Empty Slot 2/3/4/5 (Kosong)") of class `p3r-mail-row p3r-mail-row-empty`. The phone pad is a fixed-proportion SVG chassis (`viewBox="0 0 880 860"`, `preserveAspectRatio="none"`) sized for exactly 5 rows of `clamp(94px,10.4vh,110px)`. **Adding a 6th row risks overflowing/reshaping the P3R phone layout**, so the LinkedIn box is added by **converting "Empty Slot 2"** (the slot immediately below Gmail) into a real LinkedIn row. This keeps the 5-slot grid intact and places the LinkedIn box directly below Gmail with identical styling — exactly the user's intent ("a box below Gmail, same as the Gmail box, logo changed to LinkedIn").
- The LinkedIn icon-block background is **already styled** in `css/style.css`: `.p3r-mail-linkedin { background: #0a66c2; color: #ffffff; }`. So the big LinkedIn logo block just needs `class="p3r-mail-icon-block p3r-mail-linkedin"` + an inline LinkedIn "in" SVG — no new CSS for the big block.
- One CSS gotcha: `.p3r-mail-row.active .p3r-mail-env-icon { color: #ea4335 !important; }` forces the Gmail mini envelope icon red on hover/active. To avoid turning a LinkedIn mini icon red, the LinkedIn mini icon uses a **distinct class** `p3r-mail-linkedin-icon` (no existing CSS) with `fill="currentColor"`, so it inherits the sender-row text color (white normally, dark `#0c182b` on active) and a small CSS rule gives it the same `flex-shrink`/transition as the Gmail envelope.

---

- [ ] 1. **CHANGE 1 — Replace all 5 project titles & subtitles.** Edit the `slinkData` array in `js/main.js`. Keep the `numeral` fields (`"I"`…`"V"`) unchanged — they render in the separate numeral box and already provide the roman-numeral prefix. Only swap each entry's `title` and `subtitle`. Resulting array:
   ```js
   const slinkData = [
     { numeral: "I",   title: "ESP32 BLE GAMEPAD",      subtitle: "Web-Based Controller & Microcontroller UI" },
     { numeral: "II",  title: "AI VOICE ASSISTANT",     subtitle: "ESP32 Gemini AI & OLED Visual Interface" },
     { numeral: "III", title: "P3R PORTFOLIO",         subtitle: "Persona 3 Reload UI & Web Audio Engine" },
     { numeral: "IV",  title: "BEAUTY E-COMMERCE",      subtitle: "Interactive Web Platform & Frontend UI" },
     { numeral: "V",   title: "NETWORK PEN-TEST TOOL", subtitle: "ESP32 Web Interface & Security Tool" }
   ];
   ```
   (The `url` keys are removed here too — see item 2. If applying items 1 & 2 as one edit, the array above is the final state.)
   Files: `js/main.js`
   Verify: Start local server, open `http://localhost:8000/?page=project` (viewport ≥1024px, hard-refresh/bump `?v=`). Confirm 5 cards appear with roman numerals I–V and exactly the titles/subtitles above. Existing layout, slant, and numeral boxes are unchanged.

- [ ] 2. **CHANGE 2 — Remove all project navigation; clicks play only the existing SFX.** In `js/main.js`, replace the body of `confirmSlinkSelection()` so it only plays the sound and never navigates. This removes the only project navigation path (`window.location.href = item.url`). Replace:
   ```js
   function confirmSlinkSelection() {
     const item = slinkData[selectedSlinkIndex];
     if (item && item.url && item.url !== "#") {
       playSFX();
       setTimeout(() => {
         window.location.href = item.url;
       }, 200);
     } else {
       playSFX();
     }
   }
   ```
   with:
   ```js
   function confirmSlinkSelection() {
     playSFX();
   }
   ```
   Also remove the `url` property from every `slinkData` entry (already done in the item-1 array above) so no stale navigation data remains. No `href`/`data-link`/`window.open` exist on the project cards (they are `<button>`s), so no other edits are needed.
   Files: `js/main.js`
   Verify: On `?page=project`, click each card and press Enter on a selected card. Confirm: the navigation SFX plays, the browser URL does **not** change, no new tab opens, no page navigation occurs. Hovering/arrow-keying between cards still plays the selection SFX as before.

- [ ] 3. **CHANGE 3 — Skills › Languages: keep only Indonesia (100%) and English (~60%).** In `js/main.js`, replace the `skills` array of the `{ id: "languages" }` group inside `skillGroupsData` with exactly two entries. Replace:
   ```js
     skills: [
       { name: "Bahasa Indonesia (Native / Fluent)", level: 100 },
       { name: "English (Professional Working Proficiency)", level: 86 },
       { name: "English (Technical & Documentation)", level: 90 },
       { name: "English (Spoken & Conversational)", level: 82 }
     ]
   ```
   with:
   ```js
     skills: [
       { name: "Indonesia", level: 100 },
       { name: "English", level: 60 }
     ]
   ```
   Keep the group `title: "LANGUAGE PROFICIENCY"`, `code: "04"`, and the tab label `LANGUAGES` unchanged. The bar fill is driven by `level` (rendered `width: ${level}%`), so Indonesia renders a full bar and English a ~60% bar automatically — no CSS/HTML change.
   Files: `js/main.js`
   Verify: Open `http://localhost:8000/?page=skills&tab=languages` (hard-refresh/bump `?v=`). Confirm the LANGUAGES tab shows exactly two rows: "Indonesia" with a full-width bar, and "English" with a bar filled to roughly 60% (not full). No other languages appear. Tab label and group title are unchanged.

- [ ] 4. **CHANGE 4 — Contact: relabel watermark to CONTACT, keep Gmail mailto, add LinkedIn box below Gmail.** Three edits:

   **(4a) Relabel watermark** in `index.html`: change
   ```html
   <div class="p3r-mail-watermark" aria-hidden="true">MAIL</div>
   ```
   to
   ```html
   <div class="p3r-mail-watermark" aria-hidden="true">CONTACT</div>
   ```

   **(4b) Convert "Empty Slot 2" into the LinkedIn row** in `index.html`. Replace the entire block currently commented `<!-- Empty Slot 2 (Kosong) -->` (the first empty slot directly below the Gmail `<a>`):
   ```html
   <div class="p3r-mail-row p3r-mail-row-empty" aria-hidden="true">
     <div class="p3r-mail-icon-block p3r-mail-empty-icon" aria-hidden="true"></div>
     <div class="p3r-mail-content-block">
       <div class="p3r-mail-sender-row">
         <span class="p3r-mail-sender-name"></span>
       </div>
       <div class="p3r-mail-row-line" aria-hidden="true"></div>
       <div class="p3r-mail-msg-text p3r-mail-empty-text">No Message</div>
     </div>
   </div>
   ```
   with the LinkedIn contact row (styled identically to the Gmail row; opens LinkedIn in a new tab):
   ```html
   <a href="https://www.linkedin.com/in/rakesa" target="_blank" rel="noopener noreferrer" class="p3r-mail-row" data-idx="1" role="listitem" aria-label="Connect on LinkedIn">
     <div class="p3r-mail-icon-block p3r-mail-linkedin" aria-hidden="true">
       <svg viewBox="0 0 24 24" width="34" height="34" fill="currentColor">
         <path d="M19 3a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h14zM8.34 10.5H6.17V18h2.17v-7.5zM7.25 9.5a1.26 1.26 0 1 0 0-2.52 1.26 1.26 0 0 0 0 2.52zM18 18h-2.17v-3.96c0-1.02-.37-1.72-1.28-1.72-.7 0-1.12.47-1.3.92-.07.17-.09.4-.09.64V18H11V10.5h2.17v1.05c.3-.46.83-1.12 2.02-1.12 1.48 0 2.81.97 2.81 3.05V18z"/>
       </svg>
     </div>
     <div class="p3r-mail-content-block">
       <div class="p3r-mail-sender-row">
         <svg class="p3r-mail-linkedin-icon" width="18" height="18" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
           <path d="M19 3a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h14zM8.34 10.5H6.17V18h2.17v-7.5zM7.25 9.5a1.26 1.26 0 1 0 0-2.52 1.26 1.26 0 0 0 0 2.52zM18 18h-2.17v-3.96c0-1.02-.37-1.72-1.28-1.72-.7 0-1.12.47-1.3.92-.07.17-.09.4-.09.64V18H11V10.5h2.17v1.05c.3-.46.83-1.12 2.02-1.12 1.48 0 2.81.97 2.81 3.05V18z"/>
         </svg>
         <span class="p3r-mail-sender-name">LinkedIn</span>
       </div>
       <div class="p3r-mail-row-line" aria-hidden="true"></div>
       <div class="p3r-mail-msg-text">Rakesa</div>
     </div>
   </a>
   ```
   Leave the Gmail `<a href="mailto:rakesaa17@gmail.com" ...>` block untouched (the mailto is already correct). Leave the remaining 3 empty slots untouched so the phone pad keeps its 5-row grid.

   **(4c) Add the mini LinkedIn-icon CSS** in `css/style.css`, right after the existing `.p3r-mail-linkedin { ... }` rule (around line 1972), so the mini icon behaves like the Gmail envelope (flex-shrink + color transition) but is **not** forced red on active:
   ```css
   .p3r-mail-linkedin-icon {
     flex-shrink: 0;
     transition: color 0.18s ease;
   }
   ```

   **(4d) Bump cache-busting query strings** in `index.html` so the browser fetches the edited files: change `js/main.js?v=37` → `js/main.js?v=38` and `css/style.css?v=38` → `css/style.css?v=39`. (Optional but recommended; a hard refresh also works.)

   Files: `index.html`, `css/style.css`
   Verify: Open `http://localhost:8000/?page=contact` (hard-refresh / after bumping `?v=`). Confirm:
   - The large header watermark reads **CONTACT**.
   - The Gmail row is unchanged; clicking it opens the email client composing to `rakesaa17@gmail.com`.
   - A LinkedIn row appears **directly below Gmail**, same card size/shape/borders as Gmail, with a blue LinkedIn "in" logo block on the left, sender name "LinkedIn", and value "Rakesa".
   - Clicking the LinkedIn row opens `https://www.linkedin.com/in/rakesa` in a **new browser tab** (target="_blank").
   - Hovering the LinkedIn row turns the card white (active style) like Gmail, and the mini LinkedIn icon darkens with the text (does **not** turn red).
   - The remaining "No Message" empty slots still render below; the phone pad layout is not broken (no overflow/reshape).

---

## Notes for the implementer
- All project changes are JS-only (`main.js`). The projects are data-driven, so editing the `slinkData` array and `confirmSlinkSelection` is the complete and only change — do not touch `index.html` for projects (the cards are injected by `renderSlinkCards`).
- The Gmail `mailto:` link already exists in `index.html`; do not modify the Gmail block.
- `.p3r-mail-linkedin` (blue background) already exists in CSS; reuse it for the LinkedIn icon block. Only one small CSS rule (`.p3r-mail-linkedin-icon`) is added.
- If a local server is unavailable, `file://` will load the page but `fetch()` of the SFX files may be blocked by CORS — the site has HTML5 `<audio>` fallbacks so SFX still works, but a local HTTP server is the reliable verification path.
- There is no build/test command to run; "verification" above is the real manual check for this static site.
