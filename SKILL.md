---
name: design-code-sync
version: 1.0.0
description: "Audit and sync token usage between a coded component and its Figma counterpart. Detects naming drift, value drift, missing bindings, and raw Foundation tokens used in Figma. Syncs in the direction indicated by the user or inferred from recent changes. Invoked as a slash command: /design-code-sync <component-name>"
---

# Design ↔ Code Sync

Audit token usage between a coded component and its Figma counterpart, then sync any drift. The skill reads component source files and Figma variable bindings, builds a comparison table, reports every discrepancy, and offers targeted fixes.

## Prerequisites

- `figma-cli` running at `/Users/Nubeh/figma-cli` with Figma Desktop CDP on port 9222
- The component's Figma page must be open in Figma Desktop
- Run all figma-cli evals via temp `.js` files: `node src/index.js eval --file /tmp/script.js`

## Usage

```
/design-code-sync <component-name>
```

Example: `/design-code-sync Button`, `/design-code-sync TextInput`

---

## Workflow

### Step 1: Locate component source files

Find the component folder at `packages/ui/src/components/<kebab-name>/`. Collect:

1. **`<name>.module.css`** — primary token source; scan every `var(--ds-*)` reference
2. **`<name>.tsx`** — scan `vars` prop for Mantine inline CSS variable overrides (e.g. `--button-bg`, `--button-hover`)
3. **`<name>.metadata.yaml`** — optional; may list expected variants and composition info

Extract all unique token references. Classify each as Foundation or Semantic:
- Starts with `--ds-theme-` → **Semantic**
- Starts with `--ds-` (no `theme`) → **Foundation**

### Step 2: Locate the Figma component set

```javascript
// Find the page for this component
const page = figma.root.children.find(p =>
  p.name.toLowerCase() === componentName.toLowerCase()
);
figma.currentPage = page;
const set = page.findOne(n => n.type === 'COMPONENT_SET');
```

If not found by exact name, try substring match or ask the user for the Figma page URL.

### Step 3: Extract Figma variable bindings

Write to `/tmp/sync-extract.js` and run via figma-cli:

```javascript
(async () => {
  const allVars = await figma.variables.getLocalVariablesAsync();
  const varMap = Object.fromEntries(allVars.map(v => [v.id, v.name]));

  function collectBindings(node, variantName, results) {
    if (node.boundVariables) {
      for (const [prop, binding] of Object.entries(node.boundVariables)) {
        const bindings = Array.isArray(binding) ? binding : [binding];
        for (const b of bindings) {
          if (b && b.id) {
            results.push({
              variant: variantName,
              nodeName: node.name,
              nodeType: node.type,
              property: prop,
              variable: varMap[b.id] || '(unresolved:' + b.id + ')'
            });
          }
        }
      }
    }
    if (node.children) {
      for (const child of node.children) collectBindings(child, variantName, results);
    }
  }

  const results = [];
  for (const variant of set.children) collectBindings(variant, variant.name, results);

  // Deduplicate
  const seen = new Set();
  return JSON.stringify(results.filter(r => {
    const k = r.property + '::' + r.variable + '::' + r.nodeName;
    if (seen.has(k)) return false;
    seen.add(k);
    return true;
  }));
})()
```

Collect the unique set of Figma variable names used (ignoring duplicates across variants).

### Step 4: Resolve variable aliases

For any Figma variable that is an alias (VARIABLE_ALIAS), resolve the full chain to get the final value. This is needed to detect value drift. Read `references/token-naming-bridge.md` for the resolution helper pattern.

### Step 5: Build the comparison table

Map code CSS vars to Figma variable names using the naming bridge in `references/token-naming-bridge.md`. For each token pair, determine sync status:

| Status | Code has it | Figma has it | Values match |
|--------|------------|--------------|--------------|
| ✅ Synced | ✓ | ✓ | ✓ |
| 🔴 Value drift | ✓ | ✓ | ✗ |
| 🟠 Naming drift | ✓ | ✓ (different path) | ✓ |
| 🟡 Missing in Figma | ✓ | ✗ | — |
| 🟡 Missing in code | ✗ | ✓ | — |
| 🔴 Raw Foundation in Figma | ✗ | ✓ (Foundation token) | — |

Read `references/drift-patterns.md` for a full taxonomy of each type and how to fix it.

### Step 6: Detect sync direction

Check git log for the component's files:

```bash
git log --oneline -5 -- packages/ui/src/components/<name>/
```

Ask the user: **"Did you modify code or Figma since the last sync?"**

If the user is explicit (e.g. "I changed Figma", "I changed code"), use that. Otherwise, use recency:
- Recent commits to the component files → **code is source of truth**
- No recent commits but user reports Figma changes → **Figma is source of truth**

When in doubt, **always ask** rather than guess — syncing in the wrong direction overwrites correct work.

### Step 7: Report

Present a structured audit table grouped by category:

```
## Sync Audit: <ComponentName>

### ✅ In sync (N)
[brief list]

### 🔴 Value drift (N)
| Property | Code value | Figma value | Fix |
...

### 🟠 Naming drift (N)
| Property | Code var | Figma var | Note |
...

### 🟡 In code, missing from Figma (N)
...

### 🟡 In Figma, missing from code (N)
...

### 🔴 Raw Foundation tokens in Figma (N)
| Property | Raw token used | Should reference |
...
```

After the report, state the detected sync direction and ask for confirmation before making any changes.

### Step 8: Sync

Execute only after user confirms the direction.

#### Code → Figma

For each discrepancy, update Figma variable bindings via figma-cli. Read `references/drift-patterns.md` for the fix pattern per category (rebind, rename alias, add missing binding).

Key eval patterns:

```javascript
// Rebind a fill to a different variable
const newVar = allVars.find(v => v.name === 'theme/color/border/default');
node.setBoundVariable('fills', figma.variables.createVariableAlias(newVar));

// Set stroke color
node.setBoundVariable('strokes', figma.variables.createVariableAlias(newVar));

// Set strokeWeight
const bwVar = allVars.find(v => v.name === 'border-width/1');
node.setBoundVariable('strokeWeight', figma.variables.createVariableAlias(bwVar));
```

#### Figma → Code

For each discrepancy:
1. If a Figma variable exists but CSS var is missing → add the CSS var reference to the component's CSS module
2. If a Figma variable path doesn't match code's naming convention → update the CSS var name in the component (and check if the token exists in semantic.tokens.json)
3. If a net-new token is needed → add it to `foundation.tokens.json` or `semantic.tokens.json`, rebuild tokens (`npm run build:tokens`), then update the CSS

### Step 9: Rebuild and verify

If code tokens changed:

```bash
cd packages/ui && npm run build:tokens
```

Check the output for collision warnings. Then run a follow-up extraction pass (Step 2–4) to confirm the audit now shows zero discrepancies.

---

## Important rules

- **Never delete** existing Figma bindings without confirmation — missing bindings may be intentional (e.g. Mantine handles them internally)
- **Foundation tokens in Figma** should always be replaced by a semantic alias, not a direct value. If no semantic token exists for the concept, create one first
- The **typography letter-spacing exception**: all roles currently resolve to `0px`. Figma binds it explicitly; code mirrors this. If a role needs non-zero tracking in the future, both sides must be updated
- **Mantine inline styles** take precedence over class-based CSS. Color tokens set via `vars` prop in TSX cannot be overridden in CSS modules — they must be changed in the `vars` function. Check button.tsx for an example
- Padding and gap in Figma are often set with spacing tokens that code leaves to Mantine defaults. Flag these but do not auto-sync unless the user explicitly asks
