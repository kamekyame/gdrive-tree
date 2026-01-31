<script setup lang="ts">
import { computed } from 'vue'
import {
  FileText,
  Table2,
  Presentation,
  ClipboardList,
  Image,
  File,
  Folder,
  FolderOpen,
  Users,
} from 'lucide-vue-next'

export type DriveNode = {
  id: string
  name: string
  mimeType: string
  url?: string
  editUrl?: string
  isMyDrive?: boolean
  shared?: boolean
  children?: DriveNode[]
}

type Props = {
  nodes: DriveNode[]
  selectedId: string | null
  expandedIds: Set<string>
}

type Emits = {
  (event: 'select', id: string): void
  (event: 'toggle', id: string): void
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

const isFolder = (node: DriveNode) =>
  node.mimeType === 'application/vnd.google-apps.folder'

const isExpanded = (node: DriveNode) => props.expandedIds.has(node.id)

const iconFor = (node: DriveNode) => {
  if (isFolder(node)) {
    return isExpanded(node) ? FolderOpen : Folder
  }

  switch (node.mimeType) {
    case 'application/vnd.google-apps.document':
      return FileText
    case 'application/vnd.google-apps.spreadsheet':
      return Table2
    case 'application/vnd.google-apps.presentation':
      return Presentation
    case 'application/vnd.google-apps.form':
      return ClipboardList
    case 'application/pdf':
      return FileText
    default:
      if (node.mimeType.startsWith('image/')) return Image
      return File
  }
}

const sortedNodes = computed(() =>
  [...props.nodes].sort((a, b) => {
    if (isFolder(a) && !isFolder(b)) return -1
    if (!isFolder(a) && isFolder(b)) return 1
    return a.name.localeCompare(b.name)
  }),
)
</script>

<template>
  <ul class="tree-root">
    <li v-for="node in sortedNodes" :key="node.id" class="tree-item">
      <button
        v-if="isFolder(node)"
        class="tree-folder"
        :class="{ expanded: isExpanded(node) }"
        @click="emit('toggle', node.id)"
      >
        <span class="tree-caret">▸</span>
        <component :is="iconFor(node)" class="tree-icon" />
        <span class="tree-name" :title="node.name">{{ node.name }}</span>
        <span v-if="node.shared" class="tree-share" title="共有中">
          <Users class="tree-share-icon" />
        </span>
      </button>
      <button
        v-else
        class="tree-button"
        :class="{ active: selectedId === node.id }"
        @click="emit('select', node.id)"
      >
        <component :is="iconFor(node)" class="tree-icon" />
        <span class="tree-name" :title="node.name">{{ node.name }}</span>
        <span v-if="node.shared" class="tree-share" title="共有中">
          <Users class="tree-share-icon" />
        </span>
      </button>

      <div v-if="node.children?.length && isExpanded(node)" class="tree-children">
        <DriveTree
          :nodes="node.children"
          :selected-id="selectedId"
          :expanded-ids="expandedIds"
          @select="emit('select', $event)"
          @toggle="emit('toggle', $event)"
        />
      </div>
    </li>
  </ul>
</template>
