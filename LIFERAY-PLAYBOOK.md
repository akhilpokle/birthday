# Playbook — 2027 birthday takeover

Project-specific build guide. Companion to `LIFERAY-BUILD-PLAYBOOK.md` (greenfield)
and `LIFERAY-CONVERSION-PLAYBOOK.md` (React/Figma → Liferay), both of which live in
other project folders.

**Where this deviates from them:** the build playbook's §2 tells you to lay down a
"day-one skeleton" containing a screen-reader helper, a double-init guard, a
reduced-motion helper and a public API before writing any feature. On this project
that produced ~60 lines of scaffolding guarding behaviour that didn't exist —
an init guard on a script with nothing to initialise, a `dur()` helper with no
animation to time, a `destroy()` for a component nobody could open or close.

This playbook replaces that rule. Everything else in both companions still stands.

---

## 1. The rule — add it when the feature exists, not before

Write the smallest thing that makes the current step work. When a step introduces a
behaviour, add that behaviour's machinery **in the same commit as the behaviour**.

The build playbook's justification is that this stuff is "painful to add later."
That's true of *scoping decisions* — a prefix retrofitted across 800 lines is a bad
day. It is **not** true of an init guard or a `destroy()` method. Those are ten
lines, they're mechanical, and adding one to a file that has actual state is easier
than maintaining an empty one that nobody has tested.

Untested scaffolding is not insurance. It's just code that looks like insurance.

---

## 2. What triggers what

Don't add the left column until the right column is true.

| Add this | Only once… |
| -------- | ---------- |
| `<script>` block at all | there is behaviour to run |
| double-init guard (`data-*-init`) | the script mutates state, attaches listeners, or injects nodes |
| `destroy()` / public API | the host needs to open, close, or tear it down |
| `prefers-reduced-motion` handling | something actually animates |
| `.sr-only` helper | there is visually-hidden text to hold |
| `box-sizing` / `p` / `*` resets | an element exists that the reset changes |
| focus trap, Esc handler, `role="dialog"` | it becomes a dismissible overlay |
| `PANELS`-style content array | there is more than one of the thing |

The right-hand column is a fact about the file, not a plan. "We'll probably animate
it later" does not trigger the row.

---

## 3. The exception — host defence is always day one

This is the line that keeps §1 from becoming "skip the hard parts."

Two categories are **not** scaffolding and go in from the first line:

**Namespacing.** Every class, `id`, SVG `id` and global gets the `bday_` prefix.
Retrofitting this is the genuinely expensive one. See conversion playbook §1.

**Scoped resets that defend against the host theme.** These look like boilerplate
and aren't. Concrete case from this build: the harness styles bare
`img { border: 4px solid magenta }`, exactly as the real intranet might. Our
background image inherited it, gained 8px, and stopped covering its container.
The fix is one line — but only a *scoped* line:

```css
.bday_root img { display: block; border: 0; margin: 0; }
```

The test for whether a reset is earned: **does the host have a rule that would reach
this element?** If yes, it's defence. If it's tidying markup you control, it's
scaffolding — leave it out.

Accessibility follows the same split. Semantic elements (`<button>`, not
`<div onclick>`) are day one, because retrofitting them means rewriting the markup.
ARIA for a widget pattern arrives with the widget.

---

## 4. Current state

`template.html` — the single source of truth. Steps 1–2 complete:

```
.bday_root      position: fixed; inset: 0; z-index: 9999
                fixed+inset so it covers the viewport wherever it's mounted,
                rather than inheriting a host container's width
.bday_overlay   linear-gradient #172733 → #457699 (85% alpha),
                backdrop-filter: blur(16px) + -webkit- prefix
.bday_close     32x32 dismiss button, top right, z-index 305

.bday_card      Assets/card.png — the final message, centred, above the overlay
                and behind the balloons, so popping them uncovers it. No reveal
                logic; the layering does the work.
```

