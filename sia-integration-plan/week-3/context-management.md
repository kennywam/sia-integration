# Context Management Implementation

## Overview

The context management system maintains user session state and conversation history, ensuring the AI assistant provides relevant and consistent responses.

## Core Components

### 1. Context Service

```typescript
// src/ai/services/context.service.ts
import { Injectable, Logger } from '@nestjs/common'
import { ConfigService } from '@nestjs/config'
import { PrismaService } from '../prisma/prisma.service'

export interface ConversationContext {
  userId: string
  tenantId: string
  sessionId: string
  messages: Array<{
    role: 'user' | 'assistant'
    content: string
    timestamp: Date
  }>
  metadata: Record<string, any>
}

@Injectable()
export class ContextService {
  private readonly logger = new Logger(ContextService.name)

  constructor(
    private configService: ConfigService,
    private prisma: PrismaService
  ) {}

  async createContext(userId: string, tenantId: string): Promise<string> {
    const session = await this.prisma.session.create({
      data: {
        userId,
        tenantId,
        createdAt: new Date(),
      },
    })

    return session.id
  }

  async addMessage(
    sessionId: string,
    role: 'user' | 'assistant',
    content: string
  ): Promise<void> {
    await this.prisma.message.create({
      data: {
        sessionId,
        role,
        content,
        timestamp: new Date(),
      },
    })
  }

  async getContext(sessionId: string): Promise<ConversationContext> {
    const session = await this.prisma.session.findUnique({
      where: { id: sessionId },
      include: {
        messages: {
          orderBy: { timestamp: 'asc' },
        },
      },
    })

    if (!session) {
      throw new Error('Session not found')
    }

    return {
      userId: session.userId,
      tenantId: session.tenantId,
      sessionId: session.id,
      messages: session.messages.map((m) => ({
        role: m.role,
        content: m.content,
        timestamp: m.timestamp,
      })),
      metadata: session.metadata || {},
    }
  }

  async updateMetadata(
    sessionId: string,
    metadata: Record<string, any>
  ): Promise<void> {
    await this.prisma.session.update({
      where: { id: sessionId },
      data: { metadata },
    })
  }
}
```

### 2. Context Middleware

```typescript
// src/ai/middleware/context.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'
import { ContextService } from '../services/context.service'
import { TenantId } from '../common/decorators/tenant-id.decorator'

@Injectable()
export class ContextMiddleware implements NestMiddleware {
  constructor(private contextService: ContextService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const sessionId = req.headers['x-session-id'] as string
    const userId = req.user?.id
    const tenantId = req.headers['x-tenant-id'] as string

    if (!sessionId) {
      // Create new session
      const newSessionId = await this.contextService.createContext(
        userId,
        tenantId
      )
      req.headers['x-session-id'] = newSessionId
    }

    next()
  }
}
```

### 3. Session Management

```typescript
// src/ai/services/session.service.ts
import { Injectable, Logger } from '@nestjs/common'
import { PrismaService } from '../prisma/prisma.service'

@Injectable()
export class SessionService {
  private readonly logger = new Logger(SessionService.name)

  constructor(private prisma: PrismaService) {}

  async expireOldSessions(hours: number = 24): Promise<void> {
    const cutoff = new Date(Date.now() - hours * 60 * 60 * 1000)

    await this.prisma.session.deleteMany({
      where: {
        createdAt: {
          lt: cutoff,
        },
      },
    })
  }

  async getSessionStats(): Promise<{
    activeSessions: number
    totalMessages: number
  }> {
    const [activeSessions, totalMessages] = await Promise.all([
      this.prisma.session.count(),
      this.prisma.message.count(),
    ])

    return {
      activeSessions,
      totalMessages,
    }
  }
}
```

## Database Schema

```prisma
// prisma/schema.prisma
model Session {
  id        String    @id @default(cuid())
  userId    String
  tenantId  String
  createdAt DateTime  @default(now())
  messages  Message[]
  metadata  Json?     @db.Json

  user User @relation(fields: [userId], references: [id])
  tenant Tenant @relation(fields: [tenantId], references: [id])

  @@index([userId])
  @@index([tenantId])
}

model Message {
  id        String    @id @default(cuid())
  sessionId String
  role      String
  content   String
  timestamp DateTime  @default(now())

  session Session @relation(fields: [sessionId], references: [id])

  @@index([sessionId])
  @@index([timestamp])
}
```

## Best Practices

1. **Session Management**

   - Implement session expiration
   - Track session metadata
   - Handle concurrent access

2. **Context Window**

   - Limit message history
   - Implement token management
   - Handle context overflow

