# Architecture v2 — Lexus Portfolio

Single-file static site today (`index.html` + `images/`), no build step, no
framework, no server config. This document extracts the conventions already
in `index.html` so every new page reuses the same building blocks, and defines
the routing/flow rules for the pages we're about to add.

---

## A. Existing conventions (extracted from `index.html`)

### Typographic scale

| Token / class | Value | Used by |
|---|---|---|
| `--headline-size` | `clamp(2.25rem, 7vw, 6.5rem)`, weight 600, line-height 1.12, letter-spacing -0.03em | `.hero__line` |
| `--caption-size` | `--headline-size / 5.5`, weight 400, color `#6b6b6b` | `.hero__caption` |
| `--headline-statement` | `clamp(2rem, 5vw, 3.5rem)`, weight 600, letter-spacing -0.03em | `.thesis__headline`, `.contact__headline` |
| body copy | `clamp(1.25rem, 2.4vw, 1.75rem)`, weight 400, line-height 1.6, color `#111111` | `.thesis__para` |
| tile caption | `1rem`, weight 500, letter-spacing -0.01em, line-height 1.4, color `#1A1917` | `.pov__thesis`, `.pov__caption`, `.proof__title` |
| tile link label | `0.8rem`, weight 400, opacity 0.7, color `#1A1917` | `.proof__link` |
| divider label | `1.5rem`, letter-spacing `0.25em`, weight 500 | `.divider-label` |
| name | `clamp(1.5rem, 3vw, 2.25rem)`, weight 600 | `.contact__name` |
| meta | `1rem`, weight 400, color `#6b6b6b` | `.contact__studies` |
| link | `1.125rem`, weight 500, white | `.contact__link` |
| footer | `0.85rem`, color `#6b6b6b` | `.contact__footer` |

**`mix-blend-mode` — settled: removed from the project entirely.** Tile
captions used to be white text with `mix-blend-mode: difference` so they'd
stay readable across a background that swung light→black→light. That
background swing is gone (see the background system below): captions sit on
a constant `#F4F2ED` page, so they're plain `#1A1917`, and the nav carries
its own opaque surface instead of blending with whatever is behind it. No
element on the site uses `mix-blend-mode` any more, and new ones shouldn't
reintroduce it — a caption on the light page is just dark ink, and anything
over imagery gets a real scrim or its own background, not a blend mode.

### Spacing scale

Base unit `--space: 8px`, multiples exposed as custom properties:
`--space-2` (16px), `--space-4` (32px), `--space-8` (64px), `--space-12` (96px),
`--space-16` (128px), `--space-24` (192px). Horizontal page padding is the
constant `8vw` everywhere, not a token.

### Naming / structure convention

BEM-style: `.block`, `.block__element`, `.block--modifier`. Each top-level
section is `min-height: 100vh` (or `auto` for tile sections), flex column,
padded with `--space-*` tokens vertically and `8vw` horizontally. Section
dividers (`.section-divider`) are their own standalone element between
sections, not nested inside either one.

### Navigation

- Fixed, 4 links only: Point of View, Proof, Thesis, Contact.
- The bar owns its surface: `#F4F2ED` background, 1px `#1A1917` border all
  the way around, `|` separators between links in the secondary text color
  `#6B6862`. Separators are their own `.nav__sep` spans, not `::before` on
  the links, so they stay out of the links' hit area and hover state.
- **No color flip.** `.nav--light` is gone — an opaque bar doesn't need to
  know what's behind it. Don't reintroduce a luminance-derived text color;
  if a section ever needs a different nav treatment, give the bar a
  different surface, not a text-color toggle.
- Auto-hide is the one nav state: `.nav--hidden` (a `translateY` transition,
  nothing else reflows) is toggled from a single scroll-direction
  `ScrollTrigger`. The bar stays put in the top `NAV_HIDE_AFTER` px of the
  page and during a programmatic anchor scroll (`isProgrammaticScroll`).
  That ScrollTrigger needs `scrub: true` for the same reason the background
  system's does: a trigger-less ScrollTrigger with no animation attached
  doesn't fire `onUpdate` on plain scroll without it.
