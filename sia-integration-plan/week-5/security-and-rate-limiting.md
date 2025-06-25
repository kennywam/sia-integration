### week-5/Security Measures and Rate Limiting Strategies

## Overview

This document covers security measures and rate limiting strategies to protect the application from abuse and unauthorized access.

## Core Components

### 1. Authentication Guard

```typescript
// src/auth/guards/auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { JwtService } from '@nestjs/jwt'

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly jwtService: JwtService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest()
    const token = this.extractTokenFromHeader(request)

    if (!token) {
      return false
    }

    try {
      const payload = await this.jwtService.verifyAsync(token)
      request.user = payload
      return true
    } catch {
      return false
    }
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? []
    return type === 'Bearer' ? token : undefined
  }
}
```

### 2. Rate Limiting Middleware

```typescript
// src/common/middleware/rate-limit.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'
import rateLimit from 'express-rate-limit'
import { ConfigService } from '@nestjs/config'

@Injectable()
export class RateLimitMiddleware implements NestMiddleware {
  private readonly limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: 'Too many requests from this IP, please try again later.',
    standardHeaders: true,
    legacyHeaders: false,
  })

  use(req: Request, res: Response, next: NextFunction) {
    return this.limiter(req, res, next)
  }
}
```

### 3. Security Headers Middleware

```typescript
// src/common/middleware/security-headers.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'

@Injectable()
export class SecurityHeadersMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    res.header('Content-Security-Policy', "default-src 'self';")
    res.header('X-Content-Type-Options', 'nosniff')
    res.header('X-Frame-Options', 'DENY')
    res.header('X-XSS-Protection', '1; mode=block')
    res.header('Referrer-Policy', 'strict-origin-when-cross-origin')
    res.header(
      'Strict-Transport-Security',
      'max-age=31536000; includeSubDomains'
    )
    next()
  }
}
```

## Best Practices

1. **Authentication**

   - Use JWT tokens
   - Implement proper token validation
   - Use secure token storage

2. **Rate Limiting**

   - Set appropriate limits
   - Monitor usage patterns
   - Implement proper error handling

3. **Security Headers**

   - Use proper CSP policies
   - Implement XSS protection
   - Use secure transport

4. **API Security**

   - Validate all inputs
   - Sanitize outputs
   - Use proper error messages

5. **Monitoring**
   - Track security events
   - Monitor rate limits
   - Alert on suspicious activity

## Common Issues and Solutions

1. **Authentication**

   - Solution: Implement proper token validation
   - Solution: Use secure storage
   - Solution: Add proper error handling

2. **Rate Limiting**

   - Solution: Set appropriate limits
   - Solution: Monitor usage patterns
   - Solution: Implement proper error handling

3. **Security**

   - Solution: Use proper headers
   - Solution: Implement XSS protection
   - Solution: Use secure transport

4. **API Security**

   - Solution: Validate inputs
   - Solution: Sanitize outputs
   - Solution: Use proper error messages

5. **Monitoring**
   - Solution: Track security events
   - Solution: Monitor rate limits
   - Solution: Alert on suspicious activity
