# Portfolio content edits: projects, languages, contact watermark + LinkedIn

This change reworks three content areas of the Persona 3 Reload portfolio to match new copy and contact details. The five project cards get new titles and subtitles and lose all navigation — clicking a card now only plays the selection SFX with no page change. The Languages skill group is trimmed to two rows, Indonesia (full) and English (~60%), with the verbose proficiency labels dropped. The contact screen watermark switches from `MAIL` to `CONTACT`, the existing Gmail `mailto:` row is preserved, and a new LinkedIn row is added directly beneath it in the same phone-pad card style, linking out to the profile URL. This is a static HTML/CSS/JS site with no build system; edits are data/markup only.

**Watch for:** Nothing blocking. The project numeral prefix is rendered from a separate `numeral` field, not embedded in the title string — worth confirming that matches the intended visual.

**Verdict**: APPROVED

## High-level view

The project cards are data-driven from `slinkData` in `js/main.js`, so changing the five titles/subtitles and removing the `url` keys is the complete edit; the cards are `<button>` elements with no `href`, so there was never a markup-level link to strip. Navigation lived entirely in `confirmSlinkSelection()`, which now only calls `playSFX()`, so clicking, pressing Enter, or the ENTER button all resolve to sound-only with no redirect.

The Languages tab draws its rows from the `id: "languages"` group in `skillGroupsData`; the bar width is `skill.level` percent, so Indonesia at 100 renders a full bar and English at 60 renders roughly 60%. The group title and tab label were intentionally left as-is since the request only touched the language rows.

The contact watermark is a single text node swapped from `MAIL` to `CONTACT`. The LinkedIn row reuses the existing `.p3r-mail-linkedin` blue background block and reuses the slot that was an empty placeholder, so the fixed five-row phone-pad grid stays intact. It opens the profile in a new tab with `rel="noopener noreferrer"`, the safe default for an external link.

<details>
<summary>Issues (0)</summary>

No actionable issues. All four required changes are correctly implemented and confirmed against the source and the coder's verification note.

</details>

<details>
<summary>Details</summary>

### Project copy and the numeral split (CHANGE 1)

The request lists each project as `I — ESP32 BLE GAMEPAD` and so on. The implementation keeps the roman numeral in the existing `numeral` field (rendered in its own styled numeral box) and stores only `ESP32 BLE GAMEPAD` in `title`. This matches the site's existing card layout, where numeral and title are separate elements; folding the numeral into the title string would have duplicated it on screen. All five title/subtitle pairs match the spec exactly:

- I / ESP32 BLE GAMEPAD / Web-Based Controller & Microcontroller UI
- II / AI VOICE ASSISTANT / ESP32 Gemini AI & OLED Visual Interface
- III / P3R PORTFOLIO / Persona 3 Reload UI & Web Audio Engine
- IV / BEAUTY E-COMMERCE / Interactive Web Platform & Frontend UI
- V / NETWORK PEN-TEST TOOL / ESP32 Web Interface & Security Tool

This is a sound reading of the intent. If the user actually wanted the literal string `I — ESP32 BLE GAMEPAD` inside the title element, that would visibly duplicate the numeral box — not the goal. (confirmed)

### Project navigation removal (CHANGE 2)

`confirmSlinkSelection()` is now a single `playSFX()` call; the `item.url` lookup and `window.location.href` assignment are gone, and every `url:` key was removed from `slinkData`. A grep for `url:`, `item.url`, and `location.href` in `main.js` returns no hits, so the only project navigation path is eliminated. The remaining `window.location.search` reads deep-link query params and the `window.openProjectPage`-style exports are function-name assignments, not navigation — none of these open a page for a project card. (confirmed)

### Languages trim (CHANGE 3)

The `languages` group's `skills` array is exactly two entries: `{ name: "Indonesia", level: 100 }` and `{ name: "English", level: 60 }`. The old `Bahasa Indonesia (Native / Fluent)` and `English (Professional Working Proficiency)` strings are gone. Bar fill is `width: ${skill.level}%`, so Indonesia fills fully and English fills ~60%, matching "Indonesia full, English ~60%". The numeric `LV` indicator is hidden by the pre-existing `.p3r-stat-lv { display: none }` rule, so the bar is the only visible signal of the level. (confirmed)

### Contact watermark and LinkedIn row (CHANGE 4)

The watermark text node now reads `CONTACT`. The Gmail row is unchanged: `<a href="mailto:rakesaa17@gmail.com" ...>`, which opens the visitor's mail client pre-addressed to that address — the standard static-site behavior, and the correct read of "click Gmail to send to rakesaa17@gmail.com". Direct client-side sending isn't possible without a backend, so `mailto:` is the right approach.

The LinkedIn row sits directly below Gmail inside the same `p3r-mail-list-container`. It reuses the card structure: the blue `.p3r-mail-linkedin` icon block with an inline LinkedIn "in" SVG, a sender name of `LinkedIn`, and a message value of `Rakesa`. The link is `https://www.linkedin.com/in/rakesa` with `target="_blank" rel="noopener noreferrer"`, the correct secure defaults for an external opener. The new `.p3r-mail-linkedin-icon` CSS rule gives the small sender-row icon `flex-shrink: 0` and a color transition, mirroring the Gmail envelope icon. (confirmed)

### Verification evidence

The coder's `verify.md` records a tag-balance check on `index.html` (515/515 angle brackets, 2/2 anchors, 126/126 divs, 4/4 sections) and a local HTTP server run that served all three assets (200s) with string assertions for every change. The claimed cache-bust bumps (`style.css?v=39`, `main.js?v=38`) match what `index.html` actually pins. The note is honest about two un-automated items: no headless browser DOM snapshot (cards/bars are data-driven and confirmed via source), and SFX not audio-verified (wiring unchanged).

</details>

<details>
<summary>File map</summary>

- `js/main.js` — Replaced all five `slinkData` titles/subtitles and removed every `url` key; reduced `confirmSlinkSelection()` to a single `playSFX()`; trimmed the `languages` skill group to Indonesia (100) and English (60).
- `index.html` — Changed contact watermark `MAIL` → `CONTACT`; added the LinkedIn row below Gmail; bumped cache-bust to `style.css?v=39` and `main.js?v=38`.
- `css/style.css` — Added `.p3r-mail-linkedin-icon` (flex-shrink + color transition) for the LinkedIn sender-row icon.

Full diff: not available — the project is not under git (`git status` reports no repository), so the review was conducted by reading the edited files directly.
