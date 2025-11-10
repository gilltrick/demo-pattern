<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Frontend Patterns
## Modern Vue 3 + TypeScript Architecture

> **For AI:** Patterns for building scalable, type-safe frontend applications with Vue 3 Composition API.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Quick Reference

**Stack:** Vue 3 + TypeScript + Vite

**UI Library:** PrimeVue (production-ready components)

**State:** Composition API + Pinia (if complex state)

**HTTP:** Axios with interceptors

**Routing:** Vue Router 4

---

## 📂 Standard Project Structure

```
frontend-app/
├── src/
│   ├── components/           # Reusable components
│   │   ├── common/          # Buttons, inputs, cards
│   │   └── features/        # Feature-specific components
│   ├── views/               # Page components (routes)
│   ├── composables/         # Shared composition functions
│   ├── services/            # API clients, business logic
│   ├── types/               # TypeScript interfaces
│   ├── router/              # Route definitions
│   ├── assets/              # CSS, images
│   ├── App.vue
│   └── main.ts
├── public/
├── vite.config.ts
├── tsconfig.json
└── package.json
```

---

## ⚡ Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
})
```

---

## 🔧 TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "moduleResolution": "node",
    "strict": true,
    "jsx": "preserve",
    "sourceMap": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.tsx", "src/**/*.vue"]
}
```

---

## 🎨 App Entry Point

```typescript
// src/main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import PrimeVue from 'primevue/config'
import App from './App.vue'
import router from './router'

// Global styles
import 'primevue/resources/themes/lara-dark-blue/theme.css'
import 'primevue/resources/primevue.min.css'
import 'primeicons/primeicons.css'
import './style.css'

const app = createApp(App)

app.use(createPinia())
app.use(router)
app.use(PrimeVue)

app.mount('#app')
```

---

## 🧩 Component Structure (Standard Pattern)

```vue
<script setup lang="ts">
// 1. Imports
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import Button from 'primevue/button'

// 2. Props
interface Props {
  userId: string
  initialTab?: string
}
const props = withDefaults(defineProps<Props>(), {
  initialTab: 'overview'
})

// 3. Emits
interface Emits {
  (e: 'update', data: any): void
  (e: 'close'): void
}
const emit = defineEmits<Emits>()

// 4. Composables
const router = useRouter()

// 5. State
const isLoading = ref(false)
const userData = ref<User | null>(null)

// 6. Computed
const displayName = computed(() => {
  return userData.value?.name || 'Loading...'
})

// 7. Methods
async function loadUser() {
  isLoading.value = true
  try {
    userData.value = await userService.getUser(props.userId)
  } catch (error) {
    console.error('Failed to load user:', error)
  } finally {
    isLoading.value = false
  }
}

function handleUpdate(data: any) {
  emit('update', data)
}

// 8. Lifecycle
onMounted(() => {
  loadUser()
})
</script>

<template>
  <div class="user-dashboard">
    <div v-if="isLoading" class="loading">
      Loading...
    </div>

    <div v-else-if="userData" class="content">
      <h1>{{ displayName }}</h1>
      <Button label="Save" @click="handleUpdate" />
    </div>
  </div>
</template>

<style scoped>
.user-dashboard {
  padding: var(--spacing-4);
}

.loading {
  display: flex;
  justify-content: center;
  padding: var(--spacing-8);
}

/* Mobile responsive */
@media (max-width: 768px) {
  .user-dashboard {
    padding: var(--spacing-2);
  }
}
</style>
```

---

## 📦 TypeScript Types

```typescript
// src/types/models.ts
export interface User {
  id: string
  email: string
  name: string
  createdAt: string
}

export interface Video {
  id: string
  title: string
  status: 'processing' | 'completed' | 'failed'
  progress: number
  thumbnailUrl?: string
}

export interface ApiResponse<T> {
  success: boolean
  data: T
  message?: string
  error?: string
}
```

---

## 🗂️ Router Configuration

```typescript
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      name: 'home',
      component: () => import('@/views/Home.vue'),
    },
    {
      path: '/dashboard',
      name: 'dashboard',
      component: () => import('@/views/Dashboard.vue'),
      meta: { requiresAuth: true },
    },
    {
      path: '/login',
      name: 'login',
      component: () => import('@/views/Login.vue'),
    },
  ],
})

// Auth guard
router.beforeEach((to, from, next) => {
  const authStore = useAuthStore()

  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    next({ name: 'login', query: { redirect: to.fullPath } })
  } else {
    next()
  }
})

export default router
```

---

## 💾 State Management with Pinia

```typescript
// src/stores/auth.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { User } from '@/types/models'

export const useAuthStore = defineStore('auth', () => {
  // State
  const user = ref<User | null>(null)
  const accessToken = ref<string | null>(localStorage.getItem('accessToken'))

  // Getters
  const isAuthenticated = computed(() => !!accessToken.value)

  // Actions
  function setAuth(token: string, userData: User) {
    accessToken.value = token
    user.value = userData
    localStorage.setItem('accessToken', token)
  }

  function logout() {
    accessToken.value = null
    user.value = null
    localStorage.removeItem('accessToken')
  }

  return {
    user,
    accessToken,
    isAuthenticated,
    setAuth,
    logout,
  }
})
```

---

## 🎨 Theme System

```css
/* src/assets/variables.css */
:root {
  /* Colors */
  --color-primary: #6366f1;
  --color-secondary: #a855f7;
  --color-success: #22c55e;
  --color-warning: #f59e0b;
  --color-danger: #ef4444;

  --color-background: #1e1e2e;
  --color-surface: #262637;
  --color-text: #cdd6f4;
  --color-text-secondary: #9399b2;

  /* Spacing */
  --spacing-1: 0.25rem;
  --spacing-2: 0.5rem;
  --spacing-3: 0.75rem;
  --spacing-4: 1rem;
  --spacing-6: 1.5rem;
  --spacing-8: 2rem;

  /* Border radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 1rem;

  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-base: 250ms ease;
  --transition-slow: 350ms ease;
}

[data-theme="light"] {
  --color-background: #ffffff;
  --color-surface: #f3f4f6;
  --color-text: #1f2937;
  --color-text-secondary: #6b7280;
}
```

---

## ✅ Implementation Checklist

- [ ] Set up Vite + Vue 3 + TypeScript
- [ ] Install PrimeVue and configure
- [ ] Create standard folder structure
- [ ] Set up Vue Router with auth guards
- [ ] Add Pinia for state management
- [ ] Create TypeScript interfaces for data models
- [ ] Configure Axios with interceptors (see API_INTEGRATION.md)
- [ ] Add theme system with CSS variables
- [ ] Create reusable components
- [ ] Set up error boundaries
- [ ] Add loading states
- [ ] Implement responsive design
- [ ] Configure ESLint + Prettier

---

## 🔗 Related Patterns

→ [API_INTEGRATION.md](./API_INTEGRATION.md) - Backend API communication
→ [COMPONENT_PATTERNS.md](./COMPONENT_PATTERNS.md) - Component best practices

---

## 🎓 How AI Uses This Pattern

When you request: **"Create a user dashboard"**

**AI will:**
1. Generate standard project structure
2. Set up Vite config with proxy
3. Configure TypeScript with strict mode
4. Create router with auth guards
5. Set up Pinia stores for state
6. Create dashboard view with TypeScript
7. Add API integration
8. Include responsive styling

**Result:** Production-ready frontend in 3-4 hours

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
