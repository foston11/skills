---
name: cinematic-web-frontend
description: Build distinctive, production-quality interactive websites with cinematic transitions, disciplined art direction, responsive typography, canvas/WebGL effects, and measurable performance. Use when designing or refactoring landing pages, immersive product sites, portfolios, game/community sites, motion systems, WebGL scenes, or when a user rejects generic AI-generated layouts.
---

# Cinematic Web Frontend

Create a cohesive experience, not a collection of decorative components. This skill distills **implementation techniques observed in a public ToolMC frontend snapshot** into reusable design and engineering rules. It does not reproduce ToolMC branding, copy, assets, or proprietary shaders. Adapt the techniques to the user's product.

## Non-negotiable standards

1. **Design before coding.** Decide the visual thesis (one sentence), typographic hierarchy, grid, contrast, interaction metaphor, and motion language. Use the user's existing art direction when provided.
2. **No template slop.** Do not default to six rounded feature cards, floating gradients, generic hero stock art, oversized pills, glassmorphism, random neon, filler marketing copy, or fake metrics. Cards are allowed only if the information benefits from card grouping.
3. **Motion communicates state.** Every animated element must indicate navigation, reveal hierarchy, respond to intentional input, or create controlled atmosphere. Decorative effects never obstruct text or controls.
4. **One visual system.** Define tokens for color, typography, spacing, durations, easing, and z-layers. Unify transitions across pages, header, and controls.
5. **Real functionality.** Links, navigation, sound toggles, copy buttons, menus, and gestures must work; don't simulate a backend or claim live data without an API.
6. **Fast by construction.** Implement graceful static and reduced-motion fallbacks from the start. Measure performance rather than promising arbitrary FPS.
7. **No unnecessary dependency.** Prefer semantic HTML, CSS, and small vanilla JS when the project permits. Use Three.js or WebGL only when the visual effect warrants the cost.

## Workflow

### 1. Inspect the project

Read routes, markup, CSS, JS, libraries, assets, fonts, and build configuration. Map:
- sections and responsibilities
- interaction states and navigation
- canvas/WebGL instances and their ownership
- frame loops, timers, observers, and event listeners
- performance-critical effects
- device and accessibility constraints

If inheriting an existing project, preserve functionality and styling intent; don't rewrite large systems for the sake of rewriting.

### 2. Write a concise design contract

Specify:
- concept: distinctive visual identity and one primary metaphor
- layout: asymmetric or deliberate grid, restrained surfaces, strong whitespace
- typography: display and body roles, intentional line breaks, monospace only where meaningful
- palette: 1 dominant environment, 1 controlled accent, neutral hierarchy
- interaction: hover, focus, pressed, loading, page changes, escape/close, touch
- motion: entrance, transition, ambient, interaction feedback
- budgets: active contexts, DPR limits, target devices, fallback behavior

Use real text and real product content. Strip placeholder labels, unnecessary subtitles, decorative punctuation, and boilerplate paragraphs.

### 3. Build the static experience first

Implement semantic landmarks, sensible heading order, keyboard navigation, responsive layout, real copy and meaningful states. Design a compelling *still frame* before adding a shader. If disabling animation breaks comprehension, fix the interface.

### 4. Add motion in layers

1. Microinteractions: CSS transform and opacity, short and subtle.
2. Section reveals: IntersectionObserver, run once where appropriate.
3. Navigation: explicit state machine (idle → exiting → switching → entering → idle); avoid conflicting transitions.
4. Atmosphere: one bounded canvas/WebGL effect; add others only when their visual contribution outweighs cost.
5. 3D: initialize lazily, render only when needed, release resources on teardown.

Never coordinate transitions with guessed delays alone. Prefer transitionend/animationend with a guarded timeout fallback.

### 5. Engineer render loops correctly

Use a single owner per render surface. Track animation frame ID and actual visibility. Pause when offscreen, hidden, reduced-motion, or a route is inactive. On resume reset the last-frame timestamp to avoid jumps. Resize with a ResizeObserver, debounce/batch resize through rAF and cap both DPR and pixel area.

Read [references/rendering-and-performance.md](references/rendering-and-performance.md) before implementing canvas/WebGL. Read [references/motion-and-interaction.md](references/motion-and-interaction.md) before implementing navigation or complex motion.

### 6. Verify the result

Test desktop, touch, narrow phones, slow GPUs, reduced motion, keyboard-only navigation, and page hidden/visible transitions. Check browser console and failed network requests. Verify contrast, focus states, hit targets, correct language, clean text wrapping, layout shifts, and real functionality.

Report what was changed, what was measured, and what remains unverified. Never claim "60 FPS" without actual measurement on the target class of device.

## ToolMC-derived implementation patterns

The supplied snapshot contains separate modules named `aurora.js`, `dither-veil.js`, `grainient.js`, `globe.js`, `crown.js`, `intro.js`, `ranks-hero.js`, `community.js`, and `main.js`. They demonstrate:

- WebGL2 fragment shaders for procedural backgrounds and transitions
- Three.js scenes for dimensional focal objects
- IntersectionObserver and ResizeObserver driving visibility and sizing
- requestAnimationFrame loops with interpolation and pointer response
- a coordinated hero, navigation, intro and rank-transition experience
- devicePixelRatio clamping, reduced-motion checks in selected modules

**Learn from the architecture, not the exact artwork.** The snapshot also shows optimization opportunities: some 3D loops schedule frames continuously while hidden, multiple simultaneously active canvases can compete for GPU time, and CDN Tailwind plus multiple overrides inflate operational complexity. Do not imitate these weaknesses.

## Decision rules

| Need | Default | Escalate to |
| --- | --- | --- |
| Button / tab transition | CSS transform + opacity | WAAPI for sequencing |
| Scroll reveal | IntersectionObserver + CSS | JS timeline only for linked choreography |
| Noise / atmospheric overlay | static or CSS texture | 2D Canvas / fragment shader if needed |
| Organic aurora | CSS first | WebGL2 fragment shader with fallback |
| 3D object | pre-rendered visual if sufficient | lazy Three.js scene |
| Route transition | state-machine + CSS/WAAPI | canvas morph only when justified |
| Pointer parallax | CSS custom properties or rAF throttling | shader uniforms if already WebGL |
| Background audio | OFF by default, explicit user action | Web Audio with lifecycle cleanup |

## Acceptance criteria

- Visually distinctive layout, typography, spacing, and clear information hierarchy.
- No needless cards, generic AI-copy, or ornamental effects that compete with content.
- Every animation has a reason; no jump cuts or simultaneous contradictory transitions.
- No animation loop runs while its surface is inactive, unless documented and necessary.
- Reduced-motion mode is usable and largely still.
- Static fallback works without WebGL and when JS fails where feasible.
- Page works on desktop and mobile, with pointer and keyboard interactions.
- Performance validated with DevTools Performance and at least one constrained device profile.
- No broken links/buttons, console errors, uncontrolled memory growth, or fake UI.

## Output discipline

When implementing, supply functioning code and concrete files, not a moodboard or a prompt disguised as work. Keep external research/references as inspiration; do not clone branding or ship someone's third-party code without verifying license and permission.
