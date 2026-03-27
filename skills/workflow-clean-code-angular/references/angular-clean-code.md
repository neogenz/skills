# Angular 21 Clean Code Patterns

> Reference patterns for step-02. These are the **correct** modern implementations to apply when fixing anti-patterns.

---

## 1. Component Pattern (Modern Angular 21)

```typescript
@Component({
  selector: 'app-budget-card',
  imports: [CurrencyPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  host: {
    'class': 'block',
    '[class.active]': 'isActive()',
    '(click)': 'onClick()',
  },
  template: `
    @if (budget(); as budget) {
      <h3>{{ budget.name }}</h3>
      <p>{{ budget.amount | currency }}</p>
    } @else {
      <p>Aucun budget</p>
    }
  `,
})
export class BudgetCardComponent {
  // Signal inputs (replace @Input)
  readonly budget = input.required<Budget>();
  readonly isActive = input(false);

  // Signal outputs (replace @Output)
  readonly selected = output<Budget>();

  // Derived state (replace effect)
  readonly formattedAmount = computed(() =>
    this.budget().amount.toLocaleString('fr-FR', { style: 'currency', currency: 'EUR' })
  );

  onClick(): void {
    this.selected.emit(this.budget());
  }
}
```

**Key rules:**
- Standalone is the default since Angular 19 — do NOT specify `standalone: true` explicitly, it's noise
- Always `OnPush`
- Use `host: {}` not `@HostBinding`/`@HostListener`
- `input()` / `input.required()` not `@Input()`
- `output()` not `@Output()`
- `computed()` for derived state
- Inline template for small components (< 30 lines)
- **Zoneless by default** — Angular 21 uses `provideZonelessChangeDetection()`. Never use `NgZone`, `zone.run()`, or `zone.runOutsideAngular()`
- Use `hostDirectives` for reusable cross-cutting behavior (tooltips, focus trap, drag-drop)
- Use CSS transitions/animations, not `@angular/animations` DSL (unless complex orchestrated sequences)

---

## 2. Signal State Pattern

The store follows a strict **6-section anatomy**: Dependencies → State → Resource → Selectors → Mutations → Private utils.

```typescript
@Injectable({ providedIn: 'root' })
export class ProductStore {
  // ── 1. Dependencies ──────────────────────────────────────────────────────
  readonly #api = inject(ProductService);

  // ── 2. State ─────────────────────────────────────────────────────────────
  readonly #state = signal<ProductState>({
    selectedId: null,
    filter: '',
    error: null,
  });

  // ── 3. Resource ───────────────────────────────────────────────────────────
  readonly #productsResource = httpResource(
    () => `/api/products?filter=${this.#state().filter}`,
    { parse: productListSchema.parse },
  );

  // ── 4. Selectors (public computed) ────────────────────────────────────────
  readonly products = computed(() => this.#productsResource.value() ?? []);
  readonly isLoading = computed(() => this.#productsResource.isLoading());
  readonly error = computed(
    () => this.#state().error ?? this.#productsResource.error() ?? null,
  );
  readonly selectedProduct = computed(() =>
    this.products().find(p => p.id === this.#state().selectedId),
  );

  // ── 5. Mutations (public methods) ─────────────────────────────────────────
  select(id: string): void {
    this.#state.update(s => ({ ...s, selectedId: id }));
  }

  setFilter(filter: string): void {
    this.#state.update(s => ({ ...s, filter }));
  }

  async addProduct(product: NewProduct): Promise<void> {
    try {
      const created = await this.#api.create(product);
      this.#productsResource.reload();
      this.select(created.id);
    } catch (error: unknown) {
      this.#state.update(s => ({ ...s, error: this.#toMessage(error) }));
    }
  }

  // ── 6. Private utils ──────────────────────────────────────────────────────
  #toMessage(error: unknown): string {
    return isApiError(error) ? error.message : 'Unexpected error';
  }
}
```

**Why 6 sections?** Consistent structure across all stores means any developer can open any store file and immediately find what they need. Dependencies at the top show what the store depends on. State defines the shape. Resource handles data fetching. Selectors expose read-only derived views. Mutations change state. Private utils stay at the bottom, out of the way.

If a section is empty (e.g., no Resource because data comes from another store), skip it — but keep the order for the sections you do have.

**Key rules:**
- Sections always appear in order: Dependencies → State → Resource → Selectors → Mutations → Private utils
- One `#state` signal per store — shape it to what mutations need, not what templates want
- `httpResource()` in the Resource section for data fetching; call `.reload()` after mutations
- All public state via `computed()` in the Selectors section — never expose `#state` directly
- Expose read-only state with `computed()` OR `.asReadonly()` — both are valid
- Immutable updates with spread operator
- `linkedSignal()` for dependent writable state (e.g., a filter that resets when a category changes)
- `effect()` only for side-effects (localStorage, analytics) — never to sync signals
- Private utils (`#` prefix) belong in section 6, not scattered inline

---

## 3. Dependency Injection Pattern

```typescript
@Injectable({ providedIn: 'root' })
export class BudgetService {
  readonly #http = inject(HttpClient);
  readonly #config = inject(ConfigService);

  readonly budgets = httpResource(() => ({
    url: `${this.#config.apiUrl()}/budgets`,
    method: 'GET',
  }), {
    parse: budgetListSchema.parse,
  });

  // Classic approach (also valid):
  // readonly #http = inject(HttpClient);
  // getProducts() { return this.#http.get<Product[]>('/api/products'); }
}
```

### Resource API Details

The `resource()` and `httpResource()` APIs provide signal-based async data fetching:

```typescript
// resource() — generic async loader
readonly #userResource = resource({
  params: () => ({ id: this.userId() }),
  loader: async ({ params, abortSignal }) => {
    const res = await fetch(`/api/users/${params.id}`, { signal: abortSignal });
    if (!res.ok) throw new Error('Failed to load');
    return res.json();
  },
});

