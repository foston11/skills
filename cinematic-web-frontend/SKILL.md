---
name: cinematic-web-frontend
description: Design and implement original, premium-feeling websites with strong art direction, precise motion choreography, WebGL2 or Three.js only when justified, accessible interactions, mobile-first performance and production verification. Use for landing pages, interactive showcases, web games/community sites, shader-heavy interfaces, frontend audits and anti-template redesigns.
---

# Cinematic Web Frontend — engineering playbook

> A reusable frontend engineering skill distilled from inspecting the supplied ToolMC public frontend snapshot. Learn patterns, not branding, visual assets, or proprietary code. This instruction is independent of that website and must generate original designs.

## How to use this skill

- Read this document as an operational checklist and recipe catalog.
- Start with **Architecture and example implementations** below, then consult only relevant recipes.
- Read `references/rendering-and-performance.md` for GPU and lifecycle work.
- Read `references/motion-and-interaction.md` for navigation and interaction behavior.
- Favor a convincing static screen over a pile of runtime effects.
- Follow the existing stack unless change is explicitly justified.
- Never manufacture content or invent benchmarks.
- Respect the user's design brief, especially bans on cards, neon, glassmorphism, audio, and unnecessary copy.

## Source audit and limitations

ToolMC snapshot supplied by user: `index.html`, seven stylesheets and fifteen JavaScript files in a ZIP. Concrete modules include `main.js`, `aurora.js`, `dither-veil.js`, `grainient.js`, `gradient-ascii.js`, `globe.js`, `crown.js`, `ranks-hero.js`, `intro.js`, `community.js`, and `sound-engine.js`. This skill abstracts architectural observations; it intentionally does **not** copy the original scripts, shader strings, brand text, or graphics. Some observed implementations cap DPR, use observers and procedural shaders; some 3D loops unnecessarily continue scheduling frames when hidden. Reproduce the ideas, improve the lifecycle.

## Operating procedure

1. Inventory files, dependencies, sections, real user interactions, effects and active rendering contexts.
2. Write one sentence describing the visual thesis and one sentence describing what this project deliberately avoids.
3. Create typography, color, spacing, composition, motion, accessibility, and performance tokens.
4. Build usable HTML and a compelling static screen first.
5. Implement interaction states and error cases before decorative rendering.
6. Add effects progressively: CSS, then Canvas2D, then WebGL2/Three.js when needed.
7. Implement start/pause/resume/destroy for every effect.
8. Validate on narrow screens, touch, keyboard, reduced motion and offline/failing assets.
9. Measure and fix actual bottlenecks; don't claim target frame rates without results.
10. Ship verified output with clear modified files and remaining limitations.

## Architecture and example implementations

### Example A — project layout

```text
src/
  app/
    navigation.js
    state.js
  styles/
    tokens.css
    layout.css
    motion.css
  effects/
    create-loop.js
    create-canvas.js
    aurora.js
    dither.js
  components/
    menu.js
    tabs.js
  content/
    strings.js
  main.js
```

- Each visual effect owns its DOM bindings, observer, RAF, GPU objects and destroy routine.
- The navigation controller owns current route and transition state.
- UI components own input handlers but do not mutate unrelated page state.
- Shared tokens encode visual language, not anonymous global animations.

### Example B — design tokens

```css
:root {
  color-scheme: dark;
  --bg: #11151a;
  --surface: #1b2229;
  --muted: #a7b4c2;
  --text: #f1f4f6;
  --accent: #8dbddd;
  --line: color-mix(in srgb, var(--text) 12%, transparent);
  --space-1: .25rem;
  --space-2: .5rem;
  --space-3: .75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-12: 3rem;
  --ease-out: cubic-bezier(.22, 1, .36, 1);
  --fast: 140ms;
  --normal: 260ms;
  --slow: 540ms;
}
@media (prefers-reduced-motion: reduce) {
  :root { --fast: 0ms; --normal: 0ms; --slow: 0ms; }
}
```

### Example C — static layout before effects

```html
<main id="main">
  <section class="hero" aria-labelledby="hero-title">
    <div class="hero__copy">
      <p class="eyebrow">A SPECIFIC CATEGORY</p>
      <h1 id="hero-title">Make the product concept immediately clear.</h1>
      <p class="hero__support">One concise supporting sentence with a real claim.</p>
      <a class="primary-action" href="#details">Explore details</a>
    </div>
    <div class="hero__visual" aria-hidden="true">
      <div class="hero__fallback"></div>
      <canvas class="hero__canvas"></canvas>
    </div>
  </section>
</main>
```

```css
.hero { display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1.12fr);
  min-height: min(720px, 100svh); align-items: center; gap: clamp(2rem, 5vw, 6rem); }
.hero h1 { font-size: clamp(2.6rem, 5.8vw, 6.8rem); line-height: .99; letter-spacing: -.055em; }
.hero__visual { position: relative; min-height: 28rem; isolation: isolate; }
.hero__canvas { position: absolute; inset: 0; width: 100%; height: 100%; pointer-events: none; }
@media (max-width: 760px) {
  .hero { grid-template-columns: 1fr; min-height: auto; }
  .hero__visual { min-height: 18rem; }
}
```

### Example D — safe route animation

```js
let phase = "idle";
let selected = "home";
async function switchPage(next, pages, indicator) {
  if (phase !== "idle" || next === selected || !pages[next]) return false;
  const current = pages[selected];
  const incoming = pages[next];
  const reduce = matchMedia("(prefers-reduced-motion: reduce)").matches;
  phase = "exit";
  try {
    if (!reduce) {
      await current.animate([{ opacity: 1, transform: "translateY(0)" },
        { opacity: 0, transform: "translateY(-8px)" }],
        { duration: 170, easing: "ease-out", fill: "forwards" }).finished;
    }
    phase = "swap";
    current.hidden = true;
    current.getAnimations().forEach(a => a.cancel());
    incoming.hidden = false;
    selected = next;
    indicator.setAttribute("data-route", next);
    phase = "enter";
    if (!reduce) {
      await incoming.animate([{ opacity: 0, transform: "translateY(10px)" },
        { opacity: 1, transform: "translateY(0)" }],
        { duration: 260, easing: "cubic-bezier(.22,1,.36,1)" }).finished;
    }
    return true;
  } catch (error) {
    if (error?.name !== "AbortError") console.error("Route transition", error);
    return false;
  } finally {
    phase = "idle";
  }
}
```

Note: this example ignores repeat clicks while switching. For interruptible routing, use a monotonic request ID and cancel the previous animation before starting a new request. Focus management and ARIA state must be added for production.

### Example E — one RAF owner with pause/resume

```js
function createFrameLoop(node, draw) {
  let id = 0, isVisible = false, enabled = true, previous = 0;
  const motion = matchMedia("(prefers-reduced-motion: reduce)");
  const shouldRun = () => enabled && isVisible && !document.hidden && !motion.matches;
  const stop = () => { cancelAnimationFrame(id); id = 0; previous = 0; };
  function frame(now) {
    id = 0;
    if (!shouldRun()) return;
    const dt = previous ? Math.min((now - previous) / 1000, .05) : 0;
    previous = now;
    draw(dt, now);
    id = requestAnimationFrame(frame);
  }
  const sync = () => {
    if (!shouldRun()) stop();
    else if (!id) id = requestAnimationFrame(frame);
  };
  const observer = new IntersectionObserver(([e]) => {
    isVisible = e.isIntersecting;
    sync();
  }, { threshold: .01 });
  observer.observe(node);
  document.addEventListener("visibilitychange", sync);
  motion.addEventListener("change", sync);
  return {
    setEnabled(value) { enabled = !!value; sync(); },
    destroy() {
      stop();
      observer.disconnect();
      document.removeEventListener("visibilitychange", sync);
      motion.removeEventListener("change", sync);
    }
  };
}
```

### Example F — cap pixel area, not just DPR

```js
function sizeCanvas(canvas, maxDpr = 1.5, maxPixels = 1_800_000) {
  const rect = canvas.getBoundingClientRect();
  const area = Math.max(1, rect.width * rect.height);
  const dpr = Math.min(window.devicePixelRatio || 1, maxDpr,
    Math.sqrt(maxPixels / area));
  const width = Math.max(1, Math.round(rect.width * dpr));
  const height = Math.max(1, Math.round(rect.height * dpr));
  const changed = width !== canvas.width || height !== canvas.height;
  if (changed) { canvas.width = width; canvas.height = height; }
  return { changed, width, height, dpr };
}
```

### Example G — visibility-aware effect mounting

```js
const sceneHost = document.querySelector("[data-scene]");
let scene = null;
const observer = new IntersectionObserver(async ([entry]) => {
  if (entry.isIntersecting && scene === null) {
    const { mountScene } = await import("./effects/scene.js");
    scene = await mountScene(sceneHost);
  }
  scene?.setVisible(entry.isIntersecting);
});
observer.observe(sceneHost);
// On removal: observer.disconnect(); scene?.destroy();
```

Guard asynchronous imports against unmount with an AbortController or generation token; otherwise mount may complete after the route disappears.

### Example H — clipboard behavior

```js
async function copyValue(button, value) {
  const original = button.textContent;
  try {
    await navigator.clipboard.writeText(value);
    button.textContent = "Copied";
  } catch {
    button.textContent = "Unable to copy";
  }
  setTimeout(() => { if (button.isConnected) button.textContent = original; }, 1200);
}
```

### Example I — pointer interpolation

```js
let goalX = 0, goalY = 0, smoothX = 0, smoothY = 0;
hero.addEventListener("pointermove", e => {
  const r = hero.getBoundingClientRect();
  goalX = (e.clientX - r.left) / r.width * 2 - 1;
  goalY = (e.clientY - r.top) / r.height * 2 - 1;
}, { passive: true });
function updatePointer(dt) {
  const follow = 1 - Math.exp(-8 * dt);
  smoothX += (goalX - smoothX) * follow;
  smoothY += (goalY - smoothY) * follow;
  // Send smoothX/smoothY to a transform or shader uniform.
}
```

### Example J — minimal WebGL2 fullscreen shader

```js
const vertex = `#version 300 es
in vec2 a_position;
void main() { gl_Position = vec4(a_position, 0.0, 1.0); }`;
const fragment = `#version 300 es
precision mediump float;
uniform vec2 u_resolution;
uniform float u_time;
out vec4 outColor;
void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;
  float wave = 0.5 + 0.5 * sin(uv.x * 5.0 + u_time * 0.25);
  outColor = vec4(mix(vec3(0.05,0.08,0.11), vec3(0.16,0.28,0.36), wave), 1.0);
}`;
// Compile/link programs ONCE, cache attribute locations and uniforms,
// upload a single fullscreen triangle, then draw in a lifecycle-managed loop.
```

### Example K — focus and dialog contract

```js
function openOverlay(dialog, trigger) {
  if (typeof dialog.showModal !== "function") return;
  dialog.showModal();
  dialog.addEventListener("close", () => trigger?.focus(), { once: true });
}
```

### Example L — smoke test matrix

```text
Desktop 1440x900: header, hero, transitions, copy feedback.
Phone 390x844: no sideways scroll, safe area, touch interactions.
Small phone 320x568: heading wrap, nav availability, no clipped CTA.
Keyboard: tab through every action, Enter/Space, Escape.
Reduced motion: no looping canvas effects or animated route movement.
Hidden tab: active animation frames stop and don't jump on resume.
Slow CPU: transitions are cancellable and inputs remain responsive.
No WebGL: fallback art and all useful controls remain available.
Repeated navigation x20: no growth in event listeners or WebGL contexts.
Network failure: deferred visual assets fail without breaking content.
```

## Choosing effects

| Requirement | Preferred approach | Upgrade trigger | Fallback |
| --- | --- | --- | --- |
| button response | CSS transform and opacity | complex interaction state | instant style |
| scroll reveal | IntersectionObserver and CSS | linked sections | static text |
| mobile menu | CSS/WAAPI | complex stagger | immediate show |
| ambient gradient | CSS | unique procedural field | solid color |
| grain | static texture | tightly controlled procedural art | none |
| hero 3D | pre-rendered asset | real interaction/rotation | image |
| shader transition | CSS mask | branded visual transition | fade |
| ranked objects | data + semantic buttons | 3D preview matters | 2D image |

## Detailed playbook: implementation recipes

Each recipe is a practical implementation and verification contract. Choose only the recipes relevant to the user request; **do not implement all effects at once**.

### Recipe 01 — Visual thesis

**Goal:** Establish one precise visual concept.
**Primary method:** a single coherent design metaphor.
**Where it applies:** hero and section composition.
**Implementation idea:** brand-specific focal point.
**Do not:** marketing wallpaper.
**Acceptance test:** still frame retains a deliberate identity.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to hero and section composition; for this recipe prioritize brand-specific focal point.
2. Define an observable result and list the files or modules that own it. Apply it to hero and section composition; for this recipe prioritize a single coherent design metaphor.
3. Implement the smallest complete accessible version first. Apply it to hero and section composition; for this recipe prioritize brand-specific focal point.
4. Add quality and motion only after functional behavior is correct. Apply it to hero and section composition; for this recipe prioritize a single coherent design metaphor.
5. Instrument, test, and refine against the stated budget. Apply it to hero and section composition; for this recipe prioritize brand-specific focal point.
6. Document a fallback, cleanup path, and operational limitation. Apply it to hero and section composition; for this recipe prioritize a single coherent design metaphor.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply a single coherent design metaphor only to the necessary component boundaries.
- Preserve brand-specific focal point across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: marketing wallpaper.
- Product requirement: Establish one precise visual concept.
- Focus area: hero and section composition.
- Technique: a single coherent design metaphor.
- Visible result: still frame retains a deliberate identity.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of hero and section composition and save a baseline screenshot.
2. Apply brand-specific focal point with actual user-visible data rather than filler.
3. Test a single coherent design metaphor at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for hero and section composition.
5. Disable JavaScript or graphics support and check the fallback for hero and section composition.
6. Verify using the observable condition: still frame retains a deliberate identity.
7. Record any remaining limitation and associated measured cost.


### Recipe 02 — Information hierarchy

**Goal:** Make information immediately scannable.
**Primary method:** clear typographic scale.
**Where it applies:** hero headline.
**Implementation idea:** one primary and one secondary action.
**Do not:** equal-sized headings.
**Acceptance test:** user can identify primary action without scrolling.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to hero headline; for this recipe prioritize one primary and one secondary action.
2. Define an observable result and list the files or modules that own it. Apply it to hero headline; for this recipe prioritize clear typographic scale.
3. Implement the smallest complete accessible version first. Apply it to hero headline; for this recipe prioritize one primary and one secondary action.
4. Add quality and motion only after functional behavior is correct. Apply it to hero headline; for this recipe prioritize clear typographic scale.
5. Instrument, test, and refine against the stated budget. Apply it to hero headline; for this recipe prioritize one primary and one secondary action.
6. Document a fallback, cleanup path, and operational limitation. Apply it to hero headline; for this recipe prioritize clear typographic scale.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply clear typographic scale only to the necessary component boundaries.
- Preserve one primary and one secondary action across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: equal-sized headings.
- Product requirement: Make information immediately scannable.
- Focus area: hero headline.
- Technique: clear typographic scale.
- Visible result: user can identify primary action without scrolling.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of hero headline and save a baseline screenshot.
2. Apply one primary and one secondary action with actual user-visible data rather than filler.
3. Test clear typographic scale at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for hero headline.
5. Disable JavaScript or graphics support and check the fallback for hero headline.
6. Verify using the observable condition: user can identify primary action without scrolling.
7. Record any remaining limitation and associated measured cost.


### Recipe 03 — Type scale

**Goal:** Create expressive but legible type.
**Primary method:** clamp() type scale.
**Where it applies:** headings and labels.
**Implementation idea:** optical line length and controlled tracking.
**Do not:** arbitrary giant fonts.
**Acceptance test:** no clipping at 320px.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to headings and labels; for this recipe prioritize optical line length and controlled tracking.
2. Define an observable result and list the files or modules that own it. Apply it to headings and labels; for this recipe prioritize clamp() type scale.
3. Implement the smallest complete accessible version first. Apply it to headings and labels; for this recipe prioritize optical line length and controlled tracking.
4. Add quality and motion only after functional behavior is correct. Apply it to headings and labels; for this recipe prioritize clamp() type scale.
5. Instrument, test, and refine against the stated budget. Apply it to headings and labels; for this recipe prioritize optical line length and controlled tracking.
6. Document a fallback, cleanup path, and operational limitation. Apply it to headings and labels; for this recipe prioritize clamp() type scale.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply clamp() type scale only to the necessary component boundaries.
- Preserve optical line length and controlled tracking across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: arbitrary giant fonts.
- Product requirement: Create expressive but legible type.
- Focus area: headings and labels.
- Technique: clamp() type scale.
- Visible result: no clipping at 320px.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of headings and labels and save a baseline screenshot.
2. Apply optical line length and controlled tracking with actual user-visible data rather than filler.
3. Test clamp() type scale at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for headings and labels.
5. Disable JavaScript or graphics support and check the fallback for headings and labels.
6. Verify using the observable condition: no clipping at 320px.
7. Record any remaining limitation and associated measured cost.


### Recipe 04 — Font loading