**It sits on top of the live intranet — decided 2026-08-31.** This was the
project's biggest open question and it is now closed. There is no background
layer at all: the real page shows through and `.bday_overlay`'s
`backdrop-filter` blurs it. `Assets/background.svg` was a placeholder for a
screenshot and has been deleted. Adding an `<img>` back would cover the very
page this is meant to blur.

That makes the `position: fixed` containing-block risk sharper rather than
softer: if a theme ancestor carries a `transform`/`filter`/`perspective`, the
takeover boxes itself inside that ancestor and blurs only that region. Not
fixed in code — a marked comment in the script explains the ~1-line fix, since
reparenting a fragment's own DOM should be a deliberate call by the integrator.

**Dismissal — added 2026-08-31.** A 32×32 close button, top right, plus Esc.
`close()` hides the root and cancels pending timers rather than tearing down,
so `open()` can bring it back without rebuilding ~45 nodes; `destroy()` is
still the real teardown. Focus moves to the button on init and returns to the
previously focused element on close — with one focusable control in a
full-screen overlay, that is the whole keyboard story. Note confetti's canvas
lives on `<body>`, outside the root, so anything mid-flight paints over a
just-closed takeover for a moment before clearing itself.

**Sizing: count and size are both derived, not configured.** The only knob is
`balloonScale` — the balloon's size as a multiple of its grid cell. Everything
else falls out of the viewport:

1. `grid()` picks the rows/cols that keep cells closest to square, clamped into
   `MIN_BALLOONS`–`MAX_BALLOONS` (40–50).
2. `balloonPxFor()` derives the pixel size as `balloonScale × max(cellW, cellH)`.
   The larger dimension, so neighbours still overlap on odd aspect ratios; `min`
   would open gaps on a very wide screen.
3. Above `MAX_BALLOON_PX` (360) the grid grows instead of the balloons, up to
   `HARD_MAX_BALLOONS` (110).

This replaced a fixed `balloonSize`, under which the balloon-to-cell ratio
drifted with the viewport — 1.78× the cell at 1440×900 but 2.19× at 1024×768 —
and count never moved, so a 27" monitor got the same composition simply zoomed:
320px balloons at 1440×900 became **513px** at 2560×1440, nearly three-quarters
the width of the message card.

| Viewport | Grid | Count | Balloon |
| --- | --- | --- | --- |
| 1024×768 | 8×6 | 48 | 228px |
| 1440×900 | 9×5 | 45 | 320px |
| 1920×1080 | 10×6 | 60 | 342px |
| 2560×1440 | 13×8 | 104 | 351px |
| 3840×2160 | 13×8 | 104 | 526px *(ceiling)* |

Ratio holds at ~1.78 up to the cap; zero coverage gaps at every size. At 4K the
cap cannot be met within 110 balloons — that would need ~209 — so size drifts up
there by design.

**The growth loop must balance the axes.** It grows whichever of cols/rows
currently has the *larger* cell. Two sequential loops (width fully, then height)
look equivalent and are not: at 3840×2160 the width loop consumed the whole
budget at 19 columns, leaving rows at 5, so cells were 202 wide by 432 tall — and
size takes the max, producing **769px balloons, worse than doing nothing**.

Counts above 50 are cheap: every balloon reuses one of 5 source images, so more
elements cost paint area, not image decodes.

**Click interaction.** Clicking a balloon runs it through `POP_FRAMES` at
`frameMs` intervals, then hides it, and pops a set of neighbours as an outward
wave — delay is `distance / wavePxPerMs`, so balloons at a similar radius go
together and the chain reads as one expanding front rather than a queue.

**Frames: only the odd ones are used** — 1 → 3 → 5 (intact → split → dispersed).
The even frames are neither referenced nor fetched; `preloadFrames()` walks the
same `POP_FRAMES` list, so adding one back is a single-line change. Timing stays
keyed to the frame *number*, not its index, so skipping the evens holds each
remaining frame twice as long rather than shortening the pop. The 10 even-frame
files (988KB) are now dead weight in `Assets/` — see `BACKLOG.md` §3.

