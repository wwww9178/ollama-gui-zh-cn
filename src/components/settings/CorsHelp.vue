<script setup lang="ts">
import { IconCheck, IconCopy } from '@tabler/icons-vue'
import { computed, ref } from 'vue'
import { copyText } from '../../composables/useClipboard'
import { useModels } from '../../composables/useModels'

const models = useModels()
const origin = location.origin
/** Cross-origin requests need Ollama to allow this page's origin. */
const crossOrigin = computed(() => {
  try {
    return new URL(models.host()).origin !== origin
  } catch {
    return true
  }
})
const command = `OLLAMA_ORIGINS=${origin} ollama serve`
const copied = ref(false)
async function copy() {
  if (!(await copyText(command))) return
  copied.value = true
  setTimeout(() => (copied.value = false), 1500)
}
</script>

<template>
  <div class="text-text-secondary space-y-1.5 text-[12px] leading-relaxed">
    <p>
      {{
        crossOrigin
          ? '请确认 Ollama 正在运行，并允许来自本页面的请求：'
          : '请确认 Ollama 正在运行：'
      }}
    </p>
    <div
      v-if="crossOrigin"
      class="bg-list border-border flex items-center gap-2 rounded-lg border px-2.5 py-1.5 font-mono text-[11.5px]"
    >
      <span class="min-w-0 flex-1 overflow-x-auto whitespace-nowrap">{{ command }}</span>
      <button
        type="button"
        class="text-text-muted hover:text-text flex-none"
        :aria-label="copied ? '已复制' : '复制命令'"
        @click="copy"
      >
        <IconCheck v-if="copied" :size="14" />
        <IconCopy v-else :size="14" />
      </button>
    </div>
    <p v-if="crossOrigin">
      {{
        '浏览器还可能询问是否允许本站访问你的本地网络设备。请选择「允许」，或在地址栏旁的站点设置中重新开启。'
      }}
    </p>
    <p v-else>
      {{ 'Start it with "ollama serve", or check the proxy in front of this page.' }}
    </p>
  </div>
</template>
