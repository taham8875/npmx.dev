<script setup lang="ts">
import type { CollectionView, ReactionState } from '#shared/types/social'
import type { CommandPaletteContextCommandInput } from '~/types/command-palette'
import {
  collectionPath,
  deleteCollection,
  ensureCollectionWriteAccess,
  isConflict,
  updateCollection,
  type CollectionBody,
} from '~/utils/atproto/collections'

const route = useRoute('collections-identifier-rkey')
const identifier = computed(() => route.params.identifier)
const rkey = computed(() => route.params.rkey)

const {
  data: collection,
  error,
  refresh,
} = await useFetch<CollectionView>(() => collectionPath(identifier.value, rkey.value))
if (!collection.value || error.value) {
  // useFetch nulls `data` on any error, so only the status tells a missing record from a down PDS
  const notFound = !error.value || error.value.statusCode === 404 || error.value.statusCode === 400
  throw createError({
    statusCode: notFound ? 404 : 502,
    statusMessage: notFound ? $t('collections.not_found') : $t('collections.load_failed'),
    message: notFound ? $t('collections.not_found_message') : $t('collections.load_failed_message'),
  })
}

const { user } = useAtproto()
const isOwner = computed(() => !!user.value?.did && user.value.did === collection.value?.did)

// useFetch data is a shallow ref, so replace rather than mutate
const reactionModel = (key: 'likes' | 'follows') =>
  computed<ReactionState>({
    get: () => collection.value?.[key] ?? { total: 0, viewerHas: false },
    set: value => {
      if (collection.value) collection.value = { ...collection.value, [key]: value }
    },
  })
const likes = reactionModel('likes')
const follows = reactionModel('follows')

const isEditing = shallowRef(false)
async function startEditing() {
  await ensureCollectionWriteAccess(user.value?.handle, route.fullPath)
  isEditing.value = true
}
const isSaving = shallowRef(false)
const saveError = shallowRef('')

async function save(body: CollectionBody) {
  if (!collection.value) return
  isSaving.value = true
  saveError.value = ''
  try {
    await updateCollection(
      collection.value.did,
      collection.value.rkey,
      { ...body, cid: collection.value.cid },
      user.value?.handle,
    )
  } catch (e) {
    saveError.value = isConflict(e)
      ? $t('collections.editor.conflict')
      : $t('collections.editor.save_error')
    return
  } finally {
    isSaving.value = false
  }
  isEditing.value = false
  await refresh()
}

const deleteModal = useTemplateRef('deleteModal')
const isDeleting = shallowRef(false)
const deleteError = shallowRef('')
function confirmDelete() {
  deleteError.value = ''
  deleteModal.value?.showModal()
}
async function remove() {
  if (!collection.value) return
  const { did, rkey: recordKey, handle } = collection.value
  isDeleting.value = true
  deleteError.value = ''
  try {
    await deleteCollection(did, recordKey, user.value?.handle)
  } catch {
    deleteError.value = $t('collections.delete_error')
    return
  } finally {
    isDeleting.value = false
  }
  deleteModal.value?.close()
  await navigateTo({ name: 'profile-identity', params: { identity: handle } })
}

const compactNumberFormatter = useCompactNumberFormatter()
const duplicateRoute = computed(() => ({
  name: 'collections-new' as const,
  query: { from: `${collection.value?.did}/${collection.value?.rkey}` },
}))

const likeButton = useTemplateRef('likeButton')
const followButton = useTemplateRef('followButton')
useCommandPaletteContextCommands(
  computed((): CommandPaletteContextCommandInput[] => {
    const current = collection.value
    if (!current) return []
    const keywords = [current.name, current.handle]
    const commands: CommandPaletteContextCommandInput[] = [
      {
        id: 'collection-like',
        group: 'actions',
        label: current.likes.viewerHas ? $t('collections.unlike') : $t('collections.like'),
        keywords,
        iconClass: current.likes.viewerHas ? 'i-lucide:heart-minus' : 'i-lucide:heart-plus',
        action: () => likeButton.value?.toggle(),
      },
    ]
    if (!isOwner.value) {
      commands.push({
        id: 'collection-follow',
        group: 'actions',
        label: current.follows.viewerHas
          ? $t('collections.unfollow_label')
          : $t('collections.follow_label'),
        keywords,
        iconClass: current.follows.viewerHas ? 'i-lucide:bell-ring' : 'i-lucide:bell-plus',
        action: () => followButton.value?.toggle(),
      })
    }
    commands.push({
      id: 'collection-duplicate',
      group: 'actions',
      label: $t('collections.duplicate'),
      keywords,
      iconClass: 'i-lucide:copy',
      to: duplicateRoute.value,
    })
    if (isOwner.value && !isEditing.value) {
      commands.push(
        {
          id: 'collection-edit',
          group: 'actions',
          label: $t('common.edit'),
          keywords,
          iconClass: 'i-lucide:square-pen',
          action: startEditing,
        },
        {
          id: 'collection-delete',
          group: 'actions',
          label: $t('collections.delete_title'),
          keywords,
          iconClass: 'i-lucide:trash',
          action: confirmDelete,
        },
      )
    }
    return commands
  }),
)

