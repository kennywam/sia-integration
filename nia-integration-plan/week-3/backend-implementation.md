# Backend Implementation with NestJS

## Project Setup

### Prerequisites
- Node.js (v16+)
- npm or yarn
- NestJS CLI
- Docker (for database and other services)

### Initialize Project
```bash
# Install NestJS CLI globally
npm install -g @nestjs/cli

# Create new project
nest new nia-backend
cd nia-backend

# Install common dependencies
npm install @nestjs/config @nestjs/jwt @nestjs/passport @nestjs/typeorm typeorm pg
npm install class-validator class-transformer bcrypt
npm install --save-dev @types/bcrypt
```

## Project Structure

```
src/
├── auth/                    # Authentication module
│   ├── strategies/          # Authentication strategies
│   ├── guards/              # Authentication guards
│   └── decorators/          # Custom decorators
├── common/                  # Common utilities
│   ├── decorators/          # Shared decorators
│   ├── filters/             # Exception filters
│   ├── interceptors/        # Request/response interceptors
│   └── middleware/          # Global middleware
├── config/                  # Configuration
│   └── configuration.ts     # App configuration
├── database/                # Database module
│   └── migrations/          # Database migrations
├── documents/               # Document management
│   ├── entities/            # Document entities
│   ├── dto/                 # Data transfer objects
│   └── services/            # Document services
├── embeddings/              # Embedding services
│   ├── providers/           # Embedding providers
│   └── services/            # Embedding services
├── search/                  # Search functionality
│   ├── dto/                 # Search DTOs
│   └── services/            # Search services
├── tenants/                 # Multi-tenancy
│   ├── entities/            # Tenant entities
│   └── services/            # Tenant services
├── users/                   # User management
│   ├── entities/            # User entities
│   ├── dto/                 # User DTOs
│   └── services/            # User services
├── app.module.ts            # Root module
├── app.controller.ts        # Root controller
└── main.ts                  # Application entry point
```

## Core Modules

### 1. App Module

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AuthModule } from './auth/auth.module';
import { UsersModule } from './users/users.module';
import { DocumentsModule } from './documents/documents.module';
import { SearchModule } from './search/search.module';
import { TenantsModule } from './tenants/tenants.module';
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
    }),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('database.host'),
        port: +configService.get<number>('database.port'),
        username: configService.get('database.username'),
        password: configService.get('database.password'),
        database: configService.get('database.name'),
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize: configService.get('app.env') === 'development',
        logging: configService.get('app.env') === 'development',
      }),
      inject: [ConfigService],
    }),
    AuthModule,
    UsersModule,
    DocumentsModule,
    SearchModule,
    TenantsModule,
  ],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

### 2. Authentication Module

```typescript
// src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtStrategy } from './strategies/jwt.strategy';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get('jwt.secret'),
        signOptions: { 
          expiresIn: configService.get('jwt.expiresIn'),
        },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

## Database Integration

### 1. Database Module

```typescript
// src/database/database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { User } from '../users/entities/user.entity';
import { Tenant } from '../tenants/entities/tenant.entity';
import { Document } from '../documents/entities/document.entity';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('database.host'),
        port: +configService.get<number>('database.port'),
        username: configService.get('database.username'),
        password: configService.get('database.password'),
        database: configService.get('database.name'),
        entities: [User, Tenant, Document],
        synchronize: configService.get('app.env') === 'development',
      }),
      inject: [ConfigService],
    }),
  ],
})
export class DatabaseModule {}
```

### 2. Database Migrations

```bash
# Generate migration
typeorm migration:generate -n InitialSchema

# Run migrations
typeorm migration:run

# Revert last migration
typeorm migration:revert
```

## API Documentation

### 1. Swagger Setup

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Global validation pipe
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));

  // CORS
  app.enableCors();

  // Swagger documentation
  const config = new DocumentBuilder()
    .setTitle('NIA API')
    .setDescription('NestJS Implementation API documentation')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
bootstrap();
```

## Error Handling

### 1. Global Exception Filter

```typescript
// src/common/filters/all-exceptions.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const message =
      exception instanceof HttpException
        ? exception.getResponse()
        : 'Internal server error';

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
    });
  }
}
```

## Testing

### 1. Unit Test Example

```typescript
// src/auth/auth.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { AuthService } from './auth.service';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import { UnauthorizedException } from '@nestjs/common';

describe('AuthService', () => {
  let service: AuthService;
  let mockUsersService: Partial<UsersService>;
  let mockJwtService: Partial<JwtService>;

  beforeEach(async () => {
    mockUsersService = {
      findOneByEmail: jest.fn(),
    };

    mockJwtService = {
      sign: jest.fn().mockReturnValue('test-token'),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        {
          provide: UsersService,
          useValue: mockUsersService,
        },
        {
          provide: JwtService,
          useValue: mockJwtService,
        },
      ],
    }).compile();

    service = module.get<AuthService>(AuthService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('validateUser', () => {
    it('should return user if credentials are valid', async () => {
      const mockUser = {
        id: 1,
        email: 'test@example.com',
        password: 'hashed-password',
      };
      
      jest.spyOn(mockUsersService, 'findOneByEmail').mockResolvedValue(mockUser);
      jest.spyOn(service as any, 'comparePasswords').mockResolvedValue(true);

      const result = await service.validateUser('test@example.com', 'password');
      expect(result).toEqual({
        id: mockUser.id,
        email: mockUser.email,
      });
    });
  });
});
```

## Deployment

### 1. Docker Configuration

```dockerfile
# Dockerfile
FROM node:16-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy application code
COPY . .

# Build application
RUN npm run build

# Expose port
EXPOSE 3000

# Start the application
CMD ["node", "dist/main"]
```

### 2. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:password@db:5432/nia
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:13-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=nia
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - '5432:5432'
    restart: unless-stopped

volumes:
  postgres_data:
```

## Monitoring and Logging

### 1. Winston Logger

```typescript
// src/common/logger/winston.logger.ts
import { createLogger, format, transports } from 'winston';

const { combine, timestamp, printf } = format;

const logFormat = printf(({ level, message, timestamp, ...meta }) => {
  return `${timestamp} [${level.toUpperCase()}] ${message} ${
    Object.keys(meta).length ? JSON.stringify(meta, null, 2) : ''
  }`;
});

export const logger = createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: combine(
    timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    format.errors({ stack: true }),
    format.splat(),
    logFormat
  ),
  transports: [
    new transports.Console(),
    new transports.File({ filename: 'logs/error.log', level: 'error' }),
    new transports.File({ filename: 'logs/combined.log' }),
  ],
});
```

## Best Practices

1. **Code Organization**
   - Follow feature-based module organization
   - Keep modules small and focused
   - Use DTOs for request/response validation

2. **Error Handling**
   - Use custom exception filters
   - Implement proper error logging
   - Return meaningful error messages

3. **Security**
   - Use environment variables for sensitive data
   - Implement rate limiting
   - Validate all user input

4. **Performance**
   - Use caching where appropriate
   - Optimize database queries
   - Implement pagination for large datasets

5. **Testing**
   - Write unit tests for business logic
   - Implement integration tests for API endpoints
   - Use test factories for test data generation