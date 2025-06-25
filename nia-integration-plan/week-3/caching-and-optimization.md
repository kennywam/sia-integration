# Caching and Optimization Implementation

## Overview

This section covers various caching strategies and optimization techniques to improve the performance of the AI assistant system.

## Caching Strategies

### 1. Redis Cache Service

```typescript
// src/ai/services/cache.service.ts
import { Injectable, Logger } from '@nestjs/common';
import Redis from 'ioredis';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class CacheService {
  private readonly logger = new Logger(CacheService.name);
  private redis: Redis;

  constructor(private configService: ConfigService) {
    this.redis = new Redis({
      host: this.configService.get('REDIS_HOST'),
      port: this.configService.get('REDIS_PORT'),
    });
  }

  async get(key: string): Promise<string | null> {
    return this.redis.get(key);
  }

  async set(key: string, value: string, ttl?: number): Promise<void> {
    if (ttl) {
      await this.redis.setex(key, ttl, value);
    } else {
      await this.redis.set(key, value);
    }
  }

  async del(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async mget(keys: string[]): Promise<(string | null)[]> {
    return this.redis.mget(keys);
  }

  async mset(keyValuePairs: Record<string, string>, ttl?: number): Promise<void> {
    const keys = Object.keys(keyValuePairs);
    const values = Object.values(keyValuePairs);
    
    if (ttl) {
      await Promise.all(
        keys.map((key, index) => 
          this.redis.setex(key, ttl, values[index])
        )
      );
    } else {
      await this.redis.mset(keyValuePairs);
    }
  }

  async expire(key: string, seconds: number): Promise<void> {
    await this.redis.expire(key, seconds);
  }

  async keys(pattern: string): Promise<string[]> {
    return this.redis.keys(pattern);
  }
}
```

### 2. Query Cache Implementation

```typescript
// src/ai/services/query-cache.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { CacheService } from './cache.service';
import { QueryService } from '../query.service';

@Injectable()
export class QueryCacheService {
  private readonly logger = new Logger(QueryCacheService.name);
  private readonly CACHE_TTL = 3600; // 1 hour

  constructor(
    private cacheService: CacheService,
    private queryService: QueryService,
  ) {}

  async getCachedQuery(
    query: string,
    namespace: string,
    topK: number,
  ): Promise<any> {
    const cacheKey = this.generateCacheKey(query, namespace, topK);
    const cachedResult = await this.cacheService.get(cacheKey);

    if (cachedResult) {
      this.logger.log(`Cache hit for query: ${query}`);
      return JSON.parse(cachedResult);
    }

    this.logger.log(`Cache miss for query: ${query}`);
    const result = await this.queryService.ragQuery(query, namespace, topK);
    
    // Cache the result
    await this.cacheService.set(
      cacheKey,
      JSON.stringify(result),
      this.CACHE_TTL,
    );

    return result;
  }

  private generateCacheKey(
    query: string,
    namespace: string,
    topK: number,
  ): string {
    return `query:${namespace}:${query}:${topK}`;
  }

  async invalidateCache(namespace: string): Promise<void> {
    const pattern = `query:${namespace}:*`;
    const keys = await this.cacheService.keys(pattern);
    
    await Promise.all(
      keys.map(key => this.cacheService.del(key))
    );
  }
}
```

### 3. Embedding Cache

```typescript
// src/ai/services/embedding-cache.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { CacheService } from './cache.service';
import { OpenAIEmbeddings } from '@langchain/openai';

@Injectable()
export class EmbeddingCacheService {
  private readonly logger = new Logger(EmbeddingCacheService.name);
  private readonly EMBEDDING_TTL = 86400; // 24 hours

  constructor(
    private cacheService: CacheService,
    private embeddings: OpenAIEmbeddings,
  ) {}

  async getEmbedding(text: string): Promise<number[]> {
    const cacheKey = this.generateCacheKey(text);
    const cachedResult = await this.cacheService.get(cacheKey);

    if (cachedResult) {
      this.logger.log(`Cache hit for embedding: ${text.substring(0, 50)}...`);
      return JSON.parse(cachedResult);
    }

    this.logger.log(`Cache miss for embedding: ${text.substring(0, 50)}...`);
    const embedding = await this.embeddings.embedQuery(text);

    // Cache the result
    await this.cacheService.set(
      cacheKey,
      JSON.stringify(embedding),
      this.EMBEDDING_TTL,
    );

    return embedding;
  }

  private generateCacheKey(text: string): string {
    // Use hash of text to prevent cache key length issues
    const hash = this.hashText(text);
    return `embedding:${hash}`;
  }

  private hashText(text: string): string {
    // Simple hash function - in production use a proper hash
    return text.split('').reduce((a, b) => {
      a = ((a << 5) - a) + b.charCodeAt(0);
      return a & a;
    }, 0).toString();
  }
}
```

## Performance Optimization

### 1. Batch Processing

