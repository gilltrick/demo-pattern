<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Observability
## Production Visibility & Debugging

> **For AI:** Structured logging, metrics, health checks, and distributed tracing for production systems.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Quick Reference

**Pillars of Observability:**
1. **Logs** - What happened (structured JSON logs)
2. **Metrics** - How much/how fast (Prometheus)
3. **Traces** - Request flow across services (correlation IDs)
4. **Health** - Is the service healthy (liveness/readiness)

**Tools:**
- Winston (structured logging)
- Prometheus (metrics)
- Correlation IDs (distributed tracing)

---

## 📝 Structured Logging with Winston

```typescript
// src/shared/logging/logger.service.ts
import { Injectable, LoggerService } from '@nestjs/common'
import * as winston from 'winston'

@Injectable()
export class CustomLogger implements LoggerService {
  private logger: winston.Logger

  constructor() {
    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json(),  // Structured logs
      ),
      transports: [
        new winston.transports.Console({
          format: winston.format.combine(
            winston.format.colorize(),
            winston.format.printf(({ timestamp, level, message, ...meta }) => {
              return `${timestamp} [${level}]: ${message} ${JSON.stringify(meta)}`
            }),
          ),
        }),
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error',
        }),
        new winston.transports.File({
          filename: 'logs/combined.log',
        }),
      ],
    })
  }

  log(message: string, context?: Record<string, any>) {
    this.logger.info(message, { context })
  }

  error(message: string, trace?: string, context?: Record<string, any>) {
    this.logger.error(message, { trace, context })
  }

  warn(message: string, context?: Record<string, any>) {
    this.logger.warn(message, { context })
  }

  debug(message: string, context?: Record<string, any>) {
    this.logger.debug(message, { context })
  }
}
```

**Structured log example:**

```json
{
  "timestamp": "2025-01-09T10:30:00.000Z",
  "level": "info",
  "message": "User registered successfully",
  "context": {
    "userId": "user-123",
    "email": "user@example.com",
    "correlationId": "req-abc-123",
    "duration": 250
  }
}
```

**Benefits:**
- Searchable (grep, Elasticsearch)
- Machine-readable
- Includes context automatically

---

## 🔍 Correlation IDs (Distributed Tracing)

Track requests across multiple services:

```typescript
// src/shared/interceptors/logging.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common'
import { Observable } from 'rxjs'
import { tap } from 'rxjs/operators'

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private logger: CustomLogger) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest()
    const { method, url, correlationId } = request

    const start = Date.now()

    this.logger.log('Incoming request', {
      method,
      url,
      correlationId,
    })

    return next.handle().pipe(
      tap({
        next: () => {
          const duration = Date.now() - start
          this.logger.log('Request completed', {
            method,
            url,
            correlationId,
            duration,
            status: 'success',
          })
        },
        error: (error) => {
          const duration = Date.now() - start
          this.logger.error('Request failed', error.stack, {
            method,
            url,
            correlationId,
            duration,
            error: error.message,
          })
        },
      })
    )
  }
}
```

**Usage across services:**

```
Client Request
  ↓ (generates correlation-id: req-abc-123)
Registration Service
  ↓ (publishes event with correlation-id)
Mail Service (correlation-id: req-abc-123)
  ↓ (logs with same ID)
All logs searchable by req-abc-123
```

---

## 📊 Metrics with Prometheus

```typescript
// src/shared/metrics/metrics.service.ts
import { Injectable } from '@nestjs/common'
import * as promClient from 'prom-client'

@Injectable()
export class MetricsService {
  private register: promClient.Registry

  // Counters
  private httpRequestsTotal: promClient.Counter
  private httpRequestDuration: promClient.Histogram

  // Gauges
  private activeConnections: promClient.Gauge

  constructor() {
    this.register = new promClient.Registry()

    // HTTP request counter
    this.httpRequestsTotal = new promClient.Counter({
      name: 'http_requests_total',
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'route', 'status_code'],
      registers: [this.register],
    })

    // HTTP request duration histogram
    this.httpRequestDuration = new promClient.Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status_code'],
      buckets: [0.1, 0.5, 1, 2, 5, 10],
      registers: [this.register],
    })

    // Active connections gauge
    this.activeConnections = new promClient.Gauge({
      name: 'active_connections',
      help: 'Number of active database connections',
      registers: [this.register],
    })

    // Default metrics (CPU, memory, etc.)
    promClient.collectDefaultMetrics({ register: this.register })
  }

  recordRequest(method: string, route: string, statusCode: number, duration: number) {
    this.httpRequestsTotal.inc({ method, route, status_code: statusCode })
    this.httpRequestDuration.observe({ method, route, status_code: statusCode }, duration / 1000)
  }

  setActiveConnections(count: number) {
    this.activeConnections.set(count)
  }

  async getMetrics(): Promise<string> {
    return this.register.metrics()
  }
}

// Expose metrics endpoint
@Controller('metrics')
export class MetricsController {
  constructor(private metricsService: MetricsService) {}

  @Get()
  async getMetrics(): Promise<string> {
    return this.metricsService.getMetrics()
  }
}
```

**Prometheus scrapes:** `GET /metrics` every 15 seconds

---

## 🏥 Health Checks

Already covered in UNIVERSAL_PATTERNS.md, but here's the complete implementation:

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private redis: RedisHealthIndicator,
  ) {}

  @Get('liveness')
  @HealthCheck()
  checkLiveness() {
    return this.health.check([])
  }

  @Get('readiness')
  @HealthCheck()
  checkReadiness() {
    return this.health.check([
      () => this.db.pingCheck('database', { timeout: 1000 }),
      () => this.redis.pingCheck('redis', { timeout: 1000 }),
    ])
  }
}
```

**Kubernetes configuration:**

```yaml
livenessProbe:
  httpGet:
    path: /health/liveness
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/readiness
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
```

---

## 📈 Application Performance Monitoring

Track custom business metrics:

```typescript
// Track video processing metrics
await this.metricsService.recordVideoProcessing({
  resolution: '1080p',
  duration: 120,  // seconds
  fileSize: 50 * 1024 * 1024,  // bytes
})

// Track API response times
const timer = this.metricsService.startTimer()
await this.doWork()
timer.end({ operation: 'user.register' })
```

---

## ✅ Implementation Checklist

- [ ] Install Winston for structured logging
- [ ] Create custom logger service
- [ ] Add logging interceptor with correlation IDs
- [ ] Implement correlation ID propagation in events
- [ ] Set up Prometheus client
- [ ] Create metrics service with counters/histograms
- [ ] Expose /metrics endpoint
- [ ] Add health check endpoints
- [ ] Configure log levels per environment
- [ ] Set up log rotation (avoid disk fill)
- [ ] Add alerting rules in Prometheus
- [ ] Dashboard in Grafana (optional)

---

## 🔗 Related Patterns

→ [UNIVERSAL_PATTERNS.md](./UNIVERSAL_PATTERNS.md) - Correlation ID interceptor
→ [MESSAGING_STRATEGY.md](./MESSAGING_STRATEGY.md) - Propagate correlation IDs in events

---

## 🎓 How AI Uses This Pattern

When you request: **"Add observability to the service"**

**AI will:**
1. Install Winston and Prometheus client
2. Create CustomLogger with structured logging
3. Add LoggingInterceptor for all requests
4. Create MetricsService with standard metrics
5. Expose /metrics endpoint
6. Add correlation ID to all logs and events
7. Configure health checks

**Result:** Full observability in 20-30 minutes

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