**Two pop modes**, switchable at runtime via `window.bday.setPopMode()`:

- `colour` (default) — every balloon of the clicked colour.
- `proximity` — the clicked balloon plus its **nearest N still-live
  neighbours**, where N is `POP_STEPS[config.popLevel - 1]`, currently
  **9 / 15 / 25**.

**Proximity counts balloons, not rings — changed 2026-08-31.** It was Chebyshev
distance in grid cells, which made the levels 9 / 25 / 49. Asking for 15 is what
broke that model: **15 has no symmetric shape on a square grid.** Rings give
9/25/49 and diamonds give 13/21, so no radius expresses it — a shape-based rule
cannot produce the requested step at all.

Selecting a count instead is also strictly better at the field's edges, which is
where the old rule was weakest: a ring near a corner was clipped into a lopsided
wedge, while a count reaches further into the field to fill its quota. Measured
after the change — a corner click returns the full 9 / 15 / 25, identical to a
centre click.

The distance is now **real pixels**, reversing the original cells-not-pixels
call. That reasoning was sound for a ring, where a pixel radius caught lopsided
sets near the edges. For a fixed count the jitter becomes an asset: it settles
the ties that an integer cell distance would otherwise have to break
arbitrarily — and an asymmetric count like 15 is precisely where an arbitrary
tie-break would show as a blob leaning one way.

Only live balloons count, so on a field that has already been clicked a few
times "15" still means fifteen balloons you can actually see. If fewer than N
remain it takes what is left rather than under-delivering silently.

Retune by editing `POP_STEPS` in `template.html`; the preview panel reads it via
`window.bday.popSteps` rather than hardcoding the labels, so a fourth step needs
no change there beyond the slider's `max`.

Both feed the same distance-ordered scheduler, so the wave behaves identically
either way. Like `setPointer()`, neither is part of `setConfig()` — pop
behaviour is read at click time, and a rebuild would clear the popped set just
to change what the *next* click does.

- Popped cells are recorded at *click* time, not when the animation ends, so a
  resize mid-wave can't resurrect a balloon that's already on its way out.
- Frames 2–5 are preloaded at init (20 images, ~6MB — see the asset note below).
- `prefers-reduced-motion: reduce` skips the frame sequence and the wave stagger
  entirely; matching balloons just disappear. **This is a judgement call** — the
  alternative is keeping the in-place frame swap and dropping only the stagger.
  Worth a second opinion.
- The balloons are `aria-hidden` and mouse-only. 96 keyboard tab stops would be
  worse than none, but that means the interaction is currently **unavailable to
  keyboard and screen-reader users**. Fine for pure decoration; if popping ever
  does something meaningful, this needs a real control.

**Confetti — confetti.js, INLINED in `template.html` (changed 2026-08-31).**
`@hiseb/confetti` 2.2.0 (ISC, 4.6KB minified, no runtime dependencies) is now
pasted in full into its own `<script>` block above the component script, rather
than loaded via `<script src>`. **Decided on 2026-08-31: zero external
dependencies** — nothing to `npm install`, nothing to fetch, no second file for
Liferay to serve, and no CSP to fall foul of. This also retires the old
"upload it as a fragment resource or paste it in" integration decision: the
paste is done.

`vendor/confetti.min.js` stays in the repo as the unmodified source of truth for
that paste, but is **not loaded at runtime** — verified by resource timing, it
is no longer requested. The ISC licence requires the notice travel with the
code, so it is reproduced in the comment above the inlined block as well as in
`vendor/confetti-LICENSE.txt`.

It's an IIFE that assigns `window.confetti`, so its block must stay above the
component script. Each balloon fires one burst on frame 3 — the shatter
frame — in its own colour, using the library's own option names
(`count`/`size`/`velocity`/`fade`/`color`/`position`), so its docs apply as
written.

**Calls go through a guarded `burst()` wrapper, never `window.confetti`
directly.** If the vendored file fails to load — bad path after the Liferay
repoint, 404, blocked — the call returns `false` instead of throwing. That
matters: an exception here would abort the rest of the init and take the
balloons down with it. `diagnose()` reports `confettiLibLoaded` for exactly this.

