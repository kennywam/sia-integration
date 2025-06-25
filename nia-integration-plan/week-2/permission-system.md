# Permission System for Sia

## Overview

The permission system is a critical component of the Sia platform, ensuring secure and controlled access to resources across multi-tenant environments. This document outlines the design and implementation of a robust permission system.

## Permission Models

### 1. Role-Based Access Control (RBAC)

- **Roles**: Define sets of permissions (e.g., Admin, Manager, User)
- **Permissions**: Granular actions (e.g., read:document, write:document)
- **Assignment**: Users are assigned roles which grant specific permissions

### 2. Attribute-Based Access Control (ABAC)

- **Attributes**: User, resource, and environment attributes
- **Policies**: Rules that evaluate attributes to make access decisions
- **Context-Aware**: Supports dynamic permission evaluation

### 3. Multi-Tenant Access Control

- **Tenant Isolation**: Ensures data separation between organizations
- **Cross-Tenant Access**: Controlled sharing between tenants when needed
- **Hierarchical Permissions**: Support for organizational hierarchies

## Implementation

### 1. Permission Definitions

```typescript
// src/auth/permissions.ts
export enum Permission {
  // Document permissions
  DOCUMENT_CREATE = 'document:create',
  DOCUMENT_READ = 'document:read',
  DOCUMENT_UPDATE = 'document:update',
  DOCUMENT_DELETE = 'document:delete',

  // User management
  USER_INVITE = 'user:invite',
  USER_DELETE = 'user:delete',

  // Admin permissions
  SYSTEM_CONFIG = 'system:config',
  AUDIT_LOGS_VIEW = 'audit:view',
}

export const RolePermissions = {
  ADMIN: [
    Permission.DOCUMENT_CREATE,
    Permission.DOCUMENT_READ,
    Permission.DOCUMENT_UPDATE,
    Permission.DOCUMENT_DELETE,
    Permission.USER_INVITE,
    Permission.USER_DELETE,
    Permission.SYSTEM_CONFIG,
    Permission.AUDIT_LOGS_VIEW,
  ],
  MANAGER: [
    Permission.DOCUMENT_CREATE,
    Permission.DOCUMENT_READ,
    Permission.DOCUMENT_UPDATE,
    Permission.USER_INVITE,
  ],
  MEMBER: [Permission.DOCUMENT_READ],
}
```

### 2. Permission Guard

```typescript
// src/auth/permission.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { Permission } from './permissions'
import { AuthService } from './auth.service'

@Injectable()
export class PermissionGuard implements CanActivate {
  constructor(private reflector: Reflector, private authService: AuthService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPermissions =
      this.reflector.get<Permission[]>('permissions', context.getHandler()) ||
      []

    if (!requiredPermissions.length) {
      return true
    }

    const request = context.switchToHttp().getRequest()
    const user = request.user

    // Check if user has all required permissions
    const hasPermission = await this.authService.hasPermissions(
      user.id,
      requiredPermissions,
      request.params.tenantId
    )

    if (!hasPermission) {
      throw new ForbiddenException('Insufficient permissions')
    }

    return true
  }
}
```

### 3. Role-Permission Service

```typescript
// src/auth/role-permission.service.ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { Repository } from 'typeorm'
import { UserRole } from '../entities/user-role.entity'
import { RolePermission } from '../entities/role-permission.entity'
import { Permission } from './permissions'

@Injectable()
export class RolePermissionService {
  constructor(
    @InjectRepository(UserRole)
    private userRoleRepository: Repository<UserRole>,
    @InjectRepository(RolePermission)
    private rolePermissionRepository: Repository<RolePermission>
  ) {}

  async userHasPermission(
    userId: string,
    permission: Permission,
    tenantId: string
  ): Promise<boolean> {
    // Get user's roles in the tenant
    const userRoles = await this.userRoleRepository.find({
      where: { userId, tenantId },
      relations: ['role'],
    })

    if (!userRoles.length) return false

    // Check if any role has the required permission
    const roleIds = userRoles.map((ur) => ur.roleId)
    const hasPermission = await this.rolePermissionRepository
      .createQueryBuilder('rp')
      .where('rp.roleId IN (:...roleIds)', { roleIds })
      .andWhere('rp.permission = :permission', { permission })
      .getExists()

    return hasPermission
  }
}
```

