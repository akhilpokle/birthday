# Making the project lighter

Two halves: **§1 is done** — code changes already applied. **§2 is your list** —
asset work that needs tools or decisions I shouldn't make alone.

Current state: **8.0MB** repo, of which **7.9MB is `Assets/`**. The code is 1,258
lines across three files and is not the problem.

---

## 1. Code — applied

Measured problems, not speculative tidying.

| Fix | Was | Now |
| --- | --- | --- |
| `clientHeight` read inside the particle loop | up to **1400 forced layouts per frame** | hoisted; 1 read per frame |
| `save()`/`restore()` per particle | 2 canvas state-stack ops per particle | single `setTransform`, dpr folded in |
| `fillStyle`/`globalAlpha` per particle | rewritten every particle | only on change (a burst is one palette) |
| `popWave` read/write interleaving | 26 forced reflows per click | two passes: read all rects, then write |
| `balloonsRemaining()` | `querySelectorAll` + scan per pop → O(n²) per wave | O(1) counter |
| `later()` bookkeeping | `indexOf` + `splice` per timer against a ~600-long array | object keyed by timer id |
| `resize` handler | full rebuild of ~100 `<img>` + 2 canvas reallocations **per event** | coalesced to one per frame via rAF |
| Finale canvas | ~16MB allocated at init for something used once | lazy — stays 300×150 until the finale runs |
| Frame 2–5 preload | 20 images (~5MB) fetched during load, competing with visible balloons | deferred to `requestIdleCallback`, one per idle slot |
| `CONFIG` lookups in `fill()` | re-read per cell, ~100× | hoisted |
| Balloon `<img>` decode | synchronous, blocking first paint | `decoding="async"` |

Also added `desynchronized: true` on both canvas contexts, which lets the
compositor skip a sync point.

**Behaviour is unchanged** — verified after the fact: 96 balloons, wave delays
still distance-proportional (26 distinct values, 0–1338ms), finale still fires at
screen centre once the last balloon lands, lazy canvas allocates on demand,
`reset()` / `setConfig()` / `setPointer()` / double-init guard all intact, zero
console errors.

---

## 2. Assets — your list

**The single biggest fact: every balloon PNG is 800×800 and renders at 208px.**
That is a 3.8× linear downscale, so ~93% of every decoded pixel is thrown away.
The card is 2880×1436 rendering at 702px — a 4.1× downscale.

Ordered by payoff.

### 2.1 Resize the sources — biggest single win

- [ ] **Balloon frames 800×800 → 416×416** (2× of the 208px render, for retina).
      25 files, currently **6.2MB**. Expect **~1.6MB**, and roughly **6× less
      decode work and GPU memory** — which matters most at `frameMs: 30`, where
      five frames swap in 120ms.
- [ ] **`card.png` 2880×1436 → 1404×700.** Currently **1.1MB** → expect ~280KB.
- [ ] Do *not* resize by hand — ship the script, per the playbook's rule about
      generated artefacts. `sharp` or ImageMagick, one loop, committed to the repo.

### 2.2 Compress

- [ ] **Run every PNG through `pngquant` + `oxipng`.** These are flat-shaded
      renders with soft gradients — typically **50–70%** off with no visible
      loss. Stacks with 2.1: combined, expect `Assets/` at **~1.5MB from 7.9MB**.
- [ ] **Consider WebP** (`<picture>` with PNG fallback). Usually another 25–35%
      on top. Check the Liferay theme's browser floor first.

### 2.3 Delete what isn't used

- [ ] **600KB of superseded assets are committed and referenced by nothing** —
      `blue.png`, `green.png`, `red.png`, `yellow.png`, and the five `* burst.png`
      files. They predate the 5-frame sequences. Confirm, then delete.
- [ ] `crosshair.svg` (1KB) is only used by the disabled crosshair mode. Keep
      while that mode is still switchable.
