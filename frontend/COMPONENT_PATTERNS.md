<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Component Patterns
## Reusable Vue 3 Component Best Practices

> **For AI:** Standardized component structure, TypeScript patterns, and accessibility for Vue 3.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Component Organization

```vue
<script setup lang="ts">
// Order matters for readability:
// 1. Imports (external, then internal)
// 2. Props interface + defineProps
// 3. Emits interface + defineEmits
// 4. Composables (router, stores, etc.)
// 5. Reactive state (ref, reactive)
// 6. Computed properties
// 7. Methods/functions
// 8. Lifecycle hooks (onMounted, onUnmounted, etc.)
// 9. Watchers (watch, watchEffect)
</script>

<template>
  <!-- Template structure:
    1. Loading states
    2. Error states
    3. Empty states
    4. Main content
  -->
</template>

<style scoped>
/* Scoped styles:
  - Use CSS variables for theming
  - Mobile-first responsive design
  - BEM naming for clarity
*/
</style>
```

---

## 📦 Props & Emits (TypeScript)

```vue
<script setup lang="ts">
import { computed } from 'vue'

// Props with TypeScript interface
interface Props {
  title: string                  // Required
  description?: string           // Optional
  maxItems?: number             // Optional with default
  items: Item[]                 // Complex type
  variant?: 'primary' | 'secondary'  // Union type
}

const props = withDefaults(defineProps<Props>(), {
  maxItems: 10,
  variant: 'primary',
})

// Emits with TypeScript interface
interface Emits {
  (e: 'update', value: string): void
  (e: 'delete', id: number): void
  (e: 'select', item: Item): void
}

const emit = defineEmits<Emits>()

// Use props and emits
const displayItems = computed(() =>
  props.items.slice(0, props.maxItems)
)

function handleClick(item: Item) {
  emit('select', item)
}
</script>
```

---

## 🎨 Responsive Component Example

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import Button from 'primevue/button'
import Card from 'primevue/card'

interface VideoCardProps {
  video: Video
  showActions?: boolean
}

const props = withDefaults(defineProps<VideoCardProps>(), {
  showActions: true,
})

interface Emits {
  (e: 'play', videoId: string): void
  (e: 'delete', videoId: string): void
}

const emit = defineEmits<Emits>()

const isHovered = ref(false)

const statusColor = computed(() => {
  const colors = {
    processing: 'var(--color-warning)',
    completed: 'var(--color-success)',
    failed: 'var(--color-danger)',
  }
  return colors[props.video.status]
})

const progressPercentage = computed(() => {
  return `${props.video.progress}%`
})

function handlePlay() {
  emit('play', props.video.id)
}

function handleDelete() {
  emit('delete', props.video.id)
}
</script>

<template>
  <Card
    class="video-card"
    :class="{ 'video-card--hovered': isHovered }"
    @mouseenter="isHovered = true"
    @mouseleave="isHovered = false"
  >
    <template #header>
      <div class="video-card__thumbnail">
        <img
          v-if="video.thumbnailUrl"
          :src="video.thumbnailUrl"
          :alt="`Thumbnail for ${video.title}`"
          class="video-card__image"
        />
        <div v-else class="video-card__placeholder">
          <i class="pi pi-video"></i>
        </div>

        <!-- Status badge -->
        <div
          class="video-card__status"
          :style="{ backgroundColor: statusColor }"
        >
          {{ video.status }}
        </div>

        <!-- Progress bar for processing videos -->
        <div
          v-if="video.status === 'processing'"
          class="video-card__progress"
        >
          <div
            class="video-card__progress-bar"
            :style="{ width: progressPercentage }"
          ></div>
        </div>
      </div>
    </template>

    <template #title>
      <h3 class="video-card__title">{{ video.title }}</h3>
    </template>

    <template #content>
      <p class="video-card__meta">
        Duration: {{ formatDuration(video.duration) }}
      </p>
    </template>

    <template #footer>
      <div v-if="showActions" class="video-card__actions">
        <Button
          label="Play"
          icon="pi pi-play"
          @click="handlePlay"
          :disabled="video.status !== 'completed'"
        />
        <Button
          label="Delete"
          icon="pi pi-trash"
          class="p-button-danger p-button-outlined"
          @click="handleDelete"
        />
      </div>
    </template>
  </Card>
</template>

<style scoped>
.video-card {
  transition: transform var(--transition-base), box-shadow var(--transition-base);
}

.video-card--hovered {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
}

.video-card__thumbnail {
  position: relative;
  width: 100%;
  padding-top: 56.25%; /* 16:9 aspect ratio */
  background: var(--color-surface);
  overflow: hidden;
}

