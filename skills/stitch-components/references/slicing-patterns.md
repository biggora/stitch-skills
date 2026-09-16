# Slicing Patterns — Reference

Read `SKILL.md` first — this file assumes Step 1 (fetching the screen's HTML) and Step 2 (detecting the host project's conventions) are already done. Everything here informs Step 3: turning a downloaded Stitch screen into a slicing plan worth proposing.

---

## Worked example — a real measured screen

This is a screen actually observed from `get_screen` / `htmlCode.downloadUrl`: a Russian-language desktop page, 2616×5548, containing `header`×1, `nav`×2, `main`×1, `section`×1, `article`×9, `footer`×1, 263 `class="` attributes, and a `<script id="tailwind-config">` block resolving a full Material-3-style color palette. No CSS custom properties anywhere — every color reference is a Tailwind utility class resolved through that inline config.

### Input markup shape (abbreviated)

```html
<body class="bg-background-light font-body-content">
  <header class="sticky top-0 z-10 ...">
    <nav class="flex items-center justify-between ...">
      <div class="text-primary text-2xl font-bold">LensInspect</div>
      <input id="header-search-input" data-alt="site search" ... />
    </nav>
  </header>

  <main id="content">
    <section class="grid grid-cols-3 gap-6 ...">
      <article class="bg-surface-container-low rounded-xl ...">
        <img data-alt="product thumbnail" src="..." class="rounded-t-xl" />
        <h3 id="hero-title" class="text-on-surface font-headline">...</h3>
        <p class="text-on-surface-variant">...</p>
        <a href="..." data-path="/product/1" class="text-primary">Подробнее</a>
      </article>
      <!-- ...8 more <article> elements, same class signature, different content -->
    </section>
  </main>

  <nav class="fixed bottom-0 ...">
    <!-- secondary/mobile nav landmark -->
  </nav>

  <footer class="bg-surface-dim ..." data-ad-slot="footer-banner">
    ...
  </footer>
</body>
```

### Proposed plan table (what Step 3 would actually present)

| Component | Source anchor | Instances | Proposed props | Target file |
|---|---|---|---|---|
| `SiteHeader` | `<header class="sticky top-0 ...">` | 1 | `logo`, `onSearch` (as inert prop — see honesty rules) | `components/SiteHeader.tsx` |
| `PrimaryNav` | `<nav>` inside `<header>` | 1 | `items: NavItem[]` | `components/PrimaryNav.tsx` |
| `ArticleCard` | `<article class="bg-surface-container-low rounded-xl ...">`, repeated in `<section>` | 9 | `imageUrl`, `imageAlt`, `title`, `description`, `href` | `components/ArticleCard.tsx` |
| `ArticleGrid` | `<section class="grid grid-cols-3 gap-6 ...">` wrapping the 9 `<article>`s | 1 | `items: ArticleCardProps[]` | `components/ArticleGrid.tsx` |
| `MobileNav` | second `<nav class="fixed bottom-0 ...">` | 1 | `items: NavItem[]` | `components/MobileNav.tsx` |
| `SiteFooter` | `<footer data-ad-slot="footer-banner">` | 1 | `adSlot` (passed through, not implemented) | `components/SiteFooter.tsx` |

Note the two `<nav>` elements are **not** the same component with a variant prop — they're structurally distinct (top sticky nav with search vs. bottom fixed nav), which is exactly the "two components, not one with branches" case covered in Granularity guidance below. `ArticleGrid` is kept separate from `ArticleCard` rather than folded into it, because the grid's own responsibility (layout, iteration) is distinct from a single card's responsibility (rendering one item) — collapsing them would make `ArticleCard` do two jobs.

### Resulting component (after approval, Step 4)

```tsx
// components/ArticleCard.tsx
interface ArticleCardProps {
  imageUrl: string;
  imageAlt: string;
  title: string;
  description: string;
  href: string;
}

export function ArticleCard({ imageUrl, imageAlt, title, description, href }: ArticleCardProps) {
  return (
    <article className="bg-surface-container-low rounded-xl">
      <img src={imageUrl} alt={imageAlt} className="rounded-t-xl" />
      <h3 className="text-on-surface font-headline">{title}</h3>
      <p className="text-on-surface-variant">{description}</p>
      <a href={href} className="text-primary">Подробнее</a>
    </article>
  );
}
```

