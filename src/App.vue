<script setup lang="ts">
import { computed, markRaw, onMounted, onUnmounted, ref, shallowRef, watch } from 'vue'
import DriveTree, { type DriveNode } from './components/DriveTree.vue'

type DriveFile = {
  id: string
  name: string
  mimeType: string
  parents?: string[]
  driveId?: string
  ownedByMe?: boolean
  shared?: boolean
  sharedWithMeTime?: string
  starred?: boolean
  viewedByMeTime?: string
  webViewLink?: string
  webContentLink?: string
}

type DriveInfo = {
  id: string
  name: string
}

type GoogleOAuth = {
  initTokenClient: (options: {
    client_id: string
    scope: string
    callback: (response: { access_token?: string; expires_in?: number }) => void
  }) => { requestAccessToken: (options?: { prompt?: string }) => void }
  revoke: (token: string, callback: () => void) => void
}

declare global {
  interface Window {
    google?: {
      accounts?: {
        oauth2?: GoogleOAuth
      }
    }
  }
}

const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID as string | undefined
const SCOPES = 'https://www.googleapis.com/auth/drive.readonly openid email profile'
const STORAGE_KEY = 'gdrive-tree-token'
const TOKEN_SKEW_MS = 60 * 1000
const BASE_PATH = import.meta.env.BASE_URL.replace(/\/$/, '')

const nodes = ref<DriveNode[]>([])
const selectedId = ref<string | null>(null)
const accessToken = ref<string | null>(null)
const loading = ref(false)
const error = ref<string | null>(null)
const expandedIds = ref<Set<string>>(new Set())
const drives = ref<DriveInfo[]>([])
const storageQuota = ref<{ usage?: number; limit?: number } | null>(null)
const userProfile = ref<{ name?: string; email?: string; picture?: string } | null>(
  null,
)
const showProfileMenu = ref(false)
const searchQuery = ref('')
const sidebarWidth = ref(280)
const tokenClient = shallowRef<
  { requestAccessToken: (options?: { prompt?: string }) => void } | null
>(null)

const selectedUrl = computed(() => {
  const findNode = (items: DriveNode[]): DriveNode | undefined => {
    for (const item of items) {
      if (item.id === selectedId.value) return item
      if (item.children) {
        const found = findNode(item.children)
        if (found) return found
      }
    }
    return undefined
  }

  return findNode(nodes.value)?.url ?? ''
})

const selectedEditUrl = computed(() => {
  const findNode = (items: DriveNode[]): DriveNode | undefined => {
    for (const item of items) {
      if (item.id === selectedId.value) return item
      if (item.children) {
        const found = findNode(item.children)
        if (found) return found
      }
    }
    return undefined
  }

  return findNode(nodes.value)?.editUrl ?? ''
})

const filteredNodes = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  if (!query) return nodes.value

  const filterTree = (items: DriveNode[]): DriveNode[] => {
    const result: DriveNode[] = []
    for (const item of items) {
      const nameMatch = item.name.toLowerCase().includes(query)
      const childMatches = item.children ? filterTree(item.children) : []
      if (nameMatch || childMatches.length > 0) {
        result.push({
          ...item,
          children: childMatches.length > 0 ? childMatches : item.children,
        })
      }
    }
    return result
  }

  return filterTree(nodes.value)
})

const collectFolderIds = (items: DriveNode[], result: Set<string>) => {
  for (const item of items) {
    if (item.mimeType === 'application/vnd.google-apps.folder') {
      result.add(item.id)
    }
    if (item.children?.length) {
      collectFolderIds(item.children, result)
    }
  }
}

watch([searchQuery, nodes], () => {
  const query = searchQuery.value.trim()
  if (!query) return
  const next = new Set<string>()
  collectFolderIds(filteredNodes.value, next)
  expandedIds.value = next
})

watch([nodes, selectedId], () => {
  expandToSelection()
})

