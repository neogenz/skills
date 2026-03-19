# Gotchas — Common Architecture Mistakes

These are the patterns that most frequently trip up developers (and AI assistants). Check this list when reviewing code or making placement decisions.

---

## 1. UI Component Injecting App Services

**Mistake**: Adding `inject(UserService)` or `inject(AuthStateService)` to a UI component.

**Why it's wrong**: UI components are pure presentation — they must be reusable across any feature without coupling to specific app state. Injecting app services creates hidden dependencies and prevents reuse.

**What's allowed in UI**: Angular/Material framework services only — `MatDialogRef`, `ElementRef`, `DestroyRef`, `Renderer2`, `ChangeDetectorRef`, `ViewContainerRef`, etc.

**Fix**:
- If the component needs data → parent passes it via inputs
- If it needs complex behavior + data → move to `pattern/`

```typescript
// ❌ WRONG — UI with app service
@Component({ selector: 'app-avatar' })
export class AvatarComponent {
  readonly #userService = inject(UserService); // FORBIDDEN
}

// ✅ CORRECT — UI with inputs only
@Component({ selector: 'app-avatar' })
export class AvatarComponent {
  readonly imageUrl = input.required<string>();
  readonly name = input.required<string>();
}

// ✅ ALSO CORRECT — Angular framework service in UI
@Component({ selector: 'app-confirm-dialog' })
export class ConfirmDialog {
  readonly #dialogRef = inject(MatDialogRef); // OK — framework service
  readonly #data = inject(MAT_DIALOG_DATA);   // OK — framework token
}
```

---

## 2. Feature Services with `providedIn: 'root'`

**Mistake**: Using `@Injectable({ providedIn: 'root' })` for services that belong to a single feature.

**Why it's wrong**:
- Service ends up in the eager initial bundle (bigger startup)
- Service becomes a global singleton (breaks feature isolation)
- Other features can accidentally depend on it

**Fix**: Use `@Injectable()` (no `providedIn`) and register in the feature's route `providers: []`.

```typescript
// ❌ WRONG
@Injectable({ providedIn: 'root' })
export class OrderStore {}

// ✅ CORRECT
@Injectable()
export class OrderStore {}

// feature/orders/orders.routes.ts
export default [
  {
    path: '',
    providers: [OrderStore, OrderApi],
    children: [/* ... */],
  },
] satisfies Routes;
```

---

## 3. `loadComponent` for Features

**Mistake**: Using `loadComponent` in `app.routes.ts` for top-level features.

**Why it's wrong**: `loadComponent` lazy-loads a single component. You can't add child routes, guards, or providers at the route level later without restructuring.

**Fix**: Always use `loadChildren` pointing to a routes file.

```typescript
// ❌ WRONG
{ path: 'orders', loadComponent: () => import('./feature/orders/orders') }

// ✅ CORRECT
{ path: 'orders', loadChildren: () => import('./feature/orders/orders.routes') }
```

---

## 4. Eager Features

**Mistake**: Making the first (or only) feature eager — loading it directly in `AppComponent` or via static route `component:`.

**Why it's wrong**: When a second feature needs to be added later, you face an inconsistency (one eager, one lazy) that requires restructuring.

**Fix**: Even for a single-feature app, make it a lazy feature from day one.

---

## 5. Core Organized by Building Block Type

**Mistake**: `core/services/`, `core/guards/`, `core/interceptors/`.

**Why it's wrong**: As the app grows, you end up with flat folders containing unrelated files. Domain grouping keeps related logic together.

**Fix**: Use domain-based folders.

```
// ❌ WRONG
core/
├── services/
│   ├── auth.service.ts
│   ├── user.service.ts
│   └── order.service.ts
├── guards/
│   └── auth.guard.ts

// ✅ CORRECT
core/
├── auth/
│   ├── auth.service.ts
│   └── auth.guard.ts
├── user/
│   └── user.service.ts
├── order/
│   └── order.service.ts
```

---

## 6. Premature Abstraction

**Mistake**: Extracting shared code after seeing it in just 2 places.

**Why it's wrong**: The abstraction may not fit future use cases. In frontend, ad-hoc requirements are common — the third occurrence reveals whether the pattern is truly shared or just superficially similar.

**Fix**: Wait for 3+ occurrences. Accept duplication in the meantime. Isolation guarantees that duplicated code can evolve independently.

---

## 7. Importing Entity Types in UI Components

**Mistake**: `import { User } from '@core/user'` in a UI component.

**Why it's wrong**: Creates a dependency from `ui/` to `core/`, violating the dependency graph. Also couples the UI to a specific entity shape that may change.

