# snill-clips Animation Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `snill-clips`, a standalone Remotion project where each short marketing animation is authored as a React "script" composing pre-captured screenshots with zooms, captions, callouts, spotlights, and a cursor, rendered to MP4 in three aspect ratios from one source.

**Architecture:** Remotion (React → MP4 via headless Chrome + ffmpeg). Every visual kit component is split into a **pure logic function** (frame/size in → style object out, unit-tested with Vitest) and a **thin React wrapper** (spreads the computed style). All positions are stored in **image-relative coordinates (0–1)** and resolved to pixels at render time against the contain-fit image rect, so one script renders square/vertical/landscape unchanged. A small `render.ts` CLI bundles once and renders every `clip@size` composition.

**Tech Stack:** TypeScript, React 18, Remotion 4 (`remotion`, `@remotion/cli`, `@remotion/bundler`, `@remotion/renderer`, `@remotion/transitions`, `@remotion/media-utils`), Vitest, tsx.

**Licensing note:** This rests on qualifying for Remotion's free license (for-profit ≤3 employees or non-profit). If headcount reaches 4+, a paid Company License is required — revisit before that point.

---

## Architectural principle (read first)

For each visual component `Foo`, create two things in `src/kit/Foo.tsx`:

1. A **pure exported function** `fooStyle(...)` (or `fooState`) — takes plain numbers/objects (current frame, target rect, size preset, theme) and returns a plain object (CSS style / position / state). No React, no Remotion hooks inside it. **This is what tests target.**
2. A **React component** `<Foo>` — calls Remotion hooks (`useCurrentFrame`, `useVideoConfig`), calls the pure function, and renders a `<div>`/`<AbsoluteFill>` with the result spread into `style`.

This keeps the testable logic isolated from the un-unit-testable rendering. Frame-snapshot tests (Task 15) cover the wiring.

## File structure

```
snill-clips/
├── package.json                 # deps + scripts (Task 1)
├── tsconfig.json                # (Task 1)
├── remotion.config.ts           # (Task 1)
├── vitest.config.ts             # (Task 1)
├── .gitignore                   # (Task 1)
├── render.ts                    # CLI: bundle + render every clip@size (Task 13)
├── public/
│   └── music/                   # shared background tracks, e.g. upbeat-1.mp3 (Task 12)
├── src/
│   ├── index.ts                 # registerRoot(Root) (Task 2)
│   ├── Root.tsx                 # registers buildRegistry() output as <Composition>s (Task 3)
│   ├── sizes.ts                 # SizePreset registry (Task 4)
│   ├── registry.ts              # buildRegistry() pure fn + clip discovery (Task 3)
│   ├── kit/
│   │   ├── theme.ts             # snill brand tokens (Task 5)
│   │   ├── layout.ts            # fitRect / resolveAnchor / anchorCenterPct / captionSlot (Task 6)
│   │   ├── useSize.ts           # active SizePreset from useVideoConfig (Task 7)
│   │   ├── animate.ts           # fadeUp / enterTransform shared helpers (Task 8)
│   │   ├── ShotContext.ts       # React context: image rect + anchors (Task 9)
│   │   ├── Clip.tsx             # root wrapper + music (Task 9)
│   │   ├── Shot.tsx             # screenshot + provides ShotContext (Task 9)
│   │   ├── Card.tsx             # intro/outro title card (Task 10)
│   │   ├── Caption.tsx          # captionStyle + <Caption> (Task 10)
│   │   ├── Callout.tsx          # calloutStyle + <Callout> (Task 12)
│   │   ├── Spotlight.tsx        # spotlightStyle + <Spotlight> (Task 13)
│   │   ├── Cursor.tsx           # cursorState + <Cursor> (Task 14)
│   │   └── index.ts             # re-exports the kit
│   │   # NOTE: Ken Burns zoom is a pure fn (zoomStyle in layout.ts) applied via
│   │   #       <Shot zoom={{...}}> — there is no Zoom.tsx component file.
│   └── clips/
│       └── expense-intro/
│           ├── clip.tsx         # the authored script + exported config (Task 16)
│           ├── anchors.ts       # named image-relative boxes (Task 16)
│           └── assets/          # gallery.png, app-created.png (Task 16)
└── tests/
    └── snapshot/                # renderStill reference PNGs (Task 15)
```

> Note: task numbers in the tree above are indicative; the authoritative order is the task list below.

---

## Task 1: Scaffold the project

**Files:**
- Create: `snill-clips/package.json`
- Create: `snill-clips/tsconfig.json`
- Create: `snill-clips/remotion.config.ts`
- Create: `snill-clips/vitest.config.ts`
- Create: `snill-clips/.gitignore`

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "snill-clips",
  "version": "0.1.0",
  "private": true,
  "license": "UNLICENSED",
  "scripts": {
    "studio": "remotion studio",
    "render": "tsx render.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@remotion/bundler": "4.0.300",
    "@remotion/cli": "4.0.300",
    "@remotion/media-utils": "4.0.300",
    "@remotion/renderer": "4.0.300",
    "@remotion/transitions": "4.0.300",
    "react": "18.3.1",
    "react-dom": "18.3.1",
    "remotion": "4.0.300"
  },
  "devDependencies": {
    "@types/node": "20.14.0",
    "@types/react": "18.3.3",
    "tsx": "4.16.0",
    "typescript": "5.5.3",
    "vitest": "2.0.5"
  }
}
```

> If `4.0.300` is not the latest, run `npx remotion versions` after install and align all `@remotion/*` + `remotion` to the same version — they MUST match.

- [ ] **Step 2: Create `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true,
    "lib": ["ES2020", "DOM"],
    "types": ["node"]
  },
  "include": ["src", "render.ts", "tests"]
}
```

- [ ] **Step 3: Create `remotion.config.ts`**

```ts
import { Config } from "@remotion/cli/config";

Config.setVideoImageFormat("jpeg");
Config.setOverwriteOutput(true);
// Allow importing .png/.mp3 assets co-located with clips.
Config.overrideWebpackConfig((current) => current);
```

- [ ] **Step 4: Create `vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["src/**/*.test.ts", "tests/**/*.test.ts"],
  },
});
```

- [ ] **Step 5: Create `.gitignore`**

```
node_modules/
out/
.DS_Store
```

- [ ] **Step 6: Install and verify**

Run: `cd snill-clips && npm install`
Expected: installs without peer-dependency errors.

Run: `npx tsc --noEmit`
Expected: PASS (no files yet to error on; exits 0).

- [ ] **Step 7: Commit**

```bash
git add snill-clips/package.json snill-clips/tsconfig.json snill-clips/remotion.config.ts snill-clips/vitest.config.ts snill-clips/.gitignore
git commit -m "chore: scaffold snill-clips Remotion project"
```

---

## Task 2: Entry point

**Files:**
- Create: `snill-clips/src/index.ts`

- [ ] **Step 1: Create `src/index.ts`**

```ts
import { registerRoot } from "remotion";
import { Root } from "./Root";

registerRoot(Root);
```

> `./Root` does not exist yet — it is created in Task 3. Typecheck will fail until then; that is expected. Do not commit this task alone; it commits together with Task 3.

---

## Task 3: Size presets

**Files:**
- Create: `snill-clips/src/sizes.ts`
- Test: `snill-clips/src/sizes.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/sizes.test.ts
import { describe, expect, it } from "vitest";
import { SIZES, SIZE_LIST, getSizeByDimensions } from "./sizes";