```tsx
// components/ArticleGrid.tsx
import { ArticleCard } from "./ArticleCard";

interface ArticleGridProps {
  items: Array<React.ComponentProps<typeof ArticleCard>>;
}

export function ArticleGrid({ items }: ArticleGridProps) {
  return (
    <section className="grid grid-cols-3 gap-6">
      {items.map((item) => (
        <ArticleCard key={item.href} {...item} />
      ))}
    </section>
  );
}
```

`data-alt` and `data-path` from the original markup are Stitch's own placeholder attributes for image alt text and link destinations — they're folded into real `alt` and `href` props here rather than carried through verbatim, since the target is React, which has native attributes for both.

---

## Pattern catalogue

For each pattern: the seam to cut on, the props worth extracting, and the trap to avoid.

### 1. Navigation bar
- **Seam:** the `<header>` or top-level `<nav>` landmark, usually `sticky top-0` or similar.
- **Props:** `logo`/`brand`, `items: NavItem[]` (label + href + active state), optional `actions` (search input, CTA button) as a separate slot rather than baked into the nav's own markup.
- **Trap:** treating the search input's live filtering as implemented behavior. The markup shows an `<input>`; it does not show what happens on keystroke. Render it as a controlled/uncontrolled input with an inert `onSearch` prop, not a working search.

### 2. Hero
- **Seam:** the first `<section>` after the nav, usually containing a large heading, a short paragraph, one or two CTA buttons, and often a background image or illustration.
- **Props:** `heading`, `subheading`, `primaryCta: {label, href}`, `secondaryCta?`, `imageUrl?`.
- **Trap:** splitting the heading and CTA into separate components when they only ever appear together in this one place. A hero is usually one component with several props, not three components with one prop each — see Granularity guidance.

### 3. Card grid
- **Seam:** a wrapping `<section>`/`<div class="grid ...">` around N structurally identical children (the `article`×9 case above is the canonical instance).
- **Props:** the grid itself takes `items: T[]`; each card takes whatever fields distinguish one item from the next (title, image, price, tag, href).
- **Trap:** building the card and the grid as one component. Keep them separate — the grid may later need to render the same card in a carousel, a list, or a different column count, and a fused component can't do that without new branches.

