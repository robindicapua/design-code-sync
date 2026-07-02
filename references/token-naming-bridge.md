# Token Naming Bridge

Rules for converting between CSS custom property names (code) and Figma variable path names.

## Convention

CSS vars and Figma variable paths follow the same structure, just with different separators and prefix:

| Side | Format | Example |
|------|--------|---------|
| Code (CSS var) | `--ds-<path-with-dashes>` | `--ds-color-border-default` |
| Figma variable | `<path-with-slashes>` (in the matching collection) | `color/border/default` |

The tier (Foundation vs Semantic) is **not** encoded in the name on either side — see
"Foundation vs Semantic" below. In code both tiers are `--ds-<group>-*`; in Figma both live
under `color/*`, `spacing/*`, … but in **separate collections** (Foundation, Semantic).

### CSS var → Figma path

1. Strip `--ds-` prefix
2. Replace `-` with `/`

```
--ds-color-border-default   →  color/border/default     (Semantic collection)
--ds-radius-default         →  radius/default           (Semantic collection)
--ds-color-gray-300         →  color/gray/300           (Foundation collection)
--ds-border-width-1         →  border-width/1           (Foundation collection)
--ds-spacing-4              →  spacing/4                (Foundation collection)
```

### Figma path → CSS var

1. Prepend `--ds-`
2. Replace `/` with `-`

```
color/interactive/brand/hover   →  --ds-color-interactive-brand-hover   (Semantic)
typography/label/font-weight    →  --ds-typography-label-font-weight    (Semantic)
border-width/1                  →  --ds-border-width-1                  (Foundation)
```

## Ambiguous cases

### Hyphenated token names

Some token names contain hyphens that are part of the name (not separators). These survive the conversion intact:

```
--ds-color-background-utility-error  →  color/background/utility/error
--ds-typography-body-font-family     →  typography/body/font-family
--ds-spacing-gap-xs                  →  spacing/gap-xs
--ds-spacing-content-md              →  spacing/content-md
```

The bridge is purely mechanical — every `-` becomes `/` — so multi-word segment names like `gap-xs` become `gap/xs` in the Figma path, which is correct.

### Mantine internal CSS variables

These are NOT design tokens — they're Mantine's own variables set via the `vars` prop:

```
--button-bg
--button-hover
--button-color
--button-bd
--input-bd
```

These do not have Figma equivalents. The Figma component uses direct variable bindings on `fills`/`strokes` instead. Do not try to map these.

## Foundation vs Semantic

The tier is carried by **collection**, not by the token name. Figma has two variable collections:

- **Foundation** (mode: single) — raw palette/scales, *scale*-named: `color/gray/300`, `color/blue/600`, `spacing/4`, `radius/8`.
- **Semantic** (modes: Light / Dark) — roles, *role*-named: `color/background/default`, `color/border/default`, `color/interactive/brand/*`, `color/content/*`. Each aliases a Foundation variable.

Both collections use the same top-level groups (`color/`, `spacing/`, …), so tell them apart by
the **collection** a variable belongs to and by how the name reads — a **role** (`background`,
`content`, `interactive`) is Semantic; a **scale** (`gray/300`, `blue/600`, `4`) is Foundation.
This mirrors the code side, where both tiers are `--ds-<group>-*` and role-vs-scale marks the tier
(there is no `theme`/`semantic` segment in the name).

Foundation variables should only appear in Figma as **intermediate aliases** — i.e. a Semantic
variable references a Foundation variable. If a component binding points **directly** to a
Foundation-collection variable, that is a raw Foundation token and should be flagged as drift.

```
✅ component → color/border/default (Semantic) → color/gray/300 (Foundation)   [via semantic alias]
🔴 component → color/gray/300 (Foundation)                                      [raw — no semantic alias]
```

## Alias resolution helper (figma-cli)

```javascript
function resolveChain(varId, varMap, depth = 0) {
  if (depth > 10) return { error: 'circular' };
  const v = varMap[varId];
  if (!v) return { error: 'missing' };
  const modeValues = {};
  for (const [modeId, val] of Object.entries(v.valuesByMode)) {
    if (val && val.type === 'VARIABLE_ALIAS') {
      modeValues[modeId] = {
        alias: varMap[val.id]?.name,
        resolved: resolveChain(val.id, varMap, depth + 1)
      };
    } else {
      modeValues[modeId] = { value: val };
    }
  }
  return { name: v.name, type: v.resolvedType, values: modeValues };
}
```
