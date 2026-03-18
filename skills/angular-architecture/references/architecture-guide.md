# Enterprise Angular Architecture
**Summary Type:** In-Depth Technical Analysis & Implementation Guide

## 1. Core Thesis
The document proposes a strict, tooling-validated (automated) architecture aimed at maintaining development velocity over the long term. It relies on the exclusive use of **Standalone APIs**, a **clear separation between template context and injector hierarchy**, and a **unidirectional dependency graph** to prevent cyclic coupling.

The guiding principle is: **Isolation is 3 to 5 times more valuable than DRY (Don't Repeat Yourself) in frontend.**

---

## 2. Fundamental Concepts and Theory

### The Dependency Graph
*   **Problem:** Aging projects tend toward "spaghetti" graphs with cycles (circular references), making refactoring impossible and increasing bundle sizes.
*   **Solution:** Use tools like `madge` to visualize and validate a unidirectional graph (left to right).
*   **Rule:** It is always possible to implement a feature without introducing a cycle.

### Bundling and Performance
*   **Reality:** The bottleneck is no longer just the network, but **JS parsing and execution**.
*   **Strategy:** Avoid monolithic bundles. The bundler (Webpack/Esbuild) splits files via dynamic imports (`import()`).
*   **Architecture:** A clean architecture physically dictates the bundle structure (Eager vs Lazy).

### Two Parallel Angular Systems
1.  **Standalones (Template Context):**
    *   Components, Directives, Pipes.
    *   Manage their own dependencies via the `imports: []` array.
    *   No indirection via `NgModule`.
2.  **Injectables (Injector Hierarchy):**
    *   Services, Guards, Interceptors, Configs.
    *   Managed by the injector hierarchy (`EnvironmentInjector` for lazy loading).
    *   **Rule:** A service provided in a lazy feature is isolated (inaccessible to other features). A service in `root` is a global singleton.

---

## 3. The 5 Architectural Pillars (Types)

The architecture divides the application into 5 distinct block types, each with strict import rules.

| Type | Folder | Loading | Technical Description | Allowed Content |
| :--- | :--- | :--- | :--- | :--- |
| **Core** | `core/` | **Eager** | Global "headless" logic needed at startup. | Services, Guards, Interceptors, State (NgRx), Utils. **No UI components.** |
| **Layout** | `layout/` | **Eager** | Page skeletons (Shells). | Standalone Components. Consumes `ui` and `core`. |
| **UI** | `ui/` | **Mixed** | "Dumb" presentation components. | Pure Standalones. **Forbidden to import `core`**. Communication via `@Input`/`@Output` only. |
| **Feature** | `feature/` | **Lazy** | Routed pages. Isolated "black boxes". | Components, Local Services, Routes. Can contain lazy sub-features. |
| **Pattern** | `pattern/` | **Lazy/Eager** | Cross-cutting business functionalities "Drop-in" (non-routed). | Standalones + Services. E.g.: `DocumentManager`, `AuditLog`. Consumed via tag in the template. |

### Implementation Details by Type

#### A. Core (`core/`)
*   Configuration via `provideCore()` in `app.config.ts`.
*   Contains global state (Auth, User) and logic shared across multiple lazy features.
*   **Initialization:** Use `ENVIRONMENT_INITIALIZER` to launch processes at startup.
*   **Migration:** For non-standalone third-party libs, use `importProvidersFrom()`.

#### B. UI (`ui/`)
*   **Strict rule:** Never inject services or state management.
*   If a UI component needs complex data, the parent (Feature/Layout) must pass it down.
*   **Optimization:** The bundler automatically extracts UI components into shared or lazy chunks based on their usage.

#### C. Feature (`feature/`)
*   Base unit of Lazy Loading (via Router).
*   **Isolation:** Feature A must **NEVER** import Feature B.
*   **"Extract one level up" rule:** If A and B need the same logic:
    *   If it's UI -> extract to `ui/`.
    *   If it's State/Service -> extract to `core/`.
    *   If it's a complex business flow -> extract to `pattern/`.
*   **Fractal Structure:** A feature can contain lazy sub-features (`feature/orders/dashboard`).

#### D. Pattern (`pattern/`)
*   Hybrid between UI and Feature.
*   Has business logic (access to `core`) and display.
*   Differs from Feature because it is not used via the router, but via a "drop-in" component (e.g.: `<my-org-comments />`).

---

## 4. Architecture Rules and Best Practices

### Isolation vs DRY Rule
*   Accept code duplication if it guarantees feature isolation.
*   **Heuristic:** Wait for at least 3 occurrences of repetition before abstracting common logic ("3-5x" rule).
*   A feature should be deletable ("throw-away code") without breaking the rest of the app.

### Data Flow Rule (State vs View)
*   Strictly separate logic (State/Headless) from the view.
*   Components should be "logic-free": they immediately delegate user interactions to services or the store (NgRx).

### Import Rules (One-Way Flow)
The allowed dependency flow is strictly hierarchical:
1.  **Feature** depends on -> `UI`, `Core`, `Pattern`.
2.  **Pattern** depends on -> `UI`, `Core`.
3.  **Layout** depends on -> `UI`, `Core`.
4.  **UI** depends on -> **NOTHING** (no Core).
5.  **Core** depends on -> **NOTHING** (especially not Feature or UI).

### Handling "Cute Abstractions"
*   Avoid creating wrappers around Angular or state management libs ("MyOrgAngular+").
*   Stay at the framework's abstraction level. Verbosity decreases over time via updates (e.g.: `takeUntilDestroyed`).

---

## 5. Quick Implementation Guide

### Recommended Folder Structure
```text
src/
├── app/
│   ├── core/           # Singleton services, config, global state
│   │   ├── auth/
│   │   ├── user/
│   │   └── core.ts     # export function provideCore()
│   ├── layout/         # Application shells
│   ├── ui/             # Pure reusable components
│   ├── feature/        # Lazy-loaded pages (Business logic)
│   │   ├── dashboard/
│   │   └── settings/
│   ├── pattern/        # Reusable business functionalities (Drop-in)
│   └── app.routes.ts
```

### Code Sharing Scenarios ("Extract one level up")
1.  **Component used in 1 Feature:** Stays in `feature/A/components`.
2.  **Component used in Feature A and Feature B:** Move to `ui/` (if generic) or `pattern/` (if business-specific).
3.  **Service used in Feature A and Feature B:** Move to `core/`.
4.  **Lazy sub-feature:** Can be shared via routing (`loadChildren` pointing to a sibling folder), but watch out for coupling.