### 4. Data table
- **Seam:** `<table>` with a `<thead>` row and repeated `<tr>` rows in `<tbody>`, or a Stitch screen's div-based table approximation (a header row of column labels followed by repeated row-shaped divs).
- **Props:** `columns: {key, label}[]`, `rows: Record<string, ReactNode>[]`. If the project has a table primitive (shadcn's `Table`, TanStack Table), prefer it over a hand-rolled one — check Step 1 of `SKILL.md` before assuming a custom table is warranted.
- **Trap:** hardcoding column count and labels into the component instead of deriving the header row from `columns`. A table sliced from one screen with 5 columns should still work if the caller passes 4 or 6.

### 5. Form
- **Seam:** a `<form>` or a div-based approximation containing labeled inputs and a submit control.
- **Props:** per-field props or a single `fields` config array, plus `onSubmit` as an inert prop.
- **Trap:** inventing validation rules, error states, or a submit handler the markup doesn't show. A Stitch form is a static rendering of what the fields look like — required-field asterisks and placeholder text are fair game to preserve; actual validation logic is not something the screen specifies. Flag this explicitly per the honesty rules in `SKILL.md`.

### 6. Sidebar / filter panel
- **Seam:** an `<aside>` or a fixed-width `<div>` sibling to `<main>`, usually containing grouped checkboxes, a price range, or category links.
- **Props:** `groups: {label, options}[]` for filter groups, or `items: NavItem[]` if it's closer to secondary navigation.
- **Trap:** treating every checkbox as its own component. A single `FilterGroup` component rendering a list of checkboxes from props almost always beats one component per filter option.

### 7. Modal / overlay
- **Seam:** markup that's visually a dialog (centered box, backdrop, close control) but rendered inline in the static HTML — Stitch has no runtime, so a modal appears as ordinary flow markup, not as an actual overlay.
- **Props:** `title`, `content`/`children`, `onClose` (inert), `actions`.
- **Trap:** shipping the extracted "modal" as a static block that renders inline instead of wiring it to the project's actual dialog primitive (shadcn's `Dialog`/`AlertDialog`, a headless-UI equivalent, or the project's own). Check what the project already has before writing new overlay/portal logic.

### 8. Footer
- **Seam:** the trailing `<footer>` landmark.
- **Props:** `columns: {heading, links}[]` for a multi-column footer, `copyright`, `socialLinks?`.
- **Trap:** copying an ad slot or third-party embed placeholder (`data-ad-slot`, an iframe stub) as if it were content to hardcode. Pass it through as a prop or a clearly marked placeholder instead.

### 9. Stat / metric row
- **Seam:** a row of 3–5 structurally identical blocks, each a number, a label, and sometimes a trend indicator or icon.
- **Props:** the row takes `stats: {value, label, trend?}[]`; consider whether a single stat block is even worth its own component versus inlining it in the row if it never appears standalone (see Granularity guidance).
- **Trap:** hardcoding the number of stats (`stat1`, `stat2`, `stat3` as separate props) instead of an array — this is the single most common way a "reusable" component turns out to only work for the exact count in the source screen.

### 10. Empty state
- **Seam:** a centered block (icon or illustration, a heading, a short explanatory line, sometimes a CTA) that a Stitch screen renders as if data were absent, even though the surrounding list/table/grid is otherwise the "real" content.
- **Props:** `icon`/`illustration`, `heading`, `description`, `action?`.
- **Trap:** missing it entirely because it looks like "just another card" in the markup rather than a conditional state. If a screen shows both populated and empty renderings of the same list, that's a signal to extract the empty state as its own component the list conditionally renders, not to bake it into the list component's default markup.

### 11. Pagination
- **Seam:** a row of numbered controls or prev/next buttons, usually near a table or card grid.
- **Props:** `currentPage`, `totalPages`, `onPageChange` (inert prop, not wired to real state).
- **Trap:** hardcoding the page numbers visible in the one screenshot (e.g. rendering literal buttons "1", "2", "3") instead of deriving them from `totalPages`.

### 12. Tab bar
- **Seam:** a row of label buttons with one visually marked active, often above a content panel that only shows the active tab's content in the static markup (the inactive tabs' content isn't present in the HTML at all — Stitch renders one state).
- **Props:** `tabs: {label, id}[]`, `activeTab`, `onTabChange` (inert).
- **Trap:** assuming the content shown under the active tab is the *only* content that panel ever holds, and hardcoding it into the tab bar component itself instead of treating tab content as separate, swappable children the page composes.

---

## Granularity guidance

This is the hardest judgment call in this skill: when is a repeated block **one component with a variant prop**, and when is it **two components**? When do you stop subdividing?

**Rule of thumb:** two blocks are the *same* component with a variant prop when their DOM shape (tag types, nesting, count of children) is identical and only class names, text, or a boolean-like state differ. They are *separate* components when the internal structure itself differs — different tags, different number of children, different semantic role — even if they share a family resemblance.

Concretely:
- A "featured" article card with an extra badge `<span>` that the plain card doesn't have is still the same DOM shape *if* the badge is conditionally rendered from a `featured` prop (`{featured && <span>...</span>}`). One component.
- A compact list row and a full card, both showing the same underlying item (same fields, wildly different layout — one is a `<li>` with inline text, the other is a `<div>` with an image, a heading, and a footer), are two components, even though they represent "the same kind of thing." Forcing one component to render both via a `layout="list" | "card"` prop usually produces more branching inside the component than the two components would cost separately.
- The top sticky nav and the bottom fixed nav in the worked example above are separate components (`PrimaryNav`, `MobileNav`) precisely because their structure differs beyond a class swap — one contains a search input, the other doesn't; their positioning strategy differs; they serve different roles.

**When to stop subdividing:** stop at the point where a piece of markup has no independent identity — it never repeats, it's never referenced from more than one place in the plan, and it exists purely to nest other elements (a wrapper `<div>` that only exists for a Tailwind flex/grid class). Extracting that into its own component adds an indirection with no reuse payoff. The test: would you ever import this component from somewhere other than its one current parent? If not, leave it as JSX inline in the parent.

**Counterexample worth internalizing:** a stat row with 3 stat blocks does *not* need a separate `StatBlock` component if the row is the only place stats ever render and the blocks have no independent props beyond `value`/`label`. Inlining the block's JSX inside a `.map()` in `StatRow` is simpler and just as reusable at the actual call site (`<StatRow stats={...} />`), because nothing external ever needs a bare `StatBlock` on its own. Contrast this with `ArticleCard` in the worked example, which is worth extracting on its own even though it's currently only used inside `ArticleGrid`, because a card is a recognizable, independently meaningful unit likely to be reused elsewhere in the same project (a related-articles section, a search results page) — a stat block inside one specific stat row is much less likely to be.

---

## Mapping the inline `tailwind.config` palette

The downloaded HTML embeds its full resolved color palette in a `<script id="tailwind-config">` block:

```html
<script id="tailwind-config">
  tailwind.config = {
    "darkMode": "class",
    "theme": {
      "extend": {
        "colors": {
          "primary": "#4f46e5",
          "on-primary": "#ffffff",
          "primary-fixed-variant": "#3730a3",
          "on-primary-fixed-variant": "#e0e7ff",
          "surface": "#ffffff",
          "surface-dim": "#f4f4f5",
          "surface-container-low": "#fafafa",
          "on-surface": "#18181b",
          "on-surface-variant": "#52525b",
          "outline-variant": "#e4e4e7",
          "error-container": "#fee2e2",
          "on-error-container": "#7f1d1d",
          "tertiary-fixed-dim": "#a78bfa"
          /* ...dozens more Material-3 slot names */
        }
      }
    }
  };
</script>
```

**How to read it:** after downloading the HTML in Step 1, extract this block's JSON (it's assignment syntax, not pure JSON — strip the `tailwind.config = ` prefix and trailing `;` before parsing) rather than guessing color values from the screenshot. `theme.extend.colors` is a flat map of Material-3-style slot names to hex values.