- Mobile: burger opens a full-screen `.nav-mobile` overlay with the same 4
  links duplicated. It's always light now (it used to mirror `.nav--light`).
- Logo click scrolls to top; link clicks are intercepted (`preventDefault`)
  and routed through the custom scroll system below.

### Background system — one source of truth

One trigger-less `ScrollTrigger` (`start: 0, end: "max", scrub: true`) drives
one `onUpdate` (`makeBgColorSystem`). It interpolates `body.style.backgroundColor`
piecewise-linearly between color "stops." Each stop is `[() => y, [r,g,b]]`
where `y` is read live from real section `ScrollTrigger` boundaries
(`start`/`end` in px), not hardcoded viewport percentages — so stops track
actual section height on refresh. No section has its own independent
background tween.

**The system is down to one transition: Thesis → Contact.** `PAPER`
(`#F4F2ED`) holds from the top of the page all the way through Thesis, then
runs down to `DARK` (`#0a0a0a`) by the end of Contact. The POV-driven
light→black→light swing is gone on both breakpoints, and with it the
POV/Proof boundary triggers that existed only to feed it. The two
breakpoints now differ in exactly one thing — where "Thesis has finished
being read" falls: desktop reads it off the pinned timeline's trigger end,
mobile off the last paragraph's own reveal trigger end, each plus
`THESIS_BREATHING_ROOM`. The nav is not derived from this color at all any
more (see Navigation).

**Rule going forward: this is the only thing allowed to write
`body.style.backgroundColor`.** A new page/section that needs a background
transition adds a stop to the existing stops array (per breakpoint, via the
existing `bgMM` matchMedia branches) and calls the existing
`makeBgColorSystem`; it never creates a second `ScrollTrigger` that sets
background color.

### Anchor scrolling — `ScrollToPlugin`

Native anchor jump is disabled. `scrollToTarget(hash)`:
- if the target id is a "tile" section (`pov`, `proof` — via
  `imageAnchorSelectors`), it computes the target scroll `y` from the actual
  `<img>`'s live `getBoundingClientRect()`, so the image's vertical center
  lands centered in the viewport area below the fixed nav (not a fixed
  offset — stays correct at any nav/viewport height);
- otherwise (`thesis`, `contact`) it uses a fixed px offset from
  `navScrollOffsets`.

All of it animates via `gsap.to(window, { scrollTo: {...}, ease:
"power2.inOut", duration: 1 })`.

### Reveal pattern

Elements start `opacity: 0; transform: translateY(30px)` in CSS, revealed by
a scrubbed `ScrollTrigger` (typically `start: "top 85%", end: "top 45%"`), or
— for Thesis on desktop — as tweens inside the pinned timeline. The hero
headline is the one exception: it plays once on load via `gsap.from` with
stagger, not scroll-linked. The POV blocks are a deliberate exception to the
scrub rule — see section E.

---

## B. Routing

Static site, no framework — routes are folders with their own `index.html`
(works as clean URLs on GitHub Pages / Netlify / Vercel static hosting by
default):

```
/                          index.html                 (Hero, Point of View, Proof, Thesis, Contact)
/approach                  approach/index.html        (thesis expansion)
/case-study/golf           case-study/golf/index.html
/case-study/es             case-study/es/index.html
```

- `/approach` is reachable **only** by clicking the left POV tile (the
  event-photo tile) — it is not linked from nav, footer, or anywhere else.
  The right POV tile (portrait) stays non-interactive, as today.
- `/case-study/golf` and `/case-study/es` are reached from the two Proof
  tiles, which already have the `<a href="#">` shape ready to receive real
  links — replacing the placeholder `#`.
- Every subpage references `images/` at the correct relative depth
  (`../../images/...` from `case-study/*/index.html`, `../images/...` from
  `approach/index.html`).

## C. Flow rules

- **Back always goes to `/#proof`**, from every subpage — including
  `/approach`, even though `/approach` isn't linked from Proof. Back is a
  plain link (`href="/#proof"`), not a JS scroll call, since it's a
  cross-page navigation.
- Nav stays 4 items everywhere: Point of View, Proof, Thesis, Contact. No
  subpage adds itself to nav.
