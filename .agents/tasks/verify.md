# Verification Note — 3 fixes for the P3R portfolio site

Date: 2025 (run on Windows, PowerShell). Workspace: `c:/Users/rakes/OneDrive/Desktop/porto-persona3-main`.
Project type: static site (plain HTML/CSS/JS), no build/test runner. Verification done by serving
the site and driving a real headless browser (system Chrome via puppeteer-core).

## What I ran

1. Served the site with `py -m http.server 8765` (background, from the workspace root).
2. `Invoke-WebRequest http://localhost:8765/index.html` → HTTP 200, content length 24042. Confirmed
   the served HTML contains the Gmail compose anchor, the LinkedIn anchor, `css/style.css?v=40`,
   and `js/main.js?v=39`.
3. Real-browser verification with `puppeteer-core@23.11.1` against the installed system Chrome
   (`C:\Program Files\Google\Chrome\Application\chrome.exe`), headless, viewport 1280x800. Two scripts:
   - `verify.js`: contact page (`?page=contact`) — anchors, LinkedIn SVG, synthetic-click
     `defaultPrevented` trace, and a real `el.click()` navigation capture; skill page
     (`?page=skill&tab=languages`) — Indonesia/English level + bar width.
   - `verify2.js`: focused re-check of both LinkedIn `<path>` glyphs with correct selectors and
     numeric `getBBox()`.
   (Scripts live in a throwaway temp dir `%TEMP%\p3r_verify`, outside the workspace, so the project
   is untouched. Temp dir removed after the run.)

## ISSUE 1 — English proficiency 70 (was 60)

- Change: `js/main.js`, `skillGroupsData` `languages` group → `{ name: "English", level: 70 }`
  (`Indonesia` left at `100`).
- `renderSkillStats` binds the meter via `style="width: ${skill.level}%"` and the label via
  `${skill.level}`, so this one number drives both.
- Runtime result (`?page=skill&tab=languages`):
  - Indonesia → `lv:"100"`, `dataV:"100"`, `style:"width: 100%;"`
  - English  → `lv:"70"`, `dataV:"70"`, `style:"width: 70%;"`  ✅

## ISSUE 2 — LinkedIn logo shows both "i" and "n"

- Root cause (per plan): the single `<path>` `d` began with the outer rounded-square subpath
  `M19 3a2 2 0 0 1 2 2v14...h14z`. Under the default **nonzero** fill rule, that clockwise outer
  subpath cancels the "in" letter subpaths, so only part of "in" drew (user saw "i" but no "n").
- Change: `index.html` — removed the outer-square subpath from **both** LinkedIn SVGs (the big
  `.p3r-mail-icon-block.p3r-mail-linkedin` block and the small `.p3r-mail-linkedin-icon` glyph).
  The SVG now draws only the white "in" letters; the blue rounded square + white ink already come
  from `.p3r-mail-linkedin { background:#0a66c2; color:#ffffff; }` in `style.css` (unchanged), so the
  result is a blue square with a white "in", visually consistent with the Gmail red-square/envelope
  block. No CSS edit was needed.
- Runtime result (`?page=contact`), both paths:
  - `d` starts `M8.34 10.5H6.17V18h2.17v-7.5zM7.25 9.5...M18 18h-2.17...`
  - `hasI: true`, `hasN: true`, `hasOldSquare: false`
  - `getBBox()` = `{ x: 5.99, y: 6.98, width: 12.01, height: 11.02 }` → spans x≈6.0→18.0, covering
    the "i" stem/dot (x≈6.17–8.34) and the "n" (x≈11–18). Both glyphs render. ✅

## ISSUE 3 (CRITICAL) — Gmail click did nothing

### Root cause
Tracing `js/main.js`, **no code cancels the Gmail anchor's click**:
- The only listener attached to `.p3r-mail-row` elements is `mouseenter`
  (`main.js:1439-1444` → `selectContactRow(idx)`), which only toggles `.active` and plays SFX. There
  is **no `click` listener** on the contact rows.
- The only window-level click listener is the audio-unlock handler registered with
  `{ capture: true, passive: true }` (`main.js`, the `["pointerdown",...,"click",...].forEach`
  block). `passive:true` makes it legally unable to call `preventDefault`; it never does.
- All other `addEventListener("click", …)` sites are on specific buttons (menu option hitboxes,
  S.Link cards, skill tabs, modal/back buttons) — none delegated on `document`/`body`, none on the
  mail rows. Every `preventDefault`/`return false` in the file lives inside the `keydown` handler
  (keyboard only) or inside `playAudioChannel` (a `return false` status flag, not a click handler).

