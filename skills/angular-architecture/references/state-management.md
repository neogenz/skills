# State Management — Signal Store Pattern

Angular 19+ idiomatic state management using `signal()`, `computed()`, `resource()`, and `async/await`. No external state library. No BehaviorSubject. No boilerplate.

---

## Principles

| Concern | Tool | Why |
|---------|------|-----|
| State | `signal()` | Writable, reactive, synchronous |
| Derived state | `computed()` | Lazy, cached, auto-tracked |
| Async data loading | `resource()` | Declarative, lifecycle-aware, signal-driven |
| Mutations | `async/await` | Imperative, explicit error handling |
| Cross-store invalidation | Version signal in `resource` params | Triggers reload without coupling stores |

---

## Architecture Layers

```
Component  ──▶  Store  ──▶  ApiClient
   │              │              │
   │  inject()    │  inject()    │  HttpClient
   │  read state  │  owns state  │  HTTP calls
   │  call mutations             │  returns Promise<T>
```

- **Component**: reads signals, calls store mutations. No direct HTTP.
- **Store**: `@Injectable()` service owning `signal()`, `computed()`, `resource()`. Single source of truth for its domain.
- **ApiClient**: `@Injectable()` service wrapping `HttpClient`. Returns `Promise<T>` (use `firstValueFrom`). Validates responses with Zod.

---

## Store Anatomy

A complete feature store with all 6 sections:

```typescript
// feature/orders/order.store.ts

@Injectable()
export class OrderStore {
  // ── 1. Dependencies ──
  private readonly orderApi = inject(OrderApi);

  // ── 2. State ──
  private readonly _selectedId = signal<string | null>(null);

  // ── 3. Resource (async data loading) ──
  private readonly ordersResource = resource({
    params: () => ({ version: this.invalidation.version() }),
    loader: () => this.orderApi.getAll(),
  });

  private readonly orderDetailResource = resource({
    params: () => {
      const id = this._selectedId();
      return id ? { id, version: this.invalidation.version() } : undefined;
    },
    loader: ({ params }) => this.orderApi.getById(params.id),
  });

  // ── 4. Selectors (readonly, derived) ──
  readonly orders = this.ordersResource.value;
  readonly selectedOrder = this.orderDetailResource.value;
  readonly isLoading = computed(() => this.ordersResource.isLoading());
  readonly isInitialLoading = computed(
    () => this.ordersResource.isLoading() && !this.ordersResource.value(),
  );
  readonly totalCount = computed(() => this.orders()?.length ?? 0);

  // ── 5. Mutations (async/await) ──
  async create(dto: CreateOrderDto): Promise<void> {
    await this.orderApi.create(dto);
    this.invalidation.invalidate();
  }

  async delete(id: string): Promise<void> {
    await this.orderApi.delete(id);
    this.invalidation.invalidate();
  }

  // ── 6. Utils ──
  selectOrder(id: string | null): void {
    this._selectedId.set(id);
  }

  // ── Invalidation ──
  private readonly invalidation = inject(InvalidationService);
}
```

### Rules

| Rule | Applies to | Rationale |
|------|-----------|-----------|
| All public signals are `readonly` | Selectors | Consumers cannot mutate store state |
| Derived state uses `computed()` | Selectors | Cached, lazy, auto-tracked |
| No `effect()` in stores | Mutations | `effect()` hides causality — use explicit methods |
| No `.subscribe()` | Anywhere | Signals are pull-based — subscribe is push-based legacy |
| Mutations return `Promise<void>` | Mutations | Caller handles errors, no hidden side effects |
| `resource()` for all data fetching | Resources | Declarative, lifecycle-aware, auto-cleanup |

### Scoping via Route Providers

Feature stores are scoped to their feature's lazy injector:

```typescript
// feature/orders/orders.routes.ts
export default [
  {
    path: '',
    providers: [OrderStore, OrderApi],
    children: [
      { path: '', loadComponent: () => import('./order-list') },
      { path: ':id', loadComponent: () => import('./order-detail') },
    ],
  },
] satisfies Routes;
```

