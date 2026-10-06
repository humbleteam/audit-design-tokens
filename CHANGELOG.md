# Changelog

## [1.9.0] - 2026-10-06

- Every inventory count in the summary now carries the threshold that decides whether it is
  a finding. `Distinct font sizes in use: <count> (target: 8-10 or fewer)` printed its
  bound; `Radius variants: <count> · Shadow variants: <count>` printed none, although
  Step 2 sets a line for both. So `Radius variants: 5` read as a fact rather than a finding,
  and the one place a reader meets the number was the one place that could not say five is
  over the line while two is fine.
- The bound printed is the one that opens a row: 8-10 or fewer font sizes, 3-4 or fewer
  radius and shadow variants. Step 2's tighter typical scale (`sm` / `md` / `lg` radius, 2-3
  shadow depths) describes a considered scale rather than the accretion line, and printing
  it instead would mark a clean four-step radius scale as over target.
- Drift counts still carry no target, and the rule now says why: any value above zero is a
  finding for raw colors, off-scale spacing and hardcoded z-index, so a target beside one of
  those would read as a tolerance this skill does not grant. The distinction is inventory
  against drift, not an oversight on two lines.
- The bound is also what makes the summary reconcilable by eye against the P1 table. Since
  1.8.0 an over-threshold inventory category opens exactly one P1 row, so a count above its
  threshold with no row is a gap in the report - and without the bound on the line, that gap
  looked exactly like a clean count.
- The README's worked example and its closing paragraph moved with the shape: both radius
  and shadow show their target, and the paragraph that explains why all three inventory
  categories open no rows now points at the bounds the summary prints. No threshold changed,
  no severity rule changed, and the reconciliation rule for inventory counts is unchanged -
  a category still opens either no row or one row, never a row per value.

## [1.8.0] - 2026-09-30

- An inventory finding past its threshold has a row it can actually be written in. Step 3
  told the report to `open a P1 row only for what is past the threshold` while all three
  tables are keyed by `File:line`, and font-size sprawl has no single line - the count is
  spread over every file that sets the property. Two of the four columns had no legal
  content either: `Value` because the finding is a set of thirteen sizes rather than one
  value, and `Nearest token` because the finding is how many values exist, not how far one
  sits from a token. The worked example in the README never reached the case, since all
  three of its inventory categories sit inside their thresholds, so the shape existed
  nowhere in the repo.
- The case is now an edge case with the row rendered. One row per over-threshold category,
  never one per value; the location cell says there is no single location; `Nearest token`
  reads `not applicable` and the entry separates that from the `none defined` cell already
  in Step 3, which answers a different question. The per-value list Step 2 collects goes
  under the table as that row's evidence, and the reconciliation rule is unchanged - an
  inventory category opens no row or one row, so its summary number is still a count of
  distinct values and is never matched against rows.

## [1.7.0] - 2026-09-22

- The near-duplicate gate says how a role is derived instead of assuming one exists. Step 2
  flagged pairs "within the same category (background, text, border)", a classification
  neither Step 1 nor Step 2 produces: Step 1's categories are the value-shape ones the
  summary table is keyed by (color, spacing, font-size, radius, shadow, z-index), and nothing
  anywhere recorded what a color is used for. The gate is now named as a role, so it no
  longer collides with the Step 1 category it is not, and each side carries its own
  derivation - a literal's role from the property of the declaration it sits on, which the
  walk is already reading to get the line number, and a token's from its `var(--name)` call
  sites, or from its name only where the name states the role outright.
- The shape the rule could derive least is the shape it matters most for, and that is now
  written down rather than left to fail quietly. Step 1 records `name -> value` and never
  visits a call site, so a token that is referenced nowhere and named for nothing has no
  derivable role - and token-to-token is the only pair that earns its own P2 row and the only
  one where neither side sits on a declaration to be read.
- A pair with one or both roles underivable is reported, with the unknown side named in the
  fix cell and the fix asked as a question rather than written as a merge. Dropping it would
  have the gate discard exactly the findings it exists to catch, with no count, no row and
  nothing downstream showing that a pair had been seen and set aside.

## [1.6.0] - 2026-09-16

- The README's example output is a complete report instead of an abridged one. It
  carried a note saying the P2 table and the rename proposal were cut and that the
  summary counts therefore ran ahead of the rows shown - which is the one thing the
  reconciliation rule added in 1.4.0 forbids, demonstrated by the only rendered report
  in the repo. Two shapes had never been shown anywhere as a result: the token-to-token
  near-duplicate, which 1.4.0 named as the only pair that earns its own row and whose
  fix edits the token set rather than a call site, and the rename proposal that pair
  produces for the loser's call sites.
- The example's counts now reconcile the way Step 3 asks. Drift counts equal their rows
  (three raw colors, one off-scale value, three z-index literals), the pair count states
  its split, and font size, radius and shadow are listed by value as inventory counts
  that open no rows - the case Step 3 describes and nothing rendered.
- The P2 rows also render two rules that existed only as prose: a `none defined` nearest
  token where the category has no baseline, and the fix that follows from it, which is to
  define the token rather than to name one that does not exist.

## [1.5.0] - 2026-09-11

- A token's value is now recorded per theme. Step 1 has always searched `[data-theme]`
  selectors alongside `:root`, and a themed codebase defines the same name in several of
  them on purpose, but the record was `name -> value`: one slot per name, holding whichever
  block was read last. Everything downstream then read a half-palette, and the half it lost
  is the one P0 is defined by - a hardcoded color inside a component that renders under an
  alternate theme.
