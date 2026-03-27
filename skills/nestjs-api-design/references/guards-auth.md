# Guards and Authentication

## Table of Contents

1. [Guard Execution Order](#guard-execution-order)
2. [API Key Guard](#api-key-guard)
3. [Auth Guard (JWT)](#auth-guard-jwt)
4. [Roles Guard](#roles-guard)
5. [Custom Decorators](#custom-decorators)
6. [Registration](#registration)

## Guard Execution Order

Guards run in registration order (providers array), before the controller handler:

```
Request → ApiKeyGuard → AuthGuard → RolesGuard → Pipes → Controller
```

Each guard returns `true` (proceed) or throws (reject). They progressively enrich the request object with authentication context.

## API Key Guard

Validates the `x-api-key` header and checks the key's permissions against the route's requirements:

```typescript
@Injectable()
export class ApiKeyGuard implements CanActivate {
  constructor(
    private readonly authService: AuthService,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Get required permissions from @Permissions decorator (default: GENERAL)
    const permissions = this.reflector.get(Permissions, context.getClass())
      ?? [Permission.GENERAL];

    const request = context.switchToHttp().getRequest<PublicRequest>();
    const key = request.headers['x-api-key']?.toString();
    if (!key) throw new ForbiddenException();

    const apiKey = await this.authService.findApiKey(key);
    if (!apiKey) throw new ForbiddenException();

    request.apiKey = apiKey;

    // Check if key has at least one required permission
    const hasPermission = permissions.some(required =>
      apiKey.permissions.includes(required)
    );
    if (!hasPermission) throw new ForbiddenException();

    return true;
  }
}
```

## Auth Guard (JWT)

Validates the Bearer token, resolves the user, and attaches user + keystore to the request:

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private readonly authService: AuthService,
    private readonly reflector: Reflector,
    private readonly userService: UserService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Check for @Public() decorator — skip auth
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    const request = context.switchToHttp().getRequest<ProtectedRequest>();
    const token = this.extractBearerToken(request);
    if (!token) throw new UnauthorizedException();

    const payload = await this.authService.verifyToken(token);
    if (!this.authService.validatePayload(payload)) {
      throw new UnauthorizedException('Invalid Access Token');
    }

    const user = await this.userService.findById(payload.sub);
    if (!user) throw new UnauthorizedException('User not registered');

    const keystore = await this.authService.findKeystore(user, payload.prm);
    if (!keystore) throw new UnauthorizedException('Invalid Access Token');

    // Enrich request with auth context
    request.user = user;
    request.accessToken = token;
    request.keystore = keystore;

    return true;
  }

  private extractBearerToken(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

## Roles Guard

Checks user roles against the `@Roles()` decorator. Supports both handler-level and class-level roles:

```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Check handler first, then class
    let roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) roles = this.reflector.get(Roles, context.getClass());

    // No @Roles() decorator — allow all authenticated users
    if (!roles) return true;

    const request = context.switchToHttp().getRequest<ProtectedRequest>();
    if (!request.user) throw new ForbiddenException('Permission Denied');

    const hasRole = request.user.roles.some(role =>
      roles!.includes(role.code)
    );
    if (!hasRole) throw new ForbiddenException('Permission Denied');

    return true;
  }
}
```

## Custom Decorators

### @Public() — Skip authentication

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

### @Roles() — Require specific roles

```typescript
import { Reflector } from '@nestjs/core';
import { RoleCode } from '../schemas/role.schema';

export const Roles = Reflector.createDecorator<RoleCode[]>();
```

### @Permissions() — Require API key permissions

```typescript
import { Reflector } from '@nestjs/core';
import { Permission } from '../schemas/apikey.schema';

export const Permissions = Reflector.createDecorator<Permission[]>();
```

### Usage in controllers

```typescript
// Public endpoint — no auth required
@Public()
@Get('health')
health() { return 'ok'; }

// Admin-only controller
@Roles([RoleCode.ADMIN])
@Controller('admin/products')
export class ProductAdminController { ... }

// Specific permission on a controller
@Permissions([Permission.EDITOR])
@Controller('content')
export class ContentController { ... }
```

## Registration

Register guards globally in the CoreModule (or AppModule):

```typescript
@Module({
  providers: [
    { provide: APP_GUARD, useClass: ApiKeyGuard },
    { provide: APP_GUARD, useClass: AuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class CoreModule {}
```

**Order matters** — guards execute in the order they're registered in the providers array.

To apply a guard to a specific controller or handler instead of globally:

```typescript
@UseGuards(SpecificGuard)
@Get('protected')
async protectedRoute() { ... }
```