**How to reconcile it with a project that already has its own token names:**
1. Don't bulk-import the whole palette. These slot names are numerous — dozens of `on-*`, `*-container`, `*-fixed`, `*-fixed-dim`, `*-fixed-variant` combinations exist across primary/secondary/tertiary/error/neutral families — and any single screen only ever references a handful of them.
2. Take only the slot names that actually appear as class names (`bg-primary`, `text-on-surface-variant`, `bg-surface-container-low`) in the markup you're extracting. Grep the downloaded HTML for `class="` matches against the palette's keys to get the exact working set.
3. Map each one to the project's existing token vocabulary rather than importing the Material-3 name verbatim:
   - If the project already has a `primary`/`primary-foreground` pair (shadcn-style), map `primary` → `primary` and `on-primary` → `primary-foreground`. The `on-*` prefix is Material 3's "content color for this surface" convention, which is functionally the same idea as a `*-foreground` suffix in shadcn's convention — translate the naming pattern, not just the individual keys.
   - If the project has no existing token system at all, it's reasonable to add the referenced slots as new CSS variables or Tailwind theme colors, using either the Material-3 names as-is or renaming them to match the project's own casing/naming convention (check Step 2's findings) — but still only the ones actually used, not the full palette.
   - `*-fixed` / `*-fixed-dim` / `*-fixed-variant` slots represent Material 3's "fixed" color roles (colors that stay constant across light/dark rather than inverting) — if the project's dark-mode strategy doesn't distinguish fixed vs. non-fixed roles, it's fine to fold a `*-fixed` slot into the same token as its non-fixed counterpart and note the simplification to the user, rather than inventing a fixed/non-fixed distinction the project has no mechanism for.
4. Hex values, not the Material-3 slot semantics, are the ground truth for what a color actually looks like — if a slot name and its resolved hex seem to disagree with how it's actually used in the markup (rare, but a Stitch screen can theoretically apply a slot inconsistently), trust the hex value and flag the discrepancy rather than silently picking one.

---

## What NOT to componentize

Not every recurring shape in the downloaded HTML deserves a component. Leave these as inline JSX in whatever component they live inside:

- **One-off wrappers** — a `<div>` that exists only to apply a single Tailwind layout class (`flex`, `grid grid-cols-2`, `mx-auto max-w-7xl`) around content that has no independent identity or reuse elsewhere. Extracting it produces an import with no payoff.
- **Purely presentational nesting** — decorative elements (a gradient overlay `<div>`, a divider `<hr>`, an icon wrapper that exists only to size an SVG) that never carry props and never vary between the places they appear. These are implementation detail of whatever component contains them, not components in their own right.
- **Blocks that differ enough that a shared component would need more branches than it saves** — see the list-row-vs-card example in Granularity guidance. If writing the "shared" version requires a `layout` prop with two entirely different render trees behind an `if`, you have not simplified anything; you've hidden two components inside one file with a worse API than either would have had alone. Two components in that case is not over-engineering — it's the simpler outcome.