**Goal:** Avoid flashes and slow first render.
**Primary method:** font-display swap and local fallbacks.
**Where it applies:** all webfonts.
**Implementation idea:** subset glyph ranges where supported.
**Do not:** blocking imported fonts.
**Acceptance test:** readable first paint offline.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all webfonts; for this recipe prioritize subset glyph ranges where supported.
2. Define an observable result and list the files or modules that own it. Apply it to all webfonts; for this recipe prioritize font-display swap and local fallbacks.
3. Implement the smallest complete accessible version first. Apply it to all webfonts; for this recipe prioritize subset glyph ranges where supported.
4. Add quality and motion only after functional behavior is correct. Apply it to all webfonts; for this recipe prioritize font-display swap and local fallbacks.
5. Instrument, test, and refine against the stated budget. Apply it to all webfonts; for this recipe prioritize subset glyph ranges where supported.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all webfonts; for this recipe prioritize font-display swap and local fallbacks.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply font-display swap and local fallbacks only to the necessary component boundaries.
- Preserve subset glyph ranges where supported across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: blocking imported fonts.
- Product requirement: Avoid flashes and slow first render.
- Focus area: all webfonts.
- Technique: font-display swap and local fallbacks.
- Visible result: readable first paint offline.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of all webfonts and save a baseline screenshot.
2. Apply subset glyph ranges where supported with actual user-visible data rather than filler.
3. Test font-display swap and local fallbacks at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all webfonts.
5. Disable JavaScript or graphics support and check the fallback for all webfonts.
6. Verify using the observable condition: readable first paint offline.
7. Record any remaining limitation and associated measured cost.


### Recipe 05 — Copy economy

**Goal:** Cut unnecessary AI-generated prose.
**Primary method:** content inventory.
**Where it applies:** all content sections.
**Implementation idea:** specific user-facing facts.
**Do not:** filler slogans.
**Acceptance test:** each sentence answers an actual user question.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all content sections; for this recipe prioritize specific user-facing facts.
2. Define an observable result and list the files or modules that own it. Apply it to all content sections; for this recipe prioritize content inventory.
3. Implement the smallest complete accessible version first. Apply it to all content sections; for this recipe prioritize specific user-facing facts.
4. Add quality and motion only after functional behavior is correct. Apply it to all content sections; for this recipe prioritize content inventory.
5. Instrument, test, and refine against the stated budget. Apply it to all content sections; for this recipe prioritize specific user-facing facts.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all content sections; for this recipe prioritize content inventory.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply content inventory only to the necessary component boundaries.
- Preserve specific user-facing facts across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: filler slogans.
- Product requirement: Cut unnecessary AI-generated prose.
- Focus area: all content sections.
- Technique: content inventory.
- Visible result: each sentence answers an actual user question.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of all content sections and save a baseline screenshot.
2. Apply specific user-facing facts with actual user-visible data rather than filler.
3. Test content inventory at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all content sections.
5. Disable JavaScript or graphics support and check the fallback for all content sections.
6. Verify using the observable condition: each sentence answers an actual user question.
7. Record any remaining limitation and associated measured cost.


### Recipe 06 — Grid rhythm

**Goal:** Use alignment to create distinction.
**Primary method:** CSS grid with named areas.
**Where it applies:** hero and secondary sections.
**Implementation idea:** intentional asymmetric composition.
**Do not:** generic card matrix.
**Acceptance test:** alignment holds at each breakpoint.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to hero and secondary sections; for this recipe prioritize intentional asymmetric composition.
2. Define an observable result and list the files or modules that own it. Apply it to hero and secondary sections; for this recipe prioritize CSS grid with named areas.
3. Implement the smallest complete accessible version first. Apply it to hero and secondary sections; for this recipe prioritize intentional asymmetric composition.
4. Add quality and motion only after functional behavior is correct. Apply it to hero and secondary sections; for this recipe prioritize CSS grid with named areas.
5. Instrument, test, and refine against the stated budget. Apply it to hero and secondary sections; for this recipe prioritize intentional asymmetric composition.
6. Document a fallback, cleanup path, and operational limitation. Apply it to hero and secondary sections; for this recipe prioritize CSS grid with named areas.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply CSS grid with named areas only to the necessary component boundaries.
- Preserve intentional asymmetric composition across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: generic card matrix.
- Product requirement: Use alignment to create distinction.
- Focus area: hero and secondary sections.
- Technique: CSS grid with named areas.
- Visible result: alignment holds at each breakpoint.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of hero and secondary sections and save a baseline screenshot.
2. Apply intentional asymmetric composition with actual user-visible data rather than filler.
3. Test CSS grid with named areas at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for hero and secondary sections.
5. Disable JavaScript or graphics support and check the fallback for hero and secondary sections.
6. Verify using the observable condition: alignment holds at each breakpoint.
7. Record any remaining limitation and associated measured cost.


### Recipe 07 — Negative space

**Goal:** Use whitespace for hierarchy.
**Primary method:** spacing tokens.
**Where it applies:** all page regions.
**Implementation idea:** breathing room around focal content.
**Do not:** empty padding without intent.
**Acceptance test:** content remains balanced.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all page regions; for this recipe prioritize breathing room around focal content.
2. Define an observable result and list the files or modules that own it. Apply it to all page regions; for this recipe prioritize spacing tokens.
3. Implement the smallest complete accessible version first. Apply it to all page regions; for this recipe prioritize breathing room around focal content.
4. Add quality and motion only after functional behavior is correct. Apply it to all page regions; for this recipe prioritize spacing tokens.
5. Instrument, test, and refine against the stated budget. Apply it to all page regions; for this recipe prioritize breathing room around focal content.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all page regions; for this recipe prioritize spacing tokens.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply spacing tokens only to the necessary component boundaries.
- Preserve breathing room around focal content across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: empty padding without intent.
- Product requirement: Use whitespace for hierarchy.
- Focus area: all page regions.
- Technique: spacing tokens.
- Visible result: content remains balanced.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of all page regions and save a baseline screenshot.
2. Apply breathing room around focal content with actual user-visible data rather than filler.
3. Test spacing tokens at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all page regions.
5. Disable JavaScript or graphics support and check the fallback for all page regions.
6. Verify using the observable condition: content remains balanced.
7. Record any remaining limitation and associated measured cost.


### Recipe 08 — Color tokens

**Goal:** Limit accents and protect contrast.
**Primary method:** CSS custom properties.
**Where it applies:** global theme.
**Implementation idea:** purposeful surface and text roles.
**Do not:** random neon gradient.
**Acceptance test:** text contrast checked.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to global theme; for this recipe prioritize purposeful surface and text roles.
2. Define an observable result and list the files or modules that own it. Apply it to global theme; for this recipe prioritize CSS custom properties.
3. Implement the smallest complete accessible version first. Apply it to global theme; for this recipe prioritize purposeful surface and text roles.
4. Add quality and motion only after functional behavior is correct. Apply it to global theme; for this recipe prioritize CSS custom properties.
5. Instrument, test, and refine against the stated budget. Apply it to global theme; for this recipe prioritize purposeful surface and text roles.
6. Document a fallback, cleanup path, and operational limitation. Apply it to global theme; for this recipe prioritize CSS custom properties.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply CSS custom properties only to the necessary component boundaries.
- Preserve purposeful surface and text roles across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: random neon gradient.
- Product requirement: Limit accents and protect contrast.
- Focus area: global theme.
- Technique: CSS custom properties.
- Visible result: text contrast checked.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of global theme and save a baseline screenshot.
2. Apply purposeful surface and text roles with actual user-visible data rather than filler.
3. Test CSS custom properties at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for global theme.
5. Disable JavaScript or graphics support and check the fallback for global theme.
6. Verify using the observable condition: text contrast checked.
7. Record any remaining limitation and associated measured cost.


### Recipe 09 — Surface layers

**Goal:** Build depth without glass everywhere.
**Primary method:** solid layers and subtle borders.
**Where it applies:** navigation and panels.
**Implementation idea:** restrained opacity separation.
**Do not:** pervasive blur.
**Acceptance test:** low-end GPU avoids blur.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to navigation and panels; for this recipe prioritize restrained opacity separation.
2. Define an observable result and list the files or modules that own it. Apply it to navigation and panels; for this recipe prioritize solid layers and subtle borders.
3. Implement the smallest complete accessible version first. Apply it to navigation and panels; for this recipe prioritize restrained opacity separation.
4. Add quality and motion only after functional behavior is correct. Apply it to navigation and panels; for this recipe prioritize solid layers and subtle borders.
5. Instrument, test, and refine against the stated budget. Apply it to navigation and panels; for this recipe prioritize restrained opacity separation.
6. Document a fallback, cleanup path, and operational limitation. Apply it to navigation and panels; for this recipe prioritize solid layers and subtle borders.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply solid layers and subtle borders only to the necessary component boundaries.
- Preserve restrained opacity separation across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: pervasive blur.
- Product requirement: Build depth without glass everywhere.
- Focus area: navigation and panels.
- Technique: solid layers and subtle borders.
- Visible result: low-end GPU avoids blur.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of navigation and panels and save a baseline screenshot.
2. Apply restrained opacity separation with actual user-visible data rather than filler.
3. Test solid layers and subtle borders at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for navigation and panels.
5. Disable JavaScript or graphics support and check the fallback for navigation and panels.
6. Verify using the observable condition: low-end GPU avoids blur.
7. Record any remaining limitation and associated measured cost.


### Recipe 10 — Section separators

**Goal:** Prevent unnecessary borders.
**Primary method:** whitespace and alignment.
**Where it applies:** section transitions.
**Implementation idea:** one or two justified separators.
**Do not:** rules everywhere.
**Acceptance test:** sections stay distinct without clutter.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to section transitions; for this recipe prioritize one or two justified separators.
2. Define an observable result and list the files or modules that own it. Apply it to section transitions; for this recipe prioritize whitespace and alignment.
3. Implement the smallest complete accessible version first. Apply it to section transitions; for this recipe prioritize one or two justified separators.
4. Add quality and motion only after functional behavior is correct. Apply it to section transitions; for this recipe prioritize whitespace and alignment.
5. Instrument, test, and refine against the stated budget. Apply it to section transitions; for this recipe prioritize one or two justified separators.
6. Document a fallback, cleanup path, and operational limitation. Apply it to section transitions; for this recipe prioritize whitespace and alignment.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply whitespace and alignment only to the necessary component boundaries.
- Preserve one or two justified separators across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: rules everywhere.
- Product requirement: Prevent unnecessary borders.
- Focus area: section transitions.
- Technique: whitespace and alignment.
- Visible result: sections stay distinct without clutter.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of section transitions and save a baseline screenshot.
2. Apply one or two justified separators with actual user-visible data rather than filler.
3. Test whitespace and alignment at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for section transitions.
5. Disable JavaScript or graphics support and check the fallback for section transitions.
6. Verify using the observable condition: sections stay distinct without clutter.
7. Record any remaining limitation and associated measured cost.


### Recipe 11 — Navigation contract

**Goal:** Ensure predictable route changes.
**Primary method:** single route state owner.
**Where it applies:** all navigation.
**Implementation idea:** cancel-safe state machine.
**Do not:** multiple route controllers.
**Acceptance test:** rapid clicks never corrupt state.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all navigation; for this recipe prioritize cancel-safe state machine.
2. Define an observable result and list the files or modules that own it. Apply it to all navigation; for this recipe prioritize single route state owner.
3. Implement the smallest complete accessible version first. Apply it to all navigation; for this recipe prioritize cancel-safe state machine.
4. Add quality and motion only after functional behavior is correct. Apply it to all navigation; for this recipe prioritize single route state owner.
5. Instrument, test, and refine against the stated budget. Apply it to all navigation; for this recipe prioritize cancel-safe state machine.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all navigation; for this recipe prioritize single route state owner.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply single route state owner only to the necessary component boundaries.
- Preserve cancel-safe state machine across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: multiple route controllers.
- Product requirement: Ensure predictable route changes.
- Focus area: all navigation.
- Technique: single route state owner.
- Visible result: rapid clicks never corrupt state.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of all navigation and save a baseline screenshot.
2. Apply cancel-safe state machine with actual user-visible data rather than filler.
3. Test single route state owner at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all navigation.
5. Disable JavaScript or graphics support and check the fallback for all navigation.
6. Verify using the observable condition: rapid clicks never corrupt state.
7. Record any remaining limitation and associated measured cost.


### Recipe 12 — Page exit choreography

**Goal:** Maintain context during navigation.
**Primary method:** opacity and transform exit.
**Where it applies:** route switch.
**Implementation idea:** clear departing focus.
**Do not:** hard jump or arbitrary sleep.
**Acceptance test:** no doubled page content.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to route switch; for this recipe prioritize clear departing focus.
2. Define an observable result and list the files or modules that own it. Apply it to route switch; for this recipe prioritize opacity and transform exit.
3. Implement the smallest complete accessible version first. Apply it to route switch; for this recipe prioritize clear departing focus.
4. Add quality and motion only after functional behavior is correct. Apply it to route switch; for this recipe prioritize opacity and transform exit.
5. Instrument, test, and refine against the stated budget. Apply it to route switch; for this recipe prioritize clear departing focus.
6. Document a fallback, cleanup path, and operational limitation. Apply it to route switch; for this recipe prioritize opacity and transform exit.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply opacity and transform exit only to the necessary component boundaries.
- Preserve clear departing focus across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: hard jump or arbitrary sleep.
- Product requirement: Maintain context during navigation.
- Focus area: route switch.
- Technique: opacity and transform exit.
- Visible result: no doubled page content.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of route switch and save a baseline screenshot.
2. Apply clear departing focus with actual user-visible data rather than filler.
3. Test opacity and transform exit at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for route switch.
5. Disable JavaScript or graphics support and check the fallback for route switch.
6. Verify using the observable condition: no doubled page content.
7. Record any remaining limitation and associated measured cost.


### Recipe 13 — Page enter choreography

**Goal:** Introduce hierarchy gradually.
**Primary method:** sequenced WAAPI.
**Where it applies:** route switch.
**Implementation idea:** hero visual then copy then controls.
**Do not:** all nodes pop simultaneously.
**Acceptance test:** entrance completes under reduced motion.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to route switch; for this recipe prioritize hero visual then copy then controls.
2. Define an observable result and list the files or modules that own it. Apply it to route switch; for this recipe prioritize sequenced WAAPI.
3. Implement the smallest complete accessible version first. Apply it to route switch; for this recipe prioritize hero visual then copy then controls.
4. Add quality and motion only after functional behavior is correct. Apply it to route switch; for this recipe prioritize sequenced WAAPI.
5. Instrument, test, and refine against the stated budget. Apply it to route switch; for this recipe prioritize hero visual then copy then controls.
6. Document a fallback, cleanup path, and operational limitation. Apply it to route switch; for this recipe prioritize sequenced WAAPI.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply sequenced WAAPI only to the necessary component boundaries.
- Preserve hero visual then copy then controls across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: all nodes pop simultaneously.
- Product requirement: Introduce hierarchy gradually.
- Focus area: route switch.
- Technique: sequenced WAAPI.
- Visible result: entrance completes under reduced motion.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of route switch and save a baseline screenshot.
2. Apply hero visual then copy then controls with actual user-visible data rather than filler.
3. Test sequenced WAAPI at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for route switch.
5. Disable JavaScript or graphics support and check the fallback for route switch.
6. Verify using the observable condition: entrance completes under reduced motion.
7. Record any remaining limitation and associated measured cost.


### Recipe 14 — Tab transitions

**Goal:** Move selection organically.
**Primary method:** shared indicator transform.
**Where it applies:** tabs and dock.
**Implementation idea:** one continuous indicator.
**Do not:** instant underline teleport.
**Acceptance test:** selection remains clear with keyboard.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to tabs and dock; for this recipe prioritize one continuous indicator.
2. Define an observable result and list the files or modules that own it. Apply it to tabs and dock; for this recipe prioritize shared indicator transform.
3. Implement the smallest complete accessible version first. Apply it to tabs and dock; for this recipe prioritize one continuous indicator.
4. Add quality and motion only after functional behavior is correct. Apply it to tabs and dock; for this recipe prioritize shared indicator transform.
5. Instrument, test, and refine against the stated budget. Apply it to tabs and dock; for this recipe prioritize one continuous indicator.
6. Document a fallback, cleanup path, and operational limitation. Apply it to tabs and dock; for this recipe prioritize shared indicator transform.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply shared indicator transform only to the necessary component boundaries.
- Preserve one continuous indicator across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: instant underline teleport.
- Product requirement: Move selection organically.
- Focus area: tabs and dock.
- Technique: shared indicator transform.
- Visible result: selection remains clear with keyboard.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of tabs and dock and save a baseline screenshot.
2. Apply one continuous indicator with actual user-visible data rather than filler.
3. Test shared indicator transform at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for tabs and dock.
5. Disable JavaScript or graphics support and check the fallback for tabs and dock.
6. Verify using the observable condition: selection remains clear with keyboard.
7. Record any remaining limitation and associated measured cost.


### Recipe 15 — Menu animation

**Goal:** Keep menus responsive.
**Primary method:** short transform and opacity.
**Where it applies:** mobile menu.
**Implementation idea:** focus transfer and escape.
**Do not:** huge layout transitions.
**Acceptance test:** menu never traps background clicks.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to mobile menu; for this recipe prioritize focus transfer and escape.
2. Define an observable result and list the files or modules that own it. Apply it to mobile menu; for this recipe prioritize short transform and opacity.
3. Implement the smallest complete accessible version first. Apply it to mobile menu; for this recipe prioritize focus transfer and escape.
4. Add quality and motion only after functional behavior is correct. Apply it to mobile menu; for this recipe prioritize short transform and opacity.
5. Instrument, test, and refine against the stated budget. Apply it to mobile menu; for this recipe prioritize focus transfer and escape.
6. Document a fallback, cleanup path, and operational limitation. Apply it to mobile menu; for this recipe prioritize short transform and opacity.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply short transform and opacity only to the necessary component boundaries.
- Preserve focus transfer and escape across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: huge layout transitions.
- Product requirement: Keep menus responsive.
- Focus area: mobile menu.
- Technique: short transform and opacity.
- Visible result: menu never traps background clicks.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of mobile menu and save a baseline screenshot.
2. Apply focus transfer and escape with actual user-visible data rather than filler.
3. Test short transform and opacity at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for mobile menu.
5. Disable JavaScript or graphics support and check the fallback for mobile menu.
6. Verify using the observable condition: menu never traps background clicks.
7. Record any remaining limitation and associated measured cost.


