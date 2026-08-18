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

1. CSS custom properties: `--[A-Za-z0-9_-]+\s*:` inside `:root`, `:host`, or a
   `[data-theme]` selector - match the declaration, then classify by the value. Matching on
   the value instead finds only part of the set: a pattern anchored to `#|rgb|hsl|oklch`
   returns colors only, and one whose name class excludes digits also drops
   `--blue-500` and `--gray-100`, which is how most palettes are named. Both misses are
   worse than no match at all, because an undiscovered token is not treated as missing -
   it is treated as absent, and every correct use of its value elsewhere in the codebase
   is then reported as raw drift against a token that exists.
2. Tailwind config: `theme.colors`, `theme.spacing`, `theme.extend.colors`,
   `theme.extend.spacing` in `tailwind.config.js` / `.ts` / `.mjs`.
3. A dedicated tokens file: `tokens.json`, `theme.ts`, `theme.js`, `design-tokens.*`.
4. styled-components / Emotion theme objects passed to `ThemeProvider`.

Record every token found as `name -> value` per category. The value's shape decides what a
token can be, and the name only splits the categories the shape cannot tell apart, since
`--space-3: 12px`, `--radius-md: 8px` and `--font-size-lg: 1.25rem` are all one length:

| Value shape | Category |
|---|---|
| `#hex`, `rgb()`, `rgba()`, `hsl()`, `hsla()`, `oklch()`, a named CSS color | color |
| an offset/blur/spread list ending in a color | shadow |
| a bare integer | z-index |
| a single length in `px`, `rem`, `em` | spacing, radius, or font-size - split on the name (`space`/`gap`/`inset`, `radius`/`corner`, `font`/`text`/`size`) |

Shape wins over name where they disagree: `--brand-blue: 8px` is a length, so it is not a
color whatever it is called. A length whose name matches none of the three groups is
recorded as spacing, the scale it most often belongs to, and named in the report as an
assumption.

This list is the baseline every other file gets checked against. If more than one source
exists (e.g. CSS variables AND a Tailwind config), treat both as valid tokens - report a
value as drift only if it matches neither.

**Then write down which categories have a baseline and which do not.** A codebase with a
`:root` block of brand colors and no spacing scale is normal, not broken, and it is the
common starting shape. The stop rule above fires only when the token set is completely
empty, so a partial set reaches Step 2 and every category it does not cover has nothing to
be checked against. Carry that list into Steps 2 and 3 - it decides what can be audited at
all.

## Step 2: scan for drift

Walk `.css`, `.scss`, `.less`, `.tsx`, `.jsx`, `.ts`, `.js`, `.vue`, `.svelte`, and inline
`style=` attributes in `.html`. Skip `node_modules`, `dist`, `build`, `.next`, `vendor`, and
lockfiles. For each category below, collect every match with its file and line number.

- **Raw colors outside token definitions.** Any `#rrggbb`, `#rgb`, `rgb(...)`, `rgba(...)`,
  `hsl(...)`, or `hsla(...)` literal that is not itself the right-hand side of a token
  definition found in Step 1. This is the highest-value category - it is what makes theming
  and dark mode break.
- **Off-scale spacing.** Any `margin`, `padding`, `gap`, `top/right/bottom/left`, or `width/
  height` value in `px` or `rem` that is not a multiple of the codebase's spacing base and is
  not itself a spacing token reference. `padding: 13px` on a 4px scale is drift;
  `padding: 16px` is not. Infer the base from the spacing tokens Step 1 recorded - the
  greatest common divisor of the scale, usually 4px or 8px. **If Step 1 found no spacing
  tokens, there is no base and this category is not audited.** Do not fall back to 4px or
  8px because they are common: a codebase that never had a spacing scale would have every
  odd value reported as drift against a standard it never adopted, which is the same
  invented baseline the no-tokens stop rule exists to prevent, applied one category at a
  time instead of all at once.
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

**A category with no baseline is not scanned as drift.** Drift means a distance from a
known-good value, so where Step 1 found no token of that category there is nothing to
measure against and every literal would be reported as a violation of a rule the codebase
never set. This applies to raw colors and off-scale spacing, which need a token to drift
from. It does not apply to font-size sprawl, radius and shadow variants, or near-duplicate
colors: those compare the codebase against itself, so they run on any codebase. Hardcoded
z-index runs too - with no z-index token every literal is ungoverned, which is the finding
rather than a false positive, and it stays P2.

## Step 3: write the report

Output in this exact structure. Use real file paths and line numbers from the scan - never
invent a location.