// httpResource() — Angular HttpClient wrapper (preferred)
readonly #productsResource = httpResource<Product[]>(() => '/api/products');
```

**Resource status signals — use them in templates:**
```html
@if (#productsResource.isLoading()) {
  <app-spinner />
} @else if (#productsResource.error()) {
  <app-error [message]="#productsResource.error()?.message" />
} @else if (#productsResource.hasValue()) {
  @for (product of #productsResource.value(); track product.id) {
    <app-product-card [product]="product" />
  }
}
```

**Key resource signals:**
- `value()` — resolved data or `undefined`
- `hasValue()` — type-guard boolean
- `isLoading()` — loader is running
- `error()` — error thrown by loader
- `status()` — `'idle'` | `'loading'` | `'resolved'` | `'error'` | `'reloading'` | `'local'`
- `.reload()` — force re-fetch
- `.value.set(newValue)` — optimistic local update (status becomes `'local'`)

**Always pass `abortSignal`** to `fetch()` in `resource()` loaders — if params change mid-flight, Angular aborts the previous request.

**Key rules:**
- `inject()` not constructor injection
- `#` prefix for private fields (native private)
- `readonly` for injected services
- `httpResource()` for simple GET requests (Angular 19+), or HttpClient for complex operations
- Zod `parse` for response validation

---

## 4. Control Flow Pattern

```html
<!-- @if replaces *ngIf -->
@if (isLoading()) {
  <app-spinner />
} @else if (error()) {
  <app-error [message]="error()!.message" />
} @else {
  <!-- @for replaces *ngFor — track is REQUIRED -->
  @for (budget of budgets(); track budget.id) {
    <app-budget-card
      [budget]="budget"
      [isActive]="budget.id === selectedId()"
      (selected)="onSelect($event)"
    />
  } @empty {
    <p>Aucun budget trouvé</p>
  }
}

<!-- @switch replaces *ngSwitch -->
@switch (status()) {
  @case ('income') {
    <span class="text-green-600">Revenu</span>
  }
  @case ('expense') {
    <span class="text-red-600">Dépense</span>
  }
  @case ('saving') {
    <span class="text-blue-600">Épargne</span>
  }
}
```

### @defer — Lazy Loading in Templates

Wrap heavy, below-fold, or conditionally-rendered components with `@defer` to lazy-load them:

