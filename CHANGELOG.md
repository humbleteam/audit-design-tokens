# Changelog

## [1.0.0] - 2026-07-12

- Initial release: scans CSS, SCSS, Tailwind config, styled-components themes, and inline
  styles for raw colors, off-scale spacing, near-duplicate colors, font-size sprawl, and
  hardcoded z-index values.
- Severity model: P0 (breaks theming), P1 (scale drift), P2 (consolidation candidate).
- Report includes file:line evidence for every finding and an ordered, smallest-diff-first
  consolidation plan. Token renames are always proposed separately, never applied silently.