```markdown
# Design token audit - <path scanned>

## Summary
- Token sources found: <list, e.g. "CSS custom properties (32 tokens), tailwind.config.js theme (8 colors)">
- Categories with a baseline: <list> · no baseline: <list, or "none">
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

Reconcile the summary against the tables before shipping the report. The summary carries two
kinds of number, and only one of them is a row count:

- **Drift counts** - raw colors, off-scale spacing, near-duplicate pairs, hardcoded z-index.
  Each of these findings is a row in exactly one of the P0/P1/P2 tables, so the summary count
  must equal the number of rows carrying that category across all three tables. A
  near-duplicate pair is one finding on one row naming both locations, not two rows.
- **Inventory counts** - distinct font sizes in use, radius variants, shadow variants. These
  count distinct values in the codebase, most of which are not drift, so they never match a
  row count: a clean codebase with six tokenized font sizes reports 6 and opens no rows at
  all. Reconcile them against the values instead - state the count, list the distinct values,
  and open a P1 row only for what is past the threshold.

If a category has zero findings, keep its row in the summary showing 0 and omit its table.

A count of 0 and a category that could not be audited are different results and must not
share a number. `0` says the scan ran and the codebase is clean; a category with no baseline
was never scanned, and reporting it as 0 claims a clean bill of health for the one part of
the system nobody has defined yet. Write `no baseline - no <category> tokens found` in place
of the count, and say what would unlock it: define the scale first, or run
`extract-design-tokens` against the codebase's own most-used values to propose one. The same
goes for the `Nearest token` column - where no token of that category exists there is no
nearest one, so the cell says `none defined` and the suggested fix is to define the token,
never a token name invented to fill the column.

## Severity rules

- **P0**: a hardcoded color sits on an interactive state (hover/focus/active/disabled) or
  inside a component that also renders under a dark-mode or alternate-theme selector
  elsewhere in the codebase. This is the category that visibly breaks for users.
- **P1**: off-scale spacing, font-size sprawl past 8-10 sizes, radius/shadow variants past
  3-4 levels, and any raw color that meets neither P0 condition (see the default below).
  Does not break anything today but actively degrades consistency.
- **P2**: near-duplicate colors, isolated z-index literals, anything a reasonable team could
  ship as-is and clean up opportunistically.

**Every raw color gets a bucket.** A literal that sits on no interactive state and inside no
component rendered under an alternate theme - a background on a static marketing banner in a
codebase with no dark mode - still has to land somewhere. It defaults to **P1**: it breaks
nothing for a user today and degrades consistency, which is what P1 means. Downgrade it to P2
only when the literal is genuinely one-off: a single value in a component nothing else shares,
with no near-duplicate anywhere in the scan. Without this default, the category Step 2 calls
the highest-value one has no home in an unthemed codebase, since P0 requires a state or a
theme, P1 listed only spacing and type, and P2 listed only near-duplicate pairs and z-index -
leaving the finding to be dropped from the report or inflated to P0 against the rule below.

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
- **A partial token set.** Colors defined, nothing else - the commonest real shape, since a
  `:root` of brand colors is where most teams start. The audit runs, and it runs only on the
  categories that have a baseline plus the ones that compare the codebase against itself.
  Say which categories were audited and which were not, in the summary and in one line of
  the report, so a short report reads as limited coverage rather than a clean codebase. Do
  not stop the audit: the color half is real work, and the missing scales are worth naming
  as the gap they are.
- **Monorepo.** If the target path contains multiple `package.json` files with independent
  `src/` trees, ask which package to scan, or scan only the path the user named. Do not
  silently scan the whole monorepo - drift counts across unrelated apps are not comparable.
- **Tailwind.** Compare every class-based usage against `tailwind.config.js`'s `theme` and
  `theme.extend`. Arbitrary-value classes (`bg-[#2563eb]`, `p-[13px]`, `text-[15px]`) are the
  drift signal in a Tailwind codebase - treat every arbitrary-value bracket as a raw-value
  hit in the matching category (color, spacing, font-size). A safelist entry is not drift.
- **Literals that are raw on purpose.** Some values are not drift, and flagging them buries
  the ones that are. Exclude these by default: brand-mark colors inside an inline SVG logo
  (fixed by brand guidelines, not themeable), third-party or vendor stylesheets the team
  does not own and cannot edit, and email templates - most email clients do not resolve CSS
  custom properties, so a literal there is the working choice rather than a mistake. Do not
  drop them silently: list them under an "Excluded from the count" note after the
  consolidation plan, one line each with the reason, so every exclusion stays reviewable.
  If the user says one of them should be tokenized after all, move it back into the tables
  and update the summary counts.
- **CSS-in-JS with computed values.** If a color or spacing value is computed at runtime
  (e.g. `darken(theme.primary, 0.1)`), do not flag it as raw - it already derives from a
  token. Only flag literals that do not reference any token.
