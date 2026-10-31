# Emoji Copy Paste

Personal convenience page for copying emojis to the clipboard. Single self-contained
`index.html` (no build, no dependencies besides one runtime data fetch).

- Deployed via GitHub Pages from `~/projects/lukas-folkman.github.io/emoji/index.html`
  (that is the source of truth; this folder holds a mirror copy — keep them identical).
- Click an emoji → copied. Shift+click → appends to a multi-emoji buffer (buffer resets on Shift keyup).
- `FAVORITES` (top `<script>`) and `CATEGORIES` (object of name → emoji string) are the
  only things normally edited. Strings are split into graphemes with `Intl.Segmenter`
  (so ZWJ sequences/flags work) and de-duplicated per grid.

## Search
- ⌘F / Ctrl+F is intercepted and focuses `#q`; any printable key typed outside the box
  starts a fresh search. Enter copies the first result (Shift+Enter appends), Esc clears.
- Keywords: emojibase-data (pinned `@17.0.0`, jsDelivr) `en/compact.json` (CLDR label + tags)
  plus `en/shortcodes/github.json`, plus the page category name(s) each emoji appears in.
- Searches all ~1,900 Unicode emojis, not just the page ones. All query tokens must match
  (exact word 3 > prefix 2 > substring 1); ties ranked favourites → on-page → rest.
- Variation selectors (U+FE0F/FE0E) are stripped for matching page emojis to dataset entries.
- Offline/fetch failure: falls back to category-name matching only.

## Conventions
- Keep it one file, vanilla JS, no frameworks. Test edits in a browser (clipboard API needs
  a secure context: https or file://).
