# Backlog — 2027 birthday takeover

Open items as of **2026-08-20**. Companion to `LIFERAY-PLAYBOOK.md` (which records
decisions already made); this file records what's still undecided or unfinished.

Ordered roughly by cost of leaving it: the things at the top change what gets built,
the things at the bottom are chores.

---

## 1. Unanswered brief questions — these change the architecture

None of the conversion playbook's §0 questions have been settled. Answers change
the code, not just the content.

- [ ] **Where does this live?** Its own page, a full-screen overlay on top of the
      live intranet, or an inline block? Currently built as a `position: fixed`
      full-viewport takeover. The disambiguating question is *"when it's dismissed,
      what does the user see?"*
- [ ] **Should the background layer exist at all?** It's currently
      `Assets/background.svg` — a generic placeholder dashboard, swapped in on
      2026-08-20 so the public GitHub Pages repo wouldn't carry a real intranet
      capture. If this is meant to sit *over* the live page, the `<img>` should be
      deleted entirely and `backdrop-filter` left to blur the real content
      underneath. **This is the single biggest open question in the project.**
      The original screenshot is at `_local/background-REAL-do-not-publish.png`
      (gitignored) — it shows one named user's dashboard, so it could never have
      shipped as-is anyway.
- [ ] **Who sees it, how often?** No gating, no once-per-user persistence. Reloading
      restores every popped balloon.
- [ ] **How is it dismissed?** There's no close control, no Esc handler, no
      `open()`/`close()`. `destroy()` exists but nothing calls it.
- [ ] **What's the actual content?** There is no headline, copy, CTA or user data —
      only balloons. No destination URLs collected.
- [ ] **Analytics** — no events dispatched on open, pop, or dismiss.
- [ ] **Browser floor** beyond the 1024px width decision. Decides whether
      `backdrop-filter`, `clip-path` and `inset` are safe as written (all are used).

---

## 2. Known risks in the code

- [ ] **`position: fixed` may not resolve against the viewport.** Conversion
      playbook §7: `fixed` resolves against the nearest ancestor with a `transform`,
      `filter` or `perspective`, and CMS themes commonly have one. If the real theme
      does, the takeover will be positioned inside that ancestor instead of covering
      the window. The playbook's fix is to **reparent the root to `<body>` on init** —
      not implemented. Test against the real theme early; this is cheap to fix and
      expensive to discover late.
- [ ] **`z-index: 9999` is a guess.** Never checked against the real theme's stacking.
- [ ] **Particle cap can clip a large wave.** `MAX_PARTICLES` is 1400; at
      `confettiCount: 38`, a colour with 37+ balloons would hit it and later bursts
      would emit nothing. Today's worst case is 26 balloons (988 particles), so
      there's headroom — but raising the count or the balloon cap could cross it
      silently. No warning is logged when a burst is clamped.
- [ ] **Popped state is keyed by grid cell**, so `setConfig()` deliberately clears it.
      Fine for tuning; revisit if pops ever need to survive a genuine relayout.
- [ ] **Stacking is held together by two hardcoded z-index values.** `.bday_balloons`
      creates no stacking context, so the balloons' inline `z-index` (1..`Z_LEVELS`,
      currently 100) competes directly with the confetti canvas (200) and the
      crosshair layer (300) in the root's context. Raising `Z_LEVELS` above 200
      would silently put balloons back on top of the confetti. Worth making the
      layer a stacking context of its own (`isolation: isolate`) so the inline
      values can't escape it.
- [ ] **The rule colour is duplicated, not shared.** `.bday_rule`'s background
      (`#FF7E65`) has to match the stroke inside `Assets/crosshair.svg` by hand —
      they form one continuous line, so editing the svg's colour without editing
      the CSS shows as a seam at the join. Worth driving both from one custom
      property if the svg is ever inlined.

---

## 3. Assets

- [ ] **Resize the sources.** All 25 frames are 800×800 but render at 208px — a ~4×
      downscale on 96 balloons. At `frameMs: 30` five frames swap in 120ms, which is
      the most decode-sensitive moment in the whole experience. Generate ~416px
      versions (2× for retina). Preloading currently pulls ~6MB of PNGs at init.
- [ ] **Frames aren't registered to a common centre.** Alpha-weighted centroids drift
      across each sequence — Red 5 sits ~95px left and ~113px below Red 1 (>10% of the
      canvas). Swapping frames in place makes the balloon visibly jump, worst in Red.
      Fix by baking a per-frame offset into the sprite data.
