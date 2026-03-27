# Custom Decorators and Pipes

## Table of Contents

1. [Parameter Decorators](#parameter-decorators)
2. [Custom Pipes](#custom-pipes)
3. [Custom Validation Decorators](#custom-validation-decorators)

## Parameter Decorators

### @User() — Extract authenticated user from request

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);

// Usage in controller
@Get('profile')
async getProfile(@User() user: AuthenticatedUser) {
  return this.userService.getProfile(user.id);
}
```

### @SupabaseClient() — Extract authenticated Supabase client

```typescript
export const SupabaseClient = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.supabaseClient;
  },
);

// Usage — the client has RLS applied for the authenticated user
@Post()
async create(
  @Body() dto: CreateBudgetDto,
  @User() user: AuthenticatedUser,
  @SupabaseClient() supabase: AuthenticatedSupabaseClient,
) {
  return this.budgetService.create(dto, user, supabase);
}
```

## Custom Pipes

### MongoIdTransformer — Validate and convert ID params

Validates that a string parameter is a valid MongoDB ObjectId and converts it:

```typescript
import {
  PipeTransform, Injectable, BadRequestException, ArgumentMetadata,
} from '@nestjs/common';
import { Types } from 'mongoose';

@Injectable()
export class MongoIdTransformer implements PipeTransform<any> {
  transform(value: any, metadata: ArgumentMetadata): any {
    if (typeof value !== 'string') return value;

    if (metadata.metatype?.name === 'ObjectId') {
      if (!Types.ObjectId.isValid(value)) {
        throw new BadRequestException(
          `${metadata?.data ?? 'id'} must be a valid MongoDB ObjectId`
        );
      }
      return new Types.ObjectId(value);
    }

    return value;
  }
}

// Usage
@Get(':id')
async findOne(
  @Param('id', MongoIdTransformer) id: Types.ObjectId,
): Promise<ProductDto> { ... }
```

### ParseUUIDPipe — Built-in for UUID validation

```typescript
// NestJS built-in — prefer for UUID-based IDs
@Get(':id')
async findOne(
  @Param('id', new ParseUUIDPipe({ version: '4' })) id: string,
): Promise<ProductDto> { ... }
```

## Custom Validation Decorators

### @IsMongoIdObject — For DTOs

A custom class-validator decorator for validating MongoDB ObjectIds in DTOs:

```typescript
import {
  registerDecorator, ValidationArguments, ValidationOptions,
} from 'class-validator';
import { Types } from 'mongoose';

export function IsMongoIdObject(validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      name: 'IsMongoIdObject',
      target: object.constructor,
      propertyName,
      constraints: [],
      options: validationOptions,
      validator: {
        validate(value: any) {
          return Types.ObjectId.isValid(value);
        },
        defaultMessage(args?: ValidationArguments) {
          return `${args?.property ?? 'field'} must be a valid MongoDB ObjectId`;
        },
      },
    });
  };
}

// Usage in DTO
export class ProductInfoDto {
  @IsMongoIdObject()
  _id: Types.ObjectId;

  @IsString()
  name: string;
}
```

### Creating Custom Validators (General Pattern)

```typescript
import { registerDecorator, ValidationOptions, ValidationArguments } from 'class-validator';

export function IsPositiveAmount(validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      name: 'isPositiveAmount',
      target: object.constructor,
      propertyName,
      options: validationOptions,
      validator: {
        validate(value: unknown): boolean {
          return typeof value === 'number' && value > 0 && Number.isFinite(value);
        },
        defaultMessage(args?: ValidationArguments): string {
          return `${args?.property} must be a positive finite number`;
        },
      },
    });
  };
}
```
