<script setup lang="ts">
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import { $fetch } from 'ofetch'
import { useChats } from '../composables/useChats'
import { useCsrf } from '../composables/useCsrf'
import { useUserSession } from '../composables/useUserSession'
import Navbar from '../components/Navbar.vue'

const { fetchChats } = useChats()
const { csrf, headerName } = useCsrf()
const { user } = useUserSession()
const input = ref('')
const loading = ref(false)
const adviceLoading = ref(false)
const adviceResponse = ref<string | null>(null)
const adviceError = ref<string | null>(null)
const mode = ref<'chat' | 'advice'>('chat')
const router = useRouter()
const AGLIMATE_API_BASE = import.meta.env.VITE_AGLIMATE_API_BASE || 'https://nexusbert-aglimate.hf.space'

const greeting = computed(() => {
  const hour = new Date().getHours()
  let timeGreeting = 'Good evening'
  if (hour < 12) timeGreeting = 'Good morning'
  else if (hour < 18) timeGreeting = 'Good afternoon'

  const name = user.value?.name?.split(' ')[0] || user.value?.username

  return name ? `${timeGreeting}, ${name}` : `${timeGreeting}`
})

async function createChat(prompt: string) {
  input.value = prompt
  loading.value = true
  const chat = await $fetch('/api/chats', {
    method: 'POST',
    headers: { [headerName]: csrf() },
    body: { input: prompt }
  })

  await fetchChats()
  router.push(`/chat/${chat?.id}`)
}

async function requestAdvice(prompt: string) {
  adviceError.value = null
  adviceResponse.value = null
  adviceLoading.value = true

  try {
    const formData = new FormData()
    formData.append('query', prompt)
    if (user.value?.id) {
      formData.append('session_id', user.value.id)
    }

    const response = await fetch(`${AGLIMATE_API_BASE}/advise`, {
      method: 'POST',
      body: formData
    })

    if (!response.ok) {
      const body = await response.text()
      throw new Error(body || `${response.status} ${response.statusText}`)
    }

    const result = await response.json()
    adviceResponse.value = result.answer || JSON.stringify(result)
  } catch (error) {
    adviceError.value = error instanceof Error ? error.message : String(error)
  } finally {
    adviceLoading.value = false
  }
}

function onSubmit() {
  if (!input.value.trim()) {
    return
  }

  if (mode.value === 'chat') {
    createChat(input.value)
  } else {
    requestAdvice(input.value)
  }
}

const quickChats = [
  {
    label: 'Why use Nuxt UI?',
    icon: 'i-logos-nuxt-icon'
  },
  {
    label: 'Help me create a Vue composable',
    icon: 'i-logos-vue'
  },
  {
    label: 'Tell me more about UnJS',
    icon: 'i-logos-unjs'
  },
  {
    label: 'Why should I consider VueUse?',
    icon: 'i-logos-vueuse'
  },
  {
    label: 'Tailwind CSS best practices',
    icon: 'i-logos-tailwindcss-icon'
  },
  {
    label: 'What is the weather in Bordeaux?',
    icon: 'i-lucide-sun'
  },
  {
    label: 'Show me a chart of sales data',
    icon: 'i-lucide-line-chart'
  }
]
</script>

<template>
  <UDashboardPanel
    id="home"
    class="min-h-0"
    :ui="{ body: 'p-0 sm:p-0' }"
  >
    <template #header>
      <Navbar />
    </template>

    <template #body>
      <UContainer class="flex-1 flex flex-col justify-center gap-4 sm:gap-6 py-8">
        <h1 class="text-3xl sm:text-4xl text-highlighted font-bold">
          {{ greeting }}
        </h1>

        <div class="flex flex-wrap gap-2 items-center mb-4">
          <UButton
            size="sm"
            variant="outline"
            :color="mode === 'chat' ? 'primary' : 'neutral'"
            label="Chat"
            @click="mode = 'chat'"
          />
          <UButton
            size="sm"
            variant="outline"
            :color="mode === 'advice' ? 'primary' : 'neutral'"
            label="Advice"
            @click="mode = 'advice'"
          />
          <span class="text-sm text-muted ml-2">
            {{ mode === 'chat' ? 'Text chat mode' : 'Aglimate advice mode' }}
          </span>
        </div>

        <UChatPrompt
          v-model="input"
          :status="(mode === 'chat' ? (loading ? 'streaming' : 'ready') : (adviceLoading ? 'streaming' : 'ready'))"
          class="[view-transition-name:chat-prompt]"
          color="neutral"
          variant="subtle"
          :ui="{ base: 'px-1.5' }"
          @submit="onSubmit"
        >
          <template #footer>
            <ModelSelect />

            <UChatPromptSubmit
              color="neutral"
              size="sm"
            />
          </template>
        </UChatPrompt>

        <div v-if="mode === 'advice'" class="rounded-2xl border border-surface-300 bg-surface-100 p-4 text-sm text-slate-700 dark:bg-slate-900 dark:text-slate-100">
          <p class="font-semibold">Advice mode</p>
          <p class="mt-2">Send your question directly to the Aglimate advisory endpoint. This is best for crop, weather, and farm condition guidance.</p>
          <p class="mt-2 text-xs text-muted">Uses: <code>{{ AGLIMATE_API_BASE }}/advise</code></p>
        </div>

        <div v-if="adviceResponse" class="rounded-2xl border border-emerald-300 bg-emerald-50 p-4 text-sm text-slate-900 dark:bg-emerald-950 dark:text-emerald-100">
          <p class="font-semibold">Advice result</p>
          <p class="mt-2 whitespace-pre-line">{{ adviceResponse }}</p>
        </div>

        <div v-if="adviceError" class="rounded-2xl border border-rose-300 bg-rose-50 p-4 text-sm text-rose-900 dark:bg-rose-950 dark:text-rose-100">
          <p class="font-semibold">Advice request failed</p>
          <p class="mt-2">{{ adviceError }}</p>
        </div>

        <div class="flex flex-wrap gap-2">
          <UButton
            v-for="quickChat in quickChats"
            :key="quickChat.label"
            :icon="quickChat.icon"
            :label="quickChat.label"
            size="sm"
            color="neutral"
            variant="outline"
            class="rounded-full"
            @click="createChat(quickChat.label)"
          />
        </div>
      </UContainer>
    </template>
  </UDashboardPanel>
</template>
