# Contact & Skills fixes — English level, LinkedIn glyph, Gmail navigation, cache-bust

This change repairs three reported bugs on the Persona 3 Reload portfolio and bumps the asset cache-bust. English proficiency is raised from 60 to 70 (Indonesia stays 100), the LinkedIn logo SVG is corrected so both the "i" and "n" render on its blue square, and the Gmail contact row is repointed so a click reliably reaches Gmail rather than doing nothing.

Watch for: the Gmail row is implemented as Gmail's web compose URL (`https://mail.google.com/mail/?view=cm&fs=1&to=rakesaa17@gmail.com`), not a literal `mailto:`. This is deliberate and matches the user's explicit "go to Gmail" request — a `mailto:` opens the OS mail client (or nothing on a machine with no default client, which is the exact behavior the user complained about) and never reaches Gmail web. The coder's runtime capture confirms the click navigates to Gmail compose (sign-in first if not authenticated) and that no JS handler calls `preventDefault` on either contact anchor.

**Verdict**: APPROVED

## High-level view

The languages skill group in main.js now sets Indonesia to 100 and English to 70. renderSkillStats derives both the meter width and the displayed level from that single number, so the change flows to a 70% bar and an "LV 70" label with no separate width edit.

The LinkedIn icon is SVG path data over a CSS-supplied blue rounded square (`.p3r-mail-linkedin { background:#0a66c2; color:#fff }`). The fix removes the outer rounded-square subpath that previously canceled the letter subpaths under the nonzero fill rule, leaving the three "in" subpaths (i-stem, i-dot, n) to draw white on the blue square. Both the large icon block and the small sender-row glyph were corrected.

The Gmail row was a well-formed `mailto:` that nothing in main.js blocks — the only `.p3r-mail-row` listener is `mouseenter`, and every `preventDefault` in the file lives inside the keyboard `keydown` handler. The real failure was that `mailto:` opens the OS mail client (or nothing without one), never Gmail web. Repointing it to Gmail's compose URL makes the click deterministically reach Gmail, and the LinkedIn row navigates through the same unblocked path. The contact-page Enter/Space handler calls `.click()` on the focused row, which navigates rather than blocks.

Cache-bust versions were bumped in index.html: `style.css` v39→v40 and `main.js` v38→v39.

<details>
<summary>Issues (0)</summary>

No actionable findings.

</details>

<details><summary>Details</summary>

### English proficiency set to 70

main.js, `skillGroupsData` languages group (lines 188-189): `{ name: "Indonesia", level: 100 }` and `{ name: "English", level: 70 }`. renderSkillStats (line 1018) emits `<div class="p3r-stat-bar-fill" data-v="${skill.level}" style="width: ${skill.level}%;">`, so the single `level` value drives both the bar width and the "LV n" label. With `level: 70` the bar renders at 70% width and the label reads "LV 70"; Indonesia at `level: 100` renders a full bar. The coder's runtime capture returned `style:"width: 70%;"` and `lv:"70"` for English, `width: 100%` for Indonesia. *(confirmed — main.js lines 188-189, line 1018 binding)*

### LinkedIn glyph renders both "i" and "n"

Both LinkedIn SVGs in index.html now use path data beginning `M8.34 10.5H6.17V18h2.17v-7.5z` (i-stem), `M7.25 9.5a1.26 1.26 0 1 0 0-2.52...` (i-dot), and `M18 18h-2.17...` (n). The old leading outer-square subpath `M19 3a2 2 0 0 1 2 2v14...h14z` is gone from both the large `.p3r-mail-icon-block.p3r-mail-linkedin` block and the small `.p3r-mail-linkedin-icon` glyph. Under the default nonzero fill rule that outer clockwise subpath was canceling the letter subpaths, which is why the user saw only the "i". The blue rounded square and white ink come from `.p3r-mail-linkedin { background:#0a66c2; color:#ffffff }` (style.css lines 1967-1971) with the SVG using `fill="currentColor"`, so the SVG now draws only white letters on the CSS blue square — visually consistent with the Gmail block (red CSS square + white envelope SVG). The coder's `getBBox()` capture returned `{ x:5.99, y:6.98, width:12.01, height:11.02 }`, spanning x≈6→18 across both glyphs. *(confirmed — index.html both paths, style.css lines 1967-1971)*