const hasAuth = computed(() => Boolean(accessToken.value))

const loadGoogleIdentityScript = () =>
  new Promise<void>((resolve, reject) => {
    if (window.google?.accounts?.oauth2) {
      resolve()
      return
    }
    const script = document.createElement('script')
    script.src = 'https://accounts.google.com/gsi/client'
    script.async = true
    script.defer = true
    script.onload = () => resolve()
    script.onerror = () => reject(new Error('Google Identity の読み込みに失敗しました。'))
    document.head.appendChild(script)
  })

const saveToken = (token: string, expiresIn?: number) => {
  const expiresAt = expiresIn
    ? Date.now() + expiresIn * 1000 - TOKEN_SKEW_MS
    : Date.now() + 55 * 60 * 1000
  localStorage.setItem(
    STORAGE_KEY,
    JSON.stringify({ token, expiresAt, scopes: SCOPES }),
  )
}

const loadToken = () => {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (!raw) return null
  try {
    const parsed = JSON.parse(raw) as {
      token: string
      expiresAt: number
      scopes?: string
    }
    if (!parsed.token || !parsed.expiresAt) return null
    if (parsed.scopes !== SCOPES) {
      localStorage.removeItem(STORAGE_KEY)
      return null
    }
    if (Date.now() >= parsed.expiresAt) {
      localStorage.removeItem(STORAGE_KEY)
      return null
    }
    return parsed.token
  } catch {
    localStorage.removeItem(STORAGE_KEY)
    return null
  }
}

const clearToken = () => {
  localStorage.removeItem(STORAGE_KEY)
}

const buildTreeFromFiles = (files: DriveFile[]): DriveNode[] => {
  const map = new Map<string, DriveNode>()
  files.forEach((file) => {
    map.set(file.id, {
      id: file.id,
      name: file.name,
      mimeType: file.mimeType,
      url: createPreviewUrl(file),
      editUrl: createEditUrl(file),
      shared: file.shared || Boolean(file.sharedWithMeTime),
      children: [],
    })
  })

  const roots: DriveNode[] = []
  files.forEach((file) => {
    const node = map.get(file.id)
    if (!node) return

    const parentId = file.parents?.[0]
    const parent = parentId ? map.get(parentId) : undefined

    if (parent) {
      parent.children?.push(node)
    } else {
      roots.push(node)
    }
  })

  return roots
}

const buildFlatNodes = (files: DriveFile[]): DriveNode[] =>
  files.map((file) => ({
    id: file.id,
    name: file.name,
    mimeType: file.mimeType,
    url: createPreviewUrl(file),
    editUrl: createEditUrl(file),
    shared: file.shared || Boolean(file.sharedWithMeTime),
  }))

const buildTree = (files: DriveFile[], driveList: DriveInfo[]): DriveNode[] => {
  const myDriveFiles = files.filter((file) => file.ownedByMe && !file.driveId)
  const sharedWithMeFiles = files.filter(
    (file) => !file.ownedByMe && !file.driveId && file.sharedWithMeTime,
  )
  const starredFiles = files.filter((file) => file.starred)
  const recentFiles = files
    .filter((file) => file.viewedByMeTime)
    .sort(
      (a, b) =>
        new Date(b.viewedByMeTime ?? 0).getTime() -
        new Date(a.viewedByMeTime ?? 0).getTime(),
    )
    .slice(0, 20)

  const myDriveRoot: DriveNode = {
    id: 'my-drive-root',
    name: 'マイドライブ',
    mimeType: 'application/vnd.google-apps.folder',
    children: buildTreeFromFiles(myDriveFiles),
    isMyDrive: true,
  }

  const sharedWithMeRoot: DriveNode = {
    id: 'shared-with-me-root',
    name: '共有アイテム',
    mimeType: 'application/vnd.google-apps.folder',
    children: buildTreeFromFiles(sharedWithMeFiles),
  }

  const starredRoot: DriveNode = {
    id: 'starred-root',
    name: 'スター付き',
    mimeType: 'application/vnd.google-apps.folder',
    children: buildFlatNodes(starredFiles),
  }

  const recentRoot: DriveNode = {
    id: 'recent-root',
    name: '最近使用したアイテム',
    mimeType: 'application/vnd.google-apps.folder',
    children: buildFlatNodes(recentFiles),
  }

  const sharedDriveRoots: DriveNode[] = driveList
    .map((drive) => {
      const driveFiles = files.filter((file) => file.driveId === drive.id)
      return {
        id: `shared-drive-${drive.id}`,
        name: drive.name,
        mimeType: 'application/vnd.google-apps.folder',
        children: buildTreeFromFiles(driveFiles),
      }
    })
    .sort((a, b) => a.name.localeCompare(b.name))

  return [myDriveRoot, sharedWithMeRoot, starredRoot, recentRoot, ...sharedDriveRoots]
}