### Recipe 16 — Hover discipline

**Goal:** Provide subtle intention feedback.
**Primary method:** transform and opacity.
**Where it applies:** links and buttons.
**Implementation idea:** small deltas and instant response.
**Do not:** dramatic enlargement.
**Acceptance test:** no layout shift.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to links and buttons; for this recipe prioritize small deltas and instant response.
2. Define an observable result and list the files or modules that own it. Apply it to links and buttons; for this recipe prioritize transform and opacity.
3. Implement the smallest complete accessible version first. Apply it to links and buttons; for this recipe prioritize small deltas and instant response.
4. Add quality and motion only after functional behavior is correct. Apply it to links and buttons; for this recipe prioritize transform and opacity.
5. Instrument, test, and refine against the stated budget. Apply it to links and buttons; for this recipe prioritize small deltas and instant response.
6. Document a fallback, cleanup path, and operational limitation. Apply it to links and buttons; for this recipe prioritize transform and opacity.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply transform and opacity only to the necessary component boundaries.
- Preserve small deltas and instant response across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: dramatic enlargement.
- Product requirement: Provide subtle intention feedback.
- Focus area: links and buttons.
- Technique: transform and opacity.
- Visible result: no layout shift.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of links and buttons and save a baseline screenshot.
2. Apply small deltas and instant response with actual user-visible data rather than filler.
3. Test transform and opacity at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for links and buttons.
5. Disable JavaScript or graphics support and check the fallback for links and buttons.
6. Verify using the observable condition: no layout shift.
7. Record any remaining limitation and associated measured cost.


### Recipe 17 — Focus design

**Goal:** Show keyboard target clearly.
**Primary method:** focus-visible outline.
**Where it applies:** interactive controls.
**Implementation idea:** high-contrast offset outline.
**Do not:** outline:none without replacement.
**Acceptance test:** keyboard path visible.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to interactive controls; for this recipe prioritize high-contrast offset outline.
2. Define an observable result and list the files or modules that own it. Apply it to interactive controls; for this recipe prioritize focus-visible outline.
3. Implement the smallest complete accessible version first. Apply it to interactive controls; for this recipe prioritize high-contrast offset outline.
4. Add quality and motion only after functional behavior is correct. Apply it to interactive controls; for this recipe prioritize focus-visible outline.
5. Instrument, test, and refine against the stated budget. Apply it to interactive controls; for this recipe prioritize high-contrast offset outline.
6. Document a fallback, cleanup path, and operational limitation. Apply it to interactive controls; for this recipe prioritize focus-visible outline.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply focus-visible outline only to the necessary component boundaries.
- Preserve high-contrast offset outline across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: outline:none without replacement.
- Product requirement: Show keyboard target clearly.
- Focus area: interactive controls.
- Technique: focus-visible outline.
- Visible result: keyboard path visible.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of interactive controls and save a baseline screenshot.
2. Apply high-contrast offset outline with actual user-visible data rather than filler.
3. Test focus-visible outline at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for interactive controls.
5. Disable JavaScript or graphics support and check the fallback for interactive controls.
6. Verify using the observable condition: keyboard path visible.
7. Record any remaining limitation and associated measured cost.


### Recipe 18 — Pressed state

**Goal:** Acknowledge input within a frame.
**Primary method:** active scale or translation.
**Where it applies:** buttons.
**Implementation idea:** small reversible response.
**Do not:** complex delayed effect.
**Acceptance test:** response feels instantaneous.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to buttons; for this recipe prioritize small reversible response.
2. Define an observable result and list the files or modules that own it. Apply it to buttons; for this recipe prioritize active scale or translation.
3. Implement the smallest complete accessible version first. Apply it to buttons; for this recipe prioritize small reversible response.
4. Add quality and motion only after functional behavior is correct. Apply it to buttons; for this recipe prioritize active scale or translation.
5. Instrument, test, and refine against the stated budget. Apply it to buttons; for this recipe prioritize small reversible response.
6. Document a fallback, cleanup path, and operational limitation. Apply it to buttons; for this recipe prioritize active scale or translation.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply active scale or translation only to the necessary component boundaries.
- Preserve small reversible response across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: complex delayed effect.
- Product requirement: Acknowledge input within a frame.
- Focus area: buttons.
- Technique: active scale or translation.
- Visible result: response feels instantaneous.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of buttons and save a baseline screenshot.
2. Apply small reversible response with actual user-visible data rather than filler.
3. Test active scale or translation at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for buttons.
5. Disable JavaScript or graphics support and check the fallback for buttons.
6. Verify using the observable condition: response feels instantaneous.
7. Record any remaining limitation and associated measured cost.


### Recipe 19 — Loading feedback

**Goal:** Represent real progress.
**Primary method:** skeleton or compact spinner.
**Where it applies:** async tasks.
**Implementation idea:** only when work exists.
**Do not:** fake loading animation.
**Acceptance test:** completion and errors visible.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to async tasks; for this recipe prioritize only when work exists.
2. Define an observable result and list the files or modules that own it. Apply it to async tasks; for this recipe prioritize skeleton or compact spinner.
3. Implement the smallest complete accessible version first. Apply it to async tasks; for this recipe prioritize only when work exists.
4. Add quality and motion only after functional behavior is correct. Apply it to async tasks; for this recipe prioritize skeleton or compact spinner.
5. Instrument, test, and refine against the stated budget. Apply it to async tasks; for this recipe prioritize only when work exists.
6. Document a fallback, cleanup path, and operational limitation. Apply it to async tasks; for this recipe prioritize skeleton or compact spinner.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply skeleton or compact spinner only to the necessary component boundaries.
- Preserve only when work exists across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: fake loading animation.
- Product requirement: Represent real progress.
- Focus area: async tasks.
- Technique: skeleton or compact spinner.
- Visible result: completion and errors visible.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of async tasks and save a baseline screenshot.
2. Apply only when work exists with actual user-visible data rather than filler.
3. Test skeleton or compact spinner at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for async tasks.
5. Disable JavaScript or graphics support and check the fallback for async tasks.
6. Verify using the observable condition: completion and errors visible.
7. Record any remaining limitation and associated measured cost.


### Recipe 20 — Error messages

**Goal:** Explain failure without UI clutter.
**Primary method:** inline semantic status.
**Where it applies:** forms and requests.
**Implementation idea:** specific actionable message.
**Do not:** generic something went wrong.
**Acceptance test:** screen reader announces error.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to forms and requests; for this recipe prioritize specific actionable message.
2. Define an observable result and list the files or modules that own it. Apply it to forms and requests; for this recipe prioritize inline semantic status.
3. Implement the smallest complete accessible version first. Apply it to forms and requests; for this recipe prioritize specific actionable message.
4. Add quality and motion only after functional behavior is correct. Apply it to forms and requests; for this recipe prioritize inline semantic status.
5. Instrument, test, and refine against the stated budget. Apply it to forms and requests; for this recipe prioritize specific actionable message.
6. Document a fallback, cleanup path, and operational limitation. Apply it to forms and requests; for this recipe prioritize inline semantic status.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply inline semantic status only to the necessary component boundaries.
- Preserve specific actionable message across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: generic something went wrong.
- Product requirement: Explain failure without UI clutter.
- Focus area: forms and requests.
- Technique: inline semantic status.
- Visible result: screen reader announces error.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of forms and requests and save a baseline screenshot.
2. Apply specific actionable message with actual user-visible data rather than filler.
3. Test inline semantic status at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for forms and requests.
5. Disable JavaScript or graphics support and check the fallback for forms and requests.
6. Verify using the observable condition: screen reader announces error.
7. Record any remaining limitation and associated measured cost.


### Recipe 21 — Copy action

**Goal:** Copy real content and confirm it.
**Primary method:** Clipboard API with failure handling.
**Where it applies:** IP address and commands.
**Implementation idea:** brief live status.
**Do not:** silent no-op.
**Acceptance test:** user knows copied or blocked.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to IP address and commands; for this recipe prioritize brief live status.
2. Define an observable result and list the files or modules that own it. Apply it to IP address and commands; for this recipe prioritize Clipboard API with failure handling.
3. Implement the smallest complete accessible version first. Apply it to IP address and commands; for this recipe prioritize brief live status.
4. Add quality and motion only after functional behavior is correct. Apply it to IP address and commands; for this recipe prioritize Clipboard API with failure handling.
5. Instrument, test, and refine against the stated budget. Apply it to IP address and commands; for this recipe prioritize brief live status.
6. Document a fallback, cleanup path, and operational limitation. Apply it to IP address and commands; for this recipe prioritize Clipboard API with failure handling.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply Clipboard API with failure handling only to the necessary component boundaries.
- Preserve brief live status across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: silent no-op.
- Product requirement: Copy real content and confirm it.
- Focus area: IP address and commands.
- Technique: Clipboard API with failure handling.
- Visible result: user knows copied or blocked.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of IP address and commands and save a baseline screenshot.
2. Apply brief live status with actual user-visible data rather than filler.
3. Test Clipboard API with failure handling at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for IP address and commands.
5. Disable JavaScript or graphics support and check the fallback for IP address and commands.
6. Verify using the observable condition: user knows copied or blocked.
7. Record any remaining limitation and associated measured cost.


### Recipe 22 — Reduced motion

**Goal:** Honor motion accessibility.
**Primary method:** matchMedia and lifecycle state.
**Where it applies:** all animation systems.
**Implementation idea:** static stable layout.
**Do not:** CSS-only partial support.
**Acceptance test:** canvas and JS also stop.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all animation systems; for this recipe prioritize static stable layout.
2. Define an observable result and list the files or modules that own it. Apply it to all animation systems; for this recipe prioritize matchMedia and lifecycle state.
3. Implement the smallest complete accessible version first. Apply it to all animation systems; for this recipe prioritize static stable layout.
4. Add quality and motion only after functional behavior is correct. Apply it to all animation systems; for this recipe prioritize matchMedia and lifecycle state.
5. Instrument, test, and refine against the stated budget. Apply it to all animation systems; for this recipe prioritize static stable layout.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all animation systems; for this recipe prioritize matchMedia and lifecycle state.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply matchMedia and lifecycle state only to the necessary component boundaries.
- Preserve static stable layout across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: CSS-only partial support.
- Product requirement: Honor motion accessibility.
- Focus area: all animation systems.
- Technique: matchMedia and lifecycle state.
- Visible result: canvas and JS also stop.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of all animation systems and save a baseline screenshot.
2. Apply static stable layout with actual user-visible data rather than filler.
3. Test matchMedia and lifecycle state at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all animation systems.
5. Disable JavaScript or graphics support and check the fallback for all animation systems.
6. Verify using the observable condition: canvas and JS also stop.
7. Record any remaining limitation and associated measured cost.


### Recipe 23 — Contrast system

**Goal:** Maintain useful visibility.
**Primary method:** WCAG-based inspection.
**Where it applies:** small labels and navigation.
**Implementation idea:** light on dark calibrated roles.
**Do not:** near-invisible gray.
**Acceptance test:** legible on mobile brightness.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to small labels and navigation; for this recipe prioritize light on dark calibrated roles.
2. Define an observable result and list the files or modules that own it. Apply it to small labels and navigation; for this recipe prioritize WCAG-based inspection.
3. Implement the smallest complete accessible version first. Apply it to small labels and navigation; for this recipe prioritize light on dark calibrated roles.
4. Add quality and motion only after functional behavior is correct. Apply it to small labels and navigation; for this recipe prioritize WCAG-based inspection.
5. Instrument, test, and refine against the stated budget. Apply it to small labels and navigation; for this recipe prioritize light on dark calibrated roles.
6. Document a fallback, cleanup path, and operational limitation. Apply it to small labels and navigation; for this recipe prioritize WCAG-based inspection.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply WCAG-based inspection only to the necessary component boundaries.
- Preserve light on dark calibrated roles across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: near-invisible gray.
- Product requirement: Maintain useful visibility.
- Focus area: small labels and navigation.
- Technique: WCAG-based inspection.
- Visible result: legible on mobile brightness.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of small labels and navigation and save a baseline screenshot.
2. Apply light on dark calibrated roles with actual user-visible data rather than filler.
3. Test WCAG-based inspection at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for small labels and navigation.
5. Disable JavaScript or graphics support and check the fallback for small labels and navigation.
6. Verify using the observable condition: legible on mobile brightness.
7. Record any remaining limitation and associated measured cost.


### Recipe 24 — Responsive hero

**Goal:** Preserve composition on phones.
**Primary method:** container queries and clamp.
**Where it applies:** hero scene.
**Implementation idea:** reordered text and focal art.
**Do not:** desktop shrunk 50%.
**Acceptance test:** no horizontal overflow.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to hero scene; for this recipe prioritize reordered text and focal art.
2. Define an observable result and list the files or modules that own it. Apply it to hero scene; for this recipe prioritize container queries and clamp.
3. Implement the smallest complete accessible version first. Apply it to hero scene; for this recipe prioritize reordered text and focal art.
4. Add quality and motion only after functional behavior is correct. Apply it to hero scene; for this recipe prioritize container queries and clamp.
5. Instrument, test, and refine against the stated budget. Apply it to hero scene; for this recipe prioritize reordered text and focal art.
6. Document a fallback, cleanup path, and operational limitation. Apply it to hero scene; for this recipe prioritize container queries and clamp.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply container queries and clamp only to the necessary component boundaries.
- Preserve reordered text and focal art across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: desktop shrunk 50%.
- Product requirement: Preserve composition on phones.
- Focus area: hero scene.
- Technique: container queries and clamp.
- Visible result: no horizontal overflow.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of hero scene and save a baseline screenshot.
2. Apply reordered text and focal art with actual user-visible data rather than filler.
3. Test container queries and clamp at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for hero scene.
5. Disable JavaScript or graphics support and check the fallback for hero scene.
6. Verify using the observable condition: no horizontal overflow.
7. Record any remaining limitation and associated measured cost.


### Recipe 25 — Touch targets

**Goal:** Support comfortable interaction.
**Primary method:** minimum target dimensions.
**Where it applies:** mobile controls.
**Implementation idea:** spacing around adjacent buttons.
**Do not:** tiny icon clicks.
**Acceptance test:** all actions usable with thumb.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to mobile controls; for this recipe prioritize spacing around adjacent buttons.
2. Define an observable result and list the files or modules that own it. Apply it to mobile controls; for this recipe prioritize minimum target dimensions.
3. Implement the smallest complete accessible version first. Apply it to mobile controls; for this recipe prioritize spacing around adjacent buttons.
4. Add quality and motion only after functional behavior is correct. Apply it to mobile controls; for this recipe prioritize minimum target dimensions.
5. Instrument, test, and refine against the stated budget. Apply it to mobile controls; for this recipe prioritize spacing around adjacent buttons.
6. Document a fallback, cleanup path, and operational limitation. Apply it to mobile controls; for this recipe prioritize minimum target dimensions.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply minimum target dimensions only to the necessary component boundaries.
- Preserve spacing around adjacent buttons across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: tiny icon clicks.
- Product requirement: Support comfortable interaction.
- Focus area: mobile controls.
- Technique: minimum target dimensions.
- Visible result: all actions usable with thumb.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of mobile controls and save a baseline screenshot.
2. Apply spacing around adjacent buttons with actual user-visible data rather than filler.
3. Test minimum target dimensions at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for mobile controls.
5. Disable JavaScript or graphics support and check the fallback for mobile controls.
6. Verify using the observable condition: all actions usable with thumb.
7. Record any remaining limitation and associated measured cost.


### Recipe 26 — Safe-area layout

**Goal:** Respect phone insets.
**Primary method:** env(safe-area-inset-*).
**Where it applies:** sticky footer.
**Implementation idea:** padding on edges.
**Do not:** clipped CTA.
**Acceptance test:** CTA above gesture bar.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to sticky footer; for this recipe prioritize padding on edges.
2. Define an observable result and list the files or modules that own it. Apply it to sticky footer; for this recipe prioritize env(safe-area-inset-*).
3. Implement the smallest complete accessible version first. Apply it to sticky footer; for this recipe prioritize padding on edges.
4. Add quality and motion only after functional behavior is correct. Apply it to sticky footer; for this recipe prioritize env(safe-area-inset-*).
5. Instrument, test, and refine against the stated budget. Apply it to sticky footer; for this recipe prioritize padding on edges.
6. Document a fallback, cleanup path, and operational limitation. Apply it to sticky footer; for this recipe prioritize env(safe-area-inset-*).

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply env(safe-area-inset-*) only to the necessary component boundaries.
- Preserve padding on edges across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: clipped CTA.
- Product requirement: Respect phone insets.
- Focus area: sticky footer.
- Technique: env(safe-area-inset-*).
- Visible result: CTA above gesture bar.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of sticky footer and save a baseline screenshot.
2. Apply padding on edges with actual user-visible data rather than filler.
3. Test env(safe-area-inset-*) at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for sticky footer.
5. Disable JavaScript or graphics support and check the fallback for sticky footer.
6. Verify using the observable condition: CTA above gesture bar.
7. Record any remaining limitation and associated measured cost.


