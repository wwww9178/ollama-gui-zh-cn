<script setup lang="ts">
import { IconX } from '@tabler/icons-vue'
import { computed, reactive, ref, watch } from 'vue'
import { useChats } from '../../composables/useChats'
import { useModels } from '../../composables/useModels'
import { usePresets } from '../../composables/usePresets'
import { toast } from '../../composables/useToasts'
import { chatSettingsOpen } from '../../composables/useUi'
import { compactSettings, resolveSettings } from '../../domain/settings'
import {
  GLOBAL_SCOPE,
  type GenerationSettings,
  type ModelOptions,
  type ThinkLevel,
} from '../../domain/types'

type Scope = 'chat' | 'model' | 'global'

const chats = useChats()
const models = useModels()
const presets = usePresets()

const scope = ref<Scope>('chat')
const model = computed(() => chats.activeModel.value)

const NUMBER_FIELDS: {
  key: keyof ModelOptions
  label: string
  step: number
  min?: number
  max?: number
  hint: string
}[] = [
  {
    key: 'temperature',
    label: 'Temperature',
    step: 0.05,
    min: 0,
    max: 2,
    hint: '数值越高越发散',
  },
  { key: 'top_p', label: 'Top P', step: 0.05, min: 0, max: 1, hint: '核采样' },
  { key: 'top_k', label: 'Top K', step: 1, min: 0, hint: '将采样限制在前 K 个 token' },
  {
    key: 'num_ctx',
    label: 'Context length (num_ctx)',
    step: 1024,
    min: 512,
    hint: '上下文 token 数',
  },
  {
    key: 'num_predict',
    label: 'Max tokens (num_predict)',
    step: 64,
    min: -1,
    hint: '-1 for unlimited',
  },
  {
    key: 'repeat_penalty',
    label: 'Repeat penalty',
    step: 0.05,
    min: 0,
    hint: '抑制重复用词',
  },
  { key: 'seed', label: 'Seed', step: 1, hint: '固定种子，保证输出可复现' },
]

/** Settings stored at the selected scope. */
function stored(which: Scope): GenerationSettings | undefined {
  if (which === 'global') return presets.globalPreset()
  if (which === 'model') return presets.modelPreset(model.value)
  return chats.activeChat.value
    ? chats.activeChat.value.settings
    : chats.draftSettings.value
}

/** What this scope inherits from the layers below it, shown as placeholders. */
const inherited = computed(() => {
  if (scope.value === 'global') return {} as GenerationSettings
  if (scope.value === 'model') return resolveSettings(presets.globalPreset())
  return resolveSettings(presets.globalPreset(), presets.modelPreset(model.value))
})

interface Form {
  systemPrompt: string
  think: '' | 'true' | 'false' | 'low' | 'medium' | 'high'
  keepAlive: string
  options: Record<string, string>
}
const form = reactive<Form>({ systemPrompt: '', think: '', keepAlive: '', options: {} })
/** Stored values as of the last load/sync; a form field that differs from it has been edited. */
let baseline: Form = toForm(undefined)

function toForm(s: GenerationSettings | undefined): Form {
  return {
    systemPrompt: s?.systemPrompt ?? '',
    think: s?.think === undefined ? '' : (String(s.think) as Form['think']),
    keepAlive: s?.keepAlive ?? '',
    options: Object.fromEntries(
      NUMBER_FIELDS.map((f) => [f.key, s?.options?.[f.key]?.toString() ?? '']),
    ),
  }
}

function load() {
  baseline = toForm(stored(scope.value))
  Object.assign(form, structuredClone(baseline))
}
watch([scope, chatSettingsOpen, () => chats.activeChatId.value, model], load, {
  immediate: true,
})

// Settings can change elsewhere while the panel is open (e.g. the composer's Think toggle).
// Pull those changes into fields the user hasn't edited, so Save doesn't write stale values back.
watch(
  () => JSON.stringify(stored(scope.value) ?? {}),
  () => {
    const next = toForm(stored(scope.value))
    for (const key of ['systemPrompt', 'think', 'keepAlive'] as const) {
      if (String(form[key]) === baseline[key])
        (form as Record<string, unknown>)[key] = next[key]
    }
    for (const f of NUMBER_FIELDS) {
      if (String(form.options[f.key] ?? '') === baseline.options[f.key])
        form.options[f.key] = next.options[f.key]
    }
    baseline = next
  },
)

const info = computed(() => models.info.get(model.value))
const thinkOptions = computed(() => {
  const opts: { value: Form['think']; label: string }[] = [
    { value: 'true', label: 'On' },
    { value: 'false', label: 'Off' },
  ]
  for (const level of info.value?.thinkLevels ?? [])
    opts.push({ value: level, label: `${level} effort` })
  return opts
})

function toSettings(): GenerationSettings {
  const options: ModelOptions = {}
  for (const f of NUMBER_FIELDS) {
    const raw = String(form.options[f.key] ?? '').trim()
    if (raw) options[f.key] = Number(raw)
  }
  let think: ThinkLevel | undefined
  if (form.think === 'true') think = true
  else if (form.think === 'false') think = false
  else if (form.think) think = form.think
  return { systemPrompt: form.systemPrompt, think, keepAlive: form.keepAlive, options }
}

async function save() {
  const settings = compactSettings(toSettings())
  if (scope.value === 'chat') await chats.setActiveSettings(settings)
  else
    await presets.savePreset(
      scope.value === 'global' ? GLOBAL_SCOPE : model.value,
      settings ?? {},
    )
  load()
  toast('设置已保存', 'success')
}