const createPreviewUrl = (file: DriveFile) => {
  if (file.mimeType.startsWith('application/vnd.google-apps')) {
    return `https://drive.google.com/file/d/${file.id}/preview`
  }
  return file.webViewLink ?? `https://drive.google.com/file/d/${file.id}/preview`
}

const createEditUrl = (file: DriveFile) => file.webViewLink ?? createPreviewUrl(file)

const formatBytes = (bytes?: number) => {
  if (!bytes || Number.isNaN(bytes)) return '-'
  const units = ['B', 'KB', 'MB', 'GB', 'TB']
  let value = bytes
  let index = 0
  while (value >= 1024 && index < units.length - 1) {
    value /= 1024
    index += 1
  }
  return `${value.toFixed(index === 0 ? 0 : 1)} ${units[index]}`
}

const updateUrlWithSelection = (id: string | null) => {
  const url = new URL(window.location.href)
  const nextPath = id ? `/file/d/${id}` : '/'
  url.pathname = `${BASE_PATH}${nextPath}`
  window.history.replaceState({}, '', url.toString())
}

const loadSelectionFromUrl = () => {
  const pathname = window.location.pathname
  const relativePath = pathname.startsWith(BASE_PATH)
    ? pathname.slice(BASE_PATH.length)
    : pathname
  const pathMatch = relativePath.match(/\/file\/d\/([^/]+)/)
  if (pathMatch?.[1]) {
    selectedId.value = pathMatch[1]
  }
}

const findPathToId = (items: DriveNode[], id: string, path: string[] = []) => {
  for (const item of items) {
    if (item.id === id) return path
    if (item.children?.length) {
      const found = findPathToId(item.children, id, [...path, item.id])
      if (found) return found
    }
  }
  return null
}

const expandToSelection = () => {
  if (!selectedId.value) return
  const path = findPathToId(nodes.value, selectedId.value)
  if (!path) return
  const next = new Set(expandedIds.value)
  path.forEach((id) => next.add(id))
  expandedIds.value = next
}

const handlePopState = () => {
  loadSelectionFromUrl()
  expandToSelection()
}