### Recipe 27 — Scroll behavior

**Goal:** Respect natural motion.
**Primary method:** native scrolling.
**Where it applies:** long pages.
**Implementation idea:** scroll-margin for anchors.
**Do not:** forced smooth scroll everywhere.
**Acceptance test:** anchor lands with heading visible.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to long pages; for this recipe prioritize scroll-margin for anchors.
2. Define an observable result and list the files or modules that own it. Apply it to long pages; for this recipe prioritize native scrolling.
3. Implement the smallest complete accessible version first. Apply it to long pages; for this recipe prioritize scroll-margin for anchors.
4. Add quality and motion only after functional behavior is correct. Apply it to long pages; for this recipe prioritize native scrolling.
5. Instrument, test, and refine against the stated budget. Apply it to long pages; for this recipe prioritize scroll-margin for anchors.
6. Document a fallback, cleanup path, and operational limitation. Apply it to long pages; for this recipe prioritize native scrolling.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply native scrolling only to the necessary component boundaries.
- Preserve scroll-margin for anchors across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: forced smooth scroll everywhere.
- Product requirement: Respect natural motion.
- Focus area: long pages.
- Technique: native scrolling.
- Visible result: anchor lands with heading visible.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of long pages and save a baseline screenshot.
2. Apply scroll-margin for anchors with actual user-visible data rather than filler.
3. Test native scrolling at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for long pages.
5. Disable JavaScript or graphics support and check the fallback for long pages.
6. Verify using the observable condition: anchor lands with heading visible.
7. Record any remaining limitation and associated measured cost.


### Recipe 28 — Sticky header

**Goal:** Keep context without obstruction.
**Primary method:** position sticky and compact state.
**Where it applies:** header.
**Implementation idea:** small height change.
**Do not:** overlaying content.
**Acceptance test:** first heading never hidden.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to header; for this recipe prioritize small height change.
2. Define an observable result and list the files or modules that own it. Apply it to header; for this recipe prioritize position sticky and compact state.
3. Implement the smallest complete accessible version first. Apply it to header; for this recipe prioritize small height change.
4. Add quality and motion only after functional behavior is correct. Apply it to header; for this recipe prioritize position sticky and compact state.
5. Instrument, test, and refine against the stated budget. Apply it to header; for this recipe prioritize small height change.
6. Document a fallback, cleanup path, and operational limitation. Apply it to header; for this recipe prioritize position sticky and compact state.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply position sticky and compact state only to the necessary component boundaries.
- Preserve small height change across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: overlaying content.
- Product requirement: Keep context without obstruction.
- Focus area: header.
- Technique: position sticky and compact state.
- Visible result: first heading never hidden.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of header and save a baseline screenshot.
2. Apply small height change with actual user-visible data rather than filler.
3. Test position sticky and compact state at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for header.
5. Disable JavaScript or graphics support and check the fallback for header.
6. Verify using the observable condition: first heading never hidden.
7. Record any remaining limitation and associated measured cost.


### Recipe 29 — Scroll reveal

**Goal:** Reveal content only when visible.
**Primary method:** IntersectionObserver.
**Where it applies:** content sections.
**Implementation idea:** one-time observable animation.
**Do not:** listeners on scroll.
**Acceptance test:** no invisible content if observer unavailable.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to content sections; for this recipe prioritize one-time observable animation.
2. Define an observable result and list the files or modules that own it. Apply it to content sections; for this recipe prioritize IntersectionObserver.
3. Implement the smallest complete accessible version first. Apply it to content sections; for this recipe prioritize one-time observable animation.
4. Add quality and motion only after functional behavior is correct. Apply it to content sections; for this recipe prioritize IntersectionObserver.
5. Instrument, test, and refine against the stated budget. Apply it to content sections; for this recipe prioritize one-time observable animation.
6. Document a fallback, cleanup path, and operational limitation. Apply it to content sections; for this recipe prioritize IntersectionObserver.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply IntersectionObserver only to the necessary component boundaries.
- Preserve one-time observable animation across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: listeners on scroll.
- Product requirement: Reveal content only when visible.
- Focus area: content sections.
- Technique: IntersectionObserver.
- Visible result: no invisible content if observer unavailable.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of content sections and save a baseline screenshot.
2. Apply one-time observable animation with actual user-visible data rather than filler.
3. Test IntersectionObserver at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for content sections.
5. Disable JavaScript or graphics support and check the fallback for content sections.
6. Verify using the observable condition: no invisible content if observer unavailable.
7. Record any remaining limitation and associated measured cost.


### Recipe 30 — Parallax input

**Goal:** Add restrained depth.
**Primary method:** normalized pointer position.
**Where it applies:** hero object.
**Implementation idea:** rAF-coalesced pointer state.
**Do not:** onmousemove heavy DOM writes.
**Acceptance test:** touch fallback remains composed.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to hero object; for this recipe prioritize rAF-coalesced pointer state.
2. Define an observable result and list the files or modules that own it. Apply it to hero object; for this recipe prioritize normalized pointer position.
3. Implement the smallest complete accessible version first. Apply it to hero object; for this recipe prioritize rAF-coalesced pointer state.
4. Add quality and motion only after functional behavior is correct. Apply it to hero object; for this recipe prioritize normalized pointer position.
5. Instrument, test, and refine against the stated budget. Apply it to hero object; for this recipe prioritize rAF-coalesced pointer state.
6. Document a fallback, cleanup path, and operational limitation. Apply it to hero object; for this recipe prioritize normalized pointer position.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply normalized pointer position only to the necessary component boundaries.
- Preserve rAF-coalesced pointer state across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: onmousemove heavy DOM writes.
- Product requirement: Add restrained depth.
- Focus area: hero object.
- Technique: normalized pointer position.
- Visible result: touch fallback remains composed.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of hero object and save a baseline screenshot.
2. Apply rAF-coalesced pointer state with actual user-visible data rather than filler.
3. Test normalized pointer position at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for hero object.
5. Disable JavaScript or graphics support and check the fallback for hero object.
6. Verify using the observable condition: touch fallback remains composed.
7. Record any remaining limitation and associated measured cost.


### Recipe 31 — Ambient background

**Goal:** Atmosphere must remain secondary.
**Primary method:** CSS or one shader.
**Where it applies:** hero backdrop.
**Implementation idea:** dim restrained motion.
**Do not:** multiple competing canvases.
**Acceptance test:** copy remains readable.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to hero backdrop; for this recipe prioritize dim restrained motion.
2. Define an observable result and list the files or modules that own it. Apply it to hero backdrop; for this recipe prioritize CSS or one shader.
3. Implement the smallest complete accessible version first. Apply it to hero backdrop; for this recipe prioritize dim restrained motion.
4. Add quality and motion only after functional behavior is correct. Apply it to hero backdrop; for this recipe prioritize CSS or one shader.
5. Instrument, test, and refine against the stated budget. Apply it to hero backdrop; for this recipe prioritize dim restrained motion.
6. Document a fallback, cleanup path, and operational limitation. Apply it to hero backdrop; for this recipe prioritize CSS or one shader.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply CSS or one shader only to the necessary component boundaries.
- Preserve dim restrained motion across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: multiple competing canvases.
- Product requirement: Atmosphere must remain secondary.
- Focus area: hero backdrop.
- Technique: CSS or one shader.
- Visible result: copy remains readable.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of hero backdrop and save a baseline screenshot.
2. Apply dim restrained motion with actual user-visible data rather than filler.
3. Test CSS or one shader at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for hero backdrop.
5. Disable JavaScript or graphics support and check the fallback for hero backdrop.
6. Verify using the observable condition: copy remains readable.
7. Record any remaining limitation and associated measured cost.


### Recipe 32 — Aurora shader

**Goal:** Build organic color motion.
**Primary method:** WebGL2 fragment shader.
**Where it applies:** single visual section.
**Implementation idea:** low-frequency color field.
**Do not:** large costly blur chain.
**Acceptance test:** fallback static gradient.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to single visual section; for this recipe prioritize low-frequency color field.
2. Define an observable result and list the files or modules that own it. Apply it to single visual section; for this recipe prioritize WebGL2 fragment shader.
3. Implement the smallest complete accessible version first. Apply it to single visual section; for this recipe prioritize low-frequency color field.
4. Add quality and motion only after functional behavior is correct. Apply it to single visual section; for this recipe prioritize WebGL2 fragment shader.
5. Instrument, test, and refine against the stated budget. Apply it to single visual section; for this recipe prioritize low-frequency color field.
6. Document a fallback, cleanup path, and operational limitation. Apply it to single visual section; for this recipe prioritize WebGL2 fragment shader.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply WebGL2 fragment shader only to the necessary component boundaries.
- Preserve low-frequency color field across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: large costly blur chain.
- Product requirement: Build organic color motion.
- Focus area: single visual section.
- Technique: WebGL2 fragment shader.
- Visible result: fallback static gradient.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of single visual section and save a baseline screenshot.
2. Apply low-frequency color field with actual user-visible data rather than filler.
3. Test WebGL2 fragment shader at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for single visual section.
5. Disable JavaScript or graphics support and check the fallback for single visual section.
6. Verify using the observable condition: fallback static gradient.
7. Record any remaining limitation and associated measured cost.


### Recipe 33 — Dither transition

**Goal:** Use texture as a state cue.
**Primary method:** threshold matrix shader.
**Where it applies:** route veil.
**Implementation idea:** bounded short transition.
**Do not:** full-screen shader always on.
**Acceptance test:** render loop stops after transition.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to route veil; for this recipe prioritize bounded short transition.
2. Define an observable result and list the files or modules that own it. Apply it to route veil; for this recipe prioritize threshold matrix shader.
3. Implement the smallest complete accessible version first. Apply it to route veil; for this recipe prioritize bounded short transition.
4. Add quality and motion only after functional behavior is correct. Apply it to route veil; for this recipe prioritize threshold matrix shader.
5. Instrument, test, and refine against the stated budget. Apply it to route veil; for this recipe prioritize bounded short transition.
6. Document a fallback, cleanup path, and operational limitation. Apply it to route veil; for this recipe prioritize threshold matrix shader.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply threshold matrix shader only to the necessary component boundaries.
- Preserve bounded short transition across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: full-screen shader always on.
- Product requirement: Use texture as a state cue.
- Focus area: route veil.
- Technique: threshold matrix shader.
- Visible result: render loop stops after transition.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of route veil and save a baseline screenshot.
2. Apply bounded short transition with actual user-visible data rather than filler.
3. Test threshold matrix shader at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for route veil.
5. Disable JavaScript or graphics support and check the fallback for route veil.
6. Verify using the observable condition: render loop stops after transition.
7. Record any remaining limitation and associated measured cost.


### Recipe 34 — Grain overlay

**Goal:** Give surfaces material texture.
**Primary method:** small precomputed texture.
**Where it applies:** page backdrop.
**Implementation idea:** very low alpha and no pointer capture.
**Do not:** animated giant noise texture.
**Acceptance test:** GPU and text unaffected.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to page backdrop; for this recipe prioritize very low alpha and no pointer capture.
2. Define an observable result and list the files or modules that own it. Apply it to page backdrop; for this recipe prioritize small precomputed texture.
3. Implement the smallest complete accessible version first. Apply it to page backdrop; for this recipe prioritize very low alpha and no pointer capture.
4. Add quality and motion only after functional behavior is correct. Apply it to page backdrop; for this recipe prioritize small precomputed texture.
5. Instrument, test, and refine against the stated budget. Apply it to page backdrop; for this recipe prioritize very low alpha and no pointer capture.
6. Document a fallback, cleanup path, and operational limitation. Apply it to page backdrop; for this recipe prioritize small precomputed texture.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply small precomputed texture only to the necessary component boundaries.
- Preserve very low alpha and no pointer capture across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: animated giant noise texture.
- Product requirement: Give surfaces material texture.
- Focus area: page backdrop.
- Technique: small precomputed texture.
- Visible result: GPU and text unaffected.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of page backdrop and save a baseline screenshot.
2. Apply very low alpha and no pointer capture with actual user-visible data rather than filler.
3. Test small precomputed texture at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for page backdrop.
5. Disable JavaScript or graphics support and check the fallback for page backdrop.
6. Verify using the observable condition: GPU and text unaffected.
7. Record any remaining limitation and associated measured cost.


### Recipe 35 — ASCII gradient

**Goal:** Build an unusual focal visual.
**Primary method:** Canvas2D glyph grid.
**Where it applies:** decorative hero.
**Implementation idea:** coarse adaptive cell size.
**Do not:** dense DOM characters.
**Acceptance test:** screen readers skip decoration.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to decorative hero; for this recipe prioritize coarse adaptive cell size.
2. Define an observable result and list the files or modules that own it. Apply it to decorative hero; for this recipe prioritize Canvas2D glyph grid.
3. Implement the smallest complete accessible version first. Apply it to decorative hero; for this recipe prioritize coarse adaptive cell size.
4. Add quality and motion only after functional behavior is correct. Apply it to decorative hero; for this recipe prioritize Canvas2D glyph grid.
5. Instrument, test, and refine against the stated budget. Apply it to decorative hero; for this recipe prioritize coarse adaptive cell size.
6. Document a fallback, cleanup path, and operational limitation. Apply it to decorative hero; for this recipe prioritize Canvas2D glyph grid.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply Canvas2D glyph grid only to the necessary component boundaries.
- Preserve coarse adaptive cell size across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: dense DOM characters.
- Product requirement: Build an unusual focal visual.
- Focus area: decorative hero.
- Technique: Canvas2D glyph grid.
- Visible result: screen readers skip decoration.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of decorative hero and save a baseline screenshot.
2. Apply coarse adaptive cell size with actual user-visible data rather than filler.
3. Test Canvas2D glyph grid at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for decorative hero.
5. Disable JavaScript or graphics support and check the fallback for decorative hero.
6. Verify using the observable condition: screen readers skip decoration.
7. Record any remaining limitation and associated measured cost.


### Recipe 36 — Globe scene

**Goal:** Use 3D only where it adds identity.
**Primary method:** lazy Three.js renderer.
**Where it applies:** community focal point.
**Implementation idea:** small geometry and controlled rotation.
**Do not:** always-on render loop.
**Acceptance test:** offscreen RAF stops.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to community focal point; for this recipe prioritize small geometry and controlled rotation.
2. Define an observable result and list the files or modules that own it. Apply it to community focal point; for this recipe prioritize lazy Three.js renderer.
3. Implement the smallest complete accessible version first. Apply it to community focal point; for this recipe prioritize small geometry and controlled rotation.
4. Add quality and motion only after functional behavior is correct. Apply it to community focal point; for this recipe prioritize lazy Three.js renderer.
5. Instrument, test, and refine against the stated budget. Apply it to community focal point; for this recipe prioritize small geometry and controlled rotation.
6. Document a fallback, cleanup path, and operational limitation. Apply it to community focal point; for this recipe prioritize lazy Three.js renderer.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply lazy Three.js renderer only to the necessary component boundaries.
- Preserve small geometry and controlled rotation across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: always-on render loop.
- Product requirement: Use 3D only where it adds identity.
- Focus area: community focal point.
- Technique: lazy Three.js renderer.
- Visible result: offscreen RAF stops.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of community focal point and save a baseline screenshot.
2. Apply small geometry and controlled rotation with actual user-visible data rather than filler.
3. Test lazy Three.js renderer at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for community focal point.
5. Disable JavaScript or graphics support and check the fallback for community focal point.
6. Verify using the observable condition: offscreen RAF stops.
7. Record any remaining limitation and associated measured cost.


### Recipe 37 — Crown scene

**Goal:** Use 3D object as meaningful symbol.
**Primary method:** reused Three.js pipeline.
**Where it applies:** rank hero.
**Implementation idea:** stable lighting and compositing.
**Do not:** multiple renderers per card.
**Acceptance test:** renderer disposed on route exit.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to rank hero; for this recipe prioritize stable lighting and compositing.
2. Define an observable result and list the files or modules that own it. Apply it to rank hero; for this recipe prioritize reused Three.js pipeline.
3. Implement the smallest complete accessible version first. Apply it to rank hero; for this recipe prioritize stable lighting and compositing.
4. Add quality and motion only after functional behavior is correct. Apply it to rank hero; for this recipe prioritize reused Three.js pipeline.
5. Instrument, test, and refine against the stated budget. Apply it to rank hero; for this recipe prioritize stable lighting and compositing.
6. Document a fallback, cleanup path, and operational limitation. Apply it to rank hero; for this recipe prioritize reused Three.js pipeline.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply reused Three.js pipeline only to the necessary component boundaries.
- Preserve stable lighting and compositing across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: multiple renderers per card.
- Product requirement: Use 3D object as meaningful symbol.
- Focus area: rank hero.
- Technique: reused Three.js pipeline.
- Visible result: renderer disposed on route exit.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of rank hero and save a baseline screenshot.
2. Apply stable lighting and compositing with actual user-visible data rather than filler.
3. Test reused Three.js pipeline at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for rank hero.
5. Disable JavaScript or graphics support and check the fallback for rank hero.
6. Verify using the observable condition: renderer disposed on route exit.
7. Record any remaining limitation and associated measured cost.


### Recipe 38 — Rank selection

