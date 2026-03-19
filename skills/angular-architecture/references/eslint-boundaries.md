# ESLint Boundaries — Automated Architecture Validation

The `eslint-plugin-boundaries` plugin enforces the dependency graph at build time. Without it, architecture rules rely on "hope-based" code review. With it, violations are caught instantly in the IDE and CI.

---

## Why Automated Validation Matters

- Humans miss violations in code review — especially under time pressure
- Architecture degrades gradually (one small violation leads to more)
- New team members don't know the rules yet
- The plugin acts as a **friendly senior reviewer** that never forgets

---

## Element Type Definitions

Each architectural layer is registered as a "type" with a file pattern:

```javascript
// eslint.config.js
import boundaries from 'eslint-plugin-boundaries';

export default [
  {
    plugins: { boundaries },
    settings: {
      'boundaries/elements': [
        { type: 'main',           pattern: 'main.ts' },
        { type: 'app',            pattern: 'app/app*.ts' },
        { type: 'core',           pattern: 'core/**/*' },
        { type: 'ui',             pattern: 'ui/**/*' },
        { type: 'layout',         pattern: 'layout/**/*' },
        { type: 'pattern',        pattern: 'pattern/**/*' },
        { type: 'feature-routes', pattern: 'feature/([^/]+)/*.routes.ts', capture: ['feature'] },
        { type: 'feature',        pattern: 'feature/([^/]+)/**/*', capture: ['feature'] },
        { type: 'styles',         pattern: 'styles/**/*' },
        { type: 'env',            pattern: 'environments/**/*' },
        { type: 'shared',         pattern: 'shared/**/*' },
        { type: 'test-spec',      pattern: '**/*.spec.ts' },
      ],
    },
  },
];
```

The `capture` groups on `feature` and `feature-routes` are critical — they enable the "same feature" constraint that allows sub-features to share within their parent while blocking cross-feature imports.

### About `shared/` and `env/`

These are **helper types** in the eslint boundary config, not architectural layers:

- **`shared/`** — External shared code (e.g. monorepo shared package, `pulpe-shared` types). Contains TypeScript types, interfaces, constants, and pure utility functions shared between frontend and backend. All layers can import from it. It has **no Angular dependencies** — just plain TypeScript.
- **`env/`** — Build-time environment files (`environments/environment.ts`). Contains feature flags and build-time configuration. All layers can import from it.

---

## Access Rules (Dependency Matrix)

```javascript
rules: {
  'boundaries/element-types': [2, {
    default: 'disallow',
    rules: [
      // main.ts can only import app files and env
      { from: 'main',           allow: ['app', 'env'] },

      // core: headless, base layer
      { from: 'core',           allow: ['shared', 'core', 'env', 'styles'] },

      // ui: self-contained, no app dependencies
      { from: 'ui',             allow: ['shared', 'ui', 'env'] },

      // layout: app shell, can use core + ui + pattern
      { from: 'layout',         allow: ['shared', 'core', 'ui', 'layout', 'pattern', 'env', 'styles'] },

      // pattern: reusable business components
      { from: 'pattern',        allow: ['shared', 'core', 'ui', 'env', 'styles'] },
      // NOTE: pattern CANNOT import pattern (no pattern-to-pattern)

      // feature: isolated black boxes
      {
        from: 'feature',
        allow: [
          'shared', 'core', 'ui', 'pattern', 'env', 'styles',
          // Can import from SAME feature only (using capture group)
          ['feature', { feature: '${from.feature}' }],
        ],
      },

      // feature-routes: can reference sibling feature routes (for shared sub-routes)
      {
        from: 'feature-routes',
        allow: [
          'shared', 'core', 'pattern', 'env',
          ['feature', { feature: '${from.feature}' }],
          ['feature-routes', { feature: '!${from.feature}' }],
          // ↑ Can reference OTHER features' routes (for loadChildren only)
        ],
      },

      // app files: can import everything (bootstrap)
      { from: 'app', allow: ['shared', 'app', 'core', 'ui', 'layout', 'pattern', 'feature-routes', 'feature', 'env'] },

      // tests: can import anything they need
      { from: 'test-spec', allow: ['shared', 'core', 'ui', 'layout', 'pattern', 'feature', 'env'] },
    ],
  }],
},
```

---

## TypeScript Path Aliases

The boundary rules work with path aliases. Configure in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "@core/*": ["app/core/*"],
      "@ui/*": ["app/ui/*"],
      "@layout/*": ["app/layout/*"],
      "@pattern/*": ["app/pattern/*"],
      "@features/*": ["app/feature/*"],
      "@env/*": ["environments/*"],
      "@app/*": ["app/*"]
    }
  }
}
```

And configure the ESLint import resolver:

```javascript
settings: {
  'import/resolver': {
    typescript: {
      project: ['tsconfig.json', 'tsconfig.app.json'],
    },
  },
},
```

---

## What Each Rule Prevents

| Rule | What it catches |
|------|----------------|
| `feature` cannot import `feature` (different) | Cross-feature coupling |
| `ui` cannot import `core` | UI injecting app services |
| `core` cannot import `feature`, `pattern`, `ui` | Upward dependency |
| `pattern` cannot import `feature` | Circular dependency |
| `pattern` cannot import `pattern` | Pattern coupling |
| `layout` cannot import `feature` | Layout depending on lazy code |

---

## The `feature-routes` Special Case

Route files (`*.routes.ts`) get their own type because they have a unique permission: they can reference OTHER features' route files via `loadChildren`. This is the only legal cross-feature reference — and it's only for routing, not implementation imports.

```typescript
// feature/order/order.routes.ts — ALLOWED
{
  path: ':id',
  loadChildren: () => import('../detail/detail.routes'),
  // This imports a sibling feature's ROUTES file, which is OK
}

// feature/order/order.component.ts — FORBIDDEN
import { DetailService } from '../detail/detail.service';
// This imports a sibling feature's IMPLEMENTATION, which is blocked
```

---

## Setting Up From Scratch

1. Install dependencies:
```bash
npm install -D eslint-plugin-boundaries eslint-import-resolver-typescript
```

2. Add element definitions matching your folder structure
3. Add access rules (copy the matrix above as starting point)
4. Run `npx eslint .` to find existing violations
5. Fix violations or add targeted `eslint-disable` comments with migration plan

---

## Common ESLint Boundary Errors

**`boundaries/element-types: element of type "feature" is not allowed to import element of type "feature"`**
→ You're importing from a sibling feature. Extract to `core/`, `ui/`, or `pattern/`.

**`boundaries/element-types: element of type "ui" is not allowed to import element of type "core"`**
→ Your UI component is injecting an app service. Convert to inputs/outputs or move to `pattern/`.

**`boundaries/element-types: element of type "core" is not allowed to import element of type "feature"`**
→ Core is depending on a feature. Extract the shared logic within `core/` itself.
