# State Management Architecture

Where state lives in the architecture depends on its scope. The same principle applies: isolation beats DRY. Feature state stays in features. Shared state goes to core.

---

## State Placement Decision Tree

```
Is this state needed by 2+ features?
├── YES → core/ with providedIn: 'root'
│         (NgRx provideState/provideEffects in provideCore(),
│          or signalStore with providedIn: 'root')
└── NO → Is it scoped to one feature?
          ├── YES → Feature route providers: []
          │         (NgRx provideState/provideEffects in route config,
          │          or signalStore as @Injectable() in route providers)
          └── NO → Is it per-component instance?
                    ├── YES → Component providers: []
                    │         (ComponentStore, or signalStore on component)
                    └── NO → Probably feature route-scoped
```

---

## NgRx Store

### Global State (core/)

State shared across multiple features or needed at app startup:

```typescript
// core/auth/auth.state.ts
export const authFeature = createFeature({
  name: 'auth',
  reducer: createReducer(
    initialState,
    on(AuthActions.loginSuccess, (state, { user }) => ({ ...state, user })),
  ),
});

// core/core.ts
export function provideCore({ routes }: CoreOptions) {
  return [
    provideRouter(routes),
    provideHttpClient(),

    // Global NgRx state
    provideStore(),
    provideState(authFeature),
    provideState(userFeature),
    provideEffects(AuthEffects, UserEffects),

    provideAppInitializer(() => {
      const store = inject(Store);
      store.dispatch(AppActions.init());
    }),
  ];
}
```

### Feature State (feature/ route providers)

State used only within one feature — scoped to the lazy injector:

```typescript
// feature/orders/order.state.ts
export const orderFeature = createFeature({
  name: 'orders',
  reducer: createReducer(initialState, /* ... */),
});

// feature/orders/orders.routes.ts
export default [
  {
    path: '',
    providers: [
      provideState(orderFeature),
      provideEffects(OrderEffects),
    ],
    children: [
      { path: '', loadComponent: () => import('./order-list') },
      { path: ':id', loadComponent: () => import('./order-detail') },
    ],
  },
] satisfies Routes;
```

When a second feature needs the same state → extract to `core/`.

---

## Signal Store (@ngrx/signals)

### Global Signal Store (core/)

```typescript
// core/user/user.store.ts
export const UserStore = signalStore(
  { providedIn: 'root' },
  withState<UserState>({ user: null, loading: false }),
  withMethods((store, userApi = inject(UserApi)) => ({
    async loadUser() {
      patchState(store, { loading: true });
      const user = await firstValueFrom(userApi.getUser());
      patchState(store, { user, loading: false });
    },
  })),
);
```

### Feature Signal Store (route-scoped)

```typescript
// feature/orders/order.store.ts
export const OrderStore = signalStore(
  // No providedIn — scoped to feature
  withState<OrderState>({ orders: [], loading: false }),
  withMethods((store, orderApi = inject(OrderApi)) => ({
    async loadOrders() {
      patchState(store, { loading: true });
      const orders = await firstValueFrom(orderApi.getAll());
      patchState(store, { orders, loading: false });
    },
  })),
);

// feature/orders/orders.routes.ts
export default [
  {
    path: '',
    providers: [OrderStore, OrderApi],
    children: [/* ... */],
  },
] satisfies Routes;
```

### Component Signal Store (per-instance)

```typescript
// Used when each component instance needs its own state
@Component({
  selector: 'app-order-editor',
  providers: [OrderEditorStore],
  template: `...`,
})
export class OrderEditorComponent {
  readonly store = inject(OrderEditorStore);
}
```

---

## ComponentStore (@ngrx/component-store)

Always component-scoped — provided in the component's `providers: []`:

```typescript
@Component({
  selector: 'app-product-filter',
  providers: [ProductFilterStore],
  template: `...`,
})
export class ProductFilterComponent {
  readonly store = inject(ProductFilterStore);
}
```

ComponentStore is destroyed with the component. Use it for complex local state that doesn't need to persist across navigation.

---

## Lightweight Signals (No Library)

For simple state that doesn't need NgRx infrastructure:

```typescript
// core/theme/theme.service.ts — global
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private readonly _theme = signal<'light' | 'dark'>('light');
  readonly theme = this._theme.asReadonly();

  toggle() {
    this._theme.update(t => t === 'light' ? 'dark' : 'light');
  }
}

// feature/orders/order-filter.service.ts — feature-scoped
@Injectable()
export class OrderFilterService {
  readonly searchQuery = signal('');
  readonly statusFilter = signal<OrderStatus | null>(null);

  readonly activeFilters = computed(() => {
    const filters: string[] = [];
    if (this.searchQuery()) filters.push(`search: ${this.searchQuery()}`);
    if (this.statusFilter()) filters.push(`status: ${this.statusFilter()}`);
    return filters;
  });
}
```

---

## State Placement Summary

| State type | Scope | Location | Registration |
|------------|-------|----------|-------------|
| Auth, user, global config | App-wide | `core/` | `provideCore()` or `providedIn: 'root'` |
| Feature entity state | One feature | `feature/` | Route `providers: []` |
| Shared entity state (2+ features) | App-wide | `core/<domain>/` | `provideCore()` or `providedIn: 'root'` |
| Complex form/editor state | Per component | Component | Component `providers: []` |
| UI filter/sort state | Per component | Component | Component `providers: []` or inline signals |
| Theme, locale, preferences | App-wide | `core/` | `providedIn: 'root'` |

---

## Anti-patterns

**Feature state with `providedIn: 'root'`**:
```typescript
// ❌ WRONG — feature state leaks to root
export const OrderStore = signalStore(
  { providedIn: 'root' },  // Breaks isolation!
  withState({ orders: [] }),
);
```

**Global state in a feature folder**:
```typescript
// ❌ WRONG — shared state living in a feature
// feature/orders/shared-order.store.ts with providedIn: 'root'
// If it's providedIn: 'root', it belongs in core/
```

**Injecting feature store from another feature**:
```typescript
// ❌ WRONG — cross-feature state access
// feature/dashboard/dashboard.component.ts
readonly orderStore = inject(OrderStore);
// OrderStore is provided in feature/orders route — NullInjectorError!
// Fix: extract to core/order/
```