```html
<!-- Lazy-load a chart component only when visible in viewport -->
@defer (on viewport) {
  <app-analytics-chart [data]="store.chartData()" />
} @placeholder {
  <div class="h-64 animate-pulse bg-surface-variant rounded"></div>
} @loading (minimum 200ms) {
  <app-spinner />
}

<!-- Lazy-load based on condition -->
@defer (when showDetails()) {
  <app-product-details [product]="store.selectedProduct()" />
}

<!-- Lazy-load on interaction -->
@defer (on interaction) {
  <app-heavy-editor />
} @placeholder {
  <button>Open Editor</button>
}
```

**When to use `@defer`:**
- Components importing heavy 3rd-party libraries (charts, editors, maps)
- Below-the-fold content not visible on initial render
- Tabs/panels that only render on user interaction
- Any component whose JS bundle is > 50KB

**Key rules:**
- `@if` / `@else if` / `@else` — no `*ngIf`
- `@for` with mandatory `track` — no `*ngFor`
- `@switch` / `@case` — no `*ngSwitch`
- `@empty` block for empty lists
- Call signals with `()` in templates

---

## 5. Architecture Rules

### Dependency Graph

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
               └──▶ styles/

ui/      ──────▶ (nothing - self-contained)
core/    ──────▶ styles/
styles/  ──────▶ (nothing - foundation layer)
```

### FORBIDDEN Dependencies

| From | To | Why |
|------|----|-----|
| `feature/` | `feature/` | Features are isolated |
| `ui/` | `core/` | UI must not inject services |
| `ui/` | `pattern/` | UI is self-contained |
| `pattern/` | `feature/` | Would create circular dep |
| `pattern/` | `pattern/` | Patterns don't depend on each other |
| `pattern/` | `layout/` | Pattern is lower level |
| `core/` | `feature/` | Core is lower level |
| `core/` | `pattern/` | Core is lower level |
| `layout/` | `feature/` | Layout is shared |

### Feature Module Structure

```
feature/budget/
├── budget.ts                  # Main component (smart)
├── budget.routes.ts           # Lazy-loaded routes
├── budget.store.ts            # Feature state (signal store)
├── budget-list.ts             # List component
├── budget-card.ts             # Card component (dumb)
├── budget-form.ts             # Form component
└── budget.service.ts          # API service (if needed)
```

**Source:** `references/angular-architecture.md`

---

## 6. TypeScript Patterns

```typescript
// Use #field (native private)
readonly #store = inject(BudgetStore);

// Use unknown + type guard (not any)
function processData(data: unknown): Budget {
  if (!isBudget(data)) {
    throw new Error('Invalid budget data');
  }
  return data;
}

// Use structuredClone (not JSON.parse(JSON.stringify))
const copy = structuredClone(original);

// Use toSorted/toReversed (not sort/reverse)
const sorted = items.toSorted((a, b) => a.name.localeCompare(b.name));

// Use Object.groupBy (not manual reduce)
const grouped = Object.groupBy(transactions, t => t.type);
```

---

## 7. Styling Patterns

```scss
// Use CSS custom properties (not ::ng-deep)
:host {
  --card-bg: var(--mat-sys-surface-container);
  --card-radius: var(--mat-sys-corner-medium);
}

// Use Material M3 tokens
.card {
  background: var(--card-bg);
  border-radius: var(--card-radius);
}
```

```html
<!-- Tailwind v4 syntax -->
<div class="bg-(--mat-sys-surface) text-(--mat-sys-on-surface)">

<!-- Material button (modern) -->
<button mat-button="filled">Ajouter</button>
```

---

## 8. Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| Boolean signal | `is` prefix | `isLoading`, `isActive`, `isOpen` |
| Array signal | Plural | `items`, `budgets`, `transactions` |
| Computed | Descriptive | `formattedAmount`, `filteredBudgets` |
| Effect | Purpose-based | `#persistEffect`, `#analyticsEffect` |
| Store | `FeatureStore` | `BudgetStore`, `DashboardStore` |
| Private field | `#` prefix | `#state`, `#http`, `#config` |

---

## 9. API Validation (Optional)

If your project uses Zod (recommended for runtime safety), follow these patterns. Otherwise, validate API responses with type guards, io-ts, or class-validator.

```typescript
// Define schema in your shared types package
import { z } from 'zod';

export const budgetSchema = z.object({
  id: z.string(),
  name: z.string(),
  amount: z.number(),
  type: z.enum(['income', 'expense', 'saving']),
});

export type Budget = z.infer<typeof budgetSchema>;
export const budgetListSchema = z.array(budgetSchema);

// Use in service
readonly budgets = httpResource(() => `/api/budgets`, {
  parse: budgetListSchema.parse,
});
```

