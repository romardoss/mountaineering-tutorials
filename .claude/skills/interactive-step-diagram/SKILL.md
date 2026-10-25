---
name: interactive-step-diagram
description: Build an interactive, step-by-step animated SVG explainer page (e.g. rope crossing tactics, rescue procedures, knot sequences, any physical process) where the viewer switches between steps and the scene animates between states.
---

# Interactive step-by-step diagram page

Use this when someone wants a procedure they already have (or you just wrote) turned into a visual page where each step is a scene state and the viewer can move back and forth between steps. The reference example is a 4-person Tyrolean / navisna crossing over a ravine: one SVG scene, people and ropes animate between 11 steps, text panel below.

## Output

1. One self-contained `.html` file (inline CSS + JS, only Google Fonts external), published as an Artifact.
2. Page language = the language of the conversation. All step text, labels and buttons in that language.

## Workflow

1. **Get the steps first.** Reuse the step list from the conversation if it exists. Each step needs: short title, 1–3 sentence description, who acts, optional safety note.
2. **Design the scene once.** Decide the fixed world (terrain, anchors, objects) and the moving parts (actors, ropes/lines, tools, labels). Sketch coordinates in a `viewBox` (e.g. `0 0 1000 520`) before coding.
3. **Model every step as data**, not as code branches (see "Data model").
4. **Write one `render()`** that reads the current step and sets the state of every moving part. Moving parts animate via CSS transitions.
5. **Check visually**: open the file headless (Playwright), click 2–3 steps, screenshot, read the screenshot, fix overlaps and misplaced elements. Also collect `pageerror` events.
6. **Register in the catalog**: save the page in the project root and add one object to the `ITEMS` array in `index.html` — array order is display order (ascending), so append at the end unless the user says where — (`group`, `file`, `title`, `desc`, `tags`, `steps`, `date` as `YYYY-MM-DD`, `draft:true`). Every new page starts as `draft:true` (shows a "still in testing" warning); remove the flag only when the user explicitly says the page passed human review.
7. **Publish** with the Artifact tool and reply with one short paragraph on how to use it.

## Data model

```js
const ROLES = { 1:{name:"Leader", color:"var(--p1)"}, 2:{...}, 3:{...}, 4:{...} };

const S = x => [x, GROUND_Y, "s"];          // standing on ground
const H = (x, off=0) => [x, ropeY(x)+off, "h"]; // hanging on the rope at x

const STEPS = [
  { t:"Title", d:"Description", who:[1],      // who acts → highlighted
    pos:{1:S(750), 2:S(140), 3:S(100), 4:S(58)}, // EVERY actor in EVERY step
    rope:"laid-p1",            // key into PATHS, or "none"
    support:{l:262, p:2, r:860}, // or a named preset, or null
    trail:true, poly:true, carR:true,         // boolean toggles for props
    notes:[{x:500, y:440, t:"Label", a:"middle"}],
    safety:"Optional safety reminder" },
];
```

Rules:
1. Every step lists the position of every actor, so jumping to any step (not only next/prev) gives a correct scene.
2. Line shapes live in a `PATHS` map keyed by name; steps reference names.
3. Props (pulleys, carabiners, coils, arrows) are pre-drawn SVG groups toggled by opacity.
4. Curve helpers compute positions on curved lines, e.g. for a quadratic sag `M185 180 Q 500 250 815 180`: `t=(x-185)/630; y=(1-t)²·180 + 2(1-t)t·250 + t²·180`. Hanging actors are placed with that, so they sit exactly on the rope.
5. Lines connecting to an actor (support rope) are computed from the actor's position in that step.

## Scene construction

1. Static layers in order: sky gradient, distant hills, terrain/ravine polygons, anchors (trees), then trail, ropes, props, people, notes (labels on top).
2. Actors are `<g class="person">` positioned with `style.transform = translate(Xpx, Ypx)`; origin = feet (standing) or attachment point (hanging).
3. Each actor has two poses inside (`.pose.stand`, `.pose.hang`) crossfaded by opacity, a colored number badge, and a `.halo` ellipse shown when the actor is in `who`.
4. Ropes are `<path class="ropeline">` with `transition: d .9s` (animates in Chromium/Firefox, snaps in Safari, which is acceptable). Dashed stroke for a rope being retrieved, `marker-end` arrows for direction.
5. Labels: SVG `<text>` with `paint-order: stroke` and a background-colored stroke so they stay readable over terrain.
6. All colors come from CSS custom properties so dark mode works inside the SVG too.

## UI around the scene

1. Header: title + one line on how to navigate.
2. Scene in a bordered panel, `width:100%; height:auto`.
3. Controls: "Back" / "Next" (Next becomes "Start over" on the last step), counter "Step N of M".
4. Step rail: one clickable segment per step with its number (`aria-current="step"` on the current one, completed steps tinted).
   Controls and step rail live together in a `.dock` placed as the last child of the page wrapper, with `position:sticky; bottom:0`, a translucent blurred background and a top border, so they stay pinned to the bottom of the screen while scrolling.
5. Info area: numbered step title, description (max ~62ch), "who acts" chips with role color and name, safety note in a tinted box (hidden when empty). Side panel: legend of actors and line types.
6. Keyboard: ←/→ for steps, Home/End for first/last.
7. Remember the current step in `localStorage` (wrapped in try/catch).
8. `aria-live="polite"` on the text section so screen readers announce the step.

## Visual style

1. Pick a palette from the subject's world (for outdoor scenes: sky, earth, moss, a bright rope color, a contrasting second-rope color) and 4 clearly distinct actor colors.
2. Typography: a characterful display face + a readable body face, both with the needed script (for Ukrainian: Unbounded + Golos Text).
3. Avoid generic AI-page tells: no cream + terracotta, no all-caps eyebrows, no identical card grids.
4. Dark mode tokens under `@media (prefers-color-scheme: dark)` guarded by `:root:not([data-theme="light"])`, repeated for `:root[data-theme="dark"]`; explicit `body` background.
5. `@media (prefers-reduced-motion: reduce)` disables all transitions.
6. Responsive: info grid collapses to one column under ~820px.

## Skeleton

```html
<svg viewBox="0 0 1000 520">
  <!-- static world ... -->
  <path id="trail" class="fade" .../>
  <g id="ropes"><path id="support" class="ropeline"/><path id="rope" class="ropeline"/></g>
  <g id="poly" class="fade" opacity="0">...</g>
  <g id="people"></g>   <!-- built in JS -->
  <g id="notes" class="note"></g>
</svg>
<script>
function render(){
  const s = STEPS[cur];
  for (const id in personEls){
    const [x,y,mode] = s.pos[id], p = personEls[id];
    p.g.style.transform = `translate(${x}px, ${y}px)`;
    p.stand.setAttribute("opacity", mode==="s"?1:0);
    p.hang.setAttribute("opacity",  mode==="h"?1:0);
    p.g.classList.toggle("active", s.who.includes(+id));
  }
  setPath("rope", s.rope);              // PATHS[s.rope] or hide
  setPath("support", supportPath(s));    // computed from actor positions
  toggle("trail", s.trail); toggle("poly", s.poly);
  drawNotes(s.notes); fillTextPanel(s); updateControls();
}
</script>
```

## Verification checklist

1. No JS errors in headless run.
2. Screenshot at least one "standing" step and one "hanging/moving" step; actors sit on the ground or exactly on the line, labels don't overlap actors.
3. Jump directly to the last step from the first: scene is correct (proves every step is complete data).
4. Check a narrow viewport (~390px): no horizontal scroll, text readable.