**Goal:** Make progression intuitive.
**Primary method:** data-driven state.
**Where it applies:** tier previews.
**Implementation idea:** clear current vs locked states.
**Do not:** opaque hover-only behavior.
**Acceptance test:** keyboard and touch operate tiers.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to tier previews; for this recipe prioritize clear current vs locked states.
2. Define an observable result and list the files or modules that own it. Apply it to tier previews; for this recipe prioritize data-driven state.
3. Implement the smallest complete accessible version first. Apply it to tier previews; for this recipe prioritize clear current vs locked states.
4. Add quality and motion only after functional behavior is correct. Apply it to tier previews; for this recipe prioritize data-driven state.
5. Instrument, test, and refine against the stated budget. Apply it to tier previews; for this recipe prioritize clear current vs locked states.
6. Document a fallback, cleanup path, and operational limitation. Apply it to tier previews; for this recipe prioritize data-driven state.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply data-driven state only to the necessary component boundaries.
- Preserve clear current vs locked states across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: opaque hover-only behavior.
- Product requirement: Make progression intuitive.
- Focus area: tier previews.
- Technique: data-driven state.
- Visible result: keyboard and touch operate tiers.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of tier previews and save a baseline screenshot.
2. Apply clear current vs locked states with actual user-visible data rather than filler.
3. Test data-driven state at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for tier previews.
5. Disable JavaScript or graphics support and check the fallback for tier previews.
6. Verify using the observable condition: keyboard and touch operate tiers.
7. Record any remaining limitation and associated measured cost.


### Recipe 39 — Shader compilation

**Goal:** Avoid first-click freeze.
**Primary method:** precompile during idle.
**Where it applies:** critical shader.
**Implementation idea:** cache linked program and locations.
**Do not:** compile in click handler.
**Acceptance test:** first interaction fluid.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to critical shader; for this recipe prioritize cache linked program and locations.
2. Define an observable result and list the files or modules that own it. Apply it to critical shader; for this recipe prioritize precompile during idle.
3. Implement the smallest complete accessible version first. Apply it to critical shader; for this recipe prioritize cache linked program and locations.
4. Add quality and motion only after functional behavior is correct. Apply it to critical shader; for this recipe prioritize precompile during idle.
5. Instrument, test, and refine against the stated budget. Apply it to critical shader; for this recipe prioritize cache linked program and locations.
6. Document a fallback, cleanup path, and operational limitation. Apply it to critical shader; for this recipe prioritize precompile during idle.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply precompile during idle only to the necessary component boundaries.
- Preserve cache linked program and locations across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: compile in click handler.
- Product requirement: Avoid first-click freeze.
- Focus area: critical shader.
- Technique: precompile during idle.
- Visible result: first interaction fluid.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of critical shader and save a baseline screenshot.
2. Apply cache linked program and locations with actual user-visible data rather than filler.
3. Test precompile during idle at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for critical shader.
5. Disable JavaScript or graphics support and check the fallback for critical shader.
6. Verify using the observable condition: first interaction fluid.
7. Record any remaining limitation and associated measured cost.


### Recipe 40 — Shader resolution

**Goal:** Prevent blur and runaway fill-rate.
**Primary method:** pixel ratio and pixel budget.
**Where it applies:** canvas backing store.
**Implementation idea:** resize uniform and viewport together.
**Do not:** DPR 3 fullscreen.
**Acceptance test:** sharp enough under capped pixels.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to canvas backing store; for this recipe prioritize resize uniform and viewport together.
2. Define an observable result and list the files or modules that own it. Apply it to canvas backing store; for this recipe prioritize pixel ratio and pixel budget.
3. Implement the smallest complete accessible version first. Apply it to canvas backing store; for this recipe prioritize resize uniform and viewport together.
4. Add quality and motion only after functional behavior is correct. Apply it to canvas backing store; for this recipe prioritize pixel ratio and pixel budget.
5. Instrument, test, and refine against the stated budget. Apply it to canvas backing store; for this recipe prioritize resize uniform and viewport together.
6. Document a fallback, cleanup path, and operational limitation. Apply it to canvas backing store; for this recipe prioritize pixel ratio and pixel budget.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply pixel ratio and pixel budget only to the necessary component boundaries.
- Preserve resize uniform and viewport together across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: DPR 3 fullscreen.
- Product requirement: Prevent blur and runaway fill-rate.
- Focus area: canvas backing store.
- Technique: pixel ratio and pixel budget.
- Visible result: sharp enough under capped pixels.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of canvas backing store and save a baseline screenshot.
2. Apply resize uniform and viewport together with actual user-visible data rather than filler.
3. Test pixel ratio and pixel budget at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for canvas backing store.
5. Disable JavaScript or graphics support and check the fallback for canvas backing store.
6. Verify using the observable condition: sharp enough under capped pixels.
7. Record any remaining limitation and associated measured cost.


### Recipe 41 — Canvas fallback

**Goal:** Protect baseline functionality.
**Primary method:** CSS fallback below canvas.
**Where it applies:** visual hero.
**Implementation idea:** detect context failure.
**Do not:** empty black rectangle.
**Acceptance test:** page understandable without WebGL.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to visual hero; for this recipe prioritize detect context failure.
2. Define an observable result and list the files or modules that own it. Apply it to visual hero; for this recipe prioritize CSS fallback below canvas.
3. Implement the smallest complete accessible version first. Apply it to visual hero; for this recipe prioritize detect context failure.
4. Add quality and motion only after functional behavior is correct. Apply it to visual hero; for this recipe prioritize CSS fallback below canvas.
5. Instrument, test, and refine against the stated budget. Apply it to visual hero; for this recipe prioritize detect context failure.
6. Document a fallback, cleanup path, and operational limitation. Apply it to visual hero; for this recipe prioritize CSS fallback below canvas.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply CSS fallback below canvas only to the necessary component boundaries.
- Preserve detect context failure across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: empty black rectangle.
- Product requirement: Protect baseline functionality.
- Focus area: visual hero.
- Technique: CSS fallback below canvas.
- Visible result: page understandable without WebGL.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of visual hero and save a baseline screenshot.
2. Apply detect context failure with actual user-visible data rather than filler.
3. Test CSS fallback below canvas at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for visual hero.
5. Disable JavaScript or graphics support and check the fallback for visual hero.
6. Verify using the observable condition: page understandable without WebGL.
7. Record any remaining limitation and associated measured cost.


### Recipe 42 — Canvas visibility

**Goal:** Eliminate hidden GPU work.
**Primary method:** IntersectionObserver and visibilitychange.
**Where it applies:** each canvas effect.
**Implementation idea:** single RAF owner.
**Do not:** RAF scheduling with early-return only.
**Acceptance test:** zero frames scheduled when hidden.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to each canvas effect; for this recipe prioritize single RAF owner.
2. Define an observable result and list the files or modules that own it. Apply it to each canvas effect; for this recipe prioritize IntersectionObserver and visibilitychange.
3. Implement the smallest complete accessible version first. Apply it to each canvas effect; for this recipe prioritize single RAF owner.
4. Add quality and motion only after functional behavior is correct. Apply it to each canvas effect; for this recipe prioritize IntersectionObserver and visibilitychange.
5. Instrument, test, and refine against the stated budget. Apply it to each canvas effect; for this recipe prioritize single RAF owner.
6. Document a fallback, cleanup path, and operational limitation. Apply it to each canvas effect; for this recipe prioritize IntersectionObserver and visibilitychange.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply IntersectionObserver and visibilitychange only to the necessary component boundaries.
- Preserve single RAF owner across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: RAF scheduling with early-return only.
- Product requirement: Eliminate hidden GPU work.
- Focus area: each canvas effect.
- Technique: IntersectionObserver and visibilitychange.
- Visible result: zero frames scheduled when hidden.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of each canvas effect and save a baseline screenshot.
2. Apply single RAF owner with actual user-visible data rather than filler.
3. Test IntersectionObserver and visibilitychange at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for each canvas effect.
5. Disable JavaScript or graphics support and check the fallback for each canvas effect.
6. Verify using the observable condition: zero frames scheduled when hidden.
7. Record any remaining limitation and associated measured cost.


### Recipe 43 — Frame delta

**Goal:** Maintain consistent speed.
**Primary method:** clamped dt.
**Where it applies:** all continuous motion.
**Implementation idea:** reset time on resume.
**Do not:** frame-count movement.
**Acceptance test:** no jump after tab resume.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all continuous motion; for this recipe prioritize reset time on resume.
2. Define an observable result and list the files or modules that own it. Apply it to all continuous motion; for this recipe prioritize clamped dt.
3. Implement the smallest complete accessible version first. Apply it to all continuous motion; for this recipe prioritize reset time on resume.
4. Add quality and motion only after functional behavior is correct. Apply it to all continuous motion; for this recipe prioritize clamped dt.
5. Instrument, test, and refine against the stated budget. Apply it to all continuous motion; for this recipe prioritize reset time on resume.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all continuous motion; for this recipe prioritize clamped dt.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply clamped dt only to the necessary component boundaries.
- Preserve reset time on resume across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: frame-count movement.
- Product requirement: Maintain consistent speed.
- Focus area: all continuous motion.
- Technique: clamped dt.
- Visible result: no jump after tab resume.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of all continuous motion and save a baseline screenshot.
2. Apply reset time on resume with actual user-visible data rather than filler.
3. Test clamped dt at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all continuous motion.
5. Disable JavaScript or graphics support and check the fallback for all continuous motion.
6. Verify using the observable condition: no jump after tab resume.
7. Record any remaining limitation and associated measured cost.


### Recipe 44 — Resize lifecycle

**Goal:** Control backing-store reallocations.
**Primary method:** ResizeObserver batched into RAF.
**Where it applies:** canvas and 3D.
**Implementation idea:** skip unchanged dimensions.
**Do not:** setSize every frame.
**Acceptance test:** resize triggers once per size change.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to canvas and 3D; for this recipe prioritize skip unchanged dimensions.
2. Define an observable result and list the files or modules that own it. Apply it to canvas and 3D; for this recipe prioritize ResizeObserver batched into RAF.
3. Implement the smallest complete accessible version first. Apply it to canvas and 3D; for this recipe prioritize skip unchanged dimensions.
4. Add quality and motion only after functional behavior is correct. Apply it to canvas and 3D; for this recipe prioritize ResizeObserver batched into RAF.
5. Instrument, test, and refine against the stated budget. Apply it to canvas and 3D; for this recipe prioritize skip unchanged dimensions.
6. Document a fallback, cleanup path, and operational limitation. Apply it to canvas and 3D; for this recipe prioritize ResizeObserver batched into RAF.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply ResizeObserver batched into RAF only to the necessary component boundaries.
- Preserve skip unchanged dimensions across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: setSize every frame.
- Product requirement: Control backing-store reallocations.
- Focus area: canvas and 3D.
- Technique: ResizeObserver batched into RAF.
- Visible result: resize triggers once per size change.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of canvas and 3D and save a baseline screenshot.
2. Apply skip unchanged dimensions with actual user-visible data rather than filler.
3. Test ResizeObserver batched into RAF at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for canvas and 3D.
5. Disable JavaScript or graphics support and check the fallback for canvas and 3D.
6. Verify using the observable condition: resize triggers once per size change.
7. Record any remaining limitation and associated measured cost.


### Recipe 45 — Event cleanup

**Goal:** Prevent navigation memory leaks.
**Primary method:** AbortController listeners and destroy.
**Where it applies:** all modules.
**Implementation idea:** single clear teardown path.
**Do not:** anonymous unremovable listeners.
**Acceptance test:** repeat routes do not grow listeners.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all modules; for this recipe prioritize single clear teardown path.
2. Define an observable result and list the files or modules that own it. Apply it to all modules; for this recipe prioritize AbortController listeners and destroy.
3. Implement the smallest complete accessible version first. Apply it to all modules; for this recipe prioritize single clear teardown path.
4. Add quality and motion only after functional behavior is correct. Apply it to all modules; for this recipe prioritize AbortController listeners and destroy.
5. Instrument, test, and refine against the stated budget. Apply it to all modules; for this recipe prioritize single clear teardown path.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all modules; for this recipe prioritize AbortController listeners and destroy.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply AbortController listeners and destroy only to the necessary component boundaries.
- Preserve single clear teardown path across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: anonymous unremovable listeners.
- Product requirement: Prevent navigation memory leaks.
- Focus area: all modules.
- Technique: AbortController listeners and destroy.
- Visible result: repeat routes do not grow listeners.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of all modules and save a baseline screenshot.
2. Apply single clear teardown path with actual user-visible data rather than filler.
3. Test AbortController listeners and destroy at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all modules.
5. Disable JavaScript or graphics support and check the fallback for all modules.
6. Verify using the observable condition: repeat routes do not grow listeners.
7. Record any remaining limitation and associated measured cost.


### Recipe 46 — WebGL disposal

**Goal:** Release GPU resources.
**Primary method:** dispose geometry material textures.
**Where it applies:** Three.js route modules.
**Implementation idea:** dispose on unmount.
**Do not:** orphaned contexts.
**Acceptance test:** GPU memory stable over cycles.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to Three.js route modules; for this recipe prioritize dispose on unmount.
2. Define an observable result and list the files or modules that own it. Apply it to Three.js route modules; for this recipe prioritize dispose geometry material textures.
3. Implement the smallest complete accessible version first. Apply it to Three.js route modules; for this recipe prioritize dispose on unmount.
4. Add quality and motion only after functional behavior is correct. Apply it to Three.js route modules; for this recipe prioritize dispose geometry material textures.
5. Instrument, test, and refine against the stated budget. Apply it to Three.js route modules; for this recipe prioritize dispose on unmount.
6. Document a fallback, cleanup path, and operational limitation. Apply it to Three.js route modules; for this recipe prioritize dispose geometry material textures.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply dispose geometry material textures only to the necessary component boundaries.
- Preserve dispose on unmount across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: orphaned contexts.
- Product requirement: Release GPU resources.
- Focus area: Three.js route modules.
- Technique: dispose geometry material textures.
- Visible result: GPU memory stable over cycles.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of Three.js route modules and save a baseline screenshot.
2. Apply dispose on unmount with actual user-visible data rather than filler.
3. Test dispose geometry material textures at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for Three.js route modules.
5. Disable JavaScript or graphics support and check the fallback for Three.js route modules.
6. Verify using the observable condition: GPU memory stable over cycles.
7. Record any remaining limitation and associated measured cost.


### Recipe 47 — Context recovery

**Goal:** Handle graphics device changes.
**Primary method:** webglcontextlost and restored.
**Where it applies:** WebGL modules.
**Implementation idea:** recreate resources from descriptors.
**Do not:** blank screen forever.
**Acceptance test:** fallback after repeated losses.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to WebGL modules; for this recipe prioritize recreate resources from descriptors.
2. Define an observable result and list the files or modules that own it. Apply it to WebGL modules; for this recipe prioritize webglcontextlost and restored.
3. Implement the smallest complete accessible version first. Apply it to WebGL modules; for this recipe prioritize recreate resources from descriptors.
4. Add quality and motion only after functional behavior is correct. Apply it to WebGL modules; for this recipe prioritize webglcontextlost and restored.
5. Instrument, test, and refine against the stated budget. Apply it to WebGL modules; for this recipe prioritize recreate resources from descriptors.
6. Document a fallback, cleanup path, and operational limitation. Apply it to WebGL modules; for this recipe prioritize webglcontextlost and restored.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply webglcontextlost and restored only to the necessary component boundaries.
- Preserve recreate resources from descriptors across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: blank screen forever.
- Product requirement: Handle graphics device changes.
- Focus area: WebGL modules.
- Technique: webglcontextlost and restored.
- Visible result: fallback after repeated losses.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of WebGL modules and save a baseline screenshot.
2. Apply recreate resources from descriptors with actual user-visible data rather than filler.
3. Test webglcontextlost and restored at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for WebGL modules.
5. Disable JavaScript or graphics support and check the fallback for WebGL modules.
6. Verify using the observable condition: fallback after repeated losses.
7. Record any remaining limitation and associated measured cost.


### Recipe 48 — Audio opt-in

**Goal:** Respect user control.
**Primary method:** manual mute/unmute.
**Where it applies:** ambient audio.
**Implementation idea:** default silent and visible state.
**Do not:** autoplay surprise.
**Acceptance test:** no audio before gesture.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to ambient audio; for this recipe prioritize default silent and visible state.
2. Define an observable result and list the files or modules that own it. Apply it to ambient audio; for this recipe prioritize manual mute/unmute.
3. Implement the smallest complete accessible version first. Apply it to ambient audio; for this recipe prioritize default silent and visible state.
4. Add quality and motion only after functional behavior is correct. Apply it to ambient audio; for this recipe prioritize manual mute/unmute.
5. Instrument, test, and refine against the stated budget. Apply it to ambient audio; for this recipe prioritize default silent and visible state.
6. Document a fallback, cleanup path, and operational limitation. Apply it to ambient audio; for this recipe prioritize manual mute/unmute.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply manual mute/unmute only to the necessary component boundaries.
- Preserve default silent and visible state across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: autoplay surprise.
- Product requirement: Respect user control.
- Focus area: ambient audio.
- Technique: manual mute/unmute.
- Visible result: no audio before gesture.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of ambient audio and save a baseline screenshot.
2. Apply default silent and visible state with actual user-visible data rather than filler.
3. Test manual mute/unmute at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for ambient audio.
5. Disable JavaScript or graphics support and check the fallback for ambient audio.
6. Verify using the observable condition: no audio before gesture.
7. Record any remaining limitation and associated measured cost.


### Recipe 49 — Audio lifecycle

