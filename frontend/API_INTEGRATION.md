<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# API Integration
## Frontend ↔ Backend Communication Pattern

> **For AI:** Axios-based HTTP client with auth, error handling, and interceptors.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Quick Reference

**HTTP Client:** Axios

**Features:**
- Automatic JWT token injection
- Request/response interceptors
- Error handling
- Retry logic
- Loading states

---

## 🔧 Axios Client Setup

```typescript
// src/services/api.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosError } from 'axios'
import { useAuthStore } from '@/stores/auth'
import router from '@/router'

class ApiClient {
  private client: AxiosInstance

  constructor() {
    this.client = axios.create({
      baseURL: import.meta.env.VITE_API_URL || '/api',
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json',
      },
    })

    this.setupInterceptors()
  }

  private setupInterceptors() {
    // Request interceptor: Add auth token
    this.client.interceptors.request.use(
      (config) => {
        const authStore = useAuthStore()
        if (authStore.accessToken) {
          config.headers.Authorization = `Bearer ${authStore.accessToken}`
        }
        return config
      },
      (error) => Promise.reject(error)
    )

    // Response interceptor: Handle errors
    this.client.interceptors.response.use(
      (response) => response,
      async (error: AxiosError) => {
        const originalRequest = error.config as AxiosRequestConfig & { _retry?: boolean }

        // Token expired - try to refresh
        if (error.response?.status === 401 && !originalRequest._retry) {
          originalRequest._retry = true

          try {
            const newToken = await this.refreshToken()
            const authStore = useAuthStore()
            authStore.accessToken = newToken

            // Retry original request with new token
            if (originalRequest.headers) {
              originalRequest.headers.Authorization = `Bearer ${newToken}`
            }
            return this.client(originalRequest)
          } catch (refreshError) {
            // Refresh failed - redirect to login
            const authStore = useAuthStore()
            authStore.logout()
            router.push({ name: 'login' })
            return Promise.reject(refreshError)
          }
        }

        return Promise.reject(error)
      }
    )
  }

  private async refreshToken(): Promise<string> {
    const response = await axios.post('/api/auth/refresh', {}, {
      withCredentials: true,  // Send httpOnly cookie
    })
    return response.data.accessToken
  }

  // HTTP methods
  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.get<T>(url, config)
    return response.data
  }

  async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.post<T>(url, data, config)
    return response.data
  }

  async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.put<T>(url, data, config)
    return response.data
  }

  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.delete<T>(url, config)
    return response.data
  }
}

export const api = new ApiClient()
```

---

## 📦 Service Layer Pattern

Create service modules for each API domain:

```typescript
// src/services/user.service.ts
import { api } from './api'
import type { User, ApiResponse } from '@/types/models'

export const userService = {
  async getProfile(): Promise<User> {
    const response = await api.get<ApiResponse<User>>('/users/profile')
    return response.data
  },

  async updateProfile(data: Partial<User>): Promise<User> {
    const response = await api.put<ApiResponse<User>>('/users/profile', data)
    return response.data
  },

  async getUser(userId: string): Promise<User> {
    const response = await api.get<ApiResponse<User>>(`/users/${userId}`)
    return response.data
  },
}

// src/services/video.service.ts
import { api } from './api'
import type { Video, ApiResponse } from '@/types/models'

export const videoService = {
  async getVideos(): Promise<Video[]> {
    const response = await api.get<ApiResponse<Video[]>>('/videos')
    return response.data
  },

  async getVideo(videoId: string): Promise<Video> {
    const response = await api.get<ApiResponse<Video>>(`/videos/${videoId}`)
    return response.data
  },

  async uploadVideo(file: File): Promise<{ uploadUrl: string; videoId: string }> {
    // Get pre-signed URL
    const { uploadUrl, videoId } = await api.post<ApiResponse<any>>('/videos/upload-url', {
      filename: file.name,
      contentType: file.type,
    }).then(r => r.data)

    // Upload directly to S3
    await fetch(uploadUrl, {
      method: 'PUT',
      headers: { 'Content-Type': file.type },
      body: file,
    })

    // Notify backend
    await api.post(`/videos/${videoId}/upload-complete`)

    return { uploadUrl, videoId }
  },
}
```

---

## 🎣 Composables for API Calls

Create reusable composables with loading/error states:

```typescript
// src/composables/useApi.ts
import { ref } from 'vue'

export function useApi<T>(apiCall: () => Promise<T>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const isLoading = ref(false)

  async function execute() {
    isLoading.value = true
    error.value = null

    try {
      data.value = await apiCall()
    } catch (e) {
      error.value = e as Error
      console.error('API call failed:', e)
    } finally {
      isLoading.value = false
    }
  }

  return {
    data,
    error,
    isLoading,
    execute,
  }
}

// Usage in component:
<script setup lang="ts">
import { onMounted } from 'vue'
import { useApi } from '@/composables/useApi'
import { userService } from '@/services/user.service'

const { data: user, isLoading, error, execute } = useApi(() =>
  userService.getProfile()
)

onMounted(() => {
  execute()
})
</script>

<template>
  <div v-if="isLoading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else-if="user">
    <h1>{{ user.name }}</h1>
  </div>
</template>
```

---

## 🔄 Auto-Retry with Exponential Backoff

```typescript
// src/services/api.ts (add to ApiClient class)
private async retryRequest<T>(
  fn: () => Promise<T>,
  retries: number = 3,
  delay: number = 1000
): Promise<T> {
  try {
    return await fn()
  } catch (error) {
    if (retries === 0) throw error

    await new Promise(resolve => setTimeout(resolve, delay))

    return this.retryRequest(fn, retries - 1, delay * 2)  // Exponential backoff
  }
}

// Usage:
async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
  return this.retryRequest(() => this.client.get<T>(url, config).then(r => r.data))
}
```

---

## 🚨 Error Handling Pattern

```typescript
// src/composables/useErrorHandler.ts
import { useToast } from 'primevue/usetoast'

export function useErrorHandler() {
  const toast = useToast()

  function handleError(error: any, context?: string) {
    const message = error?.response?.data?.message || error?.message || 'An error occurred'

    toast.add({
      severity: 'error',
      summary: context || 'Error',
      detail: message,
      life: 5000,
    })

    console.error(`[${context}]`, error)
  }

  return { handleError }
}

// Usage in component:
const { handleError } = useErrorHandler()

async function saveChanges() {
  try {
    await userService.updateProfile(formData.value)
    toast.add({ severity: 'success', summary: 'Saved', detail: 'Profile updated' })
  } catch (error) {
    handleError(error, 'Save Profile')
  }
}
```

---

## 📡 Real-Time Updates (WebSocket)

```typescript
// src/services/websocket.service.ts
import { ref } from 'vue'
import type { Video } from '@/types/models'

class WebSocketService {
  private ws: WebSocket | null = null
  private reconnectAttempts = 0
  private maxReconnectAttempts = 5

  connect(token: string) {
    const wsUrl = import.meta.env.VITE_WS_URL || 'ws://localhost:3000'
    this.ws = new WebSocket(`${wsUrl}?token=${token}`)

    this.ws.onopen = () => {
      console.log('WebSocket connected')
      this.reconnectAttempts = 0
    }

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data)
      this.handleMessage(message)
    }

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error)
    }

    this.ws.onclose = () => {
      console.log('WebSocket closed')
      this.reconnect(token)
    }
  }

  private reconnect(token: string) {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++
      setTimeout(() => this.connect(token), 1000 * this.reconnectAttempts)
    }
  }

  private handleMessage(message: any) {
    // Emit events for components to listen to
    window.dispatchEvent(new CustomEvent('ws-message', { detail: message }))
  }

  send(data: any) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data))
    }
  }

  disconnect() {
    this.ws?.close()
  }
}

export const wsService = new WebSocketService()
```

---

## ✅ Implementation Checklist

- [ ] Create Axios client with base URL
- [ ] Add request interceptor for auth token
- [ ] Add response interceptor for token refresh
- [ ] Implement error handling
- [ ] Create service modules (user, video, etc.)
- [ ] Add retry logic with exponential backoff
- [ ] Create useApi composable
- [ ] Add error toast notifications
- [ ] Set up WebSocket for real-time updates (if needed)
- [ ] Add request cancellation for component unmount
- [ ] Implement request deduplication
- [ ] Add loading indicators

---

## 🔗 Related Patterns

→ [FRONTEND_PATTERNS.md](./FRONTEND_PATTERNS.md) - Overall frontend structure
→ [AUTHENTICATION_STRATEGY.md](../backend/AUTHENTICATION_STRATEGY.md) - Backend JWT auth

---

## 🎓 How AI Uses This Pattern

When you request: **"Connect dashboard to backend API"**

**AI will:**
1. Create Axios client with interceptors
2. Add auth token injection
3. Implement token refresh logic
4. Create service modules for API endpoints
5. Add error handling with toasts
6. Create useApi composable
7. Include retry logic
8. Add loading states

**Result:** Full API integration in 1-2 hours

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
