---
name: frontend-design
description: Visual and interaction design guidance for web UI — pages, components, dashboards, landing pages, apps, and styling/redesign work. Framework-agnostic (any UI framework or plain HTML/CSS). Use when building or restyling any user-facing interface, so the result is distinctive, accessible, responsive, and consistent with the product.
license: Complete terms in LICENSE.txt. Modified from Anthropic's original frontend-design skill.
---

# Frontend Design

Build interfaces that are genuinely designed — with a clear point of view and polished details — and that are also accessible, responsive, and fast. This guidance is framework-agnostic: implement it with whatever the project uses (any component framework, plain HTML/CSS/JS, any styling approach). For code organization and TypeScript conventions, also apply `code-structure` and `typescript-style`.

## 1. Pick the Mode First

**Existing product (there is a design system, theme, component library, or established styling):**

- Consistency wins. Use the existing tokens (colors, spacing, typography, radii, shadows), components, and patterns. Do not introduce new fonts, palettes, or styling approaches.
- Look at neighbouring screens before designing a new one; the new UI should feel like it was always there.
- Distinctiveness comes from craft inside the system: hierarchy, spacing, rhythm, states, micro-interactions.
- If the system is missing something you need, extend it in its own style (new token/variant), do not bypass it with one-off values.

**Greenfield (new project, standalone page, prototype, or the user explicitly asks for a new look):**

- Commit to a bold, specific aesthetic direction, described below.

If it is unclear which mode applies, check the codebase; if still unclear, ask.

## 2. Design Thinking (greenfield)

Before coding, decide and briefly state:

- **Purpose and audience**: what the interface does and who uses it.
- **Tone**: choose a specific direction — e.g. brutally minimal, editorial/magazine, retro-futuristic, organic, luxury/refined, playful, brutalist, art deco, soft/pastel, industrial/utilitarian — or invent one that fits the context.
- **Signature**: the one memorable thing about this design (a typographic treatment, a layout idea, a motion moment, a texture).
- **Constraints**: framework, performance budget, accessibility, brand assets.

Bold maximalism and refined minimalism both work. The key is **intentionality**: every choice should follow from the direction. Match implementation effort to the vision — maximalist designs need elaborate execution; minimal designs need precision in spacing, type, and detail.

## 3. Aesthetics (greenfield)

- **Typography**: characterful, context-specific font choices. Pair a distinctive display face with a highly legible text face. Use a deliberate type scale. Avoid defaulting to generic families (Inter, Roboto, Arial, plain system stacks) or repeatedly converging on the same "safe interesting" font across projects.
- **Color**: a cohesive palette defined as design tokens / CSS custom properties. A dominant color with sharp accents beats a timid, evenly spread palette. Avoid clichés such as purple-to-blue gradients on white.
- **Layout and composition**: avoid the predictable hero → three cards → CTA template. Consider asymmetry, overlap, grid-breaking elements, strong alignment, generous whitespace or deliberate density.
- **Depth and detail**: create atmosphere instead of flat defaults where it fits the direction — gradients, noise/grain, patterns, layered transparency, considered shadows, borders, custom cursors.
- **Motion**: prefer a few high-impact moments (an orchestrated entrance with staggered reveals, meaningful state transitions) over scattered effects. Use CSS for simple cases and the animation tooling the project already has for complex ones. Animate `transform` and `opacity`.
- **Variety**: across different projects vary light/dark, typefaces, and aesthetics — no two designs should look like they came from the same template.

## 4. Non-Negotiables (both modes)

**Accessibility**

- Semantic HTML first (`button`, `a`, `nav`, `main`, headings in order, `label` for inputs); ARIA only to fill real gaps.
- Text contrast at least WCAG AA (4.5:1 body, 3:1 large text and UI component boundaries).
- Fully keyboard operable, with a clearly visible focus style. Manage focus in dialogs, menus, and route changes.
- Respect `prefers-reduced-motion`: reduce or remove non-essential animation.
- Meaningful `alt` text (empty for decorative images); never convey information by color alone.
- Touch targets roughly 44×44 px minimum.

**Responsiveness**

- Works from small phones to wide desktops. Design the layout for each range rather than just shrinking it.
- Fluid type and spacing (`clamp()`), container queries where components adapt to their container.
- No horizontal scrolling, no content hidden behind hover-only interactions on touch devices.

**States**

- Design every state, not just the happy path: loading (skeletons over spinners for content), empty, error (with a way to recover), disabled, hover, focus, active, selected, and long/overflowing content.

**Performance**

- Load only the font weights and subsets actually used; use `font-display: swap` or optional and preload critical fonts.
- Correctly sized, modern-format, lazy-loaded images with explicit dimensions to avoid layout shift.
- Keep decorative effects cheap: avoid large blur/backdrop filters and heavy JS animation on scroll.

## 5. Implementation

- Working, production-grade code — no placeholder lorem ipsum where real copy is easy to infer; no dead buttons.
- Centralize design decisions in tokens/variables; no magic numbers scattered across components.
- Reuse existing components before creating new ones; new reusable UI goes into the component layer, not inline.
- Keep presentational components free of data fetching and business logic.

## Before Finishing

- [ ] Correct mode applied: existing system respected, or a clear and consistent new direction.
- [ ] Keyboard navigation, focus visibility, contrast, and reduced motion verified.
- [ ] Layout checked at small, medium, and large widths.
- [ ] Loading, empty, and error states implemented.
- [ ] No generic template look; every visual choice traces back to the chosen direction or the existing system.
