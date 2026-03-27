# Response Patterns

## Table of Contents

1. [Response Types](#response-types)
2. [Status Codes](#status-codes)
3. [Response Transformer Interceptor](#response-transformer-interceptor)
4. [Response Validation Interceptor](#response-validation-interceptor)
5. [Request Type Hierarchy](#request-type-hierarchy)

## Response Types

Define typed response classes so the API contract is explicit:

```typescript
// core/http/response.ts

export enum StatusCode {
  SUCCESS = 10000,
  FAILURE = 10001,
  RETRY = 10002,
  INVALID_ACCESS_TOKEN = 10003,
}

export class MessageResponse {
  readonly statusCode: StatusCode;
  readonly message: string;

  constructor(statusCode: StatusCode, message: string) {
    this.statusCode = statusCode;
    this.message = message;
  }
}

export class DataResponse<T> extends MessageResponse {
  readonly data: T;

  constructor(statusCode: StatusCode, message: string, data: T) {
    super(statusCode, message);
    this.data = data;
  }
}
```

**Why a custom statusCode instead of HTTP status codes alone?**

HTTP status codes are coarse — `401` means "unauthorized" but doesn't tell the client whether to show a login screen or trigger a token refresh. A custom status code like `INVALID_ACCESS_TOKEN` gives the client actionable instructions without parsing error messages.

## Status Codes

| Code | Meaning | Client Action |
|------|---------|---------------|
| 10000 | Success | Display data |
| 10001 | Failure | Show error message |
| 10002 | Retry | Retry the request |
| 10003 | Invalid Access Token | Refresh token or logout |

## Response Transformer Interceptor

Automatically wraps controller return values in the standard response envelope:

```typescript
// core/interceptors/response.transformer.ts
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common';
import { Observable, map } from 'rxjs';
import { DataResponse, MessageResponse, StatusCode } from '../http/response';

@Injectable()
export class ResponseTransformer implements NestInterceptor {
  intercept(_: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => {
        // Already wrapped — pass through
        if (data instanceof MessageResponse) return data;
        if (data instanceof DataResponse) return data;

        // String return → MessageResponse
        if (typeof data === 'string')
          return new MessageResponse(StatusCode.SUCCESS, data);

        // Object/DTO return → DataResponse
        return new DataResponse(StatusCode.SUCCESS, 'success', data);
      }),
    );
  }
}
```

This means controllers just return DTOs or strings — the interceptor handles wrapping.

## Response Validation Interceptor

Catches contract violations before they reach the client. If a response DTO has invalid fields, it throws `InternalServerErrorException` instead of sending malformed data:

```typescript
// core/interceptors/response.validation.ts
import {
  Injectable, NestInterceptor, ExecutionContext, CallHandler,
  InternalServerErrorException,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import { ValidationError, validateSync } from 'class-validator';

@Injectable()
export class ResponseValidation implements NestInterceptor {
  intercept(_: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => {
        if (Array.isArray(data)) {
          data.forEach((item) => {
            if (item instanceof Object) this.validate(item);
          });
        } else if (data instanceof Object) {
          this.validate(data);
        }
        return data;
      }),
    );
  }

  private validate(data: any) {
    const errors = validateSync(data);
    if (errors.length > 0) {
      const messages = this.extractMessages(errors);
      throw new InternalServerErrorException(['Response validation failed', ...messages]);
    }
  }

  private extractMessages(errors: ValidationError[], result: string[] = []): string[] {
    for (const error of errors) {
      if (error.children?.length) this.extractMessages(error.children, result);
      if (error.constraints) result.push(Object.values(error.constraints).join(', '));
    }
    return result;
  }
}
```

**When to use this:** Only with class-validator DTOs. If using Zod, schema validation already covers this.

## Request Type Hierarchy

Typed request objects carry authentication context through the pipeline:

```typescript
// core/http/request.ts
import { Request } from 'express';

// Base — all requests have an API key after ApiKeyGuard
export interface PublicRequest extends Request {
  apiKey: ApiKey;
}

// After RolesGuard checks roles
export interface RoleRequest extends PublicRequest {
  currentRoleCodes: string[];
}

// After AuthGuard authenticates — full context
export interface ProtectedRequest extends RoleRequest {
  user: User;
  accessToken: string;
  keystore: Keystore;
}
```

Guards progressively enrich the request object. A controller handler declares which type it expects, making the required auth level explicit in the type signature.
