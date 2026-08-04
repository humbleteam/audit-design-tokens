---
name: audit-design-tokens
description: Scans a codebase for design token drift - raw hex colors, off-scale spacing, near-duplicate colors, font sprawl, hardcoded z-index. Use when asked to "audit my design tokens", "find hardcoded colors in this codebase", "check for design token drift", or "clean up our CSS variables". Returns a severity-ranked report with file:line refs and a plan. Do not use to generate tokens from a screenshot or URL - use extract-design-tokens instead.
---

# Audit design tokens

Find every place a codebase drifted away from its own design tokens, rank the drift by
how much damage it does, and hand back a plan to fix it one small step at a time.

## When this skill applies

The user has an existing codebase with some notion of design tokens - a `:root` block of
CSS custom properties, a `tailwind.config.js` theme, a `theme.ts` / `tokens.json` file, or
styled-components theme object - and wants to know where the code stopped using them.

If there are no token definitions anywhere in the codebase, stop the audit and say so (see
Edge case: no tokens found below). Auditing drift against tokens that do not exist produces
nothing useful.

## Step 1: find the token source of truth

Search the target path (default: repo root, or the path the user names) for token
definitions, in this order:

1. CSS custom properties: `--[a-z-]+:\s*(#|rgb|hsl|oklch)` inside `:root`, `:host`, or a
   `[data-theme]` selector.
2. Tailwind config: `theme.colors`, `theme.spacing`, `theme.extend.colors`,
   `theme.extend.spacing` in `tailwind.config.js` / `.ts` / `.mjs`.
3. A dedicated tokens file: `tokens.json`, `theme.ts`, `theme.js`, `design-tokens.*`.
4. styled-components / Emotion theme objects passed to `ThemeProvider`.

Record every token found as `name -> value` per category (color, spacing, radius, shadow,
font-size, z-index). This list is the baseline every other file gets checked against. If
more than one source exists (e.g. CSS variables AND a Tailwind config), treat both as valid
tokens - report a value as drift only if it matches neither.

## Step 2: scan for drift

Walk `.css`, `.scss`, `.less`, `.tsx`, `.jsx`, `.ts`, `.js`, `.vue`, `.svelte`, and inline
`style=` attributes in `.html`. Skip `node_modules`, `dist`, `build`, `.next`, `vendor`, and
lockfiles. For each category below, collect every match with its file and line number.

- **Raw colors outside token definitions.** Any `#rrggbb`, `#rgb`, `rgb(...)`, `rgba(...)`,
  `hsl(...)`, or `hsla(...)` literal that is not itself the right-hand side of a token
  definition found in Step 1. This is the highest-value category - it is what makes theming
  and dark mode break.
- **Off-scale spacing.** Any `margin`, `padding`, `gap`, `top/right/bottom/left`, or `width/
  height` value in `px` or `rem` that is not a multiple of the codebase's spacing base (infer
  the base from the token list - most commonly 4px or 8px) and is not itself a spacing token
  reference. `padding: 13px` on a 4px scale is drift; `padding: 16px` is not.
- **Near-duplicate colors.** Group all raw and token colors by hue (convert to HSL). Flag
  pairs within the same category (background, text, border) whose lightness differs by less
  than 5% or whose values differ by a handful of hex units - these are merge candidates, not
  necessarily bugs.
- **Font-size sprawl.** Collect every distinct `font-size` value in the codebase, raw or
  token. More than 8-10 distinct sizes for a single product is sprawl - list every value with
  a count of how many places use it, sorted descending.
- **Radius and shadow variants beyond 3 levels.** Most products need at most `sm` / `md` /
  `lg` radius and 2-3 shadow depths. Count distinct `border-radius` and `box-shadow` values
  actually in use; anything past 3-4 is a sign of accretion, not a considered scale.
- **Hardcoded z-index stacks.** Any numeric `z-index` value not pulled from a token or a
  shared constant. Collect all of them sorted by value - an ungoverned z-index list is a
  common source of stacking bugs even when nothing else in the design system has drifted.

## Step 3: write the report

Output in this exact structure. Use real file paths and line numbers from the scan - never
invent a location.

