# Token Naming Bridge

Rules for converting between CSS custom property names (code) and Figma variable path names.

## Convention

CSS vars and Figma variable paths follow the same structure, just with different separators and prefix:

| Side | Format | Example |
|------|--------|---------|
| Code (CSS var) | `--ds-<path-with-dashes>` | `--ds-theme-color-border-default` |
| Figma variable | `<path-with-slashes>` | `theme/color/border/default` |

### CSS var → Figma path

1. Strip `--ds-` prefix
2. Replace `-` with `/`

```
--ds-theme-color-border-default  →  theme/color/border/default
--ds-theme-radius-default        →  theme/radius/default
--ds-border-width-1              →  border-width/1
--ds-typography-font-size-14     →  typography/font-size/14
--ds-spacing-4                   →  spacing/4
```

### Figma path → CSS var

1. Prepend `--ds-`
2. Replace `/` with `-`

```
theme/color/interactive/brand/hover  →  --ds-theme-color-interactive-brand-hover
theme/typography/label/font-weight   →  --ds-theme-typography-label-font-weight
border-width/1                       →  --ds-border-width-1
```

## Ambiguous cases

### Hyphenated token names

Some token names contain hyphens that are part of the name (not separators). These survive the conversion intact:

```
--ds-theme-color-background-utility-error  →  theme/color/background/utility/error
--ds-theme-typography-body-font-family     →  theme/typography/body/font-family
--ds-theme-spacing-gap-xs                  →  theme/spacing/gap-xs
--ds-theme-spacing-content-md              →  theme/spacing/content-md
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

Foundation tokens (no `theme/` prefix in Figma) should only appear in Figma as **intermediate aliases** — i.e. a Semantic variable references a Foundation variable. If a Figma component binding points directly to a Foundation token (e.g. `spacing/4`, `color/blue/600`), that is a **raw Foundation token** in Figma and should be flagged as drift.

```
✅ component → theme/color/border/default → color/gray/300  (via semantic alias)
🔴 component → color/gray/300  (raw Foundation — no semantic alias)
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
