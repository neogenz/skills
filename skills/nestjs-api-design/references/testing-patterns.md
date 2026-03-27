# Testing Patterns

## Table of Contents

1. [Test Structure](#test-structure)
2. [Unit Tests](#unit-tests)
3. [Integration Tests](#integration-tests)
4. [Mocking Patterns](#mocking-patterns)
5. [Test Configuration](#test-configuration)

## Test Structure

```
src/
├── modules/product/
│   ├── product.service.ts
│   ├── product.service.spec.ts      # Unit test — colocated with source
│   ├── product.controller.spec.ts   # Unit test
│   └── ...
test/
├── product.e2e-spec.ts              # Integration test
├── setup.ts                         # Test bootstrap
└── test-mocks.ts                    # Shared mock factories
```

**Convention:** Unit tests live next to the code they test (`.spec.ts`). Integration tests live in the `test/` directory (`.e2e-spec.ts`).

## Unit Tests

### Testing a Service

```typescript
describe('ProductService', () => {
  let service: ProductService;
  let repository: jest.Mocked<ProductRepository>;
  let mapper: ProductMapper;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        ProductService,
        ProductMapper,
        {
          provide: ProductRepository,
          useValue: {
            findOne: jest.fn(),
            create: jest.fn(),
            update: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(ProductService);
    repository = module.get(ProductRepository);
    mapper = module.get(ProductMapper);
  });

  describe('findById', () => {
    it('should return product when found', async () => {
      const mockProduct = { id: '1', name: 'Widget', price: 9.99 };
      repository.findOne.mockResolvedValue(mockProduct);

      const result = await service.findById('1');
      expect(result.data.name).toBe('Widget');
    });

    it('should throw BusinessException when not found', async () => {
      repository.findOne.mockResolvedValue(null);

      await expect(service.findById('999'))
        .rejects.toThrow(BusinessException);
    });
  });
});
```

### Testing a Guard

```typescript
describe('AuthGuard', () => {
  let guard: AuthGuard;
  let authService: jest.Mocked<AuthService>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        AuthGuard,
        { provide: AuthService, useValue: { verifyToken: jest.fn() } },
        { provide: Reflector, useValue: { getAllAndOverride: jest.fn() } },
        { provide: UserService, useValue: { findById: jest.fn() } },
      ],
    }).compile();

    guard = module.get(AuthGuard);
    authService = module.get(AuthService);
  });

  it('should allow public routes', async () => {
    const context = createMockContext({ isPublic: true });
    const result = await guard.canActivate(context);
    expect(result).toBe(true);
  });

  it('should reject missing token', async () => {
    const context = createMockContext({ headers: {} });
    await expect(guard.canActivate(context))
      .rejects.toThrow(UnauthorizedException);
  });
});
```

### Testing a Controller

```typescript
describe('ProductController', () => {
  let controller: ProductController;
  let service: jest.Mocked<ProductService>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      controllers: [ProductController],
      providers: [
        {
          provide: ProductService,
          useValue: {
            findById: jest.fn(),
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    controller = module.get(ProductController);
    service = module.get(ProductService);
  });

  it('should delegate to service and return result', async () => {
    const expected = { success: true, data: { id: '1', name: 'Widget' } };
    service.findById.mockResolvedValue(expected);

    const result = await controller.findOne('1');
    expect(result).toEqual(expected);
    expect(service.findById).toHaveBeenCalledWith('1');
  });
});
```

## Integration Tests

Integration tests boot the full NestJS app and test the HTTP pipeline end-to-end:

```typescript
// test/product.e2e-spec.ts
describe('Product (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('GET /products/:id', () => {
    it('should return 200 with product data', () => {
      return request(app.getHttpServer())
        .get('/products/valid-id')
        .set('Authorization', `Bearer ${testToken}`)
        .expect(200)
        .expect((res) => {
          expect(res.body.success).toBe(true);
          expect(res.body.data).toHaveProperty('name');
        });
    });

    it('should return 404 for missing product', () => {
      return request(app.getHttpServer())
        .get('/products/nonexistent-id')
        .set('Authorization', `Bearer ${testToken}`)
        .expect(404)
        .expect((res) => {
          expect(res.body.code).toBe('ERR_PRODUCT_NOT_FOUND');
        });
    });
  });
});
```

## Mocking Patterns

### Mock Factory Functions

Create reusable mock factories:

```typescript
// test/test-mocks.ts
export function createMockSupabaseClient() {
  return {
    from: jest.fn().mockReturnThis(),
    select: jest.fn().mockReturnThis(),
    insert: jest.fn().mockReturnThis(),
    update: jest.fn().mockReturnThis(),
    delete: jest.fn().mockReturnThis(),
    eq: jest.fn().mockReturnThis(),
    single: jest.fn().mockResolvedValue({ data: null, error: null }),
    rpc: jest.fn().mockResolvedValue({ data: null, error: null }),
  };
}

export function createMockContext(overrides: Partial<MockContextOptions> = {}) {
  const request = {
    headers: overrides.headers ?? { authorization: 'Bearer test-token' },
    user: overrides.user ?? { id: 'user-1', roles: [] },
    ...overrides,
  };

  return {
    switchToHttp: () => ({ getRequest: () => request }),
    getHandler: () => ({}),
    getClass: () => ({}),
  } as ExecutionContext;
}
```

## Test Configuration

Separate test environment:

```bash
# .env.test
NODE_ENV=test
DB_HOST=localhost
DB_PORT=5432
DB_NAME=app_test
```

Load in test setup:

```typescript
// test/setup.ts
process.env.NODE_ENV = 'test';
```

**Target: 75%+ code coverage.** Focus coverage on services (business logic) and guards (security). Controllers are thin enough that service tests indirectly cover most controller paths.
