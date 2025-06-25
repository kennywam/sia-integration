# RAG Query Service Implementation

## Core Implementation

### 1. Query Service

```typescript
// src/ai/services/query.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Pinecone } from '@pinecone-database/pinecone';
import { OpenAIEmbeddings } from '@langchain/openai';
import { ChatOpenAI } from '@langchain/openai';

@Injectable()
export class QueryService {
  private readonly logger = new Logger(QueryService.name);
  private embeddings: OpenAIEmbeddings;
  private llm: ChatOpenAI;
  private pinecone: Pinecone;

  constructor(private configService: ConfigService) {
    // Initialize services
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: this.configService.get('OPENAI_API_KEY'),
      modelName: 'text-embedding-3-small',
    });

    this.llm = new ChatOpenAI({
      openAIApiKey: this.configService.get('OPENAI_API_KEY'),
      modelName: 'gpt-4-turbo',
      temperature: 0.7,
    });

    this.pinecone = new Pinecone({
      apiKey: this.configService.get('PINECONE_API_KEY'),
    });
  }

  async search(
    query: string,
    namespace: string,
    topK = 5,
  ): Promise<any[]> {
    try {
      // Generate query embedding
      const queryEmbedding = await this.embeddings.embedQuery(query);

      // Search in Pinecone
      const index = this.pinecone.Index(
        this.configService.get('PINECONE_INDEX_NAME'),
      );

      const results = await index.query({
        vector: queryEmbedding,
        topK,
        includeMetadata: true,
        namespace,
      });

      return results.matches.map(match => ({
        id: match.id,
        score: match.score,
        metadata: match.metadata,
      }));
    } catch (error) {
      this.logger.error('Search error:', error);
      throw new Error('Failed to perform search');
    }
  }

  async generateResponse(
    query: string,
    context: string[],
  ): Promise<string> {
    try {
      // Construct prompt
      const prompt = `Context:\n${context.join('\n\n')}\n\nQuestion: ${query}\nAnswer:`;

      // Generate response
      const response = await this.llm.invoke(prompt);
      return response.content.toString();
    } catch (error) {
      this.logger.error('Generation error:', error);
      throw new Error('Failed to generate response');
    }
  }

  async ragQuery(
    query: string,
    namespace: string,
    topK = 5,
  ): Promise<{
    answer: string;
    sources: any[];
  }> {
    try {
      // 1. Search for relevant documents
      const results = await this.search(query, namespace, topK);

      // 2. Extract context
      const context = results
        .map(r => r.metadata?.text || '')
        .filter(Boolean);

      // 3. Generate response
      const answer = await this.generateResponse(query, context);

      return {
        answer,
        sources: results,
      };
    } catch (error) {
      this.logger.error('RAG query error:', error);
      throw new Error('Failed to process query');
    }
  }
}
```

### 2. Controller

```typescript
// src/ai/controllers/query.controller.ts
import { Controller, Post, Body, UseGuards, Req } from '@nestjs/common';
import { ApiBearerAuth, ApiOperation, ApiTags } from '@nestjs/swagger';
import { QueryService } from '../services/query.service';
import { JwtAuthGuard } from '../../auth/guards/jwt-auth.guard';
import { TenantId } from '../../common/decorators/tenant-id.decorator';

class QueryDto {
  query: string;
  topK?: number;
}

@ApiTags('ai-query')
@ApiBearerAuth()
@Controller('ai/query')
@UseGuards(JwtAuthGuard)
export class QueryController {
  constructor(private readonly queryService: QueryService) {}

  @Post()
  @ApiOperation({ summary: 'Process RAG query' })
  async query(
    @Body() { query, topK = 5 }: QueryDto,
    @TenantId() tenantId: string,
    @Req() req: any,
  ) {
    return this.queryService.ragQuery(query, tenantId, topK);
  }
}
```

### 3. Module Setup

```typescript
// src/ai/ai.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { QueryService } from './services/query.service';
import { QueryController } from './controllers/query.controller';

@Module({
  imports: [ConfigModule],
  controllers: [QueryController],
  providers: [QueryService],
  exports: [QueryService],
})
export class AiModule {}
```

## Error Handling

```typescript
// src/ai/middleware/error-handler.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { Logger } from '@nestjs/common';

@Injectable()
export class ErrorHandlerMiddleware implements NestMiddleware {
  private readonly logger = new Logger(ErrorHandlerMiddleware.name);

  use(req: Request, res: Response, next: NextFunction) {
    try {
      next();
    } catch (error) {
      this.logger.error(`Error in request: ${error.message}`, error.stack);
      
      res.status(500).json({
        error: 'Internal server error',
        message: error.message,
        timestamp: new Date().toISOString(),
      });
    }
  }
}
```

## Testing

```typescript
// test/ai/services/query.service.spec.ts
describe('QueryService', () => {
  let service: QueryService;
  let mockPinecone: jest.Mocked<Pinecone>;
  let mockEmbeddings: jest.Mocked<OpenAIEmbeddings>;
  let mockLlm: jest.Mocked<ChatOpenAI>;

  beforeEach(async () => {
    mockPinecone = {
      Index: jest.fn(),
    } as any;

    mockEmbeddings = {
      embedQuery: jest.fn(),
    } as any;

    mockLlm = {
      invoke: jest.fn(),
    } as any;

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        QueryService,
        {
          provide: Pinecone,
          useValue: mockPinecone,
        },
        {
          provide: OpenAIEmbeddings,
          useValue: mockEmbeddings,
        },
        {
          provide: ChatOpenAI,
          useValue: mockLlm,
        },
      ],
    }).compile();

    service = module.get<QueryService>(QueryService);
  });

  it('should search successfully', async () => {
    const mockQuery = 'test query';
    const mockNamespace = 'test-namespace';
    const mockTopK = 3;

    // Mock embeddings
    mockEmbeddings.embedQuery.mockResolvedValue([0.1, 0.2, 0.3]);

    // Mock Pinecone response
    mockPinecone.Index.mockReturnValue({
      query: jest.fn().mockResolvedValue({
        matches: [
          {
            id: '1',
            score: 0.9,
            metadata: { text: 'test document' },
          },
        ],
      }),
    });

    const results = await service.search(mockQuery, mockNamespace, mockTopK);
    expect(results).toHaveLength(1);
  });

  it('should generate response', async () => {
    const mockQuery = 'test query';
    const mockContext = ['test document'];

    // Mock LLM response
    mockLlm.invoke.mockResolvedValue({
      content: 'test response',
    });

    const response = await service.generateResponse(mockQuery, mockContext);
    expect(response).toBe('test response');
  });
});
```