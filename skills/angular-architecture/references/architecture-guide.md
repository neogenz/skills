# Enterprise Angular Architecture — Deep Guide

This document covers the theory and rationale behind the architecture. Read this when you need to understand **why** a rule exists, not just what it is.

## Core Thesis

The architecture enforces a strict, one-way dependency graph to maintain development velocity over the long term. It relies on:

1. **Standalone APIs exclusively** — no NgModules for application code
2. **Clear separation between template context and injector hierarchy** — two parallel Angular systems
3. **Unidirectional dependency graph** — prevents cyclic coupling

The guiding principle: **Isolation is 3-5x more valuable than DRY in frontend.**

---

## Two Parallel Angular Systems

Understanding these two systems is fundamental to making correct architectural decisions.

### 1. Standalones (Template Context)

Components, Directives, Pipes. They manage their own dependencies via `imports: []`. Each standalone builds its own template context — every dependency is an explicit import on the TypeScript file level, which equals a line in the dependency graph.

This parity (import = dependency graph edge) makes it easy to reason about architectural impact.

### 2. Injectables (Injector Hierarchy)

Services, Guards, Interceptors, Configs. Managed by the injector hierarchy, not by template context.

Key properties:
- **Root injector** → application-wide singletons (`providedIn: 'root'`)
- **Lazy injector** → created per lazy-loaded feature (`EnvironmentInjector`), isolated from siblings
- **Element injector** → per component instance (rare, e.g. component stores)

### Injector Hierarchy and Isolation

When a feature is lazy-loaded, it creates its own `EnvironmentInjector`:

1. **Services in the lazy injector are invisible to sibling features** — trying to inject them throws `NullInjectorError`
2. **Lazy features CAN access services from the root/eager injector** (e.g. `core/` services)
3. **Same service in multiple lazy injectors = multiple instances** (one per injector)
4. **Same service in both lazy and eager = multiple instances**

This is architecture enforcement for free — the DI system itself prevents cross-feature coupling.

---

## Bundling and Performance

### Why Bundle Size Matters

The bottleneck is NOT just network — it's **JavaScript parsing and execution**. Even on fast connections with cached bundles, the browser must parse and execute all eager JS before the app becomes interactive.

### How the Bundler Works

The bundler (webpack/esbuild) uses dynamic `import()` as split points:

- **Static imports** → bundled into eager `main.js`
- **Dynamic imports** → extracted into lazy chunks

Architecture directly dictates bundle structure:
- `core/` and `layout/` → always eager (initial bundle)
- `feature/` → always lazy (separate bundles via `loadChildren`)
- `ui/` → bundler decides based on usage patterns (see types reference)
- `pattern/` → usually lazy, loaded with consuming feature

### `@defer` Lazy Loading

For heavy components within features (charts, editors, maps, data tables), use `@defer`:

```typescript
@defer {
  <app-heavy-chart [data]="chartData()" />
}
```

Only use `@defer` for genuinely heavy components. Don't defer everything — that creates request waterfalls.

---

## Isolation: The Central Principle

### Why Isolation > DRY

In frontend:
- Requirements are often **highly ad-hoc** (custom rules for tiny user subsets)
- User flows need **independent evolution** as business needs change
- Premature abstractions become **"god" components/services** trying to handle every edge case
- **3+ repetitions** should occur before considering abstraction

### Isolation Guarantees

When maintained:
- **Local impact**: Feature changes can't affect other features
- **Local testing**: Feature tests can't break other features' tests
- **Throw-away**: Replace entire features without side effects
- **Black box**: Internal quality issues stay contained
- **Onboarding**: New team members are productive faster (smaller context to understand)

### The "Black Box" Benefit

Even if a feature's internal code quality is poor, isolation prevents that "dirtiness" from spreading. This lets teams:
- Optimize for delivery speed when needed
- Keep old features working while adopting new patterns in new features
- Replace features entirely instead of expensive refactoring

---

## Correct Level of Abstraction

### Stay at Framework Level

~95% of effort should go into **application business logic** that delivers user value. The correct abstraction level is:

1. Angular itself
2. Angular's signal-based state management (`signal()`, `computed()`, `resource()`)

That's it.

### What NOT to Abstract

- **Base components** wrapping Angular components (breaks `ng update` migrations)
- **Custom structural directives** like `*myOrgFor` / `*customIf` (broke control flow migration in Angular 17)
- **Custom testing abstractions** (broke with standalone APIs adoption)
- **State management wrappers** combining signals + services in "clever" abstractions

### Handling Verbosity

Verbosity decreases over time as Angular evolves:
- Angular 16: `takeUntilDestroyed` replaced manual `Subject` + `ngOnDestroy`
- Angular: `signal()`, `computed()`, `resource()` replaced BehaviorSubject patterns

Use schematics for code generation, consistent naming for search-and-replace, and `ng-morph` for large-scale code transformations.

---

## Recommended Folder Structure

```
src/
├── app/
│   ├── app.ts              # Root component
│   ├── app.config.ts       # Only calls provideCore({ routes })
│   ├── app.routes.ts       # Top-level route config
│   ├── core/               # Headless singletons
│   │   ├── auth/           # Domain: authentication
│   │   ├── user/           # Domain: user management
│   │   ├── order/          # Domain: shared order state
│   │   └── core.ts         # provideCore() export
│   ├── layout/             # Application shell(s)
│   │   ├── main-layout/
│   │   └── auth-layout/    # (if multiple layouts)
│   ├── ui/                 # Pure reusable components
│   │   ├── button/
│   │   ├── card/
│   │   ├── avatar/
│   │   └── dialog/
│   ├── pattern/            # Reusable business components
│   │   ├── document-manager/
│   │   └── approval-process/
│   ├── feature/            # Lazy-loaded pages
│   │   ├── dashboard/
│   │   ├── orders/
│   │   │   ├── orders.routes.ts
│   │   │   ├── orders.ts
│   │   │   ├── detail/     # Sub-feature
│   │   │   └── state/      # Feature-local state
│   │   └── settings/
│   └── styles/             # (optional) Design tokens, themes
└── environments/
```

---

## Function-Based Building Blocks

Angular supports function-based guards and interceptors. Treat them exactly like services for architecture purposes — implement in the same location:

```typescript
// core/auth/auth.guard.ts
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  return authService.isAuthenticated()
    ? true
    : inject(Router).createUrlTree(['/login']);
};
```

---

## Sharing Lazy Features as Sub-Routes

A shared sub-feature (e.g. generic detail view) can be referenced from multiple parent features via route config:

```typescript
// feature/order/order.routes.ts
export default [
  { path: '', component: OrderListComponent },
  {
    path: ':id',
    loadChildren: () => import('../detail/detail.routes'),
    // "../" — references sibling feature folder
    // This is allowed for ROUTE CONFIG only, not for implementation imports
  },
] satisfies Routes;
```

**Important distinction**:
- **Lazy sub-feature**: Located inside parent feature folder, shares parent's "black box" boundary
- **Shared feature**: Located in separate folder, complete implementation isolation, only referenced through route config
