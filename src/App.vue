<template>
  <div class="flex h-screen">
    <!-- 左侧区块 -->
    <div
      :class="[
        'bg-white overflow-auto h-screen transition-all duration-300 ease-in-out',
        isSidebarOpen ? 'w-[240px]' : 'w-0',
        'absolute z-10 md:z-auto md:relative md:w-[300px]'
      ]"
      id="leftSidebar"
    >
      <!-- 头部 -->
      <div class="flex items-center justify-between px-4 py-2">
        <h1 class="text-3xl font-bold text-pink-500">ChatFUN</h1>
        <!-- 开始新对话 -->
        <div class="hidden md:block">
          <el-button circle @click="startNewChat">
            <img src="/new.svg" width="20" height="20" alt="Start new chat" />
          </el-button>
        </div>
        <div class="md:hidden">
          <el-button circle @click="toggleSidebar">
            <el-icon>
              <Close />
            </el-icon>
          </el-button>
        </div>
      </div>

      <!-- 搜索框 -->
      <div class="mt-4 px-4 mb-4">
        <el-input placeholder="搜索..." v-model="searchValue">
          <template #append>
            <el-button :icon="Search" />
          </template>
        </el-input>
      </div>

      <!-- 聊天列表 -->
      <div class="overflow-y-auto">
        <div
          v-for="chat in filteredChatList"
          :key="chat.id"
          :class="[
            'px-4 py-2 cursor-pointer border-r-4 transition-colors',
            chat.id === currentChatId
              ? 'bg-pink-50 border-pink-500'
              : 'border-transparent hover:bg-gray-100'
          ]"
          @click="selectChat(chat.id)"
        >
          <div class="font-bold text-lg truncate">{{ chat.title }}</div>
          <div class="text-sm text-gray-500 truncate">{{ chat.lastMessage }}</div>
        </div>
      </div>
    </div>

    <!-- 右侧区块 -->
    <div class="flex flex-1 flex-col bg-gray-100 h-screen">
      <!-- 聊天标题 -->
      <div class="p-4 border-b flex items-center justify-between bg-white">
        <div :class="[isSidebarOpen ? 'hidden' : 'block', 'md:hidden']">
          <el-button circle @click="toggleSidebar">
            <el-icon>
              <More />
            </el-icon>
          </el-button>
        </div>
        <h1 class="text-xl font-bold text-gray-500 flex-grow text-center truncate px-4">
          {{ currentChatTitle }}
        </h1>
        <div class="md:hidden">
          <el-button circle @click="startNewChat">
            <el-icon>
              <Plus />
            </el-icon>
          </el-button>
        </div>
      </div>

      <!-- 消息列表 -->
      <div ref="messageListRef" class="flex-1 overflow-auto p-4">
        <div
          v-for="(message, index) in messages"
          :key="message.id || index"
          :class="['flex', { 'flex-row-reverse': message.user === 'user' }]"
        >
          <div class="min-w-[40px] min-h-[40px]">
            <img
              :src="message.user === 'ai' ? '/robot_ai.png' : '/me.png'"
              class="rounded-full"
              width="40"
              height="40"
              alt="avatar"
            />
          </div>
          <div
            :class="{
              'flex flex-col mb-5': true,
              'mr-14 ml-3': message.user === 'ai',
              'mr-3 ml-14': message.user === 'user'
            }"
          >
            <div
              :class="{
                'px-4 py-2 rounded-lg shadow-lg md:max-w-fit': true,
                'bg-white text-black': message.user === 'ai',
                'bg-green-500 text-white': message.user === 'user'
              }"
            >
              <div v-if="message.user === 'ai'">
                <div v-for="(part, partIndex) in parseMessage(message.content)" :key="partIndex">
                  <highlightjs v-if="part.isCode" :code="part.content" :language="part.language" />
                  <span v-else v-html="part.content"></span>
                </div>
              </div>
              <span v-else>{{ message.content }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 底部输入框 -->
      <div class="border-t p-4 bg-white">
        <el-input
          size="large"
          placeholder="请输入..."
          v-model="userInputValue"
          @keyup.enter.exact="sendMessage"
          :disabled="isProcessing"
        >
          <template #append>
            <el-button :icon="Promotion" @click="sendMessage" :disabled="isProcessing" />
          </template>
        </el-input>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, nextTick, onMounted, ref } from 'vue'