const listDriveFiles = async () => {
  if (!accessToken.value) return
  loading.value = true
  error.value = null
  try {
    const aboutUrl = new URL('https://www.googleapis.com/drive/v3/about')
    aboutUrl.searchParams.set('fields', 'storageQuota')
    const aboutResponse = await fetch(aboutUrl.toString(), {
      headers: {
        Authorization: `Bearer ${accessToken.value}`,
      },
    })

    if (aboutResponse.ok) {
      const aboutData = (await aboutResponse.json()) as {
        storageQuota?: { usage?: string; limit?: string }
      }
      storageQuota.value = {
        usage: aboutData.storageQuota?.usage
          ? Number(aboutData.storageQuota.usage)
          : undefined,
        limit: aboutData.storageQuota?.limit
          ? Number(aboutData.storageQuota.limit)
          : undefined,
      }
    }

    const drivesUrl = new URL('https://www.googleapis.com/drive/v3/drives')
    drivesUrl.searchParams.set('pageSize', '100')
    drivesUrl.searchParams.set('fields', 'drives(id,name)')
    const drivesResponse = await fetch(drivesUrl.toString(), {
      headers: {
        Authorization: `Bearer ${accessToken.value}`,
      },
    })

    if (drivesResponse.ok) {
      const driveData = (await drivesResponse.json()) as { drives: DriveInfo[] }
      drives.value = driveData.drives ?? []
    } else {
      drives.value = []
    }

    const url = new URL('https://www.googleapis.com/drive/v3/files')
    url.searchParams.set('pageSize', '200')
    url.searchParams.set('q', 'trashed=false')
    url.searchParams.set(
      'fields',
      'files(id,name,mimeType,parents,driveId,ownedByMe,shared,sharedWithMeTime,starred,viewedByMeTime,webViewLink,webContentLink)',
    )
    url.searchParams.set('supportsAllDrives', 'true')
    url.searchParams.set('includeItemsFromAllDrives', 'true')

    const response = await fetch(url.toString(), {
      headers: {
        Authorization: `Bearer ${accessToken.value}`,
      },
    })

    if (!response.ok) {
      const message = await response.text()
      if (response.status === 403 && message.includes('insufficientPermissions')) {
        accessToken.value = null
        clearToken()
        throw new Error(
          'Drive の権限が不足しています。ログインし直して権限を許可してください。',
        )
      }
      throw new Error(
        `Drive API の取得に失敗しました。(${response.status}) ${message}`,
      )
    }

    const data = (await response.json()) as { files: DriveFile[] }
    nodes.value = buildTree(data.files, drives.value)
  } catch (err) {
    error.value = err instanceof Error ? err.message : '未知のエラーが発生しました。'
  } finally {
    loading.value = false
  }
}

const fetchUserProfile = async () => {
  if (!accessToken.value) return
  try {
    const response = await fetch('https://www.googleapis.com/oauth2/v3/userinfo', {
      headers: {
        Authorization: `Bearer ${accessToken.value}`,
      },
    })
    if (!response.ok) {
      return
    }
    const data = (await response.json()) as {
      name?: string
      email?: string
      picture?: string
    }
    userProfile.value = data
  } catch {
    // ignore profile errors
  }
}

const handleLogin = async () => {
  error.value = null
  if (!CLIENT_ID) {
    error.value = 'VITE_GOOGLE_CLIENT_ID が未設定です。'
    return
  }

  await loadGoogleIdentityScript()
  if (!window.google?.accounts?.oauth2) {
    error.value = 'Google Identity の初期化に失敗しました。'
    return
  }

  tokenClient.value ??= markRaw(
    window.google.accounts.oauth2.initTokenClient({
      client_id: CLIENT_ID,
      scope: SCOPES,
      callback: (response) => {
        if (response.access_token) {
          accessToken.value = response.access_token
          saveToken(response.access_token, response.expires_in)
          fetchUserProfile()
          listDriveFiles()
        }
      },
    }),
  )

  tokenClient.value.requestAccessToken({ prompt: 'consent' })
}

const handleLogout = () => {
  if (!accessToken.value) return
  window.google?.accounts?.oauth2?.revoke(accessToken.value, () => {
    accessToken.value = null
    nodes.value = []
    selectedId.value = null
    userProfile.value = null
    showProfileMenu.value = false
    storageQuota.value = null
    updateUrlWithSelection(null)
    clearToken()
  })
}

const handleSelect = (id: string) => {
  selectedId.value = id
  updateUrlWithSelection(id)
  expandToSelection()
}

