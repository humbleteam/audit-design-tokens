# Changelog

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