import { Close, More, Plus, Promotion, Search } from '@element-plus/icons-vue'
import axios from 'axios'
import { nanoid } from 'nanoid'

const STORAGE_KEY = 'chatfun-vue-sessions-v1'
const chatApiUrl = 'https://www.chatfun.site/api/python'
const titleApiUrl = 'https://www.chatfun.site/api/title'

const userInputValue = ref('')
const searchValue = ref('')
const messages = ref([])
const isProcessing = ref(false)
const seq = ref(0)
const currentChatId = ref('')
const sessions = ref({})
const currentChatTitle = ref('对话标题')
const isSidebarOpen = ref(false)
const messageListRef = ref(null)

let activeController = null

const chatList = computed(() =>
  Object.values(sessions.value).sort((a, b) => (b.updatedAtMs || 0) - (a.updatedAtMs || 0))
)

const filteredChatList = computed(() => {
  const keyword = searchValue.value.trim().toLowerCase()
  if (!keyword) return chatList.value

  return chatList.value.filter(chat => {
    return (
      chat.title.toLowerCase().includes(keyword) ||
      chat.lastMessage.toLowerCase().includes(keyword)
    )
  })
})

function nowText() {
  return new Date().toLocaleString('zh-CN', {
    timeZone: 'Asia/Shanghai',
    hour12: false
  })
}

function cloneMessages(source) {
  return source.map(message => ({ ...message }))
}

function maxSeq(source) {
  return source.reduce((max, message) => Math.max(max, Number(message.seq) || 0), 0)
}

function createInitialMessage(chatId) {
  return {
    id: nanoid(),
    ssid: chatId,
    seq: 0,
    user: 'ai',
    content: '你好，有什么可以帮助你的？',
    timestamp: nowText()
  }
}

function buildSession(id, title, sourceMessages, existing = {}) {
  const savedMessages = cloneMessages(sourceMessages)
  const lastMessage = savedMessages[savedMessages.length - 1]
  const nowMs = Date.now()

  return {
    id,
    title: title || '新对话',
    messages: savedMessages,
    lastMessage: lastMessage?.content || '新对话',
    createdAt: existing.createdAt || nowText(),
    createdAtMs: existing.createdAtMs || nowMs,
    updatedAt: nowText(),
    updatedAtMs: nowMs
  }
}

function persistSessions() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(sessions.value))
}

function saveCurrentSession() {
  if (!currentChatId.value) return

  const existing = sessions.value[currentChatId.value] || {}
  sessions.value = {
    ...sessions.value,
    [currentChatId.value]: buildSession(
      currentChatId.value,
      currentChatTitle.value,
      messages.value,
      existing
    )
  }
  persistSessions()
}

function activateSession(session) {
  currentChatId.value = session.id
  currentChatTitle.value = session.title || '新对话'
  messages.value = cloneMessages(session.messages || [])
  seq.value = maxSeq(messages.value) + 1
  userInputValue.value = ''
  isSidebarOpen.value = false
  scrollToBottom()
}

function loadSessions() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    const parsed = raw ? JSON.parse(raw) : {}
    const validSessions = Object.fromEntries(
      Object.entries(parsed).filter(([, session]) => session && Array.isArray(session.messages))
    )

    if (Object.keys(validSessions).length > 0) {
      sessions.value = validSessions
      activateSession(chatList.value[0])
      return
    }
  } catch (error) {
    console.error('Error loading local sessions:', error)
  }

  startNewChat()
}

function cancelActiveRequest() {
  if (activeController) {
    activeController.abort()
    activeController = null
  }
  isProcessing.value = false
}

function toggleSidebar() {
  isSidebarOpen.value = !isSidebarOpen.value
}

function startNewChat() {
  cancelActiveRequest()

  const chatId = nanoid()
  const initialMessage = createInitialMessage(chatId)

  currentChatId.value = chatId
  currentChatTitle.value = '新对话'
  messages.value = [initialMessage]
  seq.value = 1
  userInputValue.value = ''
  saveCurrentSession()
  scrollToBottom()
}

