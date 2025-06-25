# Building a Procurement Chatbot with Next.js and Vercel AI SDK

## Project Setup

### Prerequisites
- Node.js 18+
- npm or yarn
- OpenAI API key
- Pinecone account (or alternative vector database)

### Initialize Project
```bash
# Create a new Next.js app with TypeScript
npx create-next-app@latest procurement-chatbot --typescript --tailwind --eslint
cd procurement-chatbot

# Install required dependencies
npm install ai @ai-sdk/openai @pinecone-database/pinecone pdf-parse
```

## Backend Implementation

### 1. Set Up API Route
Create `app/api/chat/route.ts`:

```typescript
import { OpenAI } from '@ai-sdk/openai';
import { StreamingTextResponse, streamText } from 'ai';
import { Pinecone } from '@pinecone-database/pinecone';

export const runtime = 'edge';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
});

const pc = new Pinecone({
  apiKey: process.env.PINECONE_API_KEY!,
});

export async function POST(req: Request) {
  try {
    const { messages } = await req.json();
    const lastMessage = messages[messages.length - 1];

    // 1. Get relevant context from vector store
    const index = pc.index('procurement-docs');
    const queryEmbedding = await generateEmbedding(lastMessage.content);
    
    const results = await index.query({
      vector: queryEmbedding,
      topK: 3,
      includeMetadata: true,
      filter: { tenantId: 'tenant-123' } // Replace with actual tenant ID
    });

    // 2. Build context from retrieved documents
    const context = results.matches
      .map(match => match.metadata?.text)
      .join('\n\n');

    // 3. Create system prompt with context
    const systemPrompt = {
      role: 'system' as const,
      content: `You are Nia, a procurement assistant. Answer questions based on the following context. If you don't know the answer, say so.\n\nContext: ${context}`
    };

    // 4. Call OpenAI with the full message history
    const result = await streamText({
      model: openai('gpt-4'),
      messages: [systemPrompt, ...messages],
    });

    // 5. Stream the response back to the client
    return new StreamingTextResponse(result.toAIStream());
  } catch (error) {
    console.error('Error:', error);
    return new Response('Error processing your request', { status: 500 });
  }
}

async function generateEmbedding(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: text,
  });
  return response.data[0].embedding;
}
```

## Frontend Implementation

### 1. Create Chat Interface
Create `app/page.tsx`:

```tsx
'use client';

import { useChat } from 'ai/react';

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat();

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      <header className="border-b pb-4 mb-4">
        <h1 className="text-2xl font-bold">Procurement Assistant</h1>
        <p className="text-gray-600">Ask me anything about procurement policies and procedures</p>
      </header>

      <div className="flex-1 overflow-y-auto mb-4 space-y-4">
        {messages.map((message) => (
          <div
            key={message.id}
            className={`p-4 rounded-lg ${
              message.role === 'user'
                ? 'bg-blue-100 ml-8'
                : 'bg-gray-100 mr-8'
            }`}
          >
            <div className="font-medium">
              {message.role === 'user' ? 'You' : 'Nia'}
            </div>
            <div className="whitespace-pre-wrap">{message.content}</div>
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="flex gap-2">
        <input
          className="flex-1 border rounded-lg px-4 py-2"
          value={input}
          onChange={handleInputChange}
          placeholder="Ask about procurement policies..."
        />
        <button
          type="submit"
          className="bg-blue-500 text-white px-6 py-2 rounded-lg hover:bg-blue-600 transition-colors"
        >
          Send
        </button>
      </form>
    </div>
  );
}
```

## Document Processing

### 1. Create Document Upload Component
Create `components/DocumentUpload.tsx`:

```tsx
'use client';

import { useState } from 'react';

export default function DocumentUpload() {
  const [isUploading, setIsUploading] = useState(false);
  const [progress, setProgress] = useState(0);

  const handleFileUpload = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    setIsUploading(true);
    setProgress(0);

    try {
      const formData = new FormData();
      formData.append('file', file);
      
      const response = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
      });

      if (!response.ok) {
        throw new Error('Upload failed');
      }

      setProgress(100);
      alert('Document uploaded and processed successfully!');
    } catch (error) {
      console.error('Error uploading document:', error);
      alert('Failed to upload document');
    } finally {
      setIsUploading(false);
    }
  };

  return (
    <div className="mb-6 p-4 border rounded-lg">
      <h2 className="text-lg font-medium mb-2">Upload Document</h2>
      <input
        type="file"
        accept=".pdf,.docx,.txt"
        onChange={handleFileUpload}
        disabled={isUploading}
        className="block w-full text-sm text-gray-500
          file:mr-4 file:py-2 file:px-4
          file:rounded-md file:border-0
          file:text-sm file:font-semibold
          file:bg-blue-50 file:text-blue-700
          hover:file:bg-blue-100"
      />
      {isUploading && (
        <div className="w-full bg-gray-200 rounded-full h-2.5 mt-2">
          <div
            className="bg-blue-600 h-2.5 rounded-full"
            style={{ width: `${progress}%` }}
          ></div>
        </div>
      )}
    </div>
  );
}
```

## Testing Your Chatbot

1. Start the development server:
   ```bash
   npm run dev
   ```

2. Open http://localhost:3000 in your browser

3. Try asking questions like:
   - "What's the approval process for purchases over $10,000?"
   - "How do I create a new vendor?"
   - "What's our policy on emergency purchases?"

## Next Steps

1. Add authentication and multi-tenant support
2. Implement document processing for different file types
3. Add chat history persistence
4. Improve UI with typing indicators and better message styling
5. Add support for file uploads and document references

## Troubleshooting

- **API Key Errors**: Ensure your OpenAI and Pinecone API keys are set in `.env.local`
- **CORS Issues**: Make sure your API routes are properly configured
- **Vector Search Not Working**: Verify your Pinecone index is properly set up with documents

## Resources
- [Vercel AI SDK Documentation](https://sdk.vercel.ai/docs)
- [Next.js API Routes](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Pinecone JavaScript Client](https://docs.pinecone.io/reference/js-client)
