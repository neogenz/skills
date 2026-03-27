# Exception Handling

## Table of Contents

1. [Philosophy](#philosophy)
2. [BusinessException Pattern](#businessexception-pattern)
3. [Error Definitions](#error-definitions)
4. [Global Exception Filter](#global-exception-filter)
5. [Anti-Patterns](#anti-patterns)

## Philosophy

**Log or throw, never both.**

- Services **throw** exceptions with context
- The global exception filter **catches, logs, and formats** the response
- This prevents duplicate logging and keeps error formatting in one place

## BusinessException Pattern

A structured exception that carries both client-facing and logging-only data:

```typescript
export class BusinessException extends HttpException {
  readonly errorDef: ErrorDefinition;
  readonly details?: Record<string, unknown>;
  readonly loggingContext?: Record<string, unknown>;

  constructor(
    errorDef: ErrorDefinition,
    details?: Record<string, unknown>,        // Interpolated into client message
    loggingContext?: Record<string, unknown>,  // For logs only, never sent to client
    options?: { cause?: unknown },             // ES2022 error cause chain
  ) {
    const message = errorDef.message(details);
    super(message, errorDef.httpStatus, options);
    this.errorDef = errorDef;
    this.details = details;
    this.loggingContext = loggingContext;
  }
}
```

### Parameters

| # | Name | Purpose | Sent to client? |
|---|------|---------|----------------|
| 1 | `errorDef` | Error code + message factory + HTTP status | Yes (code + message) |
| 2 | `details` | Data interpolated into the message | Yes |
| 3 | `loggingContext` | Operation name, userId, entityId | No (logs only) |
| 4 | `options` | `{ cause: error }` — original error for cause chain | No (logs only) |

### Usage

```typescript
// Service throws with full context
async findById(id: string, user: User): Promise<Product> {
  const product = await this.productRepository.findOne(id);

  if (!product) {
    throw new BusinessException(
      ERROR_DEFINITIONS.PRODUCT_NOT_FOUND,
      { id },
      { operation: 'findById', userId: user.id },
    );
  }

  return product;
}

// Wrapping caught errors — always pass cause
async create(dto: CreateProductDto): Promise<Product> {
  try {
    return await this.productRepository.create(dto);
  } catch (error) {
    throw new BusinessException(
      ERROR_DEFINITIONS.PRODUCT_CREATE_FAILED,
      undefined,
      { operation: 'create' },
      { cause: error },  // Preserves the original error for debugging
    );
  }
}
```

## Error Definitions

Define errors as typed constants — single source of truth for error codes, messages, and HTTP status:

```typescript
// config/error-definitions.ts
import { HttpStatus } from '@nestjs/common';

export interface ErrorDefinition {
  code: string;
  message: (details?: Record<string, unknown>) => string;
  httpStatus: HttpStatus;
}

export const ERROR_DEFINITIONS = {
  // Product errors
  PRODUCT_NOT_FOUND: {
    code: 'ERR_PRODUCT_NOT_FOUND',
    message: (d) => d?.id ? `Product '${d.id}' not found` : 'Product not found',
    httpStatus: HttpStatus.NOT_FOUND,
  },
  PRODUCT_CREATE_FAILED: {
    code: 'ERR_PRODUCT_CREATE_FAILED',
    message: () => 'Failed to create product',
    httpStatus: HttpStatus.INTERNAL_SERVER_ERROR,
  },
  PRODUCT_ALREADY_EXISTS: {
    code: 'ERR_PRODUCT_ALREADY_EXISTS',
    message: (d) => d?.slug ? `Product with slug '${d.slug}' already exists` : 'Product already exists',
    httpStatus: HttpStatus.CONFLICT,
  },

  // Auth errors
  INVALID_CREDENTIALS: {
    code: 'ERR_INVALID_CREDENTIALS',
    message: () => 'Invalid email or password',
    httpStatus: HttpStatus.UNAUTHORIZED,
  },
  INSUFFICIENT_PERMISSIONS: {
    code: 'ERR_INSUFFICIENT_PERMISSIONS',
    message: () => 'You do not have permission to perform this action',
    httpStatus: HttpStatus.FORBIDDEN,
  },
} as const satisfies Record<string, ErrorDefinition>;
```

## Global Exception Filter

Catches all exceptions and formats them into a consistent error response:

```typescript
// core/filters/global-exception.filter.ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private readonly logger: Logger) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';
    let code = 'ERR_INTERNAL';
    let errors: string[] | undefined;

    if (exception instanceof BusinessException) {
      status = exception.getStatus();
      message = exception.message;
      code = exception.errorDef.code;

      // Log with full context (loggingContext stays server-side)
      if (status >= 500) {
        this.logger.error({ err: exception, ...exception.loggingContext }, message);
      }
    } else if (exception instanceof HttpException) {
      status = exception.getStatus();
      const body = exception.getResponse();
      if (typeof body === 'string') {
        message = body;
      } else if (typeof body === 'object' && 'message' in body) {
        message = Array.isArray(body.message) ? body.message[0] : body.message;
        errors = Array.isArray(body.message) ? body.message : undefined;
      }
    } else {
      // Unknown error — log full stack, send generic message
      this.logger.error({ err: exception }, 'Unhandled exception');
    }

    response.status(status).json({
      success: false,
      statusCode: status,
      message,
      code,
      errors,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
    });
  }
}
```

## Anti-Patterns

```typescript
// NEVER: Log AND throw — causes duplicate log entries
catch (error) {
  this.logger.error('Failed', error);  // The filter will log this
  throw new BusinessException(...);
}

// NEVER: Bare string exceptions — no error code, no structure
throw new NotFoundException('Product not found');
// Use BusinessException with ERROR_DEFINITIONS

// NEVER: Original error in loggingContext — use cause chain
throw new BusinessException(def, details, { originalError: error });
// Correct: { cause: error } as 4th parameter

// NEVER: Exception without context
throw new BusinessException(ERROR_DEFINITIONS.PRODUCT_NOT_FOUND);
// Missing: details (what product?), loggingContext (what operation?), cause
```
