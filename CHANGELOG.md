# Changelog

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
