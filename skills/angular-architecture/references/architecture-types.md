# Architecture Types Reference

## Architecture Types Quick Reference

### Core (Eager, Headless)

- **Location**: `core/`
- **Contains**: Services, interceptors, guards, functions, state management
- **Bundling**: Part of initial `main.js` bundle
- **Can import from**: Nothing (base layer)
- **Use for**: Logic needed from start OR shared by 2+ features
- **Organization**: Domain-based (`core/auth/`, `core/user/`, `core/order/`)
- **Setup**: `provideCore()` function for all eager providers

### Layout (Eager, Template-based)

- **Location**: `layout/`
- **Contains**: Standalone components, directives, pipes
- **Bundling**: Part of initial bundle
- **Can import from**: `core`, `ui`, `pattern`
- **Use for**: Application shell, navigation, headers, sidebars
- **Variants**: Single layout (in AppComponent), multiple layouts (via routes)

### UI (Eager/Lazy, Pure)

- **Location**: `ui/`
- **Contains**: Only standalones (components, directives, pipes)
- **Bundling**: Bundler optimizes based on usage
- **Can import from**: Angular/Material framework only (self-contained from app code)
- **Use for**: Generic reusable widgets (buttons, cards, dialogs, menus)
- **CRITICAL**: NO app/business service injection. Angular/Material framework services are allowed (`MatDialogRef`, `ElementRef`, `DestroyRef`, etc.). Data flows via inputs/outputs.

### Pattern (Lazy, Reusable Use Cases)

- **Location**: `pattern/<pattern-name>/`
- **Contains**: Combination of standalones + injectables
- **Bundling**: Lazy loaded
- **Can import from**: `core`, `ui`
- **Use for**: Cross-cutting business features (document manager, approval flows, audit logs)
- **Consumed via**: "Drop-in" component (NOT routes)

### Feature (Lazy, Isolated)

- **Location**: `feature/<feature-name>/`
- **Contains**: Any building blocks
- **Bundling**: Separate lazy bundle per feature
- **Can import from**: `core`, `ui`, `pattern`
- **CRITICAL**: Complete isolation - features CANNOT import from each other
- **Properties**: Black box, throw-away, can have sub-features