```markdown
# Design token audit - <path scanned>

## Summary
- Token sources found: <list, e.g. "CSS custom properties (32 tokens), tailwind.config.js theme (8 colors)">
- Raw colors outside tokens: <count>
- Off-scale spacing values: <count>
- Near-duplicate color pairs: <count>
- Distinct font sizes in use: <count> (target: 8-10 or fewer)
- Radius variants: <count> · Shadow variants: <count>
- Hardcoded z-index values: <count>

## P0 - breaks theming
Hardcoded colors on interactive states (hover, focus, active, disabled) or on anything that
changes under a `[data-theme]` / dark-mode selector elsewhere in the codebase.

| File:line | Value | Nearest token | Suggested fix |
|---|---|---|---|
| src/Button.tsx:42 | #2563EB | --color-primary (#2563EB) | replace literal with var(--color-primary) |

## P1 - scale drift
Off-scale spacing, font-size sprawl, radius/shadow variants beyond the established scale.

| File:line | Value | Nearest token | Suggested fix |
|---|---|---|---|

## P2 - consolidation candidates
Near-duplicate colors, one-off z-index values, anything safe to leave but worth merging.

| File:line | Value | Nearest token | Suggested fix |
|---|---|---|---|

## Proposed renames (do not apply silently)
Only if a token's current name no longer matches its use (e.g. `--blue-500` used as an error
color). List old name -> proposed name -> every call site. Do not rename in the report itself -
this is a proposal for the user to approve.

## Consolidation plan
Ordered, smallest-diff-first. Each step is a single PR-sized change that ships independently
without waiting for the others.

1. <one file or one component, e.g. "Replace 6 raw #2563EB literals in src/Button.tsx and src/Link.tsx with var(--color-primary)">
2. <next smallest step>
3. ...
```

Counts in the summary must equal the row counts in the tables below them. If a category has
zero findings, keep its row in the summary showing 0 and omit its table.

## Severity rules

- **P0**: a hardcoded color sits on an interactive state (hover/focus/active/disabled) or
  inside a component that also renders under a dark-mode or alternate-theme selector
  elsewhere in the codebase. This is the category that visibly breaks for users.
- **P1**: off-scale spacing, font-size sprawl past 8-10 sizes, radius/shadow variants past
  3-4 levels. Does not break anything today but actively degrades consistency.
- **P2**: near-duplicate colors, isolated z-index literals, anything a reasonable team could
  ship as-is and clean up opportunistically.

Never invent a P0 finding to make the report look more urgent. If nothing qualifies as P0,
say so plainly - "no P0 findings" is a valid and common result.

## Rules

- Never rename a token in the report as if it already happened. Renames are always a
  separate proposed section the user reviews before applying.
- Never invent a file path, line number, or token value. Every row in every table must trace
  to something you actually found in the scan. If you could not scan a file (binary, too
  large, generated), say which files were skipped and why.
- Do not touch accessibility, contrast ratios, or component states - that is the design-qa
  and accessibility-audit skills. Stay inside token consistency.
- Do not apply any fix automatically unless the user explicitly asks you to apply the
  consolidation plan after reviewing it.

## Edge cases

- **No token definitions found at all.** Stop the audit. Tell the user there is nothing to
  audit drift against, and point them at the `extract-design-tokens` skill to pull a token
  set from a URL, screenshot, or the codebase's own most-used values first.
- **Monorepo.** If the target path contains multiple `package.json` files with independent
  `src/` trees, ask which package to scan, or scan only the path the user named. Do not
  silently scan the whole monorepo - drift counts across unrelated apps are not comparable.
- **Tailwind.** Compare every class-based usage against `tailwind.config.js`'s `theme` and
  `theme.extend`. Arbitrary-value classes (`bg-[#2563eb]`, `p-[13px]`, `text-[15px]`) are the
  drift signal in a Tailwind codebase - treat every arbitrary-value bracket as a raw-value
  hit in the matching category (color, spacing, font-size). A safelist entry is not drift.
- **CSS-in-JS with computed values.** If a color or spacing value is computed at runtime
  (e.g. `darken(theme.primary, 0.1)`), do not flag it as raw - it already derives from a
  token. Only flag literals that do not reference any token.
