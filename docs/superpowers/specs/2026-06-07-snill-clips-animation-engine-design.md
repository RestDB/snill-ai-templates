# snill-clips — script-driven animation engine

**Date:** 2026-06-07
**Status:** Approved design, ready for implementation planning

## Problem

Producing good explainer videos for snill.ai (how to create, change, and use apps)
with voiceover and editing is slow and hard. We want a faster path: write a *script*
that declares the purpose, the steps, and the final result of a short animation, where
each step composes pre-captured screenshots with zoom-ins, text overlays, highlights,
and an animated cursor — all timed and easily tuned by editing the script.

## Goals

- Author a short clip as **code** (React components), tune timing live, render to MP4.
- Cover marketing/social distribution: muted-autoplay-friendly, no spoken narration.
- One script renders to **multiple aspect ratios** (square, vertical, landscape) without forking.
- Visuals come from **pre-captured screenshots** the author drops in, not live app capture.
- Zero ongoing commercial license cost (see Licensing).

## Non-goals (v1 — YAGNI)

- No spoken/AI narration or audio sync — text overlays only (background music allowed).
- No live-driving of the real snill app to generate frames.
- No CI auto-rendering — render locally via CLI.
- No visual anchor-picker tool — anchors are hand-typed image-relative coordinates.
- No end-to-end video diffing — light frame-snapshot tests only.

## Audience & output

- **Audience:** marketing / social (YouTube, landing pages, Reels/Shorts, X/LinkedIn feeds).
- **Narration:** text overlays only; optional background music. No voice.
- **Output:** MP4, rendered from a single size-agnostic script into three size presets:
  - Square 1:1 (1080×1080)
  - Vertical 9:16 (1080×1920)
  - Landscape 16:9 (1920×1080)
- **GIF/WebM:** optional `--gif` flag, not a primary target.

## Engine: Remotion

React framework that renders deterministic MP4/WebM via headless Chrome + ffmpeg.
Chosen because it matches the programmatic (React component) authoring style we want and
provides, out of the box: `<Sequence>` timing, frame-accurate easing/springs, multiple
compositions from one codebase, audio tracks, and a live preview **Studio** for
scrub-while-you-tune editing.

### Licensing (important constraint)

Remotion's license grants a **free license** to: individuals, non-profits, evaluators,
and **for-profit companies with up to 3 employees**. For-profit companies with **4+
employees must buy a paid per-seat Company License**.

**This design assumes we qualify for the free license** (for-profit ≤3 employees, or
non-profit). **Revisit the engine choice if headcount reaches 4+** — the fallback is a
custom renderer (React components + Puppeteer frame capture + ffmpeg) built on
permissively-licensed dependencies only, preserving the same component-authoring model.

Sources: Remotion `LICENSE.md`, remotion.dev/docs/license, remotion.dev/blog/company-licenses.

## Repository structure

A new standalone repo, `snill-clips` (a Remotion project with a thin CLI):

```
snill-clips/
├── src/
│   ├── kit/              # reusable script vocabulary (the components below)
│   │   ├── Clip.tsx  Card.tsx  Shot.tsx  Caption.tsx
│   │   ├── Zoom.tsx  Callout.tsx  Spotlight.tsx  Cursor.tsx
│   │   ├── Transition.tsx
│   │   ├── useSize.ts    # active composition dimensions + layout helpers
│   │   └── theme.ts      # snill brand: colors, fonts, easing, per-size safe areas
│   ├── clips/            # one folder per animation = the "scripts"
│   │   └── expense-intro/
│   │       ├── clip.tsx          # the authored script (style B)
│   │       ├── anchors.ts        # named image-relative regions for this clip
│   │       └── assets/           # pre-captured screenshots for THIS clip
│   ├── sizes.ts          # the three size presets
│   └── Root.tsx          # auto-registers every clip × every size as a composition
├── out/                  # rendered output (gitignored)
├── render.ts             # CLI wrapper around Remotion's renderer
└── package.json
```

**Flow:** author `clips/<name>/clip.tsx` → preview/tune in Remotion Studio →
`npx snill-clips render <name>` → `out/<name>.1x1.mp4`, `.9x16.mp4`, `.16x9.mp4`.

