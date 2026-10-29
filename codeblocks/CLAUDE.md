# Codeblocks — CLAUDE.md

Live, one-way "code broadcast" for lectures. The teacher posts Markdown messages
(mostly fenced code blocks) and students see them appear in real time, with
one-click copy for code blocks, inline code and whole messages.

Hosted on GitHub Pages as part of `lukas-folkman.github.io`:
- Student page: `https://lukas-folkman.github.io/codeblocks` → `index.html`
- Teacher page: `https://lukas-folkman.github.io/codeblocks/post/` → `post/index.html`

## Files

| File | Role |
|---|---|
| `index.html` | Student viewer (read-only, no auth). Join by 6-char code or `?session=CODE`. |
| `post/index.html` | Teacher console (Firebase email/password auth). Create/manage sessions, post, edit, hide, export/import. |
| `icon.svg`, `icon_1024.png` | App icon (isometric cube with "C"). Same SVG is inlined in both pages' header and as a data-URI favicon. |

No build step, no package manager, no framework. Each page is a **single
self-contained HTML file** (inline CSS + JS). Keep it that way.

## Stack

- Firebase **Realtime Database** (compat SDK 12.6.0 from gstatic), project `codeblocks-io`, region asia-southeast1.
- Firebase **Auth** (email/password) — teacher page only; the student page deliberately does not load the auth SDK.
- highlight.js 11.9.0 (cdnjs) for syntax highlighting; only applied to fenced blocks that declare a language. Gracefully no-ops if the CDN fails.
- Plain ES (no modules); `var`-free but `function`-style code, IIFEs for init blocks.
- The Firebase web config is public by design (identical in both files); security comes from DB rules.

## Data model (RTDB)

```
sessions/
  {CODE}/                       # 6 chars from "ABCDEFGHJKMNPQRSTUVWXYZ23456789" (no 0/O/1/I/L)
    meta/   { name?: string, createdAt: serverTimestamp, allowCopy?: bool,
              defaultLang?: "java"|"python"|"javascript"|"sql", collapseAt?: int }
    messages/
      {pushKey}: { text: string, ts: serverTimestamp, shared: bool }
```

- Ordering = push-key order (`orderByKey()`), not `ts`.
- `shared` missing or `true` ⇒ visible; `false` ⇒ hidden from students.
- `meta/allowCopy` missing or `false` ⇒ students don't get the whole-message Copy button (default). Only written as `true` on create (duplicate/import) so session creation works even if rules don't list the field; the tick box writes `true`/`false`. **The Firebase rules must allow `meta/allowCopy`.**
- `meta/defaultLang` missing ⇒ bare ``` blocks stay unhighlighted. Set ⇒ they're highlighted as that language **at render time only** (stored Markdown never changes). Values are hljs language names, whitelisted by `normalizeLang` / `DEFAULT_LANGS`. Rules must allow it too.
- `meta/collapseAt` missing ⇒ 20. Code blocks with **more than** this many lines render collapsed. Only written when ≠ 20 (set to null when back to 20). A huge number effectively disables collapsing (no separate off switch, by design). Rules must allow it.
- Session code is generated client-side with `Math.random()`; no collision check.

### Security rules (not in this folder — live in Firebase console)
Code comments imply: writes are granted only at `sessions/{id}/meta` and
`sessions/{id}/messages` (not at `{id}` level), teacher (authed) can read
`sessions` root; students can read a session by code. Consequences baked into code:
- **Delete session** = `ref("sessions/"+id).update({meta:null, messages:null})` (multi-path update), never `.remove()` on `{id}`.
- **Create from data** (duplicate/import) writes `meta`+`messages` in one `update()` at `{id}`.
- Comments refer to an external design doc ("Section 5.1", "Section 10, step 7") — not present here.

## Shared code (duplicated verbatim in both pages)

These blocks MUST stay identical between `index.html` and `post/index.html`
(only the "shared with X page" comment differs). Change both together:
1. **Colour palettes** — CSS vars under `html[data-palette="orange|blue|green"]`; default orange. Picker stored in `localStorage["cb_palette"]`.
2. **Syntax theme** — fixed `.hljs-*` colours, deliberately palette-independent. Status colours (shared green `#3F7A39`, hidden red `#C0392B`) also fixed.
3. **Markdown engine** — hand-written: `escapeHtml`, `splitTableRow`, `sanitizeUrl`, `inline`, `parseListBlock`, `isListBreaker`, `renderMarkdown`. Supports headings (ATX + Setext), fenced code, inline code, bold/italic/strike, links/images/autolinks, blockquotes (recursive), tables with alignment, nested lists, hr.
   - Escapes HTML first, then applies patterns ⇒ raw HTML in messages is never rendered (XSS-safe by construction). `sanitizeUrl` blocks `javascript:/data:/vbscript:`.
   - Placeholders: `@@CB{n}@@` for fenced blocks, `@@IC{n}@@` for inline code.
   - **Every newline is a hard `<br>`** (chat-style, intentional — not CommonMark).