### Gmail and LinkedIn navigation, unblocked

Gmail row (index.html): `<a href="https://mail.google.com/mail/?view=cm&fs=1&to=rakesaa17@gmail.com" target="_blank" rel="noopener noreferrer" class="p3r-mail-row active" ...>`. LinkedIn row: `<a href="https://www.linkedin.com/in/rakesa" target="_blank" rel="noopener noreferrer" class="p3r-mail-row" ...>`. Both are real anchors with https hrefs and new-tab targets.

The click path is unblocked. main.js attaches only a `mouseenter` listener to each `.p3r-mail-row` (lines 1441-1442) calling `selectContactRow`, which toggles `.active` and plays SFX — no `preventDefault`. Every `preventDefault` in the file is inside the keyboard `keydown` handler (lines 1457-1535), keyed on `e.key`, so none fires on a mouse click. The contact-page Enter/Space branch calls `contactMailRows[selectedContactIndex].click()`, which triggers the anchor's default navigation rather than canceling it. The audio-unlock window listener is registered `{ capture:true, passive:true }`, so it cannot `preventDefault`. The coder's synthetic-click trace returned `defaultPrevented:false` for both anchors, and the real `el.click()` on the Gmail row navigated to `https://accounts.google.com/v3/signin/identifier?continue=...mail.google.com/mail/?view=cm&fs=1&to=rakesaa17@gmail.com...` — Gmail's sign-in page with the continue URL set to a compose addressed to rakesaa17@gmail.com. *(confirmed — index.html anchors; main.js lines 1438-1442 and 1457-1535; full-file preventDefault grep)*

Note on the href choice: the task's point 3 described the Gmail row as a `mailto:rakesaa17@gmail.com` anchor. The implementation uses Gmail's web compose URL instead, which is the correct reading of the user's actual request — message 2 says "pas di klik nga pindah ke Gmail untuk mengirimkan pesan" (when clicked it doesn't go to Gmail to send a message), i.e. the click should navigate to Gmail. A `mailto:` opens the OS mail client (Outlook, Mail.app) and is a no-op on a machine with no default client registered — exactly the "does nothing" behavior the user reported — and never reaches Gmail web. The compose URL (`view=cm&fs=1&to=...`) opens Gmail's compose window pre-addressed to rakesaa17@gmail.com, with sign-in gating if the visitor isn't authenticated. The LinkedIn row already used the same new-tab https pattern, so both rows now navigate through the identical unblocked path. Per the task's own resolution rule the verbatim user request wins over a paraphrased expectation, so this deviation is intentional and correct rather than a defect. *(confirmed — intentional deviation matching the authoritative user request)*

### Cache-bust

index.html: `css/style.css?v=40` (head link) and `js/main.js?v=39` (bottom script), bumped from v39/v38 respectively. main.js was edited for the English level, so its bump is required; style.css was bumped despite no content change. *(confirmed — index.html head link and bottom script tag)*

</details>

<details><summary>File map</summary>

- `js/main.js` — languages skill group: English `level: 60` → `70` (Indonesia stays `100`).
- `index.html` — Gmail anchor repointed to the Gmail web compose URL with `target="_blank" rel="noopener noreferrer"`; outer-square subpath removed from both LinkedIn SVG paths so the full "in" renders; cache-bust `style.css?v=40`, `main.js?v=39`.
- `css/style.css` — unchanged; the LinkedIn blue square (`#0a66c2`), white ink, and pointer-events rules were already correct.

This is an in-place edit (no git repo), so the "diff" is the current file state above against the before-values recorded in `.agents/tasks/fix-plan.md`.

</details>
