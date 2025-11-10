<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Universal Patterns
## Foundation for Every Microservice

> **For AI:** These patterns are MANDATORY for all services. Apply these first, then add specialized patterns.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Purpose

Define the standard structure, configuration, and production requirements that **EVERY microservice** must implement.

**Non-negotiable.** These patterns ensure consistency, observability, and production-readiness across all services.

---

## 📂 Standard Project Structure

```
service-name/
├── src/
│   ├── config/
│   │   └── configuration.ts      # Environment config with validation
│   ├── modules/
│   │   ├── users/
│   │   │   ├── users.controller.ts
│   │   │   ├── users.service.ts
│   │   │   ├── users.module.ts
│   │   │   └── dto/
│   │   └── health/
│   │       └── health.controller.ts
│   ├── shared/
│   │   ├── filters/               # Global exception filters
│   │   ├── interceptors/          # Logging, correlation IDs
│   │   └── guards/                # Auth guards
│   ├── database/
│   │   ├── migrations/
│   │   └── database.module.ts
│   ├── app.module.ts
│   └── main.ts                    # Bootstrap with graceful shutdown
├── test/
├── Dockerfile
├── docker-compose.yml
├── package.json
└── tsconfig.json
```

**Why this structure:**
- `config/` - Centralized configuration
- `modules/` - Feature-based organization
- `shared/` - Cross-cutting concerns
- `database/` - Schema management
- Consistent across ALL services (easy onboarding)

---

## ⚙️ Configuration with Validation

```typescript
// src/config/configuration.ts
import * as Joi from 'joi'

export const configValidationSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'staging', 'production')
    .default('development'),
  PORT: Joi.number().default(3000),
  DATABASE_URL: Joi.string().required(),
  VAULT_ADDR: Joi.string().required(),
  VAULT_TOKEN: Joi.string().required(),
  RABBITMQ_URL: Joi.string().required(),
})

export default () => ({
  env: process.env.NODE_ENV,
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    url: process.env.DATABASE_URL,
  },
  vault: {
    address: process.env.VAULT_ADDR,
    token: process.env.VAULT_TOKEN,
  },
  rabbitmq: {
    url: process.env.RABBITMQ_URL,
  },
})

// src/app.module.ts
import { Module } from '@nestjs/common'
import { ConfigModule } from '@nestjs/config'
import configuration, { configValidationSchema } from './config/configuration'

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
      validationSchema: configValidationSchema,
      validationOptions: {
        abortEarly: false,  // Show all validation errors
      },
    }),
  ],
})
export class AppModule {}
```

**Benefit:** Service won't start with missing/invalid configuration. Fails fast.

---

## 🏥 Health Checks (Mandatory)

```typescript
// src/modules/health/health.controller.ts
import { Controller, Get } from '@nestjs/common'
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
  MicroserviceHealthIndicator,
} from '@nestjs/terminus'

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private microservice: MicroserviceHealthIndicator,
  ) {}

  @Get('liveness')
  @HealthCheck()
  checkLiveness() {
    // Is the service running?
    return this.health.check([])
  }

  @Get('readiness')
  @HealthCheck()
  checkReadiness() {
    // Is the service ready to accept traffic?
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.microservice.pingCheck('rabbitmq', {
        transport: Transport.RMQ,
        options: { /* RabbitMQ config */ },
      }),
    ])
  }
}
```

**Kubernetes uses these:**
- `/health/liveness` - If fails, restart pod
- `/health/readiness` - If fails, remove from load balancer

---

## 🚨 Global Exception Filter

```typescript
// src/shared/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common'

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse()
    const request = ctx.getRequest()

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR

    const message =
      exception instanceof HttpException
        ? exception.getResponse()
        : 'Internal server error'

    // Structured error response
    response.status(status).json({
      success: false,
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: typeof message === 'string' ? message : message['message'],
      error: exception instanceof Error ? exception.message : 'Unknown error',
      correlationId: request.headers['x-correlation-id'],
    })
  }
}

// Apply globally in main.ts
app.useGlobalFilters(new GlobalExceptionFilter())
```

---

## 📝 Correlation ID Interceptor