**Goal:** Avoid lingering sound and CPU.
**Primary method:** suspend and close AudioContext.
**Where it applies:** sound engine.
**Implementation idea:** pause when hidden.
**Do not:** multiple contexts per route.
**Acceptance test:** repeated navigation has one context.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to sound engine; for this recipe prioritize pause when hidden.
2. Define an observable result and list the files or modules that own it. Apply it to sound engine; for this recipe prioritize suspend and close AudioContext.
3. Implement the smallest complete accessible version first. Apply it to sound engine; for this recipe prioritize pause when hidden.
4. Add quality and motion only after functional behavior is correct. Apply it to sound engine; for this recipe prioritize suspend and close AudioContext.
5. Instrument, test, and refine against the stated budget. Apply it to sound engine; for this recipe prioritize pause when hidden.
6. Document a fallback, cleanup path, and operational limitation. Apply it to sound engine; for this recipe prioritize suspend and close AudioContext.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply suspend and close AudioContext only to the necessary component boundaries.
- Preserve pause when hidden across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: multiple contexts per route.
- Product requirement: Avoid lingering sound and CPU.
- Focus area: sound engine.
- Technique: suspend and close AudioContext.
- Visible result: repeated navigation has one context.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of sound engine and save a baseline screenshot.
2. Apply pause when hidden with actual user-visible data rather than filler.
3. Test suspend and close AudioContext at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for sound engine.
5. Disable JavaScript or graphics support and check the fallback for sound engine.
6. Verify using the observable condition: repeated navigation has one context.
7. Record any remaining limitation and associated measured cost.


### Recipe 50 — Semantic structure

**Goal:** Keep complex visuals accessible.
**Primary method:** landmarks and headings.
**Where it applies:** every page.
**Implementation idea:** clear main nav article roles.
**Do not:** div soup.
**Acceptance test:** screen reader navigation works.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to every page; for this recipe prioritize clear main nav article roles.
2. Define an observable result and list the files or modules that own it. Apply it to every page; for this recipe prioritize landmarks and headings.
3. Implement the smallest complete accessible version first. Apply it to every page; for this recipe prioritize clear main nav article roles.
4. Add quality and motion only after functional behavior is correct. Apply it to every page; for this recipe prioritize landmarks and headings.
5. Instrument, test, and refine against the stated budget. Apply it to every page; for this recipe prioritize clear main nav article roles.
6. Document a fallback, cleanup path, and operational limitation. Apply it to every page; for this recipe prioritize landmarks and headings.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply landmarks and headings only to the necessary component boundaries.
- Preserve clear main nav article roles across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: div soup.
- Product requirement: Keep complex visuals accessible.
- Focus area: every page.
- Technique: landmarks and headings.
- Visible result: screen reader navigation works.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of every page and save a baseline screenshot.
2. Apply clear main nav article roles with actual user-visible data rather than filler.
3. Test landmarks and headings at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for every page.
5. Disable JavaScript or graphics support and check the fallback for every page.
6. Verify using the observable condition: screen reader navigation works.
7. Record any remaining limitation and associated measured cost.


### Recipe 51 — Dialog behavior

**Goal:** Avoid stuck modal states.
**Primary method:** native dialog or focus manager.
**Where it applies:** overlays.
**Implementation idea:** escape and focus return.
**Do not:** click-through overlay.
**Acceptance test:** keyboard closes and returns focus.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to overlays; for this recipe prioritize escape and focus return.
2. Define an observable result and list the files or modules that own it. Apply it to overlays; for this recipe prioritize native dialog or focus manager.
3. Implement the smallest complete accessible version first. Apply it to overlays; for this recipe prioritize escape and focus return.
4. Add quality and motion only after functional behavior is correct. Apply it to overlays; for this recipe prioritize native dialog or focus manager.
5. Instrument, test, and refine against the stated budget. Apply it to overlays; for this recipe prioritize escape and focus return.
6. Document a fallback, cleanup path, and operational limitation. Apply it to overlays; for this recipe prioritize native dialog or focus manager.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply native dialog or focus manager only to the necessary component boundaries.
- Preserve escape and focus return across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: click-through overlay.
- Product requirement: Avoid stuck modal states.
- Focus area: overlays.
- Technique: native dialog or focus manager.
- Visible result: keyboard closes and returns focus.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of overlays and save a baseline screenshot.
2. Apply escape and focus return with actual user-visible data rather than filler.
3. Test native dialog or focus manager at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for overlays.
5. Disable JavaScript or graphics support and check the fallback for overlays.
6. Verify using the observable condition: keyboard closes and returns focus.
7. Record any remaining limitation and associated measured cost.


### Recipe 52 — State architecture

**Goal:** Keep effects separate from navigation.
**Primary method:** explicit state owner.
**Where it applies:** application shell.
**Implementation idea:** inputs and outputs defined.
**Do not:** global ad-hoc flags.
**Acceptance test:** consistent state under repeats.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to application shell; for this recipe prioritize inputs and outputs defined.
2. Define an observable result and list the files or modules that own it. Apply it to application shell; for this recipe prioritize explicit state owner.
3. Implement the smallest complete accessible version first. Apply it to application shell; for this recipe prioritize inputs and outputs defined.
4. Add quality and motion only after functional behavior is correct. Apply it to application shell; for this recipe prioritize explicit state owner.
5. Instrument, test, and refine against the stated budget. Apply it to application shell; for this recipe prioritize inputs and outputs defined.
6. Document a fallback, cleanup path, and operational limitation. Apply it to application shell; for this recipe prioritize explicit state owner.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply explicit state owner only to the necessary component boundaries.
- Preserve inputs and outputs defined across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: global ad-hoc flags.
- Product requirement: Keep effects separate from navigation.
- Focus area: application shell.
- Technique: explicit state owner.
- Visible result: consistent state under repeats.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of application shell and save a baseline screenshot.
2. Apply inputs and outputs defined with actual user-visible data rather than filler.
3. Test explicit state owner at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for application shell.
5. Disable JavaScript or graphics support and check the fallback for application shell.
6. Verify using the observable condition: consistent state under repeats.
7. Record any remaining limitation and associated measured cost.


### Recipe 53 — Error boundaries

**Goal:** Keep page alive on failing effects.
**Primary method:** per-effect try/catch and fallback.
**Where it applies:** visual modules.
**Implementation idea:** visible content independent.
**Do not:** one shader crash kills app.
**Acceptance test:** controls still work after graphics failure.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to visual modules; for this recipe prioritize visible content independent.
2. Define an observable result and list the files or modules that own it. Apply it to visual modules; for this recipe prioritize per-effect try/catch and fallback.
3. Implement the smallest complete accessible version first. Apply it to visual modules; for this recipe prioritize visible content independent.
4. Add quality and motion only after functional behavior is correct. Apply it to visual modules; for this recipe prioritize per-effect try/catch and fallback.
5. Instrument, test, and refine against the stated budget. Apply it to visual modules; for this recipe prioritize visible content independent.
6. Document a fallback, cleanup path, and operational limitation. Apply it to visual modules; for this recipe prioritize per-effect try/catch and fallback.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply per-effect try/catch and fallback only to the necessary component boundaries.
- Preserve visible content independent across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: one shader crash kills app.
- Product requirement: Keep page alive on failing effects.
- Focus area: visual modules.
- Technique: per-effect try/catch and fallback.
- Visible result: controls still work after graphics failure.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of visual modules and save a baseline screenshot.
2. Apply visible content independent with actual user-visible data rather than filler.
3. Test per-effect try/catch and fallback at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for visual modules.
5. Disable JavaScript or graphics support and check the fallback for visual modules.
6. Verify using the observable condition: controls still work after graphics failure.
7. Record any remaining limitation and associated measured cost.


### Recipe 54 — Offline behavior

**Goal:** Reduce dependence on CDNs.
**Primary method:** self-host or pinned assets.
**Where it applies:** fonts and scripts.
**Implementation idea:** local fallback path.
**Do not:** unversioned runtime imports.
**Acceptance test:** usable local build.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to fonts and scripts; for this recipe prioritize local fallback path.
2. Define an observable result and list the files or modules that own it. Apply it to fonts and scripts; for this recipe prioritize self-host or pinned assets.
3. Implement the smallest complete accessible version first. Apply it to fonts and scripts; for this recipe prioritize local fallback path.
4. Add quality and motion only after functional behavior is correct. Apply it to fonts and scripts; for this recipe prioritize self-host or pinned assets.
5. Instrument, test, and refine against the stated budget. Apply it to fonts and scripts; for this recipe prioritize local fallback path.
6. Document a fallback, cleanup path, and operational limitation. Apply it to fonts and scripts; for this recipe prioritize self-host or pinned assets.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply self-host or pinned assets only to the necessary component boundaries.
- Preserve local fallback path across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: unversioned runtime imports.
- Product requirement: Reduce dependence on CDNs.
- Focus area: fonts and scripts.
- Technique: self-host or pinned assets.
- Visible result: usable local build.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of fonts and scripts and save a baseline screenshot.
2. Apply local fallback path with actual user-visible data rather than filler.
3. Test self-host or pinned assets at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for fonts and scripts.
5. Disable JavaScript or graphics support and check the fallback for fonts and scripts.
6. Verify using the observable condition: usable local build.
7. Record any remaining limitation and associated measured cost.


### Recipe 55 — Dependency budget

**Goal:** Choose tools for value not fashion.
**Primary method:** native browser first.
**Where it applies:** build and runtime.
**Implementation idea:** small focused libraries.
**Do not:** stacking UI frameworks.
**Acceptance test:** dependency audit passes.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to build and runtime; for this recipe prioritize small focused libraries.
2. Define an observable result and list the files or modules that own it. Apply it to build and runtime; for this recipe prioritize native browser first.
3. Implement the smallest complete accessible version first. Apply it to build and runtime; for this recipe prioritize small focused libraries.
4. Add quality and motion only after functional behavior is correct. Apply it to build and runtime; for this recipe prioritize native browser first.
5. Instrument, test, and refine against the stated budget. Apply it to build and runtime; for this recipe prioritize small focused libraries.
6. Document a fallback, cleanup path, and operational limitation. Apply it to build and runtime; for this recipe prioritize native browser first.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply native browser first only to the necessary component boundaries.
- Preserve small focused libraries across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: stacking UI frameworks.
- Product requirement: Choose tools for value not fashion.
- Focus area: build and runtime.
- Technique: native browser first.
- Visible result: dependency audit passes.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of build and runtime and save a baseline screenshot.
2. Apply small focused libraries with actual user-visible data rather than filler.
3. Test native browser first at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for build and runtime.
5. Disable JavaScript or graphics support and check the fallback for build and runtime.
6. Verify using the observable condition: dependency audit passes.
7. Record any remaining limitation and associated measured cost.


### Recipe 56 — CSS architecture

**Goal:** Prevent cascade overrides.
**Primary method:** tokens and component layers.
**Where it applies:** stylesheets.
**Implementation idea:** clear ownership and cascade layers.
**Do not:** v15-final-final.css.
**Acceptance test:** rules remain traceable.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to stylesheets; for this recipe prioritize clear ownership and cascade layers.
2. Define an observable result and list the files or modules that own it. Apply it to stylesheets; for this recipe prioritize tokens and component layers.
3. Implement the smallest complete accessible version first. Apply it to stylesheets; for this recipe prioritize clear ownership and cascade layers.
4. Add quality and motion only after functional behavior is correct. Apply it to stylesheets; for this recipe prioritize tokens and component layers.
5. Instrument, test, and refine against the stated budget. Apply it to stylesheets; for this recipe prioritize clear ownership and cascade layers.
6. Document a fallback, cleanup path, and operational limitation. Apply it to stylesheets; for this recipe prioritize tokens and component layers.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply tokens and component layers only to the necessary component boundaries.
- Preserve clear ownership and cascade layers across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: v15-final-final.css.
- Product requirement: Prevent cascade overrides.
- Focus area: stylesheets.
- Technique: tokens and component layers.
- Visible result: rules remain traceable.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of stylesheets and save a baseline screenshot.
2. Apply clear ownership and cascade layers with actual user-visible data rather than filler.
3. Test tokens and component layers at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for stylesheets.
5. Disable JavaScript or graphics support and check the fallback for stylesheets.
6. Verify using the observable condition: rules remain traceable.
7. Record any remaining limitation and associated measured cost.


### Recipe 57 — Module boundaries

**Goal:** Organize by responsibility.
**Primary method:** ES modules and lifecycle API.
**Where it applies:** effects and interface.
**Implementation idea:** one module per feature.
**Do not:** 100KB global UI script.
**Acceptance test:** local tests for each module.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to effects and interface; for this recipe prioritize one module per feature.
2. Define an observable result and list the files or modules that own it. Apply it to effects and interface; for this recipe prioritize ES modules and lifecycle API.
3. Implement the smallest complete accessible version first. Apply it to effects and interface; for this recipe prioritize one module per feature.
4. Add quality and motion only after functional behavior is correct. Apply it to effects and interface; for this recipe prioritize ES modules and lifecycle API.
5. Instrument, test, and refine against the stated budget. Apply it to effects and interface; for this recipe prioritize one module per feature.
6. Document a fallback, cleanup path, and operational limitation. Apply it to effects and interface; for this recipe prioritize ES modules and lifecycle API.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply ES modules and lifecycle API only to the necessary component boundaries.
- Preserve one module per feature across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: 100KB global UI script.
- Product requirement: Organize by responsibility.
- Focus area: effects and interface.
- Technique: ES modules and lifecycle API.
- Visible result: local tests for each module.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of effects and interface and save a baseline screenshot.
2. Apply one module per feature with actual user-visible data rather than filler.
3. Test ES modules and lifecycle API at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for effects and interface.
5. Disable JavaScript or graphics support and check the fallback for effects and interface.
6. Verify using the observable condition: local tests for each module.
7. Record any remaining limitation and associated measured cost.


### Recipe 58 — Animation tokens

**Goal:** Unify site timing.
**Primary method:** CSS custom properties.
**Where it applies:** all transitions.
**Implementation idea:** documented timing classes.
**Do not:** random durations.
**Acceptance test:** all controls feel related.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all transitions; for this recipe prioritize documented timing classes.
2. Define an observable result and list the files or modules that own it. Apply it to all transitions; for this recipe prioritize CSS custom properties.
3. Implement the smallest complete accessible version first. Apply it to all transitions; for this recipe prioritize documented timing classes.
4. Add quality and motion only after functional behavior is correct. Apply it to all transitions; for this recipe prioritize CSS custom properties.
5. Instrument, test, and refine against the stated budget. Apply it to all transitions; for this recipe prioritize documented timing classes.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all transitions; for this recipe prioritize CSS custom properties.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply CSS custom properties only to the necessary component boundaries.
- Preserve documented timing classes across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: random durations.
- Product requirement: Unify site timing.
- Focus area: all transitions.
- Technique: CSS custom properties.
- Visible result: all controls feel related.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of all transitions and save a baseline screenshot.
2. Apply documented timing classes with actual user-visible data rather than filler.
3. Test CSS custom properties at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all transitions.
5. Disable JavaScript or graphics support and check the fallback for all transitions.
6. Verify using the observable condition: all controls feel related.
7. Record any remaining limitation and associated measured cost.


### Recipe 59 — Transition interruptions

**Goal:** Handle quick repeated input.
**Primary method:** cancel and state machine.
**Where it applies:** navigation.
**Implementation idea:** safe animation cancel/commit.
**Do not:** parallel state mutations.
**Acceptance test:** ten rapid clicks settle correctly.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to navigation; for this recipe prioritize safe animation cancel/commit.
2. Define an observable result and list the files or modules that own it. Apply it to navigation; for this recipe prioritize cancel and state machine.
3. Implement the smallest complete accessible version first. Apply it to navigation; for this recipe prioritize safe animation cancel/commit.
4. Add quality and motion only after functional behavior is correct. Apply it to navigation; for this recipe prioritize cancel and state machine.
5. Instrument, test, and refine against the stated budget. Apply it to navigation; for this recipe prioritize safe animation cancel/commit.
6. Document a fallback, cleanup path, and operational limitation. Apply it to navigation; for this recipe prioritize cancel and state machine.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply cancel and state machine only to the necessary component boundaries.
- Preserve safe animation cancel/commit across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: parallel state mutations.
- Product requirement: Handle quick repeated input.
- Focus area: navigation.
- Technique: cancel and state machine.
- Visible result: ten rapid clicks settle correctly.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of navigation and save a baseline screenshot.
2. Apply safe animation cancel/commit with actual user-visible data rather than filler.
3. Test cancel and state machine at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for navigation.
5. Disable JavaScript or graphics support and check the fallback for navigation.
6. Verify using the observable condition: ten rapid clicks settle correctly.
7. Record any remaining limitation and associated measured cost.


### Recipe 60 — Interaction sound

**Goal:** Keep feedback restrained.
**Primary method:** opt-in short samples.
**Where it applies:** buttons and rank previews.
**Implementation idea:** user volume and mute.
**Do not:** sound on every hover.
**Acceptance test:** silent mode truly silent.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to buttons and rank previews; for this recipe prioritize user volume and mute.
2. Define an observable result and list the files or modules that own it. Apply it to buttons and rank previews; for this recipe prioritize opt-in short samples.
3. Implement the smallest complete accessible version first. Apply it to buttons and rank previews; for this recipe prioritize user volume and mute.
4. Add quality and motion only after functional behavior is correct. Apply it to buttons and rank previews; for this recipe prioritize opt-in short samples.
5. Instrument, test, and refine against the stated budget. Apply it to buttons and rank previews; for this recipe prioritize user volume and mute.
6. Document a fallback, cleanup path, and operational limitation. Apply it to buttons and rank previews; for this recipe prioritize opt-in short samples.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply opt-in short samples only to the necessary component boundaries.
- Preserve user volume and mute across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: sound on every hover.
- Product requirement: Keep feedback restrained.
- Focus area: buttons and rank previews.
- Technique: opt-in short samples.
- Visible result: silent mode truly silent.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of buttons and rank previews and save a baseline screenshot.
2. Apply user volume and mute with actual user-visible data rather than filler.
3. Test opt-in short samples at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for buttons and rank previews.
5. Disable JavaScript or graphics support and check the fallback for buttons and rank previews.
6. Verify using the observable condition: silent mode truly silent.
7. Record any remaining limitation and associated measured cost.