## Multi-Tenant Permission System

### 1. Tenant-Aware Middleware

```typescript
// src/tenant/tenant.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'
import { TenantService } from './tenant.service'

@Injectable()
export class TenantMiddleware implements NestMiddleware {
  constructor(private readonly tenantService: TenantService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const tenantId = this.extractTenantId(req)

    if (!tenantId) {
      return res.status(400).json({ message: 'Tenant ID is required' })
    }

    // Verify tenant exists and user has access
    const hasAccess = await this.tenantService.userHasAccess(
      req.user.id,
      tenantId
    )

    if (!hasAccess) {
      return res.status(403).json({ message: 'Access to tenant denied' })
    }

    // Attach tenant context to request
    req.tenant = {
      id: tenantId,
      // Add other tenant-specific context
    }

    next()
  }

  private extractTenantId(req: Request): string | null {
    // Extract from various sources (header, query param, JWT, etc.)
    return (req.headers['x-tenant-id'] ||
      req.query.tenantId ||
      req.user?.tenantId ||
      null) as string
  }
}
```

### 2. Resource-Level Permissions

```typescript
// src/auth/resource-ownership.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { ResourceService } from '../resource/resource.service'

@Injectable()
export class ResourceOwnershipGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private resourceService: ResourceService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest()
    const resourceId = request.params.id
    const userId = request.user.id
    const tenantId = request.tenant?.id

    if (!resourceId || !tenantId) {
      return false
    }

    const isOwner = await this.resourceService.isOwner(
      resourceId,
      userId,
      tenantId
    )

    if (!isOwner) {
      throw new ForbiddenException('You do not own this resource')
    }

    return true
  }
}
```

## Permission Decorators

### 1. @RequirePermissions

```typescript
// src/auth/require-permissions.decorator.ts
import { SetMetadata } from '@nestjs/common'
import { Permission } from './permissions'

export const REQUIRE_PERMISSIONS_KEY = 'require-permissions'

export const RequirePermissions = (...permissions: Permission[]) =>
  SetMetadata(REQUIRE_PERMISSIONS_KEY, permissions)
```

### 2. @Public

```typescript
// src/auth/public.decorator.ts
import { SetMetadata } from '@nestjs/common'

export const IS_PUBLIC_KEY = 'isPublic'
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true)
```

## Usage in Controllers

```typescript
// Example controller with permission checks
@Controller('documents')
@UseGuards(JwtAuthGuard, TenantMiddleware, PermissionGuard)
export class DocumentController {
  constructor(private readonly documentService: DocumentService) {}

  @Get()
  @RequirePermissions(Permission.DOCUMENT_READ)
  async findAll(@Request() req) {
    return this.documentService.findAll(req.tenant.id)
  }

  @Post()
  @RequirePermissions(Permission.DOCUMENT_CREATE)
  async create(@Body() createDocumentDto: CreateDocumentDto, @Request() req) {
    return this.documentService.create(
      createDocumentDto,
      req.user.id,
      req.tenant.id
    )
  }

  @Put(':id')
  @UseGuards(ResourceOwnershipGuard)
  @RequirePermissions(Permission.DOCUMENT_UPDATE)
  async update(
    @Param('id') id: string,
    @Body() updateDocumentDto: UpdateDocumentDto,
    @Request() req
  ) {
    return this.documentService.update(id, updateDocumentDto, req.tenant.id)
  }
}
```

## Testing the Permission System

### 1. Unit Tests

