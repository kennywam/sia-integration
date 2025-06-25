# Context Provider Implementation

## Overview

This document explains the context provider setup in the frontend, detailing how to manage user context and permissions.

## Core Components

### 1. User Context Provider

```tsx
// contexts/UserContext.tsx
'use client'

import {
  createContext,
  useContext,
  ReactNode,
  useEffect,
  useState,
} from 'react'
import { usePathname, useRouter } from 'next/navigation'
import { useToast } from '@/hooks/useToast'

interface User {
  id: string
  name: string
  email: string
  tenantId: string
  accessLevels: string[]
  permissions: string[]
}

interface CurrentPage {
  type: 'pr' | 'po' | 'rfq' | 'contract'
  id: string
}

interface UserContextType {
  user: User | null
  currentPage: CurrentPage | null
  hasAccess: (resource: string) => boolean
  isLoading: boolean
  error: string | null
  refreshContext: () => Promise<void>
}

const UserContext = createContext<UserContextType | null>(null)

export function UserContextProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [currentPage, setCurrentPage] = useState<CurrentPage | null>(null)
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  const pathname = usePathname()
  const router = useRouter()
  const { showToast } = useToast()

  useEffect(() => {
    fetchUserContext()
  }, [])

  useEffect(() => {
    updateCurrentPage()
  }, [pathname])

  const fetchUserContext = async () => {
    try {
      setIsLoading(true)
      const response = await fetch('/api/context')
      if (!response.ok) {
        throw new Error('Failed to fetch user context')
      }
      const data = await response.json()
      setUser(data.user)
      setError(null)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error')
      showToast({
        title: 'Error',
        description: 'Failed to load user context',
        type: 'error',
      })
    } finally {
      setIsLoading(false)
    }
  }

  const updateCurrentPage = () => {
    const page = extractPageContext(pathname)
    setCurrentPage(page)
  }

  const refreshContext = async () => {
    await fetchUserContext()
  }

  const hasAccess = (resource: string) => {
    if (!user) return false
    return user.permissions.includes(resource)
  }

  return (
    <UserContext.Provider
      value={{
        user,
        currentPage,
        hasAccess,
        isLoading,
        error,
        refreshContext,
      }}
    >
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

  const rfqMatch = pathname.match(/\/rfq\/([A-Z]+-\d+)/)
  if (rfqMatch) return { type: 'rfq', id: rfqMatch[1] }

  const contractMatch = pathname.match(/\/contract\/([A-Z]+-\d+)/)
  if (contractMatch) return { type: 'contract', id: contractMatch[1] }

  return null
}
```

### 2. Toast Hook

```tsx
// hooks/useToast.tsx
'use client'

import { createContext, useContext, useState } from 'react'

interface Toast {
  title: string
  description: string
  type: 'success' | 'error' | 'warning' | 'info'
  id: string
}

interface ToastContextType {
  showToast: (toast: Omit<Toast, 'id'>) => void
  removeToast: (id: string) => void
}

const ToastContext = createContext<ToastContextType | null>(null)

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([])

  const showToast = (toast: Omit<Toast, 'id'>) => {
    const newToast: Toast = {
      ...toast,
      id: crypto.randomUUID(),
    }
    setToasts((prev) => [...prev, newToast])
  }

  const removeToast = (id: string) => {
    setToasts((prev) => prev.filter((t) => t.id !== id))
  }

  return (
    <ToastContext.Provider value={{ showToast, removeToast }}>
      {children}
      <div className='fixed bottom-4 right-4 flex flex-col gap-2'>
        {toasts.map((toast) => (
          <div
            key={toast.id}
            className={`p-4 rounded-lg shadow-lg ${
              toast.type === 'success'
                ? 'bg-green-500 text-white'
                : toast.type === 'error'
                ? 'bg-red-500 text-white'
                : toast.type === 'warning'
                ? 'bg-yellow-500 text-gray-900'
                : 'bg-blue-500 text-white'
            }`}
          >
            <div className='flex justify-between items-start'>
              <div>
                <h3 className='font-medium'>{toast.title}</h3>
                <p className='text-sm'>{toast.description}</p>
              </div>
              <button
                onClick={() => removeToast(toast.id)}
                className='text-white hover:text-gray-200'
              >
                ×
              </button>
            </div>
          </div>
        ))}
      </div>
    </ToastContext.Provider>
  )
}

export function useToast() {
  const context = useContext(ToastContext)
  if (!context) {
    throw new Error('useToast must be used within a ToastProvider')
  }
  return context
}
```

## Best Practices

1. **State Management**

   - Use proper type definitions
   - Implement proper error handling
   - Add loading states
   - Include refresh functionality

2. **Security**

   - Validate permissions
   - Handle unauthorized access
   - Implement proper error messages
   - Use secure headers

3. **User Experience**

   - Show loading indicators
   - Provide error messages
   - Implement toast notifications
   - Handle page changes gracefully

4. **Performance**

   - Implement proper caching
   - Use memoization
   - Optimize re-renders
   - Add proper cleanup

5. **Testing**
   - Write component tests
   - Test error scenarios
   - Test permission checks
   - Test page context extraction

## Common Issues and Solutions

1. **State Management**

   - Solution: Use proper type definitions
   - Solution: Implement proper error handling
   - Solution: Add loading states
   - Solution: Include refresh functionality

2. **Security**

   - Solution: Validate permissions
   - Solution: Handle unauthorized access
   - Solution: Implement proper error messages
   - Solution: Use secure headers

3. **User Experience**

   - Solution: Show loading indicators
   - Solution: Provide error messages
   - Solution: Implement toast notifications
   - Solution: Handle page changes gracefully

4. **Performance**

   - Solution: Implement proper caching
   - Solution: Use memoization
   - Solution: Optimize re-renders
   - Solution: Add proper cleanup

5. **Testing**
   - Solution: Write comprehensive tests
   - Solution: Test error scenarios
   - Solution: Test permission checks
   - Solution: Test page context extraction