**Key rules:**
- Define schemas near the API boundary (in a shared package, or locally in a `types/` folder)
- Types derived with `z.infer<>`
- `parse` at API boundaries

---

## 10. ViewModel Separation

The store's `computed()` selectors form the ViewModel layer. Components just bind.

```
API (DataModel) → Store computed() (ViewModel) → Component → Template
```

| Rule | Rationale |
|------|-----------|
| No `.filter()`/`.map()`/`.reduce()` in templates | Templates should bind, not transform |
| No `computed()` in components reading store data | Move to store — shared, testable, reactive |
| Store returns numbers/enums, not formatted strings | Formatting is a view concern (pipes, Decimal extensions) |
| One `computed()` per concern, not one god object | Fine-grained signal tracking |
| No manual ViewModel construction in `ngOnInit` | Breaks reactivity — use `computed()` |

For detailed patterns and anti-patterns: see `references/viewmodel-patterns.md`.

---

## 11. AI Slop Prevention

Code should read like a senior developer wrote it by hand. Remove:

| Slop | Action |
|------|--------|
| Comments that restate the next line | Delete |
| `try/catch` around non-throwing code | Remove |
| Null checks on DI-injected or typed values | Remove — trust types |
| Single-use helper functions | Inline |
| JSDoc on obvious methods | Delete |
| Abstractions with one consumer | Inline |
| Verbose variable names (>25 chars) | Shorten |

For detailed detection guide: see `references/ai-slop-detection.md`.

---

## 12. Angular Style Guide Conventions

From angular.dev/style-guide — official Angular team recommendations.

### Member Visibility

```typescript
@Component({
  template: `<p>{{ fullName() }}</p>`,
})
export class UserProfileComponent {
  readonly firstName = input<string>();
  readonly lastName = input<string>();

  // Only accessed by template — protected, not public
  protected readonly fullName = computed(() =>
    `${this.firstName()} ${this.lastName()}`
  );

  // Public API — called by parent components
  resetForm(): void { /* ... */ }
}
```

**Rule**: `protected` for template-only members. `public` only for the component's external API.

### Readonly on Angular-Initialized Properties

```typescript
readonly userId = input<string>();
readonly userSaved = output<void>();
readonly userName = model<string>();
readonly nameInput = viewChild<ElementRef>('nameInput');
```

Applies to: `input()`, `input.required()`, `output()`, `model()`, `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()`.

### Event Handler Naming

```html
<!-- Name for what they do, not the event -->
<button (click)="saveUserData()">Save</button>
<button (click)="deleteBudget()">Delete</button>

<!-- Not this -->
<button (click)="handleClick()">Save</button>
<button (click)="onClick()">Delete</button>
```

### Component Member Order

```typescript
export class BudgetCardComponent {
  // 1. Injected dependencies
  readonly #store = inject(BudgetStore);

  // 2. Inputs
  readonly budget = input.required<Budget>();

  // 3. Outputs
  readonly selected = output<Budget>();

  // 4. Queries
  readonly nameInput = viewChild<ElementRef>('nameInput');

  // 5. Computed / derived state
  protected readonly displayName = computed(() => this.budget().name);

  // 6. Methods
  selectBudget(): void { /* ... */ }
}
```

### Lifecycle Hooks

- Always `implements OnInit`, `implements OnDestroy` — guarantees correct method name
- Keep hooks thin — delegate to well-named methods
- Empty hooks are dead code — remove them

For full reference: see `references/angular-style-guide.md`.

---

## 13. Error Handling

Errors should be handled at the callsite — the code that started the operation has the context to recover. The global `ErrorHandler` is for fatal/unexpected errors only (logging, analytics).

### Store async methods — always handle errors

```typescript
async loadBudgets(): Promise<void> {
  this.#state.update(s => ({ ...s, isLoading: true, error: null }));
  try {
    const budgets = await firstValueFrom(this.#budgetApi.getBudgets$());
    this.#state.update(s => ({ ...s, budgets, isLoading: false }));
  } catch (error) {
    this.#state.update(s => ({
      ...s,
      isLoading: false,
      error: isApiError(error) ? error.message : 'Erreur inattendue',
    }));
  }
}
```