- One CTA per page.
- Contact remains a section on `/` only — never its own subpage.
- **Nav is static, not fixed, on every reading subpage** (`/case-study/*`
  and `/approach` alike — anything meant to be read top to bottom as a
  document, not just `/case-study/golf`). Design reasoning: a case study
  (or the thesis expansion) should read like a document, not a site with a
  menu — a nav that stays on screen through the whole read is a standing
  invitation to leave. Concretely, on these pages:
  - `.nav` gets `position: static` (index.html's nav stays `fixed` —
    unchanged, home page only).
  - No fixed-nav compensations: no extra top padding/`min-height` on the
    hero to clear a nav that no longer overlays, no `scroll-margin`. Adding
    any of these back on a static nav produces genuine dead space at the
    top of the page — check for that specifically after building the hero.
  - Since nav scrolls away, the page needs its own visible way back: a
    footer back-link (`.case-footer-nav` → `.case-back`, sized up from the
    header one, e.g. `1.125rem`) after the final CTA is the only
    navigational element while reading, and must read as clearly
    intentional, not a small utility afterthought.

## D. Technical rules

1. **One property, one source of truth.** No new `ScrollTrigger` may write
   `body.style.backgroundColor`. A subpage that wants a background transition
   extends the existing stops array in the existing system (see above).
2. **Required `index.html` change for routing to work:** on load, if
   `location.hash` matches one of the existing anchor ids (`#pov`, `#proof`,
   `#thesis`, `#contact`), call the existing `scrollToTarget(hash)` after
   `ScrollTrigger.refresh()` — so arriving from `/case-study/golf` with
   `/#proof` actually lands on Proof instead of just the page top. This is
   the only change routing requires in `index.html`; nothing else in it
   should move.
3. Every change tested on `localhost` before push.
4. One prompt = one system change — routing, content, and any animation
   proposal land as separate, reviewable steps, not bundled together.
5. **No invented facts in case-study copy.** Never add numbers, dates,
   durations, or specific details (years running, attendee counts, "the
   Nth time," named amenities like a tent) that aren't in the source
   material provided for that case study. If the tone needs a concrete
   detail and none was given, ask — don't estimate or infer one. Applies
   to every case study, not just Golf.

## E. Known deviations

Places where the code knowingly departs from a convention in section A.
Each one is a decision, not drift — don't "fix" them back, and don't take
them as licence to spread the exception further.

### 1. POV reveal is not scrubbed

The Reveal pattern above describes scrubbed `ScrollTrigger`s. The POV blocks
are the one reveal on the site that isn't: each block drives a plain
`gsap.timeline` with `toggleActions: "play none none reverse"`, photo at
t=0 and text 1s behind it.

Why: the point of that reveal is a fixed, deliberate rhythm — the same
unhurried entrance whether the visitor eases down the page or flicks past
it. Scrubbing would tie the timing to scroll speed, so a fast scroll would
compress the whole thing into a blink; the "premium" reading of the section
depends on it not doing that. `reverse` (rather than leaving the block
revealed) is what keeps the entrance available on the way back up.

Scope: POV only. Everything else stays scrubbed.

### 2. POV owns its own spacing tokens and breaks the page margin

Horizontal page padding is the constant `8vw` everywhere (section A). POV is
the exception: its photos reach past that padding to `4vw` from the window
edge, on the outer side of each block — left in block 1, right in the
mirrored block 2. Mechanically that's a negative margin on `.pov__frame`
plus a matching `max-width`, so the photo grows away from the centre axis
while its inner edge — and therefore the axis and the whole text column —
stays exactly where it was.

The section also carries its own variables, all declared on `.pov`:

| Token | Role |
|---|---|
| `--pov-media-h` | shared height ceiling both photos read, so the two blocks match by construction rather than one being fitted to the other |
| `--pov-media-inset` | how close a photo gets to the window edge (`4vw`); the bleed past the page padding derives from it |
| `--pov-axis-gap` | column gap; half of it is the distance from the centre axis to the photo *and* to the text, in both blocks |

Scope: POV only. Other sections keep the plain `8vw` and don't get their own
spacing tokens — a second section wanting a bleed is a design decision to
make deliberately, not a pattern to copy because POV did it.
