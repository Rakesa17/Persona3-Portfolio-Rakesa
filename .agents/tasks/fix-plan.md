# Implementation Plan — Fix 3 bugs in the Persona 3 Reload portfolio site

Project type: static site (plain HTML / CSS / JS), data-driven. There is **no build
system, no package manager, and no automated test runner** — verification is done by
opening pages in a browser. The site supports deep-link URL shortcuts that jump
straight to each screen (handled in main.js init, ~line 1583+), so each fix can be
verified without navigating the full menu:

- Skills:  `index.html?page=skill&tab=languages`
- Contact: `index.html?page=contact`

All edits are in place (no worktree, not a git repo). Each item below leaves the site
fully functional.

---

## Root-cause summary for ISSUE 3 (Gmail click does nothing) — pinned with file:line refs

The Gmail contact row in `index.html` is a real anchor and is correct:
`<a href="mailto:rakesaa17@gmail.com" class="p3r-mail-row active" data-idx="0" ...>`.

Tracing the full click path in `js/main.js`:

- The ONLY listener attached to `.p3r-mail-row` elements is `mouseenter`
  (`main.js:1438-1444`): `row.addEventListener("mouseenter", () => { selectContactRow(idx); })`.
  `selectContactRow` (`main.js:1116-1128`) only toggles the `.active` class and calls
  `playSFX()`. It does **not** call `preventDefault`, and it is a `mouseenter` handler,
  not a `click` handler.
- There is **no** `click` listener on the contact rows and **no** delegated/global
  `click` handler on `document`/`body`.
- Every `preventDefault` / `return false` / `stopPropagation` in `main.js` lives ONLY
  inside `document.addEventListener("keydown", ...)` (`main.js:1447-1536`) — keyboard
  only, never fires on a mouse click.
- The audio-unlock listener is registered with `{ capture: true, passive: true }`
  (`main.js`, the `["pointerdown", ... "click", ...].forEach(...)` block), so it is
  legally unable to `preventDefault`.
- CSS: `.p3r-mail-phone-wrap` has `pointer-events: auto` (`style.css:1858`) and
  `.p3r-contact-screen.active .p3r-mail-phone-wrap` sets `pointer-events: auto`
  (`style.css:2128-2132`). The chassis SVG is `pointer-events: none` (`style.css:1871`).
  No overlay with `pointer-events: auto` covers the rows.

Conclusion: **No code cancels the mailto.** The anchor is well-formed and reachable, the
click fires the browser's `mailto:` handler, and that handler is a no-op on a Windows
machine with no default mail client registered. `mailto:` also never navigates to
Gmail, which is what the user asked for ("pindah ke Gmail untuk mengirimkan pesan").
The reliable fix is to point the Gmail row at Gmail's web compose URL (an `https://`
link like the LinkedIn row), so clicking deterministically opens Gmail compose
addressed to rakesaa17@gmail.com regardless of OS mail-client config. The LinkedIn row
already uses the same `.p3r-mail-row` class with the same non-blocking listeners, so it
already navigates correctly; switching Gmail to an `https:` href keeps both anchors
working through the identical (unblocked) click path.

---

## Items

- [ ] 1. ISSUE 1 — Set English proficiency to 70 in the skills/languages data.
      In `js/main.js`, in the `skillGroupsData` array, the `languages` group
      (`code: "04"`, around line 186-190) currently has:
        `{ name: "Indonesia", level: 100 }`
        `{ name: "English", level: 60 }`
      Change `English` from `level: 60` to `level: 70`. Leave `Indonesia` at `100`.
      `renderSkillStats` already binds the bar width via `style="width: ${skill.level}%"`
      and the label via `${skill.level}` (`main.js:1018-1024`), so this single number
      change updates both the meter width (to ~70%) and the displayed "LV 70".
      Files: js/main.js
      Verify: open `index.html?page=skill&tab=languages` in a browser. The LANGUAGES
      category shows Indonesia at a full bar (LV 100) and English at roughly 70% bar
      width with label "LV 70".

