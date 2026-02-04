Reflex Guide for AI Builders

Purpose
- Provide a fast reference for crafting polished marketing sites with Reflex inside this repository.
- Favor modular, declarative components and avoid hacks per the No Gambiarra policy.

Project Layout
- App entry point: `xian_tech/xian_tech.py` (declares constants, components, and `app`).
- Add new reusable sections or widgets inside `xian_tech/` and keep large files split into themed modules when they grow.
- Static media (logos, favicons) lives in `assets/` and is served via `/filename` paths.
- Configuration: `rxconfig.py`; dependency metadata: `pyproject.toml`; guidelines: `AGENTS.md`, `no-gambiarra-policy.txt`.

Component Authoring
- Every view function returns an `rx.Component` and should read like a small factory (`def hero_section() -> rx.Component:`).
- Keep layout helpers (e.g., `section(...)`) for shared patterns; store design tokens (colors, spacing) as upper-case constants.
- Derive repeated content from lists of dicts (`TECHNOLOGY_TRACKS`) and convert with list comprehensions or `rx.foreach`.
- Avoid side effects inside component factories; data shaping and stateful logic belongs in `rx.State` subclasses.

Layout & Spacing Tokens
- Reflex wraps Radix primitives; many props accept enumerated tokens rather than raw CSS.
- Use spacing tokens `"1"`…"9" for `spacing`, `gap`, and similar props; document mappings (e.g., `"6"` ≈ 1.5 rem).
- Valid Flex alignment keywords: `justify={"start","center","end","between"}`; `align_items={"start","center","end","baseline"}`.
- Enable wrapping with `wrap="wrap"`; avoid deprecated `flex_wrap`.
- Fall back to `style={...}` only for properties not exposed by first-class props.

Styling & Typography
- Backgrounds/gradients can be applied on containers with `background=` or `style={"background": "linear-gradient(...)"}`.
- Use `rx.heading`/`rx.text` with `size="1"…"9"` and optional `weight`, `line_height`; clamp long text via `no_of_lines` or CSS line clamp styles.
- Button variants use `rx.button(..., variant="outline", size="3")`; pass `_hover` dicts for interactive states.

State & Data Management
- Define state in subclasses of `rx.State` when interactivity is required; keep them in `xian_tech/state.py` if expanded.
- Use `@rx.var` for derived values; methods invoked from events (on_click, on_change) should mutate state attributes explicitly.
- For static marketing sections, prefer simple constants/lists and pure component factories to avoid unnecessary state.

Build, Run, Test
- Install deps: `poetry install`.
- Dev server: `poetry run reflex run` (binds to :3000, falls back to 3001 if occupied).
- Static export: `poetry run reflex export --frontend-only`.
- Syntax check: `poetry run python -m compileall xian_tech`.
- When adding tests, use `pytest` under `tests/`; mirror module names (`test_hero_section.py`).

Common Pitfalls & Debugging
- Python 3.14 emits a pydantic v1 warning; safe to ignore unless regressions appear.
- If Reflex raises `TypeError: Invalid var passed for prop …`, verify you are using supported tokens (`"between"` not `"space-between"`, `"end"` not `"flex-end"`).
- Ports in use: stop other processes on 3000 or rely on the automatic 3001 fallback.
- Regenerate the frontend (`.web/`) via `reflex run` rather than editing generated files manually.
- Skeletons only render when `loading=True`. When showing loading states, set `loading=True` on `rx.skeleton` and yield after toggling the loading flag in async events so the UI can render the skeletons before heavy work begins.

Design & Collaboration Notes
- Keep sections composable and self-contained; expose top-level layout in `index()` as an ordered stack of sections.
- Capture imagery/illustrations once in assets and reuse with helper components; respect brand colors defined in constants.
- Document significant UI decisions in commit messages and PR descriptions; request clarification when requirements conflict with the No Gambiarra directive.

Theme-Aware Styling with rx.cond()
- The core pattern for conditional/theme-aware styling is `rx.cond(State.theme_mode == "light", light_value, dark_value)`.
- This returns a reactive Var that updates automatically when the condition changes.
- Use this pattern for colors, backgrounds, gradients, and any theme-dependent values.
- Example: `color=rx.cond(State.theme_mode == "light", "#1f2937", "#e6edf3")`

