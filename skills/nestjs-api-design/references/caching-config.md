# Caching and Configuration

## Table of Contents

1. [Configuration Pattern](#configuration-pattern)
2. [Redis Caching](#redis-caching)
3. [Database Setup](#database-setup)

## Configuration Pattern

### Typed Config Registration

Define each config domain as a typed interface + factory:

```typescript
// config/database.config.ts
import { registerAs } from '@nestjs/config';

export const DatabaseConfigName = 'database';

export interface DatabaseConfig {
  host: string;
  port: number;
  name: string;
  user: string;
  password: string;
  minPoolSize: number;
  maxPoolSize: number;
}

export default registerAs(DatabaseConfigName, () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432', 10),
  name: process.env.DB_NAME || '',
  user: process.env.DB_USER || '',
  password: process.env.DB_USER_PWD || '',
  minPoolSize: parseInt(process.env.DB_MIN_POOL_SIZE || '5', 10),
  maxPoolSize: parseInt(process.env.DB_MAX_POOL_SIZE || '10', 10),
}));
```

### Usage

```typescript
// Prefer getOrThrow — fail fast on missing config
const dbConfig = this.configService.getOrThrow<DatabaseConfig>(DatabaseConfigName);

// Optional values with defaults
const port = this.configService.get<number>('PORT', 3000);
```

### Loading in AppModule

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig, serverConfig, cacheConfig, authConfig],
      isGlobal: true,  // No need to import ConfigModule in every module
      cache: true,      // Cache parsed config values
      envFilePath: [
        `.env.${process.env.NODE_ENV || 'development'}`,
        '.env.local',
        '.env',
      ],
    }),
  ],
})
export class AppModule {}
```

### Config Validation

Validate all required env vars at startup — fail fast, not at runtime:

```typescript
// config/environment.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3000),
  DB_HOST: z.string().min(1),
  DB_NAME: z.string().min(1),
  // ...
});

export function validateConfig(config: Record<string, unknown>) {
  return envSchema.parse(config);
}
```

## Redis Caching

### Cache Factory

```typescript
// cache/cache.factory.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { CacheModuleOptions, CacheOptionsFactory } from '@nestjs/cache-manager';
import { createKeyv } from '@keyv/redis';

@Injectable()
export class CacheConfigFactory implements CacheOptionsFactory {
  constructor(private readonly configService: ConfigService) {}

  async createCacheOptions(): Promise<CacheModuleOptions> {
    const host = this.configService.getOrThrow<string>('CACHE_HOST');
    const port = this.configService.getOrThrow<number>('CACHE_PORT');
    const password = this.configService.getOrThrow<string>('CACHE_PASSWORD');

    const keyv = createKeyv(`redis://:${password}@${host}:${port}`);
    return { stores: keyv };
  }
}
```

### Cache Service

Wrap cache-manager for a clean API:

```typescript
// cache/cache.service.ts
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject, Injectable } from '@nestjs/common';
import { Cache } from 'cache-manager';

@Injectable()
export class CacheService {
  constructor(@Inject(CACHE_MANAGER) private readonly cache: Cache) {}

  async get<T>(key: string): Promise<T | null> {
    return this.cache.get(key);
  }

  async set(key: string, value: unknown, ttlMs?: number): Promise<void> {
    await this.cache.set(key, value, ttlMs);
  }

  async delete(key: string): Promise<void> {
    await this.cache.del(key);
  }

  async invalidateForUser(userId: string): Promise<void> {
    // Pattern-based invalidation if supported, or manual key deletion
    await this.cache.del(`user:${userId}:*`);
  }

  onModuleDestroy() {
    this.cache.disconnect();
  }
}
```

### Cache Module

```typescript
// cache/cache.module.ts
@Module({
  imports: [
    CacheModule.registerAsync({
      imports: [ConfigModule],
      useClass: CacheConfigFactory,
    }),
  ],
  providers: [CacheService],
  exports: [CacheService, CacheModule],
})
export class AppCacheModule {}
```

### Using Cache on Endpoints

Apply caching per-route, not globally:

```typescript
@UseInterceptors(CacheInterceptor)
@Get()
async findAll(): Promise<ProductListResponse> {
  return this.productService.findAll();
}
```

### Cache Invalidation

**Always invalidate after mutations.** Stale cache is worse than no cache:

```typescript
async update(id: string, dto: UpdateProductDto): Promise<Product> {
  const result = await this.productRepository.update(id, dto);
  await this.cacheService.invalidateForUser(result.userId);
  return result;
}
```

## Database Setup

### Mongoose (MongoDB)

```typescript
// setup/database.factory.ts
@Injectable()
export class DatabaseFactory implements MongooseOptionsFactory {
  constructor(private readonly configService: ConfigService) {}

  createMongooseOptions(): MongooseModuleOptions {
    const db = this.configService.getOrThrow<DatabaseConfig>(DatabaseConfigName);
    const uri = `mongodb://${db.user}:${encodeURIComponent(db.password)}@${db.host}:${db.port}/${db.name}`;

    return {
      uri,
      autoIndex: true,
      minPoolSize: db.minPoolSize,
      maxPoolSize: db.maxPoolSize,
      connectTimeoutMS: 60000,
      socketTimeoutMS: 45000,
    };
  }
}

// In AppModule
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useClass: DatabaseFactory,
}),
```

### Supabase (PostgreSQL)

```typescript
// modules/supabase/supabase.service.ts
@Injectable()
export class SupabaseService {
  private readonly supabaseUrl: string;
  private readonly supabaseAnonKey: string;

  constructor(private readonly configService: ConfigService) {
    this.supabaseUrl = configService.getOrThrow('SUPABASE_URL');
    this.supabaseAnonKey = configService.getOrThrow('SUPABASE_ANON_KEY');
  }

  createAuthenticatedClient(accessToken: string): SupabaseClient {
    return createClient(this.supabaseUrl, this.supabaseAnonKey, {
      global: { headers: { Authorization: `Bearer ${accessToken}` } },
    });
  }
}
```

### Schema Definition (Mongoose)

```typescript
@Schema({ collection: 'products', versionKey: false, timestamps: true })
export class Product {
  readonly _id: Types.ObjectId;

  @Prop({ required: true, maxlength: 100, trim: true })
  name: string;

  @Prop({ required: true, min: 0 })
  price: number;

  @Prop({ default: true })
  status: boolean;
}

export const ProductSchema = SchemaFactory.createForClass(Product);

// Indexes for performance
ProductSchema.index({ name: 'text' }, { background: true });
ProductSchema.index({ _id: 1, status: 1 });
```