.video-card__image {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.video-card__placeholder {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 3rem;
  color: var(--color-text-secondary);
}

.video-card__status {
  position: absolute;
  top: var(--spacing-2);
  right: var(--spacing-2);
  padding: var(--spacing-1) var(--spacing-2);
  border-radius: var(--radius-sm);
  color: white;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
}

.video-card__progress {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 4px;
  background: rgba(0, 0, 0, 0.2);
}

.video-card__progress-bar {
  height: 100%;
  background: var(--color-primary);
  transition: width var(--transition-base);
}

.video-card__title {
  font-size: 1.25rem;
  margin: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.video-card__meta {
  color: var(--color-text-secondary);
  font-size: 0.875rem;
  margin: 0;
}

.video-card__actions {
  display: flex;
  gap: var(--spacing-2);
}

/* Mobile responsive */
@media (max-width: 768px) {
  .video-card__actions {
    flex-direction: column;
  }

  .video-card__title {
    font-size: 1rem;
  }
}
</style>
```

---

## ♿ Accessibility Patterns

```vue
<template>
  <!-- Semantic HTML -->
  <button
    type="button"
    :aria-label="`Delete video ${video.title}`"
    :aria-disabled="isDeleting"
    @click="handleDelete"
  >
    <i class="pi pi-trash" aria-hidden="true"></i>
    <span class="sr-only">Delete</span>
  </button>

  <!-- Form inputs -->
  <div class="form-field">
    <label for="email-input">Email Address</label>
    <input
      id="email-input"
      v-model="email"
      type="email"
      :aria-invalid="hasError"
      :aria-describedby="hasError ? 'email-error' : undefined"
    />
    <span v-if="hasError" id="email-error" role="alert">
      Please enter a valid email
    </span>
  </div>

  <!-- Loading states -->
  <div v-if="isLoading" role="status" aria-live="polite">
    Loading videos...
  </div>

  <!-- Screen reader only text -->
  <span class="sr-only">{{ srText }}</span>
</template>

<style>
/* Screen reader only - visually hidden but accessible */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
</style>
```

---

## 🎣 Composable Integration

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useApi } from '@/composables/useApi'
import { useErrorHandler } from '@/composables/useErrorHandler'
import { videoService } from '@/services/video.service'

const props = defineProps<{ userId: string }>()

// Composables
const { data: videos, isLoading, error, execute } = useApi(() =>
  videoService.getUserVideos(props.userId)
)

const { handleError } = useErrorHandler()

onMounted(async () => {
  try {
    await execute()
  } catch (err) {
    handleError(err, 'Load Videos')
  }
})
</script>

<template>
  <div class="videos-container">
    <div v-if="isLoading" class="loading">
      <i class="pi pi-spin pi-spinner"></i>
      Loading videos...
    </div>

    <div v-else-if="error" class="error">
      <i class="pi pi-exclamation-circle"></i>
      Failed to load videos
    </div>

    <div v-else-if="videos?.length === 0" class="empty">
      <i class="pi pi-video"></i>
      <p>No videos yet</p>
    </div>

    <div v-else class="videos-grid">
      <VideoCard
        v-for="video in videos"
        :key="video.id"
        :video="video"
        @play="handlePlay"
        @delete="handleDelete"
      />
    </div>
  </div>
</template>
```

---

## 📊 Data Table Component Pattern

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import DataTable from 'primevue/datatable'
import Column from 'primevue/column'

interface TableProps<T> {
  data: T[]
  columns: TableColumn[]
  loading?: boolean
  paginate?: boolean
}

interface TableColumn {
  field: string
  header: string
  sortable?: boolean
  filterable?: boolean
}

const props = withDefaults(defineProps<TableProps<any>>(), {
  loading: false,
  paginate: true,
})

const emit = defineEmits<{
  (e: 'row-select', row: any): void
  (e: 'row-delete', row: any): void
}>()

const selectedRows = ref([])
const filters = ref({})

const paginatorTemplate = 'FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink RowsPerPageDropdown'
</script>

<template>
  <DataTable
    :value="data"
    v-model:selection="selectedRows"
    :loading="loading"
    :paginator="paginate"
    :rows="10"
    :rows-per-page-options="[10, 25, 50]"
    :paginator-template="paginatorTemplate"
    stripedRows
    @row-select="emit('row-select', $event.data)"
  >
    <Column
      v-for="col in columns"
      :key="col.field"
      :field="col.field"
      :header="col.header"
      :sortable="col.sortable"
      :filterable="col.filterable"
    ></Column>

    <Column header="Actions">
      <template #body="slotProps">
        <Button
          icon="pi pi-trash"
          class="p-button-danger p-button-text"
          @click="emit('row-delete', slotProps.data)"
        />
      </template>
    </Column>
  </DataTable>
</template>
```

---

## ✅ Component Checklist

Every component should have:

- [ ] TypeScript interfaces for props and emits
- [ ] Default values for optional props
- [ ] Proper component organization (imports → state → methods → lifecycle)
- [ ] Loading, error, and empty states
- [ ] Accessibility attributes (aria-label, role, etc.)
- [ ] Responsive design (mobile-first)
- [ ] Scoped styles with CSS variables
- [ ] Meaningful event names
- [ ] Computed properties for derived data
- [ ] Cleanup in onUnmounted (if needed)

---

## 🔗 Related Patterns

→ [FRONTEND_PATTERNS.md](./FRONTEND_PATTERNS.md) - Overall frontend architecture
→ [API_INTEGRATION.md](./API_INTEGRATION.md) - API calls in components

---

## 🎓 How AI Uses This Pattern

When you request: **"Create a video list component with cards"**

**AI will:**
1. Create TypeScript interfaces for props/emits
2. Use PrimeVue Card component
3. Add loading/error/empty states
4. Implement responsive grid layout
5. Add accessibility attributes
6. Include hover effects
7. Emit events for user actions
8. Add progress bar for processing videos

**Result:** Production-ready component in 30-45 minutes

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
