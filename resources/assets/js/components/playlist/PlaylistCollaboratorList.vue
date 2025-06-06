<template>
  <ListSkeleton v-if="loading" />
  <ul v-else class="w-full space-y-3">
    <ListItem
      v-for="collaborator in collaborators"
      :key="collaborator.id"
      :collaborator="collaborator"
      :manageable="currentUserIsOwner"
      :removable="currentUserIsOwner && collaborator.id !== playlist.owner_id"
      :role="collaborator.id === playlist.owner_id ? 'owner' : 'contributor'"
      @remove="removeCollaborator(collaborator)"
    />
  </ul>
</template>

<script lang="ts" setup>
import { sortBy } from 'lodash'
import type { Ref } from 'vue'
import { computed, onMounted, ref, toRefs } from 'vue'
import { playlistCollaborationService } from '@/services/playlistCollaborationService'
import { eventBus } from '@/utils/eventBus'
import { useAuthorization } from '@/composables/useAuthorization'
import { useDialogBox } from '@/composables/useDialogBox'
import { useErrorHandler } from '@/composables/useErrorHandler'

import ListSkeleton from '@/components/ui/skeletons/PlaylistCollaboratorListSkeleton.vue'
import ListItem from '@/components/playlist/PlaylistCollaboratorListItem.vue'

const props = defineProps<{ playlist: Playlist }>()
const { playlist } = toRefs(props)

const { currentUser } = useAuthorization()
const { showConfirmDialog } = useDialogBox()

const collaborators: Ref<PlaylistCollaborator[]> = ref([])
const loading = ref(false)

const currentUserIsOwner = computed(() => currentUser.value?.id === playlist.value.owner_id)

const fetchCollaborators = async () => {
  loading.value = true

  try {
    collaborators.value = sortBy(
      await playlistCollaborationService.fetchCollaborators(playlist.value),
      ({ id }) => {
        if (id === currentUser.value.id) {
          return 0
        }
        if (id === playlist.value.owner_id) {
          return 1
        }
        return 2
      },
    )
  } finally {
    loading.value = false
  }
}

const removeCollaborator = async (collaborator: PlaylistCollaborator) => {
  const deadSure = await showConfirmDialog(
    `Remove ${collaborator.name} as a collaborator? This will remove their contributions as well.`,
  )

  if (!deadSure) {
    return
  }

  try {
    collaborators.value = collaborators.value.filter(({ id }) => id !== collaborator.id)
    await playlistCollaborationService.removeCollaborator(playlist.value, collaborator)
    eventBus.emit('PLAYLIST_COLLABORATOR_REMOVED', playlist.value)
  } catch (error: unknown) {
    useErrorHandler().handleHttpError(error)
  }
}

onMounted(async () => await fetchCollaborators())
</script>