- [ ] **Content runs to the canvas edge on 18 of 25 frames** (Red 3 touches all four).
      Debris terminates at a hard straight line instead of dispersing. Worst in the
      Red and Yellow sets. **This affects frame 1 too**, so it's not only a debris
      problem: `Yellow 1` has 0px padding on the left and `Red 1` has 0px on left,
      right and bottom — the balloon art is flush against the canvas edge and may
      be genuinely cut in the source. Re-export with padding if so.
- [ ] **Transparent corners are clickable again** now the clip-path is gone. A click
      on a balloon's empty corner pops it even though nothing is drawn there. Proper
      fix is alpha hit-testing: sample the frame-1 image at the click point and, if
      transparent, fall through to whatever is underneath. Not currently a reported
      problem — balloons overlap heavily at 208px, so most corners sit over another
      balloon anyway.
- [ ] **`green busrt.png` is misspelled** (in the superseded set).
- [ ] **Filenames contain spaces** (`Blue 1.png`), so the JS URL-encodes them. Liferay
      `[resources:…]` paths will be easier if these are renamed `blue-1.png`.
- [ ] **Decide the canonical set.** The old assets (`blue.png`, `red.png`,
      `blue burst.png`, `sage burst.png`, …) are still in `Assets/` and unused by the
      component. Delete or archive them.
- [ ] **Remove junk from `Assets/`** before handover: `new balloons.zip`, the empty
      `new balloons/` tree, `__MACOSX/`, and `tinified (1)/` — which contains
      unrelated images (`Claims.jpg`, `Nursing room.jpg`, `Medical protection.jpg`…)
      that appear to belong to a different project.

---

## 4. Accessibility

- [ ] **The interaction is mouse-only.** Balloons are `aria-hidden` decorative
      `<img>`s with no keyboard path. Deliberate — 96 tab stops would be worse than
      none — but it means keyboard and screen-reader users cannot pop anything.
      Acceptable while it's pure decoration; **needs a real control the moment
      popping does something meaningful.**
- [ ] **Reduced-motion behaviour needs a second opinion.** `prefers-reduced-motion:
      reduce` currently skips the frame sequence *and* the wave stagger, so matching
      balloons simply vanish. The gentler alternative is keeping the in-place frame
      swap and dropping only the stagger.
- [ ] No focus management, no live-region announcement — both moot until there's a
      keyboard path and real content.

---

## 5. Verification gaps

Everything below is unverified because the browser pane never composited during the
build: screenshots timed out all session, timers were throttled to ~1Hz, and
`requestAnimationFrame` was suspended entirely.

- [ ] **Nothing has been confirmed by eye.** Not the gradient, not the balloon
      density, not the pop, not the confetti.
- [ ] **Pop cadence unmeasured.** `frameMs: 30` means 5 frames in 120ms. Whether the
      crack and shatter stages are perceptible at that speed is unknown.
- [ ] **Confetti was verified by pumping `requestAnimationFrame` by hand.** Physics,
      colour, cap and cleanup are all confirmed — but real frame rate and visual
      weight are not. Watch for jank with a full wave.
- [ ] **Confetti's reduced-motion path is code-inspected only** — the media query
      couldn't be flipped on a live instance.
- [x] ~~`clip-path: circle(50%)` was reasoned to clip only transparent corners,
      not measured.~~ **It was cutting real artwork** — 6.1% of Red's opaque pixels,
      2.9% of Blue's, 2.2% of Yellow's, because the shapes differ per colour
      (Yellow is a star, Red is broad). Removed 2026-08-20.
- [ ] **Retina untested** — canvas DPR scaling was only exercised at `dpr: 1`.
- [ ] **Never tested on the real Liferay theme, or on Safari/iPadOS** (iPad is ≥1024
      so it's in scope).

---

## 6. Packaging — none of this exists yet

- [ ] **Build script** to split `template.html` into the three fragment fields
      (`.html` / `.css` / `.js`). Currently one file; the split is still manual.
- [ ] **`handover/` folder** per playbook §5.
- [ ] **README** with the integrator task list — the one instruction file. Must open
      by answering "which file do I open?"
- [ ] **Repoint asset paths** to `[resources:…]`.
- [ ] **Confirm the JS field has no size limit** that the inlined confetti + preload
      list would breach.
- [ ] **Do not ship `preview.html`** — say so explicitly in the README.

---

## 7. Nice to have

- [ ] Warn (console) when a confetti burst is clamped by `MAX_PARTICLES`.
- [ ] Persist tuning-panel values to `localStorage` so a reload doesn't reset them.
- [ ] The tuning panel rebuilds the whole field on every `input` event; fine at ≤100
      nodes, would need throttling if the cap ever rises.