const seoTitle = () => $t('collections.seo_title', { name: collection.value?.name ?? '' })
const seoDescription = () =>
  collection.value?.description ||
  $t('collections.seo_description', {
    name: collection.value?.name ?? '',
    handle: collection.value?.handle ?? '',
  })
useSeoMeta({
  title: seoTitle,
  ogTitle: seoTitle,
  twitterTitle: seoTitle,
  description: seoDescription,
  ogDescription: seoDescription,
  twitterDescription: seoDescription,
})

defineOgImage(
  'Page.takumi',
  {
    title: () => collection.value?.name ?? '',
    description: () => $t('collections.by', { handle: collection.value?.handle ?? '' }),
  },
  { alt: seoDescription },
)
</script>

<template>
  <main v-if="collection" class="container flex-1 flex flex-col gap-8 py-8 sm:py-12 w-full">
    <header class="flex flex-col gap-4 border-b border-border pb-8">
      <h1 class="font-mono text-2xl sm:text-3xl font-medium min-w-0 break-words">
        {{ collection.name }}
      </h1>
      <p class="font-mono text-sm">
        <LinkBase :to="{ name: 'profile-identity', params: { identity: collection.handle } }">
          {{ $t('collections.by', { handle: collection.handle }) }}
        </LinkBase>
      </p>
      <p v-if="collection.description" class="text-fg-muted">{{ collection.description }}</p>

      <div class="flex flex-wrap items-center gap-3">
        <CollectionLikeButton
          ref="likeButton"
          v-model="likes"
          :identifier="collection.did"
          :rkey="collection.rkey"
        />
        <!-- Owners follow their own collections implicitly -->
        <CollectionFollowButton
          v-if="!isOwner"
          ref="followButton"
          v-model="follows"
          :identifier="collection.did"
          :rkey="collection.rkey"
        />
        <LinkBase variant="button-secondary" classicon="i-lucide:copy" :to="duplicateRoute">
          {{ $t('collections.duplicate') }}
        </LinkBase>
        <template v-if="isOwner && !isEditing">
          <ButtonBase classicon="i-lucide:square-pen" @click="startEditing">
            {{ $t('common.edit') }}
          </ButtonBase>
          <ButtonBase classicon="i-lucide:trash" @click="confirmDelete">
            {{ $t('common.delete') }}
          </ButtonBase>
        </template>
        <LinkBase
          v-if="collection.includedInCount > 0"
          variant="button-secondary"
          classicon="i-lucide:library"
          :to="{ name: 'collections', query: { includes: `${collection.did}/${collection.rkey}` } }"
        >
          {{
            $t(
              'collections.in_collections',
              { count: compactNumberFormatter.format(collection.includedInCount) },
              collection.includedInCount,
            )
          }}
        </LinkBase>
      </div>
    </header>

    <CollectionEditor
      v-if="isEditing"
      :initial="collection"
      :pending="isSaving"
      :error-message="saveError"
      @submit="save"
      @cancel="isEditing = false"
    />
    <template v-else>
      <CollectionItemList v-if="collection.items.length > 0" :items="collection.items" />
      <p v-else class="text-fg-muted">{{ $t('collections.empty_collection') }}</p>
      <CollectionYourData class="mt-4" />
    </template>

    <Modal
      v-if="isOwner"
      id="delete-collection-modal"
      ref="deleteModal"
      :modal-title="$t('collections.delete_title')"
      class="max-w-md"
    >
      <p class="text-fg-muted">{{ $t('collections.delete_body') }}</p>
      <Alert v-if="deleteError" variant="error" class="mt-4">{{ deleteError }}</Alert>
      <div class="mt-6 flex justify-end gap-3">
        <ButtonBase @click="deleteModal?.close()">{{ $t('common.cancel') }}</ButtonBase>
        <ButtonBase
          variant="primary"
          classicon="i-lucide:trash"
          :disabled="isDeleting"
          @click="remove"
        >
          {{ $t('common.delete') }}
        </ButtonBase>
      </div>
    </Modal>
  </main>
</template>
