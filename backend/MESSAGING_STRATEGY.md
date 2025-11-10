<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Messaging Strategy
## Reliable Event-Driven Communication with RabbitMQ

> **For AI:** This pattern defines inter-service communication using RabbitMQ for reliability and scalability.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Quick Reference

**Message Broker:** RabbitMQ (AMQP protocol)

**Exchange Type:** Topic (flexible routing)

**Reliability Patterns:**
- **Outbox Pattern** - Ensures events are published (transactional)
- **Inbox Pattern** - Ensures idempotent event processing
- **Dead Letter Queues (DLQ)** - Handles failed messages

**Security:** One RabbitMQ user per service (isolation)

**Credentials:** Stored in Vault (`vault/services/{service-name}/rabbitmq`)

---

## 🏗️ Architecture Overview

```
[Registration Service]
        ↓ (publishes)
    user.registered
        ↓
   [RabbitMQ Topic Exchange]
        ↓ (routes to)
    ├─ mail-service queue
    ├─ analytics-service queue
    └─ notification-service queue
        ↓ (consumes)
   [Mail Service] sends welcome email
```

**Key benefit:** Services are decoupled. Registration service doesn't know who consumes events.

---

## 📨 Event Schema Standard

All events follow this structure:

```typescript
interface DomainEvent {
  eventId: string           // UUID (for idempotency)
  eventType: string         // e.g., "user.registered"
  version: string           // e.g., "1.0"
  timestamp: string         // ISO 8601
  payload: any              // Event-specific data
  metadata: {
    correlationId: string   // For distributed tracing
    userId?: string         // Who triggered (if applicable)
    source: string          // Which service published
  }
}
```

**Example event:**

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "user.registered",
  "version": "1.0",
  "timestamp": "2025-01-09T10:30:00Z",
  "payload": {
    "userId": "user-123",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "metadata": {
    "correlationId": "req-abc-123",
    "userId": "user-123",
    "source": "registration-service"
  }
}
```

---

## 🚀 Implementation Guide

### Step 1: RabbitMQ Connection Setup

```typescript
// src/config/configuration.ts
export default () => ({
  rabbitmq: {
    url: process.env.RABBITMQ_URL || 'amqp://localhost:5672',
    user: process.env.RABBITMQ_USER,      // From Vault
    password: process.env.RABBITMQ_PASS,  // From Vault
    exchange: 'domain_events',
    exchangeType: 'topic',
  }
})

// src/app.module.ts
import { Module } from '@nestjs/common'
import { ClientsModule, Transport } from '@nestjs/microservices'
import { ConfigModule, ConfigService } from '@nestjs/config'

