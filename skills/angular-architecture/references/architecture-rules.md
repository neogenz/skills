# Architecture Rules Reference

## Dependency Rules (One-Way Graph)

```
feature ──────┬──▶ core
              ├──▶ ui
              └──▶ pattern

pattern ──────┬──▶ core
              └──▶ ui

layout ───────┬──▶ core
              ├──▶ ui
              └──▶ pattern

ui ───────────▶ (nothing)

core ─────────▶ (nothing)
```

**FORBIDDEN Dependencies:**

- feature → feature (NEVER)
- core → feature (NEVER)
- ui → core (NEVER)
- ui → feature (NEVER)
- pattern → pattern (NEVER)
- pattern → feature (NEVER)

## Extract One Level Up Rule

When logic needs to be shared:

| Scenario                              | Extract To            |
| ------------------------------------- | --------------------- |
| Between first-level features          | `core/` or `ui/`      |
| Between sub-features of same parent   | Parent feature folder |
| Between patterns                      | `core/` or `ui/`      |
| Service used by 2+ features           | `core/<domain>/`      |
| UI component used by 2+ features      | `ui/`                 |
| UI component used only in one feature | Keep in feature       |

## Decision Tree for Placement

```
Is it headless logic (service, guard, interceptor)?
├── YES → Is it needed from app start?
│         ├── YES → core/
│         └── NO → Is it used by 2+ features?
│                  ├── YES → core/
│                  └── NO → feature/<name>/
│
└── NO (has template) → Is it a generic UI widget?
                        ├── YES → Does it use app/business services?
                        │         ├── YES → Convert to pattern/
                        │         └── NO → ui/ (Angular/Material framework services OK)
                        └── NO → Is it a drop-in reusable use case?
                                 ├── YES → pattern/
                                 └── NO → Is it layout/navigation?
                                          ├── YES → layout/
                                          └── NO → feature/<name>/
```

## Common Scenarios

### Scenario: Service needed in multiple features

```
WRONG: Import service from feature/A into feature/B
RIGHT: Extract service to core/<domain>/
```

### Scenario: UI component needs data from service

```
WRONG: Inject app/business service in UI component (UserService, AuthStateService, etc.)
RIGHT: Either:
  1. Pass data via inputs from parent component
  2. Convert to pattern if complex
NOTE: Angular/Material framework services (MatDialogRef, ElementRef, DestroyRef) ARE allowed in UI
```

### Scenario: Lazy feature needs auth state

```
CORRECT: Import from core/auth/ (core → feature is allowed)
```

### Scenario: Two features share similar form logic

```
Options:
  1. Different behavior/UI → Keep isolated (duplication OK)
  2. Same behavior, different UI → Extract behavior to core/
  3. Different behavior, same UI → Extract UI to ui/
  4. Same behavior and UI → Create pattern/
```

## Lazy Loading Best Practices

**DO:**

- Always use `loadChildren()` for features (NOT `loadComponent`)
- Use `@defer` for heavy components within features (charts, editors, maps)
- Keep sub-features lazy loaded when possible
- Start with lazy features from day one

**DON'T:**

- Create eager features (even if only one feature exists)
- Lazy load every small component (causes waterfall)
- Import eagerly from lazy features

## Isolation Guarantees

When isolation is maintained:

- **Local impact**: Feature changes can't affect other features
- **Local testing**: Feature tests can't break other features
- **Throw-away**: Features can be replaced without affecting others
- **Black box**: Internal quality issues are contained

## Anti-patterns to CATCH

**Architecture Violations:**

- Feature importing from sibling feature
- UI component with injected app/business services (Angular/Material framework services are OK)
- Core importing from feature
- Pattern importing from feature
- God services shared across too many features

**Bundling Issues:**

- Large eager bundle (move to features)
- Heavy components in eager path
- Unused code in bundles

**Isolation Breaks:**

- Shared mutable state between features
- Direct feature-to-feature communication
- Tightly coupled feature implementations