- Nearest token is resolved inside one theme. A literal is compared against the palette of
  the theme its call site renders in - the selector it sits under, or the default theme when
  it sits under none. Compared against the wrong palette, a `#111111` in a dark block returns
  the token furthest from it, and the suggested fix paints a dark surface with a light one.
- Near-duplicate pairs are found inside one theme, for the reason the pair rule already
  implies: two values that never render together are not merge candidates. One name across
  two themes is never a pair, and two names that collide in one theme and not in another are
  a pair only in the theme where they collide, which the row now names. Without this, the
  token-to-token shape - the one that earns its own row and whose fix edits the token set -
  would propose merging a light token into a dark one.
- Summary gains a `Themes found` line, so a report says which palettes it read. New edge
  case for a themed codebase, including the token defined in the base block and missing from
  an override: it falls back to the base value, which is how a dark mode ends up with one
  light surface, but nothing bypassed a token, so it is named as a gap and opens no row.
- README gains the matching FAQ answer and a How-it-works line.

## [1.4.0] - 2026-09-06

- A literal that carries two drift categories now has a defined place in the
  report. A raw `#F7F7F7` next to a `--color-surface-muted: #F8F8F8` is both a
  raw-color finding and half a near-duplicate pair, and the reconciliation rule
  demanded every drift finding be a row in exactly one severity table with the
  summary count equal to the rows. One row left the pair count unreconciled; two
  rows put the same literal in two tables and billed the same fix twice, since
  replacing the literal with the token also ends the pair.
- Near-duplicate pairs are now classified by what each side is. Token and token
  is the only shape that earns its own row, and its fix edits the token set
  rather than a call site. Literal and token dissolves into the raw-color row
  that already names the token as its nearest. Literal and literal dissolves the
  same way where a color baseline exists, and where none does it is the whole
  finding, since neither side is drift against anything.
- Near-duplicate pairs moved out of the row-matched drift counts into their own
  reconciliation rule: the summary states the count and its split
  (`3 (1 token-to-token, 2 dissolved by the raw-color rows above)`), which is
  what makes the number check out without duplicating a row.
- The README example carried the defect in miniature - a `#F7F7F7` row whose
  nearest token was a near-duplicate of it, with neither the relationship nor
  the shared fix stated. It now shows both, and the How-it-works list carries
  the one-literal-one-row rule.

## [1.3.0] - 2026-08-18

- Token discovery no longer misses most of the token set. The CSS custom-property
  pattern was anchored to the value (`--[a-z-]+:\s*(#|rgb|hsl|oklch)`), so it
  matched colors only, and its name class excluded digits, so it also missed
  `--blue-500` and `--gray-100`. Against a ten-token `:root` block it found two.
  An undiscovered token is not treated as missing but as absent, so every correct
  use of its value elsewhere was reported as raw drift against a token that
  exists. The pattern now matches the declaration and Step 1 classifies by the
  shape of the value, with a category table.
- A category with no tokens now has defined behavior. The stop rule fired only on
  a completely empty token set, so the common shape - brand colors defined, no
  spacing scale - reached Step 2, where the spacing base was to be inferred from
  a token list that contained no spacing. The fallback ("most commonly 4px or
  8px") is the invented baseline the skill's own FAQ promises it never uses.
  Categories without a baseline are not scanned as drift; categories that compare
  the codebase against itself (font-size sprawl, radius and shadow counts,
  near-duplicate colors, ungoverned z-index) still run.
- The summary can no longer report `0` for a category it never audited. It states
  which categories have a baseline, and an unaudited one reads
  `no baseline - no spacing tokens found` with what would unlock it. The
  `Nearest token` column says `none defined` rather than naming a token invented
  to fill the cell.
- Edge case for a partial token set, and the README gains a matching FAQ answer
  and a coverage line in the example summary.

## [1.2.0] - 2026-08-11

- Every raw color now has a severity bucket. A literal on no interactive state
  and in no themed component matched none of P0, P1 or P2, so the category
  Step 2 calls the highest-value one had no home in an unthemed codebase, and
  the finding could only be dropped or inflated to P0 against the skill's own
  rule. It defaults to P1, with a stated downgrade to P2 for a genuinely
  one-off literal.
- Summary/table reconciliation rewritten. "Counts in the summary must equal the
  row counts" was unsatisfiable as written: distinct font sizes, radius and
  shadow variants count values in use rather than findings, and a near-duplicate
  pair is one finding across two locations. Drift counts now reconcile against
  rows, inventory counts against the listed values.
- README example gains a P1 raw-color row showing the new default, and is
  marked abridged so its summary counts no longer read as a rule violation.

## [1.1.0] - 2026-08-05

- Edge case: literals that are raw on purpose (inline SVG brand marks, vendor
  stylesheets, email templates where CSS custom properties do not resolve) are
  excluded from the drift count and listed under an "Excluded from the count"
  note instead, so the exclusion stays reviewable rather than silent.

## [1.0.0] - 2026-07-12

- Initial release: scans CSS, SCSS, Tailwind config, styled-components themes, and inline
  styles for raw colors, off-scale spacing, near-duplicate colors, font-size sprawl, and
  hardcoded z-index values.
- Severity model: P0 (breaks theming), P1 (scale drift), P2 (consolidation candidate).
- Report includes file:line evidence for every finding and an ordered, smallest-diff-first
  consolidation plan. Token renames are always proposed separately, never applied silently.