3. **Security**

   - Implement proper authentication
   - Mask sensitive information
   - Isolate tenant data

4. **Performance**

   - Use proper indexing
   - Implement caching
   - Batch operations

5. **Monitoring**
   - Track session metrics
   - Monitor message volume
   - Log errors and exceptions

## Testing

```typescript
// test/ai/services/context.service.spec.ts
describe('ContextService', () => {
  let service: ContextService
  let mockPrisma: any

  beforeEach(async () => {
    mockPrisma = {
      session: {
        create: jest.fn(),
        findUnique: jest.fn(),
        update: jest.fn(),
      },
      message: {
        create: jest.fn(),
      },
    }

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ContextService,
        {
          provide: PrismaService,
          useValue: mockPrisma,
        },
      ],
    }).compile()

    service = module.get<ContextService>(ContextService)
  })

  it('should create context', async () => {
    const userId = 'user1'
    const tenantId = 'tenant1'
    const sessionId = 'session1'

    mockPrisma.session.create.mockResolvedValue({
      id: sessionId,
      userId,
      tenantId,
    })

    const result = await service.createContext(userId, tenantId)
    expect(result).toBe(sessionId)
  })

  it('should add message', async () => {
    const sessionId = 'session1'
    const role = 'user'
    const content = 'test message'

    await service.addMessage(sessionId, role, content)
    expect(mockPrisma.message.create).toHaveBeenCalled()
  })

  it('should get context', async () => {
    const sessionId = 'session1'
    const userId = 'user1'
    const tenantId = 'tenant1'

    mockPrisma.session.findUnique.mockResolvedValue({
      id: sessionId,
      userId,
      tenantId,
      messages: [],
    })

    const context = await service.getContext(sessionId)
    expect(context).toBeDefined()
  })
})
```

## Error Handling

```typescript
// src/ai/middleware/error-handler.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'
import { Logger } from '@nestjs/common'

@Injectable()
export class ErrorHandlerMiddleware implements NestMiddleware {
  private readonly logger = new Logger(ErrorHandlerMiddleware.name)

  use(req: Request, res: Response, next: NextFunction) {
    try {
      next()
    } catch (error) {
      this.logger.error(`Error in request: ${error.message}`, error.stack)

      res.status(500).json({
        error: 'Internal server error',
        message: error.message,
        timestamp: new Date().toISOString(),
      })
    }
  }
}
```

## Monitoring

```typescript
// src/ai/services/monitoring.service.ts
import { Injectable } from '@nestjs/common'
import { createLogger, format, transports } from 'winston'

@Injectable()
export class MonitoringService {
  private readonly logger = createLogger({
    level: 'info',
    format: format.combine(format.timestamp(), format.json()),
    transports: [
      new transports.Console(),
      new transports.File({ filename: 'logs/context.log' }),
    ],
  })

  async logContextEvent(
    sessionId: string,
    event: string,
    details: Record<string, any>
  ) {
    this.logger.info('Context Event', {
      sessionId,
      event,
      ...details,
      timestamp: new Date().toISOString(),
    })
  }

  async logError(
    sessionId: string,
    error: Error,
    context: Record<string, any>
  ) {
    this.logger.error('Context Error', {
      sessionId,
      errorMessage: error.message,
      stack: error.stack,
      ...context,
      timestamp: new Date().toISOString(),
    })
  }
}
```

## Best Practices

1. **Session Management**

   - Implement proper session expiration
   - Track session metadata
   - Handle concurrent access

2. **Context Window**

   - Limit message history
   - Implement token management
   - Handle context overflow

3. **Security**

   - Implement proper authentication
   - Mask sensitive information
   - Isolate tenant data

4. **Performance**

   - Use proper indexing
   - Implement caching
   - Batch operations

5. **Monitoring**
   - Track session metrics
   - Monitor message volume
   - Log errors and exceptions

## Common Issues and Solutions

1. **Context Overflow**

   - Solution: Implement token counting
   - Solution: Use sliding window
   - Solution: Summarize old messages

2. **Session Management**

   - Solution: Implement proper cleanup
   - Solution: Use session timeouts
   - Solution: Add session validation

3. **Performance**

   - Solution: Implement caching
   - Solution: Use batch operations
   - Solution: Add proper indexing

4. **Security**

   - Solution: Validate all inputs
   - Solution: Mask sensitive data
   - Solution: Implement proper isolation

5. **Error Handling**
   - Solution: Implement proper error logging
   - Solution: Add retry mechanisms
   - Solution: Provide meaningful error messages