Radix Theme Accent Colors
- `rx.theme(accent_color="green")` and component `color_scheme="grass"` only accept Radix palette names (not hex).
- To match an exact brand hex, keep it in your own tokens (ACCENT/ACCENT_GLOW) and apply via props like `_hover`, `_focus`, and `boxShadow`.

CRITICAL: Reactive Variables in F-Strings Do NOT Work
- You CANNOT use `rx.cond()` or other reactive Vars inside f-strings for CSS values.
- BAD: `f"linear-gradient(to right, {SURFACE_BRIGHT} 60%, transparent)"` - SURFACE_BRIGHT is an rx.cond() Var, this will NOT render correctly.
- GOOD: Build separate strings for each theme, then use rx.cond() to select:
  ```python
  light_gradient = "linear-gradient(to right, #ffffff 60%, transparent)"
  dark_gradient = "linear-gradient(to right, #192330 60%, transparent)"
  background=rx.cond(State.theme_mode == "light", light_gradient, dark_gradient)
  ```

Pseudo-Element Styling (_placeholder, _before, _after)
- Reflex props like `_placeholder={"color": "..."}` often DO NOT WORK reliably.
- The styles may be ignored or overridden by Radix UI defaults.
- SOLUTION: Prefer a component-scoped pseudo selector, or fall back to CSS injection:
  - Component-scoped: `style={"& input::placeholder": {"color": "#6b7280", "opacity": "1"}}`
  - Component-scoped: `style={"& textarea::placeholder": {"color": "#6b7280", "opacity": "1"}}`
  - Global fallback with `rx.el.style()`:
  ```python
  rx.el.style(
      rx.cond(
          State.theme_mode == "light",
          "#my-input::placeholder { color: #4b5563 !important; opacity: 1 !important; }",
          "#my-input::placeholder { color: #9ca3af !important; opacity: 1 !important; }",
      )
  )
  ```
- Give the element an `id` prop so the CSS can target it.
- Use `!important` to override Radix defaults.
- This pattern works for any pseudo-element (::before, ::after, ::placeholder, etc.).

SVG Elements in Reflex
- Use `rx.el.svg` for the container and `rx.el.svg.path`, `rx.el.svg.circle`, etc. for shapes.
- Key props: `view_box`, `preserve_aspect_ratio`, `d` (for paths), `cx`/`cy`/`r` (for circles).
- SVG coordinates in viewBox are unitless; use percentage-based thinking (0-100) for responsive layouts.
- Position SVG absolutely over content with `position="absolute"`, `top="0"`, `left="0"`, `width="100%"`, `height="100%"`.
- Use `pointer_events="none"` so SVG doesn't block clicks on underlying elements.
- Use `z_index` to control layering (lower = behind, higher = in front).
- Path commands: M=moveto, L=lineto, Q=quadratic bezier, C=cubic bezier.
- Example quadratic curve: `d="M 50 40 Q 50 55, 16 70"` (start at 50,40, control point 50,55, end at 16,70).

CSS Gradients vs Background Images
- PREFER CSS gradients over background images for simple effects (color fades, highlights).
- Benefits: No external assets, scales perfectly to any size, easy to theme, faster loading.
- Use background images only for complex visuals that can't be replicated with CSS.
- Gradient syntax: `"linear-gradient(to right, rgba(0, 179, 92, 0.08) 0%, rgba(0, 179, 92, 0.18) 100%)"`
- Multiple backgrounds: layer gradients with images using comma separation.

Keyboard Navigation in Overlays/Modals
- Pattern: Hidden buttons + JavaScript event listeners + state methods.
- Create hidden buttons that trigger state changes:
  ```python
  rx.button(on_click=State.move_up, id="move-up-btn", display="none")
  ```
- Add JavaScript to listen for keys and click the hidden buttons:
  ```python
  rx.script('''
    window.addEventListener("keydown", (e) => {
      if (e.key === "ArrowUp") {
        e.preventDefault();
        document.getElementById("move-up-btn")?.click();
      }
    });
  ''')
  ```
