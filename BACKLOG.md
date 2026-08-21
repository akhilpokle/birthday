# Backlog — 2027 birthday takeover

> **2026-08-21 — confetti is now confetti.js, vendored.** The hand-written
> particle system was removed, then replaced with `@hiseb/confetti` 2.2.0 copied
> into `vendor/`. The centre-screen finale was not restored.

Open items as of **2026-08-21**. Companion to `LIFERAY-PLAYBOOK.md` (which records
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
- [ ] **"Timothy" is hardcoded.** The heading reads "Happy birthday Timothy!" —
      the name must come from Liferay user context or every employee gets
      Timothy's card. Same for anything date-dependent ("this month").
- [ ] **Copy needs sign-off.** Two typos were corrected on the way in — "Dont" →
      "Don't" and a stray space in "Timothy !". Flagging rather than burying:
      revert either if they were deliberate.
- [ ] **Public Sans is assumed, not loaded.** The fragment declares the family and
      falls back to `system-ui`; only `preview.html` actually fetches the webfont,
      from the Google CDN. Confirm the real theme serves Public Sans — if not it
      needs self-hosting, since a strict CSP will block the CDN.
- [ ] **The card's text area has almost no headroom left.** At the specced 702px
      card with 32/20px type, clearance between the copy and the printed balloon
      band is **46px** — down from 144px when the card was 1040px wide. Measured
      failure points: a long name wraps the heading to two lines and leaves
      **7px**; one extra sentence of body copy leaves **16px**. Neither overflows
      today, but there is no room for real user data to be longer than
      "Timothy". Either the copy needs a hard length limit, or the type should
      scale with the card, or the text needs its own area rather than sitting on
      the artwork's blank band.
- [ ] **No CTA on the card.** Still no destination URLs collected.
- [ ] **Analytics** — no events dispatched on open, pop, or dismiss.
- [ ] **Browser floor** beyond the 1024px width decision. Decides whether
      `backdrop-filter`, `clip-path` and `inset` are safe as written (all are used).

---

## 2. Known risks in the code

- [ ] **`vendor/confetti.min.js` needs an integration decision.** It's a separate
      `<script src>`, but a Liferay fragment has *one* JS field. Either upload it
      as a fragment resource and repoint the src to `[resources:confetti.min.js]`,
      or paste its 4.6KB above the component script in the JS field. Until that's
      done the confetti silently no-ops — by design, but silently. Check
      `diagnose().confettiLibLoaded` after integrating.
- [ ] **Third-party code now ships in the fragment.** ISC licensed, no runtime
      dependencies, 4.6KB — but it is someone else's code inside a bank's
      intranet, and `vendor/confetti-LICENSE.txt` must travel with it. Worth a
      nod from whoever signs off on dependencies.
- [ ] **Confetti always paints on top.** The library appends its canvas to
      `<body>` at `z-index: 999999999`, outside our root — so it covers the card,
      and `destroy()` can't remove it. If confetti ever needs to sit *behind*
      something, this library can't do it without patching.

- [ ] **The pop is suppressed entirely under `prefers-reduced-motion: reduce`,**
      and managed Windows machines commonly have "Show animations" disabled by
      policy — which sets it. On such a machine balloons vanish instantly with no
      crack-and-shatter sequence at all. Run `window.bday.diagnose()` there to
      confirm. **Decide:** keep suppressing (correct by the letter of the
      guideline, but the interaction loses all its feedback), or keep the
      in-place frame swap and drop only the distance-staggered wave.

- [ ] **`position: fixed` may not resolve against the viewport.** Conversion
      playbook §7: `fixed` resolves against the nearest ancestor with a `transform`,
      `filter` or `perspective`, and CMS themes commonly have one. If the real theme
      does, the takeover will be positioned inside that ancestor instead of covering
      the window. The playbook's fix is to **reparent the root to `<body>` on init** —
      not implemented. Test against the real theme early; this is cheap to fix and
      expensive to discover late.
- [ ] **`z-index: 9999` is a guess.** Never checked against the real theme's stacking.
- [ ] **Popped state is keyed by grid cell**, so `setConfig()` deliberately clears it.
      Fine for tuning; revisit if pops ever need to survive a genuine relayout.
- [ ] **Stacking depends on hardcoded z-index values.** `.bday_balloons` creates no
      stacking context, so the balloons' inline `z-index` (1..`Z_LEVELS`, currently
      100) competes directly with the crosshair layer (300) in the root's context.
      Raising `Z_LEVELS` above 300 would silently put balloons over the crosshair.
      Worth making the layer a stacking context of its own (`isolation: isolate`)
      so the inline values can't escape it.
- [ ] **The rule colour is duplicated, not shared.** `.bday_rule`'s background
      (`#FF7E65`) has to match the stroke inside `Assets/crosshair.svg` by hand —
      they form one continuous line, so editing the svg's colour without editing
      the CSS shows as a seam at the join. Worth driving both from one custom
      property if the svg is ever inlined.

---

## 3. Assets

- [ ] **Asset weight has its own file now — see `LIGHTENING.md`.** Summary: the
      sources are 800×800 rendering at 208px, `Assets/` is 7.9MB, and resizing
      plus compression should get it to ~1.5MB. The code-side work is already done.
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

- [x] ~~The final message is an image.~~ Resolved: the card artwork stays
      decorative (`alt=""`) and the message is live text over it, so it's
      readable, translatable and selectable-by-AT. Heading is `h2`, not `h1`,
      because the host page owns the `h1` — revisit if this becomes its own page.
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
      density, not the pop.
- [ ] **Pop cadence unmeasured.** `frameMs: 30` means 5 frames in 120ms. Whether the
      crack and shatter stages are perceptible at that speed is unknown.
- [ ] **The reduced-motion path is code-inspected only** — the media query couldn't
      be flipped on a live instance.
- [x] ~~`clip-path: circle(50%)` was reasoned to clip only transparent corners,
      not measured.~~ **It was cutting real artwork** — 6.1% of Red's opaque pixels,
      2.9% of Blue's, 2.2% of Yellow's, because the shapes differ per colour
      (Yellow is a star, Red is broad). Removed 2026-08-20.
- [ ] **Retina untested** — canvas DPR scaling was only exercised at `dpr: 1`.
- [ ] **Hover scale never observed animating.** `:hover` can't be forced from
      script and the hidden tab has no real pointer, so the 100ms-in / 50ms-out
      transition is unverified in motion. The rules and the composed transform
      (scale 1 → 1.1 with tilt preserved) are both confirmed.
- [ ] **A hovered balloon can scale up behind its neighbours.** Depth is random
      and fixed, so growing by 1.1 doesn't bring it forward. Raising z-index on
      hover would fix it but fights the random stacking; worth a look once the
      effect can actually be seen.
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
- [ ] **Confirm the JS field has no size limit** the inlined script would breach.
- [ ] **Do not ship `preview.html`** — say so explicitly in the README.

---

## 7. Nice to have

- [ ] Persist tuning-panel values to `localStorage` so a reload doesn't reset them.
- [ ] The tuning panel rebuilds the whole field on every `input` event; fine at ≤100
      nodes, would need throttling if the cap ever rises.