### Resource error state in templates

```html
@if (store.budgets.status() === 'error') {
  <app-error [message]="store.budgets.error()?.message" />
} @else if (store.budgets.isLoading()) {
  <app-spinner />
} @else {
  @for (budget of store.budgets.value(); track budget.id) {
    <app-budget-card [budget]="budget" />
  }
}
```

### Zod at API boundaries

```typescript
// In httpResource parse option — Angular handles ZodError via resource status
readonly budgets = httpResource(() => `/api/budgets`, {
  parse: budgetListSchema.parse,
});

// In component code — use safeParse for graceful handling
const result = budgetSchema.safeParse(formValue);
if (!result.success) {
  this.#formErrors.set(result.error.flatten().fieldErrors);
  return;
}
```

### Error typing

```typescript
// Use unknown + narrowing, not any
catch (error: unknown) {
  if (isApiError(error)) {
    this.#state.update(s => ({ ...s, error: error.message }));
  } else {
    throw error; // Re-throw unexpected errors
  }
}
```

**Key rules:**
- Every `async` store method: `try/catch` with state revert on error
- Every `resource()` / `httpResource()`: check `.error()` or `.status()` in template
- Zod `.parse()` only in `httpResource({ parse })` — use `.safeParse()` elsewhere
- `catch (error: unknown)` — never `catch (e: any)`
- Never swallow errors silently (`catch (e) {}`)

**Source:** `references/angular-style-guide.md`, angular.dev/best-practices/error-handling

---

## 14. Signal Forms (Angular 21+)

Angular 21 introduces Signal Forms (`@angular/forms/signals`) — the recommended approach for new forms.

```typescript
import { form, FormField, validate, required, email } from '@angular/forms/signals';

@Component({
  imports: [FormField],
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [formField]="userForm.controls.name" />
      <input [formField]="userForm.controls.email" />
      <button type="submit" [disabled]="!userForm.valid()">Save</button>
    </form>
  `,
})
export class UserFormComponent {
  readonly userModel = signal({
    name: '',    // NEVER use null — use '' for strings
    email: '',
    age: 0,      // Use 0 for numbers, NOT null
  });

  readonly userForm = form(this.userModel, (path) => {
    required(path.name, { message: 'Name is required' });
    email(path.email, { message: 'Invalid email' });
  });

  onSubmit(): void {
    if (this.userForm.valid()) {
      console.log(this.userModel());
    }
  }
}
```

**Key rules:**
- Prefer Signal Forms for new forms in Angular 21+ projects
- NEVER use `null` as initial value — use `''`, `0`, `[]`
- Keep Reactive Forms (`FormGroup`/`FormControl`) for existing forms or complex scenarios
- Check `{project_profile}` — if project uses Reactive Forms throughout, don't flag them

---

## 15. Pipes

Pipes are Angular's built-in memoization for template transformations. Use them for formatting, not business logic.

### When to use pipes vs computed()

| Use case | Solution | Why |
|----------|----------|-----|
| Format a single value (currency, date, uppercase) | Pure pipe | Memoized by Angular, reusable across templates |
| Derive state from multiple signals | `computed()` in store | Reactive, testable, shared |
| Filter/sort a list | `computed()` in store | Business logic, not formatting |
| Format + locale-specific display | Pipe | View concern, locale-aware |

### Pattern:

```typescript
// Pure pipe — no DI, no side effects, memoized
@Pipe({ name: 'statusLabel', standalone: true })
export class StatusLabelPipe implements PipeTransform {
  transform(status: 'active' | 'archived' | 'draft'): string {
    const labels: Record<string, string> = {
      active: 'Active',
      archived: 'Archived',
      draft: 'Draft',
    };
    return labels[status] ?? status;
  }
}
```

```html
<!-- Good: pipe for formatting -->
{{ product.status | statusLabel }}
{{ product.price | currency:'EUR' }}

<!-- Bad: method call in template -->
{{ getStatusLabel(product.status) }}
```

**Key rules:**
- Pipes are ALWAYS pure (default) — never set `pure: false`
- Pipes format, they don't compute business logic
- If a pipe needs DI, it's not a pipe — use `computed()` in a store
- Use `NgOptimizedImage` (`ngSrc`) for images, not raw `<img src="">`