4. **Copy-to-clipboard** — `copyText` + `fallbackCopy` (textarea/execCommand), "Copied!" + checkmark flash for 1.2 s. Event-delegated via `wireCopyDelegation(stream)`; whole-message copy uses raw text from `messageStore`.
5. **Message rendering** — `appendMessage`, `updateMessage`, `removeMessage`, `highlightWrap`, `rerenderAll`, `fmtTime` (HH:MM:SS local).
   - **Default language mechanism:** `renderMarkdown` emits `<code class="language-X">` only for ```X fences. `highlightWrap` first adds `language-${defaultLang}` to any `.code-block code` without a language class (if hljs has that language), then runs `hljs.highlightElement` on all `language-*` blocks. Explicit fences always win; inline code untouched.
   - Because hljs replaces text with spans, changing `defaultLang` calls `rerenderAll(stream)`, which re-renders every on-screen message from raw text in `messageStore`.
6. **Collapsible code blocks** — `COLLAPSE_DEFAULT=20`, `PREVIEW_LINES=10`, `collapseAt`, `expandByDefault`, `blockState`, `normalizeCollapseAt`, `setupCollapsibles` (called after `highlightWrap` in `appendMessage`/`updateMessage`), `applyBlockState`, `toggleBlock`, `setAllBlocks`, `wireCollapseDelegation`.
   - Purely visual: `pre.style.maxHeight` = top padding + measured line pitch × (10 + 0.6) so 10 clear lines + a hint of the 11th under a 24px fade. Copy still copies the full text; DB/exports untouched; `renderMarkdown` unchanged.
   - Long blocks get a `.block-tools` toolbar (`N lines` badge, Expand/Collapse, and the Copy button moved into it) and a bottom `.collapse-bar` ("▾ Show all N lines" / "▴ Show less"). Clicking the faded preview expands (unless text is selected). Collapsing scrolls the block's top back into view.
   - All long blocks start collapsed (no auto-expand of the newest — avoids layout jumps). Per-viewer, per-block state in `blockState` keyed `msgId:blockIndex`, survives re-renders, not reloads. **Expand all / Collapse all** (both headers) set `expandByDefault` (also applies to blocks arriving later) and clear per-block overrides.
7. **Width slider** — 0–100 mapped piecewise: 0→420px, 50→760px (default), 100→2400px; sets `--content-width`; `localStorage["cb_content_width"]`; hidden < 820px.

Consider extracting these to shared files only if Lukas asks — single-file pages were a deliberate choice.

## Student page behaviour (`index.html`)

- Code input normalised: uppercase, strip non-alphanumerics, max 6. Existence check via one-time read of `meta`.
- URL updated with `history.replaceState` to `?session=CODE` so refresh rejoins; Leave clears it.
- Header title = live `meta/name` (falls back to code).
- Live listener on `meta/collapseAt` → `collapseAt` + `rerenderAll`; reset (and `blockState` cleared) on Leave. Header has Expand all / Collapse all (in-session only).
- Live listener on `meta/defaultLang` → sets `defaultLang` and `rerenderAll` on change; reset to "" on Leave. Hidden messages re-inserted later pick up the current value.
- Live listener on `meta/allowCopy`: toggles class `no-msg-copy` on `.stream`, which CSS-hides `.msg-copy-btn`. Class is on by default (before data arrives and after Leave). Code-block and inline-code copy buttons always shown.
- Listens `child_added` / `child_changed` / `child_removed`. Keeps `order[]` + `msgData{}` for **all** messages, including hidden ones, so a re-shared message is re-inserted at its original chronological position (`insertMessageInOrder`).
- Layout: page never scrolls (`body` fixed `100dvh`), only `.stream` scrolls.
- **Auto-scroll rule:** if viewer is at bottom (< 60 px) → follow new messages; if scrolled up → don't yank, instead the pinned bottom bar turns into an "N new messages — click to view" alert. Scrolling back to bottom clears it.
- Note: hidden messages are filtered **client-side**; anyone with the code can still read them from the DB. Treat "Unshare" as presentation, not secrecy.

## Teacher page behaviour (`post/index.html`)

- Login card → app with resizable sidebar (180–640 px, `localStorage["cb_sidebar_w"]`, drag handle) + main stream + composer.
- Sidebar: live `on("value")` on whole `sessions` root; sorted by `createdAt` desc; row shows name (or code), code · last message time. Hover tools: rename, duplicate, delete.
- Header: click title to rename active session. Controls: **Allow copy** tick box, **⇕ N** collapse-threshold number input (`#collapseInput`, commits on change/Enter, invalid ⇒ 20), frameless code-language dropdown (Plain/Java/Python/JS/SQL), Copy link (`STUDENT_PAGE_URL?session=CODE`), Expand all, Collapse all, Export Md, Export JSON (these only while a session is open), Import JSON, New session, Sign out. (There is deliberately no "Copy session code" button — the code is visible in the title/sidebar.)
- Teacher header is crowded: width slider is 80px with no Narrow/Wide labels; header buttons are compact (6px 8px, 12.5px); action group wraps to a second row below ~1440px.
- **Allow copy** (`#allowCopyBox`, unticked by default): per-session `meta/allowCopy`; synced from `sessionMeta` via `syncAllowCopy()` on open and on every sidebar refresh; reverts on write failure. Affects students only — the teacher's own message buttons are unchanged.
- **Code dropdown** (`#langSelect`): per-session `meta/defaultLang` (Plain ⇒ field removed). `syncLang()` on open and every sidebar refresh; re-renders the teacher stream too; reverts on write failure. Export Md leaves fences as typed.
- Composer: monospace textarea. Posts `{text, ts: SERVER_TIMESTAMP, shared: true}`. Shortcuts (Mod = ⌘ on Mac, Ctrl elsewhere; hint text adapts via `IS_MAC`/`MOD`):
  - Mod+Enter → post. **Mod+Shift+Enter** or the **Code** button (left of Post, same size) → post as code: `wrapAsCodeBlock` wraps the whole text in a bare ``` block (drops leading blank lines + trailing whitespace; posts unchanged if it already contains a ``` line, since nested fences aren't supported). Language comes from the session's Code dropdown.
  - **Mod+E** (composer and Edit-message modal), GitHub-style `toggleCodeInTextarea`: no selection → insert empty ``` block, caret inside; multi-line selection → wrap in ``` block; part of one line → wrap in `inline` backticks. Adds newlines so fences sit on their own lines.
  - Edits use `document.execCommand("insertText")` so native undo works (fallback `setRangeText`). On Mac only Cmd triggers (Ctrl+E is native end-of-line). Avoided ⌘⇧C/⌘⇧K (browser devtools).
