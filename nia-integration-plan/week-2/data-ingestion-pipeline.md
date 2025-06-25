# Data Ingestion Pipeline for NIA

## Overview
The data ingestion pipeline is responsible for processing various document formats, extracting text, generating embeddings, and storing them in the vector database. This document outlines the design and implementation of the pipeline.

## Pipeline Components

### 1. Document Upload Service
- Handles file uploads from the frontend
- Validates file types and sizes
- Stores original files in object storage
- Triggers the processing workflow

### 2. Document Processor
- Extracts text from different file formats (PDF, DOCX, TXT, etc.)
- Cleans and normalizes text
- Splits documents into chunks
- Extracts and preserves metadata

### 3. Text Chunking Service
- Implements intelligent text splitting
- Maintains context across chunks
- Handles different document structures
- Preserves document hierarchy

### 4. Embedding Generator
- Converts text chunks into vector embeddings
- Handles rate limiting and retries
- Supports multiple embedding models
- Caches embeddings when possible

### 5. Vector Store Writer
- Stores embeddings in the vector database
- Maintains document and chunk relationships
- Handles batch operations
- Implements error handling and retries

## Implementation

### Document Processing Service
```typescript
// src/document-processing/document-processor.service.ts
import { Injectable } from '@nestjs/common';
import { promises as fs } from 'fs';
import { extname } from 'path';
import { TextSplitter } from 'langchain/text_splitter';
import { PDFLoader } from 'langchain/document_loaders/fs/pdf';
import { DocxLoader } from 'langchain/document_loaders/fs/docx';

@Injectable()
export class DocumentProcessorService {
  private readonly textSplitter: TextSplitter;

  constructor() {
    this.textSplitter = new RecursiveCharacterTextSplitter({
      chunkSize: 1000,
      chunkOverlap: 200,
    });
  }

  async processDocument(filePath: string, metadata: Record<string, any> = {}) {
    const extension = extname(filePath).toLowerCase();
    let loader;

    switch (extension) {
      case '.pdf':
        loader = new PDFLoader(filePath);
        break;
      case '.docx':
        loader = new DocxLoader(filePath);
        break;
      case '.txt':
        return this.processTextFile(filePath, metadata);
      default:
        throw new Error(`Unsupported file type: ${extension}`);
    }

    const docs = await loader.load();
    return this.splitDocuments(docs, metadata);
  }

  private async processTextFile(filePath: string, metadata: Record<string, any>) {
    const content = await fs.readFile(filePath, 'utf-8');
    return this.textSplitter.splitText(content).map(chunk => ({
      pageContent: chunk,
      metadata: { ...metadata, source: filePath }
    }));
  }

  private async splitDocuments(docs: Document[], metadata: Record<string, any>) {
    const chunks = [];
    for (const doc of docs) {
      const docChunks = await this.textSplitter.splitDocuments([doc]);
      chunks.push(...docChunks.map(chunk => ({
        ...chunk,
        metadata: { ...chunk.metadata, ...metadata }
      })));
    }
    return chunks;
  }
}
```

### Embedding Service
```typescript
// src/embeddings/embedding.service.ts
import { Injectable } from '@nestjs/common';
import { OpenAIEmbeddings } from 'langchain/embeddings/openai';

@Injectable()
export class EmbeddingService {
  private readonly embeddings: OpenAIEmbeddings;

  constructor() {
    this.embeddings = new OpenAIEmbeddings({
      openAIApiKey: process.env.OPENAI_API_KEY,
      modelName: 'text-embedding-3-small',
      batchSize: 512,
    });
  }

  async generateEmbeddings(texts: string[]): Promise<number[][]> {
    return this.embeddings.embedDocuments(texts);
  }

  async generateEmbedding(text: string): Promise<number[]> {
    return this.embeddings.embedQuery(text);
  }
}
```

