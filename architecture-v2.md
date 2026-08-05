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
| tile caption | `1rem`, weight 500, letter-spacing -0.01em, line-height 1.4, color `#fff` + `mix-blend-mode: difference` | `.pov__thesis`, `.pov__caption`, `.proof__title` |
| tile link label | `0.8rem`, weight 400, opacity 0.7, same white+difference-blend technique | `.proof__link` |
| divider label | `1.5rem`, letter-spacing `0.25em`, weight 500 | `.divider-label` |
| name | `clamp(1.5rem, 3vw, 2.25rem)`, weight 600 | `.contact__name` |
| meta | `1rem`, weight 400, color `#6b6b6b` | `.contact__studies` |
| link | `1.125rem`, weight 500, white | `.contact__link` |
| footer | `0.85rem`, color `#6b6b6b` | `.contact__footer` |

The white-text-with-`mix-blend-mode: difference` technique is the standard
way to caption an image tile without needing a separate scrim — reuse it,
don't invent a gradient overlay.

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

- Fixed, transparent, 4 links only: Point of View, Proof, Thesis, Contact.
- Color flip is a single modifier class, `.nav--light`, toggled by script —
  never set directly in markup.
- Mobile: burger opens a full-screen `.nav-mobile` overlay with the same 4
  links duplicated.
- Logo click scrolls to top; link clicks are intercepted (`preventDefault`)
  and routed through the custom scroll system below.

### Background system — one source of truth

One trigger-less `ScrollTrigger` (`start: 0, end: "max", scrub: true`) drives
one `onUpdate` (`makeBgColorSystem`). It interpolates `body.style.backgroundColor`
piecewise-linearly between color "stops." Each stop is `[() => y, [r,g,b]]`
where `y` is read live from real section `ScrollTrigger` boundaries
(`start`/`end` in px), not hardcoded viewport percentages — so stops track
actual section height on refresh. `.nav--light` is derived in the *same*
`onUpdate` from the interpolated color's luminance (`< 128` → light nav). No
section has its own independent background tween, and there is no separate
enter/leave bookkeeping for the nav.

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
stagger, not scroll-linked.

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