- [ ] 2. ISSUE 2 — Fix the LinkedIn logo so both "i" and "n" render.
      The LinkedIn icon block CSS already supplies the blue rounded square and white
      ink: `.p3r-mail-linkedin { background: #0a66c2; color: #ffffff; }`
      (`style.css:1967-1971`), and the SVG uses `fill="currentColor"`. The bug is the
      single `<path>` includes the outer rounded-square subpath
      (`M19 3a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h14z`)
      which, under the default nonzero fill rule, cancels the "in" letter subpaths so
      only part of "in" draws. Remove that outer-square subpath so the SVG draws only
      the solid white "in" letters; the blue square stays coming from the CSS
      background (exactly like the Gmail block, whose SVG draws only the envelope and
      CSS supplies the red square).

      Apply this to BOTH LinkedIn SVGs in `index.html` (the big icon block inside
      `.p3r-mail-icon-block.p3r-mail-linkedin`, and the small sender-row glyph
      `.p3r-mail-linkedin-icon`). Replace the `d` attribute value:

      BEFORE (both occurrences):
      `M19 3a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h14zM8.34 10.5H6.17V18h2.17v-7.5zM7.25 9.5a1.26 1.26 0 1 0 0-2.52 1.26 1.26 0 0 0 0 2.52zM18 18h-2.17v-3.96c0-1.02-.37-1.72-1.28-1.72-.7 0-1.12.47-1.3.92-.07.17-.09.4-.09.64V18H11V10.5h2.17v1.05c.3-.46.83-1.12 2.02-1.12 1.48 0 2.81.97 2.81 3.05V18z`

      AFTER (both occurrences — drop the leading `M19 3...h14z` outer square subpath,
      keep the three "in" subpaths):
      `M8.34 10.5H6.17V18h2.17v-7.5zM7.25 9.5a1.26 1.26 0 1 0 0-2.52 1.26 1.26 0 0 0 0 2.52zM18 18h-2.17v-3.96c0-1.02-.37-1.72-1.28-1.72-.7 0-1.12.47-1.3.92-.07.17-.09.4-.09.64V18H11V10.5h2.17v1.05c.3-.46.83-1.12 2.02-1.12 1.48 0 2.81.97 2.81 3.05V18z`

      Keep the surrounding `<svg viewBox="0 0 24 24" width="34" height="34" fill="currentColor">`
      (big block) and `<svg class="p3r-mail-linkedin-icon" width="18" height="18" viewBox="0 0 24 24" fill="currentColor">`
      (small glyph) exactly as they are — only the `d` value changes. No CSS change is
      required; the blue square and white ink are already provided by the existing rules.
      Files: index.html
      Verify: open `index.html?page=contact` in a browser. The LinkedIn card's left icon
      block shows a blue rounded square containing a clearly visible white "in"
      (lowercase i and n both present). The Gmail card (red square + white envelope)
      looks visually consistent in size/style. The small LinkedIn glyph next to the
      "LinkedIn" label also shows both letters.

- [ ] 3. ISSUE 3 — Make the Gmail row reliably open a compose to rakesaa17@gmail.com.
      In `index.html`, replace the Gmail anchor's `href` (and add target/rel so it opens
      in a new tab like the LinkedIn anchor). Root cause and reasoning: no JS
      preventDefault blocks the click (see summary above); the existing `mailto:` fires
      but is a no-op without an OS mail client and never reaches Gmail web. The user
      asked to "pindah ke Gmail", so point it at Gmail's web compose URL.

      BEFORE:
      `<a href="mailto:rakesaa17@gmail.com" class="p3r-mail-row active" data-idx="0" role="listitem" aria-label="Email via Gmail">`

      AFTER:
      `<a href="https://mail.google.com/mail/?view=cm&fs=1&to=rakesaa17@gmail.com" target="_blank" rel="noopener noreferrer" class="p3r-mail-row active" data-idx="0" role="listitem" aria-label="Email via Gmail">`

      The Gmail compose URL (`view=cm&fs=1&to=...`) opens Gmail's compose window
      pre-addressed to rakesaa17@gmail.com. If the visitor is not signed in, Gmail shows
      its sign-in page first then the compose — either way the click reliably "goes to
      Gmail" instead of doing nothing. The LinkedIn anchor below it already has
      `target="_blank" rel="noopener noreferrer"` and uses the same `.p3r-mail-row`
      class, so both rows now navigate through the same unblocked path.
      Files: index.html
      Verify: open `index.html?page=contact` in a browser. Click the Gmail row: a new tab
      opens Gmail compose addressed to rakesaa17@gmail.com (or the Gmail sign-in page if
      not signed in). Click the LinkedIn row: a new tab opens https://www.linkedin.com/in/rakesa.
      Optionally open DevTools → Network/Console while clicking to confirm the click
      produces navigation (no "blocked" / no swallowed event). Also confirm no console
      errors from main.js.

- [ ] 4. Cache-bust — bump the asset version query strings in index.html.
      After the edits above, bump the two cache-bust query strings so browsers fetch the
      updated files. These are the `<head>` stylesheet and the bottom `<script>` tags.

      BEFORE:
      `<link rel="stylesheet" href="css/style.css?v=39">`
      `<script src="js/main.js?v=38" defer></script>`

      AFTER:
      `<link rel="stylesheet" href="css/style.css?v=40">`
      `<script src="js/main.js?v=39" defer></script>`

      (main.js must bump because ISSUE 1 edits it; style.css is bumped per the task
      instruction even though its contents are unchanged, which is harmless and forces a
      fresh fetch.)
      Files: index.html
      Verify: open `index.html` (hard refresh, e.g. Ctrl+F5). In DevTools → Network, the
      loaded assets show `style.css?v=40` and `main.js?v=39` (200, from network, not a
      cached `?v=38`/`?v=39`).

---

## Notes for the implementer

- Edit in place. No git, no worktree, no commits.
- No build/test command exists for this project; verification is manual browser
  inspection using the deep-link URLs above. The deep-links work because main.js init
  reads `?page=` / `?tab=` (see main.js ~1583+).
- Do not touch any click handlers in main.js for ISSUE 3 — the diagnosis is that none of
  them block the anchor. If a future handler is added that calls `preventDefault` on
  contact rows, re-evaluate; as of this investigation there is none.
- ISSUE 1, 2, 3 are independent and can be done in any order; item 4 (cache-bust) should
  be done last because it edits the same index.html lines region and depends on the
  other edits being final.
