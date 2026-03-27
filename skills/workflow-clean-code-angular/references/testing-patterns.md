# Angular Testing Patterns

> Reference for auditing test quality — not just "do tests exist?" but "are they testing the right thing?"

---

## Philosophy

Good tests verify **behavior**, not implementation. They answer "what does this code DO?" not "how does it do it internally?" A test that breaks when you refactor internals (without changing behavior) is a bad test.

---

## 1. Test Quality Checklist

| Check | Good | Bad | Why |
|-------|------|-----|-----|
| Tests behavior, not implementation | `expect(component.items()).toHaveLength(3)` | `expect(store.#state).toEqual(...)` | Internal state can change without breaking behavior |
| AAA structure | Clear Arrange / Act / Assert sections | Logic mixed with assertions | Readability and maintenance |
| Descriptive names | `it('should filter products by category when filter changes')` | `it('test 1')` / `it('works')` | Failing test name = instant understanding |
| One assertion per concept | Each `it()` tests one behavior | `it()` with 10 assertions | Pinpoints failures |
| Tests edge cases | Empty array, null, boundary values, error paths | Only happy path | Real bugs hide in edges |
| No DOM queries by CSS class | `data-testid="submit-btn"` | `querySelector('.btn-primary')` | CSS changes shouldn't break tests |
| Tests are independent | Each test sets up its own state | Tests depend on execution order | Flaky tests |
| Async properly handled | `fakeAsync` + `tick()` or `await` | `setTimeout` in tests | Deterministic results |

---

## 2. Component Test Patterns

### What to test in a component:
- **Inputs/outputs**: signal inputs produce correct rendering, outputs emit on user action
- **Template bindings**: `@if`/`@for` render correctly based on signal state
- **User interactions**: click → output emitted, form submit → store method called
- **Computed signals**: derived state reacts correctly to input changes

### What NOT to test in a component:
- Store internals (test the store separately)
- Third-party library behavior (Material, Tailwind)
- Template structure (exact DOM nesting) — test content, not structure

### Pattern:

```typescript
describe('ProductCardComponent', () => {
  it('should display product name from input', async () => {
    const fixture = TestBed.createComponent(ProductCardComponent);
    const component = fixture.componentInstance;

    // Arrange
    fixture.componentRef.setInput('product', { id: '1', name: 'Widget', price: 9.99 });
    fixture.detectChanges();

    // Assert
    const name = fixture.nativeElement.querySelector('[data-testid="product-name"]');
    expect(name.textContent).toContain('Widget');
  });

  it('should emit selected when clicked', () => {
    const fixture = TestBed.createComponent(ProductCardComponent);
    const spy = jest.fn();
    fixture.componentInstance.selected.subscribe(spy);

    // Arrange
    fixture.componentRef.setInput('product', mockProduct);
    fixture.detectChanges();

    // Act
    fixture.nativeElement.querySelector('[data-testid="card"]').click();

    // Assert
    expect(spy).toHaveBeenCalledWith(mockProduct);
  });
});
```

---

## 3. Store Test Patterns

### What to test in a store:
- **Selectors**: computed signals return correct derived state
- **Mutations**: state changes correctly after calling methods
- **Async operations**: loading/success/error states
- **Error handling**: error state set correctly, optimistic rollback works
- **Edge cases**: empty state, duplicate calls, concurrent operations

### Pattern:

```typescript
describe('ProductStore', () => {
  let store: ProductStore;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        ProductStore,
        { provide: HttpClient, useValue: { get: jest.fn() } },
      ],
    });
    store = TestBed.inject(ProductStore);
  });

  describe('selectors', () => {
    it('should return filtered products when filter is set', () => {
      // Arrange — set up state with products
      store.addProduct({ id: '1', name: 'Widget', category: 'tools' });
      store.addProduct({ id: '2', name: 'Gadget', category: 'electronics' });
      store.setFilter('tools');

      // Assert
      expect(store.filteredProducts()).toHaveLength(1);
      expect(store.filteredProducts()[0].name).toBe('Widget');
    });
  });

  describe('mutations', () => {
    it('should add product immutably', () => {
      const before = store.products();
      store.addProduct(mockProduct);

      expect(store.products()).toHaveLength(1);
      expect(store.products()).not.toBe(before); // new reference
    });
  });

  describe('error handling', () => {
    it('should set error state when loadProducts fails', async () => {
      httpMock.get.mockRejectedValue(new Error('Network error'));

      await store.loadProducts();

      expect(store.error()).toBe('Network error');
      expect(store.isLoading()).toBe(false);
    });
  });
});
```

---

## 4. Guard / Interceptor Test Patterns

```typescript
describe('AuthGuard', () => {
  it('should redirect to login when not authenticated', () => {
    authService.isAuthenticated.mockReturnValue(false);

    const result = guard.canActivate(mockRoute, mockState);

    expect(result).toEqual(router.createUrlTree(['/login']));
  });
});
```

---

## 5. Anti-Patterns in Tests

| Anti-Pattern | Why It's Bad | Fix |
|-------------|-------------|-----|
| Testing private methods directly | Couples test to implementation | Test the public API that uses the private method |
| `fixture.debugElement.query(By.css('.class'))` for assertions | Breaks on CSS changes | Use `data-testid` attributes |
| `spyOn(store, '#privateMethod')` | TypeScript private bypass = fragile | Test observable behavior instead |
| `expect(component.ngOnInit).toHaveBeenCalled()` | Tests Angular lifecycle, not your code | Test what ngOnInit causes (data loaded, state set) |
| Snapshot tests for templates | Break on any template change, even cosmetic | Test content and behavior, not structure |
| `setTimeout(() => expect(...), 100)` | Race condition, flaky | Use `fakeAsync`/`tick` or `await` |
| Mock everything | Tests nothing real | Mock only external boundaries (HTTP, localStorage) |
| No error path tests | Only happy path covered | Add tests for null, empty, error states |
| `beforeEach` with 50 lines of setup | Hard to understand what each test needs | Use factory functions for test data |

---

## 6. Coverage Strategy

- **Stores**: 90%+ — this is where business logic lives
- **Guards / Interceptors**: 90%+ — security-critical
- **Components**: 70%+ — test inputs/outputs/interactions, skip trivial bindings
- **Services**: 80%+ — test HTTP calls and transformations
- **Pipes**: 100% — pure functions, easy to test
- **Utils / Helpers**: 100% — pure functions

Don't chase 100% everywhere — prioritize business logic and security.