- [ ] Delete `_local/` from disk once you're sure `tinified (1)/` is backed up
      elsewhere. It is gitignored, so it costs nothing in the repo — this is
      about your disk only.

### 2.4 Load less, later

- [ ] **Frames 2–5 are still 20 files even when idle-loaded.** A user who pops
      one colour downloads all five. Fetch a colour's frames on first
      `pointerenter` of that colour instead — cuts the common case to ~4 files.
- [ ] **Sprite-sheet each colour's 5 frames into one image** and swap
      `object-position` rather than `src`. 5 requests instead of 25, one decode
      per colour, and no chance of a mid-animation cache miss.
- [ ] **Drop the background layer entirely** if this becomes a true overlay on the
      live intranet — `backdrop-filter` would blur the real page and
      `background.svg` (8KB) disappears. This is already the biggest open question
      in `BACKLOG.md` §1.

### 2.5 Delivery

- [ ] **Confirm the Liferay fragment JS field has no size limit** the inlined
      script would breach. It is currently ~30KB of JS — almost certainly fine,
      but the conversion playbook flags this.
- [ ] **Check the theme serves Public Sans.** If it does, nothing to do. If not,
      self-host a subset — Latin-only, weights 400 and 600 only, WOFF2. A full
      Public Sans family is ~200KB; a subset is ~30KB.
- [ ] **Serve assets with long-lived cache headers** via Liferay's resource
      pipeline; the takeover is once-per-user-per-year, so a cold cache is the
      normal case and raw size matters more than usual here.

### 2.6 Not worth doing

- Minifying the fragment. It's ~30KB of JS that the integrator has to read and
  maintain; the playbook explicitly says keep it hand-editable. Saves a few KB
  against a 7.9MB asset payload.
- Removing comments. Same reasoning — the comments are the handover.

---

## 3. Progress — compression pass, 2026-08-21

The 25 balloon frames were compressed. Measured:

| | Before | After | Saved |
| --- | --- | --- | --- |
| Frames 1 (always loaded) | 1.2MB | **472KB** | 61% |
| Frames 2–5 (idle preload) | 5.0MB | **1.7MB** | 66% |
| Balloon frames total | 6.2MB | **2.2MB** | **65%** |
| `card.png` | 1.1MB | 1.1MB | — *not compressed* |
| Unused legacy assets | 600KB | 600KB | — *not compressed* |
| **`Assets/` total** | **7.9MB** | **3.9MB** | **51%** |

Quality is intact — checked rather than assumed: alpha is still smooth (2–6%
partial-alpha pixels per frame, not flattened to 1-bit), edge pixels are still
darker than the core so no white matte was introduced, and 0 of 98 images fail
to decode.

**Dimensions are unchanged — 800×800.** This was compression only, not resizing,
which matters: compression cuts *download*, resizing cuts *decode and GPU
memory*. A 800×800 PNG still decodes to 2.56MB of bitmap however well the file
compresses, so §2.1 is entirely still on the table and remains the bigger win for
runtime smoothness at `frameMs: 30`.

**Still outstanding, in order:**

1. `card.png` at 1.1MB is now the **single largest file — 28% of `Assets/`** and
   the only uncompressed one. Compressing alone should get it near 300KB.
2. Resizing (§2.1) — untouched, and now worth proportionally more.
3. 600KB of unused legacy assets (§2.3) — still committed.

Doing those three takes `Assets/` from **3.9MB to roughly 800KB**.

### Palette note

The confetti palettes in `template.html` are hardcoded hexes sampled from the
*original* files. Re-checked against the compressed art: 13 of 15 match within an
RGB distance of 4 (imperceptible). Two differ — Blue's darkest (54) and Yellow's
darkest (26) — but both were weak samples to begin with: Yellow's dark bucket was
empty in the original, so that value was invented rather than sampled, and Blue's
came from only 373 pixels. Not compression damage, and invisible on 5px confetti
in flight. No action needed unless the palettes are ever re-derived.