async function reset() {
  form.systemPrompt = ''
  form.think = ''
  form.keepAlive = ''
  for (const key of Object.keys(form.options)) form.options[key] = ''
  await save()
}

const scopes = computed(() => [
  { value: 'chat' as const, label: '本对话' },
  {
    value: 'model' as const,
    label: '模型',
    disabled: !model.value,
  },
  { value: 'global' as const, label: '全部对话' },
])

const scopeHint = computed(() => {
  if (scope.value === 'chat')
    return '仅对当前对话生效。留空的项会继承模型与全局的默认值。'
  if (scope.value === 'model')
    return `对使用 ${model.value} 的所有对话生效。留空的项会继承全局默认值。`
  return '对所有对话与模型生效的默认值。'
})
</script>

<template>
  <aside
    v-if="chatSettingsOpen"
    class="bg-panel border-border fixed inset-y-0 right-0 z-40 flex w-[min(380px,100vw)] flex-col border-l shadow-xl lg:static lg:shadow-none"
    aria-label="对话设置"
    data-testid="chat-settings"
  >
    <header class="border-border flex h-[46px] flex-none items-center border-b px-4">
      <h2 class="text-[13.5px] font-semibold">对话设置</h2>
      <button
        type="button"
        class="hover:bg-hover text-text-secondary ml-auto rounded-md p-1.5"
        aria-label="关闭对话设置"
        @click="chatSettingsOpen = false"
      >
        <IconX :size="16" />
      </button>
    </header>

    <div class="min-h-0 flex-1 space-y-4 overflow-y-auto p-4">
      <div
        class="bg-list border-border grid grid-cols-3 gap-0.5 rounded-lg border p-0.5"
        role="tablist"
      >
        <button
          v-for="s in scopes"
          :key="s.value"
          type="button"
          role="tab"
          :aria-selected="scope === s.value"
          :disabled="s.disabled"
          class="rounded-md px-2 py-1 text-[12px] disabled:opacity-40"
          :class="
            scope === s.value
              ? 'bg-panel text-text font-semibold shadow-sm'
              : 'text-text-secondary hover:text-text'
          "
          @click="scope = s.value"
        >
          {{ s.label }}
        </button>
      </div>
      <p class="text-text-muted text-[11.5px] leading-relaxed">
        {{ scopeHint }}
      </p>

      <label class="block">
        <span class="text-text mb-1 block text-[12px] font-medium">系统提示词</span>
        <textarea
          v-model="form.systemPrompt"
          rows="6"
          class="border-border bg-list text-text placeholder:text-text-muted focus:border-accent block w-full resize-y rounded-lg border p-2.5 text-[12.5px] leading-relaxed outline-none"
          :placeholder="
            inherited.systemPrompt
              ? `继承自：${inherited.systemPrompt}`
              : '你是一个乐于助人的助手…'
          "
          data-testid="system-prompt-input"
        />
      </label>

      <label class="block">
        <span class="text-text mb-1 block text-[12px] font-medium">Thinking</span>
        <select
          v-model="form.think"
          class="border-border bg-list text-text focus:border-accent block w-full rounded-lg border px-2.5 py-1.5 text-[12.5px] outline-none"
        >
          <option value="">
            Inherit{{
              inherited.think !== undefined
                ? ` (${inherited.think === true ? 'on' : inherited.think === false ? 'off' : inherited.think})`
                : ' (model default)'
            }}
          </option>
          <option v-for="o in thinkOptions" :key="o.value" :value="o.value">
            {{ o.label }}
          </option>
        </select>
        <span class="text-text-muted mt-1 block text-[11px]">
          仅发送给支持 Thinking 的模型。
        </span>
      </label>

      <div class="grid grid-cols-2 gap-3">
        <label v-for="f in NUMBER_FIELDS" :key="f.key" class="block" :title="f.hint">
          <span class="text-text mb-1 block truncate text-[12px] font-medium">
            {{ f.label }}
          </span>
          <input
            v-model="form.options[f.key]"
            type="number"
            :step="f.step"
            :min="f.min"
            :max="f.max"
            class="border-border bg-list text-text placeholder:text-text-muted focus:border-accent block w-full rounded-lg border px-2.5 py-1.5 text-[12.5px] outline-none"
            :placeholder="
              inherited.options?.[f.key]?.toString() ??
              (f.key === 'num_ctx' && info?.contextLength
                ? `最大 ${info.contextLength}`
                : '默认')
            "
          />
        </label>
        <label class="block">
          <span class="text-text mb-1 block text-[12px] font-medium">Keep alive</span>
          <input
            v-model="form.keepAlive"
            class="border-border bg-list text-text placeholder:text-text-muted focus:border-accent block w-full rounded-lg border px-2.5 py-1.5 text-[12.5px] outline-none"
            :placeholder="inherited.keepAlive ?? '5m'"
          />
        </label>
      </div>
    </div>

    <footer class="border-border flex flex-none items-center gap-2 border-t px-4 py-3">
      <button
        type="button"
        class="hover:bg-hover text-text-secondary rounded-md px-3 py-1.5 text-[12.5px]"
        @click="reset"
      >
        清除
      </button>
      <button
        type="button"
        class="bg-accent ml-auto rounded-md px-4 py-1.5 text-[12.5px] font-semibold text-white hover:opacity-90"
        data-testid="save-chat-settings"
        @click="save"
      >
        保存
      </button>
    </footer>
  </aside>
</template>