### Vector Store Service
```typescript
// src/vector-store/vector-store.service.ts
import { Injectable } from '@nestjs/common';
import { Pinecone } from '@pinecone-database/pinecone';

@Injectable()
export class VectorStoreService {
  private readonly pinecone: Pinecone;
  private readonly indexName = 'procurement-docs';

  constructor() {
    this.pinecone = new Pinecone({
      apiKey: process.env.PINECONE_API_KEY,
    });
  }

  async upsertVectors(
    vectors: Array<{
      id: string;
      values: number[];
      metadata: Record<string, any>;
    }>,
    namespace: string
  ) {
    const index = this.pinecone.Index(this.indexName);
    
    // Batch process to handle Pinecone's limits
    const batchSize = 100;
    for (let i = 0; i < vectors.length; i += batchSize) {
      const batch = vectors.slice(i, i + batchSize);
      await index.upsert(
        batch.map(v => ({
          id: v.id,
          values: v.values,
          metadata: v.metadata,
        })),
        { namespace }
      );
    }
  }

  async search(
    queryEmbedding: number[],
    options: {
      namespace: string;
      topK?: number;
      filter?: Record<string, any>;
    }
  ) {
    const index = this.pinecone.Index(this.indexName);
    return index.query({
      vector: queryEmbedding,
      topK: options.topK || 5,
      includeMetadata: true,
      filter: options.filter,
      namespace: options.namespace,
    });
  }
}
```

## Pipeline Orchestration

```typescript
// src/pipelines/document-processing.pipeline.ts
import { Injectable } from '@nestjs/common';
import { DocumentProcessorService } from '../document-processing/document-processor.service';
import { EmbeddingService } from '../embeddings/embedding.service';
import { VectorStoreService } from '../vector-store/vector-store.service';

@Injectable()
export class DocumentProcessingPipeline {
  constructor(
    private readonly documentProcessor: DocumentProcessorService,
    private readonly embeddingService: EmbeddingService,
    private readonly vectorStore: VectorStoreService,
  ) {}

  async processDocument(
    filePath: string,
    tenantId: string,
    metadata: Record<string, any> = {}
  ) {
    // 1. Process document into chunks
    const chunks = await this.documentProcessor.processDocument(filePath, {
      ...metadata,
      tenantId,
      processedAt: new Date().toISOString(),
    });

    // 2. Generate embeddings for chunks
    const texts = chunks.map(chunk => chunk.pageContent);
    const embeddings = await this.embeddingService.generateEmbeddings(texts);

    // 3. Prepare vectors for storage
    const vectors = chunks.map((chunk, index) => ({
      id: `chunk-${Date.now()}-${index}`,
      values: embeddings[index],
      metadata: {
        ...chunk.metadata,
        chunkIndex: index,
        text: chunk.pageContent,
      },
    }));

    // 4. Store in vector database
    await this.vectorStore.upsertVectors(vectors, tenantId);

    return {
      documentId: metadata.documentId,
      chunksProcessed: chunks.length,
      timestamp: new Date().toISOString(),
    };
  }
}
```

## Error Handling and Monitoring

### Error Handling
- Implement retry logic for API calls
- Log all errors with context
- Notify administrators of critical failures
- Implement dead-letter queues for failed processing

### Monitoring
- Track processing times
- Monitor error rates
- Log document processing metrics
- Set up alerts for anomalies

## Performance Optimization

### Batch Processing
- Process documents in parallel when possible
- Implement batch processing for embeddings
- Use streaming for large documents

### Caching
- Cache frequently accessed embeddings
- Implement document chunk caching
- Use in-memory cache for hot data

## Security Considerations

### Data Protection
- Encrypt sensitive data at rest
- Implement access controls at all levels
- Audit all data access

### Compliance
- Implement data retention policies
- Support data deletion requests
- Log all data processing activities

## Testing

### Unit Tests
- Test document parsing for each format
- Verify text splitting logic
- Test error handling

### Integration Tests
- Test full pipeline with mock services
- Verify vector storage and retrieval
- Test error scenarios

### Performance Tests
- Test with large documents
- Measure processing times
- Identify bottlenecks
