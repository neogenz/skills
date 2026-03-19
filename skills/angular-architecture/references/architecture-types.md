# Architecture Types Reference

## The 5 Architectural Layers

Every Angular application following this architecture is divided into 5 distinct layer types, each with strict rules about what it can contain and what it can import. An optional 6th layer (`styles/`) serves as the design system foundation.

---

### Core (Eager, Headless)

- **Location**: `core/`
- **Bundling**: Eager — part of initial `main.js` bundle, available from start
- **Contains**: ONLY headless (injector-based) building blocks — services, guards, interceptors, functions, state management. **NO components, directives, or pipes.**
- **Can import from**: `styles/` only (nothing else)
- **Use for**:
  - Logic needed at app start (auth state, current user, guards, interceptors)
  - Logic shared by 2+ features (extract here when a second feature needs it)
- **Organization**: Domain-based folders (`core/auth/`, `core/user/`, `core/order/`). **NEVER** by building block type (`core/services/`, `core/guards/`)
- **Setup**: `provideCore()` function in `core.ts` — the sole entry point for all eager providers. `app.config.ts` should only call `provideCore()`

#### provideCore() Pattern

```typescript
export function provideCore({ routes }: CoreOptions) {
  return [
    provideAnimationsAsync(),
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([authInterceptor])),
    // state management, 3rd party providers...

    // Initialization — kick-start processes at startup
    provideAppInitializer(() => {
      const authService = inject(AuthService);
      // e.g. load auth token, dispatch AppInit action
    }),
  ];
}
```

#### NgModule Migration

For 3rd party libs that only export NgModules, use `importProvidersFrom()`:

```typescript
importProvidersFrom(SomeChartsModule)
```

---

### Layout (Eager, Template-based)

- **Location**: `layout/`
- **Bundling**: Eager — part of initial bundle, displayed before any lazy feature loads
- **Contains**: ONLY standalones (components, directives, pipes). **NO services here** — inject services from `core/`
- **Can import from**: `core/`, `ui/`, `pattern/`, `styles/`
- **Use for**: Application shell, navigation, headers, sidebars, footers

#### Layout Variants

**Single layout** (whole app):
```typescript
@Component({
  selector: 'app-root',
  template: `<app-main-layout />`
})
export class App {}
```

**Multiple layouts** (e.g. auth vs main):
```typescript
export const routes: Routes = [
  {
    path: '',
    component: AuthLayout,
    children: [
      { path: 'login', loadChildren: () => import('./feature/login/login.routes') }
    ]
  },
  {
    path: 'app',
    component: MainLayout,
    children: [
      { path: 'home', loadChildren: () => import('./feature/home/home.routes') }
    ]
  }
];
```

**Custom layout per feature**: `layout/` folder stays empty, each feature defines its own layout internally. `AppComponent` contains only `<router-outlet />`.

---

### UI (Eager/Lazy, Pure)

- **Location**: `ui/`
- **Bundling**: Mixed — bundler automatically optimizes based on usage (shared chunk if used by multiple lazy features, inlined if used by only one)
- **Contains**: ONLY standalones (components, directives, pipes)
- **Can import from**: Other `ui/` components (internal composition). **NOTHING ELSE.** Self-contained from all app code.
- **Use for**: Generic reusable widgets (buttons, cards, dialogs, inputs, badges, avatars, spinners)

#### CRITICAL Constraints

- **NEVER inject app/business services** (`UserService`, `AuthStateService`, etc.)
- **Angular/Material framework services ARE allowed**: `MatDialogRef`, `ElementRef`, `DestroyRef`, `Renderer2`, etc.
- **Data flows via inputs/outputs ONLY** — parent provides data, UI presents it
- **No domain knowledge** — no "User", "Order", "Invoice" concepts. Generic only.
- **Stateless** — parent manages state

#### Handling Types in UI

UI components should NOT import entity types from `core/`. Instead:

1. **Best**: Use primitive inputs (`imageUrl: string`, `name: string`, `role: string`)
2. **Good**: Define local interfaces in the UI component itself (`Avatar` instead of `User`)
3. **Acceptable**: Define a `model` type that all layers can import from

#### Bundling Behavior

| Usage | Bundle placement |
|-------|-----------------|
| Only in layout (eager) | `main.js` — consider moving to `layout/` |
| Layout + lazy features | `main.js` (eager wins) |
| Multiple lazy features | Shared chunk, loaded on first navigation |
| Single lazy feature | Inlined in that feature's bundle — consider moving to `feature/` |