Global stores (auth, user, preferences) use `providedIn: 'root'` and live in `core/`.

---

## Loading States

Two distinct loading states for good UX:

```typescript
// Initial load — show skeleton
readonly isInitialLoading = computed(
  () => this.ordersResource.isLoading() && !this.ordersResource.value(),
);

// Subsequent reload — show subtle spinner, keep showing stale data
readonly isLoading = computed(() => this.ordersResource.isLoading());
```

Usage in template:

```html
@if (store.isInitialLoading()) {
  <app-skeleton />
} @else {
  @if (store.isLoading()) {
    <app-inline-spinner />
  }
  @for (order of store.orders(); track order.id) {
    <app-order-card [order]="order" />
  }
}
```

---

## Cross-Store Invalidation

When a mutation in one store should reload data in another, use a shared version signal:

```typescript
// core/invalidation/invalidation.service.ts
@Injectable({ providedIn: 'root' })
export class InvalidationService {
  private readonly _version = signal(0);
  readonly version = this._version.asReadonly();

  invalidate(): void {
    this._version.update(v => v + 1);
  }
}
```

Any `resource()` that includes `version` in its params auto-reloads when `invalidate()` is called:

```typescript
private readonly ordersResource = resource({
  params: () => ({ version: this.invalidation.version() }),
  loader: () => this.orderApi.getAll(),
});
```

For domain-scoped invalidation (e.g., only order-related stores), create domain-specific invalidation services instead of a single global one.

---

## State Placement

Where state lives depends on its scope. Same rule as services — isolation beats DRY.

| State type | Scope | Location | Registration |
|------------|-------|----------|-------------|
| Auth, user, global config | App-wide | `core/` | `providedIn: 'root'` |
| Feature entity state | One feature | `feature/` | Route `providers: []` |
| Shared entity state (2+ features) | App-wide | `core/<domain>/` | `providedIn: 'root'` |
| Complex form/editor state | Per component | Component | Component `providers: []` |
| UI filter/sort state | Per component | Component | Inline signals or component `providers: []` |
| Theme, locale, preferences | App-wide | `core/` | `providedIn: 'root'` |

---

## Anti-patterns

| Don't | Do | Why |
|-------|-----|-----|
| `effect()` to sync state | `computed()` for derivation, explicit methods for side effects | `effect()` hides causality and creates debug nightmares |
| `.subscribe()` on observables in stores | `resource()` or `async/await` with `firstValueFrom` | Signals are pull-based — don't mix paradigms |
| Feature store with `providedIn: 'root'` | `@Injectable()` + route providers | Breaks feature isolation, leaks to initial bundle |
| Global store in a `feature/` folder | Move to `core/<domain>/` | If it's `providedIn: 'root'`, it belongs in `core/` |
| Injecting feature store from another feature | Extract to `core/` | Cross-feature access = `NullInjectorError` at runtime |
| Mutable public signals | `readonly` + `.asReadonly()` | Store owns its state — consumers read only |
| `BehaviorSubject` for state | `signal()` | Signals are simpler, synchronous, and framework-native |

---

## Per-Store Checklist

When creating a new store, verify:

- [ ] `@Injectable()` (no `providedIn` for feature stores)
- [ ] Registered in route `providers: []` (feature) or `providedIn: 'root'` (core)
- [ ] All public signals are `readonly`
- [ ] Derived state uses `computed()`, not manual sync
- [ ] Data loading uses `resource()` with params function
- [ ] Mutations use `async/await`, return `Promise<void>`
- [ ] No `effect()` — use explicit methods instead
- [ ] No `.subscribe()` — use `resource()` or `firstValueFrom`
- [ ] Invalidation wired via `InvalidationService.version()` in resource params
- [ ] `isInitialLoading` and `isLoading` both exposed for UX
