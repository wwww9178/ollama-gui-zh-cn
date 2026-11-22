<script setup lang="ts">
import { IconDownload, IconTrash, IconUpload } from '@tabler/icons-vue'
import { computed, onMounted, ref, watch } from 'vue'
import { useChats } from '../../composables/useChats'
import { confirmAction } from '../../composables/useConfirm'
import { useModels } from '../../composables/useModels'
import {
  autoTitle,
  dateLocale,
  enableMarkdown,
  gravatarEmail,
  locale,
  ollamaUrl,
  showSystem,
  theme,
  titleModel,
  type Theme,
} from '../../composables/useSettings'
import { errorMessage, toast } from '../../composables/useToasts'
import { downloadFile, exportAll, importData } from '../../composables/useTransfer'
import { closeModal, modal } from '../../composables/useUi'
import { relativeTime, resolveLocale } from '../../domain/format'
import Modal from '../ui/Modal.vue'
import Toggle from '../ui/Toggle.vue'
import CorsHelp from './CorsHelp.vue'

const chats = useChats()
const models = useModels()
const open = computed(() => modal.value === 'settings')

// Connection
const urlDraft = ref(ollamaUrl.value)
watch(open, (value) => {
  if (value) urlDraft.value = ollamaUrl.value
})
async function saveUrl() {
  ollamaUrl.value = urlDraft.value.trim()
  models.info.clear()
  await models.refresh()
  if (models.connection.value === 'ok') toast(`已连接 ${models.host()}`, 'success')
}

// Appearance
const themes: { value: Theme; label: string }[] = [
  { value: 'system', label: '系统' },
  { value: 'light', label: '浅色' },
  { value: 'dark', label: '深色' },
]
const LOCALES = [
  'en-US',
  'en-GB',
  'de-DE',
  'fr-FR',
  'es-ES',
  'it-IT',
  'pt-BR',
  'nl-NL',
  'sv-SE',
  'nb-NO',
  'da-DK',
  'fi-FI',
  'pl-PL',
  'cs-CZ',
  'ru-RU',
  'uk-UA',
  'tr-TR',
  'ar',
  'he-IL',
  'hi-IN',
  'ja-JP',
  'ko-KR',
  'zh-CN',
  'zh-TW',
]
const localeChoice = ref(
  LOCALES.includes(dateLocale.value) || !dateLocale.value ? dateLocale.value : 'custom',
)
const customLocale = ref(localeChoice.value === 'custom' ? dateLocale.value : '')
// Names in English to match the UI language, e.g. "German (Germany)"
const regionNames = new Intl.DisplayNames(['en'], { type: 'language' })
const displayName = (tag: string) => {
  try {
    return regionNames.of(tag) ?? tag
  } catch {
    return tag
  }
}
const systemLocale = resolveLocale('')
watch(localeChoice, (value) => {
  dateLocale.value = value === 'custom' ? customLocale.value.trim() : value
})
watch(customLocale, (value) => {
  if (localeChoice.value === 'custom') dateLocale.value = value.trim()
})
/** Sample of every format the setting affects: date and time, relative time, numbers. */
const preview = computed(() => {
  const sample = new Date()
  sample.setHours(14, 30, 0, 0)
  const dateTime = new Intl.DateTimeFormat(locale.value, {
    dateStyle: 'medium',
    timeStyle: 'short',
  }).format(sample)
  const relative = relativeTime(new Date(Date.now() - 2 * 86400_000), locale.value)
  const number = new Intl.NumberFormat(locale.value).format(1234567.89)
  return `${dateTime} · ${relative} · ${number}`
})
const customInvalid = computed(
  () =>
    localeChoice.value === 'custom' &&
    !!customLocale.value.trim() &&
    resolveLocale(customLocale.value, '') === '',
)

// Data
const persisted = ref<boolean | null>(null)
onMounted(async () => {
  persisted.value = (await navigator.storage?.persisted?.()) ?? null
})
async function requestPersist() {
  persisted.value = (await navigator.storage?.persist?.()) ?? false
  toast(
    persisted.value
      ? '已开启持久存储'
      : '浏览器拒绝了持久存储',
    persisted.value ? 'success' : 'info',
  )
}