### Recipe 61 — Mobile GPU

**Goal:** Preserve battery and thermals.
**Primary method:** quality levels.
**Where it applies:** canvas and shaders.
**Implementation idea:** max pixels and offscreen stop.
**Do not:** desktop shader on every phone.
**Acceptance test:** stable input on low-end phone.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to canvas and shaders; for this recipe prioritize max pixels and offscreen stop.
2. Define an observable result and list the files or modules that own it. Apply it to canvas and shaders; for this recipe prioritize quality levels.
3. Implement the smallest complete accessible version first. Apply it to canvas and shaders; for this recipe prioritize max pixels and offscreen stop.
4. Add quality and motion only after functional behavior is correct. Apply it to canvas and shaders; for this recipe prioritize quality levels.
5. Instrument, test, and refine against the stated budget. Apply it to canvas and shaders; for this recipe prioritize max pixels and offscreen stop.
6. Document a fallback, cleanup path, and operational limitation. Apply it to canvas and shaders; for this recipe prioritize quality levels.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply quality levels only to the necessary component boundaries.
- Preserve max pixels and offscreen stop across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: desktop shader on every phone.
- Product requirement: Preserve battery and thermals.
- Focus area: canvas and shaders.
- Technique: quality levels.
- Visible result: stable input on low-end phone.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of canvas and shaders and save a baseline screenshot.
2. Apply max pixels and offscreen stop with actual user-visible data rather than filler.
3. Test quality levels at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for canvas and shaders.
5. Disable JavaScript or graphics support and check the fallback for canvas and shaders.
6. Verify using the observable condition: stable input on low-end phone.
7. Record any remaining limitation and associated measured cost.


### Recipe 62 — CPU budget

**Goal:** Avoid long JS tasks.
**Primary method:** work splitting and scheduling.
**Where it applies:** initialization.
**Implementation idea:** idle init of noncritical modules.
**Do not:** compile all scenes at startup.
**Acceptance test:** no major input delay.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to initialization; for this recipe prioritize idle init of noncritical modules.
2. Define an observable result and list the files or modules that own it. Apply it to initialization; for this recipe prioritize work splitting and scheduling.
3. Implement the smallest complete accessible version first. Apply it to initialization; for this recipe prioritize idle init of noncritical modules.
4. Add quality and motion only after functional behavior is correct. Apply it to initialization; for this recipe prioritize work splitting and scheduling.
5. Instrument, test, and refine against the stated budget. Apply it to initialization; for this recipe prioritize idle init of noncritical modules.
6. Document a fallback, cleanup path, and operational limitation. Apply it to initialization; for this recipe prioritize work splitting and scheduling.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply work splitting and scheduling only to the necessary component boundaries.
- Preserve idle init of noncritical modules across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: compile all scenes at startup.
- Product requirement: Avoid long JS tasks.
- Focus area: initialization.
- Technique: work splitting and scheduling.
- Visible result: no major input delay.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of initialization and save a baseline screenshot.
2. Apply idle init of noncritical modules with actual user-visible data rather than filler.
3. Test work splitting and scheduling at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for initialization.
5. Disable JavaScript or graphics support and check the fallback for initialization.
6. Verify using the observable condition: no major input delay.
7. Record any remaining limitation and associated measured cost.


### Recipe 63 — Network waterfall

**Goal:** Prioritize essential content.
**Primary method:** defer and preload wisely.
**Where it applies:** first load.
**Implementation idea:** critical CSS before decorative JS.
**Do not:** blocking CDN bundles.
**Acceptance test:** hero readable before shaders.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to first load; for this recipe prioritize critical CSS before decorative JS.
2. Define an observable result and list the files or modules that own it. Apply it to first load; for this recipe prioritize defer and preload wisely.
3. Implement the smallest complete accessible version first. Apply it to first load; for this recipe prioritize critical CSS before decorative JS.
4. Add quality and motion only after functional behavior is correct. Apply it to first load; for this recipe prioritize defer and preload wisely.
5. Instrument, test, and refine against the stated budget. Apply it to first load; for this recipe prioritize critical CSS before decorative JS.
6. Document a fallback, cleanup path, and operational limitation. Apply it to first load; for this recipe prioritize defer and preload wisely.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply defer and preload wisely only to the necessary component boundaries.
- Preserve critical CSS before decorative JS across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: blocking CDN bundles.
- Product requirement: Prioritize essential content.
- Focus area: first load.
- Technique: defer and preload wisely.
- Visible result: hero readable before shaders.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of first load and save a baseline screenshot.
2. Apply critical CSS before decorative JS with actual user-visible data rather than filler.
3. Test defer and preload wisely at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for first load.
5. Disable JavaScript or graphics support and check the fallback for first load.
6. Verify using the observable condition: hero readable before shaders.
7. Record any remaining limitation and associated measured cost.


### Recipe 64 — Layout stability

**Goal:** Avoid unexpected shifts.
**Primary method:** aspect-ratio and size reservations.
**Where it applies:** hero images and fonts.
**Implementation idea:** known intrinsic sizes.
**Do not:** late content movement.
**Acceptance test:** minimal CLS.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to hero images and fonts; for this recipe prioritize known intrinsic sizes.
2. Define an observable result and list the files or modules that own it. Apply it to hero images and fonts; for this recipe prioritize aspect-ratio and size reservations.
3. Implement the smallest complete accessible version first. Apply it to hero images and fonts; for this recipe prioritize known intrinsic sizes.
4. Add quality and motion only after functional behavior is correct. Apply it to hero images and fonts; for this recipe prioritize aspect-ratio and size reservations.
5. Instrument, test, and refine against the stated budget. Apply it to hero images and fonts; for this recipe prioritize known intrinsic sizes.
6. Document a fallback, cleanup path, and operational limitation. Apply it to hero images and fonts; for this recipe prioritize aspect-ratio and size reservations.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply aspect-ratio and size reservations only to the necessary component boundaries.
- Preserve known intrinsic sizes across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: late content movement.
- Product requirement: Avoid unexpected shifts.
- Focus area: hero images and fonts.
- Technique: aspect-ratio and size reservations.
- Visible result: minimal CLS.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of hero images and fonts and save a baseline screenshot.
2. Apply known intrinsic sizes with actual user-visible data rather than filler.
3. Test aspect-ratio and size reservations at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for hero images and fonts.
5. Disable JavaScript or graphics support and check the fallback for hero images and fonts.
6. Verify using the observable condition: minimal CLS.
7. Record any remaining limitation and associated measured cost.


### Recipe 65 — Image strategy

**Goal:** Avoid wasted bytes.
**Primary method:** responsive image formats.
**Where it applies:** artwork and icons.
**Implementation idea:** srcset and explicit sizing.
**Do not:** megabyte hero thumbnails.
**Acceptance test:** mobile loads right size.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to artwork and icons; for this recipe prioritize srcset and explicit sizing.
2. Define an observable result and list the files or modules that own it. Apply it to artwork and icons; for this recipe prioritize responsive image formats.
3. Implement the smallest complete accessible version first. Apply it to artwork and icons; for this recipe prioritize srcset and explicit sizing.
4. Add quality and motion only after functional behavior is correct. Apply it to artwork and icons; for this recipe prioritize responsive image formats.
5. Instrument, test, and refine against the stated budget. Apply it to artwork and icons; for this recipe prioritize srcset and explicit sizing.
6. Document a fallback, cleanup path, and operational limitation. Apply it to artwork and icons; for this recipe prioritize responsive image formats.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply responsive image formats only to the necessary component boundaries.
- Preserve srcset and explicit sizing across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: megabyte hero thumbnails.
- Product requirement: Avoid wasted bytes.
- Focus area: artwork and icons.
- Technique: responsive image formats.
- Visible result: mobile loads right size.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of artwork and icons and save a baseline screenshot.
2. Apply srcset and explicit sizing with actual user-visible data rather than filler.
3. Test responsive image formats at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for artwork and icons.
5. Disable JavaScript or graphics support and check the fallback for artwork and icons.
6. Verify using the observable condition: mobile loads right size.
7. Record any remaining limitation and associated measured cost.


### Recipe 66 — Content localization

**Goal:** Keep full translations consistent.
**Primary method:** message dictionary.
**Where it applies:** all user strings.
**Implementation idea:** lang and document direction.
**Do not:** mixed-language headings.
**Acceptance test:** no untranslated controls.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all user strings; for this recipe prioritize lang and document direction.
2. Define an observable result and list the files or modules that own it. Apply it to all user strings; for this recipe prioritize message dictionary.
3. Implement the smallest complete accessible version first. Apply it to all user strings; for this recipe prioritize lang and document direction.
4. Add quality and motion only after functional behavior is correct. Apply it to all user strings; for this recipe prioritize message dictionary.
5. Instrument, test, and refine against the stated budget. Apply it to all user strings; for this recipe prioritize lang and document direction.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all user strings; for this recipe prioritize message dictionary.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply message dictionary only to the necessary component boundaries.
- Preserve lang and document direction across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: mixed-language headings.
- Product requirement: Keep full translations consistent.
- Focus area: all user strings.
- Technique: message dictionary.
- Visible result: no untranslated controls.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of all user strings and save a baseline screenshot.
2. Apply lang and document direction with actual user-visible data rather than filler.
3. Test message dictionary at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all user strings.
5. Disable JavaScript or graphics support and check the fallback for all user strings.
6. Verify using the observable condition: no untranslated controls.
7. Record any remaining limitation and associated measured cost.


### Recipe 67 — Animation debugging

**Goal:** Diagnose jitter systematically.
**Primary method:** performance recording.
**Where it applies:** route effects.
**Implementation idea:** trace frame budget and composites.
**Do not:** randomly removing effects.
**Acceptance test:** root cause identified.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to route effects; for this recipe prioritize trace frame budget and composites.
2. Define an observable result and list the files or modules that own it. Apply it to route effects; for this recipe prioritize performance recording.
3. Implement the smallest complete accessible version first. Apply it to route effects; for this recipe prioritize trace frame budget and composites.
4. Add quality and motion only after functional behavior is correct. Apply it to route effects; for this recipe prioritize performance recording.
5. Instrument, test, and refine against the stated budget. Apply it to route effects; for this recipe prioritize trace frame budget and composites.
6. Document a fallback, cleanup path, and operational limitation. Apply it to route effects; for this recipe prioritize performance recording.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply performance recording only to the necessary component boundaries.
- Preserve trace frame budget and composites across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: randomly removing effects.
- Product requirement: Diagnose jitter systematically.
- Focus area: route effects.
- Technique: performance recording.
- Visible result: root cause identified.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of route effects and save a baseline screenshot.
2. Apply trace frame budget and composites with actual user-visible data rather than filler.
3. Test performance recording at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for route effects.
5. Disable JavaScript or graphics support and check the fallback for route effects.
6. Verify using the observable condition: root cause identified.
7. Record any remaining limitation and associated measured cost.


### Recipe 68 — Long-task triage

**Goal:** Reduce interaction stalls.
**Primary method:** DevTools long task inspection.
**Where it applies:** heavy JS.
**Implementation idea:** measure and split offending work.
**Do not:** assume GPU is always culprit.
**Acceptance test:** input responsive under throttle.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to heavy JS; for this recipe prioritize measure and split offending work.
2. Define an observable result and list the files or modules that own it. Apply it to heavy JS; for this recipe prioritize DevTools long task inspection.
3. Implement the smallest complete accessible version first. Apply it to heavy JS; for this recipe prioritize measure and split offending work.
4. Add quality and motion only after functional behavior is correct. Apply it to heavy JS; for this recipe prioritize DevTools long task inspection.
5. Instrument, test, and refine against the stated budget. Apply it to heavy JS; for this recipe prioritize measure and split offending work.
6. Document a fallback, cleanup path, and operational limitation. Apply it to heavy JS; for this recipe prioritize DevTools long task inspection.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply DevTools long task inspection only to the necessary component boundaries.
- Preserve measure and split offending work across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: assume GPU is always culprit.
- Product requirement: Reduce interaction stalls.
- Focus area: heavy JS.
- Technique: DevTools long task inspection.
- Visible result: input responsive under throttle.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of heavy JS and save a baseline screenshot.
2. Apply measure and split offending work with actual user-visible data rather than filler.
3. Test DevTools long task inspection at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for heavy JS.
5. Disable JavaScript or graphics support and check the fallback for heavy JS.
6. Verify using the observable condition: input responsive under throttle.
7. Record any remaining limitation and associated measured cost.


### Recipe 69 — Automated test hooks

**Goal:** Make effects testable.
**Primary method:** deterministic state interfaces.
**Where it applies:** effect lifecycle.
**Implementation idea:** expose start stop destroy.
**Do not:** magic global DOM selectors.
**Acceptance test:** lifecycle unit tests pass.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to effect lifecycle; for this recipe prioritize expose start stop destroy.
2. Define an observable result and list the files or modules that own it. Apply it to effect lifecycle; for this recipe prioritize deterministic state interfaces.
3. Implement the smallest complete accessible version first. Apply it to effect lifecycle; for this recipe prioritize expose start stop destroy.
4. Add quality and motion only after functional behavior is correct. Apply it to effect lifecycle; for this recipe prioritize deterministic state interfaces.
5. Instrument, test, and refine against the stated budget. Apply it to effect lifecycle; for this recipe prioritize expose start stop destroy.
6. Document a fallback, cleanup path, and operational limitation. Apply it to effect lifecycle; for this recipe prioritize deterministic state interfaces.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply deterministic state interfaces only to the necessary component boundaries.
- Preserve expose start stop destroy across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: magic global DOM selectors.
- Product requirement: Make effects testable.
- Focus area: effect lifecycle.
- Technique: deterministic state interfaces.
- Visible result: lifecycle unit tests pass.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of effect lifecycle and save a baseline screenshot.
2. Apply expose start stop destroy with actual user-visible data rather than filler.
3. Test deterministic state interfaces at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for effect lifecycle.
5. Disable JavaScript or graphics support and check the fallback for effect lifecycle.
6. Verify using the observable condition: lifecycle unit tests pass.
7. Record any remaining limitation and associated measured cost.


### Recipe 70 — Visual regression

**Goal:** Catch layout drift.
**Primary method:** reference screenshots.
**Where it applies:** desktop and mobile.
**Implementation idea:** baseline screenshots at breakpoints.
**Do not:** only manual eyeballing.
**Acceptance test:** diff reviewed.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to desktop and mobile; for this recipe prioritize baseline screenshots at breakpoints.
2. Define an observable result and list the files or modules that own it. Apply it to desktop and mobile; for this recipe prioritize reference screenshots.
3. Implement the smallest complete accessible version first. Apply it to desktop and mobile; for this recipe prioritize baseline screenshots at breakpoints.
4. Add quality and motion only after functional behavior is correct. Apply it to desktop and mobile; for this recipe prioritize reference screenshots.
5. Instrument, test, and refine against the stated budget. Apply it to desktop and mobile; for this recipe prioritize baseline screenshots at breakpoints.
6. Document a fallback, cleanup path, and operational limitation. Apply it to desktop and mobile; for this recipe prioritize reference screenshots.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply reference screenshots only to the necessary component boundaries.
- Preserve baseline screenshots at breakpoints across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: only manual eyeballing.
- Product requirement: Catch layout drift.
- Focus area: desktop and mobile.
- Technique: reference screenshots.
- Visible result: diff reviewed.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of desktop and mobile and save a baseline screenshot.
2. Apply baseline screenshots at breakpoints with actual user-visible data rather than filler.
3. Test reference screenshots at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for desktop and mobile.
5. Disable JavaScript or graphics support and check the fallback for desktop and mobile.
6. Verify using the observable condition: diff reviewed.
7. Record any remaining limitation and associated measured cost.


### Recipe 71 — Accessibility regression

**Goal:** Check after animation changes.
**Primary method:** keyboard and reduced motion tests.
**Where it applies:** all routes.
**Implementation idea:** smoke tests for interaction states.
**Do not:** treat a11y as separate work.
**Acceptance test:** all controls reach and announce.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to all routes; for this recipe prioritize smoke tests for interaction states.
2. Define an observable result and list the files or modules that own it. Apply it to all routes; for this recipe prioritize keyboard and reduced motion tests.
3. Implement the smallest complete accessible version first. Apply it to all routes; for this recipe prioritize smoke tests for interaction states.
4. Add quality and motion only after functional behavior is correct. Apply it to all routes; for this recipe prioritize keyboard and reduced motion tests.
5. Instrument, test, and refine against the stated budget. Apply it to all routes; for this recipe prioritize smoke tests for interaction states.
6. Document a fallback, cleanup path, and operational limitation. Apply it to all routes; for this recipe prioritize keyboard and reduced motion tests.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply keyboard and reduced motion tests only to the necessary component boundaries.
- Preserve smoke tests for interaction states across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: treat a11y as separate work.
- Product requirement: Check after animation changes.
- Focus area: all routes.
- Technique: keyboard and reduced motion tests.
- Visible result: all controls reach and announce.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of all routes and save a baseline screenshot.
2. Apply smoke tests for interaction states with actual user-visible data rather than filler.
3. Test keyboard and reduced motion tests at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for all routes.
5. Disable JavaScript or graphics support and check the fallback for all routes.
6. Verify using the observable condition: all controls reach and announce.
7. Record any remaining limitation and associated measured cost.