---

### Pattern (Lazy, Reusable Business Use Cases)

- **Location**: `pattern/<pattern-name>/`
- **Bundling**: Usually lazy (can be deferred with `@defer` in consumer)
- **Contains**: Combination of standalones + injectables — a "pre-packaged" business feature
- **Can import from**: `core/`, `ui/`, `styles/`
- **Use for**: Cross-cutting business features consumed as "drop-in" components (NOT via routes)

#### What Makes a Pattern

A pattern is essentially a **non-routed feature** — it has both UI and business logic but is consumed via a component tag in templates, not via the router:

```html
<!-- In a feature template -->
<app-document-manager [context]="orderContext" />
<app-approval-process [entityId]="orderId()" />
<app-change-history [entityId]="orderId()" />
```

#### Pattern vs Other Types

| Question | UI | Pattern | Feature |
|----------|------|---------|---------|
| Can inject app services? | NO | YES | YES |
| Has domain knowledge? | NO | YES | YES |
| How is it consumed? | `imports: []` | Template tag | Router |
| Can import from core? | NO | YES | YES |
| Reusable across features? | YES (any app) | YES (this app) | NO (isolated) |

#### When to Create a Pattern

Two features need the same thing. Ask:
1. **Different behavior AND UI** → Keep isolated (duplication OK)
2. **Same behavior, different UI** → Extract behavior to `core/`
3. **Different behavior, same UI** → Extract UI to `ui/`
4. **Same behavior AND UI** → Create `pattern/`

#### FORBIDDEN

- Pattern importing from `feature/` (circular dependency)
- Pattern importing from another `pattern/` (extract shared parts to `core/` or `ui/`)
- Pattern importing from `layout/`

---

### Feature (Lazy, Isolated Black Boxes)

- **Location**: `feature/<feature-name>/`
- **Bundling**: Lazy — separate bundle per feature, loaded on demand via router
- **Contains**: Any building blocks (components, services, routes, sub-features)
- **Can import from**: `core/`, `ui/`, `pattern/`, `styles/`
- **CRITICAL**: Features are **completely isolated** — a feature CANNOT import from another feature. EVER.

#### Feature Properties

- **Black box**: Internal implementation quality doesn't leak. Dirty code stays contained.
- **Throw-away**: Can be replaced entirely without affecting the rest of the app.
- **Delivery-optimized**: Isolation means you can ship fast without fear of breaking other features.

#### Feature Services: Route-Level Providers

```typescript
// WRONG — leaks to root injector, breaks isolation
@Injectable({ providedIn: 'root' })
export class MyFeatureStore {}

// CORRECT — scoped to feature's lazy injector
@Injectable()
export class MyFeatureStore {}

// feature.routes.ts
export default [
  {
    path: '',
    providers: [MyFeatureApi, MyFeatureStore],
    children: [
      { path: '', loadComponent: () => import('./list-page') },
      { path: ':id', loadComponent: () => import('./detail-page') },
    ],
  },
] satisfies Routes;
```

#### Always `loadChildren`, Never `loadComponent` for Features

```typescript
// WRONG
{ path: 'orders', loadComponent: () => import('./feature/orders/orders') }

// CORRECT
{ path: 'orders', loadChildren: () => import('./feature/orders/orders.routes') }
```

`loadChildren` ensures uniform API and extensibility — you can always add child routes later.

#### Nested Sub-Features

Large features can have lazy sub-features (fractal architecture):

```typescript
// feature/order/order.routes.ts
export default [
  {
    path: '',
    component: OrderComponent,
    children: [
      { path: 'dashboard', loadChildren: () => import('./dashboard/dashboard.routes') },
      { path: 'definitions', loadChildren: () => import('./definitions/definitions.routes') },
    ],
  },
] satisfies Routes;
```

#### No Eager Features

Even if the app has only ONE feature, make it lazy. Consistency from day one, extensibility for free. Adding a second feature later won't require restructuring.

---

### Styles (Optional Foundation Layer)

- **Location**: `styles/`
- **Bundling**: Eager — design tokens and CSS foundation
- **Contains**: Design tokens, SCSS entry points, Tailwind utilities, theme configuration
- **Can import from**: Nothing (foundation layer)
- **Can be imported by**: `core/`, `layout/`, `pattern/`, `feature/`
- **Cannot be imported by**: `ui/` (UI is self-styled)

Only use this layer if your project centralizes design tokens, theming, or shared CSS utilities.