```typescript
describe('PermissionGuard', () => {
  let guard: PermissionGuard
  let reflector: Reflector
  let authService: AuthService

  beforeEach(() => {
    reflector = new Reflector()
    authService = {
      hasPermissions: jest.fn().mockResolvedValue(true),
    } as any
    guard = new PermissionGuard(reflector, authService)
  })

  it('should allow access when no permissions required', async () => {
    const context = createMockContext([])
    await expect(guard.canActivate(context)).resolves.toBe(true)
  })

  it('should check permissions when required', async () => {
    const context = createMockContext([Permission.DOCUMENT_READ])
    await guard.canActivate(context)
    expect(authService.hasPermissions).toHaveBeenCalled()
  })
})
```

## Performance Considerations

### 1. Caching

- Cache user permissions to reduce database load
- Use Redis for distributed caching in microservices
- Invalidate cache on permission changes

### 2. Optimized Queries

- Use efficient joins when checking permissions
- Consider materialized views for complex permission checks
- Batch permission checks when possible

## Security Best Practices

1. **Principle of Least Privilege**

   - Grant minimum permissions required
   - Regularly audit and clean up unused permissions

2. **Permission Review**

   - Regular security reviews
   - Automated permission validation

3. **Logging and Auditing**
   - Log all permission checks
   - Maintain audit trails for permission changes

## Deployment and Maintenance

### 1. Database Migrations

```typescript
// Example migration for permission tables
export class CreatePermissionTables1712345678901 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE roles (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL UNIQUE,
        description TEXT,
        createdAt TIMESTAMP DEFAULT NOW(),
        updatedAt TIMESTAMP DEFAULT NOW()
      );

      CREATE TABLE permissions (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL UNIQUE,
        description TEXT,
        createdAt TIMESTAMP DEFAULT NOW()
      );

      CREATE TABLE role_permissions (
        roleId INTEGER REFERENCES roles(id) ON DELETE CASCADE,
        permissionId INTEGER REFERENCES permissions(id) ON DELETE CASCADE,
        PRIMARY KEY (roleId, permissionId)
      );
    `)
  }
}
```

### 2. Permission Management API

```typescript
@Controller('admin/permissions')
@UseGuards(JwtAuthGuard, AdminGuard)
export class PermissionAdminController {
  constructor(private readonly permissionService: PermissionService) {}

  @Get('roles')
  async getRoles() {
    return this.permissionService.getAllRoles()
  }

  @Post('roles')
  async createRole(@Body() createRoleDto: CreateRoleDto) {
    return this.permissionService.createRole(createRoleDto)
  }

  @Post('roles/:roleId/permissions')
  async addPermissionToRole(
    @Param('roleId') roleId: number,
    @Body() { permissionId }: { permissionId: number }
  ) {
    return this.permissionService.addPermissionToRole(roleId, permissionId)
  }
}
```

## Benefits

1. **Efficiency**: Fast retrieval times for high-dimensional data.
2. **Scalability**: Capable of handling large datasets.
3. **Flexibility**: Supports various data types and structures.

---

### week-1/prompt-engineering.md

# Prompt Engineering for AI Models

## Introduction

Prompt engineering is the process of designing and refining prompts to elicit desired responses from AI models. This document discusses techniques for crafting effective prompts.

## Techniques

1. **Clarity**: Ensure prompts are clear and unambiguous.
2. **Context**: Provide sufficient context to guide the model's response.
3. **Examples**: Use examples to illustrate the desired output format.

## Importance

Effective prompt design is crucial for maximizing the performance of AI models and achieving accurate results.

---

### week-1/hands-on-chatbot-demo.md

# Hands-on Chatbot Demo

## Introduction

This document provides a practical guide for building a simple chatbot using the concepts learned in the first week.

## Setup Instructions

1. **Environment Setup**: Ensure you have Node.js and npm installed.
2. **Project Initialization**:
   ```bash
   mkdir chatbot-demo
   cd chatbot-demo
   npm init -y
   npm install express body-parser
   ```

## Code Snippets

```javascript
const express = require('express')
const bodyParser = require('body-parser')

const app = express()
app.use(bodyParser.json())

app.post('/chat', (req, res) => {
  const userMessage = req.body.message
  // Process the message and generate a response
  res.json({ response: 'This is a response from the chatbot.' })
})

app.listen(3000, () => {
  console.log('Chatbot is running on port 3000')
})
```