### Recipe 72 — Performance regression

**Goal:** Prevent added effects from bloating cost.
**Primary method:** documented baseline budget.
**Where it applies:** each feature PR.
**Implementation idea:** measure before vs after.
**Do not:** assume tiny shader is free.
**Acceptance test:** budget changes explicit.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to each feature PR; for this recipe prioritize measure before vs after.
2. Define an observable result and list the files or modules that own it. Apply it to each feature PR; for this recipe prioritize documented baseline budget.
3. Implement the smallest complete accessible version first. Apply it to each feature PR; for this recipe prioritize measure before vs after.
4. Add quality and motion only after functional behavior is correct. Apply it to each feature PR; for this recipe prioritize documented baseline budget.
5. Instrument, test, and refine against the stated budget. Apply it to each feature PR; for this recipe prioritize measure before vs after.
6. Document a fallback, cleanup path, and operational limitation. Apply it to each feature PR; for this recipe prioritize documented baseline budget.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply documented baseline budget only to the necessary component boundaries.
- Preserve measure before vs after across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: assume tiny shader is free.
- Product requirement: Prevent added effects from bloating cost.
- Focus area: each feature PR.
- Technique: documented baseline budget.
- Visible result: budget changes explicit.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of each feature PR and save a baseline screenshot.
2. Apply measure before vs after with actual user-visible data rather than filler.
3. Test documented baseline budget at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for each feature PR.
5. Disable JavaScript or graphics support and check the fallback for each feature PR.
6. Verify using the observable condition: budget changes explicit.
7. Record any remaining limitation and associated measured cost.


### Recipe 73 — Security boundaries

**Goal:** Never trust UI text from users.
**Primary method:** textContent and sanitization.
**Where it applies:** chat and forms.
**Implementation idea:** escape user strings.
**Do not:** innerHTML injection.
**Acceptance test:** payloads render as text.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to chat and forms; for this recipe prioritize escape user strings.
2. Define an observable result and list the files or modules that own it. Apply it to chat and forms; for this recipe prioritize textContent and sanitization.
3. Implement the smallest complete accessible version first. Apply it to chat and forms; for this recipe prioritize escape user strings.
4. Add quality and motion only after functional behavior is correct. Apply it to chat and forms; for this recipe prioritize textContent and sanitization.
5. Instrument, test, and refine against the stated budget. Apply it to chat and forms; for this recipe prioritize escape user strings.
6. Document a fallback, cleanup path, and operational limitation. Apply it to chat and forms; for this recipe prioritize textContent and sanitization.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply textContent and sanitization only to the necessary component boundaries.
- Preserve escape user strings across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: innerHTML injection.
- Product requirement: Never trust UI text from users.
- Focus area: chat and forms.
- Technique: textContent and sanitization.
- Visible result: payloads render as text.
- Performance guard: Cost: unexpected extra browser work or memory allocation.
- Interaction guard: State: rapid repeated input must not leave stale DOM or promises.
- Responsive guard: Responsiveness: inspect both 320px and 1440px layouts.
- Cleanup guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.

#### QA exercise

1. Inspect the initial state of chat and forms and save a baseline screenshot.
2. Apply escape user strings with actual user-visible data rather than filler.
3. Test textContent and sanitization at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for chat and forms.
5. Disable JavaScript or graphics support and check the fallback for chat and forms.
6. Verify using the observable condition: payloads render as text.
7. Record any remaining limitation and associated measured cost.


### Recipe 74 — Production errors

**Goal:** Expose real operational failures.
**Primary method:** logging with context.
**Where it applies:** runtime interactions.
**Implementation idea:** redact private input.
**Do not:** swallow errors silently.
**Acceptance test:** clear error paths.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to runtime interactions; for this recipe prioritize redact private input.
2. Define an observable result and list the files or modules that own it. Apply it to runtime interactions; for this recipe prioritize logging with context.
3. Implement the smallest complete accessible version first. Apply it to runtime interactions; for this recipe prioritize redact private input.
4. Add quality and motion only after functional behavior is correct. Apply it to runtime interactions; for this recipe prioritize logging with context.
5. Instrument, test, and refine against the stated budget. Apply it to runtime interactions; for this recipe prioritize redact private input.
6. Document a fallback, cleanup path, and operational limitation. Apply it to runtime interactions; for this recipe prioritize logging with context.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply logging with context only to the necessary component boundaries.
- Preserve redact private input across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: swallow errors silently.
- Product requirement: Expose real operational failures.
- Focus area: runtime interactions.
- Technique: logging with context.
- Visible result: clear error paths.
- Performance guard: Accessibility: content must remain usable without hover or animation.
- Interaction guard: Responsiveness: inspect both 320px and 1440px layouts.
- Responsive guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Cleanup guard: Robustness: handle missing nodes and browser unsupported APIs.

#### QA exercise

1. Inspect the initial state of runtime interactions and save a baseline screenshot.
2. Apply redact private input with actual user-visible data rather than filler.
3. Test logging with context at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for runtime interactions.
5. Disable JavaScript or graphics support and check the fallback for runtime interactions.
6. Verify using the observable condition: clear error paths.
7. Record any remaining limitation and associated measured cost.


### Recipe 75 — Motion prototyping

**Goal:** Compare static and animated frames.
**Primary method:** rough prototypes.
**Where it applies:** new hero effect.
**Implementation idea:** verify still state first.
**Do not:** shader before layout.
**Acceptance test:** identity stands without motion.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to new hero effect; for this recipe prioritize verify still state first.
2. Define an observable result and list the files or modules that own it. Apply it to new hero effect; for this recipe prioritize rough prototypes.
3. Implement the smallest complete accessible version first. Apply it to new hero effect; for this recipe prioritize verify still state first.
4. Add quality and motion only after functional behavior is correct. Apply it to new hero effect; for this recipe prioritize rough prototypes.
5. Instrument, test, and refine against the stated budget. Apply it to new hero effect; for this recipe prioritize verify still state first.
6. Document a fallback, cleanup path, and operational limitation. Apply it to new hero effect; for this recipe prioritize rough prototypes.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply rough prototypes only to the necessary component boundaries.
- Preserve verify still state first across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: shader before layout.
- Product requirement: Compare static and animated frames.
- Focus area: new hero effect.
- Technique: rough prototypes.
- Visible result: identity stands without motion.
- Performance guard: State: rapid repeated input must not leave stale DOM or promises.
- Interaction guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Responsive guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Cleanup guard: Validation: compare before/after behavior with a concrete checklist.

#### QA exercise

1. Inspect the initial state of new hero effect and save a baseline screenshot.
2. Apply verify still state first with actual user-visible data rather than filler.
3. Test rough prototypes at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for new hero effect.
5. Disable JavaScript or graphics support and check the fallback for new hero effect.
6. Verify using the observable condition: identity stands without motion.
7. Record any remaining limitation and associated measured cost.


### Recipe 76 — Card alternatives

**Goal:** Avoid tile-grid default.
**Primary method:** editorial columns and lists.
**Where it applies:** feature pages.
**Implementation idea:** content-led layouts.
**Do not:** same six cards.
**Acceptance test:** structure matches information.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to feature pages; for this recipe prioritize content-led layouts.
2. Define an observable result and list the files or modules that own it. Apply it to feature pages; for this recipe prioritize editorial columns and lists.
3. Implement the smallest complete accessible version first. Apply it to feature pages; for this recipe prioritize content-led layouts.
4. Add quality and motion only after functional behavior is correct. Apply it to feature pages; for this recipe prioritize editorial columns and lists.
5. Instrument, test, and refine against the stated budget. Apply it to feature pages; for this recipe prioritize content-led layouts.
6. Document a fallback, cleanup path, and operational limitation. Apply it to feature pages; for this recipe prioritize editorial columns and lists.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply editorial columns and lists only to the necessary component boundaries.
- Preserve content-led layouts across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: same six cards.
- Product requirement: Avoid tile-grid default.
- Focus area: feature pages.
- Technique: editorial columns and lists.
- Visible result: structure matches information.
- Performance guard: Responsiveness: inspect both 320px and 1440px layouts.
- Interaction guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Responsive guard: Robustness: handle missing nodes and browser unsupported APIs.
- Cleanup guard: Cost: unexpected extra browser work or memory allocation.

#### QA exercise

1. Inspect the initial state of feature pages and save a baseline screenshot.
2. Apply content-led layouts with actual user-visible data rather than filler.
3. Test editorial columns and lists at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for feature pages.
5. Disable JavaScript or graphics support and check the fallback for feature pages.
6. Verify using the observable condition: structure matches information.
7. Record any remaining limitation and associated measured cost.


### Recipe 77 — Dark themes

**Goal:** Make dark refined, not muddy.
**Primary method:** layered neutrals.
**Where it applies:** site surfaces.
**Implementation idea:** controlled contrast and one accent.
**Do not:** pure black plus neon.
**Acceptance test:** text remains comfortable.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to site surfaces; for this recipe prioritize controlled contrast and one accent.
2. Define an observable result and list the files or modules that own it. Apply it to site surfaces; for this recipe prioritize layered neutrals.
3. Implement the smallest complete accessible version first. Apply it to site surfaces; for this recipe prioritize controlled contrast and one accent.
4. Add quality and motion only after functional behavior is correct. Apply it to site surfaces; for this recipe prioritize layered neutrals.
5. Instrument, test, and refine against the stated budget. Apply it to site surfaces; for this recipe prioritize controlled contrast and one accent.
6. Document a fallback, cleanup path, and operational limitation. Apply it to site surfaces; for this recipe prioritize layered neutrals.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply layered neutrals only to the necessary component boundaries.
- Preserve controlled contrast and one accent across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: pure black plus neon.
- Product requirement: Make dark refined, not muddy.
- Focus area: site surfaces.
- Technique: layered neutrals.
- Visible result: text remains comfortable.
- Performance guard: Performance: measure in DevTools; avoid guessed FPS claims.
- Interaction guard: Robustness: handle missing nodes and browser unsupported APIs.
- Responsive guard: Validation: compare before/after behavior with a concrete checklist.
- Cleanup guard: Accessibility: content must remain usable without hover or animation.

#### QA exercise

1. Inspect the initial state of site surfaces and save a baseline screenshot.
2. Apply controlled contrast and one accent with actual user-visible data rather than filler.
3. Test layered neutrals at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for site surfaces.
5. Disable JavaScript or graphics support and check the fallback for site surfaces.
6. Verify using the observable condition: text remains comfortable.
7. Record any remaining limitation and associated measured cost.


### Recipe 78 — Pixel precision

**Goal:** Align iconography and typographic edges.
**Primary method:** grid metrics and optical adjustments.
**Where it applies:** navbar and controls.
**Implementation idea:** shared baselines.
**Do not:** magic negative margins everywhere.
**Acceptance test:** stable across fonts.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to navbar and controls; for this recipe prioritize shared baselines.
2. Define an observable result and list the files or modules that own it. Apply it to navbar and controls; for this recipe prioritize grid metrics and optical adjustments.
3. Implement the smallest complete accessible version first. Apply it to navbar and controls; for this recipe prioritize shared baselines.
4. Add quality and motion only after functional behavior is correct. Apply it to navbar and controls; for this recipe prioritize grid metrics and optical adjustments.
5. Instrument, test, and refine against the stated budget. Apply it to navbar and controls; for this recipe prioritize shared baselines.
6. Document a fallback, cleanup path, and operational limitation. Apply it to navbar and controls; for this recipe prioritize grid metrics and optical adjustments.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply grid metrics and optical adjustments only to the necessary component boundaries.
- Preserve shared baselines across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: magic negative margins everywhere.
- Product requirement: Align iconography and typographic edges.
- Focus area: navbar and controls.
- Technique: grid metrics and optical adjustments.
- Visible result: stable across fonts.
- Performance guard: Lifecycle: cancel observers, frames, timers, and listeners on teardown.
- Interaction guard: Validation: compare before/after behavior with a concrete checklist.
- Responsive guard: Cost: unexpected extra browser work or memory allocation.
- Cleanup guard: State: rapid repeated input must not leave stale DOM or promises.

#### QA exercise

1. Inspect the initial state of navbar and controls and save a baseline screenshot.
2. Apply shared baselines with actual user-visible data rather than filler.
3. Test grid metrics and optical adjustments at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for navbar and controls.
5. Disable JavaScript or graphics support and check the fallback for navbar and controls.
6. Verify using the observable condition: stable across fonts.
7. Record any remaining limitation and associated measured cost.


### Recipe 79 — Interaction audit

**Goal:** Expose every dead end.
**Primary method:** click and keyboard walkthrough.
**Where it applies:** complete application.
**Implementation idea:** failure feedback and meaningful actions.
**Do not:** decorative fake buttons.
**Acceptance test:** every visible affordance works.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to complete application; for this recipe prioritize failure feedback and meaningful actions.
2. Define an observable result and list the files or modules that own it. Apply it to complete application; for this recipe prioritize click and keyboard walkthrough.
3. Implement the smallest complete accessible version first. Apply it to complete application; for this recipe prioritize failure feedback and meaningful actions.
4. Add quality and motion only after functional behavior is correct. Apply it to complete application; for this recipe prioritize click and keyboard walkthrough.
5. Instrument, test, and refine against the stated budget. Apply it to complete application; for this recipe prioritize failure feedback and meaningful actions.
6. Document a fallback, cleanup path, and operational limitation. Apply it to complete application; for this recipe prioritize click and keyboard walkthrough.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply click and keyboard walkthrough only to the necessary component boundaries.
- Preserve failure feedback and meaningful actions across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: decorative fake buttons.
- Product requirement: Expose every dead end.
- Focus area: complete application.
- Technique: click and keyboard walkthrough.
- Visible result: every visible affordance works.
- Performance guard: Robustness: handle missing nodes and browser unsupported APIs.
- Interaction guard: Cost: unexpected extra browser work or memory allocation.
- Responsive guard: Accessibility: content must remain usable without hover or animation.
- Cleanup guard: Responsiveness: inspect both 320px and 1440px layouts.

#### QA exercise

1. Inspect the initial state of complete application and save a baseline screenshot.
2. Apply failure feedback and meaningful actions with actual user-visible data rather than filler.
3. Test click and keyboard walkthrough at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for complete application.
5. Disable JavaScript or graphics support and check the fallback for complete application.
6. Verify using the observable condition: every visible affordance works.
7. Record any remaining limitation and associated measured cost.


### Recipe 80 — Release checklist

**Goal:** Deliver verified frontend, not a promise.
**Primary method:** build and smoke tests.
**Where it applies:** deployment.
**Implementation idea:** document tests and limitations.
**Do not:** claim tested without running.
**Acceptance test:** reproducible release evidence.

#### Procedure

1. Discover existing behavior and identify non-negotiable user constraints. Apply it to deployment; for this recipe prioritize document tests and limitations.
2. Define an observable result and list the files or modules that own it. Apply it to deployment; for this recipe prioritize build and smoke tests.
3. Implement the smallest complete accessible version first. Apply it to deployment; for this recipe prioritize document tests and limitations.
4. Add quality and motion only after functional behavior is correct. Apply it to deployment; for this recipe prioritize build and smoke tests.
5. Instrument, test, and refine against the stated budget. Apply it to deployment; for this recipe prioritize document tests and limitations.
6. Document a fallback, cleanup path, and operational limitation. Apply it to deployment; for this recipe prioritize build and smoke tests.

#### Implementation contract

- Capture initial state, any user input and the required resulting state.
- Define the DOM elements and ARIA semantics before animations or decorators.
- Apply build and smoke tests only to the necessary component boundaries.
- Preserve document tests and limitations across desktop, touch and keyboard input.
- Ensure CSS has a fallback when animation or JavaScript does not load.
- Keep state changes independent of frame rate and browser scheduling.
- Use explicit ownership for resources, timers, observers and GPU objects.
- Prefer composited transforms; avoid layout work on every animation frame.
- Disable ambient effects when their owning surface is outside the viewport.
- Never use fake statistics, fabricated errors or placeholder actions in shipped UI.

#### Review against common failures

- Failure to avoid: claim tested without running.
- Product requirement: Deliver verified frontend, not a promise.
- Focus area: deployment.
- Technique: build and smoke tests.
- Visible result: reproducible release evidence.
- Performance guard: Validation: compare before/after behavior with a concrete checklist.
- Interaction guard: Accessibility: content must remain usable without hover or animation.
- Responsive guard: State: rapid repeated input must not leave stale DOM or promises.
- Cleanup guard: Performance: measure in DevTools; avoid guessed FPS claims.

#### QA exercise

1. Inspect the initial state of deployment and save a baseline screenshot.
2. Apply document tests and limitations with actual user-visible data rather than filler.
3. Test build and smoke tests at narrow, medium and wide widths.
4. Interrupt the interaction mid-flight; ensure valid final state for deployment.
5. Disable JavaScript or graphics support and check the fallback for deployment.
6. Verify using the observable condition: reproducible release evidence.
7. Record any remaining limitation and associated measured cost.

## Final sign-off

- A coherent static screen exists before any animated enhancement.
- All meaningful interactions work by mouse, touch and keyboard.
- Reduced motion, visibility suspension and teardown are implemented where needed.
- No copied branding, shaders or artwork from the reference website.
- No marketing filler, generic feature-card pile or fake functionality.
- Mobile visual hierarchy is intentional rather than squeezed desktop.
- DevTools measurements and smoke tests support performance claims.
- Output includes changed files, test evidence and known limitations.