const handleResizeStart = (event: PointerEvent) => {
  const startX = event.clientX
  const startWidth = sidebarWidth.value
  const handleMove = (moveEvent: PointerEvent) => {
    const next = Math.min(Math.max(startWidth + moveEvent.clientX - startX, 220), 420)
    sidebarWidth.value = next
  }
  const handleUp = () => {
    window.removeEventListener('pointermove', handleMove)
    window.removeEventListener('pointerup', handleUp)
  }
  window.addEventListener('pointermove', handleMove)
  window.addEventListener('pointerup', handleUp)
}

const handleToggleFolder = (id: string) => {
  const next = new Set(expandedIds.value)
  if (next.has(id)) {
    next.delete(id)
  } else {
    next.add(id)
  }
  expandedIds.value = next
}

onMounted(() => {
  loadSelectionFromUrl()
  const cachedToken = loadToken()
  if (cachedToken) {
    accessToken.value = cachedToken
    fetchUserProfile()
    listDriveFiles()
  }

  loadGoogleIdentityScript().catch((err) => {
    error.value = err instanceof Error ? err.message : '初期化に失敗しました。'
  })

  window.addEventListener('popstate', handlePopState)
})

onUnmounted(() => {
  window.removeEventListener('popstate', handlePopState)
})
</script>

<template>
  <div class="app">
    <div class="layout" :style="{ '--sidebar-width': `${sidebarWidth}px` }">
      <aside class="sidebar">
        <div class="sidebar-header">
          <input
            v-model="searchQuery"
            class="search"
            type="search"
            placeholder="検索"
            :disabled="!hasAuth"
          />
        </div>
        <div class="tree">
          <div v-if="!hasAuth" class="tree-placeholder">
            ログインするとここに Drive が表示されます。
          </div>
          <div v-else-if="loading" class="tree-placeholder">読み込み中...</div>
          <div v-else-if="error" class="tree-placeholder error">{{ error }}</div>
          <DriveTree
            v-else
            :nodes="filteredNodes"
            :selected-id="selectedId"
            :expanded-ids="expandedIds"
            @select="handleSelect"
            @toggle="handleToggleFolder"
          />
        </div>
        <div class="sidebar-footer">
          <div v-if="storageQuota" class="storage">
            <div class="storage-label">保存容量</div>
            <div class="storage-value">
              {{ formatBytes(storageQuota.usage) }} /
              {{ formatBytes(storageQuota.limit) }}
            </div>
          </div>
          <button v-if="!hasAuth" class="ghost" @click="handleLogin">
            Google でログイン
          </button>
          <div v-else class="auth-status">
            <button
              class="avatar-button"
              @click="showProfileMenu = !showProfileMenu"
              :aria-expanded="showProfileMenu"
            >
              <img
                v-if="userProfile?.picture"
                class="avatar-image"
                :src="userProfile.picture"
                alt="ユーザーアイコン"
              />
              <span v-else class="auth-icon">G</span>
            </button>
            <div v-if="showProfileMenu" class="profile-menu">
              <div class="profile-name">
                {{ userProfile?.name ?? 'ログイン中' }}
              </div>
              <div v-if="userProfile?.email" class="profile-email">
                {{ userProfile.email }}
              </div>
              <button class="ghost" @click="handleLogout">ログアウト</button>
            </div>
          </div>
        </div>
      </aside>
      <div class="sidebar-resizer" @pointerdown="handleResizeStart"></div>
      <main class="content">
        <div class="content-body">
          <iframe
            v-if="selectedUrl"
            class="preview"
            :src="selectedEditUrl"
            title="Drive preview"
            referrerpolicy="no-referrer"
          ></iframe>
          <div v-else class="empty">
            左のツリーからファイルを選択してください
          </div>
        </div>
      </main>
    </div>
    <div v-if="!hasAuth" class="auth-overlay">
      <div class="auth-overlay-card">
        <div class="auth-overlay-title">Google Drive に接続</div>
        <div class="auth-overlay-text">
          左下のボタンまたはここからログインしてください。
        </div>
        <button class="ghost" @click="handleLogin">Google でログイン</button>
      </div>
    </div>
  </div>
</template>