```typescript
// src/shared/interceptors/correlation-id.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common'
import { v4 as uuidv4 } from 'uuid'

@Injectable()
export class CorrelationIdInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest()

    // Get correlation ID from header or generate new one
    const correlationId =
      request.headers['x-correlation-id'] || uuidv4()

    // Attach to request (available in services)
    request.correlationId = correlationId

    // Add to response headers
    const response = context.switchToHttp().getResponse()
    response.setHeader('x-correlation-id', correlationId)

    return next.handle()
  }
}

// Apply globally in main.ts
app.useGlobalInterceptors(new CorrelationIdInterceptor())
```

**Use in logs:** Track requests across multiple services

---

## 🗄️ Database Setup (TypeORM)

```typescript
// src/database/database.module.ts
import { Module } from '@nestjs/common'
import { TypeOrmModule } from '@nestjs/typeorm'
import { ConfigService } from '@nestjs/config'

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        url: configService.get<string>('database.url'),
        entities: [__dirname + '/../**/*.entity{.ts,.js}'],
        migrations: [__dirname + '/migrations/*{.ts,.js}'],
        synchronize: false,  // NEVER true in production
        logging: configService.get('env') === 'development',
        poolSize: 10,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class DatabaseModule {}
```

**Always use migrations** (never `synchronize: true` in production)

---

## 🚀 Bootstrap (main.ts)

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core'
import { ValidationPipe } from '@nestjs/common'
import { AppModule } from './app.module'
import { GlobalExceptionFilter } from './shared/filters/http-exception.filter'
import { CorrelationIdInterceptor } from './shared/interceptors/correlation-id.interceptor'

async function bootstrap() {
  const app = await NestFactory.create(AppModule)

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,  // Strip unknown properties
      forbidNonWhitelisted: true,  // Throw error on unknown properties
      transform: true,  // Auto-transform to DTO types
    })
  )

  // Global filters and interceptors
  app.useGlobalFilters(new GlobalExceptionFilter())
  app.useGlobalInterceptors(new CorrelationIdInterceptor())

  // CORS
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
    credentials: true,
  })

  // Graceful shutdown
  app.enableShutdownHooks()

  const port = process.env.PORT || 3000
  await app.listen(port)
  console.log(`Service running on port ${port}`)
}

bootstrap()
```

---

## ✅ Implementation Checklist

Every service MUST have:

**Structure:**
- [ ] Standard folder structure (config, modules, shared, database)
- [ ] Configuration with Joi validation
- [ ] Environment variables loaded and validated

**Health & Observability:**
- [ ] Health check endpoints (liveness, readiness)
- [ ] Global exception filter (structured errors)
- [ ] Correlation ID interceptor (distributed tracing)
- [ ] Structured logging (see OBSERVABILITY.md)

**Data & Messaging:**
- [ ] Database connection with TypeORM
- [ ] Migrations folder set up
- [ ] RabbitMQ connection configured

**Production-Ready:**
- [ ] Graceful shutdown enabled
- [ ] Global validation pipe
- [ ] CORS configured
- [ ] Docker and docker-compose files
- [ ] README with setup instructions

---

## 🔗 Related Patterns

→ [OBSERVABILITY.md](./OBSERVABILITY.md) - Logging, metrics, tracing
→ [AUTHENTICATION_STRATEGY.md](./AUTHENTICATION_STRATEGY.md) - Auth guards
→ [MESSAGING_STRATEGY.md](./MESSAGING_STRATEGY.md) - RabbitMQ setup

---

## 📦 Standard Dependencies

```json
{
  "dependencies": {
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/config": "^3.0.0",
    "@nestjs/typeorm": "^10.0.0",
    "@nestjs/microservices": "^10.0.0",
    "@nestjs/terminus": "^10.0.0",
    "typeorm": "^0.3.0",
    "pg": "^8.11.0",
    "joi": "^17.9.0",
    "class-validator": "^0.14.0",
    "class-transformer": "^0.5.1"
  }
}
```

---

## 🎓 How AI Uses This Pattern

When you request: **"Create a new {service-name} service"**

**AI will:**
1. Read this pattern
2. Generate exact folder structure
3. Create configuration with Joi validation
4. Add health check endpoints
5. Set up global exception filter
6. Add correlation ID interceptor
7. Configure database with TypeORM
8. Create main.ts with all middleware
9. Add Docker files
10. Include all checklist items

**Result:** Production-ready service foundation in 30 minutes

**Without this pattern:** 2-3 days of setup, likely missing critical pieces

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09
**Status:** Active

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
