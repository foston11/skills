# Motion, navigation and visual composition

## Art direction without template slop

Begin with a strong visual structure. ToolMC's supplied HTML uses a split architectural grid, large typographic hero, a distinct visual area and a compact text navigation rather than a standard feature-card pile. Treat those as transferable layout **ideas**, not a template to duplicate.

Choose a site-specific design language:
- Typographic scale with confident display headings and restrained supporting copy.
- Grid lines, aligned margins, proportion and whitespace provide rhythm.
- A single focal visual such as a globe, shader field, hero illustration or dimensional object.
- Precise rather than excessive decorative elements.
- Desktop design remains intentional at 320–430px widths, not merely squeezed.

For every section ask: can this be removed, shortened or integrated into the layout? Avoid invented feature claims and marketing filler.

## Navigation state machine

Use explicit states: `idle`, `exit`, `swap`, `enter`. A new navigation request either supersedes the current one safely or queues until state is idle. Never allow accidental double-transition or an invisible clickable overlay.

```js
let phase = 'idle';
let target = null;

async function navigate(nextPage) {
  if (phase !== 'idle' || nextPage === target) return;
  phase = 'exit';
  await animateExit(); // WAAPI or transitionend, not arbitrary sleeps
  phase = 'swap';
  showPage(nextPage);
  target = nextPage;
  phase = 'enter';
  await animateEnter();
  phase = 'idle';
}
```

Implement `animateExit` and `animateEnter` using explicit completed/cancelled outcomes. Handle reduced motion via immediate resolved transitions. Protect against unmounted DOM elements.

## Easing and timing

Use a small motion scale, not fifty unrelated delays:
- response: ~100–180 ms
- control and tab change: ~180–300 ms
- section entrance: ~350–650 ms
- large page choreography: ~500–900 ms if justified

Values are design starting points. Long animations are *not automatically* premium. Prefer coherent easing over exaggerated bounce. A useful smooth ease is `cubic-bezier(.22, 1, .36, 1)`; use linear for steady ambient rotation.

Animate `transform` and `opacity` where possible. Avoid large animated blur, shadow, backdrop-filter, height and layout geometry. Don't apply `will-change` globally; use it briefly when necessary.

## Animation orchestration

For meaningful page changes:
1. Acknowledge interaction immediately.
2. Fade or shift exiting content.
3. Swap route visibility and update accessibility state.
4. Reveal incoming focal object and typography using the same motion language.
5. Release pointer/keyboard interaction locks, restore sensible focus.
6. Stop the previous page's canvas/render loop and activate the new one.

Test interrupted navigation and fast repeated clicking. Respect `prefers-reduced-motion: reduce` for ALL motion systems, including JS, WebGL and Three.js; CSS alone is insufficient.

## Interaction requirements

- Buttons use actual `button`, links actual `a`; never rely only on onclick-divs.
- Header and tab transitions expose current state through `aria-current` or equivalent.
- Copy-to-clipboard buttons have success/failure feedback; use a fallback or explain insecure-context limits.
- Hover has a keyboard focus-visible and touch equivalent.
- Escape dismisses overlays; focus never gets stranded behind them.
- Sound is muted until explicit opt-in; give honest status if unavailable.
- Text and UI labels stay consistent in the selected language.
- Interaction feedback should be visible but restrained.

## Reviewing third-party code

In the ToolMC snapshot:
- `intro.js` handles opening choreography.
- `main.js` coordinates sections, page switching and more visual work.
- `ranks-hero.js` draws a dedicated interactive rank hero.
- `aurora.js`, `dither-veil.js`, and `grainient.js` implement procedural visual layers.
- `globe.js` and `crown.js` use Three.js for 3D scenes.
- `community.js` implements interactive visual behavior.

For your own site, split by responsibility but avoid scattering route state across unrelated modules. Prefer one navigation orchestrator, effect lifecycle interfaces and shared motion tokens. Document who owns each canvas and observer.

## Review checklist

- [ ] Would the static page still look professionally composed?
- [ ] Does each animation serve hierarchy, continuity, or input feedback?
- [ ] Are route transitions smooth, cancel-safe and keyboard-friendly?
- [ ] Are loading, failure, disabled and reduced-motion states complete?
- [ ] Does mobile have a considered art direction?
- [ ] Are extra paragraphs, icons, cards, separators and decorations justified?
- [ ] Do visuals stop consuming GPU time when hidden?
- [ ] Is there real evidence that the interactive experience works?
