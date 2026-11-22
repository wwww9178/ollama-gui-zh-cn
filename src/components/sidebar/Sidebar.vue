<script setup lang="ts">
import {
  IconArchive,
  IconBox,
  IconChevronRight,
  IconDots,
  IconEdit,
  IconFileDownload,
  IconPencil,
  IconPinned,
  IconPinnedOff,
  IconSearch,
  IconSettings,
  IconTrash,
  IconX,
} from '@tabler/icons-vue'
import { computed, nextTick, ref } from 'vue'
import { useChats } from '../../composables/useChats'
import { confirmAction } from '../../composables/useConfirm'
import { useGeneration } from '../../composables/useGeneration'
import { useModels } from '../../composables/useModels'
import { locale } from '../../composables/useSettings'
import { chatToMarkdown, downloadFile, safeFileName } from '../../composables/useTransfer'
import { openModal, sidebarOpen } from '../../composables/useUi'
import { dateGroup, dateGroupLabel } from '../../domain/format'
import type { Chat } from '../../domain/types'
import Menu from '../ui/Menu.vue'
import OllamaAvatar from '../ui/OllamaAvatar.vue'
import MenuItem from '../ui/MenuItem.vue'

const chats = useChats()
const generation = useGeneration()
const models = useModels()

const filter = ref('')
const showArchived = ref(false)
const renaming = ref<string | null>(null)
const renameValue = ref('')
const renameInput = ref<HTMLInputElement[]>()

const byRecent = (a: Chat, b: Chat) => b.updatedAt.getTime() - a.updatedAt.getTime()

const matching = computed(() => {
  const q = filter.value.trim().toLowerCase()
  return chats.chats.value.filter((c) => !q || c.title.toLowerCase().includes(q))
})

const groups = computed(() => {
  const live = matching.value.filter((c) => !c.archived).sort(byRecent)
  const out: { key: string; label: string; chats: Chat[] }[] = []
  const pinned = live.filter((c) => c.pinned)
  if (pinned.length) out.push({ key: 'pinned', label: '已固定', chats: pinned })
  for (const chat of live.filter((c) => !c.pinned)) {
    const key = dateGroup(chat.updatedAt)
    let group = out.find((g) => g.key === key)
    if (!group)
      out.push((group = { key, label: dateGroupLabel(key, locale.value), chats: [] }))
    group.chats.push(chat)
  }
  return out
})

const archived = computed(() => matching.value.filter((c) => c.archived).sort(byRecent))

function open(chatId: string) {
  sidebarOpen.value = false
  void chats.openChat(chatId)
}

function startNew() {
  sidebarOpen.value = false
  chats.newChat()
}

async function startRename(chat: Chat) {
  renaming.value = chat.id
  renameValue.value = chat.title
  await nextTick()
  renameInput.value?.[0]?.select()
}

async function commitRename(chat: Chat) {
  if (renaming.value !== chat.id) return
  renaming.value = null
  if (renameValue.value.trim() && renameValue.value.trim() !== chat.title)
    await chats.renameChat(chat.id, renameValue.value)
}

async function exportMarkdown(chat: Chat) {
  downloadFile(
    `${safeFileName(chat.title)}.md`,
    await chatToMarkdown(chat.id),
    'text/markdown',
  )
}

async function remove(chat: Chat) {
  const ok = await confirmAction({
    title: '删除该对话？',
    message: `"${chat.title}" and all of its messages will be deleted.`,
    confirmLabel: '删除',
    danger: true,
  })
  if (ok) await chats.deleteChat(chat.id)
}
</script>