function selectChat(chatId) {
  if (chatId === currentChatId.value) {
    isSidebarOpen.value = false
    return
  }

  cancelActiveRequest()
  saveCurrentSession()

  const session = sessions.value[chatId]
  if (session) activateSession(session)
}

async function sendMessage() {
  const content = userInputValue.value.trim()
  if (!content || isProcessing.value) return

  isProcessing.value = true

  const userMessage = {
    id: nanoid(),
    ssid: currentChatId.value,
    seq: seq.value,
    user: 'user',
    content,
    timestamp: nowText()
  }
  seq.value += 1

  const aiMessage = {
    id: nanoid(),
    ssid: currentChatId.value,
    seq: seq.value,
    user: 'ai',
    content: '',
    timestamp: nowText()
  }
  seq.value += 1

  messages.value = [...messages.value, userMessage, aiMessage]
  userInputValue.value = ''
  saveCurrentSession()
  scrollToBottom()

  activeController = new AbortController()

  try {
    const response = await axios.post(
      chatApiUrl,
      {
        content,
        chatHistory: JSON.stringify(messages.value)
      },
      {
        responseType: 'text',
        signal: activeController.signal,
        onDownloadProgress: progressEvent => {
          const responseText =
            progressEvent.event?.target?.responseText ||
            progressEvent.currentTarget?.responseText ||
            ''

          if (responseText) {
            aiMessage.content = responseText
            replaceMessage(aiMessage)
            saveCurrentSession()
            scrollToBottom()
          }
        }
      }
    )

    if (!aiMessage.content && typeof response.data === 'string') {
      aiMessage.content = response.data
      replaceMessage(aiMessage)
    }

    await updateTitle()
  } catch (error) {
    if (error.code === 'ERR_CANCELED') return

    console.error('Error:', error)
    aiMessage.content = '抱歉，发生了错误。请稍后再试。'
    replaceMessage(aiMessage)
  } finally {
    activeController = null
    isProcessing.value = false
    saveCurrentSession()
    scrollToBottom()
  }
}

function replaceMessage(updatedMessage) {
  messages.value = messages.value.map(message =>
    message.id === updatedMessage.id ? { ...updatedMessage } : message
  )
}

async function updateTitle() {
  const nonEmptyMessages = messages.value.filter(message => message.content.trim())
  const firstUserMessage = messages.value.find(message => message.user === 'user')

  if (!firstUserMessage) {
    currentChatTitle.value = '新对话'
    saveCurrentSession()
    return
  }

  const fallbackTitle = firstUserMessage.content.slice(0, 18)

  try {
    const response = await axios.post(
      titleApiUrl,
      {
        content: nonEmptyMessages.slice(-4).map(message => message.content).join('\n')
      },
      { responseType: 'text' }
    )

    currentChatTitle.value = response.data?.trim() || fallbackTitle
  } catch (error) {
    console.error('Error updating title:', error)
    currentChatTitle.value = fallbackTitle
  }

  saveCurrentSession()
}

function parseMessage(message) {
  const parts = []
  const codeBlockRegex = /```(\w+)?\n([\s\S]*?)```/g
  let lastIndex = 0
  let match

  while ((match = codeBlockRegex.exec(message)) !== null) {
    if (match.index > lastIndex) {
      parts.push({
        isCode: false,
        content: formatText(message.slice(lastIndex, match.index))
      })
    }

    parts.push({
      isCode: true,
      language: match[1] || 'plaintext',
      content: match[2].trim()
    })

    lastIndex = match.index + match[0].length
  }

  if (lastIndex < message.length) {
    parts.push({
      isCode: false,
      content: formatText(message.slice(lastIndex))
    })
  }

  return parts
}

function formatText(text) {
  return escapeHtml(text).replace(/\n/g, '<br>')
}

function escapeHtml(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')
}

async function scrollToBottom() {
  await nextTick()
  if (messageListRef.value) {
    messageListRef.value.scrollTop = messageListRef.value.scrollHeight
  }
}

onMounted(() => {
  loadSessions()
})
</script>

<style scoped>
:deep(.hljs) {
  padding: 1em;
  border-radius: 5px;
  font-size: 0.9em;
  overflow-x: auto;
}
</style>