describe("sizes", () => {
  it("has three presets with matching ids", () => {
    expect(SIZE_LIST.map((s) => s.id).sort()).toEqual(["16x9", "1x1", "9x16"].sort());
  });

  it("square is 1080x1080", () => {
    expect(SIZES["1x1"].width).toBe(1080);
    expect(SIZES["1x1"].height).toBe(1080);
  });

  it("resolves a preset from raw dimensions", () => {
    expect(getSizeByDimensions(1080, 1920)?.id).toBe("9x16");
    expect(getSizeByDimensions(999, 999)).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd snill-clips && npx vitest run src/sizes.test.ts`
Expected: FAIL with "Cannot find module './sizes'".

- [ ] **Step 3: Implement `src/sizes.ts`**

```ts
export type SizeId = "1x1" | "9x16" | "16x9";

export interface SizePreset {
  id: SizeId;
  width: number;
  height: number;
  fps: number;
  /** Pixel margins kept clear of edges for captions/callouts. */
  safeArea: { top: number; right: number; bottom: number; left: number };
}

export const SIZES: Record<SizeId, SizePreset> = {
  "1x1": { id: "1x1", width: 1080, height: 1080, fps: 30, safeArea: { top: 80, right: 80, bottom: 120, left: 80 } },
  "9x16": { id: "9x16", width: 1080, height: 1920, fps: 30, safeArea: { top: 220, right: 80, bottom: 260, left: 80 } },
  "16x9": { id: "16x9", width: 1920, height: 1080, fps: 30, safeArea: { top: 80, right: 120, bottom: 100, left: 120 } },
};

export const SIZE_LIST: SizePreset[] = Object.values(SIZES);

export function getSizeByDimensions(width: number, height: number): SizePreset | undefined {
  return SIZE_LIST.find((s) => s.width === width && s.height === height);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/sizes.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Create `src/registry.ts` failing test**

```ts
// src/registry.test.ts
import { describe, expect, it } from "vitest";
import { buildRegistry } from "./registry";

const Dummy = () => null;

describe("buildRegistry", () => {
  it("creates one composition per clip per size", () => {
    const comps = buildRegistry([{ name: "demo", component: Dummy, durationInSeconds: 2 }]);
    expect(comps).toHaveLength(3);
    expect(comps.map((c) => c.id).sort()).toEqual(["demo-16x9", "demo-1x1", "demo-9x16"].sort());
  });

  it("computes durationInFrames from seconds × fps", () => {
    const comps = buildRegistry([{ name: "demo", component: Dummy, durationInSeconds: 2 }]);
    expect(comps.find((c) => c.id === "demo-1x1")!.durationInFrames).toBe(60);
  });

  it("carries width/height/fps from the size preset", () => {
    const comps = buildRegistry([{ name: "demo", component: Dummy, durationInSeconds: 2 }]);
    const vertical = comps.find((c) => c.id === "demo-9x16")!;
    expect([vertical.width, vertical.height, vertical.fps]).toEqual([1080, 1920, 30]);
  });
});
```

- [ ] **Step 6: Run to verify it fails**

Run: `npx vitest run src/registry.test.ts`
Expected: FAIL with "Cannot find module './registry'".

- [ ] **Step 7: Implement `src/registry.ts`**

```ts
import type { ComponentType } from "react";
import { SIZE_LIST, type SizePreset } from "./sizes";

export interface ClipModule {
  name: string;
  component: ComponentType;
  durationInSeconds: number;
}

export interface CompDescriptor {
  id: string;
  component: ComponentType;
  width: number;
  height: number;
  fps: number;
  durationInFrames: number;
}

export function buildRegistry(clips: ClipModule[], sizes: SizePreset[] = SIZE_LIST): CompDescriptor[] {
  return clips.flatMap((clip) =>
    sizes.map((size) => ({
      id: `${clip.name}-${size.id}`,
      component: clip.component,
      width: size.width,
      height: size.height,
      fps: size.fps,
      durationInFrames: Math.round(clip.durationInSeconds * size.fps),
    }))
  );
}
```

- [ ] **Step 8: Run to verify it passes**

Run: `npx vitest run src/registry.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 9: Implement `src/Root.tsx` (clip discovery + registration)**

```tsx
import { Composition } from "remotion";
import { buildRegistry, type ClipModule } from "./registry";

// Discover every clips/<name>/clip.tsx at bundle time.
// require.context is provided by Remotion's webpack bundler.
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const ctx = (require as any).context("./clips", true, /\/clip\.tsx$/);

const clips: ClipModule[] = ctx.keys().map((key: string) => {
  const mod = ctx(key);
  // key looks like "./expense-intro/clip.tsx"
  const name = key.split("/")[1];
  return {
    name,
    component: mod.default,
    durationInSeconds: mod.config.durationInSeconds as number,
  };
});

export const Root: React.FC = () => {
  return (
    <>
      {buildRegistry(clips).map((c) => (
        <Composition
          key={c.id}
          id={c.id}
          component={c.component}
          durationInFrames={c.durationInFrames}
          fps={c.fps}
          width={c.width}
          height={c.height}
        />
      ))}
    </>
  );
};
```

> Each `clip.tsx` MUST `export default` its component and `export const config = { durationInSeconds: N }`. `require.context` only works inside the Remotion bundle, so `Root.tsx` is not unit-tested — `buildRegistry` (tested above) holds the logic.

- [ ] **Step 10: Add a type declaration for `require.context`**

Create `src/require-context.d.ts`:

```ts
interface NodeRequire {
  context(
    directory: string,
    useSubdirectories: boolean,
    regExp: RegExp
  ): { keys(): string[]; (id: string): any };
}
```

- [ ] **Step 11: Typecheck**

Run: `npx tsc --noEmit`
Expected: PASS (index.ts now resolves Root; Root has no clips yet but compiles — `ctx.keys()` returns `[]` at runtime, which is fine).

- [ ] **Step 12: Commit**

```bash
git add src/index.ts src/sizes.ts src/sizes.test.ts src/registry.ts src/registry.test.ts src/Root.tsx src/require-context.d.ts
git commit -m "feat: size presets, composition registry, and root registration"
```

---

## Task 4: Theme tokens

**Files:**
- Create: `snill-clips/src/kit/theme.ts`
- Test: `snill-clips/src/kit/theme.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/theme.test.ts
import { describe, expect, it } from "vitest";
import { snill, type Theme } from "./theme";

describe("theme", () => {
  it("exposes the required brand tokens", () => {
    const t: Theme = snill;
    expect(t.colors.bg).toMatch(/^#/);
    expect(t.colors.text).toMatch(/^#/);
    expect(t.colors.accent).toMatch(/^#/);
    expect(t.fontFamily).toBeTruthy();
    expect(t.caption.fontSize).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/theme.test.ts`
Expected: FAIL with "Cannot find module './theme'".

- [ ] **Step 3: Implement `src/kit/theme.ts`**

```ts
export interface Theme {
  colors: { bg: string; text: string; accent: string; dim: string };
  fontFamily: string;
  caption: { fontSize: number; weight: number; radius: number; padX: number; padY: number };
  callout: { fontSize: number; weight: number };
  easing: { enterFrames: number };
}

// Brand values are placeholders to be replaced with snill's real design tokens
// (see spec "Open items"). Hex values keep the type contract concrete meanwhile.
export const snill: Theme = {
  colors: { bg: "#0B0B0F", text: "#FFFFFF", accent: "#3B82F6", dim: "rgba(0,0,0,0.6)" },
  fontFamily: "Inter, system-ui, sans-serif",
  caption: { fontSize: 44, weight: 600, radius: 14, padX: 28, padY: 16 },
  callout: { fontSize: 32, weight: 600 },
  easing: { enterFrames: 12 },
};
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/theme.test.ts`
Expected: PASS (1 test).

- [ ] **Step 5: Commit**

```bash
git add src/kit/theme.ts src/kit/theme.test.ts
git commit -m "feat: snill brand theme tokens"
```

---

## Task 5: Layout math (the core that makes one-script-three-sizes work)

**Files:**
- Create: `snill-clips/src/kit/layout.ts`
- Test: `snill-clips/src/kit/layout.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/layout.test.ts
import { describe, expect, it } from "vitest";
import { fitRect, resolveAnchor, anchorCenterPct, captionSlot, type Box } from "./layout";
import { SIZES } from "../sizes";

describe("fitRect (contain-fit image inside frame)", () => {
  it("letterboxes a wide image into a square frame", () => {
    // 1600x900 image into 1080x1080 frame -> width-bound
    const r = fitRect(1600, 900, 1080, 1080);
    expect(r.w).toBe(1080);
    expect(Math.round(r.h)).toBe(608); // 1080 * 900/1600
    expect(r.x).toBe(0);
    expect(Math.round(r.y)).toBe(236); // (1080-608)/2
  });

  it("pillarboxes a tall image into a landscape frame", () => {
    const r = fitRect(1080, 1920, 1920, 1080);
    expect(r.h).toBe(1080);
    expect(Math.round(r.w)).toBe(608);
  });
});

describe("resolveAnchor", () => {
  it("maps an image-relative box to pixels inside the fitted image rect", () => {
    const imageRect = { x: 0, y: 236, w: 1080, h: 608 };
    const box: Box = { x: 0.5, y: 0.5, w: 0.2, h: 0.1 };
    const r = resolveAnchor(box, imageRect);
    expect(Math.round(r.x)).toBe(540); // 0 + 0.5*1080
    expect(Math.round(r.y)).toBe(540); // 236 + 0.5*608
    expect(Math.round(r.w)).toBe(216);
  });
});

describe("anchorCenterPct", () => {
  it("returns the box center as a percentage of the frame", () => {
    const imageRect = { x: 0, y: 0, w: 1000, h: 1000 };
    const c = anchorCenterPct({ x: 0.4, y: 0.4, w: 0.2, h: 0.2 }, imageRect, 1000, 1000);
    expect(c.xPct).toBeCloseTo(50);
    expect(c.yPct).toBeCloseTo(50);
  });
});

describe("captionSlot", () => {
  it("bottom slot sits above the safe-area bottom margin", () => {
    const s = captionSlot("bottom", SIZES["1x1"]);
    expect(s.bottom).toBe(120);
    expect(s.left).toBe(80);
    expect(s.right).toBe(80);
    expect(s.textAlign).toBe("center");
  });

  it("top slot uses the top margin", () => {
    const s = captionSlot("top", SIZES["9x16"]);
    expect(s.top).toBe(220);
  });
});

describe("zoomStyle (Ken Burns transform)", () => {
  it("is identity at progress 0 with origin at the target center", () => {
    const s = zoomStyle({ xPct: 30, yPct: 70 }, 1.4, 0);
    expect(s.transform).toBe("scale(1)");
    expect(s.transformOrigin).toBe("30% 70%");
  });
  it("reaches the target scale at progress 1", () => {
    const s = zoomStyle({ xPct: 50, yPct: 50 }, 1.4, 1);
    expect(s.transform).toBe("scale(1.4)");
  });
});
```

Update the import line at the top of the test file to include `zoomStyle`:

```ts
import { fitRect, resolveAnchor, anchorCenterPct, captionSlot, zoomStyle, type Box } from "./layout";
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/layout.test.ts`
Expected: FAIL with "Cannot find module './layout'".

- [ ] **Step 3: Implement `src/kit/layout.ts`**

```ts
import type { SizePreset } from "../sizes";

/** Image-relative box, all fields in 0..1. */
export interface Box { x: number; y: number; w: number; h: number }
/** Pixel rect inside the frame. */
export interface Rect { x: number; y: number; w: number; h: number }
export type CaptionPos = "top" | "bottom" | "center";

/** Contain-fit an image (natural imgW×imgH) inside frameW×frameH, centered. */
export function fitRect(imgW: number, imgH: number, frameW: number, frameH: number): Rect {
  const scale = Math.min(frameW / imgW, frameH / imgH);
  const w = imgW * scale;
  const h = imgH * scale;
  return { x: (frameW - w) / 2, y: (frameH - h) / 2, w, h };
}

/** Map an image-relative box into frame pixels, given where the image is drawn. */
export function resolveAnchor(box: Box, imageRect: Rect): Rect {
  return {
    x: imageRect.x + box.x * imageRect.w,
    y: imageRect.y + box.y * imageRect.h,
    w: box.w * imageRect.w,
    h: box.h * imageRect.h,
  };
}

export function anchorCenterPct(box: Box, imageRect: Rect, frameW: number, frameH: number) {
  const r = resolveAnchor(box, imageRect);
  return { xPct: ((r.x + r.w / 2) / frameW) * 100, yPct: ((r.y + r.h / 2) / frameH) * 100 };
}

export interface CaptionSlotStyle {
  left: number;
  right: number;
  top?: number;
  bottom?: number;
  textAlign: "center";
}

export function captionSlot(pos: CaptionPos, size: SizePreset): CaptionSlotStyle {
  const { safeArea } = size;
  const base = { left: safeArea.left, right: safeArea.right, textAlign: "center" as const };
  if (pos === "top") return { ...base, top: safeArea.top };
  if (pos === "bottom") return { ...base, bottom: safeArea.bottom };
  return { ...base, top: Math.round(size.height / 2) - 40 };
}

/**
 * Ken Burns transform: scale a layer from 1 → `scale` as `progress` goes 0 → 1,
 * with the transform-origin pinned to the target's center (in % of the frame).
 * Pure so it can be unit-tested; consumed by <Shot zoom>.
 */
export function zoomStyle(
  center: { xPct: number; yPct: number },
  scale: number,
  progress: number
): { transformOrigin: string; transform: string } {
  const s = 1 + (scale - 1) * progress;
  return { transformOrigin: `${center.xPct}% ${center.yPct}%`, transform: `scale(${s})` };
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/layout.test.ts`
Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```bash
git add src/kit/layout.ts src/kit/layout.test.ts
git commit -m "feat: layout math for image-relative positioning across sizes"
```

---

## Task 6: Animation helpers

**Files:**
- Create: `snill-clips/src/kit/animate.ts`
- Test: `snill-clips/src/kit/animate.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/animate.test.ts
import { describe, expect, it } from "vitest";
import { fadeUp, progressIn } from "./animate";

describe("progressIn", () => {
  it("is 0 before start, 1 after the window, clamped", () => {
    expect(progressIn(0, 10, 10)).toBe(0);
    expect(progressIn(10, 10, 10)).toBe(0);
    expect(progressIn(15, 10, 10)).toBeCloseTo(0.5);
    expect(progressIn(40, 10, 10)).toBe(1);
  });
});

describe("fadeUp", () => {
  it("starts transparent and shifted, ends solid and in place", () => {
    const start = fadeUp(0, 0, 12);
    expect(start.opacity).toBe(0);
    expect(start.translateY).toBeGreaterThan(0);
    const end = fadeUp(60, 0, 12);
    expect(end.opacity).toBe(1);
    expect(end.translateY).toBe(0);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/animate.test.ts`
Expected: FAIL with "Cannot find module './animate'".

- [ ] **Step 3: Implement `src/kit/animate.ts`**

```ts
/** Linear 0..1 progress across [start, start+duration], clamped. */
export function progressIn(frame: number, start: number, duration: number): number {
  if (duration <= 0) return frame >= start ? 1 : 0;
  const t = (frame - start) / duration;
  return Math.max(0, Math.min(1, t));
}

/** Ease-out cubic. */
export function easeOut(t: number): number {
  return 1 - Math.pow(1 - t, 3);
}

export interface FadeUp { opacity: number; translateY: number }

/** Fade + rise entrance. translateY goes from +24px to 0. */
export function fadeUp(frame: number, start: number, duration: number, distance = 24): FadeUp {
  const t = easeOut(progressIn(frame, start, duration));
  return { opacity: t, translateY: distance * (1 - t) };
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/animate.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/kit/animate.ts src/kit/animate.test.ts
git commit -m "feat: shared animation helpers (progressIn, easeOut, fadeUp)"
```

---

## Task 7: useSize hook + ShotContext

**Files:**
- Create: `snill-clips/src/kit/useSize.ts`
- Create: `snill-clips/src/kit/ShotContext.ts`
- Test: `snill-clips/src/kit/useSize.test.ts`

> `useSize` is a thin hook over `useVideoConfig`. To keep its resolution logic testable without React, factor it into a pure helper `resolveSize(width, height)` and test that.

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/useSize.test.ts
import { describe, expect, it } from "vitest";
import { resolveSize } from "./useSize";

describe("resolveSize", () => {
  it("returns the matching preset", () => {
    expect(resolveSize(1080, 1920).id).toBe("9x16");
  });
  it("falls back to a synthetic preset for unknown dimensions", () => {
    const s = resolveSize(800, 600);
    expect(s.width).toBe(800);
    expect(s.height).toBe(600);
    expect(s.safeArea.left).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/useSize.test.ts`
Expected: FAIL with "Cannot find module './useSize'".

- [ ] **Step 3: Implement `src/kit/useSize.ts`**

```ts
import { useVideoConfig } from "remotion";
import { getSizeByDimensions, type SizePreset } from "../sizes";

export function resolveSize(width: number, height: number): SizePreset {
  const match = getSizeByDimensions(width, height);
  if (match) return match;
  const margin = Math.round(Math.min(width, height) * 0.07);
  return {
    id: "1x1",
    width,
    height,
    fps: 30,
    safeArea: { top: margin, right: margin, bottom: margin, left: margin },
  };
}

export function useSize(): SizePreset {
  const { width, height } = useVideoConfig();
  return resolveSize(width, height);
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/useSize.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Implement `src/kit/ShotContext.ts`**

```ts
import { createContext, useContext } from "react";
import type { Rect } from "./layout";
import type { Box } from "./layout";

export interface ShotContextValue {
  /** Where the screenshot is drawn within the frame (contain-fit). */
  imageRect: Rect;
  /** This clip's named anchors (image-relative boxes). */
  anchors: Record<string, Box>;
  /** Absolute frame at which this shot started. */
  shotStartFrame: number;
}

export const ShotContext = createContext<ShotContextValue | null>(null);

export function useShot(): ShotContextValue {
  const ctx = useContext(ShotContext);
  if (!ctx) throw new Error("Shot-scoped components must be rendered inside <Shot>");
  return ctx;
}

/** Resolve an anchor name (or throw) to its Box. */
export function useAnchor(name: string): Box {
  const { anchors } = useShot();
  const box = anchors[name];
  if (!box) throw new Error(`Unknown anchor "${name}". Add it to the clip's anchors.ts`);
  return box;
}
```

- [ ] **Step 6: Typecheck + commit**

Run: `npx tsc --noEmit`
Expected: PASS.

```bash
git add src/kit/useSize.ts src/kit/useSize.test.ts src/kit/ShotContext.ts
git commit -m "feat: useSize hook and ShotContext for anchor resolution"
```

---

## Task 8: Clip root + AnchorsContext

**Files:**
- Create: `snill-clips/src/kit/Clip.tsx`
- Create: `snill-clips/src/kit/AnchorsContext.ts`

> `<Clip>` provides the brand background, font, optional background music, and the clip's anchors to descendants. `<Shot>` (Task 9) reads anchors from here.

- [ ] **Step 1: Implement `src/kit/AnchorsContext.ts`**

```ts
import { createContext, useContext } from "react";
import type { Box } from "./layout";

export const AnchorsContext = createContext<Record<string, Box>>({});
export const useAnchors = () => useContext(AnchorsContext);
```

- [ ] **Step 2: Implement `src/kit/Clip.tsx`**

```tsx
import { AbsoluteFill, Audio, staticFile } from "remotion";
import type { ReactNode } from "react";
import type { Theme } from "./theme";
import type { Box } from "./layout";
import { AnchorsContext } from "./AnchorsContext";

export const Clip: React.FC<{
  theme: Theme;
  anchors: Record<string, Box>;
  music?: string;
  children: ReactNode;
}> = ({ theme, anchors, music, children }) => {
  return (
    <AnchorsContext.Provider value={anchors}>
      <AbsoluteFill style={{ backgroundColor: theme.colors.bg, fontFamily: theme.fontFamily }}>
        {music ? <Audio src={staticFile(`music/${music}`)} volume={0.6} /> : null}
        {children}
      </AbsoluteFill>
    </AnchorsContext.Provider>
  );
};
```

- [ ] **Step 3: Typecheck + commit**

Run: `npx tsc --noEmit`
Expected: PASS.

```bash
git add src/kit/AnchorsContext.ts src/kit/Clip.tsx
git commit -m "feat: Clip root with brand background, music, and anchors context"
```

---

## Task 9: Shot component

**Files:**
- Create: `snill-clips/src/kit/Shot.tsx`

> `<Shot>` measures the screenshot's natural size, computes its contain-fit rect, and provides `ShotContext` so children (`Callout`, `Spotlight`, `Cursor`, `Caption`) resolve anchors. It uses `@remotion/media-utils` `getImageDimensions` inside `delayRender`/`continueRender` for deterministic measurement. The contain-fit math is the already-tested `fitRect`.
>
> **Ken Burns zoom is a prop, not a child element.** `<Shot zoom={{ to, scale }}>` applies a `scale` transform (with transform-origin at the target anchor's center) to a wrapper containing the image *and* its anchored overlays (Spotlight/Callout/Cursor), so they zoom in lockstep and stay pinned. The transform math is the tested pure `zoomStyle` (Task 11). A shot with no `zoom` prop renders un-transformed.

- [ ] **Step 1: Implement `src/kit/Shot.tsx`**

```tsx
import { useEffect, useState, type ReactNode } from "react";
import { AbsoluteFill, Img, continueRender, delayRender, useCurrentFrame, useVideoConfig } from "remotion";
import { getImageDimensions } from "@remotion/media-utils";
import { fitRect, anchorCenterPct, zoomStyle } from "./layout";
import { easeOut, progressIn } from "./animate";
import { ShotContext } from "./ShotContext";
import { useAnchors } from "./AnchorsContext";

export interface ZoomSpec { to: string; scale?: number; inFrames?: number }

export const Shot: React.FC<{
  src: string;
  /** Duration in seconds; read by <Scenes> off props, not used internally. */
  dur: number;
  zoom?: ZoomSpec;
  children?: ReactNode;
}> = ({ src, zoom, children }) => {
  const { width, height } = useVideoConfig();
  const frame = useCurrentFrame();
  const anchors = useAnchors();
  const [dims, setDims] = useState<{ width: number; height: number } | null>(null);
  const [handle] = useState(() => delayRender(`measure ${src}`));

  useEffect(() => {
    let cancelled = false;
    getImageDimensions(src)
      .then((d) => {
        if (cancelled) return;
        setDims({ width: d.width, height: d.height });
        continueRender(handle);
      })
      .catch(() => continueRender(handle));
    return () => {
      cancelled = true;
    };
  }, [src, handle]);

  if (!dims) return null;

  const imageRect = fitRect(dims.width, dims.height, width, height);

  // Compute Ken Burns transform from the zoom prop (if any).
  let zoomCss: React.CSSProperties = {};
  if (zoom) {
    const box = anchors[zoom.to];
    if (!box) throw new Error(`Unknown anchor "${zoom.to}" in <Shot zoom>`);
    const center = anchorCenterPct(box, imageRect, width, height);
    const progress = easeOut(progressIn(frame, 0, zoom.inFrames ?? 30));
    zoomCss = zoomStyle(center, zoom.scale ?? 1.3, progress);
  }

  return (
    <ShotContext.Provider value={{ imageRect, anchors, shotStartFrame: 0 }}>
      <AbsoluteFill>
        <AbsoluteFill style={zoomCss}>
          <Img
            src={src}
            style={{
              position: "absolute",
              left: imageRect.x,
              top: imageRect.y,
              width: imageRect.w,
              height: imageRect.h,
            }}
          />
          {children}
        </AbsoluteFill>
      </AbsoluteFill>
    </ShotContext.Provider>
  );
};
```

> `shotStartFrame` is 0 because each `<Shot>` is wrapped in a `<TransitionSeries.Sequence>` by `<Scenes>` (Task 15), which rebases `useCurrentFrame()` to 0 at the shot's start — so entrance animations in children and the zoom all key off frame 0.
>
> Note: `<Caption>` is placed as a child here and so zooms with the shot. For the gentle scales used (≤1.4×) this reads fine in v1; pulling captions into a separate non-zooming layer is a deferred refinement.

- [ ] **Step 2: Typecheck + commit**

Run: `npx tsc --noEmit`
Expected: PASS.

```bash
git add src/kit/Shot.tsx
git commit -m "feat: Shot component with deterministic contain-fit measurement"
```

---

## Task 10: Caption component

**Files:**
- Create: `snill-clips/src/kit/Caption.tsx`
- Test: `snill-clips/src/kit/Caption.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/Caption.test.ts
import { describe, expect, it } from "vitest";
import { captionStyle } from "./Caption";
import { SIZES } from "../sizes";
import { snill } from "./theme";

describe("captionStyle", () => {
  it("places a bottom caption within the safe area and fades up on entry", () => {
    const s = captionStyle("bottom", 0, SIZES["1x1"], snill);
    expect(s.bottom).toBe(120);
    expect(s.opacity).toBe(0);
  });
  it("is fully visible after the entrance window", () => {
    const s = captionStyle("bottom", 60, SIZES["1x1"], snill);
    expect(s.opacity).toBe(1);
    expect(s.transform).toContain("translateY(0");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/Caption.test.ts`
Expected: FAIL with "Cannot find module './Caption'".

- [ ] **Step 3: Implement `src/kit/Caption.tsx`**

```tsx
import type { CSSProperties, ReactNode } from "react";
import { useCurrentFrame } from "remotion";
import { captionSlot, type CaptionPos } from "./layout";
import { fadeUp } from "./animate";
import { useSize } from "./useSize";
import { snill, type Theme } from "./theme";
import type { SizePreset } from "../sizes";

export function captionStyle(pos: CaptionPos, frame: number, size: SizePreset, theme: Theme): CSSProperties {
  const slot = captionSlot(pos, size);
  const fade = fadeUp(frame, 0, theme.easing.enterFrames);
  return {
    position: "absolute",
    left: slot.left,
    right: slot.right,
    top: slot.top,
    bottom: slot.bottom,
    textAlign: slot.textAlign,
    opacity: fade.opacity,
    transform: `translateY(${fade.translateY}px)`,
  };
}

export const Caption: React.FC<{ pos?: CaptionPos; children: ReactNode; theme?: Theme }> = ({
  pos = "bottom",
  children,
  theme = snill,
}) => {
  const frame = useCurrentFrame();
  const size = useSize();
  const outer = captionStyle(pos, frame, size, theme);
  return (
    <div style={outer}>
      <span
        style={{
          display: "inline-block",
          background: theme.colors.bg,
          color: theme.colors.text,
          fontSize: theme.caption.fontSize,
          fontWeight: theme.caption.weight,
          borderRadius: theme.caption.radius,
          padding: `${theme.caption.padY}px ${theme.caption.padX}px`,
        }}
      >
        {children}
      </span>
    </div>
  );
};
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/Caption.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/kit/Caption.tsx src/kit/Caption.test.ts
git commit -m "feat: Caption component with safe-area slots and fade-up entrance"
```

---

## Task 11: (merged) — Zoom logic lives in `layout.ts`

There is **no standalone `<Zoom>` component**. The Ken Burns transform is the pure
`zoomStyle` function, added to `src/kit/layout.ts` and unit-tested in Task 5, and applied
by `<Shot>` via its `zoom` prop (Task 9). No work in this task — it exists only to keep
subsequent task numbers stable. Proceed to Task 12.

---

## Task 12: Callout component

**Files:**
- Create: `snill-clips/src/kit/Callout.tsx`
- Test: `snill-clips/src/kit/Callout.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/Callout.test.ts
import { describe, expect, it } from "vitest";
import { calloutStyle } from "./Callout";

describe("calloutStyle", () => {
  it("positions the pill at the target rect with entrance fade", () => {
    const target = { x: 600, y: 240, w: 200, h: 80 };
    const s = calloutStyle(target, 0, 12);
    // anchored to the right edge of the target by default
    expect(s.left).toBe(820); // 600 + 200 + 20 gap
    expect(s.top).toBe(260); // 240 + 80/2 - line offset (40)
    expect(s.opacity).toBe(0);
  });
  it("is visible after entrance", () => {
    const s = calloutStyle({ x: 0, y: 0, w: 10, h: 10 }, 30, 12);
    expect(s.opacity).toBe(1);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/Callout.test.ts`
Expected: FAIL with "Cannot find module './Callout'".

- [ ] **Step 3: Implement `src/kit/Callout.tsx`**

```tsx
import type { CSSProperties } from "react";
import { useCurrentFrame } from "remotion";
import type { Rect } from "./layout";
import { resolveAnchor } from "./layout";
import { fadeUp } from "./animate";
import { useShot } from "./ShotContext";
import { useAnchors } from "./AnchorsContext";
import { snill, type Theme } from "./theme";

const GAP = 20;
const LINE_OFFSET = 40;

export function calloutStyle(target: Rect, frame: number, enterFrames: number): CSSProperties {
  const fade = fadeUp(frame, 0, enterFrames, 12);
  return {
    position: "absolute",
    left: target.x + target.w + GAP,
    top: target.y + target.h / 2 - LINE_OFFSET,
    opacity: fade.opacity,
    transform: `translateY(${fade.translateY}px)`,
  };
}

export const Callout: React.FC<{ at: string; text: string; theme?: Theme }> = ({ at, text, theme = snill }) => {
  const frame = useCurrentFrame();
  const { imageRect } = useShot();
  const anchors = useAnchors();
  const box = anchors[at];
  if (!box) throw new Error(`Unknown anchor "${at}" in <Callout>`);
  const target = resolveAnchor(box, imageRect);
  return (
    <div style={calloutStyle(target, frame, theme.easing.enterFrames)}>
      <span
        style={{
          display: "inline-block",
          background: theme.colors.accent,
          color: "#fff",
          fontSize: theme.callout.fontSize,
          fontWeight: theme.callout.weight,
          borderRadius: 999,
          padding: "10px 22px",
          whiteSpace: "nowrap",
        }}
      >
        {text}
      </span>
    </div>
  );
};
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/Callout.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/kit/Callout.tsx src/kit/Callout.test.ts
git commit -m "feat: Callout pill annotation pinned to anchors"
```

---

## Task 13: Spotlight component

**Files:**
- Create: `snill-clips/src/kit/Spotlight.tsx`
- Test: `snill-clips/src/kit/Spotlight.test.ts`

> `<Spotlight>` dims the whole frame except a rounded box over the target, using the `box-shadow` spread trick: a div sized to the target with a huge spread shadow in the dim color darkens everything outside it.

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/Spotlight.test.ts
import { describe, expect, it } from "vitest";
import { spotlightStyle } from "./Spotlight";

describe("spotlightStyle", () => {
  it("sizes a padded box over the target and dims the outside", () => {
    const target = { x: 100, y: 100, w: 200, h: 100 };
    const s = spotlightStyle(target, 1, "rgba(0,0,0,0.6)", 16, 12);
    expect(s.left).toBe(84); // 100 - 16 pad
    expect(s.width).toBe(232); // 200 + 32 pad
    expect(s.borderRadius).toBe(16);
    expect(String(s.boxShadow)).toContain("9999px");
  });
  it("fades the dim in over the entrance window (alpha scales with progress)", () => {
    const s0 = spotlightStyle({ x: 0, y: 0, w: 10, h: 10 }, 0, "rgba(0,0,0,0.6)", 16, 12);
    expect(s0.opacity).toBe(0);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/Spotlight.test.ts`
Expected: FAIL with "Cannot find module './Spotlight'".

- [ ] **Step 3: Implement `src/kit/Spotlight.tsx`**

```tsx
import type { CSSProperties } from "react";
import { useCurrentFrame } from "remotion";
import type { Rect } from "./layout";
import { resolveAnchor } from "./layout";
import { easeOut, progressIn } from "./animate";
import { useShot } from "./ShotContext";
import { useAnchors } from "./AnchorsContext";
import { snill, type Theme } from "./theme";

export function spotlightStyle(
  target: Rect,
  progress: number,
  dim: string,
  radius: number,
  _enterFrames: number
): CSSProperties {
  const pad = 16;
  return {
    position: "absolute",
    left: target.x - pad,
    top: target.y - pad,
    width: target.w + pad * 2,
    height: target.h + pad * 2,
    borderRadius: radius,
    boxShadow: `0 0 0 9999px ${dim}`,
    opacity: easeOut(progress),
  };
}

export const Spotlight: React.FC<{ on: string; radius?: number; theme?: Theme }> = ({
  on,
  radius = 16,
  theme = snill,
}) => {
  const frame = useCurrentFrame();
  const { imageRect } = useShot();
  const anchors = useAnchors();
  const box = anchors[on];
  if (!box) throw new Error(`Unknown anchor "${on}" in <Spotlight>`);
  const target = resolveAnchor(box, imageRect);
  const progress = progressIn(frame, 0, theme.easing.enterFrames);
  return <div style={spotlightStyle(target, progress, theme.colors.dim, radius, theme.easing.enterFrames)} />;
};
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/Spotlight.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/kit/Spotlight.tsx src/kit/Spotlight.test.ts
git commit -m "feat: Spotlight component dimming everything outside a target box"
```

---

## Task 14: Cursor component

**Files:**
- Create: `snill-clips/src/kit/Cursor.tsx`
- Test: `snill-clips/src/kit/Cursor.test.ts`

> `<Cursor>` moves a fake pointer from one anchor to another and optionally "clicks" (a brief scale-down pulse). Pure `cursorState` returns position (in %) and click scale for a given frame.

- [ ] **Step 1: Write the failing test**

```ts
// src/kit/Cursor.test.ts
import { describe, expect, it } from "vitest";
import { cursorState } from "./Cursor";

describe("cursorState", () => {
  const from = { xPct: 10, yPct: 10 };
  const to = { xPct: 50, yPct: 50 };

  it("sits at the start before the move", () => {
    const s = cursorState(0, from, to, 0, 30, 30);
    expect(s.xPct).toBe(10);
    expect(s.clickScale).toBe(1);
  });
  it("reaches the destination after the move window", () => {
    const s = cursorState(30, from, to, 0, 30, 30);
    expect(s.xPct).toBeCloseTo(50);
    expect(s.yPct).toBeCloseTo(50);
  });
  it("pulses (scale < 1) at the click frame", () => {
    const s = cursorState(33, from, to, 0, 30, 30);
    expect(s.clickScale).toBeLessThan(1);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run src/kit/Cursor.test.ts`
Expected: FAIL with "Cannot find module './Cursor'".

- [ ] **Step 3: Implement `src/kit/Cursor.tsx`**

```tsx
import { useCurrentFrame, useVideoConfig } from "remotion";
import { anchorCenterPct } from "./layout";
import { easeOut, progressIn } from "./animate";
import { useShot } from "./ShotContext";
import { useAnchors } from "./AnchorsContext";

interface Pct { xPct: number; yPct: number }
export interface CursorState { xPct: number; yPct: number; clickScale: number }

export function cursorState(
  frame: number,
  from: Pct,
  to: Pct,
  startFrame: number,
  moveFrames: number,
  clickFrame: number | null
): CursorState {
  const t = easeOut(progressIn(frame, startFrame, moveFrames));
  const xPct = from.xPct + (to.xPct - from.xPct) * t;
  const yPct = from.yPct + (to.yPct - from.yPct) * t;
  let clickScale = 1;
  if (clickFrame !== null) {
    const since = frame - clickFrame;
    if (since >= 0 && since < 6) clickScale = 0.8 + 0.2 * (since / 6);
  }
  return { xPct, yPct, clickScale };
}

export const Cursor: React.FC<{
  from: string;
  to: string;
  startFrame?: number;
  moveFrames?: number;
  clickFrame?: number | null;
}> = ({ from, to, startFrame = 0, moveFrames = 30, clickFrame = null }) => {
  const frame = useCurrentFrame();
  const { width, height } = useVideoConfig();
  const { imageRect } = useShot();
  const anchors = useAnchors();
  if (!anchors[from] || !anchors[to]) throw new Error(`Unknown anchor in <Cursor> (${from} -> ${to})`);
  const a = anchorCenterPct(anchors[from], imageRect, width, height);
  const b = anchorCenterPct(anchors[to], imageRect, width, height);
  const s = cursorState(frame, a, b, startFrame, moveFrames, clickFrame);
  return (
    <div
      style={{
        position: "absolute",
        left: `${s.xPct}%`,
        top: `${s.yPct}%`,
        width: 36,
        height: 36,
        transform: `translate(-4px, -2px) scale(${s.clickScale})`,
        // simple arrow cursor drawn with CSS borders
        background: "transparent",
        filter: "drop-shadow(0 2px 4px rgba(0,0,0,0.4))",
      }}
    >
      <svg width="36" height="36" viewBox="0 0 24 24" fill="white" stroke="black" strokeWidth="1">
        <path d="M3 2 L3 18 L7 14 L10 21 L13 20 L10 13 L16 13 Z" />
      </svg>
    </div>
  );
};
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run src/kit/Cursor.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/kit/Cursor.tsx src/kit/Cursor.test.ts
git commit -m "feat: Cursor component with move + click pulse"
```

---

## Task 15: Card + Transitions + kit barrel export

**Files:**
- Create: `snill-clips/src/kit/Card.tsx`
- Create: `snill-clips/src/kit/Scenes.tsx`
- Create: `snill-clips/src/kit/index.ts`

> `<Card>` is a full-screen branded title card. `<Scenes>` wraps a sequence of `<Card>`/`<Shot>` children in a `@remotion/transitions` `TransitionSeries`, applying a fade between them and assigning each child its duration — this is what rebases each shot's `useCurrentFrame()` to 0.

- [ ] **Step 1: Implement `src/kit/Card.tsx`**

```tsx
import type { ReactNode } from "react";
import { AbsoluteFill, useCurrentFrame } from "remotion";
import { fadeUp } from "./animate";
import { snill, type Theme } from "./theme";

export const Card: React.FC<{ children: ReactNode; variant?: "intro" | "outro"; theme?: Theme }> = ({
  children,
  variant = "intro",
  theme = snill,
}) => {
  const frame = useCurrentFrame();
  const fade = fadeUp(frame, 0, theme.easing.enterFrames, 30);
  return (
    <AbsoluteFill
      style={{
        backgroundColor: theme.colors.bg,
        color: theme.colors.text,
        alignItems: "center",
        justifyContent: "center",
        textAlign: "center",
        padding: 100,
      }}
    >
      <div
        style={{
          opacity: fade.opacity,
          transform: `translateY(${fade.translateY}px)`,
          fontSize: variant === "outro" ? 80 : 72,
          fontWeight: 700,
          color: variant === "outro" ? theme.colors.accent : theme.colors.text,
        }}
      >
        {children}
      </div>
    </AbsoluteFill>
  );
};
```

- [ ] **Step 2: Implement `src/kit/Scenes.tsx`**

```tsx
import { Children, type ReactElement } from "react";
import { useVideoConfig } from "remotion";
import { TransitionSeries, linearTiming } from "@remotion/transitions";
import { fade } from "@remotion/transitions/fade";

/**
 * Wrap a list of scenes (each a <Card> or <Shot> with a `dur` prop in seconds)
 * in a TransitionSeries with a fade between them. Rebases each scene to frame 0.
 */
export const Scenes: React.FC<{ children: ReactElement<{ dur: number }>[]; transitionFrames?: number }> = ({
  children,
  transitionFrames = 12,
}) => {
  const { fps } = useVideoConfig();
  const scenes = Children.toArray(children) as ReactElement<{ dur: number }>[];
  return (
    <TransitionSeries>
      {scenes.flatMap((scene, i) => {
        const frames = Math.round(scene.props.dur * fps);
        const nodes = [
          <TransitionSeries.Sequence key={`s${i}`} durationInFrames={frames}>
            {scene}
          </TransitionSeries.Sequence>,
        ];
        if (i < scenes.length - 1) {
          nodes.push(
            <TransitionSeries.Transition
              key={`t${i}`}
              timing={linearTiming({ durationInFrames: transitionFrames })}
              presentation={fade()}
            />
          );
        }
        return nodes;
      })}
    </TransitionSeries>
  );
};
```

> `<Card>` must accept a `dur` prop for `Scenes` to read it; add `dur?: number` to Card's props (it is read by `Scenes`, not used inside Card). Update Card's prop type to `{ children: ReactNode; variant?: ...; theme?: Theme; dur?: number }`.

- [ ] **Step 3: Update `src/kit/Card.tsx` props to include `dur`**

Change the Card component's prop type to:

```tsx
export const Card: React.FC<{ children: ReactNode; variant?: "intro" | "outro"; theme?: Theme; dur?: number }> = ({
  children,
  variant = "intro",
  theme = snill,
}) => {
```

(The `dur` is intentionally not destructured/used inside Card — `Scenes` reads it off `props`.)

- [ ] **Step 4: Implement `src/kit/index.ts` (barrel)**

```ts
export { Clip } from "./Clip";
export { Shot } from "./Shot";
export { Card } from "./Card";
export { Scenes } from "./Scenes";
export { Caption } from "./Caption";
export { Callout } from "./Callout";
export { Spotlight } from "./Spotlight";
export { Cursor } from "./Cursor";
export { snill } from "./theme";
export type { Theme } from "./theme";
export type { Box } from "./layout";
// Ken Burns zoom is applied via <Shot zoom={{ to, scale }}> — no <Zoom> element.
```

- [ ] **Step 5: Typecheck + commit**

Run: `npx tsc --noEmit`
Expected: PASS.

```bash
git add src/kit/Card.tsx src/kit/Scenes.tsx src/kit/index.ts
git commit -m "feat: Card title cards, Scenes transition wrapper, kit barrel export"
```

---

## Task 16: The example clip (expense-intro)

**Files:**
- Create: `snill-clips/src/clips/expense-intro/anchors.ts`
- Create: `snill-clips/src/clips/expense-intro/clip.tsx`
- Create: `snill-clips/src/clips/expense-intro/assets/gallery.png` (placeholder screenshot)
- Create: `snill-clips/src/clips/expense-intro/assets/app-created.png` (placeholder screenshot)
- Create: `snill-clips/src/asset.d.ts` (PNG import typing)

- [ ] **Step 1: Add PNG module typing — `src/asset.d.ts`**

```ts
declare module "*.png" {
  const src: string;
  export default src;
}
```

- [ ] **Step 2: Add placeholder screenshots**

Generate two solid-color placeholder PNGs (replace with real captures later). Run from `snill-clips/`:

```bash
mkdir -p src/clips/expense-intro/assets
# 1600x900 dark gray placeholder, and 1600x900 blue placeholder
node -e "const fs=require('fs');const z=require('zlib');function png(w,h,r,g,b){const raw=Buffer.alloc((w*3+1)*h);for(let y=0;y<h;y++){raw[y*(w*3+1)]=0;for(let x=0;x<w;x++){const o=y*(w*3+1)+1+x*3;raw[o]=r;raw[o+1]=g;raw[o+2]=b;}}const idat=z.deflateSync(raw);function chunk(t,d){const len=Buffer.alloc(4);len.writeUInt32BE(d.length);const tb=Buffer.from(t);const crc=Buffer.alloc(4);const c=require('zlib');let table=[];for(let n=0;n<256;n++){let cc=n;for(let k=0;k<8;k++)cc=cc&1?0xedb88320^(cc>>>1):cc>>>1;table[n]=cc;}let crcv=0xffffffff;const buf=Buffer.concat([tb,d]);for(let i=0;i<buf.length;i++)crcv=table[(crcv^buf[i])&0xff]^(crcv>>>8);crc.writeUInt32BE((crcv^0xffffffff)>>>0);return Buffer.concat([len,tb,d,crc]);}const sig=Buffer.from([137,80,78,71,13,10,26,10]);const ihdr=Buffer.alloc(13);ihdr.writeUInt32BE(w,0);ihdr.writeUInt32BE(h,4);ihdr[8]=8;ihdr[9]=2;return Buffer.concat([sig,chunk('IHDR',ihdr),chunk('IDAT',idat),chunk('IEND',Buffer.alloc(0))]);}fs.writeFileSync('src/clips/expense-intro/assets/gallery.png',png(1600,900,30,30,40));fs.writeFileSync('src/clips/expense-intro/assets/app-created.png',png(1600,900,40,80,160));"
```

Expected: two PNG files created. Verify: `ls -la src/clips/expense-intro/assets`.

- [ ] **Step 3: Create `src/clips/expense-intro/anchors.ts`**

```ts
import type { Box } from "../../kit";

const anchors: Record<string, Box> = {
  "expense-card": { x: 0.32, y: 0.4, w: 0.2, h: 0.15 },
  "dashboard-tab": { x: 0.78, y: 0.12, w: 0.16, h: 0.08 },
  "new-button": { x: 0.85, y: 0.85, w: 0.1, h: 0.08 },
};

export default anchors;
```

- [ ] **Step 4: Create `src/clips/expense-intro/clip.tsx`**

```tsx
import { Caption, Callout, Card, Clip, Scenes, Shot, Spotlight, snill } from "../../kit";
import anchors from "./anchors";
import gallery from "./assets/gallery.png";
import appCreated from "./assets/app-created.png";

// Sum of scene durs below (1.5 + 2.5 + 3 + 1.5) = 8.5s
export const config = { durationInSeconds: 8.5 };

export default function ExpenseIntro() {
  return (
    <Clip theme={snill} anchors={anchors} music="upbeat-1.mp3">
      <Scenes>
        <Card dur={1.5}>Build an expense app in seconds</Card>

        <Shot src={gallery} dur={2.5} zoom={{ to: "expense-card", scale: 1.3 }}>
          <Spotlight on="expense-card" />
          <Caption pos="bottom">Start from a template</Caption>
        </Shot>

        <Shot src={appCreated} dur={3}>
          <Caption pos="bottom">Your app is ready</Caption>
          <Callout at="dashboard-tab" text="Live in seconds" />
        </Shot>

        <Card dur={1.5} variant="outro">Try snill.ai →</Card>
      </Scenes>
    </Clip>
  );
}
```

> Music file `public/music/upbeat-1.mp3` is added in Task 17. Until then, rendering still works only if the file exists; for this task, either add a silent placeholder mp3 or temporarily omit the `music` prop. Add the placeholder now: `mkdir -p public/music && : > public/music/upbeat-1.mp3` is NOT a valid mp3 — instead download/commit a real royalty-free track, or omit `music` until Task 17.

- [ ] **Step 5: Temporarily omit music to keep this task self-contained**

Edit `clip.tsx` to remove the `music="upbeat-1.mp3"` prop for now (re-added in Task 17 once a real track is committed).

- [ ] **Step 6: Typecheck**

Run: `npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 7: Verify the composition list loads in Studio**

Run: `npm run studio`
Expected: Remotion Studio opens; left panel lists `expense-intro-1x1`, `expense-intro-9x16`, `expense-intro-16x9`. Scrub each — placeholder rectangles with captions/zoom/spotlight/callout/cards animate. Close Studio (Ctrl-C).

- [ ] **Step 8: Commit**

```bash
git add src/clips/expense-intro src/asset.d.ts
git commit -m "feat: expense-intro example clip with anchors and placeholder assets"
```

---

## Task 17: Background music

**Files:**
- Create: `snill-clips/public/music/upbeat-1.mp3` (real royalty-free track)
- Modify: `snill-clips/src/clips/expense-intro/clip.tsx`

- [ ] **Step 1: Add a royalty-free track**

Place a licensed/royalty-free MP3 at `public/music/upbeat-1.mp3`. (Source from a CC0/royalty-free library; record its license in `public/music/CREDITS.md`.)

- [ ] **Step 2: Create `public/music/CREDITS.md`**

```md
# Music credits

- `upbeat-1.mp3` — <track name> by <artist>, <license + source URL>.
```

- [ ] **Step 3: Re-add the music prop in `clip.tsx`**

In `src/clips/expense-intro/clip.tsx`, change the `<Clip>` opening tag back to:

```tsx
<Clip theme={snill} anchors={anchors} music="upbeat-1.mp3">
```

- [ ] **Step 4: Verify in Studio**

Run: `npm run studio`
Expected: clip plays with audible background music; audio track shows in the timeline.

- [ ] **Step 5: Commit**

```bash
git add public/music/upbeat-1.mp3 public/music/CREDITS.md src/clips/expense-intro/clip.tsx
git commit -m "feat: background music track for expense-intro"
```

---

## Task 18: Render CLI

**Files:**
- Create: `snill-clips/render.ts`

> Bundles the project once, then renders requested `clip@size` compositions to `out/`. Supports `--size <id>` (render only one size) and `--gif` (also emit a gif). Uses `@remotion/bundler` `bundle` + `@remotion/renderer` `selectComposition`/`renderMedia`.

- [ ] **Step 1: Implement `render.ts`**

```ts
import path from "node:path";
import { bundle } from "@remotion/bundler";
import { renderMedia, selectComposition } from "@remotion/renderer";
import { SIZE_LIST, type SizeId } from "./src/sizes";

async function main() {
  const args = process.argv.slice(2);
  const clipName = args.find((a) => !a.startsWith("--"));
  if (!clipName) {
    console.error("Usage: npm run render <clip-name> [--size 9x16] [--gif]");
    process.exit(1);
  }
  const sizeFlag = args.includes("--size") ? (args[args.indexOf("--size") + 1] as SizeId) : null;
  const wantGif = args.includes("--gif");
  const sizes = sizeFlag ? SIZE_LIST.filter((s) => s.id === sizeFlag) : SIZE_LIST;

  console.log("Bundling…");
  const serveUrl = await bundle({ entryPoint: path.join(process.cwd(), "src/index.ts") });

  for (const size of sizes) {
    // Separator is "-" not "@": Remotion's validateCompositionId rejects "@".
    // Must match the id format in src/registry.ts (`${name}-${size.id}`).
    const id = `${clipName}-${size.id}`;
    const composition = await selectComposition({ serveUrl, id });

    const mp4 = path.join("out", `${clipName}.${size.id}.mp4`);
    console.log(`Rendering ${id} → ${mp4}`);
    await renderMedia({ composition, serveUrl, codec: "h264", outputLocation: mp4 });

    if (wantGif) {
      const gif = path.join("out", `${clipName}.${size.id}.gif`);
      console.log(`Rendering ${id} → ${gif}`);
      await renderMedia({ composition, serveUrl, codec: "gif", outputLocation: gif });
    }
  }
  console.log("Done.");
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

- [ ] **Step 2: Render the example clip end-to-end**

Run: `npm run render -- expense-intro`
Expected: `out/expense-intro.1x1.mp4`, `out/expense-intro.9x16.mp4`, `out/expense-intro.16x9.mp4` created; each plays the full ~8.5s animation with music. Verify: `ls -la out`.

- [ ] **Step 3: Render a single size + gif (flag check)**

Run: `npm run render -- expense-intro --size 9x16 --gif`
Expected: `out/expense-intro.9x16.mp4` and `out/expense-intro.9x16.gif` produced.

- [ ] **Step 4: Commit**

```bash
git add render.ts
git commit -m "feat: render CLI for multi-size mp4/gif output"
```

---

## Task 19: Frame-snapshot regression tests

**Files:**
- Create: `snill-clips/tests/snapshot/snapshot.test.ts`

> Renders a few key frames of `expense-intro-1x1` via `renderStill` and asserts the output PNG is byte-stable against a committed reference. Catches kit/layout regressions. First run generates references; subsequent runs compare.

- [ ] **Step 1: Implement `tests/snapshot/snapshot.test.ts`**

```ts
import path from "node:path";
import fs from "node:fs";
import { describe, expect, it, beforeAll } from "vitest";
import { bundle } from "@remotion/bundler";
import { renderStill, selectComposition } from "@remotion/renderer";

const OUT = path.join(__dirname, "__output__");
const REF = path.join(__dirname, "__reference__");
const FRAMES = [0, 60, 150];

let serveUrl: string;

beforeAll(async () => {
  fs.mkdirSync(OUT, { recursive: true });
  fs.mkdirSync(REF, { recursive: true });
  serveUrl = await bundle({ entryPoint: path.join(process.cwd(), "src/index.ts") });
}, 120_000);

describe("expense-intro-1x1 frames", () => {
  for (const frame of FRAMES) {
    it(`frame ${frame} matches reference`, async () => {
      const composition = await selectComposition({ serveUrl, id: "expense-intro-1x1" });
      const outPath = path.join(OUT, `f${frame}.png`);
      await renderStill({ composition, serveUrl, frame, output: outPath, imageFormat: "png" });

      const refPath = path.join(REF, `f${frame}.png`);
      if (!fs.existsSync(refPath)) {
        fs.copyFileSync(outPath, refPath);
        console.warn(`Wrote new reference for frame ${frame}; re-run to compare.`);
        return;
      }
      expect(fs.readFileSync(outPath).equals(fs.readFileSync(refPath))).toBe(true);
    }, 120_000);
  }
});
```

- [ ] **Step 2: Generate references (first run)**

Run: `npx vitest run tests/snapshot/snapshot.test.ts`
Expected: PASS (references written on first run).

- [ ] **Step 3: Confirm stability (second run)**

Run: `npx vitest run tests/snapshot/snapshot.test.ts`
Expected: PASS (frames match committed references).

- [ ] **Step 4: Ignore generated output, commit references**

Add to `.gitignore`: `tests/snapshot/__output__/`

```bash
git add .gitignore tests/snapshot/snapshot.test.ts tests/snapshot/__reference__
git commit -m "test: frame-snapshot regression tests for expense-intro"
```

---

## Task 20: Full suite + README

**Files:**
- Create: `snill-clips/README.md`

- [ ] **Step 1: Run the whole unit suite**

Run: `npm test`
Expected: PASS (all kit + sizes + registry unit tests green).

- [ ] **Step 2: Typecheck the whole project**

Run: `npm run typecheck`
Expected: PASS.

- [ ] **Step 3: Write `README.md`**

````md
# snill-clips

Script-driven marketing animations for snill.ai. Each clip is a React "script"
that composes pre-captured screenshots with zooms, captions, callouts, spotlights,
and a cursor, rendered to MP4 in three aspect ratios from one source.

## Quick start

```bash
npm install
npm run studio          # preview & tune clips live
npm run render -- expense-intro          # render all 3 sizes to out/
npm run render -- expense-intro --size 9x16 --gif
```

## Authoring a clip

1. Create `src/clips/<name>/`.
2. Add `assets/` screenshots, `anchors.ts` (named image-relative boxes), and
   `clip.tsx` (`export default` component + `export const config = { durationInSeconds }`).
3. It auto-appears in Studio as `<name>@1x1`, `<name>@9x16`, `<name>@16x9`.

## Kit components

`<Clip>` `<Scenes>` `<Card>` `<Shot>` `<Caption>` `<Zoom>` `<Callout>` `<Spotlight>` `<Cursor>`.
All positions are image-relative (0–1) so one script renders at every aspect ratio.

## Licensing

Built on Remotion, which is free for individuals, non-profits, and for-profit
companies with ≤3 employees. **4+ employees require a paid Company License** —
see https://www.remotion.dev/docs/license before scaling the team.

## Tests

```bash
npm test            # unit tests (pure logic)
npm run typecheck
```
````

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: snill-clips README"
```

---

## Self-review notes (coverage map)

- Spec "standalone repo + CLI" → Tasks 1, 18.
- Spec "author once → 3 sizes" → Tasks 3 (registry), 5 (layout math), 7 (useSize); proven by Task 16/18 rendering all three.
- Spec kit vocabulary (Clip, Card, Shot, Caption, Zoom, Callout, Spotlight, Cursor, Transition, theme) → Tasks 4 (theme), 5 (zoomStyle), 8 (Clip), 9 (Shot + zoom prop), 10 (Caption), 12 (Callout), 13 (Spotlight), 14 (Cursor), 15 (Card/Scenes/Transition). Zoom is a `<Shot zoom>` prop, not a `<Zoom>` element.
- Spec "named anchors" → Task 7 (ShotContext/useAnchor), 16 (anchors.ts).
- Spec "text overlays only, optional music" → Tasks 10 (Caption), 17 (music). No TTS anywhere (correct).
- Spec "pre-captured screenshots" → Task 9 (Shot/Img), 16 (assets).
- Spec "Remotion + free-license caveat" → header, Task 1 license note, Task 20 README.
- Spec testing (typecheck + frame snapshots) → Tasks 6/10/11/… unit tests, 19 snapshots, 20 suite.
- Spec non-goals (no CI, no TTS, no live-drive, no anchor-picker) → none implemented (correct).
- Open items (safe-area margins, music handling, transition lib, brand tokens) → concrete defaults chosen in Tasks 4/5/15/17; flagged in spec for later tuning.