**Two things the library takes out of our hands.** It appends its own canvas to
`<body>` at `z-index: 999999999`, so confetti always paints above everything —
the old "behind the card" finale is not achievable without hacking its internals.
And that canvas is outside our root, so `destroy()` does not remove it.

This replaced a hand-written particle system that was removed on 2026-08-21;
that version is in git history at `cad8da7` and earlier.

**Pointer.** Three modes, switchable at runtime via `window.bday.setPointer()`:
`default` (shipped — arrow on the backdrop, 📌 pushpin over a balloon),
`pin` (`pin.svg` everywhere), `crosshair` (JS-tracked reticle plus full-viewport
rules). Driven by `data-bday-pointer` on the root.

The pushpin is `Assets/round-pin-48.png`, a 48px downscale of the 512px
`round pin.png` — **reduced from 64px on 2026-08-31.** It is generated rather
than referenced directly because **a native cursor cannot be scaled by CSS**:
its size *is* the file's pixel size. Chrome also ignores cursor images over
128px, so pointing at the 512px original would silently fall back to `pointer`
with no error. Hotspot `0 47` is the needle tip, measured from the artwork's
alpha rather than guessed — the tip reaches the bottom-left corner, so it
tracks size as `0, size-1` (confirmed: 512px source tips at 5,505; the 64px
render tipped at 0,63; the 48px at 0,47).

**"default" mode runs two pin mechanisms and the size lives in both.** The
tracked `.bday_pin` element is what you normally see — a real DOM node
following the pointer, needed because a native cursor cannot animate (the
browser hands the OS a static bitmap). The native `cursor: url(...)` is the
*fallback* when `:has()` or the JS tracking fails. They must stay matched or
the fallback jumps size at the worst moment. Changing the size means moving
**five** coupled values together, none of which assert against each other:

| Where | At 48px |
| --- | --- |
| `cursor:` hotspot, `.bday_balloon` rule | `0 47` (= `0, size-1`) |
| `.bday_pin` width/height | `48px` |
| `PIN_HOTSPOT_Y` in the script | `47` |
| `.bday_pin-art` press offset | `translate(-3px, 3px)` (= `4px × size/64`) |
| `Assets/round-pin-48.png` | regenerated; filename encodes the size |

Regenerate with System.Drawing — this is the resize the backlog refers to,
recorded here rather than left as a reference to something that was never
written down:

```powershell
Add-Type -AssemblyName System.Drawing
$in  = [System.Drawing.Bitmap]::FromFile('Assets\round pin.png')
$out = New-Object System.Drawing.Bitmap(48, 48, [System.Drawing.Imaging.PixelFormat]::Format32bppArgb)
$g = [System.Drawing.Graphics]::FromImage($out)
$g.CompositingMode  = [System.Drawing.Drawing2D.CompositingMode]::SourceCopy
$g.InterpolationMode = [System.Drawing.Drawing2D.InterpolationMode]::HighQualityBicubic
$g.PixelOffsetMode   = [System.Drawing.Drawing2D.PixelOffsetMode]::HighQuality
$g.DrawImage($in, (New-Object System.Drawing.Rectangle(0, 0, 48, 48)))
$g.Dispose(); $out.Save('Assets\round-pin-48.png', [System.Drawing.Imaging.ImageFormat]::Png)
```

`SourceCopy` matters — without it the resize composites against the empty
destination and premultiplies the alpha, which darkens the soft edge. After
regenerating, re-measure the tip from the alpha rather than assuming it scaled.
The 512px source stays in `Assets/` for this. Deliberately *not* part of `setConfig()` —
that rebuilds the field and would wipe the popped set just to swap a cursor.
The crosshair listeners stay attached in all modes and early-return, so the pin
and default modes run no per-move JS at all. Which one ships is still open: the
crosshair reads as a targeting scope, which is a questionable register for a
birthday.