<template>
  <div
    v-if="sidebarOpen"
    class="fixed inset-0 z-30 bg-black/40 md:hidden"
    @click="sidebarOpen = false"
  />
  <aside
    class="bg-chrome border-border fixed inset-y-0 left-0 z-40 flex w-[264px] flex-none flex-col border-r transition-transform md:static md:translate-x-0"
    :class="sidebarOpen ? 'translate-x-0' : '-translate-x-full'"
    aria-label="对话"
  >
    <div class="flex h-[46px] flex-none items-center gap-1 px-3">
      <OllamaAvatar :size="24" />
      <span class="text-text ml-1 text-[13px] font-semibold">Ollama GUI</span>
      <button
        type="button"
        class="hover:bg-hover text-text-secondary ml-auto rounded-md p-1.5"
        title="搜索 (⌘K)"
        aria-label="搜索"
        @click="openModal('palette')"
      >
        <IconSearch :size="17" />
      </button>
      <button
        type="button"
        class="hover:bg-hover text-text-secondary rounded-md p-1.5"
        title="新建对话 (⌘⇧O)"
        aria-label="新建对话"
        data-testid="new-chat-btn"
        @click="startNew"
      >
        <IconEdit :size="17" />
      </button>
      <button
        type="button"
        class="hover:bg-hover text-text-secondary rounded-md p-1.5 md:hidden"
        aria-label="关闭侧栏"
        @click="sidebarOpen = false"
      >
        <IconX :size="17" />
      </button>
    </div>

    <div class="px-3 pb-2">
      <input
        v-model="filter"
        type="search"
        placeholder="筛选对话"
        class="border-border bg-panel text-text placeholder:text-text-muted focus:border-accent w-full rounded-md border px-2.5 py-1.5 text-[12.5px] outline-none"
        data-testid="search-chats"
      />
    </div>

    <nav class="min-h-0 flex-1 overflow-y-auto px-2 pb-2">
      <section v-for="group in groups" :key="group.key" class="mb-3">
        <h3 class="text-text-muted px-2 pt-1 pb-1 text-[11px] font-semibold">
          {{ group.label }}
        </h3>
        <template v-for="chat in group.chats" :key="chat.id">
          <div
            class="group relative flex items-center rounded-md"
            :class="chats.activeChatId.value === chat.id ? 'bg-sel' : 'hover:bg-hover'"
            data-testid="chat-item"
          >
            <input
              v-if="renaming === chat.id"
              ref="renameInput"
              v-model="renameValue"
              class="border-accent bg-panel text-text m-0.5 w-full rounded border px-2 py-1 text-[12.5px] outline-none"
              aria-label="对话标题"
              @keydown.enter="commitRename(chat)"
              @keydown.esc="renaming = null"
              @blur="commitRename(chat)"
            />
            <button
              v-else
              type="button"
              class="text-text min-w-0 flex-1 truncate px-2 py-1.5 text-left text-[12.5px]"
              :aria-current="chats.activeChatId.value === chat.id ? 'page' : undefined"
              @click="open(chat.id)"
              @dblclick="startRename(chat)"
            >
              <span
                v-if="generation.isGenerating(chat.id)"
                class="bg-accent mr-1.5 inline-block h-1.5 w-1.5 animate-pulse rounded-full align-middle"
                title="生成中"
              />
              {{ chat.title }}
            </button>
            <Menu v-if="renaming !== chat.id" class="flex-none">
              <template #trigger="{ toggle, open: menuOpen }">
                <button
                  type="button"
                  class="text-text-muted hover:text-text mr-1 rounded p-1 opacity-0 group-focus-within:opacity-100 group-hover:opacity-100"
                  :class="{ 'opacity-100': menuOpen }"
                  :aria-label="`Actions for ${chat.title}`"
                  @click="toggle"
                >
                  <IconDots :size="15" />
                </button>
              </template>
              <MenuItem :icon="IconPencil" @click="startRename(chat)">重命名</MenuItem>
              <MenuItem
                :icon="chat.pinned ? IconPinnedOff : IconPinned"
                @click="chats.togglePin(chat.id)"
              >
                {{ chat.pinned ? '取消固定' : '固定' }}
              </MenuItem>
              <MenuItem :icon="IconArchive" @click="chats.toggleArchive(chat.id)">
                归档
              </MenuItem>
              <MenuItem :icon="IconFileDownload" @click="exportMarkdown(chat)">
                导出为 Markdown
              </MenuItem>
              <div class="border-border my-1 border-t" />
              <MenuItem
                :icon="IconTrash"
                danger
                data-testid="delete-chat-btn"
                @click="remove(chat)"
              >
                删除
              </MenuItem>
            </Menu>
          </div>
        </template>
      </section>

      <p v-if="!groups.length" class="text-text-muted px-2 py-3 text-[12px]">
        {{ filter ? '没有匹配的对话。' : '还没有对话。' }}
      </p>

      <section v-if="archived.length">
        <button
          type="button"
          class="text-text-muted hover:text-text-secondary flex w-full items-center gap-1 px-2 py-1 text-[11px] font-semibold"
          :aria-expanded="showArchived"
          @click="showArchived = !showArchived"
        >
          <IconChevronRight
            :size="12"
            class="transition-transform"
            :class="{ 'rotate-90': showArchived }"
          />
          已归档 ({{ archived.length }})
        </button>
        <template v-if="showArchived">
          <div
            v-for="chat in archived"
            :key="chat.id"
            class="group flex items-center rounded-md"
            :class="chats.activeChatId.value === chat.id ? 'bg-sel' : 'hover:bg-hover'"
          >
            <button
              type="button"
              class="text-text-secondary min-w-0 flex-1 truncate px-2 py-1.5 text-left text-[12.5px]"
              @click="open(chat.id)"
            >
              {{ chat.title }}
            </button>
            <button
              type="button"
              class="text-text-muted hover:text-text mr-1 rounded p-1 text-[11px] opacity-0 group-hover:opacity-100"
              @click="chats.toggleArchive(chat.id)"
            >
              恢复
            </button>
          </div>
        </template>
      </section>
    </nav>

    <div class="border-border flex-none space-y-0.5 border-t p-2">
      <button
        type="button"
        class="hover:bg-hover text-text flex w-full items-center gap-2 rounded-md px-2 py-1.5 text-[12.5px]"
        data-testid="models-btn"
        @click="openModal('models')"
      >
        <IconBox :size="16" class="text-text-muted" />
        模型
        <span class="text-text-muted ml-auto text-[11px]">
          {{ models.models.value.length }}
        </span>
      </button>
      <button
        type="button"
        class="hover:bg-hover text-text flex w-full items-center gap-2 rounded-md px-2 py-1.5 text-[12.5px]"
        data-testid="settings-btn"
        @click="openModal('settings')"
      >
        <IconSettings :size="16" class="text-text-muted" />
        设置
        <span
          class="ml-auto h-2 w-2 rounded-full"
          :class="{
            'bg-green': models.connection.value === 'ok',
            'bg-red': models.connection.value === 'error',
            'bg-text-muted': models.connection.value === 'unknown',
          }"
          :title="
            models.connection.value === 'ok'
              ? `Connected to ${models.host()}`
              : (models.connectionError.value ?? 'Connecting…')
          "
        />
      </button>
    </div>
  </aside>
</template>
