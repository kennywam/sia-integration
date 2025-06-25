# Monitoring and Observability Practices

## Overview
This document discusses the monitoring and observability practices for the application, detailing how to track performance and errors.

## Core Components

### 1. Metrics Collection

```typescript
// src/monitoring/metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Counter, Histogram } from 'prom-client';

@Injectable()
export class MetricsService {
  // Request metrics
  private requestCounter = new Counter({
    name: 'requests_total',
    help: 'Total number of requests',
    labelNames: ['method', 'path', 'status'],
  });

  private requestDuration = new Histogram({
    name: 'request_duration_seconds',
    help: 'Request duration in seconds',
    labelNames: ['method', 'path'],
    buckets: [0.1, 0.3, 0.5, 1, 2, 5],
  });

  // AI metrics
  private aiQueryCounter = new Counter({
    name: 'ai_queries_total',
    help: 'Total number of AI queries',
    labelNames: ['type', 'status'],
  });

  private aiQueryDuration = new Histogram({
    name: 'ai_query_duration_seconds',
    help: 'Duration of AI queries in seconds',
    labelNames: ['type'],
    buckets: [0.1, 0.3, 0.5, 1, 2, 5],
  });

  // Error metrics
  private errorCounter = new Counter({
    name: 'errors_total',
    help: 'Total number of errors',
    labelNames: ['type', 'source'],
  });

  recordRequest(
    method: string,
    path: string,
    status: number,
    duration: number,
  ) {
    this.requestCounter.labels(method, path, status.toString()).inc();
    this.requestDuration.labels(method, path).observe(duration);
  }

  recordAIQuery(
    type: string,
    status: 'success' | 'failure',
    duration: number,
  ) {
    this.aiQueryCounter.labels(type, status).inc();
    this.aiQueryDuration.labels(type).observe(duration);
  }

  recordError(
    type: string,
    source: string,
  ) {
    this.errorCounter.labels(type, source).inc();
  }
}
```

### 2. Logging Service

```typescript
// src/monitoring/logging.service.ts
import { Injectable } from '@nestjs/common';
import { createLogger, format, transports } from 'winston';

@Injectable()
export class LoggingService {
  private readonly logger = createLogger({
    level: 'info',
    format: format.combine(
      format.timestamp(),
      format.json(),
    ),
    transports: [
      new transports.Console(),
      new transports.File({ filename: 'logs/app.log' }),
    ],
  });

  log(message: string, meta?: any) {
    this.logger.info(message, meta);
  }

  error(message: string, error: Error, meta?: any) {
    this.logger.error(message, {
      ...meta,
      error: {
        message: error.message,
        stack: error.stack,
      },
    });
  }

  warn(message: string, meta?: any) {
    this.logger.warn(message, meta);
  }

  debug(message: string, meta?: any) {
    this.logger.debug(message, meta);
  }
}
```

## Monitoring Setup

```yaml
# k8s/monitoring.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nia-assistant-monitor
spec:
  selector:
    matchLabels:
      app: nia-assistant
  endpoints:
  - port: http
    interval: 15s
    path: /metrics
```

## Best Practices

1. **Metrics Collection**
   - Track request latency and success rates
   - Monitor AI query performance
   - Track error rates and types

2. **Logging**
   - Implement structured logging
   - Use proper log levels
   - Include context in logs

3. **Alerting**
   - Set up critical alerts
   - Define alert thresholds
   - Implement alert suppression

4. **Performance**
   - Monitor resource usage
   - Track response times
   - Set up capacity planning

5. **Security**
   - Monitor authentication attempts
   - Track access patterns
   - Detect suspicious activity

## Common Issues and Solutions

1. **Performance**
   - Solution: Implement proper metrics
   - Solution: Set up alerting
   - Solution: Regular log analysis

2. **Security**
   - Solution: Monitor access patterns
   - Solution: Implement rate limiting
   - Solution: Use proper authentication

3. **Monitoring**
   - Solution: Set up proper metrics
   - Solution: Implement alerting
   - Solution: Regular log analysis

4. **Alerting**
   - Solution: Set up proper thresholds
   - Solution: Implement alert suppression
   - Solution: Regular alert review

5. **Maintenance**
   - Solution: Regular log rotation
   - Solution: Proper backup procedures
   - Solution: Regular testing