**Tuning panel.** `preview.html` carries 10 sliders, a pop-mode switcher, a
pointer switcher and a readout, driving `window.bday.config` via `setConfig()`.
`×` or Esc hides it; a `tune` pill brings it back.

Count and pixel size are **outputs** now, so the panel reports them
(`45 balloons · 320px · viewport 1440×900`) rather than controlling them. The
old `Balloon count` and `Balloon size` sliders are gone — leaving them would
have been silently inert once those keys left `CONFIG`.

Current `CONFIG`: `balloonScale 1.78`, 12° tilt, 0.17 scatter, 30ms frames,
2px/ms wave, confetti 40 at size 1 / velocity 200, pop mode `colour`,
`popLevel 1` (9 balloons), pointer `default`.

`balloonScale 1.78`, `MAX_BALLOON_PX 360` and `HARD_MAX_BALLOONS 110` were
**accepted on 2026-08-21** on the strength of the measurements above. Worth
recording that this was a sign-off on the numbers, not on the rendering —
nobody had viewed the field at these settings at the time. If the density ever
looks wrong, `balloonScale` is the first dial, `MAX_BALLOON_PX` the second.

Note this **invalidates any config JSON saved before 2026-08-21** — `maxBalloons`
and `balloonSize` no longer exist and are silently ignored by `setConfig()`. Use
"Copy config" to get the current shape.

**`popLayers` was retired on 2026-08-31**, replaced by `popLevel` for the same
reason: it named rings, and proximity no longer pops rings. A saved config
carrying `popLayers` is silently ignored, so a field that was tuned to "2
layers" comes back at `popLevel 1` — re-pick the level rather than assuming it
carried over.

The panel is **preview-only** — per §1, dev UI does not ship in the fragment.
What does ship is the small `config` / `setConfig` / `reset` / `setPopMode` /
`setPointer` / `diagnose` API it drives.

**Open items** live in `BACKLOG.md` — the invented gradient alpha, asset sizing and
frame registration, the unanswered brief questions, verification gaps, packaging.
Add to that file rather than leaving notes here: **this playbook records decisions
made, the backlog records decisions pending.**

---

## 5. Verification

Per conversion playbook §5 — measure, don't eyeball. What that caught here:

- the magenta border leak in §3 (computed `borderTopWidth` on our own image)
- `preview.html` was missing `<meta name="viewport">`, so mobile emulation fell back
  to a 980px layout viewport. **The fragment can't fix this** — the meta tag belongs
  to the host Liferay page. Confirm it's there during integration.

Assert at minimum: root rect equals viewport, background rect equals root,
`document.documentElement.scrollWidth <= innerWidth`, zero console errors. Check at
1280×800 and 1024×768 — see §6 for why nothing narrower.

Screenshots were unavailable throughout this session (the browser pane wasn't
compositing). Measurement covered behaviour, but **colour and visual weight have not
been confirmed by eye** — anything appearance-related still needs a human look.

---

## 6. Viewport floor — 1024px

**Do not design or test below 1024px.** This is a desktop intranet takeover; phone
widths are out of scope.

- Verify at 1280×800 and 1024×768. Don't bother with 360px / 375px.
- No breakpoints, stacking rules or mobile-only behaviour under 1024px.
- Don't add a mobile fallback "just in case" — that's the §1 rule again.

This **overrides** the companion playbooks, both of which list "renders correctly at
~360px wide" in their definition of done. Ignore that line on this project.

Below the floor the layout is simply unsupported — it won't crash, it just isn't a
case we design for. The balloon fill in §4 caps at 100 regardless of viewport, so
narrow widths degrade to a sparse field rather than breaking.

---

## 7. Running the preview

```bash
npx --yes serve -l 5757 .
```

Then open `http://localhost:5757/preview.html`. It must be served over `http://` —
the harness `fetch`es `template.html` so the two can't drift, and `fetch` is blocked
on `file://`.

`preview.html` is a test file. It is not part of the handover.