@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        name: 'RABBITMQ_CLIENT',
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => ({
          transport: Transport.RMQ,
          options: {
            urls: [configService.get<string>('rabbitmq.url')],
            queue: `${process.env.SERVICE_NAME}_queue`,
            queueOptions: {
              durable: true,  // Survives broker restart
            },
            noAck: false,  // Manual acknowledgment for reliability
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

---

### Step 2: Outbox Pattern (Reliable Publishing)

**Problem:** What if service crashes after saving data but before publishing event?

**Solution:** Store events in database, then publish asynchronously.

```typescript
// src/outbox/outbox.entity.ts
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm'

@Entity('outbox')
export class OutboxMessage {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column()
  eventType: string

  @Column('jsonb')
  payload: any

  @Column({ default: false })
  published: boolean

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  createdAt: Date

  @Column({ type: 'timestamp', nullable: true })
  publishedAt: Date
}

// src/outbox/outbox.service.ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { Repository } from 'typeorm'
import { Cron, CronExpression } from '@nestjs/schedule'
import { ClientProxy } from '@nestjs/microservices'
import { Inject } from '@nestjs/common'

@Injectable()
export class OutboxService {
  constructor(
    @InjectRepository(OutboxMessage)
    private outboxRepo: Repository<OutboxMessage>,
    @Inject('RABBITMQ_CLIENT')
    private rabbitClient: ClientProxy,
  ) {}

  // Called within a database transaction
  async saveEvent(eventType: string, payload: any): Promise<void> {
    await this.outboxRepo.save({
      eventType,
      payload,
      published: false,
    })
  }

  // Background job: Publish unpublished events
  @Cron(CronExpression.EVERY_5_SECONDS)
  async processOutbox(): Promise<void> {
    const pending = await this.outboxRepo.find({
      where: { published: false },
      take: 100,
      order: { createdAt: 'ASC' },
    })

    for (const message of pending) {
      try {
        // Publish to RabbitMQ
        await this.rabbitClient.emit(message.eventType, message.payload).toPromise()

        // Mark as published
        message.published = true
        message.publishedAt = new Date()
        await this.outboxRepo.save(message)
      } catch (error) {
        console.error('Failed to publish event:', error)
        // Will retry on next cron run
      }
    }
  }
}
```

**Usage in service:**

```typescript
// src/users/users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User) private userRepo: Repository<User>,
    private outboxService: OutboxService,
  ) {}

  async registerUser(dto: RegisterDto): Promise<User> {
    // Use database transaction
    return await this.userRepo.manager.transaction(async (manager) => {
      // 1. Save user
      const user = manager.create(User, dto)
      await manager.save(user)

      // 2. Save event to outbox (same transaction)
      await manager.save(OutboxMessage, {
        eventType: 'user.registered',
        payload: {
          eventId: uuidv4(),
          eventType: 'user.registered',
          version: '1.0',
          timestamp: new Date().toISOString(),
          payload: { userId: user.id, email: user.email },
          metadata: { source: 'registration-service' },
        },
      })

      return user
    })
    // Event will be published by OutboxService background job
  }
}
```

**Result:** Atomic - either BOTH user and event are saved, or NEITHER.

---

### Step 3: Inbox Pattern (Idempotent Consumption)

**Problem:** What if same event is received twice? (Network retries, etc.)

**Solution:** Track processed event IDs in database.

```typescript
// src/inbox/inbox.entity.ts
@Entity('inbox')
export class InboxMessage {
  @PrimaryColumn()
  eventId: string  // From event.eventId

  @Column()
  eventType: string

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  processedAt: Date
}

// src/inbox/inbox.service.ts
@Injectable()
export class InboxService {
  constructor(
    @InjectRepository(InboxMessage)
    private inboxRepo: Repository<InboxMessage>,
  ) {}

  async isProcessed(eventId: string): Promise<boolean> {
    const existing = await this.inboxRepo.findOne({ where: { eventId } })
    return !!existing
  }

  async markProcessed(eventId: string, eventType: string): Promise<void> {
    await this.inboxRepo.save({ eventId, eventType })
  }
}
```

**Usage in consumer:**

```typescript
// src/events/user-registered.handler.ts
import { EventPattern } from '@nestjs/microservices'
import { Controller } from '@nestjs/common'

@Controller()
export class UserEventsController {
  constructor(
    private inboxService: InboxService,
    private mailService: MailService,
  ) {}

  @EventPattern('user.registered')
  async handleUserRegistered(event: DomainEvent): Promise<void> {
    // 1. Check if already processed (idempotency)
    if (await this.inboxService.isProcessed(event.eventId)) {
      console.log('Event already processed, skipping:', event.eventId)
      return
    }

    try {
      // 2. Process event
      await this.mailService.sendWelcomeEmail({
        to: event.payload.email,
        name: event.payload.name,
      })

      // 3. Mark as processed
      await this.inboxService.markProcessed(event.eventId, event.eventType)
    } catch (error) {
      // Don't mark as processed - will retry
      throw error
    }
  }
}
```

**Result:** Same event received 10 times = processed ONCE.

---

### Step 4: Dead Letter Queue (DLQ)

Handle messages that fail repeatedly:

```typescript
// src/config/rabbitmq.config.ts
export const rabbitMQConfig = {
  queueOptions: {
    durable: true,
    deadLetterExchange: 'dlx',
    deadLetterRoutingKey: 'failed',
    messageTtl: 60000,  // Retry for 1 minute
    arguments: {
      'x-max-retries': 3,  // After 3 failures → DLQ
    },
  },
}
```

**DLQ handler:**

```typescript
@EventPattern('dlq.failed')
async handleFailedMessage(message: any): Promise<void> {
  // Log to monitoring system
  console.error('Message failed after retries:', message)

  // Send alert to ops team
  await this.alertService.sendAlert({
    severity: 'high',
    message: `Failed to process event: ${message.eventType}`,
    details: message,
  })

  // Optionally: Store for manual processing
  await this.failedEventsRepo.save(message)
}
```

---

## ✅ Implementation Checklist

When adding messaging to a service:

- [ ] Get RabbitMQ credentials from Vault
- [ ] Configure ClientsModule with RabbitMQ transport
- [ ] Create outbox table and OutboxService
- [ ] Create inbox table and InboxService
- [ ] Implement background job to process outbox
- [ ] Add event handlers with @EventPattern decorator
- [ ] Use inbox pattern for idempotency in handlers
- [ ] Configure dead letter queue
- [ ] Add error handling and retry logic
- [ ] Include correlation ID in all events
- [ ] Add integration tests for event flow
- [ ] Set up monitoring for message queue depth

---

## 🔗 Related Patterns

→ [AUTHENTICATION_STRATEGY.md](./AUTHENTICATION_STRATEGY.md) - Include userId in event metadata
→ [OBSERVABILITY.md](./OBSERVABILITY.md) - Logging events and correlation IDs
→ [UNIVERSAL_PATTERNS.md](./UNIVERSAL_PATTERNS.md) - Standard service structure

---

## 🎯 Routing Patterns

**Topic Exchange routing keys:**

```
{service}.{entity}.{action}

Examples:
- user.registered              → registration-service publishes
- user.email.verified          → mail-service publishes
- video.processing.completed   → video-service publishes
- payment.succeeded            → payment-service publishes
```

**Queue binding examples:**

```typescript
// Mail service binds to user events:
queue: 'mail-service_queue'
bindingKeys: ['user.*', '*.email.*']

// Analytics service binds to everything:
queue: 'analytics-service_queue'
bindingKeys: ['#']  // Wildcard - all events

// Notification service binds to specific events:
queue: 'notification-service_queue'
bindingKeys: ['user.registered', 'video.*.completed', 'payment.succeeded']
```

---

## 📊 Event Versioning

As events evolve:

```typescript
// v1.0
{
  eventType: "user.registered",
  version: "1.0",
  payload: { userId, email }
}

// v2.0 - Added name field
{
  eventType: "user.registered",
  version: "2.0",
  payload: { userId, email, name }
}
```

**Consumers handle both versions:**

```typescript
@EventPattern('user.registered')
async handleUserRegistered(event: DomainEvent): Promise<void> {
  if (event.version === '1.0') {
    // Handle v1.0 (no name field)
  } else if (event.version === '2.0') {
    // Handle v2.0 (with name field)
  }
}
```

---

## 🎓 How AI Uses This Pattern

When you request: **"When a user registers, send a welcome email"**

**AI will:**
1. Read this pattern
2. Identify: registration-service publishes → mail-service consumes
3. In registration-service: Implement outbox pattern for "user.registered" event
4. In mail-service: Create event handler with inbox pattern
5. Configure DLQ for failures
6. Add correlation IDs for tracing
7. Include all checklist items

**Result:** Reliable, idempotent event flow in 30-45 minutes

**Without this pattern:** 4-6 hours, likely missing reliability patterns

---

## 📝 Notes

- This is a **demo pattern** - simplified for demonstration
- Full production pattern includes: saga patterns, event sourcing, CQRS
- See enterprise package for advanced messaging architectures

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09
**Status:** Active

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
