# Design ↔ Code Sync Skill

Audit and sync token usage between a coded component and its Figma counterpart. Detects naming drift, value drift, missing bindings, and raw Foundation token usage — then syncs in the right direction.

## Prerequisites

- `figma-cli` installed at `/Users/Nubeh/figma-cli`
- Figma Desktop running with CDP on port 9222
- The component's Figma page open in Figma Desktop

## Usage

```
/design-code-sync <component-name>
```

Examples:
```
/design-code-sync Button
/design-code-sync TextInput
/design-code-sync Checkbox
```

## What it does

1. Reads all `var(--ds-*)` token references from the component's `.module.css` and `.tsx`
2. Reads all variable bindings from the Figma component set via figma-cli
3. Maps both sides using the CSS ↔ Figma naming convention
4. Reports every discrepancy, categorised by type
5. Asks which side changed, then syncs accordingly

## Drift categories it detects

| Category | Example |
|---|---|
| ✅ In sync | Token name and value match on both sides |
| 🔴 Value drift | Same variable name, different resolved value |
| 🟠 Naming drift | Same concept, different token path structure |
| 🟡 Missing in Figma | CSS var set in code, no Figma binding |
| 🟡 Missing in code | Figma bound, not set in CSS |
| 🔴 Raw Foundation in Figma | Component uses Foundation token directly (no semantic alias) |
| 🟡 Variant gap | Code variant has no Figma counterpart, or vice versa |

## Sync direction

The skill asks which side changed before making any edits. When in doubt it asks rather than guesses — syncing in the wrong direction overwrites correct work.

- **Code → Figma:** Updates variable bindings on Figma component nodes via figma-cli
- **Figma → Code:** Updates CSS module vars; adds/updates tokens in `semantic.tokens.json` if needed; rebuilds with `npm run build:tokens`

## Skill structure

```
design-code-sync/
  SKILL.md                          ← Full workflow (read this first)
  README.md                         ← This file
  references/
    token-naming-bridge.md          ← CSS var ↔ Figma path conversion rules + alias resolver
    drift-patterns.md               ← All 7 drift types with detection + fix recipes
```
