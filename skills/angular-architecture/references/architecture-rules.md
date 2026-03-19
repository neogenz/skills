# Architecture Rules Reference

## Dependency Graph (One-Way)

```
feature/ ──────┬──▶ pattern/
               ├──▶ core/
               ├──▶ ui/
               └──▶ styles/

pattern/ ──────┬──▶ core/
               ├──▶ ui/
               └──▶ styles/

layout/  ──────┬──▶ core/
               ├──▶ pattern/
               ├──▶ ui/
               ├──▶ layout/ (self-composition)
               └──▶ styles/

ui/      ──────▶ ui/ (internal composition only)

core/    ──────▶ styles/

styles/  ──────▶ (nothing - foundation layer)
```

## FORBIDDEN Dependencies

| From | To | Why |
|------|----|-----|
| `feature/` → `feature/` | NEVER | Complete isolation between features |
| `ui/` → `core/` | NEVER | UI must not inject app services |
| `ui/` → `pattern/` | NEVER | UI is self-contained |
| `ui/` → `feature/` | NEVER | UI is self-contained |
| `ui/` → `layout/` | NEVER | UI is self-contained |
| `ui/` → `styles/` | NEVER | UI is self-styled (inline or component styles) |
| `pattern/` → `feature/` | NEVER | Would create circular dependency |
| `pattern/` → `pattern/` | NEVER | Extract shared parts to `core/` or `ui/` |
| `pattern/` → `layout/` | NEVER | Pattern is lower level |
| `core/` → `feature/` | NEVER | Core is lower level |
| `core/` → `pattern/` | NEVER | Core is lower level |
| `core/` → `ui/` | NEVER | Core has no templates |
| `core/` → `layout/` | NEVER | Core is lower level |
| `layout/` → `feature/` | NEVER | Layout is shared across features |

## Decision Tree for Placement

```
Is it headless logic (service, guard, interceptor, function)?
├── YES → Is it needed from app start?
│         ├── YES → core/
│         └── NO → Is it used by 2+ features?
│                  ├── YES → core/
│                  └── NO → feature/<name>/
│
└── NO (has template) → Is it a generic UI widget?
                        ├── YES → Does it inject app/business services?
                        │         ├── YES → pattern/ (or convert to pure UI)
                        │         └── NO → ui/
                        │               (Angular/Material framework services OK)
                        └── NO → Is it a drop-in reusable business component?
                                 ├── YES → pattern/
                                 └── NO → Is it layout/navigation/shell?
                                          ├── YES → layout/
                                          └── NO → feature/<name>/
```

## "Extract One Level Up" Rule

When logic needs to be shared:

| Scenario | Extract To |
|----------|------------|
| Between first-level features | `core/` (services) or `ui/` (components) or `pattern/` (both) |
| Between sub-features of same parent | Parent feature folder |
| Between patterns | `core/` (services) or `ui/` (components) |
| Service used by 2+ features | `core/<domain>/` |
| UI component used by 2+ features | `ui/` |
| UI component used only in 1 feature | Keep in that feature |
| Business component used by 2+ features | `pattern/` |

### The Sharing Decision Matrix

When two features need the same thing, ask two questions:

| Same behavior? | Same UI? | Action |
|----------------|----------|--------|
| NO | NO | Keep duplicated (isolation wins) |
| YES | NO | Extract behavior to `core/`, UI stays in each feature |
| NO | YES | Extract UI to `ui/`, behavior stays in each feature |
| YES | YES | Create `pattern/` |

## Lazy Loading Rules

**DO:**
- Always use `loadChildren()` for features (NOT `loadComponent`)
- Use `@defer` for heavy components within features (charts, editors, maps, data tables)
- Keep sub-features lazy loaded when possible
- Start with lazy features from day one — even for the first/only feature

**DON'T:**
- Create eager features (even if only one feature exists)
- Lazy load every small component (causes waterfall of requests)
- Import eagerly from lazy features
- Use `loadComponent()` at the app.routes level for features

## Feature Service Scoping

```typescript
// WRONG — breaks isolation, leaks to initial bundle
@Injectable({ providedIn: 'root' })
export class FeatureService {}

// CORRECT — scoped to feature's lazy injector
@Injectable()
export class FeatureService {}

// Register in route config
export default [
  {
    path: '',
    providers: [FeatureService],  // ← scoped here
    children: [/* ... */],
  },
] satisfies Routes;
```

Consequences of route-level scoping:
- Service is in the lazy bundle (not initial bundle)
- Service is only available within the feature
- Trying to inject it from a sibling feature throws `NullInjectorError` at runtime — architecture enforcement for free
- New instance per feature activation (not a global singleton)

## Import Organization

Within any TypeScript file, order imports as:

```typescript
// 1. Angular core
import { Component, inject, signal } from '@angular/core';

// 2. Angular ecosystem
import { MatButtonModule } from '@angular/material/button';
import { RouterLink } from '@angular/router';

// 3. Third-party
import { z } from 'zod';

// 4. Project path aliases (@core/, @ui/, @pattern/)
import { AuthService } from '@core/auth';
import { ButtonComponent } from '@ui/button';

// 5. Relative imports (same feature/layer only)
import { MyFormComponent } from './my-form.component';
```

Rules:
- Use path aliases (`@core/`, `@ui/`) over deep relative paths
- Blank line between sections
- No barrel exports within features (avoid circular deps)

## Isolation Guarantees

When isolation is maintained:

- **Local impact**: Changing a feature can't affect other features, core, or UI
- **Local testing**: Feature tests can't break other features' tests
- **Throw-away**: Features can be replaced entirely without affecting the rest
- **Black box**: Poor internal quality stays contained
- **Delivery speed**: Ship fast without fear of regressions in other features

## Anti-patterns to Detect

### Architecture Violations
- Feature importing from sibling feature
- UI component with injected app/business services
- Core importing from feature, pattern, or UI
- Pattern importing from feature or sibling pattern
- Feature service using `providedIn: 'root'`

### Bundling Issues
- Large eager bundle (move logic to features)
- Heavy components in eager path (use `@defer`)
- `loadComponent` used for features instead of `loadChildren`

### Abstraction Violations
- "Framework in a framework" — wrapping Angular/NgRx into custom abstractions
- Base components, custom `*myOrgFor`/`*customIf` directives (break `ng update` migrations)
- Organizing `core/` by type instead of domain
- God services shared across too many features
- Extracting shared code after only 1-2 occurrences (wait for 3+)
