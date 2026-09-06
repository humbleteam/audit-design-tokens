# Changelog

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