async function doExport() {
  const data = await exportAll()
  downloadFile(
    `ollama-gui-${new Date().toISOString().slice(0, 10)}.json`,
    JSON.stringify(data, null, 2),
    'application/json',
  )
}

const importInput = ref<HTMLInputElement>()
async function doImport(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  input.value = ''
  if (!file) return
  try {
    const count = await importData(JSON.parse(await file.text()))
    toast(`Imported ${count} chat${count === 1 ? '' : 's'}`, 'success')
  } catch (error) {
    toast(`Import failed: ${errorMessage(error)}`, 'error')
  }
}

async function wipe() {
  const ok = await confirmAction({
    title: '删除全部对话？',
    message:
      'Every chat and message stored in this browser will be deleted. Export first if you want a backup.',
    confirmLabel: '删除全部内容',
    danger: true,
  })
  if (ok) {
    await chats.deleteAllChats()
    toast('已删除全部对话', 'success')
  }
}

const origin = location.origin
</script>

<template>
  <Modal :open="open" title="设置" width="560px" @close="closeModal('settings')">
    <div class="divide-border divide-y">
      <section class="space-y-2 p-4">
        <h3 class="text-text-muted text-[11px] font-semibold tracking-wide uppercase">
          连接
        </h3>
        <label for="ollama-url" class="text-text block text-[12.5px]">Ollama 地址</label>
        <form class="flex gap-2" @submit.prevent="saveUrl">
          <input
            id="ollama-url"
            v-model="urlDraft"
            :placeholder="`Default: ${models.host('')}`"
            class="border-border bg-list text-text placeholder:text-text-muted focus:border-accent min-w-0 flex-1 rounded-lg border px-3 py-1.5 text-[12.5px] outline-none"
            data-testid="input-base-url"
          />
          <button
            type="submit"
            class="border-border hover:bg-hover rounded-lg border px-3 py-1.5 text-[12.5px]"
          >
            Save &amp; test
          </button>
        </form>
        <p
          class="text-[12px]"
          :class="models.connection.value === 'error' ? 'text-red' : 'text-text-muted'"
        >
          <template v-if="models.connection.value === 'ok'">
            已连接 {{ models.host() }} · {{ models.models.value.length }} 个模型
          </template>
          <template v-else-if="models.connection.value === 'error'">
            {{ models.connectionError.value }}
          </template>
          <template v-else>连接中…</template>
        </p>
        <CorsHelp v-if="models.connection.value === 'error'" />
      </section>

      <section class="space-y-3 p-4">
        <h3 class="text-text-muted text-[11px] font-semibold tracking-wide uppercase">
          外观
        </h3>
        <div class="flex items-center justify-between gap-4">
          <span class="text-text text-[12.5px]">主题</span>
          <div
            class="bg-list border-border flex gap-0.5 rounded-lg border p-0.5"
            role="radiogroup"
            aria-label="主题"
          >
            <button
              v-for="t in themes"
              :key="t.value"
              type="button"
              role="radio"
              :aria-checked="theme === t.value"
              class="rounded-md px-3 py-1 text-[12px]"
              :class="
                theme === t.value
                  ? 'bg-panel text-text font-semibold shadow-sm'
                  : 'text-text-secondary'
              "
              :data-testid="`theme-${t.value}`"
              @click="theme = t.value"
            >
              {{ t.label }}
            </button>
          </div>
        </div>
        <div>
          <div class="flex items-center justify-between gap-4">
            <label for="locale" class="text-text text-[12.5px]">
              区域格式
              <span class="text-text-muted block text-[11.5px]">
                日期、时间与数字的书写格式
              </span>
            </label>
            <select
              id="locale"
              v-model="localeChoice"
              class="border-border bg-list text-text focus:border-accent max-w-[240px] rounded-lg border px-2 py-1 text-[12.5px] outline-none"
              data-testid="locale-select"
            >
              <option value="">跟随系统（{{ displayName(systemLocale) }}）</option>
              <option v-for="tag in LOCALES" :key="tag" :value="tag">
                {{ displayName(tag) }}
              </option>
              <option value="custom">其他…</option>
            </select>
          </div>
          <div
            v-if="localeChoice === 'custom'"
            class="mt-2 flex items-center justify-end gap-2"
          >
            <span v-if="customInvalid" class="text-red text-[11.5px]">
              无法识别，使用系统格式
            </span>
            <input
              v-model="customLocale"
              placeholder="语言标签，例如 en-IE"
              class="border-border bg-list text-text focus:border-accent w-[180px] rounded-lg border px-2 py-1 text-[12.5px] outline-none"
              aria-label="语言标签"
            />
          </div>
          <p
            class="bg-list text-text-secondary mt-2 rounded-md px-2.5 py-1.5 text-[12px] tabular-nums"
            data-testid="locale-preview"
          >
            {{ preview }}
          </p>
        </div>
        <Toggle
          v-model="enableMarkdown"
          label="渲染 Markdown"
          description="回复中的排版、代码高亮与公式"
        />
        <Toggle
          v-model="showSystem"
          label="显示系统提示词"
          description="在对话上方显示系统提示词"
        />
      </section>

      <section class="space-y-2 p-4">
        <h3 class="text-text-muted text-[11px] font-semibold tracking-wide uppercase">
          对话
        </h3>
        <Toggle
          v-model="autoTitle"
          label="自动生成对话标题"
          description="首轮回复后让模型生成简短标题"
        />
        <label for="title-model" class="text-text block pt-1 text-[12.5px]">
          标题生成模型
          <span class="text-text-muted">（留空则跟随对话模型）</span>
        </label>
        <select
          id="title-model"
          v-model="titleModel"
          class="border-border bg-list text-text focus:border-accent w-full rounded-lg border px-3 py-1.5 text-[12.5px] outline-none"
          data-testid="select-title-model"
        >
          <option value="">跟随对话模型</option>
          <option v-for="m in models.models.value" :key="m.name" :value="m.name">
            {{ m.name }}
          </option>
        </select>
        <label for="gravatar" class="text-text block pt-1 text-[12.5px]">
          Gravatar 邮箱
          <span class="text-text-muted">（可选头像）</span>
        </label>
        <input
          id="gravatar"
          v-model="gravatarEmail"
          type="email"
          class="border-border bg-list text-text focus:border-accent w-full rounded-lg border px-3 py-1.5 text-[12.5px] outline-none"
          data-testid="input-gravatar"
        />
      </section>

      <section class="space-y-3 p-4">
        <h3 class="text-text-muted text-[11px] font-semibold tracking-wide uppercase">
          数据
        </h3>
        <p class="text-text-secondary text-[12px] leading-relaxed">
          对话仅保存在当前浏览器中，地址为
          <code class="bg-hover rounded px-1">{{ origin }}</code>
          。换一个地址、端口、浏览器或使用隐私窗口，都会从空历史开始。可以用导出功能在它们之间迁移对话。
        </p>
        <div
          v-if="persisted !== null"
          class="flex items-center justify-between gap-4 text-[12.5px]"
        >
          <span>
            {{
              persisted
                ? '已开启持久存储'
                : '磁盘空间不足时浏览器可能清除存储'
            }}
          </span>
          <button
            v-if="!persisted"
            type="button"
            class="border-border hover:bg-hover rounded-lg border px-3 py-1 text-[12px]"
            @click="requestPersist"
          >
            保留数据
          </button>
        </div>
        <div class="flex flex-wrap gap-2">
          <button
            type="button"
            class="border-border hover:bg-hover flex items-center gap-1.5 rounded-lg border px-3 py-1.5 text-[12.5px]"
            data-testid="export-chats"
            @click="doExport"
          >
            <IconDownload :size="15" />
            导出全部
          </button>
          <button
            type="button"
            class="border-border hover:bg-hover flex items-center gap-1.5 rounded-lg border px-3 py-1.5 text-[12.5px]"
            data-testid="import-chats"
            @click="importInput?.click()"
          >
            <IconUpload :size="15" />
            导入
          </button>
          <input
            ref="importInput"
            type="file"
            accept="application/json,.json"
            class="hidden"
            data-testid="import-input"
            @change="doImport"
          />
          <button
            type="button"
            class="text-red hover:bg-red/10 ml-auto flex items-center gap-1.5 rounded-lg px-3 py-1.5 text-[12.5px]"
            data-testid="delete-all-chats"
            @click="wipe"
          >
            <IconTrash :size="15" />
            删除全部对话
          </button>
        </div>
      </section>
    </div>
  </Modal>
</template>
