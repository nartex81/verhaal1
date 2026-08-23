# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

`verhaal1` ("story 1") is a **single-file interactive detective story** written in
Dutch: *De Schaduw van het Zilveren Uurwerk* ("The Shadow of the Silver Watch").
It is a personal gift/one-off web page for a reader named Marjan — a 4-chapter
whodunit where the player follows a branch, collects clues, reads suspect
profiles, takes notes, and finally accuses someone.

The entire application is `index.html`. There is nothing else.

```
verhaal1/
├── index.html   # the whole app: markup + CSS + JS in one file
├── LICENSE      # MIT
└── CLAUDE.md    # this file
```

## Hard constraints

- **No build step, no bundler, no package manager, no dependencies, no tests,
  no CI.** Do not introduce any of these. Do not add `package.json`,
  a framework, or a `node_modules` workflow.
- **Everything stays in `index.html`.** Do not split the CSS or JS into separate
  files unless the user explicitly asks. The page is meant to be opened directly
  from the filesystem or served as a static file (e.g. GitHub Pages).
- **No network requests, no external assets.** No CDN scripts, no web fonts, no
  images. Visual texture comes from CSS gradients and system fonts
  (Georgia serif for prose, Arial for UI labels); "icons" are Unicode glyphs
  (`◌ ◷ ◇ ✦ ✹ ▣`) in the clue data.
- **All user-facing copy is Dutch.** Keep new strings in Dutch and in the same
  register as the existing prose: literary, second person, present tense,
  Belgian/Flemish spelling (the story is set in Gierle). Code identifiers and
  comments are the only place English is appropriate.

## File layout inside `index.html`

The file is written in a **deliberately compact, near-minified style**: 8 physical
lines, with the whole stylesheet on one line and the whole script on one line.

| Line | Contents |
| --- | --- |
| 1–2 | doctype, `<head>`, `<title>`, start of `<style>` |
| 3–5 | all CSS (line 5 is the bulk: variables, layout, responsive media queries) |
| 6 | all HTML markup (`<body>` … `</body>`), one line |
| 7 | all JavaScript: data tables, state, `render()` |
| 8 | closing tags |

When editing, **preserve this style**. Do not reformat/prettify the file — a
reflow produces an unreadable diff and buries the actual change. Make surgical
edits to the existing single lines instead (`sed`, or a targeted string
replacement on a unique substring).

## Architecture

The app is a tiny **state → `render()`** loop with no framework.

### State

One object, persisted to `localStorage` under the key
**`schaduw-uurwerk-save-v2`**:

```js
{ chapter: 0, selected: [], notes: "", path: "", suspect: "", paused: false }
// plus `ending: 1` once the verdict has been spoken
```

`chapter: 0` means the hero/landing screen; 1–4 are the chapters.

### Data tables

Content lives in two array-of-arrays constants at the top of the script — this is
where you edit story content, not in the markup:

- `C` — clues: `[id, title, category, glyph, dossier text]` (6 entries)
- `S` — suspects: `[id, name, role, alibi, hidden card]` (3 entries)

The clue grid, dossier, suspect cards, and accusation buttons are all generated
from these by `render()`. Adding a 7th clue means also updating the hard-coded
`' / 6'` counter and the `progress` formula in `render()`, and checking the
3-column `.clue-grid` CSS.

### Control flow

Every mutator writes to `st`, then calls `save()`, which writes `localStorage`
and calls `render()`. `render()` rebuilds the dynamic DOM from scratch. There is
no partial update path — **follow this pattern for any new interaction**:

- `showChapter(n)` / `showIntro()` — navigation (also scrolls to top)
- `path(p)` — chapter 1 branch, `'serre'` or `'doos'`
- `toggle(id)` — add/remove a clue from the dossier
- `choose(id)` — pick a suspect
- `togglePause()` — pause overlay
- `reveal()` — lock in the verdict (`st.ending = 1`)
- `reset()` — clear the save and reload

Handlers are wired as inline `onclick=` attributes in the markup and are
therefore **global functions** — keep new handlers global too, or they will
silently fail.

### Screen switching

Sections carry `class="chapter-screen" data-chapter="N"`; `render()` sets
`.active` on the ones matching `st.chapter` and toggles `.screen-hidden` on
`.hero`. Note **two sections share `data-chapter="3"`** (suspects + private
notes) — they are shown together. Chapter 3 has no `id`; chapters 1, 2 and 4 do.

### Ending logic

The "correct" solution is hard-coded in `render()`:

```js
st.suspect === 'noor' && st.selected.includes('message') && st.selected.includes('rain')
```

Accusing Noor *without* both the unsent message and the wet paw print gives the
"something is still missing from your dossier" ending. Any change to the culprit
or the required clues must be made here **and** in the reveal prose next to it.

## Styling conventions

- Palette is four CSS custom properties on `:root`: `--ink`, `--paper`, `--red`,
  `--line`. Use these rather than new hex literals where possible; the darker
  panels (hero, suspects, finale) use their own inline hex values by design.
- Layout uses `.wrap` (`min(1120px, 100% - 48px)`) as the content container.
- Three responsive tiers: default (desktop), `701–900px` (tablet), `≤700px`
  (mobile). Any new grid needs a matching rule in **both** narrow breakpoints —
  the existing `.clue-grid` / `.suspect-grid` rules show the pattern, including
  the `nth-child` border resets.

## Development workflow

There is no toolchain. To work on this:

```bash
# open it directly
xdg-open index.html          # or just double-click

# or serve it, which is closer to production
python3 -m http.server 8000  # then http://localhost:8000
```

Verification is manual. After any change, check at minimum:

1. A fresh visitor (clear `localStorage` or use `reset()`) sees the hero and can
   reach chapter 4.
2. Reload mid-story restores chapter, clues, notes, branch and pause state.
3. The correct-accusation path and the incomplete-dossier path both render.
4. The `≤700px` layout still works (no horizontal overflow).

Note that saves are keyed `…-v2`; if you make a change that would break an
existing save shape, bump the key to `-v3` rather than leaving readers with a
corrupt state object.

## Git conventions

- Work on the branch you were assigned; do not push to `main` without being
  asked.
- History so far is upload-style commits ("Add files via upload") made through
  the GitHub web UI. Write proper descriptive commit messages for your own work
  instead of matching that.