**Fix**:
- Use primitive inputs: `name: string`, `imageUrl: string`
- Or define a local interface in the UI component: `AvatarData` instead of `User`

---

## 8. Feature-to-Feature Communication

**Mistake**: Feature A importing a service, component, or utility from Feature B.

**Why it's wrong**: Breaks the fundamental isolation guarantee. Changes in Feature B can now break Feature A.

**Fix**: Apply "extract one level up":
- Shared service → `core/<domain>/`
- Shared UI component → `ui/`
- Shared business component → `pattern/`

---

## 9. Barrel Exports Within Features

**Mistake**: Creating `index.ts` barrel files within feature folders.

**Why it's wrong**: Barrel exports can create circular dependencies within the feature. They also make the bundler's job harder and can pull in more code than needed.

**Fix**: Use direct file imports within features. Reserve barrel exports for `core/`, `ui/`, and `pattern/` public APIs.

---

## 10. Wrapping Angular APIs

**Mistake**: Creating custom abstractions around Angular or state management libraries — `BaseComponent`, `*myOrgFor`, custom DI wrappers.

**Why it's wrong**:
- Breaks automatic migrations (`ng update`)
- Diverges from framework direction
- Adds maintenance burden that scales with number of projects
- The framework's own verbosity decreases over time (e.g. `takeUntilDestroyed`, control flow, signal APIs)

**Fix**: Stay at the framework's abstraction level. Use schematics for code generation.

---

## 11. Pattern-to-Pattern Dependencies

**Mistake**: One pattern importing from another pattern.

**Why it's wrong**: Creates coupling between patterns that should be independently deployable.

**Fix**: Extract shared parts:
- Shared services → `core/`
- Shared UI → `ui/`

---

## 12. Complex Pipes in UI

**Mistake**: Implementing complex pipes with business logic in `ui/`.

**Why it's wrong**: If the pipe needs services or business logic, it violates UI's "no app services" rule.

**Fix**:
- Separate the pipe into a thin wrapper (template-side) + service (logic-side) → make it a `pattern/`
- Or use the pipe in the parent (feature/layout) and pass the result to UI via input

---

## 13. `@defer` on Pattern Components Without Thought

**Mistake**: Using `@defer` on a pattern component without considering that the parent feature controls the trigger.

**Why it's wrong**: Patterns are consumed via template tags. When you `@defer` a pattern, the defer trigger (viewport, interaction, idle, timer) must be set in the **consuming feature's template**, not inside the pattern itself. If the pattern is lightweight, deferring it adds unnecessary complexity and a request waterfall.

**Fix**: Only defer patterns that are genuinely heavy (large bundle impact). Set the `@defer` trigger in the feature template that consumes the pattern:

```html
<!-- feature/orders/order-detail.component.html -->
@defer (on viewport) {
  <app-document-manager [context]="orderContext" />
} @placeholder {
  <app-skeleton height="200px" />
}
```

---

## 14. `resource()` / `httpResource()` in UI Components

**Mistake**: Using `resource()` or `httpResource()` in a `ui/` component.

**Why it's wrong**: These APIs make HTTP calls or async operations, which means the component is fetching data directly — that's business logic, not presentation. This makes it a pattern, not UI.

**Fix**: `resource()` and `httpResource()` belong in:
- **Feature services** (route-scoped) for feature-specific data
- **Core services** (`providedIn: 'root'`) for shared data
- **Pattern components** when the data fetching is part of the reusable business component

UI components receive data via inputs. Always.

---

## 15. DOM Access Without SSR Guards

**Mistake**: Accessing `window`, `document`, `localStorage`, or DOM APIs directly in services or component constructors.

**Why it's wrong**: With Angular 21+ SSR and hydration, services in `core/` run on the server during prerendering. `window` and `document` don't exist on the server — this crashes the app.

**Fix**: Use `afterNextRender()` or `afterRender()` for DOM-dependent code. For platform checks in services, use `inject(PLATFORM_ID)` with `isPlatformBrowser()`:

```typescript
// core/analytics/analytics.service.ts
@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  private readonly platformId = inject(PLATFORM_ID);

  trackEvent(name: string) {
    if (isPlatformBrowser(this.platformId)) {
      window.analytics?.track(name);
    }
  }
}
```

For components, prefer `afterNextRender()`:
```typescript
export class ChartComponent {
  constructor() {
    afterNextRender(() => {
      // Safe to access DOM here
      this.initChart();
    });
  }
}
```

Also ensure `provideClientHydration(withEventReplay())` is in `provideCore()` for SSR apps.