- Per-message hover controls: Copy, Edit (modal textarea, Ctrl/Cmd+Enter saves, empty edit ignored), Delete (confirm; **Shift-click deletes immediately** — messages only, not sessions), Share/Unshare (no confirm — reversible). Card border: green = shared, red = hidden.
- **Themed modal** (`openModal({title, message, input|textarea, inputValue, placeholder, okLabel, danger, onOk})`) replaces native `prompt/confirm`. Single-line input is trimmed; textarea preserves exact text. Errors still use `alert()`.
- **Export JSON:** `{session, name, createdAt, allowCopy, defaultLang, collapseAt, exportedAt, messages:[{id, ts, text, shared}]}`, file `codeblocks-{CODE}.json`.
- **Export Markdown:** chat-style transcript; messages separated by `---` with blank lines around, bold timestamp, hidden ones tagged `_(hidden from students)_`; message text emitted verbatim at left margin so its own Markdown renders.
- **Import JSON / Duplicate** → `createSessionFromData(name, messages, allowCopy, lang, collapse)`: new code, preserves original `ts`, `shared`, `allowCopy`, `defaultLang` and `collapseAt` (missing in older JSON ⇒ off/none), fresh `createdAt` (so it tops the sidebar), opens it.
- Teacher stream always scrolls to bottom on new message (no "new messages" bar).

## Conventions / decisions to preserve

- Single-file HTML, no build tooling, CDN deps with pinned versions.
- All colours via palette CSS vars (except deliberate fixed ones above). Adding a palette = copy a block + add a swatch + extend `PALETTES`.
- Use `textContent` for user-supplied strings in DOM (names, codes); Markdown goes only through `renderMarkdown`.
- `localStorage` access always wrapped in try/catch; keys prefixed `cb_`.
- `<meta name="robots" content="noindex">` on both pages.
- British spelling in comments/UI ("Colour").

## Known gaps / ideas (not done)

- No SRI hashes on CDN scripts (comment suggests adding).
- `.msg-wrap` background hard-coded `#ffffff`; link underline colour hard-coded orange rgba; no dark mode.
- No collision check on generated session codes.
- No local testing setup; test by opening the pages against the live Firebase project.
