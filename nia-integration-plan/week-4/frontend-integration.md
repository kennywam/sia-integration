### Frontend Integration with Backend Services

## Overview

This document outlines the integration of the frontend with the backend services, detailing how to connect the chat interface to the AI assistant.

## Core Components

### 1. Chat Interface Component

```tsx
// components/chat/ChatInterface.tsx
'use client'

import { useChat } from 'ai/react'
import { useContext } from 'react'
import { UserContext } from '@/contexts/UserContext'

export function ChatInterface() {
  const { user, currentPage } = useContext(UserContext)

  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({
      api: '/api/chat',
      headers: {
        'X-Tenant-ID': user.tenantId,
        'X-Current-Page': JSON.stringify(currentPage),
      },
      initialMessages: [
        {
          id: 'welcome',
          role: 'assistant',
          content: `Hi! I'm Nia, your procurement assistant. I can help you with information about your procurement data. What would you like to know?`,
        },
      ],
    })

  return (
    <div className='flex flex-col h-full max-w-2xl mx-auto'>
      <div className='flex-1 overflow-y-auto p-4 space-y-4'>
        {messages.map((message) => (
          <ChatMessage
            key={message.id}
            message={message}
          />
        ))}
        {isLoading && <TypingIndicator />}
      </div>

      <form
        onSubmit={handleSubmit}
        className='border-t p-4'
      >
        <div className='flex space-x-2'>
          <input
            value={input}
            onChange={handleInputChange}
            placeholder='Ask about your procurement data...'
            className='flex-1 border rounded-lg px-3 py-2'
            disabled={isLoading}
          />
          <button
            type='submit'
            disabled={isLoading}
            className='bg-blue-500 text-white px-4 py-2 rounded-lg disabled:opacity-50'
          >
            Send
          </button>
        </div>
      </form>
    </div>
  )
}
```

### 2. Context Provider

```tsx
// contexts/UserContext.tsx
'use client'

import { createContext, useContext, ReactNode } from 'react'
import { usePathname } from 'next/navigation'

interface UserContextType {
  user: User
  currentPage: CurrentPage | null
  hasAccess: (resource: string) => boolean
}

const UserContext = createContext<UserContextType | null>(null)

export function UserContextProvider({
  children,
  user,
}: {
  children: ReactNode
  user: User
}) {
  const pathname = usePathname()
  const currentPage = extractPageContext(pathname)

  const hasAccess = (resource: string) => {
    return user.accessLevels.includes(resource)
  }

  return (
    <UserContext.Provider value={{ user, currentPage, hasAccess }}>
      {children}
    </UserContext.Provider>
  )
}

function extractPageContext(pathname: string): CurrentPage | null {
  // Extract context from URL: /procurement/pr/PR-0012
  const prMatch = pathname.match(/\/pr\/([A-Z]+-\d+)/)
  if (prMatch) return { type: 'pr', id: prMatch[1] }

  const poMatch = pathname.match(/\/po\/([A-Z]+-\d+)/)
  if (poMatch) return { type: 'po', id: poMatch[1] }

  return null
}
```

## Integration Steps

### 1. Authentication & Context Management

```typescript
// src/auth/guards/ai-access.guard.ts
@Injectable()
export class AIAccessGuard implements CanActivate {
  constructor(private readonly permissionsService: PermissionsService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest()
    const user = request.user
    const { query, context: queryContext } = request.body

    // Check if user has access to AI features
    if (!this.permissionsService.hasAIAccess(user)) {
      return false
    }

    // Check if user has access to specific data being queried
    if (queryContext?.sourceId) {
      return this.permissionsService.hasResourceAccess(
        user,
        queryContext.sourceId
      )
    }

    return true
  }
}
```

### 2. Real-time Data Synchronization

```typescript
// src/ai/services/sync.service.ts
@Injectable()
export class SyncService {
  @Cron('0 */6 * * *') // Every 6 hours
  async syncAllTenants() {
    const tenants = await this.tenantService.findAll()

    for (const tenant of tenants) {
      await this.syncTenantData(tenant.id)
    }
  }

  async syncTenantData(tenantId: string) {
    const lastSync = await this.getLastSyncTime(tenantId)

    // Sync database changes
    const changedRecords = await this.getChangedRecords(tenantId, lastSync)
    await this.updateVectorStore(changedRecords, tenantId)

    // Sync document changes
    const changedDocuments = await this.getChangedDocuments(tenantId, lastSync)
    await this.reprocessDocuments(changedDocuments, tenantId)
  }
}
```

## Best Practices

1. **State Management**

   - Use React Context for user context
   - Maintain consistent session state
   - Handle rehydration properly

2. **Error Handling**

   - Implement proper error boundaries
   - Provide user-friendly error messages
   - Log errors for debugging

3. **Performance**

   - Implement proper caching
   - Optimize API calls
   - Use pagination for large datasets

4. **Security**

   - Validate all inputs
   - Implement proper access control
   - Handle sensitive data securely

5. **Testing**
   - Write comprehensive unit tests
   - Implement integration tests
   - Test error scenarios

## Common Issues and Solutions

1. **Context Loss**

   - Solution: Implement proper session management
   - Solution: Use local storage for temporary state
   - Solution: Add proper error recovery

2. **Performance**

   - Solution: Implement caching
   - Solution: Use pagination
   - Solution: Optimize API calls

3. **Security**
   - Solution: Implement proper validation
   - Solution: Use secure headers
   - Solution: Add proper authentication
