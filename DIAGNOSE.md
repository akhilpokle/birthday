# Diagnosing "the confetti doesn't show"

Paste the block below into the browser console **on the machine where it fails**,
press Enter, and send me what it prints.

It does not depend on `window.bday` existing — that is one of the things it
checks. `Uncaught ... no property` almost certainly means the page you are on is
running an older build that predates `diagnose()`, so **hard-reload first**:
`Ctrl+Shift+R` (Windows) or `Cmd+Shift+R` (Mac).

```js
(async () => {
  const out = {};
  out.url = location.href;

  // 1. Is the component even initialised?
  out.bdayGlobal = typeof window.bday;
  out.bdayKeys = window.bday ? Object.keys(window.bday) : null;
  out.hasDiagnose = !!(window.bday && window.bday.diagnose);

  // 2. Are you on the current build? If false, it's a cache/deploy problem,
  //    not a rendering problem.
  try {
    const src = await fetch('template.html', { cache: 'reload' }).then(r => r.text());
    out.buildHasDiagnose = src.includes('diagnose');
    out.buildHasResizeObserver = src.includes('ResizeObserver');
    // Match the call, not the word — the word still appears in a comment
    // explaining why it was removed.
    out.buildStillPassesContextOptions = src.includes('getContext("2d", {');
  } catch (e) { out.buildFetchError = e.message; }

  // 3. The single most likely cause on a managed Windows machine.
  out.prefersReducedMotion =
    matchMedia('(prefers-reduced-motion: reduce)').matches;

  // A background/hidden tab suspends requestAnimationFrame, so nothing paints
  // and the burst test below reads 0 for a reason that isn't a bug. Keep the
  // page focused and visible while running this.
  out.documentHidden = document.hidden;

  // 4. Does canvas 2d actually paint on this GPU/driver?
  try {
    const c = document.createElement('canvas');
    c.width = c.height = 8;
    const x = c.getContext('2d');
    x.fillStyle = '#f00';
    x.fillRect(0, 0, 8, 8);
    out.canvas2dPaints = x.getImageData(0, 0, 1, 1).data[3] === 255;
  } catch (e) { out.canvas2dPaints = false; out.canvasError = e.message; }

  // 5. State of the two confetti canvases.
  const f = document.getElementById('bday_confetti');
  const b = document.getElementById('bday_confetti_back');
  out.frontCanvas = f ? f.width + 'x' + f.height : 'MISSING';
  out.backCanvas  = b ? b.width + 'x' + b.height : 'MISSING';
  out.balloonsInDom = document.querySelectorAll('.bday_balloon').length;
  const layer = document.getElementById('bday_balloons');
  out.layerSize = layer ? layer.clientWidth + 'x' + layer.clientHeight : 'MISSING';

  // 6. Fire a burst and see whether anything lands on the canvas.
  if (window.bday && window.bday.confetti && f) {
    const before = f.getContext('2d').getImageData(0, 0, f.width || 1, f.height || 1);
    window.bday.confetti({ position: { x: innerWidth / 2, y: innerHeight / 2 }, count: 60 });
    await new Promise(r => setTimeout(r, 250));
    const d = f.getContext('2d').getImageData(0, 0, f.width || 1, f.height || 1).data;
    let painted = 0;
    for (let i = 3; i < d.length; i += 4) if (d[i] > 0) { painted++; if (painted > 50) break; }
    out.testBurstPaintedPixels = painted;
  }

  out.devicePixelRatio = devicePixelRatio;
  out.userAgent = navigator.userAgent;
  console.log(JSON.stringify(out, null, 2));
  return out;
})();
```

## How to read the result

| What you see | What it means |
| --- | --- |
| `buildHasDiagnose: false` | **You're on a stale build.** Cache or GitHub Pages hasn't redeployed. Nothing else in the output is meaningful yet. |
| `buildStillPassesContextOptions: true` | Same — the `desynchronized` context option was removed; if it's still being passed you have old code. |
| `documentHidden: true` | The tab isn't visible, which suspends `requestAnimationFrame`. `testBurstPaintedPixels` will read 0 regardless. Focus the tab and re-run. |
| `bdayGlobal: "undefined"` | The script never ran. Look for an earlier error in the console — a CSP block would show here. |
| `prefersReducedMotion: true` | **Confetti is suppressed on purpose.** Windows: Settings → Accessibility → Visual effects → Animation effects. This is the most likely cause on a managed laptop. |
| `canvas2dPaints: false` | Canvas is blocked or broken on that GPU/driver. Rare, but decisive. |
| `frontCanvas: "0x0"` | The zero-size init bug — the component measured 0 when it started. Should be fixed by the ResizeObserver; if you see it, tell me. |
| `balloonsInDom: 0` | Same root cause as above. |
| `testBurstPaintedPixels: 0`, `documentHidden: false`, everything else fine | The burst emitted but painted nothing — a real rendering problem, and I'd want the `userAgent`. |
| `testBurstPaintedPixels > 0` | Confetti **is** working; the issue is that you can't see it — position, timing or layering rather than rendering. |
