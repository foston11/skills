# Rendering and performance playbook

Use this reference when any component owns a canvas, shader, Three.js renderer, or continuous animation.

## Render lifecycle

Every effect has the states **uninitialized → ready → running → suspended → destroyed**. The owner must provide start, suspend, resize, and destroy behavior. Never create a new RAF loop on every navigation, observer callback, or pointer event.

```js
function createLoop(draw, element) {
  let raf = 0, visible = false, active = true, last = 0;
  const reduced = matchMedia('(prefers-reduced-motion: reduce)');
  function shouldRun() {
    return active && visible && !document.hidden && !reduced.matches;
  }
  function stop() { cancelAnimationFrame(raf); raf = 0; last = 0; }
  function tick(now) {
    raf = 0;
    if (!shouldRun()) return;
    const dt = last ? Math.min((now - last) / 1000, 0.05) : 0;
    last = now;
    draw(dt, now);
    raf = requestAnimationFrame(tick);
  }
  function sync() { if (!shouldRun()) stop(); else if (!raf) raf = requestAnimationFrame(tick); }
  const io = new IntersectionObserver(([e]) => { visible = e.isIntersecting; sync(); });
  io.observe(element);
  document.addEventListener('visibilitychange', sync);
  reduced.addEventListener('change', sync);
  return {
    setActive(value) { active = value; sync(); },
    destroy() {
      stop(); io.disconnect();
      document.removeEventListener('visibilitychange', sync);
      reduced.removeEventListener('change', sync);
    }
  };
}
```

For static reduced-motion fallback, draw a representative still frame before suspending. When intersection visibility changes, control the render loop, not only the draw call.

## Canvas resolution budget

CSS size is not drawing-buffer size. A 1440×900 surface at DPR 2 uses 5.18 million pixels before depth buffers and postprocessing. Multiple full-screen canvases multiply cost.

```js
function fitCanvas(canvas, maxDpr = 1.5, maxPixels = 2_000_000) {
  const box = canvas.getBoundingClientRect();
  const pixels = Math.max(1, box.width * box.height);
  const dpr = Math.min(devicePixelRatio || 1, maxDpr, Math.sqrt(maxPixels / pixels));
  const w = Math.max(1, Math.round(box.width * dpr));
  const h = Math.max(1, Math.round(box.height * dpr));
  if (canvas.width !== w || canvas.height !== h) {
    canvas.width = w;
    canvas.height = h;
  }
  return { width: w, height: h, dpr };
}
```

Batch ResizeObserver reactions in one rAF to avoid unnecessary size allocations. For Three.js, synchronize renderer.setPixelRatio and renderer.setSize; for a shader, update uResolution. Avoid resizing if backing dimensions have not changed.

## Pointer input

Keep pointer listeners passive when appropriate and only capture latest normalized coordinates. Consume the values inside the existing render loop; do not invoke an expensive render for every pointermove event. On touch, avoid hover-only affordances.

## WebGL practices

- Prefer WebGL2 for shaders only when supported; fall back to CSS/static image.
- Compile and link shaders once. Reuse buffers, textures, and uniform locations.
- Do not call getUniformLocation, createBuffer, or setSize every frame.
- Use delta-time-based movement; clamp dt after background-tab suspension.
- Handle webglcontextlost/webglcontextrestored where feasible.
- Dispose textures, geometries, materials, framebuffers, and renderers on teardown.
- Cache expensive static results and limit fragment shader complexity, layers, blur passes, and overdraw.
- Use one visual hero effect rather than stacking several full-screen fragment shaders.
- Check GPU and browser support; low-end or battery-saving devices receive simplified variants.

## 3D scenes

The ToolMC snapshot contains globe and crown Three.js renderers, each capping pixel ratio but scheduling frames continuously. Improve by **stopping RAF scheduling** on route change/when offscreen, not just skipping renderer.render(). Consider on-demand rendering for mostly static objects. Initialize scenes only when opened.

## Audio

Audio is always opt-in. Never play before user interaction. Suspend Web Audio when hidden, release/close contexts on destruction, and preserve mute preference. A fancy adaptive-audio label is not a substitute for a functioning audio path.

## Performance verification

1. Record initial load and transitions using browser Performance panel.
2. Count simultaneous RAF loops/canvases, WebGL contexts, and event listeners.
3. Inspect long tasks, style recalculation, painting, layout shifts, shader compilation and GPU usage.
4. Verify page transitions under CPU throttling and on a real low-power mobile device.
5. Check memory across repeated route cycles and tab hide/show.
6. Optimize the worst measured bottleneck first; don't randomly remove effects.

Useful initial **budgets, not guarantees**: 1 animated fullscreen shader on low-power devices, capped DPR around 1–1.5, pause all offscreen loops; target stable motion under tested conditions. Performance targets must be revised based on measurements.

## Common bugs

- **Hidden canvas becomes 1×1**: observe container and recompute size when route becomes active again.
- **Huge jump on return**: reset `last` timestamp when resuming.
- **Multiple overlapping transitions**: lock or cancel previous animation using explicit state.
- **Overheating after navigation**: stop offscreen RAF and dispose superseded renderers.
- **First interaction freezes**: precompile critical shaders during idle time, not inside the first pointer event.
- **Blurry shader**: synchronize CSS box, drawing buffer, DPR, viewport and resolution uniform.
- **Motion sickness**: support reduced-motion and tone down large parallax/screen movement.
