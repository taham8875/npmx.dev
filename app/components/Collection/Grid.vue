<script setup lang="ts">
import type { CollectionSummary } from '#shared/types/social'

const { skeletons = 6 } = defineProps<{
  items: CollectionSummary[]
  /** Shows skeletons while there are no items yet */
  pending: boolean
  loadingMore: boolean
  error: boolean
  hasMore: boolean
  emptyText: string
  skeletons?: number
}>()

const emit = defineEmits<{ loadMore: [] }>()
</script>

<template>
  <div
    v-if="pending && items.length === 0"
    class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3"
  >
    <SkeletonBlock v-for="i in skeletons" :key="i" class="h-32 rounded-lg" />
  </div>
  <p v-else-if="error" role="alert" class="text-fg-muted">{{ $t('collections.load_error') }}</p>
  <p v-else-if="items.length === 0" class="text-fg-muted">{{ emptyText }}</p>
  <template v-else>
    <ul class="m-0 grid list-none grid-cols-1 gap-4 p-0 md:grid-cols-2 lg:grid-cols-3">
      <li v-for="collection in items" :key="collection.uri">
        <CollectionCard :collection />
      </li>
    </ul>
    <ButtonBase
      v-if="hasMore"
      class="self-center"
      :disabled="loadingMore"
      @click="emit('loadMore')"
    >
      {{ loadingMore ? $t('common.loading_more') : $t('common.load_more') }}
    </ButtonBase>
  </template>
</template>