So the previous anchor `<a href="mailto:rakesaa17@gmail.com">` was well-formed and unblocked: the
click fired the browser's `mailto:` handler, which is a **no-op on a Windows machine with no
default mail client registered**, and `mailto:` **never navigates to Gmail web**. That is why
"clicking did nothing" — it is not a JS-blocking bug. The user asked to "pindah ke Gmail untuk
mengirimkan pesan" (go to Gmail to send a message), which a `mailto:` cannot do.

### Fix (per the plan, which the task designated as authoritative for the root cause)
Repoint the Gmail row at Gmail's web compose URL (an `https` link, like the LinkedIn row), so a
click deterministically opens Gmail compose addressed to `rakesaa17@gmail.com` regardless of OS
mail-client config. If the visitor is not signed in, Gmail shows sign-in first then compose —
either way the click "goes to Gmail" instead of doing nothing. This matches the user's words and
supersedes the initial `mailto` hypothesis (keeping a `mailto:` would leave the bug in place, since
nothing blocks it yet it still does nothing). No click handlers in `main.js` were modified for
ISSUE 3 — none needed to be.

### Runtime result (`?page=contact`)
- Gmail row is a real anchor: `tag:"A"`,
  `href:"https://mail.google.com/mail/?view=cm&fs=1&to=rakesaa17@gmail.com"`, `target:"_blank"`,
  `rel:"noopener noreferrer"`. Serialized `outerHTML` shows `&amp;` (correct HTML escaping of the
  raw `&` query separators; the resolved `href` attribute value is `&`). No stray characters. ✅
- LinkedIn row: `tag:"A"`, `href:"https://www.linkedin.com/in/rakesa"`, `target:"_blank"`,
  `rel:"noopener noreferrer"`. ✅
- Synthetic-click trace (dispatch a cancelable `MouseEvent('click')` so handlers run but the
  browser default does not fire): Gmail `defaultPrevented:false`, `capturePhaseReached:true`;
  LinkedIn `defaultPrevented:false`. → **No handler calls `preventDefault` on either anchor.**
- Real click (`el.click()`): the Gmail row opened a **new tab** whose URL was
  `https://accounts.google.com/v3/signin/identifier?continue=https://mail.google.com/mail/?view%3Dcm%26fs%3D1%26to%3Drakesaa17@gmail.com&...`
  — i.e. Gmail's sign-in page with the **continue = Gmail compose addressed to
  rakesaa17@gmail.com**. This is exactly the expected "go to Gmail to compose to
  rakesaa17@gmail.com" behavior (sign-in first because the browser session is not authenticated,
  then compose). ✅✅

### Which handlers run on a Gmail-row click (before vs after fix)
The set of handlers is **identical before and after** (the fix only changed the `href`):
1. Capture phase on `window`: the audio-unlock listener (`passive`, cannot `preventDefault`).
2. At the anchor (target phase): none — `.p3r-mail-row` has no `click` listener, only `mouseenter`.
3. Bubble phase up through the ancestors to `window`: none of the ancestors have click listeners.
So `preventDefault` is **never called** on a Gmail-row click, before or after the fix. The
difference the fix makes is the *default action*: `mailto:` (no-op without an OS mail client, never
reaches Gmail) → `https://mail.google.com/...` (reliably opens Gmail compose in a new tab).

## ISSUE 4 — Cache-bust

- `index.html`: `css/style.css?v=39` → `?v=40`; `js/main.js?v=38` → `?v=39`.
- Confirmed served: `GMAIL_ANCHOR_OK`, `LINKEDIN_ANCHOR_OK`, `CSS_V40_OK`, `JS_V39_OK`.

## Summary of changes per file

- `js/main.js`: `English` language level `60` → `70` (one line; `renderSkillStats` already wires
  level → bar width and "LV n" label).
- `index.html`:
  - Gmail anchor `href` `mailto:rakesaa17@gmail.com` →
    `https://mail.google.com/mail/?view=cm&fs=1&to=rakesaa17@gmail.com`, added
    `target="_blank" rel="noopener noreferrer"` (matches the LinkedIn row).
  - LinkedIn `<path d=...>`: removed the outer rounded-square subpath (`M19 3...h14z`) from both
    the big icon block and the small sender-row glyph, leaving the white "in" letters; the blue
    square comes from existing CSS.
  - Cache-bust: `style.css?v=40`, `main.js?v=39`.
- `css/style.css`: unchanged (the LinkedIn blue square / white ink and pointer-events rules were
  already correct).

## Conclusion
All three findings are fixed and verified at runtime in a real browser:
1. English proficiency bar renders at 70% with label "LV 70".
2. The LinkedIn logo renders both the "i" and "n" glyphs on the blue square.
3. Clicking the Gmail row reliably opens Gmail (compose to rakesaa17@gmail.com, sign-in first if not
   authenticated); the LinkedIn row opens https://www.linkedin.com/in/rakesa. No JS handler blocks
   either anchor.