- For scrolling to active elements after state changes, use setTimeout to wait for DOM update:
  ```javascript
  setTimeout(() => {
    document.getElementById("active-item")?.scrollIntoView({ block: "nearest", behavior: "smooth" });
  }, 50);
  ```
- Give the currently active item a known ID using `id=rx.cond(is_active, "active-item", "")`.

Grid Layouts
- Use `rx.grid` with `columns` prop for responsive grids.
- Columns accept responsive dict: `columns={"initial": "1fr", "md": "repeat(3, 1fr)"}`
- Alternative: use `style` with `gridTemplateColumns` for more control.
- Media queries in style dict: `style={MD_MEDIA: {"gridTemplateColumns": "repeat(3, 1fr)"}}`
- Define media query constant: `MD_MEDIA = "@media (min-width: 1024px)"`
- `align="stretch"` is the correct prop for equal-height grid items (not `align_items`).
- Use `rows=` and `flow=` if you need to explicitly lock row count/order.

Responsive Props with rx.breakpoints()
- Use `rx.breakpoints(...)` to express responsive values in a single prop: `display=rx.breakpoints(initial="none", lg="flex")`.
- Valid keys are `initial`, `xs`, `sm`, `md`, `lg`, `xl` (no `"base"` key).
- Default breakpoints (min-width): xs=30em, sm=48em, md=62em, lg=80em, xl=96em.
- Prefer `rx.breakpoints(...)` over ad-hoc media query style dicts when a prop supports responsiveness.

Z-Index and Layering
- Use `z_index="0"`, `z_index="1"`, etc. as strings.
- Elements need `position="relative"` or `position="absolute"` for z-index to take effect.
- Common pattern for overlays: background layer z_index="0", content z_index="1".

Data-Driven Navigation with Highlights
- Store navigation data in dicts with optional fields like `highlighted: True`.
- Check for the field in the component: `highlighted = child.get("highlighted", False)`
- Apply conditional styling based on the flag.
- This keeps data and presentation separate - adding/removing highlights is just a data change.

Transparent vs Solid Colors
- Theme colors like SURFACE_BRIGHT often use rgba with alpha < 1 (e.g., `rgba(255, 255, 255, 0.95)`).
- This can cause visual artifacts (1px lines, bleed-through) when layering.
- For solid backgrounds, define explicit hex colors: `#ffffff` instead of `rgba(255, 255, 255, 0.95)`.
- Use transparent rgba colors only where transparency is intentional.

Hover Styles
- Use `_hover={...}` dict for hover states.
- Common props: `background`, `borderColor`, `boxShadow`, `color`, `transform`.
- Transitions: add `transition="all 0.2s ease"` to the base element for smooth hover effects.
- Remove unwanted defaults: `"textDecoration": "none"` on links.

Input Components
- `rx.input` wraps Radix UI input.
- Props: `value`, `on_change`, `placeholder`, `auto_focus`, `width`, `padding`, `border`, `border_radius`, `background`, `color`.
- `_focus={...}` for focus state styling.
- For outer rings on Radix text fields, use `_focus_within={...}` along with `_focus`.
- Placeholder styling should target the inner input/textarea element with `style={"& input::placeholder": {...}}` or `style={"& textarea::placeholder": {...}}`.

Forms and on_submit
- `rx.form` passes a form data dict to handlers; type annotate as `dict[str, Any]` to match the Reflex signature.

Accordion (Radix) Behaviors
- `rx.accordion.item` only allows `AccordionHeader`, `AccordionTrigger`, and `AccordionContent` as children.
- If you pass `header=` and `content=` props, Reflex will auto-insert a chevron icon. Do not also add your own icon, or you will get duplicates.
- To fully control the header layout (and icon placement), pass `AccordionHeader`/`AccordionTrigger`/`AccordionContent` as children instead of using `header=`/`content=`.
- `variant="ghost"` on the accordion root resets first/last item corner radius to 0; use `variant="outline"` or another variant if you need rounded corners.
- Use `type="single"` on a single shared `accordion.root` to ensure only one item is open at a time.
- There is no `spacing` prop on `accordion.root`; use a flex column style with `gap` when you need space between items.