## The component kit (script vocabulary)

| Component | Job |
|-----------|-----|
| `<Clip size music theme>` | Root of a script → Remotion `<Composition>`; sets aspect ratio, background music, brand theme. |
| `<Card>` | Full-screen branded intro/outro title card (no screenshot). `variant="outro"` for closing. |
| `<Shot src dur enter exit>` | Show one screenshot for `dur` seconds; establishes a coordinate space + named anchors for children; `enter`/`exit` per-shot transition. |
| `<Caption pos>` | Timed text overlay. `pos` is a safe-area slot (`bottom`/`top`/`center`), not raw pixels. |
| `<Zoom to scale>` | Ken-Burns pan/zoom; `to` is a named anchor or image-relative point. |
| `<Callout at text>` | Pill/arrow annotation pinned to an image-relative point. |
| `<Spotlight on>` | Dims/blurs everything except a rounded highlight box over the target (anchor or region). |
| `<Cursor from to click>` | Animated fake mouse that moves and optionally clicks — for "how to use" demos. |
| `<Transition kind>` | Between-shot transition (cut/fade/slide/push). |
| `theme.ts` | Brand tokens: colors, font, caption styling, default easing, per-size safe-area margins. |

### Author-once → three sizes

The mechanism that lets one script render at every aspect ratio without forking:

- Screenshots are **fit (contain)** inside the frame.
- Every anchor / callout / zoom target / spotlight region is stored in **image-relative
  coordinates (0–1)**, so targets stay accurate regardless of frame shape.
- Captions live in **named safe-area slots**, not absolute pixels.
- A `useSize()` hook exposes the active composition's dimensions; the kit re-lays-out
  automatically.

Authors write positions against the *image*, never against pixels.

### Named anchors

Each clip has an `anchors.ts` mapping name → image-relative box:

```ts
export default {
  "expense-card":  { x: .32, y: .40, w: .20, h: .15 },
  "dashboard-tab": { x: .81, y: .12, w: .14, h: .06 },
};
```

`<Zoom>`, `<Callout>`, and `<Spotlight>` resolve names through this map.

### Example script (target authoring experience)

```tsx
export default () => (
  <Clip size={SIZE} music="upbeat-1" theme={snill}>
    <Card dur={1.5}>Build an expense app in 10 seconds</Card>

    <Shot src="gallery.png" dur={2.5} enter="fade">
      <Caption pos="bottom">Start from a template</Caption>
      <Zoom to="expense-card" scale={1.4} />
    </Shot>

    <Shot src="app-created.png" dur={3} enter="slide-left">
      <Caption pos="bottom">Your app is ready</Caption>
      <Callout at="dashboard-tab" text="Live in seconds" />
    </Shot>

    <Card dur={1.5} variant="outro">Try snill.ai →</Card>
  </Clip>
);
```

## Authoring & render workflow

- **Preview/tune:** `npm run studio` → Remotion Studio. Pick any `clip@size` composition,
  scrub the timeline, edit `clip.tsx`, hot-reload.
- **Render:** `npx snill-clips render <name>` → all three sizes into `out/`.
  - `--size 9x16` render a single size
  - `--gif` also emit a loop
- **Auto-registration:** `Root.tsx` discovers every `clips/*/clip.tsx` and registers it
  against each size preset (composition id `name@9x16`, etc.). A new clip folder appears
  automatically — no manual wiring.

## Testing (kept light)

- **Typecheck** — typed kit props catch most authoring mistakes at compile time.
- **Frame-snapshot tests** — render ~3 key frames per clip via Remotion `renderStill` to
  PNG, compare against committed references; catches kit/layout regressions locally.
- No heavyweight end-to-end video diffing in v1.

## Open items to settle during planning

- Final list of size presets and exact safe-area margins per size.
- Background-music handling: shared `music/` folder + per-clip selection; trimming/fade to
  clip length.
- Whether `<Transition>` uses `@remotion/transitions` or a small custom set.
- Brand theme values (colors, font files) — pull from snill's design tokens.