```typescript
// src/ai/services/batch-processing.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Queue } from 'bull';
import Redis from 'ioredis';

@Injectable()
export class BatchProcessingService {
  private readonly logger = new Logger(BatchProcessingService.name);
  private queue: Queue;

  constructor(private redis: Redis) {
    this.queue = new Queue('batch-processing', {
      redis: {
        host: process.env.REDIS_HOST,
        port: parseInt(process.env.REDIS_PORT || '6379'),
      },
    });
  }

  async processBatch(
    items: any[],
    batchSize: number = 100,
  ): Promise<void> {
    const total = items.length;
    const batches = Math.ceil(total / batchSize);

    for (let i = 0; i < batches; i++) {
      const start = i * batchSize;
      const end = Math.min(start + batchSize, total);
      const batch = items.slice(start, end);

      await this.queue.add('batch-processing', {
        batch,
        batchNumber: i + 1,
        totalBatches: batches,
      });
    }
  }

  async processWorker(job: any): Promise<void> {
    const { batch, batchNumber, totalBatches } = job.data;
    
    try {
      // Process the batch
      await this.processItems(batch);
      
      // Update progress
      await this.updateProgress(batchNumber, totalBatches);
    } catch (error) {
      this.logger.error('Batch processing error:', error);
      throw error;
    }
  }

  private async processItems(items: any[]): Promise<void> {
    // Implement batch processing logic
  }

  private async updateProgress(
    current: number,
    total: number,
  ): Promise<void> {
    await this.redis.set(
      'batch-processing:progress',
      JSON.stringify({ current, total }),
    );
  }
}
```

### 2. Token Management

```typescript
// src/ai/services/token-management.service.ts
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class TokenManagementService {
  private readonly logger = new Logger(TokenManagementService.name);
  private readonly MAX_TOKENS = 4096; // GPT-4 context window

  async manageContext(
    messages: any[],
    newMessage: any,
  ): Promise<any[]> {
    const updatedMessages = [...messages, newMessage];
    return this.trimMessages(updatedMessages);
  }

  private async trimMessages(messages: any[]): Promise<any[]> {
    let totalTokens = 0;
    const trimmedMessages = [];

    // Count tokens from newest to oldest
    for (let i = messages.length - 1; i >= 0; i--) {
      const message = messages[i];
      const messageTokens = await this.countTokens(message.content);
      
      if (totalTokens + messageTokens <= this.MAX_TOKENS) {
        trimmedMessages.unshift(message);
        totalTokens += messageTokens;
      } else {
        // If message doesn't fit, try to summarize it
        const summarized = await this.summarizeMessage(message);
        if (summarized) {
          const summaryTokens = await this.countTokens(summarized);
          if (totalTokens + summaryTokens <= this.MAX_TOKENS) {
            trimmedMessages.unshift({
              ...message,
              content: summarized,
            });
            totalTokens += summaryTokens;
          }
        }
        break;
      }
    }

    return trimmedMessages;
  }

  private async countTokens(text: string): Promise<number> {
    // Implement token counting logic
    return text.length / 4; // Rough estimate
  }

  private async summarizeMessage(message: any): Promise<string | null> {
    // Implement message summarization
    return null;
  }
}
```

## Monitoring and Metrics

```typescript
// src/ai/services/performance-monitoring.service.ts
import { Injectable } from '@nestjs/common';
import { createLogger, format, transports } from 'winston';
import { Counter, Histogram } from 'prom-client';

@Injectable()
export class PerformanceMonitoringService {
  private readonly logger = createLogger({
    level: 'info',
    format: format.combine(
      format.timestamp(),
      format.json(),
    ),
    transports: [
      new transports.Console(),
      new transports.File({ filename: 'logs/performance.log' }),
    ],
  });

  // Metrics
  private queryCounter = new Counter({
    name: 'ai_queries_total',
    help: 'Total number of AI queries',
    labelNames: ['status', 'type'],
  });

  private queryDuration = new Histogram({
    name: 'ai_query_duration_seconds',
    help: 'Duration of AI queries in seconds',
    labelNames: ['type'],
    buckets: [0.1, 0.3, 0.5, 1, 2, 5],
  });

  async recordQuery(
    type: string,
    status: 'success' | 'failure',
    duration: number,
  ) {
    this.queryCounter.labels(status, type).inc();
    this.queryDuration.labels(type).observe(duration);
    
    this.logger.info('Query Metrics', {
      type,
      status,
      duration,
      timestamp: new Date().toISOString(),
    });
  }

  async logError(
    type: string,
    error: Error,
    context: Record<string, any>,
  ) {
    this.logger.error('Query Error', {
      type,
      errorMessage: error.message,
      stack: error.stack,
      ...context,
      timestamp: new Date().toISOString(),
    });
  }
}
```

## Best Practices

1. **Caching**
   - Use appropriate TTL values
   - Implement cache invalidation strategies
   - Monitor cache hit/miss rates

2. **Batch Processing**
   - Use proper batch sizes
   - Implement retry mechanisms
   - Monitor worker queues

3. **Token Management**
   - Implement proper token counting
   - Use sliding window
   - Summarize old messages

4. **Performance**
   - Monitor response times
   - Track resource usage
   - Implement proper timeouts

5. **Monitoring**
   - Track metrics
   - Set up alerts
   - Monitor error rates

## Common Issues and Solutions

1. **Cache Invalidation**
   - Solution: Implement proper invalidation strategies
   - Solution: Use cache versioning
   - Solution: Monitor cache consistency

2. **Batch Processing**
   - Solution: Implement proper error handling
   - Solution: Use exponential backoff
   - Solution: Monitor queue health

3. **Token Management**
   - Solution: Implement proper token counting
   - Solution: Use sliding window
   - Solution: Summarize old messages

4. **Performance**
   - Solution: Implement proper monitoring
   - Solution: Use proper timeouts
   - Solution: Monitor resource usage

5. **Monitoring**
   - Solution: Set up proper metrics
   - Solution: Implement alerting
   - Solution: Monitor error rates

## Caching Techniques
- **In-Memory Caching**: Using Redis for fast data access.
- **Query Optimization**: Techniques for improving query performance.
Caching and optimization are essential for ensuring a responsive application.
```