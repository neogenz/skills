# Service Scoping Reference

Angular's injector hierarchy provides three scoping levels for services. Choosing the right scope is an architectural decision with direct impact on bundling, isolation, and testability.

---

## The Three Scopes

### 1. Root Scope — `providedIn: 'root'`

**Where**: `core/` services only
**Injector**: Root `EnvironmentInjector` (eager)
**Lifetime**: Application-wide singleton
**Bundle**: Included in initial `main.js` bundle

```typescript
// core/auth/auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly http = inject(HttpClient);
  readonly isAuthenticated = signal(false);
}
```

**When to use**:
- Authentication state, current user
- HTTP interceptors, error handlers
- Guards used across multiple features
- Services shared by 2+ lazy features
- State management (global signal stores)

**When NOT to use**:
- Feature-specific services → use Route Scope
- Component-specific state → use Component Scope

---

### 2. Route Scope — `providers: []` in route config

**Where**: `feature/` services (primary), sometimes `pattern/`
**Injector**: Lazy `EnvironmentInjector` (created per lazy route)
**Lifetime**: Lives as long as the route is active
**Bundle**: Included in the feature's lazy bundle (not initial bundle)

```typescript
// feature/orders/order.store.ts
@Injectable() // ← No providedIn!
export class OrderStore {
  readonly orders = signal<Order[]>([]);
  readonly loading = signal(false);
}

// feature/orders/order.api.ts
@Injectable()
export class OrderApi {
  private readonly http = inject(HttpClient);

  getOrders() {
    return this.http.get<Order[]>('/api/orders');
  }
}

// feature/orders/orders.routes.ts
export default [
  {
    path: '',
    providers: [OrderStore, OrderApi],  // ← Route-scoped
    children: [
      { path: '', loadComponent: () => import('./order-list') },
      { path: ':id', loadComponent: () => import('./order-detail') },
    ],
  },
] satisfies Routes;
```

**Key properties**:
- Service is NOT in the initial bundle (lazy loaded)
- Service is scoped to this feature's component tree
- Injecting from a sibling feature throws `NullInjectorError` — free isolation enforcement
- New instance created each time the route is activated
- Sub-routes inherit the parent's providers

**When to use**:
- Feature-specific API services
- Feature-specific stores / state
- Feature-specific form handlers
- Any service used only within one feature

---

### 3. Component Scope — `providers: []` on a component

**Where**: Rare — specific use cases only
**Injector**: `ElementInjector` (per component instance)
**Lifetime**: Lives as long as the component instance
**Bundle**: Same bundle as the component

```typescript
@Component({
  selector: 'app-order-editor',
  providers: [OrderEditorState],  // ← One instance per component
  template: `...`,
})
export class OrderEditorComponent {
  readonly state = inject(OrderEditorState);
}
```

**Key properties**:
- Each component instance gets its own service instance
- Service is destroyed with the component
- Child components in the template can also inject it

**When to use**:
- Component-specific state (e.g., form state for an editor)
- Per-instance signal store
- State that must be unique per component instance (e.g., multiple editors on the same page)

---

## Decision Tree

```
Is the service used by 2+ features?
├── YES → providedIn: 'root' in core/
└── NO → Is it used by only one feature?
          ├── YES → @Injectable() + route providers in feature/
          └── NO → Is it per-component instance?
                    ├── YES → component providers: []
                    └── NO → Probably route-scoped in feature/
```

---

## Anti-patterns

### Feature service with `providedIn: 'root'`

```typescript
// ❌ WRONG — in feature/orders/order.store.ts
@Injectable({ providedIn: 'root' })  // Leaks to root!
export class OrderStore {}
```

**Problems**:
- Service is in the initial bundle (increases startup time)
- Available globally — any feature can accidentally depend on it
- Breaks feature isolation guarantee

### Root service without `providedIn: 'root'`

```typescript
// ❌ WRONG — in core/auth/auth.service.ts
@Injectable()  // Not provided anywhere!
export class AuthService {}
```

**Problem**: Service won't be available unless manually added to `provideCore()`.
While this works, `providedIn: 'root'` is preferred for `core/` services because it enables tree-shaking.

---

## Provider Inheritance in Route Hierarchy

Route-scoped providers cascade down to child routes:

```typescript
export default [
  {
    path: '',
    providers: [OrderStore, OrderApi],  // Available to ALL children
    children: [
      {
        path: 'list',
        loadComponent: () => import('./order-list'),
        // ← Can inject OrderStore and OrderApi
      },
      {
        path: ':id',
        loadChildren: () => import('./detail/detail.routes'),
        // ← Sub-feature can ALSO inject OrderStore and OrderApi
      },
    ],
  },
] satisfies Routes;
```

This enables sharing services between a feature and its lazy sub-features without extracting to `core/`.

---

## Migrating Feature Services to Core

When a second feature needs a service:

1. Move the service file from `feature/orders/` to `core/order/`
2. Add `providedIn: 'root'` to the `@Injectable()` decorator
3. Remove from the feature's route `providers: []`
4. Update all imports to use `@core/order/` path alias
5. Both features can now inject it

```typescript
// BEFORE: feature/orders/order.store.ts
@Injectable()
export class OrderStore {}

// AFTER: core/order/order.store.ts
@Injectable({ providedIn: 'root' })
export class OrderStore {}
```
